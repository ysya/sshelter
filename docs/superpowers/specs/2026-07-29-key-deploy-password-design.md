# 一鍵部署金鑰 + keychain 密碼儲存

- 日期：2026-07-29
- 狀態：設計待審
- 範圍：Rust 後端（新增 3 個模組 + 1 個相依）＋ 前端（新增 dialog、host row 右鍵選單）

## 背景與目標

SSHelter 目前對「還沒放上公鑰的新主機」支援不足。現有的部署流程藏在 Keys dialog 的 ✈️ 按鈕裡，而且只是**把 `ssh-copy-id` 丟到使用者的終端機**（`keys_deploy` → `launch_in_terminal`）——密碼要在終端機裡打，成功與否 SSHelter 完全不知道，app 內看不到任何結果。

本功能讓使用者在 SSHelter 內**一鍵把公鑰部署到目標主機，全程不開終端機**，成功／失敗直接顯示在 app 裡；部署所需的主機密碼可選擇存進作業系統的 keychain，並且在 app 內可以讀回、顯示、複製、刪除。

### 使用者已確認的決策

1. 密碼存**作業系統 keychain**，不寫進 `~/.ssh/config`（見下方「為什麼不寫進 config」）。
2. 密碼必須能**在 app 內讀回顯示**，不是只寫不讀。
3. 機制走 **SSH_ASKPASS**，不自建終端機模擬器。
4. 本輪**只做一鍵部署金鑰**；日常密碼登入自動填留待下一輪。
5. 入口加 **host row 右鍵選單**。
6. 沒勾「記住密碼」時，採**暫存 keychain 項目、用完即刪**的做法。

### 為什麼不寫進 `~/.ssh/config`

實測（OpenSSH 10.2p1）：

- 寫成關鍵字 `Password xxx` → `ssh` 直接罷工：`line 4: Bad configuration option: password` / `terminating, 1 bad configuration options`。而 `~/.ssh/config` 是**每次 ssh 都整份讀**，所以一行就會讓**所有** host 的 `ssh` 失敗，不只那一台。這條路不通。
- 寫成註解 `#SSHelter-Password xxx` → 語法可行（`ssh -G` 正常），但密碼明文躺在 config，且 `fsutil::backup()` 在每次改動前會把整份檔案複製到 app-data 的 mirror 目錄並保留多份歷史 —— 即使之後刪掉密碼，**舊備份裡仍留有明文**。再加上 `~/.ssh/config` 是最常被丟進 dotfiles repo / iCloud / Dropbox 的檔案之一。

結論：keychain。而且**對使用者的操作體驗沒有任何差別** —— UI 上都是一格密碼欄，差別只在字串最後落在哪。

## 非目標（YAGNI）

- **日常密碼登入自動填**（把密碼送進使用者外部終端機裡的 ssh）——下一輪。
- **Windows 平台支援**——另一輪。但本輪的部署實作刻意**不依賴 `ssh-copy-id`**（見下），使 Windows 那輪不必重寫。
- 密碼認證的 **jump host / ProxyJump**：askpass helper 只拿得到目標主機的密碼，分不出是哪一跳在問。明確不支援，UI 需說明。
- **遠端是 Windows sshd** 的情形（`administrators_authorized_keys`、ACL）：本輪只支援 POSIX 遠端，偵測不到時給明確錯誤。
- 不改動既有的 `keys_deploy`（終端機版）的 Rust 指令；前端改走新流程，舊指令暫留待下一輪清理。

## 業界調查與方案選擇

調查結果，同類產品分成兩派：

| 派別 | 產品 | 做法 | 對 SSHelter 的適用性 |
|------|------|------|---------------------|
| 內建終端機 | Termius、SecureCRT、Royal TSX、MobaXterm、Xshell | 終端機是自己寫的，密碼提示出現時由 app 直接填 | ❌ 等於重寫終端機模擬器，推翻「用你自己的 Terminal/iTerm」的產品定位 |
| SSH_ASKPASS + OS keyring | `sshelf` | 執行檔自己兼任 askpass helper，`SSH_ASKPASS_REQUIRE=force` + 從 keyring 查密碼印到 stdout | ✅ 採用 |
| ControlMaster 預連線 | （查不到任何產品在用） | 背景建 master 連線，終端機複用 | 可行但自創，且 Windows OpenSSH 可能不支援 multiplexing |

最接近的競品 **SSH Config Editor**（Hejki，Mac App Store）跟 SSHelter 幾乎同定位，其「Automatic login with securely stored passwords」是 **Pro 付費功能**，機制未公開。

`sshelf` 的文章列出兩個真實的坑，本設計都要處理：

1. **`SSH_ASKPASS_REQUIRE=force` 會讓「所有」提示都走 helper**，包含第一次連線的 host key 驗證。笨 helper 會把密碼當答案印出去 → ssh 再問 → 無窮迴圈。
2. **keyboard-interactive 的提示文字由伺服器控制** —— 惡意主機可送一個含 `password` 字樣的提示騙走密碼。

參考資料：
- [Auto-supplying SSH passwords without sshpass: the SSH_ASKPASS trick](https://dev.to/max-rh/auto-supplying-ssh-passwords-without-sshpass-the-sshaskpass-trick-49ig)
- [Termius Keychain 文件](https://docs.termius.com/keychain/identities)
- [SSH Config Editor (Hejki Apps)](https://www.hejki.org/ssheditor/)
- [Key-based authentication in OpenSSH for Windows](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement)（確認 Windows OpenSSH **沒有** `ssh-copy-id`）

## 整體方法

部署分三步，全部在 Rust 內完成，不開終端機。

### Step 0 — 先驗 host key，而不是讓 ssh 去問

跑 `ssh-keyscan -T 5 <hostname>` 取得主機金鑰，與 `~/.ssh/known_hosts` 比對：

| 狀態 | 行為 |
|------|------|
| 已存在且相符 | 直接繼續 |
| 不存在 | 在 app 內顯示 SHA256 指紋，使用者按「信任並繼續」才寫入 known_hosts |
| 存在但**不符** | **中止**並警告可能為中間人攻擊，不提供「強制繼續」 |

**這一步是整個設計的關鍵。** 因為 host key 在部署前已經確定，Step 2 的 ssh 可以用 `StrictHostKeyChecking=yes`，host key 提示**永遠不會出現** —— 上述陷阱 1（無窮迴圈）從結構上消失，比單靠 helper 的字串白名單可靠得多。白名單仍然實作，作為縱深防禦。

### Step 1 — 用 askpass 餵密碼

`SSH_ASKPASS` 指向 SSHelter 執行檔自己（`std::env::current_exe()`），並設：

```
SSH_ASKPASS_REQUIRE=force
SSHELTER_ASKPASS=1
SSHELTER_ASKPASS_ACCOUNT=<keychain account 名稱>
```

`main()` 在初始化 Tauri 之前先檢查 `SSHELTER_ASKPASS=1`，若成立則進入 helper 模式：驗證提示文字 → 從 keychain 讀密碼 → 印到 stdout → `exit(0)`，**完全不初始化 GUI**。

Step 2 的 ssh 帶 `-T`（不配置 pty），所以 ssh 本來就沒有終端機可提示；`SSH_ASKPASS_REQUIRE=force` 讓它不必依賴 `DISPLAY` 也會走 helper。

**密碼從頭到尾不進 argv、不進 `ps`。**

### Step 2 — 部署本體（不用 `ssh-copy-id`）

不使用 `ssh-copy-id`，理由有二：Windows OpenSSH 沒有這支程式；且自己實作才控制得了錯誤分類（要在 app 內分辨「密碼錯」與「連不上」）。

本機端維持**純 argv、絕不 `sh -c`**（現有安全模型）。遠端則執行一段**固定的、不含任何使用者輸入的** shell script，公鑰內容走 **stdin**：

```sh
umask 077
mkdir -p ~/.ssh || exit 90
k=$(cat)
[ -n "$k" ] || exit 91
if [ -f ~/.ssh/authorized_keys ] && grep -qxF "$k" ~/.ssh/authorized_keys; then
  echo SSHELTER_EXISTS
else
  printf '%s\n' "$k" >> ~/.ssh/authorized_keys || exit 92
  chmod 600 ~/.ssh/authorized_keys
  echo SSHELTER_ADDED
fi
```

> **公鑰必須走 stdin，不可拼進遠端指令。** `.pub` 檔的 comment 欄位是使用者可控內容，拼進遠端 shell 指令即為注入點。

ssh 的 argv：

```
ssh -T
    -o StrictHostKeyChecking=yes
    -o BatchMode=no          # 防使用者 config 全域設了 BatchMode yes 而封鎖密碼提示
    -o NumberOfPasswordPrompts=1   # 密碼錯立刻失敗，不問三次
    -o ConnectTimeout=10
    <alias>
    <上面那段固定 script>
```

**刻意不設 `PreferredAuthentications`** —— 讓 ssh 正常協商。如果使用者其實已經有可用的金鑰，ssh 根本不會呼叫 askpass，部署直接成功。

## 具體變更

### Rust

#### 1. `src-tauri/Cargo.toml`

新增 `keyring = "4.1.5"`（預設 feature `v1` 已涵蓋 macOS Keychain / Windows Credential Manager / Linux Secret Service；MSRV 1.88.0，需確認 CI 的 Rust stable 符合）。

#### 2. `src-tauri/src/secrets.rs`（新增）

keyring 的薄封裝。service 固定為 `"SSHelter"`，account 分兩種：

- 正式項目：`host:<alias>` —— 使用者勾了「記住密碼」時使用
- 暫存項目：`deploy-tmp:<alias>` —— 沒勾時使用，部署結束後**一律刪除**，無論成敗

也就是說 askpass 要查哪個 account 由 `remember` 決定，兩者擇一，不會先寫暫存再搬到正式。清理階段只刪暫存項目。

```rust
pub fn get(account: &str) -> Result<Option<String>, AppError>;
pub fn set(account: &str, secret: &str) -> Result<(), AppError>;
pub fn delete(account: &str) -> Result<(), AppError>;
pub fn available() -> bool;   // 探測本機是否有可用的密鑰環
```

alias 一律先經過 `connect::validate_alias` 的字元閘門，才能組成 account 名稱。

**Linux 無 Secret Service 的 fallback**：`available()` 為 false 時，改以環境變數 `SSHELTER_ASKPASS_SECRET` 把密碼傳給 helper，並在 UI 明示「本機無可用密鑰環，密碼不會被儲存」。威脅模型上差異有限（同 uid 的進程既能讀 `/proc/<pid>/environ`，也一樣能讀 Secret Service），但必須讓使用者知道。

#### 3. `src-tauri/src/askpass.rs`（新增）

helper 模式。ssh 會把提示文字當作 `argv[1]` 傳進來。

```rust
/// 只回應真正的密碼／passphrase 提示。其他一律拒絕。
pub fn prompt_is_answerable(prompt: &str) -> bool {
    let p = prompt.trim().to_ascii_lowercase();
    p.ends_with("password:") || p.contains("passphrase for")
}

pub fn run() -> ! { /* 驗證 → 讀 keychain/env → stdout → exit */ }
```

不符合白名單 → 不輸出任何內容、`exit(1)`，交回 ssh 以正常方式處理。

#### 4. `src-tauri/src/deploy.rs`（新增）

- `precheck_host_key(alias) -> HostKeyStatus { Trusted, New { fingerprint }, Mismatch { .. } }`
- `build_deploy_argv(alias) -> Vec<String>`（**純函式，可測**）
- `REMOTE_SCRIPT: &str`（上面那段固定 script）
- `classify_outcome(exit_code, stdout, stderr) -> DeployOutcome`（**純函式，可測**）

`DeployOutcome`：`Added` / `AlreadyPresent` / `WrongPassword` / `HostKeyFailed` / `Unreachable` / `RemoteError { code }` / `Other { message }`

分類依據：stdout 含 `SSHELTER_ADDED` / `SSHELTER_EXISTS`；exit 255 時再看 stderr（`Permission denied` → 密碼錯；`Host key verification failed` → host key；`Connection timed out` / `Could not resolve` / `Connection refused` → 連不上）。

#### 5. `src-tauri/src/main.rs`

在任何 Tauri 初始化**之前**插入：

```rust
if std::env::var_os("SSHELTER_ASKPASS").is_some() {
    sshelter_lib::askpass::run();   // 不返回
}
```

#### 6. 新增 Tauri commands（`lib.rs` 註冊）

| command | 說明 |
|---------|------|
| `deploy_precheck_host_key(alias)` | Step 0，回傳 `HostKeyStatus` |
| `deploy_trust_host_key(alias)` | 使用者確認指紋後寫入 known_hosts |
| `deploy_key(alias, public_path, password, remember)` | Step 1+2，回傳 `DeployOutcome` |
| `secrets_get(alias)` / `secrets_set(alias, password)` / `secrets_delete(alias)` / `secrets_has(alias)` | 密碼的讀寫刪 |

`public_path` 沿用既有的 `keys::validate_public_path`（必須 canonicalize 到 `~/.ssh` 內）。

### 前端

#### 7. `src/components/DeployKeyDialog.tsx`（新增）

流程：目標 host → 選公鑰（預設帶入該 host 的 `IdentityFile` 對應的 `.pub`，否則 `~/.ssh` 裡唯一那把）→ 密碼欄（keychain 已有則自動帶入，遮蔽 + 眼睛按鈕顯示）→「記住這台主機的密碼」勾選 → 部署。

host key 為 `New` 時，先顯示 SHA256 指紋與「信任並繼續」的中間步驟。

#### 8. `src/components/HostList.tsx`

host row 外層包 `ContextMenu`（沿用 v0.6.0 為檔案標題新增的 `src/components/ui/context-menu.tsx`）：

```
[▷]  Connect
[↑]  Deploy key…
────────────────
[⧉]  Copy password        （僅在 keychain 有這台的密碼時顯示）
```

wildcard-only 的 defaults 列不掛選單（與現有 `onConnect` 的處理一致）。

#### 9. `src/components/HostEditor.tsx`

新增一個 Password 區塊：顯示是否已存密碼、可顯示／編輯／刪除。這是使用者明確要求的「要可以讀取」。

> 這個區塊**不是 ssh config 的欄位**，不會寫入任何 config 檔、不參與既有的 host 欄位存檔流程；它讀寫的是 keychain 的 `host:<alias>` 項目，需在 UI 上與其他 config 欄位視覺區隔，避免誤解為 config 選項。

#### 10. `src/lib/queries.ts` + `src/bindings/`

新增對應 hooks；`HostKeyStatus`、`DeployOutcome` 經 ts-rs 匯出型別。

## 資料流

```
host row 右鍵 →「Deploy key…」
  → DeployKeyDialog 開啟（alias 已帶入）
  → secrets_has(alias) → 有則 secrets_get 帶入密碼欄
  → 按「部署」
      → deploy_precheck_host_key(alias)
          ├ Mismatch → 中止，顯示警告
          ├ New      → 顯示指紋 → 使用者確認 → deploy_trust_host_key(alias)
          └ Trusted  → 繼續
      → deploy_key(alias, publicPath, password, remember)
          → account = remember ? "host:<alias>" : "deploy-tmp:<alias>"
          → secrets::set(account, password)
          → spawn ssh (-T, StrictHostKeyChecking=yes, …)
              env: SSH_ASKPASS=<current_exe>
                   SSH_ASKPASS_REQUIRE=force
                   SSHELTER_ASKPASS=1
                   SSHELTER_ASKPASS_ACCOUNT=<account>
              stdin: <公鑰內容>
              → ssh 需要密碼時執行 SSHelter(helper 模式)
                  → prompt_is_answerable(argv[1])?
                      ├ 否 → exit 1
                      └ 是 → keychain 讀密碼 → stdout → exit 0
          → classify_outcome(exit, stdout, stderr)
          → 若 account 是暫存項目 → secrets::delete(account)   ← 無論成敗一律執行
      → app 內顯示結果 toast / 狀態
```

## 安全模型

| 保證 | 做法 |
|------|------|
| 密碼不進命令列 | 走 askpass stdout；`ps` 看不到 |
| 密碼不進 `~/.ssh/config` | 只存 keychain |
| 密碼不進備份 | 不碰 config，`fsutil::backup()` 自然帶不到 |
| 密碼不進 settings export | `settings_export` 只匯出設定 JSON，不含 keychain |
| 公鑰 comment 不成為注入點 | 公鑰走 stdin，不拼進遠端指令 |
| 本機不經 shell | ssh 以純 argv 啟動（遠端 script 由**遠端** shell 解析，本機不解析） |
| 惡意伺服器騙不走密碼 | helper 的提示文字白名單；且 host key 已於 Step 0 固定 |
| 沒勾「記住」時不留痕 | 暫存 keychain 項目，`defer` 語意一律刪除 |

## 邊界情況與風險

- **使用者 config 設了 `PreferredAuthentications publickey`** → 密碼永遠不會被使用，部署會以「Permission denied」失敗。SSHelter 已有 `ssh -G` 整合，可在部署前偵測並給明確訊息，而不是讓使用者看到誤導的「密碼錯誤」。
- **ProxyJump 主機也要密碼** → 不支援（helper 分不出是哪一跳在問）。UI 需明示。
- **遠端是 Windows sshd** → 遠端 script 會失敗；`classify_outcome` 需回 `RemoteError` 並給可讀訊息。
- **OpenSSH < 8.4** → `SSH_ASKPASS_REQUIRE` 不存在，askpass 不會被強制使用。啟動時檢查 `ssh -V`，過舊則停用本功能並說明。
- **Linux 無 Secret Service** → 走 env var fallback，UI 明示密碼不會被儲存。
- **`ssh-keyscan` 對非標準 port / ProxyJump 後方主機** → 拿不到金鑰。此時退回「讓使用者自行確認」或提示改用終端機路徑。
- **`current_exe()` 在 macOS .app bundle 內** → 指向 `SSHelter.app/Contents/MacOS/SSHelter`，可直接執行。需實機驗證（見下）。

## 待驗證假設

以下三點在實作第一步就要先確認，**不可當成既定事實**：

1. **`SSH_ASKPASS_REQUIRE=force` 是否確實攔截 password 認證的 `user@host's password:` 提示**，而不只是金鑰 passphrase。man page 寫的是「all passphrase input」，`sshelf` 的文章也主張可行，但本專案尚未親自實測。
2. **Tauri 打包後的執行檔被自己以 helper 模式再次啟動時，是否能在不初始化 GUI 的情況下乾淨退出**（macOS bundle、Linux AppImage 皆需驗證）。
3. **`keyring` 4.1.5 在 macOS 上，未簽章的 app 存取自己建立的 keychain 項目時的提示行為** —— 是否每次都會跳「允許存取鑰匙圈」。若會，需評估 UX（本專案為未公證的未簽章 build）。

第 1 點若不成立，整個方案要改走 pty（Rust 直接持有 ssh 的 pty 並在偵測到提示時寫入密碼）；架構上仍可行，因為部署流程本來就由 Rust 全程掌控、不涉及外部終端機。

## 驗證計畫

**自動測試**

- Rust 單元測試（純函式）：
  - `prompt_is_answerable`：接受 `xxx's password:`、`Password:`、`Enter passphrase for key '...'`；拒絕 host key 提示（`The authenticity of host ... can't be established`）、拒絕含 `password` 字樣但格式不符的伺服器自訂提示、拒絕空字串。
  - `build_deploy_argv`：含 `-T`、`StrictHostKeyChecking=yes`、`BatchMode=no`、`NumberOfPasswordPrompts=1`；alias 未經驗證時拒絕。
  - `classify_outcome`：六種 outcome 各一則，含 stderr 分類。
  - `secrets` 的 account 命名：alias 字元閘門、暫存與正式項目不互相覆蓋。
- vitest：DeployKeyDialog 的公鑰預選邏輯抽成純函式後測試。
- `pnpm test`、`pnpm build` 全綠，既有 283 個測試不破。

**手動驗證**（需要一台真實的密碼登入主機）

1. 全新主機（不在 known_hosts）→ 顯示指紋 → 信任 → 部署成功 → `ssh <alias>` 免密碼登入。
2. 重複部署同一把 key → 顯示「已存在，未重複加入」。
3. 故意打錯密碼 → app 內顯示「密碼錯誤」，且**只嘗試一次**。
4. 勾「記住密碼」→ 關閉 app 重開 → 密碼仍在，可顯示。
5. 不勾「記住密碼」→ 部署後檢查 keychain，**暫存項目已刪除**。
6. `ps aux | grep ssh` 於部署期間執行 → **確認密碼不出現在任何進程的命令列**。
7. known_hosts 內金鑰被竄改 → 部署中止並警告。
