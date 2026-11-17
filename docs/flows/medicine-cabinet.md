# 流程：家庭药箱，库存、处方与急救卡分开放

> 涉及模块：`inventory`、`health`、`documents`（可选）、`tasks`（可选）、`contacts`（可选）、`meals`（可选） ｜ 触发：用户整理药箱 / 用药 / 临期查看 / 急救卡 ｜ 权限：`inventory.read`、`inventory.stock.adjust`、`health.read`、`documents.read`（可选）、`tasks.write`（可选）

## 1. 目标

把「家里有什么药、还剩多少、什么时候过期、谁在吃什么药、急救时看什么」分开管：**数量与有效期归 `inventory`，处方/过敏/慢病与急救可见性归 `health`**。系统只做记录与提醒，不做剂量计算、相互作用检查、诊断或急救指导。

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 用途 |
|---|---|---|
| 药品/保健品名称、剂型、含量、批号、数量、有效期、开封后期限 | `inventory` | 药箱库存与临期 |
| 处方、用药记录、过敏、慢病、急救可见性 | `health` | 谁在用、急救卡显示什么 |
| 处方单、检验报告、疫苗记录 | `documents` | 关联 `health.record`，不复制正文 |
| 用药/补药提醒 | `tasks` | 可选；标题通用，不写诊断 |
| 医生、医院、急救联系人 | `contacts` | 「找谁」；不存病历 |
| 采购清单 | `meals` | 缺药/缺耗材时由用户确认加入；不自动下单 |

`health.metadata_json.inventoryItemId` 只作逻辑引用；两边不共享表，也不建跨模块外键。

## 3. 步骤

1. 建药品/保健品：`inventory.items.create { name, kind: "consumable", category: "medicine" | "supplement", homeLocationId: 药箱位置, metadataJson: { genericName?, form?, strength?, prescription?, forMemberIds?, instructions?, disposal? } }`。`metadata_json` 不放诊断、完整证件号或密码。
2. 入库批次：`inventory.stock.adjust { itemId, delta, reason: "purchase", lot: { locationId, expiresAt?, unit? } }`。有效期可空；空表示未知，不按购买日期猜。
3. 开封：一包多件先 `inventory.lots.split` 拆出要开的那份，再 `inventory.lots.open { openedAt, useWithinDays }`。有效期取 `expires_at` 与开封后期限的较早值；`useWithinDays` 为空时只看 `expires_at`。
4. 记录处方/用药：`health.records.create { memberId, kind: "medication", title, occurredAt, provider?, visibility, emergencyVisible: false, metadataJson: { dosage?, frequency?, startDate?, endDate?, inventoryItemId? } }`。`inventoryItemId` 只关联库存，不在健康模块扣数量。
5. 记录过敏/慢病：`kind: "allergy"` / `"condition"`；过敏标签用小写规范词。只有本人或 `health.admin` 显式把 `emergency_visible` 置 1，才会进入急救卡。
6. 处方单、检验报告：`documents.upload` 后 `documents.link { targetType: "health.record", targetId, role: "prescription" | "medical_report" | "lab_result" }`。文件权限仍由 `documents` 裁决。
7. 打开药箱：`inventory.items.list { category: ["medicine", "supplement"] }`，按有效期/开封后期限升序；同时调 `inventory.items.expiringBetween { from: 今天, to: 今天 + 90 天 }`。每行显示名称、剂型/含量、余量、位置、有效期、开封状态。
8. 用药时由用户显式登记消耗：`inventory.stock.adjust { itemId, lotId, delta: -数量, reason: "consume" }`。v1 不按处方自动扣减，也不判断是否按时服药。
9. 需要提醒时由用户显式创建 `tasks.occurrences.create` 或周期模板；任务标题用「用药提醒」这类通用文案，不写药名、剂量或诊断。健康记录不自动生成任务。
10. 快用完：`inventory.items.listReorder` 给出建议；用户确认后 `meals.grocery.add` 进采购清单。`meals` 未安装时只读展示，不自动下单。
11. 过期或不再使用：`inventory.stock.adjust { reason: "spoil", note: "过期" }` 扣减对应批次；保留历史。系统不自动丢弃、不提供冲入下水道或回收指导，按当地要求处理。
12. 急救卡：由家庭手册流程调用 `health.records.list { emergencyVisible: true }`，只显示当前用户可见且显式开启的记录；联系人来自 `contacts`。没有记录时写「未标记」，不写「无过敏/安全」。

## 4. 失败与补偿

- **有效期未知**：不计算剩余天数，不按入库时间猜；显示「未填写」。
- **单位不一致**：不做片/毫升/毫克换算，不估算剩余天数；只显示批次余量与单位。
- **处方与库存对不上**：保留两边记录，显示「未关联/库存未知」；不自动改处方或库存。
- **健康记录不可见**：药箱仍可按 `inventory.read` 显示库存，但不显示他人处方；急救卡标「部分可见/未标记」。
- **私密用药**：`private` 记录只给本人与 `health.admin`；任务标题与通知不得泄露药名或诊断。
- **重复消耗/点错批次**：用 `reason: "adjust"` 追加更正，不删除历史；不把更正计入消耗速度。
- **开封后期限已过**：标「已过期」，不自动扣减、不自动丢弃；由用户确认后 `spoil`。
- **药品/保健品已归档**：默认列表不显示；健康记录仍保留 `inventoryItemId`，显示「库存已归档」。
- **附件失败**：健康记录已保存，附件可重试；不因文件失败回滚记录。
- **任务未完成**：不推断漏服、不告警、不改变健康记录。
- **`documents` / `tasks` / `meals` 未安装**：药箱与用药记录仍可用，缺少附件、提醒或采购入口。

## 5. 事件

- 消费：`inventory.item.changed`、`inventory.lot.opened`、`inventory.stock.low`、`health.record.changed`、`contacts.contact.changed`，用于刷新药箱与急救卡。
- 产生：`inventory.stock.low` / `inventory.stock.reorderSuggested`（库存侧）、`health.record.changed`（记录变化）、`documents.changed`（附件变化）。
- 不新增事件；不跨模块事务；不自动创建任务、采购单或急救建议。

## 6. 验收

- 药品与保健品按批次记录数量、有效期与开封后期限；临期能筛出。
- 用药记录可关联库存，但修改健康记录不自动改库存，反之亦然。
- 过敏/慢病/用药可标记 `emergency_visible`；急救卡只显示显式开启且调用者可见的记录。
- 没有急救记录时显示「未标记」，不显示「无过敏」或「安全」。
- 过期批次只能由用户确认后 `spoil`；系统不自动丢弃。
- 任务标题与通知不泄露药名、剂量、诊断；`private` 记录不对其他成员可见。
- 单位不一致、有效期未知时明确提示，不编造天数。
- `guest` 无权限；日志不含药品名、剂量、诊断与处方正文。

## 7. 开放问题

- 是否需要结构化用药计划（剂量/频次/提醒）与服药打卡；v1 只用 `health.metadata_json` 记录，不自动生成任务。
- 是否需要药品相互作用、禁忌与剂量建议；v1 不做，属于专业医疗系统。
- 是否需要药品条码/包装识别与自动填充；v1 手动录入，条码只作普通物品条码。
- 是否需要药品库存与处方强关联、自动扣减；v1 只做可选 `inventoryItemId`。
- 是否需要急救卡离线打印/导出与失效日期；v1 由家庭手册流程生成只读卡。
- 是否需要按成员独立药箱与更细的可见性；v1 用 `inventory` 位置 + `health.visibility`。
