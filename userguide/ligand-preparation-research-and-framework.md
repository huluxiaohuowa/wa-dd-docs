> [English documentation](ligand-preparation-research-and-framework.EN.md)

# 配体准备功能清单与开发框架

本文面向 WA-DD 的配体准备模块，目标是满足 CADD 研究人员在分子对接、虚拟筛选、SAR/FEP 后续分析中的常见配体准备需求。

## 1. 结论

第一版必须以 Uni-Dock 对接为主线。对接的配体输入核心是 3D 结构合理、带有正确电荷和可旋转键信息的配体文件。SDF 是主要交换格式，PDBQT 是 Vina 系列对接引擎的 docking-ready 格式。

建议采用：

```text
WebApp / FastAPI
  负责：上传、表格列映射、分子编辑、参数设置、
       任务提交、任务状态、资产/文件管理

ligand-prep-worker
  负责：RDKit 主流程 + 3D 构象生成 + Meeko PDBQT 准备
       + 可选 Open Babel fallback

输出资产
  prepared_ligand / prepared_ligand_library /
  ligand_conformer_set / optional docking_ready_ligand
```

不要把重计算和复杂化学处理都塞进 WebServer 镜像。WebServer 应保持轻量；配体准备应走独立 worker，这样后续能按 CPU/GPU/商业软件 license 分开部署。

开源优先路线：

1. `RDKit` 作为核心化学对象、标准化、去盐、去重、SMILES/SDF 读写、stereo/conformer 的基础引擎。
2. `Meeko` 作为 Uni-Dock / AutoDock Vina 的 PDBQT 准备器，是第一优先的对接格式输出。
3. `Gypsum-DL` 或 `molscrub` 作为状态枚举参考实现或可选后端。
4. `Open Babel` 作为格式转换和 fallback 工具，但不要优先把 Open Babel Python API 深度嵌入主程序，GPL 传播问题需要单独评估。
5. 商业适配层预留 `Schrödinger LigPrep/Epik` 和 `OpenEye OMEGA/QUACPAC`，但不作为默认依赖。

## 2. CADD 配体准备需要覆盖的功能

### 2.0 对接优先兼容原则

Uni-Dock / AutoDock Vina 对接任务中，小分子的主输入是带有 3D 坐标、正确电荷和可旋转键信息的 PDBQT 文件。SDF 作为审计、预览和跨工具交换的主要格式。

因此配体准备模块的生产目标应是：

1. 从用户输入得到可信的 3D 结构合理的配体。
2. 生成 Uni-Dock / Vina 可直接使用的 PDBQT。
3. 保留 SDF 作为审计、预览、跨工具交换和后续 FEP/SAR 的结构资产。
4. 保留 canonical SMILES / InChIKey 用于去重和标识。

PDBQT 是对接路线的主输出，SDF 是通用交换格式。

### 2.1 输入能力

必须支持：

- 单分子 `SDF/MOL/MOL2/PDB` 上传。
- 多分子 `SDF` 上传。
- `SMILES` 列表输入。
- `CSV/TSV/XLSX` 表格上传，并允许选择：
  - SMILES 列；
  - 分子名称列；
  - compound ID 列；
  - activity / batch / series 等保留字段。
- 手动画分子：
  - 输出 molblock；
  - 同步生成 canonical SMILES；
  - 支持保存为 ligand asset。
- 分子库内单分子编辑：
  - 上传多分子 SDF 或 SMILES 列表后，用户能在表格里选择任意单个分子；
  - 点击 `编辑` 后进入 2D 分子编辑器；
  - 保存时不覆盖原分子，默认生成新的 edited ligand asset 或新版本；
  - 保留 `source_asset_id`、`source_row_index`、`source_molecule_id`、`edit_parent_id`。

建议支持：

- PubChem CID / ChEMBL ID / vendor ID 导入。
- 从蛋白 PDB 里提取已结合配体，另存为 ligand asset。
- 从 prepared protein / docking result 中复制配体到配体准备页面。
- 从对接输入文件反向加载 ligand SMILES。

### 2.1.1 分子编辑器要求

前端应内置 2D 分子编辑器，推荐 Ketcher。它要覆盖三种入口：

1. 空白画分子
   - 用户从零绘制；
   - 保存为新的 ligand asset；
   - 同时保存 molblock、canonical SMILES 和 2D 预览图。

2. 编辑上传的单个分子
   - 用户上传单分子 SDF/MOL 后，点击 `编辑`；
   - 编辑器加载该分子的 molblock；
   - 保存后生成 edited ligand asset。

3. 编辑分子库里的某一行
   - 用户上传多分子 SDF 或 SMILES 表格；
   - 分子表按行展示；
   - 任意一行都能点击 `编辑`；
   - 保存时生成新分子版本，并在 manifest 里记录来源。

编辑器保存行为必须满足：

- 默认不覆盖原始分子。
- 保存后的新分子要重新经过 RDKit sanitize。
- 如果结构不合法，前端给出错误，不创建资产。
- 保存后的新分子可直接用于对接任务。
- 保留编辑历史：
  - `source_asset_id`
  - `source_molecule_id`
  - `source_row_index`
  - `edit_parent_id`
  - `edit_reason` 或用户备注。

### 2.2 标准化与清洗

必须支持：

- 解析失败检测，并把失败分子写入 `failed.sdf/csv`。
- RDKit sanitize。
- 去盐 / 取最大有机片段。
- 金属断键 / metal disconnector。
- 规范化官能团表示。
- 电荷规范化 / uncharge / reionize。
- canonical SMILES / InChIKey 去重。
- 分子名、原始行号、输入文件名保留。

建议支持：

- 保留盐形式的选项。
- 保留原始分子和 prepared 分子的映射。
- 支持按 InChIKey first block 或 full InChIKey 去重。
- 反应性基团、PAINS、共价 warhead、金属有机物的规则标注，不强制删除。

### 2.3 状态枚举

必须支持：

- pH 范围设置，例如 `7.4 ± 1.0`。
- 质子化状态枚举。
- tautomer 枚举。
- 未定义手性中心枚举。
- 未定义 E/Z 双键枚举。
- 每个输入分子的最大变体数限制。
- 每个变体保留 genealogy / variant reason。

建议支持：

- 药化规则过滤不合理 tautomer。
- 对已定义手性默认不翻转。
- 用户可选择：
  - 严格保留输入 stereo；
  - 枚举未定义 stereo；
  - 枚举所有 stereo。
- 按 pH、tautomer、stereo 分层展示变体树。

### 2.4 3D 构象生成与优化

必须支持：

- 从 1D/2D 生成 3D。
- 多 conformer 生成。
- MMFF94 或 UFF 初步优化。
- conformer 去重。
- 最大 conformer 数限制。
- 失败分子单独输出。

建议支持：

- ring conformer 处理，尤其 6 元环 chair/boat。
- macrocycle 单独策略。
- 保留原有 3D 坐标的选项。
- 对接前只输出一个低能构象，虚拟筛选可输出多构象库。

### 2.5 对接兼容输出

必须支持：

- 每个 prepared ligand 输出带 3D 坐标的 SDF。
- 生成 Uni-Dock / AutoDock Vina 可用的 PDBQT（通过 Meeko）。
- 保留分子名称、原始行号、输入文件等元信息。
- 生成 manifest.csv 用于追踪每个分子的处理状态。
- 如果使用口袋，支持关联项目里的 pocket asset。

建议支持：

- 为每个 ligand 自动生成唯一 ID。
- 显示 2D 结构预览。
- 批量下载 prepared SDF / PDBQT。
- 输出 `manifest.csv`、`report.json`。
- 支持一个蛋白 + 多个 ligands 批量对接准备。

### 2.6 其他格式输出

可选支持：

- MOL2 输出。
- AM1-BCC 或其他更高质量 charge 后端作为可选 worker。
- 保留不可旋转键配置。
- 对共价 docking / metal coordination / boron / silicon 等特殊元素给出显式 warning。

注意：PDBQT 是 Uni-Dock / Vina 系列引擎的主输入格式。SDF 是通用交换格式。

### 2.7 质量控制与报告

必须支持：

- 每个输入分子的处理状态：
  - success；
  - warning；
  - failed。
- 失败原因。
- 生成变体数。
- 生成 conformer 数。
- canonical SMILES。
- InChIKey。
- formal charge。
- heavy atom count。
- rotatable bond count。
- molecular weight。
- clogP / TPSA / HBD / HBA 等基础属性。

建议支持：

- Lipinski / Veber / lead-like / fragment-like 规则。
- PAINS / Brenk / reactive group 标注。
- 2D 结构缩略图。
- 3D conformer 预览。
- 批量筛选后的保留/删除列表。

### 2.8 输出资产

每次配体准备任务至少生成：

```text
prepared_ligands.sdf
prepared_ligands.pdbqt
manifest.csv
failed.csv
report.json
```

在资产层面建议拆成：

| 资产类型 | 用途 |
| --- | --- |
| `ligand` | 原始单配体 |
| `ligand_library` | 原始多分子库 |
| `prepared_ligand` | 单个准备后配体 |
| `prepared_ligand_library` | 准备后的多分子库 |
| `ligand_conformer_set` | 多构象输出 |
| `docking_ready_ligand` | PDBQT 或其他 docking 专用输出 |

所有输出都必须能：

- 下载；
- 预览；
- 重命名；
- 复制到另一个项目；
- 被对接、FEP、SAR 页面作为输入资产选择。

## 3. 推荐工具栈

### 3.1 RDKit：默认核心

用途：

- 分子解析和 sanitize。
- 标准化、去盐、metal disconnect、normalize、reionize、tautomer canonicalization。
- SMILES/SDF 读写。
- canonical SMILES / InChIKey。
- stereo 识别与枚举。
- ETKDG 3D 构象生成。
- MMFF/UFF 优化。
- 基础 descriptor 和过滤。

优点：

- BSD 3-Clause，适合商业产品集成。
- Python/C++ 生态成熟。
- 与 Pandas、FastAPI、worker 队列集成成本低。

限制：

- pH 相关质子化不是 RDKit 的强项，需要 Dimorphite-DL、Gypsum-DL、molscrub、OpenEye/Schrödinger 等补充。
- 对非常复杂的 macrocycle、金属配合物、特殊元素要做失败分流。

### 3.2 Meeko PDBQT 生成器：对接主输出

用途：

- 把 prepared ligand 3D 结构转成 Uni-Dock / Vina 可用的 PDBQT。
- 分配 AutoDock atom types。
- 计算 partial charges。
- 定义 rotatable bonds / torsion tree。
- 对接输出再转回 RDKit/SDF。

适合：

- Uni-Dock 对接。
- AutoDock Vina。
- AutoDock-GPU。
- 大规模 docking 工作流。

限制：

- 输入应已有显式氢和 3D 坐标；因此 Meeko 应放在 RDKit/Gypsum-DL/其他 3D 准备之后。
- 不是通用 ligand preparation 全流程替代品。

### 3.3 Gypsum-DL：开源状态枚举后端

用途：

- 从 SMILES 或 flat SDF 生成 3D-ready 小分子。
- 枚举 ionization、tautomer、chiral、cis/trans、ring conformer 状态。

适合：

- 开源虚拟筛选。
- pH 变体和环构象枚举需求强的批量准备。

限制：

- 项目相对老，吞吐、失败恢复、现代 Python 兼容性需要实测。
- 对超大、非 drug-like 分子可能很慢。

### 3.4 molscrub：AutoDock 生态里的批量准备

用途：

- RDKit ETKDGv3 + UFF。
- tautomer 枚举。
- pH correction。
- ring chair 枚举。
- 面向 AutoDock docking 的批量处理。

适合：

- 与 Meeko/Vina/AutoDock-GPU 一起做 docking-ready 流程。

限制：

- GPL-3.0，商业分发要谨慎。
- 文档和 API 稳定性需要实测。

### 3.5 Open Babel：格式转换和 fallback

用途：

- 多格式互转。
- `--gen3d` 生成 3D。
- `-p <pH>` pH 加氢。
- partial charge。
- 最小化。

适合：

- 格式兜底。
- 某些 RDKit 读写不方便的格式。
- 命令行 fallback。

限制：

- GPL；如果深度链接或分发要做 license 评估。
- 建议先作为可选 CLI worker，不要直接嵌入 WebServer 主流程。

### 3.6 商业对标后端

后续可做 adapter，不作为默认依赖：

| 工具 | 主要能力 | 适配方式 |
| --- | --- | --- |
| Schrödinger LigPrep + Epik | 高质量 ionization、tautomer、stereo、ring conformation、3D preparation | license 环境下 worker 调命令行 |
| OpenEye OMEGA + QUACPAC | 高速 conformer generation、tautomer/protonation、charges | license 环境下 worker 调 toolkit/app |

商业后端的价值是质量和吞吐，但部署、授权、成本都更复杂。架构上只需要预留 backend adapter，不要把产品逻辑绑定到单一商业工具。

## 4. 推荐开发框架

### 4.1 后端数据模型

新增或扩展：

```text
Job
  job_type = ligand_preparation
  input_asset_ids = [ligand or ligand_library]
  options_json = preparation parameters
  output_asset_ids = [prepared_ligand_library, optional docking_ready_ligand]
  result_json = stats, report paths, failed counts

Asset
  kind = ligand / ligand_library / prepared_ligand / prepared_ligand_library /
         docking_ready_ligand
  metadata_json = source mapping, generation options, chemistry summary

AssetFile
  role = input_sdf / prepared_sdf / pdbqt /
         manifest / failed / report / preview
```

### 4.2 Worker 入口

建议定义稳定 CLI：

```bash
wa-dd-ligand-prep \
  --input input.sdf \
  --input-format sdf \
  --output-dir /data/jobs/<job_id>/outputs \
  --ph 7.4 \
  --ph-tolerance 1.0 \
  --enumerate-tautomers true \
  --enumerate-protomers true \
  --enumerate-undefined-stereo true \
  --max-variants-per-mol 16 \
  --max-conformers-per-variant 20 \
  --output-sdf true \
  --output-pdbqt true
```

worker 输出固定文件：

```text
prepared.sdf
prepared.pdbqt
manifest.csv
failed.csv
report.json
events.jsonl
```

### 4.3 Pipeline 阶段

```text
1. Load
   读 SDF/SMILES/CSV/XLSX/molblock

2. Validate
   sanitize、元素检查、重复 ID 检查、失败记录

3. Standardize
   normalize、去盐、metal disconnect、uncharge/reionize、canonical identifiers

4. Enumerate
   pH/protomer、tautomer、stereo、ring state

5. Generate 3D
   ETKDG/MMFF/UFF，多 conformer，失败 fallback

6. Filter / Rank
   energy、duplicate conformer、最大变体数、drug-like rules

7. Docking Format
   Meeko 写 PDBQT，保留 SDF 映射

8. Report
   manifest、failed、report、preview

9. Asset Commit
   输出写入 prepared_ligand_library / optional docking_ready_ligand
```

### 4.4 前端页面设计

配体处理页面建议拆成以下区域：

1. 输入区
   - 上传 SDF/MOL2/CSV/XLSX。
   - 粘贴 SMILES。
   - 手动画分子。
   - 从项目资产导入。

2. 分子表与单分子编辑区
   - 上传 SDF 或 SMILES 列表后，解析出分子表。
   - 每一行显示名称、SMILES、2D 缩略图、状态、来源行号。
   - 每个分子都有 `预览`、`编辑`、`复制为新分子`、`删除/保留`。
   - 点击 `编辑` 打开 Ketcher 这类 2D 分子编辑器。
   - 编辑保存后生成新 ligand asset 或 library 内新版本，不直接覆盖原始输入。

3. 列映射区
   - SMILES 列。
   - ID 列。
   - 名称列。
   - activity/series 透传列。

4. 准备参数区
   - pH。
   - 状态枚举开关。
   - stereo 策略。
   - conformer 数。
   - 输出格式：SDF / PDBQT。
   - 失败处理策略。

5. 预览与 QC 区
   - 输入分子表。
   - 2D 图。
   - 3D conformer。
   - warning/failed 分子。

6. 任务与输出区
   - 实时进度。
   - 成功/失败数量。
   - manifest。
   - prepared SDF 下载。
   - PDBQT 下载。
   - 复制到项目。
   - 传给 docking/FEP/SAR。

### 4.5 API 草案

```text
POST /api/v1/assets/ligands/table
POST /api/v1/assets/ligands/smiles
POST /api/v1/assets/ligands/draw
POST /api/v1/assets/ligands/{asset_id}/molecules/{molecule_id}/edit

POST /api/v1/preparations/ligand
GET  /api/v1/jobs/{job_id}
GET  /api/v1/jobs/{job_id}/events
POST /api/v1/jobs/{job_id}/retry
POST /api/v1/jobs/{job_id}/cleanup

GET  /api/v1/assets/{asset_id}/files/{file_id}/download
POST /api/v1/assets/{asset_id}/copy
```

### 4.6 Docker 镜像建议

WebServer：

```text
python:3.11-slim
FastAPI + SQLAlchemy + Redis client
不安装大型化学计算依赖
```

ligand-prep-worker：

```text
python:3.11-slim 或 micromamba
RDKit
Meeko（PDBQT 生成）
可选：Gypsum-DL / molscrub / Open Babel CLI
```

商业 worker：

```text
schrodinger-worker
openeye-worker
通过 license server 或本地 license 文件启用
只暴露同一套 worker 输入/输出协议
```

## 5. MVP 开发顺序

### 第一阶段：可用

- SDF 上传。
- SMILES 列表。
- CSV/XLSX 选择 SMILES/name/id 列。
- 分子表展示。
- 单分子打开编辑器修改并保存为新资产。
- RDKit sanitize。
- 去盐、标准化、去重。
- 生成 canonical SMILES。
- 3D 构象生成。
- 输出 prepared SDF。
- 输出 PDBQT（Meeko）。
- manifest / failed / report。
- 任务状态、重跑、清理输出。

### 第二阶段：批量对接可用

- 一个蛋白 + 多个 ligands 批量对接准备。
- 对接页面可直接选择 prepared ligand。
- 多分子库批量对接任务。
- 对接输出回写到 ligand manifest，供 SAR 排序。

### 第三阶段：状态枚举

- pH/protomer。
- tautomer。
- undefined stereo。
- ring conformer。
- max variants 控制。
- 变体树和 warning 展示。

### 第四阶段：专业 QC

- PAINS/Brenk/reactive group。
- Lipinski/Veber/lead-like/fragment-like。
- macrocycle 策略。
- metal/organometallic 分流。
- 3D conformer viewer。

### 第五阶段：商业后端

- LigPrep/Epik adapter。
- OpenEye OMEGA/QUACPAC adapter。
- per-project backend selection。
- license/worker health check。

## 6. 关键风险

1. pH/tautomer 不是“唯一正确答案”
   - UI 必须显示生成了哪些状态，不能只吐一个结果。

2. stereo 不能乱改
   - 已定义手性默认保留。
   - 未定义手性才枚举，除非用户明确要求枚举全部。

3. 文件和分子映射不能丢
   - 每个输出分子必须保留 source row、source ID、variant index、conformer index。

4. Open Babel / molscrub license
   - GPL 组件不要直接混进主 WebServer。
   - 如要分发镜像，需要明确 license 策略。

5. 大规模库要流式处理
   - 不要一次把几十万分子全加载到 Web 进程内存。
   - worker 要按 chunk 写 manifest 和事件。

6. CADD 结果需要可解释
   - 每个删除、枚举、失败、warning 都要可追踪。

## 7. 调研来源

- RDKit MolStandardize 文档：<https://www.rdkit.org/docs/source/rdkit.Chem.MolStandardize.rdMolStandardize.html>
- RDKit Overview / license：<https://www.rdkit.org/docs/Overview.html>
- Open Babel 3D generation：<https://openbabel.github.io/docs/3DStructureGen/Overview.html>
- Open Babel `obabel` 命令文档：<https://open-babel.readthedocs.io/en/latest/Command-line_tools/babel.html>
- Open Babel license FAQ：<https://openbabel.github.io/docs/Introduction/faq.html>
- Gypsum-DL GitHub：<https://github.com/durrantlab/gypsum_dl>
- Gypsum-DL paper：<https://pmc.ncbi.nlm.nih.gov/articles/PMC6534830/>
- molscrub GitHub：<https://github.com/forlilab/molscrub>
- Meeko ligand preparation：<https://meeko.readthedocs.io/en/develop/lig_prep_basic.html>
- Meeko overview：<https://meeko.readthedocs.io/en/develop/lig_overview.html>
- Uni-Dock GitHub：<https://github.com/dptech-corp/Uni-Dock>
- AutoDock Vina basic docking：<https://autodock-vina.readthedocs.io/en/latest/docking_basic.html>
- Schrödinger LigPrep：<https://www.schrodinger.com/platform/products/ligprep/>
- Schrödinger Epik：<https://www.schrodinger.com/platform/products/epik/>
- OpenEye OMEGA：<https://www.eyesopen.com/omega>
- OpenEye QUACPAC：<https://www.eyesopen.com/quacpac>
