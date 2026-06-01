# SDD Progress — 犯人在跳舞

Last updated: 2026-06-01

## Current phase
進入「**線上遊戲本體**」開發。架構決策已定（見下），正在寫規格（SDD Phase 1–2）。

## 線上本體 — 已批准的架構決策（2026-06-02）
1. **不大改引擎**：房主瀏覽器直接跑現有引擎函式，外加薄薄一層「動作接收＋個人視圖輸出」。
2. **Firebase 資料模型**：`rooms/{房號}/public`（公開：log/分數/輪到誰/phase）、`/views/{玩家}`（私密：自己手牌＋該做的提示）、`/actions`（玩家送出的動作，房主結算）。
3. **維持單一檔**（對 GitHub Pages 零編譯最省事）。
4. **安全性**：先求能玩；緊接著加「匿名登入＋安全規則（只能讀自己的 view）」防偷看（隱藏資訊是本遊戲核心）。

## 專案憲法（Project Constitution）
- 已存在於專案根目錄 [`CLAUDE.md`](../CLAUDE.md)（溝通用繁中、純 HTML+JS 單檔、引擎/UI 分離、標準版 4 人、卡表、計分、賽末門檻、Firebase 連線）。
- 主程式：[`criminal-dance.html`](../criminal-dance.html)，目前 build v11。

## 當前功能規格（Feature Spec）
- 規格檔：[`specs/inspector-警部.md`](../specs/inspector-警部.md)
- 驗收條件已定義：是（A–G）
- 待釐清決策（需使用者拍板）：
  1. 牌庫如何放入警部（提案：加警部×1、移除1張一般人，維持16張）
  2. 與犯人逃脫/偵探抓到的結算衝突如何處理
  3. 和局時的判定
  4. 本機同桌的「公開持有」呈現方式

## 整體專案進度
- ✅ 本機同桌完整可玩（12 種牌、出牌限制、三種勝利、計分、賽末門檻 WIN_SCORE=10）
- ✅ 情報交換＝每人自選傳左
- ✅ 線上大廳（建房/加入/玩家列表/房主開始/斷線退出），Firebase 連線測試通過
- ✅ 已 git init（main），準備用 GitHub Desktop 發布
- ✅ 已深度查證警部與 3–8 人規則（見 CLAUDE.md/對話記錄）
- 🚧 未完成：警部功能、多人(3–8人)規則、線上遊戲本體、上線到網址、Firebase 安全規則

## PEV log（Phase 4）
| Step | Plan | Execute | Verify | Status |
|------|------|---------|--------|--------|
| 警部牌 | 規格 specs/inspector-警部.md（4決策已批准） | build v12：加卡/牌庫/≤3限制/選目標/結算/公開橫幅 | A–G 全通過 | ✅ 完成 |
| 多人3–8人 | 規格 specs/multiplayer-多人.md（4決策已批准） | build v13：buildDeck(N)依人數建牌、引擎4→N一般化、設定選人數、WIN_SCORE隨人數 | A–K 全通過 | ✅ 完成 |
| 線上遊戲本體 | 規格 specs/online-game.md（5決策已批准） | build v17：房主裁判跑引擎、seatPrompt路由、broadcast(public+個人view)、applyAction(驗證)、netGame/renderGameView、重連playerId、房主離線標記 | 房主端邏輯：手牌不洩漏✅、開場規則✅、神犬/交易/情報交換路由✅、動作驗證✅。**待真機多人實測** | 🚧 核心完成，待實測 |

## 觀察（待你決定是否調整規格）
- 警部依現行規則「結束時目標仍持有犯人才算中」，主要在**偵探/神犬同時抓到**時連帶得分；單獨靠警部得分的情況很少（因為逃脫=犯人牌已打出、和局=手牌空）。若想讓警部更有用，可考慮新增「牌庫耗盡但仍有人持牌」之類的結束判定——這屬於改規格，先記著。
- 多人版警部開關「替換一張神犬/一般人」：小局（如 5 人）隨機建牌時若沒抽到神犬或一般人，警部會被略過 → 警部不一定每局出現。若想保證警部一定在場，需改規格為「加入警部時保證放入（替換任一非必要牌）」。先記著。

## Hashimoto log（每個 bug 留下永久護欄）
| Bug class | Guardrail added | Type |
|-----------|----------------|------|
| 交易索引錯位致崩潰 | 出牌前先移除該牌再選、測試覆蓋 | 測試案例(debug skill) |
| 交易出牌者無牌可給→卡死(v16) | chooseTarget 移除交易牌後若手空則落空換人 | 程式守衛+測試 |
| 警部開局公開(規格變更v16) | 持有時不公開、只在盯人後公開 | 規格修正+程式 |

## Next actions
1. 使用者批准 `specs/inspector-警部.md`（含 4 個決策點）。
2. 批准後進 Phase 3/4：擬計畫→拆任務→實作警部→用 criminal-dance-debug 逐項驗收（A–G）。
3. 完成後更新本檔與 CLAUDE.md，必要時把可重用模式抽成 skill（Phase 6）。

## Notes
- 相關 skill：`sdd-harness`（本流程）、`criminal-dance-dev`（開發規範）、`criminal-dance-debug`（測試流程）、`game-rules-crawler`（查規則）、`commit-and-upload`（commit 並上傳 GitHub）。
- ⚠️ 不要在本專案執行 sdd-harness 的 `init-project.sh`，它會覆蓋現有 CLAUDE.md。
