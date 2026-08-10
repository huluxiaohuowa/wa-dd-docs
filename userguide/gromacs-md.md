> [English documentation](gromacs-md.EN.md)

# GROMACS / MD

## 作用

GROMACS / MD 页面用于提交 CUDA 加速的分子动力学任务，并把输入参数、命令流、运行日志、轨迹结构和论文常见分析图统一登记到 WA-DD 任务资产中。当前入口支持能量最小化、NVT、NPT、生产 MD、aMD、Metadynamics、Umbrella、结合稳定性分析、隐式口袋发现、轨迹后处理和自定义命令流。

## 快速教程：短 MD 结果渲染闭环

下面示例在 server6 的 `Example` 项目中完成，任务 ID 为 `677e3a02`，输出资产为 `ba4fc848`。这个短流程用于验证页面交互、worker 执行、输出登记和结果渲染；它会生成代表结构和常见曲线，不能替代正式科研生产 MD。

![输入资产和协议参数](images/gromacs-short-md-01-input-params-boxed.png)

1. 打开 `GROMACS / MD`。
2. 在“结构 / 系统资产”中勾选一个或多个输入资产。每个资产左侧都有勾选框，不需要使用系统多选快捷键。
3. `Protocol` 选择 `Custom commands`。
4. 取消 `dry-run`，表示这次由 worker 真正执行命令。
5. `GPU 使用方式` 选择自动或指定单卡；指定单卡时在 `GPU ID` 填 `0`、`1` 或 `0,1`。

![高级 JSON 参数](images/gromacs-short-md-02-advanced-json-boxed.png)

高级编辑区支持三类 JSON：

- `.mdp` 文件 JSON：键是文件名，值是完整 `.mdp` 文本。可覆盖表单生成的积分器、步数、输出频率、温压耦合等设置。
- 命令流 JSON：非 dry-run 时逐条执行。每项可以是字符串，也可以是 `{ "name": "...", "command": "..." }`。
- 额外文本文件 JSON：写入 `custom/` 下的辅助文件，例如 PLUMED、index 说明、选择脚本或配置文本。

可以点击“示例：短流程+曲线”自动填入命令流。输入格式错误时，文本框下方会显示校验提示；格式正确时会显示 `JSON 格式正确。`

![任务完成和输出文件](images/gromacs-short-md-03-task-result-boxed.png)

提交后任务卡会按“进入队列 / 执行中 / 完成”显示事件。完成后点击当前任务卡里的“查看 MD 输出”，结果会在这个任务卡下方直接展开，而不是跳到页面最底部。

本次验证任务输出了 29 个文件，包括 `gromacs_plan.json`、`gromacs_results.json`、`gromacs_summary.csv`、`gromacs_system_preview.json`、代表结构帧和 `.xvg/.csv` 分析曲线。

![3D 轨迹和常见曲线](images/gromacs-short-md-04-3d-curves-boxed.png)

结果页按论文阅读顺序组织：

- 3D 轨迹播放器：加载 `.pdb/.gro/.cif` 代表结构，可旋转、缩放、切换帧和播放多帧结构。
- 论文常见曲线：自动识别并渲染 `rmsd.xvg`、`rmsf.xvg`、`gyrate.xvg`、`energy.xvg`、`hbond.xvg` 等文件。
- 每张曲线都可以下载原始数据，也可以预览原文。

![高级分析图](images/gromacs-short-md-05-advanced-analysis-boxed.png)

高级分析区会自动识别 PCA/FEL、SASA、口袋体积、配体距离、接触图、聚类图等输出。示例任务成功渲染了 `pca.xvg`、`sasa.xvg`、`pocket_volume.csv` 和 `ligand_distance.xvg`。

## GPU 观测

server6 运行这次短命令流任务时，GROMACS worker 容器为 `wa-dd-wa-dd-gromacs-worker-amd-1`，镜像为 `wa-dd-gromacs:amd_cu128_20260810`。任务前后通过 `nvidia-smi` 采样到 GPU 0/1 可见，GPU 利用率保持 0%，显存保持基线约 `496 MiB / 18 MiB`。

原因是本教程任务主要用于验证页面和输出渲染，命令流只执行 `gmx --version` 并生成短曲线/代表结构，没有运行长时间 `gmx mdrun -nb gpu -pme gpu -bonded gpu -update gpu`。正式生产 MD 或较长 benchmark 任务应在 worker 日志和 GPU 采样中看到更明显的 CUDA 负载。

## 从配体 / 受体资产直接发起

- 受体-only MD：选择蛋白准备或上传得到的 `.pdb/.gro` 资产即可。默认开启“自动从受体/蛋白资产搭建 GROMACS 体系”，会生成 topology、盒子、溶剂和离子步骤。
- 已参数化复合物 MD：选择复合物结构和对应 `.top/.itp/.prm` 资产，系统会直接生成 `grompp/mdrun` 路径。
- 未参数化配体：选择 `.sdf/.mol2` 配体资产时，默认开启“配体资产自动生成 GMX 拓扑包”。worker 会用 OpenBabel/ACPYPE 生成 `.itp/.gro` 并登记到同一个 MD 输出资产。
- 轨迹后处理：选择 `.xtc/.trr`，最好同时选择对应 `.tpr` 和 `.edr`，页面会给出 RMSD/RMSF/能量等分析命令预览。

## 输出

- 轨迹：`.xtc`、`.trr`
- 能量：`.edr`
- 分析：`.xvg`、`.csv`、`.json`、`.xpm`
- 结构：`.gro`、`.pdb`、`.cif`
- Checkpoint：`.cpt`
- 参数：`.mdp`、`.top`、`.itp`、`.ndx`、`.tpr`
- 日志：`.log`、`.txt`

`.xvg`、`.csv`、`.json` 和日志文件可以在页面中预览；`.xvg/.csv` 会渲染轻量曲线。有 `.gro/.pdb/.cif` 代表结构时，输出详情会直接显示 3D 结构预览。
