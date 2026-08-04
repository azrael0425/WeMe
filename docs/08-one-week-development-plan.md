# 08. 一周开发计划

## 1. 执行原则

开发周期：全职7天，使用Codex辅助编码、测试和文档。

原则：

- 每天结束必须存在可运行增量。
- 先实现Golden Path，再补管理功能。
- 每个复杂组件只实现一条主链路。
- 第5天结束前必须完成跨服务集成。
- 第6天后不增加新功能。
- 第7天只修复阻断演示的问题。

## 2. 优先级

### P0：必须完成

- Docker基础设施。
- Java手动预约和并发唯一性。
- Redis预占、幂等、Outbox和热门MQ链路。
- Supervisor + 3个专业Agent。
- Java Tool API。
- OR-Tools调度。
- RAG规则查询。
- HITL接受/编辑/拒绝。
- Agent热门结果恢复。
- 聊天、会议列表、手动预约和Trace页面。
- Java并发测试、Agent基础评测、README。

### P1：时间允许

- 更精细的无解解释。
- 简单日历视图。
- 通知中心。
- Prometheus端点。

### P2：一周内不做

- 真实外部集成。
- 复杂后台页面。
- 自动偏好学习。
- 完整可观测性平台。
- 故障注入。

如果进度落后，按 `P1 -> 页面美化 -> 非Golden Path接口` 顺序删除，不能删除并发正确性、Multi-Agent、OR-Tools和HITL。

## 3. Day 1：骨架、Docker和数据模型

### 上午

- 创建Monorepo目录。
- 初始化Spring Boot、FastAPI和Vue项目。
- 创建Docker Compose基础设施。
- 配置MySQL、Redis、RocketMQ和Qdrant。
- 创建 `.env.example` 和基础健康检查。

### 下午

- Java Flyway迁移：用户、部门、会议室、会议、槽位。
- Python Alembic迁移：Agent Run、Step、Tool和偏好。
- JWT登录和两种角色。
- 演示员工、部门、房间和设备数据。
- 前端完成登录页和API客户端。

### 当日验收

- `docker compose up -d`基础设施健康。
- Java和Python健康接口正常。
- 用户可以登录并查询会议室。

### 当日禁止扩展

- 不做Agent。
- 不做复杂UI。
- 不做MQ业务。

## 4. Day 2：Java预约核心

### 上午

- TimeSlotCalculator。
- 手动创建会议接口。
- 会议、参与者、房间槽位、员工忙碌槽位事务。
- MySQL唯一约束和冲突错误映射。

### 下午

- Redis Lua多键预占和token释放。
- 幂等记录。
- 修改和取消事务。
- 我的会议列表和详情。
- 基础并发集成测试。

### 当日验收

- 90分钟会议写入3个连续槽位。
- 100个并发请求抢同一房间只有一个成功。
- 相同幂等键不会产生重复会议。
- 修改失败时原会议不变。

## 5. Day 3：Outbox、RocketMQ和Tool Gateway

### 上午

- booking_request。
- message_outbox和发布器。
- RocketMQ Topic/Tag。
- 热门请求受理接口。
- MQ最终预约消费者。

### 下午

- BOOKING_RESULT事件。
- Agent结果回调消费者骨架。
- 站内通知事件。
- Java内部Tool API：员工解析、忙闲、会议室、最近会议。
- 草案、确认令牌和Tool审计。
- Java调用Python和SSE代理骨架。

### 当日验收

- 热门预约返回PENDING。
- MQ处理后进入SUCCESS或CONFLICT。
- 重复消息不重复创建会议。
- Tool API无Service Token时被拒绝。

## 6. Day 4：Multi-Agent主图

### 上午

- DeepSeek Provider。
- Pydantic State和Schema。
- Supervisor路由。
- Requirement Agent。
- Java Tool Client。

### 下午

- Policy Agent和Qdrant Retriever。
- Scheduling Agent。
- LangGraph条件边和步骤上限。
- Agent Run/Step/Tool记录。
- Python SSE事件。
- Java到Python的流式转发打通。

### 当日验收

- 普通中文请求被正确路由和结构化。
- Policy问题包含引用。
- Agent能调用Java只读工具。
- 浏览器或API客户端能看到SSE步骤。

## 7. Day 5：OR-Tools、HITL和恢复

### 上午

- 候选集合构建。
- OR-Tools硬约束、软目标和Top 3。
- 硬约束独立验证器。
- 无解基本分类。

### 下午

- create_booking_draft。
- LangGraph interrupt。
- ACCEPT/EDIT/REJECT。
- Redis checkpoint。
- confirm_booking同步与PENDING处理。
- Java BOOKING_RESULT回调Python。
- SUCCESS结束和CONFLICT重新规划。

### 当日验收

- Golden Path通过API完整运行。
- 用户编辑草案后重新校验。
- Python进程重启后可恢复待确认run。
- 热门冲突后Agent返回新候选。

这是项目最高风险的一天。如果当天未完成：

1. 优先保证ACCEPT和PENDING恢复。
2. EDIT可以限制为时间/房间两个字段。

## 8. Day 6：前端和可视化证明

### 上午

- 聊天页面。
- SSE消息流。
- Agent状态和Tool步骤。
- 候选方案卡片。
- HITL确认、编辑、拒绝。

### 下午

- 我的会议列表。
- 手动创建、修改、取消。
- 会议室基础管理。
- Agent Trace时间线。
- 热门PENDING状态刷新。
- 基础错误提示和空状态。

### 当日验收

- 所有Golden Path均可从浏览器操作。
- Trace展示Agent、Tool、RAG和业务结果。
- 页面刷新后能重新加载Run和会议状态。

前端只追求清晰和稳定，不进行复杂动画或视觉设计。

## 9. Day 7：测试、Docker和简历包装

### 上午

- 扩充Java并发测试。
- 30至50条Agent评测集。
- OR-Tools确定性测试。
- Docker从空卷启动Smoke Test。
- 修复阻断Golden Path的问题。

### 下午

- 生成压测结果。
- 生成Agent评测报告。
- 完成README、架构图和时序图。
- 编写演示账号和演示脚本。
- 录制GIF或短视频。
- 编写简历项目描述。
- 锁定依赖和镜像版本。

### 当日验收

- 新环境按README可以启动。
- Golden Path连续演示3次无阻断。
- 并发测试证明零重复预约。
- Agent硬约束违反率为0。
- 仓库不存在API Key和默认危险密码。

## 10. 每日质量门槛

每天提交前运行：

```text
Java: compile + unit tests + formatting
Python: lint + type check + pytest
Frontend: type check + build
Docker: docker compose config --quiet
```

关键接口实现后立即补测试，不把所有测试推迟到第7天。

## 11. 风险与应对

| 风险 | 概率 | 影响 | 应对 |
|---|---:|---:|---|
| RocketMQ本地配置耗时 | 中 | 高 | Day 1验证基础容器，Day 3前不推迟 |
| DeepSeek Tool输出不稳定 | 中 | 高 | Pydantic校验、temperature 0、有限重试和fixture模型 |
| HITL恢复实现超时 | 中 | 高 | 只做一类BookingDraft和Redis checkpoint |
| Java-Python-SSE链路复杂 | 中 | 高 | Day 3做空流代理，Day 4再接Agent |
| OR-Tools建模过度 | 中 | 中 | 只使用枚举候选 + 单选模型，不做全局会议重排 |
| Embedding模型首次下载慢 | 中 | 中 | 使用命名卷缓存，提前在Day 1验证 |
| 前端占用过多时间 | 高 | 中 | 只保留5个页面，复用组件库 |
| 文档和测试被挤压 | 高 | 高 | 规范已先行，每天同步更新，不留到最后从零编写 |

## 12. Codex使用建议

适合交给Codex加速：

- 根据规范生成模块骨架和DTO。
- Flyway/Alembic迁移初稿。
- Controller、Mapper、Pydantic Schema和测试fixture。
- Redis Lua脚本及其测试。
- LangGraph节点样板和Tool适配器。
- Vue页面和TypeScript类型。
- Dockerfile、Compose和README校验。
- 生成测试用例、并发测试和评测数据初稿。

必须由开发者重点复核：

- 数据库事务边界。
- 幂等与唯一约束。
- RocketMQ消费状态机。
- Agent恢复和重复回调。
- Tool权限与令牌传播。
- OR-Tools硬约束。
- Docker健康检查是否与固定镜像匹配。

## 13. Definition of Done

项目只有同时满足以下条件才算完成：

- [ ] 一条Compose命令可启动完整系统。
- [ ] 手动会议功能在Agent不可用时仍正常。
- [ ] Supervisor + 3个专业Agent真实参与工作流。
- [ ] OR-Tools负责最终候选调度。
- [ ] RAG回答包含可验证引用。
- [ ] 所有写操作都经过用户显式确认：Agent写操作通过ACCEPT/EDIT/REJECT HITL，手动页面通过最终提交确认并复用同一Java校验/事务服务。
- [ ] 同步预约并发唯一性测试通过。
- [ ] 热门异步预约最终进入终态。
- [ ] PENDING结果能恢复Agent。
- [ ] Trace可展示Agent和Tool步骤。
- [ ] Agent评测与压测报告已提交。
- [ ] README包含启动、演示、测试和限制。
- [ ] 仓库不包含真实密钥。
