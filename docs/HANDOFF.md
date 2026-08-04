# 项目开发交接

## 2026-08-14 删除视频会议链接 Mock

- Spec 升级为 1.2：删除视频会议 Provider Mock、链接创建字段、Java/Python/前端契约、评测维度、部署环境变量、Compose 服务及相关 Smoke 输入；`mock-services/video-provider/**` 已删除。
- 保留 `VIDEO_CONFERENCE` 会议室设备特征。用户提出“要视频会议设备”仍作为确定性房间硬约束处理，但系统不创建真实或 Mock 视频会议链接。
- 验证：固定 Java 21 Maven 容器离线 `verify` PASS（61 tests，0 failure/error/skip，Spotless PASS）；Python Ruff PASS、Mypy 41 source files PASS、Pytest 90 passed（1 条上游 pending-deprecation warning）；前端 type-check 与 production build PASS（1888 modules）；基础及开发覆盖 Compose `config --quiet` 均 PASS；fixture 40 条评测报告已重新生成。
- 未启动、重建或删除现有项目容器与命名卷。宿主机 Wrapper 使用 JDK 17 被 Enforcer 正常拒绝，固定 JDK 21 复验结果作为本次 Java 验收依据。

## 2026-08-14 MeetOps 前端补充闭环验收

- 浏览器 RESCHEDULE ACCEPT 已补齐：先通过“我的会议”创建隔离会议 110，再从智能编排提交真实改期请求 `run_55b08a5cc6d84f1f94c2c68e9db67632`；确认 Sheet 展示 08/25 15:00—16:00 到 16:00—17:00 的 Before/After，点击“接受并改期”后 Run 为 SUCCEEDED。公共会议 API 复核 roomId 117→116、version 0→1，随后将该隔离会议取消清理为 CANCELLED/version 2。
- 浏览器 CANCEL ACCEPT 已补齐：创建隔离会议 111 后提交 `run_0403de093b0549e7b4660a63d1369201`，取消草案明确展示会议 ID、会议号、房间和时间；点击“确认取消会议”后 Run 为 SUCCEEDED，公共会议 API 复核会议为 CANCELLED/version 1。该流程本身即完成清理，没有遗留有效会议。
- 浏览器 EDIT 操作已真实触发：`run_4e75590873924b3ebf545ab5bf991ecf` 首轮返回 Top 3，点击候选的“选择并重新校验”后，Trace 记录 `resume_dispatch` 已接收编辑请求并重新提取需求；第二轮因真实模型调用预算耗尽终止为 `BUDGET_EXHAUSTED`（10 次模型调用、3 次 Tool），没有会议写入。因此本轮只证明前端 EDIT 请求、等待态和失败呈现真实可用，不把 EDIT 后再次等待确认记为 PASS。
- HOT/`WAITING_BUSINESS_RESULT` 已安全复现：`run_76e75f3b9e68450289fe7d878a61e232` 首轮返回 Top 3，浏览器将草案切换并重新校验为 `isHot=true` 的总部楼 VIP 501（room 103），点击“接受并创建”后页面先显示“业务处理中”和请求号 `BR20260813183825F1AA3BCFEC`，随后刷新为“成功/热门预约已确认并写入会议列表”。公共会议 API 复核 Agent 会议 112 为 CONFIRMED，之后已取消清理为 CANCELLED/version 1。
- 真实模型轨迹补测 `python scripts/live-model-trajectory.py --output artifacts/live-eval/frontend-completion-trajectory.json` 得到 6/8 PASS：CREATE 容量无解 REJECT、CREATE 指名参会者 ACCEPT、政策引用、RESCHEDULE REJECT、隔离 CANCEL REJECT/ACCEPT 通过；重复 RESCHEDULE ACCEPT 因第二次模型轨迹未产生 `hitl.required` 失败，固定会议 ID 9001 因当前数据不存在返回 `MEETING_NOT_FOUND`。报告内容已据实记录后删除本轮临时 artifact，避免越出本次 `frontend/**` 与本文件的写入范围。
- 本轮所有隔离会议 110、111、112 最终均为 CANCELLED；没有清卷、没有修改 Compose 拓扑或后端/Agent 源码，也没有推送或创建 PR。

## 2026-08-14 MeetOps 前端产品化重构落地与验收

- 已按 `docs/09-frontend-product-redesign.md` 完成可运行实现，不再是固定 40/60 测试台：`WorkspaceShell` 改为 232px 桌面导航和 `<=1024px` 移动 Sheet；智能编排改为最大 920px 的 `ConversationCanvas`、底部 `AgentComposer` 与按需 `OrchestrationSheet`，需求/候选/政策/执行四类结构化结果不再永久占用主画布。左栏最近任务只读取当前标签页真实 `sessionStorage` run/thread，上线登出时清除 `meetops.chat-*` 上下文，避免不同账号继承前一账号会话。
- Golden Path 保留原 Java 公共 API、POST SSE、`X-Run-Id`、稳定 thread、recovery epoch、轮询和 per-run history；候选或 `WAITING_CONFIRMATION` 只自动打开一次 Sheet，用户关闭后不强开。HITL 继续共用真实 CREATE/RESCHEDULE/CANCEL presenter，EDIT 仍走 `/resume` 并等待新 token；Approvals 只恢复当前标签页真实 `WAITING_CONFIRMATION` Run，不伪造跨 Run 队列。
- 我的会议新增真实日/周窗口、日历/列表、状态筛选、会议详情和原 CRUD Sheet；390px 默认切为单日列表。会议室默认改为房间行 × 30 分钟列的 availability 资源轴，保留紧凑目录、楼栋/楼层/容量/设备/房型/日期/时间筛选；点击可用格会预填房间及 `[start,end)` 30 分钟时段，EMPLOYEE 只读，ADMIN 显示新增/编辑/启停入口。
- Trace 分为六步业务进度和技术 Activity 两层；Drawer 与完整 Run 页复用 Agent/Tool/Loop/错误筛选及安全详情 Sheet。Run Overview 只展示后端真实 status、intent、provider/model、Prompt/Schema、耗时、模型/Tool 次数、Token、runId/traceId；详情明确不展示隐藏推理、完整 Prompt、确认令牌或凭据。策略结果没有 citation 时显示“未找到可验证证据”，不会根据回答文本伪造出处。
- 新增/拆分组件：`ConversationCanvas.vue`、`OrchestrationSheet.vue`、`PolicyCitations.vue`、`ApprovalCard.vue`、`MeetingCalendar.vue`、`RoomDirectory.vue`、`ActivityTimeline.vue`、`TraceDetailSheet.vue`、`RunOverview.vue` 等；样式拆到 `styles/tokens.css`、`base.css`、`shell.css`、`chat.css`、`calendar-resource.css`、`trace.css`、`preview.css`。没有新增 npm 依赖或锁文件；界面文本字符图标已替换为现有 `@lucide/vue`，禁用字符扫描无命中。
- 构建/部署证据：任务起点执行 `npm ci` PASS（仅本机 Node 24.14.0 与间接包期望 24.15+ 的非阻塞 engine warning）；最终 `npm run type-check` PASS、`npm run build` PASS（Vite 7.3.6，1888 modules）；基础与 dev-overlay 两条 `docker compose ... config --quiet` PASS；使用 `docker compose -f compose.yaml -f compose.dev.yaml up -d --build frontend` 重建最新前端（Compose 因依赖关系也重建 business-service，但未清卷或改拓扑），当前 Compose 定义内全栈服务 healthy。
- Smoke：提交前再次执行 `python scripts/smoke-day6.py --public-base http://localhost` PASS，输出 `day6PublicSurface=PASS`、12 个 active room、手动会议 created-updated-cancelled、Agent candidates-hitl-reject-trace、room ADMIN RBAC PASS。`python scripts/smoke-day5.py` 首次普通 CREATE 路径实际生成并完成 `run_54c44a57115d45a9be11d3a03f879fc8`：Trace 为 SUCCEEDED、9 次模型调用/9 次 Tool，包含 EDIT 后重新创建草案和 ACCEPT 的 `confirm_booking` WRITE；脚本随后因默认内部 `localhost:8000` 未映射失败，产生的 meeting 105 已通过公共 DELETE 取消。改用 `--public-trace` 重跑通过普通路径后，在 HOT 额外路径因真实模型未选择脚本固定期待的 room 103 而失败，因此本轮不将该脚本的 HOT 固定房间断言记为 PASS。
- 浏览器真实验收（Compose 页面）：EMPLOYEE `zhangsan` 和 ADMIN `admin` 登录成功，错误密码显示“用户名或密码错误”；连续两轮会话 `run_ee2b9e61e8f345b296f7a0a60acf7da2`、`run_1c9f9a42a12849f993de3f7d461ff9f5` 在刷新后同时保留。`run_3692fbade5614ebc81f3de76a66ed352` 返回真实 Top 3 与 CREATE 草案，切到“我的会议”再返回后 URL、问题、runId 和待确认状态仍在，浏览器随后完成双确认 REJECT，终态 CANCELLED 且无正式写入。Trace Drawer/完整 Run、政策诚实空引用、会议日历/创建表单、房间资源轴/槽位预填、ADMIN 新增房间表单均已打开核验。
- 响应式/可访问性：实际覆盖 1440×900、1024×768、390×844，三档 `documentElement.scrollWidth == clientWidth`；390px 会议页为单日列表，移动导航 Esc 关闭后焦点返回“打开导航”按钮，Sheet/Dialog 有滚动锁与关闭入口。浏览器验收中发现并修复了移动侧栏焦点在解除 `inert` 前恢复，以及登出后聊天上下文跨账号残留两个问题；提交前复验登录、智能编排、我的会议、待我确认和移动导航，控制台 error 日志为空。
- 当前限制：待确认页只展示当前标签页 `sessionStorage` 中可恢复的 Run，项目没有跨浏览器的全局 Run 列表 API；Product Preview 不触发后端写入。政策 Run 未返回 citation 时继续展示诚实无证据态。RESCHEDULE/CANCEL ACCEPT 与 HOT/`WAITING_BUSINESS_RESULT` 已在上方隔离数据闭环中验收，不再列为未执行项。

## 2026-08-13 Refero 前端产品化设计定版与执行交接

- 已将 `docs/09-frontend-product-redesign.md` 更新为唯一权威的 Refero 前端重构规范，替代此前以 Cal.diy 和固定 40/60 双栏为主的旧方案。设计组合已经冻结：Meta AI 用于应用壳与智能编排，Mangomint 用于会议/资源时间轴，TravelPerk 用于 HITL 待确认，n8n 用于 Run Activity/Trace，Copy.ai 仅用于空状态快捷任务。
- 已明确当前真实起点：前端已经安装 Tailwind CSS v4、shadcn-vue 2.8.2、Reka UI、Lucide，并已有 WorkspaceShell、候选、HITL、Trace、会议/房间页面与恢复逻辑；后续实现不得重复初始化依赖或把现有项目当脚手架重建。
- 核心产品决策：智能编排移除固定 40/60 测试台双栏，改为宽对话画布 + 底部悬浮输入框 + 按需 Orchestration Sheet；会议室默认改为房间行 × 30 分钟时间列的资源时间轴；待确认只展示当前标签页真实可恢复 Run，不伪造跨 Run 队列；Trace 分普通进度和技术 Activity 两层。
- 视觉 Token 冻结为 `#F7F7F5` 页面背景、`#FFFFFF` 表面、`#18181B` 主文字、`#71717A` 次文字、`#E4E4E7` 边框和 `#4F46E5` 主色；图标统一使用现有 `@lucide/vue`，不继续使用文本符号图标。
- `docs/10-frontend-redesign-execution-prompt.md` 已更新为可以完整复制到另一对话的实施提示词，包含目录边界、现有真实能力、分阶段修改、功能红线、构建/Compose/浏览器验收和 HANDOFF 要求。
- 本次仅更新设计与执行文档，没有修改前端运行代码，因此没有把构建结果冒充为本次实施验收；下一任务是按 `docs/10` 在 `frontend/**` 实施并验证。

## 2026-08-13 左侧导航会话恢复与演示数据扩容

- 根因与修复：Vue 路由切换会卸载 `ChatView`，原实现只在组件内存中保留当前 `runId/threadId`；左侧导航回到不带查询参数的 `/chat` 后无法知道要恢复哪个 Run。现在 `frontend/src/router/index.ts` 会在离开聊天页前保存安全格式的当前 Run，并在返回 `/chat` 时先补回 `?runId=...`；`ChatView.vue` 同步保存当前 Run 与按 thread 分组的问答历史。“新建会话”仍会显式清空该指针。
- 浏览器实测：从 `run_476dc3b6e2494ddb863e69e0e792abc2` 聊天页切到“待我确认”，再点击“智能编排”，地址恢复为原 `?runId=...`，原问题与“必需参会者在请求窗口内没有共同空闲时间。”回答均仍可见。
- 新增 Flyway `V6__expand_demo_people_and_rooms.sql`，受 `demo-data-enabled` 控制且不清理已有卷：运行库现有 17 名人员、8 个部门、2 名 ADMIN、15 名 EMPLOYEE，其中 1 名停用员工；会议室共 13 间，覆盖 8 种类型、4 栋楼、容量 1–80 人、白板/大屏/视频/投影组合、5 间热门房与 1 间维护停用房。员工页面只返回 12 间 ACTIVE 房，管理员可查看停用房。
- 人员身份边界：冻结权限契约仍只有 `EMPLOYEE/ADMIN`；产品、销售、财务、人力、客户成功、法务、研发、运维等差异通过部门与展示身份表达，没有静默扩展 RBAC 角色。
- 验证：Java 21 容器 `./mvnw -B -ntp verify` PASS（61 tests，0 failure/error/skip，Spotless PASS）；`npm run type-check` PASS；`npm run build` PASS（88 modules）；business/frontend Compose 镜像重建并健康；真实 MySQL 从 V5 增量升级到 V6，统计与上述数量一致；会议室页面实显 12 间可用房。

## 2026-08-13 运行中切页恢复竞态修复

- 根因：新 Run 的 `runId` 由 Java 创建并已在 SSE 响应头 `X-Run-Id` 提供，但前端此前只等待第一条 `run.started` 帧；首次 thread 也由服务端稍后创建。用户在两者落盘前切到“我的会议”会卸载 ChatView，URL 和会话存储均没有可恢复标识。
- 修复：`apiSseRequest` 在验证 SSE 响应后立即上报响应头 Run；ChatView 在发起请求前生成稳定 thread，并同步保存 Run/thread/问题；返回运行中的 Run 后轮询 recovery/trace，短暂 404/503 在元数据可见前有限重试；恢复请求增加 epoch 隔离，旧请求不得覆盖新会话或新 Run；“新建会话”显式移除旧 runId 查询参数。
- 既有任务恢复：用户报告时最近的 `run_b6a8fde55c8c4857b529d6907d00a1c6` 已在后端确认 `SUCCEEDED`，Run/Trace 未丢失，并已导航恢复。旧版只持久化问题摘要，因此该旧 Run 的完整问题正文不能从服务端反向还原。
- 验证：`npm run type-check` PASS；`npm run build` PASS（88 modules）；frontend Compose 镜像重建并 healthy；浏览器确认响应头阶段即出现新 `?runId=`，快速离开后后端 Run 仍执行到 `SUCCEEDED`。

## 2026-08-13 会话历史展示修复

- 修复 `frontend/src/views/ChatView.vue` 只渲染当前 `submittedMessage/answerSummary`、新 Run 清空上一轮问答的问题；同一 `threadId` 的问答现在按轮次追加展示，每轮保留对应 Run 详情入口。
- 会话记录按 `threadId` 保存到当前浏览器标签页的 `sessionStorage`，刷新当前 Run 后仍可恢复；点击“新建会话”会清空当前展示并生成新的 thread，不跨浏览器或跨设备同步。
- 验证：`npm run type-check` PASS；`npm run build` PASS（88 modules）；frontend Compose 镜像重建并达到 healthy；真实浏览器连续提交两条只读政策问题后旧问题、旧答案和新问题同时可见，刷新后仍同时可见。
- 与会话展示修复不同，后续“改期/取消”仍必须能解析目标会议。`TARGET_REFERENCE_MISSING` 表示请求没有会议 ID，也没有“刚才创建的会议/最近的会议”等可解析指代；该终态不会修改会议或发送通知。

## 2026-08-13 Requirement 来源忠实度误判修复

- 修复显式中文会议类型（例如“架构评审”）被模型规范化为 `ARCHITECTURE_REVIEW` 后，`SourceFidelityEvaluator` 因规范值不是原文连续子串而错误返回 `EVIDENCE_NOT_IN_SOURCE` 的问题。
- `meetingType` 现在只接受受控枚举到中文原文锚点的确定性映射；未知会议类型仍会拒绝，未放宽姓名、时间、设备或人数的来源忠实度边界。
- 新增与浏览器原始请求同形的回归测试；`tests/test_provider_and_tools.py` 34 passed，ruff PASS，mypy 39 source files PASS。
- Agent Compose 镜像重建并 healthy；真实 `deepseek-v4-flash` 浏览器复测不再出现 `EVIDENCE_NOT_IN_SOURCE`，请求安全结束为 `NO_SOLUTION`（张三和李四在该固定窗口没有共同空闲时间），没有创建草案或会议。

## 1. 交接元信息

- 最后更新时间：2026-08-14（Asia/Shanghai）。
- 当前里程碑：Spec 1.2——在保留会议室视频设备特征的前提下删除视频会议链接 Mock；受控 Agent Loop、原生 DeepSeek Tool Calling、冲突修复和 Evaluator–Optimizer 能力不变。
- Day 1 至 Day 7 已验收回归：**PASS**。
- Spec 基线：1.2；保留四个运行时 Agent 和既有服务边界，不引入 DeepAgents 或 Critic Agent。
- Git 状态：`main` 包含 Day 4 基线提交 `31773e2 feat: complete day 4 agent foundation`；Day 5、Day 6 与 Day 7 的已验收改动由本次完成提交记录，未作重置或清理。
- 运行状态：business-service 与 agent-service 已使用 Spec 1.1 新镜像重建；当前本机 Agent 已切换为 `AGENT_MODEL_PROVIDER=deepseek`、`DEEPSEEK_MODEL=deepseek-v4-flash`，完整基础 Compose 常驻服务 healthy。真实模型业务验收结论见第 20 节。
- 维护责任：本文件只由主 Agent / Coordinator 更新。

本文件记录真实可复现状态，不替代 `SPEC.md` 和专项规范。

## 2. 当前状态总览

| 工作流 | 状态 | 证据/说明 |
|---|---|---|
| Day 1 骨架、双库和登录/会议室 | DONE | Java/Python/Frontend、MySQL 双库、Redis、RocketMQ、Qdrant、Nginx 和公共登录链路回归通过 |
| Day 2 手动预约与并发正确性 | DONE | 30 分钟槽位、MySQL 最终唯一约束、Redis Lua、幂等、CRUD、修改回滚及两组真实 100 并发回归通过 |
| Day 3 Flyway 与草案 | DONE | V4 创建六张 Day 3 表；CREATE/RESCHEDULE/CANCEL 草案确认前无会议或槽位副作用 |
| Day 3 HOT 受理与 Outbox | DONE | HOT 确认原子写 `booking_request(PENDING)`、`BOOKING_COMMAND` Outbox、幂等结果并消费确认令牌；HTTP 202 返回 PENDING |
| Day 3 RocketMQ 最终处理 | DONE | 真实 Broker 上 SUCCESS/CONFLICT、BOOKING_RESULT、通知、消费幂等和重复消息不重复创建均通过 |
| Day 3 Tool Gateway | DONE | Service Token、AgentContextToken、参数上限、员工/忙闲/房间/最近会议、草案、审计和重放均通过 |
| Day 3 SSE/回调边界 | DONE | Java SSE 代理不伪造输出；Python 端点不存在时完整返回 503 `AGENT_UNAVAILABLE`；BOOKING_RESULT 回调骨架默认关闭 |
| Day 4 Multi-Agent Golden Path 与增强切片 | DONE | 固定为 Supervisor + Requirement/Policy/Scheduling 四个 Agent；确定性 fixture 经 Java SSE 代理完成普通调度与政策引用 Smoke，Run/Step/Tool Call 已持久化 |
| Day 5 调度、HITL、恢复与 HOT 闭环 | DONE | OR-Tools Top 3、独立验证器、DRAFT/HITL、Redis checkpoint、业务回调与 HOT CONFLICT recovery 均已通过真实 Smoke |
| Day 6 浏览器业务闭环与会议室管理 | DONE | 聊天 SSE/HITL/安全 Trace、我的会议手动管理、会议室可用性与 ADMIN 管理均已通过公共接口 Smoke 和浏览器验收 |

## 3. Day 3 完成内容与责任分工

### 3.1 主 Agent / Coordinator

- 完整重读并核验 `AGENTS.md`、`SPEC.md`、`README.md`、`docs/HANDOFF.md` 和 `docs/01` 至 `docs/08`；以文件、测试、数据库、容器和 `git status` 为准核对状态。
- 冻结 Day 3 内部安全、Tool、草案、HOT、Outbox、事件和 SSE 最小契约，并同步 `docs/03-java-backend-spec.md`、`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`。
- 更新 `.env.example`、`compose.yaml`、`compose.dev.yaml` 和 `README.md`，加入 Day 3 配置、固定 Topic/Consumer Group、一次性 `rocketmq-topic-init` 和 Day 3 镜像标签。
- 新增 `scripts/smoke-day3.py`，覆盖内部鉴权、查询 Tool、草案、HOT PENDING、MQ SUCCESS/CONFLICT、Tool 重放、改期/取消确认和 SSE 不可用边界。
- 新增并修正 `scripts/replay-day3-booking-command.ps1`，发送语义完全相同且 JSON 完整的重复 `BOOKING_COMMAND`，同时核对数据库终态和 MQ 总积压。
- 亲自审查 Java 事务、Outbox 租约、失败审计、通知接收人、MQ 终态幂等和 SSE 实现；问题优先交回原 Java subagent 修复。
- 亲自执行固定 Java 21 verify、应用镜像构建、真实 MySQL V4、完整 Compose、Day 1/2 回归、两组 100 并发、Day 3 Smoke、MQ 重放和静态安全/范围扫描。
- 真实联调发现 RocketMQ 4.9.7 镜像内 Java 8 在 Docker Desktop cgroup v2 上读取消息时初始化 `StoreUtil` 失败；在固定堆参数上加入 `-XX:-UseContainerSupport` 后，原 PENDING 消息无需重发即恢复为 SUCCESS。

### 3.2 Java subagent（仅 `business-service/**`）

- Flyway V4 新增 `booking_draft`、`booking_request`、`message_outbox`、`event_consume_record`、`notification`、`agent_tool_audit`，字段和唯一约束与 `docs/05` 一致。
- 实现内部 Tool API：员工解析、员工忙闲、可用会议室、最近会议、创建草案、改期草案、取消预览和三类确认。
- 实现 Service Token 恒定时间比较、HS256 AgentContextToken audience/claims/过期校验、上下文头一致性、真实用户角色核对、参数上限、Tool 审计和调用重放。
- Tool 失败审计使用独立 `REQUIRES_NEW` 事务；相同 `(runId,toolCallId,toolName)` 同摘要返回原结果，不同摘要拒绝。
- 实现 HOT 受理事务、Outbox 发布器的抢占/有限重试/DEAD/30 秒 SENDING 租约恢复，以及 RocketMQ Producer、BOOKING_COMMAND Consumer 和可禁用的 BOOKING_RESULT 回调 Consumer。
- MQ 消费成功时在同一事务写会议、槽位、参与者、请求终态、消费记录、通知、领域事件和 BOOKING_RESULT；冲突时回滚预约写入并原子落 CONFLICT 终态。
- 手动 create/update/cancel 同样在业务事务内写领域 Outbox；通知按参会者去重，覆盖组织者和所有参与者。
- 实现本人/ADMIN 的 booking request 查询和 Java→Python SSE 代理边界。
- 真实 Tomcat 发现 503 错误曾错误走异步 `StreamingResponseBody`；修为同步普通 JSON，仅成功 SSE 使用异步流，并加入 `asyncNotStarted()` 回归测试。
- 最终固定 JDK 21 `mvn verify`：33 tests，0 failure/error/skip；Spotless 142 个 Java 文件通过。

### 3.3 Python 与 Frontend

- Day 3 没有修改 `agent-service/**` 或 `frontend/**`，没有启动对应开发 subagent；这些模块保持此前已验收能力。
- 全栈统一构建为 `:day3` 应用镜像并通过健康检查，但这不代表已实现 Multi-Agent、模型调用、聊天、Trace 或视频会议业务。
- 前端仍只通过 Nginx 访问 Java；本轮公共 HTTP 登录与 API 加载回归通过。Day 3 没有 UI 变更，因此未新增浏览器视觉验收。

## 4. Day 3 关键文件

### 公共、Compose、契约和验证

- `.env.example`
- `compose.yaml`
- `compose.dev.yaml`
- `README.md`
- `docs/03-java-backend-spec.md`
- `docs/05-data-and-api-spec.md`
- `docs/06-docker-deployment.md`
- `scripts/smoke-day3.py`
- `scripts/replay-day3-booking-command.ps1`

### Java

- `business-service/src/main/resources/db/migration/V4__create_day3_async_and_tool_tables.sql`
- `business-service/src/main/java/com/example/meeting/common/security/AgentToolSecurityFilter.java`
- `business-service/src/main/java/com/example/meeting/agentgateway/internal/AgentToolController.java`
- `business-service/src/main/java/com/example/meeting/agentgateway/audit/AgentToolAuditService.java`
- `business-service/src/main/java/com/example/meeting/booking/application/BookingDraftService.java`
- `business-service/src/main/java/com/example/meeting/booking/application/BookingConfirmationService.java`
- `business-service/src/main/java/com/example/meeting/booking/application/HotBookingAcceptanceService.java`
- `business-service/src/main/java/com/example/meeting/booking/application/BookingCompletionWriter.java`
- `business-service/src/main/java/com/example/meeting/outbox/OutboxPublisher.java`
- `business-service/src/main/java/com/example/meeting/mq/RocketMqClientManager.java`
- `business-service/src/main/java/com/example/meeting/mq/BookingCommandProcessor.java`
- `business-service/src/main/java/com/example/meeting/mq/BookingCommandFinalizationService.java`
- `business-service/src/main/java/com/example/meeting/agentgateway/api/AgentGatewayController.java`
- `business-service/src/test/java/com/example/meeting/agentgateway/AgentToolGatewayIntegrationTest.java`
- `business-service/src/test/java/com/example/meeting/outbox/OutboxPublisherIntegrationTest.java`

## 5. 契约、事务与范围核对

- 公共成功/错误信封继续为 `{data,traceId,timestamp}` 与 `{code,message,details,traceId}`；SSE 成功流除外。
- 浏览器仍只访问 Java `/api/v1/**`；前端不直连 Python，Java 不读写 Python 数据库，Python 不读写 Java 业务表。
- 时间固定 Asia/Shanghai，外部时间使用 ISO 8601 `+08:00`，槽位保持 30 分钟 `[start,end)`。
- HOT 草案确认先返回 HTTP 202 `PENDING + requestNo`，受理事务不创建 `meeting` 或槽位；MQ Consumer 才执行最终数据库裁决。
- MySQL 唯一约束仍是并发最终裁决，RocketMQ 按至少一次投递设计，`eventId + 业务终态` 保证消费幂等，没有宣称 exactly-once。
- `APP_HOT_BOOKING_ENABLED=true`、`ROCKETMQ_ENABLED=true`、`AGENT_CALLBACK_ENABLED=false` 已在最终 Java 容器核验。
- BOOKING_RESULT 回调适配器仅为 Day 3 边界骨架，默认关闭；Python 最终回调端点留到 Day 4/5。
- 基础 `compose.yaml` 解析后只发布 `frontend:80`；开发端口只来自 `compose.dev.yaml`。
- 没有引入 LangGraph、DeepSeek/OpenAI SDK、OR-Tools、RAG、HITL 或额外产品 Agent。

## 6. 实际验证记录

| 命令/检查 | 结果 | 摘要 |
|---|---|---|
| 固定 JDK 21 `./mvnw -B -ntp verify` | PASS | 主 Agent 最终独立复验：33 tests，0 failure/error/skip；Enforcer、JAR、Spotless 142 Java files 均通过 |
| `docker compose config --quiet` | PASS | 基础 Compose 有效；仅发布 frontend:80；无 `latest` |
| 组合 Compose `config --quiet` | PASS | 开发覆盖有效 |
| `docker compose ... up -d --build --wait` | PASS | 四个 `:day3` 应用镜像成功构建；完整栈达到健康状态 |
| 真实 MySQL Flyway/schema | PASS | V1-V4 全部 `success=1`；V4 六张表及 JSON/nullable 字段与规范一致 |
| `scripts/smoke-day1.ps1` | PASS | Nginx/Java/Python 健康；zhangsan 登录、当前用户和 3 个房间通过 |
| `scripts/smoke-day2.ps1` | PASS | 创建、幂等重放、同键异参拒绝、更新回滚/成功、查询、取消全部通过 |
| `concurrency-day2.py --mode room` | PASS | 100 请求：1 成功、99 冲突、成功 meetingId 唯一；p95 61.77 ms |
| `concurrency-day2.py --mode idempotency` | PASS | 100 请求：100 成功、meetingId 唯一；p95 182.27 ms |
| `python scripts/smoke-day3.py` | PASS | Tool 无 token/错误 audience/超限均拒绝；HOT=PENDING→SUCCESS；冲突请求→CONFLICT；改期/取消确认成功；SSE=AGENT_UNAVAILABLE |
| Day 3 数据库终态 | PASS | 成功 request `BR202608111225388B0843FACA` 只有 meeting 24；冲突 request `BR2026081112253978D4E91FC2` 无会议；两条命令各 1 条消费记录 |
| 草案无副作用 | PASS | `Day 3 no side effect f0e1a359` 保持 booking_draft PENDING，meeting 数为 0 |
| 通知和事件 | PASS | meeting 24 确认/取消各产生 2 条参会者通知；COMMAND/RESULT/领域 Outbox 全部 SENT、retry_count=0 |
| 重复 MQ 消息脚本 | PASS | Broker 接收完整 898-byte 重放消息；meeting 数=1、consume record 数=1、Consumer Diff Total=0 |
| Outbox 自动化 | PASS | 过期 SENDING 租约恢复并 SENT；发布失败按上限进入 RETRY/DEAD，2 个集成测试通过 |
| RocketMQ topic/group | PASS | `meeting-booking`、`meeting-domain` 和两个固定 Consumer Group 创建成功；最终消费积压 0 |
| Java/Python/Nginx 健康 | PASS | readiness `UP`；Python HTTP 200 `UP`；Nginx `UP`；本轮没有模型调用 |
| 脚本与静态扫描 | PASS | Python py_compile、全部 PowerShell AST、`.env` ignore、密钥/私钥/`latest` 扫描和 Day 4 范围扫描通过 |

## 7. 当前容器和服务健康状态

最终 `docker compose -f compose.yaml -f compose.dev.yaml ps`：

| 服务 | 镜像 | 最终状态 |
|---|---|---|
| MySQL | `mysql:8.4` | healthy；开发端口 13306 |
| Redis | `redis:7.4-alpine` | healthy；开发端口 6379 |
| RocketMQ NameServer | `apache/rocketmq:4.9.7` | healthy |
| RocketMQ Broker | `apache/rocketmq:4.9.7` | healthy；BOOKING consumer Diff Total=0 |
| RocketMQ store/topic init | `apache/rocketmq:4.9.7` | Exited (0)，预期一次性任务 |
| Qdrant | `qdrant/qdrant:v1.12.5` | healthy |
| Java | `meeting-scheduler-business-service:day3` | healthy；readiness UP；非 root；开发端口 18080 |
| Python | `meeting-scheduler-agent-service:day3` | healthy；HTTP 200；本轮没有模型调用 |
| Frontend/Nginx | `meeting-scheduler-frontend:day3` | healthy；宿主机端口 80 |

## 8. 已处理失败、已知问题与阻塞

### 已处理失败

1. MyBatis annotation SQL 曾把 `<=` 写成 XML 实体 `&lt;=`，H2 收到字面量；改为原生运算符后 Outbox 2/2 通过。
2. MySQL JSON 与 H2 VARCHAR 的 JDBC 返回形态不同，Tool 审计/Outbox 回放曾双重编码；加入统一 StoredJson 读取后两种数据库均通过。
3. Tool 失败审计最初会随业务事务回滚；改为独立 `REQUIRES_NEW` 后失败结果可持久化。
4. Outbox SENDING 若进程在发送后崩溃可能永久卡住；加入 30 秒租约与过期抢占恢复测试。
5. SSE 503 曾经用 `StreamingResponseBody` 异步写 JSON，真实 Tomcat async dispatch 丢失 SecurityContext 并截断 chunk；改为同步 ApiError 后真实 urllib Smoke 通过。
6. RocketMQ 4.9.7 的 Java 8 在 Docker Desktop cgroup v2 上初始化 `StoreUtil` 抛 NPE；`-XX:-UseContainerSupport` 修复后原消息恢复消费，`printMsg` 和 Consumer Progress 正常。
7. 重放脚本最初受 PowerShell 单行解包及 `mqadmin` shell 空格拆参影响；现强制数组、压缩 JSON、把空格编码为 `\u0020` 并校验消费积压。调试期一条截断探针只进入 MQ retry 路径，没有写业务数据库；最终 Diff Total=0。

### 已知问题

- 宿主机没有可用 JDK 21，Java 验证依赖固定 Maven/JDK 21 容器；这是环境限制，不是代码阻塞。
- Flyway 对 MySQL 8.4、测试期 Flyway 对 H2 2.3 输出“版本较新、建议升级”警告；真实迁移和全部测试成功，本阶段不升级锁定依赖。
- Java 镜像首次执行 `dependency:go-offline` 下载 RocketMQ 传递依赖约 5 分 45 秒；后续构建命中缓存约 40 秒。
- 没有 Git 提交；用户尚未授权自动提交。
- 本地开发端口仍因宿主机占用覆盖为 MySQL 13306、Java 18080；基础 Compose 的只发布前端策略不变。
- Day 3 没有前端功能变化；本轮验证公共 HTTP 入口，没有新增浏览器视觉断言。

### 当前阻塞

- 无 Day 3 阻塞。

## 9. Day 3 验收检查表

- [x] HOT 草案确认返回 HTTP 202 和 PENDING，不提前创建会议/槽位。
- [x] Outbox 与 booking request 在受理事务中原子写入。
- [x] 真实 RocketMQ 最终进入 SUCCESS 或 CONFLICT。
- [x] SUCCESS 写唯一会议，CONFLICT 不写会议。
- [x] BOOKING_RESULT、领域事件、参会者通知和消费记录在正确事务落库。
- [x] 完整重复消息不重复创建会议或消费记录，Consumer 积压为 0。
- [x] Tool API 无 Service Token、错误 audience 或超限参数均被稳定拒绝。
- [x] Tool 查询、审计、幂等重放和 CREATE/RESCHEDULE/CANCEL 草案通过。
- [x] SSE 上游不可用返回完整 503 JSON，不伪造 Agent 输出。
- [x] Java verify、真实 MySQL、四应用镜像、完整 Compose、Day 1/2/3 Smoke 和安全扫描通过。
- [x] 未进入 Day 4，没有 Multi-Agent、真实模型调用、OR-Tools、RAG 或 HITL。

## 10. 历史：Day 4 唯一明确起点（已完成）

下一任务只从 `agent-service/**` 的 DeepSeek Provider 抽象、可替换 fixture、Pydantic `AgentState`/Schema 和 Supervisor 最小路由切片开始，并接通 Java 现有只读 Tool Client 契约。先证明一个普通中文请求可被结构化、路由并通过 Java SSE 代理看到标准步骤；再在同一 Day 4 内按计划增加 Requirement/Policy/Scheduling Agent 和引用。不要提前进入 Day 5 的 OR-Tools、HITL、checkpoint 或热门结果恢复。

## 11. 历史：Day 4 启动清单（已由第 13 节取代）

> 以下是 Day 4 开始前的历史快照，不是下一任务的当前指令；下一任务只采用第 13.6 节的 Day 5 唯一起点并重新核验现场。

新对话不得依赖旧对话记忆，按下列顺序恢复上下文并核验现场：

1. 完整读取 `AGENTS.md`、`SPEC.md`、本文件、`docs/04-agent-spec.md`、`docs/05-data-and-api-spec.md`、`docs/07-test-and-evaluation.md` 和 `docs/08-one-week-development-plan.md`；发生冲突时遵守 `AGENTS.md` 规定的文档优先级。
2. 运行 `rg --files`、`git status --short`、`docker compose config --quiet` 和 `docker compose -f compose.yaml -f compose.dev.yaml ps`，以文件系统、Git 和可复现命令为准。本交接完成时全仓尚无 Git commit，现有文件显示为未跟踪内容；这些都是用户已有成果，不得删除、覆盖、重置或擅自提交。
3. 当前 Day 3 历史 Compose 状态已经过期；不要删除命名卷，按当前配置重新验证并如实更新本文件。
4. `.env` 是本地未提交文件，可能包含临时凭据。不得显示、复制到日志或写入文档；不得用 `.env.example` 覆盖它。根目录和 `docs/**` 只由主 Agent 修改。
5. Day 4 先完成最小 Golden Path：确定性 fixture 输入中文需求，得到受 Pydantic 校验的结构化状态，由 Supervisor 路由，调用一个 Java 只读 Tool，通过 Java SSE 代理输出规范事件，并把 Run/Step/Tool Call 元数据写入 `meeting_agent`。测试不依赖真实 DeepSeek 网络调用。
6. 最小切片通过后，才在 Day 4 范围内补齐 Supervisor + Requirement/Policy/Scheduling 三个专业 Agent、DeepSeek OpenAI-compatible Provider、有限重试、结构化交接和可验证引用。Retriever、普通 Tool 和确定性处理器不得包装成额外 Agent。
7. 严守边界：浏览器只访问 Java；Python 只调用 Java 白名单 Tool API且只读写 `meeting_agent`；Java 不实现 LLM 路由、Prompt、RAG 或求解；不提前实现 Day 5 的 OR-Tools、HITL、checkpoint、确认事务、热门结果恢复或新的 MQ 业务链路。
8. 完成后至少执行受影响模块测试、应用镜像构建、组合 Compose 配置检查、完整栈健康检查和一条通过 Java SSE 代理的真实 Smoke；更新本文件的完成项、命令结果、容器状态、阻塞和 Day 5 唯一起点。不得伪造未执行的验证。

Day 4 新对话的第一条安全检查命令：

```powershell
Set-Location D:\agent
rg --files
git status --short
docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml ps
```

## 12. 历史：Day 4 可复制提示词（已执行）

```text
你正在同一个工作区 D:\agent 继续开发“企业会议智能调度系统”。旧对话已完成并验收 Day 1、Day 2、Day 3；不要依赖旧对话记忆，以仓库文件和可复现命令为准。本次只执行 Day 4，完成后停止，不要提前进入 Day 5。

开始修改前，完整读取 D:\agent\AGENTS.md、D:\agent\SPEC.md、D:\agent\docs\HANDOFF.md、D:\agent\docs\04-agent-spec.md、D:\agent\docs\05-data-and-api-spec.md、D:\agent\docs\07-test-and-evaluation.md、D:\agent\docs\08-one-week-development-plan.md；随后用 rg --files、git status --short、docker compose config --quiet 和组合 Compose ps 核验真实状态。发生冲突时严格遵守 AGENTS.md 中的文档优先级。保留所有现有文件、未跟踪内容、用户改动、数据库和命名卷，不得 reset、覆盖、清理或擅自提交；不要泄露本地 .env。

按 AGENTS.md 的目录所有权使用 1 个主 Agent 和最多 3 个不再派生的内部 subagent并行开发：主 Agent 只负责根目录、deploy/**、scripts/**、docs/**、契约裁决、Compose 集成、Smoke 和 HANDOFF；Java subagent 只编辑 business-service/**；Python subagent 只编辑 agent-service/**；Frontend subagent 只编辑 frontend/**。没有实际工作时不要为凑数修改模块；只有主 Agent 可以编辑 docs/HANDOFF.md。主 Agent必须亲自审查并做跨服务验证。

Day 4 先完成最小 Golden Path：在 agent-service 中实现可替换的 DeepSeek OpenAI-compatible Provider 抽象和确定性 fixture、Pydantic AgentState/结构化 Schema、Supervisor 最小路由、Java 只读 Tool Client，以及 Run/Step/Tool Call 元数据落库；让一条普通中文调度需求经结构化、路由、一个 Java 只读 Tool 调用后，通过现有 Java SSE 代理输出 docs/05 规定的标准事件。自动测试不得调用真实 DeepSeek。

最小切片通过后，再按 Day 4 规范补齐 Supervisor + Requirement/Policy/Scheduling 三个专业 Agent、模型输出校验、最多一次模型修复重试、有限网络重试、Agent 间 Pydantic 状态交接、结构化 Trace 摘要和可验证引用。Agent 数量固定为 1 个 Supervisor + 3 个专业 Agent；Retriever、Tool 和确定性节点不能伪装成 Agent。未配置 DeepSeek Key 时健康接口仍须 HTTP 200/status=DEGRADED。

严格保持架构边界：浏览器只访问 Java 公共 API；前端不得直连 Python；Python 不读写 Java 业务表，只调用 Java 白名单 Tool API；Java 不实现 LLM 路由、Prompt、RAG 或 OR-Tools。不要提前实现 Day 5 的 OR-Tools、HITL、checkpoint、确认事务、热门结果恢复、额外 Outbox/RocketMQ 业务或产品范围外能力。

不要只输出计划，必须实际创建代码、迁移、配置和测试。完成后实际执行 Python uv sync --frozen --group dev、Ruff、mypy、pytest；执行所有受影响的 Java Maven verify、前端 type-check/build；执行 docker compose config --quiet、构建受影响镜像、启动完整组合 Compose、检查所有容器健康，并用确定性 fixture 做一条经 Java SSE 代理的真实 Smoke。若 Docker 或下载不可用，继续完成可执行部分并在 HANDOFF 中记录失败命令、错误摘要和恢复后的下一条命令，不得伪造成功。

最后由主 Agent 更新 docs/HANDOFF.md，写明 Day 4 是否通过、各 Agent 完成内容、关键文件、真实命令与结果、服务健康、未完成项和 Day 5 唯一明确起点。验收和交接完成后停止，不要自动开始 Day 5。
```

## 13. Day 4 完成交接（当前权威状态）

### 13.1 结论与范围

- **Day 4：PASS。** 本轮只实现并验收了 Day 4；没有开始 Day 5 的 OR-Tools、候选优化、HITL、checkpoint、确认事务、业务结果回调或热门结果恢复。
- Agent 数量固定为 4：`SupervisorAgent`、`RequirementAgent`、`PolicyAgent`、`SchedulingAgent`。Retriever、Java Tool Client 和 `compose_final` 都是明确命名的确定性组件，不伪装为 Agent。
- 浏览器仍只经 Nginx/Java 访问 `/api/v1/**`；Python 只读写 `meeting_agent` 元数据并调用 Java 白名单 READ Tool；Java 未实现 LLM 路由、Prompt、RAG 或 OR-Tools。

### 13.2 各执行角色完成内容

- **主 Agent / Coordinator：** 裁决并更新 Day 4 SSE、内部鉴权和 Trace 契约；更新 Compose/安全示例配置；新增真实栈 Smoke；亲自完成跨服务审查、镜像构建、Compose 健康检查和 Smoke。联调中修复 Java HTTP Client 默认 h2c 升级与 Spring Security 异步续接授权造成的 SSE 截断，并补充回归覆盖。
- **Java 开发 subagent（`business-service/**`）：** 实现 Java→Python SSE 字节透传、上游 2xx + `text/event-stream` 严格校验、稳定 `AGENT_UNAVAILABLE` 错误映射及嵌入式上游集成测试。主 Agent 收尾固定为 HTTP/1.1，并只放行已通过首个请求鉴权的 `ASYNC/ERROR` 调度类型。
- **Python 开发 subagent（`agent-service/**`）：** 实现 DeepSeek OpenAI-compatible Provider 抽象和无网络 fixture、Pydantic State/Schema、最多一次修复重试、有限网络重试、四 Agent LangGraph、Java READ Tool Client、Qdrant 确定性政策语料/可验证引用、内部 JWT/Service Token 校验，以及 Run/Step/Tool Call 安全摘要持久化与 Trace 查询。
- **Frontend：** Day 4 没有前端产品代码变更（按目录所有权不凑数修改）；仍实际运行前端 type-check 和生产 build 回归。

### 13.3 关键文件

- 契约、配置和验证：`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`、`compose.yaml`、`.env.example`、`scripts/smoke-day4.py`、`README.md`。
- Python：`agent-service/app/providers/{base,fixture,deepseek}.py`、`agent-service/app/schemas/agent.py`、`agent-service/app/workflow.py`、`agent-service/app/tools/java.py`、`agent-service/app/rag/policies.py`、`agent-service/app/persistence.py`、`agent-service/app/api/internal.py`、`agent-service/tests/test_internal_runs.py`、`agent-service/tests/test_provider_and_tools.py`。
- Java：`business-service/src/main/java/com/example/meeting/agentgateway/client/AgentSseProxyService.java`、`business-service/src/main/java/com/example/meeting/agentgateway/api/AgentGatewayController.java`、`business-service/src/main/java/com/example/meeting/common/security/SecurityConfiguration.java`、`business-service/src/test/java/com/example/meeting/agentgateway/AgentGatewaySseProxyIntegrationTest.java`。
- 持久化表已由既有 `agent-service/alembic/versions/0001_create_agent_metadata.py` 版本化创建；Day 4 复用其中的 `agent_run`、`agent_step`、`agent_tool_call`，没有通过 ORM 自动改表。

### 13.4 实际验证记录

| 命令/检查 | 结果 | 可复现证据 |
|---|---|---|
| `uv sync --frozen --group dev` | PASS | 依赖锁定同步成功。 |
| `uv run ruff check .` | PASS | `All checks passed!`。 |
| `uv run mypy app` | PASS | 26 个源文件无问题。 |
| `uv run pytest` | PASS | 14 passed；仅有 LangGraph 第三方 pending-deprecation warning。测试全部使用 fixture 或 HTTP mock，不调用真实 DeepSeek。 |
| 固定 JDK 21 容器中的 `./mvnw -B -ntp verify` | PASS | 36 tests，0 failure/error/skip，Spotless 通过。宿主机没有可用 JDK 21，因此使用 `maven:3.9.11-eclipse-temurin-21` 容器。 |
| `npm run type-check` 与 `npm run build` | PASS | Vue `vue-tsc --noEmit` 与 Vite 生产构建均通过。 |
| `docker compose config --quiet` 与组合 Compose `config --quiet` | PASS | 基础与开发端口覆盖解析均有效。 |
| `$env:AGENT_MODEL_PROVIDER='fixture'; docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait` | PASS | 应用镜像构建为 `:day4`，完整组合成功启动；未删除数据库或命名卷。 |
| `python scripts/smoke-day4.py` | PASS | 普通中文请求依次收到 `run.started`、Supervisor/Requirement/Scheduling step、`resolve_employees` READ Tool、`run.completed`；Trace 有 4 Step 和 1 Tool Call。政策请求经实际 Qdrant 返回 1 个含 `chunkId/title/headingPath` 的引用。 |
| `/internal/v1/health` 与无 Key 回归 | PASS | 当前组合健康接口为 HTTP 200；`test_health.py` 覆盖未配置 DeepSeek Key 时 HTTP 200 / `status=DEGRADED`。 |

### 13.5 当前服务状态与已处理问题

- 该历史组合 Compose 验收已由当前拓扑取代；`rocketmq-store-init`、`rocketmq-topic-init` 为预期 `Exited (0)`。
- 第一次真实 Smoke 暴露 JDK `HttpClient` 对明文上游发 h2c upgrade，而 Uvicorn 拒绝该升级；Java SSE client 已明确固定 HTTP/1.1，并有 `Upgrade` 头不存在的回归断言。
- 第二次真实 Smoke 暴露已提交 SSE 的异步续接被 Spring Security 再次拒绝；仅对 `ASYNC/ERROR` dispatcher 放行，原始 `REQUEST` 继续要求 EMPLOYEE/ADMIN。最终 Smoke 已验证完整终端事件。

### 13.6 未完成项和 Day 5 唯一明确起点

- 未完成项均属于后续 Day 5/Day 6 范围：OR-Tools Top 3、独立硬约束验证器、无解分类、`create_booking_draft`、LangGraph interrupt、ACCEPT/EDIT/REJECT、Redis checkpoint、确认与业务结果恢复，以及前端聊天/HITL/Trace 可视化。
- **Day 5 唯一明确起点：** 从 `agent-service/**` 的候选集合构建、OR-Tools 硬约束/软目标 Top 3 与独立硬约束验证器开始；先完成这三个确定性能力的测试，再进入 Day 5 下午的 draft/HITL/checkpoint 工作。

## 14. Day 5 完成交接（历史状态；已由第 15 节取代）

### 14.1 结论与范围

- **Day 5：PASS。** 已完成并验收 OR-Tools Top 3、独立硬约束验证、无解分类、DRAFT/HITL、Redis checkpoint、确认后的业务结果回调与 HOT 冲突恢复；本轮没有开始 Day 6。
- 运行时产品 Agent 固定为 4 个：`SupervisorAgent`、`RequirementAgent`、`PolicyAgent`、`SchedulingAgent`。CandidateBuilder、OR-Tools Solver、Validator、Retriever、HITL、Checkpoint 和 Java Tool 都是明确命名的确定性组件，不伪装为 Agent。
- 浏览器仍只访问 Java `/api/v1/**`；前端没有直连 Python。Python 只访问 Java 白名单 Tool API 和自身 `meeting_agent`/Redis DB 1/Qdrant 数据；Java 没有新增 LLM、Prompt、RAG 或 OR-Tools 实现。
- Day 4 基线已提交为 `31773e2 feat: complete day 4 agent foundation`。当前工作区保留本轮 Day 5 的未提交改动，未执行 Day 5 提交、重置、清理或卷/数据库删除。

### 14.2 各执行角色完成内容

- **主 Agent / Coordinator：** 裁决并更新 Day 5 API/SSE/恢复契约；将 Compose 默认镜像标签切换到 `day5`，向 Python 注入 DB 0 业务 Redis 与 DB 1 checkpoint Redis 配置，开启安全的 `AGENT_CALLBACK_ENABLED=true` 默认值；新增真实全栈 `scripts/smoke-day5.py`，亲自完成跨服务审查、构建、Compose 健康检查和 Smoke。
- **Java subagent（`business-service/**`）：** 新增受 JWT/RBAC 保护的 `POST /api/v1/agent/runs/{runId}/resume`（ACCEPT/EDIT/REJECT，EDIT 仅 roomId/startAt）；严格 HTTP/1.1 SSE 字节代理；新增安全的 public `GET /api/v1/agent/runs/{runId}` 和 `/trace` 代理；完成 `BOOKING_RESULT` 回调，使用业务记录 owner 的当前角色重签 Java AgentContext，非 2xx 由既有 MQ consumer 重投。Trace 会递归剥离 confirmation token、Authorization、JWT、Service Token 等敏感字段。
- **Python subagent（`agent-service/**`）：** 新增确定性 CandidateBuilder、CP-SAT one-hot/no-good Top 3 求解器、独立 Validator 和稳定无解分类；严格 Tool/模型结构化校验与有限重试；以 Redis 字符串实现真实 LangGraph `BaseCheckpointSaver`（DB 1，24 小时 TTL，fresh saver 可恢复 `interrupt`/`Command(resume=...)`）；接入 DRAFT/HITL/恢复视图/业务回调和 HOT CONFLICT 后重新读取、重求解、重新 Draft 的闭环。所有自动测试均使用 fixture/HTTP mock，不调用真实 DeepSeek。
- **Frontend：** 本轮按目录所有权未修改产品代码；仍实际执行 type-check 与生产 build 回归。

### 14.3 关键文件

- 契约、部署与 Smoke：`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`、`.env.example`、`compose.yaml`、`scripts/smoke-day5.py`。
- Python：`agent-service/app/scheduling/solver.py`、`agent-service/app/schemas/agent.py`、`agent-service/app/checkpoints/redis.py`、`agent-service/app/workflow.py`、`agent-service/app/api/internal.py`、`agent-service/app/tools/java.py`、`agent-service/tests/test_schedule_solver.py`、`agent-service/tests/test_redis_checkpoint.py`。
- Java：`business-service/src/main/java/com/example/meeting/agentgateway/api/AgentGatewayController.java`、`business-service/src/main/java/com/example/meeting/agentgateway/api/AgentRunResumeRequest.java`、`business-service/src/main/java/com/example/meeting/agentgateway/client/AgentSseProxyService.java`、`business-service/src/main/java/com/example/meeting/agentgateway/client/AgentBusinessResultCallback.java`、以及相应 integration tests。

### 14.4 可复现验证记录

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| `uv sync --frozen --group dev` | PASS | 锁定的 OR-Tools 9.14.6206 与 Redis 5.2.1 同步成功。 |
| `uv run ruff check .` | PASS | 全部检查通过。 |
| `uv run mypy app` | PASS | 31 个源文件无类型错误。 |
| `uv run pytest` | PASS | 48 passed；仅有上游 LangGraph pending-deprecation warning。 |
| `docker run --rm -v "${PWD}\\business-service:/workspace" -w /workspace maven:3.9.11-eclipse-temurin-21 ./mvnw -B -ntp verify` | PASS | 44 tests，0 failures/errors/skips，Spotless 与 Jar 打包通过。 |
| `npm ci`、`npm run type-check`、`npm run build` | PASS | Vue TypeScript 检查和 Vite 生产构建通过。 |
| `docker compose config --quiet` 与 `docker compose -f compose.yaml -f compose.dev.yaml config --quiet` | PASS | 基础与开发组合配置均有效。 |
| 以 fixture/Day 5 环境变量运行 `docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait` | PASS | 受影响镜像构建完成；未覆盖 `.env`，未删除数据库或命名卷。 |
| `python scripts/smoke-day5.py --restart-agent-service` | PASS | 普通中文需求经 Java SSE 产生候选和 HITL；重启 Python 后 EDIT 由 checkpoint 恢复并重求解；ACCEPT 成功且清理会议。HOT 路径先 PENDING，真实回调 CONFLICT 后 public recovery 视图返回新草案，再次 ACCEPT 成功并完成清理。 |
| `git diff --check` | PASS | Day 5 改动无空白错误。 |

### 14.5 服务健康、已知边界与下一步

- 该历史组合 Compose 验收已由当前拓扑取代；`rocketmq-store-init` 与 `rocketmq-topic-init` 为预期的 `Exited (0)` 初始化容器。
- 当前 Compose 的 Smoke 使用确定性 fixture；DeepSeek Provider 仍通过环境变量可替换。未配置 DeepSeek Key 时 Python health 保持 HTTP 200 / `DEGRADED`。
- 由于固定 `redis:7.4-alpine` 不含 RedisJSON/RediSearch，没有引入不兼容的官方 Redis checkpoint 扩展；自定义 Saver 是实际 LangGraph `BaseCheckpointSaver`，使用 Redis DB 1 字符串键、24 小时 TTL，并有跨 fresh saver 测试。
- Day 5 没有遗留实现阻塞。**Day 6 唯一明确起点（仅交接，不在本轮执行）：** 在 `frontend/**` 仅经 Java 公共 API 接入聊天 SSE、候选卡片 ACCEPT/EDIT/REJECT、恢复视图和安全 Trace 展示，再补相应浏览器可见验收。

## 15. Day 6 完成交接（当前权威状态）

### 15.1 结论与范围

- **Day 6：PASS。** 已完成聊天 SSE/HITL/安全 Trace、Run 刷新恢复、我的会议手动管理、会议室可用性和管理员管理的浏览器可操作切片；本轮没有开始 Day 7，也没有增加新的产品范围。
- 浏览器仍只调用 Java `/api/v1/**`。前端没有直连 Python；Java 没有新增 LLM、Prompt、RAG 或 OR-Tools；运行时产品 Agent 仍固定为 Supervisor、Requirement、Policy、Scheduling 四个。
- Day 4 已提交为 `31773e2 feat: complete day 4 agent foundation`；Day 5 与 Day 6 的已验收改动均保留在未提交工作区。未执行 reset、清理、卷删除、数据库删除或 `.env` 覆盖。

### 15.2 各执行角色完成内容

- **主 Agent / Coordinator：** 在 `docs/05-data-and-api-spec.md` 冻结 Day 6 会议室详情、30 分钟可用性和管理员管理契约（含 `ACTIVE|INACTIVE`、乐观版本和稳定错误码）；新增 `scripts/smoke-day6.py`；亲自审查跨服务边界、完整 Compose、Day 5 回归和真实浏览器流程。浏览器验收发现默认示例引用未在演示库解析的“李四”以及 REJECT 后残留候选卡，已回派并验证修复。
- **Java subagent（`business-service/**`）：** 新增 `GET /api/v1/rooms/{id}`、安全的 30 分钟 `[start,end)` availability 视图和 ADMIN-only 创建/更新/启停接口；EMPLOYEE 只见 ACTIVE，ADMIN 可见全部；实现 `ROOM_NOT_FOUND`、`ROOM_CODE_CONFLICT`、`ROOM_STATE_CONFLICT`，并增加会议室管理集成测试。
- **Frontend subagent（`frontend/**`）：** 实现 fetch SSE 解析、Agent 时间线、候选卡、ACCEPT/EDIT/REJECT、Run URL 恢复、安全 Trace、HOT 状态轮询；实现我的会议手动创建/编辑/取消和会议室可用性/管理员管理；Nginx 禁用 `/api/` 缓冲与缓存并设置 SSE 读取超时。默认 fixture 请求改为仅张三；候选卡只会在真实 `WAITING_CONFIRMATION` 且有草案/令牌时显示。
- **Python：** Day 6 未修改 `agent-service/**`；Day 5 已验收的 fixture、OR-Tools、HITL、checkpoint 与回调链路作为本轮跨服务回归对象继续通过。

### 15.3 关键文件

- 契约、Smoke 与交接：`docs/05-data-and-api-spec.md`、`scripts/smoke-day6.py`、`docs/HANDOFF.md`。
- Java：`business-service/src/main/java/com/example/meeting/room/api/AdminRoomController.java`、`RoomController.java`、`RoomItemView.java`、`business-service/src/main/java/com/example/meeting/room/application/RoomAdministrationService.java`、`RoomAvailabilityService.java`、`business-service/src/test/java/com/example/meeting/room/api/RoomManagementIntegrationTest.java`。
- 前端：`frontend/src/views/ChatView.vue`、`AgentRunView.vue`、`MeetingsView.vue`、`RoomsView.vue`、`frontend/src/components/{AgentTimeline,CandidateCards,HitlDecisionPanel,AppShell}.vue`、`frontend/src/api/{client,types}.ts`、`frontend/nginx/default.conf`。

### 15.4 可复现验证记录

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| 固定 JDK 21 容器中的 `./mvnw -B -ntp verify` | PASS | 主 Agent 复验 48 tests，0 failures/errors/skips；Spotless 与 Jar 打包通过。 |
| `npm ci`、`npm run type-check`、`npm run build` | PASS | 主 Agent 在最终前端修复后复验；`vue-tsc --noEmit` 与 Vite production build 均通过（49 modules）。 |
| `docker compose config --quiet` 与 `docker compose -f compose.yaml -f compose.dev.yaml config --quiet` | PASS | 基础和开发组合配置均有效。 |
| fixture 环境变量下的 `docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait` | PASS | 完整组合重建成功；随后前端修复又以 `up -d --build --wait frontend` 重建并健康。未覆盖 `.env` 或删除命名卷。 |
| `python scripts/smoke-day6.py` | PASS | Java 公共面真实验证：会议手动创建→修改→取消、Java 代理 SSE 候选→HITL REJECT→安全 Trace、会议室 availability 和管理员 RBAC。 |
| `python scripts/smoke-day5.py --restart-agent-service` | PASS | Day 5 回归：checkpoint 重启恢复、EDIT/ACCEPT/清理和 HOT CONFLICT recovery 均通过。 |
| 浏览器验收 `http://localhost` | PASS | 员工默认中文请求经 Java SSE 到达候选与 HITL；安全 Trace 刷新恢复且不显示确认令牌；REJECT 后候选区消失；会议室 30 分钟可用性可查。ADMIN 登录后可见编辑/停用与新增表单；我的会议已实际创建→修改→取消，并在页面重新加载后显示 `CANCELLED`。 |
| `git diff --check` | PASS | 当前 Day 5/Day 6 未提交改动无空白错误。 |

### 15.5 当前服务状态、未完成项与 Day 7 唯一明确起点

- 该历史 `docker compose ps` 结果已由当前拓扑取代；`rocketmq-store-init` 和 `rocketmq-topic-init` 是预期的 `Exited (0)` 初始化容器。
- Day 6 没有遗留实现阻塞。前端没有另引入测试框架；严格 TypeScript、生产构建、公共接口 Smoke 和真实浏览器验收共同覆盖本轮变更。
- **Day 7 唯一明确起点（仅交接，不在本轮执行）：** 按 `docs/08-one-week-development-plan.md` 的 Day 7，在不增加产品功能前提下先建立 Agent 评测集和可复现评测/压测证据，再进行空卷 Docker Smoke、README/架构材料和最终包装。

## 16. Day 7 完成交接（当前权威状态）

### 16.1 结论与范围

- **Day 7：PASS。** 已完成并验收 Java 并发扩充、40 条 Agent 离线评测、OR-Tools 确定性回归、空卷 Docker 三连 Golden Path、真实 HTTP 压测、README/报告/镜像内容清单，以及阻断 HOT 回调恢复的流式线程安全修复。
- 本轮没有增加产品功能，也没有开始 Day 8。运行时产品 Agent 仍严格固定为 Supervisor、Requirement、Policy、Scheduling 四个；Retriever、Solver、HITL、checkpoint、Tool 和评测节点均为确定性组件。
- Day 4 基线提交仍为 `31773e2 feat: complete day 4 agent foundation`；Day 5、Day 6、Day 7 的验收改动由本次完成提交记录。没有执行 reset、清理、数据库/命名卷删除或 `.env` 覆盖。

### 16.2 各执行角色完成内容

- **主 Agent / Coordinator（根目录、`docs/**`、`scripts/**`、Compose）：** 修正 `New-LocalEnv.ps1` 的安全占位替换与 `Test-Day7EmptyVolume.ps1` 的可重复空卷验收；扩展 Smoke 的显式 Compose/project 参数与失败诊断；为 Compose 增加保守资源上限；完成 README、部署文档、镜像清单和本交接。亲自执行全栈重建、空卷三连测、Day 5/6 公共 API Smoke、真实 HTTP 并发验证、静态安全扫描和最终 Compose 健康检查。
- **Java subagent（仅 `business-service/**`）：** 扩充 `MeetingConcurrencyIntegrationTest` 的 CT-03/04/05，并新增连续 HOT 受理后的再入回归；无生产分支改动，证明 Java HOT 标志和确认路径不会因前序 SUCCESS/CONFLICT 改走同步预约。
- **Python subagent（仅 `agent-service/**`）：** 新增离线 40 条评测集与可执行报告、fixture 兼容回归；为相同 Run 的 resume/callback 引入转换锁；修复 Starlette/AnyIO 与 LangGraph 同步生成器的跨线程续跑竞态，改为专用生产线程加逐帧 Queue SSE，并为 Redis checkpoint 的 load/mutate/save 加锁与同步 durability。新增线程亲和、checkpoint 并发和早到回调重试测试。
- **Frontend subagent：** Day 7 未增加产品功能；主 Agent 重新执行了已有 Vue TypeScript 检查和 production build，确认 Day 6 浏览器链路不回归。

### 16.3 关键文件

- 验收与交付材料：`README.md`、`docs/REPORTS.md`、`docs/image-manifest-day7.json`、`docs/06-docker-deployment.md`、`scripts/New-LocalEnv.ps1`、`scripts/Test-Day7EmptyVolume.ps1`、`scripts/smoke-day5.py`。
- Python：`agent-service/app/evaluation/`、`agent-service/app/api/internal.py`、`agent-service/app/checkpoints/redis.py`、`agent-service/app/run_locks.py`、`agent-service/app/workflow.py`、`agent-service/tests/test_agent_evaluation.py`、`agent-service/tests/test_internal_runs.py`、`agent-service/tests/test_redis_checkpoint.py`、`agent-service/tests/test_run_locks.py`。
- Java：`business-service/src/test/java/com/example/meeting/booking/MeetingConcurrencyIntegrationTest.java`、`business-service/src/test/java/com/example/meeting/agentgateway/AgentToolGatewayIntegrationTest.java`。

### 16.4 最终可复现验证记录

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| 固定 JDK 21 Maven 容器 `./mvnw -B -ntp verify` | PASS | 53 tests，0 failures/errors/skips；Spotless 与 Jar 通过。 |
| Python `uv sync --frozen --group dev`、Ruff、mypy、pytest | PASS | 79 packages audited；mypy 37 source files；**57 passed**，仅 1 条上游 LangGraph pending-deprecation warning。 |
| `uv run python -m app.evaluation` | PASS | 40 component-fixture cases；Intent/constraint/tool/component task success=1.0；60 个候选独立硬约束检查，0 违例；5/5 引用有效；networkCalls=0。 |
| `npm ci`、`npm run type-check`、`npm run build` | PASS | 49 modules production build。 |
| `docker compose -f compose.yaml -f compose.dev.yaml config --quiet` | PASS | 组合配置有效。 |
| `python scripts/smoke-day5.py --public-trace --restart-agent-service` | PASS | Java SSE、EDIT、checkpoint 重启、ACCEPT、HOT PENDING、MQ CONFLICT callback/replan 全部通过并以正常取消接口清理 Smoke 会议。 |
| `python scripts/smoke-day6.py` | PASS | 公共会议 CRUD、SSE/HITL/Trace、房间 availability 与管理员 RBAC 全部通过。 |
| 浏览器 `http://localhost/chat` | PASS | 张三登录后，普通中文需求经 Java SSE 显示 Supervisor/Requirement/Scheduling、Java READ/DRAFT Tool 摘要、3 个候选与 HITL；REJECT 后 Run 为 `CANCELLED`，未调用写入 Tool。 |
| `powershell -ExecutionPolicy Bypass -File scripts/Test-Day7EmptyVolume.ps1` | PASS | 独立 project `meeting-scheduler-day7-d0a2945b`、全新命名卷、三次 Golden Path（一次 Agent restart）；结束只停止容器/网络，不删除卷。 |
| `python scripts/concurrency-day2.py --mode room --requests 100 --workers 32` | PASS | 1 success / 99 conflict / 1 unique meeting；P50 500.78 ms、P95 803.27 ms、P99 1288.93 ms。 |
| `python scripts/concurrency-day2.py --mode idempotency --requests 100 --workers 32` | PASS | 100 success / 0 conflict / 1 unique meeting；P50 507.67 ms、P95 745.70 ms、P99 761.68 ms。 |
| 静态安全扫描（排除本地 `.env`） | PASS | `secretPrefixMatches=0`，`.env.example` 危险默认敏感值=0。 |

### 16.5 服务状态、已知限制与下一步

- 该历史组合 Compose 结果已由当前拓扑取代；两个 RocketMQ 初始化服务为预期的 `Exited (0)`。
- 最终主栈 Smoke 曾发现一个**旧本地环境**的 `AGENT_CALLBACK_ENABLED` 覆盖值关闭了回调消费者；未读取或修改 `.env`，仅为本次 Compose 进程以 `true` 覆盖，并确认 `.env.example` 的安全默认值已是 `true`。这不是代码、MQ 或 checkpoint 失败。
- Agent 评测是 fixture/InMemory RAG/确定性求解器基线，不能替代真实 DeepSeek 质量评估；HTTP 压测是单台 Docker 开发机结果，不能视为生产容量或 SLO。
- Day 7 没有遗留实现阻塞。**下一条允许任务：** 仅在用户明确给出 Day 8 或新的书面范围后再开始；本轮到此停止。

## 17. Day 8 前端产品化设计交接（尚未实施）

### 17.1 当前结论

- 用户已明确授权下一阶段优先进行前端与产品设计升级，暂不修改 Java 后端和 Python Agent。
- 产品工作名为 `MeetOps 企业协作编排助手`，视觉和信息架构以 Cal.diy 的紧凑企业 SaaS 风格为参考，并计划使用 shadcn-vue 作为 Vue 组件基础。
- 本轮只完成书面设计与执行提示词，**没有修改 `frontend/**`、`business-service/**` 或 `agent-service/**`，不得将本节解释为前端已经实施完成。**

### 17.2 权威文档

- `docs/09-frontend-product-redesign.md`：页面信息架构、视觉 Token、shadcn-vue 接入、真实/Preview 边界、实施阶段和验收标准。
- `docs/10-frontend-redesign-execution-prompt.md`：可直接复制到新 Codex 对话的完整执行提示词。

### 17.3 已核验基线

- 前端当前仍为 Vue 3.5.18、Vue Router 4.5.1、Vite 7.3.6、TypeScript 5.8.3、npm + `package-lock.json`。
- 当前尚未安装 Tailwind CSS 或 shadcn-vue，尚未配置 `@/*` 路径别名。
- 设计前只读基线验证：`npm ci`、`npm run type-check`、`npm run build` 均 PASS，Vite production build 为 49 modules。
- 工作区在新增本节与两份文档前为 clean；上一次中断没有遗留前端半完成代码。

### 17.4 下一条具体任务

在新的 Codex 对话中完整使用 `docs/10-frontend-redesign-execution-prompt.md`，先执行行为基线，再按“设计系统 → 应用壳 → 智能编排/HITL/Trace → 会议/会议室 → Product Preview → 浏览器验收”的顺序实施。浏览器仍只能访问 Java `/api/v1/**`，任何尚无后端支持的能力都必须明确标记为 Product Preview。

## 18. Day 8 前端产品化升级交接（当前权威状态）

### 18.1 结论与范围

- **Day 8 前端产品化升级：PASS。** `frontend/**` 已从 Day 6 功能演示升级为 `MeetOps 企业协作编排助手`；本轮没有修改 `business-service/**`、`agent-service/**`、Compose 拓扑或跨服务 API/事件语义。
- 浏览器仍只访问 Java `/api/v1/**`。`src/api/client.ts`、`src/api/types.ts`、auth、POST SSE、Run URL 恢复、HITL 和 HOT 状态轮询语义保持不变；Trace 不展示隐藏推理或确认令牌。
- 本节覆盖并取代第 17 节“尚未实施”的状态描述；第 17 节仍保留为设计决策来源和实施前基线。

### 18.2 已实现页面与组件

- 应用壳：桌面 240px 侧栏可折叠为 64px，移动端 Sheet 导航，分组信息架构、用户/部门/角色和退出；登录页改为中性 MeetOps 品牌样式。
- 智能编排：桌面 40/60 双栏、移动端“对话/编排结果”切换、真实 SSE 状态、需求摘要、Top 3 候选成本比较、自定义资源时间轴、政策引用、WAITING_BUSINESS_RESULT、HITL 审阅栏和按需 Trace Drawer。
- 管理页面：会议桌面紧凑 Table/移动 Card、真实创建/编辑/取消 Dialog/AlertDialog；会议室详情与 ADMIN 编辑 Sheet、启停 AlertDialog、30 分钟 `[start,end)` ResourceTimeline。
- Run 与待确认：Agent Run 详情使用脱敏 TraceTimeline/Tool Collapsible；待确认页明确说明后端没有跨 Run 列表接口，不伪造任务队列。
- Product Preview：异常重排使用 `frontend/src/demo/preview.ts` 的静态数据展示事件、受影响会议、Before/After、约束变化、放宽原因和未受影响项；会前会后展示人员、资源、议程、材料、政策、缺失项、决策、行动项、负责人、期限、依赖和任务草案。所有操作只提示“尚未连接后端”，不发送写请求。
- 业务组件已拆分：`WorkspaceShell`、`PageHeader`、`StatusBadge`、`AgentComposer`、`RequirementSummary`、`CandidateComparison`、`ResourceTimeline`、`RunStatusBar`、`HitlReviewBar`、`TraceDrawer`、`TraceTimeline`、`PlanDiff`、`EmptyState`、`LoadingState`、`ErrorState`、`ProductPreviewBadge`。
- 覆盖层统一使用 `useModalFocus`：支持 Escape、Tab 焦点循环、打开时初始焦点、关闭后焦点归还和 body 滚动锁；响应式断点覆盖 820px/520px，并遵守 reduced-motion。

### 18.3 技术版本与关键文件

- Tailwind CSS `4.3.3`、`@tailwindcss/vite 4.3.3`、shadcn-vue `2.8.2`、Reka UI `2.10.3`、`@lucide/vue 1.31.0`，均以精确顶层版本写入 `package.json` 并锁入 `package-lock.json`；未生成第二套锁文件。
- Vite 与 TypeScript 均配置 `@/* -> ./src/*`，现有 `/api` proxy 保留；`components.json` 使用 Neutral、CSS Variables、TypeScript 和 Lucide 配置。
- 主要文件：`frontend/src/styles.css`、`frontend/src/router/index.ts`、`frontend/src/views/{LoginView,ChatView,MeetingsView,RoomsView,AgentRunView,ApprovalsView,ReplanPreviewView,MeetingLifecyclePreviewView}.vue`、`frontend/src/components/**`、`frontend/src/composables/useModalFocus.ts`、`frontend/src/demo/preview.ts`。

### 18.4 可复现验证证据

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| `npm run type-check` | PASS | `vue-tsc --noEmit` 无错误。 |
| `npm run build` | PASS | Vite 7.3.6 production build，83 modules，主 CSS 34.14 kB。 |
| `npm install --include=dev` 后重复 type-check/build | PASS | 当前 Windows Node 24.14.0 环境完整安装开发依赖后可复现。 |
| `docker compose config --quiet` 与开发组合 config | PASS | 两套 Compose 配置有效。 |
| 新前端镜像 build + `up -d --force-recreate --wait frontend` | PASS | frontend、business-service、agent-service 与全部常驻依赖最终 healthy；RocketMQ 初始化容器仍为预期 `Exited (0)`。 |
| `python scripts/smoke-day6.py` | PASS | 会议 CRUD、Java SSE 候选/HITL/Trace、房间 availability 与 ADMIN RBAC 全部通过。 |
| `python scripts/smoke-day5.py --restart-agent-service` | PASS | 候选 3 个、EDIT 重规划、ACCEPT、checkpoint 重启恢复、HOT CONFLICT 恢复与清理全部通过。 |
| 浏览器 `1440x900`、`1024x768`、`390x844` | PASS | 登录成功/失败、EMPLOYEE/ADMIN、SSE→候选→WAITING_CONFIRMATION、Trace/Reject/Admin Sheet 的 Escape/焦点/滚动锁、Preview 和无横向溢出已实测。 |
| 静态扫描与 `git diff --check` | PASS | 无 Day 6 文案、`window.confirm`、前端直连 Python/内部 API、乱码或第二套锁文件；仅有 Windows LF→CRLF 提示。 |

### 18.5 已知环境差异、未连接能力与下一步

- 当前全局 npm 行为曾使一次普通 `npm ci` 只安装 463 个生产/CLI 依赖并缺少 `vue-tsc`；显式 `npm install --include=dev` 后 type-check/build PASS。随后再次执行 `npm ci --include=dev` 时遇到 Windows `node_modules/@swc/helpers` 文件锁 `EPERM`。这是本机依赖目录文件占用，不是源码或 lock 解析失败；在干净终端/关闭占用进程后应重跑标准 `npm ci` 作为环境复核。
- shadcn-vue CLI 的传递依赖 `validate-npm-package-name@8.0.0` 对本机 Node 24.14.0 给出 engine warning（要求 24.15.0 或受支持的 22.x/26.x），但实际安装、type-check、Vite build 和 Docker build 均成功。建议本地 Node 升到受支持的小版本后复跑 `npm ci`。
- 本地未提交 `.env` 覆盖会关闭 `AGENT_CALLBACK_ENABLED`；不读取或改写 `.env`，仅用进程级 `AGENT_CALLBACK_ENABLED=true` 重建 business-service 后，Day 5 HOT recovery PASS。仓库 `.env.example` 的安全默认值仍为 `true`。
- “异常重排”和“会前会后”明确是 Product Preview；后端没有对应写接口。待确认页也没有跨 Run 列表 API。这些页面不得解释为已连接真实后端。
- 下一步建议：在 Node 22.22.2+ 或 24.15.0+ 的干净环境复跑普通 `npm ci`，并补可长期执行的前端组件/浏览器自动化测试；不要为 Preview 发明后端接口。

## 19. Spec 1.1 受控 Agent 升级交接（当前权威状态）

### 19.1 结论与架构边界

- **本轮四项升级已落盘：** Scheduling 使用有预算的 `Plan -> Act -> Observe -> Verify -> Replan`；DeepSeek Provider 使用原生 `tools/tool_calls/tool`；同步 409 与异步 HOT 冲突进入统一的候选排除和事实刷新路径；Requirement 使用确定性 Evaluator–Optimizer，最多语义修复一次。
- 运行时 Agent 仍严格固定为 Supervisor、Requirement、Policy、Scheduling。Evaluator、Tool Gate、Verifier、OR-Tools、HITL 和 Conflict Repair Handler 都是确定性组件；没有引入 DeepAgents、第五个 Agent 或 Critic Agent。
- Java 仍是业务事实源，MySQL 仍是并发最终裁决；模型只获得 READ Tool Schema。草案由求解和独立验证后确定性创建，确认 Tool 只在 HITL ACCEPT 后调用。
- 全 Run 上限为 12 次模型调用、16 次 Tool 调用、20 个图节点和 2 次业务冲突重规划；Scheduling 单轮最多 4 次模型迭代，达到上限映射为稳定 `BUDGET_EXHAUSTED`。

### 19.2 关键实现

- `agent-service/app/agent_loop.py`：READ Tool Schema、Pydantic 参数校验、canonical context、稳定指纹/`toolCallId`、结果大小限制、Requirement Evaluator 和停止原因。
- `agent-service/app/providers/{base,deepseek,fixture}.py`：Provider-neutral Tool 消息协议、DeepSeek 原生多轮 Tool Calling、非思考模式、HTTP/响应边界和可复现 fixture 轨迹。
- `agent-service/app/workflow.py`：模型调用精确计数、受控 Scheduling Loop、独立求解/验证、SSE `agent.loop`、同步冲突重规划、HITL 再确认与稳定终态。
- `agent-service/app/api/internal.py`：服务端 `requestTime`、HOT 冲突反馈、checkpoint 恢复、重复回调幂等和重规划上限。
- READ Tool 的稳定业务 ID 包含 `run + tool + factEpoch + argumentsHash`：同一 epoch 重试幂等，EDIT 或冲突修复的事实刷新不会复用旧 Java 审计结果，也不会碰撞 Python Trace 主键。该问题由真实 Day 5 Smoke 发现后修复，测试 Fake 已改为保留生产 ID。
- `business-service/.../BookingConflictEvidence.java`：同步确认的服务端冲突证据；`BookingConfirmationService` 同时覆盖 Redis 预占冲突和数据库最终冲突；HOT `BOOKING_RESULT` 复用同一冲突类型常量。
- `.env.example`、`compose.yaml`、`SPEC.md`、`docs/04` 至 `docs/07`、`docs/11-controlled-agent-loop-design.md` 和 README 已同步模型名、预算、契约、评测分层和安全边界。

### 19.3 可复现验证证据

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| `uv run ruff check .` | PASS | Python 静态检查无错误。 |
| `uv run mypy app` | PASS | 38 个 source files 无类型错误。 |
| `uv run pytest -q` | PASS | **68 tests**，仅 1 条上游 LangGraph pending-deprecation warning；包含 Python Tool Trace 同语义重放幂等/异义拒绝回归。 |
| `uv run python -m app.evaluation` | PASS | `component-fixture-evaluation-v2`，40 cases，component task success=1.0，60 个候选硬约束 0 违例，5/5 引用有效，networkCalls=0。 |
| Java 21 定向 `-Dtest=AgentToolGatewayIntegrationTest test` | PASS | 10 tests；同步 409 证据、异步冲突载荷和重复消息终态幂等均通过。 |
| Java 21 完整 `mvn verify` | PASS | **54 tests**，0 failure/error/skip；Spotless、编译、测试和 Jar 打包均通过。 |
| 基础与开发组合 `docker compose ... config --quiet` | PASS | 两套 Compose 配置均可解析，新增模型/Tool/图预算环境变量已接线。 |
| 新 business/agent 镜像构建 | PASS | Java 镜像构建内再次执行 54 tests + Spotless；Python 镜像包含 Spec 1.1 Loop，实现后两服务重建为 healthy。 |
| `python scripts/smoke-day5.py --public-trace --restart-agent-service ...` | PASS | 初始 3 候选、`agent.loop` PLAN/VERIFY、EDIT 事实刷新与再 HITL、checkpoint 重启、ACCEPT、会议清理、HOT MQ CONFLICT callback/replan 全部通过。 |

### 19.4 安全、限制与下一步

- Spec 1.1 实现阶段没有使用用户在对话中提供的 DeepSeek Key。后续第 20 节在线核验只复用宿主环境中既有且不会回显的 Key；任何已暴露在聊天记录中的 Key 都应在 DeepSeek 控制台撤销并重新生成。
- 当前 `python -m app.evaluation` 是组件 fixture 评测，不是完整 Graph E2E，更不是实时模型质量。真实 DeepSeek 仍需用轮换后的 Key 单独执行并记录 provider/model、重复次数、延迟、Token、费用和失败轨迹。
- 本节记录的是切换前的 fixture 验收状态；当前真实模型状态与下一步以第 20 节为准。

## 20. DeepSeek V4 Flash 真实模型核验（当前本机状态）

### 20.1 切换与协议结论

- 2026-08-12 已将本机未提交 `.env` 的 `AGENT_MODEL_PROVIDER` 切换为 `deepseek`，并将旧模型名改为 `deepseek-v4-flash`；密钥只来自既有宿主环境变量，没有写入仓库、`.env`、命令输出、日志、fixture 或 Trace。
- 当前 `agent-service` 运行时回读为 `provider=deepseek`、`model=deepseek-v4-flash`、`deepseekConfigured=true`，基础 Compose 全部常驻服务 healthy。
- 官方文档确认该调用名当前指向 DeepSeek-V4-Flash-0731。真实 API 协议探针返回一个原生 `resolve_employees` `tool_call`，参数为合法 JSON，并包含 `prompt_tokens`、`completion_tokens`、缓存命中/未命中和总 Token 字段；因此 OpenAI-compatible Tool Calling 适配本身可用。

### 20.2 真实模型发现的问题

- 首次真实中文预约中，Supervisor 合理返回 `SCHEDULING`，但图路由只处理 `REQUIREMENT/POLICY`，导致请求被直接当作 FINAL、零 Tool 完成。已在 Supervisor 控制边界把不允许直接进入的初始路由归一为 `REQUIREMENT`，并新增 `SCHEDULING -> REQUIREMENT` 回归测试。
- 修复路由后，真实 Requirement 对自然语言仍不稳定：曾把“2人、15:00-16:00”幻觉成“张三、李四、2小时”；显式写出张三和李四时也多次返回 `requiredParticipants` 缺失。Prompt 已补充 intent、姓名逐字复制、人数仅作为容量和显式时间忠实规则，但重复在线请求仍未稳定通过 Requirement。
- “VIP会议室有哪些使用规则”被 Supervisor 路由到 Requirement/Clarification，而不是 Policy，最终要求补预约字段且没有引用。说明真实模型前置路由必须增加确定性语义校验或降级策略，不能只依赖结构合法性。
- 这些请求通常在约 1-5 秒内返回，但没有一条自然语言预约真实跑到 Java Tool、OR-Tools 候选和 HITL；因此当前只能宣称 V4 Flash 已接通和原生 Tool Calling 协议已验证，不能宣称真实模型 Golden Path 已通过。
- SSE/Trace 终态数据在数据库内为正确 UTF-8；PowerShell/Python 控制台曾显示韩文样式乱码是宿主输出解码现象，不是 Java -> Python 请求体损坏。Java 代理已有 UTF-8 请求体和中文透传集成测试。

### 20.3 本轮代码与验证

- `agent-service/app/workflow.py`：增加 Supervisor 初始路由控制边界，并强化 Requirement 的源事实约束 Prompt。
- `agent-service/tests/test_provider_and_tools.py`：新增真实模型暴露问题的确定性路由回归。
- 定向验证：ruff PASS、mypy 38 source files PASS、29 tests PASS；真实模型协议探针 PASS。完整 Python 回归为 **68 tests PASS**，仅 1 条上游 LangGraph pending-deprecation warning。

### 20.4 下一步优先级

1. 为 Supervisor 增加独立的业务路由 Evaluator（至少覆盖 policy/create/modify/cancel），非法或低置信度路由进入一次受控修复，而不是直接相信模型枚举值。
2. 为 Requirement Evaluator 增加“源文本忠实度”规则：显式姓名集合、人数/容量、时间区间/时长、设施别名和 intent 必须能追溯到原始文本；不一致时一次修复，仍失败则澄清。
3. 新增 live-model evaluation runner，使用版本化语料重复运行并记录 route/constraint/tool/terminal 成功率、P50/P95、Token 和费用；fixture 评测只保留为确定性回归。
4. 真实模型达到门槛后，再补同步冲突与 HOT 冲突的在线轨迹；当前不要用 fixture 的 100% 指标代表 V4 Flash 质量。

## 21. 真实模型修复任务设计交接（尚未实施）

- 已新增 `docs/12-live-model-agent-repair-plan.md`：冻结人数/姓名语义、安全默认、三层验证、路由/Requirement/Tool Loop、改期取消 HITL、Trace/Token 和真实模型评测门禁。
- 已新增 `docs/13-live-model-agent-repair-execution-prompt.md`：可完整复制到新 Codex 对话，要求直接实施代码、测试、真实模型评测、Compose 联调和交接，并包含反伪完成规则。
- 本节只代表修复设计和执行提示词已经完成；除第 20 节已记录的 Supervisor 控制边界/Prompt 修正外，Source Fidelity Evaluator、RequirementDraft、完整 MODIFY/CANCEL、Loop 持久化、Token 接线、前端 Loop 展示和 live-model runner **尚未实施**。
- 下一条具体任务：在新对话完整使用 `docs/13-live-model-agent-repair-execution-prompt.md`，按 Slice A → B → C → D → E 实施；不得跳过真实自然语言门禁或用 fixture 指标替代。

## 22. 真实模型修复实施交接（当前权威状态）

### 22.1 结论与范围

- 第 21 节设计已实施，当前状态以本节覆盖：Source Fidelity、受控 Route/Requirement 修复、原生 READ Tool Loop、CREATE/RESCHEDULE/CANCEL 三类 HITL、运行指标持久化、前端 Loop/Trace 展示、真实模型 component/trajectory runner 均已落盘。
- 运行时仍严格是 Supervisor + Requirement/Policy/Scheduling；Evaluator、Normalizer、Tool Gate、Retriever、Solver 与 HITL Handler 都是确定性组件。浏览器仍只访问 Java，Python 未跨库读取 Java 业务表。
- Java 对三类同 Run/operation 新草案会原子作废旧 PENDING token；取消预览绑定 meeting version。最近会议 Tool 只返回 `CONFIRMED`，避免已取消记录污染 MODIFY/CANCEL。
- `hitl.required` 已统一为 `actionType=CREATE|RESCHEDULE|CANCEL` 可辨别草案；`agent.loop` 与 Trace 已持久化 phase/iteration/decision/feedback/预算/stopReason，Run 持久化 provider/model/prompt/schema/token/耗时。

### 22.2 最终验证证据

| 层级/命令 | 结果 | 证据 |
|---|---|---|
| Java 21 `./mvnw -B -o -ntp verify` | PASS | 61 tests，0 failure/error/skip；Jar 与 Spotless PASS。最终 business 镜像构建内再次执行同一 61 tests 并 PASS。首次在线 build 曾因 Maven Central 下载中断失败，重试后成功，不计作测试失败。 |
| Python `ruff` / `mypy app` / `pytest` | PASS | Ruff PASS；mypy 39 source files；76 passed，仅 1 条上游 LangGraph pending-deprecation warning。 |
| Frontend `npm ci` / `type-check` / `build` | PASS | 463 packages；仅 Node 24.14.0 对一个传递依赖的 engine warning；Vue type-check PASS，Vite build 88 modules PASS。 |
| 基础/开发 Compose config、镜像 build、`up --wait` | PASS | 两套历史 config PASS；Java/Python/Frontend 镜像均完成构建；初始化容器 Exited (0)。未删除命名卷。当前拓扑需重新验证。 |
| `component-fixture` 40 条 | PASS（组件门禁） | `networkCalls=0`；Intent/Tool/Citation=100%，Constraint F1=96.76%，硬约束 60 candidates/0 violation；`componentTaskSuccess=82.5%`，不是 E2E。 |
| `live-model-component` core 12 × 3 | PASS | DeepSeek `deepseek-v4-flash`：36 samples；Route 100%，Intent 97.22%，Constraint F1 100%，Tool 94.44%，Source violation 0，Native Tool 100%，Citation 100%；P50 3.53s/P95 6.89s。 |
| `live-model-component` full 40 × 1 | **FAIL** | Route/Intent 100%，Constraint F1 89.05%，Native Tool 100%；但 Tool 80% < 90%、Source violation 1 > 0、Citation 80% < 100%。因此整体 live-model component 严格结论为 FAIL，不能用 core PASS 覆盖。 |
| `live-model-trajectory` 公共 Java API | PASS | 8 条隔离轨迹 7 PASS，成功率 87.5% >= 80%；P50 9.16s/P95 11.60s。覆盖 CREATE、Policy、MODIFY/CANCEL preview、REJECT/ACCEPT 与 HITL 前快照。唯一失败为固定 ID 9001 在保留数据中不存在，系统准确返回 `MEETING_NOT_FOUND`；动态 ID CANCEL 成功轨迹已验证。 |
| `git diff --check` | PASS | 仅 Windows LF→CRLF 提示，无 whitespace error。 |

脱敏报告：`artifacts/fixture-evaluation.json`、`artifacts/live-eval/component-core.json`、`artifacts/live-eval/component-full.json`、`artifacts/live-eval/trajectory-final.json`。报告不包含访问令牌、确认令牌、Key 或隐藏推理。

### 22.3 已知失败、环境和下一步

- `live-model-component` 全量 40 尚未达门禁，集中在旧语料中缺少可执行时间窗的 RECOMMEND/FIND、空列表被模型写入 `missingFields`、一次多人协调 evidence 不忠实，以及 Policy 模型未稳定选中问题期望的唯一 chunk。此项必须保持 FAIL；下一任务应修语料版本与 Production Requirement/Policy 选择边界后重新执行完整 40 条，不能只重跑成功样本。
- core component 与 Compose 轨迹已 PASS，说明当前 Golden Path 可演示；不应把它解释为 full 40 的全面质量通过。
- 当前本机 Compose 使用进程级 `AGENT_MODEL_PROVIDER=deepseek`、`DEEPSEEK_MODEL=deepseek-v4-flash`，所有常驻服务 healthy。Key 只由未提交环境注入，整个过程未回显或落盘；对话中曾暴露的任何旧 Key 仍应在供应商控制台轮换。
- 下一条具体任务：只处理 full 40 component 的失败分类并重跑 `core 12×3 + full 40×1`；在 full 40 达标前不得将整体 live-model component 标记 PASS。

## 23. 会议制度知识库源文档（当前状态）

### 23.1 已完成

- 已新增 `deploy/rag-documents/`，包含 MeetOps 科技有限公司会议制度知识库的 22 份 UTF-8 Markdown 源文档，文件名按知识库清单固定为 `01-...md` 至 `22-...md`。
- 每份文档均包含 Front Matter：`documentId`、`title`、`documentType`、`department`、`version`、`effectiveDate`、`status`、`priority`、`timezone`；生效日期统一为 `2026-08-01`，时区统一为 `Asia/Shanghai`。
- 每份文档均包含“适用范围”“规则正文”“例外与冲突处理”“常见问题”“RAG 测试问题”章节；每份附 6 至 7 个自然语言 RAG 测试问题，不附答案。
- 文档内容已统一冻结并反复校验：30 分钟槽位、`[start,end)`、专项规则优先于通用规则、Agent 创建/改期/取消必须人工确认、EDIT 后重新校验、系统不得自动移动他人会议、未找到依据时返回“未找到可验证证据”，以及真实邮件/视频供应商/IoT/SSO/多级审批边界。

### 23.2 可复现验证证据

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| `Get-ChildItem deploy\\rag-documents -Filter *.md` | PASS | 22 个文件，文件名与用户清单一致。 |
| Front Matter、必备章节和 `documentType` 枚举检查 | PASS | 22/22 通过；类型仅使用规范允许的 7 个枚举。 |
| 汉字数检查 | PASS | 22/22 约 1,500–1,700 个汉字，满足 1,500–3,000 字目标范围。 |
| RAG 测试问题检查 | PASS | 22/22 每份 6 或 7 个问题，均位于独立 `RAG 测试问题` 章节。 |
| `git diff --check -- deploy/rag-documents` | PASS | 无空白错误。 |

### 23.3 RAG ingestion 已实现

- 已实现受控 Markdown/文本型 PDF 导入器：UTF-8/LF 规范化、严格 Front Matter 校验、PDF 同名 YAML sidecar、ATX 标题路径切片、PDF 页码保留、稳定 `chunkId`、SHA-256 checksum 去重、失败状态和重复执行幂等。扫描型 PDF 明确失败，当前不支持 OCR。
- `rag_document` 使用 `INDEXING -> INDEXED|FAILED` 状态；新增 checksum 唯一约束。相同 checksum 已索引时跳过；同一 `documentId` 内容变更时先 upsert 新切片，再删除该文档不再存在的旧切片。源文件删除不会自动删除已索引文档。
- Qdrant payload 已包含 `chunkId/documentId/documentType/title/headingPath/page/content/version/priority/checksum`。检索使用确定性向量召回加标题、章节和正文关键词重排，不引入外部 embedding、Rerank 服务或额外 Agent；Citation 仍只允许来自本次检索候选。
- Compose 新增一次性 `rag-init`：先执行 Alembic，再从只读挂载的 `/app/rag-documents` 导入；`agent-service` 仅在 `rag-init` 成功后启动。仍未新增公共上传接口、真实邮件/视频供应商、OCR、SSO 或多级审批。
- 主要实现位于 `agent-service/app/rag/ingestion.py`、`agent-service/app/rag/ingest.py`、`agent-service/alembic/versions/0003_enforce_rag_document_checksum_uniqueness.py`、`scripts/smoke-rag-ingestion.py` 和 `compose.yaml`；契约已同步到 `docs/04-agent-spec.md`、`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`、`docs/07-test-and-evaluation.md`。

### 23.4 可复现实施证据

| 命令/检查 | 结果 | 证据 |
|---|---|---|
| `uv sync --frozen --group dev`、Ruff、Mypy、Pytest | PASS | Qdrant 客户端锁定为与服务端同系列的 `1.12.2`；Ruff PASS，Mypy 41 source files，**90 passed**；仅 1 条上游 LangGraph pending-deprecation warning。 |
| `docker compose build rag-init` | PASS | 正式镜像构建成功，包含 `pypdf==6.14.2` 和 `qdrant-client==1.12.2`；Dockerfile 将 uv HTTP 超时设为 120 秒以覆盖 OR-Tools 大包下载。 |
| 首次正式数据导入 | PASS | Alembic `0003_rag_checksum_unique` 已应用；22 份文档全部 `INDEXED`，共 307 个切片写入 Qdrant。 |
| 正式镜像 `docker compose run --rm --no-deps rag-init` | PASS | 对同一批源文档复跑：`indexedCount=0`、`skippedCount=22`、`chunkCount=307`，确认 checksum 去重和幂等。 |
| `scripts/smoke-rag-ingestion.py` | PASS | MySQL 为 22 个 `INDEXED` 文档/307 切片；Qdrant 为 22 个文档/307 个制度切片；VIP 会议室和架构评审问句均在 Top 1 命中对应专项文档。 |
| 基础与开发 Compose `config --quiet` | PASS | `compose.yaml` 与 `compose.yaml + compose.dev.yaml` 均有效。 |

### 23.5 边界与下一步

- 当前导入入口是部署期 CLI/一次性容器，不提供用户在线上传、删除或管理 API；PDF 只处理可提取文本并要求 Front Matter 或同名 `.yaml/.yml`，不做 OCR。
- 4 条内置种子仍保留为 fixture 和向后兼容语料；22 份正式制度文档已登记并索引。不得再将当前状态描述为“仅内置种子”。
- 本切片没有遗留实现阻塞。后续若扩展管理能力，应先设计受控的重建/删除命令和权限边界，不能因源目录文件消失而静默删除 Qdrant 与 `rag_document` 数据。
