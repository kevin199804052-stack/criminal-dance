# 技術／架構文件 — criminal-dance.html

記錄程式「**怎麼運作**」。對照：規則看 `CLAUDE.md`、各功能規格看 `specs/`、進度看 `docs/sdd-progress.md`。
（本檔描述現況 build v16。）

## 總覽
- **單一檔** `criminal-dance.html`，HTML + CSS + 原生 JS，無編譯。
- 一個全域 `state` 物件保存整局狀態；`render()` 依 `state.phase` 畫出對應畫面（狀態機）。
- 兩種模式：**本機同桌**（共用一台、靠遮擋畫面）與**線上連線**（Firebase，目前只完成大廳）。

## 全域狀態
| 變數 | 內容 |
|------|------|
| `state` | 當前牌局：見下表 |
| `scores` | 跨局累積分數，長度＝玩家數 |
| `playerNames` | 記住的玩家名字 |
| `WIN_SCORE` | 賽末門檻（依人數，3–5→5、6–8→10） |
| `mode` | `null`=主選單 / `'local'` / `'online'` |
| `withInspector` | 是否加入警部牌 |
| `setupCount`/`setupNames` | 設定畫面的人數與暫存名字 |
| `net` | 線上狀態（見「線上」段） |

### `state` 物件
```
state = {
  players: [ { name, hand:[卡片代號...], accomplice:bool } ],
  current: 目前輪到的玩家索引,
  firstMoveDone: 開場第一發現者是否已打出,
  phase: 畫面狀態(見下),
  log: [字串...],          // 事件記錄，最新在前
  pending: 暫存中的動作資料（選目標/交易/神犬/情報交換/reveal）,
  gameover: null | { title, body, points, matchWinner, ... },
  inspector: null | { by:出警部者, target:被盯者 }   // 遊戲結束時結算
}
```

## 畫面狀態機（phase）
`render()` 分流：`mode==='online'` → `netRender()`；否則 `state===null` → 主選單/設定；否則依 `state.phase`：

| phase | 畫面 | 函式 |
|-------|------|------|
| pass | 換人遮擋 | renderPass |
| turn | 出牌（手牌） | renderTurn |
| target | 選目標（偵探/目擊者/交易/神犬/警部） | renderTarget |
| tradeGive / tradePass / tradeReceive | 交易三步 | render… |
| dogPass / dogDiscard | 神犬：交給對方棄牌 | render… |
| infoSwapPass / infoSwapPick | 情報交換：各自選牌 | render… |
| reveal | 私密資訊（目擊者/少年結果） | renderReveal |
| (gameover) | 結算＋計分＋動畫 | renderGameOver |

## 卡片與牌庫
- `CARDS`：每張牌的 `name` / `desc` / `crime` 旗標（12 種＋警部）。
- `CARD_POOL`：標準 32 張各卡最大張數。
- `MANDATORY`：各人數的必放牌。
- `buildDeck(N, withInspector)`：依人數從牌池組牌（8 人＝全 32；其餘必放牌＋隨機補滿到 N×4；3 人不放共犯；警部開→替換一張神犬）。

## 出牌流程（核心）
1. `playable(card)` 判斷可否打出（犯人需手牌1、偵探/警部需≤3、開場限第一發現者）。
2. `playCard(idx)`：需選目標的牌（偵探/目擊者/交易/神犬/警部）→ 設 `pending`、進 `target`；否則 `resolveSimple`。
3. `resolveSimple(card,idx)`：先移除打出的牌，再依卡別結算（第一發現者/一般人/不在場/共犯/少年/情報交換/謠言/犯人逃脫）。
4. `resolveTargeted(card,idx,t)`：偵探指認、目擊者偷看、交易、神犬、警部。
5. 多步驟牌用 `pending` + 額外 phase 接續（交易、神犬、情報交換）。
6. 每次結束呼叫 `endTurn()`：檢查是否和局，否則 `nextActive()` 找下一個有手牌的玩家、進 `pass`。

## 計分
- `pointsFor(type, winnerIdx, criminalIdx)`：偵探+2 / 神犬+3 / 逃脫(犯人+共犯)+2，平民+1，犯人陣營0。長度＝玩家數。
- `gameOver(title, body, points)`：累加 `scores`；做**警部結算**（被盯者結束時仍持犯人牌→比照神犬，與原結局逐人取較高分）；算**賽末贏家** `matchWinner`（最高分≥WIN_SCORE且唯一）。
- `scoreHtml` 結算計分板（高→低）、`scoreStrip` 換人小條。

## 工具函式
`shuffle` / `log` / `pname(i)` / `nextActive(from)`(繞 N) / `holderOfCriminal()` / `removeCard(p,card)` / `doRumor()`（右鄰盲抽，繞 N）/ `inspectorBanner()`（警部打出後才公開）。

## 線上（Firebase Realtime Database）
- 設定 `firebaseConfig`，`fbInit()` 初始化；compat SDK 由 CDN 載入。
- `net = { phase:'entry'|'lobby', code, playerId, myName, isHost, room, ref }`。
- 大廳函式：`enterOnline / createRoom / joinRoom / attachRoom / leaveRoom`；畫面 `netRender / netEntry / netLobby`。
- `hostStart()`：**目前是待開發樁**（按開始遊戲跳提示）。
- RTDB 資料結構（大廳）：
  ```
  rooms/{房號}/
    host: playerId
    status: "lobby"
    createdAt
    players/{playerId}: { name, joinedAt }
  ```
- 玩家斷線：`onDisconnect().remove()` 自動退出。

## 引擎 vs UI 界線（現況：混在一起）
- **偏「引擎」(純邏輯)**：`buildDeck`、`playable`、`resolveSimple`/`resolveTargeted` 的狀態變更、`pointsFor`、`endTurn`、`nextActive`、`doRumor`。
- **偏「UI」**：所有 `render*`、inline `onclick`、`pending` 驅動的多步畫面。
- 現況兩者**直接互相呼叫、共用全域 state**（例如結算函式裡直接 `render()`）。要做線上本體時，需把「引擎只回傳新狀態」與「UI/同步」解耦——見下方待討論決策。

## 待討論／待決策（給線上本體鋪路）
見 `docs/sdd-progress.md` 與下次對話討論：
1. 是否把引擎抽成不碰 DOM 的純函式層。
2. 線上本體的 Firebase 資料模型（動作佇列、個人視圖）。
3. 是否維持單一檔案。
4. 線上是否沿用 phase 狀態機或另設視圖模型。
