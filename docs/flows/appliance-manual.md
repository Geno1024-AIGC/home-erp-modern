# 流程：新增电器并归档说明书

> 涉及模块：`inventory`、`documents`、`locations` ｜ 触发：用户新增电器 ｜ 权限：`inventory.write`、`documents.upload`、`documents.link`

## 1. 目标

新增一台电器，选择位置，上传说明书并关联；之后在物品详情能直接打开说明书。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 电器主数据 | `inventory` |
| 位置 | `locations` |
| 说明书文件与元数据 | `documents` |
| 物品↔说明书关联 | `documents` |

## 3. 步骤

1. 用户在 `inventory` 创建物品：`kind=durable`，填写品牌/型号/序列号/购买日期/保修期。
2. 用户选择 `location_id`；`inventory` 只校验该 ID 存在（调 `locations.node.get`），不建外键。
3. 用户上传说明书：`documents.upload` 流式写入 `files/`，返回 `documentId`。
4. 用户确认关联：`documents.link { documentId, targetType="inventory.item", targetId=itemId, role="manual" }`。
5. 物品详情调用 `documents.listFor(targetType="inventory.item", targetId=itemId)` 展示说明书。
6. 可选：同一说明书再关联到同型号的另一台电器。

## 4. 失败与补偿

- 物品已创建、上传失败：物品保留，详情页提示「未关联说明书」，可重试。
- 上传成功、关联失败：文件保留但成为孤儿；后台 GC 在安全窗口后清理，或用户在文档页手动关联。
- 位置在流程中被归档：创建时校验失败，提示重新选择。
- 重复上传同一文件：`FileStore` 按 SHA-256 去重，生成新的文档记录或复用由用户选择。

## 5. 事件

- `inventory.item.changed`
- `documents.changed`
- 物品详情缓存失效；不需要跨模块事务。

## 6. 验收

- 新增电器并上传 PDF 说明书，物品详情可见并打开。
- 不上传说明书也能完成物品创建。
- 同一说明书关联两台同型号电器，两边都能看到。
- 上传失败不阻塞物品创建；重试可完成关联。
- 权限不足时上传/关联被拒绝，物品创建不受影响。

## 7. 开放问题

- 是否在新增电器表单里直接上传说明书，还是先创建再上传。
- 是否自动从说明书文件名/DOI 提取型号；v1 不提取。
- 保修卡/发票是否作为同一流程的可选步骤。
- 说明书是否要 OCR，以便按正文搜索。
