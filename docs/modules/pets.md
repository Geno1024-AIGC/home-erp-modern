# 模块：宠物与花鸟鱼虫（pets）

> 状态：draft ｜ 类型：business ｜ 依赖：`members`、`locations`（可选）、`documents`（可选）、`tasks`（可选）、`calendar`（可选） ｜ 拥有：`m_pets_pet`、`m_pets_member`、`m_pets_care_log`

## 1. 目的与非目标

**目的**：管理家里的宠物、花鸟鱼虫与植物：档案、归属成员、位置、照片、护理记录；把喂养/换水/修剪等提醒交给 `tasks`。

**非目标**：

- 不做宠物医疗诊断、处方、保险理赔。
- 不拥有护理计划/提醒；周期计划由 `tasks` 拥有。
- 不拥有食物、药品、耗材库存；归 `inventory`。
- 不做繁育/交易/血统证书业务。
- 不把宠物健康混入人类 `health`。

## 2. 用户场景

- 登记猫、狗、鸟、鱼、昆虫、植物，记录品种/花色/生日/来源。
- 指定主人和共同照看人。
- 记录喂食、换水、清洗、遛狗、驱虫、疫苗、看病、修剪、施肥、换盆。
- 查看某只宠物/某盆植物的护理时间线。
- 给宠物/植物拍照，给疫苗本/病历附扫描件。
- 用 `tasks` 建「每两天换水」「每月驱虫」等周期提醒。
- 宠物去世或送养后保留历史，不删除记录。

## 3. 边界与依赖

- 拥有宠物/植物档案、成员关联、护理记录。
- 只引用 `member_id`、`location_id`；不建跨模块外键。
- 护理计划/提醒归 `tasks`；可用 `subject_type="pets.pet"`、`subject_id` 指向宠物。
- 预约/事件可放 `calendar`；`pets` 不复制日历事件。
- 食物、药品、耗材库存在 `inventory`；`pets` 只可选引用 `inventory_item_id`。
- 照片、疫苗本、病历、化验单归 `documents`。
- 植物用同一套档案与护理记录；不需要单独的「植物」模块。

## 4. 领域模型

| 实体 | 关键字段 |
|---|---|
| `Pet` | `id`、`name`、`kind`、`species`、`breed`、`sex`、`birth_date`、`adopted_at`、`color`、`microchip`、`location_id`、`status`、`notes`、`archived_at` |
| `PetMember` | `pet_id`、`member_id`、`role` |
| `CareLog` | `id`、`pet_id`、`kind`、`at`、`performed_by`、`summary`、`metadata_json` |

`kind`：`cat`、`dog`、`bird`、`fish`、`insect`、`plant`、`other`。

`status`：`active`、`deceased`、`rehomed`。

`role`：`owner`、`caretaker`。

`CareLog.kind`：`feed`、`water`、`clean`、`water_change`、`vet`、`vaccine`、`medication`、`grooming`、`molt`、`prune`、`fertilize`、`repot`、`other`。

规则：

- `kind` 决定 UI 的默认护理类型与图标，但不限制可记录的护理项。
- 猫狗鸟鱼虫植物共用档案结构；品种/花色/缸体/光照等差异放 `notes` 或 `metadata_json`。
- 一个人可以照看多只宠物；一只宠物可以有多个 owner/caretaker。
- 护理记录只追加，不修改历史；录错了追加更正记录。
- `status=deceased/rehomed` 保留档案与历史，默认列表可筛选隐藏。
- 护理计划不重复存；需要周期提醒时由 `tasks` 建模板并指向宠物。
- 食物/药品库存变化走 `inventory.stock.adjust`，`pets` 不自动扣减。
- 照片和医疗附件通过 `documents.link` 关联到 `targetType="pets.pet"` 或 `"pets.care_log"`。

## 5. 权限

| 权限 | 说明 | 默认 |
|---|---|---|
| `pets.read` | 查看宠物/植物档案与护理记录 | `member` |
| `pets.write` | 新建/编辑档案、追加护理记录 | `member` |
| `pets.admin` | 恢复归档、处理去世/送养记录 | `admin` |

## 6. RPC 与事件

- `pets.list`：按类型、状态、成员、位置筛选。
- `pets.get`
- `pets.create` / `pets.update` / `pets.archive`
- `pets.members.add` / `pets.members.remove`
- `pets.care.list`：按宠物、类型、时间范围筛选。
- `pets.care.add`：追加护理记录，不接受修改历史。
- 事件：
  - `pets.pet.changed { id, action }`
  - `pets.care.added { petId, kind }`

护理提醒不新增宠物 RPC；UI 调 `tasks` 建模板，`subject_type="pets.pet"`。

## 7. UI（原生 / Web）

- 原生：宠物/植物列表（按猫狗鸟鱼虫植物分组）、档案、护理时间线、照片、成员与位置。
- Web：固定壳中的列表/详情/护理记录；不做批量导入。
- 档案页显示「即将到期的护理任务」：只读调用 `tasks.occurrences.list`。
- 植物视图突出浇水/施肥/修剪/换盆；鱼缸突出换水/水温；鸟/虫突出清洁/温湿度。
- 照片和疫苗本/病历通过 `documents` 上传与展示。
- 去世/送养记录显示纪念标记，不删除历史。

## 8. 数据与迁移

```sql
CREATE TABLE m_pets_pet (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  kind        TEXT NOT NULL,
  species     TEXT,
  breed       TEXT,
  sex         TEXT,
  birth_date  TEXT,
  adopted_at  TEXT,
  color       TEXT,
  microchip   TEXT,
  location_id TEXT,
  status      TEXT NOT NULL DEFAULT 'active',
  notes       TEXT,
  archived_at INTEGER,
  created_at  INTEGER NOT NULL,
  updated_at  INTEGER NOT NULL
);
CREATE INDEX idx_pets_kind ON m_pets_pet(kind, status);
CREATE INDEX idx_pets_location ON m_pets_pet(location_id);

CREATE TABLE m_pets_member (
  pet_id    TEXT NOT NULL,
  member_id TEXT NOT NULL,
  role      TEXT NOT NULL,
  PRIMARY KEY (pet_id, member_id, role)
);
CREATE INDEX idx_pets_member ON m_pets_member(member_id);

CREATE TABLE m_pets_care_log (
  id            TEXT PRIMARY KEY,
  pet_id        TEXT NOT NULL,
  kind          TEXT NOT NULL,
  at            INTEGER NOT NULL,
  performed_by  TEXT NOT NULL,
  summary       TEXT,
  metadata_json TEXT,
  created_at    INTEGER NOT NULL
);
CREATE INDEX idx_pets_care_pet ON m_pets_care_log(pet_id, at DESC);
CREATE INDEX idx_pets_care_kind ON m_pets_care_log(kind, at DESC);
```

- 日期用 `YYYY-MM-DD`；护理时间用 UTC epoch milliseconds。
- 迁移只向前；档案默认归档，护理记录不物理删除。
- 照片/附件在 `documents` / `file_refs`，宠物模块不存文件。

## 9. 验收

- 新增猫、狗、鸟、鱼、昆虫、植物；按类型筛选。
- 指定 owner/caretaker；按成员查询。
- 追加喂食、换水、清洗、疫苗、看病、修剪、施肥等记录；历史不可改。
- 关联照片、疫苗本、病历；删除关联不删文件。
- 用 `tasks` 建「每两天换水」并指向宠物；宠物页显示到期任务。
- 食物/药品库存调整走 `inventory`，宠物模块不自动扣减。
- 去世/送养后默认列表隐藏，历史仍可查。
- `guest` 无权限；`member` 可维护；`admin` 可恢复归档。

## 10. 开放问题

- 是否需要按物种扩展字段（体重、缸体尺寸、水温/pH、光照、笼舍）。
- 是否需要体重/生长曲线、换水/施肥统计。
- 是否需要繁育、后代、领养/送养流程。
- 植物是否需要单独的「季节护理」模板。
- 宠物医疗是否要独立于护理记录，还是保持 `CareLog.kind=vet/vaccine`。
- 是否需要照片时间线与纪念页。
