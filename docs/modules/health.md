# 模块：健康与医疗（health）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`documents`（可选）、`calendar`（可选）、`tasks`（可选） ｜ 拥有：`m_health_record`

## 1. 目的与非目标

**目的**：维护家庭成员的轻量健康档案：就诊、疫苗、用药、过敏、慢病、检查；支持私密/家庭可见，并关联报告、预约与提醒。

**非目标**：

- 不做诊断、治疗建议、急救服务、医疗保险理赔。
- 不做完整 EHR、影像归档、检验设备接入。
- 不存宠物健康；宠物归 `pets`。
- 不拥有预约与提醒；预约归 `calendar`，提醒归 `tasks` / 通知能力。
- 不存报告文件；报告/处方/影像由 `documents` 拥有。

## 2. 用户场景

- 记录一次就诊：日期、医院/医生、主诉、诊断、备注。
- 记录疫苗接种：疫苗名称、剂次、日期、接种机构。
- 记录当前/历史用药：药名、剂量、频次、起止日期。
- 记录过敏与慢病：严重程度、状态、注意事项。
- 上传检查报告、处方、影像报告并关联。
- 把复诊预约加进 `calendar`，把用药提醒交给 `tasks`。
- 私密记录只给本人和管理员看；家庭记录给家庭成员看。

## 3. 边界与依赖

- 拥有健康记录；只引用 `member_id`，不建跨模块外键。
- 人类健康专属；宠物健康在 `pets`，不混用。
- 预约/随访由 `calendar` 拥有；用药/复诊提醒由 `tasks` 拥有。
- 报告、处方、影像报告归 `documents`；`health` 只调用其 RPC。
- 健康记录默认 `private`（本人 + `health.admin`）；可显式改为 `family`。
- 高敏感：Web 默认只显示摘要；附件下载、导出、详细内容优先原生。
- 不把健康正文写日志、审计或错误信息。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `HealthRecord` | `id`、`member_id`、`kind`、`title`、`occurred_at`、`provider`、`status`、`visibility`、`summary`、`metadata_json`、`archived_at` |

`kind`：`visit`、`vaccination`、`medication`、`allergy`、`condition`、`lab`、`hospitalization`、`other`。

`status`：`active`、`resolved`、`historical`。

`visibility`：`private`、`family`。

规则：

- `private`：仅 `member_id` 本人与 `health.admin` 可见。
- `family`：所有拥有 `health.read` 的家庭成员可见。
- `metadata_json` 放类型相关字段，例如：
  - 用药：`{"dosage":"5mg","frequency":"每日一次","startDate":"2026-11-01"}`。
  - 疫苗：`{"dose":2,"vaccine":"..."}`。
  - 过敏：`{"severity":"moderate","reaction":"..."}`。
- `metadata_json` 仍按高敏感数据处理；禁止放文件内容或完整证件号。
- 允许同一成员多条同类记录；不做病种/药品唯一约束。
- 归档不物理删除；默认不返回，本人/管理员可查历史。
- 报告/处方通过 `documents.link` 关联到 `targetType="health.record"`。
- 预约/提醒不复制到 `health`；只保存必要的关联 ID 或由 UI 跳转。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `health.read` | 按可见性查看健康记录 | `member` |
| `health.write` | 新建/编辑自己的记录 | `member` |
| `health.admin` | 管理所有成员、恢复归档、处理私密记录 | `admin` |

- `health.read` 不是全量读取；RPC 必须叠加记录 `visibility`。
- `member` 默认只能写 `member_id = 自己` 的记录；代写他人需要 `health.admin`。
- 附件下载由 `documents` 再校验；`health.read` 不等于自动获得文件下载权。

## 6. RPC 与事件

- `health.records.list`：按成员、类型、日期、状态、可见性筛选。
- `health.records.get`
- `health.records.create`
- `health.records.update`
- `health.records.archive` / `health.records.restore`
- `health.records.listForMember`：成员详情页时间线。
- 事件：`health.record.changed { id, memberId, action }`。

所有读取 RPC 在服务端做可见性过滤；禁止客户端拉全量后自行隐藏。搜索只支持标题、机构、类型、日期，不支持正文全文。

## 7. UI（原生 / Web）

- 原生：成员详情「健康」页签、时间线、详情、编辑、私密/家庭切换、附件。
- Web：固定壳中的摘要列表/详情；默认不显示附件预览，下载需二次确认与审计。
- 创建/编辑表单按 `kind` 显示不同字段；复杂字段用 `metadata_json`，UI 做结构化表单。
- 复诊/预约提供「加入日历」；用药/复诊提供「创建提醒」；两者都调用对应模块 RPC。
- Web 页面提示：健康数据经行星网关，行星可见明文；高敏感操作建议原生。

## 8. 数据与迁移

```sql
CREATE TABLE m_health_record (
  id            TEXT PRIMARY KEY,
  member_id     TEXT NOT NULL,
  kind          TEXT NOT NULL,
  title         TEXT NOT NULL,
  occurred_at   TEXT,
  provider      TEXT,
  status        TEXT,
  visibility    TEXT NOT NULL DEFAULT 'private',
  summary       TEXT,
  metadata_json TEXT,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX idx_health_member ON m_health_record(member_id, occurred_at DESC);
CREATE INDEX idx_health_kind ON m_health_record(kind, occurred_at DESC);
CREATE INDEX idx_health_visibility ON m_health_record(visibility, archived_at);
```

- 日期用 `YYYY-MM-DD`；时间戳用 UTC epoch milliseconds。
- 迁移只向前；删除默认归档。
- 表结构不存文件；报告/处方在 `documents` / `file_refs`。
- v1 DB 不加密；健康数据依赖磁盘加密与权限，后续评估 SQLCipher/字段加密。

## 9. 验收

- 新增就诊、疫苗、用药、过敏、慢病、检查记录；不同 `kind` 字段正确。
- `private` 记录仅本人与管理员可见；`family` 记录家庭成员可见。
- 普通成员不能代写他人记录；管理员可以。
- 上传报告并关联；删除关联不删文件；下载有审计。
- 预约能加入日历；提醒能创建任务；健康模块不复制其状态。
- Web 只显示摘要；附件下载需确认；日志无健康正文。
- 归档记录默认不返回；本人/管理员可查历史。

## 10. 开放问题

- 是否需要结构化用药计划（剂量/频次/提醒）而不是 `metadata_json`。
- 是否需要结构化检验值、趋势图、影像报告。
- 是否需要更细的 ACL（成人隐私、未成年人监护、管理员破窗）。
- 是否需要健康数据字段级加密；v1 依赖磁盘加密。
- 是否需要导出/打印健康摘要；导出属于高敏感操作。
- 是否需要保险、体检套餐、医院通讯录；可能放 `contacts` 或后续模块。
