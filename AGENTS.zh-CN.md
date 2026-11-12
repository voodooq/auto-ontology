<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# AGENTS.md — Auto-Ontology 智能体接入入口

> 本文档为 [`AGENTS.md`](./AGENTS.md) 的简体中文翻译版。

本目录提供的技能（Skills）用于教会智能体如何**安装部署**当前版本的 Auto-Ontology 以及**使用**其语义本体层。所有技能均位于仓库根目录的 **`skills/`** 下，每个技能对应一个独立目录，归属于持久命名空间 `nvidia-ontology*`。

本文档面向部署 Auto-Ontology 的开发者以及将其嵌入智能体框架（Agent Harness）的合作方。Auto-Ontology **项目维护者**（涉及构建、测试、Schema 变更、前端开发规范等）请继续遵循 [`CLAUDE.zh-CN.md`](./CLAUDE.zh-CN.md)（英文原版：[`CLAUDE.md`](./CLAUDE.md)）。请勿将维护者规范混入用户技能中。

## 技能选型指南

- [`skills/nvidia-ontology-setup/`](./skills/nvidia-ontology-setup/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-setup/SKILL.md)）— 涵盖 Helm（官方安装方式）、本地 Docker Compose、开发者 `--dev` / `--ds` 工作流、针对已运行实例配置 MCP，以及故障排查。当 Auto-Ontology 尚未启动或部署过程中出现异常时，请先阅读此技能。
- [`skills/nvidia-ontology-management/`](./skills/nvidia-ontology-management/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-management/SKILL.md)）— 通过业务术语（Glossary Terms）、SQL 属性、语义关系、模型 YAML 导入/导出以及受控结果定义来检视、建模、**修改**并安全发布语义层。由于 MCP 为只读接口，本技能涵盖所有受支持的写入路径。
- [`skills/nvidia-ontology-query/`](./skills/nvidia-ontology-query/)（[中文版](./docs/zh-CN/skills/nvidia-ontology-query/SKILL.md)）— 说明智能体如何调用 Auto-Ontology（优先 MCP，REST 作为兜底）、通过语义层理解业务含义，并对生成的 SQL、返回数据行和自然语言答案进行校验。当 Auto-Ontology 已部署就绪时，请从这里开始。

## 工作流归属与交接

| 用户操作 | 负责技能 |
| --- | --- |
| 安装、配置、连接数据源、摄取、编译、验证就绪状态 | `nvidia-ontology-setup` |
| 检视业务含义与语义关系 | `nvidia-ontology-management` |
| 设计概念、映射、关系、指标、单位、粒度与策略 | `nvidia-ontology-management` → `references/modeling.md` |
| 应用变更、校验、诚实认证、发布与回滚 | `nvidia-ontology-management` → `references/publication.md` |
| 提出描述性或诊断性问题，并校验 SQL、数据行与回答文本 | `nvidia-ontology-query` → `references/query-validation.md` |

完整的组合生命周期为：**安装/连接/就绪检查 → 检视/建模 → 分阶段应用/回读验证 → 查询校验 → 受控发布 → 新鲜查询回读验证**。若某一步骤缺乏产品接口支持，必须返回明确的能力缺口或回执（Receipt），严禁虚构不存在的操作。身份认证、权限、血缘溯源与错误处理属于横切关注点，而非彼此重叠的独立技能。

## 单一事实来源（严禁重复拷贝）

Skills 负责沉淀**操作流程、执行顺序与避坑指南**，绝对不能演变成第二份 API 文档或第二个安装脚本。

| 职责 | 事实来源（Owner） | Skills 应如何处理 |
| --- | --- | --- |
| 安装命令 | `DEPLOYMENT.md`（Helm 官方方案）、`docker-compose.yml`；开发者工作流参见 `dev_tools/setup_env.sh` | 直接调用它们；不要另写新安装器 |
| 实时只读工具 | `mcp/auto_ontology_mcp/tools.py`（握手工具列表） | 优先使用 MCP；按顺序读取；不要复制工具表格 |
| REST 请求格式与权限 | `docs/openapi/auto-ontology-api.json` | 仅说明操作名和所需权限；提供链接；不要直接粘贴 Schema |

MCP 在设计上严格保持只读。所有写入操作均通过 Next.js `/api/...` 接口完成。若技能文档与运行时的 MCP 握手工具列表存在差异，以实时握手返回的工具列表为准。

## 目录镜像同步（Catalog mirror）

`skills/` 下的目录会同步至 [github.com/nvidia/skills](https://github.com/nvidia/skills)（可通过 `npx skills add nvidia/skills` 安装）以及 `build.nvidia.com/skills`。该流水线要求每个技能目录包含：

- `SKILL.md`、`skill-card.md`、`evals/evals.json`（在本仓库中维护）
- `BENCHMARK.md` 与 `skill.oms.sig` — 由 NVSkills CI 自动生成。维护者在修改 `skills/` 的 PR 中评论 `/nvskills-ci` 后，`nv-skills-ci[bot]` 会自动推送一条 `"Attach NVSkills validation signatures"` 提交，该提交必须保留在 PR 中。切勿手工编辑或伪造这两个文件；如果后续在同一 PR 中再次改动了技能内容，需重新触发 NVSkills CI。
- 在 **nvidia/skills** 仓库的 `components.d/auto-ontology.yml` 中注册以下条目：

```yaml
name: Auto-Ontology
repo: NVIDIA/auto-ontology
description: Set up Auto-Ontology, model and publish its semantic layer, and run validated text-to-SQL queries through it.
skills:
  - path: skills/nvidia-ontology-setup/
    catalog_dir: nvidia-ontology-setup
  - path: skills/nvidia-ontology-management/
    catalog_dir: nvidia-ontology-management
  - path: skills/nvidia-ontology-query/
    catalog_dir: nvidia-ontology-query
```

即使 Auto-Ontology 本体仓库受访问限制，Skills 目录本身也是公开同步的，因此严禁在技能文件中出现预发环境主机名、集群名、Vault 路径或任何凭证。
