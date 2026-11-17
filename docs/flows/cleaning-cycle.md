# 流程：家庭清洁，按区域轮着来

> 涉及模块：`tasks`、`locations`、`inventory`、`meals`（可选）、`smart_home`（可选）、`documents`（可选） ｜ 触发：清洁日 / 用户打开「本周清洁」 ｜ 权限：`tasks.read`、`tasks.complete`、`locations.read`、`inventory.read`、`meals.write`（可选）、`smart_home.control`（仅确认后）

## 1. 目标

把「家里该打扫了」变成按区域和周期重复的清洁任务：厨房、卫生间、客厅、卧室、阳台各自有什么、多久做一次、这次谁做、清洁用品够不够。**不新建 `cleaning` 模块，不自动启动设备，不把「勾了完成」当成卫生验收。**

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 用途 |
|---|---|---|
| 清洁任务模板、实例、完成记录 | `tasks` | 周期、分派、跳过、历史 |
| 房间/柜子/阳台等区域 | `locations` | 区域树、任务 `subject_id`、路径排序 |
| 清洁用品、工具、耗材、余量 | `inventory` | 够不够、放哪、何时补货 |
| 采购清单 | `meals` | 缺耗材时由用户确认加入；不自动下单 |
| 扫地机/净化器等设备 | `smart_home` | 可选、低风险、需显式确认 |
| 清洁前后照片、说明 | `documents` | 可选关联到任务 |

本流程不拥有表；区域、任务、库存各自提交。

## 3. 步骤

1. 用户从 `locations.tree.get` 选区域。v1 以房间/家具/容器为清洁区域，不单独维护「清洁区域」表。
2. 建周期模板：`tasks.templates.create { title: "清洁：厨房台面", recurrence: "weekly", subjectType: "locations.node", subjectId, assigneeId? }`。不同区域可用 `daily` / `weekly` / `monthly` / `everyNDays`。
3. 深度清洁或换季大扫除用一次性清单组：建 `group_key="cleaning.deep"`、`recurrence="once"`、`due_offset_minutes` 的模板，再调 `tasks.checklists.instantiate { groupKey, subjectType: "locations.node", subjectId, anchorAt }`。同一锚点重复生成幂等；改锚点保留旧组。
4. 打开清洁页时调 `tasks.occurrences.list { from, to, status: "pending" }`，只取清洁相关模板/标题；按 `subject_type=locations.node` 聚合到区域。
5. 客户端按位置树路径排序（地点 → 房间 → 家具），同区域按 `due_at` 升序；v1 不提供预计时长、优先级与拖拽排程。
6. 检查耗材：`inventory.items.list { category: "cleaning" }` 与 `inventory.items.availability`；缺货时调用 `inventory.items.listReorder`。用户可显式 `meals.grocery.add` 加入采购清单；`meals` 未安装时只显示缺货，不自动创建。
7. 执行任务：完成后调 `tasks.occurrences.complete { occurrenceId, notes? }`；跳过调 `tasks.occurrences.skip { occurrenceId, reason }`。完成只记录事实，不代表已经消毒或达到卫生标准。
8. 如果确实消耗了清洁用品，由用户显式调 `inventory.stock.adjust { itemId, lotId, delta: -数量, reason: "consume" }`；任务完成不自动扣库存，也不自动补货。
9. 需要留证时可上传清洁前后照片，通过 `documents.link` 关联到 `tasks.task`；照片失败不回滚任务完成。
10. 可选设备：扫地机、空气净化器等只允许用户显式确认后调用 `smart_home.scenes.execute { sceneId, idempotencyKey }`。场景必须是低风险；不包含门锁、加热、燃气、总阀，不在有宠物/儿童时替用户判断安全。
11. 周期任务完成后由 `tasks` 生成下一次实例；深度清洁实例完成后不重复，除非用户再次实例化。
12. 页面结束即完成；v1 不做清洁评分、连续天数、排行榜或自动生成「该大扫除了」。

## 4. 失败与补偿

- **区域已归档**：关联任务仍可完成，但页面标「区域已归档」；模板应归档，避免继续生成。不自动删除历史。
- **模板重复**：同一模板同一 `due_at` 由 `tasks` 唯一约束去重；重复生成返回已有实例。
- **没有分派**：作为公共任务显示，不自动分给某个成员。
- **耗材数量未知**：显示「未知」，不当作 0，也不自动加入采购清单。
- **单位不一致**：不换算、不估算可用次数；只显示各批次余量。
- **任务被跳过**：必须填原因；不把跳过算完成，也不自动顺延到下周。
- **设备离线/部分失败**：逐设备显示失败，不回滚已完成任务，不重试风暴；用户可以手动打扫。
- **照片/说明失败**：任务完成事实保留，附件可重试。
- **`meals` 未安装**：缺货只读展示，不创建采购清单。
- **`smart_home` 未安装**：不显示设备入口，清洁任务仍可用。
- **隐私**：`guest` 不显示任务与家庭位置；通知只说「今天有清洁任务」，不写房间细节。

## 5. 事件

- 消费：`tasks.occurrence.created`、`tasks.occurrence.completed`、`inventory.stock.low`、`inventory.stock.reorderSuggested`、`locations.node.changed`，用于刷新清洁页。
- 产生：`tasks.occurrence.completed`（用户完成时）、`inventory.stock.low`（耗材低于阈值时由库存发出）、`smart_home.command.changed`（仅在用户执行场景时）。
- 不新增事件；不跨模块事务；不自动生成采购单或设备命令。

## 6. 验收

- 为厨房、卫生间、客厅建不同周期的清洁模板，生成实例的到期日正确。
- 清洁页按区域聚合，按位置路径排序；没有预计时长/优先级等未定义字段。
- 深度清洁清单组按锚点生成；重复调用幂等，改锚点保留旧组。
- 缺耗材时显示余量与补货建议；用户确认后才进采购清单，不自动下单。
- 完成任务写完成记录；跳过必须填原因；任务完成不自动扣库存。
- 设备场景必须二次确认，且只能是低风险场景；没有自动启动或自动排程。
- 区域归档、模板重复、设备离线时页面仍可用并明确提示。
- `guest` 看不到家庭位置与任务；日志不含房间细节与耗材数量。

## 7. 开放问题

- 是否需要独立 `cleaning` 模块存区域、频次与检查项；v1 用 `tasks` + `locations`。
- 是否需要预计时长、优先级、拖拽排程与「今天做不完」自动顺延；v1 不做。
- 是否需要清洁检查清单/打分/照片对比；v1 只做任务备注与可选照片。
- 是否需要按季节、湿度或宠物掉毛触发；v1 只用周期模板与用户手动实例化。
- 是否需要把清洁任务与家政/保洁联系人关联；v1 可用 `tasks` 备注与 `contacts` 入口，不建工单。
- 是否需要清洁用品按房间独立库存；v1 由批次 `location_id` 表示。
