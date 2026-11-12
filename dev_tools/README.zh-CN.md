<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# dev_tools 开发辅助工具集

> 本文档为 [`dev_tools/README.md`](./README.md) 的简体中文翻译版。

本地开发辅助脚本与测试数据工具：涵盖环境启动、测试数据库初始化与数据摄取。

## 目录结构

| 路径 | 用途说明 |
| --- | --- |
| `setup_env.sh` | 通过 `docker compose` 拉起本地开发环境，供开发者手动调用。 |
| `local_ingest.py` | 将本地 Postgres 数据源摄取写入 `pgvector` 向量库。 |
| `generate_backend_openapi.py` | 导出 FastAPI OpenAPI 规范文件至 `docs/openapi/`，由 `ci-openapi.yml` 校验。 |
| `release_helm_chart.py` | 打包 Helm Chart 并发布至 NGC Helm 仓库，由 `release-helm.yml` 执行。 |
| `_add_spdx_headers.py` | 一次性 OSRB 合规辅助脚本，保留以确保 SPDX 声明头插入可复现。 |
| `fixtures/` | 测试数据集夹具。**CI 强依赖这些数据**——详见下文。 |

`fixtures/` 下的脚本用于构建 `auto_ontology/dal/tests/test_golden.py` 和 `auto_ontology/connectors/tests/test_postgres.py` 所回放测试的数据库，并在 `.github/workflows/ci-python.yml` 中运行。修改这些夹具会直接影响 CI 验证覆盖面。各测试数据库的覆盖场景与设计原因详见 [`fixtures/sql/README.zh-CN.md`](./fixtures/sql/README.zh-CN.md)。

```bash
docker compose up -d postgres
uv run --no-sync python -m dev_tools.fixtures.seed_fixtures          # 初始化 pagila + chinook
uv run --no-sync alembic upgrade head
uv run --no-sync python -m dev_tools.fixtures.seed_graph_fixture --reset
uv run --no-sync pytest auto_ontology dev_tools -q
```

## 本地运行

运行完整应用需要同时启动 **前端**（`frontend/`）和 **Python API**（`auto_ontology/`）。

### 首次环境准备

```bash
cp .env.example .env   # 随后按需修改密码与 API Key
cd frontend && pnpm install
uv sync                # 在仓库根目录执行
```

### 启动开发环境

一键启动全栈（基础设施 + Next.js + FastAPI）：

```bash
./dev_tools/setup_env.sh
```

或者仅启动基础设施（Postgres、pgAdmin、Ingestion Service）：

```bash
./dev_tools/setup_env.sh --dev
```

随后在本地手动启动前后端应用：

```bash
cd frontend && pnpm dev                 # Next.js 运行在 3000 端口
uv run python -m auto_ontology.server   # FastAPI 运行在 3001 端口（仓库根目录执行）
```

打开浏览器访问 **<http://localhost:3000>**。

### 可选项

- 覆盖后端 API 地址：设置 **`PYTHON_API_URL`**（用于 Next.js 路由重写与服务端 API 请求）。

## `setup_env.sh` 脚本说明

通过 `docker compose` 启动 Auto-Ontology 环境。

| 命令 | 启动内容 |
| --- | --- |
| `bash ./dev_tools/setup_env.sh` | 全栈启动 —— 包含基础设施（Postgres、pgAdmin、Ingestion Service）**以及** `auto_ontology` 和 `auto-ontology-frontend` 容器镜像（带 `--build` 重新构建）。 |
| `bash ./dev_tools/setup_env.sh --dev` | 仅启动基础设施，由开发者自己在宿主机本地运行 `auto_ontology` 与 `auto-ontology-frontend`。 |
| `bash ./dev_tools/setup_env.sh --ds` | 基础设施 **+** 容器化运行的 `auto-ontology-frontend`。开发者在宿主机 `:3001` 运行 `auto_ontology`，容器内前端通过 `host.docker.internal:3001` 访问本地后端。 |

### 服务端点

| 服务 | 访问地址 |
| --- | --- |
| Postgres | `localhost:5432` |
| pgAdmin | <http://localhost:5050> |
| Auto-Ontology 摄取服务 | <http://localhost:3002> |
| Auto-Ontology 前端（仅全栈模式） | <http://localhost:3000> |
| Auto-Ontology 后端（仅全栈模式） | <http://localhost:3001> |

### `--dev` 工作流

运行 `./dev_tools/setup_env.sh --dev` 拉起基础设施后，在两个终端中分别启动前后端：

```bash
# 终端 1 — Next.js
cd frontend && pnpm install && pnpm dev

# 终端 2 — FastAPI（在仓库根目录执行）
uv sync
uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001
```

应用工厂函数是 `auto_ontology/server/__main__.py` 中的 `create_app()`；代码中不存在 `auto_ontology/server/main.py`。`uv run python -m auto_ontology.server` 启动的是同一个应用（无热重载，容器内即采用此命令）。

### `--ds` 工作流

`--ds` 模式在 Docker 中运行 Next.js 前端，而由你在宿主机本地运行 FastAPI 后端——当你专注于迭代 Python 后端代码且不想反复重建前端镜像时非常实用。

该模式下前端镜像在构建时会固化 `PYTHON_API_URL=http://host.docker.internal:3001`，使浏览器请求经由前端容器转发至宿主机的 `:3001` 端口。

```bash
bash ./dev_tools/setup_env.sh --ds

# 随后在宿主机执行：
uv sync
uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001
```

> Linux 提示：`host.docker.internal` 在 Docker Desktop（macOS / Windows）上开箱即用。在原生 Linux Docker 环境中，可能需要额外添加 `--add-host=host.docker.internal:host-gateway`，或将后端绑定到 `0.0.0.0` 并使用 Docker 网桥 IP。

### 配置说明

服务端口与凭据均读取仓库根目录下的 `.env` 文件（`docker compose` 会自动加载）。完整环境变量列表请参阅根目录 `README.zh-CN.md` 和 `CLAUDE.zh-CN.md`。

### 停止服务

```bash
docker compose down       # 停止所有容器
docker compose down -v    # 停止并清除数据卷（删除 Postgres 与 pgAdmin 数据）
```

## 容器化与 Kubernetes 部署

本仓库提供用于 Kubernetes 部署的 Dockerfile 与 Helm Chart：

- `Dockerfile` —— 后端与摄取服务（FastAPI / uvicorn）
- `frontend/Dockerfile` —— 前端（Next.js standalone 模式）
- `helm/auto-ontology/` —— Helm Chart，包含 Deployment、Service、可选 Ingress、HPA、PDB 及示例 Postgres 清单
