# MeetOps 前端产品化重设计规范（Refero 方案）

## 1. 文档状态与执行边界

- 状态：已实施的历史 UI 设计基线，仅保留信息架构与视觉决策；当前功能状态以 `SPEC.md`、`docs/16-exception-replanning-design.md`、`docs/17-pre-post-meeting-closure-design.md`、`docs/18-frontend-productization-and-demo-workspace.md` 和 `docs/HANDOFF.md` 为准。
- 保留原因：当前前端仍沿用本文的 Refero 布局、信息层级和视觉 Token；文中已被真实业务闭环取代的 Product Preview 描述不再是执行指令。
- 设计定版日期：2026-08-13（Asia/Shanghai）。
- 修改范围：`frontend/**`；实现完成后由主 Agent 更新 `docs/HANDOFF.md`。
- 本阶段不修改 Java、Python、Compose 拓扑或跨服务 API 契约。
- 本文替代本文件此前以 Cal.diy 和固定 40/60 双栏为主的方案；Cal.com 仅可作为信息密度参考，不再是主视觉模板。

本文不能覆盖 `SPEC.md` 的冻结决策。登录、Java 公共 API、SSE、候选方案、HITL、Run/checkpoint 恢复、会议 CRUD、会议室管理和安全 Trace 都是真实能力，视觉升级不得改变其语义。没有公共 API 支持的能力必须保持为清楚标记的 Product Preview，不得伪造成功。

## 2. 当前问题与重构目标

当前前端已经完成 Tailwind CSS、shadcn-vue 基础设施、产品壳、结构化候选、HITL、Trace 和响应式基础，但仍有明显的“功能测试台”观感：

- 智能编排使用固定 40/60 双栏，将聊天和结构化调试数据永久并排，主任务不够聚焦。
- 页面普遍使用“大标题 + 说明文字 + 白色边框卡片”，所有内容权重接近，缺少成熟 AI 应用的工作流层级。
- 会议室默认是两列静态资源卡片，用户不能一眼回答“哪个房间在什么时间可用”。
- 待确认页仍是空状态说明，没有形成审批工作台体验；但后端又确实没有跨 Run 待确认列表接口。
- Trace 组件已具备真实数据，却仍偏开发调试输出，没有形成普通用户与技术用户的渐进式信息披露。
- 导航没有成为 AI 会话入口和会话历史载体，智能编排仍像传统后台中的一个普通页面。

重构目标是把它升级为：

> 以 AI 会话为主入口、以会议资源时间轴为业务工作台、以显式确认保证安全、以可展开 Trace 证明技术链路的企业会议智能调度产品。

## 3. 已选 Refero 参考界面

Refero 页面用于研究真实产品的布局、信息层级和交互模式，不授权复制品牌、图片、Logo、字体文件或源代码。最终界面必须使用 MeetOps 自有文字、业务数据和组件。

### 3.1 Meta AI：应用壳与智能编排主界面

- Refero：[Meta AI — AI Assistant](https://refero.design/pages/0d258b4a-2867-4746-ab86-03518bc2a36b)
- 采用：窄左栏、宽对话画布、底部悬浮输入框、按需出现的右侧 Side Sheet、低干扰的消息排版。
- 不采用：Meta Logo、品牌渐变、媒体/Vibes 等无关入口。

### 3.2 Mangomint：会议与资源时间轴

- Refero：[Mangomint — Calendar](https://refero.design/pages/92ae5496-0aaf-4700-9a9c-fe03f9baf24a)
- 采用：日期导航、日/周切换、资源列 × 时间行、彩色占用块、点击空槽位后出现的右侧编辑器。
- 不采用：美容行业文案、销售模块、预约收款逻辑。

### 3.3 TravelPerk：待确认与审批工作台

- Refero：[TravelPerk — Approval Processes](https://refero.design/pages/9335a133-d103-4685-8152-eb2a0f7e5fc1)
- 采用：标题旁计数、搜索与筛选、紧凑审批卡、明确审核人/类型/状态、直接操作。
- 不采用：差旅政策、费用和多级审批模型。本项目仍只有 HITL ACCEPT/EDIT/REJECT。

### 3.4 n8n：Run 详情与 Trace

- Refero：[n8n — Activity / Timeline](https://refero.design/pages/1974172f-cafa-4873-96f9-8c50321e8d72)
- 采用：运行摘要、状态筛选、Activity Feed、Timeline/History、节点详情抽屉、成功/运行/等待/失败状态。
- 不采用：自由拖拽工作流画布。运行时 Agent 数量和 LangGraph 拓扑不得被前端虚构或修改。

### 3.5 Copy.ai：工作流模板仅作次级参考

- Refero：[Copy.ai — Workflow Templates](https://refero.design/pages/37159acc-20ab-47d2-9ae8-a09ca8a627bd)
- 仅采用：智能编排空状态中的少量任务模板/快捷示例。
- 不采用：营销落地页、大字号 Hero、模板商城或虚构工作流能力。

## 4. 统一设计语言

四套参考只提供布局模式，最终必须通过一套 MeetOps Design Token 统一：

| 语义 | 值 |
|---|---|
| 页面背景 | `#F7F7F5` |
| 内容背景 | `#FFFFFF` |
| 主文字 | `#18181B` |
| 次级文字 | `#71717A` |
| 边框 | `#E4E4E7` |
| 主色 | `#4F46E5` |
| 主色浅背景 | `#EEF2FF` |
| 成功 | `#16A34A` |
| 等待/警告 | `#D97706` |
| 错误/冲突 | `#DC2626` |
| 信息 | `#2563EB` |
| 卡片圆角 | `12px` |
| 输入框圆角 | `16px` |

- 字体：`Inter, PingFang SC, Microsoft YaHei, system-ui, sans-serif`。
- 页面标题 22–24px；正文 14px；辅助信息 12px。
- 间距只使用 4、8、12、16、24、32。
- 普通内容靠边框、分隔线和留白组织，只有悬浮输入框、Dialog、Sheet 使用明显阴影。
- 图标统一使用现有 `@lucide/vue`，禁止继续以 `✦ ✓ ▦ ⌂ ↻ ◫ ☰ ×` 等文本字符代替产品图标。
- 禁止大面积渐变、玻璃拟态、霓虹发光、无业务意义插画和装饰动画。
- 保持亮色主题为本阶段完整交付；暗色主题不是完成条件。

状态统一通过 `StatusBadge`：

- `SUCCESS / SUCCEEDED / CONFIRMED / COMPLETED / ACTIVE`：success。
- `WAITING_* / PENDING / PROCESSING`：warning。
- `RUNNING`：info，并提供不依赖动画的文字状态。
- `FAILED / CONFLICT / CANCELLED / INACTIVE`：destructive 或 neutral，按业务语义区分。
- 普通元数据：secondary。

## 5. 总体信息架构

桌面端采用三层结构，但同一时刻只让主任务占据中心：

```text
┌──────────────┬──────────────────────────────────┬──────────────────────┐
│ 左侧导航/会话 │ 主工作区                           │ 按需右侧 Sheet         │
│ 220–240px    │ 对话 / 时间轴 / 审批 / Trace       │ 需求、详情、HITL、节点 │
└──────────────┴──────────────────────────────────┴──────────────────────┘
```

侧栏顺序：

```text
MeetOps
工作台
  智能编排
  待我确认

协作
  我的会议
  会议室
  消息中心
  异常重排
  会前会后
  知识库

管理（仅管理员）
  员工管理

系统
  运行记录（没有列表 API 时不创建伪数据；从具体 Run 进入）

会话
  新建编排
  搜索会话
  最近任务（当前标签页可恢复的 thread/run）

底部
  用户、部门、角色、退出
```

会话操作跟在主要功能与当前运行入口之后，并与它们共享同一侧栏滚动区域；不再为会话区保留固定高度的顶部区域。

约束：

- 现有 `/login`、`/chat`、`/meetings`、`/rooms`、`/approvals`、`/agent/runs/:runId` 和 Preview 路由保持兼容。
- 路由切换不能丢失运行中的 Run、当前 thread 或本标签页历史。
- “最近任务”只能来自现有安全的本地会话上下文和真实 Run，不得生成伪记录。
- 1024px 以下侧栏改为 Sheet；移动端主区为单栏。

## 6. 页面规格

### 6.1 登录页

- 保持当前真实登录、失败、加载、redirect 和权限行为。
- 使用中性背景、紧凑居中表单和 MeetOps 文字标识。
- 不展示开发阶段文案、真实密码提示或与产品无关的营销内容。
- 390×844 下表单不溢出，键盘聚焦时主要按钮仍可操作。

### 6.2 智能编排：Meta AI 主模式

这是默认首页，也是最高优先级 Golden Path。

#### 主画布

- 删除固定 `40% conversation + 60% result` 的永久双栏。
- 中央对话列建议最大宽度 840–920px，较宽屏幕保持舒适阅读宽度。
- 历史消息使用轻量消息块，不给每条 Agent 回复套大卡片。
- 用户消息可以右对齐并使用浅主色背景；Agent 回答使用正文式布局。
- 当前 Run 状态以标题栏中的小型状态、进度条或 step chips 呈现，不显示大段调试说明。
- 页面底部使用悬浮 `AgentComposer`：16px 圆角、自动增长、发送键、字数/快捷键提示；运行中明确禁用重复提交。

#### 空状态

- 使用一句价值明确的欢迎语和 4–6 个真实可执行的快捷任务。
- 快捷任务覆盖创建、多人协调、会议室推荐、政策查询、改期和取消。
- Copy.ai 模板仅作为紧凑快捷卡参考，不做模板商城。

#### 结构化结果

- 需求解析、候选方案、政策依据和 Agent 执行过程全部进入右侧 `OrchestrationSheet`。
- Sheet 默认关闭；收到候选或进入 `WAITING_CONFIRMATION` 时可以自动打开一次，但用户关闭后不得反复强制打开。
- Sheet 内分 Tabs：`需求`、`候选`、`政策`、`执行`。
- 候选最多三项，显示时间、房间、楼宇、容量/设备、totalCost 和可理解的 costBreakdown；推荐项必须来自真实排序。
- 引用只展示真实 `citation`；没有引用则显示“未找到可验证证据”，不得让模型自由补出处。

#### HITL

- `WAITING_CONFIRMATION` 时，在右侧 Sheet 内显示完整草案，同时在输入框上方显示紧凑待确认提示。
- CREATE：会议草案摘要。
- RESCHEDULE：明确 Before/After。
- CANCEL：明确目标会议，不提供无意义的时间编辑。
- 操作保持 `接受并执行 / 编辑后重新规划 / 拒绝`。
- EDIT 必须继续调用现有 resume 契约；不能直接写正式会议。
- 倒计时来自真实 `expiresAt`，过期后按钮禁用并提示重新生成。

#### 会话和运行恢复

- 必须保留现有 `X-Run-Id` 响应头捕获、稳定 client thread、sessionStorage 会话历史、per-run context、路由 runId 恢复、RUNNING 轮询和 recovery epoch。
- 从“我的会议/会议室/待确认”返回智能编排时，不能像新会话一样清空。
- “新建编排”才显式创建新 thread 并清除当前 run query。

### 6.3 我的会议：Mangomint 日历 + 列表

- 默认提供 `日历 / 列表` 切换；移动端默认列表。
- 顶部工具条：今天、前后日期、日期标题、日/周切换、状态筛选、创建会议。
- 日历使用时间行 × 日期/资源列以及彩色会议块；只根据已经加载的真实数据绘制。
- 当前 API 不足以支撑跨大范围数据时，不得伪造完整月历；明确限制当前窗口。
- 点击会议块打开右侧详情 Sheet；编辑继续使用真实更新 API，取消使用确认 Dialog。
- 来源 `MANUAL / AGENT`、会议状态、房间和参会者保持清楚。
- 列表视图参考成熟 SaaS 的紧凑密度，不继续堆大卡片。

### 6.4 会议室：资源时间轴优先

- 默认视图从“两列静态房间卡片”改为 `资源时间轴`。
- 顶部筛选：楼宇、楼层、容量、设备、房型、日期、时间窗口、仅看可用。
- 第一列固定显示房间名称/容量/设备摘要；右侧按 30 分钟 `[start,end)` 槽位横向展示。
- 空闲槽位使用浅色可点击区域；占用块展示允许公开的会议摘要；维护/停用房不可预约。
- 点击房间名打开详情 Sheet；点击空闲槽位打开创建会议 Sheet，并预填房间和时间。
- 提供 `时间轴 / 房间目录` 切换；现有房间卡片可保留在目录视图，但降低视觉体积。
- ADMIN 的新建、编辑、启停操作与 EMPLOYEE 只读状态严格区分。
- 不引入楼层地图，除非后端/演示资产有真实布局数据。

### 6.5 待我确认：TravelPerk 审批模式

当前后端没有跨 Run 待确认列表 API，设计必须诚实：

- 如果当前标签页有可恢复的真实 `WAITING_CONFIRMATION` Run，显示为审批卡。
- 否则显示紧凑空状态，并引导回“智能编排”；不显示伪造数量或假任务。
- 顶部保留标题、真实计数，以及在有数据时启用的类型/状态筛选。
- CREATE/RESCHEDULE/CANCEL 分别显示不同摘要，操作和智能编排内 HITL 共用组件与逻辑。
- 不新增多级审批、审核人配置或审批流程创建功能。

### 6.6 Agent Run 与 Trace：n8n Activity 模式

普通用户先看：

```text
✓ 理解会议需求
✓ 查询参会者时间
✓ 检索会议制度
● 正在求解候选方案
○ 等待用户确认
○ 执行业务写入
```

技术详情页：

- 顶部显示状态、intent、模型、Prompt/Schema 版本、总耗时、model/tool 调用、Token、runId 和 traceId。
- 中间是按时间排列的 Agent Step、Loop Event 和 Tool Call Activity Feed。
- 点击一项在右侧 Sheet 查看安全输入摘要、输出摘要、riskLevel、错误码、耗时和幂等键摘要。
- 提供 `全部 / Agent / Tool / Loop / 错误` 过滤。
- 不显示隐藏推理、完整 Prompt、confirmationToken、JWT、Service Token 或完整敏感正文。
- TraceDrawer 与完整 Run 页面复用相同 presenter/组件，不维护两套状态翻译。

### 6.7 Product Preview

- 异常重排和会前会后继续保留 Preview 标识。
- Preview 数据必须放在 `frontend/src/demo/**` 或等价明确目录。
- 操作只能提示“尚未连接后端，未执行写操作”，不得模拟成功落库。
- Preview 不得占据主导航最显眼位置。

## 7. 组件与代码组织

保留并重构现有组件，目标结构至少包括：

```text
WorkspaceShell
ConversationSidebar
ConversationCanvas
AgentComposer
OrchestrationSheet
RequirementSummary
CandidateComparison
PolicyCitations
HitlReviewPanel
MeetingCalendar
ResourceTimeline
RoomDirectory
ApprovalCard
RunOverview
ActivityTimeline
TraceDetailSheet
StatusBadge
PageHeader
EmptyState / LoadingState / ErrorState
```

- 不把新逻辑继续堆进 `ChatView.vue` 和单个巨型 `styles.css`。
- 将 Token、壳层、聊天、管理页、Trace 和响应式样式拆分到可维护文件，或优先使用现有 shadcn-vue primitives + Tailwind utilities。
- 优先保留 `src/api/client.ts`、`src/api/types.ts`、`src/api/agent-view.ts`、auth、SSE 解析和恢复逻辑。
- 禁止为了换 UI 重写协议解析或删除兼容字段。
- 所有 icon-only button 必须有 `aria-label` 和可见 focus ring。

## 8. 分阶段实施

### 阶段 0：真实基线

- 记录当前登录、聊天历史、运行中切页恢复、政策问答、候选、HITL、Trace、会议 CRUD 和房间 ADMIN 的行为。
- 运行 `npm ci`、`npm run type-check`、`npm run build`。

### 阶段 1：Design Token 与 WorkspaceShell

- 更新为本文颜色、字体、圆角和状态语言。
- 用 Lucide 替换文本图标。
- 把侧栏升级为“新建编排 + 最近任务 + 产品导航”，保留响应式 Sheet。

### 阶段 2：智能编排 Golden Path

- 移除固定 40/60 布局。
- 实现 ConversationCanvas、悬浮 AgentComposer 和 OrchestrationSheet。
- 将需求、候选、政策、执行和 HITL 迁入 Sheet。
- 回归会话历史、Run URL、切页和刷新恢复。

### 阶段 3：会议和会议室

- 实现会议日历/列表切换。
- 实现资源时间轴/房间目录切换。
- 将创建、详情、编辑放入 Sheet/Dialog，保护权限和 CRUD。

### 阶段 4：待确认与 Trace

- 使用真实当前 Run 构建审批卡或诚实空状态。
- 建立普通进度和技术 Activity 两层视图。

### 阶段 5：响应式、无障碍与清理

- 覆盖 1440×900、1024×768、390×844。
- 检查键盘、焦点、Esc、滚动锁定、触屏目标、中文换行和横向溢出。
- 清理无引用 CSS、文本图标、开发阶段标签和乱码。

## 9. 验收标准

### 9.1 功能红线

- 浏览器仍只访问 Java `/api/v1/**`。
- SSE、HITL、异步结果和 Run 恢复没有回归。
- 切换左侧页面再返回，当前会话和运行状态仍在。
- ACCEPT 前无正式会议副作用；EDIT 仍重新校验；REJECT 不写入。
- 会议和房间 ADMIN/EMPLOYEE 权限不变。
- Product Preview 不伪造写入。
- 不展示隐藏推理和敏感凭据。

### 9.2 视觉完成条件

- 智能编排不再呈现固定双栏测试台。
- 会议室默认能回答“某时间哪些房间可用”。
- 我的会议具备可扫描的日历/列表结构。
- 待确认使用审批卡或真实空状态，而不是说明文档式页面。
- Trace 具备普通进度和技术详情两层信息。
- 页面主操作在 3 秒内可定位；同一页面只保留一个主要 CTA。
- 1440、1024 和 390 宽度无横向滚动和遮挡。

### 9.3 自动验证

```powershell
Push-Location D:\agent\frontend
npm ci
npm run type-check
npm run build
Pop-Location

docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml config --quiet
```

若变更影响聊天、HITL、恢复、会议或会议室交互，还要执行仓库当前可用的对应 Smoke，并以 `scripts/` 实际文件为准核对命令，不得凭旧文档编造脚本。

### 9.4 浏览器验收矩阵

- 视口：1440×900、1024×768、390×844。
- 身份：EMPLOYEE、ADMIN。
- 状态：loading、empty、error、disabled、SSE streaming、RUNNING、WAITING_CONFIRMATION、WAITING_BUSINESS_RESULT、SUCCESS、CONFLICT、FAILED。
- 流程：新建会话、连续两轮问答、运行中切页返回、刷新恢复、候选、ACCEPT/EDIT/REJECT、Trace、会议 CRUD、房间可用性和 ADMIN 管理。
- 交互：Tab、Shift+Tab、Enter、Esc、焦点返回、Sheet/Dialog 滚动锁定、触屏按钮尺寸。

## 10. 明确不做

- 不复制 Refero 截图、品牌资产或第三方源码。
- 不为了“像 AI”增加虚假思考过程或隐藏推理。
- 不增加真实邮件、真实日历/视频供应商、IoT、SSO、多租户、多级审批或自动移动他人会议。
- 不虚构跨 Run 待确认列表、楼层平面图、利用率 KPI 或运行记录列表。
- 不改变 Agent 数量、后端写入语义、30 分钟槽位和时区规则。
