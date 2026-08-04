# 系统Spec总入口

## 1. 基线

- Spec版本：1.2。
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

## 3. P0交付

- Java高并发预约与并发正确性测试。
- Redis Lua、MySQL唯一槽位、幂等、Outbox和RocketMQ。
- Supervisor + 3个专业Agent。
- DeepSeek Tool Calling。
- OR-Tools Top 3方案与硬约束验证。
- 简化RAG与有效引用。
- 持久化HITL和异步结果恢复。
- Vue聊天、会议基础管理和Agent Trace。
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
