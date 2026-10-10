# 安全黑名單記錄

用於記錄經確認的封鎖決策，供未來主機同步使用。此為私有 repository。

## 目前狀態

- `blocklist.json`：正式名單，初始為空。
- `blocklist.txt`：簡易 IP 名單，一行一個 IP，初始為空；尚未設定同步或套用。
- 尚未接入 Apache、Cloudflare/WAF 或其他主機；提交檔案本身不會執行封鎖。
- IP 的惡意判定、實際封鎖及黑名單更新是不同動作。巡檢發現不自動成為封鎖項目。

## 名單格式

頂層欄位為 `schema_version`、遞增整數 `revision`、UTC ISO 8601 `updated_at` 與 `entries`。
每筆 entry 使用以下欄位；此範例只是格式示意，不在正式名單中：

```json
{
  "id": "example-001",
  "ip": "198.51.100.23",
  "hosts": ["api.example.com"],
  "methods": ["GET"],
  "path": "/v1/members/orders/12345",
  "path_match": "exact",
  "action": "http_403",
  "created_at": "2026-10-10T01:00:00Z",
  "expires_at": "2026-10-13T01:00:00Z",
  "reason": "經核對的訂單編號探測",
  "evidence_ref": "事件紀錄編號",
  "approved_by": "授權者識別"
}
```

- 初版只描述個別 IPv4/IPv6，不以國家或機房 ASN 自動加入整段封鎖。
- `hosts` 必須明確列出；`methods` 與 `path` 為限定範圍，省略兩者才代表該 host 的全部請求。`path_match` 目前只接受 `exact`，不把 path 當正規表示式或模糊比對。
- `expires_at` 必須明確填寫；到期項目失效，即使下載失敗仍須停止套用到期封鎖。
- 理由需區分觀察、使用者回報及實際驗證，不將 HTTP 200/201 當成入侵或資料外洩證明。
- 不提交密碼、token、Cookie、會員個資或含敏感參數的原始日誌。

## 變更記錄

每次增刪或調整以新的 commit 記錄，更新 `revision` 與 `updated_at`，commit message 說明來源、理由及期限。撤銷亦使用新 commit，保留歷史供追溯。
Git 提交歷史可被改寫；目前尚未設定分支保護，不能當作不可竄改稽核。

## 後續主機同步與權限

尚未實作同步程式。預定以 GitHub Contents API 讀取 `main` 分支的 `blocklist.json`：寫入端使用只限本 repo 的 Contents read/write token；主機使用另一組 Contents read-only token。
主機先驗證格式、IP、範圍及期限，再原子套用；失敗保留最近有效版本，不能任意擴大封鎖。同步程式需另經授權部署，並核對可信代理與真實來源 IP。

## 一行一個 IP 的文字檔

`blocklist.txt` 每行僅填完整 IPv4 或 IPv6；不放註解、CIDR、網址、理由或範例 IP，重複 IP 應去重。合法空名單是 0 bytes；下載程式需核對 HTTP 成功狀態及內容格式，不能把下載失敗當空名單。

文字檔不包含 host、HTTP method、path 或 expires_at，不能完整表達 JSON 的限定規則。僅在明確核准的套用範圍內使用；不可直接把限定網址的 JSON 條目轉成整站封鎖。到期資訊仍需由 JSON 或主機持久紀錄保存，不能從純 IP 檔推算期限。

主機處理方式尚未部署：下載暫存文字檔，驗證，轉成 Apache 設定，configtest 通過且內容有變才 graceful reload。暫存下載檔成功轉換後可刪除，GitHub 名單及主機正式設定須保留。Apache 不會直接把純 IP 行當成設定指令。

若正式檔被 Apache Include 或 RewriteMap 引用，不能讀完就刪除；下次載入或請求可能需要再次讀檔。Include 檔缺失會使設定載入失敗；IncludeOptional 在支援的版本／匹配方式下忽略缺檔則可能漏套封鎖。不能把「刪除名單」當作已處理的 ACK，也不能讓第一台主機抓完就刪掉 GitHub 檔，否則其他節點拿不到相同版本。
