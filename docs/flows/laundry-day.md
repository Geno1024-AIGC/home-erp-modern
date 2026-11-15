# 流程：洗一桶衣服，并让第二天的抽卡自动避开

> 涉及模块：`wardrobe`、`smart_home`、`tasks` ｜ 触发：攒够一桶脏衣服 / 洗衣机洗完了 ｜ 权限：`wardrobe.laundry`、`wardrobe.write`、`smart_home.read`、`tasks.write`

## 1. 目标

把「洗一桶衣服」拆成可检查的步骤：先确认这桶洗护不冲突，再让机器事件只负责提醒，最后由人确认衣物回到可穿状态。抽卡永远只从可穿的搭配里选，不会推荐一件还在洗衣机里的衣服。

## 2. 参与方与数据所有权

| 数据 | 拥有者 |
|---|---|
| 衣物、洗护画像、洗涤批次、穿着记录 | `wardrobe` |
| 洗衣机设备、`cycle_state`、门磁、漏水 | `smart_home` |
| 「洗完了去晾」提醒 | `tasks` |
| 衣物照片、吊牌 | `documents` |

## 3. 步骤

1. 用户把脏衣物状态改为 `dirty`，或在建批次时直接扫入。
2. 调 `wardrobe.laundry.loads.create`，填写计划程序：水温、是否烘干、是否漂白、是否干洗；可选关联 `smart_home` 的洗衣机 `deviceId`。
3. 调 `wardrobe.laundry.loads.add_items` 加入衣物。
4. 调 `wardrobe.laundry.check`。返回 `ok` / `warn` / `conflict` 与逐条原因，例如「3 件上限 40 °C，当前 60 °C」「2 件不可烘干」。
5. `conflict` 时用户改计划（拆桶或降温度）后重新检查；用户坚持不改也允许继续，但结果与已读状态写入批次。
6. 用户到洗衣机上设置真实程序；`wardrobe.laundry.loads.start` 只把批次置 `running` 并把衣物置 `washing`，**不发送任何洗衣机命令**。
7. 洗衣机完成时，`smart_home` 产生 `cycle_state=done`；`wardrobe` 订阅后发出 `wardrobe.laundry.changed`，`tasks` 据此生成「去晾衣服」提醒。
8. 用户晾好后确认，衣物转 `drying`；晾干确认后转 `clean`，可同时更新 `location_id` 回衣柜。
9. 批次置 `done`；衣物状态回到可参与抽卡的集合。
10. 次日抽卡：候选集自动排除 `dirty` / `washing` / `drying` / `repair` / `lent`，被洗过的搭配权重按「最近 7 天抽中过」下调。

## 4. 失败与补偿

- **重复完成事件**：`smart_home` 的 `cycle_state=done` 只按设备与批次去重一次，提醒不重复弹。
- **洗衣机离线或没接**：批次仍可手动 `start` 与 `finish`，只是没有自动完成提醒；UI 明确标注「无设备联动」。
- **用户忘记确认**：衣物长时间停在 `washing` / `drying`，抽卡会把相关搭配排除；超过 24 小时给出「这桶还没确认」的提示，但不自动改状态。
- **冲突被忽略**：不阻断，只记录「存在未处理的 conflict」；不修改机器程序。
- **批次含其他成员衣物**：加入时只读取名称与洗护画像，不读价格与穿着记录；批次仅创建者与衣物所有者可见。
- **衣物在批次中被归档**：批次保留引用，冲突检查仍按归档前的洗护画像计算。
- **`smart_home` 未安装**：整个流程退化为手动批次，不依赖设备事件。
- **`tasks` 未安装**：不生成提醒，批次状态照常推进。

## 5. 事件

- `wardrobe.laundry.changed`
- `wardrobe.garment.changed`
- `smart_home.device.state.changed`（`cycle_state`）
- `tasks.template.changed` / `tasks.occurrence.completed`（如生成了提醒）
- 不需要跨模块事务；批次、提醒、设备状态最终一致即可。

## 6. 验收

- 一桶里混入 40 °C 上限衣物，选 60 °C 时返回 `conflict` 并列出件数。
- 只能干洗件混入水洗桶返回 `conflict`；深浅色混洗只 `warn`。
- `start` 不产生任何 `smart_home` 命令，洗衣机不会被 ERP 改动。
- 洗衣机 `done` 只生成一次提醒；重复事件被去重。
- 衣物停在 `washing` 时，相关搭配不会出现在抽卡结果里，并给出排除原因。
- 确认晾干后衣物回到 `clean`，次日可正常参与抽卡。
- 未安装 `smart_home` 或 `tasks` 时流程仍能手动完成。
- 日志不含衣物价格、穿着记录与批次内衣物名称。
