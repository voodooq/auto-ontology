<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 容器镜像第三方开源组件声明

> 本文档为 [`CONTAINER_THIRD_PARTY_NOTICES.md`](../../CONTAINER_THIRD_PARTY_NOTICES.md) 的简体中文翻译版（仓库中 `frontend/` 与 `mcp/` 目录下的同名文件与之内容完全一致）。

本文件说明 Auto-Ontology 容器镜像在基础镜像之上额外引入的组件。应用自身的开源依赖（Python 与 npm 软件包）单独列于 [`THIRD_PARTY_NOTICES.md`](../../THIRD_PARTY_NOTICES.md)。

每个镜像都会在 `/licenses/` 目录下打包以下文件：`LICENSE`、`THIRD_PARTY_NOTICES.md`、本文件以及 `OPEN_SOURCE_NOTICE`。

## 源代码归档

在基础镜像之上新增的开源组件源代码由 NVIDIA 归档于 <https://opensource.nvidia.com/oss/teams/nvidia>（如 `OPEN_SOURCE_NOTICE` 所述）。Auto-Ontology 自身源码位于 <https://github.com/NVIDIA/auto-ontology>。

## 基础镜像

三个镜像均基于 NVIDIA 认证的 Ubuntu 22.04 基础容器（OSRB Bug 3840915）构建，拉取自 NVIDIA 镜像仓库：

| 镜像 | 基础镜像 |
| --- | --- |
| `auto-ontology`（后端与摄取服务） | `nvcr.io/nvidia/base/ubuntu:jammy-20250619` |
| `auto-ontology-frontend` | `nvcr.io/nvidia/base/ubuntu:jammy-20250619` |
| `auto-ontology-mcp` | `nvcr.io/nvidia/base/ubuntu:jammy-20250619` |

基础容器的源代码随该基础镜像由 NVIDIA 一并提供。

## 在基础镜像之上新增的组件

### `auto-ontology`（后端与摄取服务）—— `Dockerfile`

| 组件 | 版本 | 许可协议 | 项目地址 |
| --- | --- | --- | --- |
| CPython（`python-build-standalone`，由 `uv` 安装） | 3.12 | PSF-2.0 及其打包库的许可协议 | https://github.com/astral-sh/python-build-standalone |
| `/opt/venv` 中的 Python 包 | 见 `uv.lock` | 见 `THIRD_PARTY_NOTICES.md`（Backend） | — |
| `ca-certificates`（Ubuntu 包） | jammy | MPL-2.0（证书数据）、GPL-2.0-or-later（脚本） | https://packages.ubuntu.com/jammy/ca-certificates |
| `libpq5`（Ubuntu 包） | jammy | PostgreSQL License | https://www.postgresql.org |
| `tini`（Ubuntu 包） | jammy | MIT | https://github.com/krallin/tini |

### `auto-ontology-frontend` —— `frontend/Dockerfile`

| 组件 | 版本 | 许可协议 | 项目地址 |
| --- | --- | --- | --- |
| Node.js，含内置 npm 与 Corepack（NodeSource 包） | 22.x | MIT 及其打包依赖的许可协议 | https://github.com/nodejs/node |
| Next.js standalone 打包产物中的 npm 包 | 见 `frontend/pnpm-lock.yaml` | 见 `THIRD_PARTY_NOTICES.md`（Frontend runtime） | — |
| Prisma CLI 与引擎（`/app/migrate`） | 7.7.0 | Apache-2.0 | https://github.com/prisma/prisma |
| dotenv（`/app/migrate`） | 17.4.2 | BSD-2-Clause | https://github.com/motdotla/dotenv |
| Geist 与 Geist Mono 字体（由 `next/font` 打包） | — | SIL Open Font License 1.1 | https://github.com/vercel/geist-font |
| `ca-certificates`（Ubuntu 包） | jammy | MPL-2.0（证书数据）、GPL-2.0-or-later（脚本） | https://packages.ubuntu.com/jammy/ca-certificates |
| `tini`（Ubuntu 包） | jammy | MIT | https://github.com/krallin/tini |

### `auto-ontology-mcp` —— `mcp/Dockerfile`

| 组件 | 版本 | 许可协议 | 项目地址 |
| --- | --- | --- | --- |
| CPython（`python-build-standalone`，由 `uv` 安装） | 3.12 | PSF-2.0 及其打包库的许可协议 | https://github.com/astral-sh/python-build-standalone |
| `/opt/venv` 中的 Python 包 | 见 `mcp/uv.lock` | 见 `THIRD_PARTY_NOTICES.md`（MCP server） | — |
| `ca-certificates`（Ubuntu 包） | jammy | MPL-2.0（证书数据）、GPL-2.0-or-later（脚本） | https://packages.ubuntu.com/jammy/ca-certificates |
| `tini`（Ubuntu 包） | jammy | MIT | https://github.com/krallin/tini |

仅在构建阶段使用的工具（`uv`、编译器、`curl`、`gnupg`、`pnpm`）存在于前置多阶段构建中，不会进入最终镜像。前端镜像保留了随 Node.js 附带的 npm 与 Corepack。

## 维护说明

每当任意 `Dockerfile` 变更其基础镜像、apt 软件包或安装的运行时环境时，均需同步更新本文件；依赖包变更则由 `THIRD_PARTY_NOTICES.md` 追踪。
