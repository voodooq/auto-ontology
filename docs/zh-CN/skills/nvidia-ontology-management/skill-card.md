<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

> 本文档为 [`skills/nvidia-ontology-management/skill-card.md`](../../../../skills/nvidia-ontology-management/skill-card.md) 的简体中文翻译版。

## 描述（Description） <br>
在 Auto Ontology 中建模并发布语义定义，覆盖业务术语、语义关系、指标、模型导入与受控结果管理（不涵盖部署与查询）。 <br>

本技能已就绪，可用于商业或非商业用途。 <br>

## 所有者（Owner）
NVIDIA <br>

### 许可协议 / 使用条款 <br>
Apache 2.0 <br>

## 使用场景（Use Case） <br>
开发者与数据工程师使用本技能让智能体检视并修改 Auto Ontology 语义层：编辑业务术语与 SQL 属性、定义锚定源数据的概念/关系/指标、执行带精确回读校验的模型导入与导出，以及发布受控定义或分析结果。 <br>

### 适用部署地域 <br>
全球（Global） <br>

## 依赖与前置要求 <br>
**需要 API Key 或外部凭据：** [是] <br>
**凭据类型：** [API key] <br>

切勿在提示词、日志或输出中包含任何密钥；遵循最小权限原则，并定期轮换密钥。 <br>

## 已知风险与缓解措施 <br>
风险：提案可能在技能引导中引入不准确或具误导性的定义，请在执行前复核。 <br>
缓解措施：在部署或应用变更前对技能执行内容进行人工审阅与扫描。 <br>

## 参考资料 <br>
- [本体写入 API 索引](references/write-api.md) <br>
- [锚定源数据的本体建模](references/modeling.md) <br>
- [受控发布、回读校验与回滚](references/publication.md) <br>
- [运行时契约](../../../../skills/nvidia-ontology-management/assets/runtime-contract.yaml) <br>

## 技能输出 <br>
**输出类型：** [API 调用、分析报告、配置指引] <br>
**输出格式：** [针对 Next.js `/api` 网关的内联 HTTP 请求示例的 Markdown 文档] <br>
**输出参数：** [1D] <br>
**其他输出特性：** [MCP 访问为只读；写入操作受 Token 所属用户权限约束；不支持的发布模式将在返回明确回执后停止] <br>

## 评估使用的智能体 <br>
- Claude Code (`aws/anthropic/bedrock-claude-opus-4-8`) <br>
- Codex (`openai/openai/gpt-5.5`) <br>

## 评估任务 <br>
共 13 个正向评估任务，每个任务尝试 1 次，均在隔离的 Kubernetes 沙盒 Pod 中运行，并与无技能基线（No-skill baseline）对比。 <br>

## 评估指标说明 <br>
报告的基准评估维度： <br>
- Security（安全性）：是否安全可用？（基于 `security` 打分） <br>
- Correctness（正确性）：答案是否正确？（基于 `accuracy` 打分） <br>
- Discoverability（可发现性）：在需要时是否加载了正确的技能？（基于 `skill_execution` 打分） <br>
- Effectiveness（有效性）：技能是否帮助完成了任务？（50% `goal_accuracy` + 50% `behavior_check`） <br>
- Efficiency（效率）：是否避免了无效的工具调用与 Token 消耗？（50% `skill_efficiency` + 50% `token_efficiency`） <br>

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
| Overall（综合） | 94.9%（无同口径基线；无提升值） | 85.3%（无同口径基线；无提升值） |
| Security（安全性） | 100.0% → 100.0% (±0.0 分) | 84.6% → 76.9% (-7.7 分) |
| Correctness（正确性） | 29.2% → 100.0% (+70.8 分) | 47.7% → 89.2% (+41.5 分) |
| Discoverability（可发现性） | 93.7%（无同口径基线；无提升值） | 85.8%（无同口径基线；无提升值） |
| Effectiveness（有效性） | 42.4% → 89.3% (+46.9 分) | 33.9% → 81.1% (+47.2 分) |
| Efficiency（效率） | 91.4%（无同口径基线；无提升值） | 93.3%（无同口径基线；无提升值） |

## 技能版本 <br>
0.2.1（来源：frontmatter） <br>

## 伦理考量 <br>
NVIDIA 认为可信 AI 是一项共同责任，并已建立相应政策与实践以支持各类 AI 应用的开发。依据服务条款下载或使用本技能时，开发者应与内部团队协作，确保该技能符合所属行业与应用场景的规范要求，并防范意外的产品误用。 <br>

（仅限 NVIDIA 平台发布） <br>
如遇质量、风险、安全漏洞或 NVIDIA AI 相关问题，请前往[此处报告](https://app.intigriti.com/programs/nvidia/nvidiavdp/detail)。 <br>
