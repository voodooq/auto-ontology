<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 前端说明（Next.js）

> 本文档为 [`frontend/README.md`](./README.md) 的简体中文翻译版。
>
> **注**：原文中包含若干早期历史端口与路径描述（如 `python/`、`3001`/`8000`、`server.main:app`），当前项目的实际运行方式请以根目录 [`README.zh-CN.md`](../README.zh-CN.md)、[`CLAUDE.zh-CN.md`](../CLAUDE.zh-CN.md) 与 [`dev_tools/README.zh-CN.md`](../dev_tools/README.zh-CN.md) 为准（前端默认 `3000`，后端默认 `3001` 且入口为 `auto_ontology.server.__main__:create_app`）。

这是一个 [Next.js](https://nextjs.org) 应用。要使 `/data` 等依赖后端 API 的路由正常工作，**Python API 必须处于运行状态**。完整配置请参阅**[根目录 README](../README.zh-CN.md)**（英文原版：[README.md](../README.md)）。

## 快速开始

在仓库根目录执行：

```bash
pnpm dev
```

（原文说明：该命令通过 Turborepo 同时启动 Next.js 开发服务器与 FastAPI 后端；当前版本请参照 `dev_tools/setup_env.sh --dev` 与 `cd frontend && pnpm dev`。）

在浏览器中打开 **<http://localhost:3001>**（当前默认端口为 `http://localhost:3000`）。

你可以从编辑 `app/page.tsx` 开始修改页面，保存后页面会自动热更新。

本项目使用 [`next/font`](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) 自动优化并加载 Vercel 的 [Geist](https://vercel.com/font) 字体族。

## 延伸学习

如需深入了解 Next.js，可参考以下资源：

- [Next.js 官方文档](https://nextjs.org/docs) —— 学习 Next.js 特性与 API。
- [Learn Next.js](https://nextjs.org/learn) —— 交互式 Next.js 教程。

也欢迎查看 [Next.js GitHub 仓库](https://github.com/vercel/next.js)。

```bash
# 原文附注命令（当前请使用 uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001）
run auto_ontology from root - cd auto_ontology && uv run --project .. uvicorn server.main:app --reload --host 127.0.0.1 --port 3001
```
