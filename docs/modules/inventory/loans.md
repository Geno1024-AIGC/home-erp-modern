# 模块：物品 · 借出与归还（inventory/loans）

> 状态：draft ｜ 属于：`inventory` ｜ 依赖：`contacts`（可选）、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_inventory_loan`

## 1. 目的与非目标

**目的**：记录工具、书、投影仪、充电器、餐具等物品借给谁、借了多少、什么时候还、是否归还。用独立借出记录提供历史与到期提醒，而不是只把 `Item.status` 改成 `lent`。

**非目标**：

- 不做押金、租金、合同、赔偿与账务。
- 不做快递、物流追踪、定位或远程找回。
- 不做多物品借出单、部分归还与批次拆分；v1 一条借出记录对应一个物品/批次。
- 不自动催还、不自动改状态；提醒由 `tasks` 负责。

## 2. 用户场景

- 邻居借走电钻，登记借出人、到期日；到期前提醒。
- 把书借给朋友，归还时记录品相。
- 借出的餐具/工具在物品详情显示「借出中」，不能被推荐或算作可用库存。
- 查看某联系人借过什么、还有什么没还。
- 东西丢了，标记丢失并保留记录。
- 借出登记错了，取消记录，不留下虚假历史。

## 3. 边界与依赖

- 拥有借出记录；只引用 `item_id`、`lot_id`、`borrower_contact_id`，不建跨模块外键。
- `Item.status` 的 `lent` 由活跃借出记录派生；用户不能直接把物品改成 `lent`。
- 借出人可以是 `contacts` 中的联系人，也可以只填 `borrower_name` 快照；联系人删除后快照仍可显示。
- 归还照片/品相照片通过 `documents.link` 关联到 `targetType="inventory.loan"`。
- 提醒由 `tasks` 创建；`inventory` 只发事件。
- 不改变物品所有权、价格与购买记录。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `ItemLoan` | `id`、`item_id`、`lot_id`、`borrower_name`、`borrower_contact_id`、`quantity`、`unit`、`lent_at`、`due_at`、`returned_at`、`status`、`condition_out`、`condition_in`、`note`、`created_by` |

`status`：`active`、`returned`、`lost`、`cancelled`。

规则：

- 同一物品同一批次最多一条 `active` 借出记录；重复借出被拒绝。
- `Item.status=lent` 在存在 `active` 记录时派生；归还/取消/丢失后按其他状态恢复。
- `Item.status=lost` 表示物品丢失；可由管理员在找回后恢复，不物理删除。
- `quantity` 默认为 1；消耗品可按批次借出部分数量，但一个批次仍只允许一条活跃记录。
- `borrower_name` 必填并做快照；`borrower_contact_id` 可选。
- `due_at` 可空；为空时不计算逾期，只显示借出中。
- `overdue` 是派生状态：`status=active` 且 `due_at < 今天`；不写回 `status`。
- 归还可填写 `condition_in` 与备注；不自动生成维修/赔偿任务。
- `markLost` 把借出记录置 `lost`，物品置 `lost`；不删除记录。
- `cancel` 只用于登记错误；已归还/丢失的记录不能取消。
- 借出不影响 `StockLot.quantity`，但 `inventory.items.availability` 必须扣除活跃借出量。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `inventory.loans` | 新建借出、归还、延期、标记丢失与取消 | `member` |
| `inventory.admin` | 强制取消/恢复丢失物品 | `admin` |

- 查看借出记录需要 `inventory.read`；`guest` 只读或不可见，按家庭策略。
- 借出人联系人的可见性仍由 `contacts` 裁决；无权查看联系人时只显示快照名称。
- Web 可查看借出列表；新建/归还优先原生，避免误操作。

## 6. RPC 与事件

- `inventory.loans.create`：`{ itemId, lotId?, borrowerName, borrowerContactId?, quantity?, dueAt?, conditionOut?, note? }`。
- `inventory.loans.list`：`{ status?, itemId?, contactId?, dueBefore?, limit?, cursor? }`。
- `inventory.loans.get`
- `inventory.loans.return`：`{ loanId, returnedAt?, conditionIn?, note? }`。
- `inventory.loans.extend`：`{ loanId, dueAt }`。
- `inventory.loans.markLost`：`{ loanId, note? }`。
- `inventory.loans.cancel`：`{ loanId, reason }`。
- 事件：`inventory.loan.changed { loanId, itemId, status }`，供 `tasks`、物品详情与到期雷达刷新。

不提供：

- 不提供押金、租金、赔偿、合同或支付。
- 不提供自动催还、自动归还或远程锁定。
- 不提供多物品借出单与部分归还；需要时后续再评估。

## 7. UI（原生 / Web）

- 物品详情显示「借出」按钮、当前借出人、到期日与历史；活跃借出时显示「借出中」。
- 借出列表按活跃/逾期/已归还/丢失筛选；逾期标红，但不自动催。
- 新建借出可选联系人，也可只填姓名；数量默认 1，单位取批次单位。
- 归还表单填写品相、备注、照片；归还后物品恢复可用。
- 联系人详情显示其借出记录；无 `contacts.read` 时只显示快照名称。
- Web：只读借出列表与详情；写操作优先原生。

## 8. 数据与迁移

```sql
CREATE TABLE m_inventory_loan (
  id                  TEXT PRIMARY KEY,
  item_id             TEXT NOT NULL,
  lot_id              TEXT,
  borrower_name       TEXT NOT NULL,
  borrower_contact_id TEXT,
  quantity            INTEGER NOT NULL DEFAULT 1,
  unit                TEXT,
  lent_at             INTEGER NOT NULL,
  due_at              TEXT,
  returned_at         INTEGER,
  status              TEXT NOT NULL DEFAULT 'active',
  condition_out       TEXT,
  condition_in        TEXT,
  note                TEXT,
  created_by          TEXT NOT NULL,
  created_at          INTEGER NOT NULL,
  updated_at          INTEGER NOT NULL
);
CREATE UNIQUE INDEX idx_inventory_loan_active
  ON m_inventory_loan(item_id, IFNULL(lot_id, ''))
  WHERE status = 'active';
CREATE INDEX idx_inventory_loan_item ON m_inventory_loan(item_id, lent_at DESC);
CREATE INDEX idx_inventory_loan_contact ON m_inventory_loan(borrower_contact_id, status);
CREATE INDEX idx_inventory_loan_due ON m_inventory_loan(status, due_at);
```

- `due_at` 用 `YYYY-MM-DD`；`lent_at` / `returned_at` 用 UTC epoch milliseconds。
- 借出记录只追加与状态变更，不物理删除；取消保留原因。
- 迁移只向前；已有 `Item.status=lent` 数据需人工补借出记录或恢复为 `in_use`。

## 9. 验收

- 借出电钻后物品显示借出中，`availability` 扣除数量；重复借出被拒绝。
- 归还后物品恢复可用，借出历史保留；逾期是查询态。
- 借出人可为联系人，也可只填姓名；联系人不可见时只显示快照。
- 标记丢失后物品为 `lost`；管理员可恢复。
- 取消错误记录后不再占用物品；已归还记录不能取消。
- 借出/归还照片通过 `documents.link` 关联；删除关联不删文件。
- `tasks` 未安装时借出仍可用；没有自动提醒。
- 日志与通知不含借出人电话与备注。

## 10. 开放问题

- 是否需要部分归还、多物品借出单与押金；v1 不做。
- 是否需要借出品相照片与损坏赔偿；v1 只记文本/照片，不做金额。
- 是否需要默认借出天数与到期自动计算；v1 由用户填写。
- 是否需要借出人黑名单/信用；v1 不做。
- 是否需要把借出记录纳入家庭手册/联系人详情；v1 只做只读入口。
- 是否需要离线登记；v1 在线优先。
