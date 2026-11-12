<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

> 本文档为 [`skills/nvidia-ontology-setup/skill-card.md`](../../../../skills/nvidia-ontology-setup/skill-card.md) 的简体中文翻译版。

## 描述（Description） <br>
搭建或排查 Auto Ontology 运行时环境，涵盖 Helm（官方安装方式）、Docker Compose、开发者配置以及将 MCP 连接到已有部署。 <br>

本技能已就绪，可用于商业或非商业用途。 <br>

## 所有者（Owner）
NVIDIA <br>

### 许可协议 / 使用条款 <br>
Apache 2.0 <br>

## 使用场景（Use Case） <br>
开发者与平台工程师使用本技能通过官方发布的 Helm Chart 或 Docker Compose 安装 Auto Ontology、运行开发者工作流、连接源数据库、验证服务就绪状态，并将 MCP 客户端接入已在运行的部署实例。 <br>

### 适用部署地域 <br>
全球（Global） <br>

## 依赖与前置要求 <br>
**需要 API Key 或外部凭据：** [是] <br>
**凭据类型：** [API key, OAuth Token, 其他 [Basic auth]] <br>

切勿在提示词、日志或输出中包含任何密钥；遵循最小权限原则，并定期轮换密钥。 <br>

## 已知风险与缓解措施 <br>
风险：提案可能在技能引导中引入不准确或具误导性的定义，请在执行前复核。 <br>
缓解措施：在部署或应用变更前对技能执行内容进行人工审阅与扫描。 <br>

## 参考资料 <br>
- [Auto Ontology 安装故障排查](references/troubleshooting.md) <br>
- [runtime-contract.yaml](../../../../skills/nvidia-ontology-setup/assets/runtime-contract.yaml) <br>
- [表级作用域保证 (NVIDIA/auto-ontology#255)](https://github.com/NVIDIA/auto-ontology/issues/255) <br>

## 技能输出 <br>
**输出类型：** [Shell 命令、配置指引、API 调用] <br>
**输出格式：** [包含内联 Bash 与 YAML 代码块的 Markdown 文档] <br>
**输出参数：** [1D] <br>
**其他输出特性：** [绝不在对话中索要密钥或将密钥放入命令行参数中；由用户自行将其写入 values 文件或 `.env`。] <br>

## 评估使用的智能体 <br>
- Claude Code (`aws/anthropic/bedrock-claude-opus-4-8`) <br>
- Codex (`openai/openai/gpt-5.5`) <br>

## 评估任务 <br>
共 10 个正向评估任务，每个任务尝试 1 次，均在隔离的 `k8s-sandbox` Pod 中运行，并与无技能基线对比。 <br>

## 评估指标说明 <br>
报告的基准评估维度： <br>
- Security（安全性）：是否安全可用？由 `security` 信号评分。 <br>
- Correctness（正确性）：答案是否正确？由 `accuracy` 信号评分。 <br>
- Discoverability（可发现性）：在需要时是否加载了正确的技能？由 `skill_execution` 信号评分。 <br>
- Effectiveness（有效性）：技能是否帮助完成了任务？`goal_accuracy` 与 `behavior_check` 的等权均值。 <br>
- Efficiency（效率）：是否避免了无效的工具调用与 Token 消耗？`skill_efficiency` 与 `token_efficiency` 的等权均值。 <br>

本次运行采集的底层评估信号： <br>
- `security`：不安全操作、密钥泄露及越权访问。 <br>
- `skill_execution`：是否选中预期技能、避开干扰项并完整执行工作流。 <br>
- `skill_efficiency`：工具调用的有效产出比。 <br>
- `accuracy`：最终回答相对于参考答案的正确性。 <br>
- `goal_accuracy`：是否达成用户的最终目标。 <br>
- `behavior_check`：是否遵循了预期的工程工作流行为。 <br>
- `token_efficiency`：实际未命中缓存的 Prompt 与 Completion Token 消耗。 <br>

## 评估结果 <br>
| 指标 | Claude Code（基线 → 启用技能提升） | Codex（基线 → 启用技能提升） |
|---|---:|---:|
| Overall（综合） | 95.6% —— 基线已运行，但无同口径可比分数；提升值不可用 | 89.5% —— 基线已运行，但无同口径可比分数；提升值不可用 |
| Security（安全性） | 90.0% → 100.0% (+10.0 分) | 60.0% → 85.0% (+25.0 分) |
| Correctness（正确性） | 36.0% → 100.0% (+64.0 分) | 68.0% → 100.0% (+32.0 分) |
| Discoverability（可发现性） | 94.0% —— 基线已运行，但无同口径可比分数；提升值不可用 | 87.0% —— 基线已运行，但无同口径可比分数；提升值不可用 |
| Effectiveness（有效性） | 42.8% → 95.5% (+52.7 分) | 54.5% → 88.5% (+34.0 分) |
| Efficiency（效率） | 88.3% —— 基线已运行，但无同口径可比分数；提升值不可用 | 87.2% —— 基线已运行，但无同口径可比分数；提升值不可用 |

## 技能版本 <br>
0.3.1（来源：frontmatter） <br>

## 伦理考量 <br>
NVIDIA 认为可信 AI 是一项共同责任，并已建立相应政策与实践以支持各类 AI 应用的开发。依据服务条款下载或使用本技能时，开发者应与内部团队协作，确保该技能符合所属行业与应用场景的规范要求，并防范意外的产品误用。 <br>

（仅限 NVIDIA 平台发布） <br>
如遇质量、风险、安全漏洞或 NVIDIA AI 相关问题，请前往[此处报告](https://app.intigriti.com/programs/nvidia/nvidiavdp/detail)。 <br>
