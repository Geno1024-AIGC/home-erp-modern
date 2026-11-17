# 流程：今晚吃什么，从冰箱里现有的东西倒推

> 涉及模块：`meals`、`inventory`、`health`（可选）、`tasks`（可选） ｜ 触发：用户点「今晚吃什么」 ｜ 权限：`meals.read`、`meals.write`、`inventory.read`、`health.read`（按可见性）

## 1. 目标

不先想菜，再发现缺三样；而是从现有库存、临期食材、可接受的缺料数量和过敏标签出发，给出一组确定性的候选。没有模型、没有外部菜谱抓取、没有「猜你喜欢」。选中后只把缺料加入采购清单，不自动下单、不自动扣库存。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 菜谱、食材、菜单、采购清单、做饭记录 | `meals` |
| 批次数量、过期与开封期限 | `inventory` |
| 过敏记录与可见性 | `health` |
| 做饭任务 / 提醒 | `tasks` |
| 库存扣减 | `inventory` |

`meals` 不查库存表；可用量通过 `inventory.items.availability` 读取。

## 3. 步骤

1. 用户打开「今晚吃什么」，传入 `date`、`slot`（默认 `dinner`）、`servings`、可选 `maxMinutes`、`memberIds`、`maxMissing`（默认 1）。
2. 调 `meals.recipes.suggest`。服务端先取候选菜谱，收集所有 `RecipeIngredient.inventory_item_id`，以请求者身份调 `inventory.items.availability { itemIds }`。
3. 可用量按基础单位返回，例如 `[{ itemId, unit: "g", available: 480 }]`。只有单位一致才参与匹配；自由文本食材按名称精确/别名匹配，不做模糊猜测。
4. 过滤：缺料数量超过 `maxMissing` 的菜谱不返回；已归档菜谱、已归档物品不参与；过期批次不计入可用量。
5. 过敏：调 `health.records.list { kind: "allergy", memberId in memberIds }`，只取调用者可见的记录；用 `metadata_json.allergenTag` 与 `Recipe.allergen_tags` 精确匹配。命中标 `allergenWarn`，默认不推荐。
6. 权限边界：没有 `inventory.read` 时返回 `stockUnknown`；有成员的健康记录不可见时返回 `allergenCheck: "partial"`，不指出是谁、不推断内容。
7. 排序（确定性）：先用临期食材 → 缺料少 → 最近 14 天没做过 → 评分高 → 准备时间短 → `recipe_id` 升序。相同输入必须返回相同顺序。
8. 结果卡显示：菜名、命中食材、临期食材、缺料、过敏警告、预计时间、排序原因。过敏命中时默认折叠，用户可展开查看；页面不出现「安全」「无过敏」。
9. 用户选中菜谱 → 显示缺料 → 确认后逐条 `meals.grocery.add { name, quantity, unit, inventoryItemId?, sourceRecipeId }`；只写采购清单。
10. 做完 → `meals.cookLogs.create`；如要扣库存，逐项 `inventory.stock.adjust { reason: "consume" }`，由用户确认；剩菜走 [leftovers.md](leftovers.md)。
11. 可选：`meals.plans.set` 把这顿写进菜单；或调 `tasks` 建「做饭」提醒。

## 4. 失败与补偿

- **未安装 `inventory`**：只按菜谱标签、时间、评分推荐，卡片标「无库存信息」，不假装「都能做」。
- **无 `inventory.read`**：返回 `stockUnknown`，不显示缺料数量。
- **单位不一致**：不换算、不猜密度，标 `unitMismatch`，不计入缺料数；用户可手动确认。
- **食材没有 `inventory_item_id`**：只做展示，不参与库存匹配。
- **健康记录不可见**：`allergenCheck: "partial"`；最坏情况漏掉警告，文案明确「仅按已记录且可见的过敏标签过滤」。
- **用户忽略过敏警告**：允许，但不改菜谱、不写「已确认安全」。
- **没有候选**：返回缺料最少的 3 个菜谱与缺料清单，支持一键加购；不生成假菜谱。
- **`meals` 未安装**：没有这个流程；不做跨模块临时推荐。
- **离线 / 恒星不可达**：v1 在线优先，直接报错，不返回过期缓存冒充实时库存。

## 5. 事件

- 读取时消费：`meals.recipe.changed`、`inventory.item.changed`（缓存失效）。
- 写入时产生：`meals.grocery.changed`、`meals.plan.changed`、`meals.cook_log.changed`。
- 不新增推荐事件；不做推荐曝光/点击统计。

## 6. 验收

- 库存足够的菜谱排在前面；缺料超过阈值的菜谱不出现。
- 临期食材优先被推荐，并在卡片上说明原因。
- 相同输入重复查询顺序一致，不使用随机。
- 单位不一致只提示，不被误判为缺料。
- 过敏标签命中默认不推荐；不可见记录标 `partial`，不泄露成员与内容。
- 未安装 `inventory` 或没有读权限时降级，不伪造库存。
- 加购只写采购清单，不改库存、不下单。
- 扣库存必须由用户显式确认；不做饭不扣库存。
- 日志不含菜名、健康标签与库存数量。

## 7. 开放问题

- 食材别名表由谁维护、是否允许家庭自定义；v1 只做精确匹配与少量内置别名。
- 份数缩放是否参与可用量计算；v1 只做展示。
- 是否需要营养、热量、成本；v1 不做。
- 是否允许半成品、外卖、预制菜参与候选；v1 只推荐已有菜谱。
- 过敏标签词表是否需要规范化与同义词映射；v1 用小写标签精确匹配。
- 是否只从收藏/最近做过的菜谱推荐；v1 全量候选。
