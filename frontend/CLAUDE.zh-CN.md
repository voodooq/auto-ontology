<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 前端开发规范（Frontend Rules）

> 本文档为 [`frontend/CLAUDE.md`](./CLAUDE.md) 的简体中文翻译版。

继承根目录：[`AGENTS.zh-CN.md`](../AGENTS.zh-CN.md)

## 前端规范

- **Next.js 16**：相对早期版本存在破坏性变更。编写 Next.js 专项代码前，请先查阅 `node_modules/next/dist/docs/`，详情见 [`frontend/AGENTS.zh-CN.md`](./AGENTS.zh-CN.md)。
- **代码格式化**：使用 Prettier。**修改 `frontend/` 下的任意文件后，必须在结束前运行 `pnpm format`（或 `cd frontend && npx prettier --write <file>`）自动格式化。** 切勿依赖手工缩进排版——务必交由 Prettier 处理，否则 CI 将拒绝未格式化的代码。
- **代码检查**：使用 ESLint。修复所有 lint 报错；除非附带解释原因的注释，否则严禁禁用规则。
- **组件写法**：统一使用箭头函数组件（Arrow function components），禁止使用类组件（Class components）。
- **样式规范**：使用 Tailwind CSS 4。禁止在组件样式中使用内联 `style` 或拆分的独立 CSS 文件。
- **类型安全**：开启 TypeScript 严格模式（Strict mode）。除非附带说明原因的注释，否则禁止使用 `any`。通用类型置于 `/frontend/types`，枚举置于 `/frontend/enums`。
- **API 请求层**：统一使用 `frontend/api/requests.ts` 封装的 `requests` 工具，不要直接调用 `axios`。Base URL 与统一错误处理已在该层集中管理。
