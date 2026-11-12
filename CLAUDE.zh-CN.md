<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto-Ontology 维护者开发指南

> 本文档为 [`CLAUDE.md`](./CLAUDE.md) 的简体中文翻译版。

全栈仓库架构：Next.js 前端（`/frontend`） + FastAPI 后端（`/auto_ontology`）。

本文件面向 **Auto-Ontology 维护者**。面向用户或合作方的智能体技能（Agent Skills）位于 `skills/`，可从 [`AGENTS.zh-CN.md`](./AGENTS.zh-CN.md) 开始阅读。

## 开发命令

后端命令均在仓库根目录执行：

| 命令 | 说明 |
| --- | --- |
| `uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001` | 在 `:3001` 端口启动带热重载的 FastAPI 开发服务器 |
| `uv run python -m auto_ontology.server` | 在 `:3001` 端口启动不带热重载的 FastAPI（容器内运行的命令） |
| `uv run python -m auto_ontology.ingestion_service` | 在 `:3002` 端口启动数据摄取与语义编译服务 |

应用工厂函数是 `auto_ontology/server/__main__.py` 中的 `create_app()` —— 代码中**不存在** `auto_ontology/server/main.py`。

## 前端规范

详见 [`frontend/CLAUDE.zh-CN.md`](./frontend/CLAUDE.zh-CN.md)（英文原版：[`frontend/CLAUDE.md`](./frontend/CLAUDE.md)），在 `frontend/` 目录下工作时生效。

## 后端规范

- **主存储为 Postgres**：`auto_ontology/dal/` 使用 SQLAlchemy Core 进行查询（不使用 ORM）；`auto_ontology/dal/schema.py` 是表结构的唯一事实来源，`auto_ontology/dal/session.py` 管理数据库连接池。表结构变更必须通过 Alembic 执行——`uv run alembic revision --autogenerate -m "..."`——绝不允许手工修改已经应用过的迁移脚本。`schema.py` 中有多处列类型和约束是出于特定考虑设计的（并非显而易见），修改表前请先阅读其中的注释。
- **格式与代码检查**：提交代码前请在仓库根目录运行 `uv run ruff check auto_ontology/` 和 `uv run ruff format auto_ontology/`。
- **依赖管理**：统一使用 `uv`。通过 `uv add` 添加依赖，不要使用 `pip`，也不要手动编辑 `pyproject.toml` 添加依赖。
- **API 路由前缀**：所有路由统一挂载在 `/api/` 下。路由定义在 `auto_ontology/server/<module>/router.py`（如 `datasources/router.py`），业务逻辑编排在 `service.py` 中。
- **类型注解**：所有函数签名必须标注完整的类型注解。
- **异常处理器**：所有 `@app.exception_handler` 函数使用普通 `def`（不要用 `async def`）。因为它们不涉及异步 I/O，优先使用同步处理器。

## Git 与 PR 流程

- 提交信息（Commit message）：采用简洁的祈使句风格，不要附带 `Co-Authored-By` 尾部签名。
- 从 `main` 分支拉取特性分支。

### 提交 Pull Request（PR）

**在所有 GitHub Actions 工作流全部通过之前，PR 都不算完成。** 创建 PR 后，轮询 `gh pr checks <number>` 直至没有待定检查项，若有任何失败则修复并重新推送——重复直至全部绿灯。如实汇报最终状态；严禁将未经检查或处于失败状态的 PR 交付审核。

建议先在本地运行以下门禁检查（与 CI 规则一致，本地排查仅需几秒，远快于等待 CI 往返）：

| 检查项 | 命令 | 捕获问题 |
| --- | --- | --- |
| Ruff | `uv run ruff check auto_ontology/ && uv run ruff format --check auto_ontology/` | 后端 lint 与格式问题 |
| Pytest | `uv run pytest auto_ontology dev_tools -q` | 后端单元测试与集成测试 |
| OpenAPI | `uv run python -m dev_tools.generate_backend_openapi` | **`docs/openapi/*.json` 过期未同步** |
| Frontend | `cd frontend && npx prettier --check . && pnpm lint && pnpm build` | 前端格式、lint 与构建错误 |

其中 OpenAPI 检查最容易被忽略：`docs/openapi/backend.json` 和 `ingestion.json` 是提交到仓库中的，且 OpenAPI 规范会内嵌每个路由处理函数的 **docstring**。因此只要修改了任何 `@router` 函数的文档字符串（哪怕没有改变函数签名或 Pydantic 模型），就会导致已提交的规范文件过期并在 CI 中触发 `"Backend spec is up to date"` 失败。每次修改路由代码或文档注释后，请务必重新生成并提交对应的 OpenAPI JSON。
