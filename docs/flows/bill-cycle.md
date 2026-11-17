# 流程：周期账单，到期前知道，付完留痕

> 涉及模块：`bills`、`tasks`（可选）、`documents`（可选）、`contacts`（可选）、`members`（可选） ｜ 触发：新建账单 / 打开未来到期 / 付款后 ｜ 权限：`bills.read`、`bills.write`、`bills.pay`、`tasks.write`（可选）、`documents.link`（可选）

## 1. 目标

把房租、水电、宽带、保险、订阅等周期账单记成「计划 + 每期」，到期前能看到，付完能标记并留凭证。**只记录事实，不做记账、支付、自动扣款、预算或分账。**

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 用途 |
|---|---|---|
| 账单计划、每期 occurrence、金额、到期、已付事实 | `bills` | 周期生成、逾期查询、家庭手册/到期雷达 |
| 付款人 | `members` | 逻辑引用；不复制成员资料 |
| 收款方/物业/保险公司 | `contacts` | 可选关联；不存银行账号 |
| 到期提醒 | `tasks` | 可选一次性任务；不自动扣款 |
| 小票、保单、缴费凭证 | `documents` | 关联 `bills.occurrence`，不存文件正文 |

本流程不拥有表；`bills` 只保存逻辑引用，不移动资金。

## 3. 步骤

1. 新建计划：`bills.plans.create { name, category, amountCents?, currency: "CNY", recurrence, anchorDate, dueDay?, leadDays?, payerMemberId?, payeeContactId?, autoPay?, notes? }`。
   - 金额不固定时留空，等出账后改单期；`autoPay` 只是备注，不代表系统会扣款。
   - 月租可用 `dueDay`；月末日期超出当月天数时取当月最后一天。
2. 查看未来：`bills.occurrences.list { from, to, status: "upcoming" }` 会幂等补齐区间内缺失的 occurrence；首页/到期雷达调 `bills.upcoming.list { days: 30 }`。`overdue` 由「未付且已过到期日」派生，不写回状态。
3. 可选提醒：用户显式调 `tasks.occurrences.create { title: "账单到期：<名称>", dueAt: 到期前 leadDays, subjectType: "bills.occurrence", subjectId, dedupeKey: "bill:<planId>:<dueDate>" }`。`bills` 不自动创建任务，也不每次查询都提醒。
4. 实际支付在银行/支付应用完成，本流程不发起支付、不读余额、不保存卡号。用户回到账单页点「标记已付」。
5. `bills.occurrences.markPaid { occurrenceId, paidAt?, paidBy?, method?, note? }` 只写付款事实。重复提交返回已付记录，不重复写。
6. 金额不固定：出账后先 `bills.occurrences.updateAmount { occurrenceId, amountCents, note? }`，再标记已付。已付记录不会被计划金额更新覆盖。
7. 凭证：`documents.upload` 后 `documents.link { targetType: "bills.occurrence", targetId, role: "bill_receipt" }`。附件失败不回滚已付状态。
8. 不适用的一期：`bills.occurrences.skip { occurrenceId, note? }`；暂停订阅：`bills.plans.pause`；恢复：`bills.plans.resume`；一次性账单付完或不再续：`bills.plans.end`。
9. 填错已付：管理员调 `bills.occurrences.reopen { occurrenceId, reason }`，恢复为 `upcoming` 并保留原因；普通成员不能改已付事实。
10. 归档旧计划：`bills.plans.archive`；历史 occurrence 保留。家庭手册只读引用近期账单，不显示银行账号、余额与备注明细。
11. 页面结束；v1 不做预算、分类汇总、分账、报销或税务。

## 4. 失败与补偿

- **月末日期**：`dueDay=31` 在小月落到最后一天；不跳到下月 1 日。
- **重复 occurrence**：`UNIQUE(plan_id, due_date)` 与查询补齐保证幂等；并发冲突返回已有行。
- **计划改金额/改期**：只影响之后生成的 occurrence；旧未付行需显式改单期，已付行不动。
- **重复标记已付**：返回已有已付记录，不重复写时间与付款人。
- **金额未知**：显示「以实际账单为准」，不把空金额当成 0；未来合计只统计已知金额并标注不完整。
- **`autoPay`**：只作展示，不自动扣款、不自动标记已付。
- **提醒失败/`tasks` 未安装**：账单仍可用；不因为没有提醒而自动补发。
- **附件失败**：已付事实保留，凭证可重试；不因文件失败回滚。
- **离线/恒星不可达**：直接报错，不用旧缓存冒充最新到期；不在客户端本地生成 occurrence。
- **时区**：`due_date` 是家庭时区日期；`paid_at` 是 UTC 时间戳，展示按用户时区。
- **隐私**：`guest` 无权限；通知只说「有账单到期」，不写金额、收款方与备注。

## 5. 事件

- `bills.plan.changed { id, action }`：新建、编辑、暂停、恢复、结束、归档。
- `bills.occurrence.changed { id, planId, status }`：补齐、已付、跳过、改金额、重开。
- 消费：`tasks.occurrence.completed`（用户完成提醒时）、`documents.changed`（凭证变化）。
- 不跨模块事务；账单、提醒、凭证各自提交。

## 6. 验收

- 月租、宽带、保险、年费生成正确到期日；月末日期收敛正确。
- 一次性账单只生成一条；重复查询不重复生成。
- 未付且过期显示「已逾期」；不把逾期写回 `status`。
- 金额可空，出账后可改单期；已付记录不被计划更新覆盖。
- 标记已付、跳过、暂停、恢复、结束、归档行为正确；重复标记已付不产生第二条事实。
- 附件通过 `documents` 关联；删除关联不删文件。
- 未来 30 天列表按到期日排序；金额未知时明确标不完整。
- `guest` 无权限；Web 只读；任何操作都不产生支付、扣款、余额或交易流水。
- 日志与通知不含金额明细、收款方与备注。

## 7. 开放问题

- 是否需要部分付款、多次付款与退款；v1 每期只记一个已付事实。
- 是否需要自动扣款对接与银行同步；v1 只标记 `autoPay`，不接银行。
- 是否需要预算、分类汇总、分账与家庭结算；v1 不做。
- 是否需要自定义周期（每 N 天/每 N 月）与节假日顺延；v1 只做固定周期。
- 是否需要账单模板、商家目录与账单 OCR；v1 用计划 + `contacts`，手动录入。
- 是否需要按币种汇总与汇率换算；v1 只展示原币种。
- 是否需要到期自动创建任务；v1 由用户显式创建，避免重复提醒。
