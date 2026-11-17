# 流程：家庭手册与应急卡，人不在家也能照着做

> 涉及模块：`contacts`、`locations`、`documents`、`inventory`、`tasks`、`pets`（可选）、`health`（可选）、`bills`（可选） ｜ 触发：用户打开「家庭手册」/ 打印应急卡 ｜ 权限：`contacts.read`、`locations.read`、`documents.read`、`inventory.read`、`tasks.read`、`pets.read`、`health.read`、`bills.read`（按可见性）

## 1. 目标

把「这个家怎么运转」整理成一页可打印的手册：找谁、总阀在哪、垃圾哪天收、宠物怎么喂、账单何时到期、急救时看什么。手册是**只读投影**，不新建 `house_manual` 模块；每条事实仍由原模块拥有。v1 不存 Wi-Fi 密码、门禁码、银行卡号或完整证件号。

## 2. 参与方与数据所有权

| 数据 | 拥有者 | 手册中的用法 |
|---|---|---|
| 紧急联系人、物业、维修、医生、兽医 | `contacts` | 「找谁」列表；只显示当前用户可见记录 |
| 总水阀、电闸、信箱、垃圾点、停车位 | `locations` | 位置 `note` 与路径；不存密码/门禁码 |
| 灭火器、急救包、备用钥匙、滤芯 | `inventory` | 物品、位置、有效期与数量 |
| 租约、保单、说明书、委托书 | `documents` | 文档链接；不复制正文 |
| 垃圾日、浇花、宠物护理、设备检查 | `tasks` | 近期与周期任务 |
| 宠物习惯、喂食、兽医 | `pets` | 档案与护理摘要；不复制医疗正文 |
| 过敏、慢病、当前用药、急救联系人 | `health` | 只读 `emergency_visible=1` 的可见记录 |
| 房租、水电、保险、订阅 | `bills` | 近期到期；不显示余额/银行账号 |

本流程不拥有任何表；不生成第二份「家庭真相」。

## 3. 步骤

1. 用户打开家庭手册；客户端并行调用：
   - `contacts.list { kind: ["emergency", "medical", "vet", "repair", "property", "service"] }`
   - `locations.tree.get`（取关键位置与 `note`）
   - `inventory.items.list { category: ["safety", "cleaning", "medicine"], includeArchived: false }`
   - `documents.list { kind: ["manual", "contract", "insurance", "house_manual"] }`
   - `tasks.occurrences.list { from: 今天, to: 今天 + 30 天 }`
   - `pets.list { status: "active" }`
   - `health.records.list { emergencyVisible: true }`
   - `bills.upcoming.list { days: 30 }`
2. 渲染固定分区：
   - **紧急**：急救电话、紧急联系人、附近医院、过敏/慢病/当前用药；只显示 `emergency_visible=1` 的可见记录。
   - **房屋**：地址/入口、总水阀、电闸、燃气阀、信箱、垃圾点、停车位、灭火器/急救包位置。
   - **日常**：垃圾日、浇花、宠物喂食、设备检查、近期账单。
   - **宠物**：名字、照片、喂食/用药习惯、兽医联系人；医疗正文只链接，不展开。
   - **文档**：租约、保单、说明书、委托书；点击走 `documents` 权限。
3. 每条事实显示来源模块、最后更新时间与「去编辑」入口；编辑跳回拥有者，不在手册里直接改。
4. 应急卡：只渲染紧急分区，隐藏金额、证件号、详细地址门牌与私密备注；默认仅家庭可见。
5. 打印/导出：客户端渲染 PDF；可调用 `documents.upload { kind: "house_manual" }` 保存只读副本，或 `documents.watermark` 生成带接收方/用途的水印副本。
6. 访客/保姆：由家庭成员口头或打印提供，不开放 `guest` 账号；v1 不做临时访客权限。
7. 手册不缓存敏感正文；关闭即清除，打印前二次确认。

## 4. 失败与补偿

- **模块未安装**：该分区隐藏并列出缺席模块；不显示「没有」。
- **健康记录不可见**：应急卡标「未标记/部分可见」，不推断、不显示「无过敏」。
- **联系人私密**：不出现在家庭手册；只显示调用者可见记录。
- **位置备注为空**：显示「未填写」，不猜总阀位置。
- **账单/文档无权限**：分区隐藏；不复制金额或文件。
- **宠物医疗隐私**：只显示护理摘要与兽医联系人；病历链接受 `documents` 权限控制。
- **打印泄露**：打印/导出前提示包含家庭信息；水印副本带接收方与用途；日志不记录正文。
- **密码/门禁码**：页面明确不提供存储；用户只能写「在密码管理器/实体册」，不写具体值。
- **数据过期**：显示 `updated_at`；不把旧备注当成当前事实。

## 5. 事件

- 消费：`contacts.contact.changed`、`locations.node.changed`、`inventory.item.changed`、`documents.changed`、`tasks.occurrence.completed`、`pets.pet.changed`、`health.record.changed`、`bills.occurrence.changed`，用于重新聚合。
- 产生：`documents.changed`（保存/导出副本时）。
- 不新增事件；不跨模块事务；不自动生成任务或账单。

## 6. 验收

- 手册能显示紧急联系人、总阀/电闸备注、灭火器/急救包位置、近期任务与账单。
- 只显示调用者可见的联系人、健康记录与文档；私密记录不泄露。
- 应急卡只包含 `emergency_visible=1` 的健康记录；无记录时提示「未标记」，不显示「安全」。
- 编辑任何事实都跳回拥有者模块；手册本身不写业务表。
- 打印/导出有确认与水印选项；日志不含电话、地址、诊断与金额明细。
- 不存 Wi-Fi 密码、门禁码、银行卡号、完整证件号。
- `guest` 无权限；模块缺失时分区隐藏且明确提示。

## 7. 开放问题

- 是否需要独立的 `house_manual` 模块存自由文本；v1 用 `locations.note` + 各模块聚合。
- 是否需要访客/保姆临时只读权限；v1 只打印，不做临时账号。
- 是否需要离线手册包；v1 只导出 PDF/打印。
- 是否需要多语言与无障碍版本；v1 中文优先。
- 是否需要把「家庭规则」写成可版本化文档；v1 用 `documents` 的 `house_manual` 副本。
- 是否需要把 Wi-Fi 名称（不含密码）纳入手册；v1 只写「见密码管理器」。
