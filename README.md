# Home ERP Modern

面向家庭场景的模块化 ERP。Kotlin 单体仓库 + 原生 TypeScript Web 壳，采用 **恒星 / 行星 / 卫星** 三层部署架构。

> 当前处于**设计阶段**，尚无可运行构建。设计基线见 [docs/](docs/)。

## 角色

| 角色 | 代号 | 职责 |
|---|---|---|
| 恒星 | **Star** | 家中服务器；唯一数据权威，承载 SQLite、业务模块、配对与拓扑 |
| 行星 | **Planet** | 公网服务器；原生卫星走 WSS 中继（零知识），Web 走 HTTPS 网关（可见 Web 明文） |
| 卫星（原生） | **Native Satellite** | 桌面 / Android 客户端；在家直连恒星，在外经行星中继；端到端加密 |
| 卫星（Web） | **Web Satellite** | 浏览器客户端；固定 Web 壳，经行星 HTTPS 网关访问恒星 |

## 设计原则

- **数据只在家**：恒星是唯一数据权威；行星不持久化业务数据。
- **内网优先**：原生卫星在局域网直连恒星；不可用时自动回退到行星。
- **原生端到端加密**：桌面 / Android 与恒星之间有不依赖行星的加密会话。
- **Web 是受信任网关模式**：浏览器经行星 HTTPS 网关访问，行星会看到 Web 明文与登录会话；这是明确的取舍，不用 Web 承载高敏感操作。
- **模块化**：功能是模块；服务端业务和原生客户端 UI 以**动态代码包**安装/卸载。
- **Web 固定壳**：Web 端 v1 不加载动态模块，只提供内置页面。
- **可迁移**：一个 `data/` 目录包含数据库与附件，拷贝即备份/迁移。
- **克制依赖**：只引入有明确收益的库；安全关键密码学交给 Tink，Web 端零运行时依赖。

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
satellite/shared/            原生卫星共享壳（KMP）
satellite/desktop/           桌面卫星
satellite/android/           Android 卫星
satellite/web/               Web 卫星（原生 TS/HTML/CSS，无框架）
tools/hmod/                  模块打包、签名、校验 CLI
docs/                        设计文档
```

## 文档

| 文档 | 内容 |
|---|---|
| [01-architecture.md](docs/01-architecture.md) | 目标、角色、部署拓扑、连接策略、技术栈 |
| [02-protocol.md](docs/02-protocol.md) | 外层中继帧、流复用、原生内层加密、Web HTTPS 网关、RPC |
| [03-modules.md](docs/03-modules.md) | 模块 SPI、`.hmod` 包格式、签名、原生动态加载、Web 固定壳 |
| [04-data.md](docs/04-data.md) | SQLite、迁移、附件、备份、Web 会话 |
| [05-security.md](docs/05-security.md) | 原生/Web 双路径威胁模型、配对、密钥、授权、供应链 |
| [06-roadmap.md](docs/06-roadmap.md) | 里程碑、验收标准、风险与前置验证 |
| [07-decisions.md](docs/07-decisions.md) | 架构决策记录（ADR） |

## 构建（计划）

- JDK 25（JVM 侧）；Android 字节码 17，`minSdk 26` / `compileSdk 36`
- Gradle Wrapper 9.7.1 + 版本目录 + `build-logic`
- 依赖：Ktor（WSS 传输 + 行星 HTTPS 网关）、kotlinx.*、Tink、sqlite-jdbc、Compose Multiplatform
- Web：原生 TypeScript（`tsc` 编译到 ES modules）+ HTML/CSS，零运行时依赖、无框架
- Android release 签名遵循上级 `RULES/KEYSTORE.md`；keystore、密码、别名一律不入库

## Git

提交遵循 `<括号><动作><括号> <摘要>`：大事件 `{}`、中事件 `[]`、小事件 `()`，例如：

```
{Add} project: Bootstrap the repository and design baseline.
[Add] protocol: Define the outer relay frame format.
(Typo) docs: Fix a broken link.
```

提交保持原子性；每个提交后推送到 `origin/master`。
