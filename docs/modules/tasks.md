# 模块：家务与任务（tasks）

> 状态：draft ｜ 类型：business ｜ 依赖：`locations`、`inventory`、`members`、`documents`（可选） ｜ 拥有：`m_tasks_*`

## 1. 目的与非目标

**目的**：管理家务与家庭任务：模板、周期、分派、完成记录、历史查询。

**非目标**：

- 不做项目管理（依赖图、甘特图、工时）。
- 不做复杂审批流。
- 不存物品/位置数据；只保存逻辑引用。
- v1 不做多人协同编辑同一任务。

## 2. 用户场景

- 建一个「每周浇花」模板，每周六提醒。
- 建一组「出远门」清单：出发前 7 / 3 / 1 天各自到期。
- 临时加一件事（今晚吃掉剩菜），不建周期模板。
- 建一个「每月清洗空调滤网」模板，关联到某台空调。
- 把「倒垃圾」分给某个家庭成员。
- 完成后记录谁、何时、备注；家务记录不可篡改。
- 查看今天/本周待办与已完成历史。
- 跳过本次任务并填写原因。
- 出发前订酒店、检查护照；离家期间给花浇水；任务可关联到旅行，但不复制行程。

## 3. 边界与依赖

- 拥有任务模板、任务实例、完成记录。
- 只引用 `assignee_id`（member）、`subject_type` + `subject_id`（物品/位置等）；不建跨模块外键。
- 一次性实例可以没有模板：`title` 是快照，`dedupe_key` 保证重试幂等。
- 清单组只定义模板与相对锚点的偏移，不复制旅行、宠物、物品数据。
- 完成只记录事实，不代替其他模块改状态。
- 物品状态由 `inventory` 拥有；任务完成不回写物品数量。
- 完成照片归 `documents`；`tasks` 只存文档 ID 或通过 `documents.link` 关联。
- 日历只读叠加任务到期项；任务状态、改期、完成仍由 `tasks` 拥有。
- 宠物护理计划由 `tasks` 建模板，`subject_type` 可指向 `pets.pet`。
- 锻炼提醒/计划任务由 `tasks` 建模板/实例；`fitness` 不自动创建任务，任务也不自动生成运动记录；v1 不做目标↔任务关联，避免家庭可见任务暴露私有目标。
- 旅行准备任务由 `tasks` 建模板/实例，`subject_type` 可指向 `travel.trip`；行程、清单与游记仍由 `travel` 拥有；未安装 `travel` 时退化为普通任务。
- 提醒/通知由通知能力负责（后续）；`tasks` 只产生事件。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `TaskTemplate` | `id`、`title`、`description`、`recurrence`、`group_key`、`due_offset_minutes`、`assignee_id`、`subject_type`、`subject_id`、`active` |
| `TaskOccurrence` | `id`、`template_id`、`title`、`due_at`、`anchor_at`、`dedupe_key`、`subject_type`、`subject_id`、`status`、`completed_at` |
| `TaskCompletion` | `id`、`occurrence_id`、`action`、`completed_by`、`completed_at`、`notes` |

`status`：`pending`、`done`、`skipped`、`overdue`（派生，不落库）。

`recurrence`：v1 支持 `once`、`daily`、`weekly`、`monthly`、`everyNDays`；后续再评估 RRULE。

规则：

- 模板是「应该做什么」，实例是「这一次」；完成记录是不可变历史。
- 同一模板同一 `due_at` 只生成一个实例。
- 模板实例的 `due_at` 来自周期，或来自 `anchor_at + due_offset_minutes`；两种来源不混用。
- 一次性实例 `template_id` 为空、`title` 必填；`dedupe_key` 非空时唯一，重复创建返回已有实例。
- 带 `group_key` 的模板必须 `recurrence=once` 且 `due_offset_minutes` 非空；偏移可为负，0 表示锚点当时。
- 清单组模板不写死 `subject_type` / `subject_id` / `due_at`，由 `checklists.instantiate` 在生成时填入；未实例化前不产生任务。
- `checklists.instantiate` 对同一 `(template_id, anchor_at)` 幂等；改锚点生成新实例，旧实例保留，由用户跳过或完成。
- 一次性实例与模板实例在列表、完成、跳过、逾期上同构。
- 完成/跳过写 `TaskCompletion`（`action=done/skipped`），并把实例置为 `done` / `skipped`。
- 周期任务完成后生成下一次实例；生成失败不回滚完成记录。
- 逾期是查询态，不修改实例状态；避免后台批量写。
- 归档模板不删除历史实例；默认列表隐藏归档模板。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `tasks.read` | 查看任务与历史 | `member` |
| `tasks.write` | 新建/编辑模板、生成实例 | `member` |
| `tasks.assign` | 分派给他人 | `member` |
| `tasks.complete` | 完成/跳过自己或公共任务 | `member` |
| `tasks.admin` | 删除模板、强制完成 | `admin` |

## 6. RPC 与事件

- `tasks.templates.list` / `tasks.templates.get`
- `tasks.templates.create` / `tasks.templates.update` / `tasks.templates.archive`
- `tasks.occurrences.list`：按时间范围、状态、assignee、subject 筛选。
- `tasks.occurrences.create`：`{ title, dueAt, assigneeId?, subjectType?, subjectId?, dedupeKey? }`；一次性实例，`dedupeKey` 幂等。
- `tasks.checklists.instantiate`：`{ groupKey, subjectType, subjectId, anchorAt, assigneeId? }`，按模板组生成实例，返回新建与已存在数量。
- `tasks.checklists.groups`：列出当前可用的 `groupKey` 与模板数。
- `tasks.occurrences.complete`：`{ occurrenceId, notes? }`。
- `tasks.occurrences.skip`：`{ occurrenceId, reason }`。
- `tasks.listFor`：`{ subjectType, subjectId }`，供物品/位置详情调用。
- 事件：
  - `tasks.occurrence.created { id, templateId?, dueAt }`
  - `tasks.occurrence.completed { id, templateId? }`
  - `tasks.template.changed { id, action }`

## 7. UI（原生 / Web）

- 原生：今天/本周/全部、任务详情、完成/跳过、历史；支持通知入口（后续）。
- 任务详情显示来源（周期模板 / 一次性 / 清单组）与锚点；清单组生成的实例可整组查看。
- 可临时添加任务，不要求先建模板；重复提交按 `dedupeKey` 去重。
- 清单组入口：选择 `groupKey` 与锚点时间，预览后生成；已存在项不重复。
- Web：固定壳中的列表、详情、完成/跳过；不做拖拽日历（后续）。
- 物品/位置详情显示关联任务：`tasks.listFor`。
- 旅行详情显示关联任务：`tasks.listFor(subjectType="travel.trip", subjectId=tripId)`；完成状态不同步到行程。
- 完成表单可附照片；照片通过 `documents` 上传并关联，不在 `tasks` 存文件。

## 8. 数据与迁移

```sql
CREATE TABLE m_tasks_template (
  id                  TEXT PRIMARY KEY,
  title               TEXT NOT NULL,
  description         TEXT,
  recurrence          TEXT NOT NULL,
  group_key           TEXT,
  due_offset_minutes  INTEGER,
  assignee_id         TEXT,
  subject_type        TEXT,
  subject_id          TEXT,
  active              INTEGER NOT NULL DEFAULT 1,
  created_at          INTEGER NOT NULL,
  updated_at          INTEGER NOT NULL
);

CREATE TABLE m_tasks_occurrence (
  id           TEXT PRIMARY KEY,
  template_id  TEXT,
  title        TEXT NOT NULL,
  due_at       INTEGER NOT NULL,
  anchor_at    INTEGER,
  dedupe_key   TEXT,
  subject_type TEXT,
  subject_id   TEXT,
  status       TEXT NOT NULL,
  completed_at INTEGER,
  created_at   INTEGER NOT NULL,
  UNIQUE (template_id, due_at)
);
CREATE INDEX idx_tasks_occ_due ON m_tasks_occurrence(due_at, status);
CREATE INDEX idx_tasks_occ_subject ON m_tasks_occurrence(subject_type, subject_id, due_at);
CREATE UNIQUE INDEX idx_tasks_occ_dedupe
  ON m_tasks_occurrence(dedupe_key) WHERE dedupe_key IS NOT NULL;

CREATE TABLE m_tasks_completion (
  id            TEXT PRIMARY KEY,
  occurrence_id TEXT NOT NULL,
  action        TEXT NOT NULL,
  completed_by  TEXT NOT NULL,
  completed_at  INTEGER NOT NULL,
  notes         TEXT
);
CREATE INDEX idx_tasks_completion_occ ON m_tasks_completion(occurrence_id);
```

- 时间统一 UTC epoch milliseconds；展示按用户时区。
- `title` 是实例快照；模板改名不改历史实例。
- 迁移为已有实例补 `title`（取模板标题），再允许 `template_id` 为空。
- `dedupe_key` 由调用方生成；同一 key 只建一条，重复调用返回已有实例。
- `anchor_at` 只用于清单组生成与幂等，不参与展示排序。
- `UNIQUE (template_id, due_at)` 对一次性实例不生效（`template_id IS NULL`），由 `dedupe_key` 兜底。
- 完成记录只追加；不做物理删除。
- 迁移只向前；周期生成失败要记日志并可重试。

## 9. 验收

- 创建一次性/每日/每周/每月模板并生成实例。
- 完成、跳过、逾期查询正确；历史不可改。
- 同一模板同一时间不重复生成实例。
- 一次性实例无需模板即可创建；用同一 `dedupeKey` 重试只生成一条。
- 清单组按锚点生成：T-7、T-1、T-0 的 `due_at` 正确；重复调用幂等。
- 改锚点生成新实例，旧实例仍可完成/跳过；不静默删除。
- 未安装其他模块时，清单组仍可生成普通任务。
- 分派给成员；权限不足不能改他人任务。
- 物品/位置详情能显示关联任务。
- 旅行准备任务可关联到 `travel.trip`；完成任务不回写旅行，旅行也不复制任务状态。
- 原生与 Web 都能完成/跳过。

## 10. 开放问题

- 周期规则是否直接上 RRULE，还是先保持简单枚举。
- 提醒渠道：应用内、系统通知、邮件/IM。
- 是否支持多人认领、子任务、依赖。
- 完成照片是否强制；文件保留多久。
- 完成是否可撤销；若可，如何保持历史不可变（追加补偿记录）。
- 旅行打包清单是否也走 `tasks` 清单组；v1 打包归 `travel`，离家待办归 `tasks`。
- 清单组是否需要条件、成员或地点参数；v1 只有模板与相对偏移。
- 清单组是否支持从上次实例复制；v1 每次按模板生成，已存在项跳过。
- `dedupe_key` 作用域是家庭还是成员；v1 家庭内唯一。
