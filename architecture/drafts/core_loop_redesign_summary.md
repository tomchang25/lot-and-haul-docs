# 核心循環重新設計 — 討論總結

## 起因

兩個待辦項同時指向同一個根本問題：Day Summary 的 Net 數字讓玩家覺得自己永遠在虧錢，而 Weekly Report 難以實作。根源在於買入和賣出之間的時間差——拍賣場大量買入、回家慢慢賣給商人，以「天」為單位的結算永遠呈現扭曲的畫面。拍賣日必然是大赤字，賣貨日小賺，但心理傷害已經造成。不像 Salvage Hunter 那種買一件賣一件即時閉合的模式，現有結構下任何每日結算都在說謊。

---

## 討論過程

### 週系統的探索與收斂

最初考慮將整個流程改為週曆結構：每週固定一到兩天買貨，其餘天數做 Research、賣貨、未來開店。進一步討論過「工作日分配」的批次規劃模式——把一週的剩餘天數當資源，玩家在一個週計畫畫面上分配給 Research、Merchant、其他活動。

這個方案在概念上優雅，但實際上把十幾個決策壓縮到同一個畫面，資訊密度和操作量都太高。最終收斂到一個輕量版本：「週」是結算單位，不是規劃單位。玩家的日常操作不變，遊戲只是用週來框住節奏和歸納回饋。

後續因為 Commodity 自動變現的決定，週系統的急迫性大幅降低——玩家跑一趟回來就有正的現金回饋，利潤斷裂的痛點基本消失。週系統降為低優先。

### Inspection 的根本問題

現有 Inspection 是清單操作：看到所有物品、選一個、按 Inspect、重複。物品數量多少都不改變它是個 checklist 的本質。物品太少（4 個）則多元性不足；物品太多（10 個）則操作機械化，互動性沒有改善。

決定改為類似塔科夫搜刮的 grid 模式。物品散布在格子裡，初始全黑，玩家點擊揭露。發現本身就是 gameplay，而不是走清單。

### 俯視 vs 正視圖

兩個視覺方案：俯視圖 2D grid（簡單好寫但不直觀）和正視圖散佈物品輪廓（主流做法但美工要求高、無法做 placeholder）。決定用俯視圖，但透過視覺框架區分——Cargo 的 grid 乾淨空白等你填，Inspection 的 grid 初始全滿、黑色、被蓋住。玩家心理模型是「站在倉庫門口往裡看」，俯視只是實作方便。

### 遮擋機制

因為玩家的心理模型是正面看進倉庫，前排擋住後排完全合理。Grid 下排是靠近門口的前排，上排是倉庫深處。不做失敗機率，改為後排 stamina 成本更高：前兩排花 1 點，中間排數根據同 column 前方遮擋物數量加到 2 點，後排以此類推到 3-4 點。大型物品天然比較難被完全擋住（多 column 取最佳視線），小型物品容易被藏住。確定性高，玩家能規劃。

### Veiled 系統的處置

討論了三個方案：保留 layer 0 只砍 chance、砍 layer 0 改 bool 保留接口、連接口一起改名。決定採用方案二——砍掉 layer 0 的 runtime 語意，加一個 bool inspected flag，`is_veiled()` 改為 `return not inspected`。下游呼叫點暫時保留 `is_veiled()` 這個相容接口。

已完成的 Phase 0 採用 runtime-first 範圍：現有 YAML / TRES 的 layer 0 資料先保留，runtime 新建物品固定從 `layer_index = 1` 開始，並以 `inspected = false` 控制玩家是否看得到資訊。Peek action 已移除；現有清單版 Inspection 裡，第一次 Inspect 會標記 inspected，後續 Inspect 才推進 scrutiny。真正把 YAML layer 0 從資料管線中移除，留到後續資料清理階段。

### Commodity 與 ItemEntry 的分離

最初計劃是讓 Common 物品自動變現。進一步討論後發現，PS4、工業碗、舊衣服這類東西跟 Uncommon+ 的古董在遊戲裡扮演完全不同的角色。前者有明確市場價格、不需要鑑定、不需要研究；後者才是 Layer 混淆、Clue 揭露、Research 投資的對象。硬塞進同一個 ItemEntry 管線，等於每個 PS4 都得寫一棵 identity layer 樹和一組 clue 定義，全部是無意義的樣板。

決定引入 **Commodity** 作為獨立的輕量資料型別。資料欄位維持靜態：commodity_id、display_name、category_data、base_value。condition 是 runtime roll 出來的 `CommodityEntry.condition`，不寫在 Data 裡。grid_size 從 CategoryData 上讀取。不走 Layer、Clue、Research 管線。Lot 生成時一個 lot 就是 N 個 Commodity + M 個 ItemEntry，填充物從通用 commodity pool 隨機抽，設計師只需要為有研究價值的物品寫完整 YAML。

Commodity 預設行為是 Reveal 後自動標記出售，實際現金仍由 run resolution 統一入帳。後續可允許玩家在 Run Review 時手動標記「帶回家」，佔 Storage 空間，走 market price 路線等高價賣。這產生一個小決策：顯卡現在自動賣 $150，帶回家等兩週可能賣 $200，值不值得佔一格 Storage？早期通常不值得，後期空間充裕了才有餘裕操作。Phase 1 的第一版先以「自動變現 + 正回饋顯示」為核心，帶回家可拆成 Phase 1.5。

Commodity 的名稱選擇：不叫 Junk，因為有些 commodity 挺值錢（顯卡），只是沒研究價值。Commodity 準確表達了「有固定市場價、不需鑑定」的性質。

同時解決利潤回饋（跑一趟馬上有現金進帳）、Storage 膨脹（不塞低價垃圾）、Research 決策品質（留下的都值得研究）三個問題。Run Review 顯示一行即時正回饋。

### Cargo 維持自動 Reveal

Cargo 階段不做 fog 化。俄羅斯方塊式的裝車操作已經夠忙，額外加 reveal 操作只是增加摩擦。

### Clues 系統回歸

現有系統中老手只根據 rarity 決定取捨，太直接且存在唯一最佳解。引入 clues 系統解決這個問題。

Layer 和 Clues 解決不同層次的問題。Layer 回答「這東西是什麼」（身份辨識，有階段性，共享樹是混淆機制的骨幹）。Clues 回答「我對這東西了解多少」（在已知身份的前提下，揭露材質、年代、標記、損傷等線索）。Layer 保留為 threshold bucket，Clues 在 layer 之內展開。

Clues 是獨立系統，全部掛在 Item 上定義。部分 clue 可以跨 item 共享（同品類的通識線索，如「brass material」），部分是 item 專屬的（辨別個體的關鍵線索，如「Vienna maker's mark」）。共享 clue 是對品類的通識，專屬 clue 才是辨別個體、判斷高低價值的關鍵。

每條 clue 有兩層機制：

**prerequisite** — 門檻條件，結構與現有 LayerUnlockAction 一致：category_rank、skill + level、perk、inspection_level 下限。全部 optional，沒寫就是無門檻。純 bool check。

**trigger** — 揭露方式，兩種：
- **passive** — 符合 prerequisite 的瞬間自動揭露。老手進 Inspection 點開物品，系統掃一遍，prerequisite 過了就直接亮。這是「老手一看就 3/9」的那個 3。
- **on_action** — 符合 prerequisite 之後，還需要玩家執行特定動作才揭露。Inspection 裡多花一次 stamina、或 Home 階段做一次 Study action，每次動作揭露一條未揭露且 prerequisite 已滿足的 clue。

這使得同一條共享 clue 在不同 item 上可以設不同的 trigger——普通油燈上材質是 passive，但刻意偽裝成銀器的贗品上可能是 on_action，得仔細看才發現其實是黃銅。

玩家技能等級決定一次 inspect 能翻出幾條 clue。新手看到 2/8 條，只知道粗略資訊。老手看到 7/8 條，能做出有根據的價值判斷。同樣 rarity 的物品因 clue 組合不同而有不同價值，而判讀 clue 的能力是玩家自己累積的隱性知識。

### 養成 vs 生存壓力

討論過固定扣額否則 Game Over 的模式。否決，因為延遲回報的核心循環與生存壓力矛盾——玩家會被迫快速翻貨、不研究、不冒險。壓力來源改為機會成本型：不是付不起就死，而是錢不夠就錯過好機會（特殊拍賣、好 Location 入場費）。Knowledge、Skill、Car、未來開店都是養成軸線。

---

## 工作拆分

依賴關係分析：Phase 0 是所有後續工作的前置。Phase 1 和 Phase 2 互不依賴可平行。Phase 3 資料層不依賴 Phase 2 可先做，但 UI 整合需等 Phase 2。Phase 4 是整合層。Phase 5 需等前面都穩了才知道該留什麼砍什麼。

### Phase 0 — Runtime Veil Cleanup（已完成）

已完成 runtime-first 清理：新增 `ItemEntry.inspected`，`is_veiled()` 改為 `return not inspected`，新建物品固定 `layer_index = 1`、`inspected = false`。`unveil()` 保留為相容方法，但只負責標記 inspected。Lot 生成不再使用 `veiled_chance`，`LotData.veiled_chance` 只為 YAML / TRES 相容暫留並標記 deprecated。Inspection 的 Peek action 已移除；Inspect 第一次 reveal，後續才 advance scrutiny。Reveal 階段改為標記 won items inspected。

注意：這一階段尚未刪除 YAML / TRES item layer 0，也尚未改 dev tools 的 item layer 驗證規則。這是刻意的 runtime-first 決策，避免資料遷移和玩法改動混在同一個小 PR。

### Phase 1 — Commodity 系統（已完成）

已完成第一版 Commodity 系統，並額外完成一個前台統一 refactor。

已完成範圍：

1. 定義 Commodity 資料型別（commodity_id、display_name、category_data、base_value）和通用 commodity pool
2. 新增 runtime `CommodityEntry`，condition 每次生成時從 0.0–1.0 隨機 roll
3. LotEntry / RunRecord 支援一個 lot 同時包含 N 個 Commodity + M 個 ItemEntry
4. `LotEntry.lot_objects` 以 shuffled mixed order 提供 Inspection / List Review / Reveal / Run Review 顯示
5. Reveal 後 Commodity 自動標記出售，Run Review / Day Summary / run resolution 顯示並入帳 `Commodity Sales`
6. Cargo / Storage / Merchant / Research 仍只接 `ItemEntry`，Commodity 不進 cargo/storage/research 管線
7. 新增 runtime base `LotObjectEntry`，讓 Item / Commodity 的 card、row、tooltip、action bar 走同一套前台 API，避免未知狀態和欄位格式不同步
8. 暫不做「帶回家」標記，避免 Phase 1 同時牽 Storage、market price 路線和額外 UI

這一階段已解決「跑一趟完全沒有正回饋」的核心痛點。玩家仍需在 Inspection 階段花 inspect action 才知道某個未知物件是 Item 還是 Commodity；Reveal 後 Commodity 會顯示並自動賣出。

### Phase 2 — Inspection Grid 重做（下一步，依賴 Phase 0）

最大的一塊，拆成四步：

1. Grid 佈局和物品擺放（8×8 格子、隨機放置、黑色剪影顯示）
2. 點擊 reveal 互動（花 stamina、顯示顏色和 layer 1 資訊）
3. 遮擋成本計算（row tier + column scan：前兩排 1 點，中間排視遮擋加到 2 點，後排以此類推到 3-4 點。逐 cell 掃描取最佳視線）
4. Tooltip card（hover 顯示已知資訊）

每一步都可以獨立跑起來看。沒有 Clues 的 Inspection grid 就是只顯示 layer 1 基本資訊，已經比現在的清單好。Phase 1 已經先把前台 display API 統一成 `LotObjectEntry`，Phase 2 可以直接復用這套 card/list/tooltip 表現，不需要再分 Item / Commodity 兩條 UI 路。

### Phase 3 — Clues 系統（資料層依賴 Phase 0，不依賴 Phase 2）

資料層可以獨立做，UI 整合等 Phase 2：

1. Clue 資料定義（clue_id、display_text、prerequisite、trigger enum）
2. ItemEntry 加 `revealed_clue_ids: Array[String]`
3. Prerequisite 檢查邏輯（複用 LayerUnlockAction 的條件結構）
4. Passive / on_action 觸發邏輯
5. 先掛到現有 Research / Study 流程上驗證——Home 階段做 Study 時觸發 on_action clue 揭露

Phase 2 的 Inspection grid 做完後再把 Clue 揭露接進去。

### Phase 4 — 整合（依賴 Phase 1-3）

- Inspection grid 接上 Clues 的 passive 揭露
- Tooltip card 顯示已揭露的 clues
- Commodity 在 Inspection grid 裡復用 `LotObjectEntry` card/list 表現。玩家 inspect 前不知道物件是 Item 還是 Commodity；inspect 後才揭露 Commodity 身分與 sale value

### Phase 5 — Day Summary 簡化（依賴 Phase 1-4）

等 Phase 2-4 穩了再碰。Phase 1 已先把 Commodity Sales 加進 Day Summary，但尚未做結構性簡化。後續再決定是否拿掉 Net 數字，改成輕量確認加 Balance，或非拍賣日直接跳過。

### 暫不排入開發計畫

- **週系統** — current_day 加星期概念、禮拜一拍賣日、Weekly Report。因 Commodity 自動變現已解決核心痛點，急迫性低。
- **養成方向** — 不做固定扣額 Game Over。壓力來源是機會成本（特殊拍賣事件、高入場費 Location）。Knowledge、Skill、Car 升級、未來開店作為長期養成軸線。設計層決策，不需要立即實作。

---

## 不動的東西

Location / Lot 資料結構與選擇邏輯、Merchant 談判系統、Research tick 邏輯（scrutiny 每天 tick 不變）、Car / Vehicle 系統、RunRecord 資料物件核心、ItemEntry 的 Layer 推進與 Research 管線（Commodity 分流後 ItemEntry 只服務有研究價值的物品）、Cargo 的裝車操作（維持自動 reveal）。
