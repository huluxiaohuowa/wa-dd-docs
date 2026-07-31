# Example 1TA2 完整范例 / Full Walkthrough

本范例在 server6 部署页面完成，访问地址为 `http://123.207.15.89:45103`。目标是让新用户从一个 PDB 共晶结构开始，完成蛋白准备、配体准备、对接、分子生成、二次对接、FEP 规划和相互作用分析。

本例靶点为 thrombin 复合物 `1TA2`，共晶参考配体为 `176 A:401`。

## 1. 检查 Example 项目

进入页面后，使用管理员账户登录，在右上角项目菜单选择 `Example`。每一步截图中的红框是用户需要关注或点击的位置。

![步骤 1：选择 Example 项目，并检查项目总览中的资产和任务两栏](images/example2-step-01-project-select-and-overview-boxed.jpg)

应看到以下关键资产：

- `Example 1TA2 thrombin complex`
- `Example 1TA2 reference ligand 176`
- `Example 1TA2 176 binding pocket`
- `Example 1TA2 receptor prepared ligand-removed`
- `Example 1TA2 congeneric 72 analog library prepared for docking`
- `Example 1TA2 PocketXMol de novo generated ligands`
- 两组 Uni-Dock docked pose library
- 两组 FEP result 和两组 FEP output

## 2. 从 PDB 下载蛋白并提取参考配体/口袋

进入“蛋白处理”，从 PDB 下载 `1TA2`。在组分列表中找到 `176 A:401`。

![步骤 2：在蛋白处理页输入 PDB ID，然后从 RCSB 导入](images/example2-step-02-protein-import-pdb-boxed.jpg)

![步骤 3：确认蛋白准备入口和默认参数区](images/example2-step-03-protein-prep-form-boxed.jpg)

![步骤 4：从蛋白资产列表选择受体，并确认预览、下载和准备入口](images/example2-step-04-protein-select-asset-and-options-boxed.jpg)

![步骤 5：预览准备后的受体结构，确认 3D 区和右侧显示控制](images/example2-step-05-protein-preview-selected-boxed.jpg)

![步骤 6：进入链/组分页，定位共晶配体、金属、水等组分](images/example2-step-06-protein-components-tab-boxed.jpg)

![步骤 7：进入口袋页，确认以参考配体保存的 pocket 资产](images/example2-step-07-protein-pocket-tab-boxed.jpg)

操作：

1. 将 `176 A:401` 提取为配体资产：`Example 1TA2 reference ligand 176`。
2. 以 `176 A:401` 位置提取口袋资产：`Example 1TA2 176 binding pocket`。
3. 运行蛋白准备，输出 `Example 1TA2 receptor prepared ligand-removed`。

蛋白准备时必须删除参考配体，否则后续对接时口袋仍被共晶配体占据。

## 3. 准备同系物配体库

进入“配体处理”，选择同系物库，生成准备后配体资产。

![步骤 8：配体来源按 FEP 输出、对接构象、分子生成、准备后配体和原始导入分组](images/example2-step-08-ligand-source-groups-boxed.jpg)

![步骤 9：选择当前配体资产、目标用途和准备参数，然后提交准备任务](images/example2-step-09-ligand-2d-sort-filter-export-boxed.jpg)

本例同系物库：

- 原始库：`Example 1TA2 ligand 176 congeneric 72 analog library`
- 准备后库：`Example 1TA2 congeneric 72 analog library prepared for docking`

推荐准备参数：

- 显式加氢
- 生成 3D 构象
- pH 7.4
- MMFF/UFF 优化
- 计算化学属性并写入 SDF properties

## 4. 对同系物库做 Uni-Dock 对接

进入“对接任务”，选择准备后蛋白、准备后同系物库和 176 口袋。

![步骤 10：选择受体、口袋、配体库和 Uni-Dock 参数](images/example2-step-10-docking-inputs-and-parameters-boxed.jpg)

本例参数：

- engine：Uni-Dock
- scoring：Vina
- `pose_per_ligand=3`
- `keep_top_poses=1`
- `cpu_threads=4`
- GPU：`cuda:0`

本次验证结果：

- 输入配体：66 个
- 输出 pose：194 个
- 失败/跳过：0 个
- 输出 pose library：`Example 1TA2 congeneric Uni-Dock screening result docked pose library`

![步骤 11：对接完成后，点击查看报表或分析构象库](images/example2-step-11-docking-results-report-analysis-boxed.jpg)

点击 `查看报表 · 194 poses` 查看打分表；点击 `分析构象库 · 194 poses` 进入相互作用分析。

## 5. 在同一口袋做 PocketXMol de novo 生成

进入“分子生成”，选择同一个准备后蛋白和同一个口袋。

![步骤 12：设置分子生成数量、batch、口袋半径和 GPU 参数](images/example2-step-12-generation-inputs-gpu-params-boxed.jpg)

本例参数：

- 模式：基于口袋 de novo 生成
- 生成数量：24
- batch size：8
- mean atoms：28
- min atoms：10
- sampling steps：100
- 口袋半径：12 Å
- GPU：`cuda:0`
- `prepare_for_docking=true`

本次验证结果：

- 请求 24 个，成功 23 个。
- 输出资产：`Example 1TA2 PocketXMol de novo generated ligands`
- 输出 SDF：`generated_ligands_h.sdf`
- 资产已显式加氢，页面显示每个分子的 `heavy / H` 数。

![步骤 13：生成任务完成后检查输出资产](images/example2-step-13-generation-results-assets-boxed.jpg)

## 6. 对 de novo 生成物继续对接

回到“对接任务”，输入同一个准备后蛋白、同一个口袋和 `Example 1TA2 PocketXMol de novo generated ligands`。

本次验证结果：

- 输入配体：23 个
- 输出 pose：69 个
- 失败/跳过：0 个
- 输出 pose library：`Example 1TA2 PocketXMol de novo Uni-Dock result docked pose library`

## 7. 对两组对接结果做 FEP/RBFE dry-run

进入“FEP / 分析”，分别选择两组 docked pose library 创建 RBFE 任务。

![步骤 14：选择 FEP 输入、reference ligand 和 dry-run/正式运行参数](images/example2-step-14-fep-inputs-reference-dryrun-boxed.jpg)

本例为 dry-run，用于检查 ligand map 和结果展示，不代表真实 ΔΔG。

输出：

- 同系物库：193 条 planned edge，派生 SDF `Example 1TA2 congeneric docking-pose RBFE plan clean annotated SDF`，194 条记录。
- de novo 库：68 条 planned edge，派生 SDF `Example 1TA2 de novo docking-pose RBFE plan clean annotated SDF`，69 条记录。

![步骤 15：查看 FEP edge 表、下载报告，或分析带 FEP 字段的派生 SDF](images/example2-step-15-fep-results-buttons-edge-table-boxed.jpg)

点击 `查看 FEP · N edges` 看 edge 表；点击 `分析 FEP SDF · N 分子` 把带 FEP 字段的 SDF 送入相互作用分析。

## 8. 相互作用分析与结果判断

进入“相互作用分析”，受体选择 `Example 1TA2 receptor prepared ligand-removed`。左侧按来源展开 FEP 输出、对接构象、分子生成和准备后配体。

![步骤 16：先选择受体，再按来源展开配体/pose 组](images/example2-step-16-interaction-receptor-and-source-groups-boxed.jpg)

操作：

1. 展开一个来源组。
2. 点击 `最佳 1 个`、`前 5 个`，或逐个勾选具体分子。
3. 在 3D 区查看受体、配体和作用力虚线。
4. 在右侧查看 docking score、FEP 字段、作用力数量和接触残基。
5. 使用导出按钮保存表格、SDF，或把选中分子生成新资产。

![步骤 17：点击“最佳 1 个”或逐个勾选分子，并查看 3D、分数和右侧残基接触](images/example2-step-17-interaction-select-best-and-view-boxed.jpg)

独立窗口：

![步骤 18：打开独立相互作用分析窗口，使用多选、导出表格、导出 SDF、生成资产和退出按钮](images/example2-step-18-interaction-independent-window-full-controls-boxed.jpg)

注意：点击 `打开独立分析窗口` 前必须先选中至少一个分子/pose；否则页面会提示先选择构象。

![未选配体时的独立窗口前置提示](images/example2-09-interaction-popup-boxed.jpg)

结果解释：

- docking score 越负通常越好，但不能直接等同实验自由能。
- FEP dry-run 的 edge 只是规划结果；正式 FEP 才会产生真实 ΔΔG、误差和轨迹。
- 相互作用虚线当前是距离几何候选，适合快速筛选和定位残基；发表前仍需用正式 profiler 或实验结构复核。

## 9. 2D/3D 资产查看

配体处理页面可以直接打开任意 ligand SDF、docking pose library 或 fep_output。

![独立 3D 构象预览：右侧单选/多选分子记录](images/example2-08-ligand-3d-popup-boxed.jpg)

使用建议：

- 2D 视图适合按属性排序、筛选、分页和导出。
- 3D 视图适合确认构象是否合理、是否含显式氢，以及多个构象是否叠在合理位置。
- 资产卡片上优先看资产名称、来源类别和分子数量，不需要记裸 ID。

## 10. 用户文档页面

菜单中的“用户文档”会展示构建时从 `userguide/*.md` 转换出的 HTML。

![步骤 19：用户文档左侧显示文档标题，当前文档标题下方展开目录](images/example2-step-19-userguide-title-and-toc-boxed.jpg)

每次 web 镜像构建时会重新转换文档和图片；更新文档后需要重新构建 web 镜像并部署，线上页面才会看到新内容。
