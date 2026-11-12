---
name: nvidia-ontology-query
description: >-
  查询 Auto Ontology 并校验生成的 SQL、返回行与答案。适用于 MCP 或 REST 访问、就绪检查、身份认证、多轮对话与可信问答。
license: Apache-2.0
metadata:
  version: "0.2.1"
  author: "NVIDIA <opensource@nvidia.com>"
  tags:
    - nvidia-ontology
    - mcp
    - api
    - agents
    - aiq
---

<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto Ontology 锚定式查询（Grounded queries）

> 本文档为 [`skills/nvidia-ontology-query/SKILL.md`](../../../../skills/nvidia-ontology-query/SKILL.md) 的简体中文翻译版。

## 目的

在不限定具体运行框架、SDK 或 LangGraph 模板的前提下，从智能体中调用当前版本的 Auto Ontology。类型化的工作流产物、状态、门禁与交接契约定义在 [runtime-contract.yaml](../../../../skills/nvidia-ontology-query/assets/runtime-contract.yaml) 中。

## 前置条件

需要一个正在运行的 Auto Ontology 部署以及经过身份认证的 MCP 或 REST 客户端。若 Auto Ontology 尚未运行，请先使用 `nvidia-ontology-setup`。

## 局限性

本技能仅负责**读取查询并校验答案**。它不直接暴露 FastAPI，不修改语义定义，不发布受控结果，也不提供原始 Schema 浏览器。所有变更与发布任务请交接给 `nvidia-ontology-management`。

## 操作指引

1. 在提问前先调用就绪检查（Readiness）。
2. 通过语义层发现业务术语并确认问题可回答性。
3. 优先通过 MCP 提问；若不可用，再回退至经过认证的 Next.js REST 网关。
4. 在向用户呈现任何带业务断言（Claim-bearing）的答案前，先校验实质意图、SQL、返回行、截断标记与正文描述。

## 优先使用 MCP 进行读取

当智能体框架支持 MCP 时，优先使用 MCP。**握手阶段返回的工具列表**是实时唯一事实来源（`mcp/auto_ontology_mcp/tools.py`）。不要把工具清单静态复制到会话上下文中；若握手列表与本技能不一致，以实时握手结果为准。

推荐调用顺序（MCP 服务端也会在握手提示词中广播此顺序，以防模型直接跳到 `ask_question`）：

1. `check_readiness` —— 同时确认已编译的语义层**以及**可用的源数据库连接。
2. `search_terms` —— 了解业务名词在当前部署中的具体含义。
3. `check_answerable` —— 低成本覆盖率预检。
4. `ask_question` —— 完整 Text-to-SQL 推理；耗时数十秒；返回答案 + SQL + 结果行（最多 100 行，附带 `row_count` / `truncated` 字段）。
5. 对于需要严谨决策的答案，请使用 [query-validation.md](references/query-validation.md) 逐项校验实质意图、生成的 SQL、返回数据行、截断情况与自然语言正文。

MCP 的全部工具均为**只读**。设计上刻意不提供浏览原始数据库、Schema 或物理列的工具。`describe_table` 是合法的表级视图：它返回表所参与的业务术语与 SQL 属性。

连接方式：`AUTO_ONTOLOGY_API_URL` 指向 **Web 应用**。启动 `auto-ontology-mcp`，将客户端指向 `…/mcp` 并由用户登录。MCP 服务端不配置全局 Token。详情参见仓库中的 `mcp/README.zh-CN.md` 与 `docs/zh-CN/mcp.md`。

## REST 兜底方案与写入交接

当 MCP 不可用时，外部客户端应调用经过身份认证的 **Next.js 网关**，切勿直连 FastAPI `:3001`。FastAPI 直接信任 `x-auto-ontology-user-id` 请求头，绝不能暴露在公网。直接请求 FastAPI 且不带 `conversation_id` 为无状态调用；若携带 `conversation_id` 则必须附带该内部请求头。

身份认证（支持以下任一方式，在统一入口解析）：

- 浏览器会话 Cookie
- API Token：`x-api-key: $AUTO_ONTOLOGY_API_TOKEN`（`Authorization: Bearer` 同样识别 `auto_ontology_…` 前缀 Token）。Token 继承创建者权限。
- Auto Ontology 签发的 OAuth Bearer Token（MCP 登录）
- SSO ID Token（`Authorization: Bearer <jwt>`），例如 AI-Q —— 详见 [stack.md](references/stack.md)

接口数据结构详见 `docs/openapi/auto-ontology-api.json`（无需全量抓取全部约 87 个端点）。一旦遇到写入需求，请交接给 `nvidia-ontology-management`，不要把只读查询擅自变成数据修改操作。

### 对话接口（Chat）

`POST /api/chat/completions` —— 返回 SSE 流（`step` / `result` / `error` / `charts`），以 `[DONE]` 结束。需要 `chat:use` 权限；若请求中携带 `conversation_id`，还需 `conversation:write` 权限，否则返回 `403`。

- 省略 `conversation_id` 表示单次无状态查询（无历史上下文，也不触发 `charts` 步骤）。
- 传入客户端生成的 UUID 以创建或延续会话线程（需同时具备 `chat:use` 与 `conversation:write`）。
- 必须等待上一轮流返回 `[DONE]` 后再发送下一轮；在同一会话并发重叠请求将返回 **`409 Conversation in progress`**。
- 若指定的 `conversation_id` 返回 `404`，通常表示该会话归属于其他用户。

Python 调用示例（引自 Auto Ontology README，无需额外 SDK）：

```python
import os
import requests

session = requests.Session()
session.headers["x-api-key"] = os.environ["AUTO_ONTOLOGY_API_TOKEN"]

terms = session.get("https://ontology.example.com/api/terms").json()

answer = session.post(
    "https://ontology.example.com/api/chat/completions",
    json={"question": "How many orders shipped last week?"},
)
# 响应是 SSE 事件流，而不是单个 JSON 对象
```

通过 API 创建的对话线程，归属用户登录 Web UI 后可直接访问 `/chat?focus=<conversation_id>` 查看。

### 语义发现（基于语义层，而非物理目录浏览）

优先使用 MCP，REST 兜底端点如下：

- `GET /api/terms`（`query`、`skip`、`limit`）
- `GET /api/terms/{term_id}`
- `GET /api/exploration/tables/{table_id}/details`
- `GET /api/exploration/graph`

模型修改与编译重置请移交 `nvidia-ontology-management`。

## 校验带业务断言的答案

**不能把 SQL 执行成功当作答案正确的充分证据。** 在交付面向决策的结果之前，请按照 [query-validation.md](references/query-validation.md) 核查统计总体、指标、单位、粒度、Join 路径、血缘、状态、有效期、结果行、截断情况与文字表述。如果某项关键约束无法验证，请如实报告缺口，切勿给出超出证据支撑的强结论。

## 空结果或失败响应排查

当 `ask_question` 或对话返回无用空结果时，在急着改写问题前，先调用 `check_readiness`（或 `GET /api/semantic-compilation/status`）。数据目录中存在库名并不能证明可以执行 SQL。故障排查详见 `nvidia-ontology-setup`。

## 示例

- 进行可信计数查询：依次执行 `check_readiness` → `search_terms` → `check_answerable` → `ask_question`。
- 若结果集被截断或违反指标粒度：拒绝下达全量结论，并遵循 [query-validation.md](references/query-validation.md) 处理。

## 故障排查

若返回空结果或执行失败，先检查就绪状态与编译状态，不要盲目改写提问。若 REST 请求报错，先核对 Web 网关 Origin、身份认证方式与会话归属权，再决定是否重试。

## 延伸阅读

- [stack.md](references/stack.md) —— AI-Q、Nemotron、cuOpt（本仓库已实现能力与合作方集成边界）
- [query-validation.md](references/query-validation.md) —— 实质意图、SQL、结果行与答案校验规范
- `nvidia-ontology-management` —— 通过语义层建模、修改并发布
- `nvidia-ontology-setup` —— 环境搭建与 MCP OAuth 发现排查
