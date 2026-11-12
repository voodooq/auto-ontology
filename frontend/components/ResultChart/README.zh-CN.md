<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# ResultChart 图表渲染组件

> 本文档为 [`frontend/components/ResultChart/README.md`](./README.md) 的简体中文翻译版。

将智能体生成的图表规格（Chart Spec）渲染为支持主题切换、具备无障碍访问能力的内联 SVG 图表，同时附带“查看数据（Show data）”表格与 CSV 导出功能。智能体只需在回答的 Markdown 代码块中输出一段精简的 JSON 规格；本模块负责解析、校验、规范化并绘制图表。整个过程**不涉及任何代码执行（eval）**，也**不依赖第三方图表库**。

## 接入方式

`MarkdownRenderer` 会将以下两种围栏代码块语言路由至本模块：

- ` ```chart ` → 单张图表（或 KPI 卡片）
- ` ```chart-carousel ` → 可翻页的一组关联折线图轮播

```tsx
import { ChartBlock, fenceBareSpecs } from '@/shared/components/ResultChart';

// 在 MarkdownRenderer 中，chart/chart-carousel 代码块会转为：
<ChartBlock raw={codeContent} fallback={fallback} />;
```

`fenceBareSpecs(markdown)` 还会自动识别并包裹智能体忘记加围栏标记的独立图表 JSON 行，确保其仍能正常渲染成图表而不是直接暴露原始 JSON。对于格式畸形的规格，组件会自动降级回普通代码块渲染，不会导致界面崩溃。

## 图表规格契约（Chart-spec contract）

智能体在代码块中输出 JSON。`ChartBlock` 会根据 JSON 结构自动选择对应的渲染器：完整规格绘制图表（若数据不足以构成对比，则退化为 KPI 卡片）；纯 KPI 规格绘制指标概览卡片；轮播规格绘制可翻页折线图组。

### 标准图表规格（Chart spec）

```json
{
	"type": "bar",
	"title": "GPUs by Model",
	"subtitle": "Active fleet inventory",
	"x": { "key": "model" },
	"y": { "label": "GPUs", "format": "compact" },
	"series": [{ "key": "gpus", "color": "green" }],
	"data": [
		{ "model": "H100", "gpus": 4200 },
		{ "model": "A100", "gpus": 3100 },
		{ "model": "L40S", "gpus": 1800 }
	]
}
```

| 字段 | 说明 |
| --- | --- |
| `type` | 图表类型：`bar`、`hbar`、`line`、`area`、`grouped-bar`、`delta` |
| `title` | 必填标题；`subtitle` 为可选副标题 |
| `x` | `{ key, label? }`；`key` 必须能对应 `data` 行中的字段 |
| `y` | `{ label?, format? }`；`format` 可选 `number`、`compact`、`percent` 或 `currency` |
| `series` | 1 至 6 个 `{ key, label?, color? }`；`color` 可选 `green`、`blue`、`amber`、`red` 或 `neutral` |
| `data` | 1 至 60 条行对象，包含 `x.key` 和每个 `series.key` 对应的字段 |
| `kpis` | 可选的顶部指标概览卡片（见下文） |

`delta` 图表以零值作为正负分界线，按正负号分别着绿色（增长）与红色（下降）；`grouped-bar` 会为多系列数据自动绘制图例。

### 纯 KPI 规格（KPI-only spec）

适用于单值结果或仅包含单一实体、图表对比没有意义的场景：

```json
{
	"title": "Fleet Health",
	"kpis": [
		{ "label": "Total GPUs", "value": "10,000", "sub": "across 4 models", "tone": "accent" },
		{ "label": "At-Risk Nodes", "value": "23", "tone": "warn" }
	]
}
```

`tone` 可选值为 `default`、`accent`、`warn` 或 `alarm`。

### 图表轮播规格（Chart carousel）

```json
{
	"title": "Revenue Forecast: Baseline vs Prediction",
	"charts": [
		{
			"type": "line",
			"title": "Baseline vs Prediction",
			"x": { "key": "month" },
			"y": { "format": "currency" },
			"series": [
				{ "key": "baseline", "color": "neutral" },
				{ "key": "prediction", "color": "green" }
			],
			"data": [{ "month": "Jan", "baseline": 5200000, "prediction": 5300000 }]
		},
		{
			"type": "line",
			"title": "Units Shipped",
			"x": { "key": "month" },
			"y": { "format": "compact" },
			"series": [{ "key": "units", "color": "blue" }],
			"data": [{ "month": "Jan", "units": 1800 }]
		}
	]
}
```

轮播组件支持包含 2 到 12 张折线图并分页切换。

## 模块结构

| 文件 | 职责说明 |
| --- | --- |
| `ChartBlock.tsx` | 入口组件：解析原始代码块，选择图表 / KPI / 轮播渲染器或降级回退 |
| `types.ts` | Zod 规格校验 Schema 与导出类型定义 |
| `parse.ts` | 解析与校验规格；提供 `fenceBareSpecs`、`toNumber` |
| `normalize.ts` | 安全的坐标轴默认值、Top-N 截断、KPI 退化降级处理 |
| `scale.ts`、`geometry.ts` | 数值域计算、格式化与绘图几何布局 |
| `renderers/` | 按图表类型拆分的底层图元渲染（坐标轴、柱状图、水平条形图、折线图） |
| `ResultChart.tsx` | SVG 外壳组件：映射图元、悬浮提示 Tooltip、渐变填充与工具栏 |
| `ResultChartCarousel.tsx` | 可翻页图表轮播容器 |
| `ChartKpi.tsx` | KPI 卡片与概览磁贴 |
| `ChartToolbar.tsx`、`ChartDataTable.tsx` | “查看数据”切换与 CSV 导出 |

## 安全性与无障碍设计

- 量值柱状图严格从零刻度开始；大数字在坐标轴上自动采用紧凑缩写格式化。
- 超过行数上限的排行榜会明确展示“top N of M”提示，绝不静默丢弃数据。
- 缺失数据点在折线图中渲染为断点（Gap），绝不隐式补零。
- 颜色限制在固定枚举值内，所有文本均作为转义后的 React 内容渲染，因此智能体生成的图表规格不存在任何脚本注入风险。
- CSV 导出会对电子表格公式注入字符进行安全中和处理，同时保留真实负数值。

## 可移植性

除以下两个 AIQ 集成点外，本模块完全自包含：`@/adapters/ui/icons` 中的五个工具栏/轮播图标，以及 `src/app/globals.css` 中的 `.result-chart*` CSS 样式规则。如需复用到其他项目，只需内联这两个依赖即可独立渲染。

## 测试覆盖率

`*.spec.ts(x)` 测试套件对该模块实现了 100% 语句、分支、函数与行覆盖率（通过作用域限定的 Vitest 阈值强制检查，不影响全局覆盖率统计）。
