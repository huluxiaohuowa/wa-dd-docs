> [English documentation](sar.EN.md)

# 构效关系 (SAR)

## 作用

SAR 模块整合活性数据、分子结构、对接/FEP 结果，用于药化决策和下一轮设计。

设计反馈/失败归因是 SAR 的闭环分析入口。它把候选配体、相互作用 Profile、Docking 结果、FEP 结果和口袋约束放在同一轮证据表中，归因本轮候选失败模式，并生成下一轮设计约束。

## 输入

- SMILES/SDF 配体资产
- 活性数据表
- 对接或 FEP 结果资产
- 相互作用 Profile 资产
- 口袋资产

设计反馈表单中的候选配体、相互作用 Profile、Docking 结果和 FEP 结果均使用统一资产选择器。用户可以按资产来源分组展开、多选，并把同一项目中的多轮结果一起提交。口袋资产为单选输入，用于在报告中保留结构上下文。

推荐输入组合：

- 候选配体资产：本轮生成、筛选或 FEP 标注后的 SDF 资产。
- 相互作用 Profile：由相互作用分析任务生成的 Profile，或 Docking 相互作用回填结果。
- Docking 结果：WA-DD Docking 或 Uni-Dock 输出资产。
- FEP 结果：FEP 分析输出资产，包含配体或边的相对能量、误差和排名信息。
- 必要相互作用：以分号分隔，例如 `ASP189 hydrogen bond; SER195 proximity`。

## 输出

- SAR 表格
- R-基团/MMPA 视图
- 下一轮设计候选分子

设计反馈会生成一个 `design_feedback` 资产，包含以下标准文件：

| 文件 | 用途 |
| --- | --- |
| `design_feedback_report.json` | 完整结构化报告，包含输入资产、阈值、候选证据、失败模式、约束和推荐候选。 |
| `candidate_evidence.csv` | 候选级证据表。每行对应一个折叠后的候选配体，汇总 Docking、FEP、相互作用和推荐状态。 |
| `failure_modes.csv` | 失败模式表。每个候选可有多个失败标签，用于筛选本轮主要问题。 |
| `next_round_constraints.json` | 下一轮设计约束，包含必须保留的相互作用、需要避免的失败模式、优先优化方向和阈值。 |
| `recommended_candidates.sdf` | 推荐保留进入下一轮的候选结构；没有合格候选时该文件不生成。 |

## 失败归因规则

算法以候选配体为中心融合多源证据。若同一候选存在多个 pose 或多行外部证据，会按配体名折叠为一个候选记录，保留最有利的 Docking 分数、FEP 值和相互作用计数。

内置失败模式包括：

- `weak_docking_score`：Docking 分数未达到设定阈值。
- `fep_unfavorable`：FEP 预测不利，超过坏结果阈值。
- `fep_uncertain`：FEP 误差超过误差阈值。
- `missing_required_interaction`：缺少用户指定的必要相互作用。
- `low_interaction_support`：相互作用证据数低于最小计数。
- `series_outlier`：同一系列中该候选明显偏离系列趋势。
- `insufficient_evidence`：缺少足够 Docking、FEP 或相互作用证据。

推荐候选需要同时满足：

- Docking 分数达到阈值，或不存在 Docking 证据但其他证据支持。
- FEP 不超过坏结果阈值，并且误差不超过误差阈值。
- 满足必要相互作用和最小相互作用计数。
- 不含关键失败模式。

## 约束回传

`next_round_constraints.json` 面向下一轮分子生成、筛选或人工药化设计。它会把失败模式转成可执行约束：

- 保留已经命中的必要相互作用。
- 对缺失相互作用的候选，回传 `required_interactions`。
- 对 Docking 弱的候选，提高形状互补、口袋占据和打分阈值要求。
- 对 FEP 不利或不确定的候选，回传能量和误差阈值。
- 对系列离群点，提示分系列复核而不是直接合并排序。

## 验证指标

一次设计反馈任务完成后，应满足以下验收条件：

- 任务类型为 `design_feedback_analysis`，状态为 `completed`。
- 输出资产的 `source_type` 为 `design_feedback`，并带有 `schema: wa_dd.design_feedback.v1`。
- `candidate_evidence.csv` 至少包含 `candidate_name`、`docking_score`、`fep_delta_g`、`interaction_count`、`recommended` 字段。
- `failure_modes.csv` 包含可追踪的 `candidate_name` 和 `failure_modes` 字段。
- `next_round_constraints.json` 的 schema 为 `wa_dd.next_round_constraints.v1`。
- 若输入存在合格候选，`recommended_candidates.sdf` 可下载并作为后续配体资产复用。
- 前端资产选择器支持按来源分组展开和多选，不应退回原始多选框。

## API 自动化

设计反馈 API：

```http
POST /api/v1/design-feedback
```

请求字段：

| 字段 | 说明 |
| --- | --- |
| `project_id` | 当前项目 ID。 |
| `ligand_asset_ids` | 候选配体资产 ID 列表。 |
| `interaction_profile_asset_ids` | 相互作用 Profile 资产 ID 列表。 |
| `docking_asset_ids` | Docking 结果资产 ID 列表。 |
| `fep_asset_ids` | FEP 结果资产 ID 列表。 |
| `pocket_asset_id` | 可选口袋资产 ID。 |
| `required_interactions` | 必须保留的相互作用列表。 |
| `name` | 输出资产名称，默认 `design_feedback`。 |
| `docking_good_threshold` | Docking 推荐阈值，默认 `-8.0`。 |
| `fep_bad_threshold` | FEP 坏结果阈值，默认 `1.0`。 |
| `fep_error_threshold` | FEP 误差阈值，默认 `2.0`。 |
| `min_interaction_count` | 最小相互作用证据数，默认 `1`。 |

响应为 Job 对象。任务完成后，到输出资产中下载上述标准文件。

当前实现由 Web/API 直接执行，不新增 worker。该路径适合轻量级证据融合和资产生成；只有在后续接入长耗时模型推理、批量重算或队列调度时，才需要拆成独立 worker。
