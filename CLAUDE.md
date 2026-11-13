# TMIC CRM — 專案說明（給 Claude Code）

明日國際 TMIC 自建的客戶管理系統，取代原本的 Notion。$0 架構：GitHub Pages ＋ Firebase ＋ Google Apps Script。
網址：https://jimmy198501.github.io/tmic-crm/

## 與使用者（Jimmy，管理者）溝通

- 一律用**繁體中文**回覆，直接、精簡。
- **一次只給一個明確的下一步**，不要同時丟一堆選項或指令。
- 有不清楚的地方**先問，不要自己猜**。推論要標註「這是推論」。
- 回答前先做事實檢查；沒測過的要老實說「只確認過語法，還沒實際測試」。
- 改完要說清楚：改了什麼、要上傳／貼到哪裡、要不要重新部署。
- Jimmy 不寫程式，步驟要寫到點哪個按鈕。

## 檔案

| 檔案 | 放在哪 | 說明 |
|---|---|---|
| `index.html` | 本專案（GitHub Pages） | 整個前端，單一檔案（HTML＋CSS＋`<script type="module">`） |
| `firestore.rules` | 本專案（備份用） | **實際生效要貼到 Firebase 主控台 → tmic-crm-eeac3 → Firestore → 規則 → 發布** |
| `Code.gs`、`appsscript.json` | **不放本專案** | Google Apps Script（帳號 tmicsper@gmail.com）。內含 LINE／Notion 權杖，本專案是公開的，**絕對不能 commit** |
| `quote-logo.png`、`quote-stamp.png` | 本專案 | 報價單用（報價單功能目前已隱藏） |

- 每次改 `index.html` 都要更新頁尾版本號 `版本 YYYY-MM-DDx`（同一天依序 a、b、c…），方便確認使用者有沒有載到新版（Ctrl+F5）。
- Firebase 專案 ID：`tmic-crm-eeac3`（Spark 免費方案）。注意：Jimmy 曾經把規則誤貼到 `camp-weihong` 專案，提醒他確認專案名稱。
- 另一個專案 `whc`（GitHub）與 CRM 無關，Jimmy 曾誤傳 index.html 到那裡。

## 角色與權限

角色：`admin`（管理者）、`manager`（業務主管，有 regions）、`sales`（業務）、`engineer`（工程師）。另有 `alsoEngineer`、`alsoSales` 旗標。
區域：台北、台中、台南（`regions` 陣列；舊資料可能是單一 `region`，用 `userRegions()` 取）。台中業務常同時管台南，新增行程時「台中＋台南」預設台中。

- 客戶：admin 全部；manager 自己區域＋自己負責；sales 自己負責；engineer 全部可讀。只有 admin 能改客戶資料。
- Demo（`demos` 集合，type: sales / engineer）：admin 全部；manager 看自己區域；sales 看自己的；engineer 看工程師 Demo。查詢時必須依權限分組查（`demoSets()`／`queryDemos()`），否則規則會拒絕整個查詢。
- 問題追蹤（issues）、安裝紀錄（installs）：工程師可改客戶連結（先標 `movingTo` 再刪舊的）。
- 行程（`schedules`）：所有登入者可讀；建立者／admin 可改刪；manager 可改（不能刪）自己區域的。私人行程在 `privateSchedules`（只有本人）。
- 公告（`announcements`）、人員、編號設定：只有 admin。

## 主要功能（index.html）

- 路由（hash）：`#/home`、`#/customers`、`#/customer/{id}`、`#/records`、`#/calendar`、`#/line`、`#/users`、`#/settings`、`#/import`。
- 首頁：公告欄＋跑馬燈、今天／明天／後天行程。
- 客戶頁：基本資料、Demo／採購／問題追蹤／安裝紀錄（每次進客戶頁都重抓紀錄）、LINE 詢問統計、上一位／下一位。
- 紀錄頁：五個分頁。篩選：Demo 分頁有區域／業務／工程師；採購只有搜尋；問題追蹤、安裝紀錄只有工程師（＝建立者）＋搜尋。搜尋也比對建立者。操作按鈕在每列最左邊。
- 行事曆：月／週（時間軸）／清單，各人偏好存在 `prefs/{email}`（calView、weekStart）。行程類型：Demo、裝機交貨、技術支持、上課、會議、休假、其他。可拖曳移動（會先確認），按住 Ctrl／Option 拖曳＝開啟複製表單。暫定行程不寄信、不同步 Google 日曆。
- 行程用 `onSnapshot` 即時同步（每月一個監聽，最多 6 個）。**不要改成定時重抓**：Spark 方案每天只有 5 萬次讀取，以前額度爆過（429）。
- LINE 頁：官方帳號 webhook 收到的訊息；可改名、連結客戶、編輯／不算詢問／刪除訊息、每天標「處理同事」（只列工程師）、刪除整天、刪除整筆對話。
- 地址：電腦／Android 開 Google 地圖；iPhone 跳選單選 Apple／Google。選 Apple 時先呼叫 Apps Script `geocode` 取座標再開 `maps.apple.com/?ll=`（Apple 對台灣地址常找錯路，例如三十四路變三十八路）。
- 已隱藏、程式還在的功能：報價單、維護到期追蹤＋每週提醒信。Jimmy 說之後可能再開，不要刪程式碼。

## Apps Script（Code.gs）

- `doPost` 動作：`upload`、`notify`、`gcalUpsert`、`gcalDelete`、`gcalList`、`geocode`；網址帶 `?k=` 暗碼的是 LINE webhook。每個請求用 Firebase idToken 驗證身分（`checkUser_`）。
- 每 15 分鐘觸發 `sendReminders`：行程提醒＋掃描 Gmail 匯入 IronCAD 原廠單號（Zoho 轉寄信，標籤 `crm-已匯入`）。
- 用 Firestore REST＋`ScriptApp.getOAuthToken()`（繞過規則）。
- **改了 doPost 會用到的東西，要「部署 → 管理部署作業 → 新版本」**；只改觸發排程用的函式只要儲存。
- 一次性搬家程式（Notion 序號檔、問題追蹤附件、清除「［Notion 附件：…］」文字）已經跑完，不用再執行。

## 修改後的檢查

- 抽出 module script 用 `node --check` 檢查語法：
  `sed -n '/<script type="module">/,/^<\/script>/p' index.html | sed '1d;$d' > /tmp/m.mjs && node --check /tmp/m.mjs`
- 注意：很多函式寫成一行，**行尾加 `//` 註解會把後面的 `}` 註解掉**，曾經因此壞過。
- 介面改動盡量用 Playwright（Chromium）＋假資料實際跑一次再交付。
- 已知坑：
  - `<dialog>` 內選檔案按取消會觸發 `cancel` 事件把對話框關掉——全域已攔截，別拿掉。
  - 有改過內容的編輯視窗按 Esc 會先確認。
  - 調整欄寬（`setColW`）時，有換行的欄位要保持 `pre-wrap`。
  - 共用篩選狀態（`recsUI`）只能套用在該分頁有顯示的篩選上，否則會整頁變空白。

## 待決定（等 Jimmy 通知）

- 報價單號編號規則、優惠價下限（報價單目前隱藏中）。
