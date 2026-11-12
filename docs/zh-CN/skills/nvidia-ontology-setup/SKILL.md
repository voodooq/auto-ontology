---
name: nvidia-ontology-setup
description: >-
  搭建或排查 Auto Ontology 运行时环境。适用于 Helm（官方安装方式）、Docker Compose、开发者模式，以及将 MCP 连接到已有部署。
license: Apache-2.0
metadata:
  version: "0.3.1"
  author: "NVIDIA <opensource@nvidia.com>"
  tags:
    - nvidia-ontology
    - install
    - docker
    - helm
    - mcp
    - troubleshooting
---

<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto Ontology 安装与配置

> 本文档为 [`skills/nvidia-ontology-setup/SKILL.md`](../../../../skills/nvidia-ontology-setup/SKILL.md) 的简体中文翻译版。

## 目的

启动当前版本的 Auto Ontology、排查故障或将智能体连接到正在运行的实例。切勿另起炉灶编写第二套安装脚本：官方安装方式是仓库根目录 `DEPLOYMENT.md` 中的 Helm Chart；Docker Compose 则用于在单机上运行相同的服务栈。`dev_tools/setup_env.sh` 仅是开发者辅助工具，而非正式安装程序。类型化的安装、连接、摄取、编译与就绪检查契约定义在 [runtime-contract.yaml](../../../../skills/nvidia-ontology-setup/assets/runtime-contract.yaml) 中。

遇到故障时，请优先查阅 [troubleshooting.md](references/troubleshooting.md)，不要盲目在网上搜索。

## 操作指引

### 必问确认项

如果以下信息尚不明确，请先向用户确认，不要自行猜测默认值并直接给出命令：

1. **部署目标** —— Kubernetes 上的 Helm（官方安装方式，默认推荐）、单机 Docker Compose、开发者工作流（`--dev` / `--ds`，仅限需要修改 Auto Ontology 源码时），还是“Auto Ontology 已运行，我只需要配置 MCP”？
2. **NVIDIA NIM 密钥** —— 是否已备好 `DEFAULT_MODELS_API_KEY`（Helm 中为 `defaultModelsApiKey`）？对话问答与数据摄取均依赖它。
3. **管理员账号** —— 初始管理员的邮箱与密码。由于系统禁用了自助注册，这是唯一的初始登录途径。
4. **源数据库** —— 是否通过连接字符串（`CONNECTION_STRINGS` / `connectionStrings`）提供，还是稍后在 Web UI 中添加？
5. **仅 Helm 需要** —— 需要安装的 Chart 版本号，以及用户实际访问浏览器的 URL（`appUrl`）。
6. **仅 Compose 需要** —— 宿主机 `5432` 端口是否空闲？`POSTGRES_PORT` 仅重映射宿主机侧端口，容器内部始终访问 `5432`。

**切勿要求用户在对话中粘贴明文密钥，也不要把密钥放进命令行参数中。** 应指导用户自行将其写入 values 文件或 `.env`。

## 使用 Helm 安装（官方方式）

正式版本发布在公开的 NGC `nvidia` 组织下（包含 Chart 与容器镜像）。请按照用户指定的版本号遵循 `DEPLOYMENT.md` 执行：

用户需在编辑器中创建 values 文件（不要通过命令行参数传递，以免密钥留存在 Shell 历史或进程参数中），限制为仅本人可读，并避免提交至 Git：

```yaml
# auto-ontology-values.yaml
defaultModelsApiKey: <API-KEY>
postgresPassword: <POSTGRES-PASSWORD>
adminEmail: <ADMIN-EMAIL>
adminPassword: <ADMIN-PASSWORD>
connectionStrings: <CONNECTION-STRINGS>   # 可选；也可在 UI 中添加连接
```

```bash
helm fetch https://helm.ngc.nvidia.com/nvidia/charts/auto-ontology-<VERSION>.tgz
helm install auto-ontology auto-ontology-<VERSION>.tgz -f auto-ontology-values.yaml
kubectl port-forward svc/frontend 3000:3000
```

- `adminEmail` 与 `adminPassword` 为必填项；缺失将导致安装失败。
- `appUrl` 必须与用户浏览器访问的精确 Origin 一致。默认值（`http://localhost:3000`）仅适用于上述 `port-forward` 方式；若使用 NodePort、Ingress 或 HTTPS 域名，请务必显式设置。
- `authSecret` 会在首次安装时自动生成并在升级时保留；仅当需要跨环境共享密钥时才手动指定。
- 若 `helm fetch` 返回 404，说明该版本尚未发布。请询问用户，切勿回退至内部预发仓库。

本技能不涵盖 Astra GitOps 流程。

## 本地运行：Docker Compose

适用于在单台机器上拉起完整服务栈。在仓库根目录，用户将 `.env.example` 复制为 `.env`，并填写必填项 `AUTH_SECRET`、`APP_URL`、`AUTO_ONTOLOGY_ADMIN_EMAIL` 与 `AUTO_ONTOLOGY_ADMIN_PASSWORD`。然后执行：

```bash
docker compose up -d --build
```

`AUTH_SECRET` 与 `APP_URL` 为必填项；若缺失，所有页面都会报 Better Auth `"default secret"` 错误。同时请填写 `DEFAULT_MODELS_API_KEY` 与 `CONNECTION_STRINGS`（或计划随后在 UI 中添加连接）。完整变量说明见 `.env.example`。

该命令会构建 `auto-ontology` 和 `auto-ontology-frontend`，启动 Postgres、pgAdmin 与摄取服务，运行两个一次性迁移任务后拉起主应用：

- `auto-ontology-migrate`：对后端的 `public` Schema（数据目录与语义层）执行 Alembic 迁移。后端容器会等待其完成。
- `frontend-migrate`：同步 Prisma 管理的 `frontend` Schema（用户、会话、API Key 及 MCP 登录使用的 OAuth 表）。前端容器会等待其完成。

如果普通登录或 MCP 登录失败，请执行 `docker compose ps -a` 和 `docker compose logs frontend-migrate` 检查。即使 `frontend-migrate` 退出码为 0 也建议查看其日志——历史上曾出现过它零报错退出但 MCP 登录表状态异常的情况。

| 服务 | 访问地址 |
| --- | --- |
| Web UI | http://localhost:3000 |
| FastAPI（内部） | http://localhost:3001 |
| 摄取服务 | http://localhost:3002 |
| pgAdmin | http://localhost:5050 |
| Postgres | localhost:`$POSTGRES_PORT`（读取 `.env`，默认 5432） |

## 开发者工作流（`--dev`、`--ds`）

仅供修改 Auto Ontology 自身源码的开发者使用，并非生产安装方式。`dev_tools/setup_env.sh` 会在 Docker 中启动部分组件，其余组件直接在本地源码目录运行。两种模式均需先准备上述 `.env` 并安装依赖：

```bash
cd frontend && pnpm install   # Next.js 依赖
# 在仓库根目录：
uv sync
```

`--dev` 仅在 Docker 中运行基础设施（Postgres、pgAdmin、摄取服务），你在本地运行 Next.js 与 FastAPI：

```bash
./dev_tools/setup_env.sh --dev
cd frontend && pnpm dev
# 仓库根目录：
uv run uvicorn auto_ontology.server.__main__:create_app --factory --reload --host 127.0.0.1 --port 3001
```

应用工厂函数是 `auto_ontology/server/__main__.py` 中的 `create_app()`（不存在 `auto_ontology/server/main.py`）。`uv run python -m auto_ontology.server` 启动同一个应用但不开启热重载（容器内即运行此命令）。

`--ds` 在 Docker 中运行前端，而在宿主机本地运行 FastAPI：

```bash
./dev_tools/setup_env.sh --ds
uv run python -m auto_ontology.server
```

前端镜像在构建时会写入 `PYTHON_API_URL=http://host.docker.internal:3001`，因此本地 API 必须接受来自容器的连接，而不能只监听回环地址。`python -m auto_ontology.server` 默认绑定 `0.0.0.0:3001`（这也是 `setup_env.sh --ds` 所提示的方式）；若在 Linux 上使用 `--host 127.0.0.1` 启动 uvicorn，容器将无法连通。由于 FastAPI 直接信任 `x-auto-ontology-user-id`，请务必做好防火墙隔离，禁止其他机器访问 3001 端口。原生 Linux Docker 环境下可能还需要添加 `--add-host=host.docker.internal:host-gateway`。

## 局限性

本技能严格基于仓库现有的 Helm Chart 与 Compose 文件，不涵盖 Astra GitOps，不负责修改语义定义，也不将不完整的 Vault 配置视为有效密钥存储。在执行任何破坏性数据卷删除操作前必须先征得用户确认。

## Auto Ontology 已运行 —— 仅接入 MCP

无需重新安装。只需将 `AUTO_ONTOLOGY_API_URL` 指向 **Web 应用地址**（Compose 下 UI 为 `:3000`，而非 FastAPI `:3001`）：

```bash
AUTO_ONTOLOGY_API_URL=http://localhost:3000 uvx --from "git+https://github.com/NVIDIA/auto-ontology.git#subdirectory=mcp" auto-ontology-mcp
```

在软件包尚未发布到 PyPI 前，这需要具备读取 `NVIDIA/auto-ontology` 的 GitHub 凭据。客户端配置中使用末尾带 `/mcp` 的服务器 URL。由用户在客户端弹出的登录页中自行登录，切勿在 MCP 服务端配置静态 Token。

确认目标部署足够新、支持作为 OAuth 授权服务器：

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server"
```

必须返回 `200`。如果返回其他状态码，请从当前仓库代码运行前端（`pnpm dev`）并将 `AUTO_ONTOLOGY_API_URL` 指向其端口。

## 示例

- Kubernetes：按 `DEPLOYMENT.md` 安装已发布的 Helm Chart，并正确配置管理员账号与 `appUrl`。
- 单机部署：填好 `.env` 后执行 `docker compose up -d --build`。
- 已有部署：无需重装；配置 `AUTO_ONTOLOGY_API_URL` 并在连接 MCP 客户端前先验证 OAuth 发现端点。

## 连接源数据库

数据源连接配置来自以下两个位置之一（且互斥生效）：

- **UI 管理连接**（Settings → Connections，或下列 API）：只要存在任何 UI 连接，`CONNECTION_STRINGS` 就会被忽略。
- **`.env` 中的 `CONNECTION_STRINGS`**：仅在没有任何 UI 连接时作为兜底使用。可通过 `GET /api/connections/source` 检查其是否已设置；它不支持 Schema 或表过滤规则。

通过 API 添加 UI 连接（需要 `connection:manage` 权限）：

1. `POST /api/connections/test` 验证凭据并返回可供白名单选择的 Schema 列表。若返回 422，响应中会包含数据库驱动报错，请先修复再创建。
2. `POST /api/connections` 保存连接。返回 201 仅代表配置已入库：后续摄取是异步尽力触发的，可能仍在运行或失败。
3. 验证是否摄取成功：`GET /api/datasources/dbs`（`catalog:read`）应列出目标库与预期表；`GET /api/semantic-compilation/status` 报告编译进度；MCP `check_readiness` 应报告无阻塞项。

作用域在单个连接上配置：Schema 白名单加上在目录提取阶段生效的 `table_allow_regex` / `table_deny_regex`（拒绝优先；匹配不带 Schema 的表名且区分大小写）。环境变量连接不具备这些过滤能力。完整的表级作用域强制隔离保证仍在 [issue #255](https://github.com/NVIDIA/auto-ontology/issues/255) 推进中，因此如果某张表绝对不能被编目、采样或生成向量嵌入，应如实告知当前版本尚不能端到端强制保证，不要误导用户。

## 验证就绪状态

1. 在 http://localhost:3000（或部署的 `APP_URL` / Helm `appUrl`）成功加载 UI 并显示左侧导航栏。若缺失导航栏，说明存在过期 Cookie，见 [troubleshooting.md](references/troubleshooting.md)。
2. 在 **Web 源站**请求 `GET /api/semantic-compilation/status` 能通过认证（Cookie、`x-api-key` 或 SSO Bearer）并返回 JSON。若 `calculated: false`，说明业务术语表为空——请先完成摄取/编译，不要反复重试提问。
3. MCP：上述 OAuth 发现端点返回 200；连接 MCP 客户端后调用 `check_readiness` 验证就绪。

## 停止与卸载

```bash
helm uninstall auto-ontology  # Helm；注意同时会删除 postgres-data PVC 及其中的数据
docker compose down           # Compose；保留数据卷
docker compose down -v        # Compose；彻底清除 Postgres / pgAdmin 数据卷
```

`helm uninstall` 与 `docker compose down -v` 都会销毁数据，执行前请先确认。

## 故障排查

遇到过期会话、模型密钥报错、向量模型不一致、端口冲突及部分 Vault 配置失效等问题，请查阅 [troubleshooting.md](references/troubleshooting.md)。对外分享命令输出作为排障证据前，请务必脱敏其中的密钥。

## 延伸阅读

- [troubleshooting.md](references/troubleshooting.md)
- [runtime-contract.yaml](../../../../skills/nvidia-ontology-setup/assets/runtime-contract.yaml)
- `nvidia-ontology-query` —— 服务运行后如何调用 Auto Ontology
- `nvidia-ontology-management` —— 如何编辑语义层
- `CLAUDE.zh-CN.md` —— 如何维护 Auto Ontology 仓库本身
