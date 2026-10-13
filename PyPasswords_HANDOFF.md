# PyPasswords HTML 版交接文件

## 1. 專案概述

使用者上傳 Python 程式 `PyPasswords.py`（本機多設定檔密碼管理 CLI，895 行），要求改以 HTML 重寫，注重排版美感與顯示架構，之後以 Tauri 2 打包為 Android App。  

最終前端為單一檔案 `index.html`（原生 JavaScript，無任何外部相依）。瀏覽器開啟時資料存 `localStorage`；在 Tauri 內執行時存 AppData 檔案，兩種模式由 `window.__TAURI__` 是否存在自動切換。介面支援繁體中文與英文，設定集中在「設定」對話框。  

目前交付檔案（皆在 `/mnt/user-data/outputs`）：

| 檔案 | 用途 |
| --- | --- |
| `index.html` | 已含全部修改的完整前端，直接覆蓋 `~/PyPasswords/src/index.html`，不要再執行任何修補腳本 |
| `MainActivity.kt` | Android 狀態列內縮、禁止截圖（`FLAG_SECURE`）、返回鍵攔截（呼叫 JS 的 `window.__pwBack`），覆蓋 `gen/android` 內同名檔案 |
| `PyPasswords_HANDOFF.md` | 本文件 |

`patchPyPasswords.py` 與 `patchPyPasswords2.py` 已被 `index.html` 取代，只保留作歷史紀錄；`makeIcon.py` 仍可用於產生預設圖示。  

## 2. 使用者偏好（需遵守）

- 一律以繁體中文（台灣用語）回答，禁止表情符號；避免大陸用語，無台灣譯名時保留英文原詞。  
- 程式碼簡潔優先，函數與變數用 camelCase；數學式用 LaTeX。  
- 指令列區塊內不寫註解，需解釋時於區塊後以「行號：說明」條列。  
- Markdown：標題僅一個 `#`，次標題 `##`／`###`；段落行尾加兩個空格；標題、列表、表格前後留空白行；表格附 `| --- |` 分隔線；內文中的 `\n`、`\t`、反斜線以反引號包裹。  
- 事實性主張優先引用可信來源（APA 第七版）；不確定時主動標示；使用者前提有誤時直接指出。  
- 使用者另有要求時，以使用者最新指示為準。  
- 使用者的終端機為 zsh：指令中的 `*` 通配字元須加引號，角括號佔位符貼上前必須換成實際值，環境變數寫入 `~/.zshrc`。  
- 使用者提出條件式需求（例如「有效能優勢才做」）時，必須以實測數據判斷並把結論回報，而不是直接照做或直接略過。  
- 使用者習慣直接上傳檔案並要求「直接修改附檔」，此時應交付完整修改後的檔案，而不是要求使用者再執行修補指令。  
- 使用者的建置主機已改為 macOS（見第 9 節），不再使用 Ubuntu。MacOS 的 `sed -i` 必須寫成 `sed -i ''`，且不支援 GNU 的 `0,/.../` 位址，需改用 `perl -0pi`。  
- 使用者常直接貼上終端機輸出，回答前必須逐行核對輸出（例如曾貼出兩個路徑黏在一起的 `adb install` 指令、`strings.xml` 內字串帶引號），並指出實際發現的問題。  
- 使用者簽署 APK 的實際做法是 `java -jar ~/bin/uber-apk-signer.jar -a <未簽署 APK>`（內嵌 debug 金鑰），不是 `zipalign` 加 `apksigner`；之後的指令以此為準。  
- 使用者要求「更新 Handoff.md」時，交付完整更新後的 `PyPasswords_HANDOFF.md`，並用 `present_files` 提供。  
- 使用者偶爾把專案路徑打成 `~/PyPassword`；實際路徑是 `~/PyPasswords`（有 s），遇到時直接指出並以正確路徑為準。  

## 3. 原程式（PyPasswords.py）重點

選單功能：生成密碼、新增紀錄、設定主碼、查詢密碼、列出紀錄（不顯示密碼）、編輯、刪除（可多選）、匯出備份、匯入備份、匯入 CSV、切換設定檔。  

資料位置：`~/.pwvault/<profile>.bin`，設定檔名稱規則 `[\w\-\u4e00-\u9fff]+`。  

三種格式，皆由檔頭自我描述。使用者已決定介面一律以 PWV1 至 PWV3 稱呼，不再標示 Fernet 為舊版；程式內部的格式識別值仍為 `"FERNET"`、`"PWV2"`、`"PWV3"`，不可更動：

| 名稱 | 檔案位元組配置 | 金鑰衍生與加密 |
| --- | --- | --- |
| PWV1（Fernet 格式） | `SALT(16)` + Fernet token（ASCII urlsafe base64，開頭為 `gAAAA`），無 MAGIC | PBKDF2-HMAC-SHA256，600000 次，32 位元組；Fernet 金鑰為其 urlsafe base64 |
| PWV2 | `"PWV2"` + `SALT(16)` + `NONCE(12)` + 密文含 tag | PBKDF2-HMAC-SHA256，600000 次，AES-256-GCM |
| PWV3（預設） | `"PWV3"` + `0x01`（KDF ID）+ `SALT(16)` + `NONCE(12)` + 密文含 tag | Argon2id（memory 19456 KiB、iterations 2、lanes 1、length 32、version 0x13），AES-256-GCM |

明文為 JSON：`{"records":[...]}`。每筆紀錄欄位：`id`（`token_hex(4)`）、`accountType`、`accountName`、`password`、`linkedAccount`、`name`、`notes`（字串陣列）。  

匯出備份：複製 `.bin` 並另存 `<檔名>.sha256`（內容為 SHA-256 十六進位字串）。  

CSV 匯入欄位：`帳戶類型,帳號名稱,密碼,連結帳戶,姓名,備註`；含「備註」的欄名皆視為備註欄，單欄內可用 `;` 或 `；` 分隔多筆。  

密碼生成：長度不得低於 12；字元集為英數字加 `!@#$%^&*()-_=+[]{}`；必須同時含小寫、大寫、數字、符號。  

空資料庫的最小長度：PWV2、PWV3 為 28 位元組（12 位元組 nonce 加 16 位元組 tag）。原 HTML 的 `unseal` 曾要求至少 44，導致新建的空資料庫（42 位元組）無法解鎖或新增紀錄，已修正為 28。  

## 4. HTML 版現況

### 4.1 功能

- 設定檔：建立（可選 PWV3、PWV2、PWV1；預設格式取自設定；主碼二次確認；離線弱密碼檢查可在設定中關閉）、解鎖、匯入、刪除。  
- 紀錄：新增、編輯、單筆刪除、多選刪除、依帳號名稱搜尋；單筆與多選刪除皆依序要求確認、PIN（已設定時）、主碼；備註在編輯時為 textarea，每行一筆。  
- 「生成密碼」（原名「生成器」，使用者要求改名，對話框標題一併改名）：長度 12 至 64，預設長度取自設定；強度評估（弱、中、強）可在設定中關閉。  
- 備份：匯出 `.bin` 與 `.sha256`；匯出為原樣複製，沿用該設定檔目前的格式。Tauri 內走系統儲存對話框，依序跳出兩次（`.bin` 與 `.sha256`），寫入後讀回驗證長度；匯入沿用隱藏的檔案選擇器（使用者實測正常，刻意不改成對話框外掛）。匯入前彈窗顯示檔名、大小、檔頭十六進位、偵測格式與完整 SHA-256，無法辨識格式時停用匯入鈕。  
- 管理資料（皆在「更多」選單，需要主碼的操作都會再次驗證主碼）：  
  - 轉換加密格式：轉換本機設定檔；以新 salt 重新加密，寫入前先解密比對，舊檔留成 `.bin.bak`（維持轉換前格式）。  
  - 變更主碼：輸入目前主碼、新主碼兩次；沿用目前格式，換新 salt 重新加密並先驗證；舊主碼錯誤時對話框重新開啟；新主碼不可與舊主碼相同；弱主碼警告依設定。  
  - 多選刪除：進入後清單每列出現勾選框，上方出現操作列（已選數量、全選或取消全選目前搜尋結果、刪除所選、結束）；刪除前先確認再驗證主碼，一次寫入。  
  - 刪除資料庫：「更多」選單與鎖定畫面的「刪除此設定檔」皆提供，須輸入完整設定檔名稱；若已設定 PIN 再輸入 PIN。  
  - 匯入 CSV（規則同原程式）。  
- 設定（標題列的「設定」鈕，鎖定畫面也能開啟）：見 4.2。  
- 儲存位置：Tauri 內為 `<AppData>/vault/<設定檔>.bin`（Android 的 AppData 實測就是 App 私有資料目錄的根，`run-as` 列出 `./vault/...`）；瀏覽器內為 `localStorage`，鍵為 `pwv:<profile>`，值為檔案位元組的 base64。  
- 所有 `confirm` 與 `prompt` 改為自製 `<dialog id="dlgAsk">`（`askYesNo`、`askText`、`askPin`；PIN 輸入為遮罩加數字鍵盤）。  
- 輸入框 Enter：解鎖主碼、建立表單（名稱、主碼、再次輸入依序前進，最後送出）、匯入名稱、再次輸入主碼、轉換主碼、變更主碼三欄、`dlgAsk` 輸入框皆可按 Enter；編輯表單內按 Enter 跳到下一欄（備註為多行）。判斷函式為 `isEnter`（`key` 與 `keyCode` 雙重判斷，忽略輸入法組字），主要輸入框加了 `enterkeyhint`。使用者已實機確認 Enter 正常。  
- 禁止選取文字：CSS 對全頁設 `user-select:none`，只有 `input`、`textarea`、`select` 與匯入對話框內的 SHA-256 欄位可選取；另攔截 `selectstart`、`contextmenu`、`dragstart`。顯示出來的密碼也無法反白，只能用複製鈕。這是固定行為，不是設定項。  
- 語言：繁體中文、English、跟隨系統（依 `navigator.language`，`zh` 開頭為中文，其餘為英文）。切換後立即套用，不需重開。CSV 匯入的欄位標題仍須使用中文欄名（`帳戶類型,帳號名稱,密碼,連結帳戶,姓名,備註`），英文介面的錯誤訊息會註明這點。  

### 4.2 設定項目

設定存於 `<AppData>/settings.json`（瀏覽器為 `localStorage` 的 `pwv-settings`）。讀取時逐項驗證型別與範圍，不合法的值退回預設。設定不加密；唯一敏感的欄位是 PIN，只存雜湊。  

| 設定 | 預設 | 說明 |
| --- | --- | --- |
| 語言 | 跟隨系統 | 繁體中文、English、跟隨系統 |
| 預設加密格式 | PWV3 | 建立設定檔表單的預設值，也是轉換對話框的建議目標 |
| 解鎖時自動轉換為預設格式 | 關 | 解鎖成功且格式不同時，立即以新 salt 重新加密並留 `.bak`；方向不限，所以預設格式選 PWV1 或 PWV2 會把 PWV3 降級；轉換失敗只顯示錯誤，不影響解鎖 |
| 啟用密碼強度檢查 | 開 | 同時控制主碼的弱密碼警告（建立、變更）與紀錄的強度條 |
| 生成密碼預設長度 | 20 | 12 至 64；影響生成密碼對話框與新增紀錄時的自動密碼 |
| 禁止截圖與螢幕錄影 | 關 | 僅 Android 有效；見 4.3 |
| 閒置自動鎖定 | 5 分鐘 | 1、3、5、10、30 分鐘 |
| 切到背景後鎖定 | 1 分鐘 | 30 秒、1、5、15 分鐘；刻意不提供「立即」，因為匯出入時的系統檔案對話框會讓 App 進入背景 |
| 查看密碼顯示時間 | 30 秒 | 15、30、60 秒 |
| 複製密碼後提醒我清除剪貼簿 | 開 | 關閉時複製只顯示「已複製」 |
| 刪除用 PIN | 未設定 | 適用於刪除資料庫、單筆刪除、多選刪除；4 至 8 位數字，以 PBKDF2-SHA256（100000 次，隨機 16 位元組 salt）雜湊後存入設定檔。變更、移除、還原預設都必須先輸入目前 PIN。PIN 不參與加密，只能防誤刪與旁人操作，無法保護資料；能取得設定檔或 App 資料的人可以直接刪檔 |

另有「還原預設值」按鈕（保留語言，有 PIN 時須先輸入 PIN，並提示會一併移除 PIN）。  

刻意沒有做的項目：主題（淺色、深色）手動切換。原因是狀態列後方的底色來自 Android 資源 `values` 與 `values-night`，跟隨系統深淺色，App 內手動切換會讓狀態列與頂端列顏色不一致。  

### 4.3 禁止截圖的實作（跨 JS 與 Kotlin）

JS 無法直接設定 Android 視窗旗標，也沒有自訂外掛，所以採「旗標檔」做法：設定改變時，JS 把 `1` 或 `0` 寫入 `<AppData>/secure.flag`（啟動時也會同步一次）；`MainActivity.kt` 在 `onCreate` 讀取該檔並對 `window` 加上或移除 `FLAG_SECURE`，同時以 `FileObserver` 監看檔案變動，所以切換後立即生效，不必重開 App。  

假設與風險：這依賴「`appDataDir` 就是 `dataDir`」。實測 `run-as` 列出 `./vault/...` 支持這點，Kotlin 另外也同時監看 `filesDir` 作為保險。若實機切換後截圖仍可成功，先用 `adb shell run-as com.havelight.pypasswords ls -la` 確認 `secure.flag` 是否出現在資料目錄根。`FLAG_SECURE` 也會讓最近工作畫面的縮圖變黑，這是預期行為。  

### 4.4 安全設計（使用者明確要求）

- 解鎖後記憶體只保留清單中繼資料（`id`、`accountType`、`accountName`、`name`）。  
- 查看密碼、編輯、新增、刪除、多選刪除、匯入 CSV、變更主碼、轉換格式，都要再輸入一次主碼。  
- 查看後自動隱藏並顯示倒數（時間見設定）；密碼預設遮罩，按「顯示」才露出。  
- 編輯期間只保留不可匯出的 `CryptoKey`，儲存時以該金鑰重新解密儲存區最新資料後合併寫入；編輯表單閒置 120000 ms 自動取消。  
- 主碼輸入後立即轉為 `Uint8Array` 並清空輸入框，用畢以 `fill(0)` 歸零；Argon2id 的輸入緩衝與 19 MiB 工作記憶體運算後清零（Worker 內的副本也會歸零，Worker 用完即 `terminate()`）。  
- `pagehide` 立即鎖定；進入背景時立即清除已顯示密碼與編輯表單，背景超過設定時間則整體鎖定；閒置超過設定時間自動鎖定。  
- 剪貼簿：不嘗試自動清除（使用者決定），只提醒；是否提醒可在設定中關閉。  
- 生成密碼對話框關閉時清除顯示的密碼。  
- 解鎖失敗後主碼欄會被清空，使用者須重新輸入。  
- 寫入流程：先寫 `.bin.tmp`，讀回比對，現有檔通過格式檢查才複製為 `.bin.bak`，最後改名取代；啟動時會清掉殘留的 `.bin.tmp`。  

### 4.5 程式結構（單一檔案，由上而下）

- `<style>`：設計 token 位於 `:root`，含深色模式與 `max-width:480px`、`820px` 兩個響應式斷點。頂端列上緣加了 `env(safe-area-inset-top)`，`body` 下緣加了 `env(safe-area-inset-bottom)`，作為內縮備援。寬度 480px 以下，頂端列改為兩列（品牌與設定檔標籤一列，三個按鈕平分一列），因為英文標籤較長，單列會擠壓品牌名稱。  
- HTML：`#gate`（解鎖與建立分頁）、`#vault`（左 `.list` 含 `#mbar` 多選操作列與清單，右 `#dt` 詳情）、`dialog#dlg`（生成密碼）、`dlgImp`、`dlgAuth`、`dlgAsk`、`dlgConv`、`dlgPw`（變更主碼）、`dlgSet`（設定，內容由 `renderSet()` 以 JS 產生）、`#busy`、`#toast`、隱藏的 `#fImp`、`#fCsv`。  
- `/*A2*/ ... /*A2END*/`：自行實作的 BLAKE2b（BigInt）與 Argon2id（32 位元對運算，單 lane）。  
- `argon2Async`：在 A2 之後。以 `document.currentScript.textContent` 取得頁面自己的腳本文字，切出 A2 區段，包成 Blob Worker 執行 `argon2id`，用完即終止；無法建立 Worker 時退回主執行緒。因此 `/*A2*/` 與 `/*A2END*/` 這兩個標記字串必須各只出現一次（程式內以 `"/*A2"+"*/"` 拼接避免自我比對），且 HTML 不可被壓縮或改寫腳本內容。  
- `/*CORE*/ ... /*COREEND*/`：`hdr`、`derive`、`unseal`、`seal`、`openVault`、base64 工具；`derive` 在有 `argon2Async` 時呼叫它，否則直接呼叫 `argon2id`，所以仍可在 Node 中獨立測試。  
- 設定與多語系：`DEF`、`cfg`、`cfgLoad`、`cfgSave`、`cfgFlag`；`EN` 字典（以中文原句為鍵、英文為值）、`t(key, vars)`（`{name}` 為插值）、`fmtName`、`trDom`（走訪靜態 HTML 的文字節點與 `placeholder`、`aria-label`，以 `WeakMap` 記住中文原文，切換語言時可還原）、`applyLang`。標記 `data-nt` 的元素（清單、詳情、設定檔選單、提示、對話框訊息等由 JS 填入的區域）不經 `trDom`，其內容在產生時就呼叫 `t()`。  
- 新增任何使用者看得到的字串時，必須同時加進 `EN`；`toast()`、`work()` 的訊息與 `Error.message` 都會經過 `t()`。  
- 儲存層：`tauriApi`、`vaultCache`、`storeInit`、`storeGet`、`storePut`、`storeDel`、`saveFile`、`askDlg`。`storeGet` 與 `rawVault` 為同步函式，依賴啟動時載入的 `vaultCache`，所以不可改成非同步。  
- `askDlg` 與 `ask` 的 `onclose` 必須寫成 `if(!d.open)`：連續兩次使用同一個對話框時，前一次 `close()` 延後送出的 `close` 事件會落到下一次的處理函式，若不檢查 `d.open` 會把第二個提示誤判為取消（見第 6 節）。  
- 其餘為介面與事件：`work`、`authOpen`、`commit`、`renderList`、`renderDetail`、`renderEdit`、`lock`、`importBackup`、`delDb`、`pinGate`、`parseCsv`、`window.__pwBack`（Android 返回鍵，見 4.7），以及檔尾的啟動區塊。  
- 啟動區塊：註冊 `error` 與 `unhandledrejection` 的 toast；Tauri 內先顯示「載入設定檔中…」並停用解鎖鈕，設定與設定檔並行載入完成後才套用語言、同步旗標檔、呼叫 `refreshGate`，並在主控台輸出 `[PW] storeInit 完成：N 個設定檔，X ms`，可用 `adb logcat` 過濾 `[PW]` 查看耗時。  

### 4.6 視覺設計決策

- 配色：淺色 `--bg:#e8ecef`、`--ac:#234e70`（鋼藍）、`--br:#b4852c`（黃銅，單一強調）；深色模式另有對應值。刻意避開「奶油底配陶土色」的常見預設。  
- 字型：僅系統字型堆疊（標題 serif、內文 `"Noto Sans TC"` 等、密碼 mono）。  
- 辨識度元素：密碼顯示條依字母、數字、符號分色。  
- 頂端列：不含圖標，品牌名不換行，設定檔名標籤過長以刪節號截斷，按鈕為「生成密碼」、「設定」與「鎖定」。  

### 4.7 Android 返回鍵（跨 JS 與 Kotlin）

JS 端 `window.__pwBack()` 回傳 `true` 表示已處理，回傳 `false` 表示沒有上一層。Kotlin 端 `MainActivity.kt` 以 `OnBackPressedCallback` 攔截返回鍵，用 `evaluateJavascript` 呼叫 `window.__pwBack`，結果不是 `"true"` 就呼叫 `finish()` 退出。`WebView` 以走訪 `android.R.id.content` 的檢視樹取得，不依賴 `WryActivity` 的內部欄位。  

| 目前畫面 | 按返回鍵 |
| --- | --- |
| 任一對話框（確認、PIN、主碼、設定等） | 關閉最上層對話框（等同按取消鈕） |
| 運算中（`#busy` 顯示） | 不處理 |
| 編輯或新增表單 | 取消編輯 |
| 多選模式 | 結束多選 |
| 已開啟某筆詳情 | 回到清單 |
| 搜尋框有字 | 清空搜尋 |
| 已解鎖的清單最上層 | `lock()` 後退出 App |
| 建立設定檔分頁（已有設定檔時） | 回到解鎖分頁 |
| 解鎖頁 | 退出 App |

- 對話框的檢查順序（由上而下）：`#dlgAsk`、`#dlgAuth`、`#dlgConv`、`#dlgPw`、`#dlgImp`、`#dlg`、`#dlgSet`；各自以其取消鈕的 `click()` 關閉，所以走原本的取消流程。  
- `finish()` 只結束 Activity，不保證結束行程；重開時 WebView 重建，狀態為鎖定。  
- `WryActivity.kt` 仍覆寫已棄用的 `onBackPressed`，建置只產生警告，不影響編譯。  
- 使用者回報本輪測試完成；未逐項描述返回鍵每個畫面的結果。  

## 5. 驗證狀態

已驗證（Node 搭配 Python `cryptography`，先前階段）：

- Argon2id 輸入密碼 `pässword`、salt `0123456789abcdef` 時，兩者輸出皆為 `b6683c913ce624b920b9cda438621dde7162e9a9af44a5393eb49e6095ba4119`；BLAKE2b 與 `hashlib.blake2b` 一致。Node 中 Argon2id 約 1558 ms。  
- 由 Python 產生的 PWV1、PWV2、PWV3 檔案皆可由 JS 解密；錯誤主碼回報「主碼錯誤，或資料庫已損毀」。  
- JS 寫出的 PWV1 與 PWV3 檔案，Python 皆可解密。  

已驗證（容器內無頭 Chromium，搭配記憶體內的假 Tauri API，對最終 `index.html` 執行共 36 項檢查，全部通過，無頁面錯誤）：

- 預設語言依瀏覽器語系為英文；切到中文再切回，標題列與設定標題正確；英文畫面（閘門、設定、清單、多選、詳情、編輯、查看密碼、轉換、變更主碼、生成密碼對話框）無任何殘留中文。  
- 所有含中文的字串都有英文翻譯（以腳本比對 `EN` 字典與 HTML、JS 的字串，唯一例外是語言名稱「繁體中文」、主控台記錄與 CSV 欄名提示）。  
- 設定：預設格式連動建立表單；勾選禁止截圖後 `secure.flag` 內容為 `1`；設定寫入 `settings.json` 並在重新載入後保留；PIN 以雜湊儲存，檔案內不含明文。  
- 多選刪除：勾選、計數、確認、驗證主碼後一次刪除，結束多選。  
- 變更主碼：舊主碼錯誤時顯示錯誤並重新開啟對話框；正確時成功；舊主碼不能再解鎖，新主碼可以，紀錄完整。  
- Argon2id：建立 PWV3 設定檔時確實建立了 Worker；Worker 與主執行緒輸出的金鑰位元組相同。  
- 自動轉換：預設格式 PWV2 且開啟自動轉換時，解鎖 PWV3 設定檔後檔頭變為 PWV2，`.bin.bak` 仍為 PWV3。  
- PIN：4 至 8 位數字驗證、輸入遮罩；PIN 錯誤時資料庫保留；PIN 正確時刪除。  
- 文字選取：`body` 為 `user-select:none`，輸入框為 `text`。  
- 剪貼簿提醒關閉後，複製只顯示「已複製」。  

Web Worker 效能實測（容器內 Chromium，同一組輸入，兩次）：主執行緒約 1.7 秒、期間畫面完全凍結約 1.6 秒（最大影格間隔 1670 ms）；Worker 約 1.7 秒、最大影格間隔 17 至 19 ms，輸出完全相同。結論：總運算時間沒有變快（單 lane 的 JS 實作，Worker 不會加速），優勢是運算期間畫面與觸控保持回應、運算中的轉圈動畫不會停格。手機上運算更慢，凍結時間會等比放大。使用者的條件是「有效能優勢才做」，以此為據判斷採用。  

已驗證（使用者在 Android 實機，debug APK，arm64，三星裝置，macOS 建置）：

- 第一輪（套用第一支修補腳本後）：解鎖後切到主畫面再回來按鈕恢復運作；建立、新增、強制結束後重開紀錄仍在；`run-as` 列出 `./vault/測試.bin`、`測試.bin.bak`、`back1652.bin`；轉換格式、匯出（兩次儲存對話框）、匯入皆正常。  
- 第二輪（套用 Enter、剪貼簿、命名、`MainActivity.kt` 與狀態列資源後）：Enter 全部正常；重開不再卡頓（但原因未確定，見第 6 節）；狀態列不再重疊。  
- 第三輪（設定、語言、Worker、變更主碼、多選刪除、自動轉換、PIN、禁止截圖、禁止選取等）：使用者回報第 9.5 節清單測試結果皆為 OK；未逐項附上輸出，也未說明 `secure.flag` 路徑與最近工作畫面縮圖的細節。  
- 第四輪（單筆刪除 PIN、返回鍵、自訂圖示、正式版 APK）：建置輸出顯示 `Finished 1 APK`，以 uber-apk-signer 簽署後 `adb install` 回報 `Success`，`MainActivity.kt` 含返回鍵已成功編譯；使用者回報「測試完成」，未逐項描述。改名（`PyPasswords`）的指令已提供，使用者貼出的檢查輸出顯示改名前 `productName`、視窗標題、`app_name`、`main_activity_title` 皆為小寫 `pypasswords`（`strings.xml` 內字串還帶有一對雙引號），改名後的結果尚未回報。  

未驗證：

- 禁止截圖：最近工作畫面縮圖是否變黑、`secure.flag` 位置，使用者只回報整體 OK，未逐項確認。  
- 重開偶發卡頓的真正原因：目前沒有再重現，但未確認是並行讀檔、debug 版啟動慢，還是巧合。  
- JS 寫出的 PWV2 由 Python 解密（僅驗證讀取）。  
- 實機匯出的 `.bin` 與 `.sha256` 以 `sha256sum` 比對，以及以原 Python 版解密。  
- 對話框選取的路徑是否自動納入 `fs` 範圍、Android SAF 寫入是否截斷既有檔、`defaultPath` 是否生效。  
- 開啟系統檔案對話框時，App 進入背景是否會觸發 `pagehide` 而立即鎖定（匯出入實測正常，原因未確認）。  
- Android `allowBackup` 預設值。  
- 以自有金鑰（`keytool` 產生的 `.jks`）簽署的正式版流程；目前只做過 debug 金鑰簽署。  
- macOS `.dmg` 的建置結果與桌面版（`appDataDir`、視窗尺寸、匯出入對話框）行為；已提供指令，使用者尚未回報。  
- Git 版本控制：步驟與 `.gitignore` 已提供（見第 10 節），使用者尚未回報執行結果。  

## 6. 歷次使用者回報的問題與原因

- 主碼欄失敗後無法倒退刪字：`onkeydown` 處理函式對非 Enter 鍵回傳 `false`，導致 `preventDefault`。已改用明確的 `if`。  
- 手機頂端列換行：已以 `white-space:nowrap`、`min-width:0`、刪節號與窄螢幕縮小字級處理。  
- 匯入失敗：移除檔案選擇器的 `accept` 副檔名過濾，並新增 PWV1（Fernet）支援。  
- 解鎖狀態切到主畫面再回 App，所有按鈕失效：`lock()` 內引用不存在的 `#dlgGen`，`d.open` 對 `null` 拋出 `TypeError`。已改為 `#dlg` 與 `d?.open`，實機驗證通過。  
- 空資料庫無法解鎖：見第 3 節，`unseal` 的最小長度由 44 改為 28。  
- 部分輸入框按 Enter 無效：原因是只有主碼解鎖、再次輸入主碼等少數輸入框綁了 Enter，其餘（建立表單三欄、匯入名稱、編輯表單各欄）完全沒有處理。已補上，並統一用 `isEnter`。是否另有鍵盤相容問題需實機確認。  
- 重開時有時特別久或當掉（機率性）：原因未確定。啟動時唯一的工作是 `storeInit`，原本逐檔依序讀取，每檔約 4 次 Rust 端 IPC。已改為單次查路徑加並行讀取，並讓介面先顯示載入中。仍可能的因素包括 debug 版 Rust 端未最佳化（可建置正式版並以 debug 金鑰簽署對照），以及 Argon2id 在主執行緒運算造成的停頓。若仍發生，需要在重現當下收集 `adb logcat -v time` 並過濾 `pypasswords|AndroidRuntime|FATAL|ANR|chromium|tauri`。  
- 剪貼簿：部分裝置無法真正清除，改為提醒（見第 4.2 節）。  
- 標題列在 Android 上被狀態列重疊：Tauri 範本的 `MainActivity` 啟用 edge-to-edge，內容延伸到狀態列底下。已提供 `MainActivity.kt`，以 `ViewCompat.setOnApplyWindowInsetsListener` 對 `android.R.id.content` 設定系統列、瀏覽器切角與鍵盤的內縮並回傳 `CONSUMED`；主題 `windowBackground` 以資源 `pw_window_bg` 配合 `--sf`，淺色 `#FBFCFD`、深色 `#151D25`。尚待實機驗證。  

- 刪除資料庫輸入完名稱後，PIN 提示被立刻當成取消（本輪測試發現，使用者尚未遇到，因為 PIN 是本輪新增）：`askDlg` 的 `d.close()` 會延後送出 `close` 事件，這個事件落到下一次 `askDlg` 剛設定好的 `onclose`，把第二個提示解析為取消，對話框卻仍開著。同一個潛在問題也存在於原本的「同名設定檔確認後接弱主碼確認」。已將 `onclose` 改為 `if(!d.open)` 才解析，`ask`（再次輸入主碼）同步修正。  
- 英文標籤使 Android 窄螢幕的頂端列擠壓品牌名稱（容器內 390px 與 360px 截圖發現）：480px 以下改為兩列。  
- 使用者回報並已確認解決：部分輸入框 Enter 無效、標題列被狀態列遮住、重開偶發卡頓（原因未確定）。  

## 7. 已知限制與可能的後續工作

- JavaScript 無法保證覆寫字串記憶體，只能縮短存留時間並丟棄參照。  
- 每次驗證都重新運算 KDF；Argon2id 已移到 Worker，但單 lane 的 JS 實作本身不會更快。若日後要真正加速，需要 WebAssembly 版 Argon2（容器無網路，未嘗試）。  
- 瀏覽器版資料綁定瀏覽器與網址來源，更換瀏覽器或清除網站資料即遺失，須靠匯出備份。Tauri 版存 AppData，解除安裝 App 同樣會清空，所以更換簽章（debug 換正式版）前必須先匯出備份。  
- 刪除 PIN 只防誤刪，不是存取控制；設定檔不加密。  
- 設定檔內的 PIN 雜湊與設定可被能讀取 App 資料的人修改；這與原本「裝置本身需可信」的威脅模型一致。  
- 備註逐筆編輯尚未實作（現為 textarea）。搜尋僅比對 `accountName`（與原程式一致）。  
- 尚未決定是否採用的建議：  
  1. `AndroidManifest.xml` 設定 `android:allowBackup="false"`，避免系統把 vault 備份到雲端；檢查指令為 `grep -rn allowBackup src-tauri/gen/android/app/src/main/AndroidManifest.xml`。  
  2. （已完成）返回鍵行為，見 4.7。  
  3. 提醒訊息目前是短暫的 toast，若使用者覺得容易漏看，可改為需按確認的對話框。  
  4. （已完成）單筆與多選刪除皆要求 PIN，見 4.1 與 4.2。  
  5. 建立或匯入設定檔後記住上次使用的設定檔。  

## 8. 開發與測試方式

環境：容器 `bash_tool` 網路停用，不可安裝套件。具備 `node`、Python `cryptography` 與 Playwright（Chromium 可用）。  

語法檢查：

```
cd /mnt/user-data/outputs
python3 -c "import re;s=open('index.html',encoding='utf-8').read();open('/home/claude/chk.js','w').write(re.search(r'<script>(.*)</script>',s,re.S).group(1))"
node --check /home/claude/chk.js
```

1：切換到輸出目錄。  
2：抽出 `<script>` 內容到暫存檔。  
3：以 Node 檢查語法。  

介面測試方式（本輪共 36 項，腳本不在交付檔案內，需要時重寫）：以 Playwright 的 `add_init_script` 在載入前注入假的 `window.__TAURI__`（含 `path`、`fs`、`dialog`、`clipboardManager`，檔案系統用記憶體，需跨重載保留時存入 `sessionStorage`），再以 `file://` 開啟 `index.html`，逐項操作並檢查 `pageerror`。測試時建議使用 PWV2（PBKDF2 由瀏覽器原生執行），PWV3 的 JS 版 Argon2id 每次約 1.7 秒。注入的假 `Worker` 包裝可用來計數 Worker 是否被建立。測試手機版面時，把視窗設為 360 至 390 寬，並注意手機版面下開啟詳情後清單會被隱藏，需先按「返回清單」才能點選單。翻譯覆蓋檢查：比對 `EN` 字典的鍵與 HTML 靜態文字、JS 中所有含中文的雙引號字串。  

互通測試方式：取出 `/*A2*/` 至 `/*A2END*/` 與 `/*CORE*/` 至 `/*COREEND*/` 兩段合併為一個 JS 檔，開頭加上 `globalThis.crypto??=require('crypto').webcrypto;`，再用 Python 產生測試檔與 Node 互相讀寫比對。  

交付檔案時須呼叫 `present_files`，不使用 `Artifact` 發布（密碼管理程式以本機檔案形式提供）。  

容器無法建置 Tauri 與 Android，第 9 節的指令只在使用者機器上執行過的部分才算驗證。  

## 9. Tauri 2 跨平台打包

### 9.1 目標與現況

目標：以 Tauri 2 將 `index.html` 打包為 Android App，並已要求 macOS `.dmg` 的建置指令（尚未回報結果）。  

建置主機：macOS（Apple Silicon，zsh），專案目錄 `~/PyPasswords`；舊的 Ubuntu 專案已改名為 `~/PyPasswords.old`，並從 Linux 機器取回其 `src/index.html`。這次是全新建立專案，沒有複製舊專案的 `gen/android`、`node_modules` 或 `target`。  

目前狀態：

- 專案已建立，`identifier` 為 `com.havelight.pypasswords`（定案後不可更動）。  
- 已安裝 `fs`、`dialog`、`clipboard-manager` 三個外掛，`capabilities/default.json` 已補齊權限，`app.withGlobalTauri` 為 `true`。  
- `tauri android init` 與 `tauri icon` 已執行；圖示以 `makeIcon.py`（在 `/tmp/iconEnv` 虛擬環境內，因 Homebrew 的 Python 不允許 `pip install --user`）產生，`ic_launcher_background.xml` 背景為 `#234E70`。  
- 手機上目前安裝的版本：含全部功能的 `index.html`、含返回鍵的 `MainActivity.kt`、自訂圖示，以及以 uber-apk-signer（debug 金鑰）簽署的 release APK `app-universal-release-aligned-debugSigned.apk`；使用者回報測試完成。  
- 兩個交付檔案已放入專案並建置安裝（見第 5 節第三輪與第四輪）。  
- Kotlin 編譯已通過：輸出只有 Tauri 與 `WryActivity.kt`、`RustWebView.kt` 的棄用警告，另有 AGP 對 `gradle.properties` 選項的棄用警告，皆可忽略。  
- 輸出路徑：debug 版為 `~/PyPasswords/src-tauri/gen/android/app/build/outputs/apk/universal/debug/app-universal-debug.apk`；release 版為 `.../apk/universal/release/app-universal-release-unsigned.apk`，經 uber-apk-signer 後產生 `.../release/app-universal-release-aligned-debugSigned.apk`。  

實際建置環境（取自使用者貼出的輸出）：

| 項目 | 版本或路徑 |
| --- | --- |
| JDK | Temurin 17（`/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home`） |
| Android SDK | `/Users/havelight/Android/Sdk`（`ANDROID_HOME`） |
| NDK | `28.2.13676358` |
| Node.js | 26.11.1（Homebrew） |
| jq | 1.8.2（Homebrew） |
| create-tauri-app | 4.7.5 |
| Pillow | 12.3.0（`/tmp/iconEnv` 虛擬環境，Python 3.15） |
| Rust 目標 | `aarch64-linux-android`（只需此一個） |
| Tauri 相關 crate（取自建置輸出） | `tauri` 2.12.3、`wry` 0.57.0、`tauri-build` 2.7.2 |
| APK 簽署工具 | uber-apk-signer v1.3.0（`~/bin/uber-apk-signer.jar`，內建 zipalign 與 debug 金鑰） |

### 9.2 主要決策

| 項目 | 決策 | 原因 |
| --- | --- | --- |
| 前端 | 單檔 HTML 作為 `src/index.html`，以 `window.__TAURI__` 呼叫外掛 | 不引入打包器；瀏覽器開啟時仍可退回原行為 |
| 儲存 | 改存 `<AppData>/vault/<設定檔>.bin`，位元組同原 Python 版格式 | localStorage 會因 `identifier`、`useHttpsScheme`、開發與正式版來源不同而換到另一份資料 |
| 寫入 | 先寫 `.bin.tmp`，讀回比對，現有檔通過格式檢查才複製為 `.bin.bak`，最後改名取代 | 避免寫入中斷或損毀檔覆蓋好備份 |
| 匯出入 | 匯出走 `dialog.save` 加 `fs.writeFile` 並讀回驗證；匯入維持隱藏檔案選擇器 | 匯入實機正常，所以不改 |
| 剪貼簿 | `clipboard-manager` 外掛；不嘗試清除，只提醒使用者自行清理 | 行動裝置不保證能清除 |
| 對話框 | `confirm` 與 `prompt` 全部改自製 `<dialog>` | 各平台行為未確認，也不依賴對話框外掛的 `confirm` 權限 |
| 狀態列 | `viewport-fit=cover`；`MainActivity.kt` 對內容區設內縮並回傳 `CONSUMED`；主題 `windowBackground` 配合 `--sf`；CSS `env(safe-area-inset-*)` 備援 | Android WebView 低於 140 版時 `env()` 值不可靠 |
| 設定 | 集中於「設定」對話框，存 `<AppData>/settings.json`（瀏覽器為 `localStorage`），不加密，PIN 只存雜湊 | 使用者要求將偏好集中；設定需在鎖定前就能讀取（語言、預設格式），因此不能放進加密資料庫 |
| 禁止截圖 | JS 寫 `<AppData>/secure.flag`，`MainActivity.kt` 以 `FileObserver` 監看並設定 `FLAG_SECURE` | 不另寫 Tauri Kotlin 外掛與 Rust 指令；不依賴未查證的 WebView 掛鉤 API |
| Argon2id | 以 Blob Worker 執行，失敗退回主執行緒 | 實測總時間不變，但運算期間畫面不再凍結 |
| 圖示 | `app-icon.png`（1024×1024、透明背景、主體在中央約 66% 內）；`tauri icon app-icon.png --ios-color "#234e70"`，此色碼同時寫入 Android 的背景色檔案 | 行動圖示直接寫入 `gen/android`，須先有該專案 |
| 自訂圖示 | 來源 `~/icon.png` 以 Pillow 縮放置中到 1024×1024 透明畫布（最長邊 680 像素），存成 `~/PyPasswords/app-icon.png` 後執行 `tauri icon` | Android 自適應圖示會被遮罩裁切，主體須在中央約三分之二內；同一次也產生 `.icns` 與 `.ico` |
| 返回鍵 | `MainActivity.kt` 以 `OnBackPressedCallback` 攔截，`evaluateJavascript` 呼叫 `window.__pwBack`；結果非 `"true"` 就 `finish()` | JS 端才知道目前有哪些對話框與畫面層級；用檢視樹找 `WebView`，不依賴 Tauri 內部欄位 |
| 正式版簽署（現況） | 以 uber-apk-signer 對 `app-universal-release-unsigned.apk` 對齊並以內嵌 debug 金鑰簽署 | 可覆蓋安裝 debug 版而不清資料；不是正式金鑰，未來換成自有金鑰須先匯出備份並解除安裝 |
| 應用程式名稱 | `productName`、視窗標題、`strings.xml` 的 `app_name` 與 `main_activity_title` 改為 `PyPasswords`；`identifier` 不變 | 資料目錄依 `identifier`，改名不影響 vault 與設定；指令見 9.3.2，結果尚未回報 |

固定事項：`identifier` 定案後不可更動；不更動 `useHttpsScheme`；`src-tauri/gen/android` 納入版本控制。  

### 9.3 macOS 建置流程

環境變數（寫入 `~/.zshrc`）：

```
export ANDROID_HOME="$HOME/Android/Sdk"
export NDK_HOME="$ANDROID_HOME/ndk/28.2.13676358"
export JAVA_HOME="/Library/Java/JavaVirtualMachines/temurin-17.jdk/Contents/Home"
export PATH="$PATH:$ANDROID_HOME/platform-tools"
```

建立專案與外掛：

```
cd ~
[ -e ~/PyPasswords ] && mv ~/PyPasswords ~/PyPasswords.old
npm create tauri-app@latest PyPasswords -- -m npm -t vanilla --identifier com.havelight.pypasswords -y
cd ~/PyPasswords
npm install
npm run tauri add fs
npm run tauri add dialog
npm run tauri add clipboard-manager
```

放入前端、設定並初始化 Android：

```
cp ~/Downloads/index.html src/index.html
rm -f src/main.js src/styles.css
jq '.app.withGlobalTauri=true' src-tauri/tauri.conf.json > /tmp/tc.json && mv /tmp/tc.json src-tauri/tauri.conf.json
jq '.permissions += ["dialog:allow-open","dialog:allow-save","dialog:allow-message","clipboard-manager:allow-write-text","clipboard-manager:allow-clear","fs:allow-appdata-read-recursive","fs:allow-appdata-write-recursive","fs:allow-appdata-meta-recursive","fs:allow-read-file","fs:allow-write-file"] | .permissions |= unique' src-tauri/capabilities/default.json > /tmp/cap.json && mv /tmp/cap.json src-tauri/capabilities/default.json
npm run tauri android init
```

1：交付的 `index.html` 已含全部修改，不需任何修補腳本。  
3：啟用 `window.__TAURI__`。  
4：補齊權限並去重。  
5：產生 `src-tauri/gen/android`，之後的圖示與主題修改都依賴它。  

圖示與 `MainActivity.kt`：

```
python3 -m venv /tmp/iconEnv
/tmp/iconEnv/bin/pip install pillow
/tmp/iconEnv/bin/python ~/Downloads/makeIcon.py
npm run tauri icon app-icon.png -- --ios-color "#234e70"
cp ~/Downloads/MainActivity.kt $(find src-tauri/gen/android -name MainActivity.kt)
```

3：使用預設鎖頭圖示；若要改用自己的圖片，改為把 1024×1024 的 PNG 複製成 `app-icon.png` 後再執行第 4 行，不必執行第 3 行。  
5：覆蓋前先以 `head -1` 確認原檔第一行為 `package com.havelight.pypasswords`。  

狀態列背景色資源（只需執行一次；若 `themes.xml` 已含 `pw_window_bg` 則不會重複寫入）：

```
cd ~/PyPasswords/src-tauri/gen/android/app/src/main/res
mkdir -p values values-night
printf '<?xml version="1.0" encoding="utf-8"?>\n<resources>\n    <color name="pw_window_bg">#FBFCFD</color>\n</resources>\n' > values/pw_colors.xml
printf '<?xml version="1.0" encoding="utf-8"?>\n<resources>\n    <color name="pw_window_bg">#151D25</color>\n</resources>\n' > values-night/pw_colors.xml
for f in $(find . -name themes.xml); do grep -q pw_window_bg "$f" || perl -0pi -e 's|</style>|    <item name="android:windowBackground">\@color/pw_window_bg</item>\n  </style>|' "$f"; done
cd ~/PyPasswords
```

建置與安裝：

```
npm run tauri android build -- --apk --debug --target aarch64
adb install -r src-tauri/gen/android/app/build/outputs/apk/universal/debug/app-universal-debug.apk
```

2：從 Linux 建的舊版換成 macOS 建的版本時，debug 簽章不同，會出現 `INSTALL_FAILED_UPDATE_INCOMPATIBLE`，需先 `adb uninstall com.havelight.pypasswords`（等於清空資料，所以先匯出備份）。修改 `src/index.html` 後須重新執行第 1 行才會更新 APK。  

正式版對照測試（以 debug 金鑰簽署的 release 版，可直接覆蓋安裝 debug 版，用來判斷卡頓是否來自 debug 版）：

```
npm run tauri android build -- --apk --target aarch64
apk=$(find src-tauri/gen/android -name '*unsigned.apk' | head -1)
bt=$HOME/Android/Sdk/build-tools/$(ls $HOME/Android/Sdk/build-tools | sort -V | tail -1)
$bt/zipalign -f -p 4 "$apk" /tmp/pw-aligned.apk
$bt/apksigner sign --ks ~/.android/debug.keystore --ks-pass pass:android --out /tmp/pw-release-test.apk /tmp/pw-aligned.apk
adb install -r /tmp/pw-release-test.apk
```

正式版無法使用 `run-as`。  

正式版簽署（驗證完成後再做，尚未執行）：

```
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
npm run tauri android build -- --apk --target aarch64
apk=$(find src-tauri/gen/android -name '*unsigned.apk' | head -1)
bt=$HOME/Android/Sdk/build-tools/$(ls $HOME/Android/Sdk/build-tools | sort -V | tail -1)
$bt/zipalign -v -p 4 "$apk" /tmp/pypasswords-aligned.apk
$bt/apksigner sign --ks ~/upload-keystore.jks --ks-key-alias upload --out ~/PyPasswords/PyPasswords-release.apk /tmp/pypasswords-aligned.apk
$bt/apksigner verify ~/PyPasswords/PyPasswords-release.apk
```

金鑰遺失即無法原地更新，只能解除安裝而清空 vault，須備份 `~/upload-keystore.jks` 與其密碼，並先匯出 vault。  

### 9.3.1 自訂圖示（來源 `~/icon.png`，使用者已執行並成功）

```
[ -x /tmp/iconEnv/bin/python ] || python3 -m venv /tmp/iconEnv
/tmp/iconEnv/bin/pip install pillow
cd ~/PyPasswords
/tmp/iconEnv/bin/python -c "import os;from PIL import Image;im=Image.open(os.path.expanduser('~/icon.png')).convert('RGBA');s=680/max(im.size);im=im.resize((round(im.width*s),round(im.height*s)),Image.LANCZOS);c=Image.new('RGBA',(1024,1024),(0,0,0,0));c.paste(im,((1024-im.width)//2,(1024-im.height)//2),im);c.save('app-icon.png')"
npm run tauri icon app-icon.png -- --ios-color "#234e70"
cat src-tauri/gen/android/app/src/main/res/values/ic_launcher_background.xml
jq '.bundle.icon' src-tauri/tauri.conf.json
```

1：`/tmp` 重開機後會被清掉，所以先確認虛擬環境存在。  
4：把圖縮到最長邊 680 像素並置中；來源圖建議至少 1024 像素。  
5：`#234e70` 會成為 Android 圖示背景色；若圖示有自己的底色，改成該色碼。  
6、7：檢查背景色檔與 `bundle.icon` 清單（應含 `icons/icon.icns`，`.dmg` 用）。  

圖示不更新是啟動器快取，重開機即可；解除安裝重裝會清空資料，須先匯出備份。  

### 9.3.2 改名為 PyPasswords（指令已提供，結果尚未回報）

```
cd ~/PyPasswords
jq '.productName="PyPasswords" | .app.windows[0].title="PyPasswords"' src-tauri/tauri.conf.json > /tmp/tc.json && mv /tmp/tc.json src-tauri/tauri.conf.json
perl -pi -e 's|(<string name="app_name">)[^<]*(</string>)|$1PyPasswords$2|; s|(<string name="main_activity_title">)[^<]*(</string>)|$1PyPasswords$2|' src-tauri/gen/android/app/src/main/res/values/strings.xml
grep -n "app_name\|main_activity_title" src-tauri/gen/android/app/src/main/res/values/strings.xml
jq '.productName, .app.windows[0].title' src-tauri/tauri.conf.json
```

不更動 `identifier`。`.dmg` 檔名會依 `productName` 變成 `PyPasswords_<版本>_aarch64.dmg`。  

### 9.3.3 正式版 APK（使用者實際使用的流程）

```
cd ~/PyPasswords
npm run tauri android build -- --apk --target aarch64
java -jar ~/bin/uber-apk-signer.jar -a src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release-unsigned.apk
adb install -r src-tauri/gen/android/app/build/outputs/apk/universal/release/app-universal-release-aligned-debugSigned.apk
```

此為 debug 金鑰簽署。以自有金鑰簽署的流程仍是前面「正式版簽署」一節（尚未執行）。  

### 9.3.4 macOS `.dmg`（指令已提供，尚未回報結果）

```
cd ~/PyPasswords
jq '.app.windows[0].width=1100 | .app.windows[0].height=760' src-tauri/tauri.conf.json > /tmp/tc.json && mv /tmp/tc.json src-tauri/tauri.conf.json
npm run tauri build -- --bundles dmg
ls -lh src-tauri/target/release/bundle/dmg/
```

2：選用；預設視窗 800 像素寬低於 820 像素斷點，會顯示手機版面。假設範本內已有 `app.windows[0]`。  
3：只打包 `.dmg`，預設為本機架構（Apple Silicon 即 aarch64）。同時支援 Intel 需 `rustup target add x86_64-apple-darwin` 並加 `--target universal-apple-darwin`。  

請在本機圖形介面的 Terminal 執行（`.dmg` 排版會用 AppleScript）。未簽署也未公證的 App 在別台 Mac 會被 Gatekeeper 擋下。「禁止截圖」與返回鍵只在 Android 有效。  

### 9.4 `MainActivity.kt` 內容

```
package com.havelight.pypasswords

import android.os.Bundle
import android.os.FileObserver
import android.view.View
import android.view.ViewGroup
import android.view.WindowManager
import android.webkit.WebView
import androidx.activity.OnBackPressedCallback
import androidx.activity.enableEdgeToEdge
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import java.io.File

class MainActivity : TauriActivity() {
  private val observers = mutableListOf<FileObserver>()

  private fun flagFiles() = listOf(File(dataDir, "secure.flag"), File(filesDir, "secure.flag"))

  private fun applySecure() {
    val on = flagFiles().any {
      try { it.exists() && it.readText().trim() == "1" } catch (e: Exception) { false }
    }
    runOnUiThread {
      if (on) window.addFlags(WindowManager.LayoutParams.FLAG_SECURE)
      else window.clearFlags(WindowManager.LayoutParams.FLAG_SECURE)
    }
  }

  private fun watch(dir: File) {
    @Suppress("DEPRECATION")
    val observer = object : FileObserver(
      dir.path,
      FileObserver.CLOSE_WRITE or FileObserver.MOVED_TO or FileObserver.CREATE
    ) {
      override fun onEvent(event: Int, path: String?) {
        if (path == "secure.flag") applySecure()
      }
    }
    observer.startWatching()
    observers.add(observer)
  }

  private fun findWebView(v: View): WebView? {
    if (v is WebView) return v
    if (v is ViewGroup) {
      for (i in 0 until v.childCount) {
        val r = findWebView(v.getChildAt(i))
        if (r != null) return r
      }
    }
    return null
  }

  override fun onCreate(savedInstanceState: Bundle?) {
    enableEdgeToEdge()
    super.onCreate(savedInstanceState)
    applySecure()
    watch(dataDir)
    if (filesDir.path != dataDir.path) watch(filesDir)
    val root = findViewById<View>(android.R.id.content)
    ViewCompat.setOnApplyWindowInsetsListener(root) { view, insets ->
      val types = WindowInsetsCompat.Type.systemBars() or
        WindowInsetsCompat.Type.displayCutout() or
        WindowInsetsCompat.Type.ime()
      val bars = insets.getInsets(types)
      view.setPadding(bars.left, bars.top, bars.right, bars.bottom)
      WindowInsetsCompat.CONSUMED
    }
    ViewCompat.requestApplyInsets(root)
    onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
      override fun handleOnBackPressed() {
        val wv = findWebView(root)
        if (wv == null) { finish(); return }
        wv.evaluateJavascript("(function(){return window.__pwBack?window.__pwBack():false})()") { r ->
          if (r != "true") finish()
        }
      }
    })
  }

  override fun onDestroy() {
    observers.forEach { it.stopWatching() }
    observers.clear()
    super.onDestroy()
  }
}
```

### 9.5 實機驗證清單

狀態：第 1 至 14 項使用者已回報全部 OK（未逐項附輸出）；第 15 至 17 項為本輪新增，使用者回報測試完成，未逐項描述。  

1. 以第 9.3 節的方式覆蓋 `index.html` 與 `MainActivity.kt` 後重新建置安裝；Kotlin 若編譯失敗，貼回完整錯誤。  
2. 標題列：窄螢幕為兩列，三個按鈕完整可見，中文與英文都不被截斷。  
3. 設定：切換語言即時生效；改預設格式後建立表單同步；各項設定重開 App 後仍保留。  
4. 禁止截圖：開啟後截圖與螢幕錄影應失敗或為黑畫面，最近工作畫面縮圖為黑；關閉後恢復。若無效，執行 `adb shell run-as com.havelight.pypasswords ls -la`，確認 `secure.flag` 位置與內容。  
5. 禁止選取：長按按鈕與一般文字不再出現選取把手或選單；輸入框仍可選取與貼上。  
6. Web Worker：解鎖與建立 PWV3 設定檔時，轉圈動畫應持續轉動；若動畫仍停格，用 logcat 查看是否有 Worker 相關錯誤（程式會自動退回主執行緒，不會失敗）。  
7. 變更主碼：舊主碼錯誤、新舊相同、兩次不一致、成功後以新主碼重新解鎖，紀錄完整；`.bin.bak` 保留變更前的檔案。  
8. 多選刪除：勾選、全選（含搜尋過濾後）、刪除後筆數正確；Android 返回鍵與鎖定時的行為。  
9. 自動轉換：開啟並選與目前不同的預設格式後解鎖，應提示已轉換，再用「更多」確認目前格式。  
10. PIN：設定、變更、移除、輸入錯誤；刪除資料庫時依序要求名稱與 PIN；PIN 輸入框為遮罩加數字鍵盤。  
11. 複製密碼後提醒的開關；剪貼簿提醒文字。  
12. 反覆強制結束並重開 App，確認不再卡頓；若復發，以 `adb logcat -v time | grep -iE "\[PW\]|pypasswords|AndroidRuntime|FATAL|ANR|chromium|tauri"` 收集記錄。  
13. 匯出的 `.bin` 與 `.sha256` 以 `sha256sum` 比對，並以原 Python 版解密。  
14. 自訂圖示時，確認桌面圖示未被遮罩裁切；圖示不更新是啟動器快取，重開機或解除安裝後再裝。  
15. 返回鍵：對話框逐層關閉、運算中不動作、編輯取消、多選結束、詳情回清單、搜尋清空、清單最上層鎖定並退出、建立分頁回解鎖分頁、解鎖頁退出（見 4.7）。  
16. 單筆刪除：已設定 PIN 時依序出現確認、PIN、主碼；未設定 PIN 時略過 PIN。  
17. 名稱：桌面圖示下方與設定中的應用程式名稱為 `PyPasswords`（見 9.3.2）。  

## 10. Git 版本控制（步驟已提供，尚未回報執行結果）

專案路徑為 `~/PyPasswords`（使用者曾寫成 `~/PyPassword`）。決定事項：  

- `src-tauri/gen/android`、`src-tauri/icons`、`app-icon.png`、`package-lock.json`、`src-tauri/Cargo.lock` 都納入版本控制。  
- 絕不納入：金鑰（`*.jks`、`*.keystore`、`keystore.properties`）、`*.apk`、`*.dmg`、`node_modules`、`src-tauri/target`，以及所有 vault 資料（`*.bin`、`*.sha256`、`*.bak`、`*.tmp`）。  
- 根目錄 `.gitignore` 內容如下（`src-tauri/.gitignore` 為範本自帶，保留不動）：  

```
.DS_Store
node_modules/
dist/
.vite/
*.log

src-tauri/target/
src-tauri/gen/schemas/

src-tauri/gen/android/.gradle/
src-tauri/gen/android/build/
src-tauri/gen/android/app/build/
src-tauri/gen/android/local.properties
src-tauri/gen/android/app/src/main/jniLibs/
*.iml
.idea/

*.apk
*.aab
*.dmg
*.jks
*.keystore
keystore.properties
*.p12
*.pem

*.bin
*.sha256
*.bak
*.tmp
.env
.env.*
```

建立與提交：  

```
cd ~/PyPasswords
git rev-parse --show-toplevel
git init
git add -A
git status --short
git ls-files | grep -E '\.(bin|jks|keystore|apk|dmg|sha256)$'
git commit -m "Initial commit: PyPasswords Tauri 2 app"
```

2：`git init` 前先確認上層沒有別的 repo（若印出路徑，代表 `~/PyPasswords` 已在某個 repo 內）。  
6：不應出現任何列；若有，先改 `.gitignore` 再 `git rm --cached <檔案>`。  

未確定之處：Tauri 自帶的 `src-tauri/gen/android/.gitignore` 實際內容未查證，提交後以 `git ls-files src-tauri/gen/android` 檢查是否混入 `build/`、`.gradle/`、`jniLibs/` 的產物，並以 `git check-ignore -v <路徑>` 查明被哪一條規則忽略。  
