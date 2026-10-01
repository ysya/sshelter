# SSHelter Sync Chain — 設計(Brave 式免帳號端對端同步)

日期:2026-09-27。前置討論:2026-09-08 Codex 評估(Termius-like + MCP)、本日的
四輪方案比較(檔案資料夾 / git 中心 / Brave 式中繼 / bastion)。使用者定案:
**Brave 式 sync chain,私鑰 opt-in 進 chain 以達成「輸入助記詞即可連線」**。

## 1. 目標與非目標

**目標**
- 新裝置只做一件事 —— 輸入 24 詞助記詞(或掃 QR)—— 之後主機清單、tag、
  選定的私鑰與密碼全部就位;終端機 `ssh alias`、SSHelter 連線、MCP `run` 立即可用。
- 免帳號、免自架、免 git;中繼只看得到密文。
- 保留 SSHelter 的核心價值:OpenSSH 原生(同步結果就是真實的 `~/.ssh` 檔案)、
  lossless。
- 中繼開源可自架;專案停止維護時使用者不被鎖死(本機資料永遠完整)。

**非目標(v1)**
- 即時推送(WebSocket)—— v1 用輪詢。
- 同步使用者主 config 或其他非受管檔案 —— 只同步 SSHelter 受管的同步檔。
- known_hosts 同步、Termius 式內建終端機、多人/團隊共享 chain。
- 助記詞時效字(Brave 的 time-limited word)、多 chain。

## 2. 安全模型(先定,其餘設計服從它)

- **助記詞 = 一切**:能解出主機、tag、已同步的私鑰與密碼,也能加入 chain。
  等同 Termius/1Password 的主密碼。配對畫面必須明講,並建議存進密碼管理器。
- **中繼零知識**:只儲存密文與 metadata(chain id、序號、大小、時間、IP)。
  中繼被攻破 = 拿到密文,無助記詞無法解密。
- **傳輸只走 HTTPS**:relay URL 必須是 `https://`;唯一例外是 loopback
  (`127.0.0.1`、`localhost`、`[::1]`)供本機 `wrangler dev`。client **不跟隨 redirect**
  (避免 https→http 降級把 bearer token 送上明文連線)。bearer token 有整條 chain 的讀寫刪權限,
  所以這條規則不可放寬。
- **私鑰 opt-in、逐把決定**:預設不同步任何既有金鑰;使用者在 Keys dialog
  逐把勾「Sync this key」;migration wizard 也一律**預設不勾**(被主機引用的只標示「建議」)。
  硬體/不可匯出金鑰無法同步(UI 直接不提供選項)。
- **本機狀態檔不含明文祕密**:`sync-state.json` 只保存本裝置會處理的種類
  (Phase A:`host`/`device`/`meta`)的明文快取;`key`/`password` 與任何未知種類一律以
  **原始密文 envelope** 保留在 `sealed`(供 Phase B 或更新版本處理),絕不解密進狀態檔。
  0600 不等於 keychain:祕密只能落在 keychain 或 `~/.ssh/<name>`(0600)。
- **Forget device 不是撤權**:同一條 chain 的每台裝置持有同一組助記詞,中繼無法分辨裝置,
  所以「Forget device」只是把它從清單移除(tombstone 它的 device 記錄),**不會**阻止它繼續
  同步。裝置遺失或不再信任的正確處置是 §8 的「重新建鏈 + 輪替」:在保留的裝置上建立新 chain
  (新助記詞)→ 其餘裝置重新配對 → 之後才產生新 SSH 金鑰並批次部署/撤舊。新金鑰**只能進新
  chain**:舊 chain 的所有密文對持有舊助記詞的人永遠可讀。UI 文案不得暗示 Forget 會停止
  該裝置同步。
- **密碼 opt-in、預設關**:「Sync passwords」是**每台裝置各自**的開關 —— 開啟的
  裝置才上傳自己的密碼、也才把遠端密碼記錄寫入本機 keychain;關閉的裝置兩者皆不做。
- 既有 MCP 邊界不變:MCP allowlist 是每台裝置的本機政策,**不同步**。
- 本機落地檔案權限:私鑰 0600、同步檔 0600、狀態檔 0600(既有 `fsutil` 慣例)。

## 3. 資料模型

### 3.1 受管同步檔(主機的真相來源)
- 路徑固定:`~/.ssh/sshelter/hosts.config`(目錄 `~/.ssh/sshelter/` 0700)。
- 加入 chain 或建立 chain 時,SSHelter 在每台裝置的主 config 插入
  `Include ~/.ssh/sshelter/hosts.config`,位置是**檔案最頂端**:前導註解/空行之後、任何
  既有 Include、全域指令、Host/Match 之前(`sync::hosts_file::ensure_include`,自有的位置
  規則)。刻意不用 `newfile.rs` 的 `include_insert_index`(它插在最後一個 Include **之後**):
  ssh 是 first-obtained-wins,同步檔必須是第一個被讀到的定義,§10 的「同步主機遮蔽本地同名
  主機」才成立。
- 受管檔只放具名主機:**任一** pattern 含 wildcard/否定字元(`*`、`?`、`!`)的區塊
  (`Host *`、`Host *.example`、`Host web *.internal`、`Host web !prod`)**整個**視為裝置本地的
  config 結構 —— 不擷取、不上傳、不遷入、收到遠端記錄時不套用、也不刪除。前後端用同一條規則
  (後端 `hosts_file::is_syncable_block`、前端 `isSyncableHost`),不沿用 sidebar 的
  `isWildcardOnly`(它只認「全部都是 wildcard」,且不看 `!`)。
- 同步 Include 若已存在但不在最頂端(舊版插法、使用者搬動),`ensure_include` 會把它搬上去;
  多路徑的 `Include a b` 只抽走我們的 token,其他路徑留在原地。
- **同步範圍 = 這個檔案裡的 host 區塊**。其他檔案的 host 維持裝置本地。
  提供「Move to synced」(沿用批次 move)與首次一鍵「把所有主機移入同步檔」。
- sidebar 檔案分組把它顯示為「Synced」(預設 file alias)。

### 3.2 記錄(record)
所有同步內容都是記錄;chain = 記錄集合。共同欄位:

| 欄位 | 說明 |
|---|---|
| `id` | 記錄識別(見各型別) |
| `kind` | `host` / `key` / `password` / `device` / `meta`(chain 層級,見 §10) |
| `version` | 本地遞增的邏輯版本(每次修改 +1) |
| `updated_at_ms` | 修改時間(裝置時鐘) |
| `device_id` | 最後修改的裝置 |
| `deleted` | tombstone |
| `payload` | 加密前的型別內容(JSON) |

型別內容:
- **host** —— `id = alias`。payload = 該 Host 區塊的**原始文字**(header + body 逐行 raw,
  以 `serialize_items(&[Item::Host(block)])` 產生,lossless),tags 自然包含在內
  (`#tags:` 是區塊的一行)。alias 改名 = tombstone 舊 + 新增新。
- **key** —— `id = 檔名`(`~/.ssh/<name>`)。payload = 私鑰檔內容、`.pub` 內容、
  key type、fingerprint、`has_passphrase`。**私鑰以檔案原樣同步,金鑰自身的
  passphrase 保留**(chain 加密之外的第二層)。
- **password** —— `id = alias`。payload = 密碼字串。落地到本機 keychain
  (`secrets::host_account(alias)`)。
- **device** —— `id = device_id`。payload = 顯示名稱、平台、加入時間、最後上線、
  持有的 key id 清單(供 UI 與輪替提示)。

### 3.3 金鑰路徑規則(讓同一份 config 在每台裝置都成立)
- 同步的金鑰在**每台裝置都落在 `~/.ssh/<name>`**;`~` 在 macOS/Linux/Windows
  OpenSSH 皆可解析,因此 host 區塊裡 `IdentityFile ~/.ssh/<name>` 的文字可以完全相同,
  不需要 key-id 間接層。
- 匯入既有金鑰(如 `~/.ssh/id_ed25519`)= 讀檔建記錄,不搬動檔案。
- 加入 chain 時若本機已有**不同內容**的同名檔 → 中止該筆並提示改名,不覆蓋。
- host 引用了未同步的金鑰 → sidebar/editor 顯示警示「key not synced」。

## 4. 密碼學(byte-level 定義;A1 以已知答案向量釘住)

- 助記詞:BIP39 英文 24 詞(256-bit entropy),`bip39` crate。輸入正規化:以任意空白切字 →
  全部小寫 → 單一空白連接;字數 ≠ 24 或 BIP39 checksum 不符 → 拒絕。
- `seed = PBKDF2-HMAC-SHA512(password = 正規化助記詞, salt = "mnemonic" || passphrase(""),
  2048 rounds, 64 bytes)`,即 `bip39::Mnemonic::to_seed("")`。
- HKDF-SHA256:`PRK = HKDF-Extract(salt = ASCII "sshelter-sync-v1", IKM = seed)`;各展開 32 bytes:
  - `info = ASCII "sshelter/v1/chain-id"` → `chain_id`(小寫 hex,64 字元,可公開)
  - `info = ASCII "sshelter/v1/auth"` → `auth_token`(小寫 hex,64 字元;中繼只存
    `SHA-256(auth_token 的 ASCII hex 字串)` 的小寫 hex)
  - `info = ASCII "sshelter/v1/enc"` → `enc_key`(32 raw bytes,只在記憶體)
- `id_hash = hex(HMAC-SHA256(key = enc_key, msg = kind || 0x0A || id))`(小寫 hex,64 字元)
  —— 中繼連 alias/檔名都看不到。
- 記錄加密:XChaCha20-Poly1305(`chacha20poly1305` crate),key = `enc_key`,每筆隨機 24-byte
  nonce;`AAD = chain_id || 0x0A || kind || 0x0A || id_hash`(全 ASCII)。AAD 綁 `id_hash`
  而非明文 `id`:接收端解密前只有 `id_hash`。解密後必須再驗證「明文 `kind`/`id` 算出的
  `id_hash`」與 envelope 相同,否則視為損毀。
- 明文 = 整筆記錄的 JSON(`kind, id, version, updated_at_ms, device_id, deleted, payload`)。
  tombstone 也帶明文 `id` 與時間戳(LWW 需要),所以 tombstone 一樣有密文(`payload: null`)。
- 中繼看到的 envelope:`{ idHash, kind, seq, nonce(base64), ciphertext(base64), deleted }`;
  base64 用標準字母表、含 padding。
- 已知答案向量(釘在 `crypto.rs` 測試裡,防止協定被無意改動):助記詞
  `abandon ×23 art` 的 `chain_id`、`auth_token`、`id_hash("host", "web-1")` 三個 hex 值。
- 助記詞存本機 keychain(`secrets` 模組,account `sync:mnemonic`),因為任一台既有
  裝置都要能「Show pairing code」給新裝置(與 Brave 的 View sync code 一致);
  派生值只在記憶體,啟動時重算。**不存純文字檔**。
- v1 為純桌面,配對靠輸入 24 詞;QR 留給未來的行動版(桌面 app 沒有相機)。

## 5. 中繼(relay)

Cloudflare Worker + Durable Object(每 chain 一個 DO,SQLite storage 提供順序與原子性)。
開源,repo 內 `relay/` 目錄,`wrangler deploy` 即自架;app 設定可改 relay URL。不用 Cloudflare 的自架:
`relay/selfhost/` 以 Docker Compose 跑同一份 bundle(workerd + Durable Object 存在 volume,Caddy 提供 HTTPS 並以
連線來源覆寫 `CF-Connecting-IP`)。
**兩種模式**:建置時注入 `SSHELTER_RELAY_URL`(release workflow 讀 repository variable)→ 預設指向
那個託管的 Worker;沒注入 → release 建置**沒有內建中繼**(`DEFAULT_RELAY_URL` 為空字串),需要同步的
使用者在 Settings → Sync 填入中繼網址,填好之前 Create/Join 停用、後端也拒絕;debug 建置沒注入時預設
本機 `wrangler dev`(`http://127.0.0.1:8787`)。release workflow 在變數沒設時只留 notice,設了卻不是
`https://` 才讓建置失敗。

API(全部 `Authorization: Bearer <token>`,JSON):

| Method | Path | 說明 |
|---|---|---|
| `PUT` | `/v1/chains/{chain_id}` | 建立 chain(冪等):不存在 → `201` 並記下 token hash;存在且 token 相符 → `200`;不符 → `404`。**只有 Create 走這條**;Join 必須先 `GET …/records?since=0` 驗證(`404` = 這組助記詞沒有對應的 chain),Join 路徑絕不呼叫 PUT —— 否則任何 checksum 正確的助記詞都會建出一條新 chain 而不是回報錯誤 |
| `POST` | `/v1/chains/{chain_id}/records` | 批次 upsert;body `[{idHash, kind, ciphertext, nonce, deleted, baseSeq}]`(1–200 筆);回每筆 `{status:"ok", seq}` 或 **`{status:"conflict", current: Envelope}`**(`baseSeq` 落後於伺服器現況;client 下一輪 pull 會拿到它)。伺服器先判定每筆接受/衝突,再以「接受後的用量」檢查配額;超過 → 整批 `413`、不做部分寫入 |
| `GET` | `/v1/chains/{chain_id}/records?since={seq}` | 回 `seq > since` 的 envelope 與 `latestSeq` |
| `DELETE` | `/v1/chains/{chain_id}` | 刪除整個 chain(離開最後一台裝置時);同一個 DO instance 之後必須能再次 `PUT` 建立(schema 重建) |

防濫用:
- `nonce`/`ciphertext` 必須是**嚴格的標準 base64**(ASCII、長度為 4 的倍數、padding ≤ 2、非空),
  否則 `400`。配額以字元數計 —— 只有 ASCII 才能讓 Worker 的 `.length` 與 SQLite 的 `LENGTH()`
  一致(SQLite 遇 NUL 就停),否則可繞過配額。
- 每筆 `ciphertext` ≤ 64 KiB;每 chain 儲存總量 ≤ 1 MiB(**含 tombstone 與 nonce**)、
  記錄數 ≤ 4096;request body ≤ 1 MiB:以 byte 上限的**串流**讀取,超過即取消回 `413`,
  不把整個 body 讀進記憶體再檢查;之後才 JSON 解析。
- 每 chain 每分鐘 ≤ 120 次請求(所有端點都計,含建鏈那一次;已授權但超限一律 `429`,不論 PUT/GET/
  POST/DELETE)。計數跟著 DO instance 走、`DELETE` 不歸零:同一分鐘內刪掉再建也照算(建鏈那一次
  超限同樣 `429`)。這個每 chain 計數放在記憶體裡,是 best-effort:DO 閒置被逐出後會歸零(持續打
  同一條 chain 的請求會讓它保持常駐,所以實際上只在閒置之後才歸零);硬上限是下面**持久化**的
  每 IP 桶。
- **跨 chain 的防線**(每個來源 IP,`CF-Connecting-IP`):每小時最多 20 次建鏈請求(`PUT`,
  不論結果是 201/200/404 —— 計的是嘗試,不是成功建立的 chain 數)、所有請求合計 ≤ 1200 次(`429`)。**未建立的 chain 被讀取(GET/POST/DELETE)一律 `404` 且不配置任何
  儲存**:DO 只在 `PUT` 建立時才建 schema,其餘 RPC 先看 `sqlite_master` 有沒有 `meta` 表,
  沒有就 404 —— 讀不存在的 chain 不能留下持久化的空資料庫。
- 閒置 180 天自動清除;**成功授權的 pull 也刷新閒置期限**(節流:同一 instance 每 6 小時最多
  刷新一次),所以只讀不寫的裝置(唯讀模式)不會讓 chain 被清掉。token hash 不符一律 `404`
  (不洩露 chain 存在)。

## 6. 同步引擎(Rust,`src-tauri/src/sync/`)

- 狀態:`~/.local/share/org.homelab.sshelter/sync-state.json`(`atomic_write` 0600):
  `{ version, chain_id, device_id, device_name, relay_url, cursor_seq, password_sync,
  remote_schema_version, records: {明文快取,只有 host/device/meta}, sealed: {未處理種類的
  原始 envelope}, last_sync_ms, last_error }`。
  內容讀不懂(損毀、較新版本寫的)→ 改名為同目錄的 `sync-state.unreadable-<ms>.json` 保留,以全新
  狀態啟動並在狀態列顯示該檔名;讀取發生 I/O 錯誤 → 原檔不動,本次執行不寫狀態檔、也不做
  Create/Join/Leave/設定變更(提示重啟)—— 暫時性的讀取錯誤不可讓裝置退出 chain,也不可讓新的
  Create 覆蓋 keychain 裡舊 chain 的助記詞。
- 觸發:app 啟動(前端第一次載入 config 之後;doc 還沒載入時的輪次安靜跳過,不記錯誤)、視窗取得
  焦點、每 45 秒、本地變更後立即。之後的 config 重新載入**只有在受管檔的有無或指紋改變時**才喚醒
  (外部清空要先被同步輪次看到,app 下一次存檔才不會對著過期快取規劃刪除);前端每次 `sync://applied`
  之後的重新載入不喚醒,否則受管檔讀不進來(非 UTF-8、無讀取權限)時會形成沒有間隔的迴圈。
- **基線輪**:剛 Join 的第一輪(`baseline_established = false`;Create 的 chain 是空的,建立時直接
  視為已建立基線)**不做本機 diff**,先 pull 並以 chain 為準套用 —— chain 上仍存在的區塊以 chain 版本覆蓋本機同名區塊;chain 上已
  tombstone、本機同步檔卻還留著的區塊被移除(`persist_file` 先備份);chain 不認識的區塊保留。
  成功後 `baseline_established = true` 並立刻再跑一輪,本機獨有的區塊才以新主機上傳。
  少了這一步,Leave 後保留的舊區塊會在重新加入時以「現在」的時間戳復活遠端的刪除。
- **受管檔消失**:加入中、基線已建立、快取裡有主機記錄,受管檔卻不在了(被刪除、整個 `~/.ssh`
  被換掉)→ **不**當成本機刪除所有主機(否則 tombstone 會推給每一台裝置)。引擎在 doc 鎖內丟掉
  快取裡**已上傳**的 host 記錄(尚未上傳的 dirty 修改與刪除保留;device/meta/sealed 保留)、
  `cursor_seq = 0`、`baseline_established = false`、換 generation,**狀態存檔成功後**才重建空檔;
  下一輪是基線輪,以 chain 為準把主機寫回,合併後仍以本機為準、檔案裡卻沒有的未上傳修改也一併寫回
  (輸給較新遠端版的以 `sync://conflict` 通知)。狀態存不下就不建檔,下一輪再偵測一次。要刪除同步
  主機請在 app 裡刪或 Leave;整檔消失一律視為意外。
- **受管檔的不變式**:`hosts.config` 只放具名、互不重複的 Host 區塊(§3.1)。每輪讀檔時檢查
  (`check_managed_items`);違反(含 wildcard 的區塊、同一 alias 出現兩次)→ 整輪停在讀檔階段:
  不 diff、不 pull 套用、不 push,狀態列顯示要搬走/刪掉哪個區塊。因為這個不變式成立,遠端效果
  (文字已先驗證)套用時不可能失敗;萬一失敗(bug),套用**全有或全無**:什麼都不寫、不發布、
  cursor 不前進,錯誤顯示在狀態列,下一輪重試 —— 絕不在「部分套用」的狀態上前進 cursor。
- **本機編輯的時間戳**:SSHelter 自己寫 `hosts.config`(任何 `persist_file`)時,`note_file_written`
  **在存檔當下**就做區塊 diff、產生 dirty 記錄(`updated_at_ms` = 存檔時間)並立刻持久化,同時
  換 generation 讓在途輪次的舊快照作廢 —— 所以時間戳精確,也不會把別的區塊較晚的修改時間套到
  較早修改的區塊上;**同步狀態成功寫進磁碟之後**,重啟也保留這些時間。引擎自己套用遠端效果的寫入不算本機編輯(`EngineWrite`)。
  加入中的任何受管檔 app 寫入都換 generation,即使檔案被改成違反不變式(那次不規劃,下一輪停在
  驗證錯誤)。狀態寫檔失敗時 SSH 檔照樣存成功,但狀態列顯示錯誤、記下 `unsaved`;每次 tick 先補存
  (不論是否加入中),補不成就不做任何網路操作 —— 不在未落盤的狀態上 pull/push。**明示的限制**:
  狀態一直寫不進磁碟就結束程式的話,重啟只剩磁碟上的舊狀態,那些編輯會以整檔 mtime 近似重新
  規劃(與外部編輯相同)。套用遠端效果時若檔案已寫、狀態卻存不下,記憶體快取仍跟著檔案走(不撤回),
  tray 與 `sync://applied` 照樣發,這一輪才停下。
  **外部編輯**(文字編輯器)只能由同步輪次發現,時間戳用檔案 mtime —— 整檔的近似值:同一次
  外部存檔裡改到的多個區塊會拿到同一個時間。這是明示的限制。
- **一輪的順序**(每步可獨立失敗;失敗不影響前一步已持久化的結果):
  1. 讀取受管檔目前的區塊(磁碟指紋與 in-memory 不同就先重載)、檢查不變式,記下本輪的
     **檔案指紋**與**檔案 mtime**。
  2. 本機 diff(只會抓到外部編輯;app 的存檔已在存檔當下規劃過)→ dirty 記錄(升版本、單調
     時間戳 = 檔案 mtime)並**立刻持久化**。離線時也一樣,重試沿用同一版本與時間戳。心跳
     (device 記錄)才用現在時間。
  3. `pull(since = cursor_seq)`:逐筆解密、驗證身分、LWW 合併;結果先留在記憶體(含
     `cursor_seq = latestSeq`)。
  4. **套用 + 發布是同一個交易**(`apply_and_commit`,全程持有 doc 鎖):比 generation → 比第 1 步的
     指紋(**每次發布都比**,即使這輪沒有 host 效果)—— 受管檔若在這段網路時間內被改過(UI 存檔或外部編輯),**整輪的 pull/merge 結果丟棄、
     cursor 不前進**,立刻再跑一輪 —— 指紋相同才在區塊副本上套效果、經 `persist_file` 寫進
     `hosts.config`,**仍持有 doc 鎖**時把合併後的記錄與 cursor 發布並持久化。所有會換 generation
     的路徑都先拿 doc 鎖,所以不會出現「檔案已寫入遠端內容、快取卻被拒絕」,下一輪也就不會把
     已套用的遠端內容誤判成本機修改。持久化失敗時先退回舊區塊、再從磁碟重載(磁碟可能已是新
     內容:寫入成功但指紋計算失敗)—— 重載後磁碟若正是剛寫的內容,就當作已提交照常發布,否則
     本輪作廢;重載也失敗 → 整份 in-memory doc 作廢(`None`)並發 `sync://applied` 讓前端重新
     載入,絕不留下可能與磁碟不同、看起來卻有效的 doc。tray 重建與所有事件都在**放掉鎖之後**:
     建立 tray menu 會同步等待主執行緒,持 doc 鎖呼叫會和主執行緒上等 doc 鎖的 command 互等。
  5. `push(dirty)`(分批:每批 ≤ 200 筆且 ≤ 512 KiB;唯讀模式略過)。**accepted 的 seq 只更新
     該筆 `LocalRecord.seq` 並清 dirty,絕不推進 `cursor_seq`**(否則會跳過其他裝置在中間寫入
     的序號);`conflict` 的記錄本輪不處理,只標記「立刻再跑一輪」,下一輪 pull 會拿到它。
  6. 發事件:`sync://status`;有 host 效果寫進檔案 → `sync://applied`(前端據此重新載入主機
     清單,後端重建 tray);本機未上傳修改被較新遠端蓋掉 → `sync://conflict`。
- **合併規則(記錄層級 LWW)**:遠端 vs 本地同 id:`updated_at_ms` 大者勝;相同則 tombstone
  勝過修改;再相同則 `device_id` 字典序小者勝。輸的一方若是本地未上傳的修改 → `sync://conflict`
  toast「web-1 was changed on MacBook, your local edit was replaced」。
- **遠端 host 記錄的三種結果**:`Upsert{alias, text}`(合併前就以 `validate_host_text` 檢查:
  文字**只能**含恰好一個 Host 區塊 —— 不得夾帶 Match、全域指令或區塊外的註解,否則套用後
  的檔案與快取不一致 —— 且第一個 pattern 等於 alias、所有 pattern 皆非 wildcard)/
  `Delete{alias}`(**只有**驗證過的 `deleted = true`)/ `Skip`(解不開、身分不符、payload 格式
  不支援、文字不是合法區塊、wildcard)。Skip 的記錄**不進快取**——否則套用失敗的記錄會在下一輪
  被當成「快取有、檔案沒有」而產生 tombstone,把別台的主機刪掉。格式不支援**絕不**當成刪除。
- **未處理的種類**(Phase A 的 `key`/`password`,以及任何未知 kind):原始 envelope 存進
  `sealed`(key = `kind:idHash`),不解密、不落明文;cursor 照常前進。升級後的版本從 `sealed`
  重新處理。
- **唯讀模式**:chain 的 `meta.schema_version` 持久化為 `remote_schema_version`;**每輪開始**
  用它判斷(不是只看本輪有沒有收到 meta)。比本 app 新 → 只套用可理解的記錄、不上傳、狀態列
  顯示「請更新 SSHelter」。
- **生命週期與同步的互斥**:runtime 的 `generation`、狀態、金鑰放在**同一把鎖**裡(`SyncCore`),
  永遠一起快照、一起替換 —— 分開鎖會出現「新 generation + 舊狀態」的交錯。同步輪次開始時一次
  取得三者的快照;`apply_and_commit` 在 doc 鎖內比 generation,回寫狀態(`commit_state`)也比;
  不同就整輪丟棄。**每個會換 generation 的路徑都先拿 doc 鎖**:Create/Join/Leave(換 generation/
  狀態/金鑰時)、只動狀態的命令(`mutate_state`:改裝置名、Forget device、未加入時改 relay URL)、
  以及 `note_file_written`(本來就在 `persist_file` 的 doc 鎖內)。Create/Join/Leave 與改 relay URL
  全程持有一把 **lifecycle 鎖**(含網路等待與 keychain 讀寫,跑在 `spawn_blocking`),彼此互斥、
  前置檢查也在鎖內 —— 舊 Leave 不可能刪掉新 Join 剛存的助記詞,兩個 Join 不可能同時通過檢查,
  Join 驗證中途也換不掉它驗證的 relay。鎖順序固定為 lifecycle → doc → core。同步輪次失敗時,
  錯誤只記在產生它的那一代狀態上(generation 相符才寫 `last_error`):舊 chain 的逾時不會寫進
  新 chain。會等鎖、寫磁碟、讀 keychain 或做網路的 sync command 一律 async + `spawn_blocking`
  (Tauri 的同步 command 跑在主執行緒上)。
- **relay URL 只能在未加入時更改**:cursor 與每筆記錄的 seq 都是某一個 relay 的序號,原地換
  relay 會讓 pull 跳過資料。要換 relay:Leave → 改 URL → 在新 relay 上 Create/Join。
- **從本機產生記錄**:兩個時機做「區塊 diff」——(a)SSHelter 自己寫 `hosts.config` 的當下
  (`note_file_written`,見上);(b)同步 tick 時檔案指紋與上次不同(使用者手改)。diff =
  逐區塊序列化後與快取比對,變了就升版本、標 dirty,消失的區塊 tombstone。Keys dialog 勾選/取消
  → key 記錄;密碼儲存/刪除(既有 `secrets_set/delete` 命令)→ 本機「Sync passwords」開啟時
  產生記錄(Phase B)。
- 裝置身分:首次執行產生 `device_id`(16 bytes 隨機 hex),名稱預設主機名、可改。
- Create:`PUT` 建鏈 → 存助記詞 → 種下 meta 與 device 記錄。Join:先 `GET …?since=0`
  驗證(`404` → 「no sync chain matches this recovery phrase」,不建鏈)→ 存助記詞 → 種下
  device 記錄。兩者都在 `spawn_blocking` 裡做網路。
- Leave chain:先清 `sync-state.json` 的 chain 部分並持久化(確保停止同步)→ 再刪 keychain 的
  助記詞。狀態寫檔失敗也照樣刪 keychain(磁碟上的舊狀態就算還是 joined,沒有助記詞就派生不出金鑰、
  重啟不會恢復同步),錯誤回報給使用者,`unsaved` 讓背景自動補寫。keychain 刪除失敗要**回報錯誤**、把 `phrase_cleanup_pending = true` **持久化到狀態檔**
  (重啟後警示與重試入口仍在),狀態列顯示「recovery phrase still in keychain」與「Remove
  phrase」重試按鈕(再呼叫一次 `sync_leave_chain(false)`;未加入時它只重試 keychain 清理),
  不得宣稱已清乾淨。本機檔案全部保留。若是 chain 的最後
  一台裝置,詢問是否一併 `DELETE` 中繼上的 chain。
- 網路 I/O 只在同步執行緒(std thread)或 `tauri::async_runtime::spawn_blocking` 裡跑
  (`reqwest::blocking` 在 tokio runtime 內會 panic);持有 doc 鎖時絕不做網路 I/O。
- **每個 OS 使用者只跑一個同步引擎**:引擎啟動時在 app data 目錄對 `sync.lock` 取獨占鎖
  (`File::try_lock`);拿不到鎖的 process(第二個 instance、兩個 MCP adapter 同時拉起的
  `--mcp-host`)不跑 worker、存檔時不規劃,所有會改狀態的 sync 命令與「搬進同步檔」都預先拒絕,
  狀態列顯示「Sync is running in another SSHelter process」;它對 SSH 檔的其他編輯對作用中的引擎
  而言就是外部編輯。取鎖遇到 WouldBlock 時先重試約 2 秒(app 內更新重啟時,舊 process 可能還沒退出)。
- **喚醒合併**:worker 每輪開始前先清空排隊的喚醒,批次寫入(遷入幾十台主機)只觸發一輪,
  不會把整條 chain 打到每分鐘上限。
- **push conflict 的立即重跑有上限**:連續 3 輪都以 conflict 結束後不再立即重跑,只靠 45 秒
  tick,並在狀態列說明「有變更因中繼上的較新版本無法讀取而上傳不了 —— 請更新 SSHelter」;沒有
  conflict 的一輪、Join/Leave 會歸零。
- **受管檔被清空**(解析出 0 個 Host 區塊、快取仍有存活主機、基線已建立)比照「受管檔消失」,
  從 chain 重新長出。部分截斷或還原成舊版仍是一般外部編輯(缺少的區塊會 tombstone)—— 明示限制,
  README 已說明;要刪除同步主機請在 app 裡刪或 Leave。
- **不依超前磁碟的 in-memory doc 行動**:受管檔的 in-memory 區塊序列化後(與 `persist_file` 同一個
  serializer)和磁碟位元組不同就先重載 —— 讀檔(gather)與 `apply_and_commit` 都檢查,後者不同就
  整輪作廢重跑。`config_move_host` 寫檔失敗後從磁碟重載 doc,不留下比磁碟新的記憶體內容。
- **中繼序號倒退**:pull 回來的 `latestSeq` 小於本機 cursor(自架中繼重設/從備份還原)→ cursor 歸零,
  並把快取裡所有記錄設為 `seq = 0`、dirty:重新拉取後(KeepLocal 會把 seq 更新成中繼現況)把中繼缺少
  或較舊的記錄全部重新上傳,以本機快取修復中繼(`sealed` 的原始密文不重傳)。代價是一次完整重傳,
  另一台裝置在還原後改過的主機會多一次「Sync overwrote a local change」提示(資料結果正確)。中繼序號
  已追上舊 cursor 時偵測不到 —— 需要中繼 epoch,留待協定變更。
- 啟動時 keychain 助記詞派生出的 `chain_id` 與狀態不符 → 不派生金鑰,狀態列顯示「the recovery
  phrase in the keychain belongs to a different sync chain; leave and rejoin」。加入中的輪次收到中繼
  404 → 「This sync chain no longer exists on the relay (deleted from another device or expired) —
  leave it on this device」。
- Create/Join 在 SSH config 尚未載入(例如新機器沒有 `~/.ssh/config`)時預先拒絕,不在猜測的路徑
  建檔(前端才知道使用者設定的 config 路徑);加入狀態與助記詞存下之後,受管檔準備步驟失敗不讓
  Create/Join 失敗(下一輪重做),Create 一定回傳助記詞。
- 離線:pull 失敗只記狀態,不阻擋任何本機操作;已持久化的 dirty 記錄下次上線再送。

## 7. 前端

- **Settings → Sync** pane(比照 McpPane 的輪詢/狀態模式):
  - 未加入:「Create sync chain」「Join with words」,以及 relay URL(進階)——自架或預設
    中繼不可達時,**加入前**就要能改。建置沒有內建中繼時,relay URL 改放在最上方的「Relay」區塊
    (必填),填好之前 Create/Join 停用。
  - 建立後:顯示 24 詞 + 「I have saved these words」確認(並提醒存進密碼管理器)。這個確認
    畫面的 state 必須放在**不會因 `joined` 切換而卸載**的父層(pane 本身),否則建立成功的
    瞬間畫面就消失,無法保證使用者看過並確認。
  - 已加入:狀態列(last sync、錯誤)、裝置清單(名稱/平台/last seen/持有金鑰/**Forget**)、
    「Show pairing code」(明確按鈕、再次顯示 24 詞給新裝置抄)、
    「Sync passwords」開關(本機)、relay URL(唯讀顯示,註明「Leave the chain to switch relays」)、
    「Leave chain」。
  - 助記詞(建立回傳、Show、Join 輸入)**不經 TanStack Query 的 mutation/query cache**:
    直接 `tauriInvoke` + 元件 local state,關閉視窗即清掉。
- **Keys dialog**:每把金鑰一個「Synced」開關(硬體/無私鑰檔者停用)。
- **Sidebar**:`hosts.config` 群組標籤「Synced」;host 使用未同步金鑰時列尾小警示。
- **遠端變更進 UI**:後端套用遠端 host 效果後發 `sync://applied`;前端據此讓 hosts/host
  detail/files 的 query cache 失效(光靠 status 事件不會刷新主機清單),後端同時重建 tray。
- **新裝置上手**:Join 完成 → 進度畫面(拉取記錄 → 寫入金鑰 → 寫入 config → 插入
  Include)→ 完成頁列出「N hosts, M keys, P passwords ready」。
- **Forget device**:只從清單移除(tombstone),文案明講「it keeps syncing if it still has the
  recovery phrase」;若該裝置遺失/不再信任 → 連結到 §8 的「Start a new chain」流程。

## 8. 金鑰輪替(v1 必要配套)

- 入口:Forget device 的「lost this device?」提示、Keys dialog 每把同步金鑰的「Rotate…」。
- **前置(裝置遺失/不再信任時必做)**:舊 chain 的密文對持有舊助記詞的人永遠可讀,所以先
  在保留的裝置上 Leave 舊 chain → 「Start a new chain」(新助記詞)→ 其餘保留裝置重新配對,
  之後的新金鑰才只進新 chain。單純換金鑰不換 chain,遺失的裝置照樣拿得到新金鑰。
- 流程(單一對話框、可看進度):
  1. 產生新金鑰(名稱 `<old>-<yyyymmdd>` 或使用者指定)並加入(新)chain。
  2. 找出所有 `IdentityFile` 指向舊金鑰的同步主機 → 批次部署新公鑰
     (沿用 `deploy.rs` 管線;逐台結果列表;失敗可重試)。
  3. 成功的主機 → 遠端移除舊公鑰:新的 `REMOTE_REMOVE_SCRIPT`(`grep -vxF`
     過濾該行,原子寫回,權限 0600,回 `SSHELTER_REMOVED`)。
  4. 改寫這些主機的 `IdentityFile` 指向新金鑰(記錄升版、同步)。
  5. 舊金鑰記錄 tombstone;本機檔案改名 `.revoked`(不刪,留給使用者)。
- 任何一步失敗都不會讓主機無法登入(舊鑰只在新鑰部署成功後才移除)。

## 9. 驗收與測試

- Rust 單元:HKDF 派生已知答案向量、加解密 round-trip 與 AAD 錯配失敗、
  LWW 合併全案例(時間/裝置 tie-break/tombstone)、host 區塊序列化 round-trip
  (含 `#tags:`)、applier 不動其他區塊、wildcard 區塊(含混合 pattern)不擷取/不套用/不刪除、
  既有 Include 搬到最頂端、cursor 只隨 pull 前進、格式不支援或文字不合法的記錄不刪主機也不進快取、
  基線輪不復活遠端刪除、key 落地權限與同名不同內容跳過。
- Relay:Workers Vitest 整合測試(建立、push/pull、conflict、配額含 tombstone、錯 token 404、
  delete 後同 instance 可重建、每 IP 建鏈限制)。
- 端到端手動:**兩個 OS 使用者帳號、VM 或實體機**。自訂 config path 不能模擬兩台裝置:
  `~/.ssh/sshelter/`、`sync-state.json`、device id 與 keychain 都跟著 OS 使用者走,兩個
  checkout 會互相操作同一份同步資產。步驟:配對、互改主機、同步私鑰後以 `ssh` 登入、
  Forget device、重新建鏈 → 輪替。
- 既有測試不退:Rust、vitest、四平台 CI。

## 10. 既有版本的 migration 與相容性

**升級不改行為**:sync 完全 opt-in。從 0.15.x 升上來的安裝,在使用者按下
「Create / Join」之前,不建立 `~/.ssh/sshelter/`、不插 Include、不碰 keychain、
不聯網。所有既有功能(config 編輯、部署、MCP)路徑不變。

**首台裝置建立 chain 時的資料遷入(wizard,三步、皆可跳過)**
1. **主機**:列出所有檔案中的非 wildcard 主機(依檔案分組、預設全選),以既有
   `config_move_host`(byte-identical 區塊搬移、先備份)搬進 `hosts.config`。
   `Host *` 等 wildcard 區塊是 config 結構、裝置本地,**不搬**。提供選項「以原檔名
   加上 tag」(例如 `homelab.config` → tag `homelab`),讓依 tag 分組能延續原本的
   檔案分組。搬移後空掉的 Include 檔保留不刪。
2. **金鑰**:列出 `~/.ssh` 現有金鑰供勾選是否同步;**全部預設不勾**(§2 的逐把 opt-in);
   「被已搬入主機的 `IdentityFile` 引用」的那些只標示「recommended」;硬體/無私鑰檔者不可勾。
3. **密碼**:若本機開啟「Sync passwords」,把已搬入主機在 keychain 的密碼建成記錄。

**後續裝置加入時的既有資料處理**
- 該裝置本地已有主機:Include 插在主檔**最頂端**(§3.1)→ 依 ssh 的 first-obtained-wins
  語意(**逐選項**,不是整個區塊):同步檔那份先被讀到,它設定過的選項優先;本地那份仍會補上
  同步版沒設的選項,`IdentityFile` 之類可累加的選項更會兩邊相加。所以同名主機不是「被遮蔽就
  沒事」,而是兩份定義混在一起,必須處理。加入 wizard 列出重複(alias + 本地檔案),
  逐筆選擇:保留本地(把本地區塊改名 `<alias>-local`)/ 改用同步版(移除本地區塊,先備份;
  UI 要說明本地那份提供的選項會消失)/ 稍後處理(維持混合並在 sidebar 標示)。
- 本機同步檔裡若殘留上一次成員期的區塊:加入後的**基線輪**(§6)以 chain 為準 —— chain 上仍
  存在的以 chain 版本覆蓋,chain 上已刪除的移除(先備份),chain 不認識的當新主機上傳。這兩個動作走專用命令
  `sync_resolve_shadowed(alias, file, action)`,**以檔案路徑明確定位**要改的那個區塊;
  既有的 `config_rename_host`/`config_remove_host` 以「第一個命中」定位,會依 Include 順序
  誤中同步檔那份,不可用於此。
- 搬進受管檔的防護(wizard 與 sidebar 拖進「Synced」都適用):要搬的區塊的**任一** pattern 已被同步檔
  裡任一區塊使用 → 拒絕(兩個區塊的第一個 alias 相同時指向重複處理面板;其他撞名請使用者先在本地
  主機移除或改名該名稱);加入後第一輪同步(基線輪)完成前 → 拒絕;沒有同步引擎鎖的 process → 拒絕。
  受管檔的不變式仍只要求「第一個 alias 不重複」(記錄的 key),不因次要名稱重疊而停止同步 —— 其他
  裝置同步來的合法重疊不應讓整條同步停擺。wizard 排除任一 pattern 已在同步檔裡的主機、只送出畫面上
  看得到的選取,只有 chain 沒有主機時(例如剛 Create)才預選全部;`SyncStatus.first_sync_pending`
  (加入中且基線未建立)為真時停用 Move;`sync://applied` 也讓重複清單重新查詢。
- Include 置頂的附帶效果:搬進同步檔的主機會在主 config 之前被讀到,它自己設定的選項從此優先於
  主 config 前段的 wildcard 區塊(如 `Host *`);wizard 以文案告知。「以原檔名加上 tag」只替來自
  被 Include 檔案的主機加 tag,主 config 的主機不加(否則全都會是 `config`)。
- 本地已有同名但不同內容的金鑰檔:不覆蓋,提示改名後重試(§3.3)。
- 本地 keychain 已有同 alias 密碼且「Sync passwords」開啟:以記錄 LWW 決定,
  並在完成頁列出被覆蓋的項目。

**資料格式版本化**
- 每筆記錄 payload 含 `schema: 1`;chain 有一筆 `meta` 記錄(`schema_version`、
  `created_by_app_version`)。client 把收到的 `schema_version` 持久化為
  `remote_schema_version`,每輪據此判斷:本 app 支援版本 < 它 → 唯讀模式
  (仍套用可理解的記錄、不上傳、顯示「請更新 SSHelter」)。未知 `kind` 與本版不處理的
  種類一律以原始密文保留在 `sealed`(§6),不解密、不丟棄。
- `sync-state.json` 含 `version`,讀取時逐版遷移;無法解析 → 視為未加入並提示。
- 前端 settings envelope 不新增欄位(sync 狀態全在 Rust 端),既有匯出/匯入不受影響;
  `hosts.config` 建立時若 `fileAliases` 無設定則預設顯示名「Synced」。

**降級與退出的安全網**
- `hosts.config` 只是一般 ssh config + 一行 `Include`,**退回舊版 SSHelter 或純
  OpenSSH 都照常運作**,主機不會消失。
- Leave chain 保留所有本機檔案;「Move out of synced」可把主機搬回任一檔案。
- 既有 MCP allowlist 以 alias 為 key,主機搬移檔案不影響。
- 既有部署功能寫回的 `IdentityFile ~/.ssh/<name>` 與 §3.3 的金鑰路徑規則一致,
  舊資料無需轉換。

## 11. 交付分期

1. **Phase A — 核心**:crypto、relay(Worker)、sync engine(host + device 記錄)、
   Settings Sync pane、配對/上手流程、受管同步檔 + Include。
2. **Phase B — 憑證**:key 記錄(opt-in)、password 記錄(全域開關)、警示。
3. **Phase C — 輪替**:遠端移除 script、批次部署對話框、輪替流程。

每個 phase 可獨立發版;A 出貨後就已經是「Brave 式主機同步」。
