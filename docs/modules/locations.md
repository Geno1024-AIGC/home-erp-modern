# 模块：位置（locations）

> 状态：draft ｜ 类型：core ｜ 依赖：— ｜ 拥有：`m_locations_node`

## 1. 目的与非目标

**目的**：维护家庭空间的位置树，例如「家 → 客厅 → 电视柜 → 第二层抽屉」。

**非目标**：

- 不做平面图、3D 户型或 GPS。
- 不存物品清单；「这个位置有什么」由 `inventory` 查询。
- 不做位置移动历史（v1 由 `inventory` 的流水记录）。

## 2. 用户场景

- 新建房间、柜子、抽屉、收纳盒。
- 重命名位置，调整同级顺序。
- 把一个柜子从客厅移到书房（整棵子树跟着移动）。
- 归档不再使用的位置。
- 在物品详情里选择位置；在位置页查看该位置的物品数量。

## 3. 边界与依赖

- 拥有位置树；不拥有物品、任务、文档。
- 其他模块只保存 `location_id`，不建外键。
- 删除/归档位置前，调用方需确认没有活跃引用；v1 由 `locations` 拒绝归档仍有活跃子节点的位置。
- 不做权限继承；权限由 `auth` 统一裁决。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `LocationNode` | `id`、`parent_id`、`kind`、`name`、`sort`、`archived_at` |

`kind`：`home`、`floor`、`room`、`furniture`、`container`。

规则：

- 单根：一个 `home` 根节点；其他节点必须有父节点。
- 不成环；移动时拒绝把节点移到自己的子树下。
- 同级按 `sort` 排序，重复时按名称。
- 归档节点要求其子树已归档；归档后该节点不可被选择。恢复时按原父节点恢复，父节点已归档则恢复到根下。
- 建议最大深度 8 层；超出由 UI 提示。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `locations.read` | 查看位置树 | `member` |
| `locations.write` | 新建、重命名、移动、归档 | `member` |
| `locations.admin` | 恢复归档、强制移动 | `admin` |

## 6. RPC 与事件

- `locations.tree.get`：返回整棵树或指定子树。
- `locations.node.get`
- `locations.node.create`
- `locations.node.update`：名称、类型、排序。
- `locations.node.move`：`{ id, newParentId, sort }`。
- `locations.node.archive` / `locations.node.restore`
- 事件：`locations.node.changed { id, action }`，供 `inventory` / `tasks` 刷新缓存。

所有修改类 RPC 需要 `locations.write` 或 `locations.admin`；写操作带 `idempotency-key`。

## 7. UI（原生 / Web）

- 原生：树形浏览、拖拽移动、右键菜单（重命名/归档/新建子位置）。
- Web：固定壳中的树形列表与表单；不做拖拽，v1 用「移动到」选择器。
- 位置详情显示物品数量、任务数量；数据通过 `inventory` / `tasks` RPC 聚合，不由 `locations` 直接查表。

## 8. 数据与迁移

```sql
CREATE TABLE m_locations_node (
  id          TEXT PRIMARY KEY,
  parent_id   TEXT,
  kind        TEXT NOT NULL,
  name        TEXT NOT NULL,
  sort        INTEGER NOT NULL DEFAULT 0,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_locations_parent ON m_locations_node(parent_id, sort);
```

- 迁移只向前；首次安装写入 `home` 根节点。
- 不存 `path`；树查询在内存组装。深度大时再评估缓存。
- 归档不物理删除；「彻底删除」只允许空子树，且需管理员确认。

## 9. 验收

- 创建、重命名、移动、归档、恢复位置。
- 移动到自身子树被拒绝；成环被拒绝。
- 归档含活跃子节点的位置被拒绝；先归档子节点。
- `member` 可维护，`guest` 只读；权限拒绝返回正确状态码。
- 原生与 Web 都能浏览同一棵树。
- 移动后 `locations.node.changed` 事件被 `inventory` 收到并刷新。

## 10. 开放问题

- 是否需要「位置二维码/标签」用于扫码定位。
- 是否需要位置照片；若需要，照片归 `documents`。
- 是否需要多根（例如「储藏室」「车库」独立于主宅）。
- 位置移动历史最终归 `locations` 还是 `inventory`。
