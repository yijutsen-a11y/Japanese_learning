# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案概觀

「可不可愛、日文厲害」——像素 RPG 語言學習遊戲，三大陸：日本（五十音到 N1）＋韓國（諺文到 TOPIK II）＋美國（IELTS 5.0 到 8.0）。**整個遊戲就是一個檔案 `index.html`**（HTML+CSS+JS），零依賴、無 build。目前內容狀態與玩法清單見 `PROJECT_STATUS.md`（內容大改時要同步更新它；父層工作區 `D:\claude\README.md` 的總覽表也一併更新——注意本 repo 自己的 `README.md` 只是兩行 stub，總覽表在 repo 之外的父目錄）。

執行：直接用瀏覽器開 `index.html`（建議直式視窗）。

部署：GitHub repo 為 https://github.com/yijutsen-a11y/Japanese_learning ，GitHub Pages 已啟用（https://yijutsen-a11y.github.io/Japanese_learning/ ）——push 到 `main` 即上線，無其他部署步驟。

## 硬性限制

- **Artifact 嚴格 CSP**：禁止外部圖片、字型、CDN、fetch。所有素材以 Canvas（像素 sprite）、CSS、WebAudio（音效）程式合成；發音用瀏覽器內建 `speechSynthesis`（`VLANG` 對應 ja-JP／ko-KR／en-US），沒有該語音時聽力題自動退化為閱讀題（`ttsOK()` 判斷）。
- 保持單檔：不要拆檔、不要引入框架。
- 韓文字母（諺文）沒有筆順資料（KanjiVG 只有日文假名），寫字板會自動退回字型描邊——這是預期行為。筆順資料離線嵌入在 `STROKES` 常數（KanjiVG 點列，74 字）。

## 驗證（改完必跑）

無測試框架。把 `<script>` 抽出來做語法檢查（repo 沒有 `.gitignore`，寫到 temp 而不是工作目錄）：

```powershell
node -e "const s=require('fs').readFileSync('index.html','utf8');require('fs').writeFileSync(process.env.TEMP+'/game.js',s.match(/<script>([\s\S]*)<\/script>/)[1]);"
node --check $env:TEMP\game.js
```

這個正規式假設全檔**只有一個** `<script>`（目前 401–4118 行）與一個 `<style>`——維持這個前提，別加第二個 script 標籤。

改資料層後建議再跑資料完整性檢查：寫個 stub 腳本（stub 掉 `document`/`localStorage`/`window` 等，eval 資料段），檢查：每個 sprite 所有 row 等寬、章節 id 不重複、每個 talk step 的選項恰好一個 `ok:1`。最後在瀏覽器實開確認。

## 架構（單檔內的分區）

`index.html` 三大塊：CSS（頂部 `<style>`）→ 畫面 DOM（9 個 `div.screen`：`scr-title/map/srs/region/learn/battle/talk/result/write`，用 `go(id)` 切換）→ 單一 `<script>`。JS 依註解分節（`/* ==== 節名 ==== */`）：`內容資料 → 韓國大陸 → 美國大陸 → 世界設定 → 每日一句 → 像素 sprite → 存檔 → 音效 → TTS → 小工具 → 進度與解鎖 → 錯題本/SRS → 標題/世界地圖 → 學習模式 → 戰鬥 → 勝敗結算 → 對話劇情 → 畫符文 → 啟動`。

### 檔案地圖（4120 行 / 300KB，別整檔讀）

`grep -n "====" index.html` 一次列出所有分節。粗略分界：

| 行 | 內容 |
|---|---|
| 9–230 | `<style>` |
| 236–362 | 9 個 `div.screen` 的 DOM |
| 403–3036 | **資料層**（日 403／韓 1580／美 2468 起），佔全檔約 2/3，加內容只動這段 |
| 3037–4118 | 世界設定、索引 pass、sprite、存檔、音效、TTS、SRS、UI 與戰鬥邏輯 |

### 資料層（最常改的地方）

內容來源是三個 region 陣列：`REGIONS`（日本）、`REGIONS_KR`（韓國）與 `REGIONS_US`（美國），由 `WORLDS={jp,kr,us}` 統整（各帶 `tts` 語言與 `letter` 稱呼）。一個 region 含多個 chapter，chapter 有四種 `kind`：

- `kana`：用 `kanaCh(id,name,怪物名,sprite,[[字,羅馬音,聯想,例字],...])` 建立（日文假名與韓文字母都用它；美國大陸沒有字母關）
- `vocab` / `phrase`：用 `vocabCh()`/`phraseCh()` 建立，item 用 `vh(假名,漢字,中文,聯想)`（日，假名為主）、`vk(漢字,假名,中文,聯想)`（日，漢字為主，N3+）、`kv(韓文,羅馬音,中文,聯想)`（韓）或 `ve(單字,詞性,中文,用法)`／`pe(英文句,中文,用法)`（英），**一行一字**方便持續擴充
- `talk`：Visual Novel 對話，直接寫物件 `{kind:'talk', npc:{name,sprite}, steps:[{jp,zh,ch:[選項]}]}`，每 step 的 `ch` 恰好一個 `ok:1`，錯誤選項要有 `why`（解釋為什麼失禮）

章節與區域 id 需**跨三大陸全域唯一**（韓國用 `kr*`/`kc*`/`kd*` 前綴，美國用 `us*`／章節 `e*`）。

羅馬拼音欄 `ro` 三種 helper 來源不同：`vh()`/`vk()` 由 `toRomaji(kana)` 自動產生（改假名等於改拼音，別手填）、`kv()` 第二個參數手寫、`ve()`/`pe()` 沒有 `ro`。

**加一章的檢查清單**：① id 全域唯一 ② sprite 名稱必須是既有的 7 個怪物之一 ③ 放進對應 region 的 `chapters` 陣列，位置即解鎖順序 ④ boss 別手寫 ⑤ 單字篇後面照慣例補一個配套 `phrase` 造句章 ⑥ 跑語法檢查＋瀏覽器實開。

**美國大陸走 IELTS 5.0→8.0**：假設玩家已有底子，所以沒有字母關，全是 `vocab`＋配套造句 `phrase`＋4 場情境 VN。`ve()` 的 `sub` 放詞性、`memo` 放搭配（collocation）——雅思的分數在搭配上，別寫成單純的中文對照。英文字串含 `'` 時整串改用雙引號。

**魔王戰不要手寫**：載入時程式自動為每個 region 追加 `{id:'rX_boss', boss:true}` 章節，題池＝該區全部非 talk items（抽 12 題、時限 8 秒）。加內容後魔王戰自動更新。資料層之後會建索引 `CHMAP`/`REGION_OF`/`RMAP`/`KANA_POOL`（依大陸分池，聽力干擾項不跨語言）／`ITEM_BY_KEY`，同時在每個 item 上蓋出身章節 `_cid`/`_ck`/`_cw`——錯題本靠這三個欄位還原題型與干擾項池，加新 helper 或新 kind 時別漏掉這個 pass。

**每日一句**：`DAILY={jp:[…],kr:[…]}` 各 31 句高頻口語（`{t,ro,zh,note}`），依 day-of-year 輪替，`startGame(world)` 進大陸時彈出。

### 解鎖與存檔

- 章節：需通過前一章；boss 需通關全區非 boss 章節（`chapterUnlocked`）；區域：前一區通關半數章節（`regionUnlocked`）
- 存檔：`localStorage` key `kawaii_quest_save_v1`，結構 `{exp, clears:{章節id:星數}, nextBuff, srs:{key:{box,due,miss,cid,w}}, srsDone}`。改結構時注意 `loadSave()` 用 `Object.assign` 補預設值做向後相容

### 錯題本 / 間隔複習（SRS）

Leitner 盒子制：`SRS_STEP=[0,1,2,4,7,15]` 天，答錯 `srsMiss()` 掉回第 0 盒，複習答對 `srsHit()` 往後推一盒，走完 `SRS_MAX` 就畢業（刪 entry、`srsDone++`）。字的識別碼是 `itemKey(it)=大陸|jp|zh`，靠 `ITEM_BY_KEY` 反查回 item——**改動 item 的 `jp`/`zh` 等於換 key，舊錯題會對不上**（`srsEntries()` 會直接略過查不到的舊 key，不會壞檔）。

`startReview()` 把到期的字塞進臨時章節 `CHMAP.__review__`（`review:true`），戰鬥流程共用；`nextQ()` 看到 `B.ch.review` 就改用每個字自己的 `_ck`/`_cw`/`_cid` 決定題型、TTS 語言與干擾項來源。

### 像素 sprite

`SPRITES` 是字串網格（每字元對應 `PAL` 調色盤一色，`.` 透明），`drawSprite(canvas,rows,scale)` 畫出。新怪物＝加一個 grid，**所有 row 必須等寬**。

目前只有 7 個怪物可選：`slime` `bird` `cat` `fox` `oni` `dragon` `ghost`（另有 `hero`）。章節寫了不存在的 sprite 名稱**不會**在語法檢查時報錯，要到該關開打才炸——要嘛沿用這 7 個，要嘛先加 grid。

### 對話劇情（VN）的節奏

答對後**不是**固定延遲換頁：`talkPick()` 把 `say()` 的第三參數當回呼，等整句唸完（`onend`／估時兜底）再保底 1 秒才 `renderTalk()` 或 `endTalk()`，避免話沒講完就跳下一章。`say(txt,lang,onEnd)` 的 `onEnd` 保證只觸發一次（`once()`），沒有 TTS 時同步觸發。

### 戰鬥出題

`nextQ()`：有 TTS 時 40% 機率出聽力題；kana 章節選項用假名/羅馬音，其他 kind 用中文意思。快答觸發 CRITICAL、答錯排回隊尾、時間到扣心。祝福（`BUFFS`）影響下一場的時間/EXP/心數。
