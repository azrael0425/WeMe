# MeetOps Refero 前端重构执行提示词（历史实施基线）

> 状态：已执行完成。当前实现与验收证据以 `frontend/**` 和 `docs/HANDOFF.md` 为准；后续任务不得把本文件当作“尚未开始”的指令重复执行。

将下面代码块完整复制到新的 Codex 对话。新对话的工作目录必须是 `D:\agent`。

```text
请直接在 D:\agent 仓库完成 MeetOps 前端产品化重构。不要只给建议、设计稿或实施计划；要修改代码、运行测试、启动 Compose、用浏览器验收并更新交接文档，直到形成可运行的成熟 AI 应用界面。

唯一权威前端设计规范：

- D:\agent\docs\09-frontend-product-redesign.md

开始前必须完整阅读并遵守：

- D:\agent\AGENTS.md
- D:\agent\SPEC.md
- D:\agent\docs\HANDOFF.md
- D:\agent\docs\01-functional-spec.md
- D:\agent\docs\04-agent-spec.md
- D:\agent\docs\05-data-and-api-spec.md
- D:\agent\docs\09-frontend-product-redesign.md
- D:\agent\frontend\package.json
- D:\agent\frontend\src 下现有 router、auth、api、views、components、demo 和 styles

先使用 rg、构建文件、git status、测试和运行中的 Compose 核验真实状态。工作树已有大量用户改动，必须保留，不得 reset、checkout 或覆盖无关修改。

一、真实起点

当前前端不是空项目，已经有 Vue 3.5、Vite 7、TypeScript、Tailwind CSS v4、shadcn-vue 2.8.2、Reka UI、Lucide、StatusBadge、WorkspaceShell、CandidateComparison、ResourceTimeline、HITL、Trace、会议 CRUD、房间 ADMIN、Run 恢复和 sessionStorage 会话历史。

不要重复初始化 Tailwind/shadcn，不要重建 package.json，不要生成第二套锁文件。先检查并复用现有组件。当前最大问题是产品布局仍像测试台，尤其是 ChatView 的固定 40/60 双栏、会议室两列大卡片和 Trace 的开发者输出感。

二、视觉参考已经定版

请按照 docs/09 中的映射实现：

1. Meta AI：应用壳、智能编排对话画布、底部悬浮输入框、按需右侧 Sheet。
   https://refero.design/pages/0d258b4a-2867-4746-ab86-03518bc2a36b
2. Mangomint：会议日历与会议室资源时间轴。
   https://refero.design/pages/92ae5496-0aaf-4700-9a9c-fe03f9baf24a
3. TravelPerk：待确认审批卡和筛选层级。
   https://refero.design/pages/9335a133-d103-4685-8152-eb2a0f7e5fc1
4. n8n：Run Activity、Timeline、过滤和节点详情 Sheet。
   https://refero.design/pages/1974172f-cafa-4873-96f9-8c50321e8d72
5. Copy.ai 仅用于智能编排空状态中的少量快捷任务，不做营销页或模板商城。
   https://refero.design/pages/37159acc-20ab-47d2-9ae8-a09ca8a627bd

只借鉴布局、层级和交互，不下载、不复制第三方截图、Logo、品牌素材、字体文件或源代码。

三、允许与禁止范围

允许修改：

- frontend/**
- docs/HANDOFF.md（只追加这次真实完成内容和证据）

禁止修改：

- business-service/**
- agent-service/**
- SPEC 冻结决策
- Java/Python API、SSE 事件、Tool、HITL 和数据契约
- Compose 服务拓扑
- .env 和任何真实密钥

如果发现现有 API 无法支持某个设计，不得发明接口或前端伪数据。应实现诚实的降级/空状态，并在 HANDOFF 记录缺口。

四、必须保护的真实业务行为

- 浏览器只调用 Java `/api/v1/**`，不得直连 Python。
- 登录、鉴权、角色和 redirect。
- POST SSE、`X-Run-Id` 响应头捕获、稳定 client thread。
- 当前 Run URL、sessionStorage 会话历史、per-run context。
- 运行中切换页面再返回、刷新恢复、RUNNING 轮询和 recovery epoch。
- Supervisor/Requirement/Policy/Scheduling 的真实结构化结果。
- Top 3 候选和成本解释。
- CREATE/RESCHEDULE/CANCEL 可辨别 HITL 草案。
- ACCEPT/EDIT/REJECT；EDIT 作废旧 token、重新读取/校验，不得直接写正式会议。
- WAITING_BUSINESS_RESULT、异步结果和冲突恢复。
- Trace、Loop Event、Tool 摘要、模型/Token/耗时指标。
- 会议查询、创建、修改、取消。
- 会议室查询、30 分钟可用性、ADMIN 新建/编辑/启停。
- Asia/Shanghai、ISO 8601 带偏移时间和 `[start,end)`。
- 不显示隐藏推理、完整 Prompt、JWT、Service Token、confirmationToken 或敏感正文。

换 UI 不能成为重写 SSE、API client 或恢复逻辑的理由。优先保持 `src/api/client.ts`、`src/api/types.ts`、`src/api/agent-view.ts` 和 router 的行为。

五、统一视觉 Token

- 页面背景 `#F7F7F5`
- 内容背景 `#FFFFFF`
- 主文字 `#18181B`
- 次级文字 `#71717A`
- 边框 `#E4E4E7`
- 主色 `#4F46E5`，浅背景 `#EEF2FF`
- 成功 `#16A34A`
- 等待 `#D97706`
- 错误 `#DC2626`
- 信息 `#2563EB`
- 卡片圆角 12px，输入框圆角 16px
- 字体 `Inter, PingFang SC, Microsoft YaHei, system-ui, sans-serif`
- 页面标题 22–24px，正文14px，辅助文字12px
- 间距只使用4、8、12、16、24、32

普通内容依靠边框、分隔线和留白；只有悬浮输入框、Sheet、Dialog 使用明显阴影。禁止大渐变、玻璃拟态、霓虹、无意义插画和装饰动画。

把界面中 `✦ ✓ ▦ ⌂ ↻ ◫ ☰ ×` 等文本字符图标替换成现有 `@lucide/vue` 图标。所有 icon-only button 都有 aria-label 和明显 focus ring。

六、分阶段直接实施

阶段0：行为和构建基线

- 运行 npm ci、type-check、build。
- 浏览器记录登录、聊天两轮历史、运行中切页返回、政策问答、候选、HITL、Trace、会议 CRUD、房间查询和 ADMIN 管理。
- 若基线已有失败，先准确记录并判断是否与本任务相关，不能默默忽略。

阶段1：WorkspaceShell 和设计系统

- 更新全局 token、字体、圆角和状态语言。
- 桌面左栏约 220–240px，移动端 Sheet。
- 左栏顶部放“新建编排”和搜索；显示当前标签页真实可恢复的最近 thread/run，不造假。
- 下方保留智能编排、待我确认、我的会议、会议室、具体 Run 入口和低权重 Preview。
- 底部显示真实用户、部门、角色和退出。
- 用 Lucide 替换文本图标。

阶段2：智能编排 Golden Path，最高优先级

- 删除固定 40/60 永久双栏和大边框工作台。
- 中央改为 Meta AI 风格 ConversationCanvas，建议最大宽度 840–920px。
- 消息使用轻量排版；用户消息右侧浅主色，Agent 回答正文式展示。
- AgentComposer 固定/悬浮在底部，自动增长，运行中禁止重复提交。
- 空状态提供4–6条真实可执行快捷任务，不做营销 Hero。
- 需求、候选、政策、执行过程全部移入右侧 OrchestrationSheet，Tabs 为需求/候选/政策/执行。
- 收到候选或 WAITING_CONFIRMATION 可自动开 Sheet 一次；用户关闭后不要反复强制打开。
- HITL 在 Sheet 内展示完整草案，在 composer 上方保留紧凑提醒。
- CREATE 显示草案；RESCHEDULE 显示 Before/After；CANCEL 显示目标会议且没有时间编辑。
- 保留会话历史、Run URL、切页和刷新恢复，做真实浏览器回归。

阶段3：我的会议和会议室

- 我的会议提供日历/列表切换；顶部有今天、日期导航、日/周、状态筛选和创建按钮。
- 只用真实已加载数据画时间网格，不虚构月历或跨范围数据。
- 点击会议块打开详情 Sheet，编辑和取消复用真实 API。
- 会议室默认改为资源时间轴：房间行 × 30分钟时间列；顶部有楼宇、楼层、容量、设备、房型、日期、时间窗口和仅看可用筛选。
- 提供时间轴/房间目录切换；现有大卡片降级为紧凑目录视图。
- 点击房间打开详情 Sheet；点击空闲槽位打开创建会议 Sheet 并预填房间/时间。
- EMPLOYEE 只读与 ADMIN 编辑/启停必须清楚且权限不变。

阶段4：待我确认和 Trace

- 当前没有跨 Run 待确认列表 API。只显示当前标签页真实可恢复的 WAITING_CONFIRMATION Run；否则显示紧凑诚实空状态。
- 不得显示虚假待处理数量、审核人、审批队列，不新增多级审批。
- CREATE/RESCHEDULE/CANCEL 审批卡与智能编排共用 HITL presenter 和 action。
- 普通用户先看六步简化进度；技术详情再看 n8n 风格 Activity Feed。
- Run 页顶部显示真实 status、intent、provider/model、Prompt/Schema 版本、耗时、model/tool 调用、Token、runId、traceId。
- Activity 支持全部/Agent/Tool/Loop/错误过滤；点击节点打开详情 Sheet。
- TraceDrawer 与完整 Run 页复用组件和状态映射。

阶段5：响应式、无障碍和清理

- 真实验收 1440×900、1024×768、390×844。
- 检查 Tab/Shift+Tab/Enter/Esc、焦点返回、Sheet/Dialog 滚动锁定、触屏目标、中文换行、空/错/加载/禁用状态。
- 移除无引用旧 CSS、开发阶段文案、乱码和文本图标。
- 不要把全部新样式继续堆进一个巨型 styles.css；按 design tokens、shell、chat、calendar/resource、trace 拆分，或使用 shadcn primitives 与 Tailwind utility。

七、验收红线

- 智能编排不再是固定双栏测试台。
- 会议室默认视图能回答“某个时间哪些房间可用”。
- 我的会议能快速扫描某天/某周安排。
- 待确认是审批卡或真实空状态，不是说明文档，也不造假。
- Trace 有普通进度和技术详情两层。
- 左栏切页再返回时会话、问题、回答和运行中状态不清空。
- 一个页面只有一个明显主 CTA。
- 1440、1024、390 均无横向滚动、按钮遮挡或无法关闭的 Sheet。
- Product Preview 始终标记，操作不写后端。
- 不复制 Refero/Meta/Mangomint/TravelPerk/n8n/Copy.ai 品牌资产。

八、验证命令

至少执行：

Push-Location D:\agent\frontend
npm ci
npm run type-check
npm run build
Pop-Location

docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml config --quiet

使用 `rg --files scripts` 查找当前真实存在且与聊天/HITL/恢复/会议/房间相关的 smoke，再执行与变更相称的脚本。不要照抄已不存在的旧命令。

重建并启动前端：

docker compose -f compose.yaml -f compose.dev.yaml up -d --build frontend
docker compose ps

通过浏览器实际验收：

- EMPLOYEE 与 ADMIN
- 登录成功与失败
- 连续两轮聊天历史
- 新 Run 在响应头阶段切页，再返回恢复
- 刷新 Run
- 政策引用
- 候选
- CREATE/RESCHEDULE/CANCEL HITL
- ACCEPT/EDIT/REJECT
- WAITING_BUSINESS_RESULT（若可安全复现）
- Trace Drawer 和完整 Run
- 会议 CRUD
- 房间时间轴、可用性和 ADMIN 管理
- 1440×900、1024×768、390×844
- 键盘和焦点

如果某项因数据或环境不能执行，准确记录“未执行、原因、剩余风险”，不得写 PASS。

九、交付

- 更新 D:\agent\docs\HANDOFF.md，记录真实修改文件、页面、依赖、构建、Smoke、浏览器 runId/证据、未完成能力和风险。
- `git diff --check` 检查 frontend 和 HANDOFF。
- 不提交、不推送，除非用户另行要求。
- 不清理或回滚用户已有工作树。
- 最终回答先说明已完成结果，再列验证证据和仍有限制；不要只交付设计建议。

现在开始，持续实现到可运行、可验证的完成状态。
```
