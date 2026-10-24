# AI 使用 RepoCargo 的入口

这是成品与文档发行仓库，只有 Windows x64 EXE，没有 RepoCargo 业务源码或产品构建工具。不要假定这里可以运行 Cargo、PyInstaller 或源码单元测试。

## 阅读顺序

1. `README.md`：产品选择、依赖、校验和离线示例。
2. `docs/agent.md`：计划、执行、错误和结果语义。
3. `docs/usage.md`、`docs/configuration.md`：实际参数和字段。
4. `docs/protocol.md`、`docs/limitations.md`：产物、恢复规则与已知边界。
5. `release.json`、`SHA256.json`：当前成品身份；`docs/validation.md`：已执行的核验范围。

## 操作约定

- 从本仓库根目录开始，以 `SHA256.json` 校验四个 EXE，再运行 `--version` / `--help`。PowerShell 使用 `.\程序.exe` 或程序绝对路径，不依赖当前目录隐式搜索。
- Agent 优先使用 `repocargo-cli.exe`。默认后端仅为同目录 `repocargo.exe`；选择 Lite 必须显式 `--program`。Fetch 是独立程序。
- `--project` 必须明确指定包含 `repocargo.yaml` 的项目目录。不要把本发行仓库当成用户的资源工作目录。
- 在已有配置上先执行 `status --json` 和相应操作的 `--dry-run --json`。两者只读、不联网、不启动后端；展示的是配置范围，不是完整展开的文件列表。
- `examples/basic` 可用于离线检查。其服务域名为 `.invalid` 占位符，不可执行真实上传或下载。
- 真实网络操作必须使用用户明确提供的服务、项目和凭据。显式配置 `ftpconfig`，不要推测、提取或使用二进制内部的默认连接信息。
- 密码放在 `pypass` 所指的环境变量中；不要将真实密码、账号缓存、私钥、个人目录或私有服务地址写回本仓库。
- 实际上传、下载、清理、部署须在用户授权范围内进行；`--yes` 是确认删除/清理，不是预演开关。不要将 `--index-only` 当只读命令。
- `clean`、`--clear-local`、`--clear-remote`、配置中的 `copylist` / `removelist` 和部署动作可能删除、覆盖、执行命令或修改环境。执行前展示相关范围；GUI 只在用户要求时显式打开。
- 自动化用 `--json`、非交互输入及有限超时。解析 stdout 结果和 stderr 错误，并检查退出码。后端失败/超时可能部分完成，不盲目重试、不假定回滚。
- `status`、`planned`、`backend_exited`、文件已逐个验证是不同结论；`remote_verified: false` 不可改述为远端已核验。
- 四个版本只使用各自已实现能力；未实现参数不要靠其他工具伪造成功。
- 新旧工具及并行任务使用分开的工作/输出目录和扫描范围。同名索引、版本文件和缓存没有跨产品协调锁。

## 维护这个发行仓库

仅同步已批准的成品、公开文档、安全示例和品牌素材；不迁入业务源码、开发路径、真实配置、内部报告或构建日志。第三方许可和对应第三方源码归档不属于 RepoCargo 业务源码。

更新成品时同步 `release.json`、`SHA256.json`、`SHA256SUMS`、第三方声明及核验记录；以实际 EXE 为准，不沿用旧哈希或冒称旧测试覆盖新文件。MIT 不覆盖第三方组件原许可。

提交和 push 只在用户授权时执行；不要默认改写历史、强制推送或自行发布远程 Release。
