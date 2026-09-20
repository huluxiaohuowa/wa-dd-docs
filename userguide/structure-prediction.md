# 结构预测

结构预测页统一提交 `structure_prediction` 任务。输出会登记为资产；只要结果里包含 PDB、CIF 或 mmCIF 结构文件，就会成为 `protein` 或 `complex` 资产，可继续在“蛋白处理”里查看、准备，也可作为对接、FEP 或 MD 的上游输入。

## 后端选择

- **ESMFold**：只需要蛋白序列，适合单链快速预测。不使用 MSA/template/ligand 输入。
- **Boltz-2**：支持单链蛋白、PDB/CIF 模板、一个外部 MSA 或在线 MSA，以及 diffusion samples、seed 和输出格式。在线 MSA 与外部 MSA 必须至少选择一种；worker 会生成 Boltz YAML 后再执行预测。
- **Chai-1**：支持单链蛋白、PDB/CIF 模板、A3M 外部 MSA 或在线 MSA，以及 diffusion samples、trunk samples、seed 和 device。worker 会把 A3M 转成 Chai 的 aligned parquet，并为自定义模板生成 m8 命中表和本地 CIF 缓存。
- **OpenFold3**：支持表单自动生成单链 query、PDB/CIF 模板、外部 MSA、指定 seeds、精度和输出格式，也支持在高级参数窗口粘贴完整 query JSON 和 runner YAML。

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
