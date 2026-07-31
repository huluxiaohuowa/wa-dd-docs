# API 自动化 / API Automation

## 核心规则 / Core Rule

每个自动化步骤都输出 `project_id`、`asset_id` 或 `job_id`，下一步直接引用这些 ID。

Every automation step returns a `project_id`, `asset_id`, or `job_id`; the next step consumes those IDs directly.

## 最小链路 / Minimal Chain

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

## Docking automation / 对接自动化

```http
GET /api/v1/docking/compatibility?engine=unidock&project_id=<project_id>&protein_asset_id=<protein_asset_id>&ligand_asset_id=<ligand_asset_id>&pocket_asset_id=<pocket_asset_id>&ligand_index=0
POST /api/v1/docking/unidock
GET /api/v1/jobs/{job_id}/events
```

`/api/v1/docking/compatibility` is the gate for automated workflows. Submit docking only when `compatible=true`; otherwise follow `errors`, `warnings`, and `recommendations`.

`/api/v1/docking/compatibility` 是自动化工作流的前置检查。只有 `compatible=true` 时才提交对接任务；否则按 `errors`、`warnings` 和 `recommendations` 补齐输入。

## API / Endpoints

The canonical API contract is generated from FastAPI OpenAPI:

- `/openapi.json`: machine-readable schema for agents and workflow engines.
- `/docs`: Swagger UI for interactive calls.
- `/redoc`: structured API reference.

The in-app `API 文档 / API Docs` page reads `/openapi.json` automatically and adds workflow chaining notes.

标准 API 契约由 FastAPI OpenAPI 自动生成：

- `/openapi.json`：给 agent 和工作流引擎读取的机器可读 schema。
- `/docs`：Swagger UI，可直接在线试调用。
- `/redoc`：结构化接口文档。

网页端 `API 文档 / API Docs` 页面会自动读取 `/openapi.json`，并补充工作流衔接说明。

## Synchronization Rule / 同步规则

When a backend route changes, the OpenAPI schema updates automatically. The hand-written guide should only explain cross-step workflow logic, not duplicate every request/response field.

后端路由变化后，OpenAPI schema 会自动更新。手写指导只解释跨步骤工作流逻辑，不重复维护每个请求/响应字段。
