# 模块：奖项与证书（achievements）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_achievements_record`

## 1. 目的与非目标

**目的**：记录家庭成员的奖项、荣誉、证书与资格，形成按成员/年份的时间线，并可关联证书扫描件。

**非目标**：

- 不做简历/CV 生成、证书验真、技能矩阵。
- 不存文件；证书扫描件、奖状照片由 `documents` 拥有。
- 不内置到期提醒调度；需要时通过事件让 `tasks` 生成提醒。
- 不替代 `members` 的人员资料。

## 2. 用户场景

- 孩子获得学校奖项：记录奖项、颁发方、时间、名次，附奖状照片。
- 父母获得公司奖项或荣誉。
- 父母考取证书/资格：记录证书编号、有效期，附扫描件。
- 在成员详情查看荣誉时间线；按年份、类型、颁发方筛选。
- 证书到期前收到提醒（可选，走 `tasks`）。

## 3. 边界与依赖

- 拥有奖项/证书记录；只引用 `member_id`，不建跨模块外键。
- 一条记录对应一个成员；共同获奖可建多条，后续再评估多成员关联。
- 证书文件与「记录↔文件」关联由 `documents` 拥有；`achievements` 只调用其 RPC。
- 成员归档不删除记录；记录仍显示，标记为归档成员。
- 不做证书验真；外部验证链接 v1 只作为文本字段，不抓取。
- 到期提醒不内置调度；`achievements.record.changed` 可供 `tasks` 订阅。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `AchievementRecord` | `id`、`member_id`、`title`、`kind`、`issuer`、`awarded_at`、`category`、`level`、`rank`、`credential_no`、`verify_url`、`expires_at`、`description`、`archived_at` |

`kind`：`award`（奖项）、`honor`（荣誉）、`certificate`（证书）、`license`（资格/执照）、`other`。

规则：

- `award` / `honor` 通常没有 `expires_at`；`certificate` / `license` 可选有效期。
- `expires_at` 只存日期；「已过期」是查询态，不改记录状态。
- `level` 为自由文本（学校/公司/区/市/省/国家/国际等），不强制枚举。
- 同一成员、同一奖项可在不同年份/颁发方重复记录；不做唯一约束。
- 归档不物理删除；历史仍可按成员查询。
- 证书编号、验证链接可能敏感；默认只对家庭成员可见。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `achievements.read` | 查看奖项/证书与时间线 | `member` |
| `achievements.write` | 新建、编辑、归档 | `member` |
| `achievements.admin` | 恢复归档、处理敏感记录 | `admin` |

证书扫描件的下载权限由 `documents` 再次校验；`achievements.read` 不等于自动获得文件下载权。

## 6. RPC 与事件

- `achievements.list`：按成员、类型、年份、颁发方、关键字筛选。
- `achievements.get`
- `achievements.create`
- `achievements.update`
- `achievements.archive` / `achievements.restore`
- `achievements.listForMember`：成员详情页时间线。
- 事件：`achievements.record.changed { id, memberId, action }`。

列表默认按 `awarded_at` 倒序；没有日期的记录按创建时间倒序。

## 7. UI（原生 / Web）

- 原生：成员详情「荣誉/证书」页签、全局时间线、编辑表单、证书到期标记。
- Web：固定壳中的列表/详情/编辑；不做批量导入。
- 表单可直接上传证书扫描件：先 `documents.upload`，再 `documents.link` 到 `targetType="achievements.record"`，`role="certificate"` 或 `role="evidence"`。
- 到期提醒入口显示关联 `tasks`；没有提醒时提供「创建提醒」动作（可选）。

## 8. 数据与迁移

```sql
CREATE TABLE m_achievements_record (
  id            TEXT PRIMARY KEY,
  member_id     TEXT NOT NULL,
  title         TEXT NOT NULL,
  kind          TEXT NOT NULL,
  issuer        TEXT,
  awarded_at    TEXT,
  category      TEXT,
  level         TEXT,
  rank          TEXT,
  credential_no TEXT,
  verify_url    TEXT,
  expires_at    TEXT,
  description   TEXT,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX idx_achievements_member ON m_achievements_record(member_id, awarded_at DESC);
CREATE INDEX idx_achievements_kind ON m_achievements_record(kind, awarded_at DESC);
```

- 日期用 `YYYY-MM-DD` 文本；时间戳用 UTC epoch milliseconds。
- 迁移只向前；删除默认归档。
- 不存文件路径；文件引用在 `documents` / `file_refs`。

## 9. 验收

- 为成员新增奖项、荣誉、证书、资格；列表与成员时间线正确。
- 按年份、类型、颁发方、关键字筛选。
- 证书有效期正确显示「有效/已过期」；不修改原始记录。
- 上传证书扫描件并关联；删除关联不删文件。
- 成员归档后记录仍可查，并标记归档成员。
- `guest` 无权限；`member` 可维护；`admin` 可恢复归档。
- 原生与 Web 页面行为一致。

## 10. 开放问题

- 共同获奖是否需要一条记录关联多个成员。
- 证书到期是否由 `tasks` 自动生成周期提醒，还是手动创建。
- 是否需要证书编号/验证链接的敏感级别与脱敏显示。
- 是否需要 OCR 自动提取证书名称、编号、有效期。
- 是否要把「奖项/荣誉」与「证书/资格」拆成两个模块；v1 先用 `kind` 区分。
