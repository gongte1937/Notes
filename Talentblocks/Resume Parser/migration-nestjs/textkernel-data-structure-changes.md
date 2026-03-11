# Textkernel 迁移后的数据结构变化说明

本文说明当 Resume Parser 从 Azure OpenAI + Azure OCR 切换到 Textkernel Parser 后，数据结构会发生的变化，以及如何保证现有 Marketplace 与数据库流程稳定。

相关文档：
- [migration-plan-cn.md](migration-plan-cn.md)
- [migration-plan-en.md](migration-plan-en.md)
- [03-data-models.md](../Resume Parser Documentation/03-data-models.md)
- [04-database-schema.md](../Resume Parser Documentation/04-database-schema.md)

## 一、结论先看

1. 对前端 API：建议保持 `ShortParseData` / `LongParseData` 结构不变。
2. 对后端内部：新增 `Textkernel 原始响应层` + `映射层`。
3. 对数据库：现有写库逻辑可基本保留，但字段来源和空值率会变化。
4. 最大变化点：日期粒度、技能字段、工作职责/成就、项目与证书结构。

---

## 二、当前（Azure 方案）与目标（Textkernel 方案）差异

### 1) 解析链路差异

- 旧链路：文件文本提取/OCR -> Azure OpenAI（JSON/CSV）-> 清洗 -> 写库
- 新链路：文件读取 -> Textkernel 解析 -> 映射到内部模型 -> 写库

### 2) 数据来源差异

- 旧链路的 `LongParseData` 由 Prompt 强约束输出，字段命名与格式由我们定义。
- 新链路由 Textkernel 返回其标准结构，字段命名、层级、枚举和空值策略由供应商定义。

### 3) 兼容策略

建议采用三层模型：

1. `TextkernelRawResponse`：完整保留供应商返回。
2. `NormalizedResumeData`：内部标准化中间层（适配不同解析引擎）。
3. `ShortParseData/LongParseData`：对 Marketplace 与数据库保持兼容的输出层。

---

## 三、建议新增的数据结构（NestJS 内部）

```typescript
interface TextkernelRawEnvelope {
  requestId?: string;
  parserVersion?: string;
  receivedAt: string;
  raw: unknown; // 原始 JSON，避免丢失供应商字段
}

interface MappingDiagnostics {
  warnings: string[];         // 例如：date_end missing, fallback to null
  droppedFields: string[];    // 原始有但当前模型未接入的字段
  confidenceHints?: Array<{
    field: string;
    score?: number;
  }>;
}

interface NormalizedResumeData {
  personal: Record<string, unknown>;
  workHistory: Array<Record<string, unknown>>;
  educationHistory: Array<Record<string, unknown>>;
  skills: Array<Record<string, unknown>>;
  certifications?: Array<Record<string, unknown>>;
  projects?: Array<Record<string, unknown>>;
  diagnostics?: MappingDiagnostics;
}
```

说明：
- `raw` 建议在日志或 `parsing_jobs.parsed_data` 中按需保留（注意脱敏和容量）。
- `diagnostics` 用于排查“为什么某字段为空/被丢弃”。

---

## 四、字段级变化（重点）

## 1) Short Parse 结构变化

目标仍输出 `ShortParseData`，但字段来源会变化。

| 目标字段（保持不变） | 旧来源（Azure OpenAI） | 新来源（Textkernel） | 变化说明 |
| --- | --- | --- | --- |
| `firstName` / `lastName` | Prompt JSON | 个人信息姓名节点 | 一般可映射，命名拆分规则需统一 |
| `address` / `city` / `stateProvince` / `country` | Prompt JSON | 地址节点 | 国家可能是全名或 ISO 码，需标准化 |
| `zipCode` | Prompt JSON（数字） | 地址邮编 | Textkernel 常为字符串，需转换与兜底 |
| `mobileNumber` | Prompt JSON + 自定义规范化 | 电话集合节点 | 需明确优先级：mobile > phone > null |
| `dateOfBirth` | Prompt JSON | 个人信息节点（可能缺失） | 空值率可能升高 |
| `gender` | Prompt JSON（仅明确提及时） | 可能无稳定字段 | 建议默认 `null`，不做推断 |

建议：
- `zipCode` 在内部先保留 string，再按现有 API 合约决定是否转 number。
- 手机号继续走统一规范化函数，保持前端行为一致。

## 2) Long Parse 结构变化

### work_history

| 目标字段 | 旧来源 | Textkernel 迁移后变化 |
| --- | --- | --- |
| `job_title` / `company` | Prompt JSON | 一般可映射，可能有标准化后的 title |
| `date_start` / `date_end` | Prompt JSON（YYYY-MM-DD） | 常出现年/月粒度，需补齐规则 |
| `responsibilities` | Prompt 输出数组 | 可能是单段文本或无该字段 |
| `achievements` | Prompt 输出数组 | 可能与 responsibilities 合并 |

建议规则：
- 日期补齐：仅有 YYYY-MM 时，统一补到当月 1 日或保留部分日期并在映射层处理。
- `responsibilities` / `achievements` 无法拆分时：
  - 优先放 `responsibilities`
  - `achievements` 置 `null`

### education_history

| 目标字段 | 旧来源 | Textkernel 迁移后变化 |
| --- | --- | --- |
| `institution` | Prompt JSON | 可映射 |
| `qualification` | Prompt JSON | 可能是 degree + level 的组合 |
| `field_of_study` | Prompt JSON | 可能缺失 |
| `date_start` / `date_end` | Prompt JSON | 同样面临部分日期粒度 |
| `grade` | Prompt JSON | 可能无对应字段 |

### skills

| 目标字段 | 旧来源 | Textkernel 迁移后变化 |
| --- | --- | --- |
| `skill_name` | Skills CSV | 一般可映射 |
| `skill_category` | Skills CSV | 类别体系可能不同（供应商 taxonomy） |
| `proficiency_level` | Skills CSV | 可能缺失或枚举不一致 |
| `years_of_experience` | Skills CSV（整数） | 可能为小数、区间或缺失 |

建议规则：
- `years_of_experience` 统一转 `number`，无法确定则 `0` 或 `null`（按现有写库逻辑选一套）。
- 类别不兼容时建立 `categoryMap`，未命中时回落 `Other` 或原值。

### certifications / projects

| 目标字段 | 旧来源 | Textkernel 迁移后变化 |
| --- | --- | --- |
| `certifications` | Prompt JSON | 可能存在结构差异，需字段对齐 |
| `projects` | Prompt JSON | 可能不存在单独 projects 节点 |

建议：
- 先定义“可选能力”：无稳定映射则返回 `null`，不阻塞主流程。

---

## 五、对数据库写入的影响

现有写库顺序可保留（见 [04-database-schema.md](../Resume Parser Documentation/04-database-schema.md)），主要变化在值的质量与空值比例：

1. `resume_work_experience`
- `responsibilities`/`achievements` 更可能为空或文本粒度变化。
- 日期字段更可能只有年/月，需要映射层补齐策略。

2. `resume_education`
- `grade` 可能缺失率上升。

3. `resume_skills_queue`
- `qualificationLevel` 与 `category` 枚举分布会变化。
- `experience` 需要统一整数化策略。

4. `resume_versions.parsed_content`
- 建议继续存内部兼容结构，不直接暴露原始 Textkernel 格式，避免前端联动改造。

---

## 六、建议的兼容输出规范

为减少前端和下游影响，建议在 `MappingService` 强制以下约束：

1. 响应字段名不变：继续使用现有 `ShortParseData/LongParseData`。
2. 空值规范不变：尽量使用 `null` 而不是缺字段。
3. 日期规范统一：尽可能输出 ISO 字符串；仅部分日期时走统一补齐规则。
4. 数值字段统一：`zipCode`、`years_of_experience` 的类型策略固定且可测试。
5. 字段兜底可追踪：所有 fallback 写入 `diagnostics.warnings`。

---

## 七、迁移实施清单（数据结构维度）

- [ ] 定义 `TextkernelRawResponse` TypeScript 接口（先宽后严）。
- [ ] 建立 `MappingService`，完成 raw -> normalized -> legacy model 转换。
- [ ] 编写字段映射表（source path -> target field -> transform rule）。
- [ ] 增加 contract tests：验证输出仍兼容 `ShortParseData/LongParseData`。
- [ ] 增加回归数据集：覆盖扫描件、缺失日期、弱结构化简历。
- [ ] 增加监控字段：映射失败率、关键字段空值率、skills 覆盖率。

---

## 八、建议新增文档（后续）

建议在本目录再补两份文档：

1. `textkernel-field-mapping-table.md`
- 逐字段映射（包含 transform 函数、默认值、测试样例）。

2. `textkernel-contract-test-cases.md`
- 覆盖成功、字段缺失、部分日期、异常响应、超时重试等测试场景。

> 一句话总结：迁移到 Textkernel 后，最大的工作不是“改接口”，而是“建立稳定可追踪的映射层”，确保外部数据契约不被破坏。
