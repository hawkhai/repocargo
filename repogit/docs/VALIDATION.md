# 当前成品的验证范围

当前版本 RepoGit 0.3.0，Windows x64，包含 repogit.exe 和独立 Agent 程序 repogit-cli.exe。两个 EXE 与 ZIP 的身份、大小和哈希见 release.json、SHA256.json、SHA256SUMS.txt；构建和验证摘要见 [VALIDATION.json](../VALIDATION.json)。

发布门禁在全新的编译产物目录执行干净构建、完整单元/集成测试、rustfmt、Clippy 及 CLI 检查。集成测试使用候选 ZIP 解压出的同一发布 EXE；只复用下载的依赖包，不复用旧编译产物。构建期间源文件改变、任一步骤失败或最终哈希不符均不接受新发布包。

新增测试使用真实 ssh-keygen 验证密钥加密、固定口令导入和公私钥配对，使用真实 ssh -G 检查专用身份及严格主机校验配置，Git 的读写通过隔离测试传输接到本地 bare 仓库。模拟只读/撤权测试不代表 GitLab 实测通过。

Agent 测试覆盖无工具/无 TTY 的离线发现、JSON 成败通道、参数纠错、分页、字段选择、只读预演、HEAD 前置检查、配置文件重复创建拒绝、子进程输出保密，以及真实 Windows Job Object 终止 Git/SSH 后代。两个程序都从候选 ZIP 解压后参与完整集成测试；最终 ZIP 中再核对两者哈希与全部文件内容。

[历史 GitLab 实测](GITLAB_ACCEPTANCE.md) 明确对应 0.1.0 和旧 EXE 哈希；[0.2.0 授权配置流程实测](GITLAB_ACCESS_ACCEPTANCE.md) 单独记录实际状态。B 的同机隔离模拟不能称为真实双机验证。严格分支/标签规则、Git LFS、子模块及跨网络尚未覆盖。

0.3.0 Agent 的实际验证范围单独记录在 [Agent 验证记录](AGENT_VALIDATION.md)，不继承旧成品的真实服务验证结论。

发行 ZIP 包含公开文档、两个 EXE、第三方许可和两个 EXE 的校验清单；ZIP 自身哈希记录在外部发行清单，避免归档自引用。所有真实授权文件、内部日志和密钥留在本机，不进入发行包。
