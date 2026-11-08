# 流程：计划一次旅行并在回来后记录

> 涉及模块：`travel`、`documents`、`tasks`、`calendar` ｜ 触发：计划家庭旅行 / 补记已完成的旅行 ｜ 权限：`travel.write`、`documents.upload`、`documents.link`、`tasks.write`

## 1. 目标

从计划到记录只维护一条旅行：先排出日期、同行人和逐日行程，绑定预订与票据；出发前用任务和清单准备；旅行中/结束后补游记与照片。日历只读显示旅行占用，不复制行程。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 旅行、目的地、同行人、逐日行程、清单、游记 | `travel` |
| 机票、酒店确认单、保险单、门票、旅行照片 | `documents` |
| 旅行↔文档关联 | `documents` |
| 出发前任务与提醒 | `tasks` |
| 日历事件与日历视图 | `calendar` |

## 3. 步骤

1. 用户新建旅行：`travel.trips.create`，填写标题、日期、目的地、预算与同行人；同行人可以是成员或仅记录姓名。
2. 按天添加行程：`travel.items.create`，选择日期与类型（交通/住宿/活动/餐饮/休息）；跨天住宿用 `endDate`。
3. 上传机票、酒店确认单、保险单：`documents.upload`，再调 `documents.link { targetType="travel.trip", targetId=tripId, role="ticket|booking|insurance" }`。
4. 建出发前准备任务：`tasks.templates.create` 或一次性任务，`subjectType="travel.trip"`、`subjectId=tripId`；提醒仍由通知能力调度。
5. 打包清单在 `travel.checklist.*` 中维护，可指派成员；清单不生成任务历史。
6. 日历调 `travel.trips.listInRange` 只读叠加旅行日期；用户点条目跳转旅行详情，不在日历里改行程。
7. 旅行中或结束后：`travel.items.mark` 标记完成/跳过/改期，填实际时间；`travel.journal.create` 写游记。
8. 游记照片：`documents.upload` 后 `documents.link { targetType="travel.journal_entry", targetId=entryId, role="journal_photo" }`；封面用 `role="cover"`。
9. 旅行结束把 `status` 改为 `completed`；需要归档时 `travel.trips.archive`，历史与文档关联保留。

## 4. 失败与补偿

- 旅行已创建、上传失败：旅行与行程保留，详情提示「未附票据/照片」，可重试。
- 上传成功、关联失败：文件成为孤儿；后台 GC 在安全窗口后清理，或用户在文档页手动关联。
- 任务创建失败：不影响旅行与行程；用户可在 `tasks` 中手动补建。
- 日历读取旅行失败：日历仍显示事件与任务，旅行条目降级隐藏，不阻塞日历。
- 日期缩短失败：提示范围外仍有行程/游记，用户先移动或删除。
- 重复提交：`travel.trips.create` 与 `travel.journal.create` 支持 `clientId`/幂等键，避免连点生成重复记录。
- 网络中断：v1 在线优先；表单保留草稿，恢复后重试，不做离线同步。

## 5. 事件

- `travel.trip.changed`
- `travel.item.changed`
- `travel.checklist.changed`
- `travel.journal.changed`
- `documents.changed`
- `tasks.template.changed`
- 不需要跨模块事务；旅行、文档、任务最终一致即可。

## 6. 验收

- 只建旅行与日期，也能在日历看到只读占用。
- 日历不生成重复旅行事件；点击旅行条目能回到 `travel`。
- 机票/酒店确认单/保险单关联后，旅行详情可见；删除关联不删文件。
- 出发前任务完成不回写旅行；旅行归档后任务历史仍保留。
- 补记已完成旅行不需要先建行程，游记与照片可独立存在。
- 游记照片与封面按角色展示；归档游记默认不返回。
- 旅行日期缩短、跨天行程、同行人移除等边界校验正确。
- 日志、审计与通知不含目的地、住宿地址与游记正文。
- 导出/打印必须显式触发，不自动生成公开分享。

## 7. 开放问题

- 是否允许从旅行一键生成日历事件，还是坚持只读叠加；当前只读。
- 是否允许从打包清单一键生成任务；当前分开，避免任务膨胀。
- 是否需要旅行花费与票据金额回填；v1 不做。
- 是否需要离线行程包与地图；v1 只提供打印/导出。
- 是否允许非成员同行人登录查看行程；v1 只记录姓名，不授予访问。
