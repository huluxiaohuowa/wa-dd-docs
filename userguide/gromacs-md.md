> [English documentation](gromacs-md.EN.md)

# GROMACS / MD

## 作用

GROMACS / MD 页面用于提交 CUDA 加速的分子动力学任务。当前支持能量最小化、NVT、NPT、生产 MD、aMD、Metadynamics、Umbrella、结合稳定性分析、隐式口袋发现和轨迹后处理入口。

## 推荐流程

1. 在项目中选择已有受体/蛋白结构资产。只有 `.pdb/.gro` 时，默认开启的“自动从受体/蛋白资产搭建 GROMACS 体系”会生成 topology、盒子、溶剂和离子步骤；已有 `.tpr` 或 `.gro/.pdb + .top` 时会直接复用。
2. 进入 `GROMACS / MD` 页面，选择输入资产。
3. 选择 workflow 模板或 protocol，填写力场、水模型、积分器、模拟时间、温度、压力、输出频率和 GPU。
4. 在右侧步骤检查器确认 `Workflow 依赖检查通过`。如果有 error，先补齐输入或改为 dry-run。
5. 提交任务后在任务中心查看事件和输出资产。
6. 打开 MD 输出资产，按轨迹、能量、分析、结构、checkpoint、参数和日志分组检查文件。

## 从配体 / 受体资产直接发起

- 受体-only MD：选择蛋白准备或上传得到的 `.pdb/.gro` 资产即可，页面会自动预览 `pdb2gmx → editconf → solvate → genion → grompp → mdrun`。
- 已参数化复合物 MD：选择复合物结构和对应 `.top/.itp/.prm` 资产，系统会直接生成 `grompp/mdrun`。
- 未参数化配体：如果只选择 `.sdf/.mol2` 等配体结构而没有 ligand topology，真实运行会被拦截。需要先生成或上传配体拓扑，再作为同一任务输入使用。
- 轨迹后处理：选择 `.xtc/.trr`，最好同时选择对应 `.tpr` 和 `.edr`，页面会给出 RMSD/RMSF/能量等分析命令预览。

## 高级参数

- `.mdp` 文件 JSON：写入一个或多个 MDP 文件。
- 命令流 JSON：非 dry-run 且需要完全自定义时逐条执行命令。
- 额外文本文件 JSON：写入 PLUMED、index 或辅助配置文本。
- 自定义 workflow JSON：保存步骤配置，供参数包和命令预览审计。

## Checkpoint 继续运行

完成的 MD 输出资产如果同时包含 `.cpt` 和 `.tpr`，输出详情会显示“从 checkpoint 继续 / 延长 production”。填写延长时间后，系统会创建新的 `md_production` 任务，并用 `gmx mdrun -cpi ... -append` 从 checkpoint 继续。

## 输出

- 轨迹：`.xtc`、`.trr`
- 能量：`.edr`
- 分析：`.xvg`、`.csv`、`.json`、`.xpm`
- 结构：`.gro`、`.pdb`、`.cif`
- Checkpoint：`.cpt`
- 参数：`.mdp`、`.top`、`.itp`、`.ndx`、`.tpr`
- 日志：`.log`、`.txt`

`.xvg`、`.csv`、`.json` 和日志文件可以在页面中预览；`.xvg` 会显示轻量折线图。
