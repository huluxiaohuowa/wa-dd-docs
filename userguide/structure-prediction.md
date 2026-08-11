# 结构预测

结构预测页统一提交 `structure_prediction` 任务。输出会登记为资产；只要结果里包含 PDB、CIF 或 mmCIF 结构文件，就会成为 `protein` 或 `complex` 资产，可继续在“蛋白处理”里查看、准备，也可作为对接、FEP 或 MD 的上游输入。

## 后端选择

- **ESMFold**：只需要蛋白序列，适合单链快速预测。不使用 MSA/template/ligand 输入。
- **Boltz-2**：当前接入路径使用 protein FASTA，可关闭或打开 ColabFold MSA server。复合物、配体、模板和外部 MSA 资产会被任务记录保存，但当前 worker 还没有把这些资产转换成 Boltz YAML 多组分输入。
- **Chai-1**：当前接入路径使用 protein FASTA，可关闭或打开 MSA server，并支持 diffusion samples、seed、device 等基础参数。多组分、约束、模板和外部 MSA 资产需要后续把资产解析结果转换成 Chai 专用输入文件。
- **OpenFold3**：支持表单自动生成单链 query，也支持在高级参数窗口粘贴完整 query JSON 和 runner YAML。复杂多链、模板、MSA、配体约束建议优先使用完整 query JSON。

## 输入资产

蛋白处理页可以上传 PDB/mmCIF，也可以上传 `.fa`、`.fasta`、`.faa`、`.seq`、`.txt` 格式的蛋白序列文件。序列文件会保存为 `protein` 资产，文件角色为 `sequence`，可在结构预测页直接选择。

如果只想快速预测，也可以在结构预测页直接粘贴 FASTA/单链序列，不必先创建资产。需要复用、留痕或下游追踪时，建议先在蛋白处理页上传为蛋白序列资产。

## 输出复用

预测成功后，输出资产会带有：

- `source_type`: `esmfold_structure_prediction`、`boltz2_structure_prediction`、`chai1_structure_prediction` 或 `openfold3_structure_prediction`
- `metadata.operation`: `structure_prediction`
- 结构文件角色：`structure`
- 日志文件角色：`log`
- 置信度、表格、MSA 等按文件类型保存为同一资产下的附属文件

蛋白处理页会把这些结构预测产物作为蛋白类资产显示。后续需要对接时，建议先在蛋白处理页生成 `prepared_protein`，再到对接页选择准备后的受体；如果直接对接原始预测 PDB，系统会按现有资产兼容性规则检查。

## 高级参数

高级参数窗口用于粘贴完整 OpenFold3 query JSON、runner YAML 和逐行 extra args。extra args 是命令参数列表，不支持 shell 管道、分号或 `&&`。

当表单输入不足以表达复杂体系时，不要把配体或模板资产勾选后就假设 worker 已经使用了它们；应使用后端支持的完整输入格式，或等待对应 worker 的多组分转换逻辑接入。
