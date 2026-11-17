# 流程：借出与归还，东西去哪了有账可查

> 涉及模块：`inventory`（借出记录）、`contacts`（可选）、`tasks`（可选）、`documents`（可选） ｜ 触发：用户登记借出 / 归还 / 到期查看 ｜ 权限：`inventory.read`、`inventory.loans`、`contacts.read`（可选）、`tasks.write`（可选）

## 1. 目标

把「电钻借给邻居、书借给朋友、投影仪借给同事」记成可查的借出记录：借给谁、借了什么、什么时候还、有没有归还。借出期间物品显示「借出中」，可用量扣除；归还后恢复可用。**不把物品状态当成唯一事实，不做押金、赔偿、催收或远程找回。**

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 用途 |
|---|---|---|
| 物品、批次、可用量、物品状态 | `inventory` | 判断能不能借、扣减 `availability`、派生 `lent` / `lost` |
| 借出记录 `ItemLoan` | `inventory`（[loans](../modules/inventory/loans.md)） | 借出人、数量、到期、归还与品相 |
| 借出人联系人 | `contacts` | 可选关联；无权查看时只显示快照名称 |
| 归还提醒 | `tasks` | 可选一次性任务；不自动催收 |
| 品相/归还照片 | `documents` | 关联 `targetType="inventory.loan"`，`role="loan_condition"` |

本流程不建表；借出记录是 `inventory` 的子契约，联系人、任务与文件仍归各自模块。

## 3. 步骤

1. 用户从物品详情点「借出」，或打开借出列表新建。先查 `inventory.items.get` 与 `inventory.loans.list { itemId, status: "active" }`。
2. 调 `inventory.items.availability { itemIds: [itemId] }`，确认该物品/批次没有活跃借出；已有活跃记录时直接拒绝并显示借出人，不自动合并、不覆盖。
3. 填写借出：`inventory.loans.create { itemId, lotId?, borrowerName, borrowerContactId?, quantity?, dueAt?, conditionOut?, note? }`。
   - `borrowerName` 必填并做快照；联系人可选，联系人不可见时只填姓名。
   - `quantity` 默认 1；消耗品可按批次借出部分数量，但同一物品同一批次仍只允许一条活跃记录。
   - `dueAt` 用日期；为空表示不计算逾期，只显示借出中。
4. 服务端同事务写借出记录并校验活跃唯一；`Item.status=lent` 由记录派生，不直接写物品状态。借出量从 `availability` 扣除，批次数量与流水不变。
5. 可选提醒：用户显式调 `tasks.occurrences.create { title: "归还：<物品名>", dueAt: 到期前 1 天, subjectType: "inventory.item", subjectId, dedupeKey: "loan:<loanId>:due" }`。`tasks` 未安装或创建失败不阻塞借出。
6. 借出期间物品详情显示「借出中」、借出人、到期日；借出列表按活跃/逾期/已归还/丢失筛选。逾期是查询态，不自动改状态、不自动发消息。
7. 归还时用户先检查物品，再调 `inventory.loans.return { loanId, returnedAt?, conditionIn?, note? }`。服务端把记录置 `returned`，物品恢复派生状态，`availability` 恢复。
8. 需要留证时上传照片并通过 `documents.link` 关联到 `inventory.loan`；文件失败不回滚归还，用户可稍后补传。
9. 东西确实找不回：`inventory.loans.markLost { loanId, note? }`，借出记录置 `lost`，物品置 `lost`；不生成赔偿金额、不自动联系借出人。
10. 登记错误且尚未归还：`inventory.loans.cancel { loanId, reason }`；已归还/丢失的记录不能取消。取消后释放占用，保留审计原因。
11. 管理员在物品找回后通过 `inventory.items.update` 恢复物品状态；借出记录仍保留 `lost` 历史，不删除。

## 4. 失败与补偿

- **重复借出**：活跃唯一索引拒绝；返回已有记录，不自动把两次借出合并。
- **联系人不可见/已归档**：只显示 `borrower_name` 快照；不泄露电话、地址与私密备注。
- **物品已归档/已处置**：拒绝新建借出；需先恢复物品。
- **数量不合法**：`quantity` 必须大于 0 且不超过可用量；消耗品部分借出仍只允许一条活跃记录。
- **重复归还**：已归还记录返回「已归还」，不重复写归还时间；不产生第二条历史。
- **部分归还**：v1 不支持；需要时先归还整条，再新建剩余借出。
- **逾期**：只显示，不自动催收、不自动标记丢失；是否延期由用户调 `inventory.loans.extend`。
- **标记丢失后找回**：管理员恢复物品；借出记录保留 `lost`，不伪装成正常归还。
- **照片上传失败**：借出/归还事实已提交，照片可重试；不因附件失败回滚业务状态。
- **`tasks` 未安装**：借出与归还仍可用，只是没有提醒。
- **时区**：`dueAt` 按家庭时区解释；跨时区用户看到同一日期，不按 UTC 日期截断。

## 5. 事件

- `inventory.loan.changed { loanId, itemId, status }`：借出、归还、延期、丢失、取消时发出，供物品详情、借出列表与到期雷达刷新。
- `inventory.item.changed { id, action }`：派生状态变化时发出。
- `tasks.occurrence.completed`：用户完成归还提醒时由 `tasks` 发出；不自动完成。
- `documents.changed`：附加/解除品相照片时由 `documents` 发出。
- 不跨模块事务；借出、提醒、照片各自提交。

## 6. 验收

- 借出电钻后物品显示「借出中」，`availability` 扣除数量；再次借出同一物品/批次被拒绝。
- 归还后物品恢复可用，`availability` 恢复，借出历史保留；重复归还不会写两条。
- 借出人可以是联系人，也可以只填姓名；联系人私密或归档时只显示快照。
- 到期日为空时不显示逾期；有到期日时逾期是查询态，不自动催收。
- 标记丢失后物品为 `lost`，管理员可恢复；取消错误记录后释放占用，已归还记录不能取消。
- 借出/归还照片通过 `documents` 关联；附件失败不回滚借出事实。
- `tasks` 未安装时流程仍可用；提醒只能由用户显式创建。
- 日志与通知不含借出人电话、地址与私密备注；`guest` 无写权限。

## 7. 开放问题

- 是否需要部分归还、多物品借出单与押金；v1 只做单物品/单批次。
- 是否需要借出品相评分与损坏赔偿金额；v1 只记文本与照片，不做账务。
- 是否需要默认借出天数、到期自动延期与批量催还；v1 由用户填写与点按。
- 是否需要借出人黑名单/信用；v1 不做。
- 是否需要把借出记录放进家庭手册或联系人详情；v1 只做只读入口。
- 是否需要离线登记借出；v1 在线优先。
