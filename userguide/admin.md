# 管理 / Admin

## 作用 / Role

管理页面向管理员开放，用于管理用户、项目文件和全局任务，并查看部署机器资源。普通用户只能查看和操作自己的项目、资产与任务。

The admin page is restricted to administrators. It manages users, project files, global jobs, and host resources. Standard users can access only their own projects, assets, and jobs.

## 已实现功能 / Implemented features

### 用户管理 / User management

- 查看全部注册用户及其审核状态和角色。
- 审核待批准的注册申请。
- 修改指定用户密码。
- 查看指定用户的项目、资产、资产文件和任务摘要。
- 删除用户；系统会停止并删除该用户的任务、项目、资产和持久化文件。不能删除当前登录的管理员账号。

### 文件与项目 / Files and projects

- 按用户查看项目、资产和资产文件。
- 下载资产文件；资产仍可回到项目工作流中作为后续输入。

### 任务管理 / Job management

- 查看所有用户的全局任务列表、所属用户、状态和输出资产。
- 查看任务事件，定位排队、运行、完成、失败或取消状态。
- 取消仍在执行的任务，清理任务运行文件，或删除任务记录及其关联输出。

### 系统资源 / System resources

- 查看主机名称、操作系统、架构和 CPU 逻辑线程。
- 查看内存、挂载磁盘及其已用/可用容量。
- 查看 GPU 型号、后端、驱动、显存、利用率和功耗（硬件支持时）。
- `host-metrics` runner 默认每 5 秒写入一次宿主机采样；页面支持自动刷新和立即刷新。
- 支持 x86 NVIDIA（`nvidia-smi`）、Jetson Thor（`tegrastats`/sysfs）和 ROCm（`rocm-smi`）指标来源。

![管理与系统资源页面：用户、文件/项目、任务管理和系统资源四个模块](images/admin-system-resources.jpg)

## 使用方式 / Workflow

1. 以管理员身份登录后打开“管理”。
2. 在“用户管理”审核账户、修改密码或进入该用户的数据摘要。
3. 在“文件 / 项目”检查项目资产和文件，按需下载文件。
4. 在“任务管理”查看全局任务与事件；仅在确有必要时执行取消、清理或删除。
5. 在“系统资源”开启自动刷新或点击“立即刷新”，观察 CPU、内存、磁盘和 GPU 资源。

## 输入与输出 / Inputs and outputs

- 输入：管理员登录凭据，以及用户或任务标识。
- 输出：用户状态、用户项目/资产摘要、全局 `JobOut` 列表、任务事件，以及 CPU、内存、GPU、磁盘和架构指标。

## API / Automation

```http
GET /api/v1/admin/users
POST /api/v1/admin/users/{username}/approve
POST /api/v1/admin/users/{username}/password
GET /api/v1/admin/users/{username}/summary
DELETE /api/v1/admin/users/{username}
GET /api/v1/admin/jobs
POST /api/v1/admin/jobs/{job_id}/cancel
GET /api/v1/admin/jobs/{job_id}/events
POST /api/v1/admin/jobs/{job_id}/cleanup
DELETE /api/v1/admin/jobs/{job_id}
GET /api/v1/admin/system
```

删除用户会级联停止并删除该用户任务、项目、资产和持久化文件。取消、清理和删除任务会影响其他用户正在使用或将要复用的结果，执行前应确认目标任务和输出资产。
