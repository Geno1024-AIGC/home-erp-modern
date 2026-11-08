# 模块：消息与公告（messages）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`auth`、`documents`（可选，后续附件） ｜ 拥有：`m_messages_*`

## 1. 目的与非目标

**目的**：给家庭成员一个轻量的沟通入口：家庭公屏、两人私聊、未读数与历史记录；管理员可在主页发横幅公告。消息只服务当前家庭，不接外部 IM。

**非目标**：

- 不做群组/频道管理、语音视频、贴纸、机器人、表情回应、消息搜索（v1）。
- 不做端到端加密私聊；恒星数据库仍存明文，仅做应用层可见性隔离。
- 不做推送基础设施；只发事件，由宿主通知能力消费。
- 不做公告定向、已读回执、强制确认、投票。
- 不存附件/图片；后续走 `documents`，v1 只发文本。
- 不做社交动态、公开分享、跨家庭联邦。

## 2. 用户场景

- 在家庭公屏说「今晚谁接孩子」「牛奶没了」，所有成员可见。
- 和某个成员私聊，讨论不适合放公屏的事。
- 打开消息页看到公屏与私聊的未读数；点进去从上次位置继续看。
- 往上翻历史消息；断线重连后补齐漏掉的消息。
- 发错消息，撤回后显示「已撤回」，不留下正文。
- 管理员在主页发「周六停水」横幅，设置生效时间与级别；成员可关闭，重要公告不可关闭。
- Web 上查看公屏与公告；私聊只在原生客户端。

## 3. 边界与依赖

- 拥有会话、会话成员、消息、已读位置、公告与公告关闭记录。
- 只引用 `member_id`；不建跨模块外键，不复制成员资料。
- 权限由 `auth` 裁决；`messages` 只声明并校验自己的权限。
- 公屏全家庭可见；私聊仅两位参与者可见，管理员没有读取入口。
- 公屏消息管理员可撤回；私聊消息只有发送者本人可撤回。
- 公告属于 `messages`，不是 `calendar` 事件或 `tasks` 任务；不参与完成/提醒调度。
- 附件、图片、语音后续归 `documents`；v1 不提供。
- 通知由宿主能力发送；默认不显示消息正文，只提示「有新消息」。
- Web 只提供公屏与公告；私聊仅原生，避免行星看到 Web 明文。
- 恒星数据库存消息明文；备份、DB 管理员仍可接触，隐私边界是应用层而非加密层。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Conversation` | `id`、`kind`、`title`、`direct_key`、`last_seq`、`last_message_at`、`archived_at` |
| `ConversationMember` | `conversation_id`、`member_id`、`joined_at` |
| `Message` | `id`、`conversation_id`、`seq`、`sender_id`、`sender_name`、`kind`、`text`、`client_id`、`created_at`、`redacted_at`、`redacted_by` |
| `ReadState` | `conversation_id`、`member_id`、`last_seq`、`updated_at` |
| `Announcement` | `id`、`title`、`body`、`level`、`starts_at`、`ends_at`、`dismissible`、`created_by`、`archived_at` |
| `AnnouncementDismissal` | `announcement_id`、`member_id`、`dismissed_at` |

`Conversation.kind`：`public`（家庭公屏）、`direct`（两人私聊）。

`Message.kind`：`text`、`system`（预留，v1 不发系统消息）。

`Announcement.level`：`info`、`warning`、`critical`。

规则：

- 每个家庭只有一个 `public` 会话；首次安装时创建，标题固定为产品文案，不写成员行。
- `direct` 会话由两位成员的 ID 排序生成 `direct_key`，唯一；不能和自己私聊；会话成员表写两行。
- 消息 `seq` 在会话内单调递增，历史分页与断线补齐都以 `seq` 为准，不依赖时间戳。
- 发送必须带客户端生成的 `client_id`；同一会话同一 `client_id` 幂等，重试不产生重复消息；内容不一致时拒绝。
- 同一成员发送需限流（默认 30 条/分钟，可配置）；超限返回明确错误，不静默丢弃。
- 文本为纯文本，最长 4000 字符；保留换行，不渲染 HTML/Markdown；Web 用 `textContent`。
- 公告标题最长 80 字符、正文最长 1000 字符；只存纯文本。
- 消息不编辑；撤回把 `text` 清空、记录 `redacted_at` / `redacted_by`，保留时间线与「已撤回」占位。
- 发送者可撤回自己的消息；管理员只能撤回公屏消息，不能撤回私聊。
- `sender_name` 在发送时快照，成员改名或移除后历史仍可显示。
- 已读位置按成员共享，多设备一致；v1 不提供逐条「已读」回执。
- 未读数按 `last_seq` 之后、非本人发送的消息计算；撤回消息仍占位。
- 私聊可见性只认 `ConversationMember`；管理员、其他成员一律拒绝。
- 公屏发送需要 `messages.post`；私聊发送需要 `messages.direct` 且必须是会话成员。
- Web 传输只允许公屏与公告；私聊会话的列表、读取、发送在服务端直接拒绝，不能只靠 UI 隐藏。
- 成员被移除后失去私聊访问；另一位成员保留历史。
- 公告按 `starts_at` / `ends_at` 判断生效；`ends_at` 为空表示长期有效；`ends_at` 必须晚于 `starts_at`。
- 公告排序：`critical` > `warning` > `info`，同级按 `starts_at` 倒序；过滤已关闭项后，主页展示第一条并提示剩余数量。
- 公告只有 `dismissible=true` 才能被成员关闭；`critical` 强制不可关闭，创建/更新时校验。
- 公告归档不删除；已关闭记录保留。
- 消息、公告正文不写日志、审计正文或推送预览。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `messages.read` | 查看公屏、自己的私聊、公告 | `member` |
| `messages.post` | 在公屏发言 | `member` |
| `messages.direct` | 发起与查看两人私聊 | `member` |
| `messages.announce` | 发布、编辑、归档主页公告 | `admin` |
| `messages.admin` | 撤回公屏消息、管理公屏保留策略 | `admin` |

- `guest` 默认不能读公屏与公告；如需访客可见，后续单独评估。
- 私聊只受 `messages.direct` + 参与者双重限制；管理员不因 `messages.admin` 获得私聊读取权。
- Web 只允许 `messages.read` / `messages.post` 的公屏与公告；`messages.direct` 在 Web 拒绝。
- 公告发布、撤回公屏消息写入审计，但审计不含正文。

## 6. RPC 与事件

- `messages.conversations.list`：公屏 + 当前成员参与的私聊，带最后一条摘要与未读数。
- `messages.conversations.get`
- `messages.conversations.openDirect`：`{ memberId }`，幂等返回或创建两人会话；Web 调用直接拒绝。
- `messages.messages.list`：`{ conversationId, beforeSeq?, afterSeq?, limit? }`，按 `seq` 升序返回；`limit` 上限 100。
- `messages.messages.send`：`{ conversationId, text, clientId }`，返回消息与 `seq`；公屏需 `messages.post`，私聊需 `messages.direct` 且为会话成员。
- `messages.messages.redact`：`{ messageId }`；发送者或公屏管理员。
- `messages.read.mark`：`{ conversationId, lastSeq }`；只能前进，不能超过会话 `last_seq`。
- `messages.unread.counts`
- `messages.announcements.active`：返回未归档且处于生效时间窗内的公告，按级别与 `starts_at` 排序，带当前成员的 `dismissed` 标记；主页过滤已关闭项后取第一条。
- `messages.announcements.list`：管理员查看全部/已归档。
- `messages.announcements.create` / `update` / `archive`：`ends_at` 必须晚于 `starts_at`；`level=critical` 时强制 `dismissible=false`。
- `messages.announcements.dismiss`：`{ announcementId }`；仅 `dismissible=true`，`critical` 一律拒绝（服务端校验）。
- 事件：
  - `messages.message.created { conversationId, seq, messageId }`
  - `messages.message.redacted { conversationId, seq, messageId }`
  - `messages.read.changed { conversationId, memberId, lastSeq }`
  - `messages.announcement.changed { announcementId, action }`

不提供：

- 不提供 `messages.messages.search`、`messages.messages.export`；后续再加。
- 不提供附件上传；后续接 `documents`。
- 不提供逐条已读回执、正在输入、在线状态。
- 不提供管理员读取私聊、跨会话审计正文。
- 不提供离线发件箱；离线发送直接失败，不做本地排队。

## 7. UI（原生 / Web）

- 原生：导航「消息」；公屏/私聊两个入口，列表显示最后一条、时间、未读数。
- 会话页：从旧到新加载，向上分页；新消息实时追加；断线重连按 `seq` 补齐并去重。
- 输入框：纯文本、换行、发送失败保留草稿；不做离线队列。
- 长按消息：撤回（本人）、复制；公屏管理员多一个「管理员撤回」。
- 私聊列表不显示对方在线状态；不显示「已读」。
- 主页横幅：调 `messages.announcements.active`；按级别着色，显示标题与正文，`critical` 不可忽略；可关闭时显示关闭按钮；多条时提示「还有 N 条」。
- 管理员公告页：标题、正文、级别、生效/结束时间、是否可关闭、归档。
- Web：固定壳中的「消息」页只展示公屏与公告；隐藏私聊入口，直接调用私聊 RPC 返回权限错误。
- 消息正文用纯文本渲染；不把用户输入拼进 HTML。
- 通知默认只显示「有新消息」，不显示正文；用户可在设置中关闭。

## 8. 数据与迁移

```sql
CREATE TABLE m_messages_conversation (
  id              TEXT PRIMARY KEY,
  kind            TEXT NOT NULL,          -- public / direct
  title           TEXT,
  direct_key      TEXT,
  last_seq        INTEGER NOT NULL DEFAULT 0,
  last_message_at INTEGER,
  created_at      INTEGER NOT NULL,
  archived_at     INTEGER
);
CREATE UNIQUE INDEX idx_messages_conv_public
  ON m_messages_conversation(kind) WHERE kind = 'public';
CREATE UNIQUE INDEX idx_messages_conv_direct
  ON m_messages_conversation(direct_key) WHERE direct_key IS NOT NULL;

CREATE TABLE m_messages_conversation_member (
  conversation_id TEXT NOT NULL,
  member_id       TEXT NOT NULL,
  joined_at       INTEGER NOT NULL,
  PRIMARY KEY (conversation_id, member_id)
);
CREATE INDEX idx_messages_conv_member ON m_messages_conversation_member(member_id);

CREATE TABLE m_messages_message (
  id              TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL,
  seq             INTEGER NOT NULL,
  sender_id       TEXT NOT NULL,
  sender_name     TEXT NOT NULL,
  kind            TEXT NOT NULL DEFAULT 'text',
  text            TEXT,
  client_id       TEXT NOT NULL,
  created_at      INTEGER NOT NULL,
  redacted_at     INTEGER,
  redacted_by     TEXT,
  UNIQUE (conversation_id, seq),
  UNIQUE (conversation_id, client_id)
);
CREATE INDEX idx_messages_message_conv ON m_messages_message(conversation_id, seq);

CREATE TABLE m_messages_read (
  conversation_id TEXT NOT NULL,
  member_id       TEXT NOT NULL,
  last_seq        INTEGER NOT NULL DEFAULT 0,
  updated_at      INTEGER NOT NULL,
  PRIMARY KEY (conversation_id, member_id)
);

CREATE TABLE m_messages_announcement (
  id          TEXT PRIMARY KEY,
  title       TEXT NOT NULL,
  body        TEXT NOT NULL,
  level       TEXT NOT NULL,              -- info / warning / critical
  starts_at   INTEGER NOT NULL,
  ends_at     INTEGER,
  dismissible INTEGER NOT NULL DEFAULT 1,
  created_by  TEXT NOT NULL,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_messages_announcement_active
  ON m_messages_announcement(archived_at, starts_at, ends_at);

CREATE TABLE m_messages_announcement_dismissal (
  announcement_id TEXT NOT NULL,
  member_id       TEXT NOT NULL,
  dismissed_at    INTEGER NOT NULL,
  PRIMARY KEY (announcement_id, member_id)
);
```

- `seq` 分配在同一事务内递增 `last_seq`，并写消息；`UNIQUE(conversation_id, seq)` 兜底。
- `client_id` 由客户端生成（UUID）；同会话唯一，重试返回原消息。
- `sender_name` 是发送时快照；不随成员改名回写。
- `direct_key` 用排序后的两个 `member_id` 拼接；公屏没有成员行。
- 公告时间用 UTC epoch milliseconds；展示按家庭时区。
- 私聊成员行在移除成员时删除；消息与快照保留。
- 迁移只向前；不物理删除消息，撤回保留占位；公告归档保留。

## 9. 验收

- 公屏发送后其他在线成员实时收到；离线重连后按 `seq` 补齐，无重复、无缺口。
- 两人私聊只有双方可见；第三方成员与管理员读取均被拒绝。
- 同一成员重复打开私聊只得到一个会话；同一 `client_id` 重试不重复发消息。
- 历史分页按 `seq` 正确，向上翻页不跳消息；1 万条消息仍可分页加载。
- 未读数正确；标记已读只前进；多设备共享已读位置。
- 撤回后正文清空，双方看到「已撤回」；管理员不能撤回私聊。
- 成员被移除后无法再读私聊；另一位成员仍可查看历史。
- 公告按时间窗生效；级别排序正确；可关闭公告按成员关闭；不可关闭公告无法关闭。
- 主页横幅只显示第一条并提示剩余数量；归档后消失。
- Web 可看公屏/公告，私聊入口不存在且 RPC 被拒绝。
- 日志、审计、通知不含消息/公告正文；私聊不出现在管理员审计。
- 纯文本渲染：HTML 标签只显示为文本，不执行。

## 10. 开放问题

- 私聊是否需要端到端加密；v1 只做应用层隔离，恒星仍存明文。
- Web 是否永远不提供私聊，还是后续加重新认证/端到端后再开放。
- 是否支持图片/文件消息；后续通过 `documents` 关联到 `messages.message`。
- 是否需要消息搜索、导出、保留期；v1 不做。
- 是否需要多个公屏/群组；v1 只有一个家庭公屏。
- 公告是否需要定向（按角色/成员）、强制确认、定时推送。
- 是否需要 @提及、表情回应、置顶消息。
- `guest` 是否可读公屏或公告。
- 通知是否允许显示正文预览；默认不显示。
