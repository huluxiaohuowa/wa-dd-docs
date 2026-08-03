> [English documentation](model-zoo.EN.md)

# Model Zoo

## 作用

Model Zoo 用于集中下载和更新 WA-DD 使用的模型。它采用与后续 VOS model-hub 对接兼容的路径设计，只保留模型下载、更新和路径管理，不包含模型运行实例管理。

固定项目模型会置顶显示：

- PocketXMol：分子生成使用。
- DeepTernary：TPD / PROTAC 三元复合物建模使用。

页面下方提供自选模型下载窗口，可填写 ModelScope 或 HuggingFace 仓库 ID。

## 路径约定

standalone 部署中，`wa-dd-model-zoo` 容器把 `${MODEL_HUB_SHARED_MODELS_PATH}` 挂载为 `/data`。因此模型服务维护的路径是：

```text
/data/export/ms/<org>/<repo>/snapshots/<snapshot>
/data/export/ms/<org>/<repo>/current
/data/export/hf/<org>/<repo>/snapshots/<snapshot>
/data/export/hf/<org>/<repo>/current
```

web 和 worker 容器把同一个宿主目录挂载为 `/modelhub`，读取路径为：

```text
/modelhub/export/ms/<org>/<repo>/current
/modelhub/export/hf/<org>/<repo>/current
```

不要把模型手动放到其他目录，否则 Model Zoo、后续对接的 model-hub 和 worker 无法用同一套规则识别和更新。

## 使用方式

1. 打开顶部导航中的 `Model Zoo`。
2. 在“项目模型”中查看 PocketXMol 和 DeepTernary 状态。
3. 如果模型未就绪，点击“下载模型”。
4. 如果需要强制拉取新快照，点击“更新模型”。
5. 如需下载其他模型，在“自选模型下载”中选择来源并填写仓库 ID，例如 `huluxiaohuowa/deepternary`。

下载或更新会写入新的 snapshot，必要文件检查通过后再切换 `current`。已有 `current` 满足必需文件时，普通下载不会盲目重下。

## 权限

- 查看模型状态需要登录。
- 下载和更新模型是全局操作，需要管理员权限。

## 固定模型

### PocketXMol

固定仓库：

```text
ms://huluxiaohuowa/pocketxmol
```

稳定读取路径：

```text
/modelhub/export/ms/huluxiaohuowa/pocketxmol/current
```

必需文件：

```text
data/trained_models/pxm/checkpoints/pocketxmol.ckpt
data/trained_models/pxm/train_config/train.yml
```

### DeepTernary

固定仓库：

```text
ms://huluxiaohuowa/deepternary
```

稳定读取路径：

```text
/modelhub/export/ms/huluxiaohuowa/deepternary/current
```

必需文件：

```text
checkpoints/PROTAC/checkpoint.pth
checkpoints/MGD/checkpoint.pth
configs/protac.py
configs/glue.py
```

## 部署组件

Model Zoo 是独立 CPU 服务：

```bash
./build_image.sh --profile amd --component model-zoo
./deploy.sh --profile amd --root /absolute/path/to/wa-dd-runtime --component model-zoo
```

Thor / ARM 平台使用同一个 Dockerfile，只按平台生成 `arm_YYYYMMDD` 标签：

```bash
./build_image.sh --profile thor --component model-zoo
```

相关镜像变量：

```text
WA_DD_MODEL_ZOO_AMD_IMAGE=
WA_DD_MODEL_ZOO_THOR_IMAGE=
```

web 通过内部地址访问该服务：

```text
WA_DD_MODEL_ZOO_URL=http://wa-dd-model-zoo:8810
```
