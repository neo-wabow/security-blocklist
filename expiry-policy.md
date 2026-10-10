# 到期移除規則

使用者於 2026-10-10 授權：到期直接從 IP 名單移除。沿用既有 wazuh 半小時 heartbeat，每輪另外核對本私有 repo；不建立新的巡檢排程。

以 `blocklist.json` 的明確 `expires_at` 為權威。使用含時區的 ISO 8601 時間，轉為 UTC 比較；到期時間小於或等於當下才移除。缺少時間、時間格式錯誤、回報期限而無實際起點者，不猜測到期，保留並回報待處理。

只移除已到期条目，不新增封鎖、不延長期限、不擴大範圍。若同 IP 還有另一筆未到期條目，保留該筆有效規則；不能整個 IP 一起刪。

JSON、blocklist.txt、當日 `ao_blacklist/YYYY/YYYY-MM-DD.txt` 與 `security-blocklist.conf` 在同一提交更新，JSON 的 revision 加一並更新 updated_at。TXT 僅刪去不再有適用有效條目的 IP；conf 僅刪對應到期規則，保留其他規則與既有 host/method/path 範圍。未建立JSON對應的TXT或conf規則不可臆測期限或直接刪除；先回報不一致。沒有到期項目且當日派生檔已存在並一致時，不產生空提交。

每次讀取 main 最新狀態，基於同一個 commit 判斷與準備檔案。提交前再確認分支未变，正常 fast-forward、force=false 更新；如有並行變更則重新讀取與核對，不覆蓋或 force push。成功後核對 main 及三個檔案，留存移除條目 ID、IP、到期時間與 commit SHA；記錄以 Git 歷史追溯。

這是 GitHub repo 到期刪除的狹窄寫入授權。Wazuh/SSH 巡檢仍唯讀，不在主機直接改檔、封鎖、解封、reload 或修改 Cloudflare/WAF。主機需同步新的 conf 並成功 reload，實際封鎖才會解除；不能把 repo 提交當成已解除。半小時排程為下一輪處理，執行/連線失敗可能更晚，並非精準到秒。主機同步尚未部署。

保持無變更時安靜；完成到期清除或維護失敗需要注意時才在巡檢對話通知，失敗不得宣稱完成。使用本機既有 gh 憑證，不輸出 token。每日巡檢接班需保留此維護授權及證據路徑。


## 每日派生檔例外（2026-10-10 新增授權）

使用者指定 `ao_blacklist/2026/2026-10-10.txt` 格式，並確認內容為當下仍有效的全部IP。以Asia/Taipei日期建立或同步當日TXT，一行一個IP，去重排序；由已核准且明確有效的JSON資料派生，包含前日核准未到期者。不可把監控發現自行新增成正式封鎖，也不可猜測缺失期限；異常或獨立未對應規則保留並報告。

換日第一個成功巡檢輪次可提交新日期檔；同日不變不提交。新日期派生檔本身未改名單決策，不增加JSON revision。若同轮移除到期項目，JSON、相容TXT、conf、當日TXT同一commit處理。保留舊日期檔作歷史快照，不能合併為目前有效名單。`blocklist.txt`維持當日名單相容副本；任何不一致需核對最新main與正式JSON，不盲目覆蓋人工改動。

TXT不含host/method/path，主機不得因此擴大封鎖範圍。此派生檔維護仍不授權主機寫檔、reload、WAF/IP封鎖或新增有效条目；沿用同一wazuh heartbeat。
