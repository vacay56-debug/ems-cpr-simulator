# CLAUDE.md

本檔案為 Claude Code（claude.ai/code）在此儲存庫中作業時提供的指引。

## 專案概述

TPS-EMS 韌性急救訓練模擬系統是一套針對台灣到院前緊急救護教學設計的瀏覽器端 CPR 訓練模擬器。整個應用程式是**單一自含檔案**：`index.html`——無建置系統、無外部相依、無套件管理器。

## 執行應用程式

直接在瀏覽器中開啟 `index.html` 即可，不需要伺服器。若要搭配即時重載進行本機測試：

```sh
python3 -m http.server 8080
# 然後開啟 http://localhost:8080
```

本專案沒有 lint 指令、沒有測試套件、也沒有 CI 流程。

## 架構

應用程式為純 HTML + 原生 JS，全部寫在單一 `<script>` 區塊（約 900 行）。整體資料流如下：

```
State (S) → applyRhythmDefaults() → advancePhase() → sampleChannels() → Canvas 掃描渲染
                                                                       → updateVitalsUI()
```

### 核心物件

| 符號 | 用途 |
|------|------|
| `S` | 全域狀態物件：心律、生命徵象、CPR 狀態、節律器、用藥清單、統計數據 |
| `RHYTHMS` | 14 種心律定義（名稱、HR、可電擊、有組織、有脈搏、寬 QRS 等屬性） |
| `DRUGS` | 6 種 ACLS 藥物定義，各含直接修改 `S` 的 `apply()` 閉包 |
| `clk` | 波形時鐘：`master` 主時間、`beatT0`／`atrialT0`／`respT0` 相位起點、`rr` 間期 |
| `channels` | 以 Canvas ID 為索引的渲染目標（`cEcg`、`cPleth`、`cAbp`、`cCo2`、`cCpr`） |
| `lead12` | 12 導程 ECG 覆蓋層專用的獨立 Canvas 映射表 |

### 波形管線

1. `advancePhase(dt)` — 推進 `clk.master`，並在每次心跳／心房／呼吸週期結束時重置相位起點
2. `sampleChannels()` — 以當前相位偏移 `te = clk.master - clk.beatT0` 分別呼叫 `ecgMorph`、`plethMorph`、`abpMorph`、`co2Morph`
3. `frame()` — rAF 主迴圈；每幀執行 `Math.round(PX_PER_SEC * dt)` 步，每步在掃描模式下畫一個像素（先清除前方 14px，再繪製並推進 x）
4. `renderCprWave(dt)` — CPR 通道的獨立渲染，使用 `sin(f * π)` 脈衝形狀

波形形態採用高斯疊加（`gauss(t, center, width, amplitude)`）。特殊情況：VF 使用多正弦疊加雜訊（`vfWave`）；心搏停止加入微小漂移；尖端扭轉使用正弦振幅包絡產生紡錘狀外觀。

### CPR 引擎

`compress()` 由空白鍵、浮動大圓鈕或「按壓一次」按鈕觸發。它將每次按壓間隔記錄在 8 個樣本的滾動緩衝區（`S.cpr.intervals`），計算速率並隨機產生深度（4.6–5.8 cm）；無脈搏時會拉升 EtCO₂。`qualityFactor()` 依速率分數（目標 100–120/min）與變異係數回傳 0–1。`updateCprMetrics(dt)` 在超過 1.5 秒未按壓時自動停止 CPR。

### 除顫／整流

`chargeDefib()` / `deliverShock()` 實作機率式轉復。可電擊心律的電擊成功機率：基礎 0.45 + 能量 ≥ 150 J 加 0.15 + CPR 品質因子 + 藥效加成（腎上腺素 +0.08、胺碘酮 +0.14）。成功終止 VF 且已累計 ≥2 次電擊後，有 50% 機率直接呼叫 `achieveROSC()`，否則轉為 PEA。

### 教官 ↔ 學員同步

三種角色為**單機合一（solo）**、**學員端（trainee）**、**教官端（instructor）**。非單機模式下：

- 透過 `hasStore` 守衛使用 `window.storage`（Claude Code 平台 API），輪詢間隔 800 ms
- 儲存鍵：`tpsems:{code}:cmd`（教官 → 學員）與 `tpsems:{code}:stat`（學員 → 教官）
- 指令帶有 `seq` 序號；`S.seqIn` 防止重複執行同一指令
- `ctrlPush(kind)` — 教官推送心律／生命徵象狀態
- `pushStatus()` — 學員回傳 CPR 品質給教官
- 若 `window.storage` 不存在（一般瀏覽器），`S.net` 會靜默強制設為 false，以單機模式運行

### 控制甲板渲染

`renderDeck()` 依目前 `tab` 變數（`resus`、`drug`、`ctrl`）以 `innerHTML` 重繪底部面板。`wireDeck()` 在每次重繪後重新綁定所有事件。「report」分頁會開啟 `#ovRep` 覆蓋層而非設定甲板內容。

### 12 導程 ECG 覆蓋層

`build12()` / `loop12()` / `lead12sample()` 在覆蓋層開啟時執行獨立的 rAF 迴圈（`raf12`）。每個導程對 `ecgMorph()` 的輸出套用各自的振幅係數（`cfg.f`）與反相旗標，V1／V2 另有 rS 形態特殊處理，STEMI 時特定區域導程會疊加 ST 段上升。

## 重要慣例

- 所有使用者介面文字為繁體中文（zh-Hant-TW）；程式碼註解中英混用。
- `$` / `$$` 是本地縮寫（`document.querySelector` / `querySelectorAll`），勿與 jQuery 混淆。
- `clamp(v,a,b)` 與 `rnd(a,b)` 為本地工具函式；`now()` 包裝 `performance.now()`。
- 狀態直接修改 `S` 上的屬性，不使用不可變性或響應式框架。
- 任何影響控制面板的狀態變更後必須呼叫 `renderDeck()`。生命徵象 UI 則由 `updateVitalsUI()` 在每個動畫幀自動更新。
- `window.storage` API 是 Claude Code 遠端執行環境的功能，並非標準瀏覽器 localStorage。所有依賴網路的程式碼均由 `hasStore` 守衛保護。
