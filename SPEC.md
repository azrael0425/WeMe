# 系统Spec总入口

## 1. 基线

- Spec版本：1.3。
- 项目类型：简历展示型个人完整项目。
- 开发周期：全职一周，使用Codex辅助开发。
- 项目语言：Java、Python、TypeScript。
- 默认时区：Asia/Shanghai。
- 部署基线：Docker Compose。

本文记录已经与项目开发者确认的冻结决策。详细设计以 `docs/` 下的专项文档为准。

## 2. 已冻结决策

1. 新建原创项目，不直接Fork两个参考仓库。
2. Java只参考 `lab-resource-reservation-platform` 的业务后端、预约并发、Redis、RocketMQ和Outbox设计。
3. Python只参考 `agents-from-scratch` 的LangGraph、结构化路由、Tool Calling、HITL、记忆和评测设计。
4. Agent固定为Supervisor + Requirement/Policy/Scheduling三个专业Agent。
5. 前端只访问Java，Java代理Python SSE；Python只通过Java内部Tool API访问业务。
6. 使用DeepSeek OpenAI-compatible API。
7. 使用OR-Tools进行确定性会议调度。
8. 会议时间固定为30分钟槽位。
9. 普通预约同步执行；热门时段异步受理并由RocketMQ最终裁决。
10. 热门预约结果可恢复LangGraph；冲突后重新规划并再次确认。
11. 所有预约、改期和取消都必须由用户显式确认：Agent发起的写操作使用ACCEPT/EDIT/REJECT HITL并在EDIT后重新校验；手动页面的最终提交本身作为人工确认，不绕行LangGraph，但必须复用同一Java业务校验与事务服务。
12. RAG只存会议制度和会议规范，支持Markdown与文本型PDF，不做OCR和Rerank。
13. 用户偏好只保存明确表达的内容，不自动学习隐式偏好。
14. 不接入外部协作工具；不做邮箱、视频会议链接创建和空调。`VIDEO_CONFERENCE` 仅表示会议室具备混合会议设备。
15. 不做故障注入、完整OpenTelemetry平台、多租户、SSO和复杂审批。
16. Agent 主链路升级为受控 `Plan -> Act -> Observe -> Verify -> Replan` 循环，不引入 DeepAgents，不增加运行时 Agent 数量；循环只覆盖理解、只读 Tool、验证、求解和冲突重规划，写操作仍由 HITL 后的确定性节点执行。
17. DeepSeek 使用 OpenAI-compatible 原生 `tools/tool_calls`；模型 Tool 参数必须经 Pydantic Schema、业务上下文约束、权限/风险策略和重复调用指纹校验后才能执行。
18. Requirement Agent 内部采用 Evaluator-Optimizer：先生成结构化需求，再由确定性语义评估器检查时间基准、字段完整性和跨字段一致性，最多携带结构化反馈修复一次；Evaluator 不是新的产品 Agent。
19. Scheduling Agent 对 Java 并发冲突生成结构化修复反馈，保留原硬约束、排除失败候选并最多重规划2次；超过预算必须进入可解释终态或重新请求用户决策，不允许无限循环。
20. 自然语言字段必须保留来源忠实度：人数只决定容量，不得自动扩写为姓名；显式姓名、时间、时长、设施与 intent 必须经确定性 Source Fidelity Evaluator 核对，缺失标题和会议类型使用稳定默认值，不要求无意义澄清。
21. Agent 的 CREATE、RESCHEDULE、CANCEL 使用可辨别草案结构和各自的确认 Tool；改期保留未编辑字段并显示 Before/After，取消显示目标会议。任何 EDIT 都必须作废旧 token、重新读取事实并生成新 token，三类草案在 ACCEPT 前均不得改变正式会议。
22. 真实模型质量门禁独立于 fixture：组件评测与 Compose 完整轨迹分别报告 provider/model、Prompt/Schema 版本、重复次数、终态、失败分类、延迟与 API Token；未配置 Key 或未实际执行必须标记 SKIPPED/FAIL，不能用 fixture PASS 替代。
23. 创建会议的需求收敛采用服务端持久化的多轮槽位状态：时间窗口、会议时长和必需参会范围是进入 Scheduling 前的刚需；标题、会议类型、地点和设备可以使用已冻结的安全默认。`WAITING_USER_INPUT` 只能通过独立补充输入动作恢复，不能与 HITL `ACCEPT/EDIT/REJECT` 混用。
24. 部分时间表达允许确定性补全并向用户展示依据：几号默认当前月、周几默认当前周、只说时刻默认当天；上午/早上、中午、下午、晚上分别映射为 `06:00-12:00`、`11:00-14:00`、`12:00-18:00`、`18:00-次日06:00`。补全后若已过去、日期无效或与显式信息冲突必须追问，禁止静默滚动到下月或下周。
25. 人数仍只决定容量，不得扩写为姓名；“我的小组/同组人员”可以解释为当前登录用户所属部门范围，由 Java 根据 AgentContext 查询 ACTIVE 成员并返回可纠正的通讯录名单。模型不得提供可信 userId、部门名或自行编造成员。
26. 非刚需的设备、地点等可选要求必须区分 `UNSPECIFIED`（用户尚未说明）、`EXPLICIT`（用户明确提出）和 `CLOSED`（用户明确表示没有其他要求）；`UNSPECIFIED` 只提示一次且不阻塞 Scheduling，不得展示为“已明确”。
27. 多轮修改按确定性增量应用于最后有效 Draft：新增、删除和整体替换参会人语义不得混用；“去掉/不参加/请假不会来”等删除表达只能删除历史已验证名单中的同名人员，不能让模型重建整份名单或重新扩写部门成员。
28. 同一聊天中 Run 因模型、Tool 或预算失败后，用户可显式从该失败 Run 的最后有效 Requirement checkpoint 创建一个继承基线的新 Run。新 Run 只继承需求 Draft、槽位来源、revision、可选项关闭状态和已验证人员，不继承候选、确认令牌、写入状态、调用计数或幂等指纹；不得自动从已成功、已拒绝或待确认 Run 继承。
29. 改期和取消必须把“目标会议选择条件”与“改期后的目标时间窗口”分开建模。目标会议必须由 Java 返回的可管理会议事实唯一命中；无法唯一命中时进入澄清，不得静默选择最近一场。改期默认继承原会议标题、类型、时长、必需/可选参会者和未被用户修改的要求；仅把用户明确修改的字段覆盖到目标方案。忙闲与房间查询可以携带经 Java 鉴权校验的 `excludeMeetingId` 排除目标会议自身占用，但不得排除其他会议。
30. 调度无解必须返回结构化、可核对且有界的原因：至少包含请求窗口、会议时长、无解类别和松弛建议；若由必需参会者忙碌导致，还必须列出相关人员、冲突时间段和可公开的会议标识。前端必须展示该证据，禁止只返回“没有共同空闲时间”等固定泛化文案。
31. 会议室从 `ACTIVE` 变为 `INACTIVE` 时，Java 必须在同一事务内为该房间内尚未开始的 `CONFIRMED` 会议创建幂等异常重排单，并只向会议发起人发送资源失效站内通知；系统不得自动移动会议或替用户放宽硬约束。
32. 异常重排提供两条受控处置路径：异常重排页只允许在原时段内选择满足容量、设备和占用硬约束的替代房间并显式提交；需要改变时间、参会人、设备或地点约束时进入智能编排，继续复用 RESCHEDULE 草案、Top 3、独立验证器和 `ACCEPT/EDIT/REJECT` HITL。
33. 异常单以 Java 为事实源，状态固定为 `OPEN/RESOLVED/RESTORED/CANCELLED`。会议通过手动入口或 Agent 改期后必须在同一业务事务中关闭对应异常单；会议取消则标记 `CANCELLED`，原会议室恢复且会议尚未移动则标记 `RESTORED`。状态竞争使用会议版本与异常单版本共同裁决。
34. 约束变化必须可解释：异常页和智能编排都要区分已改变约束、未受影响硬约束和用户主动放宽项。异常页的固定时段替换不是新的求解器；跨时段或约束放宽仍只由 Python OR-Tools 生成候选，并在 Java 写入前重新校验。
35. 会前会后扩展只实现不依赖外部平台的闭环：会前议程、材料元数据、动态准备清单、24 小时与 30 分钟站内提醒；会后自动完成、文本会议记录、纪要/决策/行动项草案、发起人 HITL 和行动项站内催办。明确不实现 RSVP、签到、附件二进制存储、政策检查结果绑定、统计复盘或外部协作平台同步。
36. Java 是议程、材料元数据、准备状态、会后草案、正式纪要、决策和行动项的唯一事实源。Python 不跨库持久化这些业务对象；它只让现有 Requirement Agent 以 `POST_MEETING_ANALYSIS` 模式把 Java 提供的已鉴权会议快照和文本记录转换为结构化草案，不新增运行时 Agent。
37. 会后 Agent 输出只是待审草案。`EDIT` 只更新草案并作废旧版本，`REJECT` 不产生正式记录，只有发起人或 ADMIN 的 `ACCEPT` 才在一个 Java 事务内写入正式纪要、决策和行动项。行动项负责人必须来自会议参与者事实，截止时间与状态由 Java 最终校验。
38. `CONFIRMED` 会议到达 `endAt` 后由 Java 有界定时扫描幂等转为 `COMPLETED`；提醒、缺失项通知、行动项临期和逾期催办均只使用现有站内通知。所有定时投递必须有数据库唯一键去重，不接入邮件、短信、日历或即时通信平台。
39. 用户主路径必须以完整中文产品语义呈现，不展示技术枚举、原始人员 ID、测试会议或无助于决策的实现说明。周视图固定按周一至周日、08:00 至 00:00 展示；会议室日期或筛选变化必须自动刷新可用性并丢弃过期响应。演示环境使用可审计的 Flyway 数据扩充人员、会议室、部门会议和站内消息，不删除并发与幂等测试历史，但默认业务视图不展示这些技术记录。
40. 会议制度知识库对所有登录用户提供只读浏览；ADMIN 可通过 Java 公共 API 上传 Markdown/文本型 PDF、在线编辑 Markdown 和显式删除。Python 继续持有文档元数据与 Qdrant 索引；删除必须保留 tombstone，防止部署期只读种子在重启后静默恢复。PDF 不做 OCR，内容变更必须完整重建该文档切片。

## 3. P0交付

- Java高并发预约与并发正确性测试。
- Redis Lua、MySQL唯一槽位、幂等、Outbox和RocketMQ。
- Supervisor + 3个专业Agent。
- DeepSeek Tool Calling。
- OR-Tools Top 3方案与硬约束验证。
- 简化RAG与有效引用。
- 持久化HITL和异步结果恢复。
- Vue聊天、会议基础管理和Agent Trace。
- Vue 会前准备、会后草案审核和行动项闭环。
- Vue 会议制度知识库浏览与 ADMIN 文档管理。
- Docker Compose一键部署。
- Agent评测和Java压测报告。
- 原生 Tool Calling 轨迹评测、有界 Loop 停止条件评测、Evaluator 修复评测和并发冲突重规划评测。

## 4. 文档优先级

发生冲突时按以下优先级解释：

1. `SPEC.md`中的冻结决策。
2. `docs/01-functional-spec.md`中的业务验收条件。
3. `docs/02-system-architecture.md`中的服务边界和一致性策略。
4. Java、Agent、数据/API和Docker专项规范。
5. 一周开发计划中的时间安排。

实现过程中如需改变P0范围，应先更新本文件及关联验收条件，再修改代码。

## 5. 文档导航

- [项目总览与范围](docs/00-project-overview.md)
- [功能与验收规范](docs/01-functional-spec.md)
- [系统架构规范](docs/02-system-architecture.md)
- [Java 后端规范](docs/03-java-backend-spec.md)
- [Multi-Agent 规范](docs/04-agent-spec.md)
- [数据模型与 API 契约](docs/05-data-and-api-spec.md)
- [Docker 部署规范](docs/06-docker-deployment.md)
- [测试与评测规范](docs/07-test-and-evaluation.md)
- [一周开发计划](docs/08-one-week-development-plan.md)
- [会议全生命周期 Golden Path](docs/14-meeting-lifecycle-golden-path.md)
- [员工管理与站内会议通知](docs/15-employee-and-notification-design.md)
- [资源失效与异常重排](docs/16-exception-replanning-design.md)
- [前端产品化与演示工作区](docs/18-frontend-productization-and-demo-workspace.md)
