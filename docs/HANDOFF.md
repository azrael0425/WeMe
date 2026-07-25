# 项目开发交接

## 1. 交接元信息

- 最后更新时间：2026-08-11（Asia/Shanghai）。
- 当前里程碑：Day 4——结构化 Multi-Agent、Java SSE 代理、只读 Tool、Qdrant 政策引用和 Trace 元数据，**PASS**。
- Day 1 / Day 2 回归：**PASS**。
- Spec 基线：1.0；没有修改冻结架构决策或扩大 P0 范围。
- Git 状态：仓库已初始化在 `main`，尚无提交；全部项目文件仍为未提交工作区内容。
- 运行状态：完整 Day 4 开发 Compose 正在运行；所有常驻服务 healthy，一次性 RocketMQ 初始化服务均为预期的 `Exited (0)`。
- 维护责任：本文件只由主 Agent / Coordinator 更新。

本文件记录真实可复现状态，不替代 `SPEC.md` 和专项规范。

## 2. 当前状态总览

| 工作流 | 状态 | 证据/说明 |
|---|---|---|
| Day 1 骨架、双库和登录/会议室 | DONE | Java/Python/Frontend/Mock、MySQL 双库、Redis、RocketMQ、Qdrant、Nginx 和公共登录链路回归通过 |
| Day 2 手动预约与并发正确性 | DONE | 30 分钟槽位、MySQL 最终唯一约束、Redis Lua、幂等、CRUD、修改回滚及两组真实 100 并发回归通过 |
| Day 3 Flyway 与草案 | DONE | V4 创建六张 Day 3 表；CREATE/RESCHEDULE/CANCEL 草案确认前无会议或槽位副作用 |
| Day 3 HOT 受理与 Outbox | DONE | HOT 确认原子写 `booking_request(PENDING)`、`BOOKING_COMMAND` Outbox、幂等结果并消费确认令牌；HTTP 202 返回 PENDING |
| Day 3 RocketMQ 最终处理 | DONE | 真实 Broker 上 SUCCESS/CONFLICT、BOOKING_RESULT、通知、消费幂等和重复消息不重复创建均通过 |
| Day 3 Tool Gateway | DONE | Service Token、AgentContextToken、参数上限、员工/忙闲/房间/最近会议、草案、审计和重放均通过 |
| Day 3 SSE/回调边界 | DONE | Java SSE 代理不伪造输出；Python 端点不存在时完整返回 503 `AGENT_UNAVAILABLE`；BOOKING_RESULT 回调骨架默认关闭 |
| Day 4 Multi-Agent Golden Path 与增强切片 | DONE | 固定为 Supervisor + Requirement/Policy/Scheduling 四个 Agent；确定性 fixture 经 Java SSE 代理完成普通调度与政策引用 Smoke，Run/Step/Tool Call 已持久化 |

## 3. Day 3 完成内容与责任分工

### 3.1 主 Agent / Coordinator

- 完整重读并核验 `AGENTS.md`、`SPEC.md`、`README.md`、`docs/HANDOFF.md` 和 `docs/01` 至 `docs/08`；以文件、测试、数据库、容器和 `git status` 为准核对状态。
- 冻结 Day 3 内部安全、Tool、草案、HOT、Outbox、事件和 SSE 最小契约，并同步 `docs/03-java-backend-spec.md`、`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`。
- 更新 `.env.example`、`compose.yaml`、`compose.dev.yaml` 和 `README.md`，加入 Day 3 配置、固定 Topic/Consumer Group、一次性 `rocketmq-topic-init` 和 Day 3 镜像标签。
- 新增 `scripts/smoke-day3.py`，覆盖内部鉴权、查询 Tool、草案、HOT PENDING、MQ SUCCESS/CONFLICT、Tool 重放、改期/取消确认和 SSE 不可用边界。
- 新增并修正 `scripts/replay-day3-booking-command.ps1`，发送语义完全相同且 JSON 完整的重复 `BOOKING_COMMAND`，同时核对数据库终态和 MQ 总积压。
- 亲自审查 Java 事务、Outbox 租约、失败审计、通知接收人、MQ 终态幂等和 SSE 实现；问题优先交回原 Java subagent 修复。
- 亲自执行固定 Java 21 verify、四个应用镜像构建、真实 MySQL V4、完整 Compose、Day 1/2 回归、两组 100 并发、Day 3 Smoke、MQ 重放和静态安全/范围扫描。
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

### 3.3 Python、Frontend 与 Mock

- Day 3 没有修改 `agent-service/**`、`frontend/**` 或 `mock-services/**`，没有启动对应开发 subagent；这些模块保持此前已验收能力。
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
| Video Mock | `meeting-scheduler-video-provider-mock:day3` | healthy；仅 `/health` 骨架 |

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
3. 当前 Day 3 完整 Compose 正在运行，应用镜像为 `:day3`，MySQL、Redis、RocketMQ、Qdrant、Java、Python、Frontend/Nginx 和 Video Mock 均为 healthy。不要删除命名卷；若状态已经变化，应重新验证并如实更新本文件。
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

按 AGENTS.md 的目录所有权使用 1 个主 Agent 和最多 3 个不再派生的内部 subagent并行开发：主 Agent 只负责根目录、deploy/**、scripts/**、docs/**、契约裁决、Compose 集成、Smoke 和 HANDOFF；Java subagent 只编辑 business-service/**；Python subagent 只编辑 agent-service/**；Frontend/Mock subagent 只编辑 frontend/** 和 mock-services/**。没有实际工作时不要为凑数修改模块；只有主 Agent 可以编辑 docs/HANDOFF.md。主 Agent必须亲自审查并做跨服务验证。

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
- **Frontend/Mock：** Day 4 没有前端或 Mock 产品代码变更（按目录所有权不凑数修改）；仍实际运行前端 type-check 和生产 build 回归。

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
| `$env:AGENT_MODEL_PROVIDER='fixture'; docker compose -f compose.yaml -f compose.dev.yaml up -d --build --wait` | PASS | 四个应用镜像构建为 `:day4`，完整组合成功启动；未删除数据库或命名卷。 |
| `python scripts/smoke-day4.py` | PASS | 普通中文请求依次收到 `run.started`、Supervisor/Requirement/Scheduling step、`resolve_employees` READ Tool、`run.completed`；Trace 有 4 Step 和 1 Tool Call。政策请求经实际 Qdrant 返回 1 个含 `chunkId/title/headingPath` 的引用。 |
| `/internal/v1/health` 与无 Key 回归 | PASS | 当前组合健康接口为 HTTP 200；`test_health.py` 覆盖未配置 DeepSeek Key 时 HTTP 200 / `status=DEGRADED`。 |

### 13.5 当前服务状态与已处理问题

- 最终组合 Compose 中 MySQL、Redis、RocketMQ NameServer/Broker、Qdrant、Java、Python、Frontend/Nginx 和 Video Mock 均为 `healthy`；`rocketmq-store-init`、`rocketmq-topic-init` 为预期 `Exited (0)`。
- 第一次真实 Smoke 暴露 JDK `HttpClient` 对明文上游发 h2c upgrade，而 Uvicorn 拒绝该升级；Java SSE client 已明确固定 HTTP/1.1，并有 `Upgrade` 头不存在的回归断言。
- 第二次真实 Smoke 暴露已提交 SSE 的异步续接被 Spring Security 再次拒绝；仅对 `ASYNC/ERROR` dispatcher 放行，原始 `REQUEST` 继续要求 EMPLOYEE/ADMIN。最终 Smoke 已验证完整终端事件。

### 13.6 未完成项和 Day 5 唯一明确起点

- 未完成项均属于后续 Day 5/Day 6 范围：OR-Tools Top 3、独立硬约束验证器、无解分类、`create_booking_draft`、LangGraph interrupt、ACCEPT/EDIT/REJECT、Redis checkpoint、确认与业务结果恢复，以及前端聊天/HITL/Trace 可视化。
- **Day 5 唯一明确起点：** 从 `agent-service/**` 的候选集合构建、OR-Tools 硬约束/软目标 Top 3 与独立硬约束验证器开始；先完成这三个确定性能力的测试，再进入 Day 5 下午的 draft/HITL/checkpoint 工作。
