# RepoGit 成品使用入口

本目录是独立的 RepoGit Windows x64 发行包。先读 README.md、docs/QUICKSTART.md、docs/GITLAB_SETUP.md，按 SHA256.json 校验成品，再运行 .\repogit.exe --version / --help。不要套用父目录 RepoCargo 的 CLI、资源配置或发布协议。

仅包含成品、文档及第三方许可，不包含产品源码或构建工具。使用用户明确提供的项目和 SSH 地址；文档域名均是占位符。只共享设备公钥，不读取、复制或提交个人私钥。status 只读本地，doctor 会联网读取；bind 会修改项目和用户 SSH 配置，push/pull/clone 按用户授权操作。程序不会自动提交、强推或解决历史分叉。真实 GitLab 权限验收尚未完成，不能将本地测试改述为服务器验收通过。
