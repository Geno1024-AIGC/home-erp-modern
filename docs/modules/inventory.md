# 模块：物品（inventory）

> 状态：draft ｜ 类型：business ｜ 依赖：`locations`、`documents`（可选） ｜ 拥有：`m_inventory_*`

## 1. 目的与非目标

**目的**：记录家里的物品与库存：消耗品数量、保质期、出入库；耐用品型号、序列号、保修信息；支持扫商品条码快速录入与补货；价格历史与趋势见 [inventory/pricing.md](inventory/pricing.md)。

**非目标**：

- 不做采购、供应商、发票与账务。
- 不做维护计划；v1 由 `tasks` 建提醒，后续再评估 `assets`。
- 不做服务端图像解码、商品比价与价格抓取；扫码解码在卫星/浏览器完成。
- 不存条码图片、相机帧或商品图；照片归 `documents`。
- 不建全局商品主数据；条码↔物品映射只在当前家庭内有效。
- 不自动从外部商品库创建物品；查询结果只作草稿，必须由用户确认。
- v1 不做自动抓价、比价、价格预测、预算与汇率换算；价格历史设计见 [inventory/pricing.md](inventory/pricing.md)。

## 2. 用户场景

- 新增一箱牛奶：数量、单位、保质期、位置。
- 取用/补货：记录出入库，查看当前数量。
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
| `Item` | `id`、`name`、`kind`、`category`、`brand`、`model`、`serial`、`purchase_date`、`warranty_until`、`notes`、`archived_at` |
| `StockLot` | `id`、`item_id`、`location_id`、`quantity`、`unit`、`expires_at`、`acquired_at`、`price_cents`、`currency` |
| `StockMovement` | `id`、`item_id`、`lot_id`、`delta`、`reason`、`actor_id`、`at` |
| `ItemBarcode` | `id`、`item_id`、`code`、`raw_code`、`code_type`、`pack_quantity`、`pack_unit`、`source`、`verified_at`、`archived_at` |
| `CatalogCache` | `code`、`provider_id`、`payload_json`、`fetched_at`、`expires_at` |
| `Merchant` | `id`、`name`、`kind`、`location_id`、`archived_at` |
| `PriceObservation` | `id`、`item_id`、`observed_at`、`total_price_cents`、`currency`、`quantity`、`net_amount`、`net_unit`、`unit_price_micros`、`kind`、`merchant_id`、`voided_at` |

`Item.kind`：`consumable`（消耗品，按批次管理数量/保质期）、`durable`（耐用品，通常数量 1，重型号/序列号/保修）。

`code_type`：`gtin`（EAN-8/13、UPC-A/E、GTIN-14 统一归入）、`code128`、`qr`、`other`。

规则：

- `StockMovement` 只追加，不修改历史；当前数量由批次汇总。
- 默认不允许负库存；管理员可强制调整，必须填写原因。
- 消耗品按 `StockLot` 管理；耐用品可以只有一个批次，`quantity = 1`。
- 序列号在 `kind=durable` 下建议唯一；不强制全局唯一，允许同型号多台。
- 归档物品不删除流水；历史仍可查。
- 过期/低库存只是查询与事件，不自动改数量。
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
| `inventory.stock.adjust` | 出入库、盘点调整 | `member` |
| `inventory.catalog.lookup` | 调用外部商品库查询条码；Provider 可被管理员关闭 | `member` |
| `inventory.admin` | 强制负库存、删除空物品、清除条码缓存 | `admin` |

## 6. RPC 与事件

- `inventory.items.list`：分页、按位置/类型/标签/过期筛选。
- `inventory.items.get`
- `inventory.items.create`：可带 `barcodes`，物品与条码同事务创建。
- `inventory.items.update`
- `inventory.items.archive` / `inventory.items.restore`
- `inventory.stock.adjust`：`{ itemId, lotId?, delta, reason, price? }`；入库带价格时同事务写一条 `purchase` 价格观测。
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
  - `inventory.stock.low { itemId, remaining }`
  - `inventory.stock.expiring { itemId, lotId, expiresAt }`

不提供：

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
  id             TEXT PRIMARY KEY,
  name           TEXT NOT NULL,
  kind           TEXT NOT NULL,
  category       TEXT,
  brand          TEXT,
  model          TEXT,
  serial         TEXT,
  purchase_date  TEXT,
  warranty_until TEXT,
  notes          TEXT,
  archived_at    INTEGER,
  created_at     INTEGER NOT NULL,
  updated_at     INTEGER NOT NULL
);

CREATE TABLE m_inventory_lot (
  id          TEXT PRIMARY KEY,
  item_id     TEXT NOT NULL,
  location_id TEXT,
  quantity    INTEGER NOT NULL,
  unit        TEXT NOT NULL,
  expires_at  TEXT,
  acquired_at TEXT,
  price_cents INTEGER,
  currency    TEXT,
  created_at  INTEGER NOT NULL
);
CREATE INDEX idx_inventory_lot_item ON m_inventory_lot(item_id);
CREATE INDEX idx_inventory_lot_loc  ON m_inventory_lot(location_id);

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
- 条码唯一性只约束活跃映射；归档映射保留用于历史与「恢复」提示。
- `catalog_cache` 只存查询结果，不存凭证；过期或禁用 Provider 后可清理。
- 价格表 `m_inventory_merchant`、`m_inventory_price` 与回填策略见 [inventory/pricing.md](inventory/pricing.md)；本卡不重复 DDL。
- 迁移只向前；删除物品默认归档，不物理删除流水。

## 9. 验收

- 新建消耗品、耐用品；按位置查询。
- 出入库后数量正确；负库存被拒绝（管理员强制除外）。
- 过期/低库存查询与事件正确。
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
- 价格历史（后续）：入库带价格自动生成观测；单位价可比较不同包装；不同币种/单位不混算；图表不插值。

## 10. 开放问题

- `assets` 何时拆分：出现维护计划、保修理赔、折旧等独立流程时。
- 价格历史长期留在 `inventory`，还是拆独立 `prices` / `shopping` 模块；当前先留在 `inventory`。
- 是否需要独立的商品主数据表，让同一商品对应多个库存条目；v1 用 `Item` 兼作商品与库存。
- 外部商品库 Provider 用内置 HTTP 配置还是签名 `.hmod`；v1 先留 SPI，内置一个可关闭的配置项。
- 是否支持称重商品的内嵌价格条码；v1 归入 `other`，不解析价格。
- 是否把位置/储物箱二维码也纳入同一扫码入口；当前归 `locations` 的开放问题。
- 低库存阈值放物品、分类还是全局设置。
- 耐用品是否需要「多台同型号」的实例表，而不是一个 Item 数量 1。
