<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 对话 API（Conversation API）

> 本文档为 [`docs/conversation-api.md`](../conversation-api.md) 的简体中文翻译版。

客户端需通过经过身份认证的 Next.js 网关发起提问请求：

```text
POST /api/chat/completions
```

请使用浏览器会话 Cookie 或 API Token 进行身份认证（详见 [authentication.md](./authentication.md)）。完整的请求与响应数据结构定义在 [`docs/openapi/auto-ontology-api.json`](../openapi/auto-ontology-api.json) 中。

## 响应流（Response stream）

接口响应是一个 Server-Sent Events（SSE）事件流。每条事件的 `data:` 行包含一个 JSON 对象，流结束时以非 JSON 的 `data: [DONE]` 行作为终止标记。

| `type` | 携带内容说明 |
| --- | --- |
| `step` | 某个智能体节点开始或结束执行（包含 `node`、`phase`，结束时还包含 `thought`），用于向前端展示实时进度。 |
| `sql` | 已通过校验、即将发送到数据库执行的 SQL 查询语句。 |
| `result` | 最终的 `answer` 对象：包含 `response`（自然语言回答）、`sql_code`（SQL 语句）、`sql_response_from_db`（查询结果行）、`thoughts`（步骤轨迹）。 |
| `charts` | 针对查询结果生成的图表或表格规格（`content`、`sql_response`）。仅当成功执行了 SQL 查询且请求中传入了 `conversation_id` 时，在 `result` 之后发送。 |
| `error` | 包含 `message` 字段，说明本次运行失败的原因。 |

单次问答通常涉及多次串行模型调用，耗时一般在数十秒量级，因此请按流式方式实时读取响应：

```python
import json
import os

import requests

response = requests.post(
    "https://auto-ontology.example.com/api/chat/completions",
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
    if event["type"] == "step":
        print("…", event.get("node"), event.get("phase"))
    elif event["type"] == "result":
        answer = event["answer"]
        print(answer["response"])
        print(answer["sql_code"])
    elif event["type"] == "error":
        raise RuntimeError(event["message"])
```

MCP 服务器中的 `ask_question` 工具正是对这一接口的封装，并将自然语言回答、SQL 与结果行拆分为独立字段返回；详见 [mcp.md](./mcp.md)。

## 多轮对话管理

请求体中的 `conversation_id` 为可选字段：

- 省略该字段时，表示一次无状态的单次请求（One-shot）。
- 传入客户端生成的 UUID 可创建或延续一段对话，并在同一会话线程的后续轮次中复用该 UUID。
- 请务必等待上一轮流返回 `[DONE]` 后再发送下一轮问题。在同一会话并发发起重叠请求会返回 `409 Conversation in progress`。

```json
{
  "question": "What about last month?",
  "conversation_id": "d61d8aa3-d496-4fa7-97ce-4f831c162e7f"
}
```

对于多轮对话中的后续追问，后端最多会加载最近 5 轮已完成对话的历史（总字符数上限 12,000 字符），并将依赖上下文的追问（如“What about August?”）改写为语义完整的独立问题后，再送入 Text-to-SQL 智能体。若问题本身已语义独立，则原样透传。仅历史问题、回答文本与 SQL 会参与问题改写；智能体思考过程、图表载荷、错误信息以及原始查询结果行均不会传入改写上下文。

对话历史归属于发起认证的用户。通过 API 创建的对话，其所属用户可在 Web UI 中访问 `/chat?focus=<conversation_id>` 打开查看。Web UI 本身会自动管理其创建的会话 ID。

## 信任边界（Trust boundary）

外部调用请始终经由 Next.js 网关（`APP_URL`，Docker Compose 下为 `3000` 端口）。网关完成用户鉴权后，会将内部请求头 `x-auto-ontology-user-id` 转发给 FastAPI（`3001` 端口）。FastAPI 会直接信任该请求头而不再二次校验，因此**必须将 3001 端口隔离在私有网络内**。直接请求 FastAPI 且不带 `conversation_id` 的调用按无状态处理；一旦携带 `conversation_id`，则必须附带受信任的身份请求头。
