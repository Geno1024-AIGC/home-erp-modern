# Home ERP Modern

面向家庭场景的模块化 ERP。Kotlin 单体仓库，采用 **恒星 / 行星 / 卫星** 三层部署架构。

> 当前处于**设计阶段**，尚无可运行构建。设计基线见 [docs/](docs/)。

## 角色

| 角色 | 代号 | 职责 |
|---|---|---|
| 恒星 | **Star** | 家中服务器；唯一数据权威，承载 SQLite、业务模块、配对与拓扑 |
| 行星 | **Planet** | 公网无状态中继；只转发端到端加密流量，不解密业务内容 |
| 卫星 | **Satellite** | 桌面 / Android 客户端；在家直连恒星，在外经行星中继 |

## 设计原则

- **数据只在家**：恒星是唯一数据权威；行星不保存业务数据。
- **内网优先**：同一局域网直连恒星；直连不可用时自动回退到行星。
- **端到端加密**：恒星与卫星之间始终有一层独立于行星的加密会话。
- **模块化**：功能是模块；服务端业务和客户端 UI 都以**动态代码包**形式安装/卸载。
- **可迁移**：一个 `data/` 目录包含数据库与附件，拷贝即备份/迁移。
- **克制依赖**：只引入有明确收益的库；安全关键密码学交给 Tink，不手写。

## 仓库结构（规划）

```
build-logic/                 Gradle convention plugins
core/model/                  共享领域模型（KMP）
core/protocol/               帧协议、会话、RPC 编解码
core/star-api/               服务端模块 SPI
core/client-api/             客户端模块 SPI + Compose UI 约定
core/ui/                     Compose Multiplatform 设计系统
modules/<feature>/           功能模块（star + client + 打包）
star/                        恒星应用
planet/                      行星应用
satellite/shared/            卫星共享壳（KMP）
satellite/desktop/           桌面卫星
satellite/android/           Android 卫星
tools/hmod/                  模块打包、签名、校验 CLI
docs/                        设计文档
```

## 文档

| 文档 | 内容 |
|---|---|
| [01-architecture.md](docs/01-architecture.md) | 目标、角色、部署拓扑、连接策略、技术栈 |
| [02-protocol.md](docs/02-protocol.md) | 外层中继帧、流复用、内层加密会话、RPC |
| [03-modules.md](docs/03-modules.md) | 模块 SPI、`.hmod` 包格式、签名、动态加载生命周期 |
| [04-data.md](docs/04-data.md) | SQLite、迁移、附件、备份 |
| [05-security.md](docs/05-security.md) | 威胁模型、配对、密钥、授权、供应链 |
| [06-roadmap.md](docs/06-roadmap.md) | 里程碑、验收标准、风险与前置验证 |
| [07-decisions.md](docs/07-decisions.md) | 架构决策记录（ADR） |

## 构建（计划）

- JDK 25（JVM 侧）；Android 字节码 17，`minSdk 26` / `compileSdk 36`
- Gradle Wrapper 9.7.1 + 版本目录 + `build-logic`
- 依赖：Ktor（仅 WSS 传输）、kotlinx.*、Tink、sqlite-jdbc、Compose Multiplatform
- Android release 签名遵循上级 `RULES/KEYSTORE.md`；keystore、密码、别名一律不入库

## Git

提交遵循 `<括号><动作><括号> <摘要>`：大事件 `{}`、中事件 `[]`、小事件 `()`，例如：

```
{Add} project: Bootstrap the repository and design baseline.
[Add] protocol: Define the outer relay frame format.
(Typo) docs: Fix a broken link.
```

提交保持原子性；每个提交后推送到 `origin/master`。
