<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# OpenAPI 规范说明

> 本文档为 [`docs/openapi/README.md`](../../openapi/README.md) 的简体中文翻译版。

此目录下的 JSON 文件均为**自动生成文件——请勿手工编辑**。将它们提交至版本库是为了方便在 PR Diff 中评审接口变更，并支持在不启动服务的情况下直接消费规范；CI 会自动重新生成并对比，若提交的文件过期未同步则会直接报错失败。

| 文件 | 描述对象 | 生成命令 |
| --- | --- | --- |
| [`auto-ontology-api.json`](../../openapi/auto-ontology-api.json) | **对外公开 API。** | `pnpm openapi`（在 `frontend/` 目录下运行） |
| [`backend.json`](../../openapi/backend.json) | 内部 Auto-Ontology FastAPI 应用。 | `uv run python -m dev_tools.generate_backend_openapi` |
| [`ingestion.json`](../../openapi/ingestion.json) | 内部 Ingestion Service FastAPI 应用。 | 同上 |

## 外部消费者应使用哪一份？

请使用 `auto-ontology-api.json`。Python 后端与摄取服务均仅为 Kubernetes `ClusterIP` 内部服务，只能通过 Next.js 代理层访问，因此 `backend.json` 与 `ingestion.json` 描述的是集群外部无法直接调用的内部接口面。保留这两个文件是因为 `auto-ontology-api.json` 的部分定义需要由它们合并生成。

此外，`auto-ontology-api.json` 还包含 FastAPI 规范无法感知的元数据：

- `x-auto-ontology-permissions` —— 调用方所需的权限标识，提取自各路由的 `withPermission({...})` 包装器（`withPublic` 路由则标注为 `"public"`），并自动补充对应的 `401`/`403` 响应定义。该包装器同时决定接口的 `security` 声明：`ApiToken` 认证方案，或 `withPublic` 路由对应的 `[]`。
- `x-auto-ontology-source` —— 实际提供该接口实现的 `route.ts` 源文件位置。
- `x-auto-ontology-proxied` —— 标识该处理器是透传给 FastAPI 还是由 Next.js 自行处理。
- `x-auto-ontology-upstream` —— 针对对话路由，标注所中继的后端上游路径。

## 构建机制

`frontend/scripts/generate-openapi.ts` 使用 `ts-morph` 扫描 `frontend/app/api/**/route.ts`（借助 Next.js 基于文件系统的路由确定接口路径），并从两个来源填充 Schema：

- **透传代理处理器**（`proxyToBackend`）：原封不动转发请求路径，因此直接按路径与 HTTP 方法从 `backend.json` 匹配并合并 Schema。
- **其余所有路由**（直接基于 Prisma 的处理器，后端无对应接口；以及自行构造上游请求的对话处理器）：通过同级目录下的 `app/api/**/openapi.ts` 模块使用 `zod` 声明其请求/响应结构，具体约定见 `frontend/types/openapi.ts`。之所以独立放在 `openapi.ts` 而非 `route.ts` 内部，是因为纯 Node 脚本无法直接导入依赖运行时的路由文件（如 `next/headers`、`after()`、`getPrisma()`）。

最终仅会将公开规范实际引用的 Component Schema 及其传递依赖拷贝至 `auto-ontology-api.json` 中。

## 重新生成命令

```sh
uv run python -m dev_tools.generate_backend_openapi   # 生成 backend.json、ingestion.json
cd frontend && pnpm openapi                           # 生成 auto-ontology-api.json
```

请严格按照上述顺序执行——因为公开规范脚本会读取已生成的 `backend.json`。

## 在线预览

该规范已部署至 GitHub Pages：<https://friendly-adventure-38857y4.pages.github.io/>
后续也会整合进 NVIDIA 官网的 Auto-Ontology 文档体系中。

或者也可以将文件内容粘贴至 <https://docs.scalar.com/swagger-editor> 查看——但请注意这相当于将完整的 API 接口定义提交给第三方网站。

## 命名与接口约定

**全栈 API 统一采用 `snake_case`（蛇形命名法）**——涵盖 URL 路径段、路径参数、查询参数、请求体与响应体字段。无论后端内部实现采用何种命名风格，任何新增接口都必须遵守此约定：

- 动态路由目录名直接对应所携带的参数名，采用 `snake_case`：`app/api/terms/[term_id]/route.ts` → `/api/terms/{term_id}`。
- 仅由前端路由访问的 Prisma 数据模型同样采用 `snake_case` 字段名，因此数据库行无需额外映射层即可直接序列化为 API 结构。
- 若底层依赖库的模型字段为 `camelCase` 且无法更改，则必须在路由边界完成映射转换，防止泄露给接口调用方：例如 Better Auth 生成的模型（`User`、`Session`、`Account`、`SsoProvider`）为驼峰命名，`GET /api/sso-providers` 会显式映射输出 `provider_id`。`GET /api/sso-discovery` 则按 OpenID Connect Discovery 标准原样中继标准字段名。
- TypeScript 内部代码保持常规的 `camelCase` 局部变量和 UI 类型；仅跨网络边界的数据采用 `snake_case`。

生成器会对上述约定进行静态校验：如果无法解析某条路由的 `withPermission(...)` 参数，或某个 `route.ts` 未导出任何可识别的 HTTP 方法，脚本都会直接报错退出，从而防止接口被静默遗漏。

## 已知局限

- 暂未配置 `servers` 字段：当前规范说明了如何认证，但未指定默认请求主机。
- 大多数 Schema 带有 `additionalProperties: true`。后端响应模型刻意采用宽松模式——因为在 FastAPI 中一旦指定严格的 `response_model`，未显式声明的字段会被静默*丢弃*，从而直接破坏接口功能而不仅仅是文档不完整。因此文档列出的字段是可靠的，但并不保证穷尽所有返回字段。
- `PATCH /api/nodes/{node_id}` 返回的 `NodeUpdateResult | null` 所有字段均为可选：响应体仅回显请求中实际更新的属性。同时该路由对任意节点类型都开放这些字段——例如 `sample_values` 仅对 Column 有意义，但传给 Database 时并不会被拒绝。
- `/api/auth/{all}` 是 Better Auth 的通配路由。它在规范中有体现，但具体各子端点的请求与响应格式由库本身管理，未在此规范中逐一展开。
