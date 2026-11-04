# 模块：智能家居（smart_home）

> 状态：draft ｜ 类型：business ｜ 依赖：`locations`（可选）、`members`（可选）、`inventory`（可选）、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_smart_home_*`

## 1. 目的与非目标

**目的**：把可控家居设备、协议适配、状态、命令、场景与简单联动收进一个统一模块；新增设备类型、操作或协议时，不修改核心业务代码。

**非目标**：

- 不做远程开锁、门锁用户管理、临时密码、胁迫报警等安全关键操作。
- v1 不接摄像头、麦克风、可视对讲；不存视频/音频流。
- 不实现 Matter/Zigbee/Z-Wave/BLE 等完整协议栈；协议由驱动提供。
- 不拥有物理资产、保修、发票；这些归 `inventory` / `documents`。
- 不做多房间同步音频；蓝牙音频是独立音频服务，不塞进命令总线。
- 不保证关键设备在恒星离线时仍可用；灯、门锁必须保留物理/本地控制。
- 不做云端账号默认接入；需要云的设备必须显式配置并标注依赖。

## 2. 用户场景

- 每个房间一个红外节点，控制格力空调：开关、温度、模式、风速。
- 门锁只读：开/关状态、电量、开门事件；不做远程开锁。
- 小爱音箱作为蓝牙音频输出/通知设备。
- 发现局域网设备，确认后加入家庭；未确认设备只显示为候选。
- 新增一种灯或传感器：组合已有能力，不写新代码。
- 新增一个协议：安装驱动，不改 `smart_home` 核心。
- 手动场景“回家”“睡眠”；简单联动“门锁解锁且在家 → 开空调”。
- 查看设备在线状态、命令结果、失败原因。

## 3. 边界与依赖

- 拥有设备注册、设备档案、绑定、状态、命令、事件、场景、简单联动。
- 不拥有物理资产：`inventory` 记录型号、序列号、保修；可选的 `inventory_item_id` 只做逻辑关联。
- 房间/位置只引用 `location_id`，不建跨模块外键。
- 说明书、配网二维码、发票归 `documents`。
- 换滤芯、清洗空调等人类待办归 `tasks`；`smart_home` 不复制任务状态。
- 日历事件归 `calendar`；“每天 22:00 关空调”是设备联动，不是日历事件。
- 门锁 v1 只注册只读能力 `lock_state`、`battery`、`door_event`；不注册 `unlock`。
- 蓝牙音频由宿主音频服务处理；`smart_home` 只保存媒体设备状态与调用入口。
- 发现到的设备不等于受信任设备；必须经过确认/配对。
- 高风险命令（门锁、车库、报警器）由策略层拦截；不依赖 UI 隐藏。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `DeviceProfile` | `id`、`version`、`category`、`capabilities`、`ui_hints` |
| `Capability` | `id`、`version`、`state_schema`、`commands`、`risk`、`state_kind` |
| `AdapterInstance` | `id`、`driver_id`、`name`、`config_ref`、`status` |
| `Device` | `id`、`profile_id`、`profile_version`、`name`、`category`、`location_id`、`status`、`archived_at` |
| `DeviceBinding` | `id`、`device_id`、`adapter_id`、`native_id`、`priority`、`capabilities_json` |
| `DeviceState` | `device_id`、`capability`、`state_json`、`source`、`confidence`、`observed_at`、`version` |
| `Command` | `id`、`device_id`、`capability`、`action`、`args_json`、`status`、`actor_id`、`idempotency_key` |
| `DeviceEvent` | `id`、`device_id`、`type`、`payload_json`、`at` |
| `Scene` | `id`、`name`、`actions_json`、`archived_at` |
| `Automation` | `id`、`name`、`enabled`、`trigger_json`、`conditions_json`、`actions_json` |

`category` 只用于 UI 分组（`light`、`climate`、`lock`、`media`、`sensor`、`switch`、`other`），不决定行为。行为来自 `Capability`。

v1 参考 Profile：`climate.ir_ac`、`lock.readonly`、`media_player.speaker`、`sensor.basic`。v1 参考 Driver：`ir.gree`、`homekit`、`xiaomi.miot`（可选）、`esp32.ir`。

规则：

- 设备类型是 Profile，不是 Kotlin 类；新类型优先用已有能力组合。
- 操作是 Capability 的命令，不新增 `device.turnOn` 这类 RPC。
- 协议是 Driver；核心不出现厂商分支、协议 enum 或厂商字段。
- 一个设备可有一条主绑定和零条备用绑定；Home Assistant/网关型驱动可以作为多个设备的适配器实例。
- `state.source`：`reported`（设备上报）、`inferred`（红外等推断）、`optimistic`（命令后临时状态）。
- 命令结果不等于真实状态；以 `DeviceState` 为准。
- 红外没有回传，状态必须标记 `inferred`；UI 显示“最后发送”。
- 未知设备只在发现列表出现；用户确认后才创建设备与绑定。
- Profile / Capability / Driver SPI 详见 [smart_home/spi.md](smart_home/spi.md)。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `smart_home.read` | 查看设备、状态、场景 | `member` |
| `smart_home.control` | 执行低风险命令、运行场景 | `member` |
| `smart_home.secure` | 执行高风险命令（门锁/车库/报警器） | `admin` + 原生 + 重新认证 |
| `smart_home.manage` | 配对、删除设备、管理驱动与联动 | `admin` |
| `smart_home.driver.admin` | 安装/配置协议驱动、查看驱动日志 | `admin` |

- v1 不提供门锁 `unlock`，因此 `secure` 只占位，不开放实际命令。
- 策略层校验：能力风险、命令目标、来源（LAN/行星/Web）、设备类型、是否重新认证。
- Web 只允许只读和低风险控制；配对、驱动管理、高风险命令只走原生。
- Driver 是受信任代码；安装前展示发布者、权限与网络/蓝牙需求。

## 6. RPC 与事件

- `smart_home.devices.list` / `get` / `create` / `update` / `archive` / `restore`
- `smart_home.devices.bindings.list`
- `smart_home.discovery.start` / `stop` / `candidates.list`
- `smart_home.pairing.start` / `submit` / `cancel`
- `smart_home.commands.execute`：统一命令入口，参数为 `{ deviceId, capability, action, args, idempotencyKey? }`。
- `smart_home.commands.get`
- `smart_home.scenes.list` / `create` / `update` / `execute` / `archive`
- `smart_home.automations.list` / `create` / `update` / `enable` / `disable`
- `smart_home.drivers.list` / `status`
- 事件：
  - `smart_home.device.discovered`
  - `smart_home.device.changed`
  - `smart_home.device.state.changed`
  - `smart_home.device.event`
  - `smart_home.command.changed`
  - `smart_home.pairing.step`

禁止：

- 不提供 `smart_home.devices.rawCommand` 或 `driver.sendRaw` 给 UI/自动化。
- 自动化只能引用 Capability/Command，不直接调用厂商 SDK。
- 命令必须带 `idempotencyKey`；同一个 key 不重复执行副作用。

## 7. UI（原生 / Web）

- 原生：房间/设备列表、通用能力卡片、发现与配对、驱动状态、场景/联动编辑、命令历史。
- Web：固定壳中的只读仪表盘和低风险控制；不加载模块 UI，不做配对。
- 通用渲染器：开关、滑杆、温度、模式、媒体控制、只读状态；Profile 只提供图标、顺序、文案。
- 不在 UI 写 `when (device.type)` 或厂商专用页面；特殊 UI 只作为可选扩展，不做默认路径。
- 门锁卡片只显示状态、电量、事件；不显示开锁按钮。
- 空调卡片提示“红外推断状态”；失败时显示“已发送但未确认”。
- 音箱卡片显示媒体状态/音量；音频播放由宿主音频服务执行。

## 8. 数据与迁移

```sql
CREATE TABLE m_smart_home_adapter (
  id          TEXT PRIMARY KEY,
  driver_id   TEXT NOT NULL,
  name        TEXT NOT NULL,
  config_ref  TEXT,
  status      TEXT NOT NULL DEFAULT 'unknown',
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);

CREATE TABLE m_smart_home_device (
  id              TEXT PRIMARY KEY,
  profile_id      TEXT NOT NULL,
  profile_version INTEGER NOT NULL,
  name            TEXT NOT NULL,
  category        TEXT NOT NULL,
  location_id     TEXT,
  inventory_item_id TEXT,
  status          TEXT NOT NULL DEFAULT 'unknown',
  metadata_json   TEXT,
  archived_at     INTEGER,
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);
CREATE INDEX idx_smart_home_device_loc ON m_smart_home_device(location_id, archived_at);

CREATE TABLE m_smart_home_binding (
  id                TEXT PRIMARY KEY,
  device_id         TEXT NOT NULL,
  adapter_id        TEXT NOT NULL,
  native_id         TEXT NOT NULL,
  priority          INTEGER NOT NULL DEFAULT 0,
  capabilities_json TEXT,
  created_at        INTEGER NOT NULL
);
CREATE UNIQUE INDEX idx_smart_home_binding_native
  ON m_smart_home_binding(adapter_id, native_id);
CREATE INDEX idx_smart_home_binding_device ON m_smart_home_binding(device_id, priority);

CREATE TABLE m_smart_home_state (
  device_id   TEXT NOT NULL,
  capability  TEXT NOT NULL,
  state_json  TEXT NOT NULL,
  source      TEXT NOT NULL,
  confidence  REAL,
  observed_at INTEGER NOT NULL,
  version     INTEGER NOT NULL DEFAULT 1,
  PRIMARY KEY (device_id, capability)
);

CREATE TABLE m_smart_home_command (
  id              TEXT PRIMARY KEY,
  device_id       TEXT NOT NULL,
  capability      TEXT NOT NULL,
  action          TEXT NOT NULL,
  args_json       TEXT NOT NULL,
  status          TEXT NOT NULL,
  actor_id        TEXT NOT NULL,
  idempotency_key TEXT,
  error           TEXT,
  requested_at    INTEGER NOT NULL,
  completed_at    INTEGER
);
CREATE UNIQUE INDEX idx_smart_home_command_idem
  ON m_smart_home_command(device_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;

CREATE TABLE m_smart_home_event (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  device_id  TEXT NOT NULL,
  type       TEXT NOT NULL,
  payload_json TEXT,
  at         INTEGER NOT NULL
);
CREATE INDEX idx_smart_home_event_device ON m_smart_home_event(device_id, at);

CREATE TABLE m_smart_home_scene (
  id           TEXT PRIMARY KEY,
  name         TEXT NOT NULL,
  actions_json TEXT NOT NULL,
  archived_at  INTEGER,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);

CREATE TABLE m_smart_home_automation (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  enabled         INTEGER NOT NULL DEFAULT 1,
  trigger_json    TEXT NOT NULL,
  conditions_json TEXT,
  actions_json    TEXT NOT NULL,
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);
```

- Profile 与 Capability 默认来自模块 assets/内置 JSON；v1 不做用户自定义 Profile。
- `config_ref` 指向加密配置/密钥库，不在表里存 token、配对密钥、设备私钥。
- `state` 只保留最后已知值；历史事件按保留策略清理。
- 命令保留用于审计/重试；默认 90 天，管理员可配置。
- 不存视频、音频流、摄像头截图。
- 迁移只向前；删除设备默认归档，绑定与历史保留。

## 9. 验收

- 新增一个已有能力组合的设备类型，只加 Profile 即可显示与控制。
- 新增格力空调型号，只换红外码/Profile，不改核心。
- 新增一个协议驱动，不修改 `smart_home` 的命令与 UI 代码。
- 发现设备→确认→配对→设备出现；未确认设备不进入受管列表。
- 门锁只显示状态/电量/事件，不存在 `unlock` 命令。
- 红外命令发送成功但状态标记为 `inferred`；失败/超时有明确错误。
- 场景与简单联动只引用 Capability；禁用驱动后相关设备离线，其他设备不受影响。
- Web 只能读和低风险控制；配对/驱动管理/高风险命令被拒绝。
- 驱动崩溃或超时不会拖垮 `smart_home` 主进程；命令有超时与熔断。
- 日志不含 token、配对密钥、门锁事件正文。

## 10. 开放问题

- v1 是否开放“用户自定义 Profile”还是只允许模块内置 Profile。
- 简单联动是否足够，还是需要完整规则引擎/脚本；当前先做单触发器 + 条件 + 动作。
- 设备状态是否保留历史曲线；v1 先最后值 + 事件。
- 多绑定状态冲突的合并策略：主绑定优先，还是按能力分别选源。
- 蓝牙音频是否由宿主统一 `AudioService` 提供，还是独立模块。
- 驱动是否允许携带原生库；BLE/串口驱动可能需要。
- 是否支持离线本地控制与恒星故障降级。
- 摄像头/可视门铃是否永久排除，还是后续单独高敏模块。
