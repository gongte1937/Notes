# Resume Parser 迁移计划：Azure Functions -> NestJS（双 Textkernel Endpoint 方案）

## 一、背景与最新决策

当前系统流程是 Marketplace 调用 Azure Functions 完成解析，再写入 Supabase。根据团队讨论，迁移方案更新为：

1. NestJS 后端调用 **Textkernel Resume Parse endpoint**（解析主简历信息）。
2. NestJS 后端调用 **Textkernel Skills Extract endpoint**（抽取技能）。
3. 先将 Resume Parse 的基本信息返回并填写到 frontend。
4. 再将 Skills Extract 结果做技能 mapping 后写入数据库。

> 核心原则：前端先拿到基础解析结果，技能与数据库写入作为后续步骤完成；变化集中在解析与映射层。

---

## 二、目标架构

```text
User uploads resume PDF/DOCX
  └──► S3 upload function
        └──► Supabase Storage
              └──► Marketplace API
                    └──► NestJS Resume Parser
                          ├──► Textkernel Parse Resume endpoint
                          ├──► Basic Info Adapter (fill frontend fields)
                          ├──► Textkernel Extract Skills endpoint
                          ├──► MappingService (skills mapping)
                          └──► DatabaseService (写入 Supabase)
```

---

## 三、迁移后流程（To-Be Flow）

1. 用户上传简历（建议入口限制 `pdf` / `docx`）。
2. 文件通过现有 S3 upload function 存入 Supabase Storage。
3. Marketplace 调用 NestJS parser API（保留现有认证协议）。
4. NestJS 从存储读取文件内容/文本。
5. 调用 Textkernel `parse resume` endpoint，获得主结构化信息。
6. 解析并回填基本信息到 frontend（如姓名、联系方式、教育/经历基础字段）。
7. 调用 Textkernel `extract skills` endpoint，获得技能结果。
8. `MappingService` 处理技能映射与标准化。
9. `DatabaseService` 按既有写库逻辑落库（resume/student/public 相关表）。
10. 前端继续按现有轮询流程等待完成并展示结果。

---

## 四、接口与数据处理设计

### 1) Textkernel 双接口调用

| 场景 | Endpoint | 目的 | 输出去向 |
| --- | --- | --- | --- |
| 主解析 | `parse resume` | 工作经历、教育、个人信息等主体结构 | frontend 基本信息字段 |
| 技能解析 | `extract skills` | 技能名称、等级、经验等技能信息 | skills mapping 后写入 DB（skills 相关表） |

### 2) 合并与映射策略

- 按顺序执行：先 `parse resume`，后 `extract skills`。
- `parse resume` 输出用于 frontend 基本信息回填，不再生成旧的中间解析结构。
- `extract skills` 结果进入技能 mapping 逻辑，统一技能命名、等级和经验字段。
- 映射异常写入 diagnostics 日志，避免 silent failure。

### 3) 数据库策略

- 现有 schema 保持不变。
- 现有写库规则保持不变（乐观锁、student 授权、非阻塞子表写入）。
- 写库触发点调整为：技能 mapping 完成后统一写入。
- 重点改动在 endpoint 编排与 `MappingService`，而不是数据库层。

---

## 五、NestJS 模块与职责

### Modules

- `AppModule`
- `ParseModule`
- `HealthModule`
- `ConfigModule`
- `HttpModule`

### Controllers

- `ParseController`
- `HealthController`

### Services

- `ParseService`：编排整体流程
- `TextkernelResumeService`：调用 parse resume endpoint 并抽取前端基础字段
- `FrontendPayloadService`：组织并返回 frontend 所需基本信息
- `TextkernelSkillsService`：调用 extract skills endpoint
- `MappingService`：技能数据映射与标准化
- `DatabaseService`：写入 Supabase
- `FileService`：读取文件/文本
- `ApiKeyAuthService`：认证

---

## 六、实施计划（按讨论任务拆分）

### 1) Research story

| 任务 | Owner |
| --- | --- |
| Test API calls（验证两个 Textkernel endpoint 能力） | Terry |

### 2) Textkernel API development & changes

| 任务 | Owner |
| --- | --- |
| Setup API call to parse a resume | Terry |
| Parse result to frontend basic info payload | Terry |
| Setup API call to extract skills from parsed resume text | Terry |
| Skills mapping and DB write integration | Terry |

### 3) Testing

| 任务 | Owner |
| --- | --- |
| Unit tests | Terry |
| Integration tests | Terry |
| E2E tests | Terry |
| UAT | Stefan |

### 4) Cleanup

| 任务 | Owner |
| --- | --- |
| Remove existing resume parser calls to Azure function | Terry |
| Remove existing Azure function infrastructure | Shane |
| Create a commercial account（Textkernel） | Shane |

---

## 七、分阶段迁移

### Phase 1：API 验证与最小闭环

- [ ] 打通 Textkernel 双 endpoint 调用。
- [ ] 拿到真实样本验证字段完整性。
- [ ] 完成 parse resume -> frontend 基本信息回填。
- [ ] 定义 skills mapping 规则初稿。

### Phase 2：后端集成

- [ ] 在 NestJS 引入 `TextkernelResumeService` / `TextkernelSkillsService`。
- [ ] 增加 `FrontendPayloadService` 返回基础信息。
- [ ] 完成技能 `MappingService` 并接入 `DatabaseService` 写库。

### Phase 3：测试与验收

- [ ] 单元测试（映射规则、错误处理、空值策略）。
- [ ] 集成测试（双 endpoint + DB）。
- [ ] E2E 测试（上传 -> 解析 -> 入库 -> 页面展示）。
- [ ] UAT 通过。

### Phase 4：切流与下线

- [ ] 移除 Marketplace 到旧 Azure Function 的调用。
- [ ] 分环境切流（dev -> test -> prod）。
- [ ] 观察稳定后移除旧 Azure Function 基础设施。

---

## 八、关键风险与控制

1. 双接口响应不一致
- 控制：定义顺序与职责边界（parse 负责基础信息，skills endpoint 负责技能域）。

2. 字段类型不一致（日期、经验、等级）
- 控制：在映射层统一转换并增加 contract tests。

3. 技能提取缺失或波动
- 控制：设置兜底策略（空数组/默认值）并记录 diagnostics。

4. 性能与超时
- 控制：分别设置 parse resume / extract skills 超时和重试；前端基础信息优先返回，技能与写库异步补齐。

---

## 九、待确认问题

1. 是否需要长期保留原始 resume PDF（合规/成本/追溯需求）？
2. Textkernel commercial account 的采购与开通时间点？
3. 若 Skills endpoint 失败，是否允许主流程继续并仅提示技能缺失？
4. frontend 基本信息回填完成后，技能写库状态如何在 UI 呈现（进行中/部分成功）？

---

## 十、回滚方案

迁移期间保留旧链路。若新链路异常：

1. Marketplace 切回旧 Azure Function URL。
2. 重新部署或热更新环境变量。
3. 旧流程立即接管流量。
