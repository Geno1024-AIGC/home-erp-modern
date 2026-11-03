# 模块：证件与卡片（identity）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`documents`（可选）、`tasks`（可选） ｜ 拥有：`m_identity_record`

## 1. 目的与非目标

**目的**：登记家庭成员的身份证、护照、驾驶证、港澳/台湾通行证、居住证、社保卡、银行卡等证件与卡片，管理有效期、状态和扫描件。

**非目标**：

- 不做密码管理器、支付工具、银行卡鉴权。
- 不存 CVV、PIN、密码、磁道/芯片数据。
- 不存完整证件号/银行卡号；v1 只存掩码与后四位，完整信息看扫描件。
- 不做证件验真、自动识别或联网核验。
- 不替代 `documents`：扫描件、照片和文件关联仍归 `documents`。

## 2. 用户场景

- 记录身份证、护照、驾驶证、港澳通行证、台湾通行证、居住证、社保卡。
- 记录银行卡：银行、卡组织、后四位、有效期、持卡人。
- 查看哪些证件即将过期，或已经丢失/注销。
- 在成员详情查看「证件与卡片」清单。
- 打开对应扫描件；扫描件由 `documents` 管理。
- 证书到期前提醒（可选，走 `tasks`）。

## 3. 边界与依赖

- 拥有证件/卡片的结构化记录；只引用 `member_id`，不建跨模块外键。
- 扫描件、照片由 `documents` 拥有；`identity` 只调用 `documents` RPC。
- 一条记录对应一个成员；联名卡/共享卡 v1 先各自登记，后续再评估多成员关联。
- 证件号/卡号只存掩码与后四位；不提供「查看完整号码」RPC。
- 到期是查询态；丢失、注销、换发是显式状态。
- `metadata_json` 只放非秘密字段（银行名、卡组织、准驾车型、通行证类型等）；禁止放完整号码、CVV、PIN。
- Web 路径行星可见明文；查看扫描件、导出等高敏感操作优先原生。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `IdentityRecord` | `id`、`member_id`、`kind`、`label`、`holder_name`、`number_masked`、`number_last4`、`issuer`、`country`、`region`、`issued_at`、`expires_at`、`status`、`metadata_json`、`notes`、`archived_at` |

`kind`：`id_card`、`passport`、`driver_license`、`travel_permit`、`residence_permit`、`bank_card`、`social_security`、`other`。

`status`：`active`、`lost`、`revoked`、`replaced`。

规则：

- 不存完整号码。`number_masked` 示例：`110***********1234`；`number_last4` 用于快速辨认。
- `metadata_json` 示例：
  - 银行卡：`{"bankName":"招商银行","network":"UnionPay","cardType":"debit"}`。
  - 驾驶证：`{"licenseClass":"C1"}`。
  - 通行证：`{"permitType":"HK"}`。
- 身份证、护照等号码即使脱敏也按敏感数据对待；列表默认只显示后四位。
- 丢失/注销/换发不删除记录；换发可新建记录并把旧记录标为 `replaced`。
- 过期是 `expires_at < today` 的派生状态，不写回 `status`。
- 扫描件关联由 `documents` 负责，`role` 使用 `id_scan` 或 `card_scan`。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `identity.read` | 查看脱敏记录与有效期 | `member` |
| `identity.write` | 新建、编辑、归档 | `member` |
| `identity.admin` | 恢复归档、处理敏感记录 | `admin` |

扫描件下载由 `documents` 再校验；`identity.read` 不等于自动获得文件下载权。查看/下载扫描件必须写审计日志。

## 6. RPC 与事件

- `identity.list`：按成员、类型、状态、到期范围筛选。
- `identity.get`
- `identity.create`
- `identity.update`
- `identity.archive` / `identity.restore`
- `identity.listForMember`：成员详情页清单。
- 事件：`identity.record.changed { id, memberId, action }`。

没有 `identity.reveal`；RPC 不返回完整号码。搜索只支持标签、颁发方、后四位、类型，不支持完整号码检索。

## 7. UI（原生 / Web）

- 原生：成员详情「证件与卡片」页签、列表、详情、编辑、到期/丢失标记。
- Web：固定壳中的脱敏列表与编辑；默认不显示扫描件预览。
- 上传扫描件：`documents.upload` 后关联到 `targetType="identity.record"`，`role="id_scan"` 或 `role="card_scan"`。
- 到期提醒入口显示关联 `tasks`；没有提醒时提供「创建提醒」动作（可选）。
- Web 页面显示提示：证件数据经行星网关，行星可见明文；敏感操作建议原生。

## 8. 数据与迁移

```sql
CREATE TABLE m_identity_record (
  id            TEXT PRIMARY KEY,
  member_id     TEXT NOT NULL,
  kind          TEXT NOT NULL,
  label         TEXT NOT NULL,
  holder_name   TEXT,
  number_masked TEXT,
  number_last4  TEXT,
  issuer        TEXT,
  country       TEXT,
  region        TEXT,
  issued_at     TEXT,
  expires_at    TEXT,
  status        TEXT NOT NULL DEFAULT 'active',
  metadata_json TEXT,
  notes         TEXT,
  archived_at   INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX idx_identity_member ON m_identity_record(member_id, kind);
CREATE INDEX idx_identity_expiry ON m_identity_record(expires_at);
```

- 日期用 `YYYY-MM-DD`；时间戳用 UTC epoch milliseconds。
- 迁移只向前；删除默认归档。
- 表结构不提供完整号码列；如未来需要，必须先有宿主加密字段能力与迁移方案。
- 扫描件在 `files/`，由 `documents` / `file_refs` 管理。

## 9. 验收

- 为成员新增身份证、护照、驾驶证、通行证、社保卡、银行卡。
- 列表只显示掩码/后四位；API 与数据库都没有完整号码列。
- 银行卡拒绝 CVV/PIN/磁道等字段；`metadata_json` 不含秘密。
- 有效期正确显示「有效/即将过期/已过期」；丢失/注销/换发状态正确。
- 上传扫描件并关联；删除关联不删文件；查看/下载有审计。
- `guest` 无权限；`member` 可维护；`admin` 可恢复归档。
- Web 默认只显示脱敏信息；原生可查看扫描件。

## 10. 开放问题

- 是否需要宿主提供加密字段，以保存完整证件号/银行卡号。
- 银行卡是否只保留后四位，还是允许用户录入完整 PAN（需要加密字段与审计）。
- 是否需要 OCR 自动填充证件类型、有效期、后四位。
- 联名卡/家庭共享卡是否需要多成员关联。
- 证件到期是否由 `tasks` 自动生成提醒。
- 是否需要导出/打印「证件清单」；导出属于高敏感操作。
