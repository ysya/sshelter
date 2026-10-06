# SSHelter Sync v2 — Spaces 與單一同步碼(SP1 設計)

日期:2026-10-02。前置:sync chain Phase A(`2026-09-27-sync-chain-design.md`,0.16.0 起以 Beta 發佈)、
更新頻道(`2026-10-01-update-channels-design.md`)。已經過 Codex 獨立審查,處理結果見 §12。

## 0. 定位

Sync v2 拆成四個子專案,各自一份 spec → 計畫 → 實作。**本文只涵蓋 SP1。**

| 子專案 | 內容 |
|---|---|
| **SP1 Sync v2 基礎(本文)** | Space、一組同步碼、每台裝置自選 space、批次查詢、更換同步碼、v1 升級、危險設定核准、更新已部署的 relay |
| SP2 Group 與結構化主機 | space 內的多層群組、群組預設值、逐欄合併 |
| SP3 SSH 金鑰 | 金鑰插槽與金鑰同步(`2026-10-05-sp3-key-slots-design.md`;取代原本「每台一把金鑰與自動部署/撤銷公鑰,或 SSH agent」的規劃) |
| SP4 分享 | 把單一 space 分享給其他人(以對方公鑰加密 space 金鑰) |

v1 spec 中尚未實作的 Phase B/C(私鑰與密碼同步 §2/§3.2、金鑰輪替 §8)由 SP3(私鑰同步;密碼不同步)與本文的「更換同步碼」
(金鑰輪替)取代,不再照 v1 spec 實作。

## 1. 目標與非目標

**目標**
- 同步後保留專案區隔:每個 space 在有勾選它的電腦上是一個獨立的 config 檔。
- 一組同步碼:新電腦輸入一次;任何已加入的電腦都能查看。沒有救援碼、沒有帳號。
- 每台電腦自選要同步哪些 space。
- 每個 space 有自己的金鑰與 relay 位置(SP4 分享的前提)。
- 電腦遺失時「更換同步碼」,遺失的電腦之後讀不到新內容、也寫不進來,且更換過程不遺失任何已上傳的修改。
- 同步來的主機若帶有會在本機執行程式或轉送憑證的設定,本機核准後才寫入。
- relay 查詢的 HTTP 請求數不隨 space 數量增加。
- 現有 v1 使用者自動升級,不需重新輸入同步碼。

**非目標**
- 自己的電腦之間的加密邊界:同一帳戶的每台電腦都能打開所有 space,勾選只決定要不要同步下來
  (與 1Password、Bitwarden、Termius 相同的模式)。
- 子群組、逐欄合併(SP2);SSH 金鑰(SP3);分享給其他人(SP4)。
- 即時推送;同步使用者自己的 config 檔。

## 2. 已定案的決策

| 決策 | 選擇 | 理由 / 被否決的選項 |
|---|---|---|
| 帳戶憑證 | 沿用 v1 的 24 詞同步碼,可在已加入的電腦上查看 | 使用者不要另一組無法查看的救援碼。否決「裝置核准 + 救援碼」:需要不放在任何裝置上的主鑰,救援碼因此不能查看 |
| 自己電腦之間的邊界 | 無(業界模式) | 否決「每個 space 一組 code」:每台電腦要維護多組碼,不實用 |
| Space 金鑰 | 每個 space 隨機產生,存在帳戶資料中 | 不由同步碼推導,SP4 才能只分享一個 space。例外:v1 升級的第一個 space 由同步碼推導(§5.2) |
| relay | 新增 `POST /v1/pull`(批次)、`POST /v1/chains/{id}/freeze`(凍結)、`GET /v1/info`;既有端點不變 | 否決「帳戶內放變動提示、只查有變動的 space」:多一層一致性問題,延遲最壞 10 分鐘 |
| 主 config 載入 space 檔 | 由 app 維護的**明確** Include 清單,只列勾選的檔案 | 否決目錄 glob:OpenSSH 會載入目錄裡任何 `.config`,包含殘留或未勾選的檔案 |
| space 檔名 | `<slug>-<space id 前 8 字元>.config` | 不需跨裝置協調 slug 唯一性,並發建立或改名不會撞名 |
| 撤銷遺失的電腦 | 更換同步碼(relay 凍結舊資料 → 完整複製 → 刪除),其他電腦各輸入一次新碼 | 罕見操作;換來不需要裝置金鑰與救援碼 |
| 危險設定 | 遠端新增或修改時需本機核准;同步的主機區塊禁止 `Include` | v1 已有同樣風險(任何持碼者都能推送 `ProxyCommand`) |
| relay 更新 | `GET /v1/info` 偵測 + 「Update relay」workflow(開 PR,不直接部署)+ wrangler / Docker 說明 | 部署按鈕建立的是獨立複本,不會自動更新;自動部署上游程式碼的供應鏈風險太高 |

## 3. 安全模型

- **同步碼 = 帳戶**:持有者能讀寫該帳戶的所有 space。同步碼存在 OS keychain(account `sync:mnemonic`,與 v1 相同),
  Sync pane 的「Show sync code」可查看;狀態檔不存同步碼。
- **relay 零知識**:只看到密文、chain id、序號、大小、時間、IP。HTTPS only(loopback 例外)、client 不跟隨 redirect
  (沿用 v1 §2)。
- **狀態檔不含明文祕密**:space 的權杖與金鑰在狀態檔裡只以帳戶金鑰加密的 envelope 保存,啟動時在記憶體解開(§4.4)。

| 情況 | 能做 | 不能做 |
|---|---|---|
| relay 被入侵 | 看到密文與 metadata;刪除資料或拒絕服務 | 讀取或竄改內容 |
| 網路中間人 | — | 讀取或冒充(HTTPS、不跟隨 redirect) |
| 電腦被偷(更換同步碼前) | 讀寫該帳戶所有 space | — |
| 電腦被偷(更換同步碼後) | 保有被偷當時的資料 | 讀到新內容、寫入新帳戶 |
| 帳戶裡的惡意成員 | 修改主機的 `HostName`、`User`、`Port`、`ProxyJump`、`LocalForward`、`DynamicForward`,把連線導向別處(未知主機金鑰仍會由 ssh 提示;關掉這個提示的設定受 §7.4 管制) | 未經本機核准讓 §7.4 的設定生效;透過 `Include` 引入其他檔案;藉由主機名稱、使用者等值讓 ssh 執行本機指令(§7.4 拒絕含 shell 字元的值) |
| SP4 之後被分享單一 space 的人 | 讀寫那個 space | 讀到其他 space |

## 4. 資料模型

### 4.1 帳戶 chain

relay 上的一條 chain,位置、權杖、金鑰由同步碼推導(§5.1)。記錄格式與 v1 相同(`Record`:kind、id、version、
updated_at_ms、device_id、deleted、payload),以帳戶金鑰加密。種類:

| kind | id | payload |
|---|---|---|
| `device` | device_id | v1 `DevicePayload` 加上 `spaces: Vec<String>`(這台勾選的 space id;serde default 空) |
| `space` | space id | `{ schema: 1, name, slug, created_at_ms, previous_id: Option<String> }`;刪除 = tombstone |
| `spacekey` | space id | `{ schema: 1, auth_token, enc_key }`(`enc_key` 為標準 base64);刪除 = tombstone |
| `meta` | `"account"` | `{ schema_version: 2, created_by_app_version }` |
| `meta` | `"rotation:<device_id>"` | 只出現在被淘汰的舊帳戶 chain:`{ rotated_at_ms, by_device_id, by_device_name }`(§7.5) |

- **space id = 該 space 的 chain id**(64 字元小寫 hex)。更換同步碼後 id 會變,新記錄以 `previous_id` 指向舊 id。
- `name` 用於顯示,UI 在帳戶內盡量避免重名,但不作為一致性保證。`slug` 只用來組檔名(§4.3),不要求唯一。
- 未知 kind 照 v1 規則以原始密文保存在 `sealed`,不解密、不刪除。

### 4.2 Space chain

每個 space 一條 chain;chain id、權杖(皆為 32 bytes 隨機值的小寫 hex)與 32-byte 金鑰隨機產生,寫進帳戶的
`spacekey` 記錄。內容只有 `host` 記錄(v1 `HostPayload { schema: 1, text }`,一個具名 Host 區塊的原始文字),
規則與 v1 相同(`validate_host_text`、wildcard 區塊不同步),**另外禁止區塊內出現 `Include`**(§7.4)。

### 4.3 本機檔案

- `~/.ssh/sshelter/`(0700);每個**有勾選**的 space 一個檔案(0600),檔名 `<slug>-<space id 前 8 字元>.config`。
- `slug`:由 name 產生 → 小寫 → 非 `[a-z0-9]` 的連續字元換成 `-` → 去掉頭尾 `-` → 最長 40 字元;空字串 → `space`。
  因為帶 space id,不同 space 的檔名永遠不同;Windows 保留名也不會出現(檔名一定帶 `-<hex>`)。
- **主 config 最頂端只放一行我們的 Include**,明確列出所有勾選檔案的路徑(依 space 名稱不分大小寫排序,再依名稱原字串、space id,讓每台的順序相同),
  例如 `Include ~/.ssh/sshelter/work-3fa2c1d9.config ~/.ssh/sshelter/personal-8b01e4aa.config`。沒有勾選任何 space
  時移除這一行。`ensure_include` 改寫:位置規則不變;辨識「我們的 token」= `~/.ssh/sshelter/` 這一層、以 `.config` 結尾
  (含 v1 的 `hosts.config` 與 `*.config` 這類 glob;前綴之後再有路徑分隔字元的 —— 子目錄或 `..` —— 是使用者自己的),
  整批換成目前的清單,其他 token 留在原地(同 v1 的多路徑處理)。
- **順序規則**(讓 OpenSSH 永遠不會讀到半成品,也不會讀到不該讀的檔案):
  - 勾選:先建立並寫好檔案 → 再加進 Include 清單。
  - 取消勾選、刪除 space:先從 Include 清單移除 → 再備份並刪除檔案。
  - 改名:以 hard link 建立新檔名(目標路徑已存在 → 原子地失敗:放棄改名、保留舊檔名並在狀態列提示,不覆蓋任何檔案)→
    更新 Include 清單 → 刪除舊檔名。檔案系統不支援 hard link 時退回「確認目標不存在 → rename → 更新清單」。
- 目錄裡不在 Include 清單上的檔案,OpenSSH 不會讀;app 不讀、不改、不刪,只在狀態列提示。
- 每個檔案各自檢查 v1 的不變式(`check_managed_items`:只放具名、互不重複的 Host 區塊),**另外不得含 `Include`**。
  違反時**只暫停該 space**。
- 同一個 alias 出現在兩個檔案(兩個 space,或 space 與自己的設定檔):同步照常。ssh 套用每一個符合的 Host 區塊,每個設定取
  第一個讀到的值:排在前面的區塊設定過的,以它的值為準;只有後面那份設定的(包括 ProxyCommand 等受管制的設定)仍然生效,
  可累加的設定(IdentityFile、LocalForward、RemoteForward、DynamicForward、SendEnv 等)兩份都會用。UI 照這個說明,標示
  排在後面的那份並提供改名或刪除;一個名稱有不只一份時,編輯、刪除、搬移不得默默作用在使用者沒選的那一份。

### 4.4 本機狀態

`sync-state.json` 升為 `version: 2`(`atomic_write` 0600):

```text
{ version: 2, device_id, device_name, relay_url,
  account: { chain_id, cursor_seq, baseline_established, remote_schema_version, frozen: Option<FreezeInfo>,
             records (device/space/meta 明文), sealed (spacekey 與未知種類的密文 envelope) },
  spaces: { <space id>: { selected, file_name, cursor_seq, baseline_established,
                          records (host 明文), pending_approvals, last_error } },
  rotation: Option<RotationProgress>, legacy_v1_backup: Option<String>,
  relay_features: Option<{ url, version, features, checked_at_ms }>, phrase_cleanup_pending, last_sync_ms, last_error }
```

- 讀不懂、損毀、I/O 錯誤:沿用 v1 §6 的處理(改名保留、不寫狀態、提示重啟)。
- `spaces` 只保存這台勾選的 space;未勾選的 space 只存在帳戶的 `space`/`spacekey` 記錄中。
- `frozen`:這台偵測到帳戶已被更換同步碼(§7.5),記下標記內容;非 `None` 時不做任何網路寫入。
- 「狀態寫不進磁碟」的 `unsaved` 旗標只放在記憶體(同 v1),不寫進檔案;`phrase_cleanup_pending` 供 §7.3 的離開流程使用。

## 5. 加密(byte-level;實作以已知答案向量釘住)

### 5.1 同步碼推導

- 正規化與 seed:同 v1 §4(BIP39 英文 24 詞;`seed = bip39::Mnemonic::to_seed("")`,64 bytes)。
- `PRK = HKDF-Extract(salt = ASCII "sshelter-sync-v2", IKM = seed)`,各展開 32 bytes:

| info(ASCII) | 用途 | 編碼 |
|---|---|---|
| `sshelter/v2/account/chain-id` | 帳戶 chain id | 小寫 hex |
| `sshelter/v2/account/auth` | 帳戶權杖(relay 只存 `SHA-256(權杖的 ASCII hex)`) | 小寫 hex |
| `sshelter/v2/account/enc` | 帳戶金鑰 | raw |
| `sshelter/v2/space0/chain-id` | 第一個 space(v1 升級)的 chain id | 小寫 hex |
| `sshelter/v2/space0/auth` | 第一個 space 的權杖 | 小寫 hex |
| `sshelter/v2/space0/enc` | 第一個 space 的金鑰 | raw |

- salt 與 v1 不同,所以同一組同步碼的 v1 chain 與 v2 帳戶 chain 互不相關。
- 由 v2 新建的 space 不使用推導值:`chain_id`、`auth_token` = 32 bytes 隨機值的小寫 hex,`enc_key` = 32 bytes 隨機值
  (OS CSPRNG,`getrandom`)。

### 5.2 為什麼第一個 space 要推導

兩台電腦可能同時從 v1 升級。推導值讓它們建出**同一個** space(同一個 chain id、同一把金鑰)。space0 的 `space`
payload 也固定為確定值:`{ schema: 1, name: "Synced", slug: "synced", created_at_ms: 0, previous_id: null }`;
`spacekey` payload 由推導值決定。兩台寫入的記錄 payload 相同,只有記錄 metadata(時間、device_id)不同;relay 若回
push conflict,照一般流程 pull、合併、重試(不能把 `PUT` 冪等推論成後續寫入也冪等)。之後新建的 space 一律隨機。

### 5.3 記錄加密

與 v1 完全相同,只是 (chain_id, 金鑰) 換成帳戶 chain 或該 space 的值:

- `id_hash = hex(HMAC-SHA256(key, kind || 0x0A || id))`
- XChaCha20-Poly1305,每筆隨機 24-byte nonce,`AAD = chain_id || 0x0A || kind || 0x0A || id_hash`(全 ASCII)
- 明文 = 整筆記錄 JSON;解密後驗證明文的 kind/id 算出的 `id_hash` 與 envelope 相同。

### 5.4 已知答案向量

以 `abandon ×23 art` 釘住:帳戶 chain id、帳戶權杖、`id_hash` 以帳戶金鑰計算的 `("space", "<space0 chain id>")`、
space0 chain id、space0 權杖、`id_hash` 以 space0 金鑰計算的 `("host", "web-1")`。v1 向量不變。

## 6. relay

### 6.1 `POST /v1/pull`(批次查詢)

- 不帶 `Authorization` header;body 為 JSON 陣列,1–64 項,每項 `{ chain, token, since }`:`chain`、`token` 為
  64 字元小寫 hex,`since` 為 0 ≤ n ≤ 2^53 − 1 的整數(JavaScript 的安全整數)。同一批 `chain` 不得重複。body 以串流讀取,上限 64 KiB。
  格式錯誤 → 整批 `400`。
- 回應 `200 { results: [...] }`,順序與請求相同,每項:
  - `{ chain, status: "ok", records, latestSeq }`:與 `GET .../records?since=` 相同的語意與內容;
  - `{ chain, status: "not_found" }`:chain 不存在或權杖不符(不洩露差別;不配置儲存);
  - `{ chain, status: "rate_limited" }`:該 chain 每分鐘 120 次的限制;
  - `{ chain, status: "deferred" }`:本批回應已達預算,這項沒有執行。
- **回應預算**:依序處理;每完成一項,累計該項序列化成 JSON 後的位元組數。第一項一律放入(單一 chain 上限 1 MiB 密文,
  序列化後仍在可處理範圍);之後若累計已超過 2 MiB,剩下的項目一律 `deferred`。因此回應最多約 2 MiB 加一條 chain。
- 每項各自走該 chain DO 的 `pull`:各自驗證權杖、各自計入每分鐘限制、各自刷新閒置期限。
- 每 IP 限制:整批在既有 `request` 桶計 1;另有新的 `pull` 桶,**每項計 1**,每小時 12000。兩桶在 body 驗證後以
  **一次**限流 DO 呼叫計入(格式錯誤的批次只計 `request` 桶);任一桶超過 → 整批 `429`;被拒絕的批次照樣計入。
  批次只減少 HTTP 往返;relay 的工作量(每條 chain 一次 DO 呼叫)仍與 chain 數成正比。
- 單一 chain 的 DO 發生例外會讓整批回 `5xx`(不逐項攔截);client 的處理見 §6.4。

### 6.2 `POST /v1/chains/{id}/freeze`(凍結)

- `Authorization: Bearer <該 chain 的權杖>`;成功 `204`;權杖不符或 chain 不存在 `404`;超過該 chain 每分鐘 120 次
  → `429`;冪等。
- 凍結後:該 chain 的 push 一律回 `409 { status: "frozen" }`,不寫入任何記錄;pull、批次 pull 照常;`PUT`(建立)
  回 `200` 但維持凍結。`DELETE` 只清掉記錄,保留 `meta`(權杖 hash、`latest_seq`、凍結)與閒置期限:之後的 `PUT`
  仍回 `200`、push 仍回 `409`,舊裝置不可能把它當成新 chain 重新寫入。未凍結的 chain 的 `DELETE` 照舊整條清除
  (之後 `PUT` 回 `201`)。凍結狀態存在該 chain 的 `meta` 表,不可解除;閒置 180 天整條清除時才一併消失。
- 用途:更換同步碼時建立 relay 強制的寫入截止點(§7.5)。

### 6.3 `GET /v1/info`

- 不需權杖;回應 `200 { relay: "sshelter-relay", version: "<RELAY_VERSION>", features: ["pull-batch", "freeze"] }`;
  計入 `request` 桶。舊版 relay 會回 `404`(router 的 not found)。
- `RELAY_VERSION` 是 `relay/src/index.ts` 的常數,與 `relay/package.json` 的 version 相同(測試檢查)。

### 6.4 client 行為

- 決定 relay 能力:每次 app 啟動及 relay URL 改變時呼叫 `GET /v1/info`,結果存在 `relay_features`。
  - 缺少 `pull-batch`(含 `404`)→ 退回逐條 `GET`:帳戶 chain 每輪查、space 每 3 輪查一次;Sync pane 顯示
    「relay 可以更新」,連到 README「Updating your relay」。
  - 缺少 `freeze` → 「更換同步碼」停用,說明需要先更新 relay。
- 輪詢間隔 = max(45 秒, 2 秒 × 本輪要查的 chain 數),**只在視窗有焦點或使用者幾分鐘內有操作時**;否則約每 5 分鐘
  一輪。視窗重新取得焦點、本機修改上傳之後立刻輪詢一次。例:10 個 space → 45 秒;64 個 → 130 秒。這讓單台電腦每小時
  的 `pull` 項數維持在約 1,800 以下,同一 NAT 後的多台電腦仍遠低於 12000。
- **Cloudflare 免費方案**:每個帳號每天 100,000 次 Durable Object 請求與 100,000 列寫入。每輪 = 1 次限流 DO + 每條
  chain 1 次 DO;每輪的限流計數寫 1 列,讀取刷新閒置期限最多每 6 小時寫一次。焦點規則讓幾台電腦、少量 space 的日常
  用量留在免費額度內;README 說明這個上限。
- 批次回 `5xx`:連續 2 次後,這一輪改成逐條 `GET`,壞掉的 chain 不擋住其他 chain;之後指數退避。
- 每輪一個批次請求(帳戶 chain + 所有勾選的 space;超過 64 條就分批)。有 `deferred` 時,**只**把 deferred 的項目
  排在最前面立刻再發一批;沒取得的 chain 不推進 cursor。
- `429`(整批或 `rate_limited`):間隔依序 90 秒、3 分、6 分,最長 15 分鐘;成功一輪後恢復正常間隔;狀態列說明原因。
  退避期間,存檔與視窗取得焦點不會提早開始一輪;使用者按「Sync now」仍立刻執行。
- push 回 `409 frozen`:這一輪立刻停止所有上傳,保留 dirty,下一輪拉帳戶 chain 確認更換標記(§7.5)。
- 上傳、建立、刪除、凍結仍是逐條呼叫。`RelayClient` 改成每次呼叫帶入 chain 的權杖(現在一個 client 綁一個權杖)。

### 6.5 更新已部署的 relay

- **「Update relay」workflow**:`relay/.github/workflows/update-relay.yml`,邏輯在 `relay/.github/scripts/update-relay.mjs`
  (只用 Node 內建模組)。部署按鈕複製 `relay/` 時會一起帶到使用者的 repo;在本 repo 中它們位於子目錄,不會被 GitHub 執行。
  - 觸發:`workflow_dispatch`,輸入 `ref`(預設:上游最新正式 release 的 tag;可填 `main` 或 beta tag)與
    `upstream`(預設 `ysya/sshelter`)。另有每週 `schedule`,只在 repository variable `AUTO_UPDATE == 'true'` 時執行。
  - **兩個 job(信任邊界)**:上游程式碼(`npm ci` 的安裝腳本、測試)只在 `test` job 執行,它只有 `contents: read`——
    同一台 VM 上的程式能植入 git hook / fsmonitor、寫 `$GITHUB_ENV`、用 sudo,寫入權杖不能和它同一個 job。`open-pr` job
    (`contents: write`、`pull-requests: write`,`needs: test`)重新 checkout、重新解析 `ref` 並要求與測試過的 SHA 相同、
    重新下載,只用 repo 自己的腳本複製檔案,不執行 tarball 裡的任何東西,最後開 PR。workflow 層級 `permissions: {}`;
    checkout 一律 `persist-credentials: false`;兩個 job 之間不傳 artifact。
  - 套用:把 `ref` 解析成 commit SHA → 下載 tarball → 以 repo **現有**的腳本(先複製到 `$RUNNER_TEMP`)比較版本、以上游
    `relay/` 覆蓋、把 Worker `name` 寫回(先對上游 `wrangler.jsonc` 驗證寫得回去,不行就不動任何檔案)。`.github/` 底下
    只更新這支腳本與它的測試;其他檔案(含 owner 自己的)不動、不刪。
  - **workflow 檔本身永遠不寫**:GitHub 不允許 `GITHUB_TOKEN` 推送 `.github/workflows/*` 的變更。上游版本不同時,run 發出
    警告,PR 內文請使用者手動複製。因此舊版 workflow 會執行新版腳本:腳本必須持續接受
    `apply <upstream> <repo> [--allow-downgrade]` 與 `pr-body <summary.json> <upstream> <ref> <sha>`,摘要保留 `skipped`、
    `message`、`workflowOutdated`、`direction`、`fromVersion`、`toVersion`;relay 版本保持 `X.Y.Z` 核心。
  - **不降版**:比較 `relay/package.json` 的版本(只看 `X.Y.Z` 核心)。沒有指定 `ref` 而上游較舊 → 不做任何事並說明;
    明確指定較舊的 `ref` → 開「Downgrade relay A → B」PR,內文警告。
  - PR:分支 `update-relay/<SHA 前 12 字元>`;同一個上游 commit 已有 PR(開啟、關閉或已合併)就不再開;標題
    「Update relay A → B (ref)」;內文列出上游 commit、被刪除的檔案、Durable Object binding 或 migration 的變動、workflow
    是否要手動更新、腳本是否變更(它在下一次更新時以寫入權限執行,要看)。**workflow 永遠不直接推送到預設分支**;使用者
    merge 後 Cloudflare Workers Builds 才部署。Worker 名稱不變,網址與 Durable Object 資料不變。不需要 Cloudflare 權杖。
  - repo 不允許 Actions 建立 PR:run 失敗並給出手動開 PR 的連結與設定位置;PR 內文在 run summary。
  - CI:`node --test` 跑腳本測試;actionlint(含 shellcheck)檢查 `.github/workflows/relay.yml` 與這支 workflow。
  - **合併到 main 前的手動檢查**:從 `…/tree/feat/sync-v2-spaces/relay` 用部署按鈕部署一次,確認 `.github/` 的檔案有被
    複製,改 Worker 名稱後執行 workflow,確認開出的 PR 正確。若按鈕不複製 `.github/`,README 改成請使用者手動加入。
- **README「Updating your relay」**:部署按鈕(GitHub)→ 執行 workflow 並 merge PR;**更早部署、repo 裡沒有這個
  workflow** → 把 workflow 與腳本兩個檔案複製到 relay repo 後執行(建議);或在本 repo 的 checkout 先把 `wrangler.jsonc`
  的 `name` 改成自己的 Worker 名稱再 `npx wrangler deploy`(否則會建出第二個、資料是空的 Worker),並且仍要加入那兩個
  檔案——relay repo 的下一次 push 會以它自己的程式碼重新部署;wrangler 部署 → `git pull` 後再 deploy;Docker →
  `git pull && docker compose up -d --build`;GitLab → 手動步驟。

### 6.6 不變的部分

- 既有端點、每 chain 1 MiB / 4096 筆、每 chain 每分鐘 120 次、每 IP 每小時 20 次建立 chain 與 1200 次請求、
  閒置 180 天清除(批次查詢的項目也算活動)。
- 實作調整(不改語意):讀取刷新閒置期限改以 alarm 時間節流(alarm 已超過 6 小時沒動才重設),DO 休眠後仍有效;每個請求
  只呼叫一次每 IP 限流 DO(`PUT` 一次計入 `request` 與 `create`);限流 DO 的清除 alarm 只在某個桶開始新視窗時移動,
  觸發時所有視窗都已到期。
- 每 IP 每小時 20 次建立:更換同步碼時若需要建立超過 20 條 chain,操作暫停,下一個小時自動接續(§7.5)。

## 7. 同步引擎

### 7.1 一輪的流程

沿用 v1 §6 的結構與保證(lifecycle → doc → core 的鎖順序、`sync.lock`、generation、指紋比對、持久化後才推進),
從單一檔案推廣到「帳戶 chain + 多個 space」:

1. `frozen` 非 `None` → 不做任何網路寫入,只顯示 §7.5 的提示。
2. 讀取每個勾選 space 的檔案、檢查不變式、記下指紋與 mtime。違反不變式的 space 這一輪跳過,其他照常。
3. 各 space 做本機 diff(外部編輯)→ dirty 記錄並持久化(同 v1)。
4. 一個批次請求取得帳戶與各 space 的新記錄(§6.4);逐筆解密、驗證、LWW 合併(規則同 v1)。**帳戶的結果一律先處理**:
   其中只要有 `meta` `rotation:*` 標記,記下 `frozen` 並持久化,這一輪立刻停止(不套用任何帳戶變化、不處理任何 space
   結果、不上傳)。
5. **先提交帳戶**:套用 `space` / `spacekey` / `device` / `meta` 的變化(新 space、改名、刪除、金鑰),持久化。
6. **再逐一提交 space**:每個 space 的「套用 + 發布」是獨立交易(v1 `apply_and_commit` 的語意,全有或全無),比對該
   space 檔案的指紋。**提交只對最新的記憶體狀態做局部更新**(只改該 space 的區段與它的 cursor),在 doc/core 鎖內檢查
   generation;絕不以本輪開始時取得的整份狀態副本覆蓋——否則會蓋掉已提交的帳戶或其他 space 的 cursor。任何一個
   space 失敗不回退其他已提交的部分。
7. 上傳 dirty(帳戶與各 space 分開;批次規則同 v1:每批 ≤ 200 筆且 ≤ 512 KiB)。
8. 發事件:`sync://status`、`sync://applied`、`sync://conflict`(payload 加上 space 名稱)、`sync://approval`(§7.4)。

- 單一 generation:任何生命週期或結構性變更(加入、離開、更換同步碼、建立 / 改名 / 刪除 space、勾選改變)都換
  generation,在途的輪次整輪丟棄重跑。
- `note_file_written`:依路徑找出所屬 space,在存檔當下規劃該 space 的 dirty 記錄(同 v1)。
- 受管檔消失或被清空:以 space 為單位沿用 v1 的「從 chain 重建」規則。

### 7.2 Space 操作

- **建立**:產生 chain id、權杖、金鑰 → `PUT` 建 chain → 寫入 `space` 與 `spacekey` 記錄 → 建立者預設勾選
  (依 §4.3 順序建立檔案、加進 Include,更新自己的 `device.spaces`)。
- **改名**:更新 `space` 記錄;每台勾選者在 doc 鎖內依 §4.3 改名檔案並更新 Include(先備份、更新指紋)。
- **刪除**:確認後 → tombstone `space` 與 `spacekey` → `DELETE` 該 chain。各台收到 tombstone 後依 §4.3 移除 Include
  與檔案(先備份),刪掉該 space 的狀態,並顯示「Work 已在 MacBook-A 刪除」。
- **勾選**:依 §4.3 建立空檔並加進 Include → 該 space 以基線輪開始(chain 為準,同 v1 基線規則)→ 更新 `device.spaces`。
- **取消勾選**:確認後依 §4.3 移除 Include 與檔案(先備份)、刪掉該 space 的狀態、更新 `device.spaces`;relay 與其他
  電腦不受影響。
- **跨 space 搬移主機**:同一個 doc 鎖臨界區內**先寫入目標檔、再從來源檔移除**(目標寫入失敗就不動來源;來源移除
  失敗時主機暫時同時存在兩邊,由 §4.3 的衝突標示提醒)。兩個檔案各自經 `note_file_written` 產生記錄。只能搬到這台
  有勾選的 space。
- **從自己的 config 檔搬入**(搬移精靈):沿用 v1 的搬移管線,目標改為選定的 space;選項「一個來源檔建立一個 space」
  以來源檔名作為 space 名稱(檔名 alias 優先)。含 `Include` 的區塊不能搬入,精靈列出原因。relay 對建立 space 回 `429`
  之後,剩下的組不再送出建立,列為失敗並說明約一小時後再試(§6.4)。

### 7.3 帳戶生命週期

- **建立帳戶**(第一台、非 v1 升級):產生同步碼 → 推導帳戶 chain → `PUT` 建立 → 存同步碼進 keychain → 寫入
  `meta` `account` 與 `device` → 建立預設 space「Personal」(隨機產生,不是 space0)並勾選 → 顯示同步碼,請使用者
  存進密碼管理器(之後也能隨時查看)。
- **加入帳戶**(新電腦):輸入同步碼 → 推導帳戶 chain → `GET` 驗證存在(`404` → 「找不到這組同步碼的帳戶」,不建立)
  → 存同步碼進 keychain → 拉取帳戶(若帶有 `rotation:*` 標記 → 提示「這組同步碼已被更換」,不加入)→ 讓使用者勾選
  space → 寫入 `device` 記錄 → 各勾選 space 以基線輪開始。
- **離開帳戶**(這台):同 v1 Leave——先清掉狀態檔的帳戶部分並持久化,再刪 keychain 的同步碼(失敗時
  `phrase_cleanup_pending` 與重試入口同 v1)。本機 space 檔案與 Include 保留,ssh 照常可用;它們之後就是一般的本機檔案。
- **刪除帳戶**:離開時若是裝置清單上的最後一台,詢問是否一併 `DELETE` 帳戶 chain 與所有 space chain。
- **relay URL**:只能在未加入時更改(同 v1 §6;cursor 與 seq 屬於某一個 relay)。要換 relay:離開 → 改 URL →
  建立新帳戶,再用搬移精靈把本機的 space 檔案搬進新帳戶的 space。

### 7.4 危險設定核准

- **禁止**:同步的主機區塊不得含 `Include`,也不得有 keyword 帶引號的行或以 `=` 開頭的行(`validate_host_text` 拒絕 →
  遠端記錄視為 Skip,不進快取;space 檔內任何位置出現 → 該 space 違反不變式)。`Include` 引入的檔案內容在核准之後仍可改變,
  無法靠核准涵蓋;OpenSSH 會去掉 keyword 的引號照樣執行(`"ProxyCommand" …`),也會跳過行首的 `=`、把下一個字當成
  keyword(`=Include …`、`=ProxyCommand …`、`=Host *`,皆以 OpenSSH 10.3 的 `ssh -G` 實測),解析器卻把前者的 keyword
  連同引號保留、後者的 keyword 解析成空字串,只有拒絕才不會漏判。
- **會被代入指令的值**:ssh 會把 `HostName`(`%h`)、`User`(`%r`)、`HostKeyAlias`(`%k`)、`ProxyJump`(`%j`)不加引號地
  代入 `ProxyCommand`、`LocalCommand`、`KnownHostsCommand` 與 `ProxyJump` 自己產生的 `ssh -W` 指令;ssh 只在命令列上
  拒絕含 shell 字元的主機名稱與使用者,設定檔裡的照收(OpenSSH 10.3 實測可執行任意本機指令)。同步的區塊因此要求這四個
  值以 OpenSSH 的斷詞(ASCII 空白分隔、以 `#` 開頭的字起算註解)只有一個字、不以 `-` 開頭、不含
  `' \` " $ \ ; & < > | ( ) { }` 與控制字元,否則同樣拒絕。
- **少見的字元**:keyword 或值(OpenSSH 當成註解的部分除外,所以註解裡的全形空白照常可用)含非 ASCII 空白
  (U+00A0、U+2000–U+200A、U+3000 等)、雙向文字控制或零寬字元(U+200B–U+200F、U+202A–U+202E、U+2060、U+2066–U+2069)
  或 tab 以外的控制字元的行,一律拒絕 —— 解析器把空白當成分隔,OpenSSH 不會;雙向與零寬字元會讓核准對話框顯示的文字
  和 ssh 讀到的不同。Unicode 的 Default_Ignorable 字元(含韓文填充字元 U+3164 等)與畫出來是空白的 U+2800、U+1D159、
  U+13441、U+13442、U+303F 也一樣拒絕:放在 `#` 前面會讓 ProxyCommand 的後半段看起來像註解,shell 卻照樣執行。
  ProxyCommand 等五個整行交給 shell 的 keyword,詞的開頭(值的開頭,或 ASCII 空白、`=`、`;` 等符號之後)不得是非 ASCII
  的記號或符號(例如沒有底字的組合記號,畫面上只是一個重音,卻讓 shell 不把後面的 `#` 當成註解)。核准對話框另外把
  這些字元與沒有底字的組合記號標示出來。
- **Host / Match 那一行**:以 OpenSSH 的斷詞(ASCII 空白分隔、以 `#` 開頭的字起算註解)得到的 pattern 必須和解析器的
  相同,且(OpenSSH 當成註解的部分以外)不得含引號或反斜線。`Host web#x *` 對 OpenSSH 是 `web#x` 與 `*` 兩個 pattern,解析器卻只看到 `web`(其餘當成
  行尾註解),會讓一筆看似只管 `web` 的記錄改掉每一台主機的 `HostName`(OpenSSH 10.3 實測),所以拒絕。
- **受管制的設定**(24 個,另加有綁定位址的轉送):
  - 會執行本機程式或載入本機程式庫:`ProxyCommand`、`LocalCommand`、`PermitLocalCommand`、`KnownHostsCommand`、
    `PKCS11Provider` 與它的別名 `SmartcardDevice`、`SecurityKeyProvider`、`XAuthLocation`;
  - 把本機憑證、環境或網路開放給遠端:`ForwardAgent`、`ForwardX11`、`ForwardX11Trusted`、`RemoteForward`、`SendEnv`、
    `GSSAPIDelegateCredentials`、`IdentityAgent`、`PermitRemoteOpen`;
  - 以使用者的身分在伺服器上執行指令:`RemoteCommand`;
  - 關掉主機金鑰提示(§3「導向別處仍會提示」的前提):`StrictHostKeyChecking`、`NoHostAuthenticationForProxyCommand`、
    `NoHostAuthenticationForLocalhost`、`VerifyHostKeyDNS`、`UserKnownHostsFile`、`GlobalKnownHostsFile`;
  - 把轉送的連接埠開放給區網:`GatewayPorts`;`LocalForward` 只有「兩個參數、第一個是單純埠號、第二個是 `host:port`
    (IPv6 以 `[...]` 括起)或絕對的 socket 路徑」時不受管制,`DynamicForward` 只有「唯一的參數是單純埠號」時不受管制,
    其他寫法一律受管制 —— OpenSSH 會把兩個參數接成 `第一個:第二個` 再解析,所以 `LocalForward 0 8080:host:80` 其實綁在
    0.0.0.0(OpenSSH 10.3 實測)。
- **判斷方式**:keyword 一律取 config 解析器解析出的 directive(已處理大小寫與 `Keyword=value` 寫法;帶引號的 keyword
  與行首的 `=` 已被上一條禁止,簽章仍把空 keyword 當成受管制),不以文字搜尋判斷。遠端記錄合併後要套用 `Upsert{alias, text}` 時,若新文字含任何受管制的設定,就計算「核准簽章」並與
  **目前已套用的本機區塊**比較:
  - 簽章 = (`Host` 那一行 keyword 之後的整段文字) + (所有受管制的 directive,依出現順序,每項為 keyword 小寫與 keyword
    之後的整段文字)。「整段」包含解析器當成行尾註解的部分:OpenSSH 的斷詞與解析器不同(`Host safe#x` 是一個 pattern;
    `ProxyCommand` 的整段文字交給 shell 執行),所以只改那一段也要重新核准。本機沒有這台主機 → 視為空簽章。
  - 簽章不同 → **保留不套用**。因此擴大適用範圍(`Host safe` → `Host safe prod`)、調整同名指令的先後、新增或修改
    受管制的值,都需要重新核准。
  - 新文字不含任何受管制的設定(含「移除受管制的設定」)→ 照常套用。
- **保留時**:記錄放進該 space 的 `pending_approvals[alias]`(記錄、文字、簽章差異、來源裝置名稱);同一 alias 的較新
  版本取代舊的;**不進快取**,所以下一輪的本機 diff 不會把它當成本機修改而推回去;cursor 照常推進(記錄仍在 relay 上)。
  發 `sync://approval`;Sync pane 顯示「等待你核准(N)」,並跳出通知。
- **核准**:套用該記錄(寫檔 + 進快取),同一般遠端效果。**拒絕**:丟棄這筆 pending,本機維持原狀;不推送任何東西。
  之後若本機修改這台主機,新的本機版本照 LWW 推送。
- 遠端刪除(`Delete{alias}`)照常套用。本機自己的編輯不需要核准。
- 基線輪(加入、勾選新 space)遇到多台需要核准的主機:合併成一次審核,可「全部核准」。
- v1 升級與更換同步碼後的重新加入,沿用的是本機已套用的內容,不會觸發核准。
- **不在管制範圍**(§3 已列):`HostName`、`User`、`Port`、`ProxyJump`、`LocalForward`、`DynamicForward` 等(前四類的值
  仍受上面「會被代入指令的值」限制)。
- **簽章只比單一主機區塊**:`needs_approval` 只在新舊文字都恰好是一個 `Host` 區塊時才比簽章,其他形狀一律要核准
  (fail closed;同步記錄本來就只能是一個 `Host` 區塊)。
- **移除受管制的設定照常套用**:這只會讓本機自己的設定(例如之後的 `Host *`)在原本的主機上生效;導向別處仍要經過
  主機金鑰提示,而關掉提示的設定已受管制。

### 7.5 更換同步碼

需要 relay 支援 `freeze`(§6.2)。可中斷、可接續的長時間操作;進度存在狀態檔的 `rotation`,新同步碼暫存在 keychain
account `sync:mnemonic-next`。

1. **準備**:產生新同步碼(存 `sync:mnemonic-next`)→ 推導新帳戶 → 為帳戶內**每個** space(含這台沒勾選的)產生新的
   chain id、權杖、金鑰,以**新帳戶金鑰加密後**寫進 `rotation`。
2. **送出本機修改**:把這台所有 dirty 記錄(帳戶與勾選的 space)上傳完畢。
3. **標記與凍結**:在舊帳戶 chain 寫入 `meta` `rotation:<device_id>` → `freeze` 舊帳戶 chain → `freeze` 每個舊 space
   chain。從這一步起,任何電腦都寫不進舊資料(relay 強制),所以下一步取得的就是最終內容。
4. **完整快照**:從 seq 0 拉取每個舊 space chain 與舊帳戶 chain(來源是 relay,不是本機快取;因此包含別台已上傳的修改
   與這台尚待核准的記錄)。
5. **建立與複製**:`PUT` 新帳戶 chain 與每個新 space chain(遇 `429` 暫停,下個小時自動接續)→ 每個 space 的記錄以新
   金鑰重新加密上傳,保留 version、updated_at_ms、device_id 與 tombstone → 新帳戶寫入 `space`(含 `previous_id`)、
   `spacekey`、`device`、`meta`,以及 SP3 的 `keyslot`、`key`
   (`2026-10-05-sp3-key-slots-design.md` §6.6)。
6. **刪除**:`DELETE` 每個舊 space chain。舊帳戶 chain 保留(凍結、帶著標記),閒置 180 天後由 relay 清除。
7. **切換**:keychain 的 `sync:mnemonic-next` 取代 `sync:mnemonic`,狀態改用新帳戶(以 `previous_id` 對照保留勾選、
   檔名、待核准項目),清掉 `rotation`。顯示新同步碼。之後再讀一次舊帳戶 chain(凍結的 chain 仍可讀):若有其他裝置的
   `rotation:*` 標記,提示「另一台電腦也更換了同步碼」(讀不到就這一步重來,不略過提示)。keychain 寫不進去時沿用暫存碼推導的
   金鑰、保留暫存碼,下次啟動再換;在換好之前,「Show sync code」顯示暫存的新碼。
   新帳戶沒有接續的勾選 space(例如期間被別台刪除的)改成 `~/.ssh/sshelter-local/` 的本機檔案並提示(同 §7.3 離開),ssh 照常讀得到。

第 3 步之前先確認暫存的新碼還讀得到,讀不到就停在還能取消的狀態。第 3–7 步遇到 relay 錯誤(`429`、`5xx`、額度、請求被拒)
照 §6.4 退避後重試同一步;步驟完成就恢復正常間隔、清掉錯誤。

- **取消**:只能在第 3 步之前(還沒凍結任何東西):刪除已建立的新 chain、清掉暫存的同步碼,回到原狀。第 3 步開始後
  其他電腦已被擋下,只能做完。第 3 步之前離開帳戶等於取消(與背景的凍結在同一個鎖內判定,不會一邊離開一邊凍結);
  第 3 步開始後不能離開,除非暫存的新碼已經不見或屬於別的帳戶(這次更換再也完成不了):那時可以離開。舊帳戶已凍結、帶著
  更換標記,任何電腦都不能再加入它,所以提示改由其中一台建立新的同步帳戶、其他電腦離開後加入。
- **其他電腦**:push 收到 `409 frozen`,或拉取舊帳戶時看到 `rotation:*` 標記 → 記下 `frozen`,停止一切上傳 → 狀態列
  顯示「同步碼已在 MacBook-A 更換,請輸入新碼」→ 輸入後加入新帳戶,依 `previous_id` 保留勾選、檔名;**本機尚未上傳的
  dirty 記錄沿用原時間戳帶進新 space**,第一輪以一般 LWW 合併(不是基線輪),所以被凍結擋下的修改不會遺失。
  輸入的碼先暫存在 `sync:mnemonic-next`,中斷後啟動時補完。這台有勾選 space、卻沒有任何一個對應到新帳戶(輸入的是別的帳戶,
  或之後又換過一次的碼)就拒絕,請使用者離開(檔案保留成本機檔案)再加入;部分沒有對應的(例如凍結前在這台建立、還沒上傳的
  space)改成本機檔案並提示,ssh 照常讀得到。
- **兩台同時更換**:兩台都會寫入自己的標記,凍結是冪等的,兩台取得的快照相同,各自建出一個新帳戶。其他電腦看到兩個
  標記,提示「有兩台電腦同時更換了同步碼」,輸入其中一組即可;兩台發起者在第 7 步也會看到對方的標記並得到提示。
  沒被採用的新帳戶閒置到過期。

### 7.6 從 v1 升級

觸發:v2 app 啟動時讀到 `version: 1` 的狀態檔且已加入,keychain 有同步碼。可重複執行,結果相同:

1. 推導 v2 帳戶與 space0(§5.1);先讀帳戶 chain,只 `PUT` 還不存在的 chain(relay 把每次 `PUT` 都算進每 IP 每小時 20 次的
   建立額度),帳戶顯示 space0 已被刪除時不 `PUT` space0。升級遇到 `429` / `5xx` 照 §6.4 退避。
   整個升級過程中,任何主機都不能從 ssh 消失:主 config 的 Include 改動一次寫入、失敗就還原;v1 狀態在動檔案之前就先備份;
   中斷後重跑或使用者選擇放棄升級,都要認得上一次已經寫好的 space0 檔並保留它。重跑時以主 config 目前列著的檔案決定來源;
   ssh 不讀的殘留檔只備份後移除,不併進 ssh 會讀的檔案。v1 只管 `hosts.config` 這個 token:使用者自己放在 `~/.ssh/sshelter/`、
   由我們的 Include 列著的其他檔案,在同一次寫入裡改成 `~/.ssh/sshelter-local/` 的本機檔案(同 §7.3 離開時的做法)。
   上一次留下、主 config 沒列著的 kept 檔先備份,再改寫成這一次要保留的內容;列著的才往後加。放棄升級時,主 config 沒列著的
   `hosts.config`(ssh 不讀的殘留)只備份後移除,不改成本機檔案。
2. 寫入 space0 的 `space` 與 `spacekey` 記錄(§5.2 的確定值;push conflict 照一般流程處理)。
3. v1 快取中的 `host` 記錄(含 version、updated_at_ms、device_id、tombstone 與 dirty 狀態)成為 space0 的記錄,標為
   dirty;之後的一般輪次以 LWW 上傳,兩台的副本會正確合併。v1 `sealed` 中的 `key`/`password` 記錄捨棄(Phase A 不會
   產生)。含 `Include` 的區塊(v1 允許)不搬入 space0:保留在 v1 檔案內容中另存的本機檔案,並提示使用者。
4. 依 §4.3:建立 `synced-<space0 id 前 8 字元>.config`(內容 = 原 `hosts.config` 去掉含 `Include` 的區塊)→
   `ensure_include` 把 v1 的 token 換成新的清單 → 備份並移除 `hosts.config`。
5. 寫入 `device` 記錄;v1 狀態檔備份為 `sync-state.v1-backup.json`,記在 `legacy_v1_backup`。
6. v1 chain 不再讀寫,閒置 180 天後由 relay 清除。仍在 v1 版本的電腦更新前看不到升級後的修改(README 說明)。

## 8. 前端

- **Settings → Sync**:
  - 帳戶:同步碼(Show / Change sync code)、relay URL 與版本(「relay 可以更新」提示)、裝置清單(名稱、平台、
    最後上線、勾選了哪些 space、Forget——僅從清單移除,文案同 v1)。
  - Spaces:名稱、檔名、這台是否勾選、主機數、改名、刪除、「New space」。
  - 等待核准:清單與審核對話框(顯示完整區塊,標出受管制的行與簽章差異;逐台或全部核准 / 拒絕)。
  - 更換同步碼:確認對話框(說明其他電腦需要輸入新碼、relay 需支援凍結)→ 進度 → 顯示新同步碼。
  - 未加入:輸入同步碼加入,或建立新帳戶;加入後勾選 space。
- **側邊欄**:每個勾選的 space 一個分組(標籤 = space 名稱,帶「已同步」標記);跨 space 名稱衝突在列尾標示。
- **搬移精靈**:目標 space 選擇器、「一個來源檔建立一個 space」、列出不能搬入的區塊與原因。
- **v1 升級**:完成後一次性的說明(space「Synced」可改名;其他電腦也要更新;若有含 `Include` 的區塊被留在本機;若有使用者自己的檔案被搬到 `~/.ssh/sshelter-local/`)。

## 9. 錯誤處理

| 情況 | 處理 |
|---|---|
| 離線 | 同 v1:不阻擋本機操作,dirty 持久化,連線後送出 |
| relay 沒有批次端點 | 退回逐條查詢(§6.4),提示更新 relay |
| relay 沒有 `freeze` | 「更換同步碼」停用並說明 |
| `429` | §6.4 的退避,狀態列說明 |
| `deferred` | 只把 deferred 項目排在最前面立刻補抓;未取得的 chain 不推進 cursor |
| 批次查詢 `5xx` | 連續 2 次後這一輪逐條 `GET`,之後指數退避(§6.4) |
| push `409 frozen` | 停止上傳、保留 dirty,確認更換標記後進入 §7.5「其他電腦」流程 |
| space chain `404`,帳戶已 tombstone 該 space | 依 §4.3 移除 Include 與檔案(先備份),通知 |
| space chain `404`,帳戶仍有該 space | 狀態列提示,讓使用者選「用本機內容重建」(以同一組位置與權杖重新 `PUT` 並上傳)或「移除」 |
| 帳戶 chain `404` / 有 `rotation:*` 標記 | §7.5 其他電腦的流程;無標記 → 同 v1「chain 已不存在」 |
| 更換同步碼中斷 | 下次啟動依 `rotation` 接續;第 3 步前可取消 |
| v1 升級失敗 | 保留 v1 狀態與檔案,下次再試 |
| 單一 space 違反不變式或寫入失敗 | 只暫停該 space;每個 space 的提交各自全有或全無,且只局部更新狀態 |
| 等待核准中又收到新版本 | 新版本取代舊的 pending |
| 改名的目標檔名已存在 | 不覆蓋;保留舊檔名並提示 |
| 目錄中有不在 Include 清單上的檔案 | OpenSSH 不讀;只提示 |
| keychain 讀取失敗、受管檔消失或清空、狀態寫不進磁碟 | 沿用 v1 的處理,以 space 為單位 |

## 10. 測試

- **加密**:§5.4 的已知答案向量;v1 向量不變。
- **relay**(`@cloudflare/vitest-plugin`,在 workerd 內執行):
  - `/v1/pull`:每項驗證、64 項上限、重複 chain → `400`、以序列化位元組計算的預算與「第一項一律放入」、混合狀態、
    `pull` 桶與整批 `429`、閒置期限刷新;
  - `freeze`:權杖、冪等、每分鐘限制、DO 被逐出後仍凍結、凍結後 push 回 `409` 且不寫入、pull 照常、`DELETE` 只清記錄
    且維持凍結、`PUT` 不解除凍結;未凍結的 chain 仍整條刪除;
  - 閒置期限刷新的節流(同一個視窗內第二次讀取不移動 alarm);每個請求一次限流呼叫;限流 alarm 只在新視窗移動;
  - `/v1/info` 與 `RELAY_VERSION` 一致;Docker 冒煙測試加上批次查詢與凍結。
- **同步引擎**(Rust,模擬 relay):
  - 多 space 建立 / 改名 / 刪除的傳播;勾選與取消勾選;Include 清單內容與 §4.3 的順序規則;改名目標已存在;
  - 跨 space 搬移(含目標寫入失敗);跨 space 名稱衝突;逐 space 局部提交不覆蓋其他 space 或帳戶的 cursor;
  - 更換同步碼:每一步、中斷接續、取消、凍結期間別台的 push 被擋下並在重新加入後送出、快照包含別台的修改與待核准記錄、
    兩台同時更換;
  - v1 升級:兩台同時升級得到同一個 space0、push conflict 後合併、重複執行、含 `Include` 的區塊被留在本機;
  - 舊 relay 退回、輪詢間隔計算、`429` 退避、`deferred` 補抓順序;單一 space 失敗隔離;
  - 危險設定:禁止 `Include`;簽章涵蓋 Host pattern 擴大、同名指令重排、`=` 寫法與大小寫;保留、核准、拒絕、被新版本
    取代、基線合併審核、移除受管制設定照常套用。
- **前端**:space 清單與勾選、核准對話框、側邊欄依 space 分組與衝突標示、搬移精靈、更換同步碼的流程狀態。
- **relay 更新 workflow**:actionlint(含 shellcheck);抽出各 `run:` 以替身 `git` / `gh` 執行;更新腳本的測試(保留
  `name`、`name` 寫不回去時不動任何檔案、排除 `.git/`、不寫也不刪 `.github/` 中 owner 的檔案、workflow 只比較、不降版、
  沒有變更或已有同一 commit 的 PR 時不開、PR 內文列出刪除的檔案、binding / migration 變動、workflow 與腳本的變更)。
- **手動**:更新兩台電腦的實測清單(升級、加入、勾選、搬移、更換同步碼、核准)。

## 11. 發佈

- 先以 beta 發佈,兩台電腦實測通過後才進正式版。PR #19(0.17.0)是否先發,實作完成前再決定。
- relay 更新:批次查詢有退回機制,不是必要條件;更換同步碼需要 `freeze`,所以實測「更換同步碼」前必須先更新 relay。

## 12. 審查紀錄(Codex,2026-10-02)

| # | 嚴重度 | 問題 | 處理 |
|---|---|---|---|
| 1 | Critical | 更換同步碼時,複製之後、刪除之前別台上傳到舊 space 的修改會遺失 | 新增 relay `freeze`;先標記並凍結所有舊 chain,再從 relay 取完整快照(§6.2、§7.5) |
| 2 | Critical | 「標記與所有 tombstone 同一個 push」超過 200 筆上限 | 不再 tombstone 舊帳戶;停止訊號改為標記 + 凍結,兩者都不依賴單一 push(§7.5) |
| 3 | High | 危險設定比對可被擴大 Host pattern、調整指令順序、`Include` 繞過 | 改用「核准簽章」(pattern + 依序的受管制 directive);同步區塊禁止 `Include`;以解析器判斷 keyword(§7.4) |
| 4 | High | 目錄 glob 會讓 OpenSSH 載入殘留或未勾選的 `.config` | 改成明確的 Include 清單與建立 / 移除順序規則(§4.3) |
| 5 | High | 只從本機快取複製會漏掉別台的修改與待核准記錄 | 從凍結後的 relay 完整快照複製所有 space(§7.5) |
| 6 | High | slug 唯一性只靠 UI,並發改名可能撞到同一個檔案 | 檔名帶 space id 前 8 字元;改名不覆蓋既有檔案(§4.3) |
| 7 | Medium | 同時升級時 space0 記錄 metadata 不同,不能宣稱寫入冪等 | space0 payload 改為確定值,push conflict 走一般 pull / 合併 / 重試(§5.2) |
| 8 | Medium | 逐 space 提交若發布整份狀態副本會蓋掉其他部分 | 提交只對最新狀態做局部更新並檢查 generation(§7.1) |
| 9 | Medium | 核准範圍與宣稱的安全邊界不一致 | 管制清單加入 `ForwardAgent`、`ForwardX11(Trusted)`、`RemoteForward`;明列不管制的設定(§3、§7.4) |
| 10 | Medium | 批次預算只計密文,可超標;補抓可能餓死尾端 | 以序列化位元組計算、第一項一律放入;只補抓 deferred 項並排在最前(§6.1、§6.4) |
| 11 | Medium | 每項計入 `pull` 桶,多 space 加 NAT 會撞到上限 | `pull` 桶提高到 12000;輪詢間隔隨 chain 數調整;文案區分請求數與 relay 工作量(§6.1、§6.4) |
| 12 | Medium | 更新 workflow 會自動部署可變的上游程式碼 | 解析成固定 commit、先跑測試、只開 PR、列出刪除檔與 binding / migration 變動;排程也只開 PR(§6.5) |

Codex 確認為合理的部分:v1/v2/帳戶/space0 的推導分離、AAD 綁定與解密後的身分驗證、批次項目各自驗證權杖且不洩露
「不存在」與「權杖不符」的差別、`deferred` 不推進 cursor、一般加入沿用 v1 基線輪、舊 relay 的退回路徑。

## 13. B1(relay)最終審查的修正(2026-10-02)

| # | 嚴重度 | 問題 | 處理 |
|---|---|---|---|
| C1 | Critical | 更新 workflow 單一 job:上游程式碼與寫入權杖在同一台 VM,能植入 git hook / fsmonitor、寫 `$GITHUB_ENV` 或改 PR 內文 | 拆成 `test`(唯讀,跑上游程式碼)與 `open-pr`(寫入,不執行上游程式碼、重新 checkout 並確認同一個 SHA)(§6.5) |
| I1 | Important | 每週排程可能把 relay 默默降版 | 比較版本;沒指定 `ref` 不降版,明確指定才開「Downgrade」PR(§6.5) |
| I2 | Important | 部署按鈕是否複製 `relay/.github` 未經實測 | 合併到 main 前由使用者手動部署一次確認(§6.5) |
| I3 | Important | 凍結的 chain `DELETE` 後再 `PUT` 就變回未凍結,舊裝置可以重新寫入 | 凍結的 chain 刪除時只清記錄、維持凍結(§6.2) |
| I4 | Important | 免費方案每日額度:DO 休眠後每次 45 秒輪詢都重設閒置 alarm;每個請求兩次限流呼叫與一次 alarm 寫入 | 以 alarm 時間節流、每請求一次限流呼叫、限流 alarm 只在新視窗移動;client 依焦點調整輪詢;README 說明上限(§6.4、§6.6) |
| I5 | Important | README 的「checkout 後 wrangler deploy」在 Worker 改過名稱時會建出第二個 Worker | 建議複製兩個檔案後執行 workflow;checkout 路線先改 `name`(§6.5) |
