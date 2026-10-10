# 安全黑名單記錄

用於記錄經確認的封鎖決策，供未來主機同步使用。此為 public repository，主機讀取不需要GitHub token；寫入仍須已授權憑證。

## 目前狀態

- `blocklist.json`：先前結構化格式，現在暫停同步，仍為空；目前請讀每日TXT。
- `security-blocklist.conf`：供 Apache 直接 Include 的設定檔；目前只有註解，無封鎖規則。
- `ao_blacklist/YYYY/YYYY-MM-DD.txt`：依 Asia/Taipei 日期維護的當下有效全部 IP，一行一個 IP；同日持續更新，當日TXT已填入經核對的防護來源；期限與主機套用由維運系統處理。
- `blocklist.txt`：相容既有讀取方式的名單，與當日檔同一提交同步；尚未設定主機套用。
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

尚未實作同步程式。主機可直接以 GitHub Contents API 讀取 `main` 分支的 `security-blocklist.conf`，省略主機端格式轉換：寫入端使用只限本 repo 的 Contents read/write token；主機使用另一組 Contents read-only token。
主機先驗證格式、IP、範圍及期限，再原子套用；失敗保留最近有效版本，不能任意擴大封鎖。同步程式需另經授權部署，並核對可信代理與真實來源 IP。

## 一行一個 IP 的文字檔

`blocklist.txt` 每行僅填完整 IPv4 或 IPv6；不放註解、CIDR、網址、理由或範例 IP，重複 IP 應去重。合法空名單是 0 bytes；下載程式需核對 HTTP 成功狀態及內容格式，不能把下載失敗當空名單。

文字檔不包含 host、HTTP method、path 或 expires_at，不能完整表達 JSON 的限定規則。僅在明確核准的套用範圍內使用；不可直接把限定網址的 JSON 條目轉成整站封鎖。到期資訊仍需由 JSON 或主機持久紀錄保存，不能從純 IP 檔推算期限。

主機處理方式尚未部署。推薦直接下載 repo 已轉好的 `security-blocklist.conf`，核對成功下載後套用；若選用 TXT，則仍需先驗證並轉成 Apache 設定。configtest 通過且內容有變才 graceful reload。暫存下載檔成功轉換後可刪除，GitHub 名單及主機正式設定須保留。Apache 不會直接把純 IP 行當成設定指令。

若正式檔被 Apache Include 或 RewriteMap 引用，不能讀完就刪除；下次載入或請求可能需要再次讀檔。Include 檔缺失會使設定載入失敗；IncludeOptional 在支援的版本／匹配方式下忽略缺檔則可能漏套封鎖。不能把「刪除名單」當作已處理的 ACK，也不能讓第一台主機抓完就刪掉 GitHub 檔，否則其他節點拿不到相同版本。

## Apache 直接 Include

`security-blocklist.conf` 是維護端事先產生的 Apache 設定；主機僅需下載、檢查與載入。名單增刪時，JSON 與 conf 必須在同一提交更新，保留核准的 host、method、path 範圍，不能僅依 TXT 猜測或擴大封鎖。現有 JSON 與 TXT 均為空，因此 conf 只有註解，不啟用 RewriteEngine，也不放示意 IP 的有效規則。

先把檔案下載至主機持久路徑（以下為建議路徑，尚未安裝），在要套用的既有 `<VirtualHost>` 內、既有 RewriteRule 前加入：

```apache
Include /etc/apache2/security-blocklist.conf
```

Include 只能讀本機設定路徑，不能直接抓 GitHub HTTPS URL。私有 repo 由同步程式使用專用唯讀 token 下載原始檔；token 不放在 URL 或 repo。

變更流程：下載至暫存、檢查 HTTP 成功與預期檔案格式、確認內容是否有變；有變時鎖定更新、備份目前檔案、原子替換正式檔，執行整份 Apache `configtest`。通過才 graceful reload，失敗則原子還原。若 reload 失敗也要保留錯誤與還原紀錄；只刪下載暫存，不刪正式 Include 檔。套用需核對實際 Apache 版本、mod_rewrite 是否可用及 REMOTE_ADDR 的可信代理配置。

此 repo 目前沒有主機部署；使用者已授權既有 wazuh 半小時 heartbeat 到期移除，後續新增有效規則仍需授權。Apache 不會因 repo 提交而自行更新，也不會僅讀取期限註解就自動解封。現在選用「到期後下一輪從 repo 移除，主機同步新 conf 並 reload」流程；尚不保證斷網或排程失敗時仍準時解除。若需要精準時間或離線到期保障，需另加本地到期處理。

參考：
- https://httpd.apache.org/docs/2.4/mod/core.html#include
- https://httpd.apache.org/docs/2.4/programs/apachectl.html

## 到期維護

使用者已授權到期直接移除，由既有半小時巡檢附帶執行；三份名單／設定同一提交更新，只刪到期條目，不自動新增或續期。詳見 [expiry-policy.md](expiry-policy.md)。主機同步與 reload 尚未部署，repo 移除不代表主機已解封。

## 每日 IP 檔案（2026-10-10 使用者確認）

主機每日讀取路徑 `ao_blacklist/YYYY/YYYY-MM-DD.txt`，日期使用 Asia/Taipei。例如 `ao_blacklist/2026/2026-10-10.txt`。

- 內容是當下仍有效的全部 IP，包含前幾天核准而尚未到期的條目，不只當天新增。
- 同日檔案會隨已授權名單變動或到期移除更新；一行一個完整 IPv4/IPv6，去重並排序，無註解、CIDR、JSON、網址或示意 IP。空名單為0 bytes。
- 正式決策與期限仍以 `blocklist.json` 為權威。TXT只列IP，不含host/method/path範圍；不得把限定範圍的条目直接轉成整站或所有主機封鎖。
- 每輪既有半小時巡檢檢查當日檔案；換日第一個成功輪次建立新日期檔。不增加排程，也不承諾午夜即已產生。
- 過去日期的檔案保留為最後一次寫入的歷史快照；它們不代表現在有效名單。主機應讀今天的檔，不能任意合併歷史檔或將過期快照當正式名單。
- 建立或同步每日派生檔不是新增封鎖授權；巡檢發現IP不會自動加進JSON。只有來源資料與格式都核對成功才更新。
- 到期變動時，JSON（revision/updated_at）、blocklist.txt、security-blocklist.conf與當日TXT必須同一提交更新。只有建立新日期派生檔且名單決策未變時，不需要為此改JSON revision。
- 下載必須確認HTTP成功與合法IP內容。404、403、網路失敗等不是空名單；不可因此清除現有設定。主機同步與本地到期處理目前尚未部署。

### Apache 主機讀取權限（先前私有模式）

使用獨立的GitHub fine-grained personal access token：Resource owner為`neo-wabow`，Repository access選Only select repositories，只選`security-blocklist`；Repository permissions只需Contents: Read-only（Metadata唯讀依GitHub自動包含）。不使用維護端目前的廣權限gh憑證，也不把token寫入repo、URL或Apache Include檔。

透過GitHub Contents API取得原始文字：

`GET https://api.github.com/repos/neo-wabow/security-blocklist/contents/ao_blacklist/2026/2026-10-10.txt?ref=main`

Headers：`Authorization: Bearer <主機專用唯讀TOKEN>`、`Accept: application/vnd.github.raw+json`。實際日期由主機以Asia/Taipei計算。這裡是格式範例，沒有存放真實token。

官方參考：
- https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens
- https://docs.github.com/en/rest/repos/contents#get-repository-content


## Public 免 key 讀取（2026-10-10 使用者授權）

本repo改為public。主機讀每日TXT不需要API key；維護端寫入仍使用既有GitHub憑證。前述私有模式的token步驟僅供未來改回private時參考，現在不需要建立主機讀取PAT。

今天的原始文字下載網址：

https://raw.githubusercontent.com/neo-wabow/security-blocklist/main/ao_blacklist/2026/2026-10-10.txt

主機以Asia/Taipei日期替換年份與日期；只下載當日檔，不合併歷史快照。成功的空檔為0 bytes；HTTP錯誤或連線失敗不能當成空名單。下載驗證及Apache本地套用尚未部署。

公開範圍包含repo檔案與Git提交歷史。只保存經核准的IP名單、格式與去敏決策，不提交token、Cookie、會員個資或原始應用日誌；完整巡檢證據留在本機工作區。


## 目前採用TXT-only（2026-10-10 最新指示，優先於先前格式）

使用者要求先整理今天截至查詢截止的IP進每日TXT，不在repo填到期時間，JSON與Apache conf暫不製作或更新；到期由系統維運處理。

- 正式讀取檔為`ao_blacklist/YYYY/YYYY-MM-DD.txt`，`blocklist.txt`為相容TXT副本；一行一個IP、去重、數值排序。
- `blocklist.json`與`security-blocklist.conf`暫留先前空狀態，不再用JSON entries推算或清空目前TXT，也不把TXT當作未授權獨立規則而刪除。先前JSON為權威、JSON/TXT/conf原子同步及repo到期刪除流程暫停；本段為最新流程。
- TXT是供維運接收的來源名單；有效性、期限、實際套用範圍、解除及已處理狀態由維運系統控制。跨日再次列出同IP不代表新的封鎖或續期。
- 本次加入依據為使用者已回報封鎖、明確入侵路徑探測（非403）或集中404；不因IP國家、ASN或HTTP200單独列入。未核對的會員訂單來源仍另行追蹤，不混入本次名單。
- 本次加入授權只限此輪今日整理；之後巡檢仍不能自行新增IP、補期限、續期或直接改主機。半小時巡檢只核對既有TXT，已存在且無變動不提交。換日可將最近已授權TXT原樣帶到新日期檔，保留舊快照；它代表同一批維運待處理／維護的名單，不自行認定每個IP尚未到期。
- 維運或使用者從TXT移除的IP需保留移除，不得依舊日快照重新補回。同期檔案如有人工變動，重讀main並核對，不覆蓋。
- 當日TXT與相容TXT同一提交更新；JSON/conf暫不動。主機同步尚未部署，repo寫入不代表Apache已阻擋。
