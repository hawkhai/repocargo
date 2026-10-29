# RepoGit 完整操作步骤（两台 Windows 电脑）

目标：电脑 A 上传已有 Git 项目，电脑 B 取回并继续开发。每台电脑使用自己的 Deploy Key，不注册客户 GitLab 账号，不共享管理员私钥。下面的 SSH 地址是占位符，请从自己的 GitLab 项目复制真实地址。

## 1. 两台电脑准备程序

把 `repogit.exe` 放入 `C:\Tools\RepoGit`，在 PowerShell 中执行：

```powershell
$env:Path = "C:\Tools\RepoGit;$env:Path"
git --version
ssh -V
ssh-keygen -?
repogit --version
```

`ssh-keygen -?` 显示帮助时可能返回非零状态；此处只是确认命令存在。缺少 Git 或 OpenSSH 时先安装并加入 PATH。以上 PATH 只影响当前终端，新终端可重新设置，也可自行在用户环境变量里加入程序目录。

## 2. 电脑 A 生成设备密钥

```powershell
repogit device init
repogit device show
```

按提示设置私钥口令。只把 `ssh-ed25519 ...` 公钥和指纹交给管理员。默认私钥在 `%LOCALAPPDATA%\repogit\keys\id_ed25519`，不能发送给任何人，也不要放入项目。

如需自定义位置，先设置 `$env:REPOGIT_KEY_FILE = 'C:\Keys\device-a-ed25519'` 再生成和绑定；变量保存的是文件路径。已有密钥不会被覆盖。确实需要无人值守且接受本机文件权限保护时才使用 `device init --no-passphrase`。

## 3. 管理员创建项目并授权 A

1. 在 GitLab 创建独立的私有项目，不初始化 README、许可证等文件，保持远程没有提交。
2. 项目 → Settings → Repository → Deploy Keys，添加 A 的公钥；名称建议包含客户、设备和日期。
3. 勾选写入权限（Grant write permissions / Write access），保存。客户不需要登录 GitLab。
4. 从项目 Code / Clone 页面复制 SSH 地址，确认 SSH 端口和网络/VPN要求，并向客户提供可信的服务器主机密钥指纹。

标准示例：`git@gitlab.example.invalid:my-group/my-app.git`；自定义端口示例：`ssh://git@gitlab.example.invalid:2222/my-group/my-app.git`。HTTPS 地址不能直接用于 `--repo`。

## 4. A 绑定已有 Git 项目并首次推送

```powershell
Set-Location 'D:\Projects\my app'
git status
git log -1 --oneline
repogit bind --repo 'git@gitlab.example.invalid:my-group/my-app.git' --branch sync
repogit status
repogit doctor
repogit push
```

`doctor` 首次联网时，核对管理员给的主机指纹后再确认；不要关闭主机密钥验证。`doctor` 通过表示能够读取远程，不证明可写。空仓库可以先检查认证，再用首次 `push` 创建 `sync` 分支。

项目必须已有提交。需要上传新修改时，先明确选择文件再提交：

```powershell
git add README.md src
git commit -m '首次同步应用'
repogit push
```

按实际项目替换文件名；不要将私钥、密码或 `.env` 加入提交。若 Git 提示缺少身份，在该项目中用 `git config user.name '你的名字'` 和 `git config user.email '你的邮箱'` 设置提交信息；它们不用于 GitLab 登录或授权。

绑定保留已有 `origin`、提交历史和本地分支名。`repogit` remote 将当前 HEAD 推送到远程 `sync`，不会自动提交、强推或上传全部标签。推送后应在 GitLab 页面确认 `sync` 分支及最新提交。如果空项目已配置禁止创建该分支的保护规则，管理员应先按服务器策略允许首次创建。

## 5. 电脑 B 生成独立密钥并获授权

B 按第 1 步准备工具，然后运行：

```powershell
repogit device init
repogit device show
```

管理员将 B 的公钥加入同一项目。只下载则只读；需要提交修改则授予写入权限。不要复制 A 的私钥到 B。

## 6. B 首次取回项目

等 A 已创建远程 `sync` 分支后，在 B 执行：

```powershell
repogit clone --repo 'git@gitlab.example.invalid:my-group/my-app.git' --branch sync 'D:\Projects\my app'
Set-Location 'D:\Projects\my app'
repogit status
repogit doctor
git log -1 --oneline
```

克隆目标目录应不存在；工具自动创建 `repogit` remote 并完成绑定。首次取回使用 `clone`，之后才使用 `pull`。如果 B 已有本地 Git 项目，应在该项目中使用 `bind`，先检查双方历史，不要向已有目录强行克隆或覆盖文件。

## 7. 两台电脑日常协作

```powershell
repogit pull
# 编辑文件后，选择需要提交的文件
git add README.md
git commit -m '更新说明'
repogit push
```

每次开始开发前先拉取。`pull` 仅快进更新当前本地分支，遇到任何未提交/未跟踪文件或历史分叉会停止。推送被拒绝时保留本地提交，按普通 Git 流程合并或变基并解决冲突，再重试；不要强推覆盖他人工作。

配置完成后也可 `git push repogit`、`git pull --ff-only repogit sync`。项目在别处时可用 `repogit --path 'D:\Projects\my app' status` 等命令。

## 8. 排查与撤权

先运行 `repogit status` 检查绑定，再运行 `repogit doctor` 检查读取。读取失败通常需核查密钥授权、主机/端口和网络；可读不可写需核查 Deploy Key 的写权限、到期时间和服务器保护规则。SSH 配置冲突时按 [README](../README.md) 处理，不要让工具回退到个人私钥。

停用设备时，在它获准的每个 GitLab 项目中撤销对应 Deploy Key。撤权阻止后续访问，不会收回已经下载的源码。严格限制可写分支或标签必须配置并验收 GitLab 服务器规则，见 [管理员说明](GITLAB_SETUP.md)。
