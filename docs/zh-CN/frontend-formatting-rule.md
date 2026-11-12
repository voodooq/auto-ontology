<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 前端代码格式化规范 — Prettier（强制要求）

> 本文档为 [`.cursor/rules/frontend-formatting.md`](../../.cursor/rules/frontend-formatting.md) 的简体中文翻译版。
> 原规则作用域：`frontend/**/*.{ts,tsx,js,jsx,css,json,md}`。

修改 `frontend/` 目录下的**任意**文件后，结束任务前**必须**运行 Prettier 进行格式化：

```bash
cd frontend && npx prettier --write <relative-path-to-file>
```

或者一次性格式化所有前端文件：

```bash
pnpm format
```

## 为什么这很重要

- CI 会执行 `pnpm format:check`，**任何未通过格式化的代码都会被拒绝**。
- 切勿依赖手工缩进或对齐空格——Prettier 强制统一使用制表符（tabs）、单引号（single quotes）、分号（semicolons）、100 字符行宽及尾随逗号（配置见 `frontend/.prettierrc`）。
- **每次编辑后**都应运行 Prettier，而不是等到提交前再统一补跑。如果一次重构了多个文件，请逐个格式化或最后统一执行一次 `pnpm format`。

## 常见易错点

- 对象属性缩进层级错误（Prettier 会自动修正嵌套深度）。
- 超出 100 字符的长行未换行（Prettier 会自动折行）。
- 多文件重构后遗漏运行格式化。
