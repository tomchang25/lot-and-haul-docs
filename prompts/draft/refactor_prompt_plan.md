# Skill / Perk / Ref 重構 — Prompt 執行計畫

---

## Prompt 1：Perk YAML Pipeline

**目標**：讓 Perk 走跟 Skill 一樣的 YAML → .tres 流程，後續 Prompt 才能把 perk 做成 ExtResource ref。

### 新增檔案

| 檔案 | 說明 |
|---|---|
| `dev/tools/tres_lib/entities/perk.py` | PerkSpec，照抄 `skill.py` 結構但更簡單（沒有 sub-resource levels） |
| `data/yaml/perk_data.yaml` | 把現有手動 .tres 的 perk 轉成 YAML 定義 |

### 修改檔案

| 檔案 | 改動 |
|---|---|
| `dev/tools/tres_lib/registry.py` | 在 REGISTRY 裡加 `perk_spec`，排在 `skill_spec` 之後、`identity_layer_spec` 之前（identity_layer 需要 perk 的 uid） |

### perk.py 規格

- `yaml_key = "perks"`
- `tres_subdir = "perks"`
- `uid_prefix = "perk"`
- `script_paths = {"perk_data": "res://data/definitions/perk_data.gd"}`
- `build_tres`：寫 perk_id, display_name, description 三個欄位
- `parse_tres`：讀回三個欄位，把 uid → perk_id 寫進 `ctx.uid_to_id`
- `validate`：檢查 perk_id 非空、無重複

### perk_data.yaml 格式

```yaml
perks:
  - perk_id: dealer_network
    display_name: Dealer Network
    description: "Unlocks the Antique Dealer merchant."
```

（從現有 .tres 檔案反抄內容）

### 驗證

- `python yaml_to_tres.py --godot-root . --dry-run` 成功
- 產出的 .tres 跟原有手動 .tres 內容一致
- 刪掉舊手動 .tres，用 pipeline 產出的取代
- Godot 開啟後 KnowledgeManager perk registry 正常載入

### 不動的東西

GDScript 完全不動。PerkData.gd 不改。KnowledgeManager 不改。這個 prompt 純粹是 pipeline 工作。

---

## Prompt 2：Rename mechanical → maintenance

**目標**：把 mechanical skill 改名為 maintenance。

### 修改檔案

| 檔案 | 改動 |
|---|---|
| `data/yaml/skill_data.yaml` | `skill_id: mechanical` → `skill_id: maintenance`，`display_name: Mechanical` → `display_name: Maintenance` |
| `SaveManager` | `_repair_speed_factor()` 裡的 `"mechanical"` → `"maintenance"` |
| `SaveManager` | `_read_save_file()` 加 migration：如果 `skill_levels` dict 有 key `"mechanical"`，改名為 `"maintenance"` |

### 檢查是否有其他硬編碼引用

- `inspection_scene.gd` — 只用 `"appraisal"`，不影響
- YAML item data — `required_skill` 欄位目前沒有任何 layer 用 `mechanical`（只有 `appraisal` 和 `authentication`），不影響
- `yaml_generation_prompt.md` — 裡面的 Valid values 列表要更新：`"appraisal", "authentication", "maintenance"`

### 驗證

- Pipeline dry-run 成功
- 舊存檔載入後 `skill_levels` 裡的 key 正確遷移
- Storage repair speed 正常運作

---

## Prompt 3：Ref 重構 — Perk & Skill

**目標**：把 perk 和 skill 的 string ID 引用改成 Resource ref。

### 3A. Resource 定義改 ref

| 檔案 | 改動 |
|---|---|
| `layer_unlock_action.gd` | `required_perk_id: String = ""` → `required_perk: PerkData = null` |
| `merchant_data.gd` | `required_perk_id: String = ""` → `required_perk: PerkData = null` |

### 3B. Pipeline 改 ExtResource

| 檔案 | 改動 |
|---|---|
| `identity_layer.py` — `build_tres` | perk 改用 ExtResource（照抄 required_skill 的模式）：查 `ctx.uid_cache[perk_id]` → 加 ext_resource → sub_field 寫 `required_perk = ExtResource("X_perk")` |
| `identity_layer.py` — `parse_tres` | perk 改從 ExtResource 反解（照抄 skill 的模式） |
| `identity_layer.py` — `validate` | 加 `known_perk_ids` 檢查（照抄 skill） |
| `merchant.py` — `build_tres` | `required_perk_id` 字串欄位 → `required_perk` ExtResource ref（如果 perk_id 非空則查 uid_cache 加 ext_resource，空則寫 null） |
| YAML 欄位名 | `identity_layers` 和 `merchants` 的 YAML 裡 `required_perk_id` → `required_perk`（值仍然是 perk_id 字串，pipeline 內部解析） |

### 3C. KnowledgeManager API 改 ref

| 現狀 | 改成 |
|---|---|
| `get_level(skill_id: String) -> int` | `get_level(skill: SkillData) -> int`，內部 `SaveManager.skill_levels.get(skill.skill_id, 0)` |
| `has_perk(perk_id: String) -> bool` | `has_perk(perk: PerkData) -> bool`，內部 `perk.perk_id in SaveManager.unlocked_perks` |
| `unlock_perk(perk_id: String)` | `unlock_perk(perk: PerkData)`，內部 `SaveManager.unlocked_perks.append(perk.perk_id)` |
| `get_skill(skill_id: String)` | 改名 `get_skill_by_id(skill_id: String)`（保留，供反序列化用） |
| `get_perk(perk_id: String)` | 改名 `get_perk_by_id(perk_id: String)`（保留，供反序列化用） |

### 3D. 所有呼叫處跟著改

| 檔案 | 改動 |
|---|---|
| `KnowledgeManager.can_advance()` | `action.required_perk_id != ""` → `action.required_perk != null`；`has_perk(action.required_perk_id)` → `has_perk(action.required_perk)`；`get_level(action.required_skill.skill_id)` → `get_level(action.required_skill)` |
| `KnowledgeManager.validate()` | `get_perk()` → `get_perk_by_id()` |
| `KnowledgeManager.try_upgrade_skill()` | `get_level()` 呼叫改接 ref |
| `KnowledgeManager._check_upgrade()` | 同上 |
| `advance_check_label.gd` | `action.required_perk_id` → `action.required_perk.perk_id`（或直接讀 `.display_name`）；`KnowledgeManager.get_perk(...)` → 不需要了，直接 `action.required_perk.display_name` |
| `merchant_hub.gd` | `m.required_perk_id == ""` → `m.required_perk == null` |
| `MerchantRegistry.get_available_merchants()` | 同上 |
| `SaveManager._skill_speed_factor()` | 簽名改 `(skill: SkillData)`，內部 `KnowledgeManager.get_level(skill)` |
| `SaveManager._study_speed_factor()` | 持有 `_appraisal_skill: SkillData` ref，init 時從 `KnowledgeManager.get_skill_by_id("appraisal")` 取得 |
| `SaveManager._repair_speed_factor()` | 同上，用 `_maintenance_skill` ref |
| `SaveManager._unlock_speed_factor()` | 同上，用 `_appraisal_skill` ref |
| `inspection_scene.gd` — `_inspect_multiplier()` | 持有 `_appraisal_skill: SkillData` ref，`KnowledgeManager.get_level(_appraisal_skill)` |
| `skill_panel.gd` | `KnowledgeManager.get_level(skill.skill_id)` → `KnowledgeManager.get_level(skill)` |

### 驗證

- Pipeline dry-run 成功，產出的 .tres 裡 perk 是 ExtResource ref
- 所有 GDScript 編譯通過
- 場景運行：merchant 解鎖、layer unlock 門檻、skill upgrade 全正常
- 舊存檔載入正常（SaveManager 內部持久化仍然是 string，不受影響）

---

## Prompt 4：Ref 重構 — 其餘部分

**目標**：Registry 命名統一 + Category/SuperCategory API 改 ref。

### 4A. Registry lookup 統一命名 `_by_id`

所有 registry 的 string-based 單筆查詢改名：

| Registry | 現狀 | 改成 |
|---|---|---|
| `CategoryRegistry` | `get_category(id)` | `get_category_by_id(id)` |
| `SuperCategoryRegistry` | `get_super_category(id)` | `get_super_category_by_id(id)` |
| `MerchantRegistry` | `get_merchant(id)` | `get_merchant_by_id(id)` |
| `CarRegistry` | `get_car(id)` | `get_car_by_id(id)` |
| `ItemRegistry` | `get_item(id)` | `get_item_by_id(id)` |
| `LocationRegistry` | `get_location(id)` | `get_location_by_id(id)` |

所有呼叫處跟著改名。這些函式應該只剩反序列化（`from_dict`）、`migrate()`、`validate()` 在用。如果有非序列化的呼叫處還在用 string lookup，標記為待改（可能已在 Prompt 3 處理過）。

### 4B. KnowledgeManager Category API 改 ref

| 現狀 | 改成 |
|---|---|
| `add_category_points(category_id: String, ...)` | `add_category_points(category: CategoryData, ...)`，內部 `category.category_id` 當 dict key |
| `get_category_rank(category_id: String)` | `get_category_rank(category: CategoryData)`，內部 `category.category_id` |
| `get_super_category_rank(super_category_id: String)` | `get_super_category_rank(sc: SuperCategoryData)`，內部 `sc.super_category_id` |

呼叫處：
- `item_entry.gd` 的 `add_category_points(item_data.category_data.category_id, ...)` → `add_category_points(item_data.category_data, ...)`
- `item_entry.gd` 的 category_rank 查詢同理
- `mastery_panel.gd` 的 rank 查詢
- `storage_scene.gd` 的 rank 查詢

### 4C. 砍 CategoryRegistry.get_super_category_for()

這個函式改接 ref 後等於 `return category.super_category`，沒有存在意義。砍掉，呼叫處直接用 `.super_category`。

### 4D. SuperCategoryRegistry 跨 registry 查詢改接 ref

| 現狀 | 改成 |
|---|---|
| `get_categories_for_super(super_category_id: String)` | `get_categories_for_super(sc: SuperCategoryData)` 或保留 string 版本改名 `get_categories_for_super_by_id(id)` |

### 驗證

- 全部 GDScript 編譯通過
- 搜尋整個 codebase 確認：除了序列化邊界（from_dict、to_dict、migrate、validate）外，沒有任何地方把 Resource ref 拆成 string id 再傳入 API

---

## Prompt 5：剩餘基礎 Refactor

**目標**：處理 todo 清單中跟 redesign 相關的剩餘基礎工作。

### 5A. 更新 dev/tools/prompts/yaml_generation_prompt.md

- `required_skill` 的 Valid values 更新：`"appraisal", "authentication", "maintenance"`
- `required_perk_id` → `required_perk`，值仍然是 perk_id 字串

### 5B. 更新 dev/standards/registries.md

- Required API 裡的 `get_<singular>(id)` → `get_<singular>_by_id(id)`
- 加一條規則：非序列化邊界的呼叫端應接 Resource ref，不應從 ref 拆出 id 再傳入

### 5C. SaveManager speed_factor 公式（預留 redesign）

現有公式：`1.0 + pow(1.1, skill_level) * mastery_rank * 0.2`

先不改公式本身（redesign 會重新定義），但確認：
- `_study_speed_factor` 和 `_unlock_speed_factor` 都吃 `_appraisal_skill`
- `_repair_speed_factor` 吃 `_maintenance_skill`
- 簽名已在 Prompt 3 改好

### 5D. 確認無遺漏

搜尋 codebase 中所有 `"appraisal"`、`"authentication"`、`"maintenance"`、`"mechanical"` 字串字面量，確認只出現在：
- YAML 檔案（資料定義）
- SaveManager 的 skill ref 初始化處（`get_skill_by_id("appraisal")`）
- 序列化邊界

其他地方都不應該有硬編碼 skill/perk id 字串。

---

## 依賴順序圖

```
Prompt 1 (Perk Pipeline)
    ↓
Prompt 2 (mechanical → maintenance)     ← 可跟 Prompt 1 平行，無依賴
    ↓
Prompt 3 (Ref 重構 Perk & Skill)        ← 依賴 Prompt 1（perk uid 必須存在）
    ↓
Prompt 4 (Ref 重構 其餘)                ← 依賴 Prompt 3（避免重複改呼叫處）
    ↓
Prompt 5 (清理)                         ← 依賴全部完成
```

Prompt 1 和 2 可以平行，3 必須等 1 完成，4 等 3，5 收尾。
