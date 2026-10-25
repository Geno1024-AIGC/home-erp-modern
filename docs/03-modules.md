# 03 · 模块与动态代码包

## 1. 模块是什么

一个模块同时提供**服务端能力**与**客户端 UI**：

| 产物 | 运行位置 | 形式 |
|---|---|---|
| 服务端模块 | 恒星 JVM | `star/module.jar` |
| 桌面客户端 UI | 桌面卫星 JVM | `desktop/module.jar` |
| Android 客户端 UI | Android 卫星 ART | `android/module.dex` |
| 资源 | 两端 | `assets/`（图标、图片、字体、i18n） |
| 清单 | 恒星校验 | `manifest.json` |
| 签名 | 恒星/卫星校验 | `signature.json` |

模块可以只提供服务端能力（无 UI），也可以只提供 UI（不推荐）。v1 要求每个业务模块两端齐全。

内置模块与第三方模块使用**同一套 SPI**；区别只是内置模块编译进宿主、默认启用。

模块 ID 采用反向域名：`g.erp.module.inventory`。

---

## 2. 服务端 SPI

```kotlin
package g.erp.api.star

interface StarModule {
    /** 依赖的模块与版本范围；宿主在安装前解析。 */
    val requires: List<ModuleRequirement> get() = emptyList()

    /** 本模块的迁移；宿主按依赖顺序、在 install 之前执行。 */
    fun migrations(): List<Migration> = emptyList()

    /** 注册 RPC、事件、定时任务等；只调用一次。 */
    fun install(ctx: StarContext)

    /** 迁移完成、所有依赖 install 之后调用。 */
    fun start(ctx: StarContext) {}

    /** 停用/卸载前调用；需释放线程与资源。 */
    fun stop() {}
}
```

`StarContext` 是模块能接触到的全部宿主能力：

```kotlin
interface StarContext {
    val moduleId: String
    val dataDir: Path              // 模块私有目录（非附件库）
    val db: Db                     // 共享 SQLite；迁移/查询
    val files: FileStore           // 内容寻址附件
    val rpc: RpcRegistry           // 注册 RPC
    val events: EventBus           // 发布/订阅
    val scheduler: Scheduler       // 周期任务
    val config: ModuleConfig       // 模块配置（加密存储敏感项）
    val identity: IdentityService  // 当前用户、设备、角色
    val audit: AuditLog
    val log: Logger
}
```

### 2.1 RPC 注册

```kotlin
ctx.rpc.unary(
    method = "inventory.items.list",
    permission = "inventory.read",
) { req ->
    val query = req.decode<ListItems>()
    RpcResult.ok(ctx.db.read { conn -> queryItems(conn, query) })
}
```

- 方法名全局唯一，建议 `<module>.<resource>.<action>`。
- `permission` 由宿主统一鉴权；模块不自己解析 token。
- 一元 RPC 之外提供 `upload` / `download` / `stream` 注册器。
- 请求带 `Identity`（用户、设备、角色）与 `deadline`。

### 2.2 迁移

- 每个迁移有单调递增的 `version` 与校验和。
- 宿主记录在 `schema_migrations(module_id, version, checksum, applied_at)`。
- 迁移在事务中执行；失败回滚并拒绝启动该模块。
- 只允许向前迁移；不支持自动降级。
- 模块卸载默认**保留**表；「彻底删除数据」是显式操作。

### 2.3 权限

- 模块在清单声明权限（如 `inventory.read`、`inventory.write`、`inventory.admin`）。
- 恒星在安装时展示并要求管理员批准。
- 角色到权限的映射由 `members`/`auth` 核心模块管理。
- 所有 RPC 默认拒绝；必须显式声明权限或 `public`。

### 2.4 事件与定时任务

- `ctx.events.publish(type, payload)` / `subscribe(type)`，用于模块解耦（如 `inventory.item.low`）。
- `ctx.scheduler.every(interval) { ... }`：宿主统一调度，模块停用时取消。
- 跨进程/跨设备事件通过内层 `EVENT` 流推送。

---

## 3. 客户端 SPI

```kotlin
package g.erp.api.client

interface SatelliteModule {
    /** 由宿主在加载后调用；注册 UI 贡献。 */
    fun contribute(registry: UiRegistry, ctx: SatelliteContext)
}

interface UiRegistry {
    fun nav(entry: NavEntry)
    fun settings(section: SettingsSection)
    fun dashboard(card: DashboardCard)
    fun deepLink(link: DeepLinkHandler)
}

class NavEntry(
    val id: String,
    val title: String,
    val icon: ImageBitmap?,     // 来自 assets
    val order: Int,
    val screen: @Composable () -> Unit,
)

interface SatelliteContext {
    val moduleId: String
    val rpc: RpcClient                 // 唯一网络入口
    val session: SessionInfo           // 当前恒星、连接方式、用户
    val assets: AssetReader            // 模块 assets
    val strings: Strings               // i18n
    val storage: ModuleStorage         // 模块私有键值/文件
    val capabilities: Capabilities     // 相机/扫码/文件/通知/分享
    val navigator: Navigator
    val log: Logger
}
```

- 模块 UI 是 **Compose 代码**，不是声明式 DSL。
- 模块只能用宿主的 `RpcClient` 访问恒星；宿主不向模块暴露 Ktor/HTTP 栈。
- 模块可以读取自己的 assets；不能读取其他模块或宿主私有文件。
- 模块不能注册 Android `Activity`/`Service`/`Receiver`/`Provider`，不能携带 Android 资源或原生库（v1）。
- 平台能力通过 `Capabilities` 由宿主实现：扫码、相机、文件选择、通知、分享、生物识别。

### 3.1 导航与贡献点

| 贡献点 | 用途 |
|---|---|
| `nav` | 侧边栏/主导航入口 |
| `settings` | 设置页分组 |
| `dashboard` | 首页卡片 |
| `deepLink` | `homerp://module/<id>/...` |

- 宿主负责返回栈、标题栏、深链解析、权限提示。
- 模块页面通过 `ctx.navigator` 打开其他页面。
- 模块设置持久化用 `ctx.storage`；服务端配置用 `ctx.rpc`。

### 3.2 模块本地存储

- `ctx.storage` 是模块命名空间下的键值 + 文件目录；升级保留，卸载时询问是否删除。
- 不用于业务数据；业务数据必须在恒星。

---

## 4. `.hmod` 包格式

`.hmod` 是 ZIP：

```
manifest.json
signature.json
star/module.jar
desktop/module.jar
android/module.dex
assets/icon.png
assets/strings/en.json
assets/strings/zh-CN.json
...
```

### 4.1 manifest.json

```json
{
  "format": 1,
  "id": "g.erp.module.inventory",
  "name": "家居物品",
  "version": "1.0.0",
  "api": { "star": 1, "client": 1 },
  "compose": "1.9.x",
  "requires": [
    { "id": "g.erp.module.members", "range": "^1.0" }
  ],
  "permissions": [
    { "id": "inventory.read", "title": "查看物品", "default": "member" },
    { "id": "inventory.write", "title": "编辑物品", "default": "member" }
  ],
  "entrypoints": {
    "star": "g.erp.module.inventory.star.InventoryStarModule",
    "desktop": "g.erp.module.inventory.client.InventoryClientModule",
    "android": "g.erp.module.inventory.client.InventoryClientModule"
  },
  "artifacts": {
    "star/module.jar": { "sha256": "…", "size": 12345 },
    "desktop/module.jar": { "sha256": "…", "size": 23456 },
    "android/module.dex": { "sha256": "…", "size": 34567 }
  },
  "assets": {
    "icon": "assets/icon.png",
    "strings": ["assets/strings/en.json", "assets/strings/zh-CN.json"]
  }
}
```

- `api.star` / `api.client` 是**主版本**；宿主只加载支持范围内的模块。
- `compose` 是编译时 Compose 版本；宿主校验兼容性，不匹配则拒绝加载。
- `requires` 只支持必需依赖（v1）；带环或缺失依赖 → 安装失败。
- 未列出的文件不参与安装。

### 4.2 签名

```
signature.json
{
  "alg": "Ed25519",
  "keyId": "a1b2c3d4e5f60708",
  "sig": "base64(raw 64-byte signature)"
}
```

签名内容（按顺序）：

```
SHA-256(
  manifest.json 的原始字节
  || sha256(star/module.jar)
  || sha256(desktop/module.jar)
  || sha256(android/module.dex)
)
```

- **不对 JSON 重新序列化**；签名后 manifest 字节不得改动。
- `keyId` = `SHA-256(公钥)` 前 8 字节十六进制。
- 恒星维护信任库：`data/trust/publishers/<keyId>.pub`（公钥 + 标签）。
- 只有受信任发布者签名的模块可安装；未签名模块只能在显式的 `developerMode` 下加载，且不得用于发布。

### 4.3 恒星对卫星的背书

卫星不一定配置发布者信任库。恒星在安装/启用后签发：

```
ModuleAttestation {
  homeId, moduleId, version,
  manifestSha256, artifactHashes,
  publishedBy, installedAt
}
```

用恒星 Ed25519 私钥签名，随模块包一起下发。卫星用配对时保存的恒星公钥验签，然后校验各产物哈希。

因此信任链是：**卫星信任恒星 → 恒星信任发布者 → 发布者签名模块**。

---

## 5. 构建与打包

### 5.1 Gradle

```kotlin
// modules/inventory/star/build.gradle.kts
plugins { id("g.erp.module.star") }

// modules/inventory/client/build.gradle.kts
plugins { id("g.erp.module.client") }   // KMP: androidTarget + jvm("desktop")
```

- `g.erp.module.star`：JVM 库，`compileOnly(project(":core:star-api"))`，产出 `module.jar`。
- `g.erp.module.client`：Compose Multiplatform，`compileOnly(project(":core:client-api"))` + `compileOnly(compose)`。
- `compileOnly` 是硬约束：宿主 API 与 Compose 不得打进模块。
- `g.erp.module.pack`：调用 `tools/hmod` 生成 `.hmod`、签名、写哈希。
- Android DEX：对 client 的 Android 产物跑 `d8 --min-api 26`，产出 `android/module.dex`。

### 5.2 模块依赖

- 模块之间**不共享代码依赖**；共享代码只能来自 `core:*` API。
- 需要复用逻辑时，把稳定接口放进 `core:client-api` / `core:star-api`，或升级宿主。
- 模块内部可以拆多个 Gradle 子项目，但最终打进同一个包。

### 5.3 CI 产物

- 每次构建产出：`star`/`planet` 分发包、桌面卫星包、Android APK、所有内置模块 `.hmod`。
- 模块版本独立（SemVer）；宿主版本独立。
- 产物带 SHA-256 清单；发布页附签名公钥指纹。

---

## 6. 动态加载

### 6.1 类加载器策略

宿主统一使用 **parent-first**：

| 包前缀 | 来源 |
|---|---|
| `g.erp.api.*` | 宿主（SPI） |
| `g.erp.ui.*` | 宿主（设计系统） |
| `kotlin.*`、`kotlinx.*` | 宿主 |
| `androidx.compose.*` / `org.jetbrains.compose.*` | 宿主 |
| 其他 | 模块自身（若打包） |

- 模块与宿主的 Compose / Kotlin 版本必须兼容；清单里声明 `compose`。
- 模块自带第三方库时，必须避免与宿主重复；Android 的父优先会静默使用宿主版本。
- 模块不得依赖宿主内部非 API 包（`g.erp.internal.*` 等）；不做强制隔离，靠 API 纪律与代码评审。

### 6.2 桌面（JVM）

```kotlin
val loader = URLClassLoader(arrayOf(jar.toUri().toURL()), hostClassLoader)
val module = loader.loadClass(entrypoint).getDeclaredConstructor().newInstance() as SatelliteModule
```

- 停用后 `loader.close()`；JVM 不保证卸载类，长期运行建议重启宿主。
- 升级：加载新 classloader，切换引用，旧 loader 丢弃。
- 卸载：移除 UI 与文件，进程重启后彻底释放。

### 6.3 Android

- 使用 `InMemoryDexClassLoader`（API 26+），从模块包读取 DEX 字节，避免可写目录执行代码的限制。
- 父加载器提供宿主 API、Kotlin、Compose、Tink。
- ART 不能卸载已加载的类；**升级/卸载需要重启应用进程**。UI 入口可以立即移除。
- 模块 DEX 用 `d8 --min-api 26` 生成；不得包含 Android 资源、manifest 组件或原生库。
- 首次加载前校验恒星背书签名与产物哈希；模块包存内部存储，不读外部存储。
- **发布政策**：动态代码加载在 Google Play 上受限；本项目通过 GitHub/自建渠道分发。若未来上架 Play，需改用 Play Feature Delivery 或重新评估。

### 6.4 生命周期

**恒星**

```
upload/verify → resolve deps → load classloader → migrations → install → start
```

- 启用/停用：注册/注销 RPC 与事件；停用依赖者时级联停用。
- 升级：停 → 迁移 → 换 classloader → 启动；大版本建议重启恒星。
- 卸载：停用 → 删除包；数据库保留，除非显式清理。
- 失败：回滚到上一版本；保留日志与失败原因。

**卫星**

```
download → verify attestation + hashes → store → load → contribute UI
```

- 启用/停用：即时增删导航/设置/卡片。
- 升级：Android 提示重启；桌面切换 classloader。
- 卸载：删除包与本地存储；提示是否保留模块数据。

### 6.5 依赖解析

- 按 `requires` 建图，拓扑排序；缺失或版本不满足 → 拒绝安装。
- 检测环 → 拒绝。
- 停用某模块时，所有传递依赖它的模块一起停用。
- 版本范围用 SemVer：`^1.0`、`~1.2`、`1.2.3`；v1 不实现复杂的多版本共存。

---

## 7. 版本兼容

- `core:star-api` / `core:client-api` / `core:ui` 是唯一稳定面；用 `api` 主版本号标识。
- 宿主支持 `[current-1, current]` 的模块 API 主版本（至少一个过渡版本）。
- 新增可选方法用默认实现；删除/改签名必须升主版本。
- Compose 版本升级视为客户端 API 的一部分；模块需重新编译。
- 协议版本与模块 API 版本独立演进。

---

## 8. 示例：inventory 模块骨架

```
modules/inventory/
  manifest.json
  star/
    build.gradle.kts
    src/main/kotlin/g/erp/module/inventory/star/
      InventoryStarModule.kt
      Items.kt
      Migrations.kt
  client/
    build.gradle.kts
    src/commonMain/kotlin/g/erp/module/inventory/client/
      InventoryClientModule.kt
      ItemListScreen.kt
      ItemEditScreen.kt
    src/androidMain/kotlin/...   // 平台能力适配（如有）
    src/desktopMain/kotlin/...
  assets/
    icon.png
    strings/en.json
    strings/zh-CN.json
```

服务端：

```kotlin
class InventoryStarModule : StarModule {
    override fun migrations() = InventoryMigrations.all

    override fun install(ctx: StarContext) {
        ctx.rpc.unary("inventory.items.list", "inventory.read") { req ->
            RpcResult.ok(ctx.db.read { listItems(it) })
        }
        ctx.rpc.unary("inventory.items.save", "inventory.write") { req ->
            RpcResult.ok(ctx.db.write { saveItem(it, req.decode<Item>()) })
        }
    }
}
```

客户端：

```kotlin
class InventoryClientModule : SatelliteModule {
    override fun contribute(registry: UiRegistry, ctx: SatelliteContext) {
        registry.nav(
            NavEntry(
                id = "inventory",
                title = ctx.strings["inventory.title"],
                icon = ctx.assets.image("icon.png"),
                order = 10,
                screen = { ItemListScreen(ctx) },
            )
        )
    }
}
```

---

## 9. 限制（v1）

- 模块是受信任代码；没有 JVM/ART 级沙箱。
- 不支持原生库、Android 资源、manifest 组件、后台常驻服务。
- 不支持模块间代码依赖；只允许声明的模块依赖与 RPC/事件。
- Android 卸载/升级需要重启进程才能释放代码。
- 不支持同一模块多版本并存。
- 不承诺热升级不中断；升级期间相关 RPC 可能短暂不可用。
