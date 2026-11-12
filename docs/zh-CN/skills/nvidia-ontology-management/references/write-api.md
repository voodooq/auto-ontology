<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 本体写入 API（速查索引）

> 本文档为 [`skills/nvidia-ontology-management/references/write-api.md`](../../../../../skills/nvidia-ontology-management/references/write-api.md) 的简体中文翻译版。

权威的请求/响应数据结构定义在 `docs/openapi/auto-ontology-api.json` 中，此处不再重复粘贴 Schema。每个操作的所需权限见规范中的 `x-auto-ontology-permissions` 字段。

请始终调用 **Next.js 网关源站**（`APP_URL`，本地为 `:3000`），切勿直接调用 FastAPI `:3001`。

## 业务术语（Terms）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| GET | `/api/terms` | `catalog:read` | 支持查询参数 `query`、`skip`、`limit` |
| GET | `/api/terms/{term_id}` | `catalog:read` | 获取单个术语详情 |
| PATCH | `/api/terms/{term_id}` | `catalog:edit` | `TermUpdate`：支持 `name`、`description`、`name_certified`、`description_certified`。重命名会使已缓存的 SQL 属性描述建议失效。 |

`/api/terms` 上不提供公开的“创建空术语” `POST` 接口。新术语通过语义编译、数据摄取或模型导入生成。

## 列属性（Column attributes）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| GET | `/api/terms/{term_id}/column-attributes` | `catalog:read` | 查询某术语映射到的底层物理列 |
| PATCH | `/api/terms/{term_id}/column-attributes/{attr_id}` | `catalog:edit` | 更新名称/描述/认证状态/`sample_values`。`{attr_id}` 是 ColumnAttribute ID，而非 SqlAttribute ID。会触发语义向量刷新。 |

## SQL 属性（SQL attributes）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/sql-attributes/validate` | `catalog:edit` | 请求体：`expression`，可选 `term_id`、`attribute_id`。解析失败返回 **422**，而非 `valid: false`。 |
| POST | `/api/sql-attributes` | `catalog:edit` | `SqlAttributeCreate`：必填 `name`、`description`、`expression`、`term_id`；可选 `source`（默认 `manual`）。成功返回 201。 |
| GET | `/api/sql-attributes/{attr_id}` | `catalog:read` | 等效于 MCP 工具 `get_sql_attribute` |
| PATCH | `/api/sql-attributes/{attr_id}` | `catalog:edit` | 仅更新元数据（name/description），**不会**修改 SQL 表达式。 |
| PUT | `/api/sql-attributes/{attr_id}` | `catalog:edit` | 重新解析 SQL 表达式并刷新向量嵌入，必填字段同创建接口。 |
| DELETE | `/api/sql-attributes/{attr_id}` | `catalog:edit` | 清理图谱节点并删除对应向量嵌入 |
| GET | `/api/terms/{term_id}/sql-attributes` | `catalog:read` | 列出指定术语下的全部 SQL 属性 |
| GET | `/api/sql-attributes/{attr_id}/description-suggestion` | `catalog:edit` | 为编辑流程提供 LLM 自动生成的描述建议 |

## 目录节点描述（Catalog node descriptions）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| PATCH | `/api/nodes/{node_id}` | `catalog:edit` | 修改 Database、Schema、Table 或 Column 节点的可变属性 |

## 语义编译（Compilation）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| GET | `/api/semantic-compilation/status` | `chat:use` | 全局状态。`calculated: true` 仅代表至少存在一个 Term；`running: false` 并不能证明摄取服务在线可达。 |
| PUT | `/api/configurations/semantic-compilation` | `semanticCompilation:manage` | 设置 `enabled` 开关；开启时会尽力异步触发一次编译。 |
| POST | `/api/semantic-compilation/reset` | `semanticCompilation:manage` | **破坏性异步操作。** 接收删除并重建所有数据库语义层的任务，返回 202 不代表完成，需轮询并验证。切勿将其用于小修后的常规清理。 |

## 模型导入/导出（Model interchange）

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/model/export` | `modelInterchange:export` | JSON 请求体 `ExportRequest`：在 `databases` 中传入目录数据库 **ID**（空表示全部），`format` 可选 `auto_ontology` \| `ossie`。返回 YAML。 |
| POST | `/api/model/import` | `modelInterchange:import` | YAML 导入；请求表单与查询参数见 OpenAPI 规范。**避坑提示**：`replace` 默认为 `true`，覆盖现有定义前务必先与用户确认。 |

变更保护与发布门禁请参阅 [publication.md](publication.md)。接口请求成功不等于数据库状态已精确同步。

## 可复用分析定义（Reusable analysis definitions）

这些接口用于发布可复用的**查询定义**，而非任意外部结果行：

| 方法 | 路径 | 所需权限 | 说明 |
| --- | --- | --- | --- |
| POST | `/api/custom-analyses/validate` | `analysis:manage` | 基于已编目表校验 SQL；解析或引用失败返回 422。 |
| POST/PUT/DELETE | `/api/custom-analyses` 或 `/api/custom-analyses/{analysis_id}` | `analysis:manage` | 创建、替换或删除受支持的 SQL 定义。名称或 SQL 冲突返回 409。 |
| GET | `/api/custom-analyses` | `analysis:read` | 列出可复用 SQL 分析定义。 |
| POST/PUT/DELETE | `/api/pql-analyses` 或 `/api/pql-analyses/{analysis_id}` | `analysis:manage` | 创建、替换或删除 PQL 定义。PQL 在预测执行时校验。 |
| GET | `/api/pql-analyses` | `analysis:read` | 列出可复用 PQL 定义。 |

## 语义探索接口（只读；语义关系）

以下接口均需要 `catalog:read` 权限：

- `GET /api/exploration/tables/{table_id}/details`
- `GET /api/exploration/terms/{term_id}/details`
- `GET /api/exploration/terms/{term_id}/path/{other_term_id}`
- `GET /api/exploration/graph`
- `GET /api/exploration/semantic-graph`
- `GET /api/exploration/edges`
- `GET /api/exploration/nodes/{node_id}/relationships`

## 写入后验证

执行模型导入后，首先重新导出同一数据库 ID 作用域并逐项比对持久化的结构与属性；随后使用 MCP `check_answerable` 与 `ask_question` 进行正向和负向问答测试。对应的 REST 接口为：

- `POST /api/question-entity-coverage`
- `POST /api/chat/completions`（SSE 流式响应；需 `chat:use` 权限）
