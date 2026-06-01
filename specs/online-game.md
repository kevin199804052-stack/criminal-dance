# 規格：線上遊戲本體（連線對戰）

狀態：**草稿，待使用者批准**
架構決策（已批准，見 docs/sdd-progress.md）：房主當裁判跑現有引擎、Firebase 三區資料模型、單一檔、安全性隨後加。

## 1. 目標
讓大廳「開始遊戲」之後，房內玩家能在**各自的裝置**上連線對戰：房主端跑遊戲、每人只看到自己的手牌、輪到誰誰出牌、即時同步。

## 2. 架構（房主＝裁判）
- 房主瀏覽器保有完整 `state`（重用現有單機引擎）。
- 房主每次狀態改變後，把資訊寫進 Firebase：
  - `rooms/{房號}/public`：`{ phase, current(輪到誰), log, scores, gameover, inspectorTarget, 玩家公開資料 }`
  - `rooms/{房號}/views/{playerId}`：`{ hand:[該玩家手牌], prompt:該玩家現在該做什麼 }`
  - `rooms/{房號}/actions`：玩家 push 動作，房主監聽後套用、清除。
- 非房主玩家：監聽自己的 `public` + `views/{我}`，據此畫面；操作時 push 一個 action 到 `actions`，等房主回新狀態。

## 3. 動作模型（actions）
每個 action：`{ by:playerId, type, payload }`。type 對應引擎入口：
- `play`（payload: 手牌index）→ 房主呼叫 `playCard`
- `target`（payload: 目標玩家index）→ `chooseTarget`
- `tradeGive` / `tradeReceive`（payload: index）
- `dogDiscard`（payload: index）
- `infoSwapPick`（payload: index）
- `reveal-ack`（看完私密資訊按繼續）
房主**只接受「當前應由該玩家操作」的 action**（防亂送）。

## 4. 「現在輪到誰操作」對應表（關鍵）
線上沒有換裝置，房主要算出每個 phase「該由誰動作」，把提示放進那個人的 view：
| phase | 該操作的人 |
|-------|-----------|
| turn | `current` |
| target | `current` |
| tradeGive | `current` |
| tradeReceive | 交易對象 `pending.target` |
| dogDiscard(經 dogPass) | 神犬目標 `pending.target` |
| infoSwapPick(經 infoSwapPass) | `pending.queue[qi]` |
| reveal | 觸發者 `current`（私密只給他看） |
其餘玩家的 view：顯示「等待 ○○○…」。

> 註：單機的 pass/dogPass/tradePass/infoSwapPass 這些「換人遮擋」phase，在線上**不需要**（各自看各自畫面），房主可直接跳過、把對應 phase 視為「該某人操作」。

## 5. 需要你拍板的決策點 ⚠️
1. **人數**：線上沿用房內人數 **3–8**（大廳目前限定 4 人才能開始）。提案：放寬成 **≥3 人**房主即可開始，用 `buildDeck(房內人數)`。
2. **房主中途離線**：房主是裁判，離線遊戲無法繼續。提案 v1＝**顯示「房主已離線，遊戲結束」**並回大廳（不做房主轉移，太複雜）。
3. **一般玩家重整/斷線重連**：提案＝把 `playerId` 存進 `sessionStorage`，重整後用同 id 重新監聽自己的 view → **可無痛接回**。
4. **沒有出牌時限**：提案 v1 不做 AFK 計時（朋友間玩），保持簡單。
5. **安全規則時機**：提案＝先做到「能玩」，**緊接著**加匿名登入＋規則（只能讀自己的 view）。

## 6. 不做（Out of scope，本期）
- 房主轉移、AFK 踢人、觀戰。
- 安全規則本身（緊接的下一個工作項，不在本規格實作範圍）。

## 7. 驗收條件（分階段測；多數需多瀏覽器分頁模擬）
- A. 房主開始後，`buildDeck(房內人數)` 發牌，`public.phase` 進入遊戲、`views/{每人}` 各有自己的手牌。
- B. 非當前玩家的 view 顯示「等待 ○○○」、且**拿不到別人的手牌**（自己的 view 只有自己的牌）。
- C. 當前玩家 push `play` → 房主結算 → 所有人 view/public 同步更新。
- D. 多步驟互動正確路由：神犬→目標棄牌、交易→雙方各選、情報交換→各自選，提示出現在「正確的人」view。
- E. 私密資訊（目擊者/少年）只出現在觸發者的 view。
- F. 勝負結算與計分同步到 public，所有人看到結果（含動畫/犯人GIF）。
- G. 一般玩家重整後能接回自己的視圖。
- H. 房主離線 → 其他人看到「房主已離線」。

## 8. 實作順序建議（每步可驗）
1. 房主：開始→建牌→寫第一份 public + 各人 view。
2. 一般玩家：讀 view/public 畫出「我的手牌／等待中」。
3. 動作迴路：當前玩家 play → 房主套用 → 重廣播（先只做「出無目標的牌」）。
4. 加上選目標、交易、神犬、情報交換、reveal 的路由。
5. 結算同步 + 重連。
6.（之後）安全規則。
