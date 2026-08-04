# Codex 项目协作规则

本文件是本仓库的长期执行约束。任何主 Agent 或 subagent 在修改文件前，都必须先阅读本文件、`SPEC.md`、`docs/HANDOFF.md`，再阅读与自己任务相关的专项规范。

## 1. 项目使命与权威来源

这是一个用于简历展示的企业智能会议室调度系统。技术证明优先于功能数量：Java 证明并发预约、事务、一致性、幂等、Outbox 和 RocketMQ；Python 证明 Supervisor + 3 个专业 Agent、Tool Calling、HITL、恢复、OR-Tools、简化 RAG 和评测；前端负责把完整链路可视化；全部组件通过 Docker Compose 部署。

规范发生冲突时，按以下顺序处理：

1. `SPEC.md` 中的冻结决策。
2. `docs/01-functional-spec.md` 中的验收条件。
3. `docs/02-system-architecture.md` 中的边界与一致性策略。
4. `docs/03-java-backend-spec.md`、`docs/04-agent-spec.md`、`docs/05-data-and-api-spec.md`、`docs/06-docker-deployment.md`、`docs/07-test-and-evaluation.md`。
5. `docs/08-one-week-development-plan.md` 中的排期。
6. `docs/HANDOFF.md` 中的当前状态与下一步；它不能覆盖前述冻结规范。

不要在每个新任务中重新讨论已经冻结的需求。若实现确实需要改变 P0 范围，先说明影响并更新 `SPEC.md` 和关联验收条件，再修改代码；不得静默偏离规范。

## 2. 每次任务的启动顺序

1. 读取 `SPEC.md`、`docs/HANDOFF.md` 和本任务涉及的专项规范。
2. 使用 `rg --files`、构建文件、测试结果和 `git status` 核验真实状态，不盲信可能过期的交接描述。
3. 明确本次最小可验收切片、负责目录、依赖接口和验证命令。
4. 存在可独立并行的工作时，按第 5 节划分 subagent；不要让多个 Agent 修改同一文件或同一共享契约。
5. 先实现 Golden Path，再补管理功能和视觉细节。
6. 完成代码后运行与变更相称的测试，并更新 `docs/HANDOFF.md`。

如果 `docs/HANDOFF.md` 与文件系统不一致，以文件系统和可复现命令为准，同时修正交接文档。

## 3. 固定交付范围

P0 必须保留：

- Java 手动会议闭环、30 分钟槽位和相同房间/必需参会者的并发唯一性。
- Redis Lua 预占、MySQL 最终唯一约束、幂等、Transactional Outbox 和 RocketMQ 热门异步预约。
- Supervisor + Requirement/Policy/Scheduling 三个专业 Agent。
- DeepSeek OpenAI-compatible Tool Calling、结构化输出和有限重试。
- OR-Tools Top 3 候选、独立硬约束验证器和可解释无解结果。
- 简化 RAG、可验证引用、HITL、checkpoint 和热门预约结果恢复。
- Vue 聊天、候选确认、会议基础管理和 Agent Trace。
- Docker Compose、Java 并发测试和 Agent 评测。

明确不做：真实或 Mock 邮件、视频会议链接与空调/IoT 外部工具、真实日历/视频供应商、多租户、SSO、多级审批、复杂访客流程、OCR、Rerank、知识图谱、故障注入、完整 OpenTelemetry/Grafana、Kubernetes、服务网格、分库分表和自动移动他人会议。`VIDEO_CONFERENCE` 房间设备特征不属于外部工具，继续保留。

进度不足时，按 `P1 -> 页面美化 -> 非 Golden Path 接口` 的顺序削减，不能削减并发正确性、Multi-Agent、OR-Tools、HITL、恢复和 Docker 部署。

## 4. 不可破坏的架构边界

- 浏览器只访问 Java 公共 API；前端不得直连 Python。
- Java 是鉴权、业务规则、会议状态和预约数据的最终事实源。
- Python 不直连或跨库读写 Java 业务表，只能调用 Java 白名单 Tool API。
- Java 不实现 LLM 路由、Prompt、RAG 或 OR-Tools 求解。
- LLM 负责理解、路由、工具选择和解释；OR-Tools 负责确定性候选优化；Java 在写入前重新校验所有业务约束。
- MySQL 唯一约束是并发最终裁决，Redis 只做预占、限流、缓存和 checkpoint，不能成为预约事实源。
- Agent 发起的预约、改期和取消必须先生成无占用草案，并经过 ACCEPT/EDIT/REJECT 的 HITL；EDIT 后必须重新校验。手动业务入口由用户直接填写并显式提交，仍复用同一 Java 校验/事务服务，但不强制绕行 LangGraph。
- RocketMQ 按至少一次投递设计，消费者通过 `eventId` 和业务终态幂等；不得宣称基础设施 exactly-once。
- 时间统一使用 `Asia/Shanghai`、ISO 8601 带偏移时间和 `[start, end)` 语义；会议固定为 30 分钟槽位。
- `traceId`、`runId`、`toolCallId`、`confirmationToken`、`idempotencyKey`、`requestNo` 和 `eventId` 的语义不得混用。
- Trace 只记录结构化摘要，不记录隐藏推理、密钥、JWT、Service Token 或完整敏感正文。

## 5. 主 Agent 与 subagent 分工

同一任务最多采用 1 个主 Agent + 3 个并行 subagent。主 Agent 负责拆分、接口决策、集成和最终验证，不能只等待子任务结果。

本节的 subagent 指 Codex 开发协作代理，不是产品运行时的 LangGraph Agent。Codex subagent 不得继续派生子代理，除非主 Agent 明确重新分配并发槽位；它们也不得擅自增加产品 Agent 数量。

| 执行角色 | 唯一写入范围 | 主要职责 |
|---|---|---|
| 主 Agent / Coordinator | 根目录文件、`deploy/**`、`scripts/**`、`docs/**`，以及确需统一维护的跨服务契约 | 任务编排、Compose、环境变量、公共契约、集成、Smoke Test、交接和范围控制 |
| Java subagent | `business-service/**` | Spring Boot 业务、Flyway、JWT/RBAC、并发预约、Redis、Outbox、RocketMQ、Tool Gateway、SSE 代理及其测试 |
| Python subagent | `agent-service/**` | FastAPI、LangGraph、DeepSeek Provider、三个专业 Agent、OR-Tools、RAG、HITL/checkpoint、Trace 和评测 |
| Frontend subagent | `frontend/**` | Vue UI、SSE/HITL/Trace 和会议管理 |

协作要求：

- 每个 subagent 开始前读取本文件及相关专项规范，只修改自己的目录。
- 共享 API、事件信封、环境变量、错误码和 SSE 事件由主 Agent 裁决。subagent 可提出变更，但不得单方面改变跨服务语义。
- 多个 Agent 共享工作区时，不回滚、不覆盖、不格式化其他 Agent 的无关改动；发现重叠立即通知主 Agent。
- 不把同一文件同时分配给两个 Agent。根目录 Compose 与模块 Dockerfile 可并行，但接口和环境变量必须由主 Agent统一核对。
- subagent 交付必须列出：修改文件、实现能力、执行过的命令、测试结果、未解决风险和需要主 Agent 集成的事项。
- 主 Agent 必须亲自检查 diff/文件状态，并运行跨模块验证；不能把 subagent 的“已完成”直接当作最终验收。

## 6. 各模块实现约束

### 6.1 Java

- 使用 Java 21、Spring Boot、Spring Security、MyBatis-Plus、Flyway 和 Maven Wrapper。
- 按 `auth/organization/room/meeting/booking/notification/agentgateway/outbox/mq/common` 业务模块组织，不退化为巨型通用 `controller/service/mapper` 目录。
- 手动预约和 Agent 预约复用同一领域/应用服务，不允许维护两套冲突规则。
- 禁止 ORM 自动改表；所有结构和演示数据变更使用版本化 Flyway migration。
- 事务内完成业务记录、槽位、幂等状态和 Outbox；外部 HTTP 调用不得放入数据库事务。
- 唯一键冲突统一映射为稳定业务错误码，禁止泄露 SQL 异常。
- 并发测试必须验证数据库最终状态，不只断言 HTTP 返回值。

### 6.2 Python

- 使用 Python 3.11+、FastAPI、LangGraph、Pydantic、DeepSeek OpenAI-compatible API、OR-Tools 和 Qdrant。
- Agent 数量固定为 Supervisor + Requirement/Policy/Scheduling；Retriever、Solver、HITL Handler 和普通 Tool 是确定性节点，不再包装成 Agent。
- Agent 间通过 Pydantic 结构化 State 交接，不能只依赖自然语言消息。
- 所有模型输出和工具参数必须校验；模型修复重试最多 1 次，网络重试必须有上限。
- DeepSeek 的 Base URL、模型、Key、超时和重试次数只通过环境变量配置，不硬编码密钥或易变模型名。
- OR-Tools 结果必须经过独立硬约束验证器；测试求解器时使用确定性 fixture，不调用 LLM。
- Python 只能访问自己的 `meeting_agent` 数据库、Redis checkpoint 命名空间和 Qdrant collection。
- 未配置 DeepSeek Key 时，`/internal/v1/health` 必须返回 HTTP 200 且响应体 `status=DEGRADED`；只有进程或基础启动依赖异常才使容器健康检查失败。

### 6.3 Frontend

- 前端使用 Vue、TypeScript 和 Vite，只调用 `/api/v1/**`。
- SSE、候选卡片、HITL、会议基础管理和 Trace 是功能优先级；复杂动画和页面美化不是 P0。
- 不展示隐藏推理，只展示 Agent 名称、结构化步骤、工具摘要、引用和业务结果。
- `VIDEO_CONFERENCE` 只作为会议室设备特征参与筛选；系统不创建外部视频会议链接。

### 6.4 Docker

- 基础部署入口为 `compose.yaml`，开发端口覆盖为 `compose.dev.yaml`。
- 所有镜像使用验证过的固定标签或摘要，不使用 `latest`。
- 基础 Compose 只向宿主机发布前端端口；Java、Python、数据库和中间件只在内部网络暴露。
- 健康检查必须在实际固定镜像中验证，不能假设镜像含有 `wget`、`curl` 或 `ps`。
- `.env` 永不提交；`.env.example` 只能包含安全占位值。不得运行会删除命名卷的命令，除非用户明确要求重置数据。

## 7. 跨服务契约规则

- API、表模型、Tool Schema、SSE 事件和 MQ 信封以 `docs/05-data-and-api-spec.md` 为基线。
- 新增或修改跨服务字段时，先更新规范/契约，再同步生产者、消费者、类型定义、测试和示例。
- 公共 API 保持统一成功/错误信封；SSE 和文件响应除外。
- Java 内部 Tool 必须验证 Service Token、AgentContextToken、audience、用户权限、风险等级、参数上限和调用幂等。
- Python 业务结果回调以 `eventId` 幂等，并验证 `runId + requestNo + WAITING_BUSINESS_RESULT`。
- 任何临时 Stub 都必须遵循最终接口形状，并在 `docs/HANDOFF.md` 标记替换条件；不得用与最终契约不同的临时接口打通演示。

## 8. 开发与验证规则

- 每次只推进一个可验收切片。复杂能力先完成最小正确实现，再优化吞吐或体验。
- 不提交只有注释、空实现或恒定成功返回的伪完成代码；必要 Stub 必须显式命名并有测试边界。
- 修改行为时同步增加或更新测试。修复缺陷时优先加入可复现回归测试。
- 初始脚手架必须只选择一套依赖管理方式：Java 使用 Maven Wrapper；Python 使用 `pyproject.toml` + `uv.lock`；前端使用 npm + `package-lock.json`。不要同时引入 Poetry/Pipenv/pnpm/yarn 等第二套锁文件。
- Java 格式检查使用 Maven 插件并绑定到 `verify` 生命周期，确保 `.\mvnw.cmd verify` 同时覆盖编译、测试和格式门槛；不要依赖未记录的全局格式化命令。
- 依赖版本和容器标签在首次跑通后锁定；升级必须有明确原因和验证结果。
- 优先使用仓库脚本或包装器，不依赖开发者全局安装的 Maven。

Day 1 脚手架完成后，仓库应支持下列等价命令；如实际命令变化，必须同步更新本文件和 README：

```powershell
# Java
Push-Location business-service
.\mvnw.cmd verify
Pop-Location

# Python
Push-Location agent-service
uv sync --frozen --group dev
uv run ruff check .
uv run mypy app
uv run pytest
Pop-Location

# Frontend
Push-Location frontend
npm ci
npm run type-check
npm run build
Pop-Location

# Compose（先从 .env.example 创建本地 .env，绝不覆盖已有 .env）
docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml up -d --build
docker compose ps
```

提交或交接前至少执行受影响模块的快速验证；跨服务切片还必须执行 Compose 配置检查和对应 Smoke Test。若环境原因不能执行，明确记录未执行命令、原因和剩余风险，不得写“测试通过”。

## 9. 安全与数据处理

- 真实 API Key、密码、JWT 密钥和 Service Token 只能进入未提交的 `.env` 或安全环境变量。
- 日志、Trace、fixture 和截图只使用虚构员工/会议数据。
- 模型伪造的 userId、角色、runId 或权限信息一律不可信，以 Java 签发令牌和服务端上下文为准。
- 对外错误信息脱敏；内部日志也不打印完整令牌、Prompt 敏感正文或数据库连接串。
- 不执行破坏性数据库/卷清理、历史重写或大范围删除，除非用户明确授权并已核对目标。

## 10. 完成与交接

阶段完成必须同时满足：

1. 代码和配置已落盘，不是只给出建议。
2. 相关测试/构建已执行并记录结果。
3. 跨服务契约与文档一致。
4. 没有真实密钥或默认危险密码进入版本控制。
5. `docs/HANDOFF.md` 已更新：完成项、证据、未完成项、阻塞、下一条具体任务。

只有 `docs/08-one-week-development-plan.md` 的 Definition of Done 全部有可复现证据，项目才可标记完成。
