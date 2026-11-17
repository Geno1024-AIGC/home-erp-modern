# 模块：物品（inventory）

> 状态：draft ｜ 类型：business ｜ 依赖：`locations`、`documents`（可选） ｜ 拥有：`m_inventory_*`

## 1. 目的与非目标

**目的**：记录家里的物品与库存：消耗品数量、保质期、开封后的使用期限、出入库；耐用品型号、序列号、保修与退货窗口；借出与归还；药品/保健品的批次与开封；支持扫商品条码快速录入；按消耗速度给出补货点与建议；价格历史与趋势见 [inventory/pricing.md](inventory/pricing.md)。

**非目标**：

- 不做采购、供应商、发票与账务。
- 不做菜谱、做饭记录与熟食剩菜；剩菜归 `meals`。
- 不做押金、租金、合同与赔偿；借出只记事实，见 [inventory/loans.md](inventory/loans.md)。
- 不做药品剂量计算、相互作用、医疗建议与处方审核；处方/用药记录归 `health`。
- 不自动下单、不自动补货、不预测季节性需求；建议只读，写入由人确认。
- 不做食品安全判断；开封后期限只是提醒，不是保质保证。
- 不做维护计划；v1 由 `tasks` 建提醒，后续再评估 `assets`。
- 不做服务端图像解码、商品比价与价格抓取；扫码解码在卫星/浏览器完成。
- 不存条码图片、相机帧或商品图；照片归 `documents`。
- 不建全局商品主数据；条码↔物品映射只在当前家庭内有效。
- 不自动从外部商品库创建物品；查询结果只作草稿，必须由用户确认。
- v1 不做自动抓价、比价、价格预测、预算与汇率换算；价格历史设计见 [inventory/pricing.md](inventory/pricing.md)。

## 2. 用户场景

- 新增一箱牛奶：数量、单位、保质期、位置。
- 取用/补货：记录出入库，查看当前数量。
- 拆封一盒牛奶：记录开封时间与开封后可用天数，临期提示。
- 给纸巾设补货点：剩 2 包时进入补货建议。
- 查看某物品最近 90 天的消耗速度与预计用完时间。
- 把退货包裹标记为待退，出门前能看到要带走的物品。
- 借出电钻/书/投影仪：记录借给谁、何时还；归还时记品相。
- 把药品/保健品放进药箱：记录批号、有效期、开封后可用天数与剩余量。
- 买回来入库时落到默认位置，并顺手记价格。
- 新增一台电视：品牌、型号、序列号、购买日期、保修期、位置。
- 查看某个柜子里有哪些物品。
- 查看即将过期的物品、低库存物品。
- 在物品详情打开说明书、关联家务/维护任务。
- 扫商品条码：已认识的码直接打开物品或进入补货；未知码可先建物品、稍后补资料。
- 批量扫一箱/一柜物品，逐个确认数量与位置后入库。
- 同一商品的不同包装各有条码，都指向同一物品；扫到归档物品时提示恢复，不静默复用。
- 可选：显式调用外部商品库补全名称、品牌、规格；结果可改，不联网也能继续。
- 补货时顺手记录实付价；在超市看到货架价也可单独记一条。
- 在物品详情看价格折线图：最新、最低、最高、平均与涨跌幅；按商家/促销过滤。

## 3. 边界与依赖

- 拥有物品主数据、批次、库存流水、条码映射与外部查询缓存。
- 只引用 `location_id`；不建跨模块外键。
- 开封时间、开封后可用天数、补货点与消耗速度归 `inventory`；熟食剩菜与菜谱匹配归 `meals`。
- 借出记录归 `inventory`，联系人归 `contacts`；`Item.status=lent`/`lost` 由借出记录派生，见 [inventory/loans.md](inventory/loans.md)。
- 药品/保健品库存归 `inventory`；处方、用药、过敏与急救可见性归 `health`；不做剂量建议。
- `Item.metadata_json` 只放类别相关字段（如药品剂型、含量），不放密码、完整证件号或医疗正文。
- `Item.status` 只表示流转状态（待退/待修/借出/丢失等）；位置仍由批次 `location_id` 与 `home_location_id` 表示。
- 消耗速度只从库存流水估算；补货建议只读，不改数量、不自动下单。
- 库存可用量由 `inventory.items.availability` 提供给 `meals`；`meals` 不查库存表。
- 说明书/照片/发票由 `documents` 拥有；`inventory` 调 `documents` RPC 展示。
- 家务/维护提醒由 `tasks` 拥有；`inventory` 不复制任务状态。
- v1 不单独建 `assets`；耐用品是 `Item.kind = durable`。
- 扫码解码归卫星/宿主的平台能力；恒星只接收条码字符串并在服务端校验/归一化，不接收相机帧。
- 条码标识「物品」，不标识批次；数量、保质期、位置仍由 `StockLot` 管理。
- 外部商品库查询是可选的 Provider；默认关闭或需显式触发，失败不阻塞录入。
- 二维码按不透明字符串处理，不自动打开链接、不执行动作。
- 不下载、不热链外部商品图；用户可自己拍照并归 `documents`。
- 价格观测与商家主数据归 `inventory`；价格只追加，不静默改价；图表只读。
- 价格不自动抓取、不跨币种换算；小票/货架照片归 `documents`。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Item` | `id`、`name`、`kind`、`status`、`category`、`brand`、`model`、`serial`、`purchase_date`、`warranty_until`、`return_by`、`home_location_id`、`reorder_at`、`reorder_quantity`、`opened_use_within_days`、`metadata_json`、`notes`、`archived_at` |
| `StockLot` | `id`、`item_id`、`location_id`、`quantity`、`unit`、`expires_at`、`opened_at`、`use_within_days`、`acquired_at`、`price_cents`、`currency` |
| `StockMovement` | `id`、`item_id`、`lot_id`、`delta`、`reason`、`actor_id`、`at` |
| `ItemBarcode` | `id`、`item_id`、`code`、`raw_code`、`code_type`、`pack_quantity`、`pack_unit`、`source`、`verified_at`、`archived_at` |
| `CatalogCache` | `code`、`provider_id`、`payload_json`、`fetched_at`、`expires_at` |
| `Merchant` | `id`、`name`、`kind`、`location_id`、`archived_at` |
| `PriceObservation` | `id`、`item_id`、`observed_at`、`total_price_cents`、`currency`、`quantity`、`net_amount`、`net_unit`、`unit_price_micros`、`kind`、`merchant_id`、`voided_at` |

`Item.kind`：`consumable`（消耗品，按批次管理数量/保质期）、`durable`（耐用品，通常数量 1，重型号/序列号/保修）。

`Item.status`：`in_use`（在用）、`to_return`（待退）、`in_repair`（在修）、`lent`（借出，派生）、`lost`（丢失）、`disposed`（已处置，终态，仅归档）。

`StockMovement.reason`：`purchase`、`consume`、`spoil`、`open`、`adjust`、`move`、`return`、`gift`。

`code_type`：`gtin`（EAN-8/13、UPC-A/E、GTIN-14 统一归入）、`code128`、`qr`、`other`。

规则：

- `StockMovement` 只追加，不修改历史；当前数量由批次汇总。
- 默认不允许负库存；管理员可强制调整，必须填写原因。
- 消耗品按 `StockLot` 管理；耐用品可以只有一个批次，`quantity = 1`。
- 序列号在 `kind=durable` 下建议唯一；不强制全局唯一，允许同型号多台。
- 归档物品不删除流水；历史仍可查。
- `expires_at` 是未开封保质期；`opened_at + use_within_days` 是开封后期限。有效期限取两者较早值，只用于查询与事件，不改数量。
- 开封是批次级动作：一包多件先 `lots.split` 拆出要开的那份，再 `lots.open`；不复制物品、不改条码。
- `use_within_days` 为空时只按 `expires_at`；`opened_at` 为空时视为未开封，不推断开封时间。
- `Item.status` 默认 `in_use`；`to_return` / `in_repair` / `lent` / `lost` 表示物品卡在流转中，不改变位置与数量；`disposed` 后只能归档。
- `lent` 由 `inventory.loans` 的活跃记录派生，`items.update` 不能直接设为 `lent`；`lost` 由借出丢失或用户显式标记，管理员可恢复。借出规则见 [inventory/loans.md](inventory/loans.md)。
- `Item.metadata_json` 放类别相关字段：药品/保健品可用 `genericName`、`form`、`strength`、`prescription`、`forMemberIds`、`instructions`、`disposal`；不放密码、完整证件号、医疗正文或支付信息。
- 药品/保健品仍按 `StockLot` 管理数量与有效期；`category` 用 `medicine` / `supplement`，不做剂量计算、相互作用或自动补货。
- `return_by` 是退货/换货窗口；到期只是查询与提醒，不自动改 `status`。
- 活跃借出会从 `availability` 扣除；借出不影响批次数量与流水，但物品不能同时借给两人。
- `reorder_at` 用与批次相同的单位；同一物品存在多种单位时不做低库存判断，避免把克和个相加。`reorder_quantity` 只作建议默认值，不是自动补货量。
- 只有 `consume` 与 `spoil` 计入消耗速度；`adjust` / `move` / `purchase` / `return` / `gift` 不计入，避免盘点和搬家把预测带偏。
- 消耗速度按窗口内线性平均；有效样本不足时返回 `insufficient`，不外推、不插值。
- 过期/低库存/补货建议都只是查询与事件，不自动改数量、不自动下单。
- GTIN 家族需验证校验位并规范化为 14 位数字；其他码去空白、限长 512，按原样保存。
- 一个活跃条码只指向一个物品；一个物品可以有多条码（不同包装/地区/内部码）。
- `pack_quantity` / `pack_unit` 只作为补货默认值，不自动改库存。
- 重复扫码：同一物品幂等；指向其他物品时报冲突，需先解绑或确认。
- `items.create` 带条码时与物品同事务；任一码冲突则整体失败，不留下半建物品。
- 归档物品或条码被扫到时提示恢复；不静默改绑到新物品。
- 外部查询结果不可信：只用于预填，用户确认后才写物品字段；不自动创建。
- 外部查询缓存默认 30 天；禁用 Provider 时清空；缓存不含凭证。
- 二维码内容不解析为 URL；超长或含控制字符的码拒绝。
- 价格观测只追加；作废用 `voided_at`，纠正用新记录；入库带价格时同事务写一条 `purchase` 观测。
- 单位价格按 `total_price_cents / (quantity × net_amount)` 归一化；只比较同币种、同基础单位的数据。
- 价格统计、图表口径与字段详见 [inventory/pricing.md](inventory/pricing.md)。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `inventory.read` | 查看物品与库存 | `member` |
| `inventory.write` | 新建、编辑、归档物品 | `member` |
| `inventory.stock.adjust` | 出入库、盘点调整、拆批与开封 | `member` |
| `inventory.loans` | 新建借出、归还、延期、标记丢失与取消 | `member` |
| `inventory.catalog.lookup` | 调用外部商品库查询条码；Provider 可被管理员关闭 | `member` |
| `inventory.admin` | 强制负库存、删除空物品、清除条码缓存 | `admin` |

## 6. RPC 与事件

- `inventory.items.list`：分页、按位置/类型/类别/状态/退货窗口/标签/过期筛选。
- `inventory.items.get`
- `inventory.items.create`：可带 `barcodes`，物品与条码同事务创建。
- `inventory.items.update`
- `inventory.items.archive` / `inventory.items.restore`
- `inventory.stock.adjust`：`{ itemId, lotId?, lot?, delta, reason, price?, note?, toLocationId? }`；`lotId` 为空时按 `lot { locationId?, unit?, expiresAt? }` 新建批次；`reason=move` 用 `toLocationId`；入库带价格时同事务写一条 `purchase` 价格观测。
- `inventory.lots.split`：`{ lotId, quantity }`，拆出同物品、同位置、同单位、同保质期的新批次；原批次减量。
- `inventory.lots.open`：`{ lotId, openedAt?, useWithinDays? }`；只写开封字段，不改数量；已开封时返回现值，不重复写。
- `inventory.lots.listOpened`：`{ deadlineBefore?, locationId?, limit?, cursor? }`，返回开封批次、有效期限与 `deadlineSource`。
- `inventory.items.availability`：`{ itemIds, at? }`，按批次单位分别汇总 `onHand` / `lent` / `available`（未归档、未过期批次，扣除活跃借出），不跨单位换算，供 `meals`、出门清单与借出页调用。
- `inventory.items.expiringBetween`：`{ from, to, locationId?, category? }`，返回区间内到期或开封后到期的批次。
- `inventory.items.listDeadlines`：`{ from, to, kinds?: ["warranty", "return"], locationId?, category?, limit?, cursor? }`，返回物品级保修/退货窗口；只读，供到期雷达调用。
- `inventory.usage.rate`：`{ itemId, windowDays? }`，只统计 `consume` / `spoil`，返回日均消耗、样本天数、剩余天数与 `confidence`。
- `inventory.items.listReorder`：`{ horizonDays?, locationId?, limit?, cursor? }`，按阈值与预计用完时间返回补货建议；只读。
- `inventory.loans.*`：新建/列表/归还/延期/丢失/取消；字段与规则见 [inventory/loans.md](inventory/loans.md)。
- `inventory.barcodes.resolve`：批量解析 `{ codes: [{ code, codeType? }] }`；服务端校验 GTIN、归一化并查本地映射，返回匹配、未知、非法码；不访问外网。
- `inventory.barcodes.link`：`{ itemId, code, codeType?, packQuantity?, packUnit? }`。
- `inventory.barcodes.unlink`
- `inventory.barcodes.listForItem`
- `inventory.catalog.lookup`：显式查询外部商品库，返回可编辑草稿；不落物品表。
- `inventory.catalog.providers.list`：查看可用 Provider 与启用状态。
- `inventory.prices.record` / `list` / `series` / `stats` / `void`：价格记录、原始列表、时间桶、统计与作废；口径见 [inventory/pricing.md](inventory/pricing.md)。
- `inventory.merchants.list` / `create` / `update` / `archive`：家庭内商家主数据。
- `inventory.items.listByLocation`：供 `locations` 详情页调用。
- 事件：
  - `inventory.item.changed { id, action }`
  - `inventory.barcode.changed { itemId, action }`
  - `inventory.price.changed { itemId, priceId, action }`
  - `inventory.stock.low { itemId, remaining, reorderAt }`
  - `inventory.stock.expiring { itemId, lotId, deadline, source }`
  - `inventory.lot.opened { itemId, lotId, deadline }`
  - `inventory.loan.changed { loanId, itemId, status }`
  - `inventory.stock.reorderSuggested { itemId, remaining, daysRemaining, confidence }`

不提供：

- 不提供自动下单、自动补货或自动扣减；建议与采购清单都需人确认。
- 不提供食品安全判断；开封后期限只按用户填写的天数计算。
- 不提供 `inventory.barcodes.decode`：解码在客户端，恒星不接收图像。
- 不提供「扫码即自动建物品」或「查询即入库」的隐式写入。
- 不提供全局条码目录；跨家庭不共享映射。

### 外部商品查询 Provider

```kotlin
interface ProductLookupProvider {
    val id: String
    val apiVersion: Int
    suspend fun lookup(gtin: String, ctx: LookupContext): ProductDraft?
}

data class ProductDraft(
    val name: String?,
    val brand: String?,
    val category: String?,
    val netContent: String?,
    val providerId: String,
    val fetchedAt: Long,
)
```

- 只接受已归一化的 GTIN；不接受任意文本或二维码 URL。
- 超时 5s、按 Provider 限流；失败只提示，不重试风暴。
- 返回结果不可信，只做表单预填；不返回 HTML，不下载图片，不含凭证。
- Provider 配置与凭证放 `ModuleConfig`；管理员可启用/禁用；`apiVersion` 不匹配时拒绝加载。
- v1 先内置一个可关闭的 HTTP Provider；需要更多来源时再按签名 `.hmod` 扩展。

## 7. UI（原生 / Web）

- 原生：物品列表、详情、编辑、扫码入口、批量扫码托盘、出入库表单。
- 扫码入口支持相机、蓝牙/USB 扫码枪（键盘输入）与手动输入；相机只在扫码页开启，退出即释放。
- 扫码后：已知 → 打开物品/补货；未知 → 新建物品并预填条码；归档 → 提示恢复；非法 → 说明原因。
- 批量扫码时逐条确认数量、位置、保质期；部分失败逐条提示，不做跨物品事务。
- 物品详情显示已绑定条码，可添加/解绑；显示 `code_type` 与包装规格。
- 物品详情显示「未开封 / 已开封，剩 X 天」，可拆封或拆分批次；有效期限取 `expires_at` 与开封后期限的较早值，并注明来源。
- 药品/保健品筛选：按 `category=medicine` / `supplement` 查看药箱，显示有效期、开封后期限、剩余量与存放位置；不显示剂量建议。
- 借出页：从物品详情发起借出，显示借出人、到期日、逾期与归还入口；活跃借出时物品显示「借出中」。
- 补货页显示补货点、日均消耗、预计用完与样本天数；样本不足时只显示余量与「数据不足」，不给天数。
- 状态筛选：在用 / 待退 / 在修 / 借出 / 丢失；待退显示 `return_by`，可直接跳到出门清单。
- 物品详情「价格」页（后续）：最新/最低/最高/平均、涨跌幅、折线图、商家过滤、促销标记；支持表格视图。
- 外部商品库查询必须由用户点按；展示 Provider 与数据外发说明；结果可编辑，图片不自动加载。
- Web：固定壳中的列表/详情/编辑、出入库表单；手动输入，浏览器支持时可用扫码能力；需 HTTPS 与相机授权。
- Web 不做批量导入；批量扫码 v1 以原生为主。
- 物品详情聚合：
  - 位置：`locations.node.get`
  - 说明书/照片/发票：`documents.listFor`
  - 家务/维护：`tasks.listFor`
- 位置详情中的物品数量：调 `inventory.items.listByLocation`，不由 `locations` 查表。

## 8. 数据与迁移

```sql
CREATE TABLE m_inventory_item (
  id                     TEXT PRIMARY KEY,
  name                   TEXT NOT NULL,
  kind                   TEXT NOT NULL,
  status                 TEXT NOT NULL DEFAULT 'in_use',
  category               TEXT,
  brand                  TEXT,
  model                  TEXT,
  serial                 TEXT,
  purchase_date          TEXT,
  warranty_until         TEXT,
  return_by              TEXT,
  home_location_id       TEXT,
  reorder_at             INTEGER,
  reorder_quantity       INTEGER,
  opened_use_within_days INTEGER,
  metadata_json          TEXT,
  notes                  TEXT,
  archived_at            INTEGER,
  created_at             INTEGER NOT NULL,
  updated_at             INTEGER NOT NULL
);
CREATE INDEX idx_inventory_item_status
  ON m_inventory_item(status) WHERE archived_at IS NULL;

CREATE TABLE m_inventory_lot (
  id              TEXT PRIMARY KEY,
  item_id         TEXT NOT NULL,
  location_id     TEXT,
  quantity        INTEGER NOT NULL,
  unit            TEXT NOT NULL,
  expires_at      TEXT,
  opened_at       INTEGER,
  use_within_days INTEGER,
  acquired_at     TEXT,
  price_cents     INTEGER,
  currency        TEXT,
  created_at      INTEGER NOT NULL
);
CREATE INDEX idx_inventory_lot_item ON m_inventory_lot(item_id);
CREATE INDEX idx_inventory_lot_loc  ON m_inventory_lot(location_id);
CREATE INDEX idx_inventory_lot_expires
  ON m_inventory_lot(expires_at) WHERE expires_at IS NOT NULL;
CREATE INDEX idx_inventory_lot_opened
  ON m_inventory_lot(opened_at) WHERE opened_at IS NOT NULL;

CREATE TABLE m_inventory_movement (
  id         TEXT PRIMARY KEY,
  item_id    TEXT NOT NULL,
  lot_id     TEXT,
  delta      INTEGER NOT NULL,
  reason     TEXT NOT NULL,
  actor_id   TEXT NOT NULL,
  at         INTEGER NOT NULL
);
CREATE INDEX idx_inventory_move_item ON m_inventory_movement(item_id, at);

CREATE TABLE m_inventory_barcode (
  id            TEXT PRIMARY KEY,
  item_id       TEXT NOT NULL,
  code          TEXT NOT NULL,          -- GTIN 统一 14 位；其他码去空白后原样
  raw_code      TEXT,                   -- 原始扫描值，仅用于展示
  code_type     TEXT NOT NULL,          -- gtin / code128 / qr / other
  pack_quantity INTEGER,
  pack_unit     TEXT,
  source        TEXT NOT NULL,          -- scan / manual / provider
  verified_at   INTEGER,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL
);
CREATE UNIQUE INDEX idx_inventory_barcode_code
  ON m_inventory_barcode(code) WHERE archived_at IS NULL;
CREATE INDEX idx_inventory_barcode_item ON m_inventory_barcode(item_id);

CREATE TABLE m_inventory_catalog_cache (
  code         TEXT NOT NULL,
  provider_id  TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  fetched_at   INTEGER NOT NULL,
  expires_at   INTEGER NOT NULL,
  PRIMARY KEY (code, provider_id)
);
```

- 数量用整数 + 单位；需要小数时用缩放整数（见 [04-data.md](../04-data.md)）。
- 金额用最小货币单位 + 币种；v1 可不填。
- `expires_at` / `return_by` 用 `YYYY-MM-DD`；`opened_at` 用 UTC epoch milliseconds。
- `reorder_at` / `reorder_quantity` 用与批次单位一致的整数；同一物品存在多种单位时不做低库存判断。未设置存 `NULL`，表示不参与低库存判断。
- 开封与拆批只改批次，不改物品；需要审计时写一条 `delta=0`、`reason=open` 的流水。
- `metadata_json` 只存类别相关字段；药品/保健品字段约定见 [flows/medicine-cabinet.md](../flows/medicine-cabinet.md)，不放密码、证件号、医疗正文或支付信息。
- 借出表 `m_inventory_loan` 的 DDL 与活跃唯一索引见 [inventory/loans.md](inventory/loans.md)；本卡不重复。
- 状态变更走 `items.update`；`lent` 只能由借出记录派生，置为 `disposed` 需要 `inventory.admin`。
- 条码唯一性只约束活跃映射；归档映射保留用于历史与「恢复」提示。
- `catalog_cache` 只存查询结果，不存凭证；过期或禁用 Provider 后可清理。
- 价格表 `m_inventory_merchant`、`m_inventory_price` 与回填策略见 [inventory/pricing.md](inventory/pricing.md)；本卡不重复 DDL。
- 迁移只向前；删除物品默认归档，不物理删除流水。

## 9. 验收

- 新建消耗品、耐用品；按位置查询。
- 出入库后数量正确；负库存被拒绝（管理员强制除外）。
- 未开封过期、开封后到期、两者取较早，查询与事件一致。
- 拆批后只有拆出的批次被开封，原批次仍为未开封。
- 开封不改变数量；重复开封不重复写。
- 消耗速度只计 `consume` / `spoil`；一次盘点更正不改变日均。
- 样本不足返回 `insufficient`，不显示编造的天数。
- 补货列表命中阈值或预计用完；建议数量按包装取整，不出现小数件。
- 待退物品在 `return_by` 前可查，状态不自动变化。
- 过期/低库存查询与事件正确。
- 借出后 `availability` 扣除数量，物品显示借出中；重复借出被拒绝；归还后恢复可用。
- 标记丢失后物品为 `lost`；管理员可恢复；取消错误借出记录不占用物品。
- 药品/保健品按批次管理有效期与开封期限，能筛出药箱与临期；不产生剂量建议。
- 物品级保修/退货窗口能通过 `listDeadlines` 查询，并进入到期雷达。
- 归档后不出现在默认列表，但历史流水可查。
- 物品详情能显示关联说明书与任务。
- 权限：`guest` 只读；`member` 可出入库与绑定条码；`admin` 可强制调整。
- 扫已知条码打开对应物品；扫未知条码可创建物品并绑定，重复扫不产生重复物品。
- UPC-A 与等价 EAN-13 归一到同一 GTIN；非法校验位被拒绝或明确降级为 `other`。
- 同一物品绑定多条码；一条码不能同时绑定两个活跃物品；归档后提示恢复。
- 批量扫码逐条入库，失败项可重试，已成功项不回滚。
- 离线时本地解析可用；外部查询失败不阻塞创建或入库。
- 相机帧、条码图片不上传、不落盘；审计与日志不记录完整条码内容。
- Web 无扫码能力时可用手动输入；相机不可用时不影响其他功能。
- 补货建议不自动下单、不自动入库；入库必须显式调用 `inventory.stock.adjust`。
- 归档物品不出现在低库存与补货列表。
- 价格历史（后续）：入库带价格自动生成观测；单位价可比较不同包装；不同币种/单位不混算；图表不插值。

## 10. 开放问题

- `assets` 何时拆分：出现维护计划、保修理赔、折旧等独立流程时。
- 价格历史长期留在 `inventory`，还是拆独立 `prices` / `shopping` 模块；当前先留在 `inventory`。
- 是否需要独立的商品主数据表，让同一商品对应多个库存条目；v1 用 `Item` 兼作商品与库存。
- 外部商品库 Provider 用内置 HTTP 配置还是签名 `.hmod`；v1 先留 SPI，内置一个可关闭的配置项。
- 是否支持称重商品的内嵌价格条码；v1 归入 `other`，不解析价格。
- 是否把位置/储物箱二维码也纳入同一扫码入口；当前归 `locations` 的开放问题。
- 补货阈值 v1 放物品；是否支持分类默认值与全局默认，等物品数量上来后再定。
- 消耗速度窗口是否按季节/物品类型分别配置；v1 固定 90 天，样本不足不外推。
- 借出是否支持部分归还、多物品借出单与押金；v1 只做单物品/单批次。
- 药品/保健品是否需要结构化剂量、频次、服药提醒与相互作用检查；v1 不存剂量建议，只做库存与到期。
- 是否需要把药品库存与 `health` 的处方记录双向强制关联；v1 用可选 `inventoryItemId`。
- 开封后期限是否支持按小时、冷藏/冷冻分档；v1 只做自然日。
- 是否把「上次购买时间」与价格纳入补货建议；v1 只用库存流水。
- 耐用品是否需要「多台同型号」的实例表，而不是一个 Item 数量 1。
