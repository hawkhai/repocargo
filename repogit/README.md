# RepoGit

RepoGit 0.3.0 是通用 Git 同步工具：**给它一份授权配置和一个合法 Git 仓库，就能推送已有提交。** 应用、智能体、插件和普通项目都可以使用，无需客户 GitLab 账号，个人 SSH 私钥不参与。

一个 Windows EXE 可放在任意位置，依赖 **Git 和 OpenSSH 的 `ssh`、`ssh-keygen` 已加入 PATH**。密钥、SSH 配置不随 EXE 移动。命令默认操作当前目录所属的 Git 项目，也支持在任意位置传入 `--path`。不需要服务器或客户端账号系统。

GitHub 发行目录仅提供 Windows x64 成品与使用文档，不包含产品源码或构建脚本。原有设备模式的完整步骤见 [从零开始使用](docs/QUICKSTART.md)。所有 gitlab.example.invalid 地址均为占位符，必须替换为自己的项目 SSH 地址。

## 两个独立程序

| 程序 | 主要使用者 | 输出与执行方式 |
| --- | --- | --- |
| `repogit.exe` | 人在终端中操作 | 保留文本、Git 进度和原有命令习惯 |
| `repogit-cli.exe` | Codex、脚本及其他 Agent | 默认单个 JSON 信封，禁止交互，支持离线 schema、本地预演、分页、字段筛选和执行超时 |

两个 EXE 共享 Git/SSH 和凭据逻辑，都可单独放在任意目录运行，不互相转调或要求放在同一目录。Agent 优先读 [命令行协议](docs/AGENT_PROTOCOL.md)，再按需查询：

```powershell
repogit-cli schema
repogit-cli schema push
repogit-cli -C "D:\Repos\project" status --fields repository,head,dirty,binding
repogit-cli -C "D:\Repos\project" push --dry-run
repogit-cli -C "D:\Repos\project" push --timeout-seconds 60
```

`-C` 是 Agent 版对 `--path` 的 Git 风格简写。`--dry-run` 只读本地，不代表 GitLab 已授权。Agent 版成功和失败都把 JSON 写 stdout，必须同时检查进程退出码、`ok` 与 `execution_state`；`--help`、`--version` 是离线文本输出。

## 推荐：生成一份配置，交给同事和 AI

管理员先准备空的私有 GitLab 项目，并核验 SSH 主机指纹，保存到自己的 `known_hosts`。然后生成独立授权文件：

```powershell
repogit access create --name "同事甲-项目甲" --repo "git@gitlab.example.invalid:group/project.git" --out "同事甲-项目甲.repogit.json"
repogit access show --config "同事甲-项目甲.repogit.json"
```

将输出的公钥添加到项目 Deploy Keys 并授予读写，再把生成的配置文件交给同事。配置是可打开的 UTF-8 JSON，包含仓库、默认 `sync` 分支、公钥、加密私钥和可信主机公钥。**文件即钥匙**：固定口令 `repogit` 由工具自动处理，不提供防泄露保障。每位接收人分别生成凭据，泄露时在 GitLab 撤销对应授权。

同事可直接告诉 AI：“使用 [RepoGit](https://github.com/hawkhai/repocargo/tree/main/repogit) 和这份配置，把指定 Git 仓库推上去。”AI 先读 [操作指南](docs/AI_GUIDE.md)，执行：

```powershell
repogit access show --config "同事甲-项目甲.repogit.json" --json
repogit --path "D:\Repos\project" bind --config "同事甲-项目甲.repogit.json"
repogit --path "D:\Repos\project" doctor
repogit --path "D:\Repos\project" push
```

导入时自动解锁，凭据保存在用户应用数据目录，后续无需输入口令。没有已有提交时停止；未提交修改不上传，也不自动创建提交。下载使用 `repogit clone --config "同事甲-项目甲.repogit.json" "D:\Repos\project"`。完整流程与边界见 [授权配置说明](docs/ACCESS_CONFIG.md)。

## 兼容方式：每台电脑本机生成设备密钥

### 1. 每台客户电脑生成自己的密钥

```powershell
repogit device init
```

在此设备模式下，按 OpenSSH 提示设置私钥口令；已有密钥不会被覆盖。只把输出的 **`ssh-ed25519 ...` 公钥和指纹**交给管理员，本机设备私钥不转交。上述 access 模式使用另行新建的专用交付凭据。

Windows 默认密钥位置：`%LOCALAPPDATA%\repogit\keys\id_ed25519`。可选环境变量只指定路径：

```powershell
$env:REPOGIT_KEY_FILE = "C:\Keys\repogit-device-ed25519"
repogit device init
```

这个变量不是私钥内容，也不会让密钥变得不可提取。新生成的私钥会限制文件权限。在明确接受仅靠本机文件权限保护时，可用 `device init --no-passphrase` 无交互生成。`repogit device show` 可再次输出公钥及指纹，不输出私钥。

### 2. 管理员授权

为每个应用建立独立的 **私有 GitLab 项目**，首次上传的空项目不要初始化 README。把设备公钥添加到该项目的 Deploy Keys，并按需启用写权限。具体步骤见 [GitLab 授权说明](docs/GITLAB_SETUP.md)。

同一设备可获准访问多个应用。不要让不同客户共享私钥，也不要通过同仓库不同分支隔离保密源码。

### 3. 绑定已有应用

```powershell
repogit --path "D:\Projects\my app" bind --repo "git@gitlab.example.invalid:your-group/your-app.git"
```

这是地址示例，请从实际 GitLab 项目复制 SSH 地址。工具不会猜测公司 SSH 端口。非标准端口使用：

```powershell
repogit bind --repo "ssh://git@gitlab.example.invalid:2222/your-group/your-app.git" --branch sync
```

默认远程同步分支为 `sync`，本地分支仍可叫 `main`、`master` 或其他名称。`bind` 不联网，不承诺已经授权；随后运行 `repogit doctor` 检查读取权限。

`bind` 新增 `repogit` remote，只抓取指定同步分支，默认推送规则为 `HEAD:refs/heads/sync`。原有 `origin`、本地分支和提交历史保持不变；目标冲突时拒绝覆盖。相同绑定可以重复执行。

## 日常使用

```powershell
git add app.py
git commit -m "修复配置读取"
repogit push

repogit status
repogit pull
```

- `push` 只上传当前 `HEAD` 已有的提交，不自动提交未提交文件，不强推、不推其他分支或标签。未提交变更会保留并得到提示。
- `pull` 把远程同步分支快进到**当前本地分支**，不自动合并或变基。为避免覆盖，存在未提交/未跟踪文件时会拒绝；请自行提交或 stash。历史分叉时使用正常 Git 流程处理，再重试。
- `status` 只读本地状态，不联网或写 SSH 配置；`doctor` 检查工具、绑定和实际读取权限，不会通过测试推送来验证写权限。
- Git 的传输进度和错误会原样输出。工具以失败子进程的退出码退出（Git 可能将 SSH 的失败转换为自己的退出码）。

配置后也能直接使用普通 Git：

```powershell
git push repogit
git pull --ff-only repogit sync
```

这些普通 Git 命令沿用 Git 自身行为；例如拉取前不会执行 repogit 的“所有未提交文件必须先处理”检查。

### 在另一台获准电脑取回应用

新电脑先生成自己的设备密钥，由管理员授权，然后：

```powershell
repogit clone --repo "git@gitlab.example.invalid:your-group/your-app.git" "D:\Projects\my app"
```

默认克隆 `sync` 分支，创建 `repogit` remote 并自动绑定。远程必须已有这个分支；其他同步分支使用 `--branch` 指定。工具不会删除已有目录或自动覆盖冲突目录。

## SSH 配置和密钥更换

如果此前使用过早期版本，原有密钥不会自动迁移或重新生成。请通过 `REPOGIT_KEY_FILE` 指向已有的设备私钥，然后重新执行绑定。当前版本统一使用 `repogit` remote、`repogit.binding` 配置键和 `repogit-*` SSH 别名；原有 remote 与 SSH 配置块保留，由用户确认后自行清理。

工具在当前用户 `~/.ssh/config` 顶部写入带明确标记的专用 `Host repogit-...` 配置块。别名按目标地址稳定生成，设置独立 `IdentityFile`，关闭 agent、密码和键盘交互认证，保留正常主机密钥检查。其余 SSH 内容保持原样，配置失败时恢复工具本次的修改。

首次联网时必须向管理员核对 GitLab 主机密钥指纹，确认后由 OpenSSH 保存。不要为了消除提示关闭主机密钥检查。设备私钥口令仍可按 OpenSSH 提示输入。

如果已有 `Host *` / `Include` 添加了个人 `IdentityFile`，SSH 会累加身份。工具会通过 `ssh -G` 发现并拒绝连接，不会悄悄尝试个人密钥。可将用户自己的通配规则改为例如：

```sshconfig
Host * !repogit-*
    IdentityFile ~/.ssh/personal_key
```

工具不自动修改这些用户规则。现有 `core.sshCommand`、`GIT_SSH_COMMAND`、`GIT_SSH`、非 OpenSSH variant、代理跳板或 URL 重写冲突也会被提示，避免接管原有认证方式。

更换私钥时，先生成新设备密钥并在 GitLab 授权，再在当前会话设置路径并运行相同绑定或 `doctor`：

```powershell
$env:REPOGIT_KEY_FILE = "C:\Keys\new-device-key"
repogit device init
# 将新公钥交给管理员授权
repogit doctor
```

成功后绑定会记住新路径，SSH 别名也随之更新。以后普通 Git 不需要这个环境变量。最后在 GitLab 撤销旧公钥授权。若同一设备绑定多个应用，应逐个运行 `doctor` 更新；同一目标的多个工作副本共享 SSH 别名，不要让它们交替使用不同密钥。

## 命令与环境变量

```text
repogit [--path DIR] device init [--no-passphrase]
repogit [--path DIR] device show
repogit [--path DIR] bind --repo SSH_URL [--branch sync]
repogit [--path PARENT] clone --repo SSH_URL [--branch sync] DIRECTORY
repogit [--path DIR] push | pull | status | doctor
```

| 变量 | 用途 |
| --- | --- |
| `REPOGIT_KEY_FILE` | 设备私钥路径；首次默认使用应用数据目录；绑定后未设置此变量则使用已记住的路径 |
| `REPOGIT_HOME` | 覆盖默认应用数据目录，供便携部署或隔离测试使用 |
| `REPOGIT_SSH_CONFIG` | 覆盖 SSH 配置文件位置，主要用于隔离测试；普通 Git 需自行传 `ssh -F`，repogit 会自动指定 |
| `REPOGIT_SSH` | 指定 OpenSSH 可执行文件的完整路径；默认从 PATH 查找 |

工具不使用 GitLab API 或管理令牌，不自动创建项目，也不读取你的个人私钥。它的绑定信息保存在本地 Git 配置 `repogit.binding`，不会作为项目源码提交。导入配置后，`REPOGIT_KEY_FILE` 与绑定身份不一致时明确报错，不静默换密钥。

同一仓库在同一设备只支持一套同步绑定；链接 worktree 共享仓库配置。`status` 会打印绑定仓库、分支及路径，不打印私钥正文。SSH 配置要求 UTF-8；密钥路径避免引号、控制字符、`%`、`$` 等会触发 SSH 展开的字符。

## 权限边界

GitLab 强制设备可访问哪些仓库；客户端绑定只负责固定操作目标、防止误操作，不是防篡改机制。要限制其他分支和标签，必须在 GitLab 配置并验收服务器规则。客户仍能使用其设备密钥直接运行 Git。

停用设备必须撤销它在所有相关项目上的 Deploy Key 授权。撤权不删除已下载代码，也不终止所有已开始的连接。本机管理员仍有能力取得设备私钥；独立设备密钥是为了避免泄露你的个人身份并能逐台撤权。

第一版不承诺 Git LFS、子模块的独立认证或自动同步。不支持任意 SSH 命令、自动解决冲突、自动提交、强推或远程项目创建。

支持普通仓库、linked worktree（`.git` 为文件）和 bare 仓库。Bare 仓库支持绑定、status、doctor、push，没有工作区因此不能 pull。RepoGit 按 Git 解析实际仓库根；资源在父仓库内时会涉及整个仓库历史，不按目录抽取源码。资源识别技能是可选辅助，不影响这些通用能力。

## 下载与校验

使用本目录的 `repogit.exe`，或解压 `repogit-windows-x64.zip`。PowerShell 中运行当前目录程序须使用 `.\repogit.exe`；也可把程序目录加入 PATH 后使用文中的 `repogit` 命令。客户只需 Git 和 OpenSSH，不需要 Rust。

在本发行目录运行，校验 EXE 与 ZIP：

```powershell
$hashes = Get-Content .\SHA256.json -Raw | ConvertFrom-Json
foreach ($entry in $hashes.PSObject.Properties) {
    $actual = (Get-FileHash -LiteralPath $entry.Name -Algorithm SHA256).Hash.ToLowerInvariant()
    if ($actual -ne $entry.Value) { throw "SHA-256 mismatch: $($entry.Name)" }
}
.\repogit.exe --version
.\repogit.exe --help
```

ZIP 内的 `SHA256SUMS.txt` 只记录 EXE；本目录外部校验文件同时记录 EXE 与 ZIP。

[release.json](release.json) 记录发行身份；[VALIDATION.json](VALIDATION.json) 与 [验证范围](docs/VALIDATION.md) 记录核验结果。EXE 沿用通过干净构建与全部测试的成品，ZIP 随公开文档与第三方许可一起分发。

0.1.0 的真实 GitLab 授权、只读和撤权结果见 [历史实测](docs/GITLAB_ACCEPTANCE.md)，**B 是本机隔离模拟**。该记录对应旧 EXE 哈希，不能证明新版本已经实测。0.2.0 授权文件流程的验证范围见 [当前 GitLab 记录](docs/GITLAB_ACCESS_ACCEPTANCE.md)；真实双机和严格分支/标签保护需要另行验收。

第三方组件保留各自许可，见 [第三方声明](THIRD_PARTY_NOTICES.md)。本工具的发行不包含 Git 或 OpenSSH 本体。
