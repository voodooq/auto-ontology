<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# auto-ontology-mcp

> 本文档为 [`mcp/README.md`](./README.md) 的简体中文翻译版。

这是面向 [Auto-Ontology](https://github.com/NVIDIA/auto-ontology) 的 [MCP（Model Context Protocol）](https://modelcontextprotocol.io) 服务器。它使任意支持 MCP 的智能体（如 Cursor、Claude Desktop 或企业内部智能体）能够使用自然语言查询 Auto-Ontology 所连接的数据，并检视支撑回答的语义层。

## 安装与运行

`uvx` 可直接从代码仓库拉取、构建并运行，无需事先克隆仓库：

```sh
uvx --from "git+https://github.com/NVIDIA/auto-ontology.git#subdirectory=mcp" auto-ontology-mcp
```

如果在本地检出的 `mcp/` 目录下运行，可执行：

```sh
uvx --from . auto-ontology-mcp
```

> [!NOTE]
> 目前 MCP 软件包需从 GitHub 安装。只有在正式发布至 PyPI 后，单独执行 `uvx auto-ontology-mcp` 才能直接解析。

只需运行一个服务实例，不同用户均可经由它完成登录。唯一必须传入的是你的 Auto-Ontology 部署 URL：

```sh
AUTO_ONTOLOGY_API_URL=https://auto_ontology.example.com auto-ontology-mcp
```

## 连接客户端

在 Cursor（`~/.cursor/mcp.json`）或 Claude Desktop（`claude_desktop_config.json`）中配置服务 URL 即可，配置文件中无需写入任何认证凭据：

```json
{
  "mcpServers": {
    "auto-ontology": {
      "url": "https://auto-ontology-mcp.example/mcp"
    }
  }
}
```

客户端会主动提示用户登录，随后弹出 Auto-Ontology 的常规登录页面，登录后的所有工具调用都将以该登录用户的身份执行。

随后即可尝试提问，例如：*"what does Auto-Ontology mean by an active customer, and how many were there last quarter?"*（“在 Auto-Ontology 中‘活跃客户’是如何定义的？上季度有多少位？”）

## 工具清单

`ask_question` 是核心问答工具：它会运行 Auto-Ontology 的 Text-to-SQL 智能体，并返回自然语言回答、实际执行的 SQL 以及查询结果行。其余工具——包括 `search_terms`、`describe_table`、`check_answerable` 等——用于帮助智能体预先理解业务词汇表并校验前提假设。

所有工具均为**只读**，且全部通过语义层工作：系统刻意不提供用于浏览原始数据库、Schema 或物理列的工具。任何工具都不会修改术语表、数据目录或底层数据库。

## 注意事项

本服务器只是 Auto-Ontology API 的普通 HTTP 客户端，因此既不需要数据库连接凭据，也可以部署在任意能访问 Auto-Ontology 部署的网络位置。它本身也不存储任何认证密钥——每位调用者由 Auto-Ontology 亲自认证登录——因此永远不会拥有超出调用者本人权限的访问能力。

完整文档（包含全部配置环境变量、完整工具列表以及如何扩展工具集）请参阅 [`docs/zh-CN/mcp.md`](../docs/zh-CN/mcp.md)（英文原版：[`docs/mcp.md`](../docs/mcp.md)）。
