# inventory · 价格历史与趋势（后续）

> 状态：planned（v1 不实现）｜ 归属：`inventory` ｜ 依赖：`inventory`、`locations`（可选）、`documents`（可选）

## 1. 目的与非目标

**目的**：记录同一物品在不同时间、不同商家的价格，换算成可比的单位价格，用折线图看涨跌；数据来自入库实付价、货架/网价手动记录，后续再接小票导入。

**非目标**：

- 不做自动爬价、比价、历史最低价提醒、预算与账务。
- 不做汇率换算；不同币种不混算。
- 不做价格预测、推荐购买时机。
- 不抓取外部商品图；小票/照片归 `documents`。
- 不把价格写进库存流水；库存数量与价格观测分离。

## 2. 用户场景

- 每次补货时顺手填实付价，之后看某件商品过去半年的价格曲线。
- 在超市看到货架价但没买，也先记一条，用于比较不同商家。
- 同一商品 500g 和 1kg 包装不同，按每 100g 价格比较哪个更划算。
- 过滤某个商家，看看它最近是不是普遍涨价。
- 标记促销价，图表上区分原价与实付价。
- 录错了价格：作废并补一条，不直接改历史。
- 从物品详情进入价格页，看到最新、最低、最高、平均与涨跌幅。

## 3. 数据所有权与边界

- `inventory` 拥有 `PriceObservation` 与 `Merchant`；价格只追加，不作静默修改。
- 价格观测引用 `item_id`、可选 `merchant_id`、`lot_id`；不建跨模块外键。
- 商家是家庭内主数据，不写死品牌；线下门店可引用 `location_id`，但不复制位置数据。
- 入库时可选带价格，`inventory.stock.adjust` 在同一事务写库存流水与价格观测。
- 小票、货架照片、发票归 `documents`；价格模块只存数值与备注。
- 图表是只读聚合，不在图表里改价；写操作走价格 RPC。
- 同一物品的价格历史不因物品归档而删除；归档后仍可查看。
- 价格历史属于消费行为数据；日志只记操作与计数，不记完整价格明细。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Merchant` | `id`、`name`、`kind`、`location_id`、`notes`、`archived_at` |
| `PriceObservation` | `id`、`item_id`、`observed_at`、`total_price_cents`、`currency`、`quantity`、`net_amount`、`net_unit`、`unit_price_micros`、`regular_price_cents`、`promotion`、`kind`、`merchant_id`、`lot_id`、`source`、`note`、`idempotency_key`、`supersedes_id`、`voided_at`、`void_reason`、`created_by` |

`Merchant.kind`：`supermarket`、`online`、`market`、`other`。

`PriceObservation.kind`：`purchase`（实付）、`shelf`（货架价）、`online`（网价）、`other`。

`PriceObservation.source`：`manual`、`stock_in`、`import`、`receipt`。

`net_unit`：`g`、`ml`、`piece`。

规则：

- `total_price_cents` 是本次观测为 `quantity` 个包装支付/看到的总价；`quantity >= 1`。
- `net_amount` 是**每个包装**的净含量，整数；重量用 g，容量用 ml，计件用 `piece` 且 `net_amount = 1`。
- `package_price_cents = round(total_price_cents / quantity)`，只用于展示，不单独存。
- `unit_price_micros = round(total_price_cents × 1_000_000 / (quantity × net_amount))`，表示每 1 个基础单位的价格（货币最小单位的百万分之一）；`net_amount` 为空时为空。
- 只比较相同 `currency` 与相同 `net_unit` 的数据；不自动换算币种或单位。
- `regular_price_cents` 可选；小于 `total_price_cents` 时忽略，不反推折扣。
- `promotion` 是自由文本（如「第二件半价」），v1 不结构化。
- `kind=purchase` 才代表实际购买；`shelf` / `online` 只代表观测。
- 价格观测只追加；作废用 `voided_at` + `void_reason`，纠正用新记录 + `supersedes_id` 指向旧记录。
- 同一物品的 `idempotency_key` 唯一；自动入库观测用 `stock_movement:<movementId>`。
- 商家归档不删除历史；历史仍显示归档前名称。
- 不自动从 `StockLot.price_cents` 推断所有历史；迁移时可一次性回填有价格的批次。

## 5. 统计口径

- `latest`：窗口内 `observed_at` 最新的未作废记录；同一时间按 `created_at` 取最新。
- `min` / `max` / `avg`：窗口内未作废记录，按所选指标（包装价或单位价）；`avg` 基于原始观测，不对桶均值再平均。
- `change_bp`：`latest` 相对同窗口内上一条记录的基点变化，`round((latest - previous) × 10000 / previous)`；没有上一条时为空。
- 不做插值；缺失日期就是缺口，折线不补点。
- 促销默认计入统计；图表可切换「排除促销」。
- 极端值不自动剔除；用户可作废错误记录。
- 时间桶按家庭时区切分：`day`、`week`、`month`；区间过长时自动升粒度。
- 图表最多返回 500 个桶；原始点用分页列表查看。

## 6. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `inventory.read` | 查看价格历史、统计与图表 | `member` |
| `inventory.write` | 记录价格、管理商家 | `member` |
| `inventory.admin` | 作废价格、清理商家、批量导入 | `admin` |

- 价格历史不单独设更细权限；能看物品就能看价格。
- 作废不物理删除；管理员可导出后清理，但需二次确认。

## 7. RPC 与事件

- `inventory.prices.record`：`{ itemId, observedAt?, totalPriceCents, currency, quantity?, netAmount?, netUnit?, regularPriceCents?, promotion?, kind, merchantId?, note?, idempotencyKey? }`。
- `inventory.prices.list`：按物品、时间、商家、类型分页；默认不含作废记录。
- `inventory.prices.series`：`{ itemId, from, to, granularity, metric: package|unit, currency, merchantId?, includePromotions? }`，返回时间桶。
- `inventory.prices.stats`：`{ itemId, from, to, currency, merchantId?, includePromotions? }`，返回 latest/min/max/avg/change。
- `inventory.prices.void`：`{ priceId, reason }`；不删除，只作废。
- `inventory.merchants.list` / `create` / `update` / `archive`
- `inventory.stock.adjust` 扩展可选 `price`：`{ totalPriceCents, currency, quantity?, netAmount?, netUnit?, regularPriceCents?, promotion?, merchantId? }`；入库为正向且带价格时，同事务写 `kind=purchase`、`source=stock_in` 的观测。
- 一次 `stock.adjust` 最多生成一条观测；多批次入库按行调用，避免一条观测对应多个批次。
- `price.quantity` 默认等于 `delta`；若显式提供则必须等于 `delta`，净含量按单个库存单位填写。
- `delta <= 0` 时拒绝 `price`，不把出库当购买。
- 事件：
  - `inventory.price.changed { itemId, priceId, action }`
  - `inventory.price.voided { itemId, priceId }`

不提供：

- 不提供 `inventory.prices.scrape` 或后台抓价。
- 不提供跨币种聚合；币种不一致时要求筛选。
- 不提供「按价格自动改库存」或「自动下单」。

## 8. UI 与图表

- 物品详情增加「价格」页：最新价、最低/最高/平均、涨跌幅、样本数。
- 折线图：x 轴时间，y 轴价格；切换「包装价 / 单位价」；单位价仅在 `net_amount` 存在时可用。
- 区间：1M / 3M / 6M / 1Y / 全部；粒度为日/周/月，自动选择。
- 过滤：商家、价格类型（实付/货架/网价）、是否含促销。
- 促销点用不同标记；单位价图显示「每 100g / 每 100ml / 每个」。
- 只有一个点时显示点与「数据不足」，不画假趋势。
- 缺失区间断开，不插值；桶聚合显示平均值线与最小/最大区间。
- 币种或单位不一致：提示筛选，不做换算。
- 提供表格视图与 CSV 导出（后续）；图表颜色不是唯一编码。
- 原生用 Compose Canvas，Web 用 SVG；不引入第三方图表库。
- 图表不写回数据；录入/作废走表单与 RPC。

## 9. 数据与迁移

```sql
CREATE TABLE m_inventory_merchant (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  kind        TEXT,
  location_id TEXT,
  notes       TEXT,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);

CREATE TABLE m_inventory_price (
  id                  TEXT PRIMARY KEY,
  item_id             TEXT NOT NULL,
  observed_at         INTEGER NOT NULL,
  total_price_cents   INTEGER NOT NULL,
  currency            TEXT NOT NULL,
  quantity            INTEGER NOT NULL DEFAULT 1,
  net_amount          INTEGER,
  net_unit            TEXT,
  unit_price_micros   INTEGER,
  regular_price_cents INTEGER,
  promotion           TEXT,
  kind                TEXT NOT NULL,
  merchant_id         TEXT,
  lot_id              TEXT,
  source              TEXT NOT NULL,
  note                TEXT,
  idempotency_key     TEXT,
  supersedes_id       TEXT,
  voided_at           INTEGER,
  void_reason         TEXT,
  created_by          TEXT NOT NULL,
  created_at          INTEGER NOT NULL
);
CREATE INDEX idx_inventory_price_item ON m_inventory_price(item_id, observed_at);
CREATE INDEX idx_inventory_price_merchant ON m_inventory_price(merchant_id, observed_at);
CREATE UNIQUE INDEX idx_inventory_price_idem
  ON m_inventory_price(item_id, idempotency_key)
  WHERE idempotency_key IS NOT NULL;
```

- `observed_at` 用 UTC epoch milliseconds；支持补录过去日期；展示按家庭时区。
- `currency` 用 ISO 4217；不存符号。
- `unit_price_micros` 为派生值，写入时计算；修改净含量/数量只能通过作废重录。
- 迁移回填：为有 `price_cents` 的 `m_inventory_lot` 建一条 `source=import`、`lot_id` 对应的观测，`idempotency_key=lot:<lotId>`；已存在则跳过。
- 价格只追加；作废保留行；不物理删除。
- 索引覆盖物品时间线与商家过滤；不做物化聚合。

## 10. 验收

- 记录 500g 19.80 与 1kg 35.00，单位价分别为每 100g 3.96 与 3.50，图表可比较。
- 入库带价格时自动生成一条 `purchase` 观测；重试不重复。
- 货架价不改变库存，但进入价格曲线。
- 切换包装价/单位价、商家过滤、促销排除均正确。
- 不同币种不混算；不同 `net_unit` 不混画。
- 缺失区间断开；单点显示「数据不足」。
- 作废后图表不再计入，但历史列表仍可查。
- 归档物品仍可查看历史价格。
- 5,000 条观测按桶聚合后不超过 500 个点，图表可用。
- 图表与表格内容一致；颜色不是唯一信息载体。
- 日志不含完整价格明细；导出/批量导入仅管理员。

## 11. 开放问题

- 价格历史长期留在 `inventory`，还是拆独立 `prices` / `shopping` 模块；当前先留在 `inventory`。
- 是否接小票 OCR / 电商订单导入；v1 只做手动与入库。
- 是否需要价格提醒、预算、历史最低价通知；当前不做。
- 是否需要跨币种换算；当前不做。
- 统计优先用「最后一次价格」还是「加权平均」；当前展示两者，涨跌以最后一次为准。
- 运费、押金、折扣是否单独建模；v1 只记实付总价。
- 同一商品多商家时，是否维护「常买商家」与默认商家。
