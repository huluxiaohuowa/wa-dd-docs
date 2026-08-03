> [English documentation](tpd-protac.EN.md)

# TPD / PROTAC

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

POI、E3 和 degrader/MGD 是可复用上游资产；binary ligand 和 mask PDB 是可追踪的 PDB 辅助资产。系统能保存、关联和传递这些文件，但不会替用户判断哪个共晶配体或 mask 子结构在科学上正确。

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
