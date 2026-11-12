<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto-Ontology MCP 服务器

> 本文档为 [`docs/mcp.md`](../mcp.md) 的简体中文翻译版。

让任何支持 MCP（[Model Context Protocol](https://modelcontextprotocol.io)）的智能体——例如 Cursor、Claude Desktop 或企业内部智能体——能够使用自然语言查询您的结构化数据，并检视支撑回答的语义层定义。

## 工作原理

该 MCP 服务器本质上是 Auto-Ontology 公开 API 的一个 HTTP 客户端。它本身不保存任何数据库连接凭证、不持有模型配置，也不存储任何静态认证凭据：

```text
智能体框架  ──MCP──▶  auto-ontology-mcp  ──HTTPS──▶  Next.js (公开 API 网关)
                                                         │
                                                         ▼
                                                 FastAPI, Postgres
```

由于身份认证与权限校验已经统一由 Next.js 层负责，MCP 服务器直接继承这些能力而无需重复实现。这带来两个重要特性：第一，它可以部署在任意能够访问您的 Auto-Ontology 部署的网络位置；第二，**每个会话能且仅能执行登录用户自身权限允许的操作**——例如观察者（Viewer）只能获取观察者权限范围内的回答。

## 运行方式

只需运行一个 MCP 服务实例，不同用户各自通过它完成登录即可。唯一必须配置的参数是您的 Auto-Ontology 部署 URL：

```sh
AUTO_ONTOLOGY_API_URL=https://auto_ontology.example.com auto-ontology-mcp
```

使用 `uvx` 可直接从仓库拉取、构建并运行，无需手动克隆仓库：

```sh
uvx --from "git+https://github.com/NVIDIA/auto-ontology.git#subdirectory=mcp" auto-ontology-mcp
```

> [!NOTE]
> 目前 MCP 软件包需从 GitHub 安装。只有在正式发布至 PyPI 后，单独执行 `uvx auto-ontology-mcp` 才能直接解析。

## 连接客户端

在客户端中配置 MCP 服务器 URL 即可，配置文件中无需填写任何凭证：

```json
{
  "mcpServers": {
    "auto-ontology": {
      "url": "https://auto-ontology-mcp.example/mcp"
    }
  }
}
```

请注意 URL 末尾的 `/mcp` 后缀：它是真实的 MCP 端点路径，而非服务根路径。修改配置后请重启客户端，因为大多数客户端仅在启动时加载 MCP 配置。

客户端检测到服务器需要授权后，会主动提示用户登录（例如 Cursor 会将其标记为需要登录）。用户将看到 Auto-Ontology 的常规登录页面（支持该部署所配置的任意 SSO 登录方式），登录后客户端会自行托管获取到的 Token。整个过程无需手动生成或粘贴密钥，后续每次工具调用均以登录用户的身份执行。

随后即可尝试提问，例如：*"what does Auto-Ontology mean by an active customer, and how many were there last quarter?"*（“在 Auto-Ontology 中‘活跃客户’是如何定义的？上季度有多少位？”）

## 登录与授权机制

Auto-Ontology 本身充当 OAuth 授权服务器（Authorization Server）。MCP 服务器不持有 `client_id`、`client_secret` 或预设的 `redirect_uri`，因此无需针对不同部署逐一配置，也无需在身份提供商中手动注册客户端——MCP 客户端会自动向 Auto-Ontology 完成动态注册。

客户端注册信息与授权记录均保存在 Auto-Ontology 的数据库中，因此多副本实例可以共享状态，服务重启也不会导致用户掉线。权限校验在每次调用时实时从用户账号读取而非固化在 Token 中，这意味着修改用户角色、封禁账号或撤销授权都会立即生效。

由于签发的是不透明 Token（Opaque Token），MCP 服务器会在每次请求时调用 Auto-Ontology 的 OAuth UserInfo 端点进行校验。该端点使用 Bearer Token 认证，无需 Client Secret，并会在每次调用时检查过期时间、吊销状态、客户端状态及关联浏览器会话状态。授权范围必须包含 `openid`（Better Auth 的 UserInfo 端点要求）；Auto-Ontology 会在授权服务器元数据中声明该范围，从而使读取 `scopes_supported` 的客户端自动请求它。如果网络故障导致无法连接 Auto-Ontology，调用会直接返回错误而非诱导客户端“重新登录”，避免在服务不可用时让所有用户陷入无法成功的登录循环。

该服务刻意不提供配置全局服务端 Token 的方式。一旦允许配置单一全局 Token，所有调用者都将以该 Token 拥有者的身份、权限和对话历史运行，且协议层无法感知这种越权混淆。

## 指向本地运行的 Auto-Ontology

只需更改 `AUTO_ONTOLOGY_API_URL`。若使用本地 Docker Compose 部署，Web 应用运行在 `:3000` 端口：

```sh
AUTO_ONTOLOGY_API_URL=http://localhost:3000 auto-ontology-mcp
```

MCP 登录要求部署的 Auto-Ontology 版本支持作为 OAuth 授权服务器，较旧的预构建镜像可能尚未包含此能力。建议先检查以下端点，以免后续表现为客户端无法登录：

```sh
curl -s -o /dev/null -w '%{http_code}\n' $AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server
```

返回 `200` 即表示正常。若返回其他状态码，请从当前代码检出目录启动前端（`pnpm dev`），并将 `AUTO_ONTOLOGY_API_URL` 指向对应端口。

## 工具清单（Tools）

此处暴露的所有工具均为**只读**操作。任何工具都不会修改业务术语表、数据目录或底层源数据库。

| 工具名称 | 用途说明 |
| --- | --- |
| `ask_question` | **核心主工具。** 使用自然语言提问；返回自然语言答案、Auto-Ontology 实际执行的 SQL 及结果行。 |
| `check_answerable` | 评估当前语义层是否覆盖该问题。在调用 `ask_question` 前的低成本预检查。 |
| `check_readiness` | 检查当前部署是否已经具备回答任何问题的基本条件。 |
| `search_terms` | 搜索业务术语表。 |
| `get_term` | 查看单个业务术语的详细定义：描述、同义词、关联术语。 |
| `get_term_columns` | 查询某个术语映射到的底层物理列。 |
| `get_term_sql_attributes` | 获取某个术语下定义的 SQL 属性列表。 |
| `get_sql_attribute` | 查看单个 SQL 属性的表达式及业务用途说明。 |
| `describe_table` | 一并查看某张表的列信息、关联术语以及参与的 SQL 属性。 |

设计上刻意**不提供**直接浏览原始数据库、Schema 或物理列的工具。调用方应当通过语义层理解数据，因为暴露原始目录浏览器会诱导智能体绕过语义层直接基于裸表猜 SQL。`describe_table` 已能满足合法的探查需求，它返回的是该表所参与的业务术语和 SQL 属性。

此外，MCP 服务器在初始化握手阶段还会主动向宿主声明 **instructions（使用指引）**，明确工具的推荐调用顺序，避免模型仅凭工具名自行猜测——如果不加引导，模型往往会直接跳到耗时昂贵的 `ask_question`。

### 关于 `ask_question`

该工具底层调用与 Web UI 相同的 `POST /api/chat/completions` 接口，运行完整的 Text-to-SQL 智能体：包含多步串行大模型推理，耗时通常在数十秒左右。与 Web UI 接收 Markdown 不同，MCP 工具会将答案正文、SQL 语句和结果行拆分为独立字段返回，方便智能体直接使用 SQL 而无需从文本中解析。结果集上限为 100 行，并通过 `row_count` 和 `truncated` 字段汇报是否存在截断。运行过程中，工具还会持续发送 MCP 进度通知（Progress Notifications），与 Web UI 展示的推理步骤保持同步。

### 关于 `check_readiness`

回答一个问题依赖于两个彼此独立的条件：一是用于语义解析的**已编译语义层**，二是用于执行 SQL 的**可用源数据库连接**。缺一而存另一是该部署最容易产生误导的状态——表面看一切健康（术语表能查、业务词能解析），但每个问题都会在消耗数分钟模型调用后以模糊的错误告终。`check_readiness` 能够提前识别这种状态，并一次性列出所有阻塞项。

需要注意的是，即使 `databases` 非空，`can_execute_sql` 仍可能为 `false`：因为数据目录的生命周期独立于连接，数据源删除连接后目录仍可能保留，因此有库名并不代表可以执行查询。此外，读取连接列表默认仅限管理员权限，观察者（Viewer）调用该检查时可能收到 `403`，但聊天与目录读取仍可正常使用——这种情况既非部署损坏也不是凭据错误，因此工具会将其归入 `unverified` 字段，并基于其余可核验项给出综合就绪结论。

## 环境变量配置

| 变量名 | 默认值 | 含义说明 |
| --- | --- | --- |
| `AUTO_ONTOLOGY_API_URL` | `http://localhost:3000` | Auto-Ontology Web 应用基础 URL。 |
| `AUTO_ONTOLOGY_MCP_PUBLIC_URL` | `http://localhost:3003` | MCP 服务的公网访问 Origin（不含 `/mcp` 后缀）。前端也必须配置相同值，以确保签发 Token 中的资源 URL 与客户端访问地址一致。 |

较少需要调整的可选变量：`AUTO_ONTOLOGY_MCP_HOST` 与 `AUTO_ONTOLOGY_MCP_PORT`（监听地址与端口，默认 `0.0.0.0:3003`）、`AUTO_ONTOLOGY_MCP_TIMEOUT_S` 与 `AUTO_ONTOLOGY_MCP_CHAT_TIMEOUT_S`（普通请求与问答请求超时，默认 `30` 与 `900` 秒），以及 `AUTO_ONTOLOGY_OPENAPI_SPEC`（覆盖用于生成工具的 OpenAPI 规范路径）。

## 扩展工具集合

暴露给 MCP 的工具集合是在 [`mcp/auto_ontology_mcp/tools.py`](../../mcp/auto_ontology_mcp/tools.py) 中通过显式白名单管理的。完整的 OpenAPI 规范共包含 87 个接口操作，全部暴露会严重劣化大模型的工具选择准确率。如需新增工具，请在白名单中添加一条 `ToolSpec`，指定 HTTP 方法、路径、面向智能体的工具名以及说明*何时应当选用该工具*的描述。若某个白名单条目在 OpenAPI 规范中已不存在，服务启动时会显式报错失败，防止上游接口重命名导致工具静默丢失。

工具的参数模式与描述均来自 [`auto-ontology-api.json`](../openapi/auto-ontology-api.json)，该文件在仓库中提交了两份副本：规范主本位于 `docs/openapi/`，另一份打包在 MCP Python 包目录内（确保即使在没有 `docs/` 目录的独立安装环境下也能正常启动）。运行 `pnpm openapi` 会同时更新这两份文件，CI 也会校验二者一致性，因此**切勿手工修改包内的副本**。

## 故障排查（Troubleshooting）

**客户端无法发现如何登录** —— 检查 `$AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server` 是否返回合法 JSON。若返回向登录页的重定向，说明当前部署版本过旧，尚未包含该路由。

**"Protected resource ... does not match expected ..." 报错** —— 说明 MCP 服务器对外宣称的绑定地址与客户端实际访问的主机名拼写不一致，最常见于 `127.0.0.1` 与 `localhost` 混用。请将 `AUTO_ONTOLOGY_MCP_PUBLIC_URL` 设为客户端实际使用的 URL，或取消设置 `AUTO_ONTOLOGY_MCP_HOST`。

**"This request carried no signed-in session" 报错** —— 当前请求关联的 OAuth 授权已过期或在 Auto-Ontology 中被吊销。请在 MCP 客户端重新发起登录。

**`ask_question` 返回空答案，既无 SQL 也无报错** —— 运行虽然完成，但检索阶段未找到可用于构建查询的候选定义。请先调用 `check_readiness`：多数情况是因为未配置任何数据库连接，导致无论怎么提问都无法成功。若部署已就绪，则可能是提问用词与术语表不匹配——请先用 `search_terms` 查询问题中的核心名词。

**"Auto-Ontology cannot answer right now" 提示** —— 通常是语义层从未编译过；请用 `check_readiness` 确认。如果同一个 `conversation_id` 上已有正在进行中的对话轮次，也会触发此提示。

**客户端中看不到任何 MCP 工具** —— 请检查客户端的 MCP 运行日志（MCP 服务器日志输出到 `stderr`），并确认配置中的 URL 已包含 `/mcp` 后缀且修改配置后已重启客户端。
