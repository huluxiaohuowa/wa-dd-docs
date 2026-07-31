# 项目与资产 / Projects and Assets

## 作用 / Role

项目用于隔离同一用户的不同研究任务；资产是可复用的输入、输出和中间文件。

Projects separate research contexts for one user. Assets are reusable inputs, outputs, and intermediate files.

## 输入 / Inputs

- 项目名称 / project name
- 蛋白、配体、口袋、对接结果等资产 / protein, ligand, pocket, docking results, and other assets

## 输出 / Outputs

- `project_id`
- `asset_id`
- 可下载文件 / downloadable files

## API / Automation

```http
POST /api/v1/projects
GET /api/v1/projects
GET /api/v1/assets?project_id=<project_id>
PATCH /api/v1/assets/{asset_id}
POST /api/v1/assets/{asset_id}/copy
DELETE /api/v1/assets/{asset_id}
```

资产复制到其他项目时，网页会按项目名选择目标项目；API 使用目标 `project_id`。

## Server6 Example：项目总览如何检查完整流程

本例使用 server6 部署页面 `http://123.207.15.89:45103`，项目名为 `Example`。

![项目总览：选择 Example 项目，并检查资产和任务两栏](images/example2-step-01-project-select-and-overview-boxed.jpg)

检查顺序：

1. 右上角项目菜单选择 `Example`。
2. 在“项目资产”中确认关键资产是否存在：
   - 原始蛋白：`Example 1TA2 thrombin complex`
   - 参考配体：`Example 1TA2 reference ligand 176`
   - 口袋：`Example 1TA2 176 binding pocket`
   - 准备后蛋白：`Example 1TA2 receptor prepared ligand-removed`
   - 准备后同系物库：`Example 1TA2 congeneric 72 analog library prepared for docking`
   - PocketXMol 生成库：`Example 1TA2 PocketXMol de novo generated ligands`
   - 两组 Uni-Dock pose library
   - 两组 FEP `fep_result` 和两组 `fep_output`
3. 在“项目任务”中确认任务均为“已完成”。本例包含蛋白准备、配体准备、分子生成、两组对接和两组 FEP。
4. 资产和任务过多时用分页查看，不需要记住裸 ID；实际操作优先看资产名称、类型和来源。

结果判断：

- `fep_output` 是 FEP 派生出的可加载 SDF 资产。
- `docking_pose_library` 是对接任务的合并构象 SDF 资产。
- 如果任务失败并产生失败资产，应删除失败任务和中间/输出文件，避免污染 Example 项目。 
