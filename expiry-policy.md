# 到期移除規則

使用者於 2026-10-10 授權：到期直接從 IP 名單移除。沿用既有 wazuh 半小時 heartbeat，每輪另外核對本私有 repo；不建立新的巡檢排程。

以 `blocklist.json` 的明確 `expires_at` 為權威。使用含時區的 ISO 8601 時間，轉為 UTC 比較；到期時間小於或等於當下才移除。缺少時間、時間格式錯誤、回報期限而無實際起點者，不猜測到期，保留並回報待處理。

只移除已到期条目，不新增封鎖、不延長期限、不擴大範圍。若同 IP 還有另一筆未到期條目，保留該筆有效規則；不能整個 IP 一起刪。

JSON、TXT 與 `security-blocklist.conf` 在同一提交更新，JSON 的 revision 加一並更新 updated_at。TXT 僅刪去不再有適用有效條目的 IP；conf 僅刪對應到期規則，保留其他規則與既有 host/method/path 範圍。未建立JSON對應的TXT或conf規則不可臆測期限或直接刪除；先回報不一致。沒有到期項目則不產生空提交。

每次讀取 main 最新狀態，基於同一個 commit 判斷與準備檔案。提交前再確認分支未变，正常 fast-forward、force=false 更新；如有並行變更則重新讀取與核對，不覆蓋或 force push。成功後核對 main 及三個檔案，留存移除條目 ID、IP、到期時間與 commit SHA；記錄以 Git 歷史追溯。

這是 GitHub repo 到期刪除的狹窄寫入授權。Wazuh/SSH 巡檢仍唯讀，不在主機直接改檔、封鎖、解封、reload 或修改 Cloudflare/WAF。主機需同步新的 conf 並成功 reload，實際封鎖才會解除；不能把 repo 提交當成已解除。半小時排程為下一輪處理，執行/連線失敗可能更晚，並非精準到秒。主機同步尚未部署。

保持無變更時安靜；完成到期清除或維護失敗需要注意時才在巡檢對話通知，失敗不得宣稱完成。使用本機既有 gh 憑證，不輸出 token。每日巡檢接班需保留此維護授權及證據路徑。
