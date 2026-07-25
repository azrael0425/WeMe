# 系统Spec总入口

## 1. 基线

- Spec版本：1.0。
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
14. 外部工具只保留Mock视频会议链接；不做邮箱和空调。
15. 不做故障注入、完整OpenTelemetry平台、多租户、SSO和复杂审批。

## 3. P0交付

- Java高并发预约与并发正确性测试。
- Redis Lua、MySQL唯一槽位、幂等、Outbox和RocketMQ。
- Supervisor + 3个专业Agent。
- DeepSeek Tool Calling。
- OR-Tools Top 3方案与硬约束验证。
- 简化RAG与有效引用。
- 持久化HITL和异步结果恢复。
- Vue聊天、会议基础管理和Agent Trace。
- Mock视频会议工具。
- Docker Compose一键部署。
- Agent评测和Java压测报告。

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
