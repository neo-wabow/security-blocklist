# 安全黑名單記錄

用於記錄經確認的封鎖決策，供未來主機同步使用。此為私有 repository。

## 目前狀態

- `blocklist.json`：正式名單，初始為空。
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
