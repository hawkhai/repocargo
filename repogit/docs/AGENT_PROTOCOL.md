# RepoGit Agent CLI 协议 v1

适用 RepoGit 0.3.0 的 `repogit-cli.exe`。它是独立 Windows x64 EXE，内置与人类版相同的 Git/SSH 及凭据逻辑，不寻找或转调同目录 repogit.exe。设计参考《给 Agent 设计 CLI：不猜、少耗 Token、易重试、好组合》，采用 Git 风格命令与显式、可发现的机器协议。

## 发现与最小工作流

```powershell
repogit-cli --help
repogit-cli schema
repogit-cli schema bind
repogit-cli schema access.create
repogit-cli -C "D:\Repos\project" status --fields repository,head,dirty,binding
repogit-cli -C "D:\Repos\project" push --dry-run
repogit-cli -C "D:\Repos\project" push --expect-head <刚刚核对的SHA> --timeout-seconds 60
```

顶层 schema 只返回索引与基础协议；指定命令后返回参数、默认值、副作用、结果字段、重试语义和例子。帮助、版本、schema 离线可用，不要求 Git、SSH、凭据或仓库存在。`--help` / `--version` 返回文本，退出 0；其他正常调用的 stdout 为一个 JSON 值，包含参数错误。

## 命令树和共同选项

| 命令 | 行为与关键参数 |
| --- | --- |
| `status` | 只读实际仓库根、HEAD、本地分支、绑定和文件状态 |
| `bind` | `--config FILE` 或 `--repo SSH_URL [--branch sync]`；不覆盖冲突配置 |
| `clone` | 上述目标参数加 `DIRECTORY`；克隆并绑定已有同步分支 |
| `push` | 只推当前 HEAD 到绑定分支；不自动提交、强推或上传标签 |
| `pull` | 干净工作区的快进拉取；不合并、变基或 stash |
| `doctor` | 刷新本机 SSH 配置并联网检查读取，不能证明可写 |
| `access create` | `--name NAME --repo SSH_URL --out FILE [--branch sync] [--known-hosts FILE]` |
| `access show` | `--config FILE`；只返回公开摘要，不返回私钥 |
| `device init` | 必须明确 `--no-passphrase`；否则 INTERACTION_REQUIRED，不提示口令 |
| `device show` | 返回设备公钥、指纹和密钥文件位置，不返回私钥 |

`-C DIR` 与 `--path DIR` 等价，默认当前目录，按 Git 规则解析实际根；支持仓库子目录、worktree 和 bare。资源名称识别留给可选 EasyClaw 技能，不是此协议的前置条件。

路径基准沿用人类版 `--path`：`-C` 选择 Git 操作目录，不改变进程当前目录。相对的 `--config`、`--out`、`--known-hosts` 路径按调用时当前目录解析；clone 的相对 DIRECTORY 按 `-C` 指定的父目录解析。Agent 可传绝对路径，避免混淆；密钥相关命令不要求 Git 工作区。

共同选项：

- `--json`：可显式声明，默认本来就是 JSON；不会因 TTY/管道改变数据格式。
- `--no-input`：可显式声明，本程序始终禁止交互。不代表同意删除或覆盖。
- `--dry-run`：返回本地计划，不联网、不生成密钥、不导入或改写用户配置。校验公钥可能使用自动清理的临时文件。access create 预演只验证基本参数和文件前置条件，可信主机记录及实际密钥生成列为待执行检查。
- `--timeout-seconds`：1..3600，默认 60，约束整个工作进程及 Git/SSH 后代；超时终止后允许少量进程回收时间。不会回退到不受控执行。
- `--fields a,b`：仅筛选 result 顶层字段，合法字段见 schema；未知字段在执行前拒绝，错误与信封不被过滤。允许但不适用的字段为 null。
- `--limit`：1..1000，默认 100；`--offset` 默认 0。约束返回的文件条目，不改变推送/拉取范围。
- `--expect-head SHA`：仅用于 push/pull，在执行前核对本地 HEAD；不提供远程 CAS 或跨进程原子锁。
- `--verbose`：stderr 输出一个有界诊断 JSON，只包含工作进程退出码、流字节数和超时标记，不返回原始日志。

命令和明确目标本身代表该次动作的范围，没有额外 `--yes` 门禁；也没有删除、任意 Git 转发或 `--force` 能力。只支持凭据文件路径，不通过 argv、stdout 或通用 stdin 协议传输私钥正文。

目标配置优先级不是任意覆盖链：`--config` 与 `--repo/--branch` 互斥，配置模式目标以文件为准；设备模式显式 repo、branch（默认 sync）确定目标。未重新绑定的同步命令使用已保存绑定。原有 REPOGIT_HOME、REPOGIT_SSH_CONFIG、REPOGIT_KEY_FILE 语义保留；导入凭据与密钥环境变量冲突时失败，不静默替换身份。GIT_SSH_COMMAND 等传输覆盖仍被原有校验拒绝。

## 输出信封和通道

成功例子（result 可因命令不同而变化）：

```json
{"schema_version":1,"ok":true,"command":"status","status":"local_result","execution_state":"completed","result":{"head":"<sha>","dirty":false},"error":null,"exit_code":0}
```

失败也在 stdout 返回一个 JSON，进程退出非零：

```json
{"schema_version":1,"ok":false,"command":"push","status":"failed","execution_state":"not_started","result":null,"error":{"code":"PRECONDITION_FAILED","message":"当前 HEAD 与预期提交不同","hint":"重新读取 status 并审阅新提交；不会继续写操作。","retryable":false,"child_exit_code":null,"details":null},"exit_code":2}
```

stderr 默认空，启用 verbose 才有计数诊断。Git/SSH 进度、密钥工具输出和异常原文不会混入 stdout；错误按稳定类别给出说明，不依赖 Agent 解析英文报错。无法可靠分类时使用 OPERATION_FAILED，不冒称精确根因。`child_exit_code` 能取得子进程失败状态时为整数，其余 null；父进程超时不保留可误认为成功的工作结果。

消费者应同时检查进程退出码、ok、execution_state。未知新增字段可忽略；schema_version、字段类型/含义及空值约定保持兼容，破坏性变化使用新协议版本。`exit_code` 是本次进程的协议退出码，不要求等于 Git 原始退出码。

| 退出码 | 含义 |
| --- | --- |
| 0 | 当前命令按其定义成功；预演成功不代表远程可写 |
| 2 | 参数、配置、已存在目标或前置条件错误 |
| 3 | 依赖、路径、仓库不存在或隔离执行不可用 |
| 4 | 需要无交互凭据、主机核验或仓库授权 |
| 6 | 操作失败、历史分叉或远程结果核对失败 |
| 124 | 整次请求超时，已终止受控进程树，需要核对状态 |

稳定错误码包括 INVALID_ARGUMENT、INVALID_FIELDS、UNKNOWN_COMMAND、NOT_FOUND、NOT_A_REPOSITORY、NOT_BOUND、INVALID_CONFIG、CONFIG_CONFLICT、ALREADY_EXISTS、DESTINATION_EXISTS、NO_COMMITS、DIRTY_WORKTREE、DETACHED_HEAD、BARE_WORKTREE_REQUIRED、PRECONDITION_FAILED、INTERACTION_REQUIRED、ACCESS_DENIED、HOST_KEY_REJECTED、HISTORY_DIVERGED、NETWORK_UNAVAILABLE、REMOTE_VERIFICATION_FAILED、REMOTE_CHANGED、OUTPUT_LIMIT、TIMEOUT、EXECUTION_UNAVAILABLE、WORKER_PROTOCOL_ERROR 和 OPERATION_FAILED。错误包含 hint、retryable；参数缺失与字段错误在 details 中给出安全的必需参数或有效字段列表，不回显用户传入的任意原文。

## 状态、预演、成功与结果未知

| 字段 | 解释 |
| --- | --- |
| `status=local_result` | 本地查询或公开摘要，不宣称远程同步 |
| `status=planned` | 本地计划，写操作未执行 |
| `status=succeeded` | 命令执行完成；具体远程核验范围看 verification |
| `execution_state=not_started` | 前置校验失败或仅预演，未执行写入动作 |
| `execution_state=completed` | 查询或操作已按本次定义完成 |
| `execution_state=may_have_changed` | 已进入执行阶段后失败，本地配置、凭据、目录或远程可能有部分变化 |
| `execution_state=result_unknown` | 超时或工作结果无法确认，不代表没执行；先核对状态 |

status 中的 head/local_branch/binding 在不存在或不适用时为 null；bare 的 dirty 为 null。文件分页使用 items、total（本次本地快照的精确条目数）、limit、offset、has_more、next_offset；next_offset 无下一页时为 null。每次调用都是新快照，变动工作区跨页不保证一致。状态读取超过 8 MiB 时明确 OUTPUT_LIMIT，不伪造完整 total。

status 和预演的 remote_verified=false。doctor 的 true 仅表示读取成功。clone 表示克隆传输完成；pull 表示 fetch/快进完成。push 在 Git 成功后另读绑定分支 SHA，与当前本地 HEAD 相同才返回 true，verification.scope 为 remote_branch_sha_at_verification_time；这不是持续锁定远程，其他设备随后仍可更新。

## 重试、并发与凭据

- access create 的输出路径是唯一性保护；已存在返回 ALREADY_EXISTS，不生成另一把钥匙。收到结果不确定时先 access show 核对之前的文件。
- 相同 bind 可重复；冲突明确失败。clone 不覆盖、删除或自动“修复”已有目录，超时后先检查目标。
- push 仅当前提交、不强推，Git 仍执行自身快进规则。push 成功但后续核验失败时返回 REMOTE_VERIFICATION_FAILED；不能将它当成未上传。
- 超时、网络中断、may_have_changed/result_unknown 时，本版不自动重试，retryable=false 表示需要先完成 hint 中的核对或修正；不是永远不能重试。没有伪造服务端幂等键或整体回滚保证。
- SSH 始终禁止密码/主机信任询问，要求已有可信主机记录；未知主机或加密设备私钥无法自动使用时明确失败。配置式密钥仍由核心工具按约定解锁。
- 配置文件即凭据，固定 repogit 口令不提供防泄露保障。Agent 只传文件路径、用 access show 读取公开摘要。原始日志不会发布，结果正文上限 1 MiB；超出时返回协议失败并提示缩小字段/分页，不能把缺失结果当成成功。

验证范围见 [Agent 验证记录](AGENT_VALIDATION.md)。该协议支持 Git 仓库操作，不表示应用已经构建、部署或发布。
