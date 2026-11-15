# 模块：衣物与穿搭（wardrobe）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`locations`（可选）、`documents`（可选）、`tasks`（可选）、`smart_home`（可选） ｜ 拥有：`m_wardrobe_*`

## 1. 目的与非目标

**目的**：管理家庭成员的衣物与鞋包，记录每件的洗护要求与当前状态；保存可复用的搭配；按当天天气、体感温区、季节与场合抽取一套**实际可穿**的搭配；在一次洗涤开始前检查这桶衣服的洗护冲突。

**非目标**：

- 不做 AI 穿搭推荐、生成搭配图、虚拟试衣、身材评分。
- 不记录身体尺寸与体型；身体数据归 `members` / `health`。
- 不存人像照片；衣物照片只允许平铺照。
- 不做二手估值、转卖、捐赠与干洗店对接。
- 不做吊牌 OCR、面料成分自动识别。
- 不拥有洗衣机设备；设备状态与命令归 `smart_home`。
- 不做天气长期历史与气候趋势分析。
- 不用精确位置；天气只用城市/区级粗位置。

## 2. 用户场景

- 新建一件大衣：类别、颜色、尺码、洗护符号、放在哪个衣柜。
- 出门前打开「今天穿什么」，系统按今日天气抽一套。
- 抽到的那套里有衣物在洗，系统自动排除或明确告知。
- 存三套常用搭配：「上班」「周末」「见客」。
- 洗衣服前把这桶扫进洗涤批次，系统提示「2 件不能 60 °C」。
- 洗衣机洗完，恒星提醒去晾；晾完手动确认，衣物回到 `clean`。
- 换季收纳：夏季衣物标 `stored`，位置改到收纳箱。
- 看「最近 90 天没穿过的」。
- 借给朋友：标 `lent`，记借给谁。

## 3. 边界与依赖

- 拥有衣物、洗护画像、搭配、穿着记录、洗涤批次与天气快照。
- 单件可追踪的衣物归 `wardrobe`；成打/未拆封的消耗品（袜子、内衣、毛巾）归 `inventory`。
- 衣柜、抽屉、收纳箱只引用 `location_id`，不建跨模块外键。
- 衣物照片、吊牌、小票归 `documents`；`wardrobe` 只存 `photo_doc_id`。
- 洗衣机设备、`cycle_state`、漏水、门磁归 `smart_home`；`wardrobe` 只订阅完成事件。
- 「该洗了」「洗完了去晾」由 `tasks` 建提醒；`wardrobe` 只发事件，不改任务状态。
- 身体尺寸归 `members`；衣物尺码（`M`、`40`、`XL`）是衣物属性，不做跨品牌换算。
- 天气：v1 由 `wardrobe` 拥有快照与 `WeatherProvider` SPI；多个模块都要天气时再提升为独立 `weather` 模块。
- 衣物与搭配默认家庭成员可见（便于代洗、代收拾）；穿着记录与抽卡记录默认仅本人可见。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Garment` | `id`、`member_id`、`name`、`category`、`color`、`size`、`brand`、`purchased_at`、`price_cents`、`care_profile_id`、`location_id`、`status`、`photo_doc_id`、`note`、`archived_at` |
| `CareProfile` | `id`、`name`、`wash_method`、`wash_temp_max_c`、`bleach`、`tumble_dry`、`iron`、`dry_clean`、`note` |
| `Outfit` | `id`、`member_id`、`name`、`seasons`、`formality`、`comfort_min_c`、`comfort_max_c`、`rain_ok`、`wind_ok`、`note`、`archived_at` |
| `WearLog` | `id`、`member_id`、`garment_id`、`outfit_id`、`local_date`、`note` |
| `LaundryLoad` | `id`、`member_id`、`program`、`wash_temp_c`、`tumble_dry`、`bleach`、`dry_clean`、`status`、`device_id`、`started_at`、`finished_at` |
| `Draw` | `id`、`member_id`、`local_date`、`outfit_id`、`attempt`、`seed` |
| `WeatherSnapshot` | `local_date`、`location_label`、`temp_c`、`feels_like_c`、`humidity`、`precipitation_mm`、`wind_kph`、`condition`、`source`、`fetched_at` |

枚举：

- `Garment.status`：`clean`、`worn`、`dirty`、`washing`、`drying`、`ironing`、`stored`、`lent`、`repair`、`discarded`。
- `category`：`top`、`bottom`、`outer`、`shoes`、`bag`、`accessory`、`underwear`、`socks`、`sport`、`formal`、`other`。
- `wash_method`：`machine`、`hand`、`do_not_wash`。
- `bleach`：`none`、`oxygen`、`chlorine`。
- `tumble_dry`：`allowed`、`low_heat`、`do_not`。
- `iron`：`none`、`low`、`medium`、`high`、`do_not`。
- `dry_clean`：`allowed`、`do_not`。
- `formality`：`casual`、`work`、`formal`、`sport`、`sleep`。
- `seasons`：`spring`、`summer`、`autumn`、`winter`；空数组表示四季可穿。

规则：

- 洗护用 ISO 3758 对应的枚举表达，不写自由文本；`note` 只补充符号表达不了的内容。
- `wash_temp_max_c` 取 30 / 40 / 60 / 95；`wash_method=do_not_wash` 表示只能干洗或擦拭，温度为空。
- 衣物状态只由用户操作推进；机器事件只触发提醒，不自动改状态。
- 归档不删除；穿着记录与洗涤历史保留。
- `wear_count` 与 `last_worn_on` 由 `WearLog` 派生，可以缓存，但以日志为准。
- 一条衣物属于一个成员；家庭共用物品（雨伞、备用正装）放 `inventory`。
- `comfort_min_c` / `comfort_max_c` 描述「这套搭配适合的室外体感温度」，不是衣物耐温。
- 搭配只引用本成员衣物；跨成员共用搭配 v1 不做。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `wardrobe.read` | 查看衣物、洗护画像、搭配、洗涤批次 | `member` |
| `wardrobe.write` | 新建/编辑/归档本人衣物与搭配 | `member` |
| `wardrobe.laundry` | 建洗涤批次、跑冲突检查、推进批次状态 | `member` |
| `wardrobe.draw` | 抽取搭配、写穿着记录 | `member` |
| `wardrobe.manage` | 管理洗护画像模板与家庭级类别 | `admin` |
| `wardrobe.proxy` | 代管未成年成员的衣物 | `admin` 或获授权家长 |

- 衣物、洗护画像、搭配：家庭成员可见（需要知道柜子里有什么以便代洗、代收拾）。
- 穿着记录与抽卡记录：仅本人可见；管理员也没有读取入口，没有破窗。
- 洗涤批次：创建者与该批次内衣物所有者可见；加入他人衣物时只显示名称与洗护画像，不显示价格与穿着记录。
- `guest` 无权限。
- 家庭可见不等于公开：不提供家庭外分享，导出按高敏感操作处理。

## 6. RPC 与事件

- `wardrobe.garments.list` / `get` / `create` / `update` / `archive` / `restore`
- `wardrobe.care_profiles.list` / `create` / `update`
- `wardrobe.outfits.list` / `get` / `create` / `update` / `archive`
- `wardrobe.outfits.draw`：抽卡，参数 `{ memberId, formality?, attempt? }`。
- `wardrobe.wear_log.create` / `list`
- `wardrobe.laundry.loads.create` / `add_items` / `remove_item` / `start` / `finish` / `cancel`
- `wardrobe.laundry.check`：输入衣物集合与计划程序，返回 `ok` / `warn` / `conflict` 与逐条原因。
- `wardrobe.weather.get` / `refresh` / `set_manual`
- 事件：
  - `wardrobe.garment.changed`
  - `wardrobe.outfit.changed`
  - `wardrobe.wear_log.created`
  - `wardrobe.laundry.changed`
  - `wardrobe.weather.updated`

禁止：

- 不提供控制洗衣机的 RPC；冲突检查只警告，不改程序、不调用 `smart_home` 命令。
- 不提供「自动标记穿着」；穿着记录必须由用户确认。
- 不提供身体尺寸、体型或人像字段。
- 抽卡不支持无限重抽，服务端按天计数。

## 7. 天气与抽卡

### 7.1 天气来源

- `WeatherProvider` SPI：`fetch(localDate, coarseLocation) -> WeatherSnapshot`。
- v1 内置两种：`manual`（用户手填今日天气，默认）与 `http.forecast`（城市级预报，显式开启）。
- 任一来源失败：用最近一次快照并标 `stale`，抽卡降级为只按季节与温区筛，不阻塞。
- 缓存：同一 `(local_date, location_label)` 每小时最多抓一次。
- 隐私：只存城市/区级 `location_label`；不存 GPS、不存家庭精确地址；日志不记录位置。
- 只保留最近 14 天快照，不做长期天气历史。

### 7.2 候选集（顺序固定）

1. 搭配未归档。
2. 搭配内所有衣物都可穿：排除 `dirty`、`washing`、`drying`、`ironing`、`repair`、`lent`、`discarded`。
3. 季节匹配：`seasons` 为空，或包含当前季节。
4. 温区匹配：今日 `feels_like_c` 落在 `[comfort_min_c, comfort_max_c]`；没有天气时改用当季推荐区间。
5. 天气约束：降水时要求 `rain_ok`；大风时要求 `wind_ok`。
6. 场合匹配：用户指定 `formality` 时只留匹配项。

### 7.3 权重

- 基础 1.0。
- 最近 7 天抽中过：×0.2。
- 最近 7 天穿过：×0.4。
- 30 天以上没抽到：×1.5。
- 场合完全匹配：×2.0。
- 保底：连续 5 次抽卡未出现的搭配，×3。

### 7.4 抽卡规则

- 同一成员同一天第一次抽卡用 `seed = hash(memberId, localDate)`，结果稳定；刷新页面不换。
- 每天最多 3 次；用完提示「今天抽完了，自己挑」。
- 重抽排除刚才抽到的搭配，但不清空当天已抽记录。
- 抽卡结果不自动写穿着记录；用户点「穿了」才写 `WearLog`。
- 无候选时必须返回原因（例如「3 套搭配的衣物都在洗」），不返回随机垃圾。
- 抽卡记录只保留 `outfit_id`、`attempt`、`seed`，不记录当时天气正文。

### 7.5 为什么只抽已存搭配

从单件随机组合需要判断「搭不搭」，这是审美问题；规则化之后会产出「羽绒服 + 短裤」这种既不可解释也不好看的结果。v1 只从用户自己保存的搭配里抽；没有合适搭配时提示新建，而不是硬凑。

## 8. 洗护冲突检查

输入：衣物集合 + 计划程序（水温、是否烘干、是否漂白、是否干洗）。输出：`ok` / `warn` / `conflict` + 逐条原因与涉及件数。

确定性规则：

- 任一件 `do_not_wash` 而计划水洗 → `conflict`。
- 计划水温 > 任一件 `wash_temp_max_c` → `conflict`，列出最低可接受温度与件数。
- 计划烘干而任一件 `tumble_dry=do_not` → `conflict`。
- 计划漂白而任一件 `bleach=none` → `conflict`。
- 计划干洗而任一件 `dry_clean=do_not` → `conflict`。
- 只能干洗与水洗件同桶 → `conflict`，建议拆桶。
- 手洗件与机洗件同桶 → `warn`。
- 深色与浅色同桶 → `warn`（启发式，可关闭）。
- 30 天内新购入的深色与浅色同桶 → `warn`。

约束：

- 只警告，不改洗衣机程序；程序仍由用户在机器上设置（承接 D38）。
- 不推断衣物是否「真的脏」；状态由人维护。
- 冲突检查不写日志正文，只记件数与结果。

## 9. UI（原生 / Web）

- 原生：衣橱列表（按类别、季节、状态筛选）、衣物详情、洗护符号可视化、搭配编辑、今日抽卡、洗涤批次与冲突提示、天气条。
- 抽卡页显示：今日温度与天气、抽到的搭配、每件当前状态（可穿 / 在洗）、「重抽 2/3」「穿了」「换一套」。
- 洗护符号用图标 + 文案，不能只靠颜色或只给代码。
- 冲突提示按件分组，先给结论再给原因，不弹全屏警告。
- Web v1 不提供衣物页面；穿着记录与抽卡能推断作息与出门，服务端拒绝 Web 来源的 `wardrobe.*` RPC。

## 10. 数据与迁移

```sql
CREATE TABLE m_wardrobe_care_profile (
  id              TEXT PRIMARY KEY,
  name            TEXT NOT NULL,
  wash_method     TEXT NOT NULL,
  wash_temp_max_c INTEGER,
  bleach          TEXT NOT NULL,
  tumble_dry      TEXT NOT NULL,
  iron            TEXT NOT NULL,
  dry_clean       TEXT NOT NULL,
  note            TEXT,
  builtin         INTEGER NOT NULL DEFAULT 0,
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);

CREATE TABLE m_wardrobe_garment (
  id              TEXT PRIMARY KEY,
  member_id       TEXT NOT NULL,
  name            TEXT NOT NULL,
  category        TEXT NOT NULL,
  color           TEXT,
  size            TEXT,
  brand           TEXT,
  purchased_at    INTEGER,
  price_cents     INTEGER,
  care_profile_id TEXT NOT NULL,
  location_id     TEXT,
  status          TEXT NOT NULL DEFAULT 'clean',
  photo_doc_id    TEXT,
  note            TEXT,
  archived_at     INTEGER,
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);
CREATE INDEX idx_wardrobe_garment_member
  ON m_wardrobe_garment(member_id, archived_at, category);
CREATE INDEX idx_wardrobe_garment_status
  ON m_wardrobe_garment(member_id, status);
CREATE INDEX idx_wardrobe_garment_location
  ON m_wardrobe_garment(location_id);

CREATE TABLE m_wardrobe_outfit (
  id            TEXT PRIMARY KEY,
  member_id     TEXT NOT NULL,
  name          TEXT NOT NULL,
  seasons_json  TEXT,
  formality     TEXT NOT NULL DEFAULT 'casual',
  comfort_min_c INTEGER,
  comfort_max_c INTEGER,
  rain_ok       INTEGER NOT NULL DEFAULT 1,
  wind_ok       INTEGER NOT NULL DEFAULT 1,
  note          TEXT,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX idx_wardrobe_outfit_member
  ON m_wardrobe_outfit(member_id, archived_at);

CREATE TABLE m_wardrobe_outfit_item (
  outfit_id  TEXT NOT NULL,
  garment_id TEXT NOT NULL,
  position   INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (outfit_id, garment_id)
);
CREATE INDEX idx_wardrobe_outfit_item_garment
  ON m_wardrobe_outfit_item(garment_id);

CREATE TABLE m_wardrobe_wear_log (
  id         TEXT PRIMARY KEY,
  member_id  TEXT NOT NULL,
  garment_id TEXT NOT NULL,
  outfit_id  TEXT,
  local_date TEXT NOT NULL,
  note       TEXT,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_wardrobe_wear_member
  ON m_wardrobe_wear_log(member_id, local_date);
CREATE INDEX idx_wardrobe_wear_garment
  ON m_wardrobe_wear_log(garment_id, local_date);

CREATE TABLE m_wardrobe_laundry_load (
  id          TEXT PRIMARY KEY,
  member_id   TEXT NOT NULL,
  program     TEXT,
  wash_temp_c INTEGER,
  tumble_dry  INTEGER NOT NULL DEFAULT 0,
  bleach      INTEGER NOT NULL DEFAULT 0,
  dry_clean   INTEGER NOT NULL DEFAULT 0,
  status      TEXT NOT NULL DEFAULT 'planned',
  device_id   TEXT,
  started_at  INTEGER,
  finished_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_wardrobe_load_member
  ON m_wardrobe_laundry_load(member_id, created_at);

CREATE TABLE m_wardrobe_laundry_item (
  load_id    TEXT NOT NULL,
  garment_id TEXT NOT NULL,
  PRIMARY KEY (load_id, garment_id)
);

CREATE TABLE m_wardrobe_draw (
  id         TEXT PRIMARY KEY,
  member_id  TEXT NOT NULL,
  local_date TEXT NOT NULL,
  outfit_id  TEXT NOT NULL,
  attempt    INTEGER NOT NULL,
  seed       TEXT NOT NULL,
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_wardrobe_draw_member
  ON m_wardrobe_draw(member_id, local_date);

CREATE TABLE m_wardrobe_weather (
  local_date       TEXT NOT NULL,
  location_label   TEXT NOT NULL,
  temp_c           REAL,
  feels_like_c     REAL,
  humidity         INTEGER,
  precipitation_mm REAL,
  wind_kph         REAL,
  condition        TEXT NOT NULL DEFAULT 'unknown',
  source           TEXT NOT NULL,
  fetched_at       INTEGER NOT NULL,
  PRIMARY KEY (local_date, location_label)
);
```

- 洗护画像内置若干模板（棉、羊毛、真丝、羽绒、皮革、运动），管理员可扩展；内置只能归档，不能删除。
- 不存身体尺寸、不存人像照片；衣物照片只存 `documents` 的引用。
- 天气快照只保留最近 14 天，按日清理。
- 穿着记录本人可删；删除只写事件，不保留正文。
- 洗涤批次完成后不自动改衣物状态，必须由用户确认。
- 迁移只向前；衣物默认归档，不物理删除。

## 11. 验收

- 新建衣物并选洗护画像，详情显示符号与温度上限。
- 存 3 套搭配后，「今天穿什么」按今日体感温度与季节抽出其中一套；刷新结果不变。
- 抽到的搭配里有衣物在洗，该套被排除；全部不可穿时给出明确原因。
- 每天第 4 次抽卡被拒绝，提示自己挑。
- 抽卡不自动写穿着记录；点「穿了」才写入并更新穿着次数与最后穿着日。
- 洗涤批次含 40 °C 上限衣物却选 60 °C → `conflict`，列出件数与最低温度。
- 只能干洗件混入水洗桶 → `conflict`；深浅色混洗只 `warn`。
- 冲突检查不修改洗衣机程序，也不调用 `smart_home` 命令。
- 洗衣机 `cycle_state=done` 只触发提醒，衣物状态不自动变 `clean`。
- 他人可看到衣物与搭配，但看不到穿着记录；管理员也看不到。
- 天气抓取失败时按季节降级，抽卡仍可用并标注 `stale`。
- Web 无衣物页且 `wardrobe.*` 被拒绝；日志不含位置、价格与穿着记录。

## 12. 开放问题

- 天气是否提升为独立 `weather` 模块，供 `meals`、`travel`、`calendar` 共用。
- 是否允许从单件随机组合成搭配；当前只抽已存搭配。
- 每天 3 次抽卡上限是否合适；是否可配置。
- 是否需要洗护符号拍照识别；v1 手动选画像。
- 是否需要家庭共用衣物（雨伞、备用正装）；v1 放 `inventory`。
- 穿着记录是否允许批量补记与日历叠加。
- 是否开放 Web 只读衣橱。
- 换季收纳是否生成 `tasks` 提醒。
- 是否需要衣物寿命/淘汰建议；v1 只提供穿着次数。
- `lent` 是否记录借给谁与归还提醒；v1 只记借出对象文本。
- 是否需要洗涤剂消耗记录；当前归 `inventory`。
