# 业务模块文档

本目录是**业务模块卡**，不是模块机制文档。模块机制见 [03-modules.md](../03-modules.md)，领域边界见 [08-domain.md](../08-domain.md)。

## 规则

- 一个模块一个文件：`docs/modules/<module-id>.md`。
- 控制在 1–2 页；超过 2 页再拆成同名的 `docs/modules/<module-id>/` 文件夹。
- 只写**当前契约**：目的、边界、实体、权限、RPC、UI、数据、验收。
- 历史取舍写 [07-decisions.md](../07-decisions.md)，不堆在模块卡里。
- 跨 2 个以上模块的流程写 [flows/](../flows/)，不塞进模块卡。
- 修改 RPC、表结构或权限时，必须同步更新对应模块卡。
- 模块之间只传逻辑 ID，不共享业务表、不建跨模块外键。

## 状态

`planned` → `draft` → `stable` → `deprecated`；暂不实现的标 `deferred`。

## 模板

```markdown
# 模块：<产品名>（<module-id>）

> 状态：draft ｜ 类型：core/business ｜ 依赖：... ｜ 拥有：...

## 1. 目的与非目标
## 2. 用户场景
## 3. 边界与依赖
## 4. 领域模型
## 5. 权限
## 6. RPC 与事件
## 7. UI（原生 / Web）
## 8. 数据与迁移
## 9. 验收
## 10. 开放问题
```

## 注册表

| 模块 | 产品名 | 类型 | 状态 | 依赖 | 文档 |
|---|---|---|---|---|---|
| `members` | 家庭成员 | core | planned | — | 待写 |
| `auth` | 登录与角色 | core | planned | `members` | 待写 |
| `locations` | 位置 | core | draft | — | [locations.md](locations.md) |
| `documents` | 文档与说明书 | business | draft | — | [documents.md](documents.md) |
| `inventory` | 物品 | business | draft | `locations`、`documents`（可选） | [inventory.md](inventory.md) |
| `tasks` | 家务与任务 | business | draft | `locations`、`inventory`、`members`、`documents`（可选） | [tasks.md](tasks.md) |
| `calendar` | 日历 | business | draft | `members`、`locations`（可选）、`documents`（可选）、`tasks`（可选） | [calendar.md](calendar.md) |
| `meals` | 饮食与采购 | business | draft | `inventory`（可选）、`tasks`（可选）、`calendar`（可选）、`documents`（可选） | [meals.md](meals.md) |
| `health` | 健康与医疗 | business | draft | `members`、`documents`（可选）、`calendar`（可选）、`tasks`（可选） | [health.md](health.md) |
| `achievements` | 奖项与证书 | business | draft | `members`、`documents`（可选）、`tasks`（可选） | [achievements.md](achievements.md) |
| `identity` | 证件与卡片 | business | draft | `members`、`documents`（可选）、`tasks`（可选） | [identity.md](identity.md) |
| `assets` | 资产 | deferred | — | — | 先并入 `inventory` |

当前已写：`locations`、`inventory`、`documents`、`tasks`、`calendar`、`meals`、`health`、`achievements`、`identity`。其余模块等进入路线图时再写。
