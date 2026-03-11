# PRD：NestJS Resume Parser 新解析流程

## 一、背景

当前 Resume Parser 运行在 Azure Functions 上，使用 Azure OpenAI（GPT-5-mini）进行解析，通过 OCR（Azure Document Intelligence）提取文本。此次迁移目标是将解析流程从 Azure Functions 搬到 NestJS，并将 AI 解析引擎从 Azure OpenAI 切换到 **Textkernel**（双 endpoint 方案）。

迁移后架构：

```text
User uploads resume
  └──► S3 upload function
        └──► Supabase Storage
              └──► Marketplace API
                    └──► NestJS Resume Parser
                          ├──► Textkernel Parse Resume endpoint
                          ├──► FrontendPayloadService（返回基础信息）
                          ├──► Textkernel Extract Skills endpoint
                          ├──► MappingService（技能标准化）
                          └──► DatabaseService（写入 Supabase）
```

## 二、目标

- 用 NestJS 替换 Azure Functions 作为解析服务运行时。
- 用 Textkernel 替换 Azure OpenAI 作为解析引擎。
- 保持对 Marketplace 的 API 契约不变（保留现有 `ShortParseData` / `LongParseData` 输出结构）。
- 保持数据库 schema 和写库行为不变。
- 完成后下线旧 Azure Functions 基础设施。

## 三、核心原则

1. **前端优先返回**：先调用 `parse resume` 并将基础信息返回给前端，再异步完成技能解析与写库。
2. **不破坏下游**：Marketplace 和数据库侧的接口、schema、写库行为均不变。
3. **变化集中在映射层**：从 Textkernel 到内部模型的转换逻辑封装在 `MappingService`，隔离数据变化影响。
4. **可追踪的映射**：所有字段 fallback 和映射异常写入 `diagnostics`，不允许 silent failure。

---

## Phase 1：TextKernel API Development

### 目标

在 NestJS 中建立完整的双 endpoint 调用链路，实现从 Textkernel 原始响应到内部模型的映射，并完成数据库写入。

### Task 1：Create API call to parse resume

**目的**：调用 Textkernel `parse resume` endpoint，将结构化简历信息映射为 `ShortParseData`（基础信息）并返回给前端。

**工作内容**：

- 在 `TextkernelResumeService` 中实现对 Textkernel `parse resume` endpoint 的 HTTP 调用。
- 处理认证（API key）、请求构造（传入文件内容或文本）、超时与重试策略。
- 将 Textkernel 原始响应（`TextkernelRawEnvelope`）映射为 `ShortParseData`：

  | 目标字段 | 映射规则 |
  | --- | --- |
  | `firstName` / `lastName` | 从个人信息姓名节点提取，统一拆分规则 |
  | `address` / `city` / `stateProvince` / `country` | 从地址节点映射，国家标准化为一致格式 |
  | `zipCode` | 内部保留 string，输出时按合约转换 |
  | `mobileNumber` | 优先级：mobile > phone > null，继续走规范化函数 |
  | `dateOfBirth` | 可能缺失，映射为 null |
  | `gender` | 无稳定字段时默认 null，不做推断 |

- 通过 `FrontendPayloadService` 组装并返回前端所需的基础信息 payload。
- 所有映射 fallback 写入 `MappingDiagnostics.warnings`。

**验收标准**：

- 使用真实简历样本，`parse resume` 可成功调用并返回 `ShortParseData`。
- 姓名、联系方式、地址等核心字段正确填充。
- 缺失字段返回 `null`，不返回 `undefined`。
- 超时或调用失败时返回 500 并记录日志。

---

### Task 2：Create API call to extract skills

**目的**：调用 Textkernel `extract skills` endpoint，获取技能结构化数据，为 mapping 和写库做准备。

**工作内容**：

- 在 `TextkernelSkillsService` 中实现对 Textkernel `extract skills` endpoint 的 HTTP 调用。
- 输入为 `parse resume` 返回的简历文本内容（通过 `ParseService` 传递）。
- 获得技能列表（含 `skill_name`、`skill_category`、`proficiency_level`、`years_of_experience`）。
- 若调用失败：继续主流程，技能返回空数组，写入 diagnostics 记录。
- 设置独立的超时和重试策略（不依赖 `parse resume` 调用的成败）。

**验收标准**：

- 技能 endpoint 可正常调用并返回技能列表。
- 调用失败时不阻断主流程，返回空技能并记录原因。
- 技能字段类型符合预期（`years_of_experience` 为数字类型）。

---

### Task 3：Create mapping function for TextKernel -> Supabase

**目的**：将 Textkernel 原始技能数据通过 `MappingService` 标准化，然后通过 `DatabaseService` 写入 Supabase，保持与现有写库行为一致。

**工作内容**：

- 在 `MappingService` 中实现 Textkernel 技能数据 -> 内部 `LongParseData` 的映射：

  | 目标字段 | 映射规则 |
  | --- | --- |
  | `skill_name` | trimmed，非 null |
  | `skill_category` | 建立 categoryMap，未命中回落 `Other` 或原值 |
  | `proficiency_level` | 统一枚举：`beginner / intermediate / advanced / expert` |
  | `years_of_experience` | 统一转整数；无法确定则 `0` 或 `null`（保持现有写库逻辑一致） |

- 在 `MappingService` 中同步实现 `parse resume` 结果到 `work_history` / `education_history` 的映射：

  | 目标字段 | 映射规则 |
  | --- | --- |
  | `date_start` / `date_end` | 补齐策略：仅 YYYY-MM 时统一补至当月 1 日（ISO 格式） |
  | `responsibilities` | 无法拆分时优先放 `responsibilities`，`achievements` 置 null |
  | `qualification` | 可能是 degree + level 组合，统一拆分规则 |

- 通过 `DatabaseService` 按现有写库顺序落库（保持乐观锁、非阻塞子表写入等行为不变）：

  ```
  1. resume_versions         UPDATE（阻塞）
  2. parsing_jobs            UPDATE（非阻塞）
  3. resume_education        INSERT（非阻塞）
  4. resume_work_experience  INSERT（非阻塞，返回 ID）
  5. freelance_work_history  INSERT（非阻塞）
  6. resume_skills_queue     INSERT（非阻塞）
  7. student_*               INSERT（非阻塞，isStudent = true 时）
  ```

- `resume_versions.parsed_content` 存储内部兼容结构（`NormalizedResumeData`），不直接暴露 Textkernel 原始格式。

**验收标准**：

- 映射完成后所有必要表均写入成功。
- 输出 `ShortParseData` / `LongParseData` 与现有结构完全兼容（字段名、类型、null 策略）。
- `diagnostics.warnings` 记录所有 fallback 路径。
- 乐观锁、student 授权等现有行为不变。

---

## Phase 2：Testing

### 目标

确保新解析流程的正确性、稳定性，覆盖正常路径、边界条件和错误场景，通过 UAT 验收。

### Task 1：Unit testing

覆盖范围：

- `MappingService`：字段映射规则、日期补齐、技能类别匹配、fallback 路径。
- `TextkernelResumeService` / `TextkernelSkillsService`：模拟 HTTP 响应，测试成功/超时/失败处理。
- `FrontendPayloadService`：ShortParseData 组装逻辑。
- `DatabaseService`：mock Supabase client，验证写库调用顺序和参数。

关键测试用例：

- 日期仅有 YYYY-MM 时，补齐为 YYYY-MM-01。
- `years_of_experience` 为小数时，取整。
- `skill_category` 未命中 categoryMap 时，回落 `Other`。
- skills endpoint 失败时，返回空数组不抛错。
- 乐观锁冲突时，返回对应错误信息。

### Task 2：Integration testing

覆盖范围：

- Textkernel `parse resume` + `extract skills` 双 endpoint 串联调用（可使用 sandbox/mock server）。
- MappingService -> DatabaseService 完整写库链路。
- 认证失败（无效 API key）时的错误响应。
- isStudent = true 时，student 表正确写入。

关键场景：

- 正常简历：双 endpoint 均成功，所有表写入完成。
- skills endpoint 失败：主流程继续，技能为空，diagnostics 记录。
- 乐观锁冲突：返回 409 并记录日志。

### Task 3：E2E testing

覆盖范围：

- 完整链路：上传简历 -> Marketplace 调用 NestJS -> Textkernel 解析 -> Supabase 写入 -> 前端页面展示。
- 前端 ShortParseData 字段正确填充。
- 前端轮询 `parsing_jobs` 状态，最终达到 `completed`。
- 扫描件（低质量文本）、缺失字段简历的处理行为。

UAT（Stefan）：

- 真实简历样本验收。
- 前端页面各字段展示符合预期。
- 技能、工作经历、教育经历无明显遗漏或乱码。

---

## Phase 3：Deployment Cleanup

### 目标

在新流程稳定上线后，移除旧 Azure Functions 调用路径和基础设施，完成 Textkernel 商业账号的正式开通。

### Task 1：Remove existing resume parser calls to Azure Functions

**Owner**：Terry

- 在 Marketplace（Next.js）中移除所有指向旧 Azure Functions parser 的 HTTP 调用。
- 包括：`/api/parse/short`、`/api/parse/long/start` 等路由中的请求目标。
- 切换为调用 NestJS parser API（保持现有 API 契约不变）。
- 分环境执行：dev -> test -> prod，每个环境观察稳定后再继续。
- 保留回滚能力：迁移完成前保留旧 Azure Function URL 的环境变量配置，随时可切回。

**验收标准**：

- Marketplace 所有环境已无对旧 Azure Function 的调用。
- 新流程在 dev / test / prod 均运行稳定。

---

### Task 2：Remove existing Azure Function infrastructure

**Owner**：Shane

- 下线以下资源（确认所有环境均切流完毕后操作）：
  - `af-dv-resume-parser-dev.azurewebsites.net`
  - `af-dv-resume-parser-test.azurewebsites.net`
  - `af-dv-resume-parser-prod.azurewebsites.net`
- 清理对应的 Azure Key Vault secret（parser 专用 secrets，不影响 S3 Functions）。
- 保留 S3 Functions 服务（`af-dv-s3-functions`）——NestJS 仍需通过它读取 Supabase Storage 文件。
- 归档或删除 `pvt-talentblocks-resume-parser` 代码仓库（按团队决策）。

**验收标准**：

- 旧 Azure Function 资源已从 Azure Portal 移除。
- 相关 CI/CD pipeline（`deploy-dev.yml`、`deploy-test.yml`）已停用或归档。
- 无残留账单资源。

---

### Task 3：Create TextKernel commercial account

**Owner**：Shane

- 完成 Textkernel 商业账号的采购与开通。
- 获取生产环境 API key，更新到各环境的 Secret 管理（对应 NestJS ConfigModule）。
- 确认 Textkernel 商业账号的 SLA、rate limit 和计费模型，与工程侧对齐超时/重试配置。
- 确认 Textkernel 数据处理合规要求（是否需要 DPA）。

**验收标准**：

- 生产环境 API key 已配置且可用。
- 商业账号 SLA 和 rate limit 已同步给工程团队。

---

## 四、数据结构约束（MappingService 强制约束）

为保持下游兼容性，`MappingService` 必须遵守以下输出规范：

1. 响应字段名不变：继续使用 `ShortParseData` / `LongParseData` 结构。
2. 空值规范：缺失字段统一返回 `null`，不返回 `undefined` 或缺字段。
3. 日期规范：输出 ISO 字符串（`YYYY-MM-DD`）；仅有年月时统一补到当月 1 日。
4. 数值字段：`zipCode` 和 `years_of_experience` 类型固定且可测试。
5. Fallback 可追踪：所有 fallback 路径写入 `diagnostics.warnings`。

内部三层模型：

```typescript
// 层 1：保留 Textkernel 原始响应
interface TextkernelRawEnvelope {
  requestId?: string;
  parserVersion?: string;
  receivedAt: string;
  raw: unknown;
}

// 层 2：内部标准化中间层
interface NormalizedResumeData {
  personal: Record<string, unknown>;
  workHistory: Array<Record<string, unknown>>;
  educationHistory: Array<Record<string, unknown>>;
  skills: Array<Record<string, unknown>>;
  certifications?: Array<Record<string, unknown>>;
  projects?: Array<Record<string, unknown>>;
  diagnostics?: MappingDiagnostics;
}

// 层 3：对外兼容输出（不变）
// ShortParseData / LongParseData（见现有 Data Models）
```

---

## 五、风险与控制

| 风险 | 控制措施 |
| --- | --- |
| Textkernel 响应字段与现有模型不兼容 | 三层模型隔离，MappingService 统一转换，contract tests 覆盖 |
| 日期/数值类型不一致 | 映射层强制标准化，单元测试覆盖边界值 |
| 技能提取结果缺失或不稳定 | 失败时返回空数组，不阻断主流程，diagnostics 记录 |
| 性能/超时 | parse resume 与 extract skills 分别设置超时和重试；基础信息优先返回 |
| 切流期间双链路并存 | 保留旧 Azure Function URL 为回退入口，切流完毕后再下线 |

---

## 六、回滚方案

切流期间保留旧链路：

1. Marketplace 将 parser URL 切回旧 Azure Function。
2. 重新部署或热更新环境变量。
3. 旧流程立即接管流量，不需要代码改动。

---

## 七、待确认问题

1. Textkernel `extract skills` 的输入是原始文本还是 `parse resume` 的结构化输出？
2. Textkernel commercial account 的采购时间线，是否影响 prod 上线节点？
3. 若 skills endpoint 调用失败，前端 UI 如何呈现技能状态（进行中 / 部分成功）？
4. 是否需要长期保留原始 resume 文件（合规/追溯需求）？
5. NestJS 部署目标（Azure Container Apps / App Service / 其他）及对应 CI/CD 配置？
