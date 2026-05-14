# 已完成 — Phase 0 ~ Phase 2

---

## Phase 0 — Runtime Veil Cleanup

**Commit:** `2d2a8b6`

- 新增 `ItemEntry.inspected` bool field
- `is_veiled()` → `return not inspected`（保留為相容 API）
- `unveil()` 只設定 `inspected = true`
- 新建物品固定 `layer_index = 1`、`inspected = false`
- `LotData.veiled_chance` 標記 deprecated，生成時不再使用
- Peek action 已移除（`lot_action_bar.gd` 已修改）
- Inspect 第一次觸發 reveal，後續才 advance scrutiny
- Reveal 階段改為標記 won items inspected

**當前遺留（計畫內）：**
- YAML / TRES item layer 0 資料尚未刪除
- Dev tools 的 layer 驗證規則尚未更新
- `is_veiled()` 仍有 38 處呼叫點，尚未簡化為直接使用 `inspected`

---

## Phase 1 — Commodity 系統

**Commit:** `3113983`

### 資料層
- `CommodityData` Resource class（`commodity_id`、`display_name`、`category_data`、`base_value`）
- `commodity_data.yaml` 定義檔
- `CommodityRegistry` autoload
- `LotData` 新增 `commodity_count_min/max`、`commodity_weights`

### Runtime
- `CommodityEntry` extends `LotObjectEntry`（`condition`、`inspected`、`sold`、`compute_sale_price()`）
- `LotEntry.lot_objects`：ItemEntry + CommodityEntry 混排，shuffled
- `RunRecord` 新增 `won_commodities`、`commodity_sales`、`last_lot_won_commodities`

### 顯示與結算
- `DaySummary` / `DaySummaryScene`：Commodity Sales 行
- `RunReviewScene`：Commodity Sales + Cash Flow
- `CargoScene`：右側 CommodityPanel（列表 + 總額）
- `LotObjectEntry` 統一前台 API（card、row、tooltip、action bar）

### 行為
- Reveal 後 Commodity 自動標記出售
- Commodity 不進 Cargo / Storage / Merchant / Research 管線
- 不做「帶回家」標記（留 Phase 1.5）

---

## Phase 2 — AP Grid Inspection

**Commit:** `683ab8d`（+ `2dfc3d1` 調整 speed / estimate）

### Grid 系統
- 8×8 grid，category shape-based 隨機 placement
- 每個 object 有獨立的 search duration（2–5 秒，依 size + random）
- 每個 lot 有 `action_quota`（預設 15 AP）
- SEARCH action：點擊未知 object 啟動 countdown，完成後 reveal
- ADVANCE action：已知但 non-final 的 Item 花 1 AP 推進 identity layer
- Right-click 取消進行中的 action（不耗 AP）

### 回顧流程
- Review 按鈕可中途查看 list review
- AP 耗盡自動跳 list review summary
- List Review 可選擇進入 Auction 或 Pass
- AP 有剩時可返回 Inspection（"Back to Inspection"）

### 後續調整（`2dfc3d1`）
- Search/advance countdown 從固定值改為 1:4 AP-to-seconds ratio（min 0.5s）
- Lot estimate 集中到 `LotEntry.get_player_estimate_label()`
- Cargo value estimate 改為 `(min+max)/2`
- Auction summary 使用 `lot_objects` + 統一的 estimate

### 實作 vs 原始 Pre-Plan 差異

| 項目 | 原始 Pre-Plan | 實際實作 |
|------|--------------|---------|
| 計時 | Fixed 20s real-time | AP quota（15 AP）+ per-action countdown |
| Search | 2–5s, 共 5–6 個 object | 依 shape size + random factor |
| Object 上限 | 無明確限制 | Lot 中所有 object 皆出現 |
| 已知後互動 | 不可進一步操作 | Item 可用 ADVANCE 推進 layer |
| 進入 Auction | 等 timer 結束 | Review Summary 畫面按鈕 |
