# 结构预测

结构预测页统一提交 `structure_prediction` 任务。输出会登记为资产；只要结果里包含 PDB、CIF 或 mmCIF 结构文件，就会成为 `protein` 或 `complex` 资产，可继续在“蛋白处理”里查看、准备，也可作为对接、FEP 或 MD 的上游输入。

## 后端选择

- **ESMFold**：只需要蛋白序列，适合单链快速预测。不使用 MSA/template/ligand 输入。
- **Boltz-2**：支持单链蛋白、PDB/CIF 模板、一个外部 MSA 或在线 MSA，以及 diffusion samples、seed 和输出格式。在线 MSA 与外部 MSA 必须至少选择一种；worker 会生成 Boltz YAML 后再执行预测。
- **Chai-1**：支持单链蛋白、PDB/CIF 模板、A3M 外部 MSA 或在线 MSA，以及 diffusion samples、trunk samples、seed 和 device。worker 会把 A3M 转成 Chai 的 aligned parquet，并为自定义模板生成 m8 命中表和本地 CIF 缓存。
- **OpenFold3**：支持表单自动生成单链 query、PDB/CIF 模板、外部 MSA、指定 seeds、精度和输出格式，也支持在高级参数窗口粘贴完整 query JSON 和 runner YAML。

## 参数怎么设置

先选择后端，再按下面的规则设置。页面会隐藏当前后端不支持的字段；即使绕过页面直接调用 API，不支持或互相矛盾的组合也会被拒绝。这里的“更多候选”表示增加采样数量，不代表结果一定更准确，最终仍应结合置信度、结构合理性和下游验证筛选。

### 输入和模板参数

| 页面字段 / API 字段 | 怎么设置 | 具体规则 |
| --- | --- | --- |
| 蛋白资产 / `protein_asset_id` | 推荐选择项目内已上传的目标蛋白资产，便于复用和追踪 | 接受 `protein`、`prepared_protein`、`complex`、`md_structure`。资产应包含序列或可解析序列的结构文件。 |
| FASTA / 单链序列 / `sequence` | 临时快速预测时可直接粘贴；需要留痕时优先上传为资产 | 没有选择资产时必须填写序列，或为 OpenFold3 提供完整 `query_json`。 |
| 模板结构资产 / `template_asset_ids` | 勾选一个或多个与目标同源、构象合适的 PDB/CIF/mmCIF 资产 | 模板必须包含 PDB/CIF/mmCIF 文件。选了模板就必须开启“启用模板”；ESMFold 不接受模板。 |
| 启用模板 / `use_templates` | 使用所选模板时保持开启；不做模板建模时关闭并清空模板选择 | 选中模板但关闭此项会返回 HTTP 400，不会静默忽略模板。 |
| MSA 资产 / `msa_asset_ids` | 有自己生成的 MSA 时选择；否则根据后端选择在线 MSA | 支持 `.a3m`、`.sto`、`.stk`；Chai-1 只接受 `.a3m`；Boltz-2 当前单链表单最多选择一个外部 MSA。 |
| 在线 MSA / `use_msa_server` | 没有外部 MSA 时开启；使用外部 MSA 后页面会自动关闭 | Boltz-2 必须在“在线 MSA”和“一个外部 MSA”中至少选择一种。OpenFold3 可关闭在线 MSA并只使用模板或序列。 |

目标蛋白和模板必须属于当前用户的同一项目。模板数量越多不一定越好；优先选择序列覆盖完整、分辨率较好、配体/构象状态符合研究问题的模板。系统负责把模板真正写入后端输入，但不会替用户判断模板的生物学适用性。

### 采样、seed 和输出参数

| 页面字段 / API 字段 | 允许值 | 推荐设置和影响 |
| --- | --- | --- |
| Diffusion samples / `num_diffusion_samples` | `1`–`100`，页面默认 `5` | 冒烟测试用 `1`；常规预测用 `5`；需要更多候选可用 `10`。数量增加通常会增加运行时间、输出文件和结果筛选工作。 |
| Model seeds 数量 / `num_model_seeds` | `1`–`50`，页面默认 `1` | OpenFold3 中，未填写“指定 seeds”时用于生成相应数量的 model seeds；Chai-1 中映射为 trunk samples。Boltz-2 当前不使用该字段。 |
| 指定 seeds / `seeds` | 逗号或空格分隔的整数，如 `42,100,200` | 需要复现实验时填写固定值。OpenFold3 可填写多个；Boltz-2、Chai-1 当前每个任务只接受一个显式 seed。填写 OpenFold3 显式 seeds 后，`num_model_seeds` 不再决定 seed 数量。 |
| 输出格式 / `output_format` | `cif` 或 `pdb` | OpenFold3 常规保存推荐 `cif`，信息更完整；需要兼容传统工具时选 `pdb`。Boltz-2 支持二者。ESMFold 固定输出 PDB，Chai-1 使用后端自身输出格式。 |
| 输出资产名称 / `name` | 任意易识别名称 | 建议包含靶点、模板、后端和配置，例如 `KRAS_OpenFold3_1UBQ_seed42`，便于从任务列表追踪。 |

OpenFold3 的候选规模同时受 diffusion samples 和 seed 数量影响。第一次运行建议先用 `1 × 1` 验证输入和模型，再增加采样；不要一开始就把两项都调到上限。Boltz-2 如需比较多个 seed，当前应分别提交多个任务，每个任务填写一个 seed。

### 精度、设备、模型和 GPU

| 页面字段 / API 字段 | 怎么设置 | 注意事项 |
| --- | --- | --- |
| 精度 / `precision` | OpenFold3 在 server6 推荐 `bf16`；ESMFold 的稳妥默认值为 `fp32` | `bf16`/`fp16` 通常占用更少显存；`fp32` 占用更高。页面只在 OpenFold3 和 ESMFold 显示此项。遇到不支持的低精度算子或数值异常时改为 `fp32` 重试。 |
| Device / `device` | 单卡通常填 `cuda:0` | OpenFold3、ESMFold、Chai-1直接使用该字段；Boltz-2 worker 会据此限制可见 GPU。若同时使用下面的 GPU 控制，应保持两处选择一致。 |
| 模型 / Checkpoint 路径 / `inference_ckpt_path` | 正常使用必须留空 | 留空即使用 Model Zoo 管理的部署默认模型。OpenFold3 只有在管理员确认版本兼容且路径在 worker 容器内可见时，才填写 checkpoint 文件绝对路径；其他后端把它作为模型目录。 |
| Checkpoint 名称 / `inference_ckpt_name` | 正常使用留空 | 仅 OpenFold3 使用。不要用它尝试加载旧 Preview-2 模型；OpenFold3 0.5.0 默认 checkpoint 已由部署配置确定。 |
| GPU 使用方式 | 推荐“自动” | “指定单卡/多卡可见”只限制任务可见的 GPU，不会自动把一次预测变成多卡分布式任务。 |
| GPU ID | 自动模式留空；单卡如 `0`，多卡如 `0,1` | 这里填写物理可见 GPU 编号。`device` 仍应使用任务容器内编号，通常是 `cuda:0`。 |

页面调度时使用的估算显存为：OpenFold3 `32 GB`、Boltz-2 `24 GB`、Chai-1 `24 GB`、ESMFold `16 GB`。这是排队和并发控制的估算值，不是显存上限；实际用量还会随序列长度、链数、MSA、模板、samples 和 seeds 改变。

### 可以直接采用的配置

#### 1. 输入链路冒烟测试

- 后端：OpenFold3
- 模板：选择一个 PDB/CIF 模板并开启模板
- 在线 MSA：关闭
- Diffusion samples：`1`
- Model seeds 数量：`1`
- 指定 seed：`42`
- 输出格式：`pdb`
- 精度：`bf16`
- Device：`cuda:0`
- checkpoint、GPU ID、高级参数：全部留空

这套配置适合先验证资产、模板、checkpoint 和 worker 链路，不能代替正式的多候选计算。

#### 2. 常规模板预测

- 后端：OpenFold3
- 模板：选择经过人工判断的一个或多个模板
- MSA：有外部 MSA 就选择外部 MSA；否则按研究需要启用在线 MSA
- Diffusion samples：`5`
- Model seeds 数量：`1`
- 指定 seeds：留空，或填写一个固定 seed 以便复现
- 输出格式：`cif`
- 精度：`bf16`
- GPU：自动
- checkpoint 和高级参数：留空

#### 3. 生成更多候选

- 后端：OpenFold3
- Diffusion samples：先从 `10` 开始
- Model seeds 数量：`3`
- 指定 seeds：留空，让 seed 数量生效；若要严格复现，则填写三个明确整数
- 其他参数保持常规配置

先观察单任务耗时、显存和输出规模，再继续增加。更多 samples/seeds 只扩大候选集合，不应直接表述为“高质量模式”。

#### 4. Boltz-2 模板预测

- 后端：Boltz-2
- 模板：选择 PDB/CIF/mmCIF
- MSA：启用在线 MSA，或者选择一个外部 MSA，二者至少一种
- Diffusion samples：冒烟测试 `1`，常规使用 `5`
- 指定 seed：每个任务填写一个整数，如 `42`
- 输出格式：优先 `cif`，需要传统工具兼容时选 `pdb`
- GPU：自动

#### 5. Chai-1 模板预测

- 后端：Chai-1
- 模板：选择 PDB/CIF/mmCIF
- 外部 MSA：必须是 `.a3m`；没有时启用在线 MSA
- Diffusion samples：冒烟测试 `1`，常规使用 `5`
- Model seeds 数量：这里表示 trunk samples，常规先用 `1`
- 指定 seed：每个任务一个整数
- Device：`cuda:0`

#### 6. ESMFold 快速单链预测

- 后端：ESMFold
- 输入：单链序列或可提取序列的蛋白资产
- 模板、MSA：不支持，也不要选择
- 精度：优先 `fp32`；显存紧张且硬件支持时再尝试 `bf16`/`fp16`
- Device：`cuda:0`

ESMFold 适合快速获得单链初始结构，不是模板建模后端。

## 输入资产

蛋白处理页可以上传 PDB/mmCIF，也可以上传 `.fa`、`.fasta`、`.faa`、`.seq`、`.txt` 格式的蛋白序列文件。序列文件会保存为 `protein` 资产，文件角色为 `sequence`，可在结构预测页直接选择。

如果只想快速预测，也可以在结构预测页直接粘贴 FASTA/单链序列，不必先创建资产。需要复用、留痕或下游追踪时，建议先在蛋白处理页上传为蛋白序列资产。

## 从上传到模板预测

1. 打开“蛋白处理”，选择项目。在“上传本地蛋白文件或序列”中先上传待预测蛋白的 FASTA/FA/FAA/SEQ/TXT，点击“上传蛋白资产”。也可以上传已有 PDB/CIF/mmCIF；系统会从结构中解析蛋白序列。
2. 仍在“蛋白处理”中上传模板结构 PDB/CIF/mmCIF。模板必须包含蛋白链；上传成功后会成为同一项目下可选择的蛋白结构资产。
3. 打开“结构预测”，选择 OpenFold3、Boltz-2 或 Chai-1。ESMFold 是序列快速预测后端，不接受模板。
4. 在“蛋白资产”选择待预测蛋白，在“模板结构资产”勾选一个或多个模板，并保持“启用模板”开启。
5. 按所选引擎设置 samples、seed、精度、输出格式和 device。选择外部 MSA 资产后，任务会自动关闭在线 MSA server，保证选中的 MSA 真正进入后端；Chai-1 的外部 MSA 必须是 `.a3m`。Boltz-2 必须保留在线 MSA，或选择一个外部 MSA，否则 Web 与 API 都会在提交前拒绝该无效组合。
6. 点击提交。任务详情的输入清单会记录所选资产；worker 同时把实际生成的 OpenFold3 query/runner、Boltz YAML 或 Chai m8/CIF 缓存写入任务工作目录，便于审计。
7. 任务完成后，在输出资产中查看 PDB/CIF、日志和置信度文件。需要对接或 FEP 时，先到“蛋白处理”生成 `prepared_protein`。

如果选择了模板资产却关闭“启用模板”，提交会直接报错，不会把模板只记录在任务里而静默忽略。

OpenFold3 0.5.0 默认使用公开 ModelScope 仓库 `huluxiaohuowa/openfold3-openbind-0` 中的 OpenBind-0 `of3-ob-2025-06-30-174k.pt`。旧仓库 `huluxiaohuowa/openfold3` 保留 Preview-2 `of3-p2-155k.pt`，不作为 0.5.0 的默认模型；只有在确认自定义 checkpoint 与运行版本匹配时，才在高级参数中覆盖 checkpoint 路径或名称。

## OpenFold3 模板预测完整流程

### Web 页面

1. 在“蛋白处理”上传目标 FASTA 或蛋白结构，再上传模板 PDB/CIF/mmCIF。
2. 进入“结构预测”，后端选择“OpenFold3”。确认“模型管理”显示 OpenFold3 已就绪。
3. 在“蛋白资产”选择目标，在“模板结构资产”勾选模板，保持“启用模板”开启。
4. MSA 有三种用法：
   - 快速验证模板链路：不选外部 MSA，并关闭在线 MSA。
   - 使用在线 MSA：不选外部 MSA，开启在线 MSA。
   - 使用自己的 MSA：选择 `.a3m`、`.sto` 或 `.stk` 资产；页面会自动关闭在线 MSA。
5. 第一次建议设置 samples=`1`、seeds 数量=`1`、指定 seed=`42`、输出=`pdb`、精度=`bf16`、device=`cuda:0`。跑通后再改成常规 samples=`5`、输出=`cif`。
6. “模型 / Checkpoint 路径”和“Checkpoint 名称”保持空白，使用部署默认 OpenBind-0。GPU 使用方式选“自动”。
7. 点击“提交结构预测任务”。完成后点击输出资产查看结构、日志、配置和置信度文件。

任务审计时，应能在工作目录看到：

- `openfold3/input/query.json`：实际送入 OpenFold3 的 query，所选模板会出现在 `template_cif_paths`。
- `openfold3/input/runner.yml`：seeds、精度和输出格式等 runner 配置。
- `openfold3/input/input_manifest.json`：目标、模板、MSA 资产 ID 和参数清单。
- `openfold3/output/`：结构、模型配置、实验配置和预测结果。

### API 请求

```json
{
  "engine": "openfold3",
  "project_id": "PROJECT_ID",
  "protein_asset_id": "TARGET_ASSET_ID",
  "ligand_asset_ids": [],
  "template_asset_ids": ["TEMPLATE_ASSET_ID"],
  "msa_asset_ids": [],
  "name": "OpenFold3 template prediction",
  "use_msa_server": false,
  "use_templates": true,
  "num_diffusion_samples": 1,
  "num_model_seeds": 1,
  "seeds": [42],
  "output_format": "pdb",
  "precision": "bf16",
  "device": "cuda:0",
  "inference_ckpt_path": null,
  "inference_ckpt_name": null,
  "extra_args": []
}
```

如需复杂多链或蛋白-配体输入，打开“高级参数编辑”提供完整 `query.json`；`runner.yml` 可覆盖 runner 配置。普通单链模板预测不要填写这两项。

## Boltz-2 模板预测完整流程

### Web 页面

1. 上传目标蛋白和模板结构，进入“结构预测”，后端选择“Boltz-2”。
2. 选择目标蛋白和一个或多个模板，保持“启用模板”开启。
3. 配置 MSA：
   - 没有自己的 MSA：保持“使用在线 MSA server”开启。
   - 有自己的 MSA：选择一个 `.a3m`、`.sto` 或 `.stk` 资产；当前单链表单最多一个，页面会关闭在线 MSA。
4. 冒烟测试设置 samples=`1`、seed=`42`、输出=`pdb`；常规任务设置 samples=`5`、输出优先 `cif`。
5. “Model seeds 数量”和“精度”不用于 Boltz-2。每个 Boltz-2 任务只接受一个显式 seed；要比较多个 seed，应分别创建任务。
6. 模型路径保持空白，device 通常为 `cuda:0`，GPU 使用方式选“自动”。提交后在任务列表查看输出资产。

任务工作目录中的 `boltz2/input/input.yaml` 是实际输入。应确认其中：

- `sequences[0].protein.sequence` 是目标序列。
- 使用外部 MSA 时存在 `sequences[0].protein.msa`。
- `templates` 列出所选模板路径和 `chain_id`。

### API 请求

```json
{
  "engine": "boltz2",
  "project_id": "PROJECT_ID",
  "protein_asset_id": "TARGET_ASSET_ID",
  "ligand_asset_ids": [],
  "template_asset_ids": ["TEMPLATE_ASSET_ID"],
  "msa_asset_ids": [],
  "name": "Boltz-2 template prediction",
  "use_msa_server": true,
  "use_templates": true,
  "num_diffusion_samples": 1,
  "num_model_seeds": 1,
  "seeds": [42],
  "output_format": "pdb",
  "precision": "bf16",
  "device": "cuda:0",
  "extra_args": []
}
```

这里的 `precision` 和 `num_model_seeds` 是统一请求模型中的兼容字段，Boltz-2 worker 不使用它们。若 `use_msa_server=false` 且 `msa_asset_ids=[]`，API 会返回 HTTP 400，不会创建无效任务。

## Chai-1 模板预测完整流程

### Web 页面

1. 上传目标蛋白和模板结构，进入“结构预测”，后端选择“Chai-1”。
2. 选择目标蛋白和模板，保持“启用模板”开启。worker 会把模板转换为本地 CIF 缓存，并生成 Chai 所需的 m8 模板命中表。
3. 配置 MSA：
   - 没有自己的 MSA：开启在线 MSA。
   - 使用外部 MSA：只能选择 `.a3m`；worker 会转换为 Chai 的 aligned parquet。`.sto`/`.stk` 可作为其他后端的 MSA，但不能用于 Chai-1。
4. 冒烟测试设置 diffusion samples=`1`、Model seeds 数量=`1`、seed=`42`、device=`cuda:0`。
5. 常规任务可先用 diffusion samples=`5`、trunk samples=`1`。页面上的“Model seeds 数量”在 Chai-1 中实际映射为 `--num-trunk-samples`，不是多个显式随机 seed。
6. 输出格式和精度由 Chai-1 后端管理，页面不会把它们传成 Chai 参数。模型路径保持空白，GPU 使用方式选“自动”。
7. 提交任务，完成后打开输出资产查看 CIF/PDB、NPZ、置信度文件和 `chai1.log`。

任务审计时，应能在工作目录看到：

- `chai1/input/input.fasta`：实际目标序列。
- `chai1/input/templates/template_hits.m8`：模板命中关系。
- `chai1/input/templates/cif/*.cif.gz`：worker 生成的本地模板 CIF 缓存；WA-DD 的 Chai-1 适配入口会优先从这里解析自定义模板，不会把它当成 RCSB ID 下载。
- 使用外部 MSA 时，`chai1/input/chai_msa/` 保存转换后的 aligned parquet。
- `chai1/output/`：Chai-1 结构和置信度结果。

### API 请求

```json
{
  "engine": "chai1",
  "project_id": "PROJECT_ID",
  "protein_asset_id": "TARGET_ASSET_ID",
  "ligand_asset_ids": [],
  "template_asset_ids": ["TEMPLATE_ASSET_ID"],
  "msa_asset_ids": [],
  "name": "Chai-1 template prediction",
  "use_msa_server": true,
  "use_templates": true,
  "num_diffusion_samples": 1,
  "num_model_seeds": 1,
  "seeds": [42],
  "output_format": "cif",
  "precision": "bf16",
  "device": "cuda:0",
  "extra_args": []
}
```

Chai-1 当前每个任务只接受一个显式 seed。`query_json` 和 `runner_yaml` 是 OpenFold3 专用字段，Chai-1 请求中不要填写。

## API 操作

先调用 `POST /api/v1/auth/login` 获取 Bearer token，并准备同一项目下的 `project_id`。目标序列和模板都通过公开资产接口上传，不要直接复制文件到任务目录：

```bash
curl -H "Authorization: Bearer $TOKEN" \
  -F kind=protein -F project_id="$PROJECT_ID" \
  -F name=target-sequence -F file=@target.fasta \
  "$BASE_URL/api/v1/assets/upload"

curl -H "Authorization: Bearer $TOKEN" \
  -F kind=protein -F project_id="$PROJECT_ID" \
  -F name=template-structure -F file=@template.pdb \
  "$BASE_URL/api/v1/assets/upload"
```

记录响应中的目标 `asset_id` 和模板 `asset_id`，填入上面三个后端对应的请求示例。三个后端统一把 JSON 提交到 `POST /api/v1/structure-prediction/openfold3`，由 `engine` 字段选择实际 worker。通过 `GET /api/v1/jobs/{job_id}` 和 `GET /api/v1/jobs/{job_id}/events` 查询进度，完成后响应中的 `output_asset_ids` 即为可继续复用的结构资产。

## 输出复用

预测成功后，输出资产会带有：

- `source_type`: `esmfold_structure_prediction`、`boltz2_structure_prediction`、`chai1_structure_prediction` 或 `openfold3_structure_prediction`
- `metadata.operation`: `structure_prediction`
- 结构文件角色：`structure`
- 日志文件角色：`log`
- 置信度、表格、MSA 等按文件类型保存为同一资产下的附属文件

蛋白处理页会把这些结构预测产物作为蛋白类资产显示。后续需要对接时，建议先在蛋白处理页生成 `prepared_protein`，再到对接页选择准备后的受体；如果直接对接原始预测 PDB，系统会按现有资产兼容性规则检查。

## 高级参数

高级参数窗口用于粘贴完整 OpenFold3 query JSON、runner YAML 和逐行 extra args。query JSON 和 runner YAML 只对 OpenFold3 开放；extra args 会传给当前引擎。每行填写一个命令参数，flag 和它的值分两行填写，不支持 shell 管道、分号或 `&&`。

当前简单表单不转换配体资产。OpenFold3 的复杂多链或蛋白-配体体系可使用完整 query JSON；Boltz-2 与 Chai-1 的多组分表单仍需后续单独接入。前端会隐藏当前引擎不支持的参数，API 也会拒绝不支持或相互矛盾的组合。
