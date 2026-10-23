# RepoCargo

**面向 AI Agent 的 Git 项目大文件管理工具：大文件存放在仓库之外，资源版本与代码一起管理。**

Agent-friendly large-file management for Git projects. Keep large assets outside Git while tracking their versions alongside code.

## 为什么有 RepoCargo

RepoCargo 起源于一个实际问题：大文件不适合直接存入 Git，但代码所依赖的模型、数据、安装包和其他资源，仍然需要跟随项目版本被准确管理和恢复。

项目沿用的做法是：大文件保存在仓库之外，Git 保存资源配置、索引和版本标识。取得代码后，工具根据这些信息定位并恢复所需资源；发布新资源时，再将对应的配置信息和版本标识随代码一起保存。

RepoCargo 是原 gitbigfileftp 项目的新名称。新的重点是让 AI Agent 也能清楚地检查、发布和获取项目资源，理解操作的实际副作用，并准确判断执行结果。

## 项目定位

- **资源与代码版本对应**：通过配置、索引和版本标识连接 Git 项目与外部资源。
- **支持资源发布和恢复**：不同实现承担上传、下载、小文件打包和部署等已有能力。
- **面向自动化与 Agent**：以明确的命令、结构化结果和可验证的执行行为作为演进方向。
- **尊重现有兼容性**：以 Python 完整版为业务基准，其他版本按已经实现的能力验收。

## 现有实现

以下是原项目中的现有入口和角色，**尚未迁入本仓库**：

| 原入口 | 角色 |
| --- | --- |
| `gitbigfileftp.py` | Python 完整版，作为现有业务行为和协议梳理的基准 |
| `gitbigfileftp/` | Rust 精简版，保留已有上传、下载和旧参数入口 |
| `gitbigfileftpcli/` | 独立 Agent CLI，识别并转调已有后端，提供结构化调用入口 |
| `gitbigfilenet/` | 下载部署客户端，通过 HTTP/HTTPS/FTP 消费已发布资源，不提供上传 |
| `test/gitbigfileftp_protocol/` | 原协议测试、行为分类和版本验收资料 |

这些入口的能力并不完全相同。Agent CLI 是独立包装器，Net 下载器是单独的业务实现，不能仅凭共同品牌推断它们可以互相替换。

## 当前仓库状态

本仓库目前用于 RepoCargo 的项目初始化与迁移准备，包含项目介绍、Agent 使用方向和迁移约定。源代码、正式测试和原协议文档尚未迁入，也尚未在这里建立可运行的构建或发行流程。

本次品牌准备不会改变原项目的源码、程序名、参数、配置文件名或测试预期。`gitbigfilex.json` 等已有协议名称继续沿用；后续源码迁移、接口调整与发行分别实施和验证。

## 面向 Agent 的方向

已有独立 CLI 提供了自动化调用的起点。后续 RepoCargo 将围绕以下问题整理接口和验收依据：

1. 当前项目需要哪些资源，已经具备哪些资源？
2. 本次操作会下载、上传、覆盖或删除什么？
3. 选择了哪个后端，它实际支持哪些能力？
4. 操作完成了哪些步骤，哪些失败，文件是否经过验证？

这些是演进目标，不代表本仓库已经实现了新的统一命令或完整能力。接口说明见 [Agent 使用方向](docs/agent.md)。

## 文档

- [Agent 使用方向与结果语义](docs/agent.md)
- [源码迁移与兼容约定](docs/migration.md)

## English

RepoCargo manages large project assets outside Git while keeping their configuration, indexes, and version identifiers alongside code.

It started as gitbigfileftp, a practical tool for publishing and restoring resources that do not belong in Git history. Its next focus is Agent-friendly operation: clear commands, structured results, explicit side effects, and verifiable outcomes.

This repository currently contains project and migration documentation. The existing Python, Rust, CLI, and download-client implementations have not yet been imported.
