<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 数据源连接与凭证存储

> 本文档为 [`docs/connections.md`](../connections.md) 的简体中文翻译版。

Auto-Ontology 支持从两个位置读取源数据库配置：一是在 Web UI 中添加的连接，二是通过 `CONNECTION_STRINGS` 环境变量传入的连接。**每次部署请只使用其中一种方式**，因为两条代码路径对它们的优先级处理正好相反：

- **问题回答（Text-to-SQL）**：只要存在 UI 管理的连接，就优先使用 UI 连接；仅在 UI 连接为空时回退至 `CONNECTION_STRINGS`。
- **数据摄取（Ingestion）**：只要设置了 `CONNECTION_STRINGS`，就优先使用它；仅在该环境变量为空时才读取 UI 管理的连接。

## 在 UI 中添加连接

可通过 **Settings → Connections** 页面，或调用 API（先调用 `POST /api/connections/test` 测试，再调用 `POST /api/connections` 保存；需要 `connection:manage` 权限）添加连接。

每个连接都支持配置作用域限制：包括 Schema 白名单（Allowlist）以及 `table_allow_regex` / `table_deny_regex` 正则规则，这些规则在目录提取（Catalog Extraction）阶段生效。拒绝规则优先级高于允许规则（Deny wins），表名为不带 Schema 前缀的非限定名称且区分大小写。

连接凭证的存储位置取决于系统是否启用了 HashiCorp Vault：

- **未启用 Vault 时**：包含密码在内的完整连接对象将被序列化为 JSON，**以明文形式**存储在 Postgres 数据目录的数据库记录行中。
- **已启用 Vault 时**：凭证会按数据库名称作为 Key 写入 Vault，Postgres 的目录记录行中不保存凭证，数据库名称仅作为 Vault 的查找键。

## `CONNECTION_STRINGS` 环境变量

可在部署时提供以逗号或换行符分隔的连接 URL 列表（Docker Compose 中写在 `.env`，Helm 中使用 `--set connectionStrings=...`）。该方式不支持配置 Schema 或表过滤规则。可通过 `GET /api/connections/source` 查询当前是否已配置该变量。

连接 URL 的 Scheme 决定了所选用的连接器类型：

| 数据引擎 | URL Scheme | 是否支持在 UI 中添加 |
| --- | --- | :---: |
| PostgreSQL | `postgresql://`、`postgres://` | 是 |
| MySQL | `mysql://` | 是 |
| Databricks | `databricks://` | 是 |
| Snowflake | `snowflake://` | 是 |
| ClickHouse | `clickhouse://` | 是 |
| Trino | `trino://` | 是 |
| Apache Kyuubi | `kyuubi://` | 是 |
| HeavyDB | `heavydb://` | 是 |
| DuckDB | `duckdb://` | 否 |
| SQLite | `sqlite://` | 否 |

连接器注册表位于 [`auto_ontology/connectors/registry.py`](../../auto_ontology/connectors/registry.py)。

## HashiCorp Vault

Vault 默认处于关闭状态。只有当以下四个环境变量**全部设置**时才会启用：

- `VAULT_ADDR`
- `VAULT_NAMESPACE`
- `VAULT_ROLE_ID`
- `VAULT_SECRET_ID`

`VAULT_AUTH_MOUNT` 与 `VAULT_KV_MOUNT` 为可选覆盖参数。如果只配置了部分 Vault 变量，系统会记录一条告警并忽略 Vault 配置，凭证仍会回退为在 Postgres 中明文存储。
