# 流程：到期雷达，把 90 天内要处理的事排成一屏

> 涉及模块：`identity`、`achievements`、`inventory`、`meals`、`bills`、`tasks`（可选） ｜ 触发：用户打开「到期雷达」/ 首页提醒 ｜ 权限：`identity.read`、`achievements.read`、`inventory.read`、`meals.read`、`bills.read`、`tasks.read`（按需）

## 1. 目标

把散落在各模块的到期日收进一个只读列表：证件、证书、保修、退货窗口、食品/药品、剩菜、周期账单。按「已逾期 / 今天 / 7 天 / 30 天 / 90 天」排序，点进去回到拥有者处理。**不建统一到期表，不复制业务数据，不自动续费、退货或丢弃。**

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 查询 |
|---|---|---|
| 证件/卡片有效期 | `identity` | `identity.list` |
| 证书/资格有效期 | `achievements` | `achievements.list` |
| 保修/退货窗口 | `inventory` | `inventory.items.listDeadlines` |
| 食品/药品批次有效期、开封后期限 | `inventory` | `inventory.items.expiringBetween` |
| 剩菜最晚食用日 | `meals` | `meals.leftovers.list` |
| 周期账单到期 | `bills` | `bills.occurrences.list` / `bills.upcoming.list` |
| 宠物护理、保养、其他待办 | `tasks` | `tasks.occurrences.list` |

本流程不拥有任何表；每一行都保留 `sourceModule` / `sourceType` / `sourceId`，写操作回到对应模块的 RPC。

## 3. 步骤

1. 用户选择窗口（7 / 30 / 90 天）与可选筛选：地点、成员、类型。
2. 客户端并行查询可见来源：
   - `identity.list { expiresBetween: [from, to] }`
   - `achievements.list { expiresBetween: [from, to] }`
   - `inventory.items.listDeadlines { from, to, kinds: ["warranty", "return"] }`
   - `inventory.items.expiringBetween { from, to }`
   - `meals.leftovers.list { status: ["stored", "frozen"], eatByBefore: to }`
   - `bills.occurrences.list { from, to, status: "upcoming" }`
   - 可选：`tasks.occurrences.list { from, to, status: "pending" }`，只把宠物护理、保养等「动作」放入单独的「要做」组。
3. 归一到统一行：`key`、`sourceModule`、`sourceType`、`sourceId`、`title`、`deadline`、`kind`、`severity`、`ownerMemberId?`、`locationId?`、`deepLink`、`visibility`。
4. `kind` 区分：`identity_expiry`、`certificate_expiry`、`warranty_end`、`return_by`、`stock_expiry`、`opened_expiry`、`leftover_eat_by`、`bill_due`、`task_due`。
5. 同一实体可能产生多行（例如物品同时有保修与退货窗口），按 `key = module:type:id:kind` 分别展示，不合并成假日期。
6. 排序：已逾期 → 今天 → 7 天 → 30 天 → 90 天；同一天按 `kind` 与标题稳定排序。
7. 点击一行 → 跳转拥有者详情；允许的动作只通过拥有者 RPC（如 `bills.occurrences.markPaid`、`inventory.loans.return`、`meals.leftovers.mark`）。
8. 页面显示「数据截至」时间与缺席模块；不做定时后台刷新，打开时重新查询。

## 4. 失败与补偿

- **模块未安装**：隐藏该组并列出缺席模块，不显示「没有到期项」。
- **无权限**：该来源整组隐藏；不把无权数据当成空。
- **日期缺失**：跳过该行，不按创建时间或购买日期猜测。
- **单位/时区**：日期按家庭时区解释；`deadline` 是日期字符串，时间戳只用于排序与展示。
- **过期记录**：仍显示并标「已逾期」；不自动归档、不自动改状态。
- **重复行**：同一实体同一 `kind` 只出现一次；查询重试按 `key` 去重。
- **数据陈旧**：显示最后刷新时间；不把旧缓存冒充实时。
- **隐私**：证件、健康相关行只显示标题与日期，不显示号码、诊断或备注；`guest` 无权限。

## 5. 事件

- 消费：`identity.record.changed`、`achievements.record.changed`、`inventory.item.changed`、`inventory.lot.opened`、`meals.leftover.changed`、`bills.occurrence.changed`、`tasks.occurrence.completed`，用于打开页面时刷新。
- 不新增事件；不写跨模块事务；不自动生成任务。

## 6. 验收

- 证件、证书、保修、退货、食品/药品、剩菜、账单分别出现在正确分组。
- 已逾期优先；窗口切换后只显示范围内项目。
- 同一物品的保修与退货窗口分别显示，不合并成一条假日期。
- 无权限或未安装模块时整组隐藏，并明确提示缺席。
- 点击可跳转到拥有者；页面没有直接改状态、续费、丢弃或退款的入口。
- 不新建表、不复制业务数据；日志不含证件号、诊断与金额明细。
- `guest` 看不到任何到期项。

## 7. 开放问题

- 是否把宠物疫苗/驱虫、车辆年检也纳入 `tasks` 动作组；v1 只列任务，不推断。
- 是否把健康用药结束日/复诊纳入；v1 不作为到期项，避免医疗判断。
- 是否需要固定窗口与首页摘要；v1 默认 90 天，可切换。
- 是否需要导出/打印与家庭周报 digest；v1 只做页面。
- 是否允许从雷达一键创建提醒；v1 只跳转，由用户显式创建任务。
