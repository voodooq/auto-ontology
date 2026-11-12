---
name: nvidia-ontology-management
description: >-
  在 Auto Ontology 中建模并发布语义定义。适用于术语、语义关系、指标、模型导入与受控结果管理——不用于部署或日常查询。
license: Apache-2.0
metadata:
  version: "0.2.1"
  author: "NVIDIA <opensource@nvidia.com>"
  tags:
    - nvidia-ontology
    - ontology
    - glossary
    - sql-attributes
    - catalog
---

<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Auto Ontology 管理与建模

> 本文档为 [`skills/nvidia-ontology-management/SKILL.md`](../../../../skills/nvidia-ontology-management/SKILL.md) 的简体中文翻译版。

## 目的

在不混淆“API 调用成功”与“业务语义正确/持久化发布”的前提下，检视并修改 Auto Ontology 的语义层。机器可读的输入、输出、状态、门禁与交接契约定义在 [runtime-contract.yaml](../../../../skills/nvidia-ontology-management/assets/runtime-contract.yaml) 中。

开始操作前，请先选择对应的专项工作流：

- 日常检视或元数据编辑：继续阅读下文，并配合使用 [write-api.md](references/write-api.md)；
- 概念、关系、指标、单位、粒度或策略设计：请使用 [modeling.md](references/modeling.md)；
- 模型晋级（Promotion）或可复用分析结果发布：请使用 [publication.md](references/publication.md)。

MCP 工具**仅支持只读操作**。创建、更新（Patch）、导入与编译重置必须经由 Next.js `/api/...` 网关发起。受控结果行需要经过授权的源端写入器（Source Writer）；Auto Ontology 本身没有通用的结果行回写接口。请求体与字段类型定义在 `docs/openapi/auto-ontology-api.json`；本技能仅列出操作名称与权限要求。

## 前置条件

需要一个正在运行且已完成认证的 Auto Ontology 部署。如果部署或语义层尚未就绪，请先使用 `nvidia-ontology-setup`。

## 局限性

MCP 严格只读，Auto Ontology 没有通用的结果行回写端点，且当前的模型导入与认证字段 API 尚不具备原子化晋级（Atomic Promotion）能力。请严格遵循下文的发布门禁，并在无法获得精确回读验证时按“失败关闭（Fail Closed）”处理。

## 身份认证

脚本调用请使用 API Token（`x-api-key` 或 `Authorization: Bearer`），可在 Web UI（右上角用户菜单 → API Tokens）中签发。Token **完全继承所属用户的权限**——观察者（Viewer）的 Token 无法执行管理员操作。创建和吊销 Token 需要已登录的浏览器会话，Token 本身不能再签发其他 Token。

除非另有注明，下述写入操作均需要 `catalog:edit` 权限。返回 `403` 表示 Token 所属用户角色缺少该权限，而非请求路径有误。

## 操作指引

1. **发现当前语义定义**：通过 MCP（`search_terms`、`get_term`、`get_term_columns`、`get_term_sql_attributes`、`describe_table`）获取；若无 MCP 环境，则通过 REST 接口（`GET /api/terms`、`GET /api/terms/{term_id}`、`GET /api/exploration/tables/{table_id}/details`）。
2. **明确语义契约**：当业务含义发生变化时，务必记录源数据绑定、业务标识、统计总体（Population）、关系基数（Cardinality）、指标表达式、单位、粒度、分母、时间口径、有效期以及反例校验；详见 [modeling.md](references/modeling.md)。
3. **向用户声明拟议变更**（例如术语重命名、新增 SQL 属性、模型导入等）。严禁静默重写业务术语表。
4. **在创建/更新前校验 SQL**：调用 `POST /api/sql-attributes/validate`。注意语法解析失败会返回 **HTTP 422**，而不是 `valid: false`。
5. **实施最小粒度写入**：仅执行满足需求的最小修改（详见 [write-api.md](references/write-api.md)）。
6. **编译不是隐式副作用**：先检查 `GET /api/semantic-compilation/status`。**切勿**将 `POST /api/semantic-compilation/reset` 当作常规清理操作——这是一个异步、破坏性的全库语义层重建接口，返回 `202`，且需要 `semanticCompilation:manage` 权限。
7. **验证已持久化的实际变更**，而不仅仅是请求返回成功。模型导入必须保留作用域备份并做精确的重新导出比对（见 [publication.md](references/publication.md)）。随后使用 MCP `check_answerable` / `ask_question`（或 REST `POST /api/question-entity-coverage` 与 `POST /api/chat/completions`）进行正反例行为校验。

## 语义关系与“数据集代表什么含义”

始终基于语义层工作，不要直接浏览原始物理 Schema 来回答业务含义问题。

- 术语 → 物理列：MCP `get_term_columns` 或 `GET /api/terms/{term_id}/column-attributes`
- 术语 → 派生 SQL：MCP `get_term_sql_attributes`
- 表 → 业务术语与 SQL 属性：MCP `describe_table` 或 `GET /api/exploration/tables/{table_id}/details`
- 两术语之间的语义跳数路径：`GET /api/exploration/terms/{term_id}/path/{other_term_id}`
- 语义图谱：`GET /api/exploration/graph`、`GET /api/exploration/semantic-graph`

以上返回的是语义关联路径，而非数据生成血缘、源码版本、认证审批历史或版本谱系。

## 批量导入与导出

- `POST /api/model/export` —— 导出数据目录 + 语义层的 YAML。需要 `modelInterchange:export` 权限。请求体可在 `databases` 中指定目录数据库 **ID** 列表（留空表示全部），并指定 `format`（`auto_ontology` 或 `ossie`）。
- `POST /api/model/import` —— 通过 `multipart` 上传 YAML；支持原生 Auto Ontology 格式（顶层含 `data_layer` / `semantic_layer`）或 Apache Ossie 格式（顶层含 `version`、`name`、`datasets`）。查询参数 `replace`（默认 `true`）与 `embed`（默认 `true`）。需要 `modelInterchange:import` 权限。由于默认可能覆盖现有数据，执行 `replace=true` 前请务必与用户确认，先在隔离环境验证，并严格遵循 [publication.md](references/publication.md) 中的精确回读流程。

## 示例

- 定义一个执行粒度（Run-grain）的指标时，请遵循 [modeling.md](references/modeling.md) 中的证据链与反例验证流程。
- 修订并晋级模型时，请备份精确的作用域，并遵循 [publication.md](references/publication.md) 中的分阶段回读流程。

## 故障排查

写入失败时，请先核验当前认证用户所属角色的权限，区分 HTTP 422 语法解析失败与校验逻辑结果，并在修改模型或重置编译前先查看编译状态。

## 延伸阅读

- [write-api.md](references/write-api.md)
- [modeling.md](references/modeling.md)
- [publication.md](references/publication.md)
- `nvidia-ontology-query` —— 如何调用 Auto Ontology 并校验查询结果
- `nvidia-ontology-setup` —— 部署尚未就绪时的安装与排错指南
- `mcp/auto_ontology_mcp/tools.py` —— 线上只读工具白名单
