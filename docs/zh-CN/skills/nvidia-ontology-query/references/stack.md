<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 更广泛技术栈中的 Auto Ontology

> 本文档为 [`skills/nvidia-ontology-query/references/stack.md`](../../../../../skills/nvidia-ontology-query/references/stack.md) 的简体中文翻译版。

本文件如实界定本仓库**实际已实现的能力**与**需要合作方自行集成的边界**。切勿臆造仓库中并不存在的一方集成蓝图。

## AI-Q / NemoClaw（认证契约）

Auto Ontology 通常使用 Better Auth Cookie 验证浏览器用户。而 AI-Q（及类似的 NVIDIA SSO 调用方）会通过 `Authorization: Bearer <jwt>` 转发**用户的 SSO ID Token**。Auto Ontology 会依据已配置 SSO 提供商的 JWKS（与交互式 SSO 相同的签发者 Issuer）验证该 JWT 的签名、签发者与过期时间。**不会校验 `aud`（Audience）**：因为该 Token 是为 AI-Q 的 OAuth 客户端签发的，其 `aud` 为 AI-Q 的 Client ID 而非 Auto Ontology 的。信任该 SSO 提供商签发的合法 Token 正是设计意图，无需额外的服务间共享密钥。

随后系统会按邮箱（Email）、再按 SSO 账号 `subject` 将该 Bearer Token 映射到 Auto Ontology 本地用户——系统中**必须预先存在**匹配的 Auto Ontology 用户。

此处定义的是认证契约，而非构建 AI-Q 蓝图或 NemoClaw 技能包的教程。

所有 `/api` 路由统一遵循的凭据解析顺序（[`frontend/auth/resolve-user.ts`](../../../../../frontend/auth/resolve-user.ts)）：

1. 浏览器会话 Cookie
2. Auto Ontology API Token（`auto_ontology_…`，`x-api-key` 或 Bearer）
3. Auto Ontology 签发的 OAuth Bearer Token（MCP）
4. SSO ID Token（AI-Q）

已吊销的 API Token 会直接返回 `401`，不会继续降级当作 SSO JWT 重试。

## Nemotron / NIM（部署配置）

Auto Ontology 通过调用 NVIDIA NIM 实现对话推理、向量嵌入与重排序。这属于 `.env` / Helm Values 配置层面，而非 Nemotron 模型微调或定制训练集成。

模型三元组配置：`<PREFIX>_ENDPOINT`、`<PREFIX>_API_KEY`、`<PREFIX>_MODEL`。未设置字段自动回退到 `DEFAULT_MODELS_*`。支持的前缀包括：`DEFAULT_MODELS`、`REASONING`（NL-to-SQL）、`NON_REASONING`（实体抽取）、`EMBED`、`RERANK`。`*_API_KEY` 还会继续回退至 `NVIDIA_API_KEY`。

**`EMBED_ENDPOINT` 与 `EMBED_MODEL` 在摄取与查询两个阶段必须完全一致。** `.env.example` 中的默认配置指定了 Nemotron 嵌入模型 ID；若修改了嵌入模型却不重新摄取，检索将完全失效。

## cuOpt（由合作方编排集成）

本仓库**不包含**任何 cuOpt 客户端、技能或作业桥接代码。Auto Ontology 只负责返回自然语言答案、执行的 SQL 与结果行。合作方智能体可以提取这些数据去调用 cuOpt，并通过 NVIDIA 的 `cuopt-*` 技能完成建模求解。切勿宣称 Auto Ontology 自身能够调度路径优化或求解线性规划问题。

## 集成栈中的关键调用铁律

- 合作方必须调用 Next.js 网关（`APP_URL` / `:3000`），绝不能直连公开未受保护的 FastAPI `:3001`。
- MCP 不持有全部署级身份。每位调用方都必须以自己的账号登录。
- `conversation_id` 严格归属于单个 Auto Ontology 用户，切勿跨用户共享 UUID。
- 同一会话并发重叠请求将返回 `409`。
