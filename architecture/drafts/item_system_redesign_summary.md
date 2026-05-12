# Item Information System — Redesign Summary

Based on discussion, consolidating all agreed changes from the current system.

---

## Core Architecture Change

**inspection_level 從持久化欄位變成混合計算值：**

```
intuition_bonus = 0.1 if intuition_flag else 0.0
inspection_level = clamp(computed_base / rarity_divisor + scrutiny + intuition_bonus, 0.0, 1.0)
```

- `computed_base`：由角色能力動態算出（category_rank、sc_average、appraisal），不儲存。代表玩家的「底線鑑定能力」。
- `scrutiny`：逐件累積的持久化欄位，由 Inspect 動作和 Study 動作推進。代表「特別花時間看了這件」。
- `rarity_divisor`：稀有度越高，computed_base 的效果越被稀釋。

### Rarity Divisors

| Rarity    | Divisor |
| --------- | ------- |
| Common    | 1       |
| Uncommon  | 2       |
| Rare      | 3       |
| Epic      | 4       |
| Legendary | 5       |

### Scrutiny

| 參數                  | 值  |
| --------------------- | --- |
| `SCRUTINY_BASE_DELTA` | 0.1 |
| `MAX_SCRUTINY`        | 0.6 |

單一函式處理所有推進來源：

```
advance_scrutiny()
    delta = SCRUTINY_BASE_DELTA * skill_multiplier
    scrutiny = min(scrutiny + delta, MAX_SCRUTINY)
```

`skill_multiplier` 由 Appraisal 等級決定（mastery 不參與）。Skill 影響的是每次推進的效率，不影響上限。無 speed_factor 參數——Inspection Scene 和 Storage Study 使用相同的推進量，不再有速度調節。

```
skill_multiplier = 1.0 + appraisal_level * 0.35
```

| Appraisal | Multiplier | Delta（base 0.1） | 填滿 0.6 需要幾次 |
| --------- | ---------- | ----------------- | ----------------- |
| 0         | 1.0        | 0.10              | 6                 |
| 1         | 1.35       | 0.135             | 5                 |
| 2         | 1.70       | 0.17              | 4                 |
| 3         | 2.05       | 0.205             | 3                 |

- Inspection Scene 呼叫 `advance_scrutiny()`
- Storage Study 呼叫 `advance_scrutiny()`（與 Inspection Scene 相同推進量）

### computed_base 公式

```
sc_average = sc_rank / category_count_in_super_category
computed_base = category_rank * 0.15 + sc_average * 0.03 + appraisal_level * 0.05
```

- `category_rank`（0–5）：主軸。對特定類別的直接經驗。
- `sc_average`（0–5）：該物品所屬 super category 下所有 category rank 的平均值。代表對該領域的整體熟悉度。用平均值而非加總，避免 sub-category 數量不同導致的不公平。
- `appraisal_level`（0–3）：Appraisal 技能等級。

#### 參考數值表

**新手**（cat_rank 1, appraisal 0, sc_average ~0.4）→ base ≈ 0.16

| Rarity   | 無 scrutiny | 滿 scrutiny（0.6） | Condition 顯示      |
| -------- | ----------- | ------------------ | ------------------- |
| Common   | 0.16        | 0.76               | ??? → 四分法        |
| Uncommon | 0.08        | 0.68               | ??? → 四分法        |
| Rare     | 0.05        | 0.65               | ??? → Poor/Good     |
| Epic+    | 0.04        | 0.64               | ??? → Poor/Good     |

新手花滿 scrutiny 可以對 Common/Uncommon 進入四分法，Rare 以上仍停在二分法。

**中期**（cat_rank 3, appraisal 2, sc_average ~1.6）→ base ≈ 0.60

| Rarity   | 無 scrutiny | 滿 scrutiny（0.6） | Condition 顯示    |
| -------- | ----------- | ------------------ | ----------------- |
| Common   | 0.60        | 1.0+               | Poor/Good → 滿    |
| Uncommon | 0.30        | 0.90               | ??? → 四分法      |
| Rare     | 0.20        | 0.80               | ??? → 四分法      |
| Epic     | 0.15        | 0.75               | ??? → 四分法      |

中期玩家對 Common 已經溢出，Uncommon–Epic 花 scrutiny 全部可以推進四分法。

**後期**（cat_rank 5, appraisal 3, sc_average ~3.0）→ base ≈ 0.99

| Rarity    | 無 scrutiny | 滿 scrutiny（0.6） | Condition 顯示      |
| --------- | ----------- | ------------------ | ------------------- |
| Common    | 0.99        | 1.0+               | 接近滿 → 滿         |
| Uncommon  | 0.50        | 1.0+               | Poor/Good → 滿      |
| Rare      | 0.33        | 0.93               | 剛好 Poor/Good → 四分法 |
| Epic      | 0.25        | 0.85               | ??? → 四分法        |
| Legendary | 0.20        | 0.80               | ??? → 四分法        |

後期玩家花滿 scrutiny 對所有 rarity 都能進入四分法。Legendary 到 0.80 深入四分法但離精確值 1.0 仍有距離——perk 提供最後一段突破手段。

---

## Rarity Display — 改由 layer_index 驅動

不再由 inspection_level 決定。玩家推進身份鏈（layer）多深，就能看到多精確的稀有度。

| Layer Index | 顯示規則                                        |
| ----------- | ----------------------------------------------- |
| 0（veiled） | 無稀有度顯示                                    |
| 1           | Common 顯示 "Common"，其餘顯示 "Uncommon+"      |
| 2           | Common/Uncommon 顯示真名，Rare 以上顯示 "Rare+" |
| 3           | 至 Rare 顯示真名，Epic 以上顯示 "Epic+"         |
| 4+          | 所有稀有度顯示真名                              |

稀有度認知是身份研究的永久投資，不是可磨的檢查值。

---

## Condition Display — 改由 inspection_level 驅動（0.0–1.0）

三層漸進揭露：

| inspection_level | 顯示                                                                                |
| ---------------- | ----------------------------------------------------------------------------------- |
| < 0.33           | "???"（完全未知）                                                                   |
| ≥ 0.33, < 0.66   | 二分法："Poor"（< 50%）/ "Good"（≥ 50%）                                            |
| ≥ 0.66           | 四分法："Poor"（< 25%）/ "Fair"（25–50%）/ "Good"（50–75%）/ "Excellent"（75–100%） |

---

## Condition Multiplier — 新分界點（非線性）

分界對齊顯示的四分法，但倍率保持頂部加重：

| Condition 區間 | 倍率範圍     | 玩家感知         |
| -------------- | ------------ | ---------------- |
| 0–25%          | 0.25x → 0.5x | 破爛，賣了虧錢   |
| 25–50%         | 0.5x → 1.0x  | 堪用，剛好保本   |
| 50–75%         | 1.0x → 2.0x  | 不錯，有賺頭     |
| 75–100%        | 2.0x → 4.0x  | 極品，翻倍再翻倍 |

50% 是直覺上的及格線（倍率 1.0x）。各區間內仍為線性內插。

---

## Price Range — 簡化

inspection_level 已經是 0.0–1.0，公式直接用：

```
convergence_ratio = inspection_level
spread = MAX_SPREAD * (1.0 - inspection_level)
offset = center_offset * (1.0 - inspection_level)
```

`center_offset` 仍在物品建立時骰一次（[-0.5, 0.5]），持久化儲存。確保同 category、同 rarity 的物品即使 inspection_level 相同也顯示不同的價格範圍。

---

## Inspection Scene — 兩個動作 + 一個被動

### 被動：Intuition（直覺）

進入 Inspection Scene 時自動觸發。每件非 veiled 物品各獨立骰一次，不消耗 SP、不佔行動次數。

```
intuition_chance = 0.1 + computed_base * 0.15
```

| 玩家階段 | computed_base | 觸發機率 |
| -------- | ------------- | -------- |
| 新手     | 0.16          | ~12%     |
| 中期     | 0.60          | ~19%     |
| 後期     | 0.99          | ~25%     |

- **成功**：設 `intuition_flag = true`，兩個效果：
  1. **稀有度顯示**改用 `layer_index + 1` 查表。
     - layer 1 物品：本來顯示 Common vs Uncommon+ → 觸發後顯示 Common/Uncommon vs Rare+
     - layer 2 物品：本來顯示至 Uncommon vs Rare+ → 觸發後顯示至 Rare vs Epic+
     - 已顯示真實稀有度 → 無效果
  2. **inspection_level + 0.1**（加進 computed 計算，不影響 scrutiny 上限）。
     - Common 補償：rarity 顯示已經看到 "Common" 不需要 +1 層，但 +0.1 讓 condition/price range 更精確，觸發不白費。
     - 高 rarity 雙重收益：rarity 多看一層 + inspection 推進一小步，幫助跨越 condition 顯示門檻。
- **未觸發**：無任何視覺提示，玩家不知道哪些沒骰中。

`intuition_flag` 持久化在物品上，一旦觸發永久生效。Common 物品觸發也設 flag，rarity 顯示不變但 inspection_level +0.1 仍然有效——玩家無法單從 rarity 顯示判斷是「沒觸發」還是「觸發了但是 Common」。

#### Inspection Scene VFX

- **觸發時**：成功的物品卡片播一個短暫的 shimmer 動畫（~0.5 秒），場景進入時同時觸發。
- **持久標記**：觸發成功的卡片保留一個微弱的邊框光暈或角落小圖示，方便玩家在檢查其他物品後回頭辨認哪些觸發過。
- 不加文字標籤——Intuition 是微妙的被動觸發，不應該看起來像一個重要狀態。

#### Storage 行為

Storage 不顯示任何 Intuition 相關的 VFX 或標記。玩家只看到下游效果：inspection_level 略高（price range 窄一點、condition 可能多一個層級）、rarity 可能多顯示一層。玩家不會知道原因是 Intuition——符合「直覺」的主題感：你不知道為什麼對某些東西就是有感覺。

未來擴充點：perk 加觸發機率、觸發時額外給 scrutiny、觸發時多解析兩層等。現階段只做 flag。

### 動作 1：Inspect（指定目標）

- 玩家先點選一張物品卡片，再按 Inspect 按鈕。
- 對指定物品呼叫 `advance_scrutiny()`。
- 消耗 1 SP + 1 行動次數。
- 目標必須為非 veiled、非已滿 scrutiny。

與舊系統差異：從隨機打改為指定目標，SP 從 2 降為 1。

### 動作 2：Peek（揭開遮蓋物）

- 不變。對 veiled 物品嘗試揭開，成功則推進到 layer 1。
- 消耗體力和行動次數。

### 三者的分歧

|               | 觸發方式               | 控制權            | 給的資訊                           |
| ------------- | ---------------------- | ----------------- | ---------------------------------- |
| **Intuition** | 被動，場景進入時自動骰 | 無（運氣 + 能力） | 稀有度多看一層                     |
| **Inspect**   | 主動，指定目標         | 完全控制          | scrutiny → condition / price range |
| **Peek**      | 主動，指定目標         | 完全控制          | 揭開 veil → 進入 layer 1           |

玩家看到 Intuition 的結果後，再決定把 SP 花在哪些物品的 Inspect 或 Peek 上。Intuition 是決策的輸入，不是決策本身。

---

## Repair & Restore System

REPAIR（0→50%）和 RESTORE（50→100%）是兩個獨立系統，寫同一個 `condition` 欄位，各自有獨立的公式、速度常數、skill 要求。

50% 分界對齊 condition multiplier 的 1.0x 線——REPAIR 把物品從虧損區拉到保本，RESTORE 把物品從保本推到溢價。

### REPAIR（0–50%）

Maintenance skill 驅動。目標是讓所有玩家都能在合理時間內完成基本修復。

#### Zone Speed Factor

| Condition 區間 | Zone Factor | 大約天數（Common） |
| -------------- | ----------- | ------------------- |
| 0–25%          | 1.0         | 1–2 天              |
| 25–50%         | 0.35        | 4–5 天              |

#### Rarity Factor

| Rarity    | Factor |
| --------- | ------ |
| Common    | 1.0    |
| Uncommon  | 0.9    |
| Rare      | 0.8    |
| Epic      | 0.7    |
| Legendary | 0.6    |

溫和係數。Common 與 Legendary 之間差 40%，提供微弱的 rarity 線索但不構成明確洩漏。

#### Delta 公式

```
zone_factor = 依 condition 所在區間查表（REPAIR_ZONE_FACTORS）
rarity_factor = 依 item rarity 查表（REPAIR_RARITY_FACTOR）
delta = REPAIR_BASE * zone_factor * rarity_factor
condition = min(condition + delta, 0.5)
```

移除現有的 gap-based 遞減公式（`pow(gap, REPAIR_POWER)`），改用明確的區間係數。

### RESTORE（50–100%）

Restoration skill（通用）+ super category 專屬 skill 雙重驅動。投入門檻高、耗時長，是後期專業化的投資。

#### Zone Speed Factor

| Condition 區間 | Zone Factor | 大約天數（Common，無 skill） |
| -------------- | ----------- | ----------------------------- |
| 50–75%         | 0.12        | ~21 天                        |
| 75–100%        | 0.02        | ~125 天                       |

#### Rarity Factor

| Rarity    | Factor |
| --------- | ------ |
| Common    | 1.0    |
| Uncommon  | 0.8    |
| Rare      | 0.6    |
| Epic      | 0.4    |
| Legendary | 0.2    |

比 REPAIR 陡峭得多。Common 與 Legendary 之間差 80%——Legendary 精修需要大量 skill 投資才可行。

#### Skill 影響

兩層 skill 相乘：

```
restore_skill_mult = 1.0 + restoration_level * 0.4
cat_skill_mult = 1.0 + cat_restore_skill_level * 0.5
```

- `restoration`：通用精修技能，所有 RESTORE 都吃。
- super category 專屬 skill：從物品的 `item_data.category_data.super_category.restore_skill` 取得。

| 投資程度 | restore_mult | cat_mult | 總倍率 |
| -------- | ------------ | -------- | ------ |
| 無 skill | 1.0          | 1.0      | 1.0x   |
| 各 1 級  | 1.4          | 1.5      | 2.1x   |
| 各 2 級  | 1.8          | 2.0      | 3.6x   |

#### Delta 公式

```
zone_factor = 依 condition 所在區間查表（RESTORE_ZONE_FACTORS）
rarity_factor = 依 item rarity 查表（RESTORE_RARITY_FACTOR）
restore_skill_mult = 1.0 + restoration_level * RESTORE_SKILL_COEFF
cat_skill_mult = 1.0 + cat_restore_skill_level * RESTORE_CAT_SKILL_COEFF
delta = RESTORE_BASE * zone_factor * rarity_factor * restore_skill_mult * cat_skill_mult
condition = min(condition + delta, 1.0)
```

#### 參考天數表（50→75% zone）

| Rarity    | 無 skill | 各 1 級（2.1x） | 各 2 級（3.6x） |
| --------- | -------- | ---------------- | ---------------- |
| Common    | ~21 天   | ~10 天           | ~6 天            |
| Uncommon  | ~26 天   | ~12 天           | ~7 天            |
| Rare      | ~35 天   | ~17 天           | ~10 天           |
| Epic      | ~52 天   | ~25 天           | ~14 天           |
| Legendary | ~104 天  | ~50 天           | ~29 天           |

75→100% zone 天數約為上表的 6 倍。無 skill 的 Legendary 精修在遊戲時間尺度上不可行——這是設計意圖。

### REPAIR / RESTORE 拆分

兩個獨立的 SlotAction：

| Action      | 可指派條件          | 完成條件           | 效果描述     |
| ----------- | ------------------- | ------------------ | ------------ |
| **REPAIR**  | condition < 0.50    | condition ≥ 0.50   | 修復         |
| **RESTORE** | condition ≥ 0.50    | condition ≥ 1.0    | 精修         |

不需要在 slot 上存任何額外欄位。

設計意圖：REPAIR 完成後 slot 釋放，玩家在 UI 上看到新的 RESTORE 選項。RESTORE 是一個完全不同的投資決策——需要不同的 skill、面對更陡的 rarity 懲罰、花費數十倍的時間。兩個動作名稱天然在溝通這件事。

### Restore Skills

#### 通用 skill — restoration

所有 RESTORE 的基礎 skill。

| skill_id    | display_name | 等級 | cash_cost | required_mastery_rank |
| ----------- | ------------ | ---- | --------- | --------------------- |
| restoration | Restoration  | 1    | 500       | 2                     |
| restoration | Restoration  | 2    | 2000      | 5                     |

#### Super category 專屬 skill

每個 super category 指定一個專屬 restore skill，記在 `SuperCategoryData.restore_skill` 上。

| Super Category | skill_id        | display_name    |
| -------------- | --------------- | --------------- |
| fashion        | textile_care    | Textile Care    |
| decorative     | conservation    | Conservation    |
| fine_art       | art_restoration | Art Restoration |
| weapon         | gunsmithing     | Gunsmithing     |

各 2 級，升級需求統一：

| 等級 | cash_cost | required_mastery_rank |
| ---- | --------- | --------------------- |
| 1    | 800       | 3                     |
| 2    | 3000      | 6                     |

專屬 skill 比 restoration 貴且門檻高——它們是後期專業化路線的投資。

#### Resource 定義變更

`SuperCategoryData` 新增欄位：

```
@export var restore_skill: SkillData = null
```

Pipeline `super_category.py` 的 `build_tres` / `parse_tres` 加 ExtResource 處理（照抄 `category.py` 處理 `super_category` ref 的模式）。

`super_category_data.yaml` 每個 entry 加 `restore_skill: textile_care` 等。`restore_skill` 為空或省略時，該 super category 的 RESTORE 不吃專屬 skill 加成（`cat_skill_mult = 1.0`）。

---

## Storage Research Slots

| Action      | Effect                                                          | Completion                      |
| ----------- | --------------------------------------------------------------- | ------------------------------- |
| **Study**   | 呼叫 `advance_scrutiny()`                                      | 當 scrutiny 達到 MAX_SCRUTINY   |
| **Repair**  | REPAIR 公式（zone factor + rarity factor）                      | 當 condition ≥ 0.50             |
| **Restore** | RESTORE 公式（zone factor + rarity factor + skill multipliers） | 當 condition ≥ 1.0              |
| **Unlock**  | 不變，累積 unlock_progress                                      | 當 unlock_progress ≥ difficulty |

Study 的定位改變：不再是「慢慢磨 inspection_level」，而是「補上場上沒來得及做的 scrutiny」。

---

## Unlock Action — 新增 required_inspection_level 欄位

```
required_inspection_level: float = 1.0（默認：需要完整 inspection 才能 unlock）
```

因為 inspection_level 由角色能力 + scrutiny 計算，這實際上是一個 expertise 軟門檻。Legendary 物品要達到 required_inspection_level: 1.0 需要遠高於 Common 的角色能力（因為 rarity_divisor = 5）。

---

## Perks — Inspection 相關

### keen_eye（敏銳之眼）— 增加 scrutiny 上限

```
MAX_SCRUTINY + 0.15（0.6 → 0.75）
```

效果：後期 + Legendary = 0.20 + 0.75 = 0.95 → 逼近精確值。每個 rarity 都有用，但對高 rarity 邊際效益最大。

### rarity_affinity（稀有親和）— 拉平 rarity 影響

```
所有 rarity_divisor 減 1（最低 1）
```

| Rarity    | 原 Divisor | Perk 後 |
| --------- | ---------- | ------- |
| Common    | 1          | 1       |
| Uncommon  | 2          | 1       |
| Rare      | 3          | 2       |
| Epic      | 4          | 3       |
| Legendary | 5          | 4       |

效果：Uncommon 直接跟 Common 同等待遇。後期 + Legendary = 0.99 / 4 + 0.6 = 0.85 → 深入四分法。

### quick_study（速學）— 加 scrutiny base delta

```
SCRUTINY_BASE_DELTA + 0.05（0.1 → 0.15）
```

| Appraisal | 原 Delta | Perk 後 | 填滿 0.6 次數         |
| --------- | -------- | ------- | --------------------- |
| 0         | 0.10     | 0.15    | 6 → 4                 |
| 3         | 0.205    | 0.31    | 3 → 2（溢出多，更快） |

效果：偏向效率而非能力上限。在 inspection scene 裡用更少動作做完同樣的事。跟 keen_eye 互補但不重疊。

### Perk 疊加極限測試

後期（base 0.99）+ 三個 perk 全開 + Legendary：

```
divisor = 4 (rarity_affinity), max_scrutiny = 0.75 (keen_eye)
inspection_level = 0.99 / 4 + 0.75 = 1.00（無 intuition）
inspection_level = 0.99 / 4 + 0.75 + 0.1 = 1.10 → clamp 1.0（有 intuition）
```

三個 perk 全開時 Legendary 物品剛好可以觸及精確值 1.0。這是完整 endgame 投資的獎勵——需要後期角色能力 + 兩個 perk 欄位（keen_eye + rarity_affinity）才能做到，quick_study 只加速填充不影響上限。單開 keen_eye 時 Legendary = 0.95，單開 rarity_affinity 時 = 0.85，仍然有感但不完整。

---

## 欄位變更總結

### ItemEntry — 新增

| 欄位             | 類型  | 持久化 | 說明                                                 |
| ---------------- | ----- | ------ | ---------------------------------------------------- |
| `scrutiny`       | float | 是     | 逐件累積，由 Inspect / Study 推進，上限 MAX_SCRUTINY |
| `intuition_flag` | bool  | 是     | Intuition 動作成功後設為 true，稀有度多顯示一層      |

### ItemEntry — 語意改變

| 欄位               | 變化                                                                                              |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| `inspection_level` | 從持久化欄位 → computed property（`computed_base / rarity_divisor + scrutiny + intuition_bonus`） |

### ItemEntry — 移除

| 欄位                                              | 原因 |
| ------------------------------------------------- | ---- |
| （無移除，inspection_level 保留名稱但改為計算值） |      |

### ItemEntry — API 變更

| 方法 / 屬性                            | 變化                                                                                                    |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `apply_inspect(delta: float)`           | 移除。改為 `advance_scrutiny()`，無參數                                                                  |
| `apply_study(speed_factor: float)`      | 移除。Storage Study 改為呼叫 `advance_scrutiny()`                                                        |
| `apply_repair(speed_factor: float)`     | 移除 speed_factor 參數 → `apply_repair()`。內部改用 REPAIR 公式，cap 在 0.5                               |
| `is_fully_inspected() -> bool`          | 改為 `inspection_level >= 1.0`（不再檢查 rarity bucket）                                                 |
| `get_rarity_bucket() -> int`            | 移除。rarity 顯示改由 `layer_index`（+ `intuition_flag`）驅動                                           |
| `is_rarity_resolved() -> bool`          | 移除。同上                                                                                               |
| `_rarity_thresholds() -> Array[float]`  | 移除。不再需要                                                                                           |
| `price_convergence_ratio -> float`      | 簡化為 `inspection_level`（不再需要 `max_threshold` 計算）                                               |
| `get_potential_rating() -> String`      | 改為查 `layer_index`（若 `intuition_flag` 則用 `layer_index + 1`）對照 rarity display 表                 |
| `is_condition_inspectable() -> bool`    | 改為檢查 `scrutiny < MAX_SCRUTINY`                                                                       |
| `is_repair_complete() -> bool`          | 改為 `condition >= 0.50`（REPAIR completion）                                                             |
| 新增 `apply_restore()`                  | RESTORE 公式（zone factor + rarity factor + skill multipliers），cap 在 1.0。需讀取 super category 的 restore_skill |
| 新增 `is_restore_complete() -> bool`    | `condition >= 1.0`（RESTORE completion）                                                                  |
| `add_unlock_effort(speed_factor: float)` | 移除 speed_factor 參數 → `add_unlock_effort()`                                                           |

### SaveManager — speed_factor 移除

所有 speed_factor 相關方法移除：

| 方法                      | 變化                                                  |
| ------------------------- | ----------------------------------------------------- |
| `_skill_speed_factor()`   | 移除                                                  |
| `_study_speed_factor()`   | 移除                                                  |
| `_repair_speed_factor()`  | 移除                                                  |
| `_unlock_speed_factor()`  | 移除                                                  |
| `_appraisal_skill` 快取   | 移至 ItemEntry 或 inspection_scene（按需保留）         |
| `_maintenance_skill` 快取 | 移除（repair 不再需要 skill ref）                     |

`_tick_research_slots` 內部：Study 呼叫 `entry.advance_scrutiny()`，Repair 呼叫 `entry.apply_repair()`，Restore 呼叫 `entry.apply_restore()`，皆無參數。Completion 判定各自獨立：Repair 看 `entry.is_repair_complete()`，Restore 看 `entry.is_restore_complete()`。

### ResearchSlot — 變更

| 變更                   | 說明                                                                          |
| ---------------------- | ----------------------------------------------------------------------------- |
| `SlotAction` 新增 RESTORE | `STUDY = 0, REPAIR = 1, UNLOCK = 2, RESTORE = 3`                             |
| `action_to_string`     | 新增 `RESTORE → "restore"`                                                   |
| `_slot_effect_label`   | 新增 `RESTORE → "Fully restored"`                                            |

不需要新增任何欄位。`to_dict()` / `from_dict()` 透過 action enum 值自然處理 RESTORE。

### 常數變更

| 常數                              | 舊值                 | 新值                                               |
| --------------------------------- | -------------------- | -------------------------------------------------- |
| `CONDITION_THRESHOLDS`            | [0.0, 1.0, 2.0]      | [0.0, 0.33, 0.66]（語意改為 inspection_level 0–1） |
| `RARITY_THRESHOLDS`               | 依 rarity 的多段門檻 | 移除（rarity 改由 layer_index 驅動）               |
| `REPAIR_BASE`                     | 0.15                 | 0.15（不變，但公式改為 zone factor 制，cap 0.5）   |
| `REPAIR_POWER`                    | 0.5                  | 移除（改用 zone factor）                           |
| `REPAIR_MIN_STEP`                 | 0.005                | 移除（zone factor 制不需要最低步進）               |
| `STUDY_BASE_DELTA`                | 0.08                 | 移除（被 `SCRUTINY_BASE_DELTA` 取代）              |
| 新增 `SCRUTINY_BASE_DELTA`        | —                    | 0.1                                                |
| 新增 `MAX_SCRUTINY`               | —                    | 0.6                                                |
| 新增 `SCRUTINY_SKILL_COEFF`       | —                    | 0.35（skill_multiplier = 1.0 + appraisal \* 0.35） |
| 新增 `COMPUTED_BASE_CAT_WEIGHT`   | —                    | 0.15                                               |
| 新增 `COMPUTED_BASE_SC_WEIGHT`    | —                    | 0.03                                               |
| 新增 `COMPUTED_BASE_SKILL_WEIGHT` | —                    | 0.05                                               |
| 新增 `REPAIR_ZONE_FACTORS`        | —                    | {<0.25: 1.0, <0.50: 0.35}                           |
| 新增 `REPAIR_RARITY_FACTOR`       | —                    | {C: 1.0, U: 0.9, R: 0.8, E: 0.7, L: 0.6}           |
| 新增 `REPAIR_COMPLETE_THRESHOLD`  | —                    | 0.50（REPAIR 完成 / RESTORE 可指派門檻）            |
| 新增 `RESTORE_BASE`               | —                    | 0.10                                                |
| 新增 `RESTORE_ZONE_FACTORS`       | —                    | {<0.75: 0.12, else: 0.02}                           |
| 新增 `RESTORE_RARITY_FACTOR`      | —                    | {C: 1.0, U: 0.8, R: 0.6, E: 0.4, L: 0.2}           |
| 新增 `RESTORE_SKILL_COEFF`        | —                    | 0.4（restore_mult = 1.0 + restoration \* 0.4）      |
| 新增 `RESTORE_CAT_SKILL_COEFF`    | —                    | 0.5（cat_mult = 1.0 + cat_skill \* 0.5）            |
| 新增 `RARITY_DIVISORS`            | —                    | {C: 1, U: 2, R: 3, E: 4, L: 5}                     |
| 新增 `INTUITION_BASE_CHANCE`      | —                    | 0.1                                                |
| 新增 `INTUITION_SKILL_COEFF`      | —                    | 0.15（chance = 0.1 + computed_base \* 0.15）       |
| 新增 `INTUITION_INSPECTION_BONUS` | —                    | 0.1（flag 觸發時加進 inspection_level）            |
| `INSPECT_COST`                    | 2 SP                 | 1 SP                                               |
| Perk `keen_eye`                   | —                    | MAX_SCRUTINY + 0.15                                |
| Perk `rarity_affinity`            | —                    | 所有 rarity_divisor - 1（最低 1）                  |
| Perk `quick_study`                | —                    | SCRUTINY_BASE_DELTA + 0.05                         |
