<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 受控发布、回读校验与回滚（Governed publication, readback, and rollback）

> 本文档为 [`skills/nvidia-ontology-management/references/publication.md`](../../../../../skills/nvidia-ontology-management/references/publication.md) 的简体中文翻译版。

当需要应用已审核的语义模型或将经过验证的分析结果固化为可复用资产时，请使用本工作流。**发布（Publication）指的是受控的持久化写入并伴随精确回读验证。** 一段对话回复、图表、本地回执或 HTTP 请求成功，都不等同于发布完成。

MCP 严格只读，无法执行发布。必须使用经批准的 REST 写入接口或源端写入器，并以调用者自身权限运行。发布可以暴露某项推荐结果，但绝不能在缺乏独立授权事件的情况下隐含“已批准”、“已承诺”或“已执行”。

## 选择诚实的发布模式

- **仅留存本地回执（Receipt only）**：当不存在受批写入器时，将已验证的产物保留在本地，并明确向用户说明尚未发生远端发布。
- **原生语义模型（Native semantic model）**：使用指定作用域的 Auto Ontology 模型导出/导入，维护 Terms、属性、代表表及语义关系。
- **可复用分析定义（Reusable definition）**：对于受支持的 SQL 使用经校验的 `CustomAnalysis`，对于受支持的 PQL 使用 `PqlAnalysis`。注意它们存储的是**查询定义**，而非任意外部结果行。
- **受控结果表（Governed result table）**：将不可变的、带主键的结果分区写入 Auto Ontology 外部受批的源表中，随后再摄取并建模该表以供复用。Auto Ontology 不提供通用结果行回写的 MCP 或 REST 端点。

**切勿将对话历史持久化机制当作结果存储表使用。** 如果上述模式均无法满足要求的持久性、身份标识或权限规范，请停留在本地回执阶段并说明原因。

## 发布前契约检查

发布前必须绑定并校验以下要素：

- 数据源、数据目录与语义模型标识；
- 精确的物理 ID、语义 ID 与经批准的 Join 路径；
- 运行批次（run）、结果（result）、实体（entity）、场景（scenario）与父级标识；
- 统计总体、指标定义、单位、粒度与时间基准；
- 生产者/模型/求解器标识，以及输入/定义/结果的哈希值；
- 证据类别、不确定性、有效期、状态与替代关系；
- 写入者身份、授权作用域、目标位置与回滚目标。

务必严格区分已观测（observed）、已计算（computed）、结构性（structural）、建模得出（modeled）、预测（predicted）、推荐（recommended）与已承诺（committed）这几类不同性质的证据。当下游 Join 需要完整集合时，应按运行粒度存储批次总计并保留完整的场景成员关系。

## 原生语义模型变更流程

1. 导出精确的数据库 ID 作用域，并将原始 YAML 及其 SHA-256 哈希保存为回滚基线产物。
2. 保留线上已有的物理 ID；新生成的 UUID 不能作为自然名称的 Upsert 键。
3. 校验拟议的变更 Diff 及受影响的查询。
4. 先通过公开 API 在隔离环境中试应用。
5. 重新导出同一作用域，精确比对 ID、结构、属性、关系与条目数量是否完全一致。
6. 运行正向与负向的就绪检查、可回答性检查、SQL 校验、结果行校验及问答探针。
7. 仅晋级通过审核的产物；一旦回读不匹配，立即隔离该产物并在受支持范围内恢复并验证先前的状态。

针对已知的产品局限，必须采用“失败关闭（Fail Closed）”策略：

- [issue #250](https://github.com/NVIDIA/auto-ontology/issues/250)：替换导入时可能会保留已被省略的语义外键；
- [issue #265](https://github.com/NVIDIA/auto-ontology/issues/265)：替换导入时可能会保留既有属性而不是按新文件更新；
- [issue #267](https://github.com/NVIDIA/auto-ontology/issues/267)：物理 ID 不同但自然数据库名相同时可能会触发 HTTP 500。

**严禁通过直接修改 Postgres 数据库来绕过上述问题。** 仅凭接口返回 `success: true` 或摘要计数一致，并不能证明数据库中的持久化状态完全一致。

## 受控分析结果表发布

对于受控结果表，请使用稳定的 `(run_id, result_id)` 与业务实体键、仅追加（Append-only）的运行分区或事务性暂存表（Staging），并显式记录单位、粒度、源/父级血缘、状态、有效期与替代关系。摄取前先核验行数、键唯一性、哈希值与全精度数值。

摄取完成后，通过类型兼容的键将结果重新关联回原始业务实体，编译语义层，并从已发布的目标表中执行全新查询完成回读验证。务必测试无效、过期、已被替代、未知批次以及指标歧义用例——**系统绝不能静默回退到最新一行。**

## 必备回执内容

回执中应留存：请求作用域、语义输入标识、写入器类别、权限校验结果、目标位置或定义 ID、载荷/产物哈希、变更前后的精确 Diff、行/键/哈希校验结果、编译标识、写入后执行的 SQL 与返回行，以及回滚目标。**严禁在回执中留存 Token、密码或连接字符串。**

当前 Auto Ontology 的认证（Certification）字段尚不构成一套原子化的审核/晋级系统。在该产品生命周期完备之前，请如实描述已审核状态，不要宣称具备原子化认证、晋级、替代或自动回滚能力。
