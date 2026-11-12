<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto Ontology（自动本体）

> 本文档为 [`README.md`](./README.md) 的简体中文翻译版。

智能体（Agent）可以检索文档和查询数据库，但**可靠性**才是制约智能体分析（Agentic Analytics）的核心瓶颈：如果算出来的营收指标本身是错的，回答得再快也毫无价值。智能体需要一个上下文层，告诉它数据存放在哪里、代表什么业务含义、是否应该使用、如何使用、数据从何而来，以及哪些概念彼此相似。NVIDIA Auto Ontology 是一个开源库，能够基于企业元数据构建这一上下文层、协调跨系统间不一致的定义，并通过 MCP、CLI 和 API 将其暴露给任意智能体框架，同时与客户已有的语义模型形成互补。

它能够帮助开发者实现以下能力：

- **发现（Discover）**：基于元数据自动生成企业本体；连接现有本体与语义模型；构建 AI 就绪的知识图谱。
- **锚定（Ground）**：为智能体提供指标、实体与关系的可信业务语义；统一跨系统的不一致定义。
- **校验（Validate）**：依据经过认证的定义与血缘关系校验生成的查询；随每个答案一并返回数据溯源信息。
- **暴露（Expose）**：通过 MCP、CLI 和 API 向任意智能体运行框架提供上下文服务。
- **持续学习（Learn）**：将经过认证的修正与使用反馈回流至本体中，使共享上下文随着每次提问不断演进优化。

## 基准测试（Benchmarks）

如需运行 Text-to-SQL 基准测试（例如 BIRD），请参阅 [ontology SQL evaluation 仓库](https://github.com/NVIDIA/ontology-sql-eval/)。

> **许可协议与贡献说明：** Auto Ontology 基于 [Apache License 2.0](./LICENSE) 协议分发。本项目打包、链接或使用的第三方开源组件列于 [`THIRD_PARTY_NOTICES.md`](./THIRD_PARTY_NOTICES.md)。**本项目目前暂不接受外部代码贡献（PR）。**

<p>
<img src="./docs/assets/arch.png" alt="Auto Ontology Architecture" width="800">
</p>

## 核心特性

- **结构化数据自然语言查询**：由 NVIDIA NIM 驱动，将自然语言问题翻译为 SQL 并在已连接的数据库上执行。
- **语义目录 / 本体层**：构建在关系型数据源之上的语义层，连同所描述的数据目录一并存储于 Postgres。
- **身份认证与 RBAC 权限控制**：基于 [Better Auth](https://github.com/better-auth/better-auth) 实现。
- **Web 可视化界面**：涵盖对话问答（Chat）、分析（Analysis）、数据目录（Data Catalog）、用量统计（Analytics）与系统设置（Settings）。
- **灵活部署**：支持基于 Helm Chart 的 Kubernetes 部署，或通过 Docker Compose 一键启动本地服务栈。

## 软件组件

| 组件 | 技术栈 | 职责 | 默认端口 |
| --- | --- | --- | --- |
| 前端（Frontend） | Next.js 16 (React 19, TypeScript, Tailwind CSS 4), Better Auth, Prisma | Web UI、身份认证、对外 API 网关 | 3000 |
| 后端（Backend） | FastAPI (Python 3.12+), LangChain | 对话 / NL-to-SQL 智能体、目录与数据源 API | 3001 |
| 摄取工作服务（Ingestion worker） | FastAPI (Python 3.12+) | 摄取表结构数据并生成向量嵌入（Embeddings） | 3002 |
| Postgres + pgvector | 关系型数据库 | 应用元数据、数据目录、本体与向量存储 | 5432 |
| HashiCorp Vault | 可选组件 | 安全存储数据源连接凭证 | — |

### NVIDIA NIM 配置

Auto Ontology 使用 NVIDIA NIM 端点进行推理——既可以使用托管的 [NVIDIA API Catalog](https://build.nvidia.com)（`integrate.api.nvidia.com`），也可以使用自托管（Self-hosted）的 NIM 服务。`.env.example` 中的默认配置如下：

- **大语言模型（LLM）：** `nvidia/nemotron-3.5-lightning-30b-a3b`
- **向量嵌入（Embeddings）：** `nvidia/nemotron-3-embed-1b`
- **重排序（Reranking）：** `nvidia/llama-nemotron-rerank-vl-1b-v2`

端点与模型通过 `DEFAULT_MODELS_ENDPOINT`、`DEFAULT_MODELS_MODEL`、`EMBED_ENDPOINT` 与 `EMBED_MODEL` 环境变量配置，并需要提供 `DEFAULT_MODELS_API_KEY`。

#### 模型三元组（Model triplets）

每个模型角色由一组环境变量**三元组**进行配置——`<PREFIX>_ENDPOINT`（服务 URL）、`<PREFIX>_API_KEY` 与 `<PREFIX>_MODEL`：

| 前缀 | 角色说明 |
| --- | --- |
| `DEFAULT_MODELS` | 共享默认配置；下方所有未显式设置的字段均会回退至此配置。 |
| `REASONING` | 主对话与 NL-to-SQL 推理模型。 |
| `NON_REASONING` | 用于实体抽取、意图判断等任务的轻量快速模型。 |
| `EMBED` | 文本嵌入模型（数据摄取与查询阶段必须保持一致）。 |
| `RERANK` | 检索流程中使用的重排序模型。 |

你可以为某个角色设置完整三元组、只设置其中部分字段，或完全不设置——任何留空的字段都会自动回退到对应的 `DEFAULT_MODELS_<FIELD>`。因此最简单的配置方式是填写 `DEFAULT_MODELS_*`，仅在需要差异化指定时覆盖对应字段（例如 `EMBED_MODEL`、`NON_REASONING_MODEL`）。为兼容旧版配置，`*_API_KEY` 也会继续回退到 `NVIDIA_API_KEY`。

## 快速开始（Quickstart）

以下指南使用 Docker Compose 运行 Auto-Ontology，并接入微软示例销售数据库 Wide World Importers。你需要预先安装支持 Compose v2 的 Docker、[uv](https://docs.astral.sh/uv/)，以及从 [build.nvidia.com](https://build.nvidia.com) 获取的 NVIDIA API Key（`nvapi-...`）。

1. 克隆仓库并创建 `.env` 文件：

   ```bash
   git clone https://github.com/NVIDIA/auto-ontology.git && cd auto-ontology
   cp .env.example .env
   ```

   随后在 `.env` 中设置以下配置项：

   ```dotenv
   DEFAULT_MODELS_API_KEY=nvapi-...
   AUTH_SECRET=<长随机字符串，例如通过 openssl rand -base64 32 生成>
   APP_URL=http://localhost:3000
   POSTGRES_PASSWORD=<新密码>
   CONNECTION_STRINGS=postgresql://postgres:<与上面相同的密码>@postgres:5432/wideworldimporters
   AUTO_ONTOLOGY_ADMIN_PASSWORD=<新密码>
   PGADMIN_PASSWORD=<新密码>
   ```

   Compose 会将 Postgres、pgAdmin 与 Web 应用端口暴露到宿主机上，因此启动前请务必替换所有默认密码。`.env.example` 中还为每个模型角色预设了占位密钥：请将每处 `nvapi-your-api-key-here` 替换为你的真实密钥，或直接删除这些行以自动回退至 `DEFAULT_MODELS_API_KEY`。

2. 启动 Postgres 并导入示例数据库。种子脚本位于 `auto-ontology-eval` 仓库中，运行时请传入你设置的 Postgres 密码：

   ```bash
   docker compose up -d --wait postgres
   git clone https://github.com/NVIDIA/auto-ontology-eval.git ../auto-ontology-eval
   (cd ../auto-ontology-eval && uv sync && cp .env.example .env && \
     POSTGRES_PASSWORD='<与上面相同的密码>' \
     uv run python scripts/seed/seed_wwi.py --database-name wideworldimporters --drop)
   ```

3. 构建并启动其余服务栈：

   ```bash
   docker compose up -d --build
   ```

4. 打开 <http://localhost:3000>，使用 `.env` 中配置的 `AUTO_ONTOLOGY_ADMIN_EMAIL` 和 `AUTO_ONTOLOGY_ADMIN_PASSWORD` 登录。进入 **Settings → Semantic Compilation**（设置 → 语义编译）并开启编译。服务启动时会自动运行数据摄取；开启后语义编译会立即启动，完成后状态会显示为 **Semantic compilation is ready**。

5. 在 **Chat** 中提出问题：

   > What was revenue for the USB food flash drive - sushi roll in Pennsylvania in June 2015?

   智能体会将 *revenue*、*product* 和 *state* 解析到本体中的业务术语（Terms），沿关系推导所需的表连接路径并执行 SQL。返回结果会同时附带生成的 SQL 和执行步骤，方便你与源数据核对：

   ```text
   Answer:  Revenue for USB food flash drive - sushi roll in Pennsylvania in
            June 2015 was $588.80, from 3 invoice lines.

   SQL:     SELECT SUM(il.extendedprice) AS revenue, COUNT(*) AS invoice_lines
            FROM sales.invoicelines il
            JOIN sales.invoices i             ON il.invoiceid = i.invoiceid
            JOIN warehouse.stockitems si      ON il.stockitemid = si.stockitemid
            JOIN sales.customers c            ON i.customerid = c.customerid
            JOIN application.cities ct        ON c.deliverycityid = ct.cityid
            JOIN application.stateprovinces sp ON ct.stateprovinceid = sp.stateprovinceid
            WHERE si.stockitemname = 'USB food flash drive - sushi roll'
              AND sp.stateprovincename = 'Pennsylvania'
              AND i.invoicedate >= '2015-06-01' AND i.invoicedate < '2015-07-01'

   Steps:   Resolved terms: Invoice Line (revenue), Stock Item, State Province, Invoice
            Joined: invoice line -> invoice -> customer -> city -> state province
            Validated the filter values against the data, then executed
   ```

6. 尝试询问一个当前数据无法回答的问题。Wide World Importers 不包含任何广告投放数据，因此关于广告支出的问题在本体中找不到对应定义。通过 [MCP](#mcp-服务器) 接入的智能体会先检查可回答性（Answerability），然后主动向你澄清：

   ```text
   You:    What was return on ad spend for the sushi roll flash drive in June 2015?
   Agent:  check_answerable -> coverage 0.5; no term covers "ad spend"
   Agent:  I can find revenue for that product, but this data has no
           advertising spend, so I can't compute return on ad spend. Do you
           want June 2015 revenue instead, or is ad spend tracked in another
           source I should connect?
   ```

如果重启后 UI 加载时缺少左侧导航栏，通常是因为浏览器保留了过期的会话 Cookie。请清除 `localhost:3000` 的站点数据后重新登录。其他启动与配置故障排查请参见[安装技能的故障排查指南](./docs/zh-CN/skills/nvidia-ontology-setup/references/troubleshooting.md)（英文原版：[troubleshooting.md](./skills/nvidia-ontology-setup/references/troubleshooting.md)）。

## 工作原理

<p>
<img src="./docs/assets/arch.png" alt="Auto-Ontology architecture" width="800">
</p>

Auto-Ontology 分三个阶段运行：**自动生成本体**、**人工审核认证**，以及**智能体基于本体回答问题**。

本体中保存了三类核心定义：

- **业务术语（Terms）**：如 *Customer*（客户）或 *Invoice Line*（发票明细）等业务概念，附带描述与同义词，并映射到实际承载它们的物理列。
- **SQL 属性（SQL attributes）**：归属于某个术语下经过审核的 SQL 表达式，例如利润率或滚动总计。智能体会复用这些表达式，确保所有回答采用统一口径计算指标。
- **语义关系（Relationships）**：表之间（包括桥接表/关联表）的连接方式，可基于主外键及数据特征自动推断。

**1. 生成阶段（Generate）：** 摄取服务会基于源数据库生成初始本体草稿：

1. **读取数据源**：提取 Schema 结构、列画像以及样本值并写入数据目录。
2. **实体命名**：逐表抽取业务术语，并将其映射到具体列。
3. **关系解析**：推断表间连接关系与桥接表，并结合查询历史推荐 SQL 属性。
4. **写入本体**：将术语、关系与 SQL 属性存入 Postgres，同时在 `pgvector` 中生成向量索引供检索使用。

生成的结果是一张可在 Web UI 中交互探索的语义图谱。在以下示例中，对 Wide World Importers 进行一次编译共生成了 31 个术语；选中 **Order** 即可查看其描述、关联术语、映射列及 SQL 属性：

<p>
<img src="./docs/assets/ontology-graph.png" alt="Semantic graph of the Wide World Importers ontology, with the Order term selected" width="800">
</p>

**2. 认证阶段（Certify）：** 由业务或数据人员让草稿变得完全可信：

1. 在 Chat 页面用真实业务问题进行测试，或通过 MCP 调用 `check_answerable` 和 `ask_question` 验证。
2. 在 UI 中或通过 API 调整术语名称、描述与 SQL 属性。
3. 确认无误后，对术语名称、描述和 SQL 属性进行“认证（Certify）”标记。

**3. 问答阶段（Answer）：** 面对每个用户问题，智能体会依次执行：

1. 对问题进行意图分类，并抽取问题中提及的业务实体。
2. 通过向量检索召回候选术语、列与 SQL 属性。
3. 基于候选上下文生成 SQL。
4. 针对目标数据库校验过滤条件值与 SQL 合法性；若校验失败，自动触发 SQL 修复重构。
5. 执行查询并返回自然语言答案、SQL 语句与推理步骤。

当候选语义无法支撑查询时，智能体会明确告知该问题无法从当前数据中得到答案。若配置了 KumoRFM，预测类问题会路由至 KumoRFM 处理（详见[预测功能](#预测功能predictions)）。

在 Chat 界面中，每条回答都会展示其生成的 SQL，你可以查看数据行、导出 CSV 或查看可视化图表：

<p>
<img src="./docs/assets/qa-example.png" alt="Chat answering which products drive the most revenue and highest customer count, with the generated SQL and a revenue chart" width="550">
</p>

Auto-Ontology 兼容语义模型开放交换标准 [Apache Ossie](https://ossie.apache.org/)。Ossie 提供了 [Auto-Ontology 转换器](https://github.com/apache/ossie-converters/tree/main/nvidia)，Auto-Ontology 借助它将模型定义导入和导出为 Ossie YAML 格式。可通过界面 **Settings → Import / Export**，或调用 `POST /api/model/import` 与 `POST /api/model/export` 使用。

| 组件 | 技术栈 | 职责 | 端口 |
| --- | --- | --- | --- |
| 前端（Frontend） | Next.js, Better Auth, Prisma | Web UI、身份认证、公开 API 网关 | 3000 |
| 后端（Backend） | FastAPI (Python 3.12+), LangChain | Text-to-SQL 智能体、目录与本体 API | 3001 |
| 摄取服务（Ingestion service） | FastAPI (Python 3.12+) | 数据摄取与语义编译 | 3002 |
| MCP 服务器 | FastMCP | 基于公开 API 的只读智能体工具集 | 3003 |
| Postgres + pgvector | PostgreSQL 17 | 目录、本体、应用数据与向量存储 | 5432 |
| HashiCorp Vault | 可选 | 连接凭证安全存储 | 不适用 |

请务必将 `3001` 和 `3002` 端口部署在私有网络内。FastAPI 会直接信任前端网关通过 `x-auto-ontology-user-id` 请求头转发的用户身份，而摄取服务的接口（ingest、delete、reset、compile）本身不带独立认证。所有外部客户端必须统一经由 `3000` 端口的 Web 应用访问。

## 从智能体中接入使用

### MCP 服务器

[MCP](https://modelcontextprotocol.io) 服务器为任意 MCP 客户端提供针对本体的只读工具。每位用户通过 Auto-Ontology 完成登录授权，每次工具调用均以该用户自身的权限执行。

| 用途 | 工具名称 |
| --- | --- |
| 就绪检查 | `check_readiness`：判断当前部署是否已具备回答问题的条件 |
| 术语探索 | `search_terms`、`get_term`、`get_term_columns`、`get_term_sql_attributes`、`get_sql_attribute`、`describe_table` |
| 可回答性检查 | `check_answerable`：判断当前本体是否覆盖该问题 |
| 提问查询 | `ask_question`：返回自然语言回答、生成的 SQL 与结果行 |

针对你的部署启动 MCP 服务器：

```bash
AUTO_ONTOLOGY_API_URL=http://localhost:3000 \
  uvx --from "git+https://github.com/NVIDIA/auto-ontology.git#subdirectory=mcp" auto-ontology-mcp
```

在 Cursor（`~/.cursor/mcp.json`）中配置：

```json
{
  "mcpServers": {
    "auto-ontology": { "url": "http://localhost:3003/mcp" }
  }
}
```

在 Claude Desktop（`claude_desktop_config.json`）中通过 [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) 桥接配置：

```json
{
  "mcpServers": {
    "auto-ontology": {
      "command": "npx",
      "args": ["mcp-remote", "http://localhost:3003/mcp"]
    }
  }
}
```

修改配置后请重启客户端，并在客户端提示时完成登录。详细登录流程、配置项及排错说明请参阅 [docs/zh-CN/mcp.md](./docs/zh-CN/mcp.md)（英文原版：[docs/mcp.md](./docs/mcp.md)）。

### 智能体技能（Agent skills）

仓库内置了三个 Agent Skill，用于指导编码智能体操作 Auto-Ontology。可从 [AGENTS.zh-CN.md](./AGENTS.zh-CN.md) 开始了解。

| 技能 | 覆盖范围 |
| --- | --- |
| [`nvidia-ontology-setup`](./skills/nvidia-ontology-setup/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-setup/SKILL.md)） | 安装部署、连接数据源、编译本体、验证就绪状态 |
| [`nvidia-ontology-management`](./skills/nvidia-ontology-management/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-management/SKILL.md)） | 检视、建模、认证、导入/导出及治理发布 |
| [`nvidia-ontology-query`](./skills/nvidia-ontology-query/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-query/SKILL.md)） | 发起查询并校验 SQL、返回行与答案准确性 |

### REST API

可通过 Web 应用源站的 `POST /api/chat/completions` 接口发起提问。请在右上角用户菜单 → **API Tokens** 中生成 API Token。接口返回服务器推送事件流（SSE）：其中 `result` 事件携带最终答案，`data: [DONE]` 表示流结束。

```bash
curl -N http://localhost:3000/api/chat/completions \
  -H "x-api-key: $AUTO_ONTOLOGY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "How many orders shipped last week?"}'
```

```python
import json
import os

import requests

response = requests.post(
    "http://localhost:3000/api/chat/completions",
    headers={"x-api-key": os.environ["AUTO_ONTOLOGY_API_TOKEN"]},
    json={"question": "How many orders shipped last week?"},
    stream=True,
    timeout=900,
)
response.raise_for_status()

for line in response.iter_lines(decode_unicode=True):
    if not line or not line.startswith("data:"):
        continue
    payload = line.removeprefix("data:").strip()
    if payload == "[DONE]":
        break
    event = json.loads(payload)
    if event["type"] == "result":
        print(event["answer"]["response"])
        print(event["answer"]["sql_code"])
    elif event["type"] == "error":
        raise RuntimeError(event["message"])
```

事件类型、多轮对话及网关信任边界说明请参阅 [docs/zh-CN/conversation-api.md](./docs/zh-CN/conversation-api.md)。其他所有接口定义均在 [OpenAPI 规范](./docs/openapi/auto-ontology-api.json) 中。

## 连接器、模型与预测

### 数据源连接器（Sources）

Auto-Ontology 支持连接以下十种数据引擎：

- PostgreSQL
- MySQL
- Databricks
- Snowflake
- ClickHouse
- Trino
- Apache Kyuubi
- HeavyDB
- DuckDB
- SQLite

可在 **Settings → Connections** 中添加连接，或通过 `CONNECTION_STRINGS` 环境变量配置。其中 DuckDB 与 SQLite 仅支持通过连接字符串配置。关于 URL Scheme、作用域过滤与凭证存储详情，请参阅 [docs/zh-CN/connections.md](./docs/zh-CN/connections.md)。

### 模型角色（Models）

Auto-Ontology 使用四种模型角色。配置 `nvapi-` 密钥后，各角色默认指向 [build.nvidia.com](https://build.nvidia.com) 上的 NVIDIA NIM：

| 角色 | 默认模型 | 职责 |
| --- | --- | --- |
| `REASONING` | `nvidia/nemotron-3.5-lightning-30b-a3b` | 构建本体（术语、关系、SQL 属性）、编写每个回答的 SQL 与解释，并评估可回答性 |
| `NON_REASONING` | `nvidia/nemotron-3.5-lightning-30b-a3b` | 执行智能体中的轻量快速步骤：问题意图识别、实体抽取、候选筛选、SQL 校验与修复 |
| `EMBED` | `nvidia/nemotron-3.5-embed-1b` | 在 pgvector 中为表、列、术语和 SQL 属性建立索引，并将问题与之匹配 |
| `RERANK` | `nvidia/nemotron-3.5-rerank-8b` | 在编译阶段构建桥接表 SQL 属性时对候选表进行重排序 |

如需使用自托管 NIM 或任意 OpenAI 兼容端点，请设置 `<ROLE>_ENDPOINT`、`<ROLE>_API_KEY` 与 `<ROLE>_MODEL`。未设置的字段会回退至 `DEFAULT_MODELS_<FIELD>`，因此最简配置只需填好 `DEFAULT_MODELS_*` 并覆盖差异项。请确保所用模型支持结构化输出（Structured Output），并在数据摄取和查询阶段保持嵌入模型一致。所有环境变量均在 [`.env.example`](./.env.example) 中有详细说明。

### 预测功能（Predictions）

配置 `KUMO_RFM_API_KEY`（及 `KUMO_RFM_API_URL`）后，智能体会将预测类问题路由至 KumoRFM 处理。KumoRFM 为托管服务：为了完成预测回答，智能体会将已连接的表数据上传至该服务，上传规模受 `KUMO_MAX_TABLES`（默认 20）和 `KUMO_MAX_ROWS_PER_TABLE`（未设置时默认不限）限制。在启用此功能前，请仔细评估上传内容并完成组织内部的数据合规审查。

## 集成与合作

如需将智能体平台、应用或存储引擎与 Auto-Ontology 集成，请提交 [Integration Request](https://github.com/NVIDIA/auto-ontology/issues/new?template=integration.yml)。

Auto-Ontology 由维护团队负责开发，目前不接受外部 Pull Request。欢迎通过 GitHub Issues 提交集成需求与缺陷报告。安全漏洞请遵循 [SECURITY.zh-CN.md](./SECURITY.zh-CN.md) 流程上报。维护者开发工作流请见 [CONTRIBUTING.zh-CN.md](./CONTRIBUTING.zh-CN.md)。

## 参考文档

| 主题 | 文档 |
| --- | --- |
| 基于 Helm 的 Kubernetes 部署 | [DEPLOYMENT.zh-CN.md](./DEPLOYMENT.zh-CN.md)（[英文](./DEPLOYMENT.md)） |
| 数据源连接、作用域与 Vault | [docs/zh-CN/connections.md](./docs/zh-CN/connections.md)（[英文](./docs/connections.md)） |
| 身份认证、SSO 与 API Token | [docs/zh-CN/authentication.md](./docs/zh-CN/authentication.md)（[英文](./docs/authentication.md)） |
| 对话 API（Conversation API） | [docs/zh-CN/conversation-api.md](./docs/zh-CN/conversation-api.md)（[英文](./docs/conversation-api.md)） |
| MCP 服务器 | [docs/zh-CN/mcp.md](./docs/zh-CN/mcp.md)（[英文](./docs/mcp.md)） |
| REST API（OpenAPI 规范） | [docs/openapi/auto-ontology-api.json](./docs/openapi/auto-ontology-api.json) |

## 开源协议（License）

Auto-Ontology 基于 [Apache License, Version 2.0](./LICENSE) 许可协议开源。SPDX 标识符：`Apache-2.0`。

本仓库中所有由 NVIDIA 编写的源文件均包含如下格式的 SPDX 声明头：

```text
SPDX-FileCopyrightText: Copyright (c) 2024-2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
```

Auto-Ontology 所使用的第三方开源组件及其上游协议与项目地址均列于 [`THIRD_PARTY_NOTICES.md`](./THIRD_PARTY_NOTICES.md)。容器镜像中额外引入的组件另行记录于 [`CONTAINER_THIRD_PARTY_NOTICES.md`](./CONTAINER_THIRD_PARTY_NOTICES.md)。

## 声明与免责声明（Notice and Disclaimer）

注意与免责声明（NOTICE AND DISCLAIMER）：本软件会在运行时自动检索、访问或与外部资源交互。这些外部检索到的资源并不随本软件一同分发，且完全受其各自独立的条款、条件与许可协议约束。您有责任自行查找、审阅并遵守所有适用的条款、条件与许可协议，并针对您的具体使用场景验证任何外部资源的安全性、完整性与适用性。本软件按“现状”（"AS IS"）提供，不附带任何形式的明示或暗示保证。作者不对任何外部检索资源作出陈述或保证，亦不对因您使用或无法使用本软件或任何外部资源而导致的任何损失、损害、责任或法律后果承担任何赔偿责任。使用本软件及相关外部资源的风险由您自行承担。

Auto-Ontology 在运行时交互的外部资源包括：

- 您配置的模型端点（默认为 NVIDIA API Catalog：`integrate.api.nvidia.com`、`ai.api.nvidia.com`），适用各服务商与模型的使用条款。
- KumoRFM 服务（仅在配置 `KUMO_RFM_API_KEY` 时启用），适用 Kumo 的服务条款。
- 您自己的源数据库，适用对应数据库系统的使用协议。

## 安全（Security）

如发现安全漏洞，请严格按照 [SECURITY.zh-CN.md](./SECURITY.zh-CN.md) 中的说明渠道进行上报。
