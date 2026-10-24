# 配置与凭据

项目根目录的配置文件为 UTF-8 `repocargo.yaml`，公开命令通过 `--project` 明确选择项目。使用 [完整安全示例](../examples/basic/repocargo.yaml)；`upload` 中的快照由工具维护，初始可为 `{}`。

## 推荐起点

```yaml
mainfile: repocargo.yaml
mainroot: .
others:
  assets/: 运行资源
upload: {}
gitremote: demo-project
submit_dir: release_001
generate_listfile: true
generate_versionfile: true
generate_densefile: true
ftpconfig:
  host: ftp.example.invalid
  port: 21
  user: demo-user
  pypass: REPOCARGO_FTP_PASSWORD
```

示例域名和账号不是可用服务。真实传输必须显式设置自己的 `ftpconfig`；不应依赖内置默认连接。没有显式连接配置，并不代表程序保证离线。

```powershell
$env:REPOCARGO_FTP_PASSWORD = '<your-password>'
```

`pypass` 是环境变量名，不是密码。非空环境变量优先于用户目录的同名文件，再覆盖直接 `pass`；不要在仓库提交这些凭据文件。上传可用 `user_upload`、`pass_upload`、`pypass_upload` 覆盖普通连接字段；下载使用普通字段。完整字段与版本差异见 [使用参考](usage.md#既有配置能力与版本差异)。

## YAML 规则

- YAML 1.2，UTF-8/LF；布尔值用 `true` / `false`。有前导零的 ID、像布尔值的文本和密码应加引号。
- 根必须为映射，所有映射键必须是字符串；数字键需加引号。
- 重复键、多文档、非法语法、循环别名、自定义标签和 `<<` 合并键被拒绝。
- 普通锚点与无环别名允许；不接受 `.nan` / `.inf` 等非有限数值。
- 注释、顺序、未知字段及未修改字段的引号尽量保留；自动节点更新可能整体替换该节点。
- 这些规则不等同于完整业务字段 Schema。公开 Python/Lite 入口要求非空 `mainfile`、列表/映射 `others`、映射 `upload`。

## 三个路径概念

项目目录 `P` 由 `--project` 指定；资源根 `R = P/mainroot`；资源条目相对 `R`。

| 内容 | 位置 |
| --- | --- |
| 配置、目录 ID、生成的 `datamap.txt` / `version.txt` | 项目目录 |
| `mainfile`、`others`、`datamaplist` 的资源路径 | 资源根 |
| `version/<版本>.txt` 快照 | 资源根的 `version/` |
| 小文件 ZIP | 主文件旁 |
| `copylist`、`python_pipfix` | 相对项目目录 |
| `removelist`、索引清理资源 | 相对资源根 |

推荐 `mainroot: .`。若资源根为父目录，例如 `mainroot: ..`，应将 `mainfile` 写为相对父资源根的配置路径。控制文件必须位于资源根内，生成的 `../` 越界传输会被拒绝。主文件/资源条目不要写父目录组件。不要把不同根布局都当成已修复或完全等价；见 [已知限制](limitations.md)。

## 发布 ID 和索引

`--submit ID` 优先于配置 `submit_dir`；公开 ID 为 4..128 字符的单个目录名。配置的真值 `submit_dir` 还关闭 Python 的自动远端历史清理。上传准备阶段会写默认的 `repocargo.yaml.md5.txt`，下载方需要这份文件；它的存在不证明传输成功，内容也不一定是 MD5。

同时开启清单和版本开关才生成匹配快照。小文件合包推荐与两者同时启用。索引、ZIP 元信息及缓存仍采用机器格式，只有人工维护的项目配置改为 YAML。

## 修改本机和部署相关字段

`copylist` 目标可能覆盖并在正常收尾时删除；`removelist`、清理参数会删除本地文件。`python_pipfix` 可能修复环境。Fetch 的 `post_download` 可复制文件、写配置、执行 shell 和修改 PATH。它们不是仅描述信息，使用前必须检查实际作用范围。

不要把带真实地址、凭据或业务路径的项目配置复制回这个发行仓库。公共示例仅使用保留域名、假账号和环境变量名。
