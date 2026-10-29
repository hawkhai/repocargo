# 给 AI 的 RepoGit 操作指南

用户可以直接说：

> 使用 https://github.com/hawkhai/repocargo/tree/main/repogit 和这份授权配置，把指定 Git 仓库的已有提交推送到服务端。

也可以提供资源名称，例如在线应用、本地应用、智能体或其他项目。资源识别是辅助能力，**任何合法 Git 仓库都能走相同主流程**。

## 下载并核验工具

从上述发行目录获取 repogit.exe 和 SHA256.json，固定到同一个 Git 提交下载，按清单核对哈希，再运行 `repogit --version`、`repogit --help`。需要 0.2.0 或更新版本。发行目录没有业务源码或构建工具，客户不需要 Cargo。已有匹配且校验通过的版本可复用；不替用户关闭系统防护。

## 生成配置给管理员

得到明确仓库 SSH 地址、接收人标签和输出路径后，用 `access create` 生成独立配置，分支默认 sync，输出路径放在项目外。将配置路径、公钥和指纹告诉用户；有文件打开工具时为用户打开配置，便于在 GitLab 添加 public_key。不要直接读取、回显或粘贴包含私钥的 JSON 全文。缺少可信主机公钥时，请管理员完成指纹核验后继续，不盲目信任扫描结果。

用户自行在 GitLab 添加 Deploy Key 并选择只读/读写，然后转交授权文件。配置中的固定口令由 RepoGit 自动处理，文件即凭据，不能描述成防泄露加密。

## 定位仓库

- 有路径时，直接用 Git 判断实际仓库根、HEAD 和状态；不要因为缺少特定应用清单而拒绝普通仓库。
- 只有资源名称时，若已安装 easyclawdev-next 或 easyclawdev，使用当前技能按其类型、来源和仓库上下文定位在线/本地应用、智能体、Workflow、Plugin、Skill、知识库等；读取当前技能实际返回的源码和 VCS 路径，不硬编码安装目录。只调用与定位直接相关的能力，不触发自动提交、升级迁移、构建或应用部署。
- 没有识别技能时仍可操作普通 Git 路径；确实缺少路径或多个候选无法区分时，再向用户询问。
- 若实际仓库根是资源父目录，推送会涉及整个仓库历史。先明确用户授权的实际仓库范围，不自动抽取目录、创建新历史或推送工作区其他项目。

## 检查、绑定、推送

```powershell
repogit access show --config "C:\Credentials\project.repogit.json" --json
git -C "D:\Repos\project" rev-parse --is-bare-repository
git -C "D:\Repos\project" rev-parse --verify HEAD
repogit --path "D:\Repos\project" bind --config "C:\Credentials\project.repogit.json"
repogit --path "D:\Repos\project" status
repogit --path "D:\Repos\project" doctor
repogit --path "D:\Repos\project" push
```

仓库是普通类型时再运行 `git status --short`；bare 仓库没有工作区。用户另外指定的仓库地址若与公开摘要不一致，停止核对；不要修改配置来迁就目标。

本工作流只推已有提交。没有 HEAD 时告知先完成 Git 提交，不自动 init/add/commit。存在未提交文件时说明它们不会上传；推送成功也不能报告这些修改已同步。保留原有 origin，不强推、不自动合并或变基、不为用户设置分支/标签服务器规则。导入身份冲突时不要删除用户既有密钥或配置。

doctor 通过只表示可读；检查 push 的真实退出状态并核对目标分支提交。分叉、权限不足、撤权或主机公钥不匹配时，保留本地内容并报告实际失败原因，不把报错包装成成功。结果只报告仓库、分支、提交和未上传内容，不输出私钥、凭据正文或口令输入记录。

## 下载与后续使用

新工作副本用 `repogit clone --config 配置文件 目标目录`；目标同步分支必须已存在。已有绑定后使用 `repogit pull`（仅快进，要求工作区干净）和 `repogit push`。Bare 仓库只用绑定、检查和推送。

应用识别成功不代表 GitLab 已授权，Git 操作成功也不代表应用已部署。权限及验证边界见 [配置说明](ACCESS_CONFIG.md) 和 [当前实测范围](GITLAB_ACCESS_ACCEPTANCE.md)。
