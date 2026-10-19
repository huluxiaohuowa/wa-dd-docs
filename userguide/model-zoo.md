> [English documentation](model-zoo.EN.md)

# Model Zoo

## 作用

Model Zoo 用于集中下载和更新 WA-DD 使用的模型。模型的下载、更新、删除、快照切换和必需文件检查均由 WA-DD 自行完成。VOS 版与 V-Scheduler 同步状态、进度和路径，不通过 V-Scheduler 下载，也不包含模型运行实例管理。

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

不要把模型手动放到其他目录，否则 Model Zoo、V-Scheduler 和 worker 无法用同一套规则识别模型。

## VOS 与 V-Scheduler 同步

安装时让 WA-DD 和 V-Scheduler 选择同一个 `MODEL_HUB_SHARED_MODELS_PATH` 公共目录；也兼容旧 Model Hub。V-Scheduler 是可选的信息查看方，不是 WA-DD 下载模型的前置依赖。

Model Zoo 与 V-Scheduler 均将该目录挂载为 `/data`，按已有共享存储契约同步 `registry/models.json` 与 `shared/tasks/*.json`。模型记录保留其他元数据，增加 `managed_by=wa-dd-model-zoo`、相对根路径和相对 `current` 路径；任务记录标记 WA-DD 所有者，并持续刷新下载进度。启动时同步已有项目模型和自选模型，不将从未下载的项目模型登记为全局失败模型。

下载成功后同步就绪状态和指定的 revision（未指定时使用本地快照标识）；失败时同步失败状态；删除时移除对应模型与 WA-DD 任务记录。V-Scheduler 的 `/data/...` 与 worker 的 `/modelhub/...` 是同一公共目录的不同容器路径，不是可互换的绝对路径。

WA-DD 管理的模型请在 WA-DD 中下载、更新或删除，V-Scheduler 用于查看状态和路径。发现同一模型有其他应用的未结束任务时，WA-DD 返回冲突，不覆盖其进度或启动下载、删除。共享任务检查不是跨应用的原子操作锁，仍应避免从两边同时发起操作；管理标记也不会自动禁用 V-Scheduler 的管理按钮。

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
./deploy.sh --profile amd --component model-zoo
```

已有运行部署可以省略 `--root`，由部署脚本从正在运行的 Web 容器自动发现运行根目录和端口。首次部署、恢复部署或新建共存部署时仍必须显式传入 `--root`。

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
