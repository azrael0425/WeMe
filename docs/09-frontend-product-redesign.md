# 前端产品化重设计规范

## 1. 文档状态

- 状态：已设计，尚未实施。
- 目标阶段：Day 8 / 前端产品化升级。
- 参考产品：[Cal.diy](https://github.com/calcom/cal.diy)。
- 组件基础：[shadcn-vue](https://github.com/unovue/shadcn-vue)。
- 修改范围：`frontend/**`；实现完成后由主 Agent 更新 `docs/HANDOFF.md`。
- 本阶段不修改 Java、Python、Mock 服务、Compose 拓扑或跨服务 API 契约。

本规范不能覆盖 `SPEC.md` 的冻结决策。现有登录、Java 公共 API、SSE、HITL、Run 恢复、会议管理、会议室管理和安全 Trace 均属于真实能力，视觉升级不得改变其语义。尚无后端支持的未来能力只能作为明确标注的 Product Preview 展示，不能伪造成已经执行成功的业务功能。

## 2. 产品定位

产品工作名：`MeetOps 企业协作编排助手`。

前端需要同时做到：

1. 让普通用户以成熟企业 SaaS 的方式完成会议调度和管理。
2. 让面试或技术演示能够看见 Multi-Agent、Tool、HITL、并发裁决和恢复链路。
3. 为后续“复杂会议编排、异常重排、会前准备与会后行动”预留产品界面，但不提前伪造后端能力。

## 3. 设计原则

### 3.1 借鉴 Cal.diy 的内容

- 紧凑、克制、低装饰的企业 SaaS 信息密度。
- 可折叠侧边导航、顶部页面标题、面包屑和用户菜单。
- 以细边框、留白和文字层级组织内容，而不是大面积色块。
- 列表、日历、详情抽屉和设置式分区。
- 清晰的 loading、empty、error、disabled 和危险操作确认。

### 3.2 不直接复制的内容

- Cal.com 的商标、Logo、品牌文案、插画和 React 组件代码。
- 与当前产品无关的公开预约链接、支付、App Store、团队轮询等信息架构。
- 大量无真实数据支撑的 KPI、图表和企业功能入口。

### 3.3 Agent 界面原则

- 业务结果优先，Agent Trace 次级展示。
- 不展示隐藏推理、完整 Prompt、Token、JWT、Service Token 或敏感正文。
- 只展示 Agent 名称、结构化步骤、工具摘要、引用、耗时、风险等级和业务结果。
- 所有写操作都必须保留 ACCEPT / EDIT / REJECT HITL。
- Product Preview 页面和真实页面必须在视觉与文案上明确区分。

## 4. 信息架构

侧边栏按以下分组：

```text
工作台
  智能编排
  待我确认

协作
  我的会议
  会议室资源

系统
  Agent 运行详情（从具体 Run 进入）

产品预览
  异常重排
  会前会后

底部
  当前用户、部门、角色、退出
```

路由要求：

- 保留 `/login`、`/chat`、`/meetings`、`/rooms` 和 `/agent/runs/:runId`。
- 允许增加 `/approvals`、`/preview/replan`、`/preview/meeting-lifecycle`。
- `/` 继续进入真实的智能编排页面。
- 不存在 Agent Run 列表接口时，不创建伪造的“全部运行”页面。

## 5. 视觉系统

### 5.1 基础 Token

推荐亮色主题：

| 语义 | 推荐值 |
|---|---|
| 页面背景 | `#f8f9fa` |
| 卡片/浮层背景 | `#ffffff` |
| 主文字 | `#111827` |
| 次级文字 | `#6b7280` |
| 边框/Input | `#e5e7eb` |
| 主按钮 | `#292929` |
| Hover/Accent | `#f3f4f6` |
| 基础圆角 | `0.5rem` |

- 字体：Inter、`Microsoft YaHei`、system-ui。
- 页面标题：20–24px；正文：14px；辅助信息：12px。
- 间距尺度：4、8、12、16、24、32。
- 普通卡片以边框为主，不使用厚阴影；Sheet、Dialog、Dropdown 才使用明显阴影。
- 禁止大面积渐变、玻璃拟态、霓虹发光和装饰性动画。
- 第一阶段只完整实现亮色主题；可以保留 dark token，但不以深色切换为交付条件。

### 5.2 状态映射

所有页面通过统一 `StatusBadge` 映射：

- `SUCCESS`、`CONFIRMED`、`COMPLETED`：success。
- `WAITING_*`、`PENDING`、`PROCESSING`：warning。
- `FAILED`、`CONFLICT`、`CANCELLED`：destructive。
- `RUNNING`：info。
- 普通元数据：secondary。

页面不得各自编写状态颜色和中文标签。

### 5.3 应用尺寸

- 桌面侧边栏约 240px，可折叠为 64px。
- 顶部栏约 56px。
- 主内容最大宽度约 1440px。
- 1024px 以下允许压缩双栏比例。
- 移动端侧边栏使用 Sheet，页面改为单栏且无水平滚动。

## 6. shadcn-vue 接入方案

当前前端是 Vue 3.5 + Vite 7 + TypeScript + npm，尚未使用 Tailwind。接入必须遵循 [shadcn-vue Vite 安装说明](https://www.shadcn-vue.com/docs/installation/vite) 和 [主题说明](https://www.shadcn-vue.com/docs/theming)。

### 6.1 基础设施

- 继续使用 npm 和 `package-lock.json`，禁止生成 pnpm/yarn/bun 锁文件。
- 安装 Tailwind CSS v4、`@tailwindcss/vite` 和必要的 Node 类型。
- 在 TypeScript 和 Vite 中同时配置 `@/* -> ./src/*`。
- 保留当前 Vite proxy。
- 使用 CSS Variables 和 neutral 基色。
- CLI 执行前核验当前稳定版本，并锁定明确版本；不要把 `@latest` 写入长期脚本。
- 推荐选择：Reka、Nova、Neutral、Lucide、Inter、TypeScript、`@/components/ui`。

### 6.2 分批组件

第一批：

```text
button badge card input textarea field select separator
alert skeleton sonner tooltip
```

第二批：

```text
sidebar avatar dropdown-menu breadcrumb scroll-area sheet
```

第三批：

```text
collapsible tabs table dialog alert-dialog checkbox switch
```

首阶段不引入：

- Data Table / TanStack Table。
- Chart。
- Resizable。
- Form + Zod/VeeValidate 全量迁移。
- 用普通 Calendar 代替会议室 30 分钟资源时间轴。

### 6.3 迁移风险

- Tailwind Preflight 会改变基础元素表现。
- 当前 `styles.css` 中全局 `button/input/select/textarea` 规则会覆盖生成组件，必须逐步收口。
- shadcn-vue 是把组件源码复制进项目；升级时必须检查 diff，不允许无审查 overwrite。
- Sheet/Dialog/Dropdown 的 Teleport、焦点捕获、滚动锁定和 z-index 必须在真实页面验证。
- 中文宽度、按钮换行、表格列宽和移动端溢出必须使用中文 fixture 验证。

## 7. 页面规格

### 7.1 登录页

- 中性背景和居中卡片，参考 Cal.com 登录页的克制布局。
- 使用 MeetOps 文字标识，不复制 Cal.com Logo。
- 可以使用极淡的世界时区/网格背景，但不得抢夺注意力。
- 保留现有登录、错误、加载、禁用和重定向行为。
- 删除当前蓝绿色渐变和开发阶段文案。

### 7.2 智能编排

这是首要 Golden Path 和最终演示首页。

桌面结构：左侧约 40% 为会话，右侧约 60% 为结构化结果。

左侧包括：

- 会话标题和“新建会话”。
- 用户消息、Agent 业务回复和澄清问题。
- 自然语言输入、快捷示例、字符限制和流式状态。
- 简洁 `RunStatusBar`，展示状态并提供 Trace 入口。

右侧 Tabs：

1. 需求解析。
2. 候选计划。
3. 资源日历。
4. 政策依据。

需求解析展示：意图、会议类型、时间窗口、必需/可选参与者、设备、硬/软约束和缺失信息。若当前 SSE 未提供完整结构化字段，只展示有真实数据支撑的内容，不从自然语言伪造确定结论。

候选计划使用 `CandidateComparison`：

- 推荐项、选择状态。
- 时间、房间和建筑。
- 总成本和 costBreakdown。
- 对扣分项给出用户可理解的解释。
- 候选最多 3 个，严格使用现有 SSE 数据。

Agent Trace 不永久占据主屏。点击“查看运行过程”打开右侧 Sheet，内部使用 `TraceTimeline` 展示 Step 与 Tool。

`WAITING_CONFIRMATION` 时底部展示固定 `HitlReviewBar`：

- 草案摘要和过期时间。
- 接受并执行。
- 编辑后重新规划。
- 拒绝。

EDIT 使用 Dialog/Sheet；REJECT 使用 AlertDialog。不得改变现有 resume 接口。

移动端用 Tabs 切换“对话”和“编排结果”；固定操作栏不得遮挡正文。

### 7.3 我的会议

- 桌面优先使用紧凑 Table，移动端使用 Card。
- 保留当前状态筛选和 CRUD。
- 允许基于已加载数据提供“列表/日程”视图，不得发明后端查询能力。
- 创建和编辑使用 Dialog 或 Sheet。
- 取消使用 AlertDialog。
- 清楚区分 `MANUAL` 与 `AGENT` 来源。

### 7.4 会议室资源

- 桌面使用紧凑列表或两列卡片。
- 展示名称、编号、楼宇、楼层、容量、设备、热门标记和状态。
- 详情与 ADMIN 编辑使用 Sheet。
- 员工只读与管理员操作必须清晰区分。
- 可用性使用自定义 `AvailabilityGrid` / `ResourceTimeline` 表示固定 30 分钟 `[start,end)` 槽位，不使用普通月历冒充资源时间轴。

### 7.5 Agent Run 详情

顶部展示：状态、intent、duration、model/tool call 数量、runId、traceId 和返回入口。

主体使用垂直时间线：

- Supervisor、Requirement、Policy、Scheduling 和 deterministic 节点。
- Tool Call 使用 Collapsible。
- 显示 riskLevel、duration、status、sanitizedArgs 和 resultSummary。
- 不显示隐藏推理或确认令牌。

### 7.6 待我确认

当前后端没有跨 Run 的待确认列表接口：

- 只能展示当前已知/恢复的 Run 待确认项，或者显示清晰的 Product Preview 空状态。
- 不得在页面中伪造来自后端的任务队列。
- 不得发明 API。

### 7.7 异常重排 Product Preview

静态预览数据放入 `frontend/src/demo/**` 或同等明确目录。

页面展示：

- 房间停用或参与者临时不可用事件。
- 受影响会议与资源。
- 原计划、新计划和 Before/After Diff。
- 约束变化、放宽原因和未受影响项。
- “产品预览”Badge。

执行按钮只能提示“尚未连接后端”，不得伪造保存成功。

### 7.8 会前会后 Product Preview

两个 Tabs：

- 会前准备：人员、房间/设备、议程、材料、政策检查和缺失项。
- 会后行动：决策、行动项、负责人、截止时间、依赖和待确认任务草案。

同样必须明确标记为 Preview，不发送真实写请求。

## 8. 组件与代码组织

至少拆分以下业务组件：

```text
WorkspaceShell
PageHeader
StatusBadge
AgentComposer
RequirementSummary
CandidateComparison
ResourceTimeline
RunStatusBar
HitlReviewBar
TraceDrawer
TraceTimeline
PlanDiff
EmptyState
LoadingState
ErrorState
ProductPreviewBadge
```

- 不再把新逻辑堆入 `ChatView.vue` 或一个巨型 `styles.css`。
- `api/client.ts`、`api/types.ts`、auth、SSE 解析和现有业务逻辑优先保持原状。
- 统一中文文案、标点和状态翻译，修复全部乱码。
- 删除 `Day 6` 等开发阶段标签。

## 9. 分阶段实施

### 阶段 0：行为基线

- 记录登录、聊天 SSE、候选、HITL、恢复、Trace、会议 CRUD、房间查询/管理的截图和命令结果。
- 执行 `npm ci`、`npm run type-check`、`npm run build`。

### 阶段 1：设计系统

- 接入 Tailwind v4、shadcn-vue、路径别名和主题 token。
- 用基础组件建立统一状态与反馈语言。
- 不立即删除旧 CSS。

### 阶段 2：应用框架

- 重建 WorkspaceShell、Sidebar、Topbar、用户菜单和登录页。
- 收口会覆盖 shadcn 的全局旧样式。

### 阶段 3：Golden Path

- 重构智能编排、CandidateComparison、HitlReviewBar、TraceDrawer 和 Agent Run 详情。
- 保留所有 SSE、HITL、恢复和异常处理行为。

### 阶段 4：管理页

- 重构会议和会议室页面。
- 增加桌面/移动布局、Sheet/Dialog 和自定义时间槽视图。

### 阶段 5：产品预览

- 实现异常重排和会前会后预览。
- 所有预览状态、数据和按钮均明确标记。

### 阶段 6：清理和验收

- 删除无引用旧 CSS 和开发阶段文案。
- 修复乱码和响应式问题。
- 更新测试和交接文档。

## 10. 验收标准

### 10.1 自动验证

至少执行：

```powershell
Push-Location frontend
npm ci
npm run type-check
npm run build
Pop-Location

docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml config --quiet
python scripts/smoke-day6.py
```

若变更涉及聊天、HITL 或恢复，再执行：

```powershell
python scripts/smoke-day5.py --restart-agent-service
```

### 10.2 浏览器验收

视口：

- 1440×900。
- 1024×768。
- 390×844。

状态：

- EMPLOYEE / ADMIN。
- 登录成功和失败。
- Loading / Empty / Error / Disabled。
- SSE streaming。
- WAITING_CONFIRMATION。
- ACCEPT / EDIT / REJECT。
- WAITING_BUSINESS_RESULT。
- 页面刷新后的 Run 恢复。
- Trace Sheet、Dialog、Sheet、Dropdown 的 Tab/Esc/焦点行为。
- 无横向溢出。
- 图标按钮均有 `aria-label`。

### 10.3 完成定义

- 真实业务行为没有回归。
- 浏览器仍只访问 Java `/api/v1/**`。
- Product Preview 没有伪造真实后端能力。
- Cal.diy 风格在应用壳、密度、边框、表格、抽屉和状态反馈上保持一致。
- shadcn-vue 组件经过主题化和业务组合，不是默认组件堆砌。
- `docs/HANDOFF.md` 记录版本、文件、验证证据、预览能力和未完成项。

