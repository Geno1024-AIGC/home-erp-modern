# 模块：外部联系人（contacts）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`（可选）、`documents`（可选） ｜ 拥有：`m_contacts_contact`

## 1. 目的与非目标

**目的**：维护家庭成员之外的联系人：物业、维修工、开锁、医生、兽医、学校、邻居、亲戚、紧急联系人。为借出、账单、宠物、家庭手册与应急卡提供统一的「找谁」数据。

**非目标**：

- 不做 CRM、营销、通话记录、短信/聊天、通讯录同步。
- 不存密码、门禁码、Wi-Fi 密码、银行卡号或身份证号。
- 不做健康档案；医生/兽医的医疗记录仍归 `health` / `pets`。
- 不做账单金额与到期；账单归 `bills`。
- 不自动从手机通讯录导入，也不上传整本通讯录。

## 2. 用户场景

- 存物业电话、维修工、开锁师傅、常去的医院与兽医。
- 标记紧急联系人，急救卡只读引用。
- 借出工具时选择借出人；账单填写收款方。
- 宠物档案记录兽医联系人；维修任务记录师傅。
- 记录服务时间、报价备注与「下次找谁」，但不建工单。
- 联系方式变更时更新一处，其他模块按 `contact_id` 引用。

## 3. 边界与依赖

- 拥有外部联系人主数据；只引用 `owner_member_id`，不建跨模块外键。
- 其他模块只保存 `contact_id`；联系人归档后快照名称仍可显示。
- 家庭成员归 `members`；不要把家人重复建成联系人。
- 证件号码归 `identity`；医疗记录归 `health` / `pets`；文件归 `documents`。
- 默认家庭可见；可标 `private`，仅所有者与 `contacts.admin` 可见。
- 不存密码/门禁码；需要时只记「在密码管理器/实体册」，不记内容。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Contact` | `id`、`display_name`、`kind`、`organization`、`role_label`、`phone`、`email`、`address`、`notes`、`visibility`、`owner_member_id`、`archived_at` |

`kind`：`emergency`、`medical`、`vet`、`repair`、`property`、`neighbor`、`family`、`school`、`service`、`other`。

`visibility`：`family`、`private`。

规则：

- `kind` 只用于分组与默认图标，不限制字段。
- `display_name` 必填；`phone` / `email` 用自由文本，不做格式强校验。
- `private` 仅 `owner_member_id` 本人与 `contacts.admin` 可见；`family` 所有 `contacts.read` 成员可见。
- 紧急联系人建议 `visibility=family`；急救卡只读取可见记录。
- 归档不物理删除；默认列表不返回，历史引用仍可显示快照。
- 一个联系人只有一个电话字段；多个号码写进 `notes`，v1 不做多值表。
- 不做去重合并；同名联系人允许存在，由用户自己区分。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `contacts.read` | 查看可见联系人 | `member` |
| `contacts.write` | 新建/编辑自己的联系人 | `member` |
| `contacts.admin` | 管理所有联系人、恢复归档、处理私密记录 | `admin` |

- `guest` 无权限。
- `member` 默认只能编辑 `owner_member_id = 自己` 或家庭共享联系人；代改他人私密联系人需要 `contacts.admin`。
- 读取 RPC 在服务端叠加 `visibility`，不把私密联系人返回给无权成员。

## 6. RPC 与事件

- `contacts.list`：`{ kind?, query?, includeArchived?, limit?, cursor? }`，按可见性过滤。
- `contacts.get`
- `contacts.create`：`{ displayName, kind, organization?, roleLabel?, phone?, email?, address?, notes?, visibility?, ownerMemberId? }`
- `contacts.update`
- `contacts.archive` / `contacts.restore`
- 事件：`contacts.contact.changed { id, action }`，供借出、账单、宠物、家庭手册刷新缓存。

## 7. UI（原生 / Web）

- 原生：联系人列表（按类型分组）、详情、拨号/邮件入口、紧急联系人置顶。
- 详情显示类型、机构、角色、电话、地址、备注、可见性；私密记录有明确标记。
- 借出、账单、宠物、家庭手册中的联系人选择器只显示当前用户可见的联系人。
- Web：固定壳中的列表/详情/编辑；不做通讯录导入，不做批量导出。
- 急救卡/家庭手册只读引用；不在联系人页生成医疗建议。

## 8. 数据与迁移

```sql
CREATE TABLE m_contacts_contact (
  id              TEXT PRIMARY KEY,
  display_name    TEXT NOT NULL,
  kind            TEXT NOT NULL,
  organization    TEXT,
  role_label      TEXT,
  phone           TEXT,
  email           TEXT,
  address         TEXT,
  notes           TEXT,
  visibility      TEXT NOT NULL DEFAULT 'family',
  owner_member_id TEXT,
  archived_at     INTEGER,
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);
CREATE INDEX idx_contacts_kind ON m_contacts_contact(kind, archived_at);
CREATE INDEX idx_contacts_owner ON m_contacts_contact(owner_member_id, visibility);
```

- 电话/邮箱/地址用文本；v1 不做号码归一化与去重。
- 不存密码、门禁码、Wi-Fi 密码、银行卡号或完整证件号。
- 迁移只向前；删除默认归档。

## 9. 验收

- 新建紧急联系人、维修工、兽医、物业；按类型筛选正确。
- `private` 联系人仅所有者与管理员可见；普通成员看不到，也不能通过搜索绕过。
- 联系人归档后默认列表不返回；借出/账单中的历史引用仍显示快照名称。
- `guest` 无权限；`member` 可维护自己的联系人。
- 急救卡只读取可见的紧急联系人，不泄露私密记录。
- 日志与通知不含完整电话、地址与私密备注。

## 10. 开放问题

- 是否需要多个电话/邮箱与标签；v1 单电话 + 备注。
- 是否需要联系人分组、收藏与最近使用；v1 按 `kind` 分组。
- 是否允许从手机通讯录导入；v1 不做，避免上传整本通讯录。
- 是否需要联系人合并与去重；v1 由用户自行区分。
- 是否需要把家庭医生、兽医与健康/宠物记录双向关联；v1 只存 `contact_id`。
- 是否需要联系人级审计；v1 只做权限过滤。
