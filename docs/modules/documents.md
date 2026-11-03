# 模块：文档与说明书（documents）

> 状态：draft ｜ 类型：business ｜ 依赖：— ｜ 拥有：`m_documents_doc`、`m_documents_link`、`files/`

## 1. 目的与非目标

**目的**：保存家里的文件与元数据，并关联到物品、位置、任务、奖项/证书、证件/卡片等实体。典型场景是电器说明书，也包括保修卡、发票、奖状照片、证书扫描件、身份证/银行卡扫描件。

**非目标**：

- 不做完整 DMS（版本控制、审批、流程）。
- 不复制物品/位置数据；只存 `target_type` + `target_id`。
- v1 不做 OCR/全文搜索；先按标题、类型、语言、关联实体搜索。
- 不提供公开链接；下载必须经过鉴权。

## 2. 用户场景

- 给新买的电视上传说明书 PDF，关联到该电视。
- 同一份说明书关联同型号的两台电器。
- 按标题/型号/语言找说明书。
- 在物品详情页直接打开说明书、保修卡、发票。
- 归档旧文件，但保留关联历史。

## 3. 边界与依赖

- 拥有文件、文档元数据、实体关联。
- 不拥有物品/位置/任务；只保存逻辑引用。
- 文件内容寻址由宿主 `FileStore` 负责；`documents` 只存 `file_key` 与哈希。
- 关联表由 `documents` 拥有；其他模块通过 `documents.listFor` 查询，不直接读表。
- 权限由 `auth` 统一裁决；文档可继承目标实体的可见性，但不复制权限。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Document` | `id`、`title`、`kind`、`file_key`、`mime`、`size`、`sha256`、`language`、`revision`、`notes`、`created_by`、`archived_at` |
| `DocumentLink` | `document_id`、`target_type`、`target_id`、`role`、`created_at` |

`kind`：`manual`、`receipt`、`warranty`、`photo`、`other`。

`target_type`：`inventory.item`、`locations.node`、`tasks.task`、`achievements.record`、`identity.record`。

`role`：`manual`、`photo`、`receipt`、`warranty`、`certificate`、`evidence`、`id_scan`、`card_scan`、`attachment`。

规则：

- 一份文档只有一个主文件；多页说明书用一个 PDF，不拆成多份。
- 文档可关联多个实体（多对多）；同一物品可有多份文档（不同语言/版本）。
- 删除关联不删文件；文件由 `FileStore` GC 回收。
- 归档文档保留关联；默认搜索不返回归档文档。
- 文档版本用 `revision` 文本，不做自动版本链；需要历史时上传新文档并归档旧文档。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `documents.read` | 查看/下载文档 | `member` |
| `documents.write` | 编辑元数据、归档 | `member` |
| `documents.upload` | 上传文件 | `member` |
| `documents.link` | 关联/解除关联 | `member` |
| `documents.admin` | 删除文件、处理孤儿 | `admin` |

敏感文档（证件、保单）后续可加 `visibility` 字段；v1 先按模块权限控制。

## 6. RPC 与事件

- `documents.list`：分页、按类型/语言/关联实体筛选。
- `documents.get`
- `documents.upload`：流式上传，返回 `Document`。
- `documents.update`
- `documents.archive` / `documents.restore`
- `documents.link` / `documents.unlink`
- `documents.listFor`：`{ targetType, targetId }`，供物品/位置/任务详情页调用。
- `documents.search`：标题、类型、语言、关联实体；v1 不做正文搜索。
- 事件：`documents.changed { id, action }`。

下载走宿主 `download` 流；权限与文档元数据在同一次校验中完成。

## 7. UI（原生 / Web）

- 原生：文档列表、上传、详情、关联管理；物品详情内嵌说明书列表。
- Web：固定壳中的列表/详情/上传/下载；上传走 `/api/web/v1/upload`。
- 物品详情显示 `documents.listFor(targetType=inventory.item, targetId=...)`。
- 文档详情显示适用物品/位置；点击跳转到对应模块页面。
- 缩略图后续再加；v1 用类型图标。

## 8. 数据与迁移

```sql
CREATE TABLE m_documents_doc (
  id          TEXT PRIMARY KEY,
  title       TEXT NOT NULL,
  kind        TEXT NOT NULL,
  file_key    TEXT NOT NULL,
  mime        TEXT NOT NULL,
  size        INTEGER NOT NULL,
  sha256      TEXT NOT NULL,
  language    TEXT,
  revision    TEXT,
  notes       TEXT,
  created_by  TEXT NOT NULL,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_documents_kind ON m_documents_doc(kind, archived_at);

CREATE TABLE m_documents_link (
  document_id TEXT NOT NULL,
  target_type TEXT NOT NULL,
  target_id   TEXT NOT NULL,
  role        TEXT NOT NULL,
  created_at  INTEGER NOT NULL,
  PRIMARY KEY (document_id, target_type, target_id, role)
);
CREATE INDEX idx_documents_link_target ON m_documents_link(target_type, target_id);
```

- `file_key` 由 `FileStore` 生成；`sha256` 用于校验与去重。
- 关联表不建跨模块外键；目标删除后由调用方解除关联，后台清理孤儿。
- 迁移只向前；文件不进 DB。

## 9. 验收

- 上传 PDF 说明书并关联到物品；物品详情可见。
- 上传证书扫描件并关联到奖项/证书记录。
- 上传身份证/银行卡扫描件并关联到证件/卡记录。
- 同一文档关联多个物品；同一物品多份文档。
- 按标题/类型/语言/关联实体搜索。
- 删除关联不删文件；GC 只清理无引用且超过安全窗口的文件。
- 未授权用户不能下载；归档文档默认不返回。
- 原生与 Web 上传/下载都走流式传输，不阻塞 RPC 通道。

## 10. 开放问题

- 是否在 v1 做 OCR / 全文搜索；若做，索引放 `documents` 还是宿主搜索服务。
- 是否需要文档级 `visibility`（证件、保单只给管理员）。
- 说明书是否需要「适用型号」字段，还是完全依赖关联物品。
- 是否支持外部链接（厂家网页）作为文档。
- 是否需要缩略图与预览。
