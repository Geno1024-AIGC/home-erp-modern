# 模块：旅行（travel）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_travel_*`

## 1. 目的与非目标

**目的**：管理旅行的完整生命周期：计划目的地、日期、同行人、逐日行程、预订与清单；旅行中或结束后记录实际行程、游记与照片。计划与记录是同一条旅行的两个阶段，不重复建两条数据。

**非目标**：

- 不做机票/酒店/门票搜索、预订、改签、比价。
- 不做地图导航、离线地图、GPS 轨迹；地点只存文本与可选坐标，可交给外部地图打开。
- 不做护照/签证/保险办理；证件号码归 `identity`，扫描件归 `documents`。
- 不做旅行记账、分账、报销与预算结算；v1 只存一个可选预算总额。
- 不做实时协作、旅行聊天、公开分享、社交动态。
- 不复制日历事件、任务或文件；日历只读叠加，准备任务归 `tasks`，附件归 `documents`。
- 不自动抓取天气、汇率、航班状态；天气/心情由用户手填。
- v1 在线优先，不做离线地图与离线同步；可打印/导出当前行程。
- Web 暂不提供旅行页；行程、住址与游记只在原生客户端，避免行星看到明文。

## 2. 用户场景

- 计划一次家庭旅行：目的地、日期、同行人、预算。
- 按天排行程：航班/火车、酒店、景点、餐厅、休息。
- 记录航司/酒店名称、预订编号与确认状态，附上机票、酒店确认单、保险单。
- 出发前用打包清单准备证件、药品、装备，并分派给成员。
- 把「订酒店」「检查护照」变成 `tasks` 提醒；在日历上看到旅行占用。
- 旅行中临时改计划：标记跳过/改期，记录实际时间。
- 每天写游记、传照片；旅行结束后回看整趟行程。
- 补记一次已经完成的旅行，不需要先做计划。
- 按年份、同行人、目的地翻看旅行记录。

## 3. 边界与依赖

- 拥有旅行、目的地、同行人、逐日行程、清单与游记。
- 只引用 `member_id`；同行人快照 `display_name`，非成员也能记录；不建跨模块外键。
- 日历事件由 `calendar` 拥有；`travel` 不创建日历事件，日历只读叠加旅行日期。
- 准备任务与提醒由 `tasks` 拥有；可用 `subject_type="travel.trip"` 关联，旅行不复制任务状态。
- 机票、酒店确认单、保险单、门票、游记照片归 `documents`；`travel` 只通过 `documents.link` 关联，角色用 `ticket`、`booking`、`insurance`、`trip_photo`、`journal_photo`、`cover`。
- 证件号码与证件扫描件归 `identity` / `documents`；`travel` 不存完整护照号、签证号。
- 成员资料由 `members` 拥有；同行人只快照显示名，成员移除后历史仍可读。
- 通知由宿主能力发送；默认不在锁屏/通知预览显示目的地与住宿地址。
- 旅行目的地不是家庭位置，不写入 `locations`；`locations` 只服务家中位置森林。
- 旅行日期、住宿地址与游记能推断家中无人，默认仅家庭成员可见，`guest` 无权限。
- v1 不做花费流水；预算只是计划值，实际花费后续再评估。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Trip` | `id`、`title`、`status`、`start_date`、`end_date`、`timezone`、`destination_summary`、`budget_cents`、`currency`、`notes`、`client_id`、`created_by`、`archived_at` |
| `TripDestination` | `id`、`trip_id`、`name`、`country_code`、`region`、`address`、`lat`、`lon`、`timezone`、`sort` |
| `TripTraveler` | `id`、`trip_id`、`member_id`、`display_name`、`role`、`created_at` |
| `TripDay` | `id`、`trip_id`、`date`、`title`、`notes` |
| `TripItem` | `id`、`trip_id`、`day_id`、`end_day_id`、`kind`、`title`、`start_time`、`end_time`、`place_name`、`address`、`lat`、`lon`、`timezone`、`provider`、`booking_ref`、`booking_status`、`status`、`actual_start_time`、`actual_end_time`、`note`、`sort` |
| `TripChecklistItem` | `id`、`trip_id`、`title`、`category`、`assignee_member_id`、`checked_at`、`checked_by`、`sort` |
| `TripJournalEntry` | `id`、`trip_id`、`day_id`、`at_time`、`place_name`、`title`、`body`、`mood`、`weather`、`client_id`、`created_by`、`archived_at` |

`Trip.status`：`planning`（计划中）、`confirmed`（已确认）、`completed`（已完成）、`cancelled`（已取消）。「进行中」由日期与状态派生，不落库。

`TripTraveler.role`：`organizer`（组织者）、`traveler`（同行人）。

`TripItem.kind`：`transport`、`lodging`、`activity`、`meal`、`rest`、`shopping`、`note`、`other`。

`TripItem.booking_status`：`none`、`pending`、`confirmed`、`cancelled`。

`TripItem.status`：`planned`、`done`、`skipped`、`changed`。

`TripChecklistItem.category`：`document`、`gear`、`medicine`、`other`。

规则：

- 一趟真实旅行只有一条 `Trip`；计划与记录不拆成两张主表。
- 补记已完成的旅行可直接建 `status=completed` 的 `Trip`，只写游记与照片，不要求先有行程。
- `start_date <= end_date`；日期按目的地当地日期（`YYYY-MM-DD`），`timezone` 为可选 IANA 名称。
- 行程时间用当地墙上时间（`HH:MM`），v1 不做跨时区自动换算；跨时区交通把出发/到达当地时间写在备注或拆成两条。
- `TripDay` 在首次添加行程或游记时按日期懒创建；`UNIQUE (trip_id, date)`；日期必须在旅行范围内。
- `TripItem` 默认属于一天；跨天住宿/夜车用 `end_day_id`，结束日不得早于开始日，且必须在旅行范围内。
- 行程按日期、`start_time`（空值最后）、`sort` 排序；不做拖拽依赖。
- 状态由用户显式修改，不按日期自动写 `completed`；日期已过但仍 `planning` 的旅行在列表中提示「待确认」。
- `travel.trips.create` 与 `travel.journal.create` 接受可选 `clientId`；同一用户同一 `clientId` 幂等，重试不生成重复记录。
- 缩短旅行日期时，若范围外仍有行程或游记则拒绝；先移动或删除，再改日期。
- 预算 `budget_cents` 只是计划值，不产生实际花费记录；币种用 ISO 4217。
- 清单勾选只保留当前状态，不写任务历史；它不是 `tasks` 的替代品。
- `travel.checklists.copyFrom` 只复制未归档清单项的标题、分类与顺序；不复制勾选、指派与备注；按标题去重，已存在则跳过。
- 游记可编辑；作者可改自己的正文，管理员只能归档，不能改他人正文。
- 归档旅行不删除行程、游记与文档关联；默认列表不返回，恢复后重新出现。
- 移除同行人时，清单上的指派置空；游记作者与同行人显示名保留快照。
- 旅行的封面取 `documents` 中 `role="cover"` 的第一张；同一目标最多一个 `cover`，由 `travel` 校验。
- 文档关联不产生文件引用；删除关联或归档旅行都不删文件。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `travel.read` | 查看旅行、行程、清单、游记与附件元数据 | `member` |
| `travel.write` | 新建/编辑旅行、行程、清单与自己的游记，导出旅行 | `member` |
| `travel.admin` | 归档/恢复旅行、归档他人游记、管理保留策略 | `admin` |

- `guest` 默认不能查看任何旅行信息；旅行日期与住址会暴露家中无人。
- v1 家庭内全部成员可见；私密旅行/惊喜行程后续再评估。
- 作者可编辑自己的游记；管理员可归档不适当内容，但不能改正文。
- 附件下载仍由 `documents` 权限裁决；`travel` 不绕过文档权限。
- 通知默认只显示「旅行提醒」，不显示目的地、住宿地址与游记正文。

## 6. RPC 与事件

- `travel.trips.list`：`{ status?, from?, to?, travelerMemberId?, includeArchived?, limit?, cursor? }`，按开始日期倒序；`from`/`to` 按区间相交过滤，`status` 额外支持派生的 `ongoing`。
- `travel.trips.get`：返回旅行、目的地、同行人、逐日摘要、清单进度与文档计数。
- `travel.trips.create`：`{ title, startDate, endDate, status?, timezone?, destinationSummary?, budgetCents?, currency?, notes?, travelers?, clientId? }`；组织者默认当前成员，`status` 默认 `planning`，补记可直接传 `completed`。
- `travel.trips.update`：`{ tripId, ... }`；缩短日期时校验范围外行程/游记。
- `travel.trips.archive` / `travel.trips.restore`
- `travel.trips.listInRange`：`{ from, to }`，只返回与区间相交、未归档且未取消的旅行摘要（`id`、`title`、`startDate`、`endDate`、`status`），供日历只读叠加。
- `travel.trips.export`：`{ tripId, includeJournal?, includeDocuments? }`，返回结构化行程与文档清单；需 `travel.write`，客户端渲染打印/PDF，服务端写审计但不记录正文；文档元数据仍受 `documents.read` 限制。
- `travel.destinations.list` / `create` / `update` / `reorder` / `delete`
- `travel.travelers.list` / `add` / `remove`；不能移除最后一位组织者。
- `travel.days.list` / `update`：只改标题与备注；日期由行程/游记创建时补齐。
- `travel.items.list`：`{ tripId, dayId?, limit?, cursor? }`，按日期与顺序返回。
- `travel.items.create`：`{ tripId, date, endDate?, kind, title, startTime?, endTime?, placeName?, address?, lat?, lon?, timezone?, provider?, bookingRef?, bookingStatus?, note? }`
- `travel.items.update` / `delete`
- `travel.items.mark`：`{ itemId, status, actualStartTime?, actualEndTime?, note? }`
- `travel.checklist.list` / `add` / `update` / `check` / `uncheck` / `delete` / `reorder`
- `travel.checklists.copyFrom`：`{ tripId, fromTripId, categories? }`，按标题去重复制清单项，返回新建与跳过数量。
- `travel.journal.list`：`{ tripId, dayId?, limit?, cursor? }`，按时间倒序分页。
- `travel.journal.create`：`{ tripId, date, atTime?, placeName?, title, body?, mood?, weather?, clientId? }`
- `travel.journal.update` / `archive` / `restore`：更新仅作者；归档/恢复允许作者或管理员。
- 事件：
  - `travel.trip.changed { tripId, action }`
  - `travel.item.changed { tripId, itemId, action }`
  - `travel.checklist.changed { tripId, itemId, action }`
  - `travel.journal.changed { tripId, entryId, action }`

不提供：

- 不提供机票/酒店/门票搜索、预订、改签与价格比较。
- 不提供地图导航、路线规划、GPS 轨迹与离线地图。
- 不提供旅行花费流水、分账、报销与预算结算。
- 不提供票据 OCR、天气/汇率/航班状态抓取。
- 不提供公开分享链接、社交发布与实时协作编辑。
- 不提供自动创建日历事件；日历只读叠加。
- 不提供离线同步；v1 在线优先，打印/导出用于离线查看。

## 7. UI（原生 / Web）

- 原生：导航「旅行」；按「进行中 / 计划中 / 已完成 / 已取消」分组，支持按年份、同行人、目的地筛选。
- 旅行详情：标题、日期、状态、目的地、同行人、预算、封面；标签页为「行程 / 清单 / 预订与文件 / 游记 / 同行人」。
- 行程：按天时间线；添加交通/住宿/活动/餐饮/休息；显示预订状态；可标记完成/跳过/改期并填实际时间。
- 清单：按证件/装备/药品/其他分组；可指派成员、勾选、快速添加；支持从上次旅行复制，复制时重置勾选与指派；不进入任务历史。
- 预订与文件：调 `documents.listFor(targetType="travel.trip")`；打开机票、酒店确认单、保险单；可新建关联。
- 游记：按时间倒序；快速添加日期、时间、地点、正文、心情、天气；照片通过 `documents` 关联到 `travel.journal_entry`。
- 完成视图：对比计划与实际，显示完成/跳过/改期数量、游记与照片数量。
- 日历：只读叠加 `travel.trips.listInRange`；点击跳转旅行详情，不在日历里改行程。
- 任务：旅行详情显示 `tasks.listFor(subjectType="travel.trip")`；可从旅行创建准备任务，但状态仍由 `tasks` 管理。
- 打印/导出：调 `travel.trips.export` 获取结构化行程，客户端渲染为可打印页面或 PDF；导出前提示包含个人行程，不自动分享，服务端写审计。
- 通知：默认「旅行提醒」，不显示目的地与住址；用户可在设置中开启详细预览。
- Web：v1 不提供旅行页；日历 Web 也不显示旅行条目，避免行程与住址经行星明文暴露；旅行关联文档在 Web 默认不显示旅行上下文与缩略图。
- 不内置地图 SDK；有坐标时提供「在地图中打开」，交给系统或外部应用。

## 8. 数据与迁移

```sql
CREATE TABLE m_travel_trip (
  id                  TEXT PRIMARY KEY,
  title               TEXT NOT NULL,
  status              TEXT NOT NULL,          -- planning / confirmed / completed / cancelled
  start_date          TEXT NOT NULL,          -- YYYY-MM-DD
  end_date            TEXT NOT NULL,
  timezone            TEXT,
  destination_summary TEXT,
  budget_cents        INTEGER,
  currency            TEXT,
  notes               TEXT,
  client_id           TEXT,
  created_by          TEXT NOT NULL,
  archived_at         INTEGER,
  created_at          INTEGER NOT NULL,
  updated_at          INTEGER NOT NULL
);
CREATE INDEX idx_travel_trip_dates ON m_travel_trip(start_date, end_date);
CREATE INDEX idx_travel_trip_status ON m_travel_trip(status, archived_at);
CREATE UNIQUE INDEX idx_travel_trip_client
  ON m_travel_trip(created_by, client_id) WHERE client_id IS NOT NULL;

CREATE TABLE m_travel_destination (
  id           TEXT PRIMARY KEY,
  trip_id      TEXT NOT NULL,
  name         TEXT NOT NULL,
  country_code TEXT,
  region       TEXT,
  address      TEXT,
  lat          REAL,
  lon          REAL,
  timezone     TEXT,
  sort         INTEGER NOT NULL DEFAULT 0,
  created_at   INTEGER NOT NULL
);
CREATE INDEX idx_travel_destination_trip ON m_travel_destination(trip_id, sort);

CREATE TABLE m_travel_traveler (
  id           TEXT PRIMARY KEY,
  trip_id      TEXT NOT NULL,
  member_id    TEXT,
  display_name TEXT NOT NULL,
  role         TEXT NOT NULL,                 -- organizer / traveler
  created_at   INTEGER NOT NULL,
  UNIQUE (trip_id, member_id)
);
CREATE INDEX idx_travel_traveler_trip ON m_travel_traveler(trip_id);
CREATE INDEX idx_travel_traveler_member ON m_travel_traveler(member_id);

CREATE TABLE m_travel_day (
  id         TEXT PRIMARY KEY,
  trip_id    TEXT NOT NULL,
  date       TEXT NOT NULL,                   -- YYYY-MM-DD
  title      TEXT,
  notes      TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL,
  UNIQUE (trip_id, date)
);

CREATE TABLE m_travel_item (
  id                TEXT PRIMARY KEY,
  trip_id           TEXT NOT NULL,
  day_id            TEXT NOT NULL,
  end_day_id        TEXT,
  kind              TEXT NOT NULL,
  title             TEXT NOT NULL,
  start_time        TEXT,                     -- HH:MM，当地墙上时间
  end_time          TEXT,
  place_name        TEXT,
  address           TEXT,
  lat               REAL,
  lon               REAL,
  timezone          TEXT,
  provider          TEXT,
  booking_ref       TEXT,
  booking_status    TEXT NOT NULL DEFAULT 'none',
  status            TEXT NOT NULL DEFAULT 'planned',
  actual_start_time TEXT,
  actual_end_time   TEXT,
  note              TEXT,
  sort              INTEGER NOT NULL DEFAULT 0,
  created_at        INTEGER NOT NULL,
  updated_at        INTEGER NOT NULL
);
CREATE INDEX idx_travel_item_trip ON m_travel_item(trip_id, day_id, sort);

CREATE TABLE m_travel_checklist_item (
  id                 TEXT PRIMARY KEY,
  trip_id            TEXT NOT NULL,
  title              TEXT NOT NULL,
  category           TEXT NOT NULL,           -- document / gear / medicine / other
  assignee_member_id TEXT,
  checked_at         INTEGER,
  checked_by         TEXT,
  sort               INTEGER NOT NULL DEFAULT 0,
  created_at         INTEGER NOT NULL,
  updated_at         INTEGER NOT NULL
);
CREATE INDEX idx_travel_checklist_trip ON m_travel_checklist_item(trip_id, sort);

CREATE TABLE m_travel_journal (
  id         TEXT PRIMARY KEY,
  trip_id    TEXT NOT NULL,
  day_id     TEXT,
  at_time    TEXT,                            -- HH:MM，当地墙上时间
  place_name TEXT,
  title      TEXT NOT NULL,
  body       TEXT,
  mood       TEXT,
  weather    TEXT,
  client_id  TEXT,
  created_by TEXT NOT NULL,
  archived_at INTEGER,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
CREATE INDEX idx_travel_journal_trip ON m_travel_journal(trip_id, created_at);
CREATE UNIQUE INDEX idx_travel_journal_client
  ON m_travel_journal(created_by, client_id) WHERE client_id IS NOT NULL;
```

- 日期用 `YYYY-MM-DD`，时间用 `HH:MM`；展示按旅行/目的地时区，不做自动换算。
- 金额用最小货币单位 + ISO 4217；预算可选，不存实际花费。
- `trip_id`、`member_id`、`document_id` 都是逻辑引用，不建跨模块外键。
- 行程/游记照片与票据不存本模块；通过 `documents` 关联，`targetType` 为 `travel.trip` 或 `travel.journal_entry`。
- 迁移只向前；旅行与游记默认归档，不物理删除；行程、清单项、目的地与同行人属于计划数据，可删除。
- v1 不做全文搜索；按旅行、年份、同行人、目的地过滤；后续可加 FTS。

## 9. 验收

- 新建旅行，填写日期、目的地、同行人；按天添加交通/住宿/活动，跨天住宿正确显示。
- 直接补记一次已完成旅行：不建行程也能写游记、传照片。
- 旅行日期缩短时，范围外有行程/游记会被拒绝；移动或删除后可保存。
- 日历只读显示旅行占用，点击进入旅行详情；日历不能修改旅行，也不生成重复事件。
- 旅行准备任务关联到 `subjectType="travel.trip"`；完成任务不回写旅行，旅行也不复制任务状态。
- 从上次旅行复制清单只复制标题、分类与顺序；勾选与指派重置，重复复制不产生重复项。
- 机票、酒店确认单、保险单与游记照片通过 `documents` 关联；删除关联不删文件，归档旅行后文档仍可查。
- 行程标记完成/跳过/改期与实际时间正确；计划与实际可对比。
- 游记作者可编辑，管理员只能归档；归档后默认不返回，恢复后出现。
- `guest` 无权限；家庭成员可读；附件下载仍受 `documents` 权限限制。
- 通知与日志不含目的地、住宿地址与游记正文。
- 导出/打印只包含当前行程，必须显式触发；不自动生成公开链接。
- 1,000 条行程/游记仍可分页加载；旅行列表按日期与状态过滤正常。

## 10. 开放问题

- 是否需要私密旅行/惊喜行程，以及按成员可见性。
- 是否提供 Web 只读行程页；v1 为隐私仅原生。
- 是否需要旅行花费流水、分账与预算结算；v1 只存预算总额。
- 是否需要离线缓存、离线地图与本地行程包。
- 是否需要多个组织者与更细的编辑权限。
- 是否接天气、航班状态、汇率等外部数据；v1 只手填。
- 是否需要游记评论、表情回应与家庭共享时间线。
- 是否需要独立的打包清单模板（而不只是从上次复制）；v1 用 `copyFrom` 复制上次清单。
- 跨时区交通是否要记录出发/到达双时区与精确 UTC 时间；v1 用当地墙上时间。
- 是否需要保留计划日期与实际日期两份；v1 直接改日期，不保留差异历史。
- 旅行票据是否做 OCR 与自动填充；v1 只上传与手动填写。
- 日历叠加显示目的地还是只显示「旅行占用」；v1 原生显示标题与日期。
