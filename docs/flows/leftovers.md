# 流程：开封与剩菜，先吃哪一个

> 涉及模块：`inventory`、`meals`、`tasks`（可选）、`documents`（可选） ｜ 触发：拆封一包食材 / 一顿饭没吃完 ｜ 权限：`inventory.stock.adjust`、`meals.write`、`tasks.write`

## 1. 目标

把三种「快过期」分开：未开封保质期、开封后的可用期、熟食剩菜。三者都只做提醒，不自动扣减、不自动丢弃，也不做食品安全判断。用户打开一个列表，就能看到今天该先吃哪一样。

**归属结论**：开封后的包装商品属于 `inventory`，因为它是批次的一部分，有条码、价格、位置与补货；熟食剩菜属于 `meals`，因为它是烹饪产物，有菜谱、做饭记录和口味。把剩菜塞进 `StockLot` 会制造没有条码、没有价格的假物品，污染补货与价格统计。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 未开封保质期、开封时间、开封后可用天数 | `inventory` |
| 做饭记录、剩菜（菜名、位置、`eat_by`、状态） | `meals` |
| 到期提醒 | `tasks` |
| 包装、剩菜照片 | `documents` |
| 冰箱温度（若以后有传感器） | `smart_home`；v1 不参与判定 |

本流程不拥有任何表，也不把剩菜复制成库存物品。

## 3. 步骤

1. 拆封：选批次 → `inventory.lots.open { lotId, openedAt?, useWithinDays? }`。`useWithinDays` 缺省取 `Item.opened_use_within_days`；两者都没有时只记开封时间，不猜期限。
2. 一包多件：先 `inventory.lots.split { lotId, quantity }` 拆出要开的那份，再对拆出的批次 `open`；未拆的批次保持未开封。
3. 有效期限 = `min(expires_at, opened_at + use_within_days)`；查询返回 `deadline` 与 `deadlineSource`（`expires_at` / `opened`）。
4. 做饭：`meals.cookLogs.create` 记录这顿；如果多做了一份，`meals.leftovers.add { cookLogId, dishName, storedAt?, locationId?, eatBy? }`，`eatBy` 缺省为 `storedAt + 3 天`。
5. 统一视图：客户端并行调 `inventory.lots.listOpened { deadlineBefore }`、`inventory.items.expiringBetween { from: 今天, to: 今天 + 7 天 }`、`meals.leftovers.list { status: ["stored"], eatByBefore: 今天 + 7 天 }`，按有效期限升序合并，标「开封」「未开封」「剩菜」。
6. 吃掉：剩菜 `meals.leftovers.mark { id, status: "eaten" }`；开封批次按实际用量 `inventory.stock.adjust { reason: "consume" }`，由用户确认，不自动扣。
7. 扔掉：`meals.leftovers.mark { id, status: "discarded", reason }` 或 `inventory.stock.adjust { reason: "spoil", note }`；丢弃写原因，用于回看浪费，不做评价。
8. 提醒（可选）：创建剩菜或开封时调 `tasks.occurrences.create { title, dueAt: eatBy, subjectType, subjectId, dedupeKey }`；`dedupeKey` 用「类型 + 记录 ID」，重试不重复。
9. 到期未处理：只在列表标红并计入「今天该吃」；不改状态、不自动丢、不生成负库存。
10. 冷冻：`meals.leftovers.mark { status: "frozen" }` 停提醒；解冻时用户重设 `eat_by`，系统不计算保质期。

## 4. 失败与补偿

- **忘记录开封**：只按 `expires_at`，标「未记录开封」；不根据购买日期推断。
- **批次数量为 0 或已归档**：拒绝开封，提示先入库或恢复。
- **已过期批次被开封**：允许记录，但标 `expired` 并保留在列表；不隐藏。
- **拆批数量非法**：必须大于 0 且小于原批次数量；等于原数量时拒绝，提示直接开封。
- **时区与夏令时**：`use_within_days` 按家庭时区的自然日计算；`opened_at` 存 UTC。
- **剩菜没有做饭记录**：允许 `cookLogId` 为空（补记），`dishName` 必填并做快照。
- **用户把 `eatBy` 往后改**：允许，记录修改时间；已有提醒按 `dedupeKey` 更新或由用户重建，不静默保留旧日期。
- **`tasks` 未安装**：不生成提醒，列表仍按期限排序。
- **照片**：只做展示，不做图像识别，不根据照片判断是否变质。

## 5. 事件

- `inventory.lot.opened { itemId, lotId, deadline }`
- `inventory.stock.expiring { itemId, lotId, deadline, source }`
- `meals.leftover.changed { id, action, eatBy }`
- `tasks.occurrence.created`（若生成提醒）
- 不跨模块事务：开封、剩菜、提醒各自提交，失败时由用户重试。

## 6. 验收

- 开封后按 `opened_at + use_within_days` 计算；与 `expires_at` 取较早值，并返回来源。
- 拆批只影响拆出的批次，原批次仍为未开封。
- 未记录开封的批次不会被当成已开封。
- 剩菜默认 3 天、可改；到期只标红，不自动丢弃。
- 吃掉/扔掉后从默认列表消失，历史仍可查。
- 不产生库存扣减，除非用户显式确认。
- 同一剩菜/批次只生成一条提醒；重复调用不重复。
- `tasks` 缺失时流程仍可完成，只是没有提醒。
- 日志不含剩菜备注与照片。

## 7. 开放问题

- 冷藏、冷冻、室温是否分档，开封后期限是否支持按小时；v1 只做自然日与统一 3 天。
- 是否需要浪费统计与成本估算；v1 只记状态与原因。
- 是否需要接冰箱温度传感器参与判定；v1 不接，也不做安全结论。
- 剩菜是否应作为 `CookLog` 的子记录，而不是独立实体；v1 独立，便于单独设期限。
- 是否支持「分装后冷冻」一次生成多份剩菜。
