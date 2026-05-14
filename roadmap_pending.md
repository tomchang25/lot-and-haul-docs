# 核心循環重新設計 — 待辦項目

## 起因

兩個待辦項同時指向同一個根本問題：Day Summary 的 Net 數字讓玩家覺得自己永遠在虧錢，而 Weekly Report 難以實作。根源在於買入和賣出之間的時間差——拍賣場大量買入、回家慢慢賣給商人，以「天」為單位的結算永遠呈現扭曲的畫面。拍賣日必然是大赤字，賣貨日小賺，但心理傷害已經造成。不像 Salvage Hunter 那種買一件賣一件即時閉合的模式，現有結構下任何每日結算都在說謊。

---

## 已完成的基礎（Phase 0–2）

| Phase | 內容 | Commit |
|-------|------|--------|
| 0 | Runtime Veil Cleanup — `inspected` bool、`is_veiled()` 相容層、Peek 移除 | `2d2a8b6` |
| 1 | Commodity 系統 — CommodityEntry、混排 lot_objects、自動變現與顯示 | `3113983` |
| 2 | AP Grid Inspection — 8×8 grid、search countdown、ADVANCE、review summary | `683ab8d` |

詳細內容請參閱 `docs/completed_phases_0-2.md`。

---

## 尚未進行

### Phase 3 — Clues 系統

原始 `clue_evaluator.gd` 已於 `ae684af` 刪除，當前 codebase 無任何 clue 相關程式碼。

**待辦：**
- [ ] Clue 資料定義（clue_id、display_text、prerequisite、trigger enum）
- [ ] ItemEntry 新增 `revealed_clue_ids: Array[String]`
- [ ] Prerequisite 檢查邏輯（複用 LayerUnlockAction 條件結構）
- [ ] Passive / on_action 觸發邏輯
- [ ] 先掛到 Research / Study 流程驗證

---

### Phase 4 — 整合（依賴 Phase 2–3）

- [ ] Inspection grid 接上 Clues passive 揭露
- [ ] Tooltip card 顯示已揭露 clues
- [ ] Commodity 在 grid 中 inspect 後揭露身分與 sale value

> **註：** `LotObjectEntry` 統一 API 已就緒（ItemEntry / CommodityEntry 共用 row、tooltip、action bar）。

---

### Phase 5 — Day Summary 簡化

- Day Summary 仍顯示 `Net` 數字（`day_summary.gd` line 22–24）
- 尚未決定：拿掉 Net vs 改成輕量確認 + Balance vs 非拍賣日跳過

---

## 暫不排入開發

- **週系統** — 無相關程式碼（僅 `SuperCategoryData.market_drift_per_week`）
- **固定扣額 Game Over** — 無相關程式碼

---

## 不動的東西

| 系統 | 說明 |
|------|------|
| Location / Lot 資料結構與選擇邏輯 | 未變動 |
| Merchant 談判系統 | 未變動 |
| Research tick 邏輯（scrutiny 每天 tick） | 未變動 |
| Car / Vehicle 系統 | 未變動 |
| RunRecord 核心資料物件 | 未變動 |
| ItemEntry Layer 推進與 Research 管線 | 未變動（Commodity 已分流） |
| Cargo 裝車操作（維持 auto-reveal） | 未變動 |
