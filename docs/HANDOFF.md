# 项目开发交接

## 1. 当前状态

- 最后更新：2026-08-15（Asia/Shanghai）。
- 冻结基线：`SPEC.md` 1.3；浏览器只访问 Java，Java 是业务事实源，Python 负责固定的 Supervisor + Requirement/Policy/Scheduling、OR-Tools、RAG、HITL 和恢复。
- 当前分支：`main`；本轮清理前基线提交为 `d82f7b7 fix(frontend): constrain recent tasks to sidebar top`。
- 当前修改：包含项目清理、过时文档删除、交接收敛，以及前端侧栏会话区布局调整；没有修改 API、数据库迁移、业务逻辑或 Compose 拓扑。
- 运行环境：基础 Compose 的 Java、Python、前端、MySQL、Redis、Qdrant 和 RocketMQ 长驻服务均为 `healthy`；本轮未停服、未改写 `.env`、未删除数据库或命名卷。

## 2. 已交付能力

| 模块 | 当前可用能力 |
|---|---|
| Java | JWT/RBAC、手动会议闭环、30 分钟槽位、Redis Lua 预占、MySQL 最终唯一约束、幂等、Transactional Outbox、RocketMQ HOT 预约、Tool Gateway、SSE 代理、通知、异常重排、会前会后和知识库公共 API |
| Python | 受控 Agent Loop、DeepSeek 原生 Tool Calling、Source Fidelity、OR-Tools Top 3 + 独立硬约束验证、BGE-M3 + Qdrant RAG、HITL、Redis checkpoint、HOT 冲突恢复、会后结构化草案和分层评测 |
| Frontend | Vue 企业工作台、多轮聊天/SSE、Top 3、ACCEPT/EDIT/REJECT、安全 Trace、会议/会议室/员工/消息管理、异常重排、会前会后、知识库和对话线程恢复 |
| 部署 | 固定镜像基线、基础/开发 Compose、Flyway/Alembic、RAG 一次性入库、健康检查、Smoke/并发/真实模型评测入口 |

## 3. 最新可复现证据

- Java：最新后端变更的 JDK 21 Maven `verify` 为 **84 tests，0 failure/error/skip**，Spotless/Jar PASS。
- Python：最新 Agent 回归为 Ruff、Mypy PASS，Pytest **154 passed**（仅既有 LangGraph pending-deprecation warning）。
- Frontend：最新 `npm run type-check` 与 `npm run build` PASS。
- 真实模型：`artifacts/live-eval/component-full-final-20260815.json` 的 40 条完整门禁为 Route/Intent/Tool/Native Tool/Citation 100%、Constraint F1 95.31%、Source Fidelity Violation 0。
- 公开 API 多轮对抗：`artifacts/product-scenario-evaluation.json` 为 **16/16 PASS**；基线报告为 `artifacts/product-scenario-evaluation-baseline.json`。
- 演示验收入口：`docs/21-demo-acceptance-runbook.md`；最终报告入口：`docs/REPORTS.md`。

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
| `docs/00-09`、`docs/11-21`、`SPEC.md` | 冻结规范、架构/设计依据、验收标准与演示手册；`docs/09` 仅作历史 UI 设计基线，`docs/12` 仅作 Agent 设计依据。 |
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
- 本轮没有重跑 Maven/uv/npm 模块门禁，因为清理仅删除可重建输出，立即执行安装/构建会重新生成本轮的清理目标。下次修改模块时按 `AGENTS.md` 重建依赖并运行完整门禁。

## 7. 2026-08-15 前端侧栏会话区顺序调整

- “新建编排”“搜索会话”“最近任务”已从品牌区下方的固定三分之一高度容器移入统一侧栏导航，排列在工作台、协作、管理和当前运行记录之后。
- 会话区与主要功能共享同一滚动区域；新建和搜索采用与导航一致的轻量行样式，最近任务继续按真实 `threadId` 聚合，恢复、搜索、折叠和移动端抽屉语义不变。
- 验证：`npm ci`、`npm run type-check`、`npm run build`、定向 `git diff --check` 均 PASS；本地开发页在 1440×900 与 390×844 下完成登录、真实只读制度问答、最近任务展示和移动导航检查，应用控制台无 error/warning。
- 环境提示：`npm ci` 仍报告既有传递依赖要求 Node `^24.15.0`，当前为 `24.14.0`；安装、类型检查与构建均成功。

## 8. 恢复本地开发环境

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
