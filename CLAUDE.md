# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概觀

「可不可愛、日文厲害」——像素 RPG 日文學習遊戲（完全新手到 N1）。**整個遊戲就是一個檔案 `index.html`**（HTML+CSS+JS，約 1700 行），零依賴、無 build。目前內容狀態與玩法清單見 `PROJECT_STATUS.md`（內容大改時要同步更新它與 root `README.md` 總覽表）。

執行：直接用瀏覽器開 `index.html`（建議直式視窗）。

## 硬性限制

- **Artifact 嚴格 CSP**：禁止外部圖片、字型、CDN、fetch。所有素材以 Canvas（像素 sprite）、CSS、WebAudio（音效）程式合成；日文發音用瀏覽器內建 `speechSynthesis`（ja-JP），無日文語音時聽力題自動退化為閱讀題（`ttsOK()` 判斷）。
- 保持單檔：不要拆檔、不要引入框架。

## 驗證（改完必跑）

無測試框架。把 `<script>` 抽出來做語法檢查：

```powershell
node -e "const s=require('fs').readFileSync('index.html','utf8');require('fs').writeFileSync('game.js',s.match(/<script>([\s\S]*)<\/script>/)[1]);"
node --check game.js
```

改資料層後建議再跑資料完整性檢查：寫個 stub 腳本（stub 掉 `document`/`localStorage`/`window` 等，eval 資料段），檢查：每個 sprite 所有 row 等寬、章節 id 不重複、每個 talk step 的選項恰好一個 `ok:1`。最後在瀏覽器實開確認。

## 架構（單檔內的分區）

`index.html` 三大塊：CSS（頂部 `<style>`）→ 畫面 DOM（8 個 `div.screen`：`scr-title/map/region/learn/battle/talk/result/write`，用 `go(id)` 切換）→ 單一 `<script>`。JS 依註解分節：`內容資料 → 像素 sprite → 存檔 → 音效 → TTS → 小工具 → 進度與解鎖 → 標題/世界地圖 → 學習模式 → 戰鬥 → 勝敗結算 → 對話劇情 → 畫符文 → 啟動`。

### 資料層（最常改的地方）

`REGIONS` 陣列是唯一內容來源，一個 region 含多個 chapter，chapter 有四種 `kind`：

- `kana`：用 `kanaCh(id,name,怪物名,sprite,[[假名,羅馬音,聯想,例字],...])` 建立
- `vocab` / `phrase`：用 `vocabCh()`/`phraseCh()` 建立，item 用 `vh(假名,漢字,中文,聯想)`（假名為主，N5/N4）或 `vk(漢字,假名,中文,聯想)`（漢字為主，N3+），**一行一字**方便持續擴充
- `talk`：Visual Novel 對話，直接寫物件 `{kind:'talk', npc:{name,sprite}, steps:[{jp,zh,ch:[選項]}]}`，每 step 的 `ch` 恰好一個 `ok:1`，錯誤選項要有 `why`（解釋為什麼失禮）

**魔王戰不要手寫**：載入時程式自動為每個 region 追加 `{id:'rX_boss', boss:true}` 章節，題池＝該區全部非 talk items（抽 12 題、時限 8 秒）。加內容後魔王戰自動更新。資料層之後會建索引 `CHMAP`/`REGION_OF`/`KANA_ALL`。

### 解鎖與存檔

- 章節：需通過前一章；boss 需通關全區非 boss 章節（`chapterUnlocked`）；區域：前一區通關半數章節（`regionUnlocked`）
- 存檔：`localStorage` key `kawaii_quest_save_v1`，結構 `{exp, clears:{章節id:星數}, nextBuff}`。改結構時注意 `loadSave()` 用 `Object.assign` 補預設值做向後相容

### 像素 sprite

`SPRITES` 是字串網格（每字元對應 `PAL` 調色盤一色，`.` 透明），`drawSprite(canvas,rows,scale)` 畫出。新怪物＝加一個 grid，**所有 row 必須等寬**。

### 戰鬥出題

`nextQ()`：有 TTS 時 40% 機率出聽力題；kana 章節選項用假名/羅馬音，其他 kind 用中文意思。快答觸發 CRITICAL、答錯排回隊尾、時間到扣心。祝福（`BUFFS`）影響下一場的時間/EXP/心數。
