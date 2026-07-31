# Android：Threads＋Instagram＋Facebook 複製分享網址後自動展開、清理並寫回剪貼簿

## 目標

在 Threads、Instagram 或 Facebook 點「複製連結」後，自動完成：

```text
社群平台分享網址
↓
MacroDroid 偵測剪貼簿
↓
Threads／Instagram：交給 URLCheck 查詢重新導向
Facebook：MacroDroid 下載 HTML，擷取 canonical／og:url
↓
轉成正式內容網址
↓
移除 xmt、slof、igsh、igshid、fbclid、mibextid、rdid、utm_* 等追蹤參數
↓
乾淨網址自動寫回剪貼簿
↓
URLCheck 自動關閉
```

實際效果示例：

```text
Threads 原本：
https://www.threads.com/share/xxxxxxxx

Threads 處理後：
https://www.threads.com/@帳號/post/貼文代碼
```

```text
Instagram 原本：
https://www.instagram.com/reel/ABCDEFGHIJK/?igsh=xxxxxxxx

Instagram 處理後：
https://www.instagram.com/reel/ABCDEFGHIJK/
```

```text
Facebook 原本：
https://www.facebook.com/share/r/xxxxxxxx/

Facebook 處理後：
https://www.facebook.com/reel/123456789012345/
```

> [!IMPORTANT]
> Facebook 的 `/share/` 網址通常直接回傳 `200 OK`，不一定提供標準 `301/302 Location`。因此 Facebook **不能只靠 URLCheck 的 `checkStatus`**；本文件改用 MacroDroid HTTP GET 擷取 HTML 中的 `canonical`／`og:url`，再交給 URLCheck 清理與收尾。
>
> Facebook 網頁結構可能改版。本方案採「失敗安全」設計：只有解析結果符合已驗證的正式網址白名單時，才覆寫剪貼簿；解析失敗時保留原始分享網址並顯示通知。

## 驗證狀態（2026-08-01）

- Threads：已在 Android 手機實測成功。
- Instagram：沿用 URLCheck＋MacroDroid 架構；各內容類型仍應分別測試。
- URLCheck 網址清理器：已建立 GitHub 自訂規則與 SHA-256 檔，可由更新器拉取並合併到現有規則。
- URLCheck Automations：已建立 GitHub 完整 JSON 備份；原版 URLCheck 3.5 仍需手動貼入更新。
- Facebook：
  - 一般／粉專貼文：已實測。
  - 含長標題 slug 的貼文：已實測，可縮成 `/帳號/posts/數字ID/`。
  - 社團貼文：已實測。
  - Reels：已實測。
  - `fb.watch`、相片、其他影片格式、活動、Marketplace：尚未完整驗證，不列入預設白名單。
  - `login next` 後備解析已設定，但本輪成功案例均由 `canonical` 取得，尚未實際觸發驗證。

---

## 使用工具

- URLCheck
- MacroDroid
- Shizuku
- aShell

Shizuku＋aShell 只用來授予 MacroDroid 的 `READ_LOGS` 權限。

---

# 一、URLCheck：網址清理器（ClearURLs）追蹤參數清理

## 建議方式：使用 GitHub 規則來源

本專案把自訂規則放在：

```text
urlcheck/custom-rules.json
```

進入 URLCheck 網址清理器的「更新器／規則更新」畫面，填入：

規則網址：

```text
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.json
```

SHA-256 網址：

```text
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.sha256
```

然後按「立即更新」，確認成功後開啟自動更新，並在網址清理器開啟「自動套用」。URLCheck 的自動檢查週期約為一週，也可以隨時手動更新。

### 更新時會不會覆蓋原生 `providers`？

不會。遠端檔案只有最外層的 `"自訂規則"`：

```json
{
  "自訂規則": {
    "Threads": {
      "urlPattern": "^https?://(?:www\.)?threads\.com",
      "rules": ["xmt", "slof"]
    },
    "Instagram": {
      "urlPattern": "^https?://(?:(?:www|m)\.)?instagram\.com",
      "rules": ["igsh", "igshid"]
    },
    "Facebook": {
      "urlPattern": "^https?://(?:(?:(?:www|m|mbasic|web|touch|l|lm)\.)?facebook\.com|(?:www\.)?fb\.watch)",
      "rules": ["fbclid", "mibextid", "rdid", "share_url", "ref", "refsrc", "__cft__", "__tn__"]
    }
  }
}
```

URLCheck 拉取遠端規則時採用最外層物件合併：

```text
手機目前：providers + 舊版自訂規則
GitHub 遠端：新版自訂規則
更新後：providers 保留，自訂規則替換成新版
```

遠端沒有 `"providers"`，所以不會碰到手機現有的原生規則。

> [!WARNING]
> 不要把 `custom-rules.json` 直接貼進網址清理器的 JSON 編輯器，再用它取代整份內容。JSON 編輯器儲存的是完整目錄；若只貼入 `"自訂規則"`，手機上的 `"providers"` 會消失。
>
> `custom-rules.json` 應填在更新器的規則網址中，讓 URLCheck 以合併模式套用。

### 官方規則後續更新限制

把規則網址改成此 GitHub Raw 位址後，URLCheck 會從本專案取得自訂規則，不再自動連到 ClearURLs 官方規則來源。因此手機目前已有的 `"providers"` 會保留，但官方日後新增或修改的規則不會跟著自動更新。

需要手動刷新官方規則時：

```text
1. 在更新器使用「還原／恢復預設」，回到官方規則網址
2. 按「立即更新」，更新官方 providers
3. 再填回本專案的規則網址與 SHA-256 網址
4. 再按一次「立即更新」，合併最新版自訂規則
```

官方更新會替換 `"providers"`，本專案更新會替換 `"自訂規則"`，兩者可以並存。

## 手動備援方式

無法使用遠端更新器時，才進入：

```text
URLCheck
→ 模組
→ 網址清理器
→ JSON 編輯器
```

請在完整目錄最外層、與 `"providers"` 同一層加入 `"自訂規則"`：

```json
{
  "providers": {
    "原本官方規則": {
      "urlPattern": "...",
      "rules": ["..."]
    }
  },
  "自訂規則": {
    "Threads": {
      "urlPattern": "^https?://(?:www\.)?threads\.com",
      "rules": ["xmt", "slof"]
    },
    "Instagram": {
      "urlPattern": "^https?://(?:(?:www|m)\.)?instagram\.com",
      "rules": ["igsh", "igshid"]
    },
    "Facebook": {
      "urlPattern": "^https?://(?:(?:(?:www|m|mbasic|web|touch|l|lm)\.)?facebook\.com|(?:www\.)?fb\.watch)",
      "rules": ["fbclid", "mibextid", "rdid", "share_url", "ref", "refsrc", "__cft__", "__tn__"]
    }
  }
}
```

自訂規則不要放進 `"providers"`，否則官方 `"providers"` 更新時可能把它覆蓋。

## 網址清理器模組設定

確認開啟：

```text
自動套用
```

這部分負責清除：

```text
Threads：xmt、slof
Instagram：igsh、igshid
Facebook：fbclid、mibextid、rdid、share_url、ref、refsrc、__cft__、__tn__
通用規則：utm_source、utm_medium、utm_campaign 等
```

網址清理器只能處理網址參數，不能從 Facebook 的 HTML 頁面自行推算真正貼文網址。

---

# 二、URLCheck：自動展開、清理並複製

進入：

```text
URLCheck
→ 自動化／Automations
→ JSON 編輯器
```

## GitHub 設定檔與更新限制

Automations 完整設定：

```text
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/automations.json
```

對應 SHA-256：

```text
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/automations.sha256
```

> [!IMPORTANT]
> URLCheck 3.5 的 Automations 只有本機 JSON 編輯器，沒有網址清理器那種遠端規則網址、雜湊驗證與自動更新功能。因此 GitHub 上的 `automations.json` 是最新版備份與發布來源，但手機端仍須手動更新。

更新方式：

```text
1. 開啟 automations.json Raw 網址
2. 複製完整 JSON
3. 進入 URLCheck → 自動化／Automations → JSON 編輯器
4. 全選舊內容並貼上新版完整 JSON
5. 儲存
```

`automations.sha256` 可用來確認下載內容是否一致，但 URLCheck Automations 本身不會讀取這個雜湊檔。

使用以下完整設定：

```json
{
  "Threads：展開短網址": {
    "regex": "^https?:\\/\\/(?:www\\.)?threads\\.com\\/share\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Threads：複製展開後網址": {
    "regex": "^https?:\\/\\/(?:www\\.)?threads\\.com\\/@[^\\/]+\\/post\\/[^\\/?#]+\\/?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：展開 share 短網址": {
    "regex": "^https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/share\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Instagram：展開 instagr.am": {
    "regex": "^https?:\\/\\/(?:www\\.)?instagr\\.am\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Instagram：複製貼文、Reels 與 IGTV": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/(?:p|reel|reels|tv)\\/[A-Za-z0-9_-]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：複製限時動態與精選": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/stories\\/[A-Za-z0-9._]+\\/[0-9]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：複製個人檔案": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/(?!(?:p|reel|reels|tv|stories|share|explore|accounts|direct|about|legal|web|challenge|checkpoint|privacy|terms)(?:\\/|$))[A-Za-z0-9._]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Facebook：複製已驗證正式網址": {
    "regex": [
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/reel\\/[0-9]+\\/?(?:[?#].*)?$",
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/[^\\/?#]+\\/posts\\/[0-9]+\\/?(?:[?#].*)?$",
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/groups\\/[^\\/?#]+\\/posts\\/[0-9]+\\/?(?:[?#].*)?$"
    ],
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  }
}
```

## Threads 規則說明

`threads.com/share/...` 先執行 `checkStatus`。網址變成：

```text
https://www.threads.com/@帳號/post/貼文代碼
```

後，再執行：

```json
"action": [
  "copy",
  "close"
]
```

## Instagram 規則說明

Instagram 分成三種處理路徑：

1. `/share/` 或 `instagr.am` 短網址先執行 `checkStatus`。
2. 標準網址但帶 `igsh`、`igshid`、`utm_*`，先由 ClearURLs 清除。
3. 成為乾淨的貼文、Reels、限時動態或個人檔案網址後，執行 `copy`＋`close`。

## Facebook 規則說明

Facebook `/share/...` **不在 URLCheck Automations 裡執行 `checkStatus`**。MacroDroid 會先將它解析成正式網址，再寫回剪貼簿並啟動 URLCheck。

URLCheck 的 Facebook 規則只負責最後收尾，目前預設白名單僅包含已實測的：

```text
facebook.com/reel/數字ID/
facebook.com/帳號/posts/數字ID/
facebook.com/groups/社團ID/posts/貼文ID/
```

未驗證格式不應先加入寬鬆規則，避免登入頁、錯誤頁或平台功能頁覆寫剪貼簿。

## 為什麼 Threads／Instagram 的展開與複製要拆成不同規則？

`checkStatus` 是非同步網路操作。若寫成：

```json
"action": [
  "checkStatus",
  "copy",
  "close"
]
```

URLCheck 可能在重新導向完成前，先把原始短網址複製回剪貼簿。

所以必須：

```text
短網址 → checkStatus
正式網址 → copy＋close
```

Facebook 則改由 MacroDroid 完成 HTML 解析，不使用這條 `checkStatus` 展開路徑。

---

# 三、URLCheck 模組設定

## 狀態碼／Status code

進入：

```text
URLCheck
→ 模組
→ 狀態碼
```

確認開啟：

```text
狀態碼模組
自動取代為重新導向的目標網址
```

這是 Threads `/share/`、Instagram `/share/` 與 `instagr.am` 展開的核心設定；Facebook `/share/` 不依賴它。

## 還原縮址／Unshortener

建議關閉：

```text
模組\縮址還原器
```

原因：

- Threads 與 Instagram 已由 Status code 模組處理。
- Facebook 已由 MacroDroid HTTP 解析處理。
- Unshortener 會呼叫第三方服務，屬於重複處理。
- 可能出現「伺服器錯誤：Unknown Error」。

最終配置：

```text
網址清理器：開啟＋自動套用
狀態碼：開啟＋自動取代為重新導向的目標網址
縮址還原器：關閉
Automations：開啟
```

---

# 四、授予 MacroDroid Logcat 權限

Android 新版限制背景 App 直接監聽剪貼簿，因此 MacroDroid 使用：

```text
使用 Logcat 偵測（ADB Hack）
```

需要授予：

```text
android.permission.READ_LOGS
```

啟動 Shizuku，讓 aShell 取得 Shizuku 授權後，執行：

```sh
pm grant com.arlosoft.macrodroid android.permission.READ_LOGS
```

成功時通常不會顯示任何訊息。

需要重新授權的可能情況：

- 重裝 MacroDroid
- 清除 MacroDroid 資料
- 系統或權限設定被重設

---

# 五、MacroDroid：Threads 巨集

巨集名稱：

```text
Threads 追蹤碼移除
```

## 觸發器：剪貼簿變更

選擇：

```text
觸發器
→ 剪貼簿變更
```

剪貼簿文字：

```regex
^https?://(?:www\.)?threads\.com/share/.*$
```

勾選：

```text
啟用正規表達式匹配
使用 Logcat 偵測（ADB Hack）
```

「不區分大小寫」不必開。

這條規則只會偵測：

```text
https://www.threads.com/share/...
```

不會偵測處理後的：

```text
https://www.threads.com/@帳號/post/...
```

所以不會無限循環。

## 動作：傳送 Intent

選擇：

```text
動作
→ 傳送 Intent
```

設定如下：

目標：
```text
Activity
```

分類：
```text
None
```

動作：
```text
android.intent.action.VIEW
```

套件：
```text
com.trianguloy.urlchecker
```

類別：
```text
com.trianguloy.urlchecker.activities.ShortcutsActivity
```

---

# 六、MacroDroid：Instagram 巨集

最簡單的作法是直接複製 Threads 巨集，改名：

```text
Instagram 追蹤碼移除
```

Intent 動作完全不變，只替換剪貼簿觸發器的正規表達式。

## 觸發器：剪貼簿變更

剪貼簿文字：

```regex
^(?:https?://(?:(?:www|m)\.)?instagram\.com/(?:share/.*|.*[?&](?:igsh|igshid|utm_[^=&#\s]+)=[^#\s]*)|https?://(?:www\.)?instagr\.am/.*)$
```

勾選：

```text
啟用正規表達式匹配
使用 Logcat 偵測（ADB Hack）
```

這條規則只攔截：

- `instagram.com/share/...`
- `instagr.am/...`
- 帶 `igsh` 的 Instagram 網址
- 帶 `igshid` 的 Instagram 網址
- 帶任一 `utm_*` 的 Instagram 網址

它不會攔截一般乾淨網址，例如：

```text
https://www.instagram.com/reel/ABCDEFGHIJK/
```

因此 URLCheck 寫回乾淨網址後不會再次觸發 MacroDroid。

## 動作：傳送 Intent

與 Threads 巨集相同：

```text
目標：Activity
分類：None
動作：android.intent.action.VIEW
套件：com.trianguloy.urlchecker
類別：com.trianguloy.urlchecker.activities.ShortcutsActivity
```

`ShortcutsActivity` 是 URLCheck 內建的「檢查剪貼簿」入口。

MacroDroid 呼叫它後，URLCheck 會：

1. 取得目前剪貼簿內容
2. 找到其中的網址
3. 把網址送進 URLCheck 主介面
4. 執行 Automations

---

# 七、MacroDroid：Facebook 巨集

Facebook 不能直接複製 Threads 巨集只換正規表達式，因為 `/share/...` 多數情況回傳 `200 OK`，沒有可供 URLCheck `checkStatus` 使用的 `Location` 標頭。

巨集名稱建議：

```text
Facebook 追蹤碼移除
```

## 觸發器：剪貼簿變更

剪貼簿文字：

```regex
^https?://(?:(?:www|m|mbasic|web|touch)\.)?facebook\.com/share/(?:[prv]/)?[^/?#]+/?(?:[?#].*)?$
```

勾選：

```text
啟用正規表達式匹配
使用 Logcat 偵測（ADB Hack）
```

這條規則支援：

```text
facebook.com/share/p/...
facebook.com/share/r/...
facebook.com/share/v/...
facebook.com/share/...
```

目前不把 `fb.watch` 納入預設觸發器，因為尚未完成實測。

## 動作 0：初始化區域變數

在每次 HTTP 請求前，先設定：

```text
fb_html = 空字串
fb_result = 空字串
fb_encoded_next = 空字串
fb_status = 0
```

類型：

```text
fb_html：字串
fb_result：字串
fb_encoded_next：字串
fb_status：整數
```

全部使用「區域變數」。

這一步不可省略。若 Facebook 改版導致本次解析失敗，未清空的 `fb_result` 可能殘留上一次成功網址，造成錯貼舊連結。

## 動作 1：HTTP 請求

新增：

```text
網頁互動
→ HTTP 請求
```

### 設定分頁

```text
請求方法：GET
輸入網址：{clipboard}
阻擋後續動作直到完成：勾選
允許任何憑證：不要勾
跟隨重新導向：勾選
逾時：30 秒
基本授權：不要啟用
```

### 查詢參數／內容主體

保持空白。

### 標頭（header）參數

加入：

```text
User-Agent
Mozilla/5.0 (compatible; Discordbot/2.0; +https://discordapp.com)
```

以及：

```text
Accept-Language
zh-TW,zh;q=0.9,en;q=0.8
```

### 儲存回應

```text
HTTP 回傳碼 → fb_status（整數）
HTTP headers → 不儲存
HTTP 回應本文 → fb_html（字串）
```

## 動作 2：擷取 canonical

新增：

```text
文字操作
→ 擷取文字
```

來源文字必須用變數選單插入：

```text
{lv=fb_html}
```

不要填成普通文字 `fb_html`，也不要變成 `fb_html{lv=fb_html}`。

正規表達式：

```regex
(?is)<link\b(?=[^>]*\brel\s*=\s*["']canonical["'])[^>]*\bhref\s*=\s*["'](https?://[^"']+)["']
```

設定：

```text
擷取：群組 1
儲存至：fb_result
```

## 動作 3：canonical 為空時擷取 `og:url`

新增 If：

```text
如果 fb_result 等於空字串
```

比較值欄位完全留空，不要輸入 `""` 或空格。

「如果條件不符合，則不記錄條件失敗」在測試階段保持不勾，方便查看日誌。

在 If 內新增「擷取文字」：

來源：

```text
{lv=fb_html}
```

正規表達式：

```regex
(?is)<meta\b(?=[^>]*\bproperty\s*=\s*["']og:url["'])[^>]*\bcontent\s*=\s*["'](https?://[^"']+)["']
```

設定：

```text
擷取：群組 1
儲存至：fb_result
```

結構：

```text
如果 fb_result 為空
    擷取 og:url → fb_result
結束如果
```

## 動作 4：仍為空時擷取登入頁 `next=`

在上一個 End If 後，再新增第二個 If：

```text
如果 fb_result 等於空字串
```

在 If 內新增「擷取文字」：

來源：

```text
{lv=fb_html}
```

正規表達式：

```regex
(?i)(?:[?&]|&amp;)next=(https?%3A%2F%2F[^"'&<]+)
```

設定：

```text
擷取：群組 1
儲存至：fb_encoded_next
```

接著在同一個 If 內新增 JavaScript，將百分比編碼網址解碼。

### JavaScript：解碼 `next`

輸出變數：

```text
fb_result
```

主控台輸出字串變數：

```text
不要儲存
```

程式碼：

```javascript
let value = "{lv=fb_encoded_next}"
  .replace(/&amp;/g, "&")
  .trim();

for (let i = 0; i < 3 && value; i++) {
  try {
    const decoded = decodeURIComponent(value);

    if (decoded === value) {
      break;
    }

    value = decoded;
  } catch (error) {
    break;
  }
}

value;
```

結構：

```text
如果 fb_result 為空
    擷取 next → fb_encoded_next
    JavaScript 解碼 → fb_result
結束如果
```

## 動作 5：正規化正式網址

在兩個 If 全部結束後新增 JavaScript。

不要使用瀏覽器的 `URL` 物件；MacroDroid 的 JetPack JavaScript 引擎環境可能不提供它。

輸出變數：

```text
fb_result
```

主控台輸出字串變數：

```text
不要儲存
```

程式碼：

```javascript
let value = "{lv=fb_result}"
  .replace(/&amp;/g, "&")
  .trim();

// 統一 Facebook 網域
value = value.replace(
  /^https?:\/\/(?:(?:www|m|mbasic|web|touch)\.)?facebook\.com/i,
  "https://www.facebook.com"
);

// 將：帳號/posts/文字標題/數字ID/
// 縮成：帳號/posts/數字ID/
value = value.replace(
  /^(https:\/\/www\.facebook\.com\/[^/?#]+\/posts\/)(?:[^/?#]+\/)+([0-9]+)\/?(?:[?#].*)?$/i,
  "$1$2/"
);

// 移除常見分享／追蹤參數
value = value
  .replace(
    /([?&])(fbclid|mibextid|rdid|share_url|ref|refsrc|__cft__|__tn__|utm_[^=&#]+)=[^&#]*/gi,
    "$1"
  )
  .replace(/\?&/g, "?")
  .replace(/&&+/g, "&")
  .replace(/[?&]$/g, "");

value;
```

例如：

```text
https://www.facebook.com/PTS1997/posts/一大串百分比編碼標題/1463295339178218/
```

會整理成：

```text
https://www.facebook.com/PTS1997/posts/1463295339178218/
```

中間的百分比編碼原本是貼文標題 slug，不是追蹤碼；移除是為了讓網址更短且穩定。

## 動作 6：安全白名單判斷

新增 If，選擇 `fb_result`「符合正規表達式」：

```regex
^https://www\.facebook\.com/(?:reel/[0-9]+/?|[^/?#]+/posts/[0-9]+/?|groups/[^/?#]+/posts/[0-9]+/?)(?:[?#].*)?$
```

這份預設白名單只包含本輪已實測的三類：

```text
Reels
一般／粉專貼文
社團貼文
```

在 If 裡依序加入：

```text
填入剪貼簿：{lv=fb_result}
傳送 Intent：URLCheck ShortcutsActivity
```

Intent 設定：

```text
目標：Activity
分類：None
動作：android.intent.action.VIEW
套件：com.trianguloy.urlchecker
類別：com.trianguloy.urlchecker.activities.ShortcutsActivity
資料：留空
MIME 類型：留空
Flags：預設
```

URLCheck 會讀取新的正式網址，ClearURLs 再清理一次，最後由 Facebook Automation 執行 `copy`＋`close`。

## 動作 7：Else 失敗保護

Else 不在一般新增動作清單中。

請點一下或長按最後那個紫色 If：

```text
如果 fb_result 符合安全白名單
```

選擇：

```text
新增 Else 子句
```

介面也可能顯示：

```text
新增否則子句
```

在 Else 與 End If 之間加入「顯示通知」：

標題：

```text
Facebook 網址解析失敗
```

內容：

```text
未取得已驗證的正式網址，已保留原始分享網址。

HTTP 狀態碼：{lv=fb_status}
解析結果：{lv=fb_result}
```

Else 裡不要放：

```text
填入剪貼簿
啟動 URLCheck
copy
close
```

完整收尾結構：

```text
如果 fb_result 符合安全白名單
    填入剪貼簿：{lv=fb_result}
    傳送 Intent：URLCheck
否則
    顯示通知：Facebook 網址解析失敗，已保留原始網址
結束如果
```

## Facebook 巨集完整動作順序

```text
初始化 fb_html、fb_result、fb_encoded_next、fb_status
↓
HTTP GET {clipboard}
↓
擷取 canonical → fb_result
↓
如果 fb_result 為空
    擷取 og:url → fb_result
結束如果
↓
如果 fb_result 為空
    擷取 next → fb_encoded_next
    JavaScript 解碼 → fb_result
結束如果
↓
JavaScript 正規化 → fb_result
↓
如果 fb_result 符合已驗證白名單
    寫入剪貼簿
    啟動 URLCheck
否則
    通知解析失敗，保留原始短網址
結束如果
```

## 設定期間的測試對話框

正式使用前，可在所有 If 結束後暫時加入「顯示對話方塊」：

```text
HTTP 狀態碼：
{lv=fb_status}

canonical／og:url 結果：
{lv=fb_result}

next 原始值：
{lv=fb_encoded_next}
```

測試對話框：

- 取消「HTML 格式化」。
- 可勾「阻擋後續動作直到完成」。
- 確認流程正常後刪除，避免日常使用一直彈窗。

---

# 八、手機系統設定

為避免背景觸發失敗，建議設定：

## MacroDroid

```text
電池：不受限制
允許背景活動：開啟
顯示在其他應用程式上層：允許
```

## URLCheck

```text
電池：不受限制或最佳化
```

URLCheck 本身不是長期常駐，通常不一定需要設成不受限制；若偶爾叫不出來，再改為不受限制。

---

# 九、完整執行流程

## Threads

```text
1. 在 Threads 貼文按「分享」
2. 選擇「複製連結」
3. 剪貼簿得到 threads.com/share/...
4. MacroDroid 的 Threads 巨集觸發
5. MacroDroid 開啟 URLCheck ShortcutsActivity
6. URLCheck 讀取剪貼簿網址
7. Automation 執行 checkStatus
8. Threads 回傳正式貼文網址
9. Status code 自動重新導向
10. ClearURLs 移除 xmt、slof
11. 最終網址規則執行 copy＋close
12. 完整乾淨網址寫回剪貼簿
```

## Instagram：帶追蹤參數

```text
1. 在 Instagram 按「複製連結」
2. 剪貼簿得到帶 igsh／igshid／utm_* 的網址
3. MacroDroid 的 Instagram 巨集觸發
4. URLCheck 讀取剪貼簿網址
5. ClearURLs 自動移除追蹤參數
6. 最終網址規則執行 copy＋close
7. 乾淨網址寫回剪貼簿
```

## Instagram：分享短網址

```text
1. 剪貼簿得到 instagram.com/share/... 或 instagr.am/...
2. MacroDroid 的 Instagram 巨集觸發
3. URLCheck 執行 checkStatus
4. Status code 取得正式網址
5. ClearURLs 移除追蹤參數
6. 最終網址規則執行 copy＋close
7. 乾淨網址寫回剪貼簿
```

## Facebook：分享短網址

```text
1. 在 Facebook 貼文或 Reel 按「複製連結」
2. 剪貼簿得到 facebook.com/share/...
3. MacroDroid 的 Facebook 巨集觸發
4. 巨集清空上一次的區域變數
5. 以 Discordbot User-Agent 下載 HTML
6. 依序擷取 canonical、og:url、login next
7. 正規化網域、移除貼文標題 slug 與追蹤參數
8. 若結果符合已驗證白名單，寫回剪貼簿
9. 啟動 URLCheck
10. ClearURLs 再清理一次
11. URLCheck Automation 執行 copy＋close
```

若解析失敗或結果不在白名單內：

```text
不覆寫剪貼簿
不啟動 URLCheck
顯示失敗通知
原始 facebook.com/share/... 保留在剪貼簿
```

---

# 十、測試方式

## Threads

複製一篇 Threads 貼文連結，等待 URLCheck 跳出並自動關閉後，到任意輸入框貼上。

應該得到：

```text
https://www.threads.com/@帳號/post/貼文代碼
```

而不是：

```text
https://www.threads.com/share/xxxxxxxx
```

## Instagram 貼文／Reels

原本：

```text
https://www.instagram.com/reel/ABCDEFGHIJK/?igsh=xxxxxxxx
```

處理後：

```text
https://www.instagram.com/reel/ABCDEFGHIJK/
```

## Facebook：一般／粉專貼文

已實測輸出：

```text
https://www.facebook.com/帳號/posts/數字貼文ID/
```

若 canonical 含長標題：

```text
https://www.facebook.com/帳號/posts/百分比編碼標題/數字貼文ID/
```

正規化後應縮成：

```text
https://www.facebook.com/帳號/posts/數字貼文ID/
```

## Facebook：Reels

已實測輸出：

```text
https://www.facebook.com/reel/數字ID/
```

## Facebook：社團貼文

已實測輸出：

```text
https://www.facebook.com/groups/社團ID/posts/貼文ID/
```

## Facebook：失敗安全測試

暫時把安全白名單改成不可能符合的內容，或測試尚未支援的公開格式。應該：

```text
顯示「Facebook 網址解析失敗」通知
剪貼簿仍保留原始 /share/ 網址
URLCheck 不啟動
```

完成後務必恢復正式白名單。

---

# 十一、問題排查

## 1. 複製連結後完全沒反應

檢查：

```text
MacroDroid 對應巨集是否啟用
剪貼簿觸發器是否勾選正規表達式
是否勾選「使用 Logcat 偵測」
READ_LOGS 是否已授予
MacroDroid 是否允許背景活動
```

重新授權：

```sh
pm grant com.arlosoft.macrodroid android.permission.READ_LOGS
```

## 2. Threads／Instagram share 網址沒有展開

檢查：

```text
Automations 總開關是否開啟
Status code 模組是否開啟
自動重新導向是否開啟
對應規則 action 是否為 checkStatus
```

Facebook 不適用這一項；Facebook 應檢查 MacroDroid HTTP 流程。

## 3. Facebook HTTP 狀態碼是 200，但仍停在 `/share/`

這正是不用 URLCheck `checkStatus` 的原因。請確認：

```text
HTTP 回應本文已儲存到 fb_html
User-Agent 已設成 Discordbot
canonical 來源為 {lv=fb_html}
canonical 正規表達式沒有貼錯
```

## 4. `fb_result` 一直是空白

依序檢查：

```text
HTTP 動作是否勾「阻擋後續動作直到完成」
fb_html 是否為字串變數
來源文字是否只有 {lv=fb_html}
canonical／og:url 是否選「群組 1」
輸出變數是否為 fb_result
```

可暫時重新加入測試對話框查看 `fb_status`、`fb_result`、`fb_encoded_next`。

## 5. Facebook 能解析，但網址仍有超長標題

確認「正規化 fb_result」JavaScript：

- 位於兩個解析 If 全部結束後。
- 輸出變數是 `fb_result`。
- 使用本文件的純正規表達式版本，不使用 `new URL(...)`。

## 6. Facebook 解析成功但沒有啟動 URLCheck

檢查安全白名單是否涵蓋目前路徑。

預設只支援：

```text
/reel/數字ID/
/帳號/posts/數字ID/
/groups/社團ID/posts/貼文ID/
```

遇到其他正常公開格式時，先記錄實際 `fb_result`，充分測試後再精確新增，不要改成所有 `facebook.com` 都通過。

## 7. Facebook 改版後開始失敗

目前解析順序：

```text
canonical
→ og:url
→ login next
```

若三者都失效：

1. 保留原始分享網址。
2. 查看 `fb_html` 是否仍含其他正式網址欄位。
3. 更新擷取規則。
4. 不要移除安全白名單或失敗 Else。

## 8. 出現「還原縮址：伺服器錯誤」

關閉：

```text
Unshortener／還原縮址
```

## 9. ClearURLs JSON 無法儲存

常見錯誤：

```text
少了 ]
少了 }
兩個物件之間少了逗號
自訂規則被放在最外層物件之外
```

---

# 十二、備份用設定

## GitHub 維護檔案與 Raw 位址

| 用途 | Repository 路徑 | Raw 位址 |
|---|---|---|
| 網址清理器自訂規則 | `urlcheck/custom-rules.json` | `https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.json` |
| 自訂規則 SHA-256 | `urlcheck/custom-rules.sha256` | `https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.sha256` |
| Automations 完整設定 | `urlcheck/automations.json` | `https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/automations.json` |
| Automations SHA-256 | `urlcheck/automations.sha256` | `https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/automations.sha256` |

`.github/workflows/update-urlcheck-hashes.yml` 會在 `urlcheck/*.json` 變更時驗證 JSON，並重新計算相應的 `.sha256` 檔案。

目前工作流程只負責：

```text
驗證 JSON 格式
重新計算 SHA-256
提交變更後的雜湊檔
```

它目前不會下載或合併 ClearURLs 官方完整目錄，因此不能解決官方 `"providers"` 自動同步問題。

```text
custom-rules.json → 填入網址清理器更新器，以合併模式更新「自訂規則」
automations.json  → 手動貼入 Automations JSON 編輯器
*.sha256          → 驗證對應 JSON 的內容完整性
```

## URLCheck ClearURLs 自訂規則

```json
"自訂規則": {
  "Threads": {
    "urlPattern": "^https?://(?:www\\.)?threads\\.com",
    "rules": [
      "xmt",
      "slof"
    ]
  },
  "Instagram": {
    "urlPattern": "^https?://(?:(?:www|m)\\.)?instagram\\.com",
    "rules": [
      "igsh",
      "igshid"
    ]
  },
  "Facebook": {
    "urlPattern": "^https?://(?:(?:(?:www|m|mbasic|web|touch|l|lm)\\.)?facebook\\.com|(?:www\\.)?fb\\.watch)",
    "rules": [
      "fbclid",
      "mibextid",
      "rdid",
      "share_url",
      "ref",
      "refsrc",
      "__cft__",
      "__tn__"
    ]
  }
}
```

注意：這一段是手動編輯完整目錄時使用的片段。GitHub 上的 `custom-rules.json` 雖然是有效 JSON，但只包含頂層 `"自訂規則"`，應交給更新器合併，不能拿去取代 JSON 編輯器中的完整目錄。

## URLCheck Automations 完整設定

```json
{
  "Threads：展開短網址": {
    "regex": "^https?:\\/\\/(?:www\\.)?threads\\.com\\/share\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Threads：複製展開後網址": {
    "regex": "^https?:\\/\\/(?:www\\.)?threads\\.com\\/@[^\\/]+\\/post\\/[^\\/?#]+\\/?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：展開 share 短網址": {
    "regex": "^https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/share\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Instagram：展開 instagr.am": {
    "regex": "^https?:\\/\\/(?:www\\.)?instagr\\.am\\/.*",
    "action": "checkStatus",
    "enabled": true,
    "stop": true
  },
  "Instagram：複製貼文、Reels 與 IGTV": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/(?:p|reel|reels|tv)\\/[A-Za-z0-9_-]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：複製限時動態與精選": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/stories\\/[A-Za-z0-9._]+\\/[0-9]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Instagram：複製個人檔案": {
    "regex": "^(?!.*[?&](?:igsh|igshid|utm_[^=&#]+)=)https?:\\/\\/(?:(?:www|m)\\.)?instagram\\.com\\/(?!(?:p|reel|reels|tv|stories|share|explore|accounts|direct|about|legal|web|challenge|checkpoint|privacy|terms)(?:\\/|$))[A-Za-z0-9._]+\\/?(?:\\?[^#]*)?$",
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  },
  "Facebook：複製已驗證正式網址": {
    "regex": [
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/reel\\/[0-9]+\\/?(?:[?#].*)?$",
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/[^\\/?#]+\\/posts\\/[0-9]+\\/?(?:[?#].*)?$",
      "^(?!.*[?&](?:fbclid|mibextid|rdid|share_url|utm_[^=&#]+)=)https?:\\/\\/(?:www\\.)?facebook\\.com\\/groups\\/[^\\/?#]+\\/posts\\/[0-9]+\\/?(?:[?#].*)?$"
    ],
    "action": [
      "copy",
      "close"
    ],
    "enabled": true,
    "stop": true
  }
}
```

## MacroDroid Threads 剪貼簿正規表達式

```regex
^https?://(?:www\.)?threads\.com/share/.*$
```

## MacroDroid Instagram 剪貼簿正規表達式

```regex
^(?:https?://(?:(?:www|m)\.)?instagram\.com/(?:share/.*|.*[?&](?:igsh|igshid|utm_[^=&#\s]+)=[^#\s]*)|https?://(?:www\.)?instagr\.am/.*)$
```

## MacroDroid Facebook 剪貼簿正規表達式

```regex
^https?://(?:(?:www|m|mbasic|web|touch)\.)?facebook\.com/share/(?:[prv]/)?[^/?#]+/?(?:[?#].*)?$
```

## Facebook canonical 正規表達式

```regex
(?is)<link\b(?=[^>]*\brel\s*=\s*["']canonical["'])[^>]*\bhref\s*=\s*["'](https?://[^"']+)["']
```

## Facebook `og:url` 正規表達式

```regex
(?is)<meta\b(?=[^>]*\bproperty\s*=\s*["']og:url["'])[^>]*\bcontent\s*=\s*["'](https?://[^"']+)["']
```

## Facebook `next=` 正規表達式

```regex
(?i)(?:[?&]|&amp;)next=(https?%3A%2F%2F[^"'&<]+)
```

## Facebook 安全白名單

```regex
^https://www\.facebook\.com/(?:reel/[0-9]+/?|[^/?#]+/posts/[0-9]+/?|groups/[^/?#]+/posts/[0-9]+/?)(?:[?#].*)?$
```

## MacroDroid Intent

```text
目標：Activity
分類：None
動作：android.intent.action.VIEW
套件：com.trianguloy.urlchecker
類別：com.trianguloy.urlchecker.activities.ShortcutsActivity
資料：留空
MIME 類型：留空
Flags：預設
```

## MacroDroid Logcat 權限

```sh
pm grant com.arlosoft.macrodroid android.permission.READ_LOGS
```

---

# 十三、未來可能需要修改的地方

## Threads

目前短網址：

```text
threads.com/share/...
```

目前正式網址：

```text
threads.com/@帳號/post/貼文代碼
```

若 Threads 改變 `/share/` 或 `/post/` 路徑，要同步修改 MacroDroid 與 URLCheck Automations。

## Instagram

目前需要攔截的特徵：

```text
/share/
instagr.am
igsh
igshid
utm_*
```

若 Instagram 更換分享網域、追蹤參數或正式內容路徑，需要同步更新：

1. ClearURLs 自訂規則。
2. URLCheck Automations。
3. MacroDroid Instagram 剪貼簿正規表達式。

## Facebook

目前處理的分享路徑：

```text
facebook.com/share/p/...
facebook.com/share/r/...
facebook.com/share/v/...
facebook.com/share/...
```

目前預設安全白名單：

```text
facebook.com/reel/數字ID/
facebook.com/帳號/posts/數字ID/
facebook.com/groups/社團ID/posts/貼文ID/
```

Facebook 改版時，最可能需要更新：

1. MacroDroid Facebook 剪貼簿觸發器。
2. HTTP User-Agent 或 Header。
3. `canonical`／`og:url`／`next=` 擷取正規表達式。
4. JavaScript 正規化規則。
5. MacroDroid 安全白名單。
6. URLCheck Facebook 最終網址 Automation。

請保留下列防護：

```text
每次執行先清空區域變數
只有白名單命中才覆寫剪貼簿
解析失敗時執行 Else 通知
```

這能讓 Facebook 改版後「停止自動處理」，而不是「寫入錯誤或上一筆網址」。

---

# 十四、Repository 更新流程

## 修改追蹤參數

編輯 `urlcheck/custom-rules.json`。GitHub Actions 會重新計算 `urlcheck/custom-rules.sha256`。已啟用 URLCheck 自動更新的裝置，之後會在週期檢查時取得新版 `"自訂規則"`，也可以手動按「立即更新」。

## 修改 URLCheck Automations

編輯 `urlcheck/automations.json`。GitHub Actions 會重新計算 `urlcheck/automations.sha256`。原版 URLCheck 3.5 不會自動拉取 Automations，每次修改後仍須在手機重新貼入完整 JSON。

## 修改 MacroDroid 巨集

`.macro` 檔位於 `MacroDroid腳本/`，不會因為 GitHub 更新而自動套用到手機。更新後需要在 MacroDroid 手動匯入，或依 README 修改現有巨集。

## 避免破壞更新相容性

```text
custom-rules.json 最外層只使用「自訂規則」
不要在遠端自訂檔加入 providers
Automations 必須保存為完整 JSON
每次變更後確認對應 SHA-256 已同步
```

若未來要讓官方 `"providers"` 與自訂規則完全自動更新，應新增工作流程：

```text
定期下載 ClearURLs 官方目錄
→ 合併 urlcheck/custom-rules.json
→ 輸出完整合併版 JSON
→ 計算合併版 SHA-256
→ 讓 URLCheck 使用合併版 Raw 位址
```

目前尚未實作這個官方目錄鏡像流程。
