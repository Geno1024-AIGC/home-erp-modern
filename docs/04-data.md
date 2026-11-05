# 04 · 数据层

## 1. 数据单元

恒星的 `data/` 是唯一需要备份的目录：

```
data/
  star.db              SQLite（WAL 时还有 -wal / -shm）
  files/               内容寻址附件库
    sha256/ab/cd/<hash>
  modules/             已安装模块包
    <moduleId>/<version>/
  trust/               发布者公钥
  devices/             设备公钥与元数据（也可放 DB）
  config/              恒星配置（非机密）
  secrets/             主密钥、敏感配置（0600）
  audit/               审计日志（可选）
  backups/             手动/定时备份
```

原则：

- **业务数据只进 SQLite 与 `files/`**；模块不得写 `data/` 根目录。
- 附件不进数据库，只存相对 key。
- 拷贝整个 `data/` 即可迁移；备份需保证 SQLite 一致性（见 §7）。

---

## 2. SQLite 配置

```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;      -- 可配置 FULL
PRAGMA foreign_keys = ON;
PRAGMA busy_timeout = 5000;
PRAGMA wal_autocheckpoint = 1000;
PRAGMA temp_store = MEMORY;
```

- 单写者：`Db.write {}` 串行化，事务尽量短。
- 多读者：WAL 下并发读；`Db.read {}` 不阻塞写。
- 连接策略：1 个写连接 + 少量读连接（建议 2–4）；避免每请求新建连接。
- 不使用 ORM / 代码生成；模块写 SQL，经 `Db` 执行。
- 大字段（> 64 KiB）禁止进 DB；放附件库。

---

## 3. Db API（概念）

```kotlin
interface Db {
    fun <T> read(block: (Connection) -> T): T
    fun <T> write(block: (Connection) -> T): T
    fun transaction(block: (Connection) -> Unit)
}

interface FileStore {
    fun put(stream: InputStream, expectedSha256: String? = null): FileRef
    fun open(ref: FileRef): InputStream
    fun addRef(conn: Connection, ref: FileRef, ownerType: String, ownerId: String, role: String = "attachment")
    fun removeRef(conn: Connection, ref: FileRef, ownerType: String, ownerId: String, role: String = "attachment")
    fun gc(dryRun: Boolean = false): GcResult   // 两阶段清理无引用且超过安全窗口的文件
}
```

- `Db` 不暴露连接生命周期给模块；事务边界由宿主管理。
- 查询结果由模块自行映射为 DTO；不要求注解。
- 预编译语句缓存由宿主实现；模块传 SQL + 参数。
- 迁移通过 `Migration(version, statements)` 声明，不直接执行 DDL。

---

## 4. 迁移

```kotlin
data class Migration(
    val version: Int,
    val statements: List<String>,
)
```

- 宿主表：

```sql
CREATE TABLE schema_migrations (
  module_id TEXT NOT NULL,
  version   INTEGER NOT NULL,
  checksum  TEXT NOT NULL,
  applied_at INTEGER NOT NULL,
  PRIMARY KEY (module_id, version)
);
```

- 执行顺序：按模块依赖拓扑排序；模块内按 `version` 升序。
- 每个迁移一个事务；失败则回滚并拒绝启动该模块。
- `checksum` 校验已应用迁移未被篡改；不一致 → 拒绝启动。
- 只向前；不自动降级。大版本升级前自动备份。
- 模块卸载默认保留表；「卸载并删除数据」需管理员二次确认。

---

## 5. 命名、ID 与类型

### 表名

`m_<moduleShort>_<entity>`，例如：

- `m_inventory_item`
- `m_tasks_task`
- `m_docs_document`
- `m_members_person`

原因：SQLite 没有 schema；用前缀避免模块冲突，也便于按模块导出/清理。

### 跨模块引用

- **不建跨模块外键**；模块必须可独立卸载。
- 存稳定的逻辑 ID（如 `personId`），由调用方校验存在性。
- 需要强一致时，通过 RPC/事件协作，不共享表。

### ID

- 业务实体用 **UUIDv7**（时间有序），存 `TEXT`（36 字符）或 `BLOB(16)`；默认 `TEXT` 便于调试。
- 设备 ID、模块 ID 等用固定字符串。
- 不使用自增整数作为跨模块引用。

### 时间

- 统一 UTC epoch milliseconds，存 `INTEGER`。
- 展示层按用户时区转换。
- 日期（无时间）用 `TEXT` `YYYY-MM-DD` 或 epoch day；模块自定，但需在 README 说明。

### 金额与数量

- 金额：整数最小货币单位（分）+ 币种；禁止浮点。
- 数量：整数 + 单位枚举；需要小数时用缩放整数（如 `qty_milli`）。
- 汇率/比例：整数基点或十进制字符串；禁止 `Double` 参与业务计算。

### 删除

- 默认硬删除；需要同步/审计的模块用 `deleted_at` 软删除。
- 软删除的表必须有部分索引或查询约束，避免遗漏过滤。
- 附件引用行随业务记录事务增删；不维护单独的计数器，引用数按 `file_refs` 统计。

### Web 会话表（宿主）

```sql
CREATE TABLE host_web_session (
  id               TEXT PRIMARY KEY,       -- 128-bit 随机 ID
  user_id          TEXT NOT NULL,
  token_hash       BLOB NOT NULL UNIQUE,   -- SHA-256(token)，32 字节
  csrf_hash        BLOB NOT NULL,          -- SHA-256(csrf)
  created_at       INTEGER NOT NULL,
  last_seen_at     INTEGER NOT NULL,
  idle_expires_at  INTEGER NOT NULL,
  expires_at       INTEGER NOT NULL,       -- 绝对过期
  user_agent       TEXT,
  remote_ip        TEXT,
  revoked_at       INTEGER
);
CREATE INDEX idx_web_session_user ON host_web_session(user_id);
CREATE INDEX idx_web_session_exp  ON host_web_session(expires_at);
```

- 令牌与 CSRF 只存哈希；明文只存在于 Cookie/响应，不落库、不写日志。
- 过期/撤销的行由宿主定时清理；`revoked_at` 非空立即失效。
- 密码哈希属于 `auth` 模块（`m_auth_user`），不在宿主表。
- 恢复备份后清空本表，所有 Web 会话强制重新登录。

---

## 6. 附件库

### 6.1 内容寻址

```
files/sha256/<前2位>/<次2位>/<完整 sha256>
```

- 写入：临时文件 → 计算 SHA-256 → `fsync` → 原子 rename 到目标路径。
- 相同内容自动去重。
- 引用表（宿主维护）：

```sql
CREATE TABLE file_refs (
  id          TEXT PRIMARY KEY,                 -- 引用 id，一次引用一行
  sha256      TEXT NOT NULL,
  owner_type  TEXT NOT NULL,                    -- "<module>.<entity>"，如 documents.document
  owner_id    TEXT NOT NULL,                    -- 业务实体主键
  role        TEXT NOT NULL DEFAULT 'attachment',
  created_at  INTEGER NOT NULL,
  UNIQUE (sha256, owner_type, owner_id, role)
);
CREATE INDEX idx_file_refs_sha ON file_refs(sha256);
CREATE INDEX idx_file_refs_owner ON file_refs(owner_type, owner_id);
```

引用规则：

- 一个文件可以被多条业务记录引用；同一实体同一 `role` 只建一条引用，重复上传幂等。
- `owner_type` 用 `<module>.<entity>`，不是模块 ID；`owner_id` 是该实体的主键。
- 只有**真正持有文件**的实体建引用。`documents.link` 把文档关联到 `meals.cook_log` 只是业务关联，不新增 `file_refs`；文件引用始终属于 `documents.document`。
- 引用行与业务记录在同一事务增删；软删除/归档不删引用，恢复后文件仍可用。
- 不维护引用计数器：引用数 = `COUNT(*) WHERE sha256 = ?`，避免计数漂移。
- 删除业务记录时必须在同一事务删除其引用；漏删由 GC 的孤儿引用检查兜底，但不能长期依赖。

### 6.2 GC（两阶段）

1. **标记候选**：磁盘上存在、但 `file_refs` 中无对应 `sha256`，且文件写入时间超过 24 小时的文件；同时报告 `file_refs` 中指向不存在文件的行。
2. **延迟删除**：候选至少再保留 24 小时（或等到下一次 GC），第二次扫描仍无引用才物理删除；每轮有数量上限、可 dry-run、写审计。指向缺失文件的引用行连续两轮仍缺失才清理，避免恢复窗口误删。

边界：

- 写入顺序是先落盘再建引用，「已落盘、事务未提交」的文件在 24 小时宽限期内不会被删。
- 宽限期必须大于最长事务与备份恢复耗时；恢复备份后先跑一次引用校验，再允许 GC。
- GC 与备份/恢复互斥；GC 运行期间不开始新的备份或恢复。
- GC 只认 `file_refs`，不遍历业务表猜测引用关系；有引用的文件永不删除。
- 卸载模块并选择「删除数据」时，先删除该模块前缀的引用（`owner_type` 以 `<module>.` 开头），再触发 GC；不影响其他模块。
- 文件权限 `0600`；目录 `0700`。

### 6.3 模块使用

```kotlin
val ref = ctx.files.put(input, expectedSha256 = req.sha256)
ctx.db.write { conn ->
    insertAttachment(conn, itemId, ref)
    ctx.files.addRef(conn, ref, ownerType = "documents.document", ownerId = documentId)
}
```

- `ownerType` 用 `<module>.<entity>`；`addRef` 对同一 `(sha256, ownerType, ownerId, role)` 幂等。
- `FileRef` 只含相对 key、sha256、大小、MIME；不含绝对路径。
- 禁止模块直接 `Files.write` 到 `data/files`。

---

## 7. 备份与恢复

### 7.1 一致性备份

1. 暂停写入（或获取短写锁）。
2. `VACUUM INTO 'backups/star-<timestamp>.db'`。
3. 复制 `files/`、`modules/`、`trust/`、`config/`（不含 `secrets/`，除非加密备份）。
4. 生成 `manifest.json`：文件列表、大小、SHA-256、恒星版本、模块版本。
5. 恢复写入。

- 备份可在线执行；`VACUUM INTO` 产生一致快照。
- 备份目录排除 `backups/` 自身，避免递归。
- 可选：定时备份 + 保留 N 份；失败告警写入审计。

### 7.2 恢复

1. 停止恒星。
2. 校验备份 `manifest.json` 与哈希。
3. 替换 `star.db`、`files/`、`modules/`、`trust/`、`config/`。
4. 启动；宿主校验 `schema_migrations` 与模块版本，并清空 `host_web_session`。
5. 若数据库版本高于当前宿主，拒绝启动并提示升级宿主。

### 7.3 迁移

- **完整迁移**：停机后拷贝整个 `data/`（含 `secrets/`），可保留恒星身份、LAN 证书与已配对设备。
- **仅数据备份**：默认不含 `secrets/`；恢复后恒星会生成新身份/证书，原生卫星需要重新配对或更新固定指纹。
- 换机器：拷贝 `data/`；LAN 地址变化需更新拓扑，证书不变则无需重新配对。
- 数据库文件与附件必须一起迁移；只拷 DB 会导致附件丢失。
- `secrets/` 与完整 `data/` 都按最高敏感级别保护，不得进入普通备份或版本库。

---

## 8. 并发与长任务

- 写事务目标 < 50ms；批量导入分批提交。
- 不要在事务内做网络/文件 IO（除附件原子写）。
- 大文件上传：先写临时文件，完成后单事务登记引用。
- 长查询用只读连接；必要时加索引或分页。
- 模块定时任务通过宿主调度器，避免自建线程池。

---

## 9. 索引与搜索

- 每个模块负责自己的索引；外键列、常用过滤列建索引。
- 全文搜索用 SQLite FTS5，表名 `m_<module>_<entity>_fts`；由模块维护触发器或手动更新。
- 搜索接口是模块 RPC，不暴露 SQL。

---

## 10. 敏感数据

- 设备公钥、角色、审计可明文存 DB。
- 设备私钥只在原生卫星；恒星不保存。
- Web 密码只存 Argon2id 哈希；Web 会话令牌/CSRF 只存 SHA-256 哈希。
- 模块敏感配置（如第三方 API token）用宿主 `ModuleConfig`，用主密钥加密后存 DB。
- 业务表不存完整证件号/银行卡号/CVV/PIN；证件模块只存掩码/后四位，完整信息放扫描件；扫描件按最高敏感级别保护。
- 水印分享副本默认不落盘；如落盘，按派生文件纳入 `file_refs` 与 GC，不得覆盖原文件。
- 健康记录属高敏感数据；默认 `private`，正文与附件不得写入日志，导出/附件下载按高敏操作处理。
- 条码映射可推断购买习惯，按普通敏感数据保护；日志与审计不记录完整条码，外部查询缓存不含凭证并按 TTL 清理。
- 价格历史与商家同样可推断消费习惯，按普通敏感数据保护；日志不记录完整价格明细，导出/批量导入按管理员操作处理。
- 智能家居驱动配置/设备 token/配对密钥放 `ModuleConfig` 或 `secrets/`，DB 只存 `config_ref`；门锁只读，不存开锁密钥。
- 主密钥 `data/secrets/master.key`（0600），首次启动生成；备份时单独处理。
- DB 文件本身不加密（v1）；依赖磁盘加密与文件权限。后续可评估 SQLCipher。

---

## 11. 导出

- 模块可注册 `export` RPC：输出 JSONL + 附件清单 + manifest。
- 宿主提供全量导出：`star.db` 逻辑导出（按表 JSONL）+ `files/`；排除 `host_web_session`、`secrets/`。
- 导入是显式管理操作；v1 只保证「同版本/可迁移版本」导入。
- 导出文件带 SHA-256 清单，便于校验。

---

## 12. 容量与限制

| 项 | 建议 |
|---|---|
| 单附件 | 流式处理，默认上限 2 GiB（可配） |
| 单行大字段 | ≤ 64 KiB，超出放附件 |
| 并发读连接 | 2–4 |
| 写事务 | < 50ms |
| WAL 文件 | 定期 checkpoint；备份前 `wal_checkpoint(TRUNCATE)` |
| Web 会话 | 默认 12h 绝对 / 30min 空闲；定时清理 |
| Web 上传 | 流式写临时文件，完成后登记引用；默认 2 GiB |
| DB 大小 | 家庭规模无硬限制；> 1 GiB 时评估归档 |
