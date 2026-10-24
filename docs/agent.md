# Agent 操作指南与结果语义

本指南描述当前 0.2.0 成品已经实现的接口。四个程序放在仓库根目录，配置使用 YAML；不要使用历史斜杠参数或 `repocargo.json` 作为公开入口。

## 最小工作流

```powershell
$tool = (Resolve-Path .\repocargo-cli.exe).Path
$project = (Resolve-Path .\examples\basic).Path
& $tool status --project $project --json
& $tool upload --project $project --dry-run --json
```

上述命令可离线执行，不需要密码，不写项目，不启动后端。真实操作前用用户的项目目录替换示例，并准备显式连接配置和凭据。

```powershell
& $tool upload --project $project --yes --no-input --timeout-seconds 300 --json
& $tool download --project $project --local-cache --no-input --timeout-seconds 300 --json
& $tool clean --project $project --yes --no-input --json
```

后三条是实际操作示意，不要对占位示例直接执行。下载前需有发布方配置和目录 ID；删除/覆盖范围需先按计划及配置核对。

## 后端与转调

默认只选择 CLI 同目录的 `repocargo.exe`；不搜索 PATH、不自动回退。显式选择 Lite：

```powershell
.\repocargo-cli.exe --program .\repocargo-lite.exe download --project C:\demo-project --dry-run --json
```

CLI 静态检查 EXE 的资源身份，不靠改名识别程序。旧 EXE 即使改名为 `repocargo.exe` 也不应冒充新后端。Fetch 不属于 CLI 后端。

`exec --` 直接转发参数、标准流及退出码，不提供结构化执行保护：

```powershell
.\repocargo-cli.exe exec --project C:\demo-project -- download --project C:\demo-project
```

CLI 的 `--project` 只设置 cwd，`--` 后仍需提供后端自己的 `--project`。普通自动化优先使用结构化命令。

## 计划不是远端验证

`status` / `--dry-run` 返回本地配置计划：`program`、`project`、`mainroot`、`directory_id`、`items`、`total`、分页、`effects`、`remote_verified`、`atomic` 等。`scope` 为 `configured_paths_not_expanded`，不展开为完整文件列表，也不校验远端。

`program` 是含 `kind`、`path` 等身份信息的对象。上传计划的 `items` 来自配置 `others`；状态、下载和清理计划来自 `upload.others`。新示例的 `upload: {}` 因而会返回空状态条目，而上传预演有一个 `assets/` 条目。它们不包含完整的主文件、生成文件及运行时展开范围，不能把空条目当成“没有副作用”。

- `--limit` 默认 100，范围 1..1000；`--offset` 默认 0。
- `has_more` / `next_offset` 用于取后续配置项；分页不会缩小真实操作范围。
- `directory_id` 为空可能只是尚未发布，并不意味着可以猜一个 ID 下载。
- `effects` 提示配置写回、Git 暂存、复制/删除、环境修复等可能副作用；仍需阅读配置。

## stdout、stderr 和退出码

CLI 成功 stdout 为单个 JSON（使用 `--json`），含 `schema_version: 1`、`ok: true`、`command`、`status`、`result`。只读状态是 `local_status`，预演是 `planned`；实际后端完成为 `backend_exited`，不是逐个远端资源校验。

失败时 stdout 为空，结构化错误写 stderr；查看 `code`、`hint`、`retryable`、`execution_state`，并检查进程退出码。`--verbose` 给出脱敏、有界的捕获诊断；不应据此保证任意应用输出都没有敏感内容。

| CLI 退出码 | 含义 |
| --- | --- |
| 0 | 当前命令成功 |
| 2 | 参数或配置错误 |
| 3 | 项目/程序不存在、身份不识别或启动失败 |
| 4 | 需要确认或凭据 |
| 6 | 后端失败或报告异常 |
| 124 | 超时，已尝试终止进程树 |

Python/Lite/Fetch 的退出码不是这张 CLI 映射表。它们成功时 `--json` 提供最终结果；失败和参数错误在 stderr，退出非零。Fetch 不保证每类失败都提供结构化 JSON，不能要求 stderr 必须可作为 JSON 解析。Fetch 的 `shell.run` 继承子进程输出，可能在最终 JSON 之前写入 stdout；有部署命令时不能保证整个 stdout 是单个 JSON 文档，见 [Fetch 说明](fetch.md)。

## 失败后

固定 ID 不是事务，也不保证并发发布。上传、下载、部署可能部分完成；不要自动声称旧状态已恢复。先保存经脱敏的诊断，检查项目及远端状态，再在用户授权下决定修复或重试。已知限制见 [limitations.md](limitations.md)。
