> [English documentation](api-automation.EN.md)

# API 自动化

## 核心规则

每个自动化步骤都输出 `project_id`、`asset_id` 或 `job_id`，下一步直接引用这些 ID。

## 最小链路

```text
login
  -> project_id
  -> protein asset_id
  -> pocket asset_id
  -> ligand asset_id
  -> prepared_ligand asset_id
  -> docking compatibility check
  -> job_id
  -> output_asset_ids
  -> file download
```

## 对接自动化

```http
GET /api/v1/docking/compatibility?engine=unidock&project_id=<project_id>&protein_asset_id=<protein_asset_id>&ligand_asset_id=<ligand_asset_id>&pocket_asset_id=<pocket_asset_id>&ligand_index=0
POST /api/v1/docking/unidock
GET /api/v1/jobs/{job_id}/events
```

`/api/v1/docking/compatibility` 是自动化工作流的前置检查。只有 `compatible=true` 时才提交对接任务；否则按 `errors`、`warnings` 和 `recommendations` 补齐输入。

## GROMACS / MD 自动化

```http
GET /api/v1/md/workflows/templates
POST /api/v1/md/workflows/validate
POST /api/v1/md/jobs
POST /api/v1/md/jobs/resume
GET /api/v1/jobs/{job_id}/events
```

`/api/v1/md/workflows/validate` 是 MD 任务的前置检查：它返回步骤检查器、缺失输入、命令预览和输出分组。真实运行前应确保 `ok=true`。`/api/v1/md/jobs/resume` 从已有 `md_result` 资产中的 `.cpt` 和 `.tpr` 创建继续运行任务，用于延长 production 或恢复中断模拟。

## Model Zoo 自动化

```http
GET /api/v1/model-zoo/catalog
GET /api/v1/model-zoo/models/{model_id}
POST /api/v1/model-zoo/models/{model_id}/download
POST /api/v1/model-zoo/models/{model_id}/update
POST /api/v1/model-zoo/custom/download
POST /api/v1/model-zoo/custom/update
```

`catalog` 返回置顶项目模型和自选模型状态。下载和更新是全局操作，需要管理员 token。模型路径遵循 ModelHub 兼容结构：`/data/export/ms|hf/<org>/<repo>/current`。

## API 端点

标准 API 契约由 FastAPI OpenAPI 自动生成：

- `/openapi.json`：给 agent 和工作流引擎读取的机器可读 schema。
- `/docs`：Swagger UI，可直接在线试调用。
- `/redoc`：结构化接口文档。

网页端 `API 文档 / API Docs` 页面会自动读取 `/openapi.json`，并补充工作流衔接说明。

## 同步规则

后端路由变化后，OpenAPI schema 会自动更新。手写指导只解释跨步骤工作流逻辑，不重复维护每个请求/响应字段。
