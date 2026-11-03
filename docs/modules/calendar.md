# 模块：日历（calendar）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`locations`（可选）、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_calendar_event`、`m_calendar_event_member`

## 1. 目的与非目标

**目的**：维护家庭日历：事件、纪念日、生日、多日活动；支持参与者、地点、附件与简单重复。提供月/周/日视图和「某天有什么事」的聚合视图。

**非目标**：

- 不做待办/完成/分派；这些归 `tasks`。
- 不做提醒调度；提醒由通知能力或 `tasks` 负责。
- v1 不支持农历、复杂 RRULE、单次例外、资源预约。
- 不存成员/地点/文件数据；只保存逻辑引用。

## 2. 用户场景

- 记录结婚纪念日，每年显示「第 N 周年」。
- 记录孩子和家人的生日。
- 记录家庭旅行、学校活动、家长会、公司出差。
- 查看某个月/某周；按成员、类型筛选。
- 在某天同时看到事件和待办任务。
- 给活动附邀请函、门票、行程单。

## 3. 边界与依赖

- 拥有事件、纪念日、参与者关联；不拥有任务、成员、地点、文件。
- 引用 `member_id`、`location_id`；不建跨模块外键。
- 附件归 `documents`；`calendar` 只调用 `documents` RPC。
- 日历可以只读叠加 `tasks` 的到期项；任务状态仍由 `tasks` 拥有，日历不复制、不完成、不改期。
- 生日/纪念日 v1 由 `calendar` 建事件，避免和 `members` 资料双份维护。
- 不内置提醒；事件变化只发事件，通知/任务自行消费。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `CalendarEvent` | `id`、`title`、`kind`、`description`、`all_day`、`start_date`、`end_date`、`start_at`、`end_at`、`recurrence`、`recurrence_until`、`location_id`、`created_by`、`archived_at` |
| `CalendarEventMember` | `event_id`、`member_id`、`role` |

`kind`：`event`、`anniversary`、`birthday`、`holiday`、`other`。

`role`：`subject`（当事人，如生日/纪念日）、`participant`、`host`。

规则：

- 全天事件用 `start_date` / `end_date`（`YYYY-MM-DD`，结束日包含）；`start_at` / `end_at` 为空。
- 定时事件用 `start_at` / `end_at`（UTC epoch ms）；日期字段为空，结束必须晚于开始。
- `anniversary` / `birthday` 默认全天、`recurrence=yearly`；`start_date` 是原始日期。
- `recurrence`：`none`、`daily`、`weekly`、`monthly`、`yearly`；`recurrence_until` 为可选结束日期。
- 重复按同日展开；不存在的日期跳过（例如 2 月 29 日只在闰年出现，31 日只在有 31 日的月份出现）。
- v1 不支持单次例外；修改重复事件即修改整个系列。
- `calendar.events.list` 在服务端展开重复事件；限制查询范围与展开数量。
- 归档事件不删除历史；参与者和附件关联保留。
- 地点可选；附件通过 `documents.link` 关联到 `targetType="calendar.event"`。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `calendar.read` | 查看日历与事件 | `member` |
| `calendar.write` | 新建、编辑、归档事件 | `member` |
| `calendar.admin` | 恢复归档、处理敏感事件 | `admin` |

v1 默认家庭内可见；私密事件、按成员可见性后续再加。

## 6. RPC 与事件

- `calendar.events.list`：`{ from, to, kinds?, memberId?, locationId? }`，返回展开后的 occurrence。
- `calendar.events.get`：返回系列定义。
- `calendar.events.create`
- `calendar.events.update`
- `calendar.events.archive` / `calendar.events.restore`
- `calendar.events.listForMember`：成员详情页时间线。
- 事件：`calendar.event.changed { id, action }`。

限制：

- 单次查询范围 ≤ 1 年，展开结果 ≤ 500 条；超出返回范围错误。
- 任务叠加不由 `calendar` 查表；客户端并行调 `tasks.occurrences.list`。

## 7. UI（原生 / Web）

- 原生：月/周/日视图、事件详情、新建/编辑、参与者选择、地点选择、附件上传。
- Web：固定壳中的月视图 + 列表；v1 不做拖拽改期。
- 纪念日/生日显示「第 N 周年」；v1 生日需要完整日期，是否支持只记月日见开放问题。
- 同一天展示两类条目：
  - 事件：`calendar.events.list`。
  - 任务：只读叠加 `tasks.occurrences.list`，点击跳转到任务详情。
- 事件详情显示参与者、地点、附件；附件通过 `documents.listFor(targetType="calendar.event")`。
- 按类型着色；颜色只在前端映射，不落库。

## 8. 数据与迁移

```sql
CREATE TABLE m_calendar_event (
  id               TEXT PRIMARY KEY,
  title            TEXT NOT NULL,
  kind             TEXT NOT NULL,
  description      TEXT,
  all_day          INTEGER NOT NULL,
  start_date       TEXT,
  end_date         TEXT,
  start_at         INTEGER,
  end_at           INTEGER,
  recurrence       TEXT NOT NULL DEFAULT 'none',
  recurrence_until TEXT,
  location_id      TEXT,
  created_by       TEXT NOT NULL,
  archived_at      INTEGER,
  created_at       INTEGER NOT NULL,
  updated_at       INTEGER NOT NULL
);
CREATE INDEX idx_calendar_event_date ON m_calendar_event(start_date, end_date);
CREATE INDEX idx_calendar_event_time ON m_calendar_event(start_at, end_at);
CREATE INDEX idx_calendar_event_kind ON m_calendar_event(kind, archived_at);

CREATE TABLE m_calendar_event_member (
  event_id  TEXT NOT NULL,
  member_id TEXT NOT NULL,
  role      TEXT NOT NULL,
  PRIMARY KEY (event_id, member_id, role)
);
CREATE INDEX idx_calendar_event_member ON m_calendar_event_member(member_id);
```

- 全天用日期字符串；定时用 UTC epoch milliseconds；展示按家庭时区。
- 重复事件不预生成 occurrence 行；查询时展开。
- 迁移只向前；删除默认归档。
- 附件在 `documents` / `file_refs`，日历不存文件。

## 9. 验收

- 新建全天单日、多日、定时事件；结束时间/日期校验正确。
- 新建周年纪念日，跨年查询能展开并显示「第 N 周年」。
- 生日、纪念日默认按年重复；重复日期不存在时跳过。
- 按成员、类型、地点、时间范围筛选正确。
- 日历同一天能只读显示任务，点击可跳转；任务状态不被日历修改。
- 参与者、地点、附件关联正确；删除关联不删文件。
- 归档事件默认不返回；恢复后重新出现。
- 查询范围和展开数量超限被拒绝。
- `guest` 无权限；`member` 可维护；`admin` 可恢复归档。

## 10. 开放问题

- 是否支持农历生日/纪念日；v1 先公历。
- 是否需要单次例外、修改本次/后续、完整 RRULE。
- 提醒由 `tasks` 还是统一通知服务负责。
- 私密事件/按成员可见性是否需要。
- 生日是否记录出生年份、是否显示年龄。
- 任务是否要反过来在日历中直接改期，还是坚持只读叠加。
- 家庭时区变化、跨时区出差事件的展示规则。
