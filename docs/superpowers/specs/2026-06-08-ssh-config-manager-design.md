# SSHelter — SSH Config Manager 設計文件 (Design Spec)

- **日期**: 2026-06-08
- **狀態**: Draft（待使用者審閱）
- **產品名稱**: **SSHelter**（ssh + shelter，「A shelter for all your SSH hosts」）；binary/crate `sshelter`、bundle id `org.homelab.sshelter`（見 §16）
- **參考對象**: [hejki SSH Editor](https://www.hejki.org/ssheditor/)（macOS-only 商業 app，本專案要做跨平台版）

---

## 1. 概述 (Overview)

一個**跨平台桌面應用**，用視覺化介面管理使用者**本機**的 OpenSSH 設定，類似 hejki SSH Editor，但支援 macOS + Linux（Windows 後置）。核心價值是：把 `~/.ssh/config`、金鑰、ssh-agent、`known_hosts` 的管理，從手動編輯文字檔變成安全、不破壞格式的圖形化操作。

### 目標 (Goals)

1. **無損 round-trip 編輯** `~/.ssh/config`（含 `Include` 多檔）——保留註解、空行、縮排、大小寫、`=` vs 空格分隔符、未知指令與順序。
2. **完整 SSH 管理**：config 編輯、SSH 金鑰管理、ssh-agent 整合、`known_hosts` 管理、port forwarding 設定。
3. **一鍵連線**：在使用者慣用的終端機開 `ssh <alias>`。
4. **以檔案為唯一真實來源**（single source of truth）：不維護會與檔案分歧的旁路資料庫。

### 非目標 (Non-Goals，v1)

- ❌ 內嵌終端機（embedded terminal）—— 列為後續 stretch（§10）。
- ❌ 憑證/密碼保險庫（credential vault）—— 機密一律交給 ssh-agent / macOS Keychain。
- ❌ SFTP 檔案瀏覽器、in-process SSH client（russh）。
- ❌ Mac App Store 上架（sandbox 會擋住直接存取 `~/.ssh` 與呼叫 ssh）。
- ❌ 編輯系統層級 `/etc/ssh/*`（需提權）—— 只處理使用者檔案。

---

## 2. 平台與技術線 (Stack)

| 層 | 技術 | 版本（2026-06 查證） |
|---|---|---|
| 桌面殼層 | **Tauri 2** | core 2.9.x（最新 2.9.6） |
| 後端 | **Rust**（所有特權操作） | 1.77.2+ |
| 前端框架 | **React + TypeScript + Vite** | @tauri-apps/api 2.11.x、vite 8.x、plugin-react 6.x |
| UI | **shadcn/ui + Tailwind CSS v4** | shadcn CLI 4.10.x、tailwind 4.3.x（CSS-first、`@tailwindcss/vite`） |
| 後端狀態 | **TanStack Query v5** | 5.101.x |
| UI 狀態 | **Zustand v5** | 5.0.x |
| 表單 | **react-hook-form + zod** | rhf 7.78.x、zod 4.4.x、@hookform/resolvers 5.4.x |
| 清單虛擬化 | **@tanstack/react-virtual** | 3.14.x |
| 指令面板 | **cmdk**（shadcn Command） | 1.1.x |
| 通知 | **sonner** | 2.0.x |
| 型別橋接 | **tauri-specta** 或 **ts-rs**（自動產生 Rust↔TS 型別） | 社群 crate |

**平台優先序**：macOS + Linux 為 v1；Windows 在程式中以 `#[cfg]` 預留 groundwork，正式支援後置。

---

## 3. 已定案決策 (Decided)

| # | 決策 | 選擇 |
|---|---|---|
| D1 | 功能範圍 | 完整 SSH 管理（config + 金鑰 + agent + known_hosts + port forwarding） |
| D2 | 技術線 | Tauri 2 + React + shadcn/ui（見 §2） |
| D3 | Config 策略 | 無損 round-trip（保留格式與註解） |
| D4 | 平台 | macOS + Linux 優先，Windows 後置 |
| D5 | 連線策略 | **外部終端機**；內嵌終端機列為後續 stretch |
| D6 | 機密範圍 | **只靠 ssh-agent / Keychain**，app 不自存任何密碼/passphrase |
| D7 | 分組來源 | **檔內 sentinel 註解**（如 `#group:Work/Prod`），config 檔為唯一真實來源 |
| D8 | Include 範圍 | **v1 就支援多檔編輯**（`~/.ssh/config` + 所有 Include 進來的檔案） |

實作路線採 **Hybrid**（§5 說明為何不採 Pure-Rust 全包或全 shell-out）。

---

## 4. 架構 (Architecture)

三層、中間隔一層 typed IPC：

```
┌─────────────────────────── React WebView (untrusted) ──────────────────────────┐
│  UI 層: shadcn/ui + Tailwind v4 (master-detail, 表單, cmdk, 視覺化)              │
│  狀態層: TanStack Query (後端讀寫) ┊ Zustand (UI 狀態)                           │
│  資料層: tauriQuery(cmd,args) → invoke() ┊ Channel<T> 串流                        │
└───────────────────────────────────┬─────────────────────────────────────────────┘
                                     │  typed IPC（tauri-specta/ts-rs 產生型別）
                       ┌─────────────┴─────────────┐  ← 安全邊界在這裡
┌──────────────────────┴──── Rust Core (trusted, src-tauri) ──────────────────────┐
│  #[tauri::command] handlers + AppError(thiserror) + Channel 串流                  │
│  config │ keys │ agent │ known_hosts │ connect │ fsutil                          │
│  → 直接用 std/tokio 做檔案 IO 與呼叫 ssh 工具（刻意繞過 fs/shell 沙箱）          │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 安全模型（關鍵）

- **WebView 拿到零 fs、零 shell-exec 權限。** 所有 `~/.ssh` 讀寫與 `ssh/ssh-keygen/ssh-add` 呼叫都在 Rust command 內用 `std::fs`/`tokio`/`std::process` 完成，**刻意繞過** `tauri-plugin-fs`/`tauri-plugin-shell` 沙箱——那層沙箱只約束 WebView，而無損編輯器需要任意 `~/.ssh` 存取會一直跟它打架（且有著名地雷：fs-scope 的 `$HOME/**/*` **匹配不到 `.ssh`** 這種 dotfolder）。
- **真正的安全邊界 = Rust command 介面**：每個前端傳入的路徑都 canonicalize + 驗證（防 path traversal）；連線 alias 用 `^[A-Za-z0-9._@%-]+$` 嚴格驗證、參數以 vector 傳入（**絕不走 `sh -c`**）。
- **Capabilities**（`src-tauri/capabilities/*.json`）只授予：`core:*` 預設 + `dialog`（原生檔案挑選）+ `clipboard-manager`（複製公鑰）+ `os`（平台分支）+ `opener`（在檔案管理員顯示）+ `updater`（自動更新）。**不給 `fs`、不給 `shell`。**

### 串流

`ssh-keygen` 進度、`ssh-add` 輸出、連線 log 用 `tauri::ipc::Channel<TaggedEvent>` 參數串流（非全域 event bus，後者大量輸出會因 JSON eval 退化）。全域 event 只用於少數訊號（agent 鎖定、偵測到 config 被外部改動）。

---

## 5. 為何採 Hybrid 路線

> **共同前提**：無損 round-trip parser **三條路都得自己寫**——沒有任何 Rust crate 能無損 round-trip（`ssh2-config` v0.7.1、`russh-config` v0.58 都是唯讀解析器，寫回會丟註解/空行/順序）。差別只在 keys/agent/known_hosts/connect。

| 路線 | 否決理由 |
|---|---|
| Pure-Rust 全包 | 丟掉 macOS Keychain passphrase 持久化（`ssh-add --apple-use-keychain` 無純 Rust 等價物）、無法加 FIDO/PKCS#11 key；連線若用 russh 等於重寫整套 ssh_config/Match/Include/ProxyJump/agent 語意（數月工程 + 永久維護負擔，且行為會與系統 ssh 默默分歧）。 |
| 全部 shell-out | 解析 `ssh-keygen`/`ssh-add`/`-F` 的 stdout 跨版本/語系脆弱、無 typed 值、硬綁 OpenSSH 在 PATH、拿不到 randomart/即時進度等 typed UI。 |
| **Hybrid（採用）** | 純 Rust 做 byte 相容且自包含的部分（CST parser、金鑰、fingerprint、known_hosts 解析、agent list/sign）；只在系統工具**明顯更強**處 shell-out（`ssh-add` 的 Keychain/FIDO/PKCS#11、終端機連線交給真正的 `ssh` 以 100% 繼承語意）。 |

**Hybrid 的代價（需處理）**：keys/agent 有兩條碼路（純 Rust 查詢 + shell-out add），要偵測 Apple vs Homebrew 的 `ssh-add`；GUI 沒 tty，`ssh-add` 的 passphrase 要靠 askpass helper（§11、§14）。

---

## 6. Rust Core 模組 (src-tauri)

| 模組 | 職責 | 主要 crate（pin 版本） |
|---|---|---|
| `config` | **無損 CST parser/serializer**（§7）、多檔 Include 模型（§8）、HostBlock/MatchBlock/global 分組、per-line/per-block 註解切換、新增/刪除/重排、group/tag sentinel 註解 | 手寫 line scanner（**不用 nom/pest**）、`shellexpand`（僅 Include 路徑/`~` 展開）、`ssh2-config` 0.7.1（**僅唯讀** effective-config 預覽） |
| `keys` | 產生 Ed25519(預設)/ECDSA/RSA(2048/3072/4096)、passphrase 加解密(bcrypt-pbkdf+aes256-ctr)、SHA256 fingerprint + randomart、掃描 `~/.ssh` 配對 `*.pub`/私鑰、偵測加密狀態 | `ssh-key` 0.6.7（features: ed25519, rsa, ecdsa, encryption, getrandom, zeroize）、`rand_core`/OsRng、`secrecy`/`zeroize` |
| `agent` | `SSH_AUTH_SOCK` **執行期**解析（macOS 路徑每次登入會變、絕不快取）、list/lock/unlock/sign（純 Rust）；**add/remove shell-out `/usr/bin/ssh-add`**（Keychain `--apple-use-keychain`、`-t` lifetime、`-c` confirm、FIDO/PKCS#11） | `ssh-agent-lib` 0.6、`service-binding`、`tokio::process` |
| `known_hosts` | 解析所有行型（patterns、`[host]:port`、`|1|salt|hash`、`@cert-authority`/`@revoked`）；自補 **HMAC-SHA1 matcher**（crate 無法比對 hashed）；line-preserving 改寫（移除/停用） | `ssh-key`（known_hosts module）、`hmac` 0.12、`sha1`、`base64ct`、`getrandom` |
| `connect` | `TerminalLauncher` 策略 trait（§10）；macOS `osascript`、Linux 探測 + 可覆寫範本；alias 驗證；參數 vector 傳入；繼承 `DISPLAY`/`WAYLAND_DISPLAY`/`SSH_AUTH_SOCK` | `std::process`/`tokio::process`、`tauri-plugin-os` |
| `fsutil` | 原子寫入（同目錄 temp→fsync→rename）、寫密鑰前先設 0600/0644、`~/.ssh` 不存在則建 0700、session 首次寫入前做時間戳備份、mtime+hash 漂移偵測、路徑 canonicalize/驗證 | `tempfile`、`PermissionsExt`（`#[cfg(unix)]`）、`sha2` |
| `ipc`/`error`/`bindings` | 所有 `#[tauri::command]`、`tauri::Builder`/plugin 註冊/`generate_handler!` 放 `lib.rs`（`main.rs` 為薄 wrapper，為 mobile-ready 而分）、`thiserror` AppError（手動 `Serialize` 成字串）、`Channel<T>` 串流、自動產生 TS 型別 | `tauri` 2、`serde`、`thiserror`、`tauri-specta`/`ts-rs` |

---

## 7. 無損 round-trip parser 設計（核心賭注）

### 設計原則

採 **Concrete Syntax Tree (CST)**，非 abstract——每個節點保留足夠的原始上下文，未編輯時能 byte-for-byte 重吐。編輯只動目標節點，序列化時把每個節點串接：未動的吐 `raw`，改過的用相同分隔符/縮排風格重繪。（技術同 `toml_edit`/`systemd-unit-edit`。）

### Rust AST 草圖

```rust
/// 檔案內依序排列的一個元素
enum Item {
    Blank { raw: String },                  // 空行/純空白行
    Comment { raw: String },                // 整行註解 (# ...)
    Directive(Directive),                   // 一行 key value（可被停用）
    HostBlock(HostBlock),
    MatchBlock(MatchBlock),
}

struct Directive {
    keyword: String,            // 原始大小寫，如 "HostName"
    key: String,                // 正規化小寫 "hostname"（僅供比對）
    value: String,             // quote-aware 的原始值
    separator: Separator,      // 空格 or '='（連同周邊空白原樣保存）
    indent: String,            // 行首空白
    inline_comment: Option<String>, // 行尾 " # ..." 原樣保存
    enabled: bool,             // false → 輸出時整行註解掉
    raw: String,               // fallback：未改動時原樣吐出
}

enum Separator { Space(String), Equals(String) }

struct HostBlock {
    patterns: Vec<String>,      // Host 的 alias / 萬用字元樣式
    header: Directive,          // `Host ...` 那一行本身
    items: Vec<Item>,           // 到下一個 Host/Match 前的所有行
    enabled: bool,              // 整塊註解切換
}

struct MatchBlock { criteria: String, header: Directive, items: Vec<Item> }
```

文件模型（多檔）：

```rust
struct SshConfigDoc { files: Vec<ConfigFile> }     // ~/.ssh/config + 所有 Include 目標
struct ConfigFile {
    path: PathBuf,
    items: Vec<Item>,           // 該檔的 CST（依序）
    mtime: SystemTime,
    hash: [u8; 32],             // 漂移偵測
}
```

UI 的 host 清單 = 所有檔案的 `HostBlock` 串起來（每個標上來源檔，如 badge `config.d/work`）。

### 不變量與測試（最高優先）

- **硬性不變量**：對真實 config 語料庫，`parse(text)` 後 `serialize()` **未編輯時 byte-identical**；單一欄位編輯**只改到該行**。用 golden-file property test 守住——這是整個專案最重要的正確性保證。
- 估計：核心 parser+serializer 約 600–900 LOC（含測試），robust 實作約 1–2 週。

### ssh_config 語意陷阱（parser 必須處理）

1. **First-value-wins，不是 last**：重排或在後面加重複指令在 ssh 裡**無效**。UI 要對被 shadow 的重複指令**警示**。
2. **無 line continuation**：每個實體行就是一個指令，**不要**做反斜線換行接合。
3. **引號內的 `#` 不是註解**：tokenizer 拆行尾註解時必須 quote-aware。
4. **`Key=value` 合法**（`-o` 貼進來的常見），naive `split_whitespace` 會毀掉——分隔符風格逐行保存。
5. **大小寫保留**：`HostName`/`Hostname`/`hostname` 都合法；比對用小寫，未動的行吐原始大小寫。
6. **縮排是裝飾但對使用者有意義**：保留每行行首空白；往區塊加新行時沿用該區塊縮排風格。
7. **未知/未來關鍵字一律當 Directive 直通保存**，永不丟棄（forward-compat + 不毀檔）。

---

## 8. 多檔 Include 模型 (v1)

- 載入時遞迴解析 `Include`（glob 以 `~/.ssh` 為相對基準、**lexical order** 展開），每個目標檔建一份 `ConfigFile` CST。
- **編輯時寫回該 host 實體所在的檔案**（不是無腦寫主檔）。
- UI 明確標示每個 host 的來源檔；**拒絕默默合併**——includes 顯式呈現。
- 新增 host 時讓使用者選目標檔（預設 `~/.ssh/config`）；可在 `config.d/` 建新檔。
- `Include` 影響 first-value-wins（在它的行位置 inline 處理）——若要算 effective config，交給 `ssh2-config` 在正確位置展開，不要自己 append。

---

## 9. IPC Command 介面（示意）

```
# config
config_load()                         -> 解析 ~/.ssh/config(+Includes) 成 DTO；回 hosts/groups/來源檔
config_get_host(alias)                -> 結構化欄位 + extraOptions 供編輯器
config_save_host(alias, changedFields)-> 對 CST 做最小 diff、原子寫 0600、備份；路由到正確來源檔
config_add_host(targetFile, host) / config_remove_host(alias) / config_reorder_hosts(order)
config_toggle_line(alias, lineId, enabled) / config_toggle_host(alias, enabled)  # 註解切換
config_set_group(alias, groupPath) / config_set_tags(alias, tags)                # sentinel 註解
config_effective(hostname)            -> 唯讀解析後的 effective config 預覽（ssh2-config）
config_check_drift()                  -> 比對磁碟 mtime/hash 與快照；偵測外部改動
config_list_files()                   -> 主檔 + 所有 Include 目標檔（供新增 host 選目標）

# keys
keys_list()                           -> 掃 ~/.ssh：配對、演算法、comment、bits、fingerprint、加密旗標
keys_generate(algorithm, bits, path, comment, passphrase?, onProgress: Channel)
keys_change_passphrase(path, oldPass?, newPass?)
keys_fingerprint(path)                -> SHA256 fingerprint + randomart
keys_copy_public(path)                -> 公鑰文字供剪貼簿

# agent
agent_status()                        -> agent 可達？(SSH_AUTH_SOCK) 供優雅 UI 狀態
agent_list_identities()               -> 已載入金鑰 (fingerprint, comment, type)
agent_add(path, passphrase?, lifetime?, confirm?, useKeychain?, onEvent: Channel)  # shell-out ssh-add
agent_remove(path) / agent_remove_all() / agent_lock(pass) / agent_unlock(pass)

# known_hosts
known_hosts_list()                    -> 條目（type/fingerprint/marker；hashed 標為 name-hidden）
known_hosts_find(hostname)            -> 比對明文 + hashed（HMAC matcher）
known_hosts_remove(hostname) / known_hosts_hash_all()   # line-preserving 改寫

# connect / app
connect_launch(alias, terminalOverride?)   -> 在使用者終端機開 ssh <alias>
connect_list_terminals()                   -> 偵測到的終端機（供設定挑選）
app_reveal_in_file_manager(path) / app_platform()
```

所有 command 回 `Result<T, AppError>`；Ok/Err 皆 `Serialize`（Err → reject promise）。

---

## 10. 連線 (Connect)

**v1 = 外部終端機（PATH A）。** `connect_launch(alias)` 只傳 alias，讓 OpenSSH 自己解析 `~/.ssh/config`（自動繼承 ProxyJump/Match/agent，避免「終端機能連、app 連不上」）。

- **alias 驗證** `^[A-Za-z0-9._@%-]+$` 且必須對應到已解析的 config；參數以 vector 傳入，**不走 `sh -c`**（防注入——惡意 config 可能夾帶 shell metacharacter）。
- **macOS**：用 `osascript` AppleScript `do script`/`write text`（**不能用 `open -a`**，它無法帶指令）；Terminal.app 預設、可選 iTerm2。
- **Linux**：解析順序 (1) 使用者自訂範本（含 `{cmd}` placeholder，預設 `gnome-terminal -- {cmd}`）→ (2) `$TERMINAL` → (3) 探測 `ptyxis`(優先，GNOME 新預設)/`gnome-terminal`/`konsole`/`kitty`/`alacritty`/`wezterm`/`foot`/`xfce4-terminal`/`xterm`/`x-terminal-emulator`。各 emulator 的 exec 語法（`--` vs `-e "..."` vs bare args）放內建對照表；永遠保留「自訂指令」設定。Wayland 下無法可靠 raise 新視窗——設定使用者期待。
- 架構：`TerminalLauncher` trait（macOS/Linux/Windows impl + `ExternalTerminal`/未來 `EmbeddedTerminal` 模式切換），讓 PATH B 能並存。
- 也提供「複製 ssh 指令」。

**後續 stretch = 內嵌終端機（PATH B）**：`@xterm/xterm` 6.0 + addon-fit/webgl + `portable-pty` 0.9 spawn **系統 ssh**（保有完整 ssh_config 保真）。參考 `marc2332/tauri-terminal`、`Tnze/tauri-plugin-pty` 但自己擁有程式碼（該 plugin 才 v0.3、~19 stars，不宜當 load-bearing 依賴）。**不用 russh** 除非要做 in-app SFTP/agent。

---

## 11. 金鑰 / Agent / known_hosts 細節

### 金鑰

- 預設 Ed25519；提供 ECDSA、RSA 2048/3072/4096（**RSA-4096 純 Rust 產生慢，放 `spawn_blocking`** 背景跑 + 進度 Channel）。**不產生 DSA**（僅讀取顯示）。
- passphrase 加解密用 `ssh-key` 的 bcrypt-pbkdf + aes256-ctr。secrets 用 `secrecy`/`zeroize`，用完即抹。
- 掃描：`*.pub` 配對私鑰、讀演算法/comment/bits/fingerprint、偵測是否加密（鎖頭圖示）。
- 可一鍵把 IdentityFile 寫進某個 host；寫檔原子化 0600/0644。
- **限制**：`ssh-key` 0.6.x 讀不了 legacy PEM/PKCS#1，遇到罕見舊格式 fallback `ssh-keygen`（§14）。

### Agent（hybrid）

- list/lock/unlock/sign 用純 Rust `ssh-agent-lib`；**add/remove shell-out `/usr/bin/ssh-add`**（拿到 Keychain 持久化、FIDO、PKCS#11）。
- 偵測 Apple vs Homebrew `ssh-add`；macOS 用長旗標 `--apple-use-keychain`/`--apple-load-keychain`（`-K`/`-A` 已棄用），若 PATH 上第一個是非 Apple `ssh-add` 要警示。
- 在 config 編輯器露出 `AddKeysToAgent`/`UseKeychain`（後者僅 macOS）。
- **passphrase UX**：GUI 沒 tty，在 app 對話框收 passphrase，透過內附 askpass helper（`SSH_ASKPASS` + `SSH_ASKPASS_REQUIRE=force`）或 stdin 餵入，用完 zeroize（§14）。

### known_hosts

- 解析所有行型；自補 HMAC-SHA1 matcher（key=salt、msg=小寫 host、非 22 用 `[host]:port`）做 `ssh-keygen -F` 等價查詢。
- 移除/停用走 line-preserving 改寫（parser 對 known_hosts 是 lossy，**絕不整檔重序列化**）。
- 支援 `HashKnownHosts` 轉換（`ssh-keygen -H` 等價）；hashed 條目在 UI 標為「名稱隱藏」，提供「驗證某主機名」輸入框；`@cert-authority`/`@revoked` 標記顯示 badge。

---

## 12. 安全性 (Security) — 彙整

1. 前端零 fs/零 shell 權限；安全邊界在 Rust command 介面。
2. 所有前端傳入路徑 canonicalize + 驗證（防 traversal）。
3. 連線 alias 嚴格字元集 + 參數 vector（防注入）。
4. 寫檔原子化 + session 備份 + 權限（config/key 0600、`.pub`/known_hosts 0644、`~/.ssh` 0700）+ mtime/hash 漂移偵測（檔案是與 ssh CLI 共用的 live config）。
5. 機密只經 ssh-agent/Keychain，app 不落地任何密碼/passphrase；記憶體中 secrets 用 `zeroize`。

---

## 13. 分期交付計畫 (Phased Delivery)

| Phase | 範圍 | 驗收 |
|---|---|---|
| **0 鷹架與邊界** | create-tauri-app(React+TS+Vite)、Tailwind v4(`@tailwindcss/vite`)+shadcn init（`@/` alias 要同時設 tsconfig.json/tsconfig.app.json/vite.config.ts）、capabilities 只給 core+dialog+clipboard+os+opener（**不給 fs/shell**）、TanStack Query/Zustand 接好、`AppError`、tauri-specta/ts-rs 型別管線、`fsutil` 地基（原子寫/權限/備份/漂移） | app 能啟動；一條 echo command 走完 typed IPC；`fsutil` 單元測試過 |
| **1 Config 無損 CRUD** | CST parser/serializer + **golden-file byte-identical property test**；HostBlock/MatchBlock/global；per-line/per-block 註解切換；新增/刪除/重排 host；master-detail 虛擬清單 + 搜尋/過濾；groups+tags(sentinel 註解)；**多檔 Include 編輯**(D8) + 來源檔標示；外部改動 reload 提示；Host 編輯器(Connection/Auth/Forwarding/Reliability 分頁 + extraOptions)，rhf+zod、dirtyFields 最小 diff 存檔；macOS-only 才顯示 `UseKeychain` | golden-file 全過；編輯任一 host 只改到該行；跨檔編輯寫對檔 |
| **2 連線** | `TerminalLauncher` trait；macOS osascript(Terminal+iTerm)；Linux 探測 + 範本 + `$TERMINAL`；alias 驗證 + 環境繼承；cmdk 快速連線；複製 ssh 指令；終端機設定挑選 | 在三種以上 Linux 終端機 + macOS Terminal/iTerm 成功開連線 |
| **3 金鑰** | 掃描 + 加密狀態鎖頭；產生 Ed25519/ECDSA/RSA(RSA-4096 背景+進度 Channel)；改/移除 passphrase；fingerprint+randomart 面板；複製公鑰；一鍵寫 IdentityFile | 產生的金鑰用系統 ssh 可正常連線；權限正確 |
| **4 Agent** | `SSH_AUTH_SOCK` 執行期解析 + 優雅 no-agent 狀態；list（純 Rust）；lock/unlock；add/remove(ssh-add shell-out，askpass 餵 passphrase，Apple vs Homebrew 偵測，`--apple-use-keychain`) | 加/移除 key 反映在 `ssh-add -l`；macOS Keychain 持久化生效 |
| **5 known_hosts + port forward** | hashed 查詢(HMAC matcher)；移除/停用(line-preserving)；`HashKnownHosts` 轉換；`@cert-authority`/`@revoked` badge；Local/Remote/Dynamic Forward 結構化編輯器(bind/listen/dest host/dest port，可重複) + 視覺化圖(友善 port 名 + 複製 `ssh -L`)；ProxyJump 引用已設定 alias 的下拉 | 轉發編輯 round-trip 無損；hashed 主機查得到 |
| **6 打磨與發佈** | 深色模式(`.dark` 持久化)、sonner toast、內嵌 ssh_config(5) 關鍵字說明、模板/snippets、per-host 備註；macOS Developer ID 簽章 + notarytool 公證 + hardened runtime（dmg 安裝 + `.app.tar.gz` 更新）；Linux AppImage(更新) + `.deb`/`.rpm`(安裝)；自動更新(獨立 `TAURI_SIGNING_PRIVATE_KEY`，pubkey 進 config，GitHub Releases 放 `latest.json`)；Windows groundwork 藏 `#[cfg]`(ACL 權限、`\\.\pipe\openssh-ssh-agent`、`wt.exe`) | 簽章/公證後 macOS Gatekeeper 不擋；自動更新通道可走 |

---

## 14. 風險與緩解 (Risks & Mitigations)

1. **無損保真（#1 風險）**：naive parser 會默默正規化、毀掉 live 檔。→ `raw` fallback + golden-file property test + 只寫 dirtyFields。
2. **First-match-wins + 重排/分組改變 ssh 行為**：→ 保留評估順序、對被 shadow 的重複指令警示、group 預設預設只當 UI 便利不默默寫 wildcard Host 區塊。
3. **Agent passphrase 無 tty**：→ app 對話框收 passphrase + 內附 askpass helper（`SSH_ASKPASS`/`SSH_ASKPASS_REQUIRE=force`）或 stdin，zeroize；macOS pin `/usr/bin/ssh-add`。
4. **`SSH_AUTH_SOCK` 多變 / 無 agent**：→ 執行期解析、不硬編、優雅降級給指引。
5. **macOS 簽章/公證 + updater 金鑰**：→ dev 用 ad-hoc unsigned；release 才上 $99/年 Developer ID + notarytool；updater 私鑰離線備份（遺失永久斷更新）；安裝用 dmg/.deb/.rpm，自動更新用 `.app.tar.gz`/AppImage（deb/rpm 不能自更新）。
6. **Include / Match 複雜度**：→ Include 解析保留並可編輯(D8)；**Match 區塊 v1 採 preserve-but-raw**（強塞進 Host UI 會 mangle），明確編輯後續再做。
7. **Path traversal（因繞過 fs-scope）**：→ canonicalize + 驗證所有路徑、連線 alias 嚴格字元集 + vector 參數。
8. **Crate 版本變動 / 格式限制**：→ pin `ssh-key` 0.6.7 與 `ssh-agent-lib` 0.6（對齊 transitive `ssh-key` 版本避免型別 skew）；legacy PEM/FIDO/PKCS#11 fallback `ssh-keygen`/`ssh-add`；0.7 升級待穩定再排。
9. **Linux 終端機/Wayland 碎片化**：→ per-emulator 語法表、優先探測 ptyxis、永遠提供可覆寫自訂指令、繼承 session 環境。

---

## 15. 已採預設的次要決策（可於審閱時推翻）

| 項目 | 預設 | 理由 |
|---|---|---|
| 金鑰產生後端 | 純 Rust `ssh-key`（typed、無二進位依賴、有 randomart） | 罕見 legacy PEM 匯入才 fallback `ssh-keygen` |
| 型別橋接 | 採 `tauri-specta` 或 `ts-rs` 自動產生 | ssh_config 欄位眾多，手寫契約易 drift |
| 新金鑰預設演算法 | Ed25519；RSA 提供 2048/3072/4096；不產生 DSA | 現代 OpenSSH 最佳實務 |
| Match 區塊支援 | v1 preserve-but-raw | 結構化編輯後續再做 |
| 自動更新伺服器 | GitHub Releases 靜態 `latest.json` | 簡單、免維運 |
| macOS 發佈時機 | dev/早期測試 ad-hoc unsigned（附 Gatekeeper 繞過說明）；公開釋出前才上簽章/公證 | 控成本；**永不上 Mac App Store**（sandbox 擋 `~/.ssh`） |
| group 預設是否寫成真實 wildcard Host 區塊 | 否，預設只當 UI 便利 | 避免改變 ssh 行為 |
| 編輯範圍 | 僅使用者檔案（`~/.ssh/*`），不碰 `/etc/ssh` | 免提權 |

---

## 16. 名稱與識別碼 (Decided)

- **產品名稱**：**SSHelter**（ssh + shelter，標語「A shelter for all your SSH hosts」）
- **binary / crate 名**：`sshelter`（查證：crates.io ✅ 空、npm ✅ 空、GitHub 僅 3 個 0★ 無描述 repo）
- **bundle identifier**：`org.homelab.sshelter`
- 影響檔案：`tauri.conf.json`、`src-tauri/capabilities/*.json`、macOS 簽章、updater pubkey 設定。

---

## 附錄 A — 主要參考來源

- OpenSSH `ssh_config(5)` man page（欄位語意與內嵌說明文字的權威來源）
- hejki SSH Editor（UX 主要參考；v2.6.10，macOS-only 商業 app）
- 其他比較對象：Termius、Core Shell、Royal TSX、WindTerm、electerm、Muon(原 Snowflake)
- Tauri 2 docs（capabilities/permissions、IPC、Channel、bundling/signing）
- RustCrypto `ssh-key`、`ssh-agent-lib`、`ssh2-config`、`portable-pty`、`russh` crate 文件
- shadcn/ui + Tailwind v4 + TanStack Query/Virtual + react-hook-form/zod 官方文件
