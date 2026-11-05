# 流程：记录一顿饭并附照片

> 涉及模块：`meals`、`documents` ｜ 触发：做完饭后记录 ｜ 权限：`meals.write`、`documents.upload`、`documents.link`

## 1. 目标

低摩擦记录一次实际做饭：时间、菜品、份数、评分、备注和成品照片；可从菜谱或当天菜单带入，但不要求先有菜谱或计划。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 做饭记录、菜品、评分、备注 | `meals` |
| 菜单计划、菜谱引用 | `meals` |
| 照片文件与元数据 | `documents` |
| 记录↔照片关联 | `documents` |

## 3. 步骤

1. 用户在「做饭记录」或当天菜单点「记录这顿」。
2. 可选：从 `meal_plan_id` 或 `recipe_id` 预填菜品、份数；用户可改。
3. 调 `meals.cookLogs.create` 创建 `CookLog` 与 `CookLogDish`，返回 `cookLogId`。
4. 拍照/选图后调 `documents.upload`，得到 `documentId`。
5. 调 `documents.link { documentId, targetType="meals.cook_log", targetId=cookLogId, role="cook_result" }`。
6. 记录详情调 `documents.listFor(targetType="meals.cook_log", targetId=cookLogId)` 显示照片。
7. 可选：用户在菜谱详情看到「做过 N 次」，点进历史记录。

## 4. 失败与补偿

- 记录已创建、拍照/上传失败：记录保留，提示「未附照片」，可重试；不阻塞记录。
- 上传成功、关联失败：文件成为孤儿；后台 GC 在安全窗口后清理，或用户在文档页手动关联。
- 菜谱/菜单已归档：记录仍保留菜品名称与 `recipe_id` 快照，不阻塞创建。
- 重复提交：`cookLogs.create` 带 `idempotencyKey`，避免连点生成两条记录。
- 网络中断：先保存草稿到本地，恢复后重试；v1 至少保留表单内容。

## 5. 事件

- `meals.cook_log.changed`
- `documents.changed`
- 不需要跨模块事务；记录和照片最终一致即可。

## 6. 验收

- 无菜谱也能快速记录一顿饭并上传照片。
- 从当天菜单/菜谱创建时能预填菜品，修改后不回写计划。
- 记录创建成功、照片上传失败时，记录仍可查看并重试。
- 多张照片按上传顺序展示；归档记录默认不返回。
- 菜谱归档后历史记录仍显示菜品名称。
- 做饭记录不自动扣库存、不自动完成 `tasks`。

## 7. 开放问题

- 先创建记录再上传，还是先上传再创建；当前选择先创建，失败可重试。
- 是否支持离线拍照后排队上传；v1 先在线。
- 是否设封面图；v1 取第一张 `cook_result`。
- 是否把做饭记录同步到日历；v1 不同步。
- 是否允许从照片识别菜品；v1 不做。
