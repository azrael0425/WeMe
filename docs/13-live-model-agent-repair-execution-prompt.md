# 13. 真实模型 Agent 修复执行提示词

以下内容可完整复制到一个新的 Codex 对话中执行。

---

你正在 `D:\agent` 仓库继续开发企业智能会议室调度系统。请直接完成代码、测试、真实模型评测、Compose 联调和交接，不要只给建议或再次写一份空泛方案。

## 一、启动要求

修改任何文件前必须完整阅读：

1. `AGENTS.md`
2. `SPEC.md`
3. `docs/HANDOFF.md`
4. `docs/01-functional-spec.md`
5. `docs/02-system-architecture.md`
6. `docs/04-agent-spec.md`
7. `docs/05-data-and-api-spec.md`
8. `docs/07-test-and-evaluation.md`
9. `docs/11-controlled-agent-loop-design.md`
10. `docs/12-live-model-agent-repair-plan.md`

随后使用 `rg --files`、`git status --short`、实际代码、当前测试和容器状态核验真实基线。当前工作区有上一轮未提交改动，它们属于用户；不得 reset、checkout、覆盖或格式化无关修改。不得自动提交或暂存。

## 二、任务目标

修复真实 `deepseek-v4-flash` 下的 Agent 业务链路，使其从“协议接通但自然语言失败”升级为可验证的真实模型 Golden Path：

1. Supervisor 路由经过确定性 Route Evaluator，不再把预约直接 FINAL，也不把 Policy 问答路由到 Requirement。
2. Requirement 改为 `RequirementDraft -> Source Fidelity/Semantic Evaluator -> 一次 Optimizer -> Deterministic Normalizer -> MeetingRequest`，禁止虚构姓名、时间、设施和意图。
3. “6个人、要白板”这种只有人数没有姓名的预约可以生成候选；人数是容量，不得被转换成虚构姓名。组织者由 AgentContext 确定并参与忙闲。
4. 自然中文 CREATE 请求真实走过 DeepSeek 原生 `tools/tool_calls/tool`、Java READ Tool、OR-Tools、独立验证器、Top 3、无占用草案和 HITL。
5. 完成 MODIFY/CANCEL 的真正草案与 HITL 闭环，不能再以“完成只读事实查询”伪装成功。
6. 持久化并展示 Loop phase/iteration/decision/stopReason、模型/Prompt/Schema 版本和 Token usage。
7. 建立 live-model evaluation runner 和门禁；fixture 报告继续只叫 component fixture。

运行时 Agent 仍固定为 Supervisor、Requirement、Policy、Scheduling。不要引入 DeepAgents、Critic Agent 或第五个产品 Agent。

## 三、已知真实失败证据

上一轮已经真实切换到：

```text
provider=deepseek
model=deepseek-v4-flash
baseUrl=https://api.deepseek.com
```

原生 Tool Calling 协议探针已 PASS：真实模型能返回 `resolve_employees` tool call、合法 JSON 参数和 Token usage。失败集中在业务语义：

- Supervisor 曾返回 `SCHEDULING`，旧图把它当 FINAL，零 Tool 完成。
- Requirement 曾把“2人、15:00-16:00”幻觉成“张三、李四、2小时”。
- 显式“张三、李四”也曾被输出为缺少 `requiredParticipants`。
- “VIP会议室有哪些使用规则”曾被错路由为 Requirement 并要求补预约字段。
- 多条自然语言请求没有到达 Java Tool、候选和 HITL。

不要重复证明 API 格式本身可用；重点修复业务状态机和真实模型适配。

## 四、必须遵守的业务裁决

### 1. 人数与姓名

- 只有人数没有姓名是有效 CREATE/RECOMMEND 输入。
- `6个人` => `minimumCapacity=6`，不是六个虚构姓名。
- 未点名时 `requiredParticipants=[]`；组织者来自 AgentContext，并纳入忙闲和最终 REQUIRED 人员。
- FIND_COMMON_TIME 若没有要协调的人，需要澄清。

### 2. 安全默认

- title 缺失：确定性默认“会议安排”。
- meetingType 缺失：确定性默认 `GENERAL`。
- 设备缺失：空列表。
- 不得默认日期/时间窗口；不能从起止时间或类型推导时长时必须澄清。

### 3. 初始路由

Supervisor 初始只允许 `POLICY/REQUIREMENT/CLARIFICATION`。禁止直接进入 `SCHEDULING/HITL/FINAL/FAIL`。Schema 合法不代表业务路由合法。

### 4. 写操作

模型只获得 READ Tool。CREATE/RESCHEDULE/CANCEL 的 DRAFT 和 WRITE 都由确定性节点执行；三者均必须经过 ACCEPT/EDIT/REJECT HITL。

## 五、实现要求

### Slice A：Route 与 Requirement

1. 在 Pydantic Schema 中新增/调整：
   - `SupervisorDecision{route,intentHint,confidence,evidence,summary}`
   - `RequirementDraft`
   - `FieldEvidence{field,source,provenance}`
   - 结构化 route/source/semantic feedback
   - `NormalizationReport`
2. 实现确定性 `RouteEvaluator`：验证初始允许路由、领域锚点、evidence 原文子串；失败最多修复一次，第二次安全降级或澄清。
3. 实现 Source Fidelity Evaluator，至少覆盖：
   - 虚构/遗漏姓名
   - 人数被当姓名
   - 人数/容量不一致
   - 显式时间被改写
   - 区间/时长不一致
   - 无原文设施
   - intent 与执行动词/Policy 问句冲突
4. Requirement 不再直接被必填 MeetingRequest 迫使猜测；Normalizer 才能填 title/meetingType 等安全默认。
5. 使用 `docs/12-live-model-agent-repair-plan.md` 第 5、6 节的运行时 Prompt。Prompt 必须有版本常量，不要散落无法追踪的字符串。
6. 增加失败先行回归测试，必须复现上一轮真实问题。

### Slice B：Scheduling READ Loop

1. CREATE 姓名为空时不得报 `REQUIREMENT_MISSING`，也不得调用 `resolve_employees`。
2. 必须查询组织者忙闲；明确姓名存在时查询组织者 + 解析后的 REQUIRED 人员。
3. 房间查询容量使用 `max(explicit minimumCapacity, unique organizer + resolved participants)`。
4. Verifier 按 Intent 判断事实是否齐备，不能以 `resolved=[]` 一律失败。
5. 保留 Pydantic 参数、canonical context、风险、身份、重复指纹、结果大小和预算门禁。
6. 使用文档第 7 节 Scheduling Prompt。

### Slice C：MODIFY/CANCEL

1. 复用 Java 已有接口，不创建另一套业务规则：
   - `/internal/v1/tools/reschedule-drafts`
   - `/internal/v1/tools/reschedule-drafts/{token}/confirm`
   - `/internal/v1/tools/cancellation-previews`
   - `/internal/v1/tools/cancellation-previews/{token}/confirm`
2. 在 Python Tool Client 补齐 Pydantic 输入、响应、稳定 toolCallId/idempotencyKey 和错误映射。
3. AgentState 增加明确 operation type，draft 使用 CREATE/RESCHEDULE/CANCEL 可辨别结构。
4. MODIFY：唯一解析目标会议，合并未变字段，重新读取事实/求解，生成 Before/After 草案，HITL 后确认。
5. CANCEL：唯一解析目标，生成取消预览，HITL 后确认；多匹配必须澄清。
6. EDIT 重新校验并换新 token；REJECT 零 WRITE；ACCEPT 只能分派对应确认 Tool。
7. 用数据库前后快照证明三类草案在 HITL 前零正式副作用。

### Slice D：Trace、Token 与前端

1. 统一 Provider 返回值，让 structured completion 和 tool completion 都带 usage/model；不能只统计 Tool 分支。
2. 失败模型调用也要计入 bounded call count；Token 使用 API 返回值，不估算。
3. 用版本化 Alembic migration 持久化模型/Prompt/Schema 版本、input/output/cache tokens 和 loop events；不得 ORM 自动改表。
4. Trace 返回 `loopEvents` 和安全运行统计，不包含隐藏推理、Key、JWT、Service Token、confirmation token 或完整敏感正文。
5. 前端处理 `agent.loop`，实时与刷新恢复后都显示 PLAN/ACT/OBSERVE/VERIFY/REPLAN、iteration、decision、stopReason、预算、Token 和模型名。
6. HITL 卡片区分创建/改期/取消；改期显示 Before/After，取消显示目标会议。

### Slice E：Live Model Eval

1. 新增显式 CLI，例如：

```powershell
uv run python -m app.evaluation.live --suite core --repeats 3 --output ../artifacts/live-eval
```

2. 未配置 Key 时必须 `SKIPPED`，不能 PASS。
3. core 12 条每条重复 3 次，全量 40 条至少单次；用例和门槛严格遵循 `docs/12-live-model-agent-repair-plan.md` 第 10 节。
4. 报告 provider、模型、Prompt/Schema 版本、重复次数、route/intent/field/tool/terminal、失败分类、P50/P95 和 Token。不得写入 Key、隐藏推理或确认令牌。
5. 费用只有在显式版本化价格配置存在时才估算，不硬编码易变价格。

## 六、测试要求

至少新增以下回归：

1. Supervisor `SCHEDULING -> REQUIREMENT` 控制边界。
2. Policy 强锚点错路由被修复；第二次失败安全降级。
3. `2人` 不产生张三/李四。
4. 显式张三/李四不会被遗漏，模型虚构姓名被拒绝。
5. 15:00-16:00 不会变成两小时。
6. title/meetingType 缺失使用确定性默认，不追问无意义字段。
7. 6人无姓名 CREATE 跳过 resolve，查询组织者忙闲 + 容量6房间并进入 HITL。
8. Tool 参数偏离 canonical context 被拒绝且无 Java 副作用。
9. MODIFY 草案 Before/After、EDIT 重校验、ACCEPT 成功、REJECT 无副作用。
10. CANCEL 多匹配澄清、唯一目标预览、ACCEPT/REJECT。
11. Loop/usage 持久化与 Trace 脱敏。
12. 前端 `agent.loop` 和三类 HITL 类型/构建验证。

执行并记录：

```powershell
Push-Location agent-service
uv sync --frozen --group dev
uv run ruff check .
uv run mypy app
uv run pytest
uv run python -m app.evaluation
Pop-Location

Push-Location business-service
.\mvnw.cmd verify
Pop-Location

Push-Location frontend
npm ci
npm run type-check
npm run build
Pop-Location

docker compose config --quiet
docker compose -f compose.yaml -f compose.dev.yaml config --quiet
```

环境允许时再构建镜像、运行 Compose Smoke 和真实 live eval。开发 Compose 若因宿主端口占用失败，先确认端口，不要删除卷；可使用基础 Compose 验证内部网络。

## 七、真实验收输入

真实模型验收必须至少包含以下自然中文，不得改写成 JSON 或字段标签：

```text
帮我预约2026年8月20日下午3点到4点的会议室，6个人，要白板，先给我候选。

请安排张三和李四在2026年8月20日15:00到16:00开一小时架构评审，需要白板，先别替我确认。

VIP会议室有哪些使用规则？请只根据制度回答并给引用。

把我刚才那个架构评审改到2026年8月20日16:00，其他不变，先给我看变更草案。

取消会议 ID 9001，先给我预览，不要直接取消。
```

CREATE 至少到 `WAITING_CONFIRMATION`；先用 REJECT 验证无写副作用，再用一条隔离数据执行 ACCEPT 并通过正常取消接口清理。Policy 必须只走 Policy 且引用有效。MODIFY/CANCEL 必须展示对应草案并证明确认前业务数据未变化。

## 八、验收门槛和反伪完成

门槛：Route >=95%、Intent >=90%、Constraint F1 >=85%、Tool Selection >=90%、Source Fidelity Violation=0、Native Tool Protocol=100%、Hard Constraint Violation=0、Citation Validity=100%、HITL Before Side Effects=100%、core 自然语言轨迹成功率>=80%。

以下均不得标为成功：

- 切回 fixture 后通过。
- 零 Tool 的 `SUCCEEDED` 被当成预约成功。
- `WAITING_USER_INPUT` 被当成 Golden Path 成功。
- 要求用户输入 JSON/字段标签后才通过。
- 只改 Prompt，没有确定性 Evaluator 和失败测试。
- MODIFY/CANCEL 只查询不生成草案。
- live eval 没运行却写 PASS。
- 把 Key、隐藏推理、JWT、Service Token 或 confirmation token 写入文件/日志/Trace。

用户曾在聊天里公开过一个 Key。不得复制、打印、写入 `.env` 或命令；只使用当前宿主环境中已配置且不会回显的 Key。如果真实模型未配置或鉴权失败，准确记录阻塞，不得伪造结果。

## 九、交付

完成后：

1. 亲自检查全部 diff 和跨服务契约。
2. 更新 `SPEC.md`/专项规范中确实发生变化的内部协议和验收条件。
3. 更新 `docs/HANDOFF.md`：修改文件、实现能力、命令、结果、真实 live eval 指标、失败样本、未解决风险和下一条任务。
4. 最终答复必须明确区分：fixture PASS、integration PASS、live-model component PASS/FAIL、live-model trajectory PASS/FAIL。
5. 不提交、不暂存、不推送，除非用户另行明确授权。

终端条件：只在代码落盘、受影响模块测试通过、Compose/真实模型按环境实际验证、文档一致且无密钥泄漏后结束；若真实模型指标未达门槛，保留修复后的代码和报告，明确写 FAIL 及具体失败，不得宣称完成。

---
