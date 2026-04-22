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
| `MAX_SCRUTINY`        | 0.4 |

單一函式處理所有推進來源：

```
advance_scrutiny(speed_factor: float = 1.0)
    delta = SCRUTINY_BASE_DELTA * speed_factor * skill_multiplier
    scrutiny = min(scrutiny + delta, MAX_SCRUTINY)
```

`skill_multiplier` 由 Appraisal 等級決定（mastery 不參與）。Skill 影響的是每次推進的效率，不影響上限。

```
skill_multiplier = 1.0 + appraisal_level * 0.35
```

| Appraisal | Multiplier | Delta（base 0.1） | 填滿 0.4 需要幾次 |
| --------- | ---------- | ----------------- | ----------------- |
| 0         | 1.0        | 0.10              | 4                 |
| 1         | 1.35       | 0.135             | 3                 |
| 2         | 1.70       | 0.17              | 3                 |
| 3         | 2.05       | 0.205             | 2                 |

- Inspection Scene 呼叫 `advance_scrutiny()`（默認 speed_factor 1.0）
- Storage Study 呼叫 `advance_scrutiny(study_speed_factor)`

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

| Rarity   | 無 scrutiny | 滿 scrutiny（0.4） | Condition 顯示  |
| -------- | ----------- | ------------------ | --------------- |
| Common   | 0.16        | 0.56               | ??? → Poor/Good |
| Uncommon | 0.08        | 0.48               | ??? → Poor/Good |
| Rare     | 0.05        | 0.45               | ??? → Poor/Good |
| Epic+    | 0.04        | 0.44               | ??? → Poor/Good |

新手即使花滿 scrutiny 也只能到二分法。

**中期**（cat_rank 3, appraisal 2, sc_average ~1.6）→ base ≈ 0.60

| Rarity   | 無 scrutiny | 滿 scrutiny（0.4） | Condition 顯示  |
| -------- | ----------- | ------------------ | --------------- |
| Common   | 0.60        | 1.0                | Poor/Good → 滿  |
| Uncommon | 0.30        | 0.70               | ??? → 四分法    |
| Rare     | 0.20        | 0.60               | ??? → Poor/Good |
| Epic     | 0.15        | 0.55               | ??? → Poor/Good |

中期玩家對 Common 已經接近滿，Uncommon 花 scrutiny 可以推進四分法。

**後期**（cat_rank 5, appraisal 3, sc_average ~3.0）→ base ≈ 0.99

| Rarity    | 無 scrutiny | 滿 scrutiny（0.4） | Condition 顯示          |
| --------- | ----------- | ------------------ | ----------------------- |
| Common    | 0.99        | 1.0+               | 接近滿 → 滿             |
| Uncommon  | 0.50        | 0.90               | Poor/Good → 四分法      |
| Rare      | 0.33        | 0.73               | 剛好 Poor/Good → 四分法 |
| Epic      | 0.25        | 0.65               | ??? → Poor/Good         |
| Legendary | 0.20        | 0.60               | ??? → Poor/Good         |

後期玩家 Legendary 仍然需要 perk 才能突破二分法。

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
| 0.33 – 0.66      | 二分法："Poor"（< 50%）/ "Good"（≥ 50%）                                            |
| 0.66 – 1.0       | 四分法："Poor"（< 25%）/ "Fair"（25–50%）/ "Good"（50–75%）/ "Excellent"（75–100%） |

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

## Repair System — Soft Cap + Rarity 係數

### Zone Speed Factor

每個 condition 區間有獨立速度係數，乘進 repair delta：

| Condition 區間 | Zone Factor | 大約天數（走完該段） |
| -------------- | ----------- | -------------------- |
| 0–25%          | 1.0         | 1–2 天               |
| 25–50%         | 0.35        | 4–5 天               |
| 50–75%         | 0.08        | 15–20 天             |
| 75–100%        | 0.01        | 非常久               |

設計意圖：

- 0→25%：快速脫離虧損區，Repair 的基本回報。
- 25→50%：有感投入，高技能可加速到 3 天。
- 50→75%：除非確定是高價值物品且需要滿足 Unlock 的 condition 門檻，否則不值得。
- 75→100%：幾乎靜止，留給未來 perk 加速。

### Rarity 影響 Repair 速度

溫和係數，額外乘進 delta：

| Rarity    | Factor |
| --------- | ------ |
| Common    | 1.0    |
| Uncommon  | 0.9    |
| Rare      | 0.8    |
| Epic      | 0.7    |
| Legendary | 0.6    |

Common 與 Legendary 之間差 40%。在低區間幾乎感覺不到差異（2 天 vs 3 天），到中區間才開始可被觀察到。提供微弱的 rarity 線索，但不構成明確洩漏。

### 完整 Repair Delta 公式

```
zone_factor = 依 condition 所在區間查表
rarity_factor = 依 item rarity 查表
delta = REPAIR_BASE * speed_factor * zone_factor * rarity_factor
condition = min(condition + delta, 1.0)
```

移除現有的 gap-based 遞減公式（`pow(gap, REPAIR_POWER)`），改用明確的區間係數。

---

## Storage Research Slots

| Action     | Effect                                      | Completion                                               |
| ---------- | ------------------------------------------- | -------------------------------------------------------- |
| **Study**  | 呼叫 `advance_scrutiny(study_speed_factor)` | 當 scrutiny 達到 MAX_SCRUTINY                            |
| **Repair** | 新公式（zone factor + rarity factor）       | 當 condition ≥ 1.0（實際受 zone factor 限制，75%+ 極慢） |
| **Unlock** | 不變，累積 unlock_progress                  | 當 unlock_progress ≥ difficulty                          |

Study 的定位改變：不再是「慢慢磨 inspection_level」，而是「補上場上沒來得及做的 scrutiny」。

---

## Unlock Action — 新增 required_inspection 欄位

```
required_inspection: float = 1.0（默認：需要完整 inspection 才能 unlock）
```

因為 inspection_level 由角色能力 + scrutiny 計算，這實際上是一個 expertise 軟門檻。Legendary 物品要達到 required_inspection: 1.0 需要遠高於 Common 的角色能力（因為 rarity_divisor = 5）。

---

## Perks — Inspection 相關

### keen_eye（敏銳之眼）— 增加 scrutiny 上限

```
MAX_SCRUTINY + 0.15（0.4 → 0.55）
```

效果：後期 + Legendary = 0.20 + 0.55 = 0.75 → 穩進四分法。每個 rarity 都有用，但對高 rarity 邊際效益最大。

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

效果：Uncommon 直接跟 Common 同等待遇。後期 + Legendary = 0.99 / 4 + 0.4 = 0.65 → 接近四分法門檻。

### quick_study（速學）— 加 scrutiny base delta

```
SCRUTINY_BASE_DELTA + 0.05（0.1 → 0.15）
```

| Appraisal | 原 Delta | Perk 後 | 填滿 0.4 次數         |
| --------- | -------- | ------- | --------------------- |
| 0         | 0.10     | 0.15    | 4 → 3                 |
| 3         | 0.205    | 0.31    | 2 → 2（溢出多，更快） |

效果：偏向效率而非能力上限。在 inspection scene 裡用更少動作做完同樣的事。跟 keen_eye 互補但不重疊。

### Perk 疊加極限測試

後期（base 0.99）+ 三個 perk 全開 + Legendary：

```
divisor = 4 (rarity_affinity), max_scrutiny = 0.55 (keen_eye)
inspection_level = 0.99 / 4 + 0.55 = 0.80
```

0.80 深入四分法但仍然不到精確值 1.0。Legendary 物品永遠保有一層神秘感，除非未來再加新的突破手段。

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

### 常數變更

| 常數                              | 舊值                 | 新值                                               |
| --------------------------------- | -------------------- | -------------------------------------------------- |
| `CONDITION_THRESHOLDS`            | [0.0, 1.0, 2.0]      | [0.0, 0.33, 0.66]（語意改為 inspection_level 0–1） |
| `RARITY_THRESHOLDS`               | 依 rarity 的多段門檻 | 移除（rarity 改由 layer_index 驅動）               |
| `REPAIR_BASE`                     | 0.15                 | 0.15（不變，但公式改為 zone factor 制）            |
| `REPAIR_POWER`                    | 0.5                  | 移除（改用 zone factor）                           |
| 新增 `SCRUTINY_BASE_DELTA`        | —                    | 0.1                                                |
| 新增 `MAX_SCRUTINY`               | —                    | 0.4                                                |
| 新增 `SCRUTINY_SKILL_COEFF`       | —                    | 0.35（skill_multiplier = 1.0 + appraisal \* 0.35） |
| 新增 `COMPUTED_BASE_CAT_WEIGHT`   | —                    | 0.15                                               |
| 新增 `COMPUTED_BASE_SC_WEIGHT`    | —                    | 0.03                                               |
| 新增 `COMPUTED_BASE_SKILL_WEIGHT` | —                    | 0.05                                               |
| 新增 `REPAIR_ZONE_FACTORS`        | —                    | {<0.25: 1.0, <0.50: 0.35, <0.75: 0.08, else: 0.01} |
| 新增 `REPAIR_RARITY_FACTOR`       | —                    | {C: 1.0, U: 0.9, R: 0.8, E: 0.7, L: 0.6}           |
| 新增 `RARITY_DIVISORS`            | —                    | {C: 1, U: 2, R: 3, E: 4, L: 5}                     |
| 新增 `INTUITION_BASE_CHANCE`      | —                    | 0.1                                                |
| 新增 `INTUITION_SKILL_COEFF`      | —                    | 0.15（chance = 0.1 + computed_base \* 0.15）       |
| 新增 `INTUITION_INSPECTION_BONUS` | —                    | 0.1（flag 觸發時加進 inspection_level）            |
| `INSPECT_COST`                    | 2 SP                 | 1 SP                                               |
| Perk `keen_eye`                   | —                    | MAX_SCRUTINY + 0.15                                |
| Perk `rarity_affinity`            | —                    | 所有 rarity_divisor - 1（最低 1）                  |
| Perk `quick_study`                | —                    | SCRUTINY_BASE_DELTA + 0.05                         |
