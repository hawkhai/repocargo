# Fetch 下载与部署

`repocargo-fetch.exe` 是独立下载器，消费发布方已经生成的版本和索引，不扫描 `others`，不上传，不是 Agent CLI 的后端。

```powershell
.\repocargo-fetch.exe download --help
.\repocargo-fetch.exe download --url https://cdn.example.invalid/repocargo/demo/release_001/ --output C:\demo-output --json
```

将占位 URL 替换为可信发布地址，明确指定输出目录。URL 对应目录必须提供 `version.txt`、`version/<版本>.txt` 和索引所引用资源。支持 HTTP、HTTPS、FTP；FTP 额外要求 `--ftp-user`、`--ftp-pass`，没有与 Python/Lite 相同的 `pypass` 配置入口。不要把真实密码写进文档、命令历史或公开日志；命令行参数也可能被本机进程查看工具读取。

输出目录被独占锁保护；不同目录可以独立执行。下载会写 `.md5cache.json` 及持久锁文件，锁文件存在不等于仍有进程持锁。不要删除正在使用的锁来强行并发更新。

## 下载和失败

索引支持的三种格式及远端路径规则见 [交换协议](protocol.md)。下载先校验大小和 MD5，再替换对应目标；可复用已验证内容及小文件 ZIP。资源完整性失败不把坏内容替换为成功结果，版本标记在下载及部署完成后提交。

单文件保护不是整批回滚。某个文件或部署步骤失败时，之前成功的文件可能已经更新；失败后先检查状态再决定是否重试。输出目录只有旧版 `gitbigfilex.json` 时会拒绝，不自动迁移它。

## 部署条件和顺序

只有本次发布索引包含根目录 `repocargo.yaml`，且其中 `mainfile` 恰为 `repocargo.yaml`，才运行部署。未被本次索引引用的旧本地配置不会触发部署；自定义主文件不触发这一阶段。

部署前先检查全部动作类型、必填字段及删除模式，然后按顺序恢复 `upload.metainfos` 元信息、处理 `removelist`、执行 `post_download` 动作。这个预检查不提前执行命令，也不保证目标存在、权限可用或所有后续运行条件成立。下载阶段在部署前已经发生。

## 已有动作字段

| `type` | 字段 | 行为 |
| --- | --- | --- |
| `file.copy` | `from`、`to` | 从输出目录下的文件复制到目标；暂存并校验后替换单文件 |
| `config.set` | `file`、`key`、`value` | 在普通 `key=value` 文件中设置键值；不是 YAML 编辑器，不按 INI 节区定位 |
| `shell.run` | `cmd`、可选 `args` 字符串数组 | 执行指定 EXE、BAT 或 CMD；工作目录为该命令所在目录，非零退出停止后续动作 |
| `env.path.add` | `path` | 向系统 PATH 添加目录；系统注册表无写权限时尝试用户 PATH，保留已有注册表值类型 |

`config.set` 保留 UTF-8/GBK 编码及已有 UTF-8 BOM，按 CRLF 写回；不可解码内容失败并保留原文件。键不能含 `=` 或换行，值不能含换行；空值可用。

支持的文本变量为 `<HOME>`、`<CURDIR>`、`<SYSTEM_BIN>`、`<PROGRAM_FILES>`、`<TEMP>`。`<CURDIR>` 是输出目录，`<SYSTEM_BIN>` 实际取 Windows 的 `SystemRoot`，通常为 `C:\Windows`。它不是通用环境变量或 shell 表达式展开。

变量用于 `file.copy.to`、`config.set.file/value`、`shell.run.cmd/args` 和 `env.path.add.path`；`file.copy.from` 不展开变量。复制、配置文件和命令的相对路径以输出目录为基准，绝对路径可指向目录外；PATH 动作直接使用展开后的路径，推荐明确写为 `<CURDIR>/bin` 或绝对路径。

下面仅展示输出目录内的部署字段，需与发布配置合并，并将配置及所需文件纳入发布索引：

```yaml
mainfile: repocargo.yaml
upload: {}
post_download:
  - type: file.copy
    from: assets/hello.txt
    to: deployed/hello.txt
  - type: config.set
    file: deployed/settings.ini
    key: ResourceDir
    value: <CURDIR>
```

`removelist` 只接受输出目录内的相对模式，不能指向父目录、绝对路径或核心控制项。动作可以写目录外、执行程序和改环境，所以应先审阅可信发布配置并取得相关操作授权。Fetch 没有为这些部署动作提供 `--dry-run` 或额外确认提示；Agent CLI 的预演也不能模拟 Fetch 部署。

## 输出契约

成功时 `--json` 输出 `schema_version: 1`、`ok: true`、`command: download`、`status: completed`。诊断在 stderr；业务失败退出 1，参数错误退出 2，失败诊断可能是普通文本。

`shell.run` 的子进程继承标准流，可能把自己的输出写到最终 JSON 前面。需要严格解析 JSON 时，发布配置应避免有 stdout 输出的部署命令；不要将含任意 shell 动作的整个 stdout 假定为单个 JSON。
