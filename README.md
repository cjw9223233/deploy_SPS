# 車站端 SPS 新舊切換流程 — 線上操作手冊

把紙本／Word 版的「車站端SPS新舊切換流程（正式用）」轉成**可勾選、可複製指令、會自動帶入車站 IP** 的單頁網頁，供現場人員在更換車站 SPS 主機時照表操作。

- **線上版（GitHub Pages）**：<https://cjw9223233.github.io/deploy_SPS/>（預期網址，請以 repo Settings → Pages 顯示為準）
- **原始依據**：`車站端SPS新舊切換流程_正式用.docx`（未放在本 repo）
- **技術組成**：單一 HTML 檔（內嵌 CSS + 原生 JavaScript），**無後端、無建置步驟、無外部相依套件**

---

## 1. 快速開始

### 現場使用者

1. 開啟線上版網址（或直接用瀏覽器開啟 `index.html`）。
2. 在最上方「總覽」填入 **車站、人員、SPS IP**（例如 `172.4.12.240`）。
   - 輸入正確的 `.240` IP 後，右側會顯示對應車站名稱（例如「✓ 科技大樓」）。
   - 全文中的 IP 佔位字會自動換成該站網段，按「複製」即可取得可直接貼上的指令。
3. 由上往下執行，**每完成一步就打勾**；上方進度條會同步更新，「跳到未完成」可快速定位。
4. 進度會存在本機瀏覽器，重新整理不會遺失；換電腦／換瀏覽器需重填。
5. 整站做完、換下一站前，按「重置進度」清空。

### 開發者

```bash
git clone https://github.com/cjw9223233/deploy_SPS.git
cd deploy_SPS
# 直接用瀏覽器開啟 index.html 即可預覽，不需要任何伺服器或 npm
```

改完 → 用瀏覽器重新整理確認 → commit → push 到 `master`，約 1–2 分鐘後線上版自動更新（見〈5. 發佈流程〉）。

---

## 2. 目錄結構

| 路徑 | 角色 | 是否上線 |
|---|---|---|
| `index.html` | **唯一的正式檔案**，所有修改都改這裡 | 是 |
| `.github/workflows/static.yml` | push 到 `master` 時自動部署到 GitHub Pages | — |
| `src/manual.src.html` | 09/11 當時的原始檔快照，外包一層 `initManual()`，預留給「加密版」使用（加密版未實作） | 否（但仍可經由網址被存取） |
| `docs/車站端SPS新舊切換流程_操作手冊_v1_1_09_11.html` | v1.1_09.11 的封存版本 | 否（同上） |

> **注意**：`src/` 與 `docs/` **沒有**跟著 `index.html` 更新（例如缺少車站對照表、MMIServerCOMMSend 版本仍是舊的）。請勿以它們為基礎修改，否則會把新功能蓋掉。

---

## 3. 頁面架構

`index.html` 由上到下分為：

| 區塊 | 錨點 id | 內容 |
|---|---|---|
| 頂部固定列 | `header` | 標題、版本徽章 `.ver-badge`、進度條、跳到未完成、重置進度、快速導覽 `nav.quicknav` |
| 總覽 | `#overview` | 車站／人員／SPS IP 輸入欄、預估時程、整體流程圖 |
| A. 前置作業 | `#sec-a` | BIOS、RAID、傳送小工具、打包舊參數資料夾 |
| B. 舊SPS執行步驟 | `#sec-b-old` | 在舊主機上執行的腳本 |
| B. 新SPS執行步驟 | `#sec-b-new` | 在新主機上的資料恢復與更新 |
| 51/230主機 ssh修正 | `#sec-ssh` | 清除 known_hosts、SPSAgent 拋轉確認 |
| 安裝結束檢查 | `#sec-check` | 完工檢查清單 |
| 版本確認 | `#sec-version` | 各程式安裝包版本對照表 |
| 頁尾 + `<script>` | — | 所有互動邏輯都在檔案最後這一段 `<script>` |

新增區塊時，記得同步在 `nav.quicknav` 加一個 `<a href="#新id">` 連結。

---

## 4. 功能與程式邏輯

以下功能全部寫在 `index.html` 最下方的 `<script>` 內。

### 4.1 IP 自動帶入（`applyIp()`）

使用者輸入 SPS IP 後，取**前三段**作為網段（prefix），將全文（`<main>` 內的文字節點與按鈕的 `data-cmd` 屬性）中的佔位字替換：

| 佔位字（寫在 HTML 裡） | 替換結果 | 意義 |
|---|---|---|
| `172.xxx.xxx` | `<prefix>` | 網段，後面自行接 `.240`、`.241` 等 |
| `該站SPS IP` | `<prefix>.240` | 舊 SPS |
| `<新SPS IP>` | `<prefix>.241` | 新 SPS（HTML 中需寫成 `&lt;新SPS IP&gt;`） |

原始模板在頁面載入時就記起來（`textTargets` / `attrTargets`），所以改 IP 可以反覆套用、清空即還原。

### 4.2 車站對照（`STATION_MAP`）

`{ "完整 .240 IP": "車站名稱" }` 的物件。輸入的 IP 完全符合某個 key 時，`#ip-hint` 顯示車站名稱；否則顯示「未找到對應車站」（但網段照樣套用）。

### 4.3 複製按鈕（`copyCmd()`）

每個 `.copy-btn` 會複製自己的 `data-cmd` 屬性（**不是** `<code>` 的內容）。優先使用 Clipboard API，失敗時退回 `document.execCommand('copy')`（`fallbackCopy()`），讓 `file://` 開啟時也能用。

### 4.4 勾選進度

- 所有要追蹤的 checkbox 都有 class `track`。
- **母子連動**：同一個 `.step` 內，`.step-head` 裡的是母框，`.step-body` 裡的（class `track sub`）是子框。勾母框 = 全勾子框；子框部分勾選時母框呈現半勾（indeterminate）。
- **進度只算葉節點**（沒有子框的 checkbox），避免重複計數。
- 「跳到未完成」捲動到文件順序中第一個未勾的項目（`jumpToNext()`）；`HEADER_OFFSET = 170` 是為了避開固定頂列。

### 4.5 本機儲存（localStorage）

透過 `LS` 包裝（`file://` 下被瀏覽器擋掉時不會讓整支腳本掛掉）。使用的 key：

| 資料 | key |
|---|---|
| SPS IP | `sps_switch_v1_0909_ip` |
| 車站、人員欄位 | `sps_switch_v1_1_0909_txt_<input id>` |
| 每個勾選框 | `sps_switch_v1_1_0909_<第幾個 .track>` |

> **重要陷阱**：勾選框的 key 是用「在頁面中的順序編號」產生的。**新增或刪除任何一個 checkbox，後面所有框的編號都會位移**，正在操作中的人重新整理後，勾選狀態會錯位。
> 因此調整步驟（增刪 checkbox）時，請一併把 `STORAGE_PREFIX` 改成新的版本字串（例如 `sps_switch_v1_2_1001_`），讓舊進度自動失效，並通知現場人員。

---

## 5. 常見修改指南

### 5.1 新增／修改一個步驟

在對應的 `<section>` 內複製一個既有 `.step`，照下列結構改：

```html
<div class="step">
  <label class="step-head"><input type="checkbox" class="track"><span>3. 步驟標題</span></label>
  <div class="step-body">
    <p>說明文字（可省略）</p>
    <div class="cmd-block">
      <div class="cmd-row"><code>cd /home/sps</code><button class="copy-btn" data-cmd="cd /home/sps" onclick="copyCmd(this)">複製</button></div>
    </div>
  </div>
</div>
```

需要子步驟時，在 `.step-body` 裡加：

```html
<label class="sub-head"><input type="checkbox" class="track sub"><span>1. 子步驟標題</span></label>
<div class="cmd-block">
  <div class="cmd-row"><code>指令</code><button class="copy-btn" data-cmd="指令" onclick="copyCmd(this)">複製</button></div>
</div>
```

檢查要點：

- `<code>` 內文字與 `data-cmd` **兩邊都要改且一致**（按鈕複製的是 `data-cmd`）。
- 指令裡有 `<`、`>`、`&`、`"` 時要寫成 HTML 實體：`&lt;` `&gt;` `&amp;` `&quot;`。
- 需要帶入站點 IP 的地方用 4.1 的佔位字，例如 `scp WL.tar.gz sps@172.xxx.xxx.241:/home/sps`。
- 有增刪 checkbox → 依 4.5 更新 `STORAGE_PREFIX`。
- 需要警示時可用 `<div class="note">…</div>`。

### 5.2 新增車站

在 `STATION_MAP` 加一行（參考 commit `ae6131c` 加入「廣慈/奉天宮」的做法）：

```js
"172.8.104.240":"廣慈/奉天宮",
```

key 必須是**完整且結尾為 `.240`** 的 IP；同名站依路線加前綴區分（如 `BL台北車站`、`R台北車站`）。

### 5.3 更新程式版本

修改 `#sec-version` 表格對應列的第二欄：

```html
<tr><td class="mono">MMIServerCOMMSend</td><td>1.4.3a1</td><td></td></tr>
```

第三欄「新版」保留空白，供現場填寫／確認。

### 5.4 發佈新版手冊

1. 修改頂部 `<span class="ver-badge">v1.1_09.11</span>` 的版本字串。
2. 若步驟有增刪，同步更新 `STORAGE_PREFIX`（見 4.5）。
3. commit 訊息寫清楚改了什麼（例如「加入XX站」「更新XX版本」），方便日後追溯。

---

## 6. 發佈流程

```
改 index.html → 本機瀏覽器驗證 → git push origin master
        → GitHub Actions「Deploy static content to Pages」自動執行
        → 約 1–2 分鐘後線上版更新
```

- 部署狀態：GitHub repo 的 **Actions** 分頁。也可在該分頁手動 `Run workflow` 重新部署。
- 線上版有快取，看不到更新時請強制重新整理（Ctrl+F5）。
- 改壞了要回復：`git revert <commit>` 後 push，不要用 force push 改寫歷史。

### 修改後自我檢查清單

- [ ] 瀏覽器開啟無錯誤（F12 → Console 無紅字）
- [ ] 輸入一個已知 IP（如 `172.4.12.240`），確認顯示車站名稱，且指令中的 IP 正確替換
- [ ] 隨機點幾個「複製」，貼到記事本確認內容正確
- [ ] 勾選／取消母框與子框，進度數字正確
- [ ] 「重置進度」可清空全部欄位
- [ ] 手機寬度下版面正常（F12 → 裝置模擬）

---

## 7. 注意事項與已知問題

- **本 repo 與 GitHub Pages 皆為公開**，工作流程設定為「上傳整個 repo」，repo 內任何檔案都能被外部讀取。頁面中目前含有主機帳號密碼等內部資訊，**請評估改為私有 repo／內網部署，或移除敏感資訊**；日後也請勿再加入任何密碼、內部文件。
- `index.html` 使用 **CRLF** 換行，編輯器請維持原換行格式，避免整份檔案都顯示為變更。
- `src/`、`docs/` 為舊快照，未同步（見〈2. 目錄結構〉）；`src/` 中的 `window.__ENCRYPTED__` / `initManual()` 是預留給加密版的掛勾，目前沒有任何加密流程。
- IP 的 key 前綴（`sps_switch_v1_0909_ip`）與其他 key（`sps_switch_v1_1_0909_…`）命名不一致，屬歷史遺留，改動時請留意不要讓使用者已填的 IP 消失。
- 原始 Word 檔不在 repo 中；Word 與網頁內容若有出入，請與流程負責人確認以何者為準。

---

## 8. 版本紀錄

| 日期 | 內容 |
|---|---|
| 2026-09-11 | v1.1_09.10 初版上線；改名 `index.html` 並設定 GitHub Pages；v1.1_09.11 定版 |
| 2026-09-14 | 版本確認表新增 SPS_MMI |
| 2026-09-21 | 更新 SPSEventAlarmStatusController、SPSFNR、SPSWatchDog、SPS_MMI 版本 |
| 2026-09-23 | 新增「輸入 IP 後顯示車站名稱」（`STATION_MAP`） |
| 2026-09-23 | 加入廣慈/奉天宮站；更新 MMIServerCOMMSend 版本 |

完整紀錄請看 `git log`。

## 9. 聯絡窗口

| 角色 | 人員 |
|---|---|
| 專案負責／維護 | _（請填寫）_ |
| 流程內容確認 | _（請填寫）_ |
