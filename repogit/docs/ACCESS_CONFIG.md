# 一份授权配置交给同事

RepoGit 0.2.0。主线只需要 Git 仓库，不限制应用、智能体或项目类型。生成端和使用端都需要 Windows x64、Git、OpenSSH 的 ssh/ssh-keygen，客户不需要 Rust 或 GitLab 账号。

## 管理员生成、授权、交付

1. 每个项目准备独立私有 GitLab 仓库，接收已有历史的空仓库不初始化 README。
2. 从项目复制 SSH 地址，向可信管理员核对主机指纹。生成配置使用当前用户 `~/.ssh/known_hosts` 中匹配主机和端口的记录，支持哈希主机名；不会盲目信任 ssh-keyscan 结果。可以传 `--known-hosts "C:\Keys\verified_known_hosts"` 指定已核验文件。
3. 让 AI 执行下列命令，每位接收人、每个仓库单独生成一份。已有输出文件拒绝覆盖。

```powershell
repogit access create --name "同事甲-项目甲" --repo "git@gitlab.example.invalid:group/project.git" --branch sync --out "同事甲-项目甲.repogit.json"
repogit access show --config "同事甲-项目甲.repogit.json"
```

4. 打开生成的 JSON 文本文件，从 `public_key` 字段复制完整公钥，在 GitLab 项目 Settings → Repository → Deploy Keys 添加。需要推送时勾选写权限。也可从 `access show` 输出复制公钥与指纹。
5. 将这一份配置文件交给指定同事。它包含专门新建的加密私钥，不包含个人 SSH 私钥。固定口令 `repogit` 由工具自动处理，**不能防止持有文件的人使用它**；文件本身应按仓库钥匙保管，不放入项目 Git、公共仓库或公开聊天记录。

## 同事或 AI 使用

```powershell
repogit access show --config "同事甲-项目甲.repogit.json" --json
repogit --path "D:\Repos\project" bind --config "同事甲-项目甲.repogit.json"
repogit --path "D:\Repos\project" status
repogit --path "D:\Repos\project" doctor
repogit --path "D:\Repos\project" push
```

仓库与分支以配置为准，`--config` 不和 `--repo`、`--branch` 混用。`bind` 导入并配置身份但不联网，`doctor` 只验证读取；首次 `push` 才证明实际写权限。授权文件包含已核验主机公钥，使用端严格验证，不弹出口令或主机信任询问；主机公钥不匹配时直接拒绝，应由管理员重新核验、发放新配置。

首次取回已有同步分支：

```powershell
repogit clone --config "同事甲-项目甲.repogit.json" "D:\Repos\project"
```

相同配置可重复导入；不同凭据、仓库、分支或主机信任冲突会停止，不静默替换已有绑定。同一电脑上的同一目标共享 SSH 别名，不要交替导入不同身份。导入后绑定记住本机密钥路径，不必每次保留环境变量或提供配置文件；普通 Git 仍可使用 `git push repogit`。自定义 `REPOGIT_SSH_CONFIG` 的用户须为普通 Git 显式指定相同的 `ssh -F` 文件。

## 存储与防误提交

配置字段为 `format_version`、`name`、`repo`、`branch`、`public_key`、`fingerprint`、`encrypted_private_key`、`host_keys`。文件格式版本为 1；不要手工拼接或编辑密钥、指纹和主机公钥。`access show --json` 仅输出公开摘要，不输出私钥；AI 应传文件路径给工具，不直接读取或回显全文。

导入目录为 `%LOCALAPPDATA%\repogit\credentials\<公钥标识>\`，可以由 `REPOGIT_HOME` 指定应用数据根，但不能位于目标项目内。工具自动解锁后保存无口令的本机私钥，并限制文件访问权限；这不是操作系统凭据保险箱或硬件防复制机制。其余文件包含公钥、专用 known_hosts 和无私钥的配置摘要。

若 `REPOGIT_KEY_FILE` 指向其他密钥，先移除该变量再导入，工具不会回退个人身份。配置已经被 Git 跟踪时拒绝使用；未跟踪配置在仓库内时加入 `.git/info/exclude` 的精确本地排除规则，不改项目 `.gitignore`。已提交或外发过的秘密不能靠本地排除收回，应撤销并重新发放。

## Git 与权限边界

只推已有 HEAD，不自动 add/commit、不强推、不推其他分支或标签。未提交修改不会上传；没有提交时停止。推送整个仓库的已有历史，不是单独文件夹备份。普通仓库、linked worktree 与 bare 仓库都可推送；bare 没有工作区，不能 pull。

同一文件复制到多台电脑后，这些副本使用同一身份，撤权会影响全部副本，不能宣称仍可按物理设备区分。要单独撤销就分别生成授权文件。配置固定目标用于防误操作，GitLab 决定实际可访问哪些仓库；未配置服务器保护规则时，不能宣称密钥只能写 sync。撤权阻止新的远程访问，不收回已下载代码。
