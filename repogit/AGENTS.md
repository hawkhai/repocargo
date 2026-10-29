# RepoGit：AI 操作入口

这是 Windows x64 成品与文档目录，依赖 Git、OpenSSH，不包含业务源码或构建工具。Agent 优先使用独立的 repogit-cli.exe，先读 [命令行协议](docs/AGENT_PROTOCOL.md)，按 SHA256.json 校验两个成品。repogit.exe 继续服务人类终端；两者不互相依赖。

repogit-cli 默认成功/失败都输出一个 JSON 到 stdout，退出非零时检查 error.code、hint、execution_state；stderr 默认空，verbose 只给进程计数，不暴露原始日志。先 status，再写操作 --dry-run，明确范围后执行。预演不联网、不证明远程授权；超时或 may_have_changed/result_unknown 时先核对状态，不盲目重试。无需 --yes，没有删除或 force 入口。

主线能力：对任意合法 Git 仓库，将已有 HEAD 推送到授权配置指定的仓库与分支。资源类型不限；easyclawdev-next / easyclawdev 只是可选定位辅助，不安装它们也可以使用 RepoGit。不要执行应用构建、部署或自动提交。

配置文件即访问凭据，固定口令 repogit 不提供防泄露保障。使用 access show --config 文件 --json 查看公开摘要，不 cat 配置、不输出私钥、不把配置正文写入对话或日志。设备模式的本机私钥不转交；access 模式只转交专门新建的授权文件，个人 SSH 私钥始终不参与。

使用用户授权的目标和目录。实际仓库根可能是资源的父目录，推送包含整个 HEAD 的历史；意图不清时先核对范围。只推已有提交，不自动提交、强推、合并或修改已有 origin；工作区未提交文件不会上传。doctor 只验证读取。权限由 GitLab 执行，客户端固定分支不构成服务器分支限制。

历史 [GitLab 实测](docs/GITLAB_ACCEPTANCE.md) 仅覆盖其标明的 0.1.0 EXE；B 是本机隔离模拟。新授权文件流程的实际验证范围以 [当前记录](docs/GITLAB_ACCESS_ACCEPTANCE.md) 和 VALIDATION.json 为准，不把历史或模拟结果沿用到新成品。
