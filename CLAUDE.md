# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

DevFlow AI 是面向 GitHub PR / Issue 研发协作的全栈 AI Agent 项目：同步仓库数据（Issue、PR diff、Review 评论、CI 日志），叠加项目知识库与本地工作区代码，输出结构化分析与**安全的操作草稿**。

后端 FastAPI（`backend/`，imports 为 `app.*`）+ 前端 Next.js 15 App Router（`frontend/`）+ Postgres/Milvus（Docker Compose）。代码注释、prompt、文档以中文为主。

## 常用命令

### 基础设施

```bash
docker compose up -d postgres etcd minio milvus   # redis 属于 optional profile，默认不启动
docker compose ps
```

Docker Desktop 必须先启动。Postgres 用 `pgvector/pgvector:pg16`，`scripts/init_pgvector.sql` 会挂载执行 `CREATE EXTENSION vector`。

### 后端

```bash
cd backend
python -m venv .venv && .venv/Scripts/activate      # Windows
pip install -r requirements.txt
uvicorn app.main:app --reload                        # :8000，文档 /docs，健康检查 /health
```

**必须从 `backend/` 目录启动**：`app/core/config.py` 用 `env_file=("../.env", ".env")`，是相对 CWD 解析的，从别处启动会读不到仓库根目录的 `.env`。

### 前端

```bash
cd frontend
npm install
npm run dev      # :3000
npm run build
npm run lint     # next lint（.eslintrc.json 只 extends next/core-web-vitals）
npx tsc --noEmit # 没有 typecheck script，类型检查用这个
```

`next.config.js` 里 `distDir` 在开发模式是 `.next-dev`（不是 `.next`）。

### 测试

```bash
cd backend
python -m pytest app/tests                              # 全量，209 个用例，约 50s
python -m pytest app/tests/test_chat_agent.py           # 单文件
python -m pytest app/tests/test_chat_agent.py::test_xxx # 单个用例
python -m pytest app/tests -k "rag and not ragas"       # 按名筛选
```

测试**不需要 Postgres / Milvus / LLM key**：用内存 SQLite（模块顶部 `os.environ.setdefault("DATABASE_URL", "sqlite+pysqlite:///:memory:")` + 自建 `create_engine`），Milvus 与 `llm_api_key` 由 `monkeypatch` 置空。没有 `conftest.py`、没有 `pytest.ini`/`pyproject.toml`，pytest-asyncio 是 strict 模式，异步用例都显式带 `@pytest.mark.asyncio`。没有配置任何 Python linter/formatter（无 ruff/black/flake8 配置文件）。

### 演示与种子数据

```powershell
.\scripts\start_rag.ps1                       # 轻量演示：SQLite + deterministic embedding，关掉 Milvus
.\scripts\start_rag.ps1 -UseConfiguredLLM     # 同上但启用已配置的 LLM
```

该脚本显式关闭 Milvus、用 deterministic embedding 与启发式重排，并停用演示库的 0.8 分数阈值；向量检索状态会显示 `degraded`，**不能用来证明完整 RAG 跑通**。

```bash
cd backend && PYTHONPATH=$(pwd) python ../scripts/seed_demo_data.py   # 灌演示仓库数据
```

`seed_demo_data.py` import 了 `app.*`，所以必须让 `backend/` 在 `PYTHONPATH` 上（`start_rag.ps1:40` 就是这么做的）。

## 架构

### 请求主链路：一次对话

入口全在 `app/api/routes/chat.py`：`POST /api/chat`（非流式）、`/api/chat/stream`（SSE）、`/api/chat/plan` + `/api/chat/plan/execute/stream`（两阶段计划）。一轮对话的顺序：

1. `ProgressiveContextManager.ensure_headroom()` —— **在存用户消息之前**就可能压缩历史
2. `MessageStore.append()` 存用户消息；`EvidenceStore.rebuild_repo()` 重建证据索引
3. `ContextAssembler.assemble()` 产出 `system_context` / `history_messages` / `evidence` / `compression_stats`
4. `context["tools"] = _build_conversation_tools(repo, conversation, db)`
5. `ChatAgent().run_stream()` —— LLM 工具调用循环
6. 落库：assistant `ChatMessage`、`AgentRun`、`SessionSealer.seal()` 更新 `ThreadMemory`/`ConversationMemory`

`ChatAgent`（`app/services/agents/chat_agent.py`，约 2k 行）是核心。工具循环是一个 **LangGraph `StateGraph`**：`model_decision` → `tool_action` → 回到 `model_decision`，上限 `max_steps=12`、同一工具重复调用 2 次即停。在进图之前还有一系列前置分支：Skill 激活、memory policy 快捷路径（近期历史已能回答就直接答）、project overview 预取。**未配置 `llm_api_key` 时 `self.model is None`，走确定性降级模式**（用预取证据作答）。

SSE 事件由 `chat_agent.py::encode_sse` 发出：`thought` / `tool_use` / `tool_result` / `action` / `observation` / `delta` / `final`。

### 多 Agent 工作流

`ChatAgent` 可调用 `workflow.run_engineering_review`，或客户端走 `POST /plan` → 审阅 spec → `POST /plan/execute/stream`。

`WorkflowOrchestrator`（`app/services/agents/workflow_orchestrator.py`）按 DAG 分批并行执行任务（`asyncio.gather` + 每任务超时 `workflow_task_timeout_seconds`），每轮后 `ObserverAgent` 出 finding，必要时 `PlannerAgent.replan()`（最多 1–2 次），最后 `SynthesisAgent` 出统一决策备忘。环境变量式地持久化 `AgentWorkflowRun` + `AgentTaskRun`。

**关键：Planner / Observer / Synthesis 是确定性规则代码，不调用 LLM。** 只有 Issue / PR / CI 三个专用 Agent 真正打模型。

任务 runner 的接线在 `chat.py::_build_engineering_workflow_runtime`，**不在 orchestrator 里**——按 `task_type` 注册 `run_repo_health` / `run_issue_task` / `run_pr_task` / `run_ci_task` / `run_rag_task` / `run_safety_task`。

### 工具注册与 Skill Runtime

**没有中央工具注册表。** 工具是每次请求在 `chat.py::_build_conversation_tools(repo, conversation, db)` 里构造成闭包的。公开工具名（带点）：`workspace.list_files` / `workspace.read_file` / `workspace.search_code`、`rag.search_similar_documents`、`repo_health.generate_summary`、`issue_agent.analyze_issue`、`pr_review_agent.analyze_pr`、`ci_debug_agent.analyze_workflow_run`、`report_agent.generate_weekly_report`、`safety_agent.classify_action_risk`、`workflow.run_engineering_review`、以及三个 `devflow_*` 记忆工具。

**新增一个工具要改多处**：在 `_build_conversation_tools` 加闭包 + 在 `ChatAgent.available_tools` 登记公开名；若还要做成工作流任务，再去 `_build_engineering_workflow_runtime` 注册 runner。

三个 `devflow_*` 记忆工具**不在进程内**：`app/services/mcp_client.py::StdioMcpClient` 每次调用都 spawn 一个 `python app/mcp/memory_server.py` stdio JSON-RPC 子进程（initialize → `tools/call` → kill）。

Skill Runtime（`app/services/skills/registry.py`）加载 `app/skills/*/SKILL.md`，frontmatter 是手写解析器解析的（不是 PyYAML），字段含 `entrypoint`、`tools`、`triggers`、`safety_level` 等。两条激活路径：按消息（`activate`，显式或关键词打分）用于 ChatAgent；按 entrypoint（`activate_entrypoint`）用于工作流 runner。**frontmatter 里的 `entrypoint` 必须等于某个工具名。** 现有 6 个 skill：`ci-debug` / `issue-triage` / `pr-review` / `project-investigation` / `safety-draft` / `weekly-report`。

### RAG 管道

`app/services/rag/`，HTTP 面在 `app/api/routes/rag.py`（prefix `/api/rag`）。

检索核心是 `retrieval.py::search_similar_documents`，两条召回分支：
- **向量**：`vector_store.py::search_milvus_documents`，Milvus cosine，按 `repo_id` 过滤
- **关键词**：`retrieval.py::_keyword_candidates`，纯 Python 实现的加权字段 BM25（`K1=1.5/B=0.75`），**直接扫描 Postgres 的 `Document` 行，不用 Postgres FTS**。字段权重：symbol 4.5 / title 4.0 / path 3.0 / labels 2.5 / content 1.0

融合与后处理顺序：`_merge_candidates`（`0.62*vector + 0.30*keyword + 0.08*dual_recall_bonus`）→ 近重复去除（SHA-256 指纹 + Jaccard ≥0.9）→ 每个 parent 最多 2 个 chunk → rerank → `compress_retrieval_results` 压到 token 预算。

`knowledge_base.py::retrieve_knowledge` 是 API 用的编排层：校验 KB 配置 → 查询向量化 → `search_similar_documents` → rerank → `expand_parent_context`（取同一 `parent_id` 的相邻 chunk）→ 分数阈值 → 压缩 → **answer gate**（`answer_gate.py` 判定 `answer` / `insufficient_evidence` / `conflict` / `ask_clarification`）。

外部 rerank（`knowledge_base.py::_qwen_rerank`，Cohere 兼容的 `/reranks` 端点）**任一失败都回退到 `rerank.py` 的启发式重排**，并在 trace 里记 `fallback=True`。

### 数据层与存储边界

`app/db/models.py`（全部表在单个模块）+ `app/db/session.py`（`Base`、`engine`、`SessionLocal`、`get_db`）。

**Postgres 是所有结构化数据的权威**（仓库、Issue、PR/文件/评论、CI run、`Document` chunk 全文与 metadata、KB 配置、代码符号/关系、会话/记忆/评测/草稿）。**Milvus 只持有部分 `Document` 的一份拷贝**，仅供向量召回分支使用。

哪些进向量由 `app/services/rag/source_policy.py` 单一决定：
- 向量化：`issue`、`pull_request`、`workflow_run`、`project_doc`、`knowledge_file`、`memory_note`
- 只进 Postgres 不向量化：`project_overview`、`project_manifest`、`team_member`、`weekly_report`、`chat_session`、`conversation_memory`、`thread_memory`
- `code_file` **被硬拦截**：插入即抛 `ValueError`，`purge_legacy_code_documents` 清历史行，启动时还会 `delete_milvus_documents(source_types=["code_file"])`。源码永远走 `code_search.py` 的实时检索（ripgrep + Python 兜底）和工作区工具

这就是 README 说的「不是所有私域数据都向量化」。

### 前端

`lib/api.ts` 是唯一的客户端：`apiGet` / `apiPost` / `apiPatch` / `apiUpload` / `apiDelete` / **`apiPostStream`**（SSE：`getReader()` 逐块解码，按 `\n\n` 切分，解析 `event:` / `data:` 行）。`API_BASE_URL` 默认 `http://localhost:8000`。

**`@tanstack/react-query` 是依赖但代码里完全没用到**——所有页面都是 `useState`/`useEffect` + `lib/api.ts`，轮询是各处内联的 `setInterval`。不要假设存在 React Query 模式。

`app/page.tsx` 是 **5,933 行的单体**主工作台（聊天、仓库/会话侧栏、Issue/PR/CI 标签页、团队与记忆、知识图谱、skill/MCP 面板、plan mode）。其余页面较薄。所有页面都是 `"use client"`。

## 必须遵守的约束

1. **一个 Milvus collection 只能有一个向量空间。** `validate_embedding_contract` 要求每次调用的 `(provider, model, dimensions)` 等于全局配置，`MilvusVectorStore._ensure_collection` 会比对 collection 实际维度并拒绝启动。换 embedding 模型/维度**必须换 collection 名并全量重建索引**；`PATCH /api/rag/repositories/{id}/config` 对破坏契约的改动返回 409。不同模型的向量在 cosine 下不可比，混用会静默污染排序。跨仓库隔离只靠 `repo_id` 过滤，不是分 collection。
2. **改了 chunking / source_policy / embedding 之后必须 reindex。** `POST /api/rag/{repo_id}/reindex`（`qa.py::reindex_documents`）是唯一会重建 issue/PR/CI 的 `Document` 并重写向量的入口。只部署代码不会触发，README 明确要求升级后对每个知识库调一次。
3. **Milvus 与 Postgres 要手工保持同步。** 写入成对（`Document` 行 + Milvus 向量），删除成对（`delete_document_group` / `replace_documents`）。新增任何删除或变更路径都必须同时更新两个存储，否则检索会返回幽灵结果。
4. **不要「修掉」`code_file` 的拦截。** 源码按设计不被向量化，去 `source_policy.py` 放开会让索引迅速过时且不可信。
5. **写 GitHub 永远不自动发生。** 聊天只产出 `ActionDraft`（`pending_confirmation`）；真正执行需另调 `POST /api/action-drafts/{id}/confirm`（权限 `draft:approve`）。Observer 的 `human_confirmation_required` 与 synthesis 的措辞都围绕这个两阶段模式。
6. **schema 变更是内联的，没有迁移工具。** `main.py` 启动时做 `create_all` 加一串 inspector 守护的 `ALTER TABLE ADD COLUMN`（还有一次 `documents.embedding` 的 DROP）。`alembic` 虽在 requirements 里但**没有接进启动流程**。加新列 `create_all` 能处理；**改列类型/改名必须自己再加一个内联 ALTER 块**。
7. **两套 LLM 栈，行为完全不同。** `ChatAgent` 用 `langchain_openai.ChatOpenAI` + LangGraph 原生 tool calling；**其它所有地方**（issue/pr/ci agent、`SessionSealer`、压缩摘要、`rag/qa.py`）用 `app/services/llm/client.py::LLMClient`（裸 httpx）。`LLMClient` **失败时静默返回 `{}` / `""`，不抛异常**——配置错了的表现是「答案为空」，不是报错。别假定 LangChain 到处都在用。
8. **RAG 内容（尤其 CI 日志）是不可信数据。** `indexing.sanitize_ci_log` 在入库前脱敏（私钥、GitHub token、AWS key、JWT、bearer、`::add-mask::`、`*_key/*_token/*secret*`），answer prompt 把 `<knowledge>` 标为不可信并要求 `[n]` 引用。改动这条链路时不要丢掉脱敏。
9. **普通用户配置下不允许静默换向量空间。** 默认 embedding 依赖缺失时必须明确报错；只有 `scripts/start_rag.ps1` 那条演示路径会显式选 `deterministic`。

## 文档可靠性

写代码前注意，`docs/` 与 README 有几处需要留意：

- README 说 RAGAS 固定评测基准在 `docs/datas/evals/rag_cases.json`，`suites.py:15` 也确实指向 `docs/datas/evals/`，但**该目录在仓库里不存在**（全仓库搜不到该文件）。没有它时 `list_eval_suites` 只会返回内置的 `auto-document-smoke`，README 里那个 `devflow-real-project-rag-baseline` 跑不起来。这是文档与实现唯一真正冲突的地方。
- `docs/api/openapi.md` 是**准确的，但只是精选子集**（列了 17 条，实际注册 93 条）。查全量以运行中的 `/docs` 或 `/openapi.json` 为准。
- `docs/architecture/system_design.md` 只有一张极简 mermaid 图，信息量低于 README；`database_schema.md` 提到的表都真实存在，但只列了一小部分，别当成完整 schema。

`docs/architecture/agent_workflow.md` 与当前实现基本一致，可以放心作为 Agent 链路入门的起点。

## 协作约定与项目背景（持久记忆）

本节记录跨会话有效、且无法从代码或 git 历史中直接看出的信息。需要更新时**直接修改本节**，不要另建 `~/.claude/.../memory/` 下的记忆文件。

### 用户背景

- 用户是 **Java / MySQL 背景**，对 Python、PostgreSQL、GitHub 操作还不熟悉。
- 不要假设用户熟悉 Python 生态惯例（虚拟环境、装饰器、`async/await`、pytest、依赖注入风格等）。首次遇到时用一句话说明，并尽量给 Java / MySQL 类比，例如：pytest ↔ JUnit、SQLAlchemy Session ↔ MyBatis SqlSession、Pydantic 模型 ↔ DTO + Bean Validation、Alembic ↔ Flyway/Liquibase。
- PostgreSQL 与 MySQL 的差异（`jsonb`、`RETURNING`、schema 搜索路径、pgvector 扩展等）在首次出现时点明。
- GitHub 相关操作（`gh` CLI、PR/Review 流程、remote/fork 等）需说明命令在做什么，不要默认用户已经熟练。

### 项目定位

- 这是一个**学习项目**。用户后续会在此基础上改造成自己的简历项目，目标是 **Agent / Java 实习**。
- 因此改动应满足：
  1. **可解释**：不引入用户无法在面试中讲清楚的复杂度或"魔法"。
  2. **可迁移**：涉及 Agent 编排、RAG、工具调用、两阶段确认等核心设计时，说明取舍，方便用户转述。
  3. **突出亮点**：遇到适合写进简历的能力点（多 Agent DAG 编排、混合检索、安全草稿确认、记忆机制），主动指出并说明价值。
- 不要为了让 demo 好看而绕过上文「必须遵守的约束」中的安全性与一致性要求。

### 协作偏好

- 默认用中文回答，代码、命令、标识符保留英文。
- 解释优先于堆代码：改核心链路时说明"为什么这样改"，而不只是"改了什么"。
- 节奏：多文件或架构性改动先给出简短计划再动手；明确的单点修复直接做，不反复确认。
- 需求不明确或存在多种合理方案时，先问清楚，并尽量给出带推荐的具体选项，而不是开放式追问。
- 代码改动后优先运行相关测试（`python -m pytest app/tests -k ...`）；前端改动跑 `npx tsc --noEmit` 和 `npm run lint`。如果无法验证，明确说明。

### 记忆维护

- 只记录**跨会话有用、且代码 / git / 本文件其他章节中没有**的事实。
- 不记录：代码结构与实现细节、临时任务进度、可随时重新推导的信息、个人隐私或求职投递细节。
- 优先修改已有条目，不重复追加。
