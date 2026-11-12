<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 身份认证、SSO 与 API Token

> 本文档为 [`docs/authentication.md`](../authentication.md) 的简体中文翻译版。

Auto-Ontology 通过 [Better Auth](https://github.com/better-auth/better-auth) 完成身份认证，并依据调用者的角色对每一次请求进行权限校验。系统默认禁用自助注册：首个账号为通过环境变量 `AUTO_ONTOLOGY_ADMIN_EMAIL` / `AUTO_ONTOLOGY_ADMIN_PASSWORD`（在 Helm 中为 `adminEmail` / `adminPassword`）引导创建的初始管理员。

所有 `/api/...` 路由都在统一入口 [`frontend/auth/resolve-user.ts`](../../frontend/auth/resolve-user.ts) 中解析调用者身份，按以下顺序依次匹配：

1. 浏览器会话 Cookie
2. Auto-Ontology API Token（请求头 `x-api-key` 或 `Authorization: Bearer`）
3. Auto-Ontology 签发给 MCP 客户端的 OAuth Bearer Token
4. 由受信任调用方转发的 SSO ID Token

## 单点登录（SSO）

可在 Web 界面的 **Settings → Single Sign-On** 中配置 SSO。请在您的身份提供商（IdP）中注册以下回调重定向 URI：

```text
<APP_URL>/api/auth/sso/callback
```

## API Token

浏览器用户使用会话 Cookie；而脚本、Notebook 与定时任务则应使用 API Token 进行认证。

1. 打开右上角用户菜单 → **API Tokens** → **New token**。
2. 填写名称并可选设置过期时间（默认永不过期），随后复制生成的 Token。Auto-Ontology 服务端仅保存 Token 的 SHA-256 哈希值，因此明文 Token 仅在创建时展示一次。如不慎遗失，请重新签发一个新 Token。

调用任意 `/api/...` 接口时，在请求头中通过 `x-api-key`（或 `Authorization: Bearer <token>`）携带该 Token：

```sh
curl -H "x-api-key: $AUTO_ONTOLOGY_API_TOKEN" https://auto-ontology.example.com/api/terms
```

```python
import os
import requests

session = requests.Session()
session.headers["x-api-key"] = os.environ["AUTO_ONTOLOGY_API_TOKEN"]
terms = session.get("https://auto-ontology.example.com/api/terms").json()
```

关于流式问答接口，请参阅 [conversation-api.md](./conversation-api.md)。

Token 的行为特性如下：

- **Token 继承所属用户的权限**：Token 本身没有独立权限集；每次接口调用都会按照创建者的角色进行鉴权，效果与在浏览器中登录一致。例如，观察者（Viewer）创建的 Token 仅具备观察者权限。
- **所有认证接口均支持 Token**：[`docs/openapi/auto-ontology-api.json`](../openapi/auto-ontology-api.json) 中任何需要身份认证的操作均可使用 API Token 调用。
- **吊销立即生效**：一旦删除该 Token，或者删除/封禁其所属用户，下一次请求就会立即返回 `401`。
- **Token 无法管理 Token**：创建与吊销 API Token 必须使用已登录的浏览器会话，从而防止泄露的 Token 自行签发新的后继 Token。
