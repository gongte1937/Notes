# Talentblocks Refactor Plan（分阶段）

> 适用范围：`/src`、`/pages/api`、`/tests`
>
> 目标原则：严格遵循 `coding-rules/coding-principles.md`，优先级为 **KISS → DRY → SOLID**。

## 1. 背景与目标

本计划基于你提供的 Next.js best practice 结构（`app + features + components + lib`）和当前项目现状制定，目标是：

1. 将路由层与业务层解耦：`src/app` 只保留页面装配、布局和 route handlers。
2. 建立 `src/features/*` 业务域分层，收敛分散在 `app/*`、`helper/*`、`redux/*` 的业务代码。
3. 明确"共享组件 vs 业务组件"边界，消除重复组件并建立防重复机制。
4. 拆分超大文件，降低复杂度与回归风险。
5. 在重构过程中保持可回滚、可测试、可持续迁移（不是一次性大爆炸）。

---

## 2. 现状问题（基于当前仓库扫描）

### 2.1 结构层问题

1. 路由目录中承载了大量业务逻辑与组件实现（例如 `src/app/browse/HireTalent.tsx`、`src/app/freelancer/*/page.tsx` 大体积文件）。
2. `src/helper` 承载了多域混合逻辑（格式化、API 请求、业务流程、排序、时间换算等）。
3. `src/helper/interfaces.ts` 汇集了跨多个业务域的类型定义，难以维护。
4. API 同时存在 `src/app/api/*` 和 `pages/api/*` 两套体系，演进路径不统一。

### 2.2 重复组件问题（已识别）

当前可明确看到的重复/近重复组件（同名或职责重叠）：

1. `bookingModal.tsx`（client/freelancer 双份）
2. `cancelationModal.tsx`（client/freelancer 双份）
3. `addBankAccountForm.tsx`（client/freelancer 双份）
4. `bankDetailModal.tsx`（client/freelancer 双份）
5. `paymentOption.tsx`（client/freelancer 双份）
6. `sidebar.tsx`（client/freelancer 双份）
7. `calendar.tsx`（client/freelancer timesheets 双份，且与 `components/ui/calendar.tsx` 命名冲突风险）
8. `individualModal.tsx`（client/freelancer 双份）
9. `timepicker.tsx`（weeklyschedule/manual 双份）
10. `progressbar.tsx`（weeklyschedule/manual 双份）
11. `profilecard.tsx`（browse 双份）

### 2.3 高优先拆分文件（体积与职责）

1. `src/app/browse/HireTalent.tsx`（2218 行）
2. `src/app/freelancer/resume/page.tsx`（1777 行）
3. `src/helper/helperFunctions.tsx`（1333 行）
4. `src/app/freelancer/workhistory/page.tsx`（1161 行）
5. `src/app/freelancer/personaldetail/page.tsx`（1051 行）
6. `src/helper/interfaces.ts`（977 行）
7. `src/app/client/companydetail/page.tsx`（984 行）

---

## 3. 目标结构（落地版）

```text
src/
  app/           # 路由层：page/layout/loading/error/not-found + route handlers
    (marketing)/
    (dashboard)/ # 可逐步把 client/freelancer 路由迁入该组
    api/

  features/      # 业务域
    auth/
    browse/
    booking/
    payments/
    profile/
    scheduling/
    timesheets/
    resumes/
    shared/      # 仅放跨 feature 的业务能力（非 UI）

  components/
    ui/          # design system primitives（已存在）
    layout/      # AppShell, Header, Sidebar, Nav 等
    common/      # 无业务语义的复合组件

  lib/
    db/
    http/
    auth/
    analytics/
    time/
    format/
    utils/
    env.ts

  styles/
  types/
  constants/
```

---

## 4. 设计与治理规则（防重复 + 原则落地）

### 4.1 组件分层判定（必须执行）

1. 只在单一路由使用：放 `app/**/_components`（不导出到全局）。
2. 单业务域复用：放 `features/<domain>/components`。
3. 跨业务域复用：放 `components/common` 或 `components/layout`。
4. Design system primitive：只放 `components/ui`。

### 4.2 防重复机制（工程化）

1. 新增组件前必须先查重：按组件职责和 props 模型，而不是仅文件名。
2. 建立组件目录清单：`docs/component-inventory.md`（每次新增组件同步更新）。
3. 增加重复检查脚本：`scripts/check-duplicate-components.mjs`。
4. CI 增加门禁：`npm run lint && npm run test:run && node scripts/check-duplicate-components.mjs`。
5. 建立命名约定避免误导：业务组件统一前缀（如 `TimesheetCalendar`，避免与 `ui/calendar` 同名）。

### 4.3 SOLID/DRY/KISS 的执行准则

1. **KISS**：优先拆文件与提纯函数，避免"先上复杂抽象"。
2. **DRY**：第三次重复前必须抽取共享实现（Rule of Three）。
3. **SOLID**：
   - **SRP**：页面组件只负责组装，业务逻辑下沉到 hooks/services。
   - **OCP**：角色差异使用配置策略（config + mapper），避免复制一个同构组件。
   - **DIP**：API 调用通过 feature api 层/`lib/http` 适配，不在组件中直接散落请求细节。

---

## 5. 分阶段重构计划

### Phase 0：基线与保护网（1 周）

#### 目标

在不改变业务行为前提下，建立"可安全重构"的基线与检查机制。

#### 主要任务

1. 冻结重构范围并建立模块 owner（按 feature）。
2. 补齐关键路径测试（browse、booking、timesheets、payments）。
3. 增加重复组件检测脚本与 CI gate。
4. 记录当前页面/接口行为快照（E2E + API smoke）。

#### 输出

1. `docs/refactor-tracking.md`（迁移追踪）
2. `scripts/check-duplicate-components.mjs`
3. CI 任务中接入重复检查

#### 验收标准

1. 主流程回归测试可稳定通过。
2. 新增同职责组件会在 CI 被拦截。

---

### Phase 1：搭建 features 骨架与边界（1 周）

#### 目标

引入 `features` 分层，但不立即大规模迁移业务。

#### 主要任务

1. 创建目录：`src/features/{auth,browse,booking,payments,profile,scheduling,timesheets,resumes,shared}`。
2. 每个 feature 建立 `index.ts`、`types.ts`、`components/`、`hooks/`、`api/`、`schemas/`（按需）。
3. 建立 import 边界约束：
   - `app` 可依赖 `features/components/lib/types/constants`
   - `features` 不反向依赖 `app`

#### 输出

1. feature 模板目录 + README（每个 feature 说明职责）
2. import 规则（ESLint `no-restricted-imports`）

#### 验收标准

新增代码优先落在 `features`，`app` 不再新增业务逻辑。

---

### Phase 2：重复组件收敛（2 周）

#### 目标

先解决重复组件和命名冲突，降低后续迁移成本。

#### 主要任务

1. 统一 booking modal：
   - `src/app/client/bookings/bookingModal.tsx`
   - `src/app/freelancer/bookings/bookingModal.tsx`
   - 合并为 `src/features/booking/components/BookingModal.tsx`
   - 角色差异移入 `src/features/booking/components/bookingModal.config.ts`

2. 统一 cancelation modal：两份 `cancelationModal.tsx` 合并为 `src/features/booking/components/CancellationModal.tsx`

3. 统一 payment 组件：`addBankAccountForm.tsx`、`bankDetailModal.tsx`、`paymentOption.tsx` 合并至 `src/features/payments/components/*`，角色差异通过 props + strategy（client/freelancer）

4. 统一 timesheets calendar/individual modal：
   - `src/app/client/timesheets/calendar.tsx`
   - `src/app/freelancer/timesheets/calendar.tsx`
   - 合并为 `src/features/timesheets/components/TimesheetCalendar.tsx`
   - `individualModal.tsx` 同步合并

5. 统一 sidebar：
   - `src/app/client/components/sidebar.tsx`
   - `src/app/freelancer/components/sidebar.tsx`
   - 合并为 `src/components/layout/Sidebar.tsx`（role-based nav config）

6. 统一 `timepicker/progressbar`：合并到 `src/features/scheduling/components/{TimePicker,ProgressBar}.tsx`

#### 输出

1. 删除重复组件源文件（以迁移 PR 为单位逐步删）
2. 建立统一导出入口：`features/*/index.ts`

#### 验收标准

1. 重复组件清单中的项目完成收敛。
2. 页面 UI/交互无行为回归（E2E 通过）。

---

### Phase 3：大文件拆分与业务下沉（2-3 周）

#### 目标

把"巨型 page/helper 文件"拆为容器 + 领域组件 + hooks/services。

#### 核心拆分清单

1. `src/app/browse/HireTalent.tsx` 拆分为：
   - `src/features/browse/components/HireTalentContainer.tsx`
   - `src/features/browse/components/filters/*`
   - `src/features/browse/components/results/*`
   - `src/features/browse/hooks/useBrowseFilters.ts`
   - `src/features/browse/hooks/useFreelancerSearch.ts`
   - `src/features/browse/api/browse.api.ts`
   - `src/app/browse/page.tsx` 仅保留装配

2. `src/app/freelancer/resume/page.tsx` 拆分为：
   - `src/features/resumes/components/ResumePageContainer.tsx`
   - `src/features/resumes/components/ResumeList.tsx`
   - `src/features/resumes/components/ResumeUploader.tsx`
   - `src/features/resumes/hooks/useResumeManager.ts`
   - `src/features/resumes/api/resume.api.ts`

3. `src/helper/helperFunctions.tsx`（按职责拆）拆分到：
   - `src/lib/time/*`（时间/时区换算）
   - `src/lib/format/*`（日期、字符串格式）
   - `src/features/booking/api/*`（booking draft/email/onsched 相关）
   - `src/features/browse/lib/sorters.ts`
   - 保留过渡层 `src/helper/helperFunctions.tsx`（deprecated re-export，最后删除）

4. `src/helper/interfaces.ts`（按域拆）拆分到：
   - `src/features/browse/types.ts`
   - `src/features/booking/types.ts`
   - `src/features/payments/types.ts`
   - `src/features/timesheets/types.ts`
   - `src/features/profile/types.ts`
   - `src/types/common.ts`（仅通用跨域类型）

5. 其他大 page（`workhistory/personaldetail/companydetail/skills/scheduling`）按同一模板拆：`Container + Section Components + Hooks + Feature API`。

#### 验收标准

1. 单文件行数建议控制在 300 行以内（特殊页面可例外，但需说明）。
2. 页面层不再直接包含复杂业务流程与第三方请求编排。

---

### Phase 4：API 与基础设施统一（1-2 周）

#### 目标

收敛 API 调用方式与运行时边界，减少散落请求逻辑。

#### 主要任务

1. 规划 `pages/api/*` 到 `app/api/*` 的迁移路线（按风险优先级分批）。
2. 新建 `src/lib/http`（统一 fetch 封装、错误处理、重试策略）。
3. feature 的请求统一放在 `features/<domain>/api/*`，UI 组件仅调用 feature api/hook。
4. 敏感配置统一入 `src/lib/env.ts`（zod 校验）。

#### 验收标准

1. 新增接口只允许落在 `app/api`。
2. 组件层无裸 `fetch/axios`（除极少数过渡代码并标注 TODO）。

---

### Phase 5：状态管理与表单验证收敛（1-2 周）

#### 目标

降低状态耦合，避免重复验证逻辑。

#### 主要任务

1. 评估 `redux/reducer/*` 与 feature state 的边界，按域迁移 slice。
2. 将 `src/helper/validators/*` 迁移到对应 `features/*/schemas`。
3. 保持"schema 单一事实源"：前后端复用同一验证模型（可复用 zod/yup schema）。

#### 验收标准

1. 业务验证逻辑不再散落在 page 与 helper 中重复出现。
2. 状态读写路径清晰（组件 -> hook -> store/api）。

---

### Phase 6：收尾与治理固化（1 周）

#### 目标

完成遗留清理并固化协作规则，防止架构回退。

#### 主要任务

1. 删除过渡兼容层（deprecated re-export）。
2. 统一文档：`docs/architecture.md`、`docs/component-inventory.md`、`docs/refactor-tracking.md`。
3. 设立 PR 检查清单：
   - 是否引入重复组件？
   - 是否违反 import 边界？
   - 是否新增了超过阈值的大文件？

#### 验收标准

1. 目录分层稳定，重构目标结构落地。
2. 重复组件检查、测试、lint 全部自动化。

---

## 6. 文件拆分与迁移总表（可直接作为执行 backlog）

| 当前文件 | 问题 | 目标位置 | 拆分方式 |
|---|---|---|---|
| `src/helper/interfaces.ts` | 多域类型耦合 | `src/features/*/types.ts` + `src/types/common.ts` | 先按业务域分组迁移，再删除集中式类型仓 |
| `src/helper/helperFunctions.tsx` | 工具/API/业务混杂 | `src/lib/time`、`src/lib/format`、`src/features/*/api` | 先抽纯函数，再抽副作用函数 |
| `src/app/browse/HireTalent.tsx` | 巨型页面 | `src/features/browse/*` | Container + Filters + Results + Hooks |
| `src/app/freelancer/resume/page.tsx` | 巨型页面 | `src/features/resumes/*` | 容器、上传、列表、hooks 分离 |
| `src/app/freelancer/workhistory/page.tsx` | 页面职责过重 | `src/features/profile/workhistory/*` | 逻辑转 hooks，UI 拆 section |
| `src/app/freelancer/personaldetail/page.tsx` | 表单逻辑耦合 | `src/features/profile/personaldetail/*` | schema + form components + mapper |
| `src/app/client/companydetail/page.tsx` | 客户资料逻辑集中 | `src/features/profile/company/*` | 按步骤组件拆分 |
| `src/app/client/bookings/bookingModal.tsx` + 对应 freelancer 文件 | 重复组件 | `src/features/booking/components/BookingModal.tsx` | 合并并参数化角色差异 |
| `src/app/client/bookings/cancelationModal.tsx` + 对应 freelancer 文件 | 重复组件 | `src/features/booking/components/CancellationModal.tsx` | 合并并抽文案策略 |
| `src/app/client/timesheets/calendar.tsx` + 对应 freelancer 文件 | 重复组件/命名冲突 | `src/features/timesheets/components/TimesheetCalendar.tsx` | 合并并统一 props |
| `src/app/client/components/sidebar.tsx` + `src/app/freelancer/components/sidebar.tsx` | 重复布局组件 | `src/components/layout/Sidebar.tsx` | nav 配置化（role config） |
| `src/app/client/addpaymentmethod/*` + `src/app/freelancer/marketplaceprofile/paymentdetails/*` | 支付组件重复 | `src/features/payments/components/*` | 表单、Modal、Option 分层复用 |

---

## 7. 风险与回滚策略

1. 每个 phase 使用小步 PR（建议每个 PR 只迁移 1 个 feature 或 1 组重复组件）。
2. 每个 PR 包含"迁移前后快照"与测试证据（至少 `npm run lint`、`npm run test:run`、关键 E2E）。
3. 高风险迁移（支付、排班）先做无行为改变的结构迁移，再做逻辑优化。
4. 为迁移中的旧导入提供短期 re-export 兼容层，并在 Phase 6 统一移除。

---

## 8. 执行建议顺序（最小风险路径）

1. Phase 0-1（护栏 + 骨架）
2. Phase 2（重复组件先收敛）
3. Phase 3（拆巨型文件）
4. Phase 4（API 收敛）
5. Phase 5（状态与校验统一）
6. Phase 6（收尾固化）

> 该顺序满足：先控风险再动结构；先消灭重复再抽象复用；先保证可读性（KISS）再追求复用（DRY）和可扩展（SOLID）。
