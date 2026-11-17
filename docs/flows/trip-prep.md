# 流程：出远门之前，把家里和行李都交代清楚

> 涉及模块：`travel`、`tasks`、`inventory`、`meals`、`pets`（可选）、`identity`、`documents`、`smart_home`（可选） ｜ 触发：旅行已确认且出发日在 14 天内 / 用户点「准备出发」 ｜ 权限：`travel.read`、`travel.write`、`tasks.write`、`inventory.read`、`meals.read`、`pets.read`、`identity.read`、`documents.read`、`smart_home.control`（仅确认后）

## 1. 目标

把「出远门」拆成两组清单：**带走的**（证件、票据、行李）和**留下的**（冰箱、宠物、垃圾、门窗、水电）。清单按出发时间倒排，能复用、能改期、能跳过；系统只提示，不替人锁门、断电、关阀，也不假装知道东西已经带齐。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 行程、出发/返回时间、打包清单、票据关联 | `travel` |
| 离家待办（倒垃圾、关窗、交钥匙、停牛奶） | `tasks` |
| 冰箱里会坏的、要带走的物品 | `inventory` |
| 剩菜与最晚食用日 | `meals` |
| 宠物/植物档案与护理记录 | `pets` |
| 证件有效期与状态 | `identity` |
| 机票、酒店确认单、保险、委托书 | `documents` |
| 离家场景、设备状态 | `smart_home` |

本流程不建表；不复制证件号、宠物医疗细节或票据内容。

## 3. 步骤

1. 触发：`travel.trips.get` 返回 `status=confirmed` 且 `start_date` 在 14 天内，准备视图出现；用户也可以手动打开。
2. 打包：`travel.checklists.copyFrom { tripId, fromTripId, categories? }` 从上次旅行复制清单，只复制标题、分类与顺序，勾选与指派重置；新旅行也可以手工添加。
3. 离家待办：调 `tasks.checklists.instantiate { groupKey: "trip.away", subjectType: "travel.trip", subjectId, anchorAt: 出发时间 }`。模板组示例：
   - T-7：停牛奶/报纸订阅、确认宠物安排、买猫粮。
   - T-3：洗一桶衣服、把钥匙交给邻居/家人、确认垃圾日。
   - T-1：清冰箱、倒垃圾、关窗、给充电宝充电、把证件放包里。
   - T-0：关水阀/燃气（手动）、拍照记录、带钥匙。
   模板只存标题与偏移，不复制其他模块数据。
4. 冰箱：`inventory.items.expiringBetween { from: 出发日, to: 返回日 }` + `meals.leftovers.list { status: ["stored"], eatByBefore: 返回日 }`，列出「走之前吃掉 / 带走 / 冷冻 / 送人」的批次与剩菜；只列，不自动丢、不自动改数量。
5. 宠物/植物：`pets.list { status: "active" }`；为每只创建一次任务（喂食、换水、铲屎、寄养交接），`subjectType="pets.pet"`。任务只写宠物名快照与动作，不复制医疗记录。
6. 证件：`identity.list { memberIds: travelers, kind: ["passport", "travel_permit"] }`；`expires_at < 返回日 + 180 天`、`status` 为 `expired` / `lost` / `revoked` 时警告；列表只显示后四位。
7. 票据与保险：`documents.listFor { targetType: "travel.trip", targetId }`；缺 `ticket` / `booking` / `insurance` 时提示「未附」，不判定行程无效，也不要求必须上传。
8. 家电与安全：显示 `smart_home` 设备只读状态；用户确认后执行低风险「离家」场景。不包含门锁、加热、燃气、总阀、冰箱断电；高风险设备即使已接入也不提供远程命令。
9. 出发当天：`tasks.occurrences.list { subjectType: "travel.trip", subjectId }`；逐项勾选。未完成项标黄，不阻止出发，也不自动标记完成。
10. 改期：用新 `anchorAt` 重新 `instantiate`；旧实例保留并提示「有 N 条按旧时间生成」，用户逐条跳过或保留，不静默删除。
11. 回来：旅行置 `completed`；未完成待办可批量跳过并填原因（例如「邻居已代收」），保留历史。

## 4. 失败与补偿

- **出发时间未定**：锚点用 `start_date` 当天 09:00，标 `anchorEstimated`；清单仍可生成，用户可改。
- **临时改期**：生成新一组，旧组不自动删；提示数量，由用户跳过。`dedupeKey` 含锚点，重试不重复。
- **`pets` / `identity` / `documents` 未安装**：对应卡片缺席，清单组仍可用；不因为缺模块而阻塞生成。
- **护照有效期不足**：只警告，不判断目的地签证规则；用户自行确认。
- **票据未附**：只提示，不阻断；不自动从邮箱或外部抓取。
- **冰箱数据不完整**：只列已知批次与剩菜，不估算「大概还有什么」。
- **有人留在家**：离家清单组整体跳过；v1 由用户手动选择，不自动判断。
- **场景执行失败**：逐设备报错，不重试风暴；不把 `optimistic` 当成已关闭。
- **网络中断**：v1 在线优先，直接报错；需要离线查看时用 `travel.trips.export` 打印。
- **隐私**：待办与通知不写证件号、住址、宠物医疗细节；`guest` 无权限。

## 5. 事件

- `travel.trip.changed`、`travel.checklist.changed`
- `tasks.occurrence.created`、`tasks.occurrence.completed`
- `inventory.stock.expiring`、`meals.leftover.changed`
- `pets.pet.changed`
- `identity.record.changed`
- `smart_home.command.changed`
- 不跨模块事务；行程、清单、待办、场景各自提交，失败项由用户重试。

## 6. 验收

- 旅行确认且出发日在 14 天内出现准备视图。
- 一键生成离家待办，按 T-7 / T-3 / T-1 / T-0 排期；重复生成不重复。
- 改期后生成新一组，旧组保留并可跳过，不静默删除。
- 冰箱里在旅行期间到期的批次与剩菜被列出，且不自动丢弃。
- 有宠物时生成护理任务；无宠物不生成；任务不含医疗细节。
- 护照在返回后 6 个月内到期会警告；界面只显示后四位。
- 未附机票/保险只提示，不阻断。
- 离家场景必须确认；页面没有开锁、断电、关阀入口。
- 未完成项不阻止出发，也不自动标记完成。
- 未安装 `pets` / `identity` / `smart_home` 时仍能完成其余步骤。
- 日志与通知不含证件号、住址与宠物医疗细节。

## 7. 开放问题

- 是否提供独立的打包清单模板，而不是只从上次旅行复制；v1 用 `copyFrom`。
- 账单、订阅、房租等是否纳入离家清单；当前没有 `bills` 模块，只能写成普通任务。
- 交钥匙、邻居代收、紧急联系人是否引入 `contacts` 模块；v1 只写任务文本。
- 是否支持「家里有人留下」的自动判断与差异清单；v1 手动跳过。
- 是否接航班状态、天气、目的地入境要求；v1 不接。
- 是否需要离家期间的设备异常汇总；v1 只读设备状态。
- 是否需要「回家清单」（开窗、买菜、取快递、恢复热水器）。
