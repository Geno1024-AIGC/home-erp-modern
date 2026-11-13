# 模块：运动与健身（fitness）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`tasks`（可选）、`health`（可选） ｜ 拥有：`m_fitness_activity_type`、`m_fitness_workout`、`m_fitness_goal`

## 1. 目的与非目标

**目的**：记录家庭成员的实际运动，并按成员给出只读统计：总量、活跃天数、连续天数、按运动类型分布、个人最佳与目标进度。运动是独立于医疗档案的生活记录，统计口径可解释、可复算。

**非目标**：

- 不做诊断、康复、训练建议、热量/心率解读；身体不适归 `health`。
- 不做训练计划、周期化、组数/次数/重量、1RM、动作库；v1 只记录一次运动。
- 不做 GPS 轨迹、地图、配速分段、海拔、步频、功率。
- 不接穿戴设备、HealthKit、Google Fit、Strava 或任何第三方运动服务；v1 只手动录入。
- 不做社交动态、家庭排行榜、挑战赛、评论、公开分享。
- 不做自动热量估算、自动配速、自动识别运动类型。
- 不自动完成 `tasks`、不自动创建提醒；提醒归 `tasks`。
- 不存照片/视频/身体围度；v1 不做运动照片。
- 不提供 Web 页面；运动时间与频率可推断作息与家中是否有人，v1 仅原生。
- 不做设备/器械接入；`smart_home` 不参与运动记录。

## 2. 用户场景

- 跑完步快速记一条：类型、时长、距离、备注。
- 记录力量训练、瑜伽、游泳、骑行、徒步、球类等；类型可扩展。
- 补记昨天的运动；修改录错的时长/距离。
- 看本周/本月运动次数、总时长、总距离、活跃天数。
- 看连续运动天数与最长连续天数。
- 按运动类型看分布与个人最佳（最长时长、最远距离、最多步数）。
- 设一个「每周 3 次」「每月 20 公里」目标，看进度。
- 把运动记录设为仅自己可见；需要时改为家庭可见。
- 不适/受伤时另建 `health` 记录；运动模块不自动判断。

## 3. 边界与依赖

- 拥有运动类型、运动记录、目标与统计口径。
- 只引用 `member_id`；不建跨模块外键。
- 成员资料归 `members`；成员归档后运动记录保留，显示名由 `members` 解析，不复制成员资料。
- 医疗记录归 `health`；运动损伤、疼痛、心率异常、就医归 `health`，`fitness` 不做医学解释。
- 提醒与计划归 `tasks`；锻炼提醒是普通任务，不与私有目标自动关联；任务完成状态不回写目标，运动也不自动完成任务。
- 日历 v1 不叠加运动记录；运动时间可能暴露作息与家中无人。
- 文件/照片归 `documents`；v1 运动不产生附件。
- 成就不自动生成；个人最佳是统计结果，不是 `achievements` 记录。
- 通知默认不发运动提醒/排名；`tasks` 的提醒由 `tasks` 自己发送。
- 统计只基于请求者可见的记录；私有记录不因管理员身份而可见。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `ActivityType` | `id`、`code`、`name`、`category`、`primary_metric`、`icon`、`color`、`sort`、`builtin`、`archived_at` |
| `Workout` | `id`、`member_id`、`activity_type_id`、`started_at`、`local_date`、`duration_sec`、`distance_m`、`steps`、`calories_kcal`、`avg_heart_rate`、`max_heart_rate`、`rpe`、`visibility`、`source`、`note`、`client_id`、`archived_at` |
| `Goal` | `id`、`member_id`、`activity_type_id`、`metric`、`period`、`target_value`、`start_date`、`end_date`、`status` |

`ActivityType.category`：`cardio`、`strength`、`flexibility`、`sport`、`outdoor`、`other`。

`ActivityType.primary_metric`：`duration`、`distance`、`steps`。

`Workout.visibility`：`private`、`family`。

`Workout.source`：v1 只有 `manual`；字段保留给未来的导入，但 v1 不接受其他值。

`Goal.metric`：`duration_sec`、`distance_m`、`steps`、`workout_count`、`active_days`。

`Goal.period`：`day`、`week`、`month`、`year`。

`Goal.status`：`active`、`archived`。

规则：

- 内置类型：走路、跑步、骑行、游泳、徒步、瑜伽、力量训练、球类、其他；名称保持通用，不含厂商/设备型号。管理员可新增自定义类型；`code` 唯一且不可改；内置类型只能归档，不能删除。
- 运动类型归档后，历史记录仍保留并可统计；新记录不能使用归档类型；统计结果标记 `archived`。
- `primary_metric` 变更会改变该类型的个人最佳口径；UI 需提示，历史记录的其他指标不丢失。
- `started_at` 用 UTC epoch ms；`local_date` 是成员本地日期 `YYYY-MM-DD`。服务端校验 `local_date` 与 `started_at` 的 UTC 日期相差不超过 1 天。统计按 `local_date` 分桶，不按 UTC。
- 跨午夜运动按开始日 `local_date` 计一次，不拆分到两天；活跃天数按开始日计算。
- `duration_sec` 必须大于 0，且不超过 7 天；`distance_m`、`steps`、`calories_kcal` 不得为负；心率 20–250 bpm；`avg_heart_rate <= max_heart_rate`；`rpe` 为 1–10。
- 同一成员的非归档运动不能时间重叠：区间为 `[started_at, started_at + duration_sec * 1000)`；重叠返回 `409 OVERLAP`，用户需修改时长/时间或拆分记录。相邻不算重叠。归档记录恢复时重新校验重叠，冲突则拒绝恢复。
- `visibility` 默认 `private`；改成 `family` 必须由用户显式操作。归档后不参与默认列表与统计。
- `note` 为纯文本，最多 2,000 字符；可能包含敏感信息，禁止写入日志、审计与通知。
- `client_id` 可选；同一 `member_id` 下唯一；重试幂等，不生成重复记录。
- `Goal.activity_type_id` 为空表示所有运动类型；`target_value > 0`，单位由 `metric` 决定；`start_date`/`end_date` 为本地日期，且 `start_date <= end_date`（若填写结束日期）。
- 目标周期：`day` 为当天；`week` 为周一至周日；`month` 为自然月；`year` 为自然年。`end_date` 早于周期结束时按 `end_date` 截断。
- 同一成员、同一运动类型（含「全部」）、同一 `metric`、同一 `period` 只能有一个 `active` 目标；重复返回 `GOAL_CONFLICT`。
- 目标 v1 仅本人可见；不支持家庭目标。
- 运动与目标都只归档、不物理删除；迁移只向前。

### 4.1 统计口径

- 所有统计只读，按请求者可见的记录计算；`private` 记录只有本人可见，`family` 记录家庭成员可见，`guest` 无权限。
- 查看他人统计时，只汇总其 `family` 记录；UI 必须标注「仅统计家庭可见记录」，避免把私有记录通过聚合泄露。
- `summary`：运动次数、总时长、总距离、总步数、总热量、活跃天数、当前连续天数、最长连续天数、按类型分布。
- 活跃天数按 `local_date` 去重；连续天数按连续的自然日计算。
- 当前连续天数：从 `to` 当天向前数；如果 `to` 当天没有运动，允许从 `to - 1` 天开始；若 `to` 与 `to - 1` 都没有运动，则为 0。`to` 由客户端传本地日期，避免服务端时区猜测。
- 最长连续天数：在请求区间 `[from, to]` 内计算，不跨区间外推断。
- `timeline`：按 `day`、`week`、`month` 分桶；缺失桶补 0；不做插值、平滑或移动平均；周从周一开始；最多 500 个桶，超出返回 `BUCKET_LIMIT`。
- `timeline` 的 `active_days` 在日桶为 0 或 1，在周/月桶为该桶内活跃天数；`workout_count` 为桶内运动次数。
- 桶边界按 `local_date` 计算，不按 UTC；跨时区旅行时以成员填写的本地日期为准。
- 个人最佳：按运动类型的 `primary_metric` 计算，只统计该指标非空的记录；默认全量历史，可限定区间；值相同取更早的 `started_at`，再按 `id` 排序。
  - `duration`：单次最长时长。
  - `distance`：单次最远距离。
  - `steps`：单次最多步数。
- 个人最佳不是 `achievements` 记录，也不自动生成奖项/证书。
- 目标进度按目标周期汇总可见记录；进度可以超过 100%，UI 进度条封顶显示，但数值保留真实超出量。
- 不提供排行榜、家庭总量、挑战赛与成员对比；这些会放大隐私与公平性问题。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `fitness.read` | 查看自己的运动/目标，以及他人 `family` 可见的运动 | `member` |
| `fitness.write` | 新建/编辑/归档自己的运动与目标 | `member` |
| `fitness.admin` | 管理运动类型；归档/恢复家庭可见的不适当记录 | `admin` |

- `guest` 默认无任何运动权限。
- `fitness.read` 不是全量读取；RPC 必须叠加 `visibility` 与 `member_id`。
- 普通成员只能写 `member_id = 自己` 的记录；v1 不支持代录。
- 管理员不能读取或修改 `private` 运动与目标，也没有破窗入口；数据库/备份管理员不属于模块权限范围。
- 管理员可归档 `family` 可见的运动记录，但不能改正文、时长、距离、心率或备注。
- 运动类型的新增/改名/归档需要 `fitness.admin`；普通成员只能使用现有类型。
- 统计与个人最佳仍受可见性约束；管理员查看他人统计时同样只看到 `family` 记录。

## 6. RPC 与事件

- `fitness.activityTypes.list`：`{ includeArchived? }`，按 `sort`、`name` 排序。
- `fitness.activityTypes.create`：`{ code, name, category, primaryMetric, icon?, color?, sort? }`；需 `fitness.admin`。
- `fitness.activityTypes.update`：`{ activityTypeId, name?, category?, primaryMetric?, icon?, color?, sort? }`；`code` 不可改。
- `fitness.activityTypes.archive` / `fitness.activityTypes.restore`；需 `fitness.admin`。
- `fitness.workouts.list`：`{ memberId?, from?, to?, activityTypeId?, visibility?, includeArchived?, limit?, cursor? }`；默认本人、按 `started_at` 倒序。
- `fitness.workouts.get`：`{ workoutId }`；私有记录仅本人可读。
- `fitness.workouts.create`：`{ activityTypeId, startedAt, localDate, durationSec, distanceM?, steps?, caloriesKcal?, avgHeartRate?, maxHeartRate?, rpe?, visibility?, note?, clientId? }`。
- `fitness.workouts.update`：`{ workoutId, ... }`；仅本人可改；改时间/时长时重新校验重叠。
- `fitness.workouts.archive` / `fitness.workouts.restore`：本人可归档/恢复；管理员只能归档/恢复 `family` 记录。
- `fitness.goals.list`：`{ status?, includeArchived? }`；v1 仅本人。
- `fitness.goals.create`：`{ activityTypeId?, metric, period, targetValue, startDate, endDate? }`。
- `fitness.goals.update`：`{ goalId, ... }`；改周期/指标/目标值时重新校验冲突。
- `fitness.goals.archive`
- `fitness.stats.summary`：`{ memberId?, from, to, activityTypeId? }`；`from`/`to` 为本地日期，必填。
- `fitness.stats.timeline`：`{ memberId?, metric, bucket, from, to, activityTypeId? }`；`metric` 为 `duration_sec`、`distance_m`、`steps`、`workout_count` 或 `active_days`；`bucket` 为 `day`、`week` 或 `month`。
- `fitness.stats.personalBests`：`{ memberId?, activityTypeId?, from?, to? }`；返回每个运动类型 `primary_metric` 的最佳值与对应 `workoutId`。
- `fitness.stats.goalProgress`：`{ asOfDate }`；仅本人，返回所有 `active` 目标在当前周期的进度。
- 事件：
  - `fitness.workout.changed { workoutId, memberId, action }`
  - `fitness.goal.changed { goalId, memberId, action }`
  - `fitness.activityType.changed { activityTypeId, action }`

事件只带 ID 与动作，不带时长、距离、心率、备注或日期。

不提供：

- 不提供 GPS/路线/配速分段/海拔/功率。
- 不提供穿戴设备、HealthKit、Google Fit、Strava 导入或同步。
- 不提供训练计划、组数/次数/重量、动作库与教练建议。
- 不提供家庭排行榜、挑战赛、成员对比、评论与公开分享。
- 不提供运动照片/视频、身体围度、体重记录；这些归 `health` 或后续模块。
- 不提供自动热量估算、自动运动识别与医疗解读。
- 不提供 Web 页面；Web 网关拒绝 `fitness.*` RPC。
- 不提供导出/打印；后续若加，按高敏感操作审计。

## 7. UI（原生 / Web）

- 原生：导航「运动」；概览显示今日/本周统计、连续天数、目标进度与最近记录。
- 快速记录：运动类型选择、时长（分钟/小时）、可选距离/步数/热量/平均心率/最大心率/RPE/备注；可见性默认「仅自己」，可改「家庭可见」。
- 历史：按日期分组，按类型/可见性筛选；点击编辑、归档、恢复；不展示他人私有记录。
- 统计：时间范围预设（7/30/90 天、本月、本年、自定义）；指标切换（时长/距离/步数/次数/活跃天数）；分桶（日/周/月）；展示总量、活跃天数、连续天数、按类型分布与个人最佳。
- 图表：缺失桶显示 0；不做插值、平滑或预测；超过 500 个桶时提示缩小范围。
- 目标：新建/编辑周期目标；展示当前值、目标值、百分比与剩余天数；超过 100% 时进度条封顶，数值保留。
- 设置：管理员维护运动类型；成员可设置默认可见性与距离显示单位（公里/英里，存储仍用米）。
- Web：v1 不提供运动页；不显示运动统计、目标或日历叠加；服务端拒绝 Web 来源的 `fitness.*` 调用。
- 通知：`fitness` 不主动发提醒、排名或「久未运动」推送；需要提醒时由 `tasks` 创建。
- 健康联动：不适/受伤时提供「记录健康」跳转，调用 `health.records.create`；未安装 `health` 时隐藏入口；不自动复制运动数据，也不做诊断。

## 8. 数据与迁移

```sql
CREATE TABLE m_fitness_activity_type (
  id             TEXT PRIMARY KEY,
  code           TEXT NOT NULL,
  name           TEXT NOT NULL,
  category       TEXT NOT NULL,           -- cardio / strength / flexibility / sport / outdoor / other
  primary_metric TEXT NOT NULL,           -- duration / distance / steps
  icon           TEXT,
  color          TEXT,
  sort           INTEGER NOT NULL DEFAULT 0,
  builtin        INTEGER NOT NULL DEFAULT 0,
  archived_at    INTEGER,
  created_at     INTEGER NOT NULL,
  updated_at     INTEGER NOT NULL
);
CREATE UNIQUE INDEX idx_fitness_activity_code ON m_fitness_activity_type(code);
CREATE INDEX idx_fitness_activity_sort ON m_fitness_activity_type(archived_at, sort);

CREATE TABLE m_fitness_workout (
  id                 TEXT PRIMARY KEY,
  member_id          TEXT NOT NULL,
  activity_type_id   TEXT NOT NULL,
  started_at         INTEGER NOT NULL,     -- UTC epoch ms
  local_date         TEXT NOT NULL,        -- YYYY-MM-DD，成员本地日期
  duration_sec       INTEGER NOT NULL,
  distance_m         INTEGER,
  steps              INTEGER,
  calories_kcal      INTEGER,
  avg_heart_rate     INTEGER,
  max_heart_rate     INTEGER,
  rpe                INTEGER,
  visibility         TEXT NOT NULL DEFAULT 'private',
  source             TEXT NOT NULL DEFAULT 'manual',
  note               TEXT,
  client_id          TEXT,
  archived_at        INTEGER,
  created_at         INTEGER NOT NULL,
  updated_at         INTEGER NOT NULL
);
CREATE INDEX idx_fitness_workout_member_date ON m_fitness_workout(member_id, local_date DESC);
CREATE INDEX idx_fitness_workout_activity_date ON m_fitness_workout(activity_type_id, local_date DESC);
CREATE INDEX idx_fitness_workout_visibility ON m_fitness_workout(visibility, local_date DESC);
CREATE UNIQUE INDEX idx_fitness_workout_client
  ON m_fitness_workout(member_id, client_id) WHERE client_id IS NOT NULL;

CREATE TABLE m_fitness_goal (
  id               TEXT PRIMARY KEY,
  member_id        TEXT NOT NULL,
  activity_type_id TEXT,
  metric           TEXT NOT NULL,          -- duration_sec / distance_m / steps / workout_count / active_days
  period           TEXT NOT NULL,          -- day / week / month / year
  target_value     INTEGER NOT NULL,
  start_date       TEXT NOT NULL,          -- YYYY-MM-DD
  end_date         TEXT,
  status           TEXT NOT NULL DEFAULT 'active',
  created_at       INTEGER NOT NULL,
  updated_at       INTEGER NOT NULL
);
CREATE INDEX idx_fitness_goal_member ON m_fitness_goal(member_id, status);
CREATE UNIQUE INDEX idx_fitness_goal_active
  ON m_fitness_goal(member_id, COALESCE(activity_type_id, ''), metric, period)
  WHERE status = 'active';
```

- 时间戳用 UTC epoch ms；`local_date`、`start_date`、`end_date` 用 `YYYY-MM-DD`。
- 距离存米、时长存秒、热量存千卡、心率存 bpm；展示单位由客户端换算。
- `member_id`、`activity_type_id` 是逻辑引用，不建跨模块外键。
- 不建统计汇总表；v1 读时计算。数据量增大后再评估按日 rollup，但口径必须与实时统计一致。
- 运动记录默认归档，不物理删除；迁移只向前。
- 不存文件；v1 运动无附件。
- DB 不加密；运动数据依赖磁盘加密与权限。备份/导出包含运动数据，需按敏感数据处理。

## 9. 验收

- 新建跑步记录：类型、开始时间、本地日期、时长、距离、备注；保存后出现在历史与统计。
- 补记昨天、修改时长/距离、归档/恢复均正确；归档后默认列表与统计不返回。
- 同一成员的时间重叠被拒绝；相邻记录允许；修改时间/时长时重新校验。
- `private` 记录仅本人可见；`family` 记录家庭成员可见；`guest` 无权限；管理员无法读取私有记录。
- 查看他人统计只汇总 `family` 记录；UI 标注「仅统计家庭可见记录」。
- 统计总数、活跃天数、当前/最长连续天数与手工计算一致；今天没运动但昨天有运动时连续天数不归零。
- 时间线缺失桶补 0，不做插值；超过 500 个桶返回 `BUCKET_LIMIT`。
- 个人最佳按运动类型的主指标计算；并列时取更早记录。
- 每周 3 次、每月 20 公里等目标进度正确；超过 100% 保留真实数值；重复目标被拒绝。
- 归档运动类型后历史仍可显示/统计；新记录不能使用归档类型。
- `clientId` 重试不生成重复记录；事件不含时长、距离、心率与备注。
- 日志、审计与通知不含运动备注、心率、RPE 与精确运动时间；Web 无运动页面且拒绝 `fitness.*` RPC。
- 10,000 条运动记录下，按月/年统计与分页历史仍可用。

## 10. 开放问题

- 是否需要家庭总量、排行榜或挑战赛；v1 不做，避免隐私与公平性问题。
- 是否需要穿戴设备/HealthKit/Google Fit/Strava 导入；v1 只手动录入。
- 是否需要 GPS 轨迹、配速分段、海拔与地图；v1 不做。
- 是否需要力量训练的组数/次数/重量与动作库；v1 只记一次运动。
- 是否需要训练计划、教练建议与周期化；v1 不做，计划可放 `tasks`。
- 是否允许原生日历只读叠加运动；v1 不做，避免暴露运动时间。
- 是否需要运动记录导出/打印；v1 不做，后续按高敏感操作处理。
- 是否需要运动照片/视频；v1 不做，照片应归 `documents`。
- 目标是否需要家庭可见或多人共同目标；v1 仅本人。
- 是否允许目标↔任务关联；v1 不做，避免家庭可见任务暴露私有目标。
- 家长是否可代录孩子运动；v1 只能本人录入。
- 距离显示单位是否需要英里；v1 存储公制，距离可切换显示。
- 个人最佳是否只按主指标，还是每个指标都算；v1 只按主指标。
- 一次运动是否可关联多个成员；v1 每人各记一条。
- 心率/RPE 是否需要趋势、区间与训练负荷解释；v1 只存原始可选值。
- 自定义运动类型是否允许普通成员创建；v1 仅管理员。
- 是否需要数据保留期与自动归档；v1 手动归档，长期保留。
