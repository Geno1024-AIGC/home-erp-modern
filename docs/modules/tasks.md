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
- 建一个「每月清洗空调滤网」模板，关联到某台空调。
- 把「倒垃圾」分给某个家庭成员。
- 完成后记录谁、何时、备注；家务记录不可篡改。
- 查看今天/本周待办与已完成历史。
- 跳过本次任务并填写原因。

## 3. 边界与依赖

- 拥有任务模板、任务实例、完成记录。
- 只引用 `assignee_id`（member）、`subject_type` + `subject_id`（物品/位置等）；不建跨模块外键。
- 物品状态由 `inventory` 拥有；任务完成不回写物品数量。
- 完成照片归 `documents`；`tasks` 只存文档 ID 或通过 `documents.link` 关联。
- 提醒/通知由通知能力负责（后续）；`tasks` 只产生事件。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `TaskTemplate` | `id`、`title`、`description`、`recurrence`、`assignee_id`、`subject_type`、`subject_id`、`active` |
| `TaskOccurrence` | `id`、`template_id`、`due_at`、`status`、`completed_at` |
| `TaskCompletion` | `id`、`occurrence_id`、`action`、`completed_by`、`completed_at`、`notes` |

`status`：`pending`、`done`、`skipped`、`overdue`（派生，不落库）。

`recurrence`：v1 支持 `once`、`daily`、`weekly`、`monthly`、`everyNDays`；后续再评估 RRULE。

规则：

- 模板是「应该做什么」，实例是「这一次」；完成记录是不可变历史。
- 同一模板同一 `due_at` 只生成一个实例。
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
- `tasks.occurrences.complete`：`{ occurrenceId, notes? }`。
- `tasks.occurrences.skip`：`{ occurrenceId, reason }`。
- `tasks.listFor`：`{ subjectType, subjectId }`，供物品/位置详情调用。
- 事件：
  - `tasks.occurrence.completed { id, templateId }`
  - `tasks.template.changed { id, action }`

## 7. UI（原生 / Web）

- 原生：今天/本周/全部、任务详情、完成/跳过、历史；支持通知入口（后续）。
- Web：固定壳中的列表、详情、完成/跳过；不做拖拽日历（后续）。
- 物品/位置详情显示关联任务：`tasks.listFor`。
- 完成表单可附照片；照片通过 `documents` 上传并关联，不在 `tasks` 存文件。

## 8. 数据与迁移

```sql
CREATE TABLE m_tasks_template (
  id           TEXT PRIMARY KEY,
  title        TEXT NOT NULL,
  description  TEXT,
  recurrence   TEXT NOT NULL,
  assignee_id  TEXT,
  subject_type TEXT,
  subject_id   TEXT,
  active       INTEGER NOT NULL DEFAULT 1,
  created_at   INTEGER NOT NULL,
  updated_at   INTEGER NOT NULL
);

CREATE TABLE m_tasks_occurrence (
  id           TEXT PRIMARY KEY,
  template_id  TEXT NOT NULL,
  due_at       INTEGER NOT NULL,
  status       TEXT NOT NULL,
  completed_at INTEGER,
  created_at   INTEGER NOT NULL,
  UNIQUE (template_id, due_at)
);
CREATE INDEX idx_tasks_occ_due ON m_tasks_occurrence(due_at, status);

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
- 完成记录只追加；不做物理删除。
- 迁移只向前；周期生成失败要记日志并可重试。

## 9. 验收

- 创建一次性/每日/每周/每月模板并生成实例。
- 完成、跳过、逾期查询正确；历史不可改。
- 同一模板同一时间不重复生成实例。
- 分派给成员；权限不足不能改他人任务。
- 物品/位置详情能显示关联任务。
- 原生与 Web 都能完成/跳过。

## 10. 开放问题

- 周期规则是否直接上 RRULE，还是先保持简单枚举。
- 提醒渠道：应用内、系统通知、邮件/IM。
- 是否支持多人认领、子任务、依赖。
- 完成照片是否强制；文件保留多久。
- 完成是否可撤销；若可，如何保持历史不可变（追加补偿记录）。
