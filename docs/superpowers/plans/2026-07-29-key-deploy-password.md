# 一鍵部署金鑰 + keychain 密碼儲存 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓使用者在 SSHelter 內一鍵把公鑰部署到目標主機，全程不開終端機，成敗直接顯示在 app 內；部署所需密碼可存進作業系統 keychain 並在 app 內讀回。

**Architecture:** 部署分三步 —— (0) 先用 `ssh-keyscan` 驗好 host key，讓後續 ssh 能用 `StrictHostKeyChecking=yes`，結構性消除「askpass 誤答 host key 提示造成無窮迴圈」；(1) SSHelter 執行檔以 `SSH_ASKPASS` 再次啟動自己進入 helper 模式，從 keychain 讀密碼印到 stdout，密碼全程不進 argv；(2) 不使用 `ssh-copy-id`（Windows OpenSSH 沒有），改以純 argv 啟動 `ssh -T` 執行一段固定的遠端 script，公鑰內容走 stdin。

**Tech Stack:** Rust / Tauri 2 / `keyring` 4.1.5 / React 19 / TanStack Query / shadcn + Radix / vitest

## Global Constraints

- 設計來源：`docs/superpowers/specs/2026-07-29-key-deploy-password-design.md`（commit `11d5937`）。
- 語言慣例：**文件與註解用繁體中文或英文皆可（跟隨該檔案既有風格），程式碼識別字與 commit message 一律英文**。commit 遵循 Conventional Commits。
- 本 repo 直接 commit 到 `main`，不開 worktree、不開 PR。
- **本機端絕不使用 `sh -c`** —— 所有本機進程一律以 argv vector 啟動（既有安全模型，見 `connect.rs` 模組註解）。遠端 script 由**遠端** shell 解析，不違反此規則。
- **公鑰內容一律走 stdin**，絕不拼進遠端指令字串（`.pub` 的 comment 欄位是使用者可控內容）。
- 所有 alias 進入任何指令前必須先過 `connect::validate_alias`（拒絕前導 `-` 的選項注入）。
- 所有 `.pub` 路徑必須先過 `keys::validate_public_path`（canonicalize 後仍須在 `~/.ssh` 內且仍以 `.pub` 結尾）。
- keychain service 名稱固定為字串 `"SSHelter"`。account 命名：正式 `host:<alias>`、暫存 `deploy-tmp:<alias>`。
- `keyring` 版本固定 `4.1.5`（MSRV 1.88.0）。API：`keyring::Entry::new(service, username) -> keyring::Result<Entry>`、`.set_password(&str)`、`.get_password() -> Result<String>`、`.delete_credential()`。`keyring::Error::NoEntry` 代表「無此項目」，是 `get`/`delete` 要當成正常情況處理的變體。
- **`available()` 不可靠 `NoDefaultStore` 判斷（2026-07-29 實測更正，使用者已裁定）**：`keyring::Entry::new` 的初始化旗標是「先 `compare_exchange` 設 true，再呼叫 `set_credential_store()?`」，而 Linux 的 `zbus_secret_service_keyring_store::Store::new()` 內部會 `Service::new()?` 真的連 D-Bus。因此在沒有 Secret Service 的 Linux 上，**第一次**呼叫回傳的是連線類錯誤而非 `NoDefaultStore`，第二次以後才是 `NoDefaultStore` —— 亦即在這個機制唯一存在意義的平台上，第一次呼叫會錯答「可用」。正確寫法是**任何 `Err` 都視為不可用**：`keyring::Entry::new(SERVICE, "availability-probe").is_ok()`。
- 前端所有後端呼叫走 `src/lib/ipc.ts` 的 `tauriInvoke<T>(cmd, args)`。
- ts-rs 型別以 `#[cfg_attr(test, derive(ts_rs::TS))]` + `#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]` 宣告，執行 `cargo test` 時產生到 `src/bindings/`。
- **serde 命名的陷阱（會咬人，請先讀）**：`#[serde(rename_all = "camelCase")]` 標在 **enum** 上只改**變體名**，**不改變體內的欄位名**；標在 **struct** 上才會改欄位名。所以 `HostKeyStatus::New { key_line }` 在 JSON 裡是 `{"kind":"new","key_line":"…"}`（欄位維持 snake_case），而 `DeployPreflight` 的欄位會變成 `askpassSupported`。前端存取時務必以產生出來的 `src/bindings/*.ts` 為準，不要憑印象寫。
- 既有測試數為 283（197 Rust + 86 vitest），每個 task 結束時既有測試不得變紅。

---

### Task 0: 驗證三個未證實假設（spike）

Spec 的「待驗證假設」列了三件不可當成事實的事。**假設 1 若不成立，Task 4–6 的架構要整個改走 pty**，所以必須先做完這個 task 才能開始寫 production code。本 task 不寫任何 production code。

**Files:**
- Create: `docs/superpowers/plans/2026-07-29-key-deploy-spike-results.md`

**Interfaces:**
- Consumes: 無
- Produces: 一份驗證報告，以及 **Task 2 要用的真實提示字串**（`SSHELTER_REAL_PASSWORD_PROMPT`、`SSHELTER_REAL_HOSTKEY_PROMPT`），Task 2 的白名單測試會直接引用這兩個實測字串。

- [ ] **Step 1: 起一台可用密碼登入的拋棄式 sshd**

```bash
docker run -d --name sshelter-spike -p 2222:2222 \
  -e PASSWORD_ACCESS=true -e USER_NAME=spike -e USER_PASSWORD=hunter2 \
  -e PUID=1000 -e PGID=1000 \
  linuxserver/openssh-server
# 等容器就緒
sleep 10 && docker logs sshelter-spike | tail -5
```

若環境沒有 Docker，改用任何一台你有密碼的真實主機，並把下面指令的 `-p 2222 spike@localhost` 換掉。

- [ ] **Step 2: 寫一支會記錄提示文字的假 askpass**

```bash
SP=/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike
mkdir -p "$SP"
cat > "$SP/askpass.sh" <<'EOF'
#!/bin/sh
printf '%s\n' "PROMPT>>>$1<<<" >> "$SPIKE_LOG"
printf '%s\n' 'hunter2'
EOF
chmod +x "$SP/askpass.sh"
```

- [ ] **Step 3: 驗證假設 1 —— `force` 是否攔截 password 認證提示**

```bash
SP=/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike
rm -f "$SP/prompts.log"
SPIKE_LOG="$SP/prompts.log" \
SSH_ASKPASS="$SP/askpass.sh" \
SSH_ASKPASS_REQUIRE=force \
  ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
      -o NumberOfPasswordPrompts=1 -o PreferredAuthentications=password \
      -p 2222 spike@localhost 'echo DEPLOY_OK'
echo "exit=$?"
cat "$SP/prompts.log"
```

Expected（假設成立）：終端印出 `DEPLOY_OK`、**完全沒有互動要求輸入密碼**，且 `prompts.log` 裡有一行形如 `PROMPT>>>spike@localhost's password: <<<`。

**假設不成立的話**（ssh 仍在終端要求輸入密碼、或 log 是空的）：停止本計畫，回報結果，Task 4–6 需改寫為 pty 方案（Rust 以 `portable-pty` 持有 ssh 的 pty，偵測提示後寫入密碼）。架構其餘部分（Step 0 host key 預檢、keychain、前端）不受影響。

- [ ] **Step 4: 取得真實的 host key 提示字串（給 Task 2 的白名單測試用）**

```bash
SP=/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike
rm -f "$SP/prompts.log"
SPIKE_LOG="$SP/prompts.log" \
SSH_ASKPASS="$SP/askpass.sh" \
SSH_ASKPASS_REQUIRE=force \
  ssh -T -o StrictHostKeyChecking=ask -o UserKnownHostsFile="$SP/empty_known_hosts" \
      -p 2222 spike@localhost 'echo X'
cat "$SP/prompts.log"
```

把 log 裡的 host key 提示原文抄下來 —— Task 2 的測試要用它驗證白名單**拒絕**這個字串。

- [ ] **Step 5: 驗證假設 3 —— macOS 未簽章 app 存取 keychain 的提示行為**

```bash
# 用 security CLI 模擬：建立一筆 generic password 再讀回
security add-generic-password -s SSHelter-spike -a probe -w hunter2
security find-generic-password -s SSHelter-spike -a probe -w
security delete-generic-password -s SSHelter-spike -a probe
```

記錄：讀回時是否跳出「允許存取鑰匙圈」對話框。**假設 2（Tauri bundle 自我啟動為 helper）留待 Task 3 驗證**，因為需要先有 helper 模式的程式碼。

- [ ] **Step 6: 寫下結果並 commit**

把三個假設各自的「成立／不成立 + 實測輸出」寫進 `docs/superpowers/plans/2026-07-29-key-deploy-spike-results.md`，包含 Step 3 與 Step 4 抄下來的兩段真實提示字串。

```bash
docker rm -f sshelter-spike 2>/dev/null || true
git add docs/superpowers/plans/2026-07-29-key-deploy-spike-results.md
git commit -m "docs(plans): record askpass/keychain spike results"
```

---

### Task 1: `secrets.rs` —— keychain 封裝

**Files:**
- Create: `src-tauri/src/secrets.rs`
- Modify: `src-tauri/Cargo.toml`（新增相依）
- Modify: `src-tauri/src/lib.rs:1-10`（加入 `mod secrets;`）

**Interfaces:**
- Consumes: `crate::error::AppError`
- Produces:
  - `pub const SERVICE: &str = "SSHelter";`
  - `pub fn host_account(alias: &str) -> String`（回傳 `host:<alias>`）
  - `pub fn tmp_account(alias: &str) -> String`（回傳 `deploy-tmp:<alias>`）
  - `pub fn get(account: &str) -> Result<Option<String>, AppError>`（無此項目回 `Ok(None)`）
  - `pub fn set(account: &str, secret: &str) -> Result<(), AppError>`
  - `pub fn delete(account: &str) -> Result<(), AppError>`（無此項目視為成功）
  - `pub fn available() -> bool`

- [ ] **Step 1: 加相依**

編輯 `src-tauri/Cargo.toml`，在 `[dependencies]` 區塊的 `dirs = "5"` 之後加入一行：

```toml
keyring = "4.1.5"
```

- [ ] **Step 2: 寫失敗的測試**

建立 `src-tauri/src/secrets.rs`，先只放測試模組與空的函式宣告：

```rust
//! 作業系統 keychain 的薄封裝（macOS Keychain / Windows Credential Manager / Linux Secret Service）。
//!
//! 密碼只存在這裡：絕不寫進 `~/.ssh/config`（會被 `fsutil::backup()` 帶進備份歷史），
//! 也不進 settings export。account 命名見 `host_account` / `tmp_account`。

use crate::error::AppError;

/// keychain 的 service 名稱，全 app 固定。
pub const SERVICE: &str = "SSHelter";

/// 正式項目：使用者勾了「記住密碼」時使用，不會被自動刪除。
pub fn host_account(alias: &str) -> String {
    format!("host:{alias}")
}

/// 暫存項目：沒勾「記住密碼」時使用，部署結束後一律刪除。
pub fn tmp_account(alias: &str) -> String {
    format!("deploy-tmp:{alias}")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accounts_are_namespaced_and_distinct() {
        assert_eq!(host_account("web"), "host:web");
        assert_eq!(tmp_account("web"), "deploy-tmp:web");
        assert_ne!(host_account("web"), tmp_account("web"));
    }

    #[test]
    fn account_namespaces_cannot_collide_across_aliases() {
        // 一個 alias 的正式項目不可能等於另一個 alias 的暫存項目。
        assert_ne!(host_account("deploy-tmp:web"), tmp_account("web"));
    }
}
```

- [ ] **Step 3: 執行測試確認通過（純字串函式，先建立基線）**

Run: `cd src-tauri && cargo test secrets:: -- --nocapture`
Expected: 2 passed

- [ ] **Step 4: 實作 keychain 存取**

在 `secrets.rs` 的 `tmp_account` 之後、`#[cfg(test)]` 之前插入：

```rust
/// 把 keyring 的錯誤轉成 AppError。`NoEntry` 由呼叫端各自處理，不會走到這裡。
fn map_err(e: keyring::Error) -> AppError {
    match e {
        keyring::Error::NoDefaultStore => AppError::Other(
            "no OS credential store available on this machine".to_string(),
        ),
        other => AppError::Other(format!("keychain error: {other}")),
    }
}

/// 讀取一筆密碼。沒有這筆項目時回 `Ok(None)`（不是錯誤）。
pub fn get(account: &str) -> Result<Option<String>, AppError> {
    let entry = keyring::Entry::new(SERVICE, account).map_err(map_err)?;
    match entry.get_password() {
        Ok(secret) => Ok(Some(secret)),
        Err(keyring::Error::NoEntry) => Ok(None),
        Err(e) => Err(map_err(e)),
    }
}

/// 寫入（或覆蓋）一筆密碼。
pub fn set(account: &str, secret: &str) -> Result<(), AppError> {
    let entry = keyring::Entry::new(SERVICE, account).map_err(map_err)?;
    entry.set_password(secret).map_err(map_err)
}

/// 刪除一筆密碼。項目本來就不存在時視為成功（清理路徑要能重複執行）。
pub fn delete(account: &str) -> Result<(), AppError> {
    let entry = keyring::Entry::new(SERVICE, account).map_err(map_err)?;
    match entry.delete_credential() {
        Ok(()) => Ok(()),
        Err(keyring::Error::NoEntry) => Ok(()),
        Err(e) => Err(map_err(e)),
    }
}

/// 本機是否有可用的密鑰環。false 時呼叫端要改走環境變數 fallback 並告知使用者。
///
/// 刻意把「任何錯誤」都視為不可用，而不是只認 `NoDefaultStore`：在沒有 Secret Service
/// 的 Linux 上，第一次呼叫拿到的是 D-Bus 連線錯誤，第二次以後才是 `NoDefaultStore`
/// （`Entry::new` 會先把初始化旗標設成 true 才嘗試建 store）。只認單一變體會在這個
/// 機制唯一存在意義的平台上、第一次呼叫時錯答「可用」。
pub fn available() -> bool {
    keyring::Entry::new(SERVICE, "availability-probe").is_ok()
}
```

- [ ] **Step 5: 加註冊並寫 round-trip 測試**

編輯 `src-tauri/src/lib.rs`，在第 8 行 `mod settings_io;` 之前插入 `mod secrets;`（維持字母序）。

在 `secrets.rs` 的 `mod tests` 內追加：

```rust
    /// 測試結束時一律嘗試刪除，避免中途 panic 把明文密碼留在開發者的真實鑰匙圈裡。
    struct CleanupGuard<'a>(&'a str);
    impl Drop for CleanupGuard<'_> {
        fn drop(&mut self) {
            let _ = delete(self.0);
        }
    }

    /// 真的碰作業系統 keychain。CI 上沒有可用的密鑰環時自動跳過。
    #[test]
    fn round_trip_set_get_delete() {
        if !available() {
            eprintln!("skipping: no credential store on this machine");
            return;
        }
        let account = "test:round-trip";
        // 必須在 set 之前建立：斷言 panic 時 unwind 會跳過後面的 delete。
        let _cleanup = CleanupGuard(account);
        set(account, "hunter2").expect("set ok");
        assert_eq!(get(account).unwrap().as_deref(), Some("hunter2"));
        delete(account).expect("delete ok");
        assert_eq!(get(account).unwrap(), None, "deleted entry reads back as None");
        // 重複刪除不報錯（清理路徑必須可重入）。
        delete(account).expect("second delete is a no-op");
    }
```

- [ ] **Step 6: 執行測試**

Run: `cd src-tauri && cargo test secrets:: -- --nocapture`
Expected: 3 passed（若本機無密鑰環，第 3 個印出 skipping 仍算 pass）

**同時記錄假設 3 的結果**：跑這個測試時 macOS 是否跳出「允許存取鑰匙圈」對話框，寫進 spike results 文件。

- [ ] **Step 7: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/secrets.rs src-tauri/src/lib.rs
git commit -m "feat(secrets): add OS keychain wrapper for per-host passwords"
```

---

### Task 2: `askpass.rs` —— helper 模式的提示白名單

**Files:**
- Create: `src-tauri/src/askpass.rs`
- Modify: `src-tauri/src/lib.rs`（加入 `pub mod askpass;`）

**Interfaces:**
- Consumes: `crate::secrets::{get, SERVICE}`
- Produces:
  - `pub fn prompt_is_answerable(prompt: &str) -> bool`
  - `pub fn resolve_secret(account: &str, env_secret: Option<String>) -> Option<String>`
  - `pub fn run() -> !`（helper 模式進入點，永不返回）

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/askpass.rs`：

```rust
//! SSH_ASKPASS helper 模式。
//!
//! SSHelter 的執行檔會被 ssh 以 `SSH_ASKPASS` 再次啟動；`main()` 偵測到 `SSHELTER_ASKPASS=1`
//! 就呼叫 `run()`，完全不初始化 GUI。ssh 把提示文字當作 argv[1] 傳進來。
//!
//! 安全性：`SSH_ASKPASS_REQUIRE=force` 會讓「所有」提示都走這裡，包含 host key 驗證。
//! 若無條件印出密碼，就會拿密碼去回答 host key 提示、ssh 再問一次 → 無窮迴圈（已實測，
//! 見 spike 記錄）；而且 keyboard-interactive 的提示文字由伺服器控制，惡意主機可藉此
//! 騙走密碼。因此這裡採白名單：只回應真正的密碼／passphrase 提示。
//!
//! **但「拒絕」不是免費的，也不是安全的預設。** `readpass.c` 的 `read_passphrase` 在
//! askpass 失敗時，若呼叫端沒帶 `RP_ALLOW_EOF` 就 `return xstrdup("")`，而 `sshconnect2.c`
//! 的兩個認證呼叫點都沒帶。所以 exit 1 會讓 ssh **送出一個空密碼**，在遠端留下一筆真實
//! 的失敗認證；只有 host key 的 `confirm()` 會把空字串當成 "no" 而安全失敗。
//!
//! 真正的結構性防線因此不在這個白名單，而在部署 argv 的 `-o KbdInteractiveAuthentication=no`
//! （見 `deploy::build_deploy_argv`）：關掉之後，helper 收到的提示全部由 client 產生，
//! 伺服器可控的文字根本進不來。**移除那個選項會讓這個白名單重新暴露在伺服器可控的輸入
//! 之下 —— 它不是效能調校。**

/// 只回應真正的密碼／passphrase 提示。
///
/// 兩端錨定，永遠不用 `contains` —— 對攻擊者可控的字串做無錨定子字串比對等於沒有防禦。
///
/// OpenSSH 8.5 起，client 會把 keyboard-interactive 提示加上自己產生的 `(user@host) `
/// 前綴（`sshconnect2.c` 的 `asmprintf(&display_prompt, …, "(%s@%s) %s", …)`），前綴
/// 「之後」的文字則完全由伺服器控制。因此：有前綴 → 剝掉後只接受完全等於 `password:`；
/// 無前綴 → 是 client 自己組的固定形狀，逐一錨定比對。
pub fn prompt_is_answerable(prompt: &str) -> bool {
    // 多行提示只有 host key 確認一種，一律拒絕。
    if prompt.contains('\n') {
        return false;
    }
    let trimmed = prompt.trim();

    if let Some(rest) = strip_kbdint_prefix(trimmed) {
        return rest.trim().eq_ignore_ascii_case("password:");
    }

    let lower = trimmed.to_ascii_lowercase();
    is_client_password_prompt(&lower) || lower.starts_with("enter passphrase for ")
}

/// 剝除 client 產生的 `(user@host) ` 前綴；沒有前綴時回 `None`。
/// 前綴內容由 `"%s@%s"` 組成，不含空白。
fn strip_kbdint_prefix(s: &str) -> Option<&str> {
    let rest = s.strip_prefix('(')?;
    let close = rest.find(')')?;
    let inside = &rest[..close];
    if inside.is_empty() || inside.contains(char::is_whitespace) || !inside.contains('@') {
        return None;
    }
    Some(rest[close + 1..].trim_start())
}

/// `<user>@<host>'s password:` —— user 與 host 皆非空且不得含空白。
/// 這道形狀檢查正是 `Please enter your account's password:` 被擋下的原因。
fn is_client_password_prompt(lower: &str) -> bool {
    let Some(head) = lower.strip_suffix("'s password:") else {
        return false;
    };
    if head.contains(char::is_whitespace) {
        return false;
    }
    match head.split_once('@') {
        Some((user, host)) => !user.is_empty() && !host.is_empty(),
        None => false,
    }
}

/// 診斷紀錄一律走 stderr —— stdout 是 ssh 讀取答案的通道，寫任何東西進去都會被當成密碼。
/// 絕不記錄密碼本身。前綴讓 `deploy::classify_outcome` 能濾掉這些行。
fn log_decision(prompt: &str, decision: &str) {
    eprintln!("[sshelter-askpass] {decision}: {prompt:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_real_openssh_password_prompts() {
        // Task 0 Step 3 實測到的字串。
        assert!(prompt_is_answerable("spike@localhost's password: "));
        assert!(prompt_is_answerable("frank@10.0.0.9's password: "));
        // keyboard-interactive：OpenSSH 8.5+ 一定帶 client 產生的 `(user@host) ` 前綴。
        // 裸的 `Password: ` 沒有任何 ssh 路徑會產生，刻意不接受。
        assert!(prompt_is_answerable("(frank@10.0.0.9) Password: "));
        assert!(!prompt_is_answerable("Password: "));
    }

    #[test]
    fn accepts_key_passphrase_prompts() {
        assert!(prompt_is_answerable(
            "Enter passphrase for key '/Users/frank/.ssh/id_ed25519': "
        ));
    }

    #[test]
    fn rejects_host_key_prompt() {
        // 這是造成無窮迴圈的關鍵案例（Task 0 Step 4 實測字串）。
        assert!(!prompt_is_answerable(
            "Are you sure you want to continue connecting (yes/no/[fingerprint])? "
        ));
        assert!(!prompt_is_answerable(
            "The authenticity of host '[localhost]:2222' can't be established."
        ));
    }

    #[test]
    fn rejects_server_controlled_lookalike_prompts() {
        // 伺服器可任意指定 keyboard-interactive 的提示文字。
        assert!(!prompt_is_answerable("Please enter your password:"));
        assert!(!prompt_is_answerable("Type the password: now"));
        assert!(!prompt_is_answerable("Verification code: "));
        assert!(!prompt_is_answerable(""));
        assert!(!prompt_is_answerable("   "));
    }
}
```

> 執行前先把 Task 0 Step 3 / Step 4 抄下來的真實字串替換掉上面對應的 assert，確保測的是實測值而非推測值。

- [ ] **Step 2: 執行測試確認通過**

Run: `cd src-tauri && cargo test askpass:: -- --nocapture`
Expected: 5 passed

- [ ] **Step 3: 加上密碼取得與 run() 進入點**

在 `askpass.rs` 的 `prompt_is_answerable` 之後、`#[cfg(test)]` 之前插入：

```rust
/// 取得要回覆的密碼：優先用環境變數 fallback（本機無密鑰環時），否則查 keychain。
///
/// 空字串一律當成「沒有密碼」。否則 `run()` 會印出一行空白並 exit 0，而 ssh 會把那個
/// 空字串當成密碼送給伺服器（見 `run()` 的說明）。
pub fn resolve_secret(account: &str, env_secret: Option<String>) -> Option<String> {
    if let Some(s) = env_secret {
        return if s.is_empty() { None } else { Some(s) };
    }
    crate::secrets::get(account)
        .ok()
        .flatten()
        .filter(|s| !s.is_empty())
}

/// helper 模式進入點。永不返回。
///
/// 退出碼：0 = 已把密碼完整寫到 stdout；1 = 沒有回答。
///
/// **重要：exit 1 不等於「ssh 會安全地放棄」。** `readpass.c` 的 `read_passphrase` 在
/// askpass 失敗時，若呼叫端沒帶 `RP_ALLOW_EOF` 就 `return xstrdup("")`，而
/// `sshconnect2.c` 的兩個認證呼叫點都沒帶。也就是說 exit 1 會讓 ssh 送出一個「空密碼」，
/// 在遠端留下一筆真實的失敗認證。只有 host key 的 `confirm()` 會把空字串當成 "no" 而
/// 安全失敗。這正是部署 argv 必須帶 `-o KbdInteractiveAuthentication=no` 的原因：讓
/// 伺服器可控的提示根本不會出現，helper 就不必在「洩漏」與「送空密碼」之間二選一。
pub fn run() -> ! {
    use std::io::Write;

    let prompt = std::env::args().nth(1).unwrap_or_default();
    if !prompt_is_answerable(&prompt) {
        log_decision(&prompt, "refused");
        std::process::exit(1);
    }

    let account = std::env::var("SSHELTER_ASKPASS_ACCOUNT").unwrap_or_default();
    let env_secret = std::env::var("SSHELTER_ASKPASS_SECRET").ok();

    match resolve_secret(&account, env_secret) {
        Some(secret) => {
            let stdout = std::io::stdout();
            let mut lock = stdout.lock();
            // ssh 讀一行；結尾必須有換行。寫入或 flush 失敗時絕不能 exit 0 ——
            // 那會讓 ssh 把「只寫出一半的密碼前綴」當成答案送給對方。
            if writeln!(lock, "{secret}").is_err() || lock.flush().is_err() {
                log_decision(&prompt, "write-failed");
                std::process::exit(1);
            }
            std::process::exit(0);
        }
        None => {
            log_decision(&prompt, "no-secret");
            std::process::exit(1)
        }
    }
}
```

編輯 `src-tauri/src/lib.rs`，在第 1 行 `mod config;` 之前插入 `pub mod askpass;`（`main.rs` 要用，必須是 `pub`）。

- [ ] **Step 4: 為 resolve_secret 加測試**

在 `mod tests` 內追加：

```rust
    #[test]
    fn env_secret_takes_priority_over_keychain() {
        // 環境變數 fallback 存在時，不去碰 keychain（本機無密鑰環也能運作）。
        let got = resolve_secret("host:nonexistent-alias", Some("from-env".to_string()));
        assert_eq!(got.as_deref(), Some("from-env"));
    }

    #[test]
    fn missing_account_and_no_env_yields_none() {
        let got = resolve_secret("host:definitely-not-a-real-account-xyz", None);
        assert_eq!(got, None);
    }
```

- [ ] **Step 5: 執行測試**

Run: `cd src-tauri && cargo test askpass:: -- --nocapture`
Expected: 7 passed

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/askpass.rs src-tauri/src/lib.rs
git commit -m "feat(askpass): add SSH_ASKPASS helper mode with prompt whitelist"
```

---

### Task 3: 把 helper 模式掛上 `main()` 並驗證假設 2

**Files:**
- Modify: `src-tauri/src/main.rs:4-6`

**Interfaces:**
- Consumes: `sshelter_lib::askpass::run`
- Produces: 執行檔在 `SSHELTER_ASKPASS=1` 時的 helper 行為（Task 6 依賴此行為）

- [ ] **Step 1: 修改 main.rs**

把 `src-tauri/src/main.rs` 全檔改成：

```rust
// Prevents additional console window on Windows in release, DO NOT REMOVE!!
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

fn main() {
    // SSH_ASKPASS helper 模式：ssh 會用這支執行檔再次啟動我們來要密碼。
    // 必須在任何 Tauri 初始化「之前」攔截，helper 模式完全不開 GUI。
    if std::env::var_os("SSHELTER_ASKPASS").is_some() {
        sshelter_lib::askpass::run();
    }
    sshelter_lib::run()
}
```

- [ ] **Step 2: 建置**

Run: `cd src-tauri && cargo build`
Expected: 編譯成功，無 warning 新增

- [ ] **Step 3: 驗證 helper 模式不開 GUI 且行為正確**

```bash
cd src-tauri
BIN=./target/debug/sshelter

# 白名單拒絕 → exit 1，無輸出，且沒有視窗跳出
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 \
  "$BIN" "Are you sure you want to continue connecting (yes/no/[fingerprint])? "
echo "rejected exit=$?"   # 期望 1

# 白名單接受 → exit 0 並印出密碼
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 \
  "$BIN" "spike@localhost's password: "
echo "accepted exit=$?"   # 期望 0，前一行印出 hunter2
```

Expected: 第一次 `exit=1` 且無任何輸出；第二次印出 `hunter2` 且 `exit=0`。**兩次都不得有視窗閃現。**

- [ ] **Step 4: 驗證假設 2 —— 打包後的 bundle 同樣可行**

```bash
cd /Users/ysya/project/homelab/ssheditor
# DMG 打包步驟（bundle_dmg.sh，走 Finder/AppleScript）在非互動 shell 會失敗，
# 導致整個指令回傳非零 —— 但 .app 在那之前就已經完整產出，不影響本步驟。
pnpm tauri build --debug
# bundle 內的執行檔名來自 Cargo package name（小寫 `sshelter`），不是 productName
# （`SSHelter`）。不要寫死 —— 用萬用字元探測，順便確認只有一個候選。
APP=$(ls src-tauri/target/debug/bundle/macos/SSHelter.app/Contents/MacOS/*)
echo "bundle executable: $APP"
# 用「會被接受」的提示形狀。裸的 `Password: ` 在 Task 2 收緊白名單後已刻意被拒絕。
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 "$APP" "spike@localhost's password: "
echo "bundle exit=$?"
```

Expected: 印出 `hunter2`、`exit=0`、**Dock 沒有出現 app 圖示、沒有視窗**。

> 「沒有視窗」這半件事**無法從非互動 shell 證實**。可用的替代訊號：程序存活時間（helper 模式應為毫秒級，真正啟動 GUI 是秒級）、Launch Services 有無新註冊、有無殘留程序或當機報告。這些都成立時只能說「傾向成立」；要真正確認需要人在互動式終端機執行並親眼看。**不要把替代訊號寫成已證實。**

若 bundle 版會啟動 GUI 或無法乾淨退出 → 假設 2 不成立，改為以 Tauri sidecar（`bundle.externalBin`）提供一支獨立的 helper 執行檔，並在 Task 6 把 `SSH_ASKPASS` 指向該 sidecar 而非 `current_exe()`。把結果補進 spike results 文件。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/main.rs docs/superpowers/plans/2026-07-29-key-deploy-spike-results.md
git commit -m "feat(askpass): dispatch to helper mode before Tauri init"
```

---

### Task 4: `deploy.rs` 純函式 —— argv 組裝、遠端 script、結果分類

**Files:**
- Create: `src-tauri/src/deploy.rs`
- Modify: `src-tauri/src/lib.rs`（加入 `mod deploy;`）

**Interfaces:**
- Consumes: `crate::error::AppError`
- Produces:
  - `pub const REMOTE_SCRIPT: &str`
  - `pub fn build_deploy_argv(alias: &str) -> Vec<String>`
  - `pub enum DeployOutcome { Added, AlreadyPresent, WrongPassword, HostKeyFailed, Unreachable, RemoteError { code: i32 }, Other { message: String } }`（Serialize + ts-rs 匯出）
  - `pub fn classify_outcome(code: Option<i32>, stdout: &str, stderr: &str) -> DeployOutcome`

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/deploy.rs`：

```rust
//! 一鍵部署公鑰：全程在 Rust 內完成，不開終端機。
//!
//! 刻意不使用 `ssh-copy-id` —— Windows OpenSSH 沒有這支程式，且自己實作才控制得了
//! 錯誤分類（要在 app 內分辨「密碼錯」與「連不上」）。
//!
//! 安全模型：本機以純 argv 啟動 ssh（絕不 `sh -c`）；遠端 script 是一段固定字串，
//! 不含任何使用者輸入；公鑰內容走 stdin，因為 `.pub` 的 comment 欄位是使用者可控的，
//! 拼進遠端指令即為注入點。

use serde::{Deserialize, Serialize};

/// 在遠端執行的固定 script。不含任何使用者輸入；公鑰從 stdin 讀入。
/// 退出碼 90/91/92 用來區分遠端失敗的階段。
pub const REMOTE_SCRIPT: &str = r#"umask 077
mkdir -p ~/.ssh || exit 90
k=$(cat)
[ -n "$k" ] || exit 91
if [ -f ~/.ssh/authorized_keys ] && grep -qxF "$k" ~/.ssh/authorized_keys; then
  echo SSHELTER_EXISTS
else
  printf '%s\n' "$k" >> ~/.ssh/authorized_keys || exit 92
  chmod 600 ~/.ssh/authorized_keys
  echo SSHELTER_ADDED
fi"#;

/// 部署結果。前端據此顯示對應文案。
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "camelCase")]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub enum DeployOutcome {
    /// 公鑰已新增到遠端 authorized_keys。
    Added,
    /// 遠端本來就有這把公鑰，未重複加入。
    AlreadyPresent,
    /// 密碼錯誤（只嘗試一次）。
    WrongPassword,
    /// host key 驗證失敗（理論上 Step 0 已擋掉，屬防禦性分類）。
    HostKeyFailed,
    /// 連不上：DNS 解析失敗、逾時、連線被拒。
    Unreachable,
    /// 遠端 script 失敗，帶回退出碼。
    RemoteError { code: i32 },
    /// 其他未分類錯誤，帶回可讀訊息。
    Other { message: String },
}

/// 組出 `ssh` 的 argv（不含程式名）。`alias` 必須已通過 `connect::validate_alias`。
pub fn build_deploy_argv(alias: &str) -> Vec<String> {
    vec![
        // 不配置 pty：ssh 沒有終端機可提示，只能走 askpass；同時讓 stdin 乾淨傳遞公鑰。
        "-T".to_string(),
        // host key 已於 Step 0 驗好並寫入 known_hosts，這裡不接受任何提示。
        "-o".to_string(), "StrictHostKeyChecking=yes".to_string(),
        // 防使用者 config 全域設了 `BatchMode yes` 而封鎖密碼提示。
        "-o".to_string(), "BatchMode=no".to_string(),
        // 密碼錯就立刻失敗，不問三次。
        "-o".to_string(), "NumberOfPasswordPrompts=1".to_string(),
        // 關掉 keyboard-interactive。這是安全關鍵，不是效能調校：kbdint 的提示文字由
        // 「伺服器」控制，而 OpenSSH 只在前面加一個 client 產生的 `(user@host) ` 前綴就
        // 原文轉交 askpass。關掉之後，helper 收到的提示全部由 client 產生，惡意主機再也
        // 無法構造提示來誘騙 helper 印出密碼。與 Step 0 用 StrictHostKeyChecking=yes
        // 消除 host key 提示是同一個思路：讓危險的輸入根本不存在，而不是事後過濾。
        // 代價：只提供 kbdint 的主機（PAM 2FA 等）無法自動部署，但會「乾淨地」失敗。
        "-o".to_string(), "KbdInteractiveAuthentication=no".to_string(),
        "-o".to_string(), "ConnectTimeout=10".to_string(),
        alias.to_string(),
        REMOTE_SCRIPT.to_string(),
    ]
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn argv_pins_host_key_and_limits_password_attempts() {
        let argv = build_deploy_argv("web");
        assert!(argv.contains(&"-T".to_string()), "must not allocate a pty");
        assert!(argv.contains(&"StrictHostKeyChecking=yes".to_string()));
        assert!(argv.contains(&"BatchMode=no".to_string()));
        assert!(argv.contains(&"NumberOfPasswordPrompts=1".to_string()));
        assert!(argv.contains(&"ConnectTimeout=10".to_string()));
    }

    #[test]
    fn argv_disables_keyboard_interactive() {
        // 安全關鍵：kbdint 的提示文字由伺服器控制，關掉它才能保證 askpass helper
        // 收到的提示全部是 client 產生的。移掉這個選項會讓白名單重新暴露在
        // 伺服器可控的輸入之下。
        assert!(build_deploy_argv("web").contains(&"KbdInteractiveAuthentication=no".to_string()));
    }

    #[test]
    fn argv_puts_alias_before_the_remote_script() {
        let argv = build_deploy_argv("web");
        let alias_at = argv.iter().position(|a| a == "web").expect("alias present");
        let script_at = argv.iter().position(|a| a == REMOTE_SCRIPT).expect("script present");
        assert!(alias_at < script_at, "alias must precede the remote command");
        assert_eq!(script_at, argv.len() - 1, "remote script is the last argv element");
    }

    #[test]
    fn argv_never_sets_preferred_authentications() {
        // 刻意不設：使用者若已有可用金鑰，ssh 根本不會呼叫 askpass，部署直接成功。
        let argv = build_deploy_argv("web");
        assert!(
            !argv.iter().any(|a| a.starts_with("PreferredAuthentications")),
            "PreferredAuthentications must be left to the user's config"
        );
    }

    #[test]
    fn remote_script_reads_key_from_stdin_not_argv() {
        // 公鑰必須來自 stdin（`$(cat)`），絕不出現在 script 字串裡。
        assert!(REMOTE_SCRIPT.contains("k=$(cat)"));
        assert!(REMOTE_SCRIPT.contains("umask 077"));
        assert!(REMOTE_SCRIPT.contains("SSHELTER_ADDED"));
        assert!(REMOTE_SCRIPT.contains("SSHELTER_EXISTS"));
        assert!(!REMOTE_SCRIPT.contains("ssh-ed25519"), "no key material in the script");
    }

    #[test]
    fn classify_added_and_already_present_from_stdout_markers() {
        assert_eq!(classify_outcome(Some(0), "SSHELTER_ADDED\n", ""), DeployOutcome::Added);
        assert_eq!(
            classify_outcome(Some(0), "SSHELTER_EXISTS\n", ""),
            DeployOutcome::AlreadyPresent
        );
    }

    #[test]
    fn classify_wrong_password() {
        let stderr = "spike@localhost: Permission denied (publickey,password).";
        assert_eq!(classify_outcome(Some(255), "", stderr), DeployOutcome::WrongPassword);
    }

    #[test]
    fn classify_host_key_failure() {
        let stderr = "Host key verification failed.";
        assert_eq!(classify_outcome(Some(255), "", stderr), DeployOutcome::HostKeyFailed);
    }

    #[test]
    fn classify_unreachable_variants() {
        for stderr in [
            "ssh: Could not resolve hostname nope: nodename nor servname provided",
            "ssh: connect to host 10.0.0.9 port 22: Connection timed out",
            "ssh: connect to host 10.0.0.9 port 22: Connection refused",
            "ssh: connect to host 10.0.0.9 port 22: No route to host",
            "ssh: connect to host 10.0.0.9 port 22: Operation timed out",
        ] {
            assert_eq!(
                classify_outcome(Some(255), "", stderr),
                DeployOutcome::Unreachable,
                "stderr {stderr:?} should classify as Unreachable"
            );
        }
    }

    #[test]
    fn classify_remote_script_exit_codes() {
        assert_eq!(
            classify_outcome(Some(90), "", ""),
            DeployOutcome::RemoteError { code: 90 }
        );
        assert_eq!(
            classify_outcome(Some(92), "", ""),
            DeployOutcome::RemoteError { code: 92 }
        );
    }

    #[test]
    fn classify_unknown_falls_back_to_other_with_stderr() {
        let out = classify_outcome(Some(255), "", "something entirely new");
        assert_eq!(
            out,
            DeployOutcome::Other { message: "something entirely new".to_string() }
        );
    }

    #[test]
    fn classify_signal_killed_process_has_no_exit_code() {
        let out = classify_outcome(None, "", "");
        assert!(matches!(out, DeployOutcome::Other { .. }), "got {out:?}");
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test deploy:: 2>&1 | head -20`
Expected: FAIL —— `cannot find function 'classify_outcome' in this scope`

- [ ] **Step 3: 實作 classify_outcome**

在 `deploy.rs` 的 `build_deploy_argv` 之後、`#[cfg(test)]` 之前插入：

```rust
/// stderr 中代表「連不上」的片段。
const UNREACHABLE_MARKERS: &[&str] = &[
    "Could not resolve",
    "Connection timed out",
    "Connection refused",
    "No route to host",
    "Operation timed out",
];

/// 把 ssh 的退出碼與輸出翻譯成使用者看得懂的結果。
///
/// 順序很重要：部署成功時退出碼是 0，所以先看 stdout 的標記；255 是 ssh 自己的
/// 錯誤（連線／認證），其餘退出碼來自遠端 script。
pub fn classify_outcome(code: Option<i32>, stdout: &str, stderr: &str) -> DeployOutcome {
    if stdout.contains("SSHELTER_ADDED") {
        return DeployOutcome::Added;
    }
    if stdout.contains("SSHELTER_EXISTS") {
        return DeployOutcome::AlreadyPresent;
    }
    match code {
        Some(255) => {
            if stderr.contains("Permission denied") {
                DeployOutcome::WrongPassword
            } else if stderr.contains("Host key verification failed") {
                DeployOutcome::HostKeyFailed
            } else if UNREACHABLE_MARKERS.iter().any(|m| stderr.contains(m)) {
                DeployOutcome::Unreachable
            } else {
                DeployOutcome::Other {
                    message: first_line_or(stderr, "ssh failed"),
                }
            }
        }
        Some(code @ (90 | 91 | 92)) => DeployOutcome::RemoteError { code },
        Some(0) => DeployOutcome::Other {
            message: "deploy finished but reported no result".to_string(),
        },
        Some(code) => DeployOutcome::RemoteError { code },
        None => DeployOutcome::Other {
            message: "ssh was terminated by a signal".to_string(),
        },
    }
}

/// 取 stderr 第一行非空內容，作為給使用者看的訊息。
/// askpass helper 的診斷行也走 stderr（見 `askpass::log_decision`），要濾掉 ——
/// 否則使用者看到的錯誤訊息會變成我們自己的除錯輸出。
fn first_line_or(stderr: &str, fallback: &str) -> String {
    stderr
        .lines()
        .map(str::trim)
        .find(|l| !l.is_empty() && !l.starts_with("[sshelter-askpass]"))
        .unwrap_or(fallback)
        .to_string()
}
```

編輯 `src-tauri/src/lib.rs`，在 `mod discover;` 之前插入 `mod deploy;`（維持字母序）。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test deploy:: -- --nocapture`
Expected: 13 passed

- [ ] **Step 5: 確認 ts-rs 型別已產生**

Run: `ls src/bindings/DeployOutcome.ts && cat src/bindings/DeployOutcome.ts`
Expected: 檔案存在，內容是帶 `kind` 判別欄位的 union 型別

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/deploy.rs src-tauri/src/lib.rs src/bindings/DeployOutcome.ts
git commit -m "feat(deploy): add pure argv builder, remote script and outcome classifier"
```

---

### Task 5: `deploy.rs` host key 預檢

**Files:**
- Modify: `src-tauri/src/deploy.rs`（追加）
- Modify: `src-tauri/src/keys.rs:54`（把 `fn ssh_dir` 改成 `pub fn ssh_dir`）

**Interfaces:**
- Consumes: `crate::known_hosts::fingerprint_sha256`（`effective_config` 要到 Task 6 才用到）
- Produces:
  - `pub struct Endpoint { pub hostname: String, pub port: String }`
  - `pub fn endpoint_from_effective(pairs: &[(String, String)]) -> Option<Endpoint>`
  - `pub fn keyscan_target(ep: &Endpoint) -> Vec<String>`
  - `pub enum HostKeyStatus { Trusted, New { fingerprint: String, key_line: String }, Mismatch { fingerprint: String }, Unavailable { message: String } }`（Serialize + ts-rs）
  - `pub fn compare_host_keys(scanned: &str, known_hosts_text: &str, ep: &Endpoint) -> HostKeyStatus`

- [ ] **Step 1: 寫失敗的測試**

在 `src-tauri/src/deploy.rs` 的 `mod tests` 內追加：

```rust
    // ── endpoint 解析 ─────────────────────────────────────────────────────────

    fn pairs(kv: &[(&str, &str)]) -> Vec<(String, String)> {
        kv.iter().map(|(k, v)| (k.to_string(), v.to_string())).collect()
    }

    #[test]
    fn endpoint_reads_hostname_and_port_from_ssh_dash_g() {
        let p = pairs(&[("user", "frank"), ("hostname", "10.0.0.9"), ("port", "2222")]);
        let ep = endpoint_from_effective(&p).expect("endpoint");
        assert_eq!(ep.hostname, "10.0.0.9");
        assert_eq!(ep.port, "2222");
    }

    #[test]
    fn endpoint_defaults_port_to_22_when_absent() {
        let p = pairs(&[("hostname", "example.com")]);
        let ep = endpoint_from_effective(&p).expect("endpoint");
        assert_eq!(ep.port, "22");
    }

    #[test]
    fn endpoint_is_none_without_hostname() {
        assert!(endpoint_from_effective(&pairs(&[("user", "frank")])).is_none());
    }

    #[test]
    fn keyscan_target_passes_port_and_host_as_separate_argv() {
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "2222".into() };
        assert_eq!(
            keyscan_target(&ep),
            vec!["-T", "5", "-p", "2222", "10.0.0.9"]
                .into_iter()
                .map(String::from)
                .collect::<Vec<_>>()
        );
    }

    // ── host key 比對 ─────────────────────────────────────────────────────────

    const SCANNED: &str =
        "# 10.0.0.9:22 SSH-2.0-OpenSSH_9.6\n10.0.0.9 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAAAA\n";

    #[test]
    fn host_key_trusted_when_known_hosts_matches() {
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "22".into() };
        let known = "10.0.0.9 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAAAA\n";
        assert_eq!(compare_host_keys(SCANNED, known, &ep), HostKeyStatus::Trusted);
    }

    #[test]
    fn host_key_new_when_absent_from_known_hosts() {
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "22".into() };
        match compare_host_keys(SCANNED, "", &ep) {
            HostKeyStatus::New { fingerprint, key_line } => {
                assert!(fingerprint.starts_with("SHA256:"), "got {fingerprint}");
                assert!(key_line.contains("ssh-ed25519"));
            }
            other => panic!("expected New, got {other:?}"),
        }
    }

    #[test]
    fn host_key_mismatch_is_reported_not_silently_trusted() {
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "22".into() };
        let known = "10.0.0.9 ssh-ed25519 AAAADIFFERENTKEYMATERIALZZZ\n";
        assert!(
            matches!(compare_host_keys(SCANNED, known, &ep), HostKeyStatus::Mismatch { .. }),
            "a changed host key must never be auto-trusted"
        );
    }

    #[test]
    fn host_key_unavailable_when_keyscan_returned_nothing() {
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "22".into() };
        assert!(matches!(
            compare_host_keys("", "", &ep),
            HostKeyStatus::Unavailable { .. }
        ));
    }

    #[test]
    fn host_key_matches_bracketed_nonstandard_port_entries() {
        // 非 22 埠在 known_hosts 裡寫成 [host]:port。
        let ep = Endpoint { hostname: "10.0.0.9".into(), port: "2222".into() };
        let scanned = "[10.0.0.9]:2222 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAAAA\n";
        let known = "[10.0.0.9]:2222 ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIAAAA\n";
        assert_eq!(compare_host_keys(scanned, known, &ep), HostKeyStatus::Trusted);
    }
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test deploy:: 2>&1 | head -20`
Expected: FAIL —— `cannot find type 'Endpoint' in this scope`

- [ ] **Step 3: 實作**

先把 `src-tauri/src/keys.rs` 第 54 行的 `fn ssh_dir()` 改成 `pub fn ssh_dir()`（Task 6 需要）。

在 `deploy.rs` 的 `first_line_or` 之後、`#[cfg(test)]` 之前插入：

```rust
/// `ssh -G` 解析出的實際連線目標。
#[derive(Debug, Clone, PartialEq)]
pub struct Endpoint {
    pub hostname: String,
    pub port: String,
}

/// 從 `ssh -G` 的 key/value 對取出 hostname 與 port。沒有 hostname 就回 None。
pub fn endpoint_from_effective(pairs: &[(String, String)]) -> Option<Endpoint> {
    let find = |key: &str| {
        pairs
            .iter()
            .find(|(k, _)| k.eq_ignore_ascii_case(key))
            .map(|(_, v)| v.clone())
    };
    let hostname = find("hostname").filter(|h| !h.is_empty())?;
    Some(Endpoint {
        hostname,
        port: find("port").filter(|p| !p.is_empty()).unwrap_or_else(|| "22".to_string()),
    })
}

/// `ssh-keyscan` 的 argv（不含程式名）。純函式，方便測試。
pub fn keyscan_target(ep: &Endpoint) -> Vec<String> {
    vec![
        "-T".to_string(),
        "5".to_string(),
        "-p".to_string(),
        ep.port.clone(),
        ep.hostname.clone(),
    ]
}

/// host key 的三種狀態（外加「掃不到」）。
#[derive(Debug, Clone, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "camelCase")]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub enum HostKeyStatus {
    /// known_hosts 已有且相符，可直接部署。
    Trusted,
    /// known_hosts 沒有這台；前端顯示指紋讓使用者確認後寫入。
    New { fingerprint: String, key_line: String },
    /// known_hosts 有但金鑰不同 —— 可能是中間人，一律中止。
    Mismatch { fingerprint: String },
    /// `ssh-keyscan` 掃不到（非標準網路路徑、ProxyJump 後方等）。
    Unavailable { message: String },
}

/// known_hosts 的 host 欄位寫法：22 埠是裸主機名，其他埠是 `[host]:port`。
fn known_hosts_host_field(ep: &Endpoint) -> String {
    if ep.port == "22" {
        ep.hostname.clone()
    } else {
        format!("[{}]:{}", ep.hostname, ep.port)
    }
}

/// 從一行 `<host> <type> <base64>` 取出 (type, base64)。註解行與空行回 None。
fn split_key_line(line: &str) -> Option<(&str, &str)> {
    let line = line.trim();
    if line.is_empty() || line.starts_with('#') {
        return None;
    }
    let mut parts = line.split_whitespace();
    let _host = parts.next()?;
    let key_type = parts.next()?;
    let material = parts.next()?;
    Some((key_type, material))
}

/// 比對 `ssh-keyscan` 的輸出與 known_hosts 內容。
///
/// 只比對同一台 host 欄位下的 (type, base64)。掃到多把金鑰（ed25519 + rsa）時，
/// 只要 known_hosts 裡有任何一把相符即視為已信任 —— 這與 ssh 自己的行為一致。
pub fn compare_host_keys(
    scanned: &str,
    known_hosts_text: &str,
    ep: &Endpoint,
) -> HostKeyStatus {
    let host_field = known_hosts_host_field(ep);

    let scanned_keys: Vec<(&str, &str)> = scanned.lines().filter_map(split_key_line).collect();
    if scanned_keys.is_empty() {
        return HostKeyStatus::Unavailable {
            message: format!("ssh-keyscan returned no host key for {host_field}"),
        };
    }

    let known_keys: Vec<(&str, &str)> = known_hosts_text
        .lines()
        .filter(|l| {
            l.split_whitespace()
                .next()
                .is_some_and(|h| h.split(',').any(|p| p == host_field))
        })
        .filter_map(split_key_line)
        .collect();

    // 指紋一律取掃到的第一把，作為要顯示給使用者的代表。
    let (first_type, first_material) = scanned_keys[0];
    let fingerprint = crate::known_hosts::fingerprint_sha256(first_material)
        .unwrap_or_else(|| "SHA256:<unreadable>".to_string());

    if known_keys.is_empty() {
        return HostKeyStatus::New {
            fingerprint,
            key_line: format!("{host_field} {first_type} {first_material}"),
        };
    }
    if scanned_keys.iter().any(|s| known_keys.contains(s)) {
        return HostKeyStatus::Trusted;
    }
    HostKeyStatus::Mismatch { fingerprint }
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test deploy:: -- --nocapture`
Expected: 20 passed

若 `fingerprint_sha256` 對測試中的假 base64 回 `None`，測試的 `starts_with("SHA256:")` 會失敗 —— 此時把 `SCANNED` 常數換成一段真實的 ed25519 公鑰 base64（可用 `ssh-keyscan github.com` 取得）。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/deploy.rs src-tauri/src/keys.rs src/bindings/HostKeyStatus.ts
git commit -m "feat(deploy): add host key precheck against known_hosts"
```

---

### Task 6: 執行部署 + Tauri commands

**Files:**
- Modify: `src-tauri/src/deploy.rs`（追加執行層與 commands）
- Modify: `src-tauri/src/lib.rs:84-122`（註冊 commands）

**Interfaces:**
- Consumes: Task 1 的 `secrets::*`、Task 4 的 `build_deploy_argv` / `classify_outcome`、Task 5 的 `endpoint_from_effective` / `keyscan_target` / `compare_host_keys`
- Produces: 6 個 Tauri commands（見下）

- [ ] **Step 1: 實作執行層**

在 `deploy.rs` 的 `compare_host_keys` 之後、`#[cfg(test)]` 之前插入：

```rust
// ─── 執行層（有副作用，不做單元測試；以 Task 12 的手動驗證涵蓋） ─────────────

use std::io::Write;
use std::process::{Command, Stdio};

/// 這個 alias 是否經過跳板。`ssh -G` 對 ProxyJump 會輸出 `proxyjump`，
/// ProxyJump 也會被展開成 `proxycommand`；兩者都要擋。`none` 是明確停用的寫法。
pub fn has_proxy(pairs: &[(String, String)]) -> bool {
    pairs.iter().any(|(k, v)| {
        (k.eq_ignore_ascii_case("proxyjump") || k.eq_ignore_ascii_case("proxycommand"))
            && !v.trim().is_empty()
            && !v.trim().eq_ignore_ascii_case("none")
    })
}

/// 取得 alias 的 `ssh -G` key/value 對（驗證 alias 後）。
fn effective_pairs(
    state: &tauri::State<crate::state::AppState>,
    alias: &str,
) -> Result<Vec<(String, String)>, AppError> {
    let main_path = {
        let doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock
            .as_ref()
            .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        crate::connect::validate_alias(doc, alias)?;
        doc.files.first().map(|f| f.path.clone())
    };
    crate::config::intel::effective_config(alias, main_path.as_deref())
}

/// 解析 alias 的實際連線目標（走既有的 `ssh -G` 整合）。
fn resolve_endpoint(
    state: &tauri::State<crate::state::AppState>,
    alias: &str,
) -> Result<Endpoint, AppError> {
    let main_path = {
        let doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock
            .as_ref()
            .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        crate::connect::validate_alias(doc, alias)?;
        doc.files.first().map(|f| f.path.clone())
    };
    let pairs = crate::config::intel::effective_config(alias, main_path.as_deref())?;
    endpoint_from_effective(&pairs)
        .ok_or_else(|| AppError::NotFound(format!("ssh -G returned no hostname for '{alias}'")))
}

/// 執行部署本體。回傳 (outcome)。密碼已事先放進 `account` 指向的 keychain 項目。
fn run_ssh_deploy(
    alias: &str,
    pub_material: &str,
    account: &str,
    env_secret: Option<&str>,
) -> Result<DeployOutcome, AppError> {
    let exe = std::env::current_exe().map_err(AppError::Io)?;

    let mut cmd = Command::new("ssh");
    cmd.args(build_deploy_argv(alias))
        .env("SSH_ASKPASS", &exe)
        .env("SSH_ASKPASS_REQUIRE", "force")
        .env("SSHELTER_ASKPASS", "1")
        .env("SSHELTER_ASKPASS_ACCOUNT", account)
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .stderr(Stdio::piped());
    if let Some(secret) = env_secret {
        // 本機無密鑰環時的 fallback；UI 已告知使用者密碼不會被儲存。
        cmd.env("SSHELTER_ASKPASS_SECRET", secret);
    }

    let mut child = cmd.spawn().map_err(|e| {
        if e.kind() == std::io::ErrorKind::NotFound {
            AppError::NotFound("ssh not found".to_string())
        } else {
            AppError::Io(e)
        }
    })?;

    {
        // 公鑰走 stdin。handle 必須在這個 scope 結束時 drop，遠端的 `cat` 才會看到 EOF。
        let mut stdin = child
            .stdin
            .take()
            .ok_or_else(|| AppError::Other("failed to open ssh stdin".to_string()))?;
        stdin.write_all(pub_material.as_bytes()).map_err(AppError::Io)?;
    }

    let output = child.wait_with_output().map_err(AppError::Io)?;
    Ok(classify_outcome(
        output.status.code(),
        &String::from_utf8_lossy(&output.stdout),
        &String::from_utf8_lossy(&output.stderr),
    ))
}

// ─── Tauri commands ──────────────────────────────────────────────────────────

#[tauri::command]
pub fn deploy_precheck_host_key(
    state: tauri::State<crate::state::AppState>,
    alias: String,
) -> Result<HostKeyStatus, AppError> {
    let ep = resolve_endpoint(&state, &alias)?;

    let scanned = match Command::new("ssh-keyscan").args(keyscan_target(&ep)).output() {
        Ok(o) => String::from_utf8_lossy(&o.stdout).into_owned(),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => {
            return Ok(HostKeyStatus::Unavailable {
                message: "ssh-keyscan not found".to_string(),
            });
        }
        Err(e) => return Err(AppError::Io(e)),
    };

    let known = crate::known_hosts::read_known_hosts_text()?;
    Ok(compare_host_keys(&scanned, &known, &ep))
}

#[tauri::command]
pub fn deploy_trust_host_key(
    state: tauri::State<crate::state::AppState>,
    alias: String,
    key_line: String,
) -> Result<(), AppError> {
    // key_line 只接受 precheck 回傳的形狀，重新驗一次避免前端傳入任意內容。
    let ep = resolve_endpoint(&state, &alias)?;
    let expected_host = known_hosts_host_field(&ep);
    let ok = key_line
        .split_whitespace()
        .next()
        .is_some_and(|h| h == expected_host)
        && split_key_line(&key_line).is_some()
        && !key_line.contains('\n');
    if !ok {
        return Err(AppError::ForbiddenPath(format!(
            "refusing to append malformed known_hosts line for '{alias}'"
        )));
    }
    crate::known_hosts::append_known_hosts_line(&key_line)
}

#[tauri::command]
pub fn deploy_key(
    state: tauri::State<crate::state::AppState>,
    alias: String,
    public_path: String,
    password: String,
    remember: bool,
) -> Result<DeployOutcome, AppError> {
    {
        let doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock
            .as_ref()
            .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        crate::connect::validate_alias(doc, &alias)?;
    }
    // ProxyJump／ProxyCommand 必須「主動拒絕」，不能只是「不支援」。
    // `SSH_ASKPASS` 與 `SSHELTER_ASKPASS_*` 會被 ProxyCommand 子進程繼承，於是跳板
    // 主機的 `user@jump's password: ` 提示會被 helper 用「目標主機的密碼」回答 ——
    // 那是把密碼洩漏給另一台機器。白名單擋不住這個，因為那是一個完全合法的提示。
    {
        let ep_pairs = effective_pairs(&state, &alias)?;
        if has_proxy(&ep_pairs) {
            return Err(AppError::Other(format!(
                "'{alias}' goes through a jump host; in-app deploy is refused because the \
                 password would be offered to the jump host as well"
            )));
        }
    }

    let dir = crate::keys::ssh_dir()?;
    let pub_path = crate::keys::validate_public_path(&public_path, &dir)?;
    let pub_material = std::fs::read_to_string(&pub_path)?;

    let use_keychain = crate::secrets::available();
    let account = if remember {
        crate::secrets::host_account(&alias)
    } else {
        crate::secrets::tmp_account(&alias)
    };
    if use_keychain {
        crate::secrets::set(&account, &password)?;
    }

    let env_secret = if use_keychain { None } else { Some(password.as_str()) };
    let result = run_ssh_deploy(&alias, &pub_material, &account, env_secret);

    // 清理：暫存項目一律刪除，無論部署成敗。
    if use_keychain && !remember {
        let _ = crate::secrets::delete(&account);
    }
    result
}

#[tauri::command]
pub fn secrets_has(alias: String) -> Result<bool, AppError> {
    Ok(crate::secrets::get(&crate::secrets::host_account(&alias))?.is_some())
}

#[tauri::command]
pub fn secrets_get(alias: String) -> Result<Option<String>, AppError> {
    crate::secrets::get(&crate::secrets::host_account(&alias))
}

#[tauri::command]
pub fn secrets_set(alias: String, password: String) -> Result<(), AppError> {
    crate::secrets::set(&crate::secrets::host_account(&alias), &password)
}

#[tauri::command]
pub fn secrets_delete(alias: String) -> Result<(), AppError> {
    crate::secrets::delete(&crate::secrets::host_account(&alias))
}
```

同時在 `deploy.rs` 檔案頂端的 `use serde::{Deserialize, Serialize};` 之後加入：

```rust
use crate::error::AppError;
```

- [ ] **Step 2: 在 known_hosts.rs 補兩個共用函式**

`deploy.rs` 用到的 `read_known_hosts_text` 與 `append_known_hosts_line` 目前不存在。在 `src-tauri/src/known_hosts.rs` 的 `known_hosts_path()` 之後插入：

```rust
/// 讀取 known_hosts 全文；檔案不存在時回空字串（首次使用的正常狀態）。
pub fn read_known_hosts_text() -> Result<String, AppError> {
    let path = known_hosts_path()?;
    match std::fs::read_to_string(&path) {
        Ok(t) => Ok(t),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(String::new()),
        Err(e) => Err(e.into()),
    }
}

/// 追加一行到 known_hosts。呼叫端必須先驗證 `line` 的形狀（見 `deploy_trust_host_key`）。
/// 檔案不存在時以 0600 建立，並確保與前一行之間有換行。
pub fn append_known_hosts_line(line: &str) -> Result<(), AppError> {
    let path = known_hosts_path()?;
    let mut text = read_known_hosts_text()?;
    if !text.is_empty() && !text.ends_with('\n') {
        text.push('\n');
    }
    text.push_str(line.trim_end());
    text.push('\n');
    crate::fsutil::atomic_write(&path, text.as_bytes(), 0o600)
}
```

> 若 `fsutil::atomic_write` 的簽章與此不符，以該檔案的實際簽章為準調整（它已被 `known_hosts::remove_from_file` 使用，可照抄該處的呼叫方式）。

- [ ] **Step 3: 註冊 commands**

編輯 `src-tauri/src/lib.rs`：

在 import 區（第 14 行 `use connect::...` 之後）加入：

```rust
use deploy::{
    deploy_key, deploy_precheck_host_key, deploy_trust_host_key, secrets_delete, secrets_get,
    secrets_has, secrets_set,
};
```

在 `invoke_handler` 的 `keys_deploy,`（第 112 行）之後加入：

```rust
            deploy_precheck_host_key,
            deploy_trust_host_key,
            deploy_key,
            secrets_has,
            secrets_get,
            secrets_set,
            secrets_delete,
```

- [ ] **Step 4: 建置並跑全部測試**

Run: `cd src-tauri && cargo build && cargo test`
Expected: 建置成功；測試全綠，總數 ≥ 217 Rust 測試（原 197 + 本計畫新增）

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/deploy.rs src-tauri/src/known_hosts.rs src-tauri/src/lib.rs
git commit -m "feat(deploy): add deploy/precheck/secrets Tauri commands"
```

---

### Task 7: 前端 query hooks

**Files:**
- Modify: `src/lib/queries.ts`（追加）

**Interfaces:**
- Consumes: Task 6 的 7 個 commands、`src/bindings/DeployOutcome.ts`、`src/bindings/HostKeyStatus.ts`
- Produces:
  - `useHostPassword(alias, opts)` → `UseQueryResult<string | null>`
  - `useHasHostPassword(alias)` → `UseQueryResult<boolean>`
  - `useSetHostPassword()` / `useDeleteHostPassword()` mutations
  - `usePrecheckHostKey()` mutation → `HostKeyStatus`
  - `useTrustHostKey()` mutation
  - `useDeployKeyDirect()` mutation → `DeployOutcome`
  - `queryKeys.hostPassword(alias)`

- [ ] **Step 1: 加 query key**

在 `src/lib/queries.ts` 的 `queryKeys` 物件中加入一行（照該物件既有的寫法與縮排）：

```ts
  hostPassword: (alias: string) => ["hostPassword", alias] as const,
```

- [ ] **Step 2: 追加 hooks**

在 `useDeployKey` 之後插入：

```ts
/** keychain 是否已存這台主機的密碼（不取回密碼本身）。 */
export function useHasHostPassword(alias: string | null) {
  return useQuery<boolean>({
    queryKey: alias ? queryKeys.hostPassword(alias) : ["hostPassword", "none"],
    queryFn: () => tauriInvoke<boolean>("secrets_has", { alias }),
    enabled: !!alias,
  });
}

/**
 * 取回這台主機在 keychain 的密碼。刻意做成 mutation 而非 query——
 * 密碼只在使用者明確按下「顯示」時才讀出，不隨畫面渲染快取。
 */
export function useRevealHostPassword() {
  return useMutation<string | null, unknown, { alias: string }>({
    mutationFn: ({ alias }) =>
      tauriInvoke<string | null>("secrets_get", { alias }),
    onError: (e) =>
      toast.error("Failed to read password", { description: errMessage(e) }),
  });
}

export function useSetHostPassword() {
  const queryClient = useQueryClient();
  return useMutation<void, unknown, { alias: string; password: string }>({
    mutationFn: ({ alias, password }) =>
      tauriInvoke<void>("secrets_set", { alias, password }),
    onSuccess: (_d, { alias }) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.hostPassword(alias) });
    },
    onError: (e) =>
      toast.error("Failed to save password", { description: errMessage(e) }),
  });
}

export function useDeleteHostPassword() {
  const queryClient = useQueryClient();
  return useMutation<void, unknown, { alias: string }>({
    mutationFn: ({ alias }) => tauriInvoke<void>("secrets_delete", { alias }),
    onSuccess: (_d, { alias }) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.hostPassword(alias) });
    },
    onError: (e) =>
      toast.error("Failed to delete password", { description: errMessage(e) }),
  });
}

/** 部署前先驗 host key，讓後續 ssh 能用 StrictHostKeyChecking=yes。 */
export function usePrecheckHostKey() {
  return useMutation<HostKeyStatus, unknown, { alias: string }>({
    mutationFn: ({ alias }) =>
      tauriInvoke<HostKeyStatus>("deploy_precheck_host_key", { alias }),
    onError: (e) =>
      toast.error("Failed to check host key", { description: errMessage(e) }),
  });
}

/** 使用者確認指紋後，把主機金鑰寫入 known_hosts。 */
export function useTrustHostKey() {
  return useMutation<void, unknown, { alias: string; keyLine: string }>({
    mutationFn: ({ alias, keyLine }) =>
      tauriInvoke<void>("deploy_trust_host_key", { alias, keyLine }),
    onError: (e) =>
      toast.error("Failed to trust host key", { description: errMessage(e) }),
  });
}

/** 在 app 內直接部署公鑰（不開終端機）。 */
export function useDeployKeyDirect() {
  return useMutation<
    DeployOutcome,
    unknown,
    { alias: string; publicPath: string; password: string; remember: boolean }
  >({
    mutationFn: ({ alias, publicPath, password, remember }) =>
      tauriInvoke<DeployOutcome>("deploy_key", {
        alias,
        publicPath,
        password,
        remember,
      }),
    onError: (e) =>
      toast.error("Deploy failed", { description: errMessage(e) }),
  });
}
```

在檔案頂端的 import 區加入：

```ts
import type { DeployOutcome } from "@/bindings/DeployOutcome";
import type { HostKeyStatus } from "@/bindings/HostKeyStatus";
```

- [ ] **Step 3: 型別檢查**

Run: `pnpm build`
Expected: tsc 與 vite build 皆無誤

- [ ] **Step 4: Commit**

```bash
git add src/lib/queries.ts
git commit -m "feat(queries): add deploy and host-password hooks"
```

---

### Task 8: `DeployKeyDialog` 元件

**Files:**
- Create: `src/lib/deploy-key-select.ts`
- Create: `src/lib/deploy-key-select.test.ts`
- Create: `src/components/DeployKeyDialog.tsx`
- Modify: `src/stores/ui.ts`（加入開啟狀態）

**Interfaces:**
- Consumes: Task 7 的所有 hooks、既有的 `useKeys`、`useHostsQuery`
- Produces:
  - `pickDefaultPublicKey(identityFiles: string[], keys: KeyInfo[]): string | null`
  - `<DeployKeyDialog />`（由 ui store 的 `deployKeyAlias` 驅動開啟）
  - ui store：`deployKeyAlias: string | null`、`setDeployKeyAlias(alias: string | null)`

- [ ] **Step 1: 寫失敗的測試（公鑰預選邏輯）**

建立 `src/lib/deploy-key-select.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import type { KeyInfo } from "@/bindings/KeyInfo";
import { pickDefaultPublicKey } from "./deploy-key-select";

function key(name: string, pub: string | null): KeyInfo {
  return {
    name,
    private_path: `/home/f/.ssh/${name}`,
    public_path: pub,
    key_type: "ED25519",
    bits: 256,
    fingerprint_sha256: "SHA256:x",
    comment: null,
    in_agent: false,
  };
}

describe("pickDefaultPublicKey", () => {
  it("prefers the host's IdentityFile when it has a sibling .pub", () => {
    const keys = [
      key("id_ed25519", "/home/f/.ssh/id_ed25519.pub"),
      key("work", "/home/f/.ssh/work.pub"),
    ];
    expect(pickDefaultPublicKey(["/home/f/.ssh/work"], keys)).toBe(
      "/home/f/.ssh/work.pub",
    );
  });

  it("falls back to the only key when the host has no IdentityFile", () => {
    const keys = [key("id_ed25519", "/home/f/.ssh/id_ed25519.pub")];
    expect(pickDefaultPublicKey([], keys)).toBe("/home/f/.ssh/id_ed25519.pub");
  });

  it("returns null when several keys exist and none is indicated", () => {
    const keys = [
      key("a", "/home/f/.ssh/a.pub"),
      key("b", "/home/f/.ssh/b.pub"),
    ];
    expect(pickDefaultPublicKey([], keys)).toBeNull();
  });

  it("ignores keys that have no .pub — they cannot be deployed", () => {
    const keys = [key("a", null), key("b", "/home/f/.ssh/b.pub")];
    expect(pickDefaultPublicKey([], keys)).toBe("/home/f/.ssh/b.pub");
  });

  it("ignores an IdentityFile whose key has no .pub and falls back", () => {
    const keys = [key("a", null), key("b", "/home/f/.ssh/b.pub")];
    expect(pickDefaultPublicKey(["/home/f/.ssh/a"], keys)).toBe(
      "/home/f/.ssh/b.pub",
    );
  });

  it("returns null when there are no keys at all", () => {
    expect(pickDefaultPublicKey([], [])).toBeNull();
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm test -- deploy-key-select`
Expected: FAIL —— 找不到模組 `./deploy-key-select`

- [ ] **Step 3: 實作純函式**

建立 `src/lib/deploy-key-select.ts`：

```ts
import type { KeyInfo } from "@/bindings/KeyInfo";

/**
 * 決定部署對話框預設要選哪一把公鑰。
 *
 * 優先順序：該 host 的 IdentityFile 對應的 `.pub` → `~/.ssh` 裡唯一那把可部署的 key
 * → null（讓使用者自己選）。沒有 `.pub` 的 key 無法部署，一律排除。
 */
export function pickDefaultPublicKey(
  identityFiles: string[],
  keys: KeyInfo[],
): string | null {
  const deployable = keys.filter((k) => k.public_path !== null);

  for (const identity of identityFiles) {
    const match = deployable.find((k) => k.private_path === identity);
    if (match) return match.public_path;
  }
  if (deployable.length === 1) return deployable[0].public_path;
  return null;
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `pnpm test -- deploy-key-select`
Expected: 6 passed

- [ ] **Step 5: 加 ui store 狀態**

在 `src/stores/ui.ts` 中，照 `addHostTargetFile` 的既有寫法加入（session-only，不持久化）：

```ts
  /** 開啟部署金鑰對話框的目標 host（null = 關閉）。 */
  deployKeyAlias: string | null;
  setDeployKeyAlias: (alias: string | null) => void;
```

以及對應的 setter 實作：

```ts
  deployKeyAlias: null,
  setDeployKeyAlias: (alias) => set({ deployKeyAlias: alias }),
```

- [ ] **Step 6: 實作 DeployKeyDialog**

建立 `src/components/DeployKeyDialog.tsx`。元件由 ui store 的 `deployKeyAlias` 驅動，內部三個階段：`form`（填公鑰＋密碼）→ `hostkey`（僅在 precheck 回 `New` 時出現，顯示指紋要求確認）→ `result`（顯示 `DeployOutcome`）。

要點：

- 進入 `form` 階段時呼叫 `useHasHostPassword(alias)`；為 true 時在密碼欄下方顯示「已存密碼」與一個「載入」按鈕（按下才 `useRevealHostPassword`），密碼欄用 `type={revealed ? "text" : "password"}` 搭配眼睛按鈕切換。
- 「記住這台主機的密碼」用既有的 `Checkbox`（若 `src/components/ui/` 尚無 checkbox，照 `context-menu.tsx` 的方式從 radix 包一個最小版本）。
- 按下「Deploy」時的順序：`usePrecheckHostKey` → 依 `status.kind` 分派：
  - `trusted` → 直接 `useDeployKeyDirect`
  - `new` → 切到 `hostkey` 階段，顯示 `status.fingerprint`；使用者按「Trust & continue」後 `useTrustHostKey({ alias, keyLine: status.key_line })` 再 `useDeployKeyDirect`
  - `mismatch` → 切到 `result` 階段顯示紅色警告，**不提供任何繼續的按鈕**
  - `unavailable` → 切到 `result` 階段，顯示 `status.message` 與「請改用終端機部署」的說明
- `DeployOutcome` 的文案對照：

```ts
const OUTCOME_TEXT: Record<DeployOutcome["kind"], { title: string; tone: "ok" | "warn" | "err" }> = {
  added: { title: "Key deployed", tone: "ok" },
  alreadyPresent: { title: "Key was already there — nothing added", tone: "ok" },
  wrongPassword: { title: "Wrong password", tone: "err" },
  hostKeyFailed: { title: "Host key verification failed", tone: "err" },
  unreachable: { title: "Could not reach the host", tone: "err" },
  remoteError: { title: "The remote command failed", tone: "err" },
  other: { title: "Deploy failed", tone: "err" },
};
```

- 對話框底部固定顯示一行說明：`ProxyJump hosts that also need a password are not supported.`

在 `src/App.tsx`（或目前掛載其他 store 驅動 dialog 的位置，照 `AddHostDialog` 的掛法）掛上 `<DeployKeyDialog />`。

- [ ] **Step 7: 型別檢查與測試**

Run: `pnpm build && pnpm test`
Expected: 皆通過，vitest 總數 ≥ 92

- [ ] **Step 8: Commit**

```bash
git add src/lib/deploy-key-select.ts src/lib/deploy-key-select.test.ts \
        src/components/DeployKeyDialog.tsx src/stores/ui.ts src/App.tsx
git commit -m "feat(deploy): add in-app key deployment dialog"
```

---

### Task 9: host row 右鍵選單

**Files:**
- Modify: `src/components/HostList.tsx:90-110`（`HostRowProps`）、`136-144`（`<li>` 外層）、`689-700`（呼叫處）

**Interfaces:**
- Consumes: Task 8 的 `setDeployKeyAlias`、Task 7 的 `useHasHostPassword`
- Produces: host row 的右鍵選單（三項）

- [ ] **Step 1: 擴充 HostRowProps**

在 `src/components/HostList.tsx` 的 `HostRowProps`（第 90 行起）加入：

```ts
  /** 右鍵「Deploy key…」。省略時不掛右鍵選單（defaults 列）。 */
  onDeployKey?: () => void;
```

並在 `function HostRow({ ... })` 的解構參數中加入 `onDeployKey,`。

> **刻意不加「Copy password」到這個選單。** 那需要對每一列都查一次 keychain 才知道要不要顯示，在大型 config 上會很慢；複製密碼的入口改由 Task 10 的 HostEditor 提供。

- [ ] **Step 2: 包上 ContextMenu**

`HostList.tsx` 已 import `ContextMenu` 系列（第 33–37 行）。把 `HostRow` 回傳的整個 `<li>`（第 136 行起）包起來 —— 只在 `onDeployKey` 存在時包，否則原樣回傳：

```tsx
  const row = (
    <li className="animate-row-enter group/row relative" /* …既有屬性原封不動… */>
      {/* …既有內容原封不動… */}
    </li>
  );

  if (!onDeployKey) return row;

  return (
    <ContextMenu>
      <ContextMenuTrigger asChild>{row}</ContextMenuTrigger>
      <ContextMenuContent>
        {onConnect && (
          <ContextMenuItem onSelect={onConnect}>
            <Play className="size-3.5" />
            Connect
          </ContextMenuItem>
        )}
        <ContextMenuItem onSelect={onDeployKey}>
          <Upload className="size-3.5" />
          Deploy key…
        </ContextMenuItem>
      </ContextMenuContent>
    </ContextMenu>
  );
```

在檔案頂端的 `lucide-react` import 加入 `Upload`。

- [ ] **Step 3: 接上呼叫處**

在 `HostList.tsx` 第 689 行附近 `variant="host"` 的 `<HostRow>` 加上：

```tsx
                              onDeployKey={() => setDeployKeyAlias(host.alias)}
```

`variant="defaults"` 的那個（第 668 行附近）**不加** —— wildcard 區塊不是真實主機，與既有 `onConnect` 的處理一致。

同時在 `HostList` 元件本體取得 setter：

```tsx
const setDeployKeyAlias = useUiStore((s) => s.setDeployKeyAlias);
```

（照該檔案既有取用 ui store 的寫法，例如 `setAddHostOpen` 那一行。）

- [ ] **Step 4: 型別檢查、測試、實機驗證**

Run: `pnpm build && pnpm test`
Expected: 皆通過

Run: `pnpm tauri dev`，右鍵任一 host 列 → 出現 Connect / Deploy key… 兩項；右鍵 `Host *` 的 defaults 列 → **不出現**選單。

- [ ] **Step 5: Commit**

```bash
git add src/components/HostList.tsx
git commit -m "feat(host-list): right-click a host to deploy a key"
```

---

### Task 10: HostEditor 的 Password 區塊

**Files:**
- Modify: `src/components/HostEditor.tsx`（在 `HostActions` 附近新增區塊）

**Interfaces:**
- Consumes: Task 7 的 `useHasHostPassword` / `useRevealHostPassword` / `useSetHostPassword` / `useDeleteHostPassword`
- Produces: `<HostPasswordSection alias={alias} />`

- [ ] **Step 1: 實作區塊**

在 `src/components/HostEditor.tsx` 新增一個元件，並在 `HostEditorForm` 內、`ConfigInspector` 之前渲染：

```tsx
/**
 * 這台主機的密碼（存在作業系統 keychain）。
 *
 * 注意：這「不是」ssh config 的欄位 —— 它不會寫進任何 config 檔，也不參與表單的
 * 存檔流程。ssh_config 沒有密碼指令，寫進去會讓 ssh 對「所有」host 都罷工。
 */
function HostPasswordSection({ alias }: { alias: string }) {
  const [draft, setDraft] = useState("");
  const [revealed, setRevealed] = useState(false);
  const has = useHasHostPassword(alias);
  const reveal = useRevealHostPassword();
  const save = useSetHostPassword();
  const remove = useDeleteHostPassword();

  // 切換主機時清掉草稿，避免把 A 的密碼存到 B。
  useEffect(() => {
    setDraft("");
    setRevealed(false);
  }, [alias]);

  return (
    <div className="space-y-2 border-t pt-3">
      <span className="section-label px-0">Password</span>
      {/* …輸入框 + 顯示/複製/儲存/刪除按鈕… */}
      <p className="text-xs text-muted-foreground">
        Stored in your operating system’s keychain — never written to{" "}
        <span className="font-mono">~/.ssh/config</span>.
      </p>
    </div>
  );
}
```

行為要點：

- `has.data` 為 true 時顯示「Saved」徽章與「Show」「Copy」「Delete」三個按鈕；「Show」按下才呼叫 `reveal.mutate({ alias })` 並把結果填進輸入框、`setRevealed(true)`。
- 「Copy」呼叫 `reveal.mutate` 後 `navigator.clipboard.writeText(...)`，照 `KeysDialog.copyPublicKey` 的既有寫法（含 clipboard 失敗的 toast）。
- 輸入框 `type={revealed ? "text" : "password"}`。
- 「Save」在 `draft.trim()` 非空時啟用，呼叫 `save.mutate({ alias, password: draft })`，成功後 toast 並 `setDraft("")`。
- 「Delete」呼叫 `remove.mutate({ alias })`，成功後 toast 並清空輸入框與 `revealed`。

在檔案頂端 import 上述四個 hooks。

- [ ] **Step 2: 型別檢查與測試**

Run: `pnpm build && pnpm test`
Expected: 皆通過

- [ ] **Step 3: 實機驗證**

Run: `pnpm tauri dev` → 選一台 host → Password 區塊 → 輸入密碼 → Save → 切到別台再切回來 → 顯示「Saved」→ Show 讀回相同密碼 → Delete 後徽章消失。

- [ ] **Step 4: Commit**

```bash
git add src/components/HostEditor.tsx
git commit -m "feat(host-editor): manage the host password stored in the OS keychain"
```

---

### Task 11: 環境前置條件檢查

**Files:**
- Modify: `src-tauri/src/deploy.rs`（追加）
- Modify: `src-tauri/src/lib.rs`（註冊 command）
- Modify: `src/components/DeployKeyDialog.tsx`（顯示警告）

**Interfaces:**
- Consumes: `crate::config::intel::effective_config`
- Produces:
  - `pub fn openssh_supports_askpass_require(version_line: &str) -> bool`
  - `pub fn password_auth_is_blocked(pairs: &[(String, String)]) -> bool`
  - `pub struct DeployPreflight { pub askpass_supported: bool, pub password_auth_blocked: bool }`（Serialize + ts-rs）
  - command `deploy_preflight(alias) -> DeployPreflight`

- [ ] **Step 1: 寫失敗的測試**

在 `deploy.rs` 的 `mod tests` 內追加：

```rust
    #[test]
    fn openssh_version_gate_requires_8_4() {
        // SSH_ASKPASS_REQUIRE 是 OpenSSH 8.4 引入的，但 kbdint 的 `(user@host) ` 前綴要到
        // 8.5 才有。白名單已不接受裸的 `Password: `，所以閘門必須是 8.5 而非 8.4。
        assert!(openssh_supports_askpass_require("OpenSSH_10.2p1, LibreSSL 3.3.6"));
        assert!(openssh_supports_askpass_require("OpenSSH_9.6p1, LibreSSL 3.3.6"));
        assert!(openssh_supports_askpass_require("OpenSSH_8.5p1, OpenSSL 1.1.1"));
        assert!(!openssh_supports_askpass_require("OpenSSH_8.4p1, OpenSSL 1.1.1"));
        assert!(!openssh_supports_askpass_require("OpenSSH_8.3p1, OpenSSL 1.1.1"));
        assert!(!openssh_supports_askpass_require("OpenSSH_7.9p1, LibreSSL 2.7.3"));
        // 認不出來時保守放行，讓實際部署去回報真正的錯誤。
        assert!(openssh_supports_askpass_require("something unparseable"));
    }

    #[test]
    fn detects_jump_hosts_so_deploy_can_refuse_them() {
        // 這不是「不支援」而是「必須拒絕」：askpass 的環境變數會被 ProxyCommand 子進程
        // 繼承，跳板的密碼提示是完全合法的形狀，白名單擋不住 —— 結果是把目標主機的密碼
        // 送給跳板主機。
        assert!(has_proxy(&pairs(&[("proxyjump", "bastion")])));
        assert!(has_proxy(&pairs(&[("proxycommand", "ssh -W %h:%p bastion")])));
        // `none` 是明確停用，不算跳板。
        assert!(!has_proxy(&pairs(&[("proxyjump", "none")])));
        assert!(!has_proxy(&pairs(&[("proxycommand", "none")])));
        assert!(!has_proxy(&pairs(&[("proxyjump", "")])));
        assert!(!has_proxy(&pairs(&[("hostname", "10.0.0.9")])));
    }

    #[test]
    fn detects_config_that_blocks_password_auth() {
        // 使用者若全域設了 PreferredAuthentications publickey，密碼永遠用不到，
        // 部署會以 "Permission denied" 失敗 —— 那個訊息會被誤讀成「密碼錯」。
        let blocked = pairs(&[("preferredauthentications", "publickey")]);
        assert!(password_auth_is_blocked(&blocked));

        let ok = pairs(&[("preferredauthentications", "publickey,password")]);
        assert!(!password_auth_is_blocked(&ok));

        let ki = pairs(&[("preferredauthentications", "keyboard-interactive")]);
        assert!(!password_auth_is_blocked(&ki));

        // 沒設就是 ssh 的預設值，包含 password。
        assert!(!password_auth_is_blocked(&pairs(&[("user", "frank")])));
    }
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test deploy::tests::openssh 2>&1 | head -10`
Expected: FAIL —— `cannot find function 'openssh_supports_askpass_require'`

- [ ] **Step 3: 實作**

在 `deploy.rs` 的 commands 區之前插入：

```rust
/// 閘門是 OpenSSH **8.5**：`SSH_ASKPASS_REQUIRE` 雖然 8.4 就有，但 kbdint 提示的
/// `(user@host) ` 前綴要到 8.5 才加入，而白名單已不接受裸的 `Password: `。
/// 認不出版本時回 true（保守放行，讓真正的部署去回報實際錯誤）。
pub fn openssh_supports_askpass_require(version_line: &str) -> bool {
    let Some(rest) = version_line.split("OpenSSH_").nth(1) else {
        return true;
    };
    let digits: String = rest
        .chars()
        .take_while(|c| c.is_ascii_digit() || *c == '.')
        .collect();
    let mut parts = digits.split('.');
    let (Some(major), Some(minor)) = (parts.next(), parts.next()) else {
        return true;
    };
    match (major.parse::<u32>(), minor.parse::<u32>()) {
        (Ok(major), Ok(minor)) => major > 8 || (major == 8 && minor >= 5),
        _ => true,
    }
}

/// 使用者的設定是否讓密碼認證根本用不上。
pub fn password_auth_is_blocked(pairs: &[(String, String)]) -> bool {
    let Some((_, value)) = pairs
        .iter()
        .find(|(k, _)| k.eq_ignore_ascii_case("preferredauthentications"))
    else {
        return false; // 未設定 → ssh 預設含 password
    };
    !value
        .split(',')
        .any(|m| matches!(m.trim(), "password" | "keyboard-interactive"))
}

/// 部署前的環境檢查結果。
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct DeployPreflight {
    /// 本機 ssh 是否夠新（>= 8.5）可自動填密碼。
    pub askpass_supported: bool,
    /// 這台 host 的設定是否封鎖了密碼認證。
    pub password_auth_blocked: bool,
    /// 本機是否有可用的密鑰環。false 時密碼只能經環境變數傳給 helper，且無法「記住」。
    pub keychain_available: bool,
}

#[tauri::command]
pub fn deploy_preflight(
    state: tauri::State<crate::state::AppState>,
    alias: String,
) -> Result<DeployPreflight, AppError> {
    let version = Command::new("ssh")
        .arg("-V")
        .output()
        .map(|o| {
            // ssh -V 寫到 stderr。
            let mut s = String::from_utf8_lossy(&o.stderr).into_owned();
            s.push_str(&String::from_utf8_lossy(&o.stdout));
            s
        })
        .unwrap_or_default();

    let main_path = {
        let doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock
            .as_ref()
            .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        crate::connect::validate_alias(doc, &alias)?;
        doc.files.first().map(|f| f.path.clone())
    };
    let pairs =
        crate::config::intel::effective_config(&alias, main_path.as_deref()).unwrap_or_default();

    Ok(DeployPreflight {
        askpass_supported: openssh_supports_askpass_require(&version),
        password_auth_blocked: password_auth_is_blocked(&pairs),
        keychain_available: crate::secrets::available(),
    })
}
```

在 `lib.rs` 的 `use deploy::{...}` 加入 `deploy_preflight`，並在 `invoke_handler` 的 `deploy_key,` 之後加入 `deploy_preflight,`。

- [ ] **Step 4: 執行測試**

Run: `cd src-tauri && cargo test deploy:: -- --nocapture`
Expected: 22 passed

- [ ] **Step 5: 前端顯示警告**

在 `DeployKeyDialog.tsx` 的 `form` 階段開啟時呼叫 `deploy_preflight`（新增 `useDeployPreflight()` mutation 到 `queries.ts`，寫法照 `usePrecheckHostKey`），並：

- `askpassSupported` 為 false → 在對話框頂端顯示紅色說明「This machine’s OpenSSH is older than 8.5 and cannot auto-fill the password. Use the terminal-based deploy instead.」並停用 Deploy 按鈕。
- `passwordAuthBlocked` 為 true → 顯示黃色說明「This host’s config sets `PreferredAuthentications` without `password`, so the password will never be used. Deploy will fail with “Permission denied”.」但**不**停用按鈕（使用者可能就是想用金鑰部署）。
- `keychainAvailable` 為 false → **停用並取消勾選「記住這台主機的密碼」**，旁邊顯示「No credential store on this machine — the password will be used once and not saved.」。這一步不可省略：後端在無密鑰環時走環境變數 fallback，`remember` 為 true 也不會有任何東西被存下來，勾選框若仍可勾就是在騙使用者。

- [ ] **Step 6: 建置與測試**

Run: `cd src-tauri && cargo test && cd .. && pnpm build && pnpm test`
Expected: 全綠

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/deploy.rs src-tauri/src/lib.rs src/bindings/DeployPreflight.ts \
        src/lib/queries.ts src/components/DeployKeyDialog.tsx
git commit -m "feat(deploy): warn about old OpenSSH and password-blocking config"
```

---

### Task 12: 端到端手動驗證與文件

**Files:**
- Modify: `README.md`
- Create: `docs/superpowers/plans/2026-07-29-key-deploy-manual-verification.md`

**Interfaces:**
- Consumes: 前面所有 task
- Produces: 一份填好的驗證記錄

- [ ] **Step 1: 起驗證用的 sshd**

```bash
docker run -d --name sshelter-verify -p 2222:2222 \
  -e PASSWORD_ACCESS=true -e USER_NAME=spike -e USER_PASSWORD=hunter2 \
  -e PUID=1000 -e PGID=1000 \
  linuxserver/openssh-server
sleep 10
```

在 `~/.ssh/config` 加入（驗證後移除）：

```
Host sshelter-verify
    HostName localhost
    Port 2222
    User spike
```

- [ ] **Step 2: 逐項執行 spec 的手動驗證清單**

Run: `pnpm tauri dev`，依序驗證並把每一項的實際結果記進驗證文件：

1. 全新主機（不在 known_hosts）→ 顯示指紋 → Trust & continue → 部署成功 → 終端機執行 `ssh sshelter-verify` **免密碼登入**。
2. 對同一台重複部署同一把 key → 顯示「Key was already there — nothing added」。
3. 故意打錯密碼 → app 內顯示「Wrong password」，且從 sshd log 確認**只嘗試一次**（`docker logs sshelter-verify | grep -c "Failed password"`）。
4. 勾「記住密碼」→ 完全結束 app 再開 → HostEditor 的 Password 區塊顯示 Saved、Show 讀得回來。
5. 不勾「記住密碼」→ 部署後執行 `security find-generic-password -s SSHelter -a "deploy-tmp:sshelter-verify"` → **必須回 not found**。
6. 部署期間另開終端機執行 `ps aux | grep -i ssh`，確認**密碼不出現在任何進程的命令列**。
7. 竄改 known_hosts 中該主機的金鑰 → 部署中止並顯示 Mismatch 警告，**且沒有任何繼續的按鈕**。
8. 停掉容器後再部署 → 顯示「Could not reach the host」。

- [ ] **Step 3: 清理**

```bash
docker rm -f sshelter-verify
# 移除 ~/.ssh/config 裡的 sshelter-verify 區塊與 known_hosts 中的 [localhost]:2222 項目
```

- [ ] **Step 4: 更新 README**

在 README 的功能列表中，把既有的 ssh-copy-id 敘述改為說明新的 app 內部署流程，並加上一句：密碼存於作業系統 keychain，不寫入 `~/.ssh/config`。

- [ ] **Step 5: 全套測試**

Run: `cd src-tauri && cargo test && cd .. && pnpm test && pnpm build`
Expected: 全綠

- [ ] **Step 6: Commit**

```bash
git add README.md docs/superpowers/plans/2026-07-29-key-deploy-manual-verification.md
git commit -m "docs: record manual verification for in-app key deployment"
```

---

## 自我檢查結果

**Spec 覆蓋**：spec 的 10 個具體變更逐項對應 —— Cargo.toml/Task 1、secrets.rs/Task 1、askpass.rs/Task 2、deploy.rs/Task 4–6、main.rs/Task 3、Tauri commands/Task 6+11、DeployKeyDialog/Task 8、HostList/Task 9、HostEditor/Task 10、queries+bindings/Task 7。安全模型 8 條保證分別落在 Task 2（白名單）、4（stdin、純 argv）、5（host key）、6（清理）。三個待驗證假設在 Task 0 與 Task 3 Step 4 處理。

**已知的刻意取捨**（都寫進了對應 task）：

1. Task 9 不做 host row 右鍵的「Copy password」—— 逐列查 keychain 在大型 config 上會很慢，改由 HostEditor 提供入口。
2. `run_ssh_deploy` 與各 command 屬於有副作用的執行層，不做單元測試，改由 Task 12 的手動驗證涵蓋；所有可測邏輯都已抽成純函式。
3. Task 6 的 `deploy_trust_host_key` 重新驗證前端傳來的 `key_line`，不信任前端傳值。

**自我檢查揪出並已修正的問題**：

- `DeployPreflight` 原本沒有 `keychain_available`，導致本機無密鑰環時「記住密碼」勾選框仍可勾、但實際上什麼都沒存 —— 等於騙使用者。已補欄位並在 Task 11 Step 5 要求停用該勾選框。
- serde 的 `rename_all` 在 enum 上只改變體名、不改變體內欄位名，但在 struct 上會改欄位名。這會讓前端寫錯 `key_line` / `askpassSupported`，已寫進 Global Constraints 明示。
- Task 9 原本同時「示範 Copy password 的程式碼」又「要求把它刪掉」，自相矛盾，已統一為不實作。
- Task 5 的 Interfaces 誤列了 `effective_config`（實際到 Task 6 才用），已更正。
- `password_auth_is_blocked` 宣告為私有但 Interfaces 寫 `pub fn`，已統一為 `pub fn`。
