# ＳＴＹＬＡＤＡＹ 程式碼審查與優化報告（REVIEW）

- 審查對象：使用者貼上的 StylaDay 單頁 HTML 原始碼
- 原始碼大小：約 **130,501 字元**
- 優化版大小：**138,647 字元 / 149,967 bytes**（`index.html`）
- 優化版為何沒有變小？移除了 Tailwind CDN 的外部請求（原碼需額外下載並即時編譯整套 Tailwind，遠大於本檔案增量），同時補回了原本由 CDN 產生、但內建 CSS 缺漏的 utility class、無障礙屬性與修正註解；換來的是完全離線可用、無 FOUC、首屏不再等待 CDN。檔案本體的增量主要來自「補齊缺漏 CSS」章節，屬必要成本。
- 驗證方式：抽出 `<script>` 以 `node --check` 做語法檢查（通過）；以 grep / 字串掃描確認 CDN、寫死日期、隨機假日期邏輯已不存在；以程式檢查 JS 以 `getElementById` 引用的 id 全部存在於 HTML 中（missing ids: []）。

---

## 一、效能

### 1. 移除多餘的 Tailwind CDN script
- **問題**：原碼 `<head>` 載入 `https://cdn.tailwindcss.com/3.4.17`，但檔案內已內含完整的 standalone CSS。CDN 版 Tailwind 會在瀏覽器即時掃描 DOM 並產生樣式，造成 FOUC（無樣式閃爍）、拖慢首屏、離線時樣式有失效風險，且與內建 CSS 重複定義互相覆蓋，行為不可預期。
- **修正**：刪除該 `<script>`。逐一核對 HTML 與 JS 模板字串用到的 class，發現以下 class 原本只靠 CDN 產生、內建 CSS 並未定義，已補進內建 CSS：`object-contain`、`appearance-none`、`backdrop-blur-md / backdrop-blur-xs / backdrop-blur-2xs`、`ring-2`、`transition-all / transition-colors / transition-shadow`、`duration-300 / 500`、`cursor-grab / active:cursor-grabbing`、`border-emerald-100 / 200`、常用 `hover:` 變體（bg / text / border / shadow）、`focus:` 變體、`sm:grid-cols-2`、`sm:px-6`、`sm:text-*`、`sm:p-6`、`sm:h-24 / sm:w-24 / sm:w-9 / sm:h-9`、`sm:inline`、`xl:grid-cols-4`、`lg:flex-row / lg:items-center`、`line-clamp-2`、`::selection`（取代原 `selection:` class 寫法）。

### 2. `renderAll()` 全量重繪 → 依分頁局部重繪
- **問題**：原碼任何小操作（例如打字搜尋、點一個篩選）都會呼叫 `renderAll()`，一次重繪衣櫥、月曆、畫板、分析、趨勢共 5 個 view 的全部清單，單品多時明顯卡頓。
- **修正**：`renderAll()` 改為先 `computeMetrics()` 並更新頂部 KPI（全域資訊，保持行為等價），之後只依 `currentTab` 重繪目前可見的那一個 view。切換分頁時 `switchTab()` 會設定 `currentTab` 後再呼叫 `renderAll()`，因此切過去的 view 一定會被完整重繪，不會看到舊資料。

### 3. 搜尋輸入加上 debounce
- **問題**：`closet-search` 與 `log-search-input` 每輸入一個字就觸發一次 `renderAll()` 全頁重繪。
- **修正**：新增 `debounce(fn, wait)` 小工具，兩個搜尋框改為停止輸入約 **150ms** 後才重繪。篩選下拉與分類 pills 屬低頻操作，維持即時重繪。

### 4. 去背圖片解析度提高
- **問題**：`autoCutoutImageFile` 把上傳照片壓到最長邊 260px，去背邊緣鋸齒明顯、衣物圖案細節遺失。
- **修正**：上限提高為 **512px**。取捨說明：512px 的 PNG data URL 體積約為 260px 的數倍，多件去背單品會更快吃滿 localStorage 配額；因此同時落實了第 9 項「配額不足提醒」，讓使用者在存不進去時會收到提示並可改走備份匯出，而不是以為有存好。

---

## 二、安全

### 5. `escapeHtml` 未轉義單引號
- **問題**：原碼只轉義 `& < > "`，未處理單引號 `'`；單品名稱等使用者輸入被插入 HTML 屬性或字串情境時，有注入與版面破壞風險。
- **修正**：補上 `.replace(/'/g, '&#39;')`。

### 6. `customCutout` 的 `<img src>` 未驗證來源
- **問題**：`renderGarmentCutoutSVG` 直接把 `g.customCutout` 塞進 `<img src="...">`；若備份還原或舊資料帶入非圖片、非 data URL 的字串，可能載入外部資源或造成注入。
- **修正**：只有以 `data:image/` 開頭的字串才會被當成去背圖輸出，其餘一律忽略並改用內建 SVG 圖示。新增/編輯單品表單送出時、開啟編輯 modal 時也同步套用同一條規則。備份還原流程原本即有 `normalizeParsedState` 結構驗證，維持不變。

---

## 三、正確性

### 7. 日期全部寫死在 2026 年 10 月
- **問題**：原碼多處寫死日期——`selectedLogDate = '2026-10-25'`、月曆預設 `calendarYear = 2026; calendarMonth = 9`、「⚡今天穿+1」的 fallback `'2026-10-24'`、「清空資料」也重置到寫死日期、備份檔名用 UTC 日期切片。過了該月，App 會停在錯誤的「今天」，月曆也停在錯誤月份。
- **修正**：新增 `formatLocalDate(d)` / `todayLocalStr()`，以瀏覽器**本地時間**組出 `YYYY-MM-DD`（不使用 UTC 換算，避免台灣凌晨時日期差一天）。初始日期、月曆年月、快速穿搭的目標日期、清空資料後的日期、備份檔名，全部改用本地今天/目前年月。

### 8. 「⚡今天穿+1」會捏造隨機過去日期灌水穿著天數（嚴重 bug）
- **問題**：原碼在「當日紀錄已包含該單品」時，會用隨機天數產生一個過去日期、再新增一筆 log。結果是穿著天數被無中生有地 +1、月曆上冒出從沒記錄過的日期，CPW、使用率、冷宮分析全部被污染，且行為不可預期、無法重現。
- **修正**：改為——當日已包含該單品時，只跳出提示「今天已記錄過囉，不重複計算穿著天數」，**不新增任何紀錄**。當日尚無紀錄或未包含該單品時，維持原本的正常 +1 行為。

### 9. localStorage 寫入失敗被靜默吞掉
- **問題**：原碼 `saveState()` 對 `localStorage.setItem` 的例外（最常見為 QuotaExceededError，尤其去背圖變大後）以空 `catch` 吞掉，使用者以為已自動儲存，重新整理後資料消失。
- **修正**：寫入失敗時以 `showToast(..., 'warning')` 明確告知「本機儲存空間不足、最新變更可能未存入」，並請使用者到「備份/還原記憶」匯出備份檔；同一連續失敗只提醒一次（成功寫入後重置旗標），避免 toast 洗版。IndexedDB 與 `window.name` 兩條備援儲存維持原有行為。

### 10. 其他正確性小修
- `btn-canvas-random` 的 `pick()` 對空陣列加上長度檢查，避免分類完全沒有單品時出錯（原碼回傳 `undefined` 後靠 `filter(Boolean)` 勉強通過，語意不清）。
- 自由畫板拖曳補上 `pointercancel` 處理，中斷拖曳時也會儲存位置，避免狀態遺失。
- 編輯單品時保留原有的 `shape` 欄位，避免編輯後趨勢範例單品的圖形被重猜成不同形狀。

---

## 四、無障礙與 SEO

### 11. 缺少基本 meta 與 favicon
- **修正**：新增 `<meta name="description">`、`<meta name="theme-color" content="#1c1917">`，以及以 inline SVG data URI 製成的 🎀 emoji favicon（零外部請求）。`<title>` 維持原標題不變。

### 12. Modal 與分頁缺少 ARIA 語意
- **修正**：兩個 modal 加上 `role="dialog"`、`aria-modal="true"` 與 `aria-labelledby`；關閉按鈕加上 `aria-label`；上方導覽列加上 `role="tablist"`，各分頁按鈕加上 `role="tab"` 與 `aria-selected`，`switchTab()` 切換時同步更新 `aria-selected`。月曆的上一月/下一月按鈕、月曆日期格、移除已選單品按鈕補上 `aria-label`；toast 容器加上 `aria-live="polite"`；篩選用 `<select>` 補上 `aria-label`。

### 13. 未啟用 JavaScript 時完全無提示
- **修正**：在 `<body>` 開頭加上 `<noscript>` 提示區塊，說明本 App 需要 JavaScript 才能使用衣櫥、日記與本機儲存功能。

---

## 五、可維護性與清理

### 14. CSS 重複的 reset 與重複定義
- **問題**：原碼有兩段 `*, ::before, ::after { box-sizing: border-box; }`，且 `button { cursor: pointer; }` 重複宣告，分散在兩個 `<style>` 區塊，增加日後改版互相覆蓋的風險。
- **修正**：合併為單一 reset（保留 `margin: 0; padding: 0;` 的完整版本），刪除重複宣告；自訂樣式（衣櫥掛桿、去背棋盤格等）集中在第二個 `<style>` 區塊，內容與視覺效果不變。

### 15. JS 死碼
- **問題**：`comparisonChartInstance`、`doughnutChartInstance` 兩個變數宣告後從未使用（圖表實際以 innerHTML/SVG 直接繪製），容易誤導維護者以為有未完成的圖表函式庫整合。
- **修正**：移除這兩個未使用變數；`renderAnalyticsCharts` 同步移除未用到的參數。

### 16. 內嵌 `onclick` 事件
- **問題**：衣櫥空白狀態的「為第一件單品附魔」按鈕使用 inline `onclick="openGarmentModal(null)"`，與全站以 `addEventListener` / 事件委派為主的寫法不一致，也不利於日後套用 CSP。
- **修正**：改為 `id="btn-empty-add-garment"`，在全域 click 委派處理器中優先判斷該 id 後呼叫 `openGarmentModal(null)`。全站已無任何 inline `onclick`（grep 驗證為 0）。

### 17. 嚴格模式
- **修正**：主 `<script>` 開頭加上 `'use strict';`，讓未宣告變數等錯誤及早暴露。

---

## 未更動、刻意保留的部分

- 儲存鍵名 `styladay_wardrobe_iq_clean_v3` 與舊版 `styladay_closet_v3 / styladay_logs_v3` 的相容讀取邏輯保留，舊使用者資料可直接延用。
- LocalStorage + IndexedDB + `window.name` 三重儲存架構、備份代碼/JSON 匯出還原流程保留。
- 2026 趨勢公式的內容、來源連結與比對邏輯保留（趨勢文案屬內容資料，非程式缺陷）。
- 去背演算法本身（取左上角像素為背景色 + 近白去除）維持原邏輯，僅調整輸出尺寸；此演算法對複雜背景照片效果有限，屬已知限制，見 README。
- 未加入任何外部網路資源：無 Google Fonts、無 CDN、無外部圖片；字體沿用系統字體堆疊（`Inter` 若使用者裝置有安裝才會命中，否則自動落到 Noto Sans TC / PingFang TC / Microsoft JhengHei 等系統字體）。
