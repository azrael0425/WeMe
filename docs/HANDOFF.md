# 项目开发交接

## 1. 当前状态

- 最后更新：2026-08-15（Asia/Shanghai）。
- 冻结基线：`SPEC.md` 1.3；浏览器只访问 Java，Java 是业务事实源，Python 负责固定的 Supervisor + Requirement/Policy/Scheduling、OR-Tools、RAG、HITL 和恢复。
- 当前分支：`main`；本轮清理前基线提交为 `d82f7b7 fix(frontend): constrain recent tasks to sidebar top`。
- 当前修改：项目清理与侧栏调整已提交为 `f29b21e`；工作区未提交内容包含参会人姓名输入、账号隔离会话恢复、待确认草案恢复、RAG/Agent 首问性能修复，以及第 11 节的双场景演示准备。演示准备不改变服务边界或拓扑。
- 运行环境：基础 Compose 的 Java、Python、前端、MySQL、Redis、Qdrant 和 RocketMQ 长驻服务当前均为 `healthy`；仅前端发布到宿主机 `http://localhost`。未删除数据库或命名卷，浏览器已停在张三账号的空白“新建编排”页面。

## 2. 已交付能力

| 模块 | 当前可用能力 |
|---|---|
| Java | JWT/RBAC、手动会议闭环、30 分钟槽位、Redis Lua 预占、MySQL 最终唯一约束、幂等、Transactional Outbox、RocketMQ HOT 预约、Tool Gateway、SSE 代理、通知、异常重排、会前会后和知识库公共 API |
| Python | 受控 Agent Loop、DeepSeek 原生 Tool Calling、Source Fidelity、OR-Tools Top 3 + 独立硬约束验证、BGE-M3 + Qdrant RAG、HITL、Redis checkpoint、HOT 冲突恢复、会后结构化草案和分层评测 |
| Frontend | Vue 企业工作台、多轮聊天/SSE、Top 3、ACCEPT/EDIT/REJECT、安全 Trace、会议/会议室/员工/消息管理、异常重排、会前会后、知识库和对话线程恢复 |
| 部署 | 固定镜像基线、基础/开发 Compose、Flyway/Alembic、RAG 一次性入库、健康检查、Smoke/并发/真实模型评测入口 |

## 3. 最新可复现证据

- Java：最新后端变更的 JDK 21 Maven `verify` 为 **87 tests，0 failure/error/skip**，Spotless/Jar PASS。
- Python：最新 Agent 回归为 Ruff、Mypy PASS，Pytest **158 passed**（仅既有 LangGraph pending-deprecation warning）。
- Frontend：最新 `npm run type-check` 与 `npm run build` PASS。
- 真实模型：`artifacts/live-eval/component-full-final-20260815.json` 的 40 条完整门禁为 Route/Intent/Tool/Native Tool/Citation 100%、Constraint F1 95.31%、Source Fidelity Violation 0。
- 公开 API 多轮对抗：`artifacts/product-scenario-evaluation.json` 为 **16/16 PASS**；基线报告为 `artifacts/product-scenario-evaluation-baseline.json`。
- 演示验收入口：`docs/21-demo-acceptance-runbook.md`；本轮双场景逐步手册：`docs/22-two-scenario-demo-runbook.md`；最终报告入口：`docs/REPORTS.md`。

## 4. 2026-08-15 项目清理

### 已删除

- 可重建本地依赖与构建产物：`agent-service/.venv/`、`business-service/target/`、`frontend/node_modules/`、`frontend/dist/`。
- Python/工具缓存：`.mypy_cache/`、`.pytest_cache/`、`.ruff_cache/` 及全部 `__pycache__/`。
- 本地运行日志：`artifacts/**/*.log`；受版本控制的脱敏 JSON 评测证据未删除。
- 空残余目录：`agent-service/build/`、`frontend/src/demo/`、`mock-services/`。
- 已执行完且不再是项目规范的跨对话提示词：`docs/10-frontend-redesign-execution-prompt.md`、`docs/13-live-model-agent-repair-execution-prompt.md`。
- 部署文档中与 Spec 1.3 冲突的 Mock 服务段落，以及 `.gitignore` 中的空 Mock 注释。
- 约回收 **1.59 GiB** 可重建内容；清理后 Git 忽略项预览只剩必须保留的根目录 `.env`。

### 防止再生

- `.gitignore` 新增 `agent-service/build/` 和 `agent-service/*.egg-info/`，保留既有 Maven、uv、pytest/mypy/ruff、Vite 和 npm 生成物规则。
- `docs/09-frontend-product-redesign.md` 与 `docs/12-live-model-agent-repair-plan.md` 已标记为“已实施的历史设计记录”，不再被解读为待执行任务。

## 5. 保留项与位置（不可当作残余清理）

| 位置 | 保留原因 |
|---|---|
| `.env` | 本地秘密与运行配置；已被 Git 忽略，不得覆盖或回显。 |
| `deploy/rag-documents/` | `rag-init` 必需的 22 份版本化会议制度源文档，不是测试输出。 |
| `artifacts/**/*.json` | 已脱敏、被 README/演示手册引用的 fixture、真实模型和对抗评测证据。 |
| `docs/00-09`、`docs/11-22`、`SPEC.md` | 冻结规范、架构/设计依据、验收标准与演示手册；`docs/09` 仅作历史 UI 设计基线，`docs/12` 仅作 Agent 设计依据。 |
| `uv.lock`、`package-lock.json`、Maven Wrapper/构建文件 | 可复现依赖与验收的必需输入，不是已安装依赖。 |
| `.git/` | 项目历史和回滚边界；本轮不改写历史。 |
| Docker 卷 `meeting-scheduler_mysql_data`、`meeting-scheduler_redis_data`、`meeting-scheduler_qdrant_data`、`meeting-scheduler_rocketmq_broker_store` | 业务事实、checkpoint、索引和 MQ 审计数据；项目规则禁止在未明确重置数据时删卷。 |
| Docker 卷 `meeting-scheduler_agent_model_cache` | 当前 Compose 已不引用，属于疑似历史残留；但命名卷删除需要用户另行明确授权“重置/删除卷”，因此本轮只标记、不删除。 |
| `D:/rag001/bge-m3` | 项目外部的本地 Embedding 模型，由 Compose 只读挂载；不在本工作区清理范围内。 |

## 6. 本轮验证

- `docker compose config --quiet`：PASS。
- `docker compose -f compose.yaml -f compose.dev.yaml config --quiet`：PASS。
- `docker compose -f compose.yaml -f compose.dev.yaml ps`：8 个长驻服务均 `healthy`。
- 清理后 `git clean -nd` 无未跟踪残留；`git clean -ndX` 只列出受保护的 `.env`。
- 本轮后续功能修复已重建依赖并运行完整模块门禁；详细证据见第 10 节。

## 7. 2026-08-15 前端侧栏会话区顺序调整

- “新建编排”“搜索会话”“最近任务”已从品牌区下方的固定三分之一高度容器移入统一侧栏导航，排列在工作台、协作、管理和当前运行记录之后。
- 会话区与主要功能共享同一滚动区域；新建和搜索采用与导航一致的轻量行样式，最近任务继续按真实 `threadId` 聚合，恢复、搜索、折叠和移动端抽屉语义不变。
- 验证：`npm ci`、`npm run type-check`、`npm run build`、定向 `git diff --check` 均 PASS；本地开发页在 1440×900 与 390×844 下完成登录、真实只读制度问答、最近任务展示和移动导航检查，应用控制台无 error/warning。
- 环境提示：`npm ci` 仍报告既有传递依赖要求 Node `^24.15.0`，当前为 `24.14.0`；安装、类型检查与构建均成功。

## 8. 2026-08-15 参会人姓名输入与 ID 解析

- 已移除 `EmployeeMultiSelect`（原生多选/复选框人员选择器）。手动创建、编辑会议及从会议室空闲槽位创建会议共用 `ParticipantNameInput`：用户输入姓名，多个姓名可用逗号、顿号、分号或换行分隔，再添加为对应角色的参会人。
- 前端只以安全在职员工目录精确解析姓名，并将结果写入既有 `requiredParticipantIds` 或 `optionalParticipantIds` 后提交；浏览器不展示也不要求输入原始 ID。重名时提示输入“姓名（部门）”，未匹配或无法唯一解析时不会更新人员列表。
- 同一员工仍只能属于一个参会角色：另一类别已添加该员工时，姓名输入会给出即时提示，符合 `meeting_participant` 的 `(meeting_id, employee_id)` 唯一约束和 Java 的 `PARTICIPANT_TYPE_OVERLAP` 校验。
- 验证：`npm run type-check`、`npm run build`、定向 `git diff --check` PASS；本地创建会议抽屉实际输入“李四，王五”及“孙琪，孟欣”并各解析为 2 人，交叉输入“李四”得到角色冲突提示，应用控制台无 error/warning，未提交创建请求。

## 9. 恢复本地开发环境

```powershell
Push-Location agent-service
uv sync --frozen --group dev
Pop-Location

Push-Location frontend
npm ci
Pop-Location

Push-Location business-service
.\mvnw.cmd verify
Pop-Location
```

Maven 首次验证会重建 `business-service/target/`，uv 会重建 `agent-service/.venv/`，npm 会重建 `frontend/node_modules/`；这些目录都已正确忽略。

## 10. 2026-08-15 会话恢复、待确认草案与首问性能修复

- Python 新增 `agent_message` 迁移和用户可见消息持久化。会话列表、会话详情严格按当前 `userId` 过滤，即使当前账号是 ADMIN 也不会跨账号读取；历史接口不返回 `confirmationToken`、JWT、Service Token 或隐藏推理。既有会话在没有消息明细时继续以脱敏 run 摘要兼容展示。
- Java 新增 `GET /api/v1/agent/threads` 与 `GET /api/v1/agent/threads/{threadId}` 公共代理，继续由 Java JWT/RBAC 和 AgentContextToken 保护，并统一返回 `Cache-Control: no-store`。Python 404 映射为稳定错误码 `AGENT_THREAD_NOT_FOUND`。
- 前端侧栏和聊天页改为从服务端恢复当前账号历史；退出、401 和重新登录统一清理 `weme.chat-*` 瞬态缓存。待确认页会列出当前账号全部 `WAITING_CONFIRMATION` 草案，逐项恢复令牌；过期草案保留可见并提供返回原会话重新生成入口。
- RAG 首问增加真实 BGE-M3 启动预热、绕过 query cache 的周期保温、进程内 TTL 向量缓存、Qdrant collection 校验缓存、默认 8 秒 embedding 预算和最多 200 个真实 Qdrant payload 的词法降级；降级结果仍使用真实 chunk 引用。日志只记录 query hash 与 `embeddingMs/vectorSearchMs/totalMs/cacheHit/fallback`。
- DeepSeek 客户端改为进程内连接复用，默认单次超时从 45 秒降为 20 秒、重试从 2 次降为 1 次；Scheduling Tool Loop 从 6 轮收敛为最多 4 轮，确定性事实齐备后立即进入求解。
- 验证：`uv run ruff check .` PASS；`uv run mypy app` PASS；`uv run pytest` 为 **157 passed**；JDK 21 Docker build 中 `mvn verify` 为 **86 tests** 且 Spotless/Jar PASS；`npm run type-check`、`npm run build` PASS；`docker compose config --quiet` 与 `git diff --check` PASS。
- 第 10 节交付当时尚未替换长驻 Compose；本轮已完成 Java、Agent 与前端镜像重建并替换为健康容器，也已使用真实 DeepSeek/BGE-M3 执行第 11 节演示原文。独立的冷态、保温态、缓存命中和 embedding 超时性能基准仍可按需补测。

## 11. 2026-08-15 双场景演示准备与真实演练

- Java Flyway V11 新增两场永久演示基线：李四在 2026-08-26 13:00–14:00 与 14:00–15:00 连续忙碌。会议号固定为 `MTG-DEMO-LISI-20260826-1300/1400`，清理脚本明确排除这两场会议。
- Agent 修复了演示原文中的四个确定性边界：VIP“能直接使用吗”进入 Policy/RAG；带空格的中文绝对日期正确解析；“我，李四……”识别当前登录用户；“没有设备要求”闭合可选要求；续轮“其他不变”不会再把 09:00–12:00 搜索窗口误当成 180 分钟会议。
- 异常重排页进入智能编排时，预填文本以用户计划的短句开头，并自动保留异常重排单号、会议 ID、失效房间和处置约束，确保目标唯一且生成 `RESCHEDULE` HITL 草案。
- 新增 `scripts/demo-two-scenarios.py`：`status` 只读核验永久基线和残留；`cleanup` 默认 dry-run；`cleanup --apply` 通过 Java 公共 API 取消场景 1/2 会议并恢复失效房间，能追踪 OPEN/RESOLVED/RESTORED/CANCELLED 异常单和已取消会议。它不物理删除审计记录、不访问数据库、不删除卷。
- 真实 DeepSeek + BGE-M3 + Java 公共 API 演练已覆盖：VIP 引用问答；三轮需求补全；管理员并发占位后的 Java 最终冲突与 `REPLAN`；李四两段 `REQUIRED_AVAILABILITY` 无解；60 分钟续轮；房间停用后会议不移动、唯一 OPEN 单和仅发起人通知；异常页预填后的 RESCHEDULE 接受。
- 页面级核验确认无解卡片展示请求窗口、60 分钟、会议 198/199、李四冲突区间和两条建议；异常单页展示变化/保留约束；浏览器控制台无 error/warning。
- 演练结束后，场景 1 占位会议和场景 2 会议均已正常取消，失效房间已恢复，李四两场永久冲突会议仍为 `CONFIRMED`；`python scripts/demo-two-scenarios.py status` 返回 `ready: true`。
- 验证：Java JDK 21 Docker build 中 Maven `verify` 为 **87 tests**；Python Ruff/Mypy PASS、Pytest **158 passed**；前端 Docker production build（含 type-check）PASS。完整操作步骤见 `docs/22-two-scenario-demo-runbook.md`。

## 12. 2026-08-15 演示中当前用户与部门名单去重修复

- 演示输入“安排在下周三下午，由我和我们组内的人参加”同时产生第一人称参会标记和 `MY_DEPARTMENT` 人员范围；Java 目录返回的部门成员已经包含当前登录的张三，旧实现却又在需求摘要前追加“当前登录用户（我）”，因此把同一员工显示成两人。
- Requirement 确定性合并现在使用 AgentContext 的当前用户 ID 与目录成员 ID 比对；目录已经包含当前用户时，移除冗余的第一人称标记，但保留完整部门名单、容量和必需参会者约束。“组内的人”也加入确定性范围表达和 fixture 覆盖。
- 新增完整内部流回归，直接覆盖上述演示原文，断言需求摘要为“4人：张三、李四、王五、赵六”，且不再出现额外的“当前登录用户（我）”。普通“我和李四”路径仍保留第一人称身份映射。
- 验证：针对性回归 PASS；`uv run ruff check .` PASS；`uv run mypy app` PASS；`uv run pytest` 为 **159 passed**（仅既有 LangGraph pending-deprecation warning）。Agent 镜像已重建并替换，容器恢复 `healthy`。
- 已执行 `python scripts/demo-two-scenarios.py cleanup --apply`；没有待取消的演示会议或待恢复房间。随后 `status` 返回 `ready: true`、`residualDemoMeetings: []`、`inactiveDemoRooms: []`，李四两场永久冲突会议继续保持 `CONFIRMED`。

## 13. 2026-08-15 演示占位遗漏与 Top 3 房间多样性修复

- 现场发现管理员手动会议 `204`（标题“演示并发占用”）仍在 2026-08-19 12:30–16:30 占用研发楼评审室。旧清理规则只识别包含 `WeMe1.1` 或以“演示并发占位”开头的标题，因而错误报告 `ready: true`；研发楼评审室在 12:00–18:00 内只剩 30 分钟和 90 分钟两段，无法承载 120 分钟草案，求解器正确将其排除。
- 清理识别已覆盖现场标题“演示并发占用”，并增加脚本回归；会议 `204` 已通过 Java 公共取消接口处理。实时房间可用性复核显示研发楼评审室在 12:00–18:00 的 12 个槽位全部可用，演示状态再次为 `ready: true`。
- OR-Tools Top 3 从“只排除已选房间+开始时间组合”升级为房间多样性优先：可行房间不少于3间时先返回3间不同会议室各自的最优时间，会议室不足时才使用同一房间的其他开始时间补齐；候选仍保持总成本升序、确定性 tie-break 和独立硬约束复核。
- 验证：求解器定向测试 **12 passed**，清理识别脚本测试 **2 passed**；Ruff、Mypy、脚本编译 PASS；Python 全量 Pytest **160 passed**（仅既有 LangGraph pending-deprecation warning）。Agent 镜像已重建，全部长驻服务健康。

## 14. 2026-08-15 过期草案“返回原会话重新生成”修复

- 根因是审批卡片原先只有一个普通 `/chat?runId=...` 路由链接：它能恢复旧会话，却没有创建新 Run 或向 Agent 提交任何输入，所以页面不会解释草案过期，也不会重新规划。
- 过期卡片现在显式携带 `regenerate=expired`。聊天页恢复并核验旧 Run 后，在原 thread 自动提交一条可见的重新生成请求；新 Run 只继承最后有效 Requirement 基线，重新读取人员、会议室和忙闲事实。旧候选、旧草案、旧确认令牌、工具预算、写入状态和幂等指纹均不继承。
- Python 只允许同用户、同 thread 的 `FAILED` 或确认令牌已经过期的 `WAITING_CONFIRMATION` Run 作为 `baseRunId`；未过期待确认、成功、拒绝和其他状态继续返回 409 `REQUIREMENT_BASELINE_NOT_RECOVERABLE`。前端在基线不可恢复或请求失败时展示可操作原因，并把重试文本留在输入框，不再无声跳转。
- 回归验证：过期待确认可以创建新 Run 和新 token，且未调用 WRITE Tool；未过期待确认与其他不可恢复状态仍被拒绝。Ruff、Mypy PASS，Python 全量 Pytest **162 passed**（仅既有 LangGraph pending-deprecation warning），前端 type-check/production build、Compose config 与 `git diff --check` PASS。
- Compose 已重建并替换 Agent、前端及依赖重建触发的 Java 容器，8 个长驻服务全部 `healthy`。真实页面从旧 Run `run_4cab4ee917464aad8bb584510eaffaef` 点击入口后创建新 Run `run_14a7c343c1384422b207725ca0c24be7`，页面立即显示“原确认草案已过期”的用户动作，随后生成研发楼评审室、研发楼 301、总部楼贵宾厅三个不同房间候选和新待确认草案；验证后已拒绝该草案，未创建正式会议。
- 清理验证：本轮录屏创建的正式会议 `208` 已通过 Java 公共取消接口处理；`python scripts/demo-two-scenarios.py status` 返回 `ready: true`、无残留演示会议、无失效房间，李四会议 `198/199` 继续保持 `CONFIRMED`。浏览器已停在张三账号的空白“新建编排”页面，可重新开始录屏。

## 15. 2026-08-15 最终并发冲突后的候选恢复修复

- 现场 Run `run_300d19e931d24cf2b85b7a647a3c52b3` 中，Java 最终确认已经正确返回 `BOOKING_CONFLICT`，确定性 `conflict_repair` 也已进入 `REPLAN`；但前序多轮需求补全恰好用完 9 次模型调用，Scheduling 重入先预留一次模型调用而触发 `BUDGET_EXHAUSTED`，所以页面没有收到新候选。旧 Run 已是 `FAILED`，不能原地恢复，需用修复后的新 Run 重新演示。
- 冲突修复现在跳过 Scheduling LLM Tool Loop，使用已校验的 Requirement 和人员身份确定性重新读取 Java 最新目标会议（改期场景）、忙闲和房间事实，再经过 OR-Tools 与独立硬约束验证器生成新草案。模型调用数保持不变，创建与改期冲突均不会再被模型预算覆盖。
- 新 `hitl.required` 增加用户可见 `answerSummary` 和布尔字段 `conflictRepair`。创建冲突后的固定提示为“当前会议室已被占用，请切换其他的编排选项。已重新读取最新占用情况并生成其他可用方案。”；前端收到 `conflictRepair=true` 后直接打开“候选”标签，保留最多 3 个方案及“使用此方案重新校验”操作。
- 回归用例把 `AGENT_MAX_MODEL_CALLS` 动态压到初始 Run 已用调用数，断言冲突后仍以新 `hitl.required` 结束、模型调用数不增长、忙闲/房间事实各刷新一次且草案切换房间；既有改期写冲突回归也继续通过。
- 验证：Ruff PASS；Mypy 对 44 个源文件 PASS；Python 全量 Pytest **162 passed**（仅既有 LangGraph pending-deprecation warning）；前端 `npm run type-check` 与 production build PASS；基础和开发 Compose 配置校验 PASS。Agent/前端镜像已重建，最终使用基础 Compose 启动，8 个长驻服务全部 `healthy`。
- 清理验证：`python scripts/demo-two-scenarios.py cleanup --apply` 无待处理演示会议或房间；随后 `status` 返回 `ready: true`、`residualDemoMeetings: []`、`inactiveDemoRooms: []`，李四永久冲突会议 `198/199` 保持 `CONFIRMED`。张三页面已重置为 `http://localhost/chat` 的空白“新建编排”。

## 16. 2026-08-15 双场景最终录屏闭环与房间失效约束修复

- Java 的近期会议事实原先只从启用会议室列表构造 `roomFeaturesByMeetingId`。房间一旦停用，原房间设备快照会丢失，异常重排因此可能把有大屏/白板的原房间换成设备降级房间。`get_recent_meeting` 现在从全部会议室读取受信任快照，并增加“停用后仍返回原设备能力”的集成回归。
- Flyway V12 增加专用于重复演示的“研发楼小组讨论室”（`RD-TEAM-202`，容量 4，无设备标签）。场景 2 的四人无设备请求会稳定把它排在首位；该房间没有其他未来基线会议，因此停用只为本次会议创建一张异常单，不再波及无关演示数据。
- Run 恢复接口现在对待确认状态同样返回 `requirementItems` 与引用。前端在主对话直接展示政策引用，在编排详情展示时间、时长、人员和设备的结构化事实；无解卡展示内部类别码；失败 Tool 详情从安全参数展示真实 `BOOKING_CONFLICT`，不再退化为通用 `TOOL_CALL_FAILED`。
- 真实页面第二轮完整演练通过：VIP 回答显示《VIP 与高管会议室使用规则》引用；场景 1 Run `run_a29cfd211b384edc9296790acaea7801` 正确解析 5 人、120 分钟、大屏/白板，管理员会议 `215` 制造冲突后显示固定占用提示、三个新候选、`confirm_booking` 失败、`BOOKING_CONFLICT`、事实重读和 `REPLAN`；新草案已拒绝。
- 场景 2 Run `run_867fca76ca0d4bd78a213791128ca764` 显示 `REQUIRED_AVAILABILITY`、李四会议 `198/199` 的两段冲突和两条有限建议；改到 27 日后仍为 60 分钟、四人，创建会议 `217` 于“研发楼小组讨论室”09:00–10:00。停用房间时会议保持 `CONFIRMED` 且未移动，数据库只产生一张异常单，`RESOURCE_UNAVAILABLE` 只发送给 `zhangsan` 一次。
- 异常重排 Run `run_0e0898bd957d4e55b1c652069ffeeb88` 排除失效房间并生成三个新房间候选；接受后会议改到“研发楼评审室”，异常单为 `RESOLVED / AGENT_RESCHEDULE`。最终页面“变化约束”只有会议室，“保留约束”包含时间、时长、参会人和原设备能力。
- 验证：JDK 21 Docker Maven `verify` **88 tests**、Spotless/Jar PASS；Python Ruff、Mypy PASS，Pytest **162 passed**；前端 type-check 与 production build PASS；脚本单元测试 **2 passed**；Compose config PASS。基础 Compose 的 8 个长驻服务健康；开发覆盖文件因当前 Windows 保留端口范围包含 6379 未用于最终启动。
- 演练后已通过 Java 公共 API 取消会议 `215/217`、恢复房间 `128` 并清除演示未读角标；永久会议 `198/199` 保留。最终 `python scripts/demo-two-scenarios.py status` 返回 `ready: true`、无残留演示会议、无失效演示房间；浏览器停在张三账号的空白“新建编排”页面，可直接开始录屏。

## 17. 2026-08-15 登录页演示凭据默认回填移除

- 登录表单不再把 `zhangsan` 与演示密码硬编码为响应式初值，首次进入和退出登录后的用户名、密码字段均为空。
- 表单关闭账号自动完成，密码字段使用 `new-password` 语义，避免浏览器继续回填此前使用过的演示凭据；用户仍可手动输入任意企业账号登录。
- 验证：前端 `npm run type-check`、`npm run build` 与定向 `git diff --check` PASS；前端镜像已重建并替换，容器健康。真实登录页重新加载后两个输入框均为空，浏览器控制台无应用错误。

## 18. 2026-08-15 全项目品牌改名为 WeMe

- 前端品牌名、品牌标记、页面标题、对话缓存键、跨页事件名与下载文件名前缀已统一为 `WeMe` / `weme`；演示脚本、测试、制度知识源、产品文档和 Agent 临时目录前缀同步完成改名。
- 新增 Flyway V13 与 Alembic 0006，对业务库会议、通知、草案、工具审计以及 Agent 会话标题、消息正文中的旧品牌数据执行原位替换，不删除会议、审计记录、会话或运行历史。
- RAG 初始化共处理 22 份制度文档，其中 4 份品牌相关文档重新索引、18 份内容未变文档跳过重复写入，当前索引共 285 个 chunk。
- 静态仓库扫描与两个 MySQL schema 的文本/JSON 字段扫描均未发现旧品牌残留；浏览器实测侧栏、历史任务和知识库页面均显示 `WeMe`，控制台无应用错误。
- 验证：Java Maven `verify` **88 tests**、Spotless/Jar PASS；Python Ruff、Mypy PASS，Pytest **162 passed**；前端 `npm run type-check` 与 production build PASS；演示脚本单元测试 **2 passed**、脚本编译 PASS；基础 Compose 配置有效，8 个长驻服务全部健康。

## 19. 2026-08-15 正式录屏前运行数据清理

- 先通过 `scripts/demo-two-scenarios.py cleanup --apply` 执行受控业务清理；本轮没有新的待取消演示会议或待恢复房间，李四会议 `198/199` 和专用房间 `RD-TEAM-202` 基线保持不变。
- 已清空全部 Agent 最近任务运行数据：312 个会话、371 次运行、180 条用户可见消息、1885 个步骤、484 个循环事件、1177 条 Python Tool 轨迹及 Redis DB 1 中 157 个 LangGraph checkpoint。RAG 文档、业务会议、Outbox/MQ 和 Java Tool 审计未删除。
- 已删除张三与管理员的 342 条站内会议通知，并把 133 张遗留 `PENDING` 预约草案统一转为 `REJECTED`；其他员工通知和既有业务审计记录保留。
- 清理后数据库复核显示 Agent 最近任务六张运行表均为 0、演示账号通知为 0、待确认草案为 0、checkpoint Redis DB 大小为 0；`rag_document` 仍为 23 条，Alembic/Flyway 分别保持 0006/V13。
- `python scripts/demo-two-scenarios.py status` 返回 `ready: true`、无残留演示会议和失效房间。浏览器实测张三侧栏没有最近任务，消息中心显示空态，最终停在 `http://localhost/chat` 的空白“新建编排”页。
