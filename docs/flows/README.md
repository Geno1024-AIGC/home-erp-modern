# 跨模块流程

这里只放**跨 2 个以上模块**的流程。单模块内部流程写在该模块卡里。

## 模板

```markdown
# 流程：<名称>

> 涉及模块：... ｜ 触发：... ｜ 权限：...

## 1. 目标
## 2. 参与方与数据所有权
## 3. 步骤
## 4. 失败与补偿
## 5. 事件
## 6. 验收
## 7. 开放问题
```

## 索引

| 流程 | 涉及模块 | 状态 |
|---|---|---|
| [新增电器并归档说明书](appliance-manual.md) | `inventory`、`documents`、`locations` | draft |
| [扫码录入物品](barcode-intake.md) | `inventory`、`locations`、`documents` | draft |
| [记录一顿饭并附照片](cook-log.md) | `meals`、`documents` | draft |
| [洗一桶衣服](laundry-day.md) | `wardrobe`、`smart_home`、`tasks` | draft |
| [计划一次旅行](trip-planning.md) | `travel`、`tasks`、`documents`、`identity` | draft |
| [记录一次运动](workout-log.md) | `fitness`、`tasks`、`health` | draft |
| [出门前 30 秒](leave-home.md) | `calendar`、`tasks`、`inventory`、`wardrobe`、`smart_home` | draft |
| [开封与剩菜](leftovers.md) | `inventory`、`meals`、`tasks`、`documents` | draft |
| [今晚吃什么](cook-from-stock.md) | `meals`、`inventory`、`health`、`tasks` | draft |
| [消耗速度与补货](restock.md) | `inventory`、`meals`、`tasks`、`locations`、`documents` | draft |
| [出远门之前](trip-prep.md) | `travel`、`tasks`、`inventory`、`meals`、`pets`、`identity`、`documents`、`smart_home` | draft |
