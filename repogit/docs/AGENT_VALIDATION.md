# RepoGit 0.3.0 Agent 验证范围

本记录对应新增的 repogit-cli.exe。两个 EXE 的当前身份及完整构建结果以 release.json 和 VALIDATION.json 为准。

本地端到端测试覆盖离线帮助/schema、无 TTY 参数失败、单 JSON 成败通道、分页与字段筛选、只读预演、HEAD 前置条件、独立可执行文件、配置生成/重复创建/导入、Git 推送/克隆/拉取、模拟授权拒绝，以及大量子进程输出不泄露到协议。

超时测试使用实际 Windows Job Object：让 Git/SSH 测试后代等待后尝试写入标记，CLI 超时后终止整个执行范围，随后确认标记没有生成、远程没有更新。该测试验证本地进程终止，不证明远程服务器回滚。

构建在全新目录进行，两个发布 EXE 从候选 ZIP 解压后参与集成测试，最终 ZIP 再验证哈希和文件内容。历史 0.1.0、0.2.0 实测仅对应各自 EXE 哈希，不自动覆盖 0.3.0。

## 真实 GitLab 协议实测

2026-10-29，使用最终发布的 repogit-cli.exe 在真实私有测试项目完成以下流程。精确 Agent EXE 哈希和脱敏摘要见 [AGENT_VALIDATION.json](AGENT_VALIDATION.json)。这是针对 Agent EXE 的单独证据，不代表两个 EXE 都已逐一做完真实服务验收。

**发送端与接收端为同一台 Windows 电脑的两套隔离工作副本，使用已明确授权的测试密钥，不是两台物理电脑。** 原始服务地址、仓库路径、本机路径、密钥和原始日志均未发布。

| 场景 | 结果 |
| --- | --- |
| schema、status、clone --dry-run | JSON 可直接解析；预演明确 planned、remote_verified=false |
| clone、绑定已有测试身份 | 成功，无交互输入 |
| 提交前置条件不满足 | PRECONDITION_FAILED，退出 2、not_started |
| push、远程 SHA 核对 | 成功，remote_verified=true 且远程提交匹配 HEAD |
| 工作区含未提交内容 | 提示未上传；另一工作副本确认未取得该文件 |
| 脏工作区 pull | DIRTY_WORKTREE，退出 2 |
| 发送端更新、接收端 pull | 成功快进到同一提交 |
| doctor | 成功，只声明 read_access_only，不宣称写权限 |
| 导入已撤权的旧格式授权文件后 doctor | ACCESS_DENIED，退出 4 |
| 导入错误主机公钥配置后 doctor | HOST_KEY_REJECTED，退出 4，未关闭主机验证 |
| 输出通道与退出状态 | 每次 stdout 为单个 JSON，默认 stderr 为空，进程退出码与信封一致 |

27 项程序测试、12 项打包检查和 20 项最终 CLI 检查的精确构建记录见外部 VALIDATION.json。真实服务实测是在构建测试后针对同一 Agent EXE 执行，随后追加文档和重打 ZIP，没有重编译 EXE。

真实双机、严格分支/标签保护、LFS、子模块及客户端超时后的远端回滚不在本轮范围内。超时只能终止本地执行范围，不能承诺远程已发生的写入回滚。
