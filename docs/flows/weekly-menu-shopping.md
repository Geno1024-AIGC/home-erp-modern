# 流程：一周菜单变成一张采购清单

> 涉及模块：`meals`、`inventory`、`tasks`（可选）、`documents`（可选） ｜ 触发：用户排完一周菜单 / 点「生成本周采购清单」 ｜ 权限：`meals.read`、`meals.write`、`inventory.read`（可选）、`tasks.write`（可选）

## 1. 目标

把「这周吃什么」直接变成「这周买什么」：按菜单里的菜谱和份数汇总食材，扣掉现有可用库存，只把缺的写进采购清单。预览与写入分开，重复生成不重复，已买/手动项不被覆盖；不自动下单、不自动扣库存。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 菜谱、食材、菜单计划、采购清单 | `meals` |
| 批次数量、过期/开封期限、借出占用 | `inventory` |
| 买菜任务/提醒 | `tasks` |
| 菜谱照片、小票 | `documents` |

`meals` 不查库存表；可用量通过 `inventory.items.availability` 读取。采购清单仍归 `meals`，库存写回仍归 `inventory`。

## 3. 步骤

1. 用户排好 `from`–`to` 的菜单，或先调用 `meals.plans.list { from, to }` 查看缺口。
2. 调 `meals.plans.groceryPreview { from, to, servingsScale? }`。服务端按每条菜单的 `servings / recipe.servings` 缩放食材，汇总同名同单位项。
3. 预览返回每行：`name`、`inventoryItemId?`、`required`、`unit`、`available`、`missing`、`unitMismatch`、`sources`（来自哪些菜谱/餐次）。
4. 以请求者身份调 `inventory.items.availability { itemIds }`，扣除已过期批次与活跃借出；没有 `inventory.read` 时 `stockUnknown`，不把库存当成 0。
5. 用户勾选要买的行，确认 `mode`：
   - `missing`：只买缺料；
   - `all`：按菜单全量买，忽略现有库存。
6. 调 `meals.plans.generateGrocery { from, to, servingsScale?, mode, replaceGenerated? }`。按 `inventoryItemId` 或「名称 + 单位」精确合并，写入 `source=menu`、`source_ref=menu:<from>:<to>` 的清单项。
7. 重复生成只增补缺失项；已勾选项、手动项、其他来源项不覆盖。`replaceGenerated=true` 时只清理未被勾选的 `source=menu` 旧项。
8. 采购时 `meals.grocery.check` 只表示「已买」，不代表已入库。
9. 可选：`tasks.occurrences.create { title: "买菜", dueAt, dedupeKey }` 生成一次提醒；不每次改菜单都提醒。
10. 买回来后由用户确认入库：`inventory.stock.adjust { reason: "purchase", delta, lot: { locationId, expiresAt?, unit? }, price? }`，默认位置取 `Item.home_location_id`。不做自动入库。

## 4. 失败与补偿

- **菜谱没有份数**：按 1 份处理并标 `servingsAssumed`；不静默放大。
- **单位不一致**：不做密度/体积换算，单独成行并标 `unitMismatch`；由用户决定是否购买。
- **自由文本食材**：只按名称 + 单位精确合并；不做别名/模糊匹配。
- **库存不可见**：标 `stockUnknown`，缺料显示「未知」而不是「全部要买」。
- **重复生成**：按 `source_ref` 与名称/单位去重；已勾选项不重置。
- **菜单临时改动**：重新预览；用户可选择保留旧生成项或替换未勾选项。
- **`inventory` 未安装**：仍可生成全量采购清单，标「无库存信息」。
- **`tasks` 未安装**：不生成提醒，清单仍可用。
- **离线/恒星不可达**：v1 在线优先，直接报错；不返回过期库存冒充实时。

## 5. 事件

- 读取时消费：`meals.plan.changed`、`inventory.item.changed`、`inventory.loan.changed`（缓存失效）。
- 写入时产生：`meals.grocery.changed { id, action }`。
- 入库时产生：`inventory.stock.adjust` 对应流水与 `inventory.price.changed`（带价格时）。
- 不新增事件；不跨模块事务。

## 6. 验收

- 排 7 天菜单后，预览能汇总食材并按份数缩放。
- 现有库存足够时缺料为 0；库存不足时只生成缺料，不重复购买。
- 同一食材来自多道菜时合并为一行，并显示来源。
- 重复生成不重复；已勾选与手动项不被覆盖。
- 单位不一致只提示，不强行换算；自由文本不做模糊匹配。
- `inventory` 缺失或无权限时降级为全量清单，并明确标未知。
- 加购只写采购清单，不改库存、不下单；入库必须显式确认。
- 日志不含菜名、库存数量与价格明细。

## 7. 开放问题

- 是否需要「家里常备」清单，默认不进采购；v1 只按显式库存与手动排除。
- 是否支持按商店拆多张清单；v1 只有一张共享清单。
- 是否需要单位换算表（g/kg、ml/L）；v1 不换算，只提示。
- 是否需要把剩菜/开封食材优先消耗后自动减少采购量；v1 只按库存可用量。
- 菜单份数缩放是否要支持按人数/按餐次不同；v1 用菜单条目份数 / 菜谱份数。
- 是否需要采购清单与补货建议合并展示；v1 两个入口都写同一份清单。
