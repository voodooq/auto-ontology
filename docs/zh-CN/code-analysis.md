<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto-Ontology 代码架构深度分析与中文文档索引

## 一、项目定位与总体架构

**Auto-Ontology**（当前版本 `1.2.0`）是 NVIDIA 开源的企业级智能体语义上下文层与可信 Text-to-SQL 平台。它从企业关系型数据库元数据出发，自动生成包含**业务术语（Terms）**、**SQL 属性（SQL Attributes）**和**语义关系（Relationships）**的知识本体图谱，结合人工审核认证与向量检索，为对话界面、REST API 与 MCP 智能体提供可解释、可校验的结构化数据问答能力。

系统采用前后端分离的多服务架构（见 [`docker-compose.yml`](../../docker-compose.yml)）：

```text
  浏览器 / REST 客户端 / MCP 客户端
                 │
                 ▼  (HTTP/HTTPS, 唯一对外入口 :3000)
  ┌──────────────────────────────────────────┐
  │ Next.js 16 前端与统一网关 (:3000)          │
  │ Better Auth 认证 / RBAC / API 代理       │
  └──────────────┬───────────────────────────┘
                 │ (转发 x-auto-ontology-user-id，私网调用)
                 ▼
  ┌──────────────────────────────────────────┐       ┌──────────────────────────────────┐
  │ FastAPI 主后端服务 (:3001)                │──────▶│ Ingestion Service 摄取服务 (:3002)│
  │ - 15 组 /api 业务路由                     │       │ - DataScheduler (元数据摄取)      │
  │ - WarmPool 隔离运行 LangGraph 智能体      │       │ - SemanticScheduler (语义编译)    │
  └──────────────┬───────────────────────────┘       └──────────────┬───────────────────┘
                 │                                                  │
                 └──────────────────────┬───────────────────────────┘
                                        ▼
                        ┌───────────────────────────────┐
                        │ PostgreSQL 17 + pgvector(:5432)│
                        │ - public schema (Alembic 管理) │
                        │ - frontend schema (Prisma管理) │
                        └───────────────────────────────┘
```

另外可独立运行 [`mcp/auto_ontology_mcp/`](../../mcp/auto_ontology_mcp/server.py)（默认 `:3003`）作为只读 MCP 服务器，经由 Next.js `:3000` 网关对外提供工具能力；若配置 `VAULT_*`，连接凭据可托管至 HashiCorp Vault；若配置 `KUMO_RFM_API_URL`，预测型问题可路由至 KumoRFM。

---

## 二、核心模块代码剖析

### 1. 前端与统一安全网关（`frontend/`）

- **技术栈**（[`frontend/package.json`](../../frontend/package.json)）：Next.js `16.3.8`、React `19.2.4`、Tailwind CSS `^4`、Better Auth `1.7.4`、Prisma `^7.7.0`、Sigma `^3.0.3` + Graphology + `d3-force`（用于本体图谱渲染）。
- **四级统一身份解析**（[`frontend/auth/resolve-user.ts`](../../frontend/auth/resolve-user.ts)）：所有 `/api/...` 接口按严格优先级解析调用者身份：
  1. Better Auth 浏览器会话 Cookie；
  2. API Token（`x-api-key` 或带 `auto_ontology_` 前缀的 Bearer Token）；
  3. Auto-Ontology 签发的 OAuth Access Token（供 MCP 客户端使用）；
  4. 外部 SSO ID Token（如 AI-Q 转发的 JWT，按 Email 或 Account `subject` 映射本地用户）。
- **信任边界设计**：Next.js 鉴权后将 `x-auto-ontology-user-id` 注入上游请求头透传给 FastAPI `:3001`。后端信任该请求头而不再二次验证，因此 `:3001` 与 `:3002` 必须隔离在私网内。
- **自研零依赖图表引擎**（[`frontend/components/ResultChart/`](../../frontend/components/ResultChart/README.zh-CN.md)）：通过 Zod 校验智能体输出的 JSON 规格，以纯内联 SVG 渲染柱状图、折线图、差值图、KPI 卡片与轮播图，并支持表格切换和防公式注入的 CSV 导出。

### 2. 数据摄取与语义编译流水线（`auto_ontology/ingestion_service/` & `semantic/`）

- **双调度器防竞态设计**（[`auto_ontology/ingestion_service/__main__.py`](../../auto_ontology/ingestion_service/__main__.py)）：
  - `DataScheduler` 负责周期性拉取外部数据源 Schema、列画像与样本值；
  - `SemanticScheduler` 构造时传入 `depends_on=data_scheduler`，保证首次语义编译严格等待首轮数据摄取完成后再启动，之后各自按 24 小时周期独立调度；
  - [`router.py`](../../auto_ontology/ingestion_service/router.py) 暴露 `/ingest`、`/ingest/delete`、`/semantic/reset`、`/semantic/compile`（均为 `202 Accepted` 非阻塞异步触发），且 `/semantic/reset` 严格串行执行“停止运行中编译 → 删除旧语义与向量 → 触发重建”，防止并发读写撕裂。
- **并发抽取 + 确定性提交队列**（[`auto_ontology/semantic/pipeline.py`](../../auto_ontology/semantic/pipeline.py)）：
  - `compile_semantic_layer` 使用线程池（`_WORKERS = 3`）并发执行耗时的 LLM 术语提取与外键检测；
  - 写入数据库前通过 `_OrderedCommitQueue` 与 `_OrderedCommitSlot` 按规范表序（`_table_commit_key`）排队提交，既保留了 LLM 并发吞吐，又确保同名业务术语归属完全确定；取消任务时通过 `skip_if_unused()` 非阻塞释放工作线程，避免排队死等。

### 3. 基于 LangGraph 的多阶段 Text-to-SQL 状态机（`auto_ontology/retrieval/text_to_sql/`）

核心图编排位于 [`text_to_sql_graph.py`](../../auto_ontology/retrieval/text_to_sql/text_to_sql_graph.py)（`create_graph()`）：

1. **意图与召回**：`question_intent`（分类普通查询 / 元数据信息查询 / 预测查询） → `question_extraction`（实体抽取） → `retrieve_candidates`（pgvector 向量检索候选 Term、列与 SQL 属性）。
2. **智能分流**：
   - 元数据类问题直接路由至 `information_agent` 结束；
   - 若启动时配置了 `KUMO_RFM_API_URL`，预测类问题在 `prepare_candidates` 后分支至 `prepare_prediction_graph` → `kumo_predict`；
   - 常规查询经 `prepare_candidates` 与可选 `refine_evidence` 后进入 `construct_sql_from_candidates`。
3. **多层校验与自愈闭环**：
   - `validate_sql_values`（可选值锚定校验） → `validate_sql_query`（SQLGlot 语法解析、静态检查与意图校验）；
   - `precheck_combined`（按环境变量开启合并探测：字面值、Join 路径、JSONB 路径，把多次探测合并为一次往返以降低修复开销）；
   - `execute_sql_query` 执行查询后，视结果情况经过 `check_empty_like_result` 与 `check_value_repair`（针对空结果的实时库值修复）；
   - 任一校验失败均流转至 `reconstruct_sql` 重写 SQL，硬性上限为 `MAX_RECONSTRUCTION_ATTEMPTS = 7` 次；超出上限或底层数据库不可达则流转至 `unconstructable_sql_response` 礼貌拒答，避免无限循环。

### 4. 主后端服务与并发隔离（`auto_ontology/server/`）

- **应用工厂与 Schema 门禁**（[`auto_ontology/server/__main__.py`](../../auto_ontology/server/__main__.py)）：通过 `create_app()` 注册了 15 组 `/api` 路由（拆分于测试与 OpenAPI 导出，无需启动数据库连接即可构建应用），实际服务启动时在 `lifespan` 中强制执行 `require_current_schema()` 校验数据库迁移版本。
- **子进程隔离与断线不中断**（[`auto_ontology/server/chat/router.py`](../../auto_ontology/server/chat/router.py)）：
  - 每次问答从 `WarmPool` 预热进程池获取独立子进程执行同步/阻塞的 LangGraph 图，从而实现可控的硬取消并消除冷启动耗时；
  - 以 `(user_id, conversation_id)` 控制单会话并发：活跃连接并发提问返回 `409`，若前端已断开（`client_alive` 被约 100ms 轮询的 ASGI disconnect watchdog 清除）则允许新请求抢占孤儿槽位；
  - 浏览器关闭或跳转不会中断进行中的推理：后台 `_pump` 继续消费输出，将回答及异步生成的图表持久化入库，客户端重连 `/chat/watch` 即可恢复查看。

### 5. 数据访问层、迁移与连接器（`auto_ontology/dal/` & `connectors/`）

- **双 Schema 隔离治理**：Postgres 中 `public` Schema 由 Python 侧 Alembic 管理（模型唯一来源为 [`auto_ontology/dal/schema.py`](../../auto_ontology/dal/schema.py)，采用 SQLAlchemy Core 而非 ORM），`frontend` Schema 由 Prisma 管理。
- **表命名与外键设计哲学**（[`schema.py`](../../auto_ontology/dal/schema.py)）：目录实体统一前缀 `catalog_` 避开 SQL 保留字冲突；关联关系表严格包含一个双下划线 `__`（如 `column__join`、`column__semantic_fk`、`column__has_attribute`）以消除与实体表的歧义；层层级联通过父级外键 `ON DELETE CASCADE` 实现单语句整库删除。
- **多引擎连接器与方言对齐**（[`auto_ontology/connectors/registry.py`](../../auto_ontology/connectors/registry.py)）：统一注册了 PostgreSQL、MySQL、Databricks、Snowflake、ClickHouse、Trino、Apache Kyuubi、HeavyDB、DuckDB、SQLite 共 10 种数据引擎，强制每个连接器声明有效的 `sqlglot` 方言（如 HeavyDB 映射为 `postgres`），并在日志输出前通过 `_redact` 脱敏密码、PAT 与大体积 truststore。

### 6. MCP 只读协议代理（`mcp/auto_ontology_mcp/`）

- [`server.py`](../../mcp/auto_ontology_mcp/server.py) 基于 `FastMCP` 与裁剪后的 OpenAPI 白名单动态挂载只读工具（`ask_question`、`check_answerable`、`check_readiness`、`search_terms`、`get_term` 等）。
- 本身无状态、不持数据库或模型密钥，每次请求携带用户 OAuth 不透明 Token 到 Next.js 的 OAuth UserInfo 端点实时核验权限，且在握手阶段向宿主注入工具使用顺序指引，防止智能体跳过语义探索直接盲目调用耗时较长的 `ask_question`。

---

## 三、中文说明文档全索引

本项目所有原版英文文档均已完整保留，并生成了对应的简体中文副本：

### 根目录文档
- 项目主说明：[`README.zh-CN.md`](../../README.zh-CN.md)（原文 [`README.md`](../../README.md)）
- Kubernetes / Helm 部署指南：[`DEPLOYMENT.zh-CN.md`](../../DEPLOYMENT.zh-CN.md)（原文 [`DEPLOYMENT.md`](../../DEPLOYMENT.md)）
- 智能体接入总入口：[`AGENTS.zh-CN.md`](../../AGENTS.zh-CN.md)（原文 [`AGENTS.md`](../../AGENTS.md)）
- 项目维护者开发规范：[`CLAUDE.zh-CN.md`](../../CLAUDE.zh-CN.md)（原文 [`CLAUDE.md`](../../CLAUDE.md)）
- 贡献指南：[`CONTRIBUTING.zh-CN.md`](../../CONTRIBUTING.zh-CN.md)（原文 [`CONTRIBUTING.md`](../../CONTRIBUTING.md)）
- 安全漏洞披露政策：[`SECURITY.zh-CN.md`](../../SECURITY.zh-CN.md)（原文 [`SECURITY.md`](../../SECURITY.md)）
- 社区行为准则：[`CODE_OF_CONDUCT.zh-CN.md`](../../CODE_OF_CONDUCT.zh-CN.md)（原文 [`CODE_OF_CONDUCT.md`](../../CODE_OF_CONDUCT.md)）

### `docs/zh-CN/` 技术参考文档
- 身份认证、SSO 与 API Token：[`authentication.md`](./authentication.md)（原文 [`docs/authentication.md`](../authentication.md)）
- 数据源连接与 Vault 存储：[`connections.md`](./connections.md)（原文 [`docs/connections.md`](../connections.md)）
- 对话 API 与 SSE 流规范：[`conversation-api.md`](./conversation-api.md)（原文 [`docs/conversation-api.md`](../conversation-api.md)）
- MCP 服务器深度文档：[`mcp.md`](./mcp.md)（原文 [`docs/mcp.md`](../mcp.md)）
- OpenAPI 规范生成与约定：[`openapi/README.md`](./openapi/README.md)（原文 [`docs/openapi/README.md`](../openapi/README.md)）
- 前端格式化 Cursor 规则：[`frontend-formatting-rule.md`](./frontend-formatting-rule.md)（原文 [`.cursor/rules/frontend-formatting.md`](../../.cursor/rules/frontend-formatting.md)）
- 容器镜像第三方声明：[`CONTAINER_THIRD_PARTY_NOTICES.md`](./CONTAINER_THIRD_PARTY_NOTICES.md)（原文 [`CONTAINER_THIRD_PARTY_NOTICES.md`](../../CONTAINER_THIRD_PARTY_NOTICES.md)）

### 子模块文档
- MCP 包说明：[`mcp/README.zh-CN.md`](../../mcp/README.zh-CN.md)（原文 [`mcp/README.md`](../../mcp/README.md)）
- 本地开发工具集：[`dev_tools/README.zh-CN.md`](../../dev_tools/README.zh-CN.md)（原文 [`dev_tools/README.md`](../../dev_tools/README.md)）
- SQL 测试夹具说明：[`dev_tools/fixtures/sql/README.zh-CN.md`](../../dev_tools/fixtures/sql/README.zh-CN.md)（原文 [`dev_tools/fixtures/sql/README.md`](../../dev_tools/fixtures/sql/README.md)）
- 前端应用说明：[`frontend/README.zh-CN.md`](../../frontend/README.zh-CN.md)（原文 [`frontend/README.md`](../../frontend/README.md)）
- 前端 Next.js 16 提示：[`frontend/AGENTS.zh-CN.md`](../../frontend/AGENTS.zh-CN.md)（原文 [`frontend/AGENTS.md`](../../frontend/AGENTS.md)）
- 前端开发规范：[`frontend/CLAUDE.zh-CN.md`](../../frontend/CLAUDE.zh-CN.md)（原文 [`frontend/CLAUDE.md`](../../frontend/CLAUDE.md)）
- 图表渲染组件：[`frontend/components/ResultChart/README.zh-CN.md`](../../frontend/components/ResultChart/README.zh-CN.md)（原文 [`frontend/components/ResultChart/README.md`](../../frontend/components/ResultChart/README.md)）

### Agent Skills 中文镜像（`docs/zh-CN/skills/`）
> 为保持 `skills/` 原目录下 NVSkills CI 签名校验（`BENCHMARK.md` 与 `skill.oms.sig`）纯净，所有技能中文文档均镜像存放于此目录：
- **部署与排错（`nvidia-ontology-setup`）**：[`SKILL.md`](./skills/nvidia-ontology-setup/SKILL.md)、[`skill-card.md`](./skills/nvidia-ontology-setup/skill-card.md)、[`troubleshooting.md`](./skills/nvidia-ontology-setup/references/troubleshooting.md)
- **建模与发布（`nvidia-ontology-management`）**：[`SKILL.md`](./skills/nvidia-ontology-management/SKILL.md)、[`skill-card.md`](./skills/nvidia-ontology-management/skill-card.md)、[`modeling.md`](./skills/nvidia-ontology-management/references/modeling.md)、[`publication.md`](./skills/nvidia-ontology-management/references/publication.md)、[`write-api.md`](./skills/nvidia-ontology-management/references/write-api.md)
- **查询与校验（`nvidia-ontology-query`）**：[`SKILL.md`](./skills/nvidia-ontology-query/SKILL.md)、[`skill-card.md`](./skills/nvidia-ontology-query/skill-card.md)、[`query-validation.md`](./skills/nvidia-ontology-query/references/query-validation.md)、[`stack.md`](./skills/nvidia-ontology-query/references/stack.md)

> **未翻译的特殊文件说明**：
> - `THIRD_PARTY_NOTICES.md`（根目录及 `frontend/`、`mcp/` 下三份哈希完全相同的副本）：为第三方开源软件包版本清单与英文法律许可协议索引，按合规惯例保留英文原文。
> - `skills/*/BENCHMARK.md`：由 NVSkills CI 机器人自动生成并受 `skill.oms.sig` 签名校验的评测基准日志，严禁手工改动。
