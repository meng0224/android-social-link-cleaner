# URLCheck 遠端設定來源

本目錄保存 URLCheck 使用的遠端 JSON。

## 檔案

- `custom-rules.json`：網址清理器的自訂規則。
- `custom-rules.sha256`：上述檔案的 SHA-256。
- `automations.json`：URLCheck Automations 完整設定。
- `automations.sha256`：上述檔案的 SHA-256，供未來更新器或外部工具驗證。

## 網址清理器：可由 URLCheck 原生自動更新

進入：

```text
URLCheck
→ 模組
→ 網址清理器
→ 更新
```

填入：

```text
規則網址：
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.json

雜湊網址：
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/custom-rules.sha256
```

開啟：

```text
自動更新
```

目前 URLCheck 會將下載 JSON 的最外層物件合併到既有 catalog，因此這份檔案只提供 `自訂規則`；原本官方 `providers` 會保留。

URLCheck 目前的自動檢查週期為一週。也可以在同一畫面按「立即更新」。

## Automations：現版 URLCheck 尚不能直接遠端更新

遠端來源：

```text
https://raw.githubusercontent.com/meng0224/android-social-link-cleaner/main/urlcheck/automations.json
```

目前 URLCheck 3.5 的 Automations 頁面只有 JSON 編輯器，沒有遠端 URL、雜湊或自動更新設定。因此這份檔案目前是倉庫中的單一真實來源，但手機端仍需手動複製到：

```text
URLCheck
→ 自動化／Automations
→ JSON 編輯器
```

若要讓 Automations 也像網址清理器一樣直接更新，必須使用加入遠端更新器的 URLCheck 修改版，或由具備權限的外部工具寫入 URLCheck 的內部 `automations` 檔案。

## 維護方式

修改任何 `urlcheck/*.json` 後，GitHub Actions 會：

1. 驗證 JSON 語法。
2. 重新產生對應的 `.sha256`。
3. 雜湊有變動時自動提交。

請不要手動把雜湊與 JSON 分開維護。
