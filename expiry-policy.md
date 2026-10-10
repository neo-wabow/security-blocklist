# 黑名單維護規則（TXT-only，2026-10-10最新指示）

使用者已授權本輪整理台北日期2026-10-10截至查詢截止的防護IP，寫入ao_blacklist/2026/2026-10-10.txt。一行一個完整IPv4/IPv6、去重排序、無註解、理由、CIDR或示意IP。blocklist.txt為相容副本，兩TXT同一提交。

使用者最新指示：「先不要到期時間，JSON跟Apache conf可以先不做」「到期時間系統維運那邊會處理」。此指示優先於先前JSON權威及repo到期刪除流程。blocklist.json與security-blocklist.conf保持原狀，既有半小時巡檢不依JSON空名單清空TXT，不猜期限、不在repo自動到期刪除，不把目前TXT判成無JSON對應異常。實際期限、套用範圍、解除與狀態由維運系統管理，跨日出現同IP不代表新的封鎖或續期。

本輪寫入是使用者明確授權的一次今日整理，並非巡檢可永久自動新增IP。之後每輪讀最新政策、當日與相容TXT，已存在且相同不提交；不自行加IP、回補舊IP、填期限、續期、改JSON/conf、改主機或WAF。換日第一個成功輪次可將最近已授權TXT原樣帶到新日期檔，保留舊歷史快照；它是同一批維運維護名單，不由repo判定各IP是否仍有效。若當日或相容檔已被人工修改／移除，保留最新變動；不一致重讀核對，不盲目覆蓋或合併舊快照。

每次寫入以最新main為基礎、提交前重核分支、force=false，並行變更重讀；成功核對TXT及commitSHA並保存本機證據。無變化保持安靜；維護失敗需要注意時回報。repo是public，主機原始TXT免key下載；寫入仍需既有gh憑證且不輸出token。不得把token、Cookie、會員個資或原始日誌提交publicrepo。

主機同步尚未部署。repo寫入不等於已阻擋、解封或reload；Wazuh/SSH/Apache/Cloudflare/WAF仍唯讀。沿用同一wazuh heartbeat，不增排程；每日接班保留此最新TXT-only模式。
