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
- **私鑰 opt-in、逐把決定**:預設不同步任何既有金鑰;使用者在 Keys dialog
  逐把勾「Sync this key」。硬體/不可匯出金鑰無法同步(UI 直接不提供選項)。
- **裝置遺失的止血 = 輪替**(§8):產生新金鑰 → 批次部署 → 批次撤舊 → 更新 chain。
  這是「一把鑰匙多台裝置」模型的必要配套,列為 v1 範圍。
- **密碼 opt-in、預設關**:「Sync passwords」是**每台裝置各自**的開關 —— 開啟的
  裝置才上傳自己的密碼、也才把遠端密碼記錄寫入本機 keychain;關閉的裝置兩者皆不做。
- 既有 MCP 邊界不變:MCP allowlist 是每台裝置的本機政策,**不同步**。
- 本機落地檔案權限:私鑰 0600、同步檔 0600、狀態檔 0600(既有 `fsutil` 慣例)。

## 3. 資料模型

### 3.1 受管同步檔(主機的真相來源)
- 路徑固定:`~/.ssh/sshelter/hosts.config`(目錄 `~/.ssh/sshelter/` 0700)。
- 加入 chain 或建立 chain 時,SSHelter 用既有的 Include 機制(`config/newfile.rs`
  的 `include_insert_index`)在每台裝置的主 config 插入
  `Include ~/.ssh/sshelter/hosts.config`(位置規則已測試:top-level、第一個
  Host/Match 之前)。
- **同步範圍 = 這個檔案裡的 host 區塊**。其他檔案的 host 維持裝置本地。
  提供「Move to synced」(沿用批次 move)與首次一鍵「把所有主機移入同步檔」。
- sidebar 檔案分組把它顯示為「Synced」(預設 file alias)。

### 3.2 記錄(record)
所有同步內容都是記錄;chain = 記錄集合。共同欄位:

| 欄位 | 說明 |
|---|---|
| `id` | 記錄識別(見各型別) |
| `kind` | `host` / `key` / `password` / `device` |
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

## 4. 密碼學

- 助記詞:BIP39 英文 24 詞(256-bit entropy),`bip39` crate。
- 派生:`seed = bip39.to_seed("")`;`HKDF-SHA256(seed, info)`:
  - `info="sshelter/v1/chain-id"` → 32 bytes → hex,作為 chain id(可公開)
  - `info="sshelter/v1/auth"` → 32 bytes → bearer token(中繼只存 SHA-256(token))
  - `info="sshelter/v1/enc"` → 32 bytes → 記錄加密金鑰
- 記錄加密:XChaCha20-Poly1305(`chacha20poly1305` crate),每筆隨機 24-byte nonce;
  AAD = `chain_id || kind || id`(防止記錄被搬到別的 chain 或改型別)。
- 中繼看到的 envelope:`{ id_hash, kind, seq, ciphertext(base64), nonce, deleted }`,
  其中 `id_hash = HMAC(enc_key, kind||id)` —— 中繼連 alias/檔名都看不到。
- 助記詞存本機 keychain(`secrets` 模組,account `sync:mnemonic`),因為任一台既有
  裝置都要能「Show pairing code」給新裝置(與 Brave 的 View sync code 一致);
  派生值只在記憶體,啟動時重算。**不存純文字檔**。
- v1 為純桌面,配對靠輸入 24 詞;QR 留給未來的行動版(桌面 app 沒有相機)。

## 5. 中繼(relay)

Cloudflare Worker + Durable Object(每 chain 一個 DO,SQLite storage 提供順序與原子性)。
開源,repo 內 `relay/` 目錄,`wrangler deploy` 即自架;app 設定可改 relay URL,
預設指向專案託管的 Worker。

API(全部 `Authorization: Bearer <token>`,JSON):

| Method | Path | 說明 |
|---|---|---|
| `PUT` | `/v1/chains/{chain_id}` | 建立 chain(冪等);首次寫入記下 token hash |
| `POST` | `/v1/chains/{chain_id}/records` | 批次 upsert;body `[{id_hash, kind, ciphertext, nonce, deleted, base_seq?}]`;回每筆新 `seq`。**若 `base_seq` 落後於伺服器現況 → 該筆回 409 並附最新 envelope**(client 端合併後重送) |
| `GET` | `/v1/chains/{chain_id}/records?since={seq}` | 回 `seq > since` 的 envelope 與 `latest_seq` |
| `DELETE` | `/v1/chains/{chain_id}` | 刪除整個 chain(離開最後一台裝置時) |

防濫用:每 chain 總量上限 1 MiB、每筆 64 KiB、每 IP 與每 chain 速率限制、
閒置 180 天自動清除、token hash 不符一律 404(不洩露 chain 存在)。

## 6. 同步引擎(Rust,`src-tauri/src/sync/`)

- 狀態:`~/.local/share/org.homelab.sshelter/sync-state.json`(`atomic_write` 0600):
  `{ chain_id, device_id, relay_url, cursor_seq, password_sync: bool, records: {…本地快取…} }`。
- 觸發:app 啟動、視窗取得焦點、每 45 秒(有焦點時)、本地變更後立即 push。
- 流程:`pull(since=cursor)` → 套用遠端變更 → `push(dirty records)` → 409 者
  合併後重送。
- **合併規則(記錄層級 LWW)**:遠端 vs 本地同 id:`updated_at_ms` 大者勝,
  相同則 `device_id` 字典序小者勝;輸的一方若是本地未上傳的修改 → 發事件
  `sync://conflict` 讓前端 toast「web-1 was changed on MacBook, your local edit
  was replaced」。tombstone 勝過同時間的修改。
- **套用到本機**(每種型別一個 applier):
  - host → 重寫 `hosts.config` 中該區塊(以 CST 替換整個 block 文字;不動其他區塊、
    註解、順序;新區塊附加於檔尾;tombstone 移除區塊),經既有 `persist_file`
    (備份 + 衝突指紋)。
  - key → 寫 `~/.ssh/<name>`(0600)與 `.pub`;同名不同內容 → 跳過並記錄 issue。
  - password → keychain set/delete。
  - device → 只更新快取。
- **從本機產生記錄**:兩個時機做「區塊 diff」——(a)SSHelter 自己對 `hosts.config`
  的每次 `persist_file` 之後;(b)同步 tick 時檔案指紋與上次不同(使用者手改)。
  diff = 逐區塊序列化後與快取比對,變了就升版本、標 dirty,消失的區塊 tombstone。
  Keys dialog 勾選/取消 → key 記錄;密碼儲存/刪除(既有 `secrets_set/delete`
  命令)→ 本機「Sync passwords」開啟時產生記錄。
- 裝置身分:首次執行產生 `device_id`(16 bytes 隨機 hex),名稱預設主機名、可改。
- Leave chain:清掉 keychain 的 sync 項目與 `sync-state.json`;本機檔案全部保留。
  若是 chain 的最後一台裝置,詢問是否一併 `DELETE` 中繼上的 chain。
- 離線:pull 失敗只記狀態,不阻擋任何本機操作;dirty 記錄下次上線再送。

## 7. 前端

- **Settings → Sync** pane(比照 McpPane 的輪詢/狀態模式):
  - 未加入:「Create sync chain」「Join with words」。
  - 建立後:顯示 24 詞 + 「I have saved these words」確認(並提醒存進密碼管理器)。
  - 已加入:狀態列(last sync、錯誤)、裝置清單(名稱/平台/last seen/持有金鑰/
    Remove)、「Show pairing code」(明確按鈕、再次顯示 24 詞給新裝置抄)、
    「Sync passwords」開關(本機)、relay URL(進階)、「Leave chain」。
- **Keys dialog**:每把金鑰一個「Synced」開關(硬體/無私鑰檔者停用)。
- **Sidebar**:`hosts.config` 群組標籤「Synced」;host 使用未同步金鑰時列尾小警示。
- **新裝置上手**:Join 完成 → 進度畫面(拉取記錄 → 寫入金鑰 → 寫入 config → 插入
  Include)→ 完成頁列出「N hosts, M keys, P passwords ready」。
- **裝置移除**:Remove → tombstone → 若該裝置持有同步金鑰 → 提示「Rotate keys
  it had access to」直達 §8。

## 8. 金鑰輪替(v1 必要配套)

- 入口:裝置移除提示、Keys dialog 每把同步金鑰的「Rotate…」。
- 流程(單一對話框、可看進度):
  1. 產生新金鑰(名稱 `<old>-<yyyymmdd>` 或使用者指定)並加入 chain。
  2. 找出所有 `IdentityFile` 指向舊金鑰的同步主機 → 批次部署新公鑰
     (沿用 `deploy.rs` 管線;逐台結果列表;失敗可重試)。
  3. 成功的主機 → 遠端移除舊公鑰:新的 `REMOTE_REMOVE_SCRIPT`(`grep -vxF`
     過濾該行,原子寫回,權限 0600,回 `SSHELTER_REMOVED`)。
  4. 改寫這些主機的 `IdentityFile` 指向新金鑰(記錄升版、同步)。
  5. 舊金鑰記錄 tombstone;本機檔案改名 `.revoked`(不刪,留給使用者)。
- 任何一步失敗都不會讓主機無法登入(舊鑰只在新鑰部署成功後才移除)。

## 9. 驗收與測試

- Rust 單元:HKDF 派生向量固定測試、加解密 round-trip 與 AAD 錯配失敗、
  LWW 合併全案例(時間/裝置 tie-break/tombstone)、host 區塊序列化 round-trip
  (含 `#tags:`)、applier 不動其他區塊、key 落地權限與同名不同內容跳過。
- Relay:`wrangler dev` 下的整合測試(建立、push/pull、409 衝突、配額、錯 token 404)。
- 端到端手動:兩個 SSHelter profile(以自訂 config path 模擬兩台)配對、
  互改主機、同步私鑰後以 `ssh` 登入、移除裝置 → 輪替。
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
2. **金鑰**:列出 `~/.ssh` 現有金鑰供勾選是否同步;預設勾選「被已搬入主機的
   `IdentityFile` 引用」的那些;硬體/無私鑰檔者不可勾。
3. **密碼**:若本機開啟「Sync passwords」,把已搬入主機在 keychain 的密碼建成記錄。

**後續裝置加入時的既有資料處理**
- 該裝置本地已有主機:Include 插在主檔頂部 → 依 ssh 的 first-match 語意,**同步檔
  的同名主機會遮蔽本地定義**。加入 wizard 用既有 lint 的「shadowed alias」偵測
  列出重複,逐筆選擇:保留本地(把本地區塊改名 `<alias>-local`)/ 改用同步版
  (移除本地區塊,先備份)/ 稍後處理(維持遮蔽並在 sidebar 標示)。
- 本地已有同名但不同內容的金鑰檔:不覆蓋,提示改名後重試(§3.3)。
- 本地 keychain 已有同 alias 密碼且「Sync passwords」開啟:以記錄 LWW 決定,
  並在完成頁列出被覆蓋的項目。

**資料格式版本化**
- 每筆記錄 payload 含 `schema: 1`;chain 有一筆 `meta` 記錄(`schema_version`、
  `created_by_app_version`)。client 支援版本 < chain `schema_version` → 唯讀模式
  (仍套用可理解的記錄、不上傳、顯示「請更新 SSHelter」)。未知 `kind` 一律忽略保留。
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
