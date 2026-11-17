# 模块：饮食与采购（meals）

> 状态：draft ｜ 类型：business ｜ 依赖：`inventory`（可选）、`tasks`（可选）、`calendar`（可选）、`documents`（可选） ｜ 拥有：`m_meals_recipe`、`m_meals_recipe_ingredient`、`m_meals_plan`、`m_meals_grocery_item`、`m_meals_cook_log`、`m_meals_cook_log_dish`

## 1. 目的与非目标

**目的**：管理家庭菜谱、每周菜单、采购清单、剩菜和实际做饭记录；把「这周吃什么」「需要买什么」「今天做了什么」「哪盒剩菜该吃了」与现有库存、任务、日历、照片串起来。

**非目标**：

- 不做营养分析、医疗饮食建议；过敏过滤只按用户显式标签精确匹配，不做过敏原计算，也不保证「无过敏」。
- 不做模型推荐或自动生成菜谱；候选只来自已存菜谱与确定性排序。
- 不拥有库存数量；库存仍归 `inventory`。
- 不拥有买菜/做饭任务；任务仍归 `tasks`。
- 不做餐厅点单、外卖、价格比价、账务。
- v1 不自动扣库存；需要时显式调用 `inventory.stock.adjust`。
- 不把做饭记录当社交动态或公开相册；照片仍归 `documents` 管理。

## 2. 用户场景

- 保存家常菜谱：食材、用量、步骤、份数、标签。
- 打开「今晚吃什么」：按冰箱现有食材、缺料容忍度与过敏标签给出候选。
- 从候选菜谱把缺的食材加入采购清单。
- 多做了一份：记一条剩菜，设「最晚哪天吃掉」；吃掉或扔掉都留记录。
- 排一周菜单：早餐/午餐/晚餐/加餐。
- 买菜时逐项勾选；已买项不自动入库。
- 查看今天/本周吃什么。
- 菜谱关联库存物品；没有对应物品时用自由文本。
- 给菜谱附成品照片。
- 做完饭后一键记录：时间、菜品、份数、评分、备注和成品照片。
- 从当天菜单带入做饭记录；查看做饭时间线和相册。
- 在菜谱详情查看「做过几次」，按菜谱回看历史照片。

## 3. 边界与依赖

- 拥有菜谱、食材、菜单计划、采购清单、做饭记录与剩菜。
- 只引用 `inventory_item_id` 作为可选关联；不读/写库存表。
- 剩菜是烹饪产物，不建成库存物品，也不进库存流水。
- 库存可用量通过 `inventory.items.availability` 读取；不做跨模块 JOIN，不缓存过期结果。
- 过敏过滤只读调用者可见的 `health` 记录；不可见时返回 `allergenCheck: "partial"`，不指出是谁、不推断。
- 推荐是确定性排序，不调用模型、不抓外部菜谱。
- 采购任务/做饭任务由 `tasks` 拥有；`meals` 只产生事件或由 UI 创建任务。
- 日历可展示菜单计划，但餐次仍归 `meals`。
- 菜谱照片、做饭照片/附件归 `documents`；菜谱图关联 `meals.recipe`，做饭照片关联 `meals.cook_log`。
- 做饭记录记录实际发生了什么；菜谱是模板，菜单是计划，三者不自动同步。
- 做饭记录不自动扣库存、不自动完成任务；需要时由 UI 显式调用对应模块。
- 采购清单是共享清单；v1 不按成员隔离，不做价格与账务。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Recipe` | `id`、`title`、`description`、`servings`、`prep_minutes`、`cook_minutes`、`tags`、`allergen_tags`、`source`、`created_by`、`archived_at` |
| `RecipeIngredient` | `recipe_id`、`name`、`quantity`、`unit`、`inventory_item_id`、`note`、`sort` |
| `MealPlan` | `id`、`date`、`slot`、`title`、`recipe_id`、`servings`、`notes` |
| `GroceryItem` | `id`、`name`、`quantity`、`unit`、`checked`、`source`、`source_ref`、`source_recipe_id`、`inventory_item_id`、`created_at` |
| `CookLog` | `id`、`cooked_at`、`title`、`meal_plan_id`、`servings`、`rating`、`notes`、`created_by`、`archived_at` |
| `CookLogDish` | `id`、`cook_log_id`、`name`、`recipe_id`、`note`、`sort` |
| `Leftover` | `id`、`cook_log_id`、`dish_name`、`stored_at`、`location_id`、`eat_by`、`status`、`frozen_at`、`note`、`created_by`、`archived_at` |

`slot`：`breakfast`、`lunch`、`dinner`、`snack`。

`rating`：可选 1–5；用于个人回看，不作为推荐算法输入。

`Leftover.status`：`stored`（冷藏/常温）、`frozen`（冷冻）、`eaten`（吃掉）、`discarded`（丢弃）。

`GroceryItem.source`：`manual`、`recipe`、`menu`、`restock`、`other`。`source_ref` 存生成来源的逻辑键（如 `menu:<from>:<to>`），用于重复生成时去重；不是外键。

规则：

- 菜谱食材用自由文本 + 可选 `inventory_item_id`；不强制所有食材都有库存物品。
- 采购清单项可来自菜谱、菜单汇总、手动添加或补货建议；`checked` 只表示已买，不代表已入库。
- 一周菜单生成采购清单时，按 `inventory_item_id` 或「名称 + 单位」精确合并；不做模糊匹配、不换算单位。
- 重复生成只增补缺失项，不覆盖已勾选手动项；`source=menu` 且未勾选的旧项可按用户选择替换。
- 菜单汇总可调用 `inventory.items.availability` 扣减现有库存；没有 `inventory.read` 时标 `stockUnknown`，不假装库存为 0。
- 菜单计划按本地日期 `YYYY-MM-DD`；同一天同一餐可有多条（例如多道菜）。
- 删除菜谱默认归档；已被菜单/采购/做饭记录引用的菜谱不物理删除。
- 份数缩放 v1 只做展示计算，不自动改库存或采购数量。
- 不自动生成菜单；推荐/随机搭配后续再做。
- 做饭记录是实际发生的一次烹饪；同一天可以有多条，不要求和菜单计划一一对应。
- 一条 `CookLog` 至少填写 `title` 或一个 `CookLogDish`；有菜品时名称做快照，菜谱归档后仍可显示。
- `meal_plan_id` 只用于从计划带入和回链；删除计划不删除做饭记录。
- 做饭照片通过 `documents.link` 关联到 `targetType="meals.cook_log"`，角色用 `cook_result`；一条记录可多张。
- 做饭记录不自动扣库存，也不自动把 `tasks` 的做饭任务标记完成；UI 可提供显式操作。
- 剩菜引用一条 `CookLog` 或至少菜名快照；`cook_log_id` 可空（补记）。
- `eat_by` 默认 `stored_at + 3 天`，用户可改；它是提醒，不是食品安全保证。
- 冷冻后停提醒；解冻由用户重设 `eat_by`，系统不计算保质期。
- 吃掉/丢弃只改状态，不删除；丢弃写 `reason`。
- 剩菜不产生库存流水、不建 `Item`、不参与价格与补货。
- `allergen_tags` 由用户维护，规范化小写；只做精确匹配，不做同义词推断。
- 过敏命中默认不推荐，用户可展开查看；UI 不出现「无过敏」「安全」字样。
- `suggest` 排序确定性：临期食材优先、缺料少、最近 14 天没做过、评分高、耗时短，最后按 `recipe_id` 稳定排序；不随机。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `meals.read` | 查看菜谱、菜单、采购清单、做饭记录与剩菜 | `member` |
| `meals.write` | 新建/编辑菜谱、菜单、清单、做饭记录与剩菜 | `member` |
| `meals.admin` | 恢复归档、清理共享清单与记录 | `admin` |

## 6. RPC 与事件

- `meals.recipes.list` / `meals.recipes.get`
- `meals.recipes.create` / `meals.recipes.update` / `meals.recipes.archive`
- `meals.plans.list`：按日期范围查询。
- `meals.plans.set`：新增/更新某天某餐的条目。
- `meals.plans.remove`
- `meals.plans.groceryPreview`：`{ from, to, servingsScale? }`，汇总区间内菜单食材，返回 `required` / `available` / `missing` / `unitMismatch` / `sources`；只读。
- `meals.plans.generateGrocery`：`{ from, to, servingsScale?, mode?: "missing" | "all", replaceGenerated?: false }`，按 `source=menu` 幂等写入采购清单，返回新建/更新/跳过数量。
- `meals.grocery.list`
- `meals.grocery.add` / `meals.grocery.update` / `meals.grocery.check`
- `meals.grocery.clearChecked`
- `meals.cookLogs.list`：按日期范围、菜谱、菜品查询。
- `meals.cookLogs.get`
- `meals.cookLogs.create`：含菜品数组或标题；可带 `mealPlanId` 预填。
- `meals.cookLogs.update` / `meals.cookLogs.archive` / `meals.cookLogs.restore`
- `meals.cookLogs.listForRecipe`：菜谱详情页历史记录。
- `meals.recipes.canCook`：`{ recipeId, servings?, at? }`，返回可用食材、缺料与单位不一致项。
- `meals.recipes.suggest`：`{ date, slot?, servings?, maxMinutes?, memberIds?, maxMissing?, limit? }`，返回候选、临期食材、缺料、过敏警告与排序原因。
- `meals.leftovers.list`：`{ status?, eatByBefore?, locationId?, limit?, cursor? }`。
- `meals.leftovers.add`：`{ cookLogId?, dishName, storedAt?, locationId?, eatBy?, note? }`；`eatBy` 缺省为 `storedAt + 3 天`。
- `meals.leftovers.update`：`{ id, eatBy?, locationId?, note? }`。
- `meals.leftovers.mark`：`{ id, status, at?, reason? }`，用于吃掉、丢弃、冷冻与解冻。
- `meals.leftovers.listForCookLog`：做饭记录详情页回链。
- 事件：
  - `meals.recipe.changed { id, action }`
  - `meals.plan.changed { date }`
  - `meals.grocery.changed { id, action }`
  - `meals.cook_log.changed { id, action }`
  - `meals.leftover.changed { id, action, eatBy }`

`recipes.canCook` / `recipes.suggest` / `plans.groceryPreview` / `plans.generateGrocery` 以请求者身份调用 `inventory.items.availability`；没有 `inventory.read` 时返回 `stockUnknown`，不绕过权限。过敏过滤调用 `health.records.list(kind=allergy)`，只读可见记录；有成员记录不可见时返回 `allergenCheck: "partial"`，不指出是谁、不推断。

不提供自动库存扣减 RPC；需要扣库存时由 UI 显式调用 `inventory.stock.adjust`。做饭记录与照片的上传/关联由 UI 编排：先建记录，再调 `documents.upload` / `documents.link`；跨模块流程见 [cook-log.md](../flows/cook-log.md)。不提供模型推荐、外部菜谱抓取、过敏原计算与食品安全判断。

## 7. UI（原生 / Web）

- 原生：菜谱列表/详情、周菜单、采购清单、做饭记录时间线/相册；食材可搜索库存物品。
- Web：固定壳中的菜谱、菜单、采购清单、做饭记录；上传照片，不做拖拽排餐。
- 菜谱详情显示成品照片、食材、步骤；点击库存关联跳到物品详情。
- 菜单页显示今天/本周；可一键「创建买菜任务」（调 `tasks`）；某餐可「记录这顿」带入做饭记录。
- 周菜单提供「生成本周采购清单」：先预览需求/现有/缺料，再选择写入；已勾选与手动项不被覆盖。
- 首页「今晚吃什么」：显示候选、命中与临期食材、缺料、过敏警告与排序原因；不显示「安全」「无过敏」。
- 剩菜页：按 `eat_by` 排序，显示位置、存放天数、吃掉/扔掉/冷冻；到期标红，但不自动丢弃。
- 菜谱编辑可维护 `allergen_tags`，给出规范词建议但不自动推断。
- 采购清单支持勾选、手动添加、从菜谱导入食材；不做价格合计。
- 做饭记录支持拍照/选图、时间、菜品、份数、评分、备注；从菜谱或当天菜单预填。
- 做饭记录时间线按日期倒序，显示封面图、菜品、评分；可切换相册视图。
- 菜谱详情显示「做过 N 次」和最近记录；从记录可回到菜谱。
- 日历可选只读展示当天菜单；做饭记录不自动写入日历。

## 8. 数据与迁移

```sql
CREATE TABLE m_meals_recipe (
  id            TEXT PRIMARY KEY,
  title         TEXT NOT NULL,
  description   TEXT,
  servings      INTEGER,
  prep_minutes  INTEGER,
  cook_minutes  INTEGER,
  tags          TEXT,
  allergen_tags TEXT,
  source        TEXT,
  created_by    TEXT NOT NULL,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);

CREATE TABLE m_meals_recipe_ingredient (
  id                TEXT PRIMARY KEY,
  recipe_id         TEXT NOT NULL,
  name              TEXT NOT NULL,
  quantity          TEXT,
  unit              TEXT,
  inventory_item_id TEXT,
  note              TEXT,
  sort              INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_meals_ing_recipe ON m_meals_recipe_ingredient(recipe_id, sort);

CREATE TABLE m_meals_plan (
  id         TEXT PRIMARY KEY,
  date       TEXT NOT NULL,
  slot       TEXT NOT NULL,
  title      TEXT NOT NULL,
  recipe_id  TEXT,
  servings   INTEGER,
  notes      TEXT,
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
CREATE INDEX idx_meals_plan_date ON m_meals_plan(date, slot);

CREATE TABLE m_meals_grocery_item (
  id                TEXT PRIMARY KEY,
  name              TEXT NOT NULL,
  quantity          TEXT,
  unit              TEXT,
  checked           INTEGER NOT NULL DEFAULT 0,
  source            TEXT NOT NULL DEFAULT 'manual',
  source_ref        TEXT,
  source_recipe_id  TEXT,
  inventory_item_id TEXT,
  created_at        INTEGER NOT NULL,
  updated_at        INTEGER NOT NULL
);
CREATE INDEX idx_meals_grocery_checked ON m_meals_grocery_item(checked, created_at);

CREATE TABLE m_meals_cook_log (
  id           TEXT PRIMARY KEY,
  cooked_at    INTEGER NOT NULL,
  title        TEXT,
  meal_plan_id TEXT,
  servings     INTEGER,
  rating       INTEGER,
  notes        TEXT,
  created_by   TEXT NOT NULL,
  archived_at  INTEGER,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);
CREATE INDEX idx_meals_cook_at ON m_meals_cook_log(cooked_at DESC);
CREATE INDEX idx_meals_cook_plan ON m_meals_cook_log(meal_plan_id);

CREATE TABLE m_meals_cook_log_dish (
  id          TEXT PRIMARY KEY,
  cook_log_id TEXT NOT NULL,
  name        TEXT NOT NULL,
  recipe_id   TEXT,
  note        TEXT,
  sort        INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_meals_cook_dish_log ON m_meals_cook_log_dish(cook_log_id, sort);
CREATE INDEX idx_meals_cook_dish_recipe ON m_meals_cook_log_dish(recipe_id);

CREATE TABLE m_meals_leftover (
  id          TEXT PRIMARY KEY,
  cook_log_id TEXT,
  dish_name   TEXT NOT NULL,
  stored_at   INTEGER NOT NULL,
  location_id TEXT,
  eat_by      TEXT NOT NULL,                 -- YYYY-MM-DD
  status      TEXT NOT NULL DEFAULT 'stored',-- stored / frozen / eaten / discarded
  frozen_at   INTEGER,
  note        TEXT,
  created_by  TEXT NOT NULL,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_meals_leftover_eatby ON m_meals_leftover(status, eat_by);
CREATE INDEX idx_meals_leftover_cook ON m_meals_leftover(cook_log_id);
```

- `eat_by` 用 `YYYY-MM-DD`；`stored_at` 用 UTC epoch milliseconds；提醒按家庭时区当天计算。
- `allergen_tags` 存规范化小写、逗号分隔；空表示未标注，不表示安全。
- `source` / `source_ref` 只用于采购清单去重与展示；`source_ref` 不是跨模块外键。
- 剩菜不写库存表；`cook_log_id` 与 `location_id` 都是逻辑引用。

- 数量用文本，允许「一把」「适量」；需要精确换算时后续再加结构化数量。
- `cooked_at` 用 UTC epoch milliseconds；展示按家庭时区。
- `rating` 可空；范围 1–5，由 RPC 校验。
- 迁移只向前；菜谱默认归档，采购清单可清理已买项，做饭记录默认归档。
- 照片在 `documents` / `file_refs`，`meals` 不存文件；做饭照片关联 `meals.cook_log`。

## 9. 验收

- 新建菜谱、食材、步骤、标签；上传成品照片。
- 按周排菜单；同一天同一餐多道菜；查看今天/本周。
- 从菜谱加入采购清单；手动添加；勾选已买；清理已买项。
- 从一周菜单预览并生成采购清单；重复生成不重复，已勾选/手动项不被覆盖。
- 菜单汇总按库存可用量扣减；无 `inventory.read` 时标 `stockUnknown`，不伪造缺料。
- 单位不一致不换算，单独提示；自由文本食材按名称 + 单位精确合并。
- 采购清单勾选不改变库存；显式调用库存调整才改数量。
- 新建做饭记录：时间、菜品、份数、评分、备注；上传多张成品照片并关联。
- 从当天菜单或菜谱创建记录时能预填菜品；记录与计划/菜谱不自动同步。
- 做饭记录时间线、相册、菜谱「做过 N 次」正确。
- 删除/归档菜谱后，历史做饭记录和菜品名称仍可显示。
- 做饭记录不自动扣库存、不自动完成任务。
- 开封批次不生成 `Item`，剩菜不写库存流水；两者只在临期视图里合并展示。
- `suggest` 对相同输入返回相同顺序，不使用随机；库存够的排前，缺料超阈值不出现。
- 临期食材优先被推荐；单位不一致只提示，不计入缺料。
- 过敏标签命中默认不推荐；不可见健康记录返回 `allergenCheck: "partial"`，不显示「无过敏」。
- 从候选加购只写采购清单，不改库存。
- 剩菜默认 3 天且可改；到期标红不自动丢弃；吃掉/丢弃后进入历史。
- 冷冻停提醒，解冻需重设 `eat_by`。
- 菜谱关联库存物品；库存物品归档后关联仍可显示但提示失效。
- 归档菜谱不出现在默认列表；历史菜单仍可查。
- `guest` 无权限；`member` 可维护；`admin` 可清理共享清单。

## 10. 开放问题

- 过敏原只用显式 `allergen_tags` 精确匹配；是否需要规范词表与同义词映射，等有真实菜谱后再定。
- 是否需要营养/热量字段；v1 不做。
- 是否需要份数缩放后自动换算采购数量；v1 按计划份数缩放，但单位不一致不换算。
- 是否需要多份采购清单（按商店/按周）；v1 只有一份共享清单。
- 菜单汇总是否要识别「家里常备」并默认不进清单；v1 只按显式库存与手动排除。
- 是否需要菜谱导入/导出、网页抓取。
- 是否需要「做饭后自动扣库存」；v1 只做显式调整。
- 菜单计划是否要直接显示在日历主视图，还是只在 `meals` 内。
- 做饭记录是否要记录剩菜/浪费/下次改进；v1 剩菜独立记录状态，做饭记录先用备注。
- 冷藏/冷冻/室温是否分档，剩菜默认天数是否可配置；v1 统一 3 天，冷冻只停提醒。
- `suggest` 的权重是否需要用户可调，或加入收藏/最近做过；v1 固定确定性排序。
- 评分是否只做 1–5，还是加「是否愿意再做」等标签。
- 一条记录是否允许多个菜品；v1 允许，但 UI 默认按一顿饭记录。
- 做饭照片是否设封面图；v1 取最新/第一张，不做独立封面字段。
