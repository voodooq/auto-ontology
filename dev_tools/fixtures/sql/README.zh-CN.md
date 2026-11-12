<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 测试夹具数据库（Fixture databases）

> 本文档为 [`dev_tools/fixtures/sql/README.md`](./README.md) 的简体中文翻译版。

确保 Postgres 已启动后，一条命令即可构建全部测试夹具数据库：

```bash
docker compose up -d postgres
uv run --no-sync python -m dev_tools.fixtures.seed_fixtures
```

该脚本具有幂等性——重复运行会自动跳过已加载的库。如需从零重建 Postgres 夹具库，请先删库（Drop Database）；对于 SQLite，直接删除对应的 `.sqlite` 文件即可。

| 夹具库 | 引擎 | 来源 | 协议 |
| --- | --- | --- | --- |
| `pagila` | Postgres | [devrimgunduz/pagila] `pagila-v3.1.0` 裁剪版 | MIT |
| `pagila_analytics` | Postgres | Auto-Ontology 为 `pagila` 补充编写的分析 Schema | — |
| `chinook` | SQLite | [lerocha/chinook-database] 原版 | MIT |

[devrimgunduz/pagila]: https://github.com/devrimgunduz/pagila
[lerocha/chinook-database]: https://github.com/lerocha/chinook-database

## 为什么选择这些库，而不是手写小型 Schema？

目录提取（Catalog）测试的核心评判标准是**高保真度**：代码能否准确发现相同的表、列、数据类型与外键？一个手写的玩具 Schema（几张表、一个外键、没有视图）根本无法验证这一点。而 Pagila 覆盖了代码中需要特殊处理的各种真实结构：

| 数据库特性 | 覆盖验证的逻辑 |
| --- | --- |
| 7 个视图 + 1 个物化视图（`rental_by_category`） | `TableTypes` —— `base table` / `view` / `materialized view`。此前测试仅覆盖普通基础表。 |
| 按月拆分为 22 个子分区的 `payment` 表 | 分区子表会出现在 `information_schema` 中，迫使数据目录逻辑明确是否应当展示分区。 |
| `film.special_features text[]`、`mpaa_rating` 枚举、`fulltext tsvector` | 非标量列 `data_type` 的类型解析处理。 |
| 2 个 Domain 类型，含 `bıgınt`（土耳其语无点 i） | 非 ASCII 标识符在全链路往返中的保真度。 |
| `film_actor`、`film_category` 多对多中间表 | 桥接表检测（`auto_ontology/semantic/bridge_tables.py`）。 |
| `customer → address → city → country` | 多跳关联路径查找（`find_join_path`）。 |
| 两个 Schema：`public` 与 `analytics` | `Schema` 层级与基于 Schema 的域（Zone）隔离。 |
| 2 条跨 Schema 外键 | 跨越 Schema 边界的 Join 路径解析。 |
| 1000 部电影 / 4581 条库存记录 | 样本少于几行时，`store_column_sample_values` 与 `store_column_uniqueness` 统计不具备代表性。 |

引入 `chinook` 则是为了让测试集具备**多数据库与多 SQL 方言**覆盖能力。以下行为仅靠单数据库无法测试：`find_join_path` 中的跨库拒绝逻辑（`auto_ontology/dal/attributes.py:404`）、`delete_by_database` 作用域隔离、按数据库维度的 `pgvector` 集合重置，以及跨数据库的 Zone 划分。此外，它还能驱动 `auto_ontology/connectors/sqlite.py` 和第二种 `sqlglot` 方言解析路径，并包含 Pagila 中缺失的自引用外键（`Employee.ReportsTo`）。

## `pagila.sql` 的生成方式

仓库中的 `pagila.sql` 并非上游原样文件，而是通过 `dev_tools/fixtures/sql/build_pagila.sh` 加工生成，可按以下方式复现：

1. **锁定 `pagila-v3.1.0`，而非 `master`。** 上游 `master` 分支面向 PostgreSQL 18，使用了 `uuidv7()` 默认值与 `VIRTUAL` 生成列，这两者均不被 PG17 支持。而 Auto-Ontology 运行在 `pgvector/pgvector:pg17` 上。`v3.1.0` 是无需修改即可在 PG17 上加载的最新 Tag，同时保留了分区表、Domain、枚举类型与全部 8 个视图。锁定版本比维护本地补丁更稳妥。
2. **数据裁剪。** 按顺序依次执行 `DELETE FROM payment WHERE rental_id > 600` 和 `DELETE FROM rental WHERE rental_id > 600`（保证不破坏外键约束），其余数据原封不动。这样在完整保留所有表、视图、类型和约束的同时，将 SQL 转储体积从约 13 MB 缩减至约 1.2 MB，且校验后无任何孤儿行。
3. **导出为 `INSERT` 而非 `COPY`。** `pg_dump` 默认生成的 `COPY ... FROM stdin` 依赖 `psql` 的 COPY 协议，而种子脚本使用的 `psycopg` `cur.execute()` 无法执行该协议。通过 `--rows-per-insert=200` 可避免单行一条 `INSERT` 带来的体积膨胀。
4. **剔除 `\restrict` / `\unrestrict`。** `pg_dump` 17.6+ 会输出这些 `psql` 元命令，它们不是合法 SQL，`psycopg` 执行时会报错。

## 对早期假设的修正记录

此前重构方案在未核实前曾提出以下假设，经实际验证发现有三处有误，特此记录：

- Pagila **没有自引用外键**。库中并不存在 `staff.reports_to`（那是 Sakila/Northwind 里的表结构）。同 Schema 自引用由 Chinook 的 `Employee.ReportsTo` 覆盖，非 `public` Schema 的自引用则由 `analytics.category_rollup.parent_rollup_id` 覆盖。
- 上游 Pagila 原版**只有单个 Schema**。第二个 Schema 来自本仓库补充编写的 `pagila_analytics.sql`——详见该文件头部的注释，说明为何缺少它就无法测试 `Schema` 层级及 `find_join_path` 中的 `-Schema` 排除逻辑。
- Pagila 共有 **22 张基础表，而不是 15 张** —— 差额正来自 `payment` 的按月分区子表。
