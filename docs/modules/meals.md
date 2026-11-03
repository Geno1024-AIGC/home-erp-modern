# 模块：饮食与采购（meals）

> 状态：draft ｜ 类型：business ｜ 依赖：`inventory`（可选）、`tasks`（可选）、`calendar`（可选）、`documents`（可选） ｜ 拥有：`m_meals_recipe`、`m_meals_recipe_ingredient`、`m_meals_plan`、`m_meals_grocery_item`

## 1. 目的与非目标

**目的**：管理家庭菜谱、每周菜单和采购清单；把「这周吃什么」「需要买什么」与现有库存、任务、日历串起来。

**非目标**：

- 不做营养分析、医疗饮食建议、过敏原计算。
- 不拥有库存数量；库存仍归 `inventory`。
- 不拥有买菜/做饭任务；任务仍归 `tasks`。
- 不做餐厅点单、外卖、价格比价、账务。
- v1 不自动扣库存；需要时显式调用 `inventory.stock.adjust`。

## 2. 用户场景

- 保存家常菜谱：食材、用量、步骤、份数、标签。
- 排一周菜单：早餐/午餐/晚餐/加餐。
- 从菜谱把食材加入采购清单。
- 买菜时逐项勾选；已买项不自动入库。
- 查看今天/本周吃什么。
- 菜谱关联库存物品；没有对应物品时用自由文本。
- 给菜谱附成品照片。

## 3. 边界与依赖

- 拥有菜谱、食材、菜单计划、采购清单。
- 只引用 `inventory_item_id` 作为可选关联；不读/写库存表。
- 采购任务/做饭任务由 `tasks` 拥有；`meals` 只产生事件或由 UI 创建任务。
- 日历可展示菜单计划，但餐次仍归 `meals`。
- 菜谱照片/附件归 `documents`。
- 采购清单是共享清单；v1 不按成员隔离，不做价格与账务。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Recipe` | `id`、`title`、`description`、`servings`、`prep_minutes`、`cook_minutes`、`tags`、`source`、`created_by`、`archived_at` |
| `RecipeIngredient` | `recipe_id`、`name`、`quantity`、`unit`、`inventory_item_id`、`note`、`sort` |
| `MealPlan` | `id`、`date`、`slot`、`title`、`recipe_id`、`servings`、`notes` |
| `GroceryItem` | `id`、`name`、`quantity`、`unit`、`checked`、`source_recipe_id`、`inventory_item_id`、`created_at` |

`slot`：`breakfast`、`lunch`、`dinner`、`snack`。

规则：

- 菜谱食材用自由文本 + 可选 `inventory_item_id`；不强制所有食材都有库存物品。
- 采购清单项可来自菜谱，也可手动添加；`checked` 只表示已买，不代表已入库。
- 菜单计划按本地日期 `YYYY-MM-DD`；同一天同一餐可有多条（例如多道菜）。
- 删除菜谱默认归档；已被菜单/采购引用的菜谱不物理删除。
- 份数缩放 v1 只做展示计算，不自动改库存或采购数量。
- 不自动生成菜单；推荐/随机搭配后续再做。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `meals.read` | 查看菜谱、菜单、采购清单 | `member` |
| `meals.write` | 新建/编辑菜谱、菜单、清单 | `member` |
| `meals.admin` | 恢复归档、清理共享清单 | `admin` |

## 6. RPC 与事件

- `meals.recipes.list` / `meals.recipes.get`
- `meals.recipes.create` / `meals.recipes.update` / `meals.recipes.archive`
- `meals.plans.list`：按日期范围查询。
- `meals.plans.set`：新增/更新某天某餐的条目。
- `meals.plans.remove`
- `meals.grocery.list`
- `meals.grocery.add` / `meals.grocery.update` / `meals.grocery.check`
- `meals.grocery.clearChecked`
- 事件：
  - `meals.recipe.changed { id, action }`
  - `meals.plan.changed { date }`
  - `meals.grocery.changed { id, action }`

不提供自动库存扣减 RPC；需要扣库存时由 UI 显式调用 `inventory.stock.adjust`。

## 7. UI（原生 / Web）

- 原生：菜谱列表/详情、周菜单、采购清单；食材可搜索库存物品。
- Web：固定壳中的菜谱、菜单、采购清单；不做拖拽排餐。
- 菜谱详情显示成品照片、食材、步骤；点击库存关联跳到物品详情。
- 菜单页显示今天/本周；可一键「创建买菜任务」（调 `tasks`）。
- 采购清单支持勾选、手动添加、从菜谱导入食材；不做价格合计。
- 日历可选只读展示当天菜单。

## 8. 数据与迁移

```sql
CREATE TABLE m_meals_recipe (
  id           TEXT PRIMARY KEY,
  title        TEXT NOT NULL,
  description  TEXT,
  servings     INTEGER,
  prep_minutes INTEGER,
  cook_minutes INTEGER,
  tags         TEXT,
  source       TEXT,
  created_by   TEXT NOT NULL,
  archived_at  INTEGER,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
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
  source_recipe_id  TEXT,
  inventory_item_id TEXT,
  created_at        INTEGER NOT NULL,
  updated_at        INTEGER NOT NULL
);
CREATE INDEX idx_meals_grocery_checked ON m_meals_grocery_item(checked, created_at);
```

- 数量用文本，允许「一把」「适量」；需要精确换算时后续再加结构化数量。
- 迁移只向前；菜谱默认归档，采购清单可清理已买项。
- 照片在 `documents` / `file_refs`，`meals` 不存文件。

## 9. 验收

- 新建菜谱、食材、步骤、标签；上传成品照片。
- 按周排菜单；同一天同一餐多道菜；查看今天/本周。
- 从菜谱加入采购清单；手动添加；勾选已买；清理已买项。
- 采购清单勾选不改变库存；显式调用库存调整才改数量。
- 菜谱关联库存物品；库存物品归档后关联仍可显示但提示失效。
- 归档菜谱不出现在默认列表；历史菜单仍可查。
- `guest` 无权限；`member` 可维护；`admin` 可清理共享清单。

## 10. 开放问题

- 是否需要营养/过敏原/热量字段；v1 不做。
- 是否需要份数缩放后自动换算采购数量。
- 是否需要多份采购清单（按商店/按周）。
- 是否需要菜谱导入/导出、网页抓取。
- 是否需要「做饭后自动扣库存」；v1 只做显式调整。
- 菜单计划是否要直接显示在日历主视图，还是只在 `meals` 内。
