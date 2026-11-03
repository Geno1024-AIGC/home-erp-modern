# 模块：物品（inventory）

> 状态：draft ｜ 类型：business ｜ 依赖：`locations`、`documents`（可选） ｜ 拥有：`m_inventory_*`

## 1. 目的与非目标

**目的**：记录家里的物品与库存：消耗品数量、保质期、出入库；耐用品型号、序列号、保修信息。

**非目标**：

- 不做采购、供应商、发票与账务。
- 不做维护计划；v1 由 `tasks` 建提醒，后续再评估 `assets`。
- 不做条形码/二维码扫描（后续）。
- 不存说明书文件；文件与关联归 `documents`。

## 2. 用户场景

- 新增一箱牛奶：数量、单位、保质期、位置。
- 取用/补货：记录出入库，查看当前数量。
- 新增一台电视：品牌、型号、序列号、购买日期、保修期、位置。
- 查看某个柜子里有哪些物品。
- 查看即将过期的物品、低库存物品。
- 在物品详情打开说明书、关联家务/维护任务。

## 3. 边界与依赖

- 拥有物品主数据、批次、库存流水。
- 只引用 `location_id`；不建跨模块外键。
- 说明书/照片/发票由 `documents` 拥有；`inventory` 调 `documents` RPC 展示。
- 家务/维护提醒由 `tasks` 拥有；`inventory` 不复制任务状态。
- v1 不单独建 `assets`；耐用品是 `Item.kind = durable`。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Item` | `id`、`name`、`kind`、`category`、`brand`、`model`、`serial`、`purchase_date`、`warranty_until`、`notes`、`archived_at` |
| `StockLot` | `id`、`item_id`、`location_id`、`quantity`、`unit`、`expires_at`、`acquired_at`、`price_cents`、`currency` |
| `StockMovement` | `id`、`item_id`、`lot_id`、`delta`、`reason`、`actor_id`、`at` |

`Item.kind`：`consumable`（消耗品，按批次管理数量/保质期）、`durable`（耐用品，通常数量 1，重型号/序列号/保修）。

规则：

- `StockMovement` 只追加，不修改历史；当前数量由批次汇总。
- 默认不允许负库存；管理员可强制调整，必须填写原因。
- 消耗品按 `StockLot` 管理；耐用品可以只有一个批次，`quantity = 1`。
- 序列号在 `kind=durable` 下建议唯一；不强制全局唯一，允许同型号多台。
- 归档物品不删除流水；历史仍可查。
- 过期/低库存只是查询与事件，不自动改数量。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `inventory.read` | 查看物品与库存 | `member` |
| `inventory.write` | 新建、编辑、归档物品 | `member` |
| `inventory.stock.adjust` | 出入库、盘点调整 | `member` |
| `inventory.admin` | 强制负库存、删除空物品 | `admin` |

## 6. RPC 与事件

- `inventory.items.list`：分页、按位置/类型/标签/过期筛选。
- `inventory.items.get`
- `inventory.items.create`
- `inventory.items.update`
- `inventory.items.archive` / `inventory.items.restore`
- `inventory.stock.adjust`：`{ itemId, lotId?, delta, reason }`。
- `inventory.items.listByLocation`：供 `locations` 详情页调用。
- 事件：
  - `inventory.item.changed { id, action }`
  - `inventory.stock.low { itemId, remaining }`
  - `inventory.stock.expiring { itemId, lotId, expiresAt }`

## 7. UI（原生 / Web）

- 原生：物品列表、详情、编辑、扫码入口（后续）、出入库表单。
- Web：固定壳中的列表/详情/编辑、出入库表单；不做批量导入（后续）。
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
```

- 数量用整数 + 单位；需要小数时用缩放整数（见 [04-data.md](../04-data.md)）。
- 金额用最小货币单位 + 币种；v1 可不填。
- 迁移只向前；删除物品默认归档，不物理删除流水。

## 9. 验收

- 新建消耗品、耐用品；按位置查询。
- 出入库后数量正确；负库存被拒绝（管理员强制除外）。
- 过期/低库存查询与事件正确。
- 归档后不出现在默认列表，但历史流水可查。
- 物品详情能显示关联说明书与任务。
- 权限：`guest` 只读；`member` 可出入库；`admin` 可强制调整。

## 10. 开放问题

- `assets` 何时拆分：出现维护计划、保修理赔、折旧等独立流程时。
- 是否支持条形码/二维码与扫码入库。
- 是否记录价格与购买渠道；v1 只留字段，不做账务。
- 低库存阈值放物品、分类还是全局设置。
- 耐用品是否需要「多台同型号」的实例表，而不是一个 Item 数量 1。
