<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto Ontology 安装故障排查

> 本文档为 [`skills/nvidia-ontology-setup/references/troubleshooting.md`](../../../../../skills/nvidia-ontology-setup/references/troubleshooting.md) 的简体中文翻译版。

联网搜索前请先查阅此文档。对外分享任何命令输出前，请务必脱敏其中的密钥信息。

## UI 加载缺少左侧导航栏（或重启后页面显示异常）

这是由于浏览器保留了过期的 Better Auth 会话 Cookie。常见于更改 `AUTH_SECRET` 或重置数据库之后。请清除 `localhost:3000` 的站点数据（或删除 `better-auth.session_token` Cookie）后重新登录。

如果每个页面都报错 Better Auth `"default secret"`，说明 `.env` 中缺少 `AUTH_SECRET` 或 `APP_URL`。这两项均为必填。

## 修改 `POSTGRES_PORT` 后应用内部没有生效

`POSTGRES_PORT` 仅重映射**宿主机侧**端口（`localhost:<port> → 容器 5432`）。在 Compose 内部网络中，各容器始终通过 `5432` 端口访问 Postgres。仅当宿主机 `5432` 端口被占用时才需要调整此变量。

## NIM / 对话 / 数据摄取失败，或向量检索看似为空

对话与摄取均需要 NVIDIA NIM 密钥。请配置 `DEFAULT_MODELS_API_KEY`（旧版兜底变量为 `NVIDIA_API_KEY`）。每个模型角色由一组三元组配置——`<PREFIX>_ENDPOINT`、`<PREFIX>_API_KEY`、`<PREFIX>_MODEL`，任何未设置字段均回退至 `DEFAULT_MODELS_*`。

**`EMBED_ENDPOINT` 与 `EMBED_MODEL` 在摄取与查询阶段必须完全一致。** 一旦不一致，入库向量与查询向量就会落在不同的向量空间中，导致提问总是表现为“什么都查不到”。

请核对 `.env`（或 Helm Values）中摄取服务与后端服务的模型三元组配置是否完全一致。

## 连接配置冲突：UI vs `CONNECTION_STRINGS` vs Vault

Auto Ontology 从以下两个位置解析源数据库：

1. 在 UI 中添加的连接。除非完整配置了 Vault，否则凭证会以明文形式存储在目录记录行中。
2. `CONNECTION_STRINGS`（逗号分隔），包括 Helm 的 `connectionStrings` 参数。这是一项**兜底配置**：仅当不存在任何 UI 添加的连接时才会生效。

只有当 `VAULT_ADDR`、`VAULT_NAMESPACE`、`VAULT_ROLE_ID` 与 `VAULT_SECRET_ID` **全部四项都设置**时，Vault 才会启用。不完整的 Vault 配置会被忽略并输出告警，凭证仍会回退为明文存储。`VAULT_AUTH_MOUNT` 与 `VAULT_KV_MOUNT` 为可选覆盖项。

## MCP 客户端无法发现如何登录

`$AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server` 必须返回 JSON（HTTP `200`）。如果返回向登录页的重定向，说明当前部署版本过旧，尚未包含该路由——请从代码库目录运行前端（`pnpm dev`）。

`AUTO_ONTOLOGY_API_URL` 应指向 **Web 应用**，默认为 `http://localhost:3000`，而不是 FastAPI `:3001`。

## 发现端点正常，但普通登录或 MCP 登录失败

用户、会话、API Key 以及 MCP OAuth 数据表位于 Prisma 管理的 `frontend` Schema 中，由 `frontend-migrate` 作业负责同步，而不是由后端的 `auto-ontology-migrate` 迁移。即使 `frontend-migrate` 退出码为 `0` 也请检查其日志，历史上曾出现过它零报错退出却导致 MCP 登录失败的情况：

- Compose：`docker compose logs frontend-migrate`
- Helm：`kubectl logs job/frontend-migrate-<release revision>`

## 报错 `"Protected resource ... does not match expected ..."`

MCP 服务器对外宣称的监听地址与客户端实际访问时的主机名拼写不一致——绝大多数情况下是混用了 `127.0.0.1` 和 `localhost`。请将 `AUTO_ONTOLOGY_MCP_PUBLIC_URL` 设为客户端实际访问的 URL，或取消设置 `AUTO_ONTOLOGY_MCP_HOST`。

## 报错 `"This request carried no signed-in session"`

当前关联的 OAuth 授权已过期或在 Auto Ontology 中被吊销。请在 MCP 客户端重新发起登录。MCP 服务器端没有可手动粘贴的静态 Token，切勿尝试配置一个。

## `ask_question` 返回空答案，既无 SQL 也无报错

请先调用 `check_readiness`。最常见的原因是未配置任何可用数据库连接。`databases` 列表非空并不能证明 SQL 可以执行：数据目录的生命周期独立于连接，即使连接被删，已编目的库依然存在。此外，默认仅管理员可读取连接列表；观察者（Viewer）调用时会收到 `403`，但这并非部署损坏或凭据错误，因此结果会归入 `unverified`。

如果确认部署已完全就绪，则可能是提问用词与术语表不匹配——请先调用 `search_terms` 检索问题中的关键业务名词。

## 报错 `"Auto Ontology cannot answer right now"`

通常是因为语义层从未被编译；请调用 `check_readiness` 或 `GET /api/semantic-compilation/status` 确认。如果同一个 `conversation_id` 上仍有未完成的对话轮次（`409`），也会返回此提示。

## 客户端看不到任何 MCP 工具

请查看客户端的 MCP 日志（服务端日志输出至 `stderr`），并确认客户端配置中的 URL 末尾已包含 `/mcp` 后缀，且修改配置后已重启客户端。

## SSO 重定向 URI 问题

请在身份提供商（IdP）中配置重定向 URI 为 `APP_URL/api/auth/sso/callback`。若配置不匹配，现象会表现为登录失败，而不是容器启动报错。
