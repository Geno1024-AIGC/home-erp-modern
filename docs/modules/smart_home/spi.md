# smart_home · Profile / Capability / Driver SPI

> 本文定义 `smart_home` 的扩展面。模块边界见 [../smart_home.md](../smart_home.md)。

## 1. 分层

```text
DeviceProfile   设备类型：Capability 的组合 + UI 提示，数据驱动
Capability      能力：状态 schema、命令、参数 schema、风险等级
ProtocolDriver  协议：发现、配对、连接、快照、命令、事件
CommandDispatcher 统一调度：校验、策略、路由、状态、审计
```

扩展原则：

- **设备类型**优先加 Profile，不写 Kotlin 类。
- **操作**优先加 Capability 命令，不新增 RPC。
- **协议**加 Driver 模块，不改核心业务。
- **UI**用通用能力渲染器，不写厂商分支。
- 红外码、设备型号、按钮布局是数据；协议实现是代码。

## 2. DeviceProfile

```json
{
  "id": "climate.ir_ac",
  "version": 1,
  "category": "climate",
  "name": "红外空调",
  "capabilities": [
    "power",
    "target_temperature",
    "hvac_mode",
    "fan_speed"
  ],
  "uiHints": {
    "icon": "ac",
    "primaryCapability": "target_temperature",
    "order": 10
  }
}
```

规则：

- `id` 全局唯一，建议 `<category>.<variant>`。
- `version` 单调递增；设备实例记住创建时的版本。
- `capabilities` 只引用注册表中的能力；未知能力拒绝加载。
- Profile 不包含厂商 token、IP、MAC、红外码；这些属于 Driver/绑定配置。
- Profile 可以声明能力是必需还是可选；配对后由 Driver 报告实际能力。
- 新设备类型如果只是已有能力的组合，**只加 Profile**。

## 3. Capability

```kotlin
data class CapabilityDescriptor(
    val id: String,                    // "target_temperature"
    val version: Int,
    val stateSchema: JsonSchema,
    val commands: List<CommandDescriptor>,
    val risk: Risk,                    // low / medium / high / critical
    val stateKind: StateKind,          // reported / inferred / command_only
    val localOnly: Boolean = false,
    val requiresReauth: Boolean = false,
)

data class CommandDescriptor(
    val action: String,                // "set"
    val argsSchema: JsonSchema,
    val resultSchema: JsonSchema? = null,
    val idempotent: Boolean = true,
)
```

建议的核心能力：

| Capability | 命令 | 说明 |
|---|---|---|
| `power` | `set` | 布尔开关 |
| `brightness` | `set` | 0–100 |
| `target_temperature` | `set` | 温度与单位 |
| `hvac_mode` | `set` | 制冷/制热/送风/除湿 |
| `fan_speed` | `set` | 自动/低/中/高 |
| `lock_state` | 只读 | 上锁/解锁/未知 |
| `battery` | 只读 | 电量 |
| `door_event` | 只读 | 开门/关门/异常 |
| `media_player` | `play` / `pause` / `set_volume` | 音频服务 |
| `sensor_value` | 只读 | 温度/湿度/人体等 |

规则：

- 命令名用语义动作，不用厂商名；`set_temperature` 不是 `gree_set_temp`。
- `risk` 决定策略：`critical` 必须原生、重新认证、审计；v1 不注册门锁 `unlock`。
- `stateKind=inferred` 表示无法确认真实状态，例如红外空调。
- 自定义能力必须命名空间化，例如 `vendor.example.filter_life`；核心只做通用展示，不进入内置自动化语义。
- 删除/改变命令语义必须升主版本；新增可选字段升次版本。

## 4. Driver SPI

```kotlin
interface SmartHomeDriver {
    val id: String                    // "ir.gree", "homekit", "xiaomi.miot"
    val apiVersion: Int

    suspend fun start(ctx: DriverContext)
    fun discover(): Flow<DiscoveryCandidate>
    suspend fun pair(candidate: DiscoveryCandidate): PairingSession
    suspend fun open(binding: DeviceBindingRef): DeviceSession
    suspend fun stop()
}

interface DeviceSession {
    val events: Flow<DeviceEvent>
    suspend fun snapshot(): List<StateValue>
    suspend fun execute(command: CapabilityCommand): CommandResult
    suspend fun close()
}
```

### 4.1 适配器与绑定

Driver 支持两种形态：

- **点对点**：一个 ESP32 红外节点、一个 BLE 设备。
- **网关型**：Home Assistant、Zigbee 网关、米家云；一个连接下挂多个设备。

因此核心使用 `AdapterInstance` + `DeviceBinding`：

```text
AdapterInstance
  id, driverId, name, configRef, status
      │
      └── DeviceBinding
            id, deviceId, adapterId, nativeId, priority, capabilities
```

- `nativeId` 由 Driver 定义：MAC、UUID、did、Matter node ID 等。
- `adapterId + nativeId` 在驱动内唯一。
- 一个设备可有多条绑定；v1 只启用最高优先级的主绑定。
- 驱动不得把 `nativeId` 或厂商字段写进核心设备表。

### 4.2 发现

```kotlin
data class DiscoveryCandidate(
    val driverId: String,
    val nativeId: String,
    val suggestedProfile: String?,
    val name: String?,
    val address: String?,
    val metadata: JsonObject,
    val alreadyBound: Boolean,
)
```

- 发现结果只是候选，不自动成为受管设备。
- 发现必须可取消、可超时、可限速；不做无边界端口扫描。
- 广播/组播发现优先于主动扫描；不要做全流量嗅探。
- 未知设备可以展示为“未识别”，但不能自动控制。

### 4.3 配对

```kotlin
sealed interface PairingStep {
    data class QrCode(val value: String) : PairingStep
    data class Pin(val length: Int, val hint: String?) : PairingStep
    data class SelectDevice(val candidates: List<DiscoveryCandidate>) : PairingStep
    data class Confirm(val message: String) : PairingStep
    data class Waiting(val message: String) : PairingStep
    data class Failed(val code: String, val message: String) : PairingStep
}

interface PairingSession {
    val steps: Flow<PairingStep>
    suspend fun submit(input: JsonObject)
    suspend fun cancel()
}
```

- UI 只渲染步骤，不写 HomeKit/BLE/Matter 配对逻辑。
- 配对会话有超时与取消；失败不留下半绑定。
- 配对成功后返回 `DeviceBinding` 与驱动实际支持的能力。
- 配对可能要求重新认证；由策略层在 `pairing.start` 前检查。

## 5. 命令调度

```text
request
  → schema 校验
  → 权限 + 风险策略（risk/localOnly/reauth）
  → 幂等检查
  → Driver.execute
  → 状态更新（reported / inferred / optimistic）
  → 命令审计 + 事件
```

规则：

- 所有命令走 `smart_home.commands.execute`；不按设备/动作新增 RPC。
- 命令状态：`pending`、`sent`、`acked`、`failed`、`timeout`、`cancelled`。
- 有回传的协议才允许 `acked`；红外只能是 `sent`。
- 命令带 `idempotencyKey`；同一 key 不重复执行副作用。
- 可选 `expectedStateVersion`；不匹配时拒绝或提示刷新。
- Driver 超时/异常不得阻塞调度器；按设备熔断，记录错误。
- 高风险命令在策略层拒绝，不依赖 UI 隐藏。

## 6. 状态模型

```kotlin
data class StateValue(
    val capability: String,
    val value: JsonElement,
    val source: StateSource,       // reported / inferred / optimistic
    val confidence: Double?,       // 0..1，可选
    val observedAt: Long,
    val version: Long,
)
```

- `reported`：设备明确上报，可信。
- `inferred`：红外/无回传协议，根据最后命令推断。
- `optimistic`：命令发出后的临时状态，等待上报校准。
- 命令结果不是状态；必须以 `DeviceState` 为准。
- 多绑定冲突时按绑定优先级选源；v1 不做复杂融合。
- 状态历史不是 v1 必选；先最后值 + 事件流。

## 7. 扩展矩阵

| 想新增 | 扩展点 | 需要改核心？ |
|---|---|---|
| 新设备类型，能力已存在 | 新 `DeviceProfile` | 不用 |
| 新格力空调型号 | 新红外码 + Profile 数据 | 不用 |
| 已有能力的新命令 | Capability 增加命令 + schema | 少量 |
| 全新通用能力 | 核心 Capability 注册表 | 要 |
| 厂商私有能力 | 命名空间化 Capability | 不用改核心，但自动化只做通用展示 |
| 新协议 | 新 `ProtocolDriver` 模块 | 不用 |
| 新网关/云 | 新 Driver 或已有 Driver 的新 AdapterInstance | 不用 |
| 新 UI 卡片 | 通用渲染器 + `uiHints` | 不用 |
| 厂商专用 UI | 可选 Screen 扩展 | 少数情况 |
| 新场景/联动 | 只组合 Capability 命令 | 不用 |

## 8. 版本与兼容

- `SmartHomeDriver.apiVersion` 与宿主 SPI 主版本匹配。
- `DeviceProfile.version`、`Capability.version` 独立演进。
- 新增可选字段/命令：次版本；删除/改语义：主版本 + 迁移。
- 驱动模块在 `manifest.json` 声明 `requires: [{ "id": "g.erp.module.smart_home", "range": "^1.0" }]`。
- 设备实例保存 `profile_version`；升级时按迁移表转换状态。
- 未知 JSON 字段保留，不因驱动升级丢数据。

## 9. 打包与安全

- `smart_home` 核心是业务模块；协议驱动是独立 `.hmod`，依赖 `smart_home`。
- 驱动是受信任代码；安装时展示发布者、权限、网络/蓝牙/串口需求。
- 驱动配置与密钥放加密配置/`secrets/`，`config_ref` 只存引用。
- 日志脱敏：token、配对码、门锁事件正文、音频内容不落日志。
- 驱动崩溃/超时按设备熔断；核心仍可查看离线状态。
- 高风险能力（门锁、车库、报警）要求原生、重新认证、审计；v1 不开放门锁控制。
- Web 只能只读和低风险控制；配对/驱动管理只走原生。

## 10. 契约测试

每个驱动至少覆盖：

- 发现返回稳定 `nativeId`，重复发现不重复建绑定。
- 配对可取消、可超时、失败不留半绑定。
- `snapshot()` 符合 Capability schema。
- 命令编码/解码符合参数 schema，错误参数被拒绝。
- 事件解析正确，未知事件不崩溃。
- 离线、超时、设备重启、重复命令。
- 驱动停用后设备离线，其他驱动不受影响。

## 11. 反模式

- `when (device.type)` 散落在核心、UI、自动化。
- 每个设备动作一个 RPC。
- 协议 enum + `when (protocol)`。
- 厂商字段进 `m_smart_home_device`。
- 红外码写死在 Kotlin。
- 自动化直接调 `driver.sendRaw()`。
- 把“命令已发送”当成“设备已执行”。
- 用通用 DSL 描述所有协议；只对码表/Profile 做数据化。
- 让驱动直接写核心表；驱动只通过 SPI 返回状态/事件。
