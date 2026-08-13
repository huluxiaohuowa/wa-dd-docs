> [English documentation](tpd-protac.EN.md)

# TPD / PROTAC

完整可操作案例见：[TPD / PROTAC 5T35 案例：BRD4-MZ1-VHL 三元复合物](tpd-protac-5t35-case.md)。

## 作用

TPD/PROTAC 模块用于 POI-E3 三元复合物建模、warhead、E3 ligand 和 linker 设计。

## 输入资产链

TPD 不是“蛋白输入、蛋白输出”的单一流程。DeepTernary 输入由三类来源组成：

- POI 蛋白资产：`protein`、`prepared_protein` 或复合物结构。
- E3 ligase 蛋白资产：`protein`、`prepared_protein` 或复合物结构。
- Degrader / MGD 配体资产：`ligand` 或 `prepared_ligand`。

PROTAC 任务还需要四个 PDB 辅助资产：

- POI 侧 binary ligand PDB。
- E3 侧 binary ligand PDB。
- POI 侧 mask PDB。
- E3 侧 mask PDB。

这些辅助 PDB 不是新的蛋白输出。binary ligand 通常来自 POI-warhead 或 E3-ligand 二元结构中的共晶小分子；mask PDB 是用于匹配 PROTAC 全分子和两端 anchor/warhead 原子的子结构。需要用户提供时，请先在蛋白处理页从 PDB 组分生成 TPD PDB 资产，或在配体处理页上传 PDB 形式的配体/mask 资产，再回到 TPD 页面选择。界面会按“来源 → 资产 → 文件”展示当前选择，后端任务只接收资产 ID，不接收容器内路径。

辅助 PDB 的选择规则：

- **不能选择完整蛋白或完整共晶复合物**作为 binary ligand 或 mask。即使文件里“包含配体”，只要同时包含蛋白链，就不是这个字段需要的文件。
- **binary ligand PDB** 应该是从二元共晶结构中单独抽出的 ligand-only PDB。例如 POI-warhead 结构中的 warhead 小分子，或 E3-ligand 结构中的 E3 配体小分子。
- **mask PDB** 是 full PROTAC 两端用于对齐的子结构。若二元共晶小分子正好等于 PROTAC 端基，mask 可以和对应 binary ligand 使用同一份 PDB；若共晶小分子多出不属于 PROTAC 端基的原子，应只导出共享端基子结构。
- **Degrader / MGD 配体** 是完整 PROTAC 小分子，推荐使用可被 RDKit 解析的 3D SDF。若配体准备失败或日志出现 RDKit 手性/氢/键级错误，应先修复 full PROTAC 文件，而不是继续更换蛋白资产。
- 如果用户已经有带合理 3D 坐标的 full PROTAC，并希望保留上传坐标，可勾选“禁用 ligand correction”。该选项会避免 DeepTernary 再尝试从 RCSB 下载 `{ligand_id}_ideal.sdf` 进行坐标重建。

POI、E3 和 degrader/MGD 是可复用上游资产；binary ligand 和 mask PDB 是可追踪的 PDB 辅助资产。系统能保存、关联和传递这些文件，但不会替用户判断哪个共晶配体或 mask 子结构在科学上正确。

## PLK1 PROTAC 输入选择示例

在 `PLK1 PROTAC` 这类任务中，常见资产来源是：

1. 在蛋白处理页上传或准备 POI 蛋白，把共晶配体删除后得到 `prepared_protein`。
2. 在蛋白处理页上传或准备 E3 ligase 蛋白，把共晶配体删除后得到 `prepared_protein`。
3. 在配体处理页上传完整 PROTAC / degrader 分子，得到 `ligand` 或 `prepared_ligand`。
4. 在蛋白处理页的 3D 组分/链视图里，从 POI 二元共晶结构中选中 HETATM 配体，生成 POI 侧 TPD PDB 资产。
5. 同样从 E3 二元共晶结构中选中 HETATM 配体，生成 E3 侧 TPD PDB 资产。

提交 DeepTernary 时，字段应这样对应：

| TPD 字段 | 应选择的资产 | 不应选择 |
| --- | --- | --- |
| POI 蛋白结构 | 删除配体后的 POI `prepared_protein` | 只含小分子的 PDB |
| E3 ligase 结构 | 删除配体后的 E3 `prepared_protein` | 只含小分子的 PDB |
| Degrader / MGD 分子 | 完整 PROTAC / degrader 的配体资产 | POI warhead 或 E3 ligand 片段 |
| POI binary ligand PDB | POI 二元结构中单独抽出的 ligand-only PDB | POI 完整蛋白、POI-配体复合物 |
| E3 binary ligand PDB | E3 二元结构中单独抽出的 ligand-only PDB | E3 完整蛋白、E3-配体复合物 |
| POI ligand mask PDB | full PROTAC 中 POI 端用于对齐的 ligand-only 子结构 | 完整蛋白或完整复合物 |
| E3 ligand mask PDB | full PROTAC 中 E3 端用于对齐的 ligand-only 子结构 | 完整蛋白或完整复合物 |

如果 POI/E3 二元共晶配体本身就等于 full PROTAC 的对应端基，binary ligand PDB 和 ligand mask PDB 可以选择同一份 ligand-only PDB。若二元共晶配体包含不属于 full PROTAC 端基的额外原子，应单独导出 mask 子结构。

本次 PLK1 校验任务使用的正确选择是：

| 字段 | 示例资产 |
| --- | --- |
| POI 蛋白结构 | `2YAC prepared` |
| E3 ligase 结构 | `4CI3 prepared` |
| Degrader / MGD 分子 | `PLK1-PROTAC` |
| POI binary ligand PDB | `2YAC 937 POI binary ligand PDB` |
| E3 binary ligand PDB | `4CI3 Y70 E3 binary ligand PDB` |
| POI ligand mask PDB | `2YAC_937_A501_lig1_mask` |
| E3 ligand mask PDB | `4CI3_Y70_B1429_lig2_mask` |

这里的 `2YAC pdb_id`、`4CI3 pdb_id` 这类完整 PDB 不能填到 binary ligand 或 mask 字段。它们包含蛋白链，不是 DeepTernary 这四个辅助字段需要的小分子 PDB。

若 full PROTAC 已经是可靠 3D SDF，且不希望 DeepTernary 再按 ligand ID 重建坐标，可以勾选“禁用 ligand correction”。PLK1 示例使用这个模式。

## 输出与复用

DeepTernary 的可复用输出资产是 `ternary_complex`。它代表一次三元复合物预测结果，通常包含：

- PDB ensemble：三元复合物结构，可进入相互作用分析或作为复合物结构继续复用。
- summary CSV：候选构象、seed 和运行摘要。
- 运行日志：用于排查输入、GPU 和 ligand correction 问题。

后续流程应引用 `ternary_complex` 资产本身，而不是要求用户手工复制 PDB 或 CSV 文件路径。

## 结果查看

推荐交互顺序：

1. 先在输出资产链中确认 `ternary_complex` 的来源：POI、E3、ligand 是否是本次预期输入。
2. 打开或下载 PDB ensemble，检查 POI/E3/degrader 的相对构象。
3. 查看 summary CSV，比较候选构象数量、seed 和失败记录。
4. 如需解释结合界面，把三元复合物资产送入相互作用分析页面。

## API 自动化

任务提交和结果查询复用 `project_id`、`asset_id`、`job_id` 的统一自动化模型。自动化调用也应保存并传递资产 ID，而不是容器内文件路径。
