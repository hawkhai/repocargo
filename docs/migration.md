# 配置迁移与旧工具共存

本仓库已提供四个 RepoCargo 0.2.0 Windows 成品，不是待迁入源码的占位仓库。RepoCargo 业务源码和构建脚本不在这里；无需安装开发环境即可运行成品。

## 从 RepoCargo 0.1 JSON 切换

另建 UTF-8 `repocargo.yaml`，将对象/数组转为 YAML 映射/序列，保留原 `repocargo.json` 作为历史配置。工具不自动转换或回退 JSON。

默认 `mainfile` 改为 `repocargo.yaml`，目录 ID 默认改为 `repocargo.yaml.md5.txt`，小文件包仍为 `repocargo_dense.zip`。配置字节参与摘要，必须用新配置重新生成索引、快照、ZIP 和 ID；不能仅改名复用旧摘要。

公开调用改为 `upload|download|clean|gui --project DIR`；Fetch 为 `download --url URL --output DIR`。公开斜杠参数和无参数自动打开界面的行为不再保留。先用 CLI 做只读检查和预演。

## 与 gitbigfileftp 并行保留

| 项目 | gitbigfileftp | RepoCargo |
| --- | --- | --- |
| 配置 | `gitbigfilex.json` | `repocargo.yaml` |
| 默认 ID | `gitbigfilex.json.md5.txt` | `repocargo.yaml.md5.txt` |
| 默认 ZIP | `gitbigfilex_dense.zip` | `repocargo_dense.zip` |
| 普通资源根 | `/gitremote` | `/repocargo` |
| 符号根 | `/macos_symbols` | `/repocargo-symbols` |

新程序不自动改写旧配置或迁移旧服务器数据。仅有旧配置时应另建新配置；不要将提示理解为必须重命名或删除旧文件。自定义主文件仍按配置派生产物。

两套工具应使用独立工作/输出目录和扫描范围。同名 `datamap.txt`、`version.txt`、`version/` 和缓存没有整体重设计；新旧锁不互相协调。旧配置一旦进入新索引，清理时可能被当普通资源删除。

## 建议步骤

1. 保留旧 EXE、项目、配置、ID 和发布地址。
2. 建立新的资源工作目录和 YAML，参考安全示例填写资源与自己的服务，不复制真实凭据进本发行仓库。
3. 核对 `mainroot`、`mainfile` 及生成控制文件布局；推荐从 `mainroot: .` 开始。
4. 使用 `status` 和 `--dry-run` 检查；确认配置、覆盖/删除范围后，再用新工具发布。
5. 将新配置、ID、索引和快照等应跟踪的控制文件提交到资源项目；配置中的密码用环境变量引用。
6. 在独立下载目录验证自己的有效发布地址；不要让新旧工具并发写相同目录。

这份发行仓库不附带旧 EXE，不提供旧服务器或真实项目数据。[能力差异](usage.md#既有配置能力与版本差异) 和 [限制](limitations.md) 不能因迁移而忽略。
