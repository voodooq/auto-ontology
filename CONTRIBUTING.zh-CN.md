<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# 参与贡献 Auto-Ontology

> 本文档为 [`CONTRIBUTING.md`](./CONTRIBUTING.md) 的简体中文翻译版。

**本项目目前不接受外部 Pull Request。** 代码变更均由 Auto-Ontology 维护团队完成。欢迎通过 GitHub Issues 提交集成需求与缺陷报告；若希望集成某个平台、应用或存储引擎，请使用 [集成请求（Integration Request）](https://github.com/NVIDIA/auto-ontology/issues/new?template=integration.yml) 模板。这与 README 中的 [集成与合作](README.zh-CN.md#集成与合作) 章节说明一致。

**请勿通过 GitHub Issues 上报安全漏洞。** 请参阅 [SECURITY.zh-CN.md](SECURITY.zh-CN.md) 了解 NVIDIA 的协调漏洞披露流程。

本文件其余部分面向 Auto-Ontology 维护者。

## 开发环境搭建

Auto-Ontology 使用 [uv](https://docs.astral.sh/uv/) 进行 Python 依赖管理：

```bash
uv sync                # 安装依赖
uv run pytest          # 运行测试套件
uv run ruff check .    # 代码 lint 检查
```

构建与数据库迁移目标可查看 `Makefile`（运行 `make help`），本地开发服务栈启动配置参见 `docker-compose.yml`。

## Pull Request 规范

1. 从 `main` 分支创建新分支。
2. 为新增功能补充测试用例，并确保全量测试套件通过。
3. 保持 PR 职责单一——每个 PR 只做一项逻辑变更。
4. 合并至 `main` 的 Pull Request 必须先获得**至少一位 Code Owner 的批准审查**。系统会自动请求 Code Owner（见 `.github/CODEOWNERS`）进行审查；由于 Code Owner 不能批准自己的 PR，因此需要由其他 Owner 之一完成审批。
5. 新增文件必须包含全仓库统一的 NVIDIA 版权与 SPDX 许可声明头。

## 开源协议

Auto-Ontology 基于 [Apache License 2.0](LICENSE) 协议开源。
