# 前端产品化升级执行提示词

把下面代码块中的内容完整复制到新的 Codex 对话。新对话的工作目录应为 `D:\agent`。

```text
请在 D:\agent 仓库中执行前端产品化升级。不要只给设计建议，要持续实现、验证并交付可运行代码。

权威设计文档：

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
- D:\agent\frontend\src 下的现有 router、auth、api、views、components 和 styles

先使用 rg、构建文件、git status 和实际命令核验仓库状态，不盲信可能过期的交接描述。

一、目标

把当前可用但简陋的 Vue 功能演示升级成 `MeetOps 企业协作编排助手`，视觉和信息架构高度参考：

- https://github.com/calcom/cal.diy
- https://github.com/unovue/shadcn-vue
- https://www.shadcn-vue.com/docs/installation/vite
- https://www.shadcn-vue.com/docs/theming

Cal.diy 用于产品视觉、应用壳、信息密度、列表、日历和详情交互参考；shadcn-vue 用于 Vue 组件实现。不要复制 Cal.com 的 Logo、品牌素材或 React 源码。

二、允许和禁止范围

允许修改：

- frontend/**
- 实现完成后由主 Agent/Coordinator 更新 docs/HANDOFF.md

禁止修改：

- business-service/**
- agent-service/**
- mock-services/**
- Java/Python API、事件语义和跨服务契约
- Compose 服务拓扑
- SPEC 冻结架构边界

如果 AGENTS.md 的职责边界要求 frontend/** 只能由 Frontend/Mock subagent 写入，请按其要求分工；主 Agent 必须亲自检查 diff、运行集成验证并维护 docs/HANDOFF.md。不得让多个 subagent 同时修改相同前端文件。

三、必须保持的真实功能

- 登录、鉴权和重定向。
- 浏览器只访问 `/api/v1/**`，前端不直连 Python。
- POST SSE、Run URL、刷新恢复。
- 候选方案。
- ACCEPT / EDIT / REJECT HITL。
- WAITING_BUSINESS_RESULT 和热门预约轮询/恢复。
- Agent Trace。
- 会议查询、创建、修改、取消。
- 会议室查询、30 分钟可用性、ADMIN 管理。
- Asia/Shanghai、ISO 8601 带偏移时间、固定 30 分钟槽位。
- 现有 API 类型、错误处理、权限和安全边界。
- 不展示隐藏推理，只展示结构化摘要、工具摘要、引用和业务结果。

视觉重构不得重写或弱化这些行为。优先保留 `src/api/client.ts`、`src/api/types.ts`、auth 和 SSE 解析逻辑。

四、技术方案

当前项目为 Vue 3.5 + Vite 7 + TypeScript + npm，尚未安装 Tailwind。

1. 继续使用 npm 和 package-lock.json，禁止生成 pnpm-lock.yaml、yarn.lock 或 bun.lock。
2. 核验当前 shadcn-vue 稳定版本，并使用明确版本初始化；不要把 @latest 写入长期脚本。
3. 接入 Tailwind CSS v4 和 @tailwindcss/vite。
4. 在 TypeScript 和 Vite 中同时配置 `@/* -> ./src/*`，保留现有 proxy。
5. shadcn-vue 推荐：Reka + Nova + Neutral + Lucide + Inter + CSS Variables + TypeScript。
6. 分批添加：
   - button badge card input textarea field select separator alert skeleton sonner tooltip
   - sidebar avatar dropdown-menu breadcrumb scroll-area sheet
   - collapsible tabs table dialog alert-dialog checkbox switch
7. 暂时不要引入 Data Table、Chart、Resizable、Form/Zod 全量迁移；不要用通用 Calendar 冒充会议室资源时间轴。
8. Tailwind Preflight 和旧 styles.css 分阶段共存；收口全局 button/input/select/textarea 样式，最后删除无引用规则。

五、视觉规范

- 页面背景 #f8f9fa。
- 表面 #ffffff。
- 主文字 #111827。
- 次级文字 #6b7280。
- 边框 #e5e7eb。
- 主按钮 #292929。
- Hover/Accent #f3f4f6。
- 基础圆角 0.5rem。
- 字体 Inter、Microsoft YaHei、system-ui。
- 页面标题20–24px，正文14px，辅助文字12px。
- 间距使用4、8、12、16、24、32。
- 普通卡片以边框为主，浮层才使用明显阴影。
- 禁止大面积渐变、玻璃拟态、霓虹发光、无业务意义 KPI 和复杂动画。
- 第一阶段完整做好亮色主题；暗色不是首要交付。
- 状态统一通过 StatusBadge：成功绿、等待黄、冲突/失败红、运行蓝、元数据灰。

六、信息架构

重建 WorkspaceShell：

工作台
- 智能编排
- 待我确认

协作
- 我的会议
- 会议室资源

系统
- Agent Run 详情从具体 Run 进入

产品预览
- 异常重排
- 会前会后

底部是用户头像、姓名、部门、角色和退出。

桌面侧栏约240px，可折叠为64px；移动端使用 Sheet。保留原有路由兼容性。

七、核心页面

1. 登录页

- 中性、居中、克制的 Cal.com 风格。
- 使用 MeetOps 文字标识，不复制 Cal.com Logo。
- 可以加入极淡的时区网格背景。
- 保持现有登录行为，删除渐变和开发文案。

2. 智能编排

- 桌面左40%会话，右60%结构化结果。
- 左侧：会话、Agent 业务回复、澄清问题、输入框、快捷示例、流式状态、RunStatusBar。
- 右侧 Tabs：需求解析、候选计划、资源日历、政策依据。
- 需求解析只展示真实数据支撑的字段，不能从文本伪造后端未返回的结构化结论。
- 将三个普通候选卡升级成 CandidateComparison，展示推荐项、时间、房间、建筑、totalCost、costBreakdown、扣分解释和选择状态。
- Agent Trace 改为点击后打开右侧 TraceDrawer/Sheet，不永久占据主屏。
- WAITING_CONFIRMATION 时显示底部 HitlReviewBar：草案、过期时间、接受、编辑、拒绝。
- EDIT 使用 Dialog/Sheet；REJECT 使用 AlertDialog。
- 移动端切换“对话/编排结果”，无横向溢出，固定栏不遮挡正文。

3. 我的会议

- 桌面紧凑 Table，移动端 Card。
- 保留筛选和真实 CRUD。
- 可基于已加载数据提供列表/日程切换，不发明 API。
- 创建/编辑使用 Dialog 或 Sheet，取消使用 AlertDialog。
- 清楚显示状态和 MANUAL/AGENT 来源。

4. 会议室资源

- 紧凑列表或两列卡片。
- 展示编号、名称、楼宇、楼层、容量、设备、状态、热门标记。
- 详情和 ADMIN 编辑使用 Sheet。
- EMPLOYEE 只读和 ADMIN 操作清晰区分。
- 用自定义 AvailabilityGrid/ResourceTimeline 表示30分钟 `[start,end)` 槽位，不用普通月历。

5. Agent Run 详情

- 顶部展示状态、intent、duration、model/tool call 数量、runId、traceId 和返回入口。
- 使用垂直 TraceTimeline。
- Tool Call 使用 Collapsible，显示 riskLevel、duration、status、sanitizedArgs、resultSummary。
- 不显示隐藏推理、密钥或 confirmationToken。

6. 待我确认

- 后端没有跨 Run 列表接口时，只展示当前可恢复项或清晰 Preview/Empty 状态。
- 不得伪造任务列表，不得发明 API。

7. 异常重排 Product Preview

- 静态数据必须放在 frontend/src/demo/** 或同等明确目录。
- 展示故障事件、受影响会议、原计划、新计划、Before/After Diff、约束变化和放宽原因。
- 显示 Product Preview Badge。
- 操作按钮只提示“尚未连接后端”，不得伪造保存成功。

8. 会前会后 Product Preview

- Tabs：会前准备、会后行动。
- 会前：人员、资源、设备、议程、材料、政策、缺失项。
- 会后：决策、行动项、负责人、截止时间、依赖、任务草案。
- 同样明确标注 Preview。

八、业务组件

至少拆分：

- WorkspaceShell
- PageHeader
- StatusBadge
- AgentComposer
- RequirementSummary
- CandidateComparison
- ResourceTimeline
- RunStatusBar
- HitlReviewBar
- TraceDrawer
- TraceTimeline
- PlanDiff
- EmptyState
- LoadingState
- ErrorState
- ProductPreviewBadge

不要继续把全部内容堆在 ChatView.vue 和一个巨型 styles.css 中。

九、实施阶段

按可验收切片持续执行：

阶段0：记录行为基线，运行 npm ci/type-check/build。
阶段1：Tailwind、shadcn-vue、路径别名、主题 Token。
阶段2：WorkspaceShell、Sidebar、Topbar、登录页。
阶段3：智能编排、候选、HITL、Trace，这是首要 Golden Path。
阶段4：会议和会议室。
阶段5：两条明确标记的 Product Preview。
阶段6：响应式、无障碍、乱码/开发文案和旧 CSS 清理。

每个阶段完成后运行受影响的 type-check/build，不要等全部完成才发现问题。

十、测试和验收

至少执行：

Push-Location D:\agent\frontend
npm ci
npm run type-check
npm run build
Pop-Location

docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml config --quiet
python scripts/smoke-day6.py

聊天/HITL/恢复变更后还要执行：

python scripts/smoke-day5.py --restart-agent-service

使用浏览器实际验收：

- 1440×900
- 1024×768
- 390×844
- EMPLOYEE 和 ADMIN
- 登录成功/失败
- loading/empty/error/disabled
- SSE streaming
- WAITING_CONFIRMATION
- ACCEPT/EDIT/REJECT
- WAITING_BUSINESS_RESULT
- 刷新后的 Run 恢复
- Trace Sheet
- Dialog/Sheet/Dropdown 的 Tab、Esc、焦点和滚动锁定
- 无横向溢出
- 图标按钮均有 aria-label

如果环境原因无法执行，准确记录命令、原因和风险，不得声称通过。

十一、交付要求

- 保留用户已有改动，不覆盖无关文件。
- 不修改后端和 Agent。
- 不改变 API 契约。
- 不伪造后端能力。
- 不引入真实密钥。
- 不复制 Cal.com 品牌资产。
- 新增依赖必须锁进 package-lock.json。
- 修复全部中文乱码并删除 Day 6 等开发阶段文案。
- 完成后更新 docs/HANDOFF.md：修改文件、真实页面、Preview 页面、shadcn-vue 版本、命令、测试结果、未连接能力、剩余风险和下一步。
- 主 Agent 必须亲自检查 git diff、git status、构建和跨页面验收，不能直接采信 subagent 的完成声明。

现在开始执行。持续实现到可运行、可验证的完成状态，不要只输出计划。
```

