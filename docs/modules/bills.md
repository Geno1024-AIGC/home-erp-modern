# 模块：周期账单（bills）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`（可选）、`contacts`（可选）、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_bills_plan`、`m_bills_occurrence`

## 1. 目的与非目标

**目的**：管理家庭周期账单与续费：房租、水电、燃气、物业、宽带、手机、保险、会员、订阅、税费。记录「什么时候到期、多少钱、谁付、付了没有、凭证在哪」，并给到期雷达与家庭手册提供只读数据。

**非目标**：

- 不做记账、复式账、资产/负债、发票、报销、预算与报表。
- 不连银行、不发起支付、不自动扣款；`auto_pay` 只是备注标记。
- 不做分账、AA、家庭结算。
- 不做账单 OCR；小票/保单归 `documents`。
- 不做提醒调度；提醒归 `tasks` / 通知能力。
- 不做多币种换算；金额只存最小货币单位 + 币种。

## 2. 用户场景

- 每月房租、宽带、手机套餐到期前看到提醒。
- 每年保险、车辆年检、会员续费提前 30 天出现。
- 水电燃气金额每月不同：先记计划，出账后改成实际金额。
- 付完标记已付，附上小票/保单；不填也能标记。
- 暂停一个订阅，或跳过某一期。
- 查看未来 30/60 天要付多少，不把它当预算。
- 家庭手册/到期雷达只读引用即将到期账单。

## 3. 边界与依赖

- 拥有账单计划与每期账单（occurrence）；只引用 `payer_member_id`、`payee_contact_id`，不建跨模块外键。
- 不拥有钱、余额、银行账户、交易流水；`markPaid` 只记录事实，不移动资金。
- 收据/保单通过 `documents.link` 关联到 `targetType="bills.occurrence"`；本模块不存文件。
- 提醒由 `tasks` 或通知能力负责；`bills` 只发事件。
- 到期雷达与家庭手册通过只读 RPC 聚合；不复制账单表。
- 金额只用于展示与到期提醒，不做预算分析。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `BillPlan` | `id`、`name`、`category`、`amount_cents`、`currency`、`recurrence`、`anchor_date`、`due_day`、`lead_days`、`payer_member_id`、`payee_contact_id`、`auto_pay`、`status`、`notes`、`archived_at` |
| `BillOccurrence` | `id`、`plan_id`、`period_start`、`period_end`、`due_date`、`amount_cents`、`currency`、`status`、`paid_at`、`paid_by`、`payment_method`、`note` |

`category`：`rent`、`utility`、`internet`、`phone`、`insurance`、`subscription`、`membership`、`tax`、`other`。

`recurrence`：`once`、`monthly`、`quarterly`、`semiannual`、`yearly`。

`BillPlan.status`：`active`、`paused`、`ended`。

`BillOccurrence.status`：`upcoming`、`paid`、`skipped`。

`payment_method`：`auto`、`bank`、`card`、`cash`、`other`。

规则：

- `anchor_date` 是首次到期日（`YYYY-MM-DD`）；`once` 只生成一条 occurrence。
- `monthly` 可用 `due_day`（1–31）指定每月几号；缺省取 `anchor_date` 的日；超出当月天数时取当月最后一天。
- `quarterly` / `semiannual` / `yearly` 按 `anchor_date` 加月数推进；不按「每季度最后一天」猜测。
- `amount_cents` 可空，表示金额不固定；occurrence 可单独改金额，已付记录不被计划更新覆盖。
- `lead_days` 只用于展示「还有几天」；提醒由 `tasks` 决定，不自动生成。
- `paused` 停止生成新 occurrence；已有 occurrence 保留，可继续标记已付/跳过。
- `ended` 用于一次性账单付完或用户结束计划；不删除历史。
- `overdue` 是派生状态：`status=upcoming` 且 `due_date < 今天`；不写回 `status`。
- 每期只生成一条 occurrence；`UNIQUE(plan_id, due_date)` 保证幂等。
- `bills.occurrences.list` 在查询范围内补齐缺失的 occurrence；不后台预生成全年。
- `markPaid` 只写 `paid_at` / `paid_by` / `payment_method` / `note`，不调用支付、不改余额。
- `skip` 表示本期不付/免除；需要原因时写 `note`。
- 计划改金额只影响之后生成的 occurrence；未付的旧 occurrence 需显式 `updateAmount`。
- 归档计划不删除 occurrence；历史仍可查。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `bills.read` | 查看账单计划、到期与已付记录 | `member` |
| `bills.write` | 新建/编辑/暂停/结束计划 | `member` |
| `bills.pay` | 标记已付、跳过、修改单期金额 | `member` |
| `bills.admin` | 恢复归档、重新打开已付账单 | `admin` |

- `guest` 无权限；账单金额默认家庭可见。
- Web 只提供只读列表与到期摘要；新建、编辑、标记已付与附件优先原生。
- 通知与日志不显示完整账号、地址与备注；只显示「有账单到期」。

## 6. RPC 与事件

- `bills.plans.list`：`{ status?, category?, limit?, cursor? }`。
- `bills.plans.get`
- `bills.plans.create`：`{ name, category, amountCents?, currency?, recurrence, anchorDate, dueDay?, leadDays?, payerMemberId?, payeeContactId?, autoPay?, notes? }`
- `bills.plans.update`
- `bills.plans.pause` / `bills.plans.resume` / `bills.plans.end`
- `bills.plans.archive` / `bills.plans.restore`
- `bills.occurrences.list`：`{ from, to, status?, planId?, category? }`，返回区间内 occurrence，并幂等补齐缺失行。
- `bills.occurrences.get`
- `bills.occurrences.markPaid`：`{ occurrenceId, paidAt?, paidBy?, method?, note? }`。
- `bills.occurrences.skip`：`{ occurrenceId, note? }`。
- `bills.occurrences.updateAmount`：`{ occurrenceId, amountCents, note? }`。
- `bills.occurrences.reopen`：`{ occurrenceId, reason }`，仅 `bills.admin`。
- `bills.upcoming.list`：`{ days?, limit? }`，按到期日升序，供首页/到期雷达调用。
- 事件：
  - `bills.plan.changed { id, action }`
  - `bills.occurrence.changed { id, planId, status }`

不提供：

- 不提供支付、扣款、退款、余额、交易流水或银行连接。
- 不提供自动续费执行；`autoPay` 只作展示。
- 不提供预算、分账、报销、税务与多币种换算。
- 不提供账单 OCR 或自动从邮箱抓取。

## 7. UI（原生 / Web）

- 原生：计划列表、未来 30/60 天时间线、账单详情、标记已付、跳过、暂停/恢复、附小票。
- 首页/到期雷达显示「本周到期」「已逾期」；点击回到账单详情。
- 计划详情显示周期、下次到期、金额、付款人、收款方、凭证；金额不固定时显示「以实际账单为准」。
- 已付记录显示付款时间、付款人、方式、备注；可附多张凭证。
- Web：只读列表与到期摘要；不显示完整备注，不提供标记已付与编辑。
- 家庭手册只读引用近期账单；不显示银行账号（本模块也不存）。

## 8. 数据与迁移

```sql
CREATE TABLE m_bills_plan (
  id               TEXT PRIMARY KEY,
  name             TEXT NOT NULL,
  category         TEXT NOT NULL,
  amount_cents     INTEGER,
  currency         TEXT NOT NULL DEFAULT 'CNY',
  recurrence       TEXT NOT NULL,
  anchor_date      TEXT NOT NULL,
  due_day          INTEGER,
  lead_days        INTEGER NOT NULL DEFAULT 7,
  payer_member_id  TEXT,
  payee_contact_id TEXT,
  auto_pay         INTEGER NOT NULL DEFAULT 0,
  status           TEXT NOT NULL DEFAULT 'active',
  notes            TEXT,
  archived_at      INTEGER,
  created_at       INTEGER NOT NULL,
  updated_at       INTEGER NOT NULL
);
CREATE INDEX idx_bills_plan_status ON m_bills_plan(status, archived_at);
CREATE INDEX idx_bills_plan_anchor ON m_bills_plan(anchor_date);

CREATE TABLE m_bills_occurrence (
  id           TEXT PRIMARY KEY,
  plan_id      TEXT NOT NULL,
  period_start TEXT,
  period_end   TEXT,
  due_date     TEXT NOT NULL,
  amount_cents INTEGER,
  currency     TEXT NOT NULL,
  status       TEXT NOT NULL DEFAULT 'upcoming',
  paid_at      INTEGER,
  paid_by      TEXT,
  payment_method TEXT,
  note         TEXT,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL,
  UNIQUE (plan_id, due_date)
);
CREATE INDEX idx_bills_occ_due ON m_bills_occurrence(due_date, status);
CREATE INDEX idx_bills_occ_plan ON m_bills_occurrence(plan_id, due_date);
```

- `due_date` / `anchor_date` 用 `YYYY-MM-DD`；`paid_at` 用 UTC epoch milliseconds。
- 金额用最小货币单位整数；`currency` 用 ISO 4217；v1 默认 `CNY`，不换算。
- 不存银行账号、卡号、CVV、密码或支付凭证。
- 迁移只向前；删除默认归档计划，occurrence 不物理删除。

## 9. 验收

- 月租、宽带、保险、年费分别生成正确的到期日；月末日期正确收敛。
- 一次性账单只生成一条；重复查询不重复生成。
- 金额不固定的账单可先留空，出账后改单期金额；已付记录不被计划更新覆盖。
- 标记已付、跳过、暂停、恢复、结束、归档行为正确；已付/跳过不重复。
- 逾期是查询态，不写回状态；未来 30 天列表按到期日排序。
- 附件通过 `documents.link` 关联；删除关联不删文件。
- `guest` 无权限；Web 只读；日志与通知不含金额明细与备注。
- 不产生任何支付、扣款、余额或交易记录。

## 10. 开放问题

- 是否需要部分付款/多次付款；v1 每期只记一个已付事实。
- 是否需要自动扣款对接；v1 只标记 `autoPay`。
- 是否需要账单分账、家庭结算与预算；v1 不做。
- 是否需要自定义周期（每 N 天/每 N 月）；v1 只做固定周期。
- 是否需要账单模板/商家目录；v1 用计划 + `contacts`。
- 是否需要按币种汇总；v1 只展示原币种。
- 是否需要账单 OCR、邮件抓取与到期自动创建任务；v1 手动。
- 是否需要年付/一次性账单的摊销展示；v1 不做会计口径。
