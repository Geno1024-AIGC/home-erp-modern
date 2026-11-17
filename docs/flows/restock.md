# 流程：快用完了才买，买回来知道放哪

> 涉及模块：`inventory`、`meals`、`tasks`（可选）、`locations`、`documents`（可选） ｜ 触发：用户打开补货页 / 低库存事件 ｜ 权限：`inventory.read`、`inventory.stock.adjust`、`meals.write`、`tasks.write`（可选）

## 1. 目标

用库存流水估算消耗速度，在「还剩一点」和「彻底用完」之间给出补货建议。建议只进采购清单，不自动下单、不自动入库；买回来由用户确认数量、价格与位置。阈值、消耗速度与建议都可解释，数据不足时明确说不足。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 批次、流水、补货点、默认位置、消耗速度 | `inventory` |
| 采购清单 | `meals` |
| 补货提醒 | `tasks` |
| 位置树 | `locations` |
| 小票、发票、货架照片 | `documents` |

本流程不建表；采购清单项与库存批次是两套数据，只在用户确认后通过 RPC 衔接。

## 3. 步骤

1. 日常取用记 `inventory.stock.adjust { reason: "consume" }`；盘点用 `adjust`，换位置用 `move`，丢弃用 `spoil`，收到退货用 `return`。区分 `reason` 是估算的前提。
2. `inventory.usage.rate { itemId, windowDays: 90 }`：只统计 `consume` 与 `spoil`，返回 `perDayMilli`（每日千分之一单位）、`sampleDays`、`daysRemaining`、`confidence`、`unit`。有效样本少于 14 天或少于 3 次事件时返回 `insufficient`。
3. `inventory.items.listReorder { horizonDays: 14, locationId? }`：命中「剩余 ≤ `reorder_at`」或「`daysRemaining` ≤ horizon」的物品；跨位置合并总数，但保留各位置余量，便于决定放哪。
4. 建议数量：`ceil(perDay × (horizonDays + leadDays) − remaining)`，再按包装向上取整；`leadDays` v1 固定 3 天，可后续改成按物品配置。建议不写库存。
5. 用户确认 → `meals.grocery.add { inventoryItemId, name, quantity, unit }`；只写清单，不下单。来源记为清单项的备注或后续 `source` 字段，v1 不新增枚举。
6. 采购时 `meals.grocery.check` 只表示「已买」，不代表已入库；`clearChecked` 前提示「还有 N 项未入库」。
7. 入库：`inventory.stock.adjust { reason: "purchase", delta, lot: { locationId, expiresAt?, unit? }, price? }`，默认位置取 `Item.home_location_id`；带价格时同事务写一条 `purchase` 价格观测。多件、多位置、多批次可分多次入库。
8. 可选提醒：`inventory.stock.low` 或 `inventory.stock.reorderSuggested` 只用于刷新补货页；需要推送时由 UI 显式创建 `tasks.occurrences.create` 或周期模板，不每次消耗都提醒。
9. 结束：剩余量回到 `reorder_at` 以上，或用户手动把物品移出补货列表（本次跳过，不改阈值）。

## 4. 失败与补偿

- **单位不一致**：同一物品存在多种单位时不做低库存判断与速率估算，只显示各批次余量；包装换算用 `ItemBarcode.pack_quantity` 作为录入默认值，不在估算时猜。
- **消耗不规律或季节性强**：`confidence` 标 `low`，只显示余量与阈值，不给天数。
- **盘点更正被当消耗**：`adjust` 不计入速率；一次大更正因为 `reason` 不同，不会把预测带偏。
- **忘记记录消耗**：余量偏高、建议偏晚；用户可手动「校正余量」，写 `reason: "adjust"`。
- **批量采购未拆包**：入库数量 = 包装数 × `pack_quantity`，单位统一后再写批次。
- **物品归档或没有 `reorder_at`**：不产生建议；归档物品不出现在列表。
- **`meals` 未安装**：建议只读，可复制文本，不创建采购清单。
- **`tasks` 未安装**：不提醒，补货页仍可用。
- **负库存**：默认拒绝；管理员强制调整必须填原因，且不计入消耗。
- **价格缺失**：允许入库不带价；不因为没有价格而阻塞入库。

## 5. 事件

- `inventory.stock.low { itemId, remaining, reorderAt }`
- `inventory.stock.reorderSuggested { itemId, remaining, daysRemaining, confidence }`
- `meals.grocery.changed { id, action }`
- `inventory.price.changed`（入库带价格时）
- 不跨模块事务；清单、库存、价格各自提交，失败项由用户重试。

## 6. 验收

- 连续 3 周每周消耗 1 盒，剩余量低于补货点时出现在补货列表，并给出合理的预计天数。
- 一次盘点大更正不改变日均消耗。
- 样本不足返回 `insufficient`，页面不显示编造的天数。
- 建议数量按包装向上取整，不出现小数件。
- 勾选采购清单不改变库存；入库才改，且带价格时写价格观测。
- 入库默认落到 `home_location_id`，用户可改；多位置分批正确。
- 未装 `meals` 或 `tasks` 时仍可用。
- 归档物品与未设阈值的物品不出现在建议里。
- 建议不触发任何外部下单或自动入库。

## 7. 开放问题

- 阈值放物品、分类还是全局默认；v1 只做物品级，分类默认值后续再评估。
- 是否按季节或月份调整消耗速度；v1 固定 90 天窗口。
- 是否支持订阅制商品与固定周期补货；v1 只按库存流水。
- 是否把「上次购买时间/价格」纳入建议；v1 只用流水与当前余量。
- 是否合并多个物品为一次采购行程；v1 只做清单。
- 提前期 `leadDays` 是否按物品/商家配置；v1 固定 3 天。
