# RepoCargo 使用参考

本文针对仓库根目录随附的 Windows x64 EXE；不需要获取业务源码。

用 Git 保存资源配置和目录 ID，在仓库外发布、分发与恢复资源。Python 完整版是业务协议基准；其他版本只提供各自已实现的能力。

当前接口版本为 **0.2**：项目配置统一为 `repocargo.yaml`，所有程序使用显式子命令。项目 JSON 配置和公开斜杠参数不再作为入口；索引、ZIP 元信息和内部缓存格式保持原样。旧工具 `gitbigfileftp` 独立保留。

`gitbigfileftp` 的 Python/Rust 继续维护 JSON/斜杠入口；RepoCargo 不替换它们的配置或历史安装 EXE。两套程序使用独立工作/输出目录及扫描范围，远端分别为 `/gitremote`、`/repocargo`。`datamap.txt`、`version.txt`、`version/` 和缓存仍有同名项，不支持同时写同一个工作目录。

| 程序 | 作用 | 发行物 |
| --- | --- | --- |
| `repocargo` | 完整上传、下载、索引清理与桌面界面 | `repocargo.exe` |
| `repocargo-lite` | Rust 精简上传、下载、索引清理与文本菜单 | `repocargo-lite.exe` |
| `repocargo-cli` | Agent 入口：状态、只读预演、确认、超时、脱敏诊断、分页 | `repocargo-cli.exe` |
| `repocargo-fetch` | 消费 HTTP/HTTPS/FTP 发布索引，校验、下载及部署 | `repocargo-fetch.exe` |

## 快速开始

在项目中创建 `repocargo.yaml`：

```yaml
# 运营人员维护这些字段；路径推荐使用 /。
mainfile: repocargo.yaml
others:
  assets: 运行资源
  models: 模型文件
upload: {} # 程序自动维护，请勿手工修改其中的文件快照。
gitremote: demo-project
submit_dir: release_001 # 固定发布目录；不是文件 MD5。
ftpconfig:
  host: ftp.example.invalid
  port: 21
  user: demo-user
  pypass: DEMO_FTP_PASSWORD # 环境变量名，不是密码本身。
generate_listfile: true
generate_versionfile: true
generate_densefile: true
```

```powershell
$env:DEMO_FTP_PASSWORD = '<实际密码>'
.\repocargo.exe upload --project C:\demo-project
.\repocargo.exe download --project C:\demo-project --local-cache --json
.\repocargo-lite.exe download --project C:\demo-project --local-cache --json
.\repocargo-fetch.exe download --url https://cdn.example.invalid/releases/release_001/ --output C:\demo-output --json
```

完整版 EXE 自带 Python 运行时，无需安装 Python 或 Rust。Lite/Fetch 需要可用的 `curl.exe`；Git 暂存和 Git 检查需要 `git.exe`。`--help` 和 `--version` 可离线使用。首次启动完整版会解包到可写的用户缓存目录。示例域名是占位符，真实传输前必须显式填写自己的服务配置。

## 命令行契约

Python/Lite 的 `upload`、`download`、`clean`、`gui` 都要求 `--project DIR`，目录须已有 YAML 配置。无命令、缺参数、未知选项、无效配置立即失败，不会默认进入界面。不从当前目录猜测项目。GUI 只由显式 `gui` 打开，要求可交互终端；`--no-input` / `--non-interactive` 禁止交互。

| 命令/参数 | Python 完整版 | Lite |
| --- | --- | --- |
| `upload --submit ID` | 发布，ID 为 4..128 字符的单个目录名 | 同类发布 |
| `upload --quick-upload` / `--first-upload` | 按远程索引过滤；FirstUpload 仅允许空版本，不忽略网络错误 | 支持 |
| `upload --index-only` | 写配置/索引/ZIP/ID，可能暂存 Git；仍有上传前后副作用 | 支持 |
| `upload --no-check-file` / `--clear-remote` / `--git-check-ignore` | 已有文件检查、远程清理、Git 检查开关 | 不支持，报参数错误 |
| `upload --gen-symbols` / `--upload-symbols-index` | 既有 macOS 符号功能及其平台限制 | 不支持 |
| `upload --report` / `--no-print-touch` | 共享 FTP 层输出开关 | 不支持 |
| `download --local-cache` | 使用下载缓存 | 支持 |
| `download --clear-local` / `--clear-local-empty` | 下载后的范围清理/空目录清理 | 支持 |
| `clean --yes` | 删除索引及上传快照记录的本地资源，保留核心控制文件 | 支持 |
| `gui` | Tk 桌面界面 | 文本菜单 |
| `--json` | 单个成功 JSON 结果 | 同样提供 |

stdout 只包含最终结果，诊断/进度写 stderr；JSON 成功结果为 `schema_version: 1`、`ok: true`、`command`、`status: completed`。失败 stdout 为空、退出非零，诊断在 stderr。参数错误退出 2；Python 业务异常为非零，Lite/Fetch 业务错误退出 1。默认文本适用于终端，自动化应显式选择 `--json`。Lite 的非 TTY 输出及 `NO_COLOR` 环境关闭颜色。

`--yes` 只确认删除/清理，`--no-input` 不表示同意。Python 上传在未配置真值 `submit_dir` 时可能清理远程历史；配置的 `copylist`、`check_pdbfile`，下载的 `removelist`，以及显式清理开关，都可能要求 `--yes`。Lite 历史远程删除禁用，但本地复制目标清理与下载删除仍需确认。`--index-only` **不是预演**。

## Agent 入口

```powershell
.\repocargo-cli.exe status --project C:\demo-project --json
.\repocargo-cli.exe upload --project C:\demo-project --dry-run --json
.\repocargo-cli.exe upload --project C:\demo-project --yes --no-input --json
.\repocargo-cli.exe --program C:\tools\repocargo-lite.exe download --project C:\demo-project --json
.\repocargo-cli.exe exec -- download --project C:\demo-project
```

默认只选择同目录 `repocargo.exe`；`--program` 显式指定其他后端，不搜索 PATH、不自动切换。`exec --` 原样传递参数、标准流及退出码；它的 `--project` 仅设置 cwd，后端参数仍需自己的 `--project`。

`status` 和 `--dry-run` 仅读取配置及构造计划，不启动后端、不写文件、不联网。计划是配置范围，不展开为完整文件列表。`--limit` 默认 100、范围 1..1000，`--offset` 默认 0；JSON 返回 `has_more`、`next_offset`、精确配置项 `total`。分页仅限制返回条目，不限制实际操作范围。数组按原顺序，对象按键排序。

`--timeout-seconds` 默认 300，超时终止进程树；`--verbose` 返回脱敏且有界的诊断。结构化错误包含稳定 `code`、`hint`、`retryable` 和 `execution_state`，写 stderr。退出码：0 成功，2 参数/配置错误，3 程序或项目不存在/启动失败，4 需确认/凭据，6 后端失败，124 超时。后端失败/超时可能部分完成，必须检查诊断及远程状态再决定重试。`backend_exited` 不表示远程资源已逐一核验；`status` 也只检查本地。

## 配置及产物

配置采用 UTF-8 YAML 1.2，写回使用 LF。注释、字段顺序、未修改字段的引号与未知字段会保留；`upload` 等自动字段变更时允许整体替换该节点。重复键、语法错误、多文档或非映射配置失败并保留原文件。路径使用 `/`；ID、密码等含前导零或看似布尔值的字符串建议加引号；布尔值使用 `true`/`false`。不使用自定义 YAML 标签或合并键。

映射键必须为字符串，数字键请加引号；数值必须有限，不接受 `.nan`、`.inf`。普通锚点和无环别名可用，循环别名、自定义标签及 `<<` 合并键会被拒绝。Python/Lite 在写回前验证原配置及新内容，避免键转换碰撞、别名丢值或异常字段被静默删除；Agent CLI 在启动后端前拒绝这些配置，Fetch 在部署及提交版本标记前拒绝。

配置没有隐式环境覆盖；显式 `--submit` 优先于 `submit_dir`。FTP 上传专用 `*_upload` 字段覆盖普通字段；`pypass` 指定的非空环境变量优先于用户目录同名文件，再覆盖直接 `pass`。Python 共享 FTP 账号缓存及主机映射仍按已有规则生效。不要在配置里提交明文凭据。

`mainroot` 相对项目目录，资源路径相对它；`copylist` 和 `python_pipfix` 相对项目目录。`mainfile`、`others` 不允许父目录组件，需要父目录通过 `mainroot` 设置。

发布/下载映射不能越出所选 ID 目录；发现越界路径即以 `UNSAFE_RELEASE_PATH` 失败。开启清单、版本或 ZIP 生成时，请使项目控制文件位于资源根内（默认 `mainroot: .`，或让资源根包含项目目录）。子根/旁支根仍可用于普通资源；生成的控制文件需要 `../` 才能定位的布局会被拒绝。已有越界布局不自动迁移。

Lite 的 FTP 上传与 Python 一样：先把远程 MD5 置为 `null`，再传资源，最后写入新 MD5。第一步失败时不传该资源；最后一步失败时保留 `null` 并返回失败，读端重新获取资源，不能用旧摘要跳过。该顺序降低静默漏更新风险，不提供多文件原子发布、并发写隔离或失败回滚。

| 产物 | 含义 |
| --- | --- |
| `repocargo.yaml` | 项目配置与自动维护的上传快照 |
| `<mainfile文件名>.md5.txt` | 远程目录 ID；默认 `repocargo.yaml.md5.txt`，不一定是 MD5 |
| `datamap.txt` | 路径、MD5、大小、可选远程路径的文本索引 |
| `version.txt`、`version/<摘要>.txt` | 内容摘要和对应索引快照 |
| `<主文件去扩展名>_dense.zip` | 小文件包，默认 `repocargo_dense.zip` |
| `metainfo.json`、`tempdir/` 内部 JSON | ZIP 元信息及缓存，保留 JSON，不是人维护的项目配置 |

普通资源地址为 `/repocargo/<gitremote>/<目录ID>/<资源路径>`，`gitremote` 默认 `repocargo`。程序不自动 commit/push；生成索引可能执行 `git add -f`。发布不是事务，复用 ID 固定目标目录但不保证中断时整体一致性。

### 既有配置能力与版本差异

| 字段 | Python 行为/默认值 | Lite 差异 |
| --- | --- | --- |
| `mainfile` | 主文件；无配置时初始化为 `repocargo.yaml`。通配符要求带目录、单层且恰好一个文件 | 默认同名；还支持根目录的主文件通配符 |
| `others` | 文件、递归目录、通配符列表，或“路径→说明”字典 | 支持数组、字典、通配符 |
| `mainroot` | 默认 `.`；需要父目录时在这里设置，例如 `..` | 支持；父目录模式已实测 |
| `datamaplist` | 从索引追加文件路径 | 支持，非法索引行报错 |
| `ignorelist` | 扫描结果排除模式，可用字典键说明 | 支持数组/字典 |
| `gitignore` | 默认 false，将 Git 忽略规则传给扫描器；不是自动挑选全部忽略文件 | 支持 Git 忽略扫描 |
| `ignore_japanese`、`ignore_korean` | 默认 false，按扫描回调文件名过滤相关字符 | 支持 |
| `upload` | 自动维护文件列表、数量、macOS 元信息 | 自动维护列表与数量；生成侧元信息能力不同 |
| `gitremote` | 默认 `repocargo`，远程项目根目录 | 同默认值 |
| `submit_dir` | 固定上传 ID；真值还会关闭 Python 历史目录清理 | 支持 ID；时间清理本已禁用 |
| `generate_listfile` | 默认 false，生成并尝试暂存 datamap | 支持 |
| `generate_versionfile` | 默认 false，生成版本；与索引一起启用时生成快照 | 当前成品同样仅在两个开关都开启时复制快照 |
| `generate_densefile` | 默认 false，小文件合包并跳过普通包内文件的独立上传 | 支持；下载依赖索引与版本 |
| `densefile_ctrl` | 默认 524288 字节；严格小于阈值入包，0 回到默认 | 默认相同，0 按 0 处理 |
| `filemd5_cachesize` | 默认 5000，入口按数量扩容，配置最低按 1000 处理 | 不使用该缓存配置 |
| `overwrite` | 通常 true；macOS 符号索引模式默认 false | 未实现同等覆盖限制 |
| `overwrite_symbols`、`overwrite_ignore` | 符号覆盖开关及允许例外的地址模式 | 未实现 |
| `gitcheck` | 默认 false，上传前检查本地 Git 改动 | 未实现同等检查 |
| `linux_line_ending` | 保留字段；YAML 配置始终按 UTF-8/LF 写回 | YAML 配置始终按 UTF-8/LF 写回 |
| `timeout`、`timeout_reserved` | 默认 365 天、20 个条目；上传历史清理 | 写回配置并保留接口，删除逻辑已禁用，只统计列表 |
| `copylist` | `[源,目标]` 列表，递归复制；Windows 尝试附带匹配 PDB；正常结束删复制的目标文件 | 仅处理直接文件及目录一层文件，不具备同等递归/PDB 能力 |
| `python_pipfix` | 上传前处理、下载后恢复 Python 环境 | 仅实现下载后修复 |
| `removelist` | 下载后删除文件、目录或匹配路径 | 支持 |
| `check_pdbfile` | Windows 下复制匹配 PDB，或删除不匹配/缺失 PE 的 PDB | 未实现 |
| `meta_symlink` | macOS 保存软链接元信息；默认 false | 没有同等上传生成分支；下载可恢复已有元信息 |
| `autopinyin` | 非 macOS 默认 false；macOS 始终转换 | 与 Python 一致；两个 Rust 版共用固定基线词典和转换实现 |
| `autopinyin_replist` | 转换后依次执行 `[旧,新]` 替换 | 支持 |

字典示例（合并到完整配置，不是完整配置）：

```yaml
others:
  assets: 运行资源
  models/*.bin: 模型
ignorelist:
  '*.tmp': 临时文件
```

Python 的 `mainfile`、`others` 检查不允许路径含 `..`，父目录通过 `mainroot` 选择。Rust 也拒绝这些路径的父目录组件。`others` 支持 `目录|数字`：Python 按目录相对 `mainroot` 的分隔符数量限制，Rust 按扫描根的递归深度限制，两者不可当作完全相同的层数定义。


### Fetch 部署边界

Fetch 只消费已发布索引，不扫描配置 `others`，也不提供上传。支持 datamap 的现有三种格式、第四列远程路径、MD5/大小校验、小文件 ZIP、重复内容复用、输出目录锁和已有部署动作。只从本次索引指定的 `repocargo.yaml` 读取配置；旧本地配置不会触发部署。部署仅在根配置的 `mainfile: repocargo.yaml` 时执行。

`post_download` 现有动作包括 `file.copy`、`config.set`、`shell.run`、`env.path.add`；`removelist` 和元信息恢复保持已有边界。Fetch 的部署专用能力不要求 Python/Lite 补齐。下载和部署完成后才提交版本标记；部分文件/动作成功后不会整体回滚。失败保留旧目标文件，不把校验失败当作成功版本。

具体参数、动作字段和输出边界见 [Fetch 下载与部署](fetch.md)。


## 发行与边界

当前成品、平台、大小及 SHA-256 见仓库根目录 `release.json`、`SHA256.json`。本仓库不含业务源码、构建脚本或开发期测试夹具，也不提供在本仓库重建 EXE 的命令。

进一步阅读：[Agent 操作指南](agent.md)、[配置说明](configuration.md)、[交换协议](protocol.md)、[已知限制](limitations.md)、[迁移说明](migration.md)、[发行核验](validation.md)。
