# SSHelter — Phase 0: Foundation & Boundary 實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 SSHelter 的專案骨架、安全邊界（前端零 fs/零 shell 權限）、錯誤型別、型別橋接，以及經過完整單元測試的安全檔案 IO 工具 (`fsutil`) 建立起來，作為後續所有功能的地基。

**Architecture:** Tauri 2（Rust 後端做所有特權操作）+ React/TS/Vite 前端。所有 `~/.ssh` 檔案 IO 都在 Rust `#[tauri::command]` 內用 `std`/`tempfile` 完成，刻意不給 WebView 任何 `fs`/`shell` 權限；安全邊界在 Rust command 介面。本階段先把骨架、`AppError`、`fsutil`（原子寫入／權限／備份／漂移偵測）做到綠燈，UI 與 SSH 功能留待後續 plan。

**Tech Stack:** Tauri 2.9.x、Rust 1.77.2+、React + TypeScript + Vite、Tailwind CSS v4、shadcn/ui、TanStack Query v5、Zustand v5、thiserror、sha2、tempfile、ts-rs。

> 完整設計見 `docs/superpowers/specs/2026-06-08-ssh-config-manager-design.md`。本 plan 對應 spec §13 的 **Phase 0**。

---

## 先決條件 (Prerequisites)

執行前確認本機已安裝（不需寫進步驟，但缺一不可）：

- **rustup / Rust 1.77.2+**：`rustc --version`
- **Node 18+ 與 pnpm**：`pnpm --version`（使用者既有環境以 pnpm 為主）
- **macOS**：Xcode Command Line Tools（`xcode-select --install`）
- **Linux (Debian/Ubuntu)**：`libwebkit2gtk-4.1-dev build-essential libssl-dev librsvg2-dev libayatana-appindicator3-dev`

repo 已是 git repo（branch `main`），且已含 `docs/` 與 `.gitignore`。**scaffold 時不要覆蓋既有的 `.gitignore` 與 `docs/`。**

---

## File Structure（本階段建立／修改的檔案）

由 scaffold 產生（Task 1）：`package.json`、`pnpm-lock.yaml`、`index.html`、`vite.config.ts`、`tsconfig.json`、`tsconfig.app.json`、`src/main.tsx`、`src/App.tsx`、`src/index.css`、`src-tauri/Cargo.toml`、`src-tauri/tauri.conf.json`、`src-tauri/src/main.rs`、`src-tauri/src/lib.rs`、`src-tauri/capabilities/default.json`。

本階段新增／改寫，各檔單一職責：

| 檔案 | 職責 |
|---|---|
| `src-tauri/src/error.rs` | `AppError`（thiserror 列舉 + 手動 `Serialize` 成訊息字串），所有 command 的統一錯誤型別 |
| `src-tauri/src/fsutil.rs` | 安全檔案 IO：`ensure_dir_secure`、`atomic_write`、`backup`、`file_fingerprint`、`has_changed`（漂移偵測）；含完整 `#[cfg(test)]` 測試 |
| `src-tauri/src/lib.rs`（改寫） | `tauri::Builder`、plugin 註冊、`invoke_handler`、smoke command `app_platform`、宣告 `mod error/fsutil` |
| `src-tauri/capabilities/default.json`（改寫） | 最小權限 capability：只給 `core:default` + `opener:default` + `os:default`，**不給 fs/shell** |
| `src-tauri/Cargo.toml`（修改） | 加 `thiserror`、`sha2`、`tempfile`、`tauri-plugin-os`；dev-dep `ts-rs` |
| `src-tauri/tauri.conf.json`（修改） | `productName: SSHelter`、`identifier: org.homelab.sshelter`、視窗標題 |
| `src/lib/ipc.ts` | 前端 typed `invoke` 包裝 `tauriInvoke<T>()` |
| `src/stores/ui.ts` | Zustand UI 狀態 store（theme），確立 UI 狀態層模式 |
| `src/main.tsx`（改寫） | 包上 `QueryClientProvider` |
| `src/App.tsx`（改寫） | 用 `useQuery` 呼叫 `app_platform`，渲染平台字串 + 一個 shadcn `Button`（端到端 smoke） |
| `src/components/ui/button.tsx` | shadcn 產生 |
| `src/bindings/Fingerprint.ts` | ts-rs 由 Rust 型別產生（型別橋接 pattern） |

---

## Task 1: Scaffold Tauri 2 + React/TS/Vite 進既有 repo

**Files:**
- Create（由 scaffold）：`package.json`、`vite.config.ts`、`tsconfig*.json`、`index.html`、`src/*`、`src-tauri/*`

- [ ] **Step 1: 在暫存目錄 scaffold（避免汙染既有 repo）**

```bash
rm -rf /tmp/sshelter-scaffold
pnpm create tauri-app@latest sshelter-scaffold \
  --template react-ts --manager pnpm \
  --identifier org.homelab.sshelter
# 若 CLI 不吃旗標，改互動模式並選擇：Frontend=React、Language=TypeScript、Package manager=pnpm
mv /tmp/sshelter-scaffold 2>/dev/null; true   # no-op guard
ls sshelter-scaffold
```

> 註：`create-tauri-app` 會建在「目前目錄下的新資料夾」，所以先在一個乾淨位置產生，再搬進 repo。執行上面這段時請 `cd /tmp` 後再跑，使 `sshelter-scaffold` 落在 `/tmp/sshelter-scaffold`。

- [ ] **Step 2: 把 scaffold 內容搬進 repo（保留既有 .gitignore / docs / .git）**

```bash
rsync -a \
  --exclude node_modules \
  --exclude .git \
  --exclude .gitignore \
  /tmp/sshelter-scaffold/ /Users/ysya/project/homelab/ssheditor/
```

- [ ] **Step 3: 安裝相依並驗證可建置**

Run（在 repo 根目錄）:
```bash
pnpm install
pnpm tauri dev
```
Expected: 編譯 Rust 後彈出一個應用程式視窗（預設 greet 範例）。確認能開窗後 `Ctrl-C` 結束。

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "chore: scaffold Tauri 2 + React/TS/Vite app"
```

---

## Task 2: 設定產品識別 (product identity)

**Files:**
- Modify: `src-tauri/tauri.conf.json`

- [ ] **Step 1: 編輯 tauri.conf.json 的 productName / identifier / 視窗標題**

把最上層的 `productName`、`identifier`，以及 `app.windows[0].title` 改成：

```json
{
  "productName": "SSHelter",
  "identifier": "org.homelab.sshelter",
  "app": {
    "windows": [
      {
        "title": "SSHelter",
        "width": 1100,
        "height": 720,
        "minWidth": 800,
        "minHeight": 560
      }
    ]
  }
}
```

> 只改這幾個鍵，其餘 `build`/`bundle`/`security` 維持 scaffold 預設。

- [ ] **Step 2: 驗證設定有效**

Run（在 repo 根目錄）:
```bash
pnpm tauri dev
```
Expected: 視窗標題顯示 `SSHelter`，視窗約 1100×720。確認後結束。

- [ ] **Step 3: Commit**

```bash
git add src-tauri/tauri.conf.json
git commit -m "chore: set product name SSHelter and bundle identifier"
```

---

## Task 3: 最小權限 capability（鎖死安全邊界）

**Files:**
- Modify: `src-tauri/capabilities/default.json`
- Modify: `src-tauri/Cargo.toml`
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: 改寫 capability 為最小權限**

把 `src-tauri/capabilities/default.json` 整檔改為（**不含任何 `fs:` 或 `shell:` 權限**）：

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Least-privilege capability for SSHelter's main window. The WebView gets NO filesystem and NO shell-exec permission; all privileged IO/exec happens in Rust commands. This is the security boundary.",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "opener:default",
    "os:default"
  ]
}
```

- [ ] **Step 2: 加入 `tauri-plugin-os` crate**

在 `src-tauri/Cargo.toml` 的 `[dependencies]` 加一行（其餘相依在 Task 4 一併加好）：

```toml
tauri-plugin-os = "2"
```

- [ ] **Step 3: 在 lib.rs 註冊 os plugin**

在 `src-tauri/src/lib.rs` 的 `tauri::Builder` 鏈上，於既有的 `.plugin(tauri_plugin_opener::init())` 之後加：

```rust
        .plugin(tauri_plugin_os::init())
```

- [ ] **Step 4: 驗證 app 仍能啟動（capability 與 plugin 對得上）**

Run:
```bash
pnpm tauri dev
```
Expected: 視窗正常開啟、主控台無「permission not found」錯誤。確認後結束。

> 若出現 capability 找不到某 permission，代表 capability 宣告了某 plugin 但 plugin 未註冊——本步驟只宣告 os/opener/core，兩者都已註冊。`dialog`/`clipboard-manager` 等留到實際使用的 phase 再加。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/capabilities/default.json src-tauri/Cargo.toml src-tauri/src/lib.rs
git commit -m "feat: lock least-privilege capability (no fs/shell to WebView)"
```

---

## Task 4: `AppError` 統一錯誤型別（TDD）

**Files:**
- Create: `src-tauri/src/error.rs`
- Modify: `src-tauri/src/lib.rs`（宣告 `mod error;`）
- Modify: `src-tauri/Cargo.toml`（加 `thiserror`、`serde_json`、`sha2`、`tempfile`）

- [ ] **Step 1: 加入本階段所需相依**

在 `src-tauri/Cargo.toml` 的 `[dependencies]` 確保有（`serde`/`serde_json` scaffold 已含，缺則補）：

```toml
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
sha2 = "0.10"
tempfile = "3"
```

- [ ] **Step 2: 建立 error.rs 並寫失敗測試**

建立 `src-tauri/src/error.rs`，先只放型別宣告的空殼 + 測試：

```rust
use serde::{Serialize, Serializer};

/// 所有 #[tauri::command] 回傳的統一錯誤型別。
/// 序列化成「訊息字串」，讓前端 invoke() 的 promise 以可讀字串 reject。
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("io error: {0}")]
    Io(#[from] std::io::Error),

    #[error("path is not allowed: {0}")]
    ForbiddenPath(String),

    #[error("not found: {0}")]
    NotFound(String),

    #[error("{0}")]
    Other(String),
}

impl Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_str(&self.to_string())
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn app_error_serializes_to_its_message() {
        let e = AppError::Other("boom".into());
        let json = serde_json::to_string(&e).unwrap();
        assert_eq!(json, "\"boom\"");
    }

    #[test]
    fn io_error_converts_and_serializes() {
        let io = std::io::Error::new(std::io::ErrorKind::NotFound, "missing");
        let e: AppError = io.into();
        let json = serde_json::to_string(&e).unwrap();
        assert_eq!(json, "\"io error: missing\"");
    }
}
```

在 `src-tauri/src/lib.rs` 最上方加：

```rust
mod error;
```

- [ ] **Step 3: 跑測試確認通過**

Run（在 `src-tauri/`）:
```bash
cargo test error::
```
Expected: `app_error_serializes_to_its_message` 與 `io_error_converts_and_serializes` 兩個 PASS。

> 本任務的「失敗→通過」其實在於先確認專案能編譯並跑出綠燈；型別與測試一起寫入。若 `thiserror = "2"` 解析失敗，確認已 `pnpm tauri dev` 觸發過一次 `cargo` fetch，或手動 `cargo build`。

- [ ] **Step 4: Commit**

```bash
git add src-tauri/src/error.rs src-tauri/src/lib.rs src-tauri/Cargo.toml
git commit -m "feat: add AppError unified command error type"
```

---

## Task 5: `fsutil::ensure_dir_secure`（TDD）

**Files:**
- Create: `src-tauri/src/fsutil.rs`
- Modify: `src-tauri/src/lib.rs`（宣告 `mod fsutil;`）

- [ ] **Step 1: 建立 fsutil.rs，寫失敗測試**

建立 `src-tauri/src/fsutil.rs`：

```rust
use std::fs;
use std::path::Path;

use crate::error::AppError;

/// 確保目錄存在，且（unix）權限為 0700。不存在才建立。
pub fn ensure_dir_secure(dir: &Path) -> Result<(), AppError> {
    if !dir.exists() {
        fs::create_dir_all(dir)?;
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            fs::set_permissions(dir, fs::Permissions::from_mode(0o700))?;
        }
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[cfg(unix)]
    #[test]
    fn ensure_dir_secure_creates_0700() {
        use std::os::unix::fs::PermissionsExt;
        let dir = tempfile::tempdir().unwrap();
        let sub = dir.path().join(".ssh");
        ensure_dir_secure(&sub).unwrap();
        let mode = fs::metadata(&sub).unwrap().permissions().mode() & 0o777;
        assert_eq!(mode, 0o700);
    }
}
```

在 `src-tauri/src/lib.rs` 加：

```rust
mod fsutil;
```

- [ ] **Step 2: 跑測試確認通過**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::ensure_dir_secure_creates_0700
```
Expected: PASS（macOS/Linux）。

- [ ] **Step 3: Commit**

```bash
git add src-tauri/src/fsutil.rs src-tauri/src/lib.rs
git commit -m "feat(fsutil): ensure_dir_secure creates 0700 dirs"
```

---

## Task 6: `fsutil::atomic_write`（TDD）

**Files:**
- Modify: `src-tauri/src/fsutil.rs`

- [ ] **Step 1: 先加失敗測試**

在 `src-tauri/src/fsutil.rs` 的 `mod tests` 內加：

```rust
    #[test]
    fn atomic_write_creates_file_with_contents() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("config");
        atomic_write(&p, b"Host x\n", 0o600).unwrap();
        assert_eq!(fs::read(&p).unwrap(), b"Host x\n");
    }

    #[cfg(unix)]
    #[test]
    fn atomic_write_sets_mode_0600() {
        use std::os::unix::fs::PermissionsExt;
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("config");
        atomic_write(&p, b"data", 0o600).unwrap();
        let mode = fs::metadata(&p).unwrap().permissions().mode() & 0o777;
        assert_eq!(mode, 0o600);
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::atomic_write
```
Expected: 編譯失敗，`cannot find function atomic_write`。

- [ ] **Step 3: 實作 atomic_write**

在 `src-tauri/src/fsutil.rs` 頂部 `use` 補上 `use std::io::Write;`，並在 `ensure_dir_secure` 之後加：

```rust
/// 原子寫入：在目標同目錄建 temp 檔，設好權限後寫入、fsync，再 rename 蓋回。
/// `mode` 為 unix 權限（如 config 用 0o600、known_hosts 用 0o644）。
pub fn atomic_write(path: &Path, contents: &[u8], mode: u32) -> Result<(), AppError> {
    let parent = path
        .parent()
        .ok_or_else(|| AppError::Other(format!("no parent dir for {}", path.display())))?;
    ensure_dir_secure(parent)?;

    let mut tmp = tempfile::NamedTempFile::new_in(parent)?;
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        tmp.as_file()
            .set_permissions(fs::Permissions::from_mode(mode))?;
    }
    #[cfg(not(unix))]
    {
        let _ = mode; // Windows 權限後續以 ACL 處理
    }
    tmp.write_all(contents)?;
    tmp.as_file().sync_all()?;
    tmp.persist(path).map_err(|e| AppError::Io(e.error))?;
    Ok(())
}
```

- [ ] **Step 4: 跑測試確認通過**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::atomic_write
```
Expected: `atomic_write_creates_file_with_contents`、`atomic_write_sets_mode_0600` 皆 PASS。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/fsutil.rs
git commit -m "feat(fsutil): atomic_write with unix mode + same-dir temp + rename"
```

---

## Task 7: `fsutil::backup`（TDD）

**Files:**
- Modify: `src-tauri/src/fsutil.rs`

- [ ] **Step 1: 先加失敗測試**

在 `mod tests` 內加：

```rust
    #[test]
    fn backup_copies_existing_file() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("config");
        fs::write(&p, b"original").unwrap();
        let b = backup(&p).unwrap().expect("backup should be created");
        assert_eq!(fs::read(&b).unwrap(), b"original");
        assert!(b.to_string_lossy().ends_with(".bak"));
    }

    #[test]
    fn backup_returns_none_when_missing() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("nope");
        assert!(backup(&p).unwrap().is_none());
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::backup
```
Expected: 編譯失敗，`cannot find function backup`。

- [ ] **Step 3: 實作 backup**

在 `src-tauri/src/fsutil.rs` 的 `use` 補 `use std::path::PathBuf;` 與 `use std::time::SystemTime;`，並加：

```rust
/// 若 `path` 存在，複製成 `<path>.<unix_millis>.bak` 並回傳備份路徑；不存在則回 None。
/// 在每個 session 第一次寫入 live 檔案前呼叫。
pub fn backup(path: &Path) -> Result<Option<PathBuf>, AppError> {
    if !path.exists() {
        return Ok(None);
    }
    let millis = SystemTime::now()
        .duration_since(SystemTime::UNIX_EPOCH)
        .map_err(|e| AppError::Other(e.to_string()))?
        .as_millis();

    let mut name = path
        .file_name()
        .ok_or_else(|| AppError::Other(format!("no file name for {}", path.display())))?
        .to_os_string();
    name.push(format!(".{millis}.bak"));
    let backup_path = path.with_file_name(name);

    fs::copy(path, &backup_path)?;
    Ok(Some(backup_path))
}
```

- [ ] **Step 4: 跑測試確認通過**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::backup
```
Expected: `backup_copies_existing_file`、`backup_returns_none_when_missing` 皆 PASS。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/fsutil.rs
git commit -m "feat(fsutil): timestamped backup before first write"
```

---

## Task 8: `fsutil` 漂移偵測 `file_fingerprint` + `has_changed`（TDD）

**Files:**
- Modify: `src-tauri/src/fsutil.rs`

- [ ] **Step 1: 先加失敗測試**

在 `mod tests` 內加：

```rust
    #[test]
    fn fingerprint_is_stable_for_same_content() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("config");
        fs::write(&p, b"v1").unwrap();
        let a = file_fingerprint(&p).unwrap();
        let b = file_fingerprint(&p).unwrap();
        assert_eq!(a.sha256, b.sha256);
    }

    #[test]
    fn has_changed_detects_modification() {
        let dir = tempfile::tempdir().unwrap();
        let p = dir.path().join("config");
        fs::write(&p, b"v1").unwrap();
        let snap = file_fingerprint(&p).unwrap();
        assert!(!has_changed(&p, &snap).unwrap());
        fs::write(&p, b"v2").unwrap();
        assert!(has_changed(&p, &snap).unwrap());
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::fingerprint fsutil::tests::has_changed
```
Expected: 編譯失敗，`cannot find function file_fingerprint` / `has_changed`。

- [ ] **Step 3: 實作 fingerprint 與 has_changed**

在 `src-tauri/src/fsutil.rs` 的 `use` 補 `use serde::{Deserialize, Serialize};` 與 `use sha2::{Digest, Sha256};`，並加：

```rust
/// 檔案指紋：用於偵測 SSH config 被外部工具（ssh CLI、其他編輯器）改動。
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
pub struct Fingerprint {
    /// 修改時間（unix 毫秒；取不到時為 0）
    pub mtime_ms: u64,
    /// 檔案內容的小寫 hex SHA-256
    pub sha256: String,
}

pub fn file_fingerprint(path: &Path) -> Result<Fingerprint, AppError> {
    let bytes = fs::read(path)?;
    let mut hasher = Sha256::new();
    hasher.update(&bytes);
    let sha256 = hex_lower(&hasher.finalize());

    let mtime_ms = fs::metadata(path)?
        .modified()
        .ok()
        .and_then(|t| t.duration_since(SystemTime::UNIX_EPOCH).ok())
        .map(|d| d.as_millis() as u64)
        .unwrap_or(0);

    Ok(Fingerprint { mtime_ms, sha256 })
}

/// 檔案目前內容雜湊是否與快照不同（內容導向，比 mtime 可靠）。
pub fn has_changed(path: &Path, snapshot: &Fingerprint) -> Result<bool, AppError> {
    let current = file_fingerprint(path)?;
    Ok(current.sha256 != snapshot.sha256)
}

fn hex_lower(bytes: &[u8]) -> String {
    use std::fmt::Write as _;
    let mut s = String::with_capacity(bytes.len() * 2);
    for b in bytes {
        let _ = write!(s, "{b:02x}");
    }
    s
}
```

- [ ] **Step 4: 跑測試確認通過**

Run（在 `src-tauri/`）:
```bash
cargo test fsutil::tests::fingerprint fsutil::tests::has_changed
```
Expected: 兩個 PASS。

- [ ] **Step 5: 跑整包測試確認全綠**

Run（在 `src-tauri/`）:
```bash
cargo test
```
Expected: 所有 `error::` 與 `fsutil::` 測試 PASS。

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/fsutil.rs
git commit -m "feat(fsutil): file fingerprint + drift detection"
```

---

## Task 9: smoke command `app_platform` 並接上 invoke_handler

**Files:**
- Modify: `src-tauri/src/lib.rs`

- [ ] **Step 1: 加入 command 與註冊**

把 `src-tauri/src/lib.rs` 改成（保留既有 plugin 註冊，移除 scaffold 的 `greet` 範例）：

```rust
mod error;
mod fsutil;

/// 端到端 smoke command：回傳目前作業系統（"macos" / "linux" / "windows"）。
#[tauri::command]
fn app_platform() -> String {
    std::env::consts::OS.to_string()
}

#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        .plugin(tauri_plugin_opener::init())
        .plugin(tauri_plugin_os::init())
        .invoke_handler(tauri::generate_handler![app_platform])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

> 若 scaffold 的 `lib.rs` 仍 `pub fn greet(...)` 並被 `src/App.tsx` 呼叫，下一個 Task 會改寫 `App.tsx`，此處可安全移除 `greet`。`main.rs` 維持 scaffold 產生的內容（已呼叫 `sshelter_lib::run()`）。

- [ ] **Step 2: 驗證 app 啟動且 command 可被呼叫**

Run:
```bash
pnpm tauri dev
```
Expected: 視窗開啟、Rust 編譯無誤、主控台無錯誤。（前端實際呼叫在 Task 11 驗證。）結束。

- [ ] **Step 3: Commit**

```bash
git add src-tauri/src/lib.rs
git commit -m "feat: add app_platform smoke command and register handler"
```

---

## Task 10: Tailwind v4 + shadcn/ui 初始化

**Files:**
- Modify: `vite.config.ts`、`tsconfig.json`、`tsconfig.app.json`、`src/index.css`
- Create: `components.json`、`src/components/ui/button.tsx`、`src/lib/utils.ts`（由 shadcn 產生）

- [ ] **Step 1: 安裝 Tailwind v4 與型別**

Run（repo 根目錄）:
```bash
pnpm add tailwindcss @tailwindcss/vite
pnpm add -D @types/node
```

- [ ] **Step 2: 設定 Vite plugin 與 `@/` alias**

把 `vite.config.ts` 改為（保留 scaffold 既有的 `@tauri-apps` 相關設定，如 `clearScreen`、`server` 區塊）：

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import tailwindcss from "@tailwindcss/vite";
import path from "node:path";

// @tauri-apps scaffold 既有設定請保留（server.port/strictPort、clearScreen 等）
export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
  clearScreen: false,
  server: { port: 1420, strictPort: true },
});
```

- [ ] **Step 3: 在「三個地方」設定 `@/` alias**

`tsconfig.json` 與 `tsconfig.app.json` 的 `compilerOptions` 都要加：

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": { "@/*": ["./src/*"] }
  }
}
```

> alias 漏設任一處是 shadcn 最常見的安裝 bug；vite.config.ts（Step 2）+ 上面兩個 tsconfig 共三處。

- [ ] **Step 4: 切換到 Tailwind v4 CSS-first**

把 `src/index.css` 開頭（或整檔）替換成：

```css
@import "tailwindcss";
```

- [ ] **Step 5: 初始化 shadcn 並加 Button**

Run（repo 根目錄）:
```bash
pnpm dlx shadcn@latest init
# 互動選項：Base color 任選（如 Neutral）；其餘用預設（會偵測 Vite + Tailwind v4）
pnpm dlx shadcn@latest add button
```
Expected: 產生 `components.json`、`src/lib/utils.ts`、`src/components/ui/button.tsx`。

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "chore: set up Tailwind v4 + shadcn/ui with @/ alias"
```

---

## Task 11: 前端資料層（TanStack Query + Zustand）+ 端到端 smoke

**Files:**
- Create: `src/lib/ipc.ts`、`src/stores/ui.ts`
- Modify: `src/main.tsx`、`src/App.tsx`

- [ ] **Step 1: 安裝資料層相依**

Run（repo 根目錄）:
```bash
pnpm add @tanstack/react-query zustand @tauri-apps/api
```

- [ ] **Step 2: 建立 typed invoke 包裝**

建立 `src/lib/ipc.ts`：

```ts
import { invoke } from "@tauri-apps/api/core";

/** 所有後端呼叫都經這個包裝，之後可在此統一處理錯誤/型別。 */
export function tauriInvoke<T>(
  cmd: string,
  args?: Record<string, unknown>,
): Promise<T> {
  return invoke<T>(cmd, args);
}
```

- [ ] **Step 3: 建立 Zustand UI store（確立 UI 狀態層模式）**

建立 `src/stores/ui.ts`：

```ts
import { create } from "zustand";

interface UiState {
  theme: "light" | "dark";
  setTheme: (theme: "light" | "dark") => void;
}

/** 只放 UI 狀態，永不鏡像後端資料（後端資料由 TanStack Query 持有）。 */
export const useUiStore = create<UiState>((set) => ({
  theme: "light",
  setTheme: (theme) => set({ theme }),
}));
```

- [ ] **Step 4: 包上 QueryClientProvider**

把 `src/main.tsx` 改為：

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import App from "./App";
import "./index.css";

const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </React.StrictMode>,
);
```

- [ ] **Step 5: App.tsx 呼叫 app_platform（端到端 smoke）**

把 `src/App.tsx` 改為：

```tsx
import { useQuery } from "@tanstack/react-query";
import { tauriInvoke } from "@/lib/ipc";
import { Button } from "@/components/ui/button";

function App() {
  const { data: platform, isLoading } = useQuery({
    queryKey: ["app", "platform"],
    queryFn: () => tauriInvoke<string>("app_platform"),
  });

  return (
    <main className="p-8 space-y-4">
      <h1 className="text-2xl font-semibold">SSHelter</h1>
      <p className="text-muted-foreground">
        platform: {isLoading ? "…" : platform}
      </p>
      <Button>It works</Button>
    </main>
  );
}

export default App;
```

- [ ] **Step 6: 驗證端到端（前端→IPC→Rust→前端）**

Run:
```bash
pnpm tauri dev
```
Expected: 視窗顯示「SSHelter」、`platform: macos`（或 `linux`）、一顆可點的 shadcn Button，且 Tailwind 樣式生效（標題大字、按鈕有底色）。確認後結束。

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: wire TanStack Query + Zustand + end-to-end IPC smoke"
```

---

## Task 12: ts-rs 型別橋接（確立 Rust↔TS 型別產生 pattern）

**Files:**
- Modify: `src-tauri/Cargo.toml`、`src-tauri/src/fsutil.rs`
- Create（產生）：`src/bindings/Fingerprint.ts`

- [ ] **Step 1: 加入 ts-rs dev-dependency**

在 `src-tauri/Cargo.toml` 加：

```toml
[dev-dependencies]
ts-rs = "10"
```

- [ ] **Step 2: 在 Fingerprint 上 derive TS 並標記匯出**

把 `src-tauri/src/fsutil.rs` 的 `Fingerprint` 定義改為（用 `cfg_attr` 讓 ts-rs 只在測試建置時介入，避免污染正式相依）：

```rust
#[derive(Debug, Clone, Serialize, Deserialize, PartialEq, Eq)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../src/bindings/"))]
pub struct Fingerprint {
    /// 修改時間（unix 毫秒；取不到時為 0）
    pub mtime_ms: u64,
    /// 檔案內容的小寫 hex SHA-256
    pub sha256: String,
}
```

- [ ] **Step 3: 跑 cargo test 觸發型別產生**

Run（在 `src-tauri/`）:
```bash
cargo test export_bindings_fingerprint || cargo test
```
Expected: ts-rs 於測試時把型別輸出到 `src/bindings/Fingerprint.ts`。

- [ ] **Step 4: 確認產生的 TS 型別檔存在且內容正確**

Run（repo 根目錄）:
```bash
cat src/bindings/Fingerprint.ts
```
Expected: 內含
```ts
export type Fingerprint = { mtime_ms: number, sha256: string };
```
（欄位名與型別需與 Rust 對齊：`mtime_ms: number`、`sha256: string`。）

- [ ] **Step 5: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/src/fsutil.rs src/bindings/Fingerprint.ts
git commit -m "feat: establish ts-rs Rust->TS type bridge (Fingerprint)"
```

---

## Self-Review（已執行）

**1. Spec 覆蓋（對應 spec §13 Phase 0「鷹架與邊界」）：**
- create-tauri-app React+TS+Vite → Task 1 ✅
- lib.rs/main.rs split → scaffold 既有，Task 9 改寫 lib.rs，main.rs 維持 ✅
- Tailwind v4 + shadcn init + `@/` alias（三處）→ Task 10 ✅
- capabilities 只給 core+os+opener、**不給 fs/shell** → Task 3 ✅（dialog/clipboard 依 YAGNI 延到使用時再加，已註明）
- TanStack Query + Zustand 接線 → Task 11 ✅
- thiserror `AppError` + 手動 Serialize → Task 4 ✅
- 型別橋接 ts-rs → Task 12 ✅
- fsutil：原子寫、0700/0600 權限、備份、mtime/hash 漂移 → Task 5–8 ✅

**2. Placeholder 掃描：** 無 TBD/TODO；每個會改碼的步驟都附完整程式碼與測試。

**3. 型別一致性：** `AppError`（error.rs）被 fsutil 各函式以 `Result<_, AppError>` 回傳一致；`Fingerprint` 欄位 `mtime_ms: u64` / `sha256: String` 在定義、測試、ts-rs 匯出三處一致；`atomic_write(path, contents, mode)`、`backup(path) -> Option<PathBuf>`、`file_fingerprint(path) -> Fingerprint`、`has_changed(path, &Fingerprint) -> bool` 簽章在使用處一致。

**偏離 spec 之處（刻意）：** Phase 0 capability 暫不註冊 `dialog`/`clipboard-manager`（YAGNI——宣告未註冊的 plugin permission 會在執行期報錯），延到 Phase 3/連線等實際使用的 phase 再加。

---

## 後續 Plan 路線（每個 phase 一份獨立 plan）

| 下一份 plan | 內容（對應 spec phase） | 交付的可測試軟體 |
|---|---|---|
| **Phase 1a — config CST 核心** | 無損 lexer/parser/serializer + golden-file byte-identical 測試 + 編輯操作（set field、add/remove/reorder、註解切換）+ 多檔 Include 模型 + `config_*` commands | 一個經完整測試、能無損讀寫 `~/.ssh/config`（含 Include）的 Rust core，behind tauri commands |
| **Phase 1b — config 編輯器 UI** | master-detail 虛擬清單、Host 編輯器（Connection/Auth/Forwarding/Reliability 分頁 + extraOptions）、rhf+zod、dirtyFields 最小 diff 存檔、groups/tags、漂移 reload 提示 | 可用的圖形化 SSH config 編輯器 |
| **Phase 2** | 連線（TerminalLauncher、外部終端機、cmdk 快速連線） | 一鍵在終端機連線 |
| **Phase 3** | 金鑰管理（ssh-key、產生/passphrase/fingerprint、寫 IdentityFile） | 金鑰管理 |
| **Phase 4** | ssh-agent（list/lock/add via ssh-add、askpass） | agent 整合 |
| **Phase 5** | known_hosts + port forwarding 視覺化 | known_hosts 管理與轉發編輯 |
| **Phase 6** | 打磨、簽章/公證、AppImage/deb/rpm、自動更新、Windows groundwork | 可發佈版本 |
