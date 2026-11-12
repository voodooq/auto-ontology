<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# frontend/AGENTS.md — Next.js 智能体开发提示

> 本文档为 [`frontend/AGENTS.md`](./AGENTS.md) 的简体中文翻译版。

<!-- BEGIN:nextjs-agent-rules -->

# 这不是你熟悉的传统 Next.js

当前使用的版本（Next.js 16）包含重大破坏性变更——API、工程约定和文件结构都可能与模型训练数据中的旧版认知不同。编写任何代码前，请先阅读 `node_modules/next/dist/docs/` 下的相关指南（从本文件所在目录解析；在 Monorepo 结构下，仓库根目录可能找不到 `next` 包），并严格留意弃用警告（Deprecation Notices）。

此提示块由 `next dev` 自动写入并在每次启动时重新生成（可参见 `node_modules/next/dist/server/lib/generate-agent-files.js`）。在 Diff 中手动删除它只会导致下次重新生成未提交变更；将其随工作一并提交即可保持工作区干净。

<!-- END:nextjs-agent-rules -->
