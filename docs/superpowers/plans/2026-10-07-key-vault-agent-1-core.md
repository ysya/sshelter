# Key Vault and SSH Agent — Plan 1: Core Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** SSHelter keeps private keys in its own encrypted vault and serves them to the system `ssh` through its own SSH agent, asking for approval in an always-on-top window; a synced key slot can be switched to "Only in SSHelter" and used from any terminal or from SSHelter's Connect.

**Architecture:** A vault file next to `sync-state.json` holds per-slot private keys sealed with a random key from the OS keychain. An agent (Unix socket / Windows named pipe, plain threads like the MCP bridge) speaks the RFC 9987 protocol, learns the destination host from `session-bind@openssh.com`, identifies the requesting program from the peer PID, and asks a prompt hub that drives a second Tauri window. SSHelter writes `~/.ssh/sshelter/agent/config` (only hosts whose slot is vault-delivered) and Includes it as the first line of `~/.ssh/config`. SP3 slots gain a `SlotSource::Vault` delivery; Connect opens a one-shot pre-approved channel.

**Tech Stack:** Rust (Tauri 2.11, std threads), `ssh-key` 0.6.7, `chacha20poly1305` 0.10, `zeroize` 1, `windows-sys` 0.61 (Windows), React 19 + TypeScript (Vite, vitest with `renderToStaticMarkup`).

**Spec:** `docs/superpowers/specs/2026-10-07-key-vault-agent-design.md` (this plan is §14 item 1; read §4, §5, §6, §11, §15, §16).

## Global Constraints

- Tests never touch the real `~/.ssh`, the real Keychain, or the real app data directory: use `TestDevice` homes, `MemKeychain`, `tempfile::tempdir()`.
- Rust tests run from `src-tauri/` with exactly one `--`: `PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib <filter> -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain` (cargo accepts one filter before the `--`; with several filters, put all of them after the single `--`, before the skips: `cargo test --offline --lib -- vault:: agent:: --skip … --skip …`). Homebrew's rustc is broken here; always use that PATH.
- Frontend checks from the repo root: `./node_modules/.bin/tsc --noEmit`, `./node_modules/.bin/vitest run`, `./node_modules/.bin/vite build`. Never run `pnpm` or `npm install`.
- `src-tauri/Cargo.lock` is committed together with every dependency change (user decision 2026-10-07).
- Commits: conventional, English, no attribution lines. Rust doc comments in Traditional Chinese (repo style, ASCII punctuation inside Chinese text); TypeScript comments in English.
- Paths (spec §4.4): agent directory `~/.ssh/sshelter/agent/` (0700) holding `config`, `sock` and `run/`; Windows pipe `\\.\pipe\sshelter-agent-<first 16 hex of SHA-256 of the user SID string>`, written in config as `//./pipe/sshelter-agent-<same>`; one-shot channels `~/.ssh/sshelter/agent/run/<8 hex>` (only 8 bytes longer than `sock`: a socket path has a length limit) and `\\.\pipe\sshelter-connect-<32 hex>`.
- Vault (spec §4.1): `vault.json` beside `sync-state.json`; one XChaCha20-Poly1305 seal per entry with AAD `sshelter-vault-v1` + `\n` + slot id; the 32-byte key is standard base64 in keychain account `vault:key`; remembered passphrases in `vault:passphrase:<slot id>`.
- Approvals (spec §5.3): 60-second prompt timeout; remember choices 15, 60, 240 (default), 720 minutes; an unknown host, "ask every time" or "always ask on this computer" is never remembered; forwarded requests are refused; screen lock (Plan 3) and quit clear everything remembered.
- One-shot Connect channel (spec §5.6): first connection only, served only within 60 seconds (`ONE_SHOT_TIMEOUT`), closes after its connection ends;
  pre-approved; `ssh -o IdentityAgent=<channel> -o ForwardAgent=no <alias>`. The socket stays until `CHANNEL_LIFETIME` (10 minutes) only to tell a late
  `ssh` to Connect again (Task 12 fix round 2: ssh contacts the agent only when it first tries public-key authentication).
- Include (spec §6): `Include ~/.ssh/sshelter/agent/config` is item 0 of `~/.ssh/config`; sync's own Include goes after it; the config loader never loads `agent/config`; `agent/config` starts with `# Managed by SSHelter. Changes here are overwritten.` and each host block gets `IdentityAgent <socket>` and `IdentitiesOnly yes`.
- Exact UI strings (spec §7.4): window title `Allow {program} to use {key}?`; unknown host `an unknown host`; unknown program `an unknown program`; remember checkbox `Remember for {duration}` (`15 minutes`, `1 hour`, `4 hours`, `12 hours`); passphrase checkbox `Remember on this computer`; buttons `Deny` and `Allow`; Connect unlock title `Unlock {key} to connect to {host}` with `Cancel` and `Unlock`; slot toggle labels `Only in SSHelter` and `Keep a file`.
- Display strings coming from other programs or servers (user names, host names, program names) go through `revealHidden` in the UI.

## Review Focus

1. Parallel connections asking for the same key, host and program (for example `git fetch` of several repos, or two terminals) — expected: one prompt; the requests waiting behind it share its answer instead of prompting again. Pinned in Task 8.
2. `~/.ssh/config` that is empty, lacks a trailing newline, uses CRLF line endings, or starts with comments — expected: the Include becomes item 0 and every other byte stays as it was. Pinned in Task 10.
3. A stale `sock` file left by a crash, and a second SSHelter process — expected: a live first instance keeps serving and the second does not start an agent; a socket nobody answers on is replaced. Pinned in Task 9.
4. The requesting process exits before the agent inspects it (short-lived `ssh`) — expected: the prompt says `an unknown program`; nothing panics. Pinned in Task 5.
5. User, host or program names carrying control or bidi characters — expected: shown escaped in the approval window, never rendered raw. Pinned in Task 6.

## File Structure

| File | Responsibility |
|---|---|
| `src-tauri/src/sync/crypto.rs` (modify) | Add `seal_raw` / `open_raw` for a raw 32-byte key |
| `src-tauri/src/vault/mod.rs` (create) | `pub mod store; pub mod material;` |
| `src-tauri/src/vault/store.rs` (create; Task 7 adds `stored_ids`) | `vault.json`: entries, settings header, keychain key, set-aside |
| `src-tauri/src/vault/material.rs` (create) | Parse, decrypt and sign with OpenSSH private keys |
| `src-tauri/src/agent/mod.rs` (create) | Module list, `AgentRuntime` (in `AppState`), `agent_dir`, `identity_agent_value`, `AppAgentHost`, `start`, `AgentProblem` and its commands |
| `src-tauri/src/agent/approval.rs` (create) | Approval verdict and the remembered-approval cache (pure) |
| `src-tauri/src/agent/protocol.rs` (create) | Wire format: frames, messages, session-bind, userauth parsing |
| `src-tauri/src/agent/session.rs` (create) | Per-connection dispatch over a `SignAuthority` |
| `src-tauri/src/agent/peer.rs` (create) | Requesting program chain and identity from a PID |
| `src-tauri/src/agent/prompt.rs` (create) | Pending prompts, `agent_pending` / `agent_resolve`, the approval window |
| `src-tauri/src/agent/broker.rs` (create) | `Broker`: which keys, approvals, passphrases, opened keys; `AgentHost`; per-connection `Connection` |
| `src-tauri/src/agent/server.rs` (create) | Unix socket listener, the `lock` that keeps one agent per user, peer credentials, connection threads |
| `src-tauri/src/agent/pipe_windows.rs` (create) | Windows named pipe with an owner-only DACL; the one-shot pipe |
| `src-tauri/src/agent/wiring.rs` (create) | `agent/config` content and the Include line |
| `src-tauri/src/agent/oneshot.rs` (create) | Connect's one-shot channel and `prepare` |
| `src-tauri/src/agent/openssh_tests.rs` (create) | Tests with the real `ssh-add` and `ssh-keygen` |
| `src-tauri/src/sync/slots.rs`, `slot_setup.rs`, `state_v2.rs`, `dto.rs`, `engine.rs` (modify) | `SlotSource::Vault` and the delivery toggle |
| `src-tauri/src/sync/hosts_file.rs`, `config/include.rs`, `config/intel.rs` (modify) | Sync Include skips the agent Include; loader skips `agent/config`; lint accepts vault slots |
| `src-tauri/src/connect.rs`, `tray.rs` (modify) | Connect through a one-shot channel |
| `src-tauri/src/sync/slot_files_windows.rs` (modify) | `current_user_token` shared with the pipe |
| `src-tauri/src/lib.rs`, `state.rs`, `Cargo.toml` (modify), `capabilities/approval.json` (create) | Wiring, commands, dependencies, the approval window's minimal capability |
| `src/main.tsx` (modify), `src/components/AgentApprovalWindow.tsx` (create), `src/lib/agent.ts` (create) | Approval window UI |
| `src/components/KeySlotsSection.tsx`, `src/lib/key-slots.ts`, `src/lib/sync.ts`, `src/lib/sync-fixtures.ts` (modify) | "Only in SSHelter" toggle, the agent problem line |
| `.github/workflows/test-windows.yml` (modify) | Run vault and agent tests on Windows |
| `docs/superpowers/plans/2026-10-07-key-vault-agent-manual-verification.md` (create) | Two-computer checklist for this plan |

---

### Task 1: Vault store

> Implemented as c1e7669 + 9211912. The review changed it: a newer vault file stays in place (version probed first), a vault whose `vault:key` is gone is set aside as `vault.keyless-<ms>.json`, set-aside never overwrites, and the lock tolerates poisoning. The code below is the original text; the commits and the SDD ledger are authoritative.

**Files:**
- Modify: `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock`
- Modify: `src-tauri/src/sync/crypto.rs` (after `open`, before `#[cfg(test)]`)
- Modify: `src-tauri/src/sync/runtime.rs` (`SyncRuntime`)
- Modify: `src-tauri/src/lib.rs` (module list)
- Create: `src-tauri/src/vault/mod.rs`, `src-tauri/src/vault/store.rs`

**Interfaces:**
- Consumes: `crate::sync::env::Keychain` (`get`/`set`/`delete`), `crate::sync::slot_files::write_private(&Path, &[u8]) -> Result<(), AppError>`, `crate::sync::testkit::MemKeychain` (tests), `crate::sync::slot_rules::test_keys` (tests).
- Produces:
  - `crate::sync::crypto::seal_raw(key: &[u8; 32], aad: &[u8], plaintext: &[u8]) -> Result<(String, String), AppError>` (nonce, ciphertext; standard base64)
  - `crate::sync::crypto::open_raw(key: &[u8; 32], aad: &[u8], nonce_b64: &str, ciphertext_b64: &str) -> Result<Vec<u8>, AppError>`
  - `crate::vault::store::{VAULT_FILE, VAULT_KEY_ACCOUNT, DEFAULT_REMEMBER_MINUTES, vault_path, EntryOrigin, VaultEntry, AgentSettings, VaultError, Vault, with_vault}`
  - `Vault::open(path: &Path, keychain: &dyn Keychain, now_ms: u64) -> Result<Vault, VaultError>`
  - `Vault::{get(&self, slot_id) -> Result<Option<VaultEntry>, VaultError>, put(&mut self, keychain, slot_id, &VaultEntry) -> Result<(), VaultError>, remove(&mut self, slot_id) -> Result<bool, VaultError>, ids(&self) -> Vec<String>, settings(&self) -> &AgentSettings, set_settings(&mut self, AgentSettings) -> Result<(), VaultError>}`
  - `with_vault<T>(runtime: &SyncRuntime, path: &Path, keychain: &dyn Keychain, now_ms: u64, f: impl FnOnce(&mut Vault) -> Result<T, VaultError>) -> Result<T, VaultError>` (holds `runtime.vault` while opening and running `f`)
  - `SyncRuntime.vault: Mutex<()>`

- [ ] **Step 1: Add the dependency and the module skeleton**

In `src-tauri/Cargo.toml` `[dependencies]`, after `chacha20poly1305 = "0.10"`, add:

```toml
zeroize = "1"
```

Create `src-tauri/src/vault/mod.rs`:

```rust
//! 金鑰保管庫(key roadmap 第 2 階段 spec §4.1)。`store`:保管庫檔;`material`:私鑰的解析、解密與簽章。
pub mod store;
```

In `src-tauri/src/lib.rs`, add `mod vault;` after `mod updater_channel;` (keep the list's order otherwise unchanged).

In `src-tauri/src/sync/runtime.rs`, add a field to `SyncRuntime` (it derives `Default`, so nothing else changes):

```rust
    /// 保管庫檔(`vault::store`)的寫入互斥:開檔、改動、存檔全程持有(`vault::store::with_vault`)。同步執行緒、命令與 agent 共用。
    pub vault: Mutex<()>,
```

- [ ] **Step 2: Write the failing crypto tests**

Append inside `mod tests` of `src-tauri/src/sync/crypto.rs`:

```rust
    #[test]
    fn raw_seal_round_trips_and_binds_the_aad() {
        let key = [7u8; 32];
        let (nonce, ciphertext) = seal_raw(&key, b"sshelter-vault-v1\nabc", b"secret").unwrap();
        assert_eq!(open_raw(&key, b"sshelter-vault-v1\nabc", &nonce, &ciphertext).unwrap(), b"secret");
        assert!(open_raw(&key, b"sshelter-vault-v1\nxyz", &nonce, &ciphertext).is_err(), "another AAD does not open it");
        assert!(open_raw(&[8u8; 32], b"sshelter-vault-v1\nabc", &nonce, &ciphertext).is_err(), "another key does not open it");
        let (second, _) = seal_raw(&key, b"a", b"secret").unwrap();
        assert_ne!(nonce, second, "every seal draws a fresh nonce");
    }

    #[test]
    fn raw_open_rejects_a_malformed_nonce() {
        let key = [7u8; 32];
        let (_, ciphertext) = seal_raw(&key, b"a", b"x").unwrap();
        assert!(open_raw(&key, b"a", "AAAA", &ciphertext).is_err());
        assert!(open_raw(&key, b"a", "not base64!", &ciphertext).is_err());
    }
```

- [ ] **Step 3: Run them to verify they fail**

Run: `cargo test --offline --lib sync::crypto -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain` (with the PATH prefix)
Expected: compile error `cannot find function seal_raw`.

- [ ] **Step 4: Implement `seal_raw` / `open_raw`**

Add to `src-tauri/src/sync/crypto.rs` after `pub fn open(...)`:

```rust
/// 用一把原始的 32 bytes 金鑰加密(保管庫,key roadmap 第 2 階段 spec §4.1):AAD 由呼叫端給。回傳(nonce, ciphertext),都是 base64。
pub fn seal_raw(key: &[u8; 32], aad: &[u8], plaintext: &[u8]) -> Result<(String, String), AppError> {
    let mut nonce = [0u8; NONCE_LEN];
    getrandom::fill(&mut nonce).map_err(|e| AppError::Other(format!("cannot draw nonce: {e}")))?;
    let cipher = XChaCha20Poly1305::new(Key::from_slice(key));
    let ciphertext = cipher
        .encrypt(XNonce::from_slice(&nonce), Payload { msg: plaintext, aad })
        .map_err(|_| AppError::Other("encryption failed".to_string()))?;
    Ok((B64.encode(nonce), B64.encode(ciphertext)))
}

/// `seal_raw` 的反向。金鑰、AAD 或密文不對都回錯誤(不分辨原因)。
pub fn open_raw(key: &[u8; 32], aad: &[u8], nonce_b64: &str, ciphertext_b64: &str) -> Result<Vec<u8>, AppError> {
    let nonce = B64
        .decode(nonce_b64)
        .map_err(|_| AppError::Other("vault nonce is not valid base64".to_string()))?;
    if nonce.len() != NONCE_LEN {
        return Err(AppError::Other("vault nonce has the wrong length".to_string()));
    }
    let ciphertext = B64
        .decode(ciphertext_b64)
        .map_err(|_| AppError::Other("vault ciphertext is not valid base64".to_string()))?;
    let cipher = XChaCha20Poly1305::new(Key::from_slice(key));
    cipher
        .decrypt(XNonce::from_slice(&nonce), Payload { msg: &ciphertext, aad })
        .map_err(|_| AppError::Other("vault entry cannot be decrypted".to_string()))
}
```

- [ ] **Step 5: Run the crypto tests to verify they pass**

Run: same command as Step 3.
Expected: PASS (all `sync::crypto` tests).

- [ ] **Step 6: Write the failing vault tests**

Create `src-tauri/src/vault/store.rs` with only this test module for now (the implementation comes in Step 8):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::runtime::SyncRuntime;
    use crate::sync::slot_rules::test_keys;
    use crate::sync::testkit::MemKeychain;

    fn entry(text: &str) -> VaultEntry {
        VaultEntry {
            private_key: text.to_string(),
            public_key: test_keys::PLAIN_PUBLIC.to_string(),
            fingerprint: test_keys::PLAIN_FINGERPRINT.to_string(),
            origin: EntryOrigin::Synced,
            added_at_ms: 5,
        }
    }

    #[test]
    fn an_absent_vault_opens_empty_and_writes_nothing() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let vault = Vault::open(&path, &keychain, 1).unwrap();
        assert!(vault.ids().is_empty());
        assert_eq!(vault.settings(), &AgentSettings::default());
        assert!(!path.exists(), "nothing is written until something is stored");
        assert_eq!(keychain.entry(VAULT_KEY_ACCOUNT), None, "no key is drawn until something is stored");
    }

    #[test]
    fn entries_round_trip_and_the_file_never_holds_the_private_key() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let mut vault = Vault::open(&path, &keychain, 1).unwrap();
        vault.put(&keychain, "a".repeat(32).as_str(), &entry(&test_keys::plain())).unwrap();

        let text = std::fs::read_to_string(&path).unwrap();
        assert!(!text.contains(test_keys::PLAIN_BODY[1]), "the file holds no private key text");
        assert!(!text.contains("PRIVATE KEY"));
        assert!(keychain.entry(VAULT_KEY_ACCOUNT).is_some(), "the key lives in the keychain");

        let reopened = Vault::open(&path, &keychain, 2).unwrap();
        assert_eq!(reopened.ids(), vec!["a".repeat(32)]);
        let got = reopened.get(&"a".repeat(32)).unwrap().unwrap();
        assert_eq!(got.private_key, test_keys::plain());
        assert_eq!(got.fingerprint, test_keys::PLAIN_FINGERPRINT);
        assert_eq!(got.origin, EntryOrigin::Synced);
        assert_eq!(reopened.get(&"b".repeat(32)).unwrap(), None);
    }

    #[test]
    fn a_missing_keychain_key_is_reported_and_never_replaced() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let mut vault = Vault::open(&path, &keychain, 1).unwrap();
        vault.put(&keychain, &"a".repeat(32), &entry(&test_keys::plain())).unwrap();

        let empty = MemKeychain::default();
        assert!(matches!(Vault::open(&path, &empty, 2), Err(VaultError::KeyMissing)));
        assert_eq!(empty.entry(VAULT_KEY_ACCOUNT), None, "no new key replaces the lost one");
        assert!(std::fs::read_to_string(&path).unwrap().contains(&"a".repeat(32)), "the file is untouched");
    }

    #[test]
    fn an_entry_copied_under_another_slot_id_does_not_open() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let mut vault = Vault::open(&path, &keychain, 1).unwrap();
        vault.put(&keychain, &"a".repeat(32), &entry(&test_keys::plain())).unwrap();
        let mut file: serde_json::Value = serde_json::from_str(&std::fs::read_to_string(&path).unwrap()).unwrap();
        let sealed = file["entries"][&"a".repeat(32)].clone();
        file["entries"][&"b".repeat(32)] = sealed;
        std::fs::write(&path, serde_json::to_vec(&file).unwrap()).unwrap();

        let reopened = Vault::open(&path, &keychain, 2).unwrap();
        assert!(reopened.get(&"b".repeat(32)).is_err(), "the AAD binds each entry to its slot id");
    }

    #[test]
    fn an_unreadable_file_is_set_aside_and_a_newer_one_is_left_alone() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        std::fs::write(&path, b"{ not json").unwrap();
        match Vault::open(&path, &keychain, 77) {
            Err(VaultError::Unreadable { kept_as: Some(name), .. }) => {
                assert_eq!(name, "vault.unreadable-77.json");
                assert!(dir.path().join(&name).exists());
            }
            other => panic!("expected Unreadable, got {other:?}"),
        }
        assert!(!path.exists());
        assert!(Vault::open(&path, &keychain, 78).unwrap().ids().is_empty(), "the next open starts empty");

        std::fs::write(&path, br#"{"version": 99, "entries": {}}"#).unwrap();
        assert!(matches!(Vault::open(&path, &keychain, 79), Err(VaultError::Newer { version: 99 })));
        assert!(path.exists(), "a file from a newer SSHelter is never moved");
    }

    #[test]
    fn settings_have_defaults_and_persist() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let mut vault = Vault::open(&path, &keychain, 1).unwrap();
        assert_eq!(vault.settings().remember_minutes, DEFAULT_REMEMBER_MINUTES);
        vault.set_settings(AgentSettings { remember_minutes: 60, always_ask: true }).unwrap();
        let reopened = Vault::open(&path, &keychain, 2).unwrap();
        assert_eq!(reopened.settings(), &AgentSettings { remember_minutes: 60, always_ask: true });
    }

    #[test]
    fn remove_drops_an_entry_and_reports_whether_it_was_there() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let mut vault = Vault::open(&path, &keychain, 1).unwrap();
        vault.put(&keychain, &"a".repeat(32), &entry(&test_keys::plain())).unwrap();
        assert!(vault.remove(&"a".repeat(32)).unwrap());
        assert!(!vault.remove(&"a".repeat(32)).unwrap());
        assert!(Vault::open(&path, &keychain, 2).unwrap().ids().is_empty());
    }

    #[test]
    fn with_vault_holds_the_runtime_lock_while_it_runs() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join(VAULT_FILE);
        let keychain = MemKeychain::default();
        let runtime = SyncRuntime::default();
        with_vault(&runtime, &path, &keychain, 1, |vault| {
            assert!(runtime.vault.try_lock().is_err(), "held during the closure");
            vault.put(&keychain, &"a".repeat(32), &entry(&test_keys::plain()))
        })
        .unwrap();
        assert!(runtime.vault.try_lock().is_ok(), "released afterwards");
    }

    #[test]
    fn the_debug_output_hides_the_private_key() {
        let shown = format!("{:?}", entry(&test_keys::plain()));
        assert!(!shown.contains("PRIVATE KEY"));
        assert!(shown.contains(test_keys::PLAIN_FINGERPRINT));
    }
}
```

Add `pub mod store;` is already in `vault/mod.rs`; the module compiles only after Step 8.

- [ ] **Step 7: Run them to verify they fail**

Run: `cargo test --offline --lib vault::store -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`Vault`, `VaultEntry`, … not found).

- [ ] **Step 8: Implement the store**

Put this above the test module in `src-tauri/src/vault/store.rs`:

```rust
//! 金鑰保管庫檔(key roadmap 第 2 階段 spec §4.1):這台電腦持有的私鑰,存在 `sync-state.json` 旁邊的 `vault.json`。每一筆以
//! XChaCha20-Poly1305 個別加密(AAD = `sshelter-vault-v1` + 換行 + 插槽 id),金鑰是 32 bytes 的隨機值,base64 存在系統 keychain 的
//! `vault:key`。私鑰原文原樣保存:有 passphrase 的仍是加密狀態(spec §5.5)。檔頭只有格式版本與這台的 agent 設定(不含祕密)。
//! 寫入一律原子、只有擁有者能讀寫(`slot_files::write_private`;Windows 是只給擁有者的 DACL)。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};

use base64::engine::general_purpose::STANDARD as B64;
use base64::Engine as _;
use serde::{Deserialize, Serialize};
use zeroize::{Zeroize, Zeroizing};

use crate::error::AppError;
use crate::sync::crypto::{open_raw, seal_raw};
use crate::sync::env::Keychain;
use crate::sync::runtime::SyncRuntime;
use crate::sync::slot_files;

pub const VAULT_FILE: &str = "vault.json";
pub const VAULT_KEY_ACCOUNT: &str = "vault:key";
/// 記住核准的預設時間(分鐘;spec §5.3)。
pub const DEFAULT_REMEMBER_MINUTES: u32 = 240;
const VAULT_VERSION: u32 = 1;
const AAD_PREFIX: &str = "sshelter-vault-v1";

/// 保管庫檔的路徑:和 `sync-state.json` 同一個資料夾。
pub fn vault_path(state_path: &Path) -> PathBuf {
    state_path.with_file_name(VAULT_FILE)
}

/// 這筆私鑰從哪裡來(spec §4.1):在這台產生、從檔案匯入、或從帳戶同步來。補寫帳戶裡的 `key`(SP3 §6.6)時,只有同步來的才不必這台的同意。
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum EntryOrigin {
    Generated,
    Imported,
    Synced,
}

/// 保管庫裡的一筆(解密之後)。`private_key` 是 OpenSSH 私鑰的原文;離開作用域時清掉。
#[derive(Clone, PartialEq, Serialize, Deserialize)]
pub struct VaultEntry {
    pub private_key: String,
    pub public_key: String,
    pub fingerprint: String,
    pub origin: EntryOrigin,
    pub added_at_ms: u64,
}

impl std::fmt::Debug for VaultEntry {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("VaultEntry")
            .field("fingerprint", &self.fingerprint)
            .field("origin", &self.origin)
            .finish_non_exhaustive()
    }
}

impl Drop for VaultEntry {
    fn drop(&mut self) {
        self.private_key.zeroize();
    }
}

fn default_remember_minutes() -> u32 {
    DEFAULT_REMEMBER_MINUTES
}

/// 這台電腦的 agent 設定(spec §4.3,只能比每把金鑰的設定更嚴):記住核准多久、這台一律每次都問。
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct AgentSettings {
    #[serde(default = "default_remember_minutes")]
    pub remember_minutes: u32,
    #[serde(default)]
    pub always_ask: bool,
}

impl Default for AgentSettings {
    fn default() -> Self {
        Self { remember_minutes: DEFAULT_REMEMBER_MINUTES, always_ask: false }
    }
}

#[derive(Clone, Debug, Serialize, Deserialize)]
struct SealedEntry {
    nonce: String,
    ciphertext: String,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
struct VaultFile {
    version: u32,
    #[serde(default)]
    settings: AgentSettings,
    #[serde(default)]
    entries: BTreeMap<String, SealedEntry>,
}

impl Default for VaultFile {
    fn default() -> Self {
        Self { version: VAULT_VERSION, settings: AgentSettings::default(), entries: BTreeMap::new() }
    }
}

#[derive(Debug)]
pub enum VaultError {
    /// 保管庫檔裡有金鑰,keychain 裡卻沒有 `vault:key`:不產生新的(會讓既有的每一筆都解不開),保管庫停用(spec §11)。
    KeyMissing,
    /// 讀不懂的保管庫檔:已搬到同一個資料夾的 `kept_as`(搬不動是 None),之後從空的保管庫開始(spec §4.1)。
    Unreadable { kept_as: Option<String>, reason: String },
    /// 更新版 SSHelter 寫的格式:原地不動,保管庫停用。
    Newer { version: u32 },
    Other(AppError),
}

impl std::fmt::Display for VaultError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            VaultError::KeyMissing => write!(f, "the vault's key is missing from the keychain, so its keys can't be opened"),
            VaultError::Unreadable { kept_as: Some(name), reason } => {
                write!(f, "the vault file was unreadable ({reason}); it was kept as {name}")
            }
            VaultError::Unreadable { kept_as: None, reason } => write!(f, "the vault file is unreadable ({reason})"),
            VaultError::Newer { version } => write!(f, "the vault was written by a newer SSHelter (format {version})"),
            VaultError::Other(e) => write!(f, "{e}"),
        }
    }
}

impl From<AppError> for VaultError {
    fn from(e: AppError) -> Self {
        VaultError::Other(e)
    }
}

impl From<VaultError> for AppError {
    fn from(e: VaultError) -> Self {
        match e {
            VaultError::Other(inner) => inner,
            other => AppError::Other(other.to_string()),
        }
    }
}

fn aad(slot_id: &str) -> Vec<u8> {
    format!("{AAD_PREFIX}\n{slot_id}").into_bytes()
}

fn invalid_key() -> VaultError {
    VaultError::Other(AppError::Other("the vault key in the keychain is not valid".to_string()))
}

fn read_key(keychain: &dyn Keychain) -> Result<Option<Zeroizing<[u8; 32]>>, VaultError> {
    let Some(text) = keychain.get(VAULT_KEY_ACCOUNT)? else { return Ok(None) };
    let text = Zeroizing::new(text);
    let bytes = Zeroizing::new(B64.decode(text.as_bytes()).map_err(|_| invalid_key())?);
    let key: [u8; 32] = bytes.as_slice().try_into().map_err(|_| invalid_key())?;
    Ok(Some(Zeroizing::new(key)))
}

/// 讀不懂的檔案搬到同一個資料夾的 `vault.unreadable-<ms>.json`(同 Sync v2 對讀不懂的狀態檔的處理):之後的存檔寫到原路徑,不搬就會蓋掉它。
fn set_aside(path: &Path, now_ms: u64, reason: String) -> VaultError {
    let name = format!("vault.unreadable-{now_ms}.json");
    match std::fs::rename(path, path.with_file_name(&name)) {
        Ok(()) => VaultError::Unreadable { kept_as: Some(name), reason },
        Err(_) => VaultError::Unreadable { kept_as: None, reason },
    }
}

/// 開著的保管庫。改動(`put`、`remove`、`set_settings`)立刻寫回檔案;同一個行程裡的寫入要經 `with_vault` 互斥。
pub struct Vault {
    path: PathBuf,
    key: Option<Zeroizing<[u8; 32]>>,
    file: VaultFile,
}

impl Vault {
    /// 開啟 `path` 的保管庫。檔案不存在 → 空的(什麼都不寫,金鑰也還不產生);有金鑰的檔案一定要 keychain 裡有 `vault:key`。
    pub fn open(path: &Path, keychain: &dyn Keychain, now_ms: u64) -> Result<Vault, VaultError> {
        let bytes = match std::fs::read(path) {
            Ok(bytes) => bytes,
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => {
                return Ok(Vault { path: path.to_path_buf(), key: None, file: VaultFile::default() });
            }
            Err(e) => return Err(VaultError::Other(AppError::Io(e))),
        };
        let file = match serde_json::from_slice::<VaultFile>(&bytes) {
            Ok(file) if file.version > VAULT_VERSION => return Err(VaultError::Newer { version: file.version }),
            Ok(file) => file,
            Err(e) => return Err(set_aside(path, now_ms, e.to_string())),
        };
        let key = read_key(keychain)?;
        if key.is_none() && !file.entries.is_empty() {
            return Err(VaultError::KeyMissing);
        }
        Ok(Vault { path: path.to_path_buf(), key, file })
    }

    /// 保管庫裡的插槽 id(排序過)。
    pub fn ids(&self) -> Vec<String> {
        self.file.entries.keys().cloned().collect()
    }

    pub fn get(&self, slot_id: &str) -> Result<Option<VaultEntry>, VaultError> {
        let Some(sealed) = self.file.entries.get(slot_id) else { return Ok(None) };
        let key = self.key.as_ref().ok_or(VaultError::KeyMissing)?;
        let plaintext = Zeroizing::new(open_raw(key, &aad(slot_id), &sealed.nonce, &sealed.ciphertext)?);
        let entry = serde_json::from_slice::<VaultEntry>(&plaintext)
            .map_err(|e| VaultError::Other(AppError::Other(format!("a vault entry is not readable: {e}"))))?;
        Ok(Some(entry))
    }

    /// 放進(或取代)一筆,立刻存檔。第一次存東西時才產生金鑰並寫進 keychain。
    pub fn put(&mut self, keychain: &dyn Keychain, slot_id: &str, entry: &VaultEntry) -> Result<(), VaultError> {
        let key = self.ensure_key(keychain)?;
        let plaintext = Zeroizing::new(
            serde_json::to_vec(entry).map_err(|e| VaultError::Other(AppError::Other(e.to_string())))?,
        );
        let (nonce, ciphertext) = seal_raw(&key, &aad(slot_id), &plaintext)?;
        self.file.entries.insert(slot_id.to_string(), SealedEntry { nonce, ciphertext });
        self.save()
    }

    /// 拿掉一筆並存檔;回傳原本有沒有。
    pub fn remove(&mut self, slot_id: &str) -> Result<bool, VaultError> {
        if self.file.entries.remove(slot_id).is_none() {
            return Ok(false);
        }
        self.save()?;
        Ok(true)
    }

    pub fn settings(&self) -> &AgentSettings {
        &self.file.settings
    }

    pub fn set_settings(&mut self, settings: AgentSettings) -> Result<(), VaultError> {
        self.file.settings = settings;
        self.save()
    }

    fn ensure_key(&mut self, keychain: &dyn Keychain) -> Result<Zeroizing<[u8; 32]>, VaultError> {
        if let Some(key) = &self.key {
            return Ok(key.clone());
        }
        if let Some(key) = read_key(keychain)? {
            self.key = Some(key.clone());
            return Ok(key);
        }
        if !self.file.entries.is_empty() {
            return Err(VaultError::KeyMissing);
        }
        let mut key = Zeroizing::new([0u8; 32]);
        getrandom::fill(key.as_mut()).map_err(|e| VaultError::Other(AppError::Other(format!("cannot draw the vault key: {e}"))))?;
        let encoded = Zeroizing::new(B64.encode(key.as_ref()));
        keychain.set(VAULT_KEY_ACCOUNT, &encoded)?;
        self.key = Some(key.clone());
        Ok(key)
    }

    fn save(&self) -> Result<(), VaultError> {
        let bytes = serde_json::to_vec_pretty(&self.file).map_err(|e| VaultError::Other(AppError::Other(e.to_string())))?;
        if let Some(dir) = self.path.parent() {
            std::fs::create_dir_all(dir).map_err(|e| VaultError::Other(AppError::Io(e)))?;
        }
        slot_files::write_private(&self.path, &bytes)?;
        Ok(())
    }
}

/// 持有 `runtime.vault` 開啟保管庫並執行 `f`:同一個行程裡改動保管庫的地方都經過這裡,寫入不會互相蓋掉。
pub fn with_vault<T>(
    runtime: &SyncRuntime,
    path: &Path,
    keychain: &dyn Keychain,
    now_ms: u64,
    f: impl FnOnce(&mut Vault) -> Result<T, VaultError>,
) -> Result<T, VaultError> {
    let _guard = runtime.vault.lock().unwrap();
    let mut vault = Vault::open(path, keychain, now_ms)?;
    f(&mut vault)
}
```

- [ ] **Step 9: Run the vault tests to verify they pass**

Run: `cargo test --offline --lib vault::store -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS (9 tests). If `cargo` refuses to resolve `zeroize` offline, run `cargo fetch` once (without `--offline`) and retry.

- [ ] **Step 10: Run the whole suite and commit**

Run the full Rust suite (the Global Constraints command with no filter). Expected: everything passes; the only warning is the pre-existing `set_host_enabled` one.

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/lib.rs src-tauri/src/sync/crypto.rs src-tauri/src/sync/runtime.rs src-tauri/src/vault
git commit -m "feat(vault): add the encrypted key vault file"
```

---

### Task 2: Key material — parse, decrypt and sign

**Files:**
- Modify: `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock`
- Create: `src-tauri/src/vault/material.rs`
- Modify: `src-tauri/src/vault/mod.rs` (`pub mod material;`)
- Modify: `src-tauri/src/sync/slot_rules.rs` (`test_keys`: add an RSA fixture)

**Interfaces:**
- Consumes: `crate::sync::slot_rules::test_keys::{plain, encrypted, ecdsa, PLAIN_PUBLIC, ENC_PUBLIC, ECDSA_PUBLIC}` (tests; the passphrase of `encrypted()` is `test-passphrase`, see its doc comment).
- Produces:
  - `crate::vault::material::{SSH_AGENT_RSA_SHA2_256 = 0x02, SSH_AGENT_RSA_SHA2_512 = 0x04, SUPPORTED_CIPHERS}`
  - `#[derive(Debug, PartialEq, Eq)] pub enum OpenError { NeedsPassphrase, WrongPassphrase, UnsupportedCipher(String), Unreadable }`
  - `pub fn public_key_data(public_key_line: &str) -> Option<ssh_key::public::KeyData>`
  - `pub fn is_encrypted(private_key: &str) -> bool`
  - `pub fn open(private_key: &str, passphrase: Option<&str>) -> Result<Material, OpenError>`
  - `pub struct Material` with `key_data(&self) -> KeyData` and `sign(&self, data: &[u8], flags: u32) -> Result<Vec<u8>, AppError>` (returns the agent's signature blob: `string algorithm, string signature`)
  - `test_keys::{RSA_BODY, RSA_PUBLIC, RSA_FINGERPRINT, rsa()}`

Facts from the crate spike (spec §15), which this task relies on:
- `ssh-key` 0.6.7 with features `crypto` and `encryption` parses OpenSSH private keys and decrypts aes-ctr/cbc/gcm and chacha20-poly1305; legacy PEM and PKCS#8 are unreadable; a wrong passphrase and an unsupported cipher both fail with the same `Error::Crypto`, so the cipher is checked first.
- `ssh-key` 0.6.7's RSA signing is broken (it builds the private key from `[p, p]`): RSA is signed with the `rsa` crate from n, e, d, p, q. Flags: 4 → rsa-sha2-512, 2 → rsa-sha2-256, 0 → ssh-rsa (SHA-1). Ed25519 and ECDSA use `ssh-key`'s own `Signer`.
- `ssh-encoding` must be 0.2 (0.3 belongs to ssh-key 0.7 pre-releases and does not compile with 0.6.7).

- [ ] **Step 1: Add the dependencies**

In `src-tauri/Cargo.toml` `[dependencies]`, change `sha2 = "0.10"` to `sha2 = { version = "0.10", features = ["oid"] }` (`rsa::pkcs1v15::SigningKey<Sha256/Sha512>` needs the digest's OID, as in the spike), and add:

```toml
ssh-key = { version = "0.6.7", features = ["crypto", "encryption"] }
ssh-encoding = "0.2"
rsa = { version = "0.9", features = ["sha2"] }
sha1 = { version = "0.10", features = ["oid"] }
rand_core = { version = "0.6", features = ["getrandom"] }
signature = "2"
```

The crates are in the local registry cache from the crate spike (`agent-crate-spike/Cargo.toml` in the session scratchpad used the same versions, plus ssh-key's `getrandom` feature — add it only if the compiler asks for it). Resolve offline: the first `cargo test --offline …` updates `Cargo.lock`. Do not run a networked `cargo fetch` or `cargo update`; if offline resolution fails, stop and report.

- [ ] **Step 2: Add an RSA test key**

Generate a throwaway RSA key in the session scratchpad (never in `~/.ssh`):

```bash
T=$(mktemp -d) && ssh-keygen -q -t rsa -b 2048 -N '' -C sp3-rsa -f "$T/rsa" && ssh-keygen -l -E sha256 -f "$T/rsa.pub" && cat "$T/rsa" "$T/rsa.pub"
```

In `src-tauri/src/sync/slot_rules.rs` `mod test_keys`, add, following the existing `PLAIN_*` pattern (body lines without the BEGIN/END lines, public key without its comment):

```rust
    /// rsa 2048,沒有 passphrase,comment `sp3-rsa`(金鑰保管庫計畫 Task 2 產生,沒有在任何地方使用)。指紋由 `ssh-keygen -l` 核對過。
    pub const RSA_BODY: &[&str] = &[/* the base64 lines of the private key, one string per line */];
    pub const RSA_PUBLIC: &str = "ssh-rsa AAAA…"; // the .pub line without the trailing comment
    pub const RSA_FINGERPRINT: &str = "SHA256:…"; // from ssh-keygen -l

    pub fn rsa() -> String {
        armor(RSA_BODY)
    }
```

Fill the three values from the command's output (they are generated data, copied verbatim). Then `rm -rf "$T"`.

- [ ] **Step 3: Write the failing tests**

Add `pub mod material;` to `src-tauri/src/vault/mod.rs`. Create `src-tauri/src/vault/material.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::slot_rules::test_keys;
    use ssh_encoding::Decode;

    /// Verify an Ed25519, ECDSA or rsa-sha2 signature blob with ssh-key.
    fn verify(public_line: &str, data: &[u8], blob: &[u8]) {
        use signature::Verifier;
        let key = public_key_data(public_line).unwrap();
        let signature = ssh_key::Signature::decode(&mut &blob[..]).unwrap();
        key.verify(data, &signature).unwrap();
    }

    /// Split a signature blob into (algorithm, raw signature).
    fn split(blob: &[u8]) -> (String, Vec<u8>) {
        let mut r = blob;
        let algorithm = String::decode(&mut r).unwrap();
        let raw = Vec::<u8>::decode(&mut r).unwrap();
        (algorithm, raw)
    }

    #[test]
    fn ed25519_and_ecdsa_sign_and_verify() {
        for (private, public) in [(test_keys::plain(), test_keys::PLAIN_PUBLIC), (test_keys::ecdsa(), test_keys::ECDSA_PUBLIC)] {
            let material = open(&private, None).unwrap();
            assert_eq!(material.key_data(), public_key_data(public).unwrap());
            let blob = material.sign(b"hello", 0).unwrap();
            verify(public, b"hello", &blob);
        }
    }

    #[test]
    fn rsa_signs_with_the_hash_the_flags_ask_for() {
        let material = open(&test_keys::rsa(), None).unwrap();
        let blob = material.sign(b"hello", SSH_AGENT_RSA_SHA2_512).unwrap();
        assert_eq!(split(&blob).0, "rsa-sha2-512");
        verify(test_keys::RSA_PUBLIC, b"hello", &blob);
        let blob = material.sign(b"hello", SSH_AGENT_RSA_SHA2_256).unwrap();
        assert_eq!(split(&blob).0, "rsa-sha2-256");
        verify(test_keys::RSA_PUBLIC, b"hello", &blob);

        // No flag: legacy ssh-rsa (SHA-1), which ssh-key 0.6.7 cannot decode; verify with the rsa crate.
        let (algorithm, raw) = split(&material.sign(b"hello", 0).unwrap());
        assert_eq!(algorithm, "ssh-rsa");
        use rsa::signature::Verifier as _;
        let ssh_key::public::KeyData::Rsa(public) = public_key_data(test_keys::RSA_PUBLIC).unwrap() else { panic!("rsa") };
        let n = rsa::BigUint::try_from(&public.n).unwrap();
        let e = rsa::BigUint::try_from(&public.e).unwrap();
        let verifying = rsa::pkcs1v15::VerifyingKey::<sha1::Sha1>::new(rsa::RsaPublicKey::new(n, e).unwrap());
        verifying.verify(b"hello", &rsa::pkcs1v15::Signature::try_from(raw.as_slice()).unwrap()).unwrap();
    }

    #[test]
    fn an_encrypted_key_needs_the_right_passphrase() {
        let text = test_keys::encrypted();
        assert!(is_encrypted(&text));
        assert!(!is_encrypted(&test_keys::plain()));
        assert_eq!(open(&text, None).err(), Some(OpenError::NeedsPassphrase));
        assert_eq!(open(&text, Some("wrong")).err(), Some(OpenError::WrongPassphrase));
        let material = open(&text, Some("test-passphrase")).unwrap();
        verify(test_keys::ENC_PUBLIC, b"x", &material.sign(b"x", 0).unwrap());
    }

    #[test]
    fn non_openssh_keys_are_unreadable() {
        let pem = "-----BEGIN RSA PRIVATE KEY-----\nMIIBOgIBAAJBAK\n-----END RSA PRIVATE KEY-----\n";
        assert_eq!(open(pem, None).err(), Some(OpenError::Unreadable));
        assert_eq!(open("not a key", None).err(), Some(OpenError::Unreadable));
        assert!(!is_encrypted(pem));
    }

    #[test]
    fn public_key_lines_parse_with_or_without_a_comment() {
        let bare = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        let commented = public_key_data(&format!("{} someone@host\n", test_keys::PLAIN_PUBLIC)).unwrap();
        assert_eq!(bare, commented);
        assert_eq!(public_key_data("garbage"), None);
    }
}
```

If `test_keys::encrypted()`'s passphrase is not `test-passphrase`, use the one its doc comment states.

- [ ] **Step 4: Run them to verify they fail**

Run: `cargo test --offline --lib vault::material -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`open`, `Material`, … not found).

- [ ] **Step 5: Implement**

Put above the tests in `src-tauri/src/vault/material.rs`:

```rust
//! 私鑰的解析、解密與簽章(金鑰保管庫 spec §5.2、§5.5、§15)。只收 OpenSSH 格式。`ssh-key` 0.6.7 的 RSA 簽章有錯(組私鑰時把 `p` 傳了兩次),
//! RSA 改用 `rsa` 套件從 n、e、d、p、q 組私鑰;Ed25519 與 ECDSA 用 `ssh-key` 自己的簽章。解開的私鑰只在記憶體,`ssh-key` 的私密欄位 drop 時清掉。

use rsa::signature::{RandomizedSigner, SignatureEncoding};
use ssh_encoding::Encode;
use ssh_key::private::{KeypairData, RsaKeypair};
use ssh_key::public::KeyData;
use ssh_key::{PrivateKey, PublicKey};

use crate::error::AppError;

pub const SSH_AGENT_RSA_SHA2_256: u32 = 0x02;
pub const SSH_AGENT_RSA_SHA2_512: u32 = 0x04;

/// `ssh-key` 0.6.7 解得開的私鑰加密方式(spike 實測)。不在清單上的(例如 3des-cbc)不必問 passphrase:一定解不開。
pub const SUPPORTED_CIPHERS: &[&str] = &[
    "aes128-ctr",
    "aes192-ctr",
    "aes256-ctr",
    "aes128-cbc",
    "aes192-cbc",
    "aes256-cbc",
    "aes128-gcm@openssh.com",
    "aes256-gcm@openssh.com",
    "chacha20-poly1305@openssh.com",
];

#[derive(Debug, PartialEq, Eq)]
pub enum OpenError {
    /// 有 passphrase,沒有給。
    NeedsPassphrase,
    /// passphrase 不對(加密方式是支援的那幾種,所以錯的一定是 passphrase)。
    WrongPassphrase,
    /// 這種加密方式解不開。
    UnsupportedCipher(String),
    /// 不是 OpenSSH 格式的私鑰,或讀不懂。
    Unreadable,
}

/// 解開的私鑰。
pub struct Material {
    key: PrivateKey,
}

/// 公鑰那一行(`<type> <base64> [comment]`)的 `KeyData`:列出 agent 的金鑰、比對簽章請求用,不必解密私鑰。
pub fn public_key_data(public_key_line: &str) -> Option<KeyData> {
    PublicKey::from_openssh(public_key_line.trim()).ok().map(|key| key.key_data().clone())
}

pub fn is_encrypted(private_key: &str) -> bool {
    PrivateKey::from_openssh(private_key).is_ok_and(|key| key.is_encrypted())
}

/// 解析私鑰原文;有 passphrase 的用 `passphrase` 解開。
pub fn open(private_key: &str, passphrase: Option<&str>) -> Result<Material, OpenError> {
    let key = PrivateKey::from_openssh(private_key).map_err(|_| OpenError::Unreadable)?;
    if !key.is_encrypted() {
        return Ok(Material { key });
    }
    let cipher = key.cipher().as_str().to_string();
    if !SUPPORTED_CIPHERS.contains(&cipher.as_str()) {
        return Err(OpenError::UnsupportedCipher(cipher));
    }
    let Some(passphrase) = passphrase else { return Err(OpenError::NeedsPassphrase) };
    key.decrypt(passphrase).map(|key| Material { key }).map_err(|_| OpenError::WrongPassphrase)
}

fn signing_failed(e: impl std::fmt::Display) -> AppError {
    AppError::Other(format!("signing failed: {e}"))
}

/// `ssh-key` 0.6.7 的轉換用 `[p, p]` 組私鑰,`rsa` 驗證時拒絕;這裡用 `[p, q]`。
fn rsa_private_key(keypair: &RsaKeypair) -> Result<rsa::RsaPrivateKey, AppError> {
    let invalid = || AppError::Other("the RSA key is not valid".to_string());
    let n = rsa::BigUint::try_from(&keypair.public.n).map_err(|_| invalid())?;
    let e = rsa::BigUint::try_from(&keypair.public.e).map_err(|_| invalid())?;
    let d = rsa::BigUint::try_from(&keypair.private.d).map_err(|_| invalid())?;
    let p = rsa::BigUint::try_from(&keypair.private.p).map_err(|_| invalid())?;
    let q = rsa::BigUint::try_from(&keypair.private.q).map_err(|_| invalid())?;
    rsa::RsaPrivateKey::from_components(n, e, d, vec![p, q]).map_err(|_| invalid())
}

/// `string 演算法, string 簽章`。
fn signature_blob(algorithm: &str, raw: &[u8]) -> Result<Vec<u8>, AppError> {
    let mut blob = Vec::new();
    algorithm.encode(&mut blob).map_err(signing_failed)?;
    raw.encode(&mut blob).map_err(signing_failed)?;
    Ok(blob)
}

impl Material {
    pub fn key_data(&self) -> KeyData {
        self.key.public_key().key_data().clone()
    }

    /// 簽章,回傳 agent 協定的 signature blob。RSA 依 flag 選 SHA-512、SHA-256,沒有 flag 用 SHA-1(`ssh-rsa`,同 OpenSSH 的 agent)。
    pub fn sign(&self, data: &[u8], flags: u32) -> Result<Vec<u8>, AppError> {
        match self.key.key_data() {
            KeypairData::Rsa(keypair) => {
                let private = rsa_private_key(keypair)?;
                let mut rng = rand_core::OsRng;
                if flags & SSH_AGENT_RSA_SHA2_512 != 0 {
                    let signature = rsa::pkcs1v15::SigningKey::<sha2::Sha512>::new(private)
                        .try_sign_with_rng(&mut rng, data)
                        .map_err(signing_failed)?;
                    signature_blob("rsa-sha2-512", &signature.to_vec())
                } else if flags & SSH_AGENT_RSA_SHA2_256 != 0 {
                    let signature = rsa::pkcs1v15::SigningKey::<sha2::Sha256>::new(private)
                        .try_sign_with_rng(&mut rng, data)
                        .map_err(signing_failed)?;
                    signature_blob("rsa-sha2-256", &signature.to_vec())
                } else {
                    let signature = rsa::pkcs1v15::SigningKey::<sha1::Sha1>::new(private)
                        .try_sign_with_rng(&mut rng, data)
                        .map_err(signing_failed)?;
                    signature_blob("ssh-rsa", &signature.to_vec())
                }
            }
            _ => {
                use signature::Signer;
                let signature: ssh_key::Signature = self.key.try_sign(data).map_err(signing_failed)?;
                let mut blob = Vec::new();
                signature.encode(&mut blob).map_err(signing_failed)?;
                Ok(blob)
            }
        }
    }
}
```

- [ ] **Step 6: Run the tests to verify they pass**

Run: same command as Step 4.
Expected: PASS (5 tests).

- [ ] **Step 7: Run the full Rust suite and commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/vault src-tauri/src/sync/slot_rules.rs
git commit -m "feat(vault): parse, decrypt and sign with OpenSSH private keys"
```

---

### Task 3: Approval policy and cache

**Files:**
- Create: `src-tauri/src/agent/mod.rs`, `src-tauri/src/agent/approval.rs`
- Modify: `src-tauri/src/lib.rs` (module list)

**Interfaces:**
- Consumes: `crate::vault::store::{AgentSettings, DEFAULT_REMEMBER_MINUTES}` (Task 1).
- Produces:
  - `crate::agent::approval::REMEMBER_CHOICES_MINUTES: [u32; 4]` = `[15, 60, 240, 720]`
  - `crate::agent::approval::remember_minutes(settings: &AgentSettings) -> u32` (falls back to the default for a value not in the choices)
  - `#[derive(Clone, Debug, PartialEq, Eq, Hash)] pub struct ApprovalKey { pub key_fingerprint: String, pub host_fingerprint: String, pub program: String }`
  - `#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)] pub struct KeyProtection { pub ask_every_time: bool, pub require_user_presence: bool }`
  - `#[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum Verdict { Remembered, Ask { rememberable: bool } }`
  - `pub fn verdict(protection: KeyProtection, settings: &AgentSettings, host_known: bool, remembered: bool) -> Verdict`
  - `#[derive(Default)] pub struct ApprovalCache` with `is_remembered(&mut self, key: &ApprovalKey, now_ms: u64) -> bool`, `remember(&mut self, key: ApprovalKey, now_ms: u64, minutes: u32)`, `clear(&mut self)`, `len(&self) -> usize`

- [ ] **Step 1: Write the failing tests**

Create `src-tauri/src/agent/mod.rs`:

```rust
//! SSHelter 的 SSH agent(key roadmap 第 2 階段 spec §5)。
pub mod approval;
```

Add `mod agent;` to `src-tauri/src/lib.rs` right before `mod askpass;`… keep `pub mod askpass;` as it is and insert `mod agent;` as the first line of the module list.

Create `src-tauri/src/agent/approval.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn key(program: &str) -> ApprovalKey {
        ApprovalKey { key_fingerprint: "SHA256:k".into(), host_fingerprint: "SHA256:h".into(), program: program.into() }
    }

    #[test]
    fn a_remembered_approval_skips_the_prompt_only_when_nothing_forbids_remembering() {
        let settings = AgentSettings::default();
        let open = KeyProtection::default();
        assert_eq!(verdict(open, &settings, true, true), Verdict::Remembered);
        assert_eq!(verdict(open, &settings, true, false), Verdict::Ask { rememberable: true });
        let every_time = KeyProtection { ask_every_time: true, require_user_presence: false };
        assert_eq!(verdict(every_time, &settings, true, true), Verdict::Ask { rememberable: false });
        assert_eq!(verdict(open, &settings, false, true), Verdict::Ask { rememberable: false }, "an unknown host is never remembered");
        let strict = AgentSettings { always_ask: true, ..AgentSettings::default() };
        assert_eq!(verdict(open, &strict, true, true), Verdict::Ask { rememberable: false }, "this computer can only be stricter");
    }

    #[test]
    fn user_presence_does_not_change_the_verdict() {
        let presence = KeyProtection { ask_every_time: false, require_user_presence: true };
        assert_eq!(verdict(presence, &AgentSettings::default(), true, true), Verdict::Remembered);
    }

    #[test]
    fn the_cache_expires_and_is_keyed_by_key_host_and_program() {
        let mut cache = ApprovalCache::default();
        cache.remember(key("claude"), 1_000, 15);
        assert!(cache.is_remembered(&key("claude"), 1_000 + 15 * 60_000 - 1));
        assert!(!cache.is_remembered(&key("iterm2"), 1_000), "another program asks again");
        assert!(!cache.is_remembered(&key("claude"), 1_000 + 15 * 60_000), "expired at the boundary");
        assert_eq!(cache.len(), 0, "expired entries are dropped");
    }

    #[test]
    fn clear_forgets_everything() {
        let mut cache = ApprovalCache::default();
        cache.remember(key("a"), 0, 240);
        cache.remember(key("b"), 0, 240);
        cache.clear();
        assert!(!cache.is_remembered(&key("a"), 1));
        assert_eq!(cache.len(), 0);
    }

    #[test]
    fn remember_minutes_only_accepts_the_offered_choices() {
        for minutes in REMEMBER_CHOICES_MINUTES {
            assert_eq!(remember_minutes(&AgentSettings { remember_minutes: minutes, always_ask: false }), minutes);
        }
        assert_eq!(remember_minutes(&AgentSettings { remember_minutes: 7, always_ask: false }), DEFAULT_REMEMBER_MINUTES);
    }
}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib agent::approval -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`ApprovalKey`, `verdict`, … not found).

- [ ] **Step 3: Implement**

Put above the tests in `src-tauri/src/agent/approval.rs`:

```rust
//! agent 的核准規則與記住的核准(spec §5.3)。純邏輯:不碰視窗、檔案與時鐘(`now_ms` 由呼叫端給)。

use std::collections::HashMap;

use crate::vault::store::{AgentSettings, DEFAULT_REMEMBER_MINUTES};

/// 「記住」可選的時間(分鐘)。
pub const REMEMBER_CHOICES_MINUTES: [u32; 4] = [15, 60, 240, 720];

/// 這台設定的記住時間;不在可選的值裡(手改的檔案)就用預設。
pub fn remember_minutes(settings: &AgentSettings) -> u32 {
    if REMEMBER_CHOICES_MINUTES.contains(&settings.remember_minutes) {
        settings.remember_minutes
    } else {
        DEFAULT_REMEMBER_MINUTES
    }
}

/// 記住核准的單位:金鑰 × 主機 × 發出請求的程式(spec §5.3、§5.4)。
#[derive(Clone, Debug, PartialEq, Eq, Hash)]
pub struct ApprovalKey {
    pub key_fingerprint: String,
    pub host_fingerprint: String,
    pub program: String,
}

/// 一把金鑰的保護(跟著金鑰同步,`keyprefs`;Plan 1 一律是預設值)。
#[derive(Clone, Copy, Debug, Default, PartialEq, Eq)]
pub struct KeyProtection {
    pub ask_every_time: bool,
    pub require_user_presence: bool,
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Verdict {
    /// 記住的核准還有效:直接簽。
    Remembered,
    /// 要問;`rememberable` = 視窗可以提供「記住」。
    Ask { rememberable: bool },
}

/// 這次請求要不要問(spec §5.3 的規則 1–3)。「每次都問」的金鑰、這台的「一律每次都問」、未知的主機:一律問,而且不記住。
/// 系統驗證(`require_user_presence`)只在問的時候做,不影響這裡的結果。
pub fn verdict(protection: KeyProtection, settings: &AgentSettings, host_known: bool, remembered: bool) -> Verdict {
    if protection.ask_every_time || settings.always_ask || !host_known {
        return Verdict::Ask { rememberable: false };
    }
    if remembered {
        Verdict::Remembered
    } else {
        Verdict::Ask { rememberable: true }
    }
}

/// 記住的核准(只在記憶體;螢幕鎖定、SSHelter 結束時清除)。值是到期時間(ms)。
#[derive(Default)]
pub struct ApprovalCache {
    entries: HashMap<ApprovalKey, u64>,
}

impl ApprovalCache {
    /// 有沒有還沒到期的核准;順便丟掉所有到期的。
    pub fn is_remembered(&mut self, key: &ApprovalKey, now_ms: u64) -> bool {
        self.entries.retain(|_, expires| *expires > now_ms);
        self.entries.contains_key(key)
    }

    pub fn remember(&mut self, key: ApprovalKey, now_ms: u64, minutes: u32) {
        self.entries.insert(key, now_ms.saturating_add(u64::from(minutes) * 60_000));
    }

    pub fn clear(&mut self) {
        self.entries.clear();
    }

    pub fn len(&self) -> usize {
        self.entries.len()
    }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: same command as Step 2.
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/lib.rs src-tauri/src/agent
git commit -m "feat(agent): add the approval policy and the remembered-approval cache"
```

---

### Task 4: Agent protocol and sessions

> Implemented as 49822f0 + 7b8883d. The review changed it: a userauth blob counts only when its key is the requested key and its service is `ssh-connection`, and every length-prefixed field must be read to its end (`read_prefixed_exact`). The code below is the original text; the commits and the SDD ledger are authoritative.

**Files:**
- Create: `src-tauri/src/agent/protocol.rs`, `src-tauri/src/agent/session.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`pub mod protocol; pub mod session;`)

**Interfaces:**
- Consumes (Task 2): `crate::vault::material::{open, public_key_data}` (tests only).
- Produces:
  - `crate::agent::protocol::{MAX_MESSAGE = 256 * 1024, SSH_AGENT_FAILURE = 5, SSH_AGENT_SUCCESS = 6, SSH_AGENTC_REQUEST_IDENTITIES = 11, SSH_AGENT_IDENTITIES_ANSWER = 12, SSH_AGENTC_SIGN_REQUEST = 13, SSH_AGENT_SIGN_RESPONSE = 14, SSH_AGENTC_EXTENSION = 27, SSH_AGENT_EXTENSION_FAILURE = 28, SESSION_BIND = "session-bind@openssh.com"}`
  - `read_frame(r: &mut impl Read) -> io::Result<Option<Vec<u8>>>` (EOF → `Ok(None)`; length 0 or above `MAX_MESSAGE` → `Err`), `write_frame(w: &mut impl Write, body: &[u8]) -> io::Result<()>`
  - `pub enum Request { Identities, Sign { key: KeyData, data: Vec<u8>, flags: u32 }, SessionBind { host_key: KeyData, session_id: Vec<u8>, signature: ssh_key::Signature, forwarding: bool }, Extension(String), Unsupported }`, `parse_request(msg: &[u8]) -> Request`
  - `identities_answer(keys: &[(KeyData, String)]) -> Vec<u8>`, `sign_response(signature_blob: &[u8]) -> Vec<u8>`
  - `pub struct Userauth { pub session_id: Vec<u8>, pub user: String, pub key: KeyData, pub hostbound_host_key: Option<KeyData> }`, `parse_userauth(data: &[u8]) -> Option<Userauth>`
  - `crate::agent::session::SignRequest { pub key: KeyData, pub data: Vec<u8>, pub flags: u32, pub user: Option<String>, pub host_key: Option<KeyData>, pub forwarded: bool }`
  - `pub trait SignAuthority { fn identities(&self) -> Vec<(KeyData, String)>; fn sign(&self, request: &SignRequest) -> Option<Vec<u8>>; }`
  - `#[derive(Default)] pub struct Session` with `handle(&mut self, authority: &dyn SignAuthority, msg: &[u8]) -> Vec<u8>`
  - `serve(stream: &mut (impl Read + Write), authority: &dyn SignAuthority) -> io::Result<()>`

Rules (spec §5.2, §15):
- Unknown or unreadable messages get `SSH_AGENT_FAILURE` and the connection stays open; a frame of length 0 or above 256 KiB closes it.
- Add/remove/lock/unlock and every extension other than session-bind get `SSH_AGENT_FAILURE`.
- session-bind: verify the server's signature over the session id with the host key; on failure reply 28 and keep nothing; on success remember the host key and session id, and remember forwarding for the rest of the connection.
- A sign request's host is known only when its userauth data's session id equals the last bound session id and, for the hostbound method, its host key equals the bound host key.

- [ ] **Step 1: Write the failing tests**

Add `pub mod protocol; pub mod session;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/session.rs` with the tests first (they exercise both files):

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::agent::protocol::*;
    use crate::sync::slot_rules::test_keys;
    use crate::vault::material::{open, public_key_data};
    use ssh_encoding::{Decode, Encode};
    use std::sync::Mutex;

    struct Fake {
        keys: Vec<(KeyData, String)>,
        seen: Mutex<Vec<SignRequestSummary>>,
        answer: Option<Vec<u8>>,
    }

    #[derive(Clone, Debug, PartialEq)]
    struct SignRequestSummary {
        user: Option<String>,
        host: Option<KeyData>,
        forwarded: bool,
        flags: u32,
    }

    impl SignAuthority for Fake {
        fn identities(&self) -> Vec<(KeyData, String)> {
            self.keys.clone()
        }
        fn sign(&self, r: &SignRequest) -> Option<Vec<u8>> {
            self.seen.lock().unwrap().push(SignRequestSummary { user: r.user.clone(), host: r.host_key.clone(), forwarded: r.forwarded, flags: r.flags });
            self.answer.clone()
        }
    }

    fn fake() -> Fake {
        Fake { keys: vec![(public_key_data(test_keys::PLAIN_PUBLIC).unwrap(), "id_mac".into())], seen: Mutex::new(vec![]), answer: Some(b"sig".to_vec()) }
    }

    fn sign_request(key: &KeyData, data: &[u8], flags: u32) -> Vec<u8> {
        let mut msg = vec![SSH_AGENTC_SIGN_REQUEST];
        key.encode_prefixed(&mut msg).unwrap();
        data.encode(&mut msg).unwrap();
        flags.encode(&mut msg).unwrap();
        msg
    }

    fn userauth(session_id: &[u8], user: &str, key: &KeyData, hostbound: Option<&KeyData>) -> Vec<u8> {
        let mut d = Vec::new();
        session_id.encode(&mut d).unwrap();
        50u8.encode(&mut d).unwrap();
        user.encode(&mut d).unwrap();
        "ssh-connection".encode(&mut d).unwrap();
        (if hostbound.is_some() { "publickey-hostbound-v00@openssh.com" } else { "publickey" }).encode(&mut d).unwrap();
        1u8.encode(&mut d).unwrap();
        "ssh-ed25519".encode(&mut d).unwrap();
        key.encode_prefixed(&mut d).unwrap();
        if let Some(host) = hostbound {
            host.encode_prefixed(&mut d).unwrap();
        }
        d
    }

    /// A session-bind signed by the ECDSA test key acting as the server's host key.
    fn bind(session_id: &[u8], forwarding: bool, tamper: bool) -> (Vec<u8>, KeyData) {
        let host = open(&test_keys::ecdsa(), None).unwrap();
        let mut signed = session_id.to_vec();
        if tamper {
            signed.push(0);
        }
        let blob = host.sign(&signed, 0).unwrap();
        let signature = ssh_key::Signature::decode(&mut &blob[..]).unwrap();
        let mut msg = vec![SSH_AGENTC_EXTENSION];
        SESSION_BIND.encode(&mut msg).unwrap();
        host.key_data().encode_prefixed(&mut msg).unwrap();
        session_id.encode(&mut msg).unwrap();
        signature.encode_prefixed(&mut msg).unwrap();
        (forwarding as u8).encode(&mut msg).unwrap();
        (msg, host.key_data())
    }

    #[test]
    fn identities_are_listed() {
        let mut s = Session::default();
        let reply = s.handle(&fake(), &[SSH_AGENTC_REQUEST_IDENTITIES]);
        assert_eq!(reply[0], SSH_AGENT_IDENTITIES_ANSWER);
        let mut r = &reply[1..];
        assert_eq!(u32::decode(&mut r).unwrap(), 1);
    }

    #[test]
    fn a_bound_session_names_the_host_and_the_user() {
        let authority = fake();
        let mut s = Session::default();
        let (bind_msg, host) = bind(b"session-1", false, false);
        assert_eq!(s.handle(&authority, &bind_msg), vec![SSH_AGENT_SUCCESS]);
        let key = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        let reply = s.handle(&authority, &sign_request(&key, &userauth(b"session-1", "root", &key, Some(&host)), 4));
        assert_eq!(reply[0], SSH_AGENT_SIGN_RESPONSE);
        let seen = authority.seen.lock().unwrap()[0].clone();
        assert_eq!(seen, SignRequestSummary { user: Some("root".into()), host: Some(host), forwarded: false, flags: 4 });
    }

    #[test]
    fn a_session_id_or_host_key_that_does_not_match_the_bind_leaves_the_host_unknown() {
        let authority = fake();
        let mut s = Session::default();
        let (bind_msg, host) = bind(b"session-1", false, false);
        s.handle(&authority, &bind_msg);
        let key = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        s.handle(&authority, &sign_request(&key, &userauth(b"other-session", "root", &key, None), 0));
        let other_host = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        s.handle(&authority, &sign_request(&key, &userauth(b"session-1", "root", &key, Some(&other_host)), 0));
        let seen = authority.seen.lock().unwrap().clone();
        assert_eq!(seen[0].host, None);
        assert_eq!(seen[1].host, None);
        let _ = host;
    }

    #[test]
    fn a_bad_bind_signature_is_an_extension_failure_and_is_not_kept() {
        let authority = fake();
        let mut s = Session::default();
        let (bind_msg, _) = bind(b"session-1", false, true);
        assert_eq!(s.handle(&authority, &bind_msg), vec![SSH_AGENT_EXTENSION_FAILURE]);
        let key = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        s.handle(&authority, &sign_request(&key, &userauth(b"session-1", "root", &key, None), 0));
        assert_eq!(authority.seen.lock().unwrap()[0].host, None);
    }

    #[test]
    fn a_forwarded_bind_marks_the_rest_of_the_connection() {
        let authority = fake();
        let mut s = Session::default();
        s.handle(&authority, &bind(b"hop-1", true, false).0);
        s.handle(&authority, &bind(b"hop-2", false, false).0);
        let key = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        s.handle(&authority, &sign_request(&key, &userauth(b"hop-2", "root", &key, None), 0));
        assert!(authority.seen.lock().unwrap()[0].forwarded);
    }

    #[test]
    fn unsupported_requests_fail_and_a_refusal_is_a_failure() {
        let mut authority = fake();
        let mut s = Session::default();
        assert_eq!(s.handle(&authority, &[99]), vec![SSH_AGENT_FAILURE], "unknown type");
        assert_eq!(s.handle(&authority, &[17, 1, 2, 3]), vec![SSH_AGENT_FAILURE], "add identity is refused");
        assert_eq!(s.handle(&authority, &[22]), vec![SSH_AGENT_FAILURE], "lock is refused");
        let mut other = vec![SSH_AGENTC_EXTENSION];
        "query".encode(&mut other).unwrap();
        assert_eq!(s.handle(&authority, &other), vec![SSH_AGENT_FAILURE], "other extensions");
        authority.answer = None;
        let key = public_key_data(test_keys::PLAIN_PUBLIC).unwrap();
        assert_eq!(s.handle(&authority, &sign_request(&key, b"x", 0)), vec![SSH_AGENT_FAILURE], "refused");
    }

    /// An in-memory duplex for `serve`.
    struct Duplex {
        input: std::io::Cursor<Vec<u8>>,
        output: Vec<u8>,
    }
    impl std::io::Read for Duplex {
        fn read(&mut self, buf: &mut [u8]) -> std::io::Result<usize> {
            self.input.read(buf)
        }
    }
    impl std::io::Write for Duplex {
        fn write(&mut self, buf: &[u8]) -> std::io::Result<usize> {
            self.output.write(buf)
        }
        fn flush(&mut self) -> std::io::Result<()> {
            Ok(())
        }
    }

    fn frame(body: &[u8]) -> Vec<u8> {
        let mut out = (body.len() as u32).to_be_bytes().to_vec();
        out.extend_from_slice(body);
        out
    }

    #[test]
    fn serve_answers_every_frame_and_keeps_going_after_an_unknown_one() {
        let mut input = frame(&[99]);
        input.extend(frame(&[SSH_AGENTC_REQUEST_IDENTITIES]));
        let mut d = Duplex { input: std::io::Cursor::new(input), output: vec![] };
        serve(&mut d, &fake()).unwrap();
        let mut r = &d.output[..];
        assert_eq!(read_frame(&mut r).unwrap().unwrap(), vec![SSH_AGENT_FAILURE]);
        assert_eq!(read_frame(&mut r).unwrap().unwrap()[0], SSH_AGENT_IDENTITIES_ANSWER);
    }

    #[test]
    fn an_oversized_or_empty_frame_closes_the_connection() {
        let mut d = Duplex { input: std::io::Cursor::new(((MAX_MESSAGE + 1) as u32).to_be_bytes().to_vec()), output: vec![] };
        assert!(serve(&mut d, &fake()).is_err());
        let mut d = Duplex { input: std::io::Cursor::new(0u32.to_be_bytes().to_vec()), output: vec![] };
        assert!(serve(&mut d, &fake()).is_err());
    }
}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib agent::session -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors.

- [ ] **Step 3: Implement `protocol.rs`**

```rust
//! agent 協定的訊框與訊息(RFC 9987,加上 OpenSSH 的 `session-bind@openssh.com`;金鑰保管庫 spec §5.2)。只用 `ssh-encoding` 0.2 與 `ssh-key` 0.6.7,
//! 不用 `ssh-agent-lib`(spec §15:它遇到未知訊息會斷線、沒有長度上限、回不了 SHA-1 RSA 簽章)。

use std::io::{self, Read, Write};

use ssh_encoding::{Decode, Encode, Reader};
use ssh_key::public::KeyData;
use ssh_key::Signature;

/// 同 OpenSSH 的 `AGENT_MAX_LEN`。
pub const MAX_MESSAGE: usize = 256 * 1024;
pub const SSH_AGENT_FAILURE: u8 = 5;
pub const SSH_AGENT_SUCCESS: u8 = 6;
pub const SSH_AGENTC_REQUEST_IDENTITIES: u8 = 11;
pub const SSH_AGENT_IDENTITIES_ANSWER: u8 = 12;
pub const SSH_AGENTC_SIGN_REQUEST: u8 = 13;
pub const SSH_AGENT_SIGN_RESPONSE: u8 = 14;
pub const SSH_AGENTC_EXTENSION: u8 = 27;
pub const SSH_AGENT_EXTENSION_FAILURE: u8 = 28;
pub const SESSION_BIND: &str = "session-bind@openssh.com";

/// 讀一個訊框(`uint32` 長度 + 內容)。對方關閉 → `Ok(None)`。長度 0 或超過上限 → `Err`(呼叫端關閉連線)。
pub fn read_frame(r: &mut impl Read) -> io::Result<Option<Vec<u8>>> {
    let mut len = [0u8; 4];
    match r.read_exact(&mut len) {
        Ok(()) => {}
        Err(e) if e.kind() == io::ErrorKind::UnexpectedEof => return Ok(None),
        Err(e) => return Err(e),
    }
    let len = u32::from_be_bytes(len) as usize;
    if len == 0 || len > MAX_MESSAGE {
        return Err(io::Error::new(io::ErrorKind::InvalidData, format!("bad agent frame length {len}")));
    }
    let mut msg = vec![0u8; len];
    r.read_exact(&mut msg)?;
    Ok(Some(msg))
}

pub fn write_frame(w: &mut impl Write, body: &[u8]) -> io::Result<()> {
    let mut out = Vec::with_capacity(4 + body.len());
    out.extend_from_slice(&(body.len() as u32).to_be_bytes());
    out.extend_from_slice(body);
    w.write_all(&out)?;
    w.flush()
}

#[derive(Debug)]
pub enum Request {
    Identities,
    Sign { key: KeyData, data: Vec<u8>, flags: u32 },
    SessionBind { host_key: KeyData, session_id: Vec<u8>, signature: Signature, forwarding: bool },
    /// session-bind 以外的擴充(名稱)。
    Extension(String),
    /// 不支援的種類,或讀不懂的內容。
    Unsupported,
}

pub fn parse_request(msg: &[u8]) -> Request {
    let Some((&kind, mut body)) = msg.split_first() else { return Request::Unsupported };
    let parsed = (|| -> ssh_encoding::Result<Request> {
        Ok(match kind {
            SSH_AGENTC_REQUEST_IDENTITIES => body.finish(Request::Identities)?,
            SSH_AGENTC_SIGN_REQUEST => {
                let key = body.read_prefixed(KeyData::decode)?;
                let data = Vec::<u8>::decode(&mut body)?;
                let flags = u32::decode(&mut body)?;
                body.finish(Request::Sign { key, data, flags })?
            }
            SSH_AGENTC_EXTENSION => {
                let name = String::decode(&mut body)?;
                if name != SESSION_BIND {
                    return Ok(Request::Extension(name));
                }
                let host_key = body.read_prefixed(KeyData::decode)?;
                let session_id = Vec::<u8>::decode(&mut body)?;
                let signature = body.read_prefixed(Signature::decode)?;
                let forwarding = u8::decode(&mut body)? != 0;
                body.finish(Request::SessionBind { host_key, session_id, signature, forwarding })?
            }
            _ => Request::Unsupported,
        })
    })();
    parsed.unwrap_or(Request::Unsupported)
}

pub fn identities_answer(keys: &[(KeyData, String)]) -> Vec<u8> {
    let mut out = vec![SSH_AGENT_IDENTITIES_ANSWER];
    // 寫進 Vec 不會失敗。
    (keys.len() as u32).encode(&mut out).expect("writing to a Vec");
    for (key, comment) in keys {
        key.encode_prefixed(&mut out).expect("writing to a Vec");
        comment.encode(&mut out).expect("writing to a Vec");
    }
    out
}

pub fn sign_response(signature_blob: &[u8]) -> Vec<u8> {
    let mut out = vec![SSH_AGENT_SIGN_RESPONSE];
    signature_blob.encode(&mut out).expect("writing to a Vec");
    out
}

/// 被簽的 userauth 資料(RFC 4252 §7;OpenSSH 的 `publickey-hostbound-v00@openssh.com` 在公鑰之後多一個主機金鑰)。
#[derive(Debug)]
pub struct Userauth {
    pub session_id: Vec<u8>,
    pub user: String,
    pub key: KeyData,
    pub hostbound_host_key: Option<KeyData>,
}

/// 不是 userauth(例如 `ssh-keygen -Y sign` 的 SSHSIG)或讀不懂 → None。多出來的資料一律拒絕(同 OpenSSH 的 agent)。
pub fn parse_userauth(data: &[u8]) -> Option<Userauth> {
    let mut r = data;
    let parsed = (|| -> ssh_encoding::Result<Option<Userauth>> {
        let session_id = Vec::<u8>::decode(&mut r)?;
        if u8::decode(&mut r)? != 50 {
            return Ok(None);
        }
        let user = String::decode(&mut r)?;
        let _service = String::decode(&mut r)?;
        let method = String::decode(&mut r)?;
        let hostbound = match method.as_str() {
            "publickey" => false,
            "publickey-hostbound-v00@openssh.com" => true,
            _ => return Ok(None),
        };
        if u8::decode(&mut r)? != 1 {
            return Ok(None);
        }
        let _algorithm = String::decode(&mut r)?;
        let key = r.read_prefixed(KeyData::decode)?;
        let hostbound_host_key = if hostbound { Some(r.read_prefixed(KeyData::decode)?) } else { None };
        r.finish(Some(Userauth { session_id, user, key, hostbound_host_key }))
    })();
    parsed.ok().flatten()
}
```

- [ ] **Step 4: Implement `session.rs`**

Put above the tests:

```rust
//! 一條 agent 連線(金鑰保管庫 spec §5.2、§5.3):記住這條連線綁定的主機(session-bind)與是否轉送,把每個簽章請求交給 `SignAuthority` 決定。

use std::io::{self, Read, Write};

use ssh_key::public::KeyData;

use crate::agent::protocol::{
    identities_answer, parse_request, parse_userauth, read_frame, sign_response, write_frame, Request, SSH_AGENT_EXTENSION_FAILURE,
    SSH_AGENT_FAILURE, SSH_AGENT_SUCCESS,
};

/// 交給 `SignAuthority` 的一個簽章請求。`host_key` 只在 userauth 資料的 session id 等於這條連線最後一次 bind 的(hostbound 方法的主機金鑰也相同)
/// 時才有;其他情況主機未知。
pub struct SignRequest {
    pub key: KeyData,
    pub data: Vec<u8>,
    pub flags: u32,
    pub user: Option<String>,
    pub host_key: Option<KeyData>,
    /// 這條連線曾經 bind 過轉送(`is_forwarding`):agent 被轉送到遠端了。
    pub forwarded: bool,
}

/// 決定給哪些金鑰、簽不簽(`agent::broker::Connection`)。
pub trait SignAuthority {
    fn identities(&self) -> Vec<(KeyData, String)>;
    /// 要簽就回傳 signature blob(`string 演算法, string 簽章`),不簽回 None。可能等核准視窗(最多 60 秒)。
    fn sign(&self, request: &SignRequest) -> Option<Vec<u8>>;
}

#[derive(Default)]
pub struct Session {
    bound_host: Option<KeyData>,
    session_id: Option<Vec<u8>>,
    forwarded: bool,
}

impl Session {
    /// 處理一則訊息,回傳回應的內容(不含長度)。
    pub fn handle(&mut self, authority: &dyn SignAuthority, msg: &[u8]) -> Vec<u8> {
        match parse_request(msg) {
            Request::Identities => identities_answer(&authority.identities()),
            Request::Sign { key, data, flags } => {
                let userauth = parse_userauth(&data);
                let host_key = match (&userauth, &self.bound_host, &self.session_id) {
                    (Some(u), Some(host), Some(sid))
                        if &u.session_id == sid && u.hostbound_host_key.as_ref().is_none_or(|h| h == host) =>
                    {
                        Some(host.clone())
                    }
                    _ => None,
                };
                let request = SignRequest { key, data, flags, user: userauth.map(|u| u.user), host_key, forwarded: self.forwarded };
                match authority.sign(&request) {
                    Some(blob) => sign_response(&blob),
                    None => vec![SSH_AGENT_FAILURE],
                }
            }
            Request::SessionBind { host_key, session_id, signature, forwarding } => {
                use signature::Verifier;
                if host_key.verify(&session_id, &signature).is_err() {
                    return vec![SSH_AGENT_EXTENSION_FAILURE];
                }
                self.bound_host = Some(host_key);
                self.session_id = Some(session_id);
                self.forwarded |= forwarding;
                vec![SSH_AGENT_SUCCESS]
            }
            Request::Extension(_) | Request::Unsupported => vec![SSH_AGENT_FAILURE],
        }
    }
}

/// 處理一條連線直到對方關閉;訊框不對就斷線(回 Err)。
pub fn serve(stream: &mut (impl Read + Write), authority: &dyn SignAuthority) -> io::Result<()> {
    let mut session = Session::default();
    while let Some(msg) = read_frame(stream)? {
        let reply = session.handle(authority, &msg);
        write_frame(stream, &reply)?;
    }
    Ok(())
}
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: same command as Step 2.
Expected: PASS (8 tests). If `KeyData` lacks `PartialEq` for the test summary, compare fingerprints (`key.fingerprint(ssh_key::HashAlg::Sha256)`) instead.

- [ ] **Step 6: Run the full Rust suite and commit**

```bash
git add src-tauri/src/agent
git commit -m "feat(agent): speak the SSH agent protocol with session-bind"
```

---

### Task 5: Program identification

> Implemented as 05a90ec + f3e3216 + 0b0eb93 + fd87494. The review changed it: macOS reads parents with `PROC_PIDT_SHORTBSDINFO` (the full flavor fails on root-owned `login`), interpreter scripts are read only before the first non-option argument (`interpreted`), login shells are recognised by `argv[0]`, only the needed arguments are kept (`needed_args`), `KERN_PROCARGS2` parsing is bounded (`parse_procargs2`), the walk stops on cycles (`walk`), Linux `(deleted)` suffixes and any-case `.exe` are handled. The code below is the original text; the commits and the SDD ledger are authoritative.

**Files:**
- Modify: `src-tauri/Cargo.toml`, `src-tauri/Cargo.lock` (`libc` for Unix; Windows features)
- Create: `src-tauri/src/agent/peer.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`pub mod peer;`)

**Interfaces:**
- Produces:
  - `#[derive(Clone, Debug, PartialEq, Eq)] pub struct ProcInfo { pub pid: u32, pub ppid: u32, pub path: Option<String>, pub argv: Vec<String> }`
  - `#[derive(Clone, Debug, PartialEq, Eq)] pub struct Program { pub chain: Vec<String>, pub identity: String }` — `chain` is display names, outermost first (`["Claude", "disclaimer", "claude", "zsh", "ssh"]`); `identity` is `"<app path>|<program path>[ + <script>]"`
  - `pub fn identify(chain: &[ProcInfo]) -> Option<Program>` (pure; `chain[0]` is the process connected to the agent, then its parent, …)
  - `pub fn process_chain(pid: u32) -> Vec<ProcInfo>` (macOS: libc; Linux: `/proc`; Windows: Toolhelp + `QueryFullProcessImageNameW`, no argv; other targets: empty)

Rules (spec §5.4 as amended by the spike):
- Walk up from the peer. Stop before system processes: `launchd`, `init`, `systemd`, `explorer`, `services`, `wininit`, `svchost`, `System`, or PID 1/0.
- **Program** = the nearest process whose base name (lowercased, without `.exe`) is not one of `ssh`, `ssh-keygen`, `sh`, `bash`, `zsh`, `fish`, `dash`, `ksh`, `tcsh`, `csh`, `nu`, `pwsh`, `powershell`, `cmd`, `login`, `env`, `sudo`, and does not start with `-` (login shells).
- **App** = the outermost useful process that is a macOS bundle main executable (`…/<X>.app/Contents/MacOS/<exe>` → `X`); if none, the outermost useful process.
- Display name of a process: its bundle name when its path is a bundle main executable, else its file name without `.exe`. A path inside a bundle that is not `Contents/MacOS/<exe>` (for example Xcode's git) shows its file name.
- An interpreter program (`node`, `python`, `python3`, `ruby`, `perl`, `bun`, `deno`, or `python3.*`) adds its script: the first argument after `argv[0]` not starting with `-`; inline code (`-e`, `-c`, `--eval`) adds `<inline>`. The full command line is never kept.
- No useful process (empty chain or only system processes) → `None` ("an unknown program", never remembered).

- [ ] **Step 1: Add the dependencies**

In `src-tauri/Cargo.toml`:

```toml
[target.'cfg(unix)'.dependencies]
libc = "0.2"
```

Add `"Win32_System_Diagnostics_ToolHelp"` to the existing `windows-sys` feature list. (`libc` is already in the lockfile as a dependency of other crates; making it a direct dependency only changes this package's entry in `Cargo.lock`. Features are not recorded in the lockfile.)

- [ ] **Step 2: Write the failing tests**

Add `pub mod peer;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/peer.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn p(pid: u32, ppid: u32, path: &str, argv: &[&str]) -> ProcInfo {
        ProcInfo { pid, ppid, path: Some(path.to_string()), argv: argv.iter().map(|s| s.to_string()).collect() }
    }

    #[test]
    fn claude_code_running_ssh_names_the_app_and_the_program() {
        let chain = vec![
            p(50, 40, "/usr/bin/ssh", &["ssh", "web"]),
            p(40, 30, "/bin/zsh", &["zsh", "-c", "ssh web"]),
            p(30, 20, "/Users/u/Library/Application Support/Claude/claude-code/2.1/x/claude.app/Contents/MacOS/claude", &["claude"]),
            p(20, 10, "/Applications/Claude.app/Contents/Helpers/disclaimer", &["disclaimer"]),
            p(10, 1, "/Applications/Claude.app/Contents/MacOS/Claude", &["Claude"]),
            p(1, 0, "/sbin/launchd", &["launchd"]),
        ];
        let program = identify(&chain).unwrap();
        assert_eq!(program.chain, vec!["Claude", "disclaimer", "claude", "zsh", "ssh"]);
        assert_eq!(
            program.identity,
            "/Applications/Claude.app/Contents/MacOS/Claude|/Users/u/Library/Application Support/Claude/claude-code/2.1/x/claude.app/Contents/MacOS/claude"
        );
    }

    #[test]
    fn git_from_a_terminal_is_git_not_xcode_and_is_told_apart_from_claudes_git() {
        let terminal = vec![
            p(60, 50, "/usr/bin/ssh", &["ssh"]),
            p(50, 40, "/Applications/Xcode.app/Contents/Developer/usr/bin/git", &["git", "fetch"]),
            p(40, 30, "/bin/zsh", &["-zsh"]),
            p(30, 20, "/usr/bin/login", &["login"]),
            p(20, 1, "/System/Applications/Utilities/Terminal.app/Contents/MacOS/Terminal", &["Terminal"]),
            p(1, 0, "/sbin/launchd", &[]),
        ];
        let program = identify(&terminal).unwrap();
        assert_eq!(program.chain, vec!["Terminal", "login", "zsh", "git", "ssh"]);
        assert_eq!(program.identity, "/System/Applications/Utilities/Terminal.app/Contents/MacOS/Terminal|/Applications/Xcode.app/Contents/Developer/usr/bin/git");

        let claude = vec![
            p(60, 50, "/usr/bin/ssh", &["ssh"]),
            p(50, 40, "/Applications/Xcode.app/Contents/Developer/usr/bin/git", &["git", "fetch"]),
            p(40, 10, "/bin/zsh", &["zsh"]),
            p(10, 1, "/Applications/Claude.app/Contents/MacOS/Claude", &["Claude"]),
            p(1, 0, "/sbin/launchd", &[]),
        ];
        assert_ne!(identify(&claude).unwrap().identity, program.identity, "the same git under another app is another program");
    }

    #[test]
    fn interpreters_add_their_script_or_inline() {
        let script = vec![
            p(50, 40, "/usr/bin/ssh", &[]),
            p(40, 10, "/opt/node/bin/node", &["node", "--no-warnings", "/x/tool.js", "--flag"]),
            p(10, 1, "/Applications/iTerm.app/Contents/MacOS/iTerm2", &[]),
            p(1, 0, "/sbin/launchd", &[]),
        ];
        assert_eq!(identify(&script).unwrap().identity, "/Applications/iTerm.app/Contents/MacOS/iTerm2|/opt/node/bin/node + /x/tool.js");
        let inline = vec![
            p(50, 40, "/usr/bin/ssh", &[]),
            p(40, 10, "/usr/bin/python3.12", &["python3", "-c", "import os"]),
            p(10, 1, "/Applications/iTerm.app/Contents/MacOS/iTerm2", &[]),
        ];
        assert_eq!(identify(&inline).unwrap().identity, "/Applications/iTerm.app/Contents/MacOS/iTerm2|/usr/bin/python3.12 + <inline>");
    }

    #[test]
    fn windows_paths_and_system_parents() {
        let chain = vec![
            p(50, 40, r"C:\Windows\System32\OpenSSH\ssh.exe", &[]),
            p(40, 30, r"C:\Program Files\PowerShell\7\pwsh.exe", &[]),
            p(30, 20, r"C:\Program Files\WindowsApps\Microsoft.WindowsTerminal\WindowsTerminal.exe", &[]),
            p(20, 4, r"C:\Windows\explorer.exe", &[]),
        ];
        let program = identify(&chain).unwrap();
        assert_eq!(program.chain, vec!["WindowsTerminal", "pwsh", "ssh"]);
        assert_eq!(
            program.identity,
            r"C:\Program Files\WindowsApps\Microsoft.WindowsTerminal\WindowsTerminal.exe|C:\Program Files\WindowsApps\Microsoft.WindowsTerminal\WindowsTerminal.exe"
        );
    }

    #[test]
    fn nothing_useful_is_unknown() {
        assert_eq!(identify(&[]), None);
        assert_eq!(identify(&[p(1, 0, "/sbin/launchd", &[])]), None);
        assert_eq!(identify(&[ProcInfo { pid: 9, ppid: 1, path: None, argv: vec![] }]), None, "a process we cannot read");
    }

    #[cfg(unix)]
    #[test]
    fn the_current_process_chain_starts_with_this_test_binary() {
        let chain = process_chain(std::process::id());
        assert!(!chain.is_empty());
        assert_eq!(chain[0].pid, std::process::id());
        let exe = std::env::current_exe().unwrap().canonicalize().unwrap();
        let path = std::path::PathBuf::from(chain[0].path.clone().unwrap()).canonicalize().unwrap();
        assert_eq!(path, exe);
        assert!(chain.len() >= 2, "it has a parent");
    }

    #[test]
    fn a_process_that_is_gone_gives_an_empty_chain() {
        assert!(process_chain(u32::MAX - 7).is_empty());
    }
}
```

- [ ] **Step 3: Run them to verify they fail**

Run: `cargo test --offline --lib agent::peer -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors.

- [ ] **Step 4: Implement the pure part**

Put above the tests in `src-tauri/src/agent/peer.rs`:

```rust
//! 發出請求的程式(金鑰保管庫 spec §5.4):從連上 agent 的程序往上找父程序。只是推測:同一使用者的程式可以偽造,所以只用來分組記住核准
//! 與顯示,不當成安全保證。完整的命令列不保存。

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct ProcInfo {
    pub pid: u32,
    pub ppid: u32,
    /// 執行檔的實際路徑(macOS 的 `proc_pidpath`,Linux 的 `/proc/<pid>/exe`,Windows 的 `QueryFullProcessImageNameW`)。
    pub path: Option<String>,
    /// argv(只用來找直譯器的腳本;Windows 不取)。
    pub argv: Vec<String>,
}

#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Program {
    /// 顯示用的名稱,由外到內(最外層的 App 在前,連上 agent 的程序在後)。
    pub chain: Vec<String>,
    /// 記住核准用:`<App 的路徑>|<程式的路徑>[ + <腳本>]`。
    pub identity: String,
}

const SKIP: &[&str] = &[
    "ssh", "ssh-keygen", "sh", "bash", "zsh", "fish", "dash", "ksh", "tcsh", "csh", "nu", "pwsh", "powershell", "cmd", "login", "env", "sudo",
];
const SYSTEM: &[&str] = &["launchd", "init", "systemd", "explorer", "services", "wininit", "svchost", "system"];
const INTERPRETERS: &[&str] = &["node", "python", "python3", "ruby", "perl", "bun", "deno"];

/// 路徑的檔名(`/` 與 `\` 都當分隔),去掉 `.exe`。
fn file_name(path: &str) -> &str {
    let name = path.rsplit(['/', '\\']).next().unwrap_or(path);
    name.strip_suffix(".exe").or_else(|| name.strip_suffix(".EXE")).unwrap_or(name)
}

/// `…/<X>.app/Contents/MacOS/<exe>` → `X`(bundle 的主程式);其他(bundle 裡的其他執行檔)→ None。
fn bundle_name(path: &str) -> Option<&str> {
    let parts: Vec<&str> = path.split('/').collect();
    let n = parts.len();
    (n >= 4 && parts[n - 2] == "MacOS" && parts[n - 3] == "Contents" && parts[n - 4].ends_with(".app"))
        .then(|| parts[n - 4].trim_end_matches(".app"))
}

fn base_lower(p: &ProcInfo) -> Option<String> {
    p.path.as_deref().map(|path| file_name(path).to_ascii_lowercase())
}

fn is_system(p: &ProcInfo) -> bool {
    p.pid <= 1 || base_lower(p).is_some_and(|b| SYSTEM.contains(&b.as_str()))
}

fn is_skipped(p: &ProcInfo) -> bool {
    base_lower(p).is_some_and(|b| SKIP.contains(&b.as_str()) || b.starts_with('-'))
}

fn is_interpreter(base: &str) -> bool {
    INTERPRETERS.contains(&base) || base.starts_with("python3.")
}

fn display(p: &ProcInfo) -> String {
    let path = p.path.as_deref().unwrap_or("?");
    bundle_name(path).map(str::to_string).unwrap_or_else(|| file_name(path).to_string())
}

/// 由程序鏈(`chain[0]` 是連上 agent 的程序,往後是父程序)算出名稱鏈與識別值;認不出來 → None。
pub fn identify(chain: &[ProcInfo]) -> Option<Program> {
    let useful: Vec<&ProcInfo> = chain.iter().take_while(|p| !is_system(p)).filter(|p| p.path.is_some()).collect();
    let program = useful.iter().find(|p| !is_skipped(p))?;
    let app = useful
        .iter()
        .rev()
        .find(|p| p.path.as_deref().and_then(bundle_name).is_some())
        .or_else(|| useful.last())?;
    let program_path = program.path.clone()?;
    let base = file_name(&program_path).to_ascii_lowercase();
    let program_id = if is_interpreter(&base) {
        let inline = program.argv.iter().skip(1).any(|a| a == "-e" || a == "-c" || a == "--eval");
        match program.argv.iter().skip(1).find(|a| !a.starts_with('-')) {
            _ if inline => format!("{program_path} + <inline>"),
            Some(script) => format!("{program_path} + {script}"),
            None => program_path.clone(),
        }
    } else {
        program_path.clone()
    };
    Some(Program {
        chain: useful.iter().rev().map(|p| display(p)).collect(),
        identity: format!("{}|{}", app.path.clone()?, program_id),
    })
}
```

- [ ] **Step 5: Implement `process_chain` per platform**

Add below the pure part:

```rust
/// 從 `pid` 往上找父程序(最多 64 層)。讀不到的程序(已經結束)→ 鏈到那裡為止;一開始就讀不到 → 空的。
pub fn process_chain(pid: u32) -> Vec<ProcInfo> {
    let mut out = Vec::new();
    let mut current = pid;
    while out.len() < 64 {
        let Some(info) = proc_info(current) else { break };
        let parent = info.ppid;
        let stop = current <= 1 || parent == current;
        out.push(info);
        if stop {
            break;
        }
        current = parent;
    }
    out
}

#[cfg(target_os = "macos")]
fn proc_info(pid: u32) -> Option<ProcInfo> {
    let pid = i32::try_from(pid).ok()?;
    // SAFETY: `proc_bsdinfo` is plain old data; `proc_pidinfo` fills at most `size` bytes.
    let info = unsafe {
        let mut info = std::mem::zeroed::<libc::proc_bsdinfo>();
        let size = std::mem::size_of::<libc::proc_bsdinfo>() as libc::c_int;
        let n = libc::proc_pidinfo(pid, libc::PROC_PIDTBSDINFO, 0, &mut info as *mut _ as *mut libc::c_void, size);
        (n == size).then_some(info)?
    };
    let mut buf = vec![0u8; libc::PROC_PIDPATHINFO_MAXSIZE as usize];
    // SAFETY: the buffer is `PROC_PIDPATHINFO_MAXSIZE` bytes long.
    let n = unsafe { libc::proc_pidpath(pid, buf.as_mut_ptr() as *mut libc::c_void, buf.len() as u32) };
    let path = (n > 0).then(|| String::from_utf8_lossy(&buf[..n as usize]).into_owned());
    Some(ProcInfo { pid: pid as u32, ppid: info.pbi_ppid, path, argv: macos_argv(pid).unwrap_or_default() })
}

/// `KERN_PROCARGS2`:`int argc`、執行時給的路徑與 NUL 填充、`argv[0..argc]`、環境變數。
#[cfg(target_os = "macos")]
fn macos_argv(pid: i32) -> Option<Vec<String>> {
    let mut mib = [libc::CTL_KERN, libc::KERN_PROCARGS2, pid];
    let mut size: libc::size_t = 0;
    // SAFETY: sysctl with a null buffer only reports the size; the second call fills at most `size` bytes.
    unsafe {
        if libc::sysctl(mib.as_mut_ptr(), 3, std::ptr::null_mut(), &mut size, std::ptr::null_mut(), 0) != 0 {
            return None;
        }
    }
    let mut buf = vec![0u8; size];
    unsafe {
        if libc::sysctl(mib.as_mut_ptr(), 3, buf.as_mut_ptr() as *mut libc::c_void, &mut size, std::ptr::null_mut(), 0) != 0 {
            return None;
        }
    }
    buf.truncate(size);
    let argc = i32::from_ne_bytes(buf.get(0..4)?.try_into().ok()?);
    let mut rest = &buf[4..];
    let nul = rest.iter().position(|&b| b == 0)?;
    rest = &rest[nul..];
    while rest.first() == Some(&0) {
        rest = &rest[1..];
    }
    let mut argv = Vec::new();
    for _ in 0..argc.max(0) {
        let end = rest.iter().position(|&b| b == 0).unwrap_or(rest.len());
        argv.push(String::from_utf8_lossy(&rest[..end]).into_owned());
        rest = &rest[(end + 1).min(rest.len())..];
    }
    Some(argv)
}

#[cfg(target_os = "linux")]
fn proc_info(pid: u32) -> Option<ProcInfo> {
    let stat = std::fs::read_to_string(format!("/proc/{pid}/stat")).ok()?;
    // `pid (comm) state ppid …`:comm 可能含空白與括號,從最後一個 `)` 之後讀。
    let after = &stat[stat.rfind(')')? + 1..];
    let ppid = after.split_whitespace().nth(1)?.parse().ok()?;
    let path = std::fs::read_link(format!("/proc/{pid}/exe")).ok().map(|p| p.display().to_string());
    let argv = std::fs::read(format!("/proc/{pid}/cmdline"))
        .map(|bytes| bytes.split(|&b| b == 0).filter(|s| !s.is_empty()).map(|s| String::from_utf8_lossy(s).into_owned()).collect())
        .unwrap_or_default();
    Some(ProcInfo { pid, ppid, path, argv })
}

#[cfg(windows)]
fn proc_info(pid: u32) -> Option<ProcInfo> {
    use windows_sys::Win32::Foundation::{CloseHandle, INVALID_HANDLE_VALUE};
    use windows_sys::Win32::System::Diagnostics::ToolHelp::{CreateToolhelp32Snapshot, Process32FirstW, Process32NextW, PROCESSENTRY32W, TH32CS_SNAPPROCESS};
    use windows_sys::Win32::System::Threading::{OpenProcess, QueryFullProcessImageNameW, PROCESS_NAME_WIN32, PROCESS_QUERY_LIMITED_INFORMATION};
    // SAFETY: standard Toolhelp iteration; `dwSize` is set before the first call; every handle is closed.
    let ppid = unsafe {
        let snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if snapshot == INVALID_HANDLE_VALUE {
            return None;
        }
        let mut entry: PROCESSENTRY32W = std::mem::zeroed();
        entry.dwSize = std::mem::size_of::<PROCESSENTRY32W>() as u32;
        let mut found = None;
        let mut more = Process32FirstW(snapshot, &mut entry) != 0;
        while more {
            if entry.th32ProcessID == pid {
                found = Some(entry.th32ParentProcessID);
                break;
            }
            more = Process32NextW(snapshot, &mut entry) != 0;
        }
        CloseHandle(snapshot);
        found?
    };
    // SAFETY: the handle is checked and closed; the buffer length is passed in and updated by the call.
    let path = unsafe {
        let handle = OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, 0, pid);
        if handle.is_null() {
            None
        } else {
            let mut buf = vec![0u16; 32768];
            let mut len = buf.len() as u32;
            let ok = QueryFullProcessImageNameW(handle, PROCESS_NAME_WIN32, buf.as_mut_ptr(), &mut len);
            CloseHandle(handle);
            (ok != 0).then(|| String::from_utf16_lossy(&buf[..len as usize]))
        }
    };
    Some(ProcInfo { pid, ppid, path, argv: Vec::new() })
}

#[cfg(not(any(target_os = "macos", target_os = "linux", windows)))]
fn proc_info(_pid: u32) -> Option<ProcInfo> {
    None
}
```

Windows note: a stale parent PID can name an unrelated new process (Windows does not re-parent). Plan 1 accepts this (identity is a heuristic); the spike's creation-time check is a later refinement.

- [ ] **Step 6: Run the tests to verify they pass; type-check Windows**

Run: same command as Step 3. Expected: PASS (7 tests on macOS).
Then type-check the Windows branch: the whole crate is not checked for Windows from macOS, so build a scratch crate in the session scratchpad that includes the real file with `#[path = ".../src-tauri/src/agent/peer.rs"] mod peer;`, depends on the same `windows-sys` version and features, and run `cargo check --offline --tests --target x86_64-pc-windows-msvc` there (the method SP3 Task 2 and Task 1's fix round used; the target is installed). Prove the check is real once: a deliberate type error under `cfg(windows)` must be reported. Report the result; if the target's std is missing, say so instead of skipping.

- [ ] **Step 7: Run the full Rust suite and commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/agent
git commit -m "feat(agent): name the program that asks for a key"
```

---

### Task 6: Approval prompts and the approval window

**Files:**
- Create: `src-tauri/src/agent/prompt.rs`
- Modify: `src-tauri/src/agent/mod.rs` (module + `AgentRuntime`), `src-tauri/src/state.rs` (`AppState.agent`), `src-tauri/src/lib.rs` (commands)
- Create: `src-tauri/capabilities/approval.json` (the approval window's own, minimal capability)
- Create: `src/lib/agent.ts`, `src/lib/agent.test.ts`, `src/components/AgentApprovalWindow.tsx`, `src/components/AgentApprovalWindow.test.tsx`
- Modify: `src/main.tsx`
- Generated: `src/bindings/AgentApprovalRequest.ts`, `src/bindings/AgentApprovalAnswer.ts` (written by `cargo test`; commit them)

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `crate::agent::prompt::{APPROVAL_WINDOW = "approval", APPROVAL_TIMEOUT = 60 s, APPROVALS_EVENT = "agent://approvals"}`
  - `AgentApprovalRequest { id: String, key_name: String, key_fingerprint: String, program_chain: Vec<String>, user: Option<String>, host: Option<String>, host_fingerprint: Option<String>, rememberable: bool, remember_minutes: u32, needs_passphrase: bool, passphrase_error: Option<String>, preapproved: bool }` (ts-rs exported). `program_chain` is outermost first (`["claude", "zsh", "ssh"]`); empty = unknown.
  - `AgentApprovalAnswer { allow: bool, remember: bool, passphrase: Option<String>, remember_passphrase: bool }` (ts-rs exported; `Debug` hides the passphrase)
  - `trait PromptSurface: Send + Sync { fn changed(&self, pending: &[AgentApprovalRequest]); }`
  - `PromptHub::{ask(&self, surface: &dyn PromptSurface, request: AgentApprovalRequest, timeout: Duration) -> Option<AgentApprovalAnswer>, resolve(&self, id: &str, answer: AgentApprovalAnswer) -> Result<(), AppError>, pending(&self) -> Vec<AgentApprovalRequest>}` (`ask` assigns `id`)
  - `TauriPromptSurface { pub app: tauri::AppHandle }` (opens, updates and destroys the `approval` window)
  - `#[derive(Default)] pub struct AgentRuntime { pub prompts: PromptHub }` in `crate::agent` (later tasks add fields); `AppState.agent: AgentRuntime`
  - Commands `agent_pending() -> Vec<AgentApprovalRequest>`, `agent_resolve(request_id: String, answer: AgentApprovalAnswer) -> Result<(), AppError>`
  - TS: `src/lib/agent.ts` exports `fetchPending`, `resolveApproval`, `onApprovals`, `rememberLabel`, `programName`, `programChainLine`, `destination`, `approvalTitle`

- [ ] **Step 1: Write the failing Rust tests**

Add `pub mod prompt;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/prompt.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;

    #[derive(Default)]
    struct Recorder {
        seen: Mutex<Vec<Vec<String>>>,
    }

    impl PromptSurface for Recorder {
        fn changed(&self, pending: &[AgentApprovalRequest]) {
            self.seen.lock().unwrap().push(pending.iter().map(|r| r.id.clone()).collect());
        }
    }

    fn request(key: &str) -> AgentApprovalRequest {
        AgentApprovalRequest {
            id: String::new(),
            key_name: key.into(),
            key_fingerprint: "SHA256:k".into(),
            program_chain: vec!["claude".into(), "ssh".into()],
            user: Some("root".into()),
            host: Some("web".into()),
            host_fingerprint: Some("SHA256:h".into()),
            rememberable: true,
            remember_minutes: 240,
            needs_passphrase: false,
            passphrase_error: None,
            preapproved: false,
        }
    }

    #[test]
    fn an_answer_reaches_the_waiting_request_and_the_window_is_told_both_times() {
        let hub = Arc::new(PromptHub::default());
        let surface = Arc::new(Recorder::default());
        let (h, s) = (Arc::clone(&hub), Arc::clone(&surface));
        let waiter = std::thread::spawn(move || h.ask(s.as_ref(), request("id_mac"), Duration::from_secs(5)));
        let id = loop {
            if let Some(r) = hub.pending().first() {
                break r.id.clone();
            }
            std::thread::sleep(Duration::from_millis(5));
        };
        hub.resolve(&id, AgentApprovalAnswer { allow: true, remember: true, ..Default::default() }).unwrap();
        let answer = waiter.join().unwrap().unwrap();
        assert!(answer.allow && answer.remember);
        assert!(hub.pending().is_empty());
        let seen = surface.seen.lock().unwrap().clone();
        assert_eq!(seen, vec![vec![id.clone()], vec![]], "shown with the request, then told it is gone");
    }

    #[test]
    fn a_request_nobody_answers_times_out_as_none() {
        let hub = PromptHub::default();
        let surface = Recorder::default();
        assert_eq!(hub.ask(&surface, request("id_mac"), Duration::from_millis(20)), None);
        assert!(hub.pending().is_empty());
    }

    #[test]
    fn resolving_an_unknown_or_answered_request_is_not_found() {
        let hub = Arc::new(PromptHub::default());
        assert!(matches!(hub.resolve("approval-99", AgentApprovalAnswer::default()), Err(AppError::NotFound(_))));
        let h = Arc::clone(&hub);
        let waiter = std::thread::spawn(move || h.ask(&Recorder::default(), request("a"), Duration::from_secs(5)));
        let id = loop {
            if let Some(r) = hub.pending().first() {
                break r.id.clone();
            }
            std::thread::sleep(Duration::from_millis(5));
        };
        hub.resolve(&id, AgentApprovalAnswer { allow: true, ..Default::default() }).unwrap();
        let second = hub.resolve(&id, AgentApprovalAnswer::default());
        assert!(second.is_err() || hub.pending().is_empty(), "a second answer never replaces the first");
        assert!(waiter.join().unwrap().unwrap().allow);
    }

    #[test]
    fn requests_get_distinct_ids_in_arrival_order() {
        let hub = Arc::new(PromptHub::default());
        let mut waiters = Vec::new();
        for key in ["a", "b"] {
            let h = Arc::clone(&hub);
            waiters.push(std::thread::spawn(move || h.ask(&Recorder::default(), request(key), Duration::from_millis(300))));
            while hub.pending().iter().all(|r| r.key_name != key) {
                std::thread::sleep(Duration::from_millis(5));
            }
        }
        let pending = hub.pending();
        assert_eq!(pending.iter().map(|r| r.key_name.as_str()).collect::<Vec<_>>(), vec!["a", "b"]);
        assert_ne!(pending[0].id, pending[1].id);
        for w in waiters {
            assert_eq!(w.join().unwrap(), None);
        }
    }

    #[test]
    fn the_answer_debug_output_hides_the_passphrase() {
        let shown = format!("{:?}", AgentApprovalAnswer { passphrase: Some("hunter2".into()), ..Default::default() });
        assert!(!shown.contains("hunter2"));
    }
}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib agent::prompt -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (types not found).

- [ ] **Step 3: Implement the hub, the surface and the commands**

Put above the tests in `src-tauri/src/agent/prompt.rs`:

```rust
//! 核准視窗與等待中的請求(spec §5.3、§7.4)。agent 的連線執行緒呼叫 `PromptHub::ask` 等待答案(最多 60 秒;伺服器預設 120 秒內
//! 沒完成認證就斷線);核准視窗(標籤 `approval`)用 `agent_pending` 取得請求、用 `agent_resolve` 回答。

use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{mpsc, Mutex};
use std::time::Duration;

use serde::{Deserialize, Serialize};

use crate::error::AppError;

pub const APPROVAL_WINDOW: &str = "approval";
pub const APPROVAL_TIMEOUT: Duration = Duration::from_secs(60);
pub const APPROVALS_EVENT: &str = "agent://approvals";

/// 一個等待回答的請求。顯示用的字串(程式、使用者、主機)來自別的程式或伺服器,前端一律經 `revealHidden` 顯示。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct AgentApprovalRequest {
    pub id: String,
    pub key_name: String,
    pub key_fingerprint: String,
    /// 發出請求的程式,由外到內(例如 `["claude", "zsh", "ssh"]`);空 = 認不出來。
    pub program_chain: Vec<String>,
    pub user: Option<String>,
    /// 顯示用的主機:known_hosts 裡的名稱,找不到名稱就是主機金鑰的指紋;None = 未知的主機(這條連線沒有可信的 session-bind)。
    pub host: Option<String>,
    pub host_fingerprint: Option<String>,
    /// 視窗可以提供「記住」(spec §5.3)。
    pub rememberable: bool,
    pub remember_minutes: u32,
    pub needs_passphrase: bool,
    /// 上一次輸入的 passphrase 不對時的說明。
    pub passphrase_error: Option<String>,
    /// 已經核准,只需要 passphrase:從 SSHelter 按 Connect(spec §5.6),或剛允許了、記住的 passphrase 卻不對、或上一次輸入的不對。
    pub preapproved: bool,
}

/// 視窗的回答。
#[derive(Clone, Default, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct AgentApprovalAnswer {
    pub allow: bool,
    pub remember: bool,
    pub passphrase: Option<String>,
    pub remember_passphrase: bool,
}

impl std::fmt::Debug for AgentApprovalAnswer {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("AgentApprovalAnswer")
            .field("allow", &self.allow)
            .field("remember", &self.remember)
            .field("passphrase", &self.passphrase.as_ref().map(|_| "<hidden>"))
            .field("remember_passphrase", &self.remember_passphrase)
            .finish()
    }
}

/// 等待中的請求變了(新增或回答)時通知畫面:production 打開或更新核准視窗,沒有請求時關掉它(`TauriPromptSurface`)。
pub trait PromptSurface: Send + Sync {
    fn changed(&self, pending: &[AgentApprovalRequest]);
}

struct Waiting {
    request: AgentApprovalRequest,
    answer: mpsc::SyncSender<AgentApprovalAnswer>,
}

/// 等待中的請求(依到達順序)。
#[derive(Default)]
pub struct PromptHub {
    pending: Mutex<Vec<Waiting>>,
    next_id: AtomicU64,
}

impl PromptHub {
    /// 送出請求並等待回答。逾時回 None(呼叫端當成拒絕)。`request.id` 由這裡指定。
    pub fn ask(&self, surface: &dyn PromptSurface, mut request: AgentApprovalRequest, timeout: Duration) -> Option<AgentApprovalAnswer> {
        let id = format!("approval-{}", self.next_id.fetch_add(1, Ordering::Relaxed) + 1);
        request.id = id.clone();
        let (tx, rx) = mpsc::sync_channel(1);
        let shown = {
            let mut pending = self.pending.lock().unwrap();
            pending.push(Waiting { request, answer: tx });
            pending.iter().map(|w| w.request.clone()).collect::<Vec<_>>()
        };
        surface.changed(&shown);
        let answer = rx.recv_timeout(timeout).ok();
        let left = {
            let mut pending = self.pending.lock().unwrap();
            pending.retain(|w| w.request.id != id);
            pending.iter().map(|w| w.request.clone()).collect::<Vec<_>>()
        };
        surface.changed(&left);
        answer
    }

    /// 回答一個等待中的請求。不存在、已經回答過或已逾時 → NotFound。
    pub fn resolve(&self, id: &str, answer: AgentApprovalAnswer) -> Result<(), AppError> {
        let gone = || AppError::NotFound("that request was already answered or has expired".to_string());
        let pending = self.pending.lock().unwrap();
        let waiting = pending.iter().find(|w| w.request.id == id).ok_or_else(gone)?;
        waiting.answer.try_send(answer).map_err(|_| gone())
    }

    pub fn pending(&self) -> Vec<AgentApprovalRequest> {
        self.pending.lock().unwrap().iter().map(|w| w.request.clone()).collect()
    }
}

/// production 的畫面:核准視窗是獨立的小視窗,永遠在最上層;SSHelter 縮在系統匣時也會出現(spec §7.4)。沒有請求時銷毀它
/// (`destroy` 不經過 `CloseRequested`,不會被「關閉視窗時縮到系統匣」攔下)。
pub struct TauriPromptSurface {
    pub app: tauri::AppHandle,
}

impl PromptSurface for TauriPromptSurface {
    fn changed(&self, pending: &[AgentApprovalRequest]) {
        use tauri::{Emitter, Manager, WebviewUrl, WebviewWindowBuilder};
        if pending.is_empty() {
            if let Some(window) = self.app.get_webview_window(APPROVAL_WINDOW) {
                let _ = window.destroy();
            }
            return;
        }
        let window = match self.app.get_webview_window(APPROVAL_WINDOW) {
            Some(window) => Some(window),
            None => WebviewWindowBuilder::new(&self.app, APPROVAL_WINDOW, WebviewUrl::App("index.html".into()))
                .title("SSHelter")
                .inner_size(460.0, 340.0)
                .resizable(false)
                .always_on_top(true)
                .center()
                .build()
                .ok(),
        };
        if let Some(window) = window {
            let _ = window.show();
            let _ = window.unminimize();
            let _ = window.set_focus();
        }
        let _ = self.app.emit(APPROVALS_EVENT, pending);
    }
}

#[tauri::command]
pub fn agent_pending(state: tauri::State<crate::state::AppState>) -> Vec<AgentApprovalRequest> {
    state.agent.prompts.pending()
}

#[tauri::command]
pub fn agent_resolve(
    state: tauri::State<crate::state::AppState>,
    request_id: String,
    answer: AgentApprovalAnswer,
) -> Result<(), AppError> {
    state.agent.prompts.resolve(&request_id, answer)
}
```

In `src-tauri/src/agent/mod.rs` (which already has `pub mod prompt;` from Step 1), add:

```rust
/// agent 的執行期狀態(`AppState::agent`)。
#[derive(Default)]
pub struct AgentRuntime {
    pub prompts: prompt::PromptHub,
}
```

In `src-tauri/src/state.rs`, add the field `pub agent: crate::agent::AgentRuntime,` to `AppState` (after `sync`), and `agent: crate::agent::AgentRuntime::default(),` to its `Default` impl.

In `src-tauri/src/lib.rs`, add `use agent::prompt::{agent_pending, agent_resolve};` next to the other command imports and add `agent_pending, agent_resolve,` to the `tauri::generate_handler![...]` list.

Create `src-tauri/capabilities/approval.json` (every file in `capabilities/` is loaded; `tauri.conf.json` lists none). The approval window only calls the two app commands above and listens for `agent://approvals`, so it gets `core:default` and none of the main window's updater, autostart, shortcut, dialog or clipboard permissions:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "approval",
  "description": "The SSH agent's approval window: it lists and answers pending requests (app commands) and listens for their updates. Nothing else.",
  "windows": ["approval"],
  "permissions": ["core:default"]
}
```

- [ ] **Step 4: Run the Rust tests to verify they pass, and generate the bindings**

Run: `cargo test --offline --lib agent::prompt -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS (5 tests); `src/bindings/AgentApprovalRequest.ts` and `src/bindings/AgentApprovalAnswer.ts` now exist.

- [ ] **Step 5: Write the failing frontend tests**

Create `src/lib/agent.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import type { AgentApprovalRequest } from "@/bindings/AgentApprovalRequest";
import { approvalTitle, destination, programChainLine, programName, rememberLabel } from "@/lib/agent";

function request(over: Partial<AgentApprovalRequest> = {}): AgentApprovalRequest {
  return {
    id: "approval-1",
    key_name: "id_mac",
    key_fingerprint: "SHA256:k",
    program_chain: ["claude", "zsh", "ssh"],
    user: "root",
    host: "web",
    host_fingerprint: "SHA256:h",
    rememberable: true,
    remember_minutes: 240,
    needs_passphrase: false,
    passphrase_error: null,
    preapproved: false,
    ...over,
  };
}

describe("approval text", () => {
  it("names the program, the key and the destination", () => {
    expect(approvalTitle(request())).toBe("Allow claude to use id_mac?");
    expect(programChainLine(request())).toBe("claude → zsh → ssh");
    expect(destination(request())).toBe("root@web");
  });

  it("says when the program or the host is unknown", () => {
    expect(programName(request({ program_chain: [] }))).toBe("an unknown program");
    expect(programChainLine(request({ program_chain: ["ssh"] }))).toBeNull();
    expect(destination(request({ host: null }))).toBe("root@an unknown host");
    expect(destination(request({ user: null, host: null }))).toBe("an unknown host");
  });

  it("shows hidden characters from other programs instead of rendering them", () => {
    expect(destination(request({ user: "ro\u202eot" }))).toBe("ro⟨U+202E⟩ot@web");
    expect(approvalTitle(request({ program_chain: ["cl\u0007aude"] }))).toContain("⟨U+0007⟩");
  });

  it("titles a Connect unlock differently", () => {
    expect(approvalTitle(request({ preapproved: true }))).toBe("Unlock id_mac to connect to root@web");
  });

  it("labels each remember choice", () => {
    expect(rememberLabel(15)).toBe("Remember for 15 minutes");
    expect(rememberLabel(60)).toBe("Remember for 1 hour");
    expect(rememberLabel(240)).toBe("Remember for 4 hours");
    expect(rememberLabel(720)).toBe("Remember for 12 hours");
  });
});
```

Create `src/components/AgentApprovalWindow.test.tsx`:

```tsx
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";

import type { AgentApprovalRequest } from "@/bindings/AgentApprovalRequest";
import { ApprovalCard } from "@/components/AgentApprovalWindow";

function request(over: Partial<AgentApprovalRequest> = {}): AgentApprovalRequest {
  return {
    id: "approval-1",
    key_name: "id_mac",
    key_fingerprint: "SHA256:k",
    program_chain: ["claude", "zsh", "ssh"],
    user: "root",
    host: "web",
    host_fingerprint: "SHA256:h",
    rememberable: true,
    remember_minutes: 240,
    needs_passphrase: false,
    passphrase_error: null,
    preapproved: false,
    ...over,
  };
}

const render = (r: AgentApprovalRequest) => renderToStaticMarkup(<ApprovalCard request={r} busy={false} onAnswer={() => {}} />);

describe("ApprovalCard", () => {
  it("asks with Deny and Allow and offers to remember", () => {
    const html = render(request());
    expect(html).toContain("Allow claude to use id_mac?");
    expect(html).toContain("claude → zsh → ssh");
    expect(html).toContain("root@web");
    expect(html).toContain("Remember for 4 hours");
    expect(html).toContain(">Deny<");
    expect(html).toContain(">Allow<");
    expect(html).not.toContain("Passphrase");
  });

  it("does not offer to remember when the request cannot be remembered", () => {
    expect(render(request({ rememberable: false }))).not.toContain("Remember for");
  });

  it("asks for the passphrase and shows the previous error", () => {
    const html = render(request({ needs_passphrase: true, passphrase_error: "That passphrase didn't work." }));
    expect(html).toContain('aria-label="Passphrase"');
    expect(html).toContain("Remember on this computer");
    expect(html).toContain("That passphrase didn&#x27;t work.");
  });

  it("unlocks for Connect with Cancel and Unlock", () => {
    const html = render(request({ preapproved: true, needs_passphrase: true }));
    expect(html).toContain("Unlock id_mac to connect to root@web");
    expect(html).toContain(">Cancel<");
    expect(html).toContain(">Unlock<");
    expect(html).not.toContain("Remember for");
  });

  it("escapes hidden characters in names", () => {
    expect(render(request({ host: "we\u202eb" }))).toContain("we⟨U+202E⟩b");
  });
});
```

- [ ] **Step 6: Run them to verify they fail**

Run: `./node_modules/.bin/vitest run src/lib/agent.test.ts src/components/AgentApprovalWindow.test.tsx`
Expected: FAIL (modules not found).

- [ ] **Step 7: Implement the helpers, the window and the entry switch**

Create `src/lib/agent.ts`:

```ts
import { listen, type UnlistenFn } from "@tauri-apps/api/event";

import type { AgentApprovalAnswer } from "@/bindings/AgentApprovalAnswer";
import type { AgentApprovalRequest } from "@/bindings/AgentApprovalRequest";
import { tauriInvoke } from "@/lib/ipc";
import { revealHidden } from "@/lib/sync-approvals";

/**
 * SSHelter's SSH agent (key vault spec §5.3, §7.4): the approval window's requests and answers. Names come from other
 * programs and servers, so every one of them is shown through `revealHidden`.
 */

export const APPROVALS_EVENT = "agent://approvals";

export function fetchPending(): Promise<AgentApprovalRequest[]> {
  return tauriInvoke<AgentApprovalRequest[]>("agent_pending");
}

export function resolveApproval(requestId: string, answer: AgentApprovalAnswer): Promise<void> {
  return tauriInvoke<void>("agent_resolve", { requestId, answer });
}

export function onApprovals(handler: (pending: AgentApprovalRequest[]) => void): Promise<UnlistenFn> {
  return listen<AgentApprovalRequest[]>(APPROVALS_EVENT, (event) => handler(event.payload));
}

export function rememberLabel(minutes: number): string {
  switch (minutes) {
    case 15:
      return "Remember for 15 minutes";
    case 60:
      return "Remember for 1 hour";
    case 720:
      return "Remember for 12 hours";
    default:
      return "Remember for 4 hours";
  }
}

export function programName(r: AgentApprovalRequest): string {
  return r.program_chain.length > 0 ? revealHidden(r.program_chain[0]) : "an unknown program";
}

/** "claude → zsh → ssh"; null when there is nothing beyond the program itself. */
export function programChainLine(r: AgentApprovalRequest): string | null {
  return r.program_chain.length > 1 ? r.program_chain.map(revealHidden).join(" → ") : null;
}

export function destination(r: AgentApprovalRequest): string {
  const host = r.host === null ? "an unknown host" : revealHidden(r.host);
  return r.user === null ? host : `${revealHidden(r.user)}@${host}`;
}

export function approvalTitle(r: AgentApprovalRequest): string {
  const key = revealHidden(r.key_name);
  return r.preapproved ? `Unlock ${key} to connect to ${destination(r)}` : `Allow ${programName(r)} to use ${key}?`;
}
```

Create `src/components/AgentApprovalWindow.tsx`:

```tsx
import { useEffect, useState } from "react";

import type { AgentApprovalAnswer } from "@/bindings/AgentApprovalAnswer";
import type { AgentApprovalRequest } from "@/bindings/AgentApprovalRequest";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Input } from "@/components/ui/input";
import { approvalTitle, destination, fetchPending, onApprovals, programChainLine, rememberLabel, resolveApproval } from "@/lib/agent";
import { isImeKey } from "@/lib/ime";

/** One request: what asks, for which key and destination, and the answer. Exported for the markup tests. */
export function ApprovalCard({
  request,
  busy,
  onAnswer,
}: {
  request: AgentApprovalRequest;
  busy: boolean;
  onAnswer: (answer: AgentApprovalAnswer) => void;
}) {
  const [remember, setRemember] = useState(request.rememberable);
  const [passphrase, setPassphrase] = useState("");
  const [rememberPassphrase, setRememberPassphrase] = useState(false);
  const allowDisabled = busy || (request.needs_passphrase && passphrase.length === 0);
  const chain = programChainLine(request);
  const answer = (allow: boolean) =>
    onAnswer({
      allow,
      remember: allow && request.rememberable && !request.preapproved && remember,
      passphrase: allow && request.needs_passphrase ? passphrase : null,
      remember_passphrase: allow && request.needs_passphrase && rememberPassphrase,
    });
  return (
    <div className="space-y-3 p-4">
      <h1 className="text-sm font-semibold break-words">{approvalTitle(request)}</h1>
      {chain && <p className="text-xs text-muted-foreground break-all">{chain}</p>}
      {!request.preapproved && <p className="text-sm break-all">{destination(request)}</p>}
      <p className="font-mono text-xs text-muted-foreground break-all">{request.key_fingerprint}</p>
      {request.needs_passphrase && (
        <div className="space-y-2">
          <Input
            type="password"
            autoFocus
            value={passphrase}
            aria-label="Passphrase"
            placeholder="Passphrase"
            onChange={(e) => setPassphrase(e.target.value)}
            onKeyDown={(e) => {
              if (isImeKey(e)) return;
              if (e.key === "Enter" && !allowDisabled) answer(true);
            }}
          />
          {request.passphrase_error && <p className="text-xs text-destructive">{request.passphrase_error}</p>}
          <label className="flex items-center gap-2 text-xs">
            <Checkbox checked={rememberPassphrase} onCheckedChange={(v) => setRememberPassphrase(v === true)} />
            Remember on this computer
          </label>
        </div>
      )}
      {request.rememberable && !request.preapproved && (
        <label className="flex items-center gap-2 text-xs">
          <Checkbox checked={remember} onCheckedChange={(v) => setRemember(v === true)} />
          {rememberLabel(request.remember_minutes)}
        </label>
      )}
      <div className="flex justify-end gap-2">
        <Button type="button" variant="outline" size="sm" disabled={busy} onClick={() => answer(false)}>
          {request.preapproved ? "Cancel" : "Deny"}
        </Button>
        <Button type="button" size="sm" disabled={allowDisabled} onClick={() => answer(true)}>
          {request.preapproved ? "Unlock" : "Allow"}
        </Button>
      </div>
    </div>
  );
}

/** The `approval` window (spec §7.4): shows the oldest waiting request; Rust opens and destroys the window. */
export default function AgentApprovalWindow() {
  const [pending, setPending] = useState<AgentApprovalRequest[]>([]);
  const [busy, setBusy] = useState(false);
  useEffect(() => {
    let unlisten: (() => void) | undefined;
    let live = true;
    void fetchPending().then((list) => live && setPending(list));
    void onApprovals((list) => setPending(list)).then((u) => {
      if (live) unlisten = u;
      else u();
    });
    return () => {
      live = false;
      unlisten?.();
    };
  }, []);
  const request = pending[0];
  if (!request) return null;
  return (
    <ApprovalCard
      key={request.id}
      request={request}
      busy={busy}
      onAnswer={(answer) => {
        setBusy(true);
        void resolveApproval(request.id, answer).finally(() => setBusy(false));
      }}
    />
  );
}
```

In `src/main.tsx`, render the approval window when this webview is the `approval` window:

```tsx
import React from "react";
import ReactDOM from "react-dom/client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { getCurrentWindow } from "@tauri-apps/api/window";
import App from "./App";
import AgentApprovalWindow from "./components/AgentApprovalWindow";
import "./index.css";

const queryClient = new QueryClient();

const rootEl = document.getElementById("root");
if (!rootEl) throw new Error("Root element #root not found in index.html");

// The agent's approval prompt (key vault spec §7.4) is a second window that loads the same page.
const isApprovalWindow = getCurrentWindow().label === "approval";

ReactDOM.createRoot(rootEl).render(
  <React.StrictMode>
    <QueryClientProvider client={queryClient}>{isApprovalWindow ? <AgentApprovalWindow /> : <App />}</QueryClientProvider>
  </React.StrictMode>,
);
```

Check `src/components/ui/checkbox.tsx` exports `Checkbox` with `checked` / `onCheckedChange` (it is the shadcn/Radix checkbox); if its API differs, use it the way `SettingsDialog.tsx` or another existing caller does.

- [ ] **Step 8: Run the frontend checks**

Run: `./node_modules/.bin/vitest run src/lib/agent.test.ts src/components/AgentApprovalWindow.test.tsx`, then `./node_modules/.bin/tsc --noEmit` and `./node_modules/.bin/vitest run`.
Expected: all pass.

- [ ] **Step 9: Run the full Rust suite and commit**

```bash
git add src-tauri/src/agent src-tauri/src/state.rs src-tauri/src/lib.rs src-tauri/capabilities/approval.json src/lib/agent.ts src/lib/agent.test.ts src/components/AgentApprovalWindow.tsx src/components/AgentApprovalWindow.test.tsx src/main.tsx src/bindings/AgentApprovalRequest.ts src/bindings/AgentApprovalAnswer.ts
git commit -m "feat(agent): add the approval prompt hub and the approval window"
```

---

### Task 7: Vault delivery for key slots

> Ruled during implementation (see the SDD ledger): `use_synced` on a vault slot is refused with `VAULT_FIRST_MESSAGE` like `pick` (the vault key may be the last copy), and its test is a refusal; `set_delivery(false)` commits the record before removing the vault entry; `republish`'s vault arm also checks the vault text's own fingerprint. The text below predates these rulings.

**Files:**
- Modify: `src-tauri/src/sync/state_v2.rs` (`SlotSource`)
- Modify: `src-tauri/src/sync/slots.rs` (every `SlotSource` match, `reconcile`, `republish`, `set_mode`, `pick`, `use_synced`, `delete_copy`, `views`, new `set_delivery`, `vault_slot_files`, `VaultKeys`)
- Modify: `src-tauri/src/sync/slot_setup.rs` (`kept_key`, `held_in_place`)
- Modify: `src-tauri/src/sync/round.rs` (the reconcile call at step 6b)
- Modify: `src-tauri/src/sync/dto.rs` (`SyncKeySlotView.in_vault`)
- Modify: `src-tauri/src/sync/engine.rs`, `src-tauri/src/lib.rs` (command `sync_key_set_delivery`)
- Modify: `src-tauri/src/vault/store.rs` (`stored_ids`)
- Generated: `src/bindings/SyncKeySlotView.ts`; update `src/lib/sync-fixtures.ts` so every `SyncKeySlotView` fixture has `in_vault: false` (tsc will point at them)

**Interfaces:**
- Consumes (Task 1): `crate::vault::store::{vault_path, with_vault, EntryOrigin, VaultEntry, VaultError}`; `Vault::open` sets an unreadable or keyless vault file aside (`vault.unreadable-<ms>.json`, `vault.keyless-<ms>.json`) and starts empty (Task 1 fix round 1).
- Produces:
  - `SlotSource::Vault { fingerprint: String, public_key: String, has_passphrase: bool }`
  - `pub trait VaultKeys { fn private_key(&self, slot_id: &str) -> Option<(zeroize::Zeroizing<String>, EntryOrigin)>; fn holds(&self, slot_id: &str) -> Option<bool>; fn restore(&self, slot_id: &str, entry: &VaultEntry) -> Result<(), AppError>; }`, `pub struct NoVault;`, `pub struct EnvVault<'a, 'b> { pub env: &'a SyncEnv<'b> }` (all in `crate::sync::slots`)
  - `crate::vault::store::stored_ids(path: &Path) -> Result<BTreeSet<String>, VaultError>` (no keychain, never moves the file)
  - `pub const VAULT_ENTRY_LOST: &str = "This key was lost from SSHelter's vault. Pick it again on this computer.";`
  - `pub fn reconcile_with_vault(state: &mut SyncStateV2, account_keys: &ChainKeys, home: &Path, in_use: &BTreeMap<String, Vec<String>>, now_ms: u64, vault: &dyn VaultKeys) -> SlotRound` (`reconcile` keeps its signature and passes `&NoVault`)
  - `pub fn vault_slot_files(state: &SyncStateV2) -> BTreeSet<String>`
  - `pub fn set_delivery(env: &SyncEnv, slot_id: &str, vault: bool) -> Result<(), AppError>`
  - `pub const VAULT_FIRST_MESSAGE: &str = "This key is only in SSHelter. Choose Keep a file first.";`
  - `SyncKeySlotView.in_vault: bool`
  - Command `sync_key_set_delivery(slot_id: String, vault: bool) -> Result<SyncOverview, AppError>`

Semantics (spec §4.3, §4.4, SP3 §6.6):
- "Only in SSHelter" = `SlotSource::Vault`: the private key is in `vault.json`, the slot directory holds only `<file>.pub`, nothing at `<file>`.
- Moving in: a `SyncedCopy` becomes an entry with origin `Synced` (or `Imported` when `copy_from_another_account` is set); a `Linked` key becomes `Imported`, and only SSHelter's link at the slot path goes away — the user's original file is never touched; a hard link or copy that may be the key's last name is retired (`retire_key`), not deleted.
- Moving out writes a private copy at the slot path from the vault entry, then removes the entry; the record becomes `SyncedCopy`, and `copy_from_another_account` becomes true unless the entry came from this account (`Synced`), so the SP3 consent rule still guards it.
- Rounds never land a file into a `Vault` slot; they only keep its `.pub` and report a file that appears at the slot path as in the way.
- `republish` restores a vault key's `key` only with this computer's consent (`uploaded_fingerprint`) or when the entry came from this account (`Synced` and not `copy_from_another_account`).
- A lost vault entry (spec §11: the vault file was unreadable or its `vault:key` left the keychain, so `Vault::open` set it aside): each round checks every `Vault` slot with `holds` (reads only the entry ids, never the keychain). When the vault is readable and lacks the entry, the account's `key` for that slot with the same fingerprint is put back (origin `Synced`, or `Imported` when `copy_from_another_account`); with no such key the slot drops to "no key yet" (`source = None`, `last_error = VAULT_ENTRY_LOST`) and SP3's usual flow lands the synced key or asks the user to pick one. A vault that cannot be read (`holds` is `None`) changes nothing.

- [ ] **Step 1: Write the failing tests**

Add to the `tests` module of `src-tauri/src/sync/slots.rs` (reuse the existing helpers `pair`, `create_slot_on`, `use_slot`, `settle`, `home`, `account_keys`, `record_key`, `key_secret_key`, `NO_OTHER_HOSTS`, `view_of`):

```rust
    // ── 只在 SSHelter 的插槽(金鑰保管庫 spec §4.3、§4.4)────────────────────────────────────────────────

    fn vault_entry(d: &TestDevice, id: &str) -> Option<crate::vault::store::VaultEntry> {
        let env = d.env();
        let path = crate::vault::store::vault_path(&env.state_path);
        crate::vault::store::with_vault(env.runtime, &path, env.keychain, 1, |v| v.get(id)).unwrap()
    }

    #[test]
    fn a_synced_copy_moves_into_the_vault_and_back_to_a_file() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        let slot = home(&b).join(SLOT_DIR).join(&file);
        assert_eq!(std::fs::read_to_string(&slot).unwrap(), test_keys::plain());

        set_delivery(&b.env(), &id, true).unwrap();
        assert!(!slot.exists(), "no private key file is left in the slot");
        assert_eq!(std::fs::read_to_string(public_path(&slot)).unwrap().trim(), test_keys::PLAIN_PUBLIC);
        let entry = vault_entry(&b, &id).expect("the key is in the vault");
        assert_eq!(entry.private_key, test_keys::plain());
        assert_eq!(entry.origin, crate::vault::store::EntryOrigin::Synced);
        assert!(matches!(&b.state().key_slots[&id].source, Some(SlotSource::Vault { fingerprint, .. }) if fingerprint == test_keys::PLAIN_FINGERPRINT));
        assert_eq!(vault_slot_files(&b.state()), BTreeSet::from([file.clone()]));
        assert!(view_of(&b).remove(0).in_vault);

        settle(&b);
        assert!(!slot.exists(), "a round never lands a file into a vault slot");
        assert!(matches!(b.state().key_slots[&id].source, Some(SlotSource::Vault { .. })));

        set_delivery(&b.env(), &id, false).unwrap();
        assert_eq!(std::fs::read_to_string(&slot).unwrap(), test_keys::plain());
        assert!(matches!(&b.state().key_slots[&id].source, Some(SlotSource::SyncedCopy { fingerprint }) if fingerprint == test_keys::PLAIN_FINGERPRINT));
        assert!(vault_entry(&b, &id).is_none(), "the vault no longer holds it");
        assert!(!b.state().key_slots[&id].copy_from_another_account, "it came from this account");
        assert!(vault_slot_files(&b.state()).is_empty());
    }

    #[test]
    fn a_linked_key_moves_into_the_vault_and_the_original_file_stays() {
        let (_relay, _clock, a, _b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        let original = a.ssh_dir().join("id_mac");
        let slot = home(&a).join(SLOT_DIR).join(&file);

        set_delivery(&a.env(), &id, true).unwrap();
        assert_eq!(std::fs::read_to_string(&original).unwrap(), test_keys::plain(), "the user's own file is untouched");
        assert!(!slot_files::occupied(&slot), "SSHelter's link is gone");
        assert!(public_path(&slot).is_file());
        assert_eq!(vault_entry(&a, &id).unwrap().origin, crate::vault::store::EntryOrigin::Imported);

        set_delivery(&a.env(), &id, false).unwrap();
        assert!(b_copy_flag(&a, &id), "an imported key returns as a copy that still needs this computer's consent to upload");
    }

    fn b_copy_flag(d: &TestDevice, id: &str) -> bool {
        d.state().key_slots[id].copy_from_another_account
    }

    #[test]
    fn a_vault_slot_gets_its_pub_back_and_reports_a_file_in_the_way() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let slot = home(&b).join(SLOT_DIR).join(&file);

        std::fs::remove_file(public_path(&slot)).unwrap();
        settle(&b);
        assert_eq!(std::fs::read_to_string(public_path(&slot)).unwrap().trim(), test_keys::PLAIN_PUBLIC, "the .pub is written again");

        std::fs::write(&slot, "someone else's file").unwrap();
        settle(&b);
        assert_eq!(b.state().key_slots[&id].last_error.as_deref(), Some(in_the_way_message(&slot).as_str()));
        assert_eq!(std::fs::read_to_string(&slot).unwrap(), "someone else's file", "never overwritten");
    }

    #[test]
    fn republish_restores_a_vault_key_from_this_account_but_not_an_imported_one_without_consent() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let keys = account_keys(&b);
        let lost = |d: &TestDevice| {
            let mut state = d.state();
            let account = state.account.as_mut().unwrap();
            account.records.remove(&record_key(RecordKind::KeySlot, &id));
            account.sealed.remove(&key_secret_key(&keys, &id));
            state
        };

        let mut state = lost(&b);
        let env = b.env();
        reconcile_with_vault(&mut state, &keys, &home(&b), &NO_OTHER_HOSTS, 1_000, &EnvVault { env: &env });
        assert_eq!(open_key_secret(state.account.as_ref().unwrap(), &keys, &id).as_deref(), Some(test_keys::plain().as_str()), "a synced key from this account comes back");

        mutate(&b.env(), |s| {
            s.key_slots.get_mut(&id).unwrap().copy_from_another_account = true;
            Ok(())
        })
        .unwrap();
        let mut state = lost(&b);
        let env = b.env();
        reconcile_with_vault(&mut state, &keys, &home(&b), &NO_OTHER_HOSTS, 2_000, &EnvVault { env: &env });
        assert!(slot(state.account.as_ref().unwrap(), &id).is_some(), "the keyslot is written again");
        assert_eq!(open_key_secret(state.account.as_ref().unwrap(), &keys, &id), None, "but no key without consent");
    }

    #[test]
    fn using_the_synced_key_replaces_the_vault_entry() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        // A syncs another key into the slot.
        std::fs::write(a.ssh_dir().join("id_mac"), test_keys::ecdsa()).unwrap();
        settle(&a);
        set_mode(&a.env(), &id, SlotMode::Synced).unwrap();
        settle(&a);
        settle(&b);
        assert!(matches!(view_of(&b).remove(0).status, SlotStatusView::SyncedAvailable { .. }));

        use_synced(&b.env(), &id).unwrap();
        assert_eq!(vault_entry(&b, &id).unwrap().private_key, test_keys::ecdsa());
        assert!(matches!(&b.state().key_slots[&id].source, Some(SlotSource::Vault { fingerprint, .. }) if fingerprint == test_keys::ECDSA_FINGERPRINT));
        let slot = home(&b).join(SLOT_DIR).join(&file);
        assert!(!slot.exists());
        assert_eq!(std::fs::read_to_string(public_path(&slot)).unwrap().trim(), test_keys::ECDSA_PUBLIC);
    }

    #[test]
    fn deleting_an_unused_vault_key_removes_the_entry_and_the_pub() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        b.save_in_app(&b.space_path(&personal), "Host web\n  HostName 1.1.1.1\n");
        settle(&b);
        assert!(matches!(view_of(&b).remove(0).status, SlotStatusView::NotInUse { .. }));

        delete_copy(&b.env(), &id).unwrap();
        assert!(vault_entry(&b, &id).is_none());
        assert!(!public_path(&home(&b).join(SLOT_DIR).join(&file)).exists());
        assert!(!b.state().key_slots.contains_key(&id));
    }

    #[test]
    fn a_lost_vault_entry_comes_back_from_the_account() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        // The vault file is gone (for example set aside because its key left the keychain).
        std::fs::remove_file(crate::vault::store::vault_path(&b.env().state_path)).unwrap();
        settle(&b);
        assert_eq!(vault_entry(&b, &id).unwrap().private_key, test_keys::plain(), "restored from the account");
        assert!(matches!(b.state().key_slots[&id].source, Some(SlotSource::Vault { .. })));
        assert_eq!(b.state().key_slots[&id].last_error, None);
        assert!(!home(&b).join(SLOT_DIR).join(&file).exists(), "still no private key file in the slot");
    }

    #[test]
    fn a_lost_vault_key_the_account_does_not_have_leaves_the_slot_without_a_key() {
        let (_relay, _clock, a, _b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Own, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        set_delivery(&a.env(), &id, true).unwrap();
        std::fs::remove_file(crate::vault::store::vault_path(&a.env().state_path)).unwrap();
        settle(&a);
        let local = a.state().key_slots.get(&id).cloned();
        assert!(local.as_ref().is_none_or(|l| l.source.is_none()), "no longer delivered from the vault: {local:?}");
        assert!(vault_slot_files(&a.state()).is_empty());
        assert_eq!(std::fs::read_to_string(a.ssh_dir().join("id_mac")).unwrap(), test_keys::plain(), "the user's own file is untouched");
    }

    #[test]
    fn an_unreadable_vault_changes_nothing_in_a_round() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let vault = crate::vault::store::vault_path(&b.env().state_path);
        std::fs::write(&vault, "{ not json").unwrap();
        settle(&b);
        assert!(matches!(b.state().key_slots[&id].source, Some(SlotSource::Vault { .. })), "a round never acts on a vault it cannot read");
        assert_eq!(std::fs::read_to_string(&vault).unwrap(), "{ not json", "and never moves it");
    }

    #[test]
    fn picking_a_key_for_a_vault_slot_is_refused() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let other = b.ssh_dir().join("id_other");
        std::fs::write(&other, test_keys::ecdsa()).unwrap();
        refused(&b, VAULT_FIRST_MESSAGE, || pick(&b.env(), &id, &other.display().to_string()));
    }
```

Check the exact helper names in the existing test module (`refused`, `view_of`, `public_path` import, `test_keys::ECDSA_PUBLIC`); adjust only the names, not the assertions. If `test_keys` has no `ECDSA_PUBLIC`, assert with `inspect_private_key(&test_keys::ecdsa()).unwrap().public_key` instead.

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib sync::slots -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`set_delivery`, `SlotSource::Vault`, `reconcile_with_vault`, `EnvVault`, `vault_slot_files`, `in_vault` not found).

- [ ] **Step 3: Add the variant**

In `src-tauri/src/sync/state_v2.rs`, add to `SlotSource`:

```rust
    /// 只在 SSHelter(金鑰保管庫 spec §4.3):私鑰在保管庫(`vault.json`),插槽目錄只放 `.pub`,插槽路徑本身沒有檔案;`ssh` 經 SSHelter 的
    /// agent 取用。`public_key` 用來在每一輪重寫 `.pub`(不必開保管庫),`has_passphrase` 給畫面說明。
    Vault { fingerprint: String, public_key: String, has_passphrase: bool },
```

Add a round-trip assertion to the existing state serialization test in `state_v2.rs` if there is one that lists every `SlotSource` variant; otherwise none is needed.

- [ ] **Step 4: Implement the slots changes**

In `src-tauri/src/sync/slots.rs`:

1. Imports: add `use zeroize::Zeroizing;`, `use crate::vault::store::{vault_path, with_vault, EntryOrigin, VaultEntry};`, and `public_path` to the `slot_rules` import list if it is not there.

2. The vault reader, near the top of the file:

```rust
/// 同步的一輪用到的保管庫:補寫帳戶裡的 `key`(SP3 spec §6.6)要讀私鑰;「只在 SSHelter」的插槽要確認保管庫裡還有它,沒有就從帳戶放回去
/// (金鑰保管庫 spec §11)。`holds` 只讀檔案裡的 id,不讀 keychain;`private_key` 與 `restore` 才開保管庫。
pub trait VaultKeys {
    fn private_key(&self, slot_id: &str) -> Option<(Zeroizing<String>, EntryOrigin)>;
    /// 保管庫裡有沒有這一筆;保管庫讀不了(格式不認得、讀不懂、I/O 錯誤)→ None,呼叫端什麼都不做。
    fn holds(&self, slot_id: &str) -> Option<bool>;
    /// 放回一筆(從帳戶取回的同步金鑰)。
    fn restore(&self, slot_id: &str, entry: &VaultEntry) -> Result<(), AppError>;
}

/// 沒有保管庫可讀(測試、或只處理檔案來源的呼叫端)。
pub struct NoVault;

impl VaultKeys for NoVault {
    fn private_key(&self, _slot_id: &str) -> Option<(Zeroizing<String>, EntryOrigin)> {
        None
    }

    fn holds(&self, _slot_id: &str) -> Option<bool> {
        None
    }

    fn restore(&self, _slot_id: &str, _entry: &VaultEntry) -> Result<(), AppError> {
        Err(AppError::Other("SSHelter's vault is not available here".to_string()))
    }
}

/// 經 `SyncEnv` 開保管庫(`vault::store::with_vault`)。開不了(keychain 鎖著、檔案讀不懂)就當成沒有。
pub struct EnvVault<'a, 'b> {
    pub env: &'a SyncEnv<'b>,
}

impl VaultKeys for EnvVault<'_, '_> {
    fn private_key(&self, slot_id: &str) -> Option<(Zeroizing<String>, EntryOrigin)> {
        let path = vault_path(&self.env.state_path);
        with_vault(self.env.runtime, &path, self.env.keychain, self.env.now(), |vault| vault.get(slot_id))
            .ok()
            .flatten()
            .map(|entry| (Zeroizing::new(entry.private_key.clone()), entry.origin))
    }

    fn holds(&self, slot_id: &str) -> Option<bool> {
        crate::vault::store::stored_ids(&vault_path(&self.env.state_path)).ok().map(|ids| ids.contains(slot_id))
    }

    fn restore(&self, slot_id: &str, entry: &VaultEntry) -> Result<(), AppError> {
        let path = vault_path(&self.env.state_path);
        Ok(with_vault(self.env.runtime, &path, self.env.keychain, self.env.now(), |vault| vault.put(self.env.keychain, slot_id, entry))?)
    }
}

pub const VAULT_ENTRY_LOST: &str = "This key was lost from SSHelter's vault. Pick it again on this computer.";

/// 只在 SSHelter 的插槽,保管庫裡卻沒有它(保管庫檔讀不懂、或 `vault:key` 不見而搬到旁邊之後;金鑰保管庫 spec §11):帳戶裡有同一把同步金鑰
/// 就放回保管庫;沒有就讓這台回到「還沒有金鑰」,之後照 SP3 的流程落地同步的金鑰或請使用者挑。保管庫讀不了(`holds` 是 None)就不動。
fn recover_vault_entry(
    local: &mut LocalSlot,
    slot_id: &str,
    account: &AccountState,
    account_keys: &ChainKeys,
    now_ms: u64,
    vault: &dyn VaultKeys,
) {
    let Some(SlotSource::Vault { fingerprint, .. }) = &local.source else { return };
    if vault.holds(slot_id) != Some(false) {
        return;
    }
    let restored = open_key_secret(account, account_keys, slot_id).and_then(|text| {
        let facts = inspect_private_key(&text).ok()?;
        (&facts.fingerprint == fingerprint).then(|| VaultEntry {
            private_key: text.clone(),
            public_key: facts.public_key.clone(),
            fingerprint: facts.fingerprint.clone(),
            origin: if local.copy_from_another_account { EntryOrigin::Imported } else { EntryOrigin::Synced },
            added_at_ms: now_ms,
        })
    });
    match restored {
        Some(entry) => local.last_error = vault.restore(slot_id, &entry).err().map(|e| e.to_string()),
        None => {
            local.source = None;
            local.last_error = Some(VAULT_ENTRY_LOST.to_string());
        }
    }
}

/// 這台「只在 SSHelter」的插槽檔名(`SlotSource::Vault`):`agent::wiring` 據此列出要走 agent 的主機。
pub fn vault_slot_files(state: &SyncStateV2) -> BTreeSet<String> {
    state
        .key_slots
        .values()
        .filter(|local| matches!(local.source, Some(SlotSource::Vault { .. })))
        .map(|local| local.file_name.clone())
        .collect()
}

pub const VAULT_FIRST_MESSAGE: &str = "This key is only in SSHelter. Choose Keep a file first.";
```

3. Rename the existing `pub fn reconcile(...)` to `pub fn reconcile_with_vault(..., now_ms: u64, vault: &dyn VaultKeys) -> SlotRound` (add the last parameter), pass `vault` through to its `republish(...)` call, and add the old name back as a wrapper:

```rust
/// 不讀保管庫的版本(補寫時讀不到保管庫裡的私鑰):測試與只處理檔案來源的呼叫端用。
pub fn reconcile(
    state: &mut SyncStateV2,
    account_keys: &ChainKeys,
    home: &Path,
    in_use: &BTreeMap<String, Vec<String>>,
    now_ms: u64,
) -> SlotRound {
    reconcile_with_vault(state, account_keys, home, in_use, now_ms, &NoVault)
}
```

   In `reconcile_with_vault`, call the recovery in both loops:
   - live slots: right after the `if maintain(&mut local, &keys_dir, &path) { … }` block inside `else if used(&file) {`, add `recover_vault_entry(&mut local, id, account, account_keys, now_ms, vault);` (before the `asked` check);
   - slots no longer in the account: right after the `if !recorded && used(&local.file_name) { maintain(…) } else { drop_link(…) }` block, add `recover_vault_entry(&mut local, &id, account, account_keys, now_ms, vault);` (before `if local.source.is_none()`).

   `account` is `&mut AccountState` there; pass it as is (it reborrows as `&AccountState`).

   In `src-tauri/src/vault/store.rs`, add (it reuses `file_version`, the private version probe `Vault::open` uses):

```rust
/// 保管庫檔裡的插槽 id。不讀 keychain、不搬任何檔案:同步的每一輪用它確認「只在 SSHelter」的插槽還在保管庫裡(金鑰保管庫 spec §11)。
/// 檔案不存在 → 空的;讀不懂 → `Unreadable { kept_as: None, .. }`;更新版的格式 → `Newer`。
pub fn stored_ids(path: &Path) -> Result<BTreeSet<String>, VaultError> {
    let bytes = match std::fs::read(path) {
        Ok(bytes) => bytes,
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => return Ok(BTreeSet::new()),
        Err(e) => return Err(VaultError::Other(AppError::Io(e))),
    };
    let unreadable = |e: serde_json::Error| VaultError::Unreadable { kept_as: None, reason: e.to_string() };
    let version = file_version(&bytes).map_err(unreadable)?;
    if version > VAULT_VERSION {
        return Err(VaultError::Newer { version });
    }
    let file = serde_json::from_slice::<VaultFile>(&bytes).map_err(unreadable)?;
    Ok(file.entries.keys().cloned().collect())
}
```

   with a test in `store.rs`: a missing file → empty; after two `put`s → both ids; `"{ not json"` → `Unreadable { kept_as: None, .. }` and the file is still there.

4. `republish`: add the parameter `vault: &dyn VaultKeys` and this arm to its `readable` match, next to the `SyncedCopy` arm:

```rust
        // 保管庫裡的私鑰:這台同意上傳過的那把,或從這個帳戶同步來、不是之前的帳戶留下的。
        Some(SlotSource::Vault { fingerprint, .. }) => vault.private_key(slot_id).and_then(|(text, origin)| {
            let consented = local.uploaded_fingerprint.as_deref() == Some(fingerprint.as_str());
            let from_this_account = origin == EntryOrigin::Synced && !local.copy_from_another_account;
            (consented || from_this_account).then(|| text.to_string())
        }),
```

5. `device_slot`: add

```rust
        Some(SlotSource::Vault { fingerprint, .. }) => {
            Some(DeviceSlot { slot_id: slot_id.to_string(), fingerprint: Some(fingerprint.clone()), synced_copy: true })
        }
```

6. `maintain`: add this arm before the `SyncedCopy` arms:

```rust
        Some(SlotSource::Vault { public_key, .. }) => {
            // 只在 SSHelter:插槽路徑上不該有檔案(`ssh` 會先讀它)。出現了就擋路、不碰;`.pub` 不見或不對就從記錄重寫。
            if slot_files::occupied(path) {
                local.last_error = Some(in_the_way_message(path));
                return false;
            }
            let current = std::fs::read_to_string(public_path(path)).ok();
            if current.as_deref().map(str::trim) != Some(public_key.trim()) {
                if let Err(e) = slot_files::ensure_keys_dir(keys_dir).and_then(|()| slot_files::write_public(path, &public_key)) {
                    local.last_error = Some(e.to_string());
                    return false;
                }
            }
            local.last_error = None;
            false
        }
```

7. `slot_status`: add

```rust
        Some(SlotSource::Vault { fingerprint, .. }) => {
            if synced && has_secret && Some(fingerprint) != payload.fingerprint.as_ref() {
                SlotStatusView::SyncedAvailable { file: here }
            } else if !used {
                SlotStatusView::NotInUse { file: here }
            } else {
                SlotStatusView::Ready { file: here, synced_copy: false, fingerprint: Some(fingerprint.clone()) }
            }
        }
```

8. `local_key_passphrase`: return early for the vault — `if let Some(SlotSource::Vault { has_passphrase, .. }) = &local.source { return Some(*has_passphrase); }` (before computing `file`).

9. `readable_key`: add `SlotSource::Vault { .. } => None,` (callers that need the vault's text read it themselves).

10. `occupant`: add `Some(SlotSource::Vault { .. }) => Occupant::NotOurs,` (a vault slot owns nothing at the slot path).

11. `holds_recorded_copy`: add `SlotSource::Vault { .. } => false,`.

12. `set_mode`: when the source is the vault, read the text from the vault instead of `readable_key`:

```rust
        SlotMode::Synced => Some(match state.key_slots.get(slot_id).and_then(|l| l.source.as_ref()) {
            Some(SlotSource::Vault { .. }) => vault_text(env, slot_id)?.ok_or_else(not_here)?,
            _ => readable_key(state.key_slots.get(slot_id), &home.join(SLOT_DIR)).ok_or_else(not_here)?,
        }),
```

with the helper

```rust
/// 保管庫裡這個插槽的私鑰原文。
fn vault_text(env: &SyncEnv, slot_id: &str) -> Result<Option<String>, AppError> {
    let path = vault_path(&env.state_path);
    Ok(with_vault(env.runtime, &path, env.keychain, env.now(), |vault| vault.get(slot_id))?.map(|entry| entry.private_key.clone()))
}
```

13. `pick`: right after the snapshot, refuse a vault slot:

```rust
    if state.key_slots.get(slot_id).is_some_and(|l| matches!(l.source, Some(SlotSource::Vault { .. }))) {
        return Err(AppError::Other(VAULT_FIRST_MESSAGE.to_string()));
    }
```

14. `use_synced`: keep the facts from the existing `check_synced_key` call (`let facts = check_synced_key(&secret, &payload).map_err(AppError::Other)?;`), and before the `occupant` logic handle the vault:

```rust
    if matches!(local.and_then(|l| l.source.as_ref()), Some(SlotSource::Vault { .. })) {
        let now = env.now();
        let entry = VaultEntry {
            private_key: secret.clone(),
            public_key: facts.public_key.clone(),
            fingerprint: facts.fingerprint.clone(),
            origin: EntryOrigin::Synced,
            added_at_ms: now,
        };
        with_vault(env.runtime, &vault_path(&env.state_path), env.keychain, now, |vault| vault.put(env.keychain, slot_id, &entry))?;
        slot_files::ensure_keys_dir(&keys_dir)?;
        slot_files::write_public(&slot_path, &facts.public_key)?;
        mutate(env, |s| {
            let learned_here = learned_now(s, &keys.chain_id);
            let local = s.key_slots.get_mut(slot_id).ok_or_else(not_found)?;
            local.source = Some(SlotSource::Vault {
                fingerprint: facts.fingerprint.clone(),
                public_key: facts.public_key.clone(),
                has_passphrase: facts.has_passphrase,
            });
            local.last_error = None;
            if learned_here.is_some() {
                local.learned_in = learned_here;
            }
            local.copy_from_another_account = local.learned_in.as_deref() != Some(keys.chain_id.as_str());
            Ok(())
        })?;
        env.events.wake();
        return Ok(());
    }
```

15. `delete_copy`: before the `let copy = match ...`, handle an unused vault key:

```rust
    if matches!(local.source, Some(SlotSource::Vault { .. })) {
        let path = home.join(SLOT_DIR).join(&local.file_name);
        if slot_files::occupied(&path) {
            return Err(AppError::Other(in_the_way_message(&path)));
        }
        with_vault(env.runtime, &vault_path(&env.state_path), env.keychain, env.now(), |vault| vault.remove(slot_id))?;
        let _ = std::fs::remove_file(public_path(&path));
        mutate(env, |s| {
            s.key_slots.remove(slot_id);
            Ok(())
        })?;
        env.events.wake();
        return Ok(());
    }
```

16. `set_delivery` (new, after `delete_copy`):

```rust
/// 這台的插槽改成只在 SSHelter(`vault` = true:私鑰放進保管庫,插槽目錄只留 `.pub`)或改回檔案(插槽路徑放私鑰的副本,保管庫不再留它)。
/// 使用者自己的原檔一律不碰;插槽路徑上可能是某把金鑰僅存名字的 hard link 或複製檔,改名保留(`retire_key`)而不刪除。
pub fn set_delivery(env: &SyncEnv, slot_id: &str, vault: bool) -> Result<(), AppError> {
    let (state, _keys, home) = snapshot(env)?;
    if contested_and_not_held(&state, slot_id) {
        return Err(contested_error());
    }
    let local = state.key_slots.get(slot_id).cloned().ok_or_else(not_found)?;
    let keys_dir = home.join(SLOT_DIR);
    let slot_path = keys_dir.join(&local.file_name);
    let vault_file = vault_path(&env.state_path);
    let now = env.now();
    if vault {
        let origin = match &local.source {
            Some(SlotSource::Vault { .. }) => return Ok(()),
            Some(SlotSource::SyncedCopy { .. }) if !local.copy_from_another_account => EntryOrigin::Synced,
            Some(SlotSource::SyncedCopy { .. } | SlotSource::Linked { .. }) => EntryOrigin::Imported,
            None => return Err(AppError::Other("This computer doesn't have this key yet.".to_string())),
        };
        let text = readable_key(Some(&local), &keys_dir)
            .ok_or_else(|| AppError::Other(source_gone_message(&slot_path.display().to_string())))?;
        let facts = inspect_private_key(&text).map_err(|e| AppError::Other(e.message().to_string()))?;
        match occupant(Some(&local), &local.file_name, &slot_path) {
            Occupant::NotOurs => return Err(AppError::Other(in_the_way_message(&slot_path))),
            Occupant::OwnLink | Occupant::OwnKey | Occupant::Empty => {}
        }
        let entry = VaultEntry {
            private_key: text,
            public_key: facts.public_key.clone(),
            fingerprint: facts.fingerprint.clone(),
            origin,
            added_at_ms: now,
        };
        with_vault(env.runtime, &vault_file, env.keychain, now, |v| v.put(env.keychain, slot_id, &entry))?;
        match occupant(Some(&local), &local.file_name, &slot_path) {
            Occupant::OwnLink => slot_files::remove_slot(&slot_path)?,
            Occupant::OwnKey => match &local.source {
                // 同步來的副本:私鑰已在保管庫,也還在帳戶裡,直接拿掉。
                Some(SlotSource::SyncedCopy { .. }) => slot_files::remove_slot(&slot_path)?,
                // 複製檔或可能是僅存名字的 hard link:改名保留。
                _ => retire_key(&slot_path)?,
            },
            Occupant::Empty | Occupant::NotOurs => {}
        }
        slot_files::ensure_keys_dir(&keys_dir)?;
        slot_files::write_public(&slot_path, &facts.public_key)?;
        mutate(env, |s| {
            let local = s.key_slots.get_mut(slot_id).ok_or_else(not_found)?;
            local.source = Some(SlotSource::Vault {
                fingerprint: facts.fingerprint.clone(),
                public_key: facts.public_key.clone(),
                has_passphrase: facts.has_passphrase,
            });
            local.last_error = None;
            local.parked = false;
            Ok(())
        })?;
    } else {
        let Some(SlotSource::Vault { .. }) = &local.source else { return Ok(()) };
        if slot_files::occupied(&slot_path) {
            return Err(AppError::Other(in_the_way_message(&slot_path)));
        }
        let entry = with_vault(env.runtime, &vault_file, env.keychain, now, |v| v.get(slot_id))?
            .ok_or_else(|| AppError::Other("This key is missing from SSHelter's vault.".to_string()))?;
        slot_files::ensure_keys_dir(&keys_dir)?;
        slot_files::write_private(&slot_path, entry.private_key.as_bytes())?;
        slot_files::write_public(&slot_path, &entry.public_key)?;
        with_vault(env.runtime, &vault_file, env.keychain, now, |v| v.remove(slot_id))?;
        let from_this_account = entry.origin == EntryOrigin::Synced;
        let fingerprint = entry.fingerprint.clone();
        mutate(env, |s| {
            let local = s.key_slots.get_mut(slot_id).ok_or_else(not_found)?;
            local.source = Some(SlotSource::SyncedCopy { fingerprint: fingerprint.clone() });
            local.copy_from_another_account = local.copy_from_another_account || !from_this_account;
            local.last_error = None;
            Ok(())
        })?;
    }
    env.events.wake();
    Ok(())
}
```

17. `views`: add `in_vault` to the `SyncKeySlotView` the `view` closure builds — `in_vault: state.key_slots.get(id).is_some_and(|l| matches!(l.source, Some(SlotSource::Vault { .. })))`.

In `src-tauri/src/sync/dto.rs`, add to `SyncKeySlotView` after `in_account`:

```rust
    /// 這台只在 SSHelter(私鑰在保管庫,經 agent 提供;金鑰保管庫 spec §4.3)。
    pub in_vault: bool,
```

In `src-tauri/src/sync/slot_setup.rs`: in `kept_key`, add `SlotSource::Vault { .. } => None,` (Plan 1 does not offer kept vault keys); in `held_in_place`, add `Some(SlotSource::Vault { .. }) if !public_path(&slot_path).is_file() => gone(&slot_path),` (import `public_path` if needed).

In `src-tauri/src/sync/round.rs` step 6b, replace `crate::sync::slots::reconcile(&mut work, &keys, home, &in_use, now)` with:

```rust
        let slot_round = crate::sync::slots::reconcile_with_vault(&mut work, &keys, home, &in_use, now, &crate::sync::slots::EnvVault { env });
```

In `src-tauri/src/sync/engine.rs`, after `sync_key_delete_copy`:

```rust
/// 這台的插槽改成只在 SSHelter,或改回檔案(金鑰保管庫 spec §4.3;只改這台)。
#[tauri::command]
pub async fn sync_key_set_delivery(app: AppHandle, slot_id: String, vault: bool) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| crate::sync::slots::set_delivery(env, &slot_id, vault)).await
}
```

Register `sync_key_set_delivery` in `src-tauri/src/lib.rs` the same way `sync_key_delete_copy` is imported and listed.

- [ ] **Step 5: Run the tests to verify they pass**

Run: `cargo test --offline --lib -- sync:: vault:: --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS, including every existing SP3 test. Then run `./node_modules/.bin/tsc --noEmit` from the repo root: it lists the `SyncKeySlotView` fixtures to update with `in_vault: false` (in `src/lib/sync-fixtures.ts` and any test that builds one). Then `./node_modules/.bin/vitest run`.

- [ ] **Step 6: Run the full Rust suite and commit**

```bash
git add src-tauri/src/sync src-tauri/src/vault/store.rs src-tauri/src/lib.rs src/bindings/SyncKeySlotView.ts src/lib/sync-fixtures.ts
git commit -m "feat(sync): let a key slot live only in SSHelter's vault"
```
(Add any other fixture file tsc pointed at.)

---

### Task 8: The broker — which keys, approvals, passphrases

**Files:**
- Create: `src-tauri/src/agent/broker.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`pub mod broker;`, `AgentRuntime.broker`)

(The module is `broker`, not `core`: a module named `core` inside `agent` would shadow the `core` crate in `agent/mod.rs`.)

**Interfaces:**
- Consumes:
  - Task 3: `crate::agent::approval::{remember_minutes, verdict, ApprovalCache, ApprovalKey, KeyProtection, Verdict}`
  - Task 6: `crate::agent::prompt::{AgentApprovalRequest, AgentApprovalAnswer, APPROVAL_TIMEOUT}`, `crate::agent::AgentRuntime`
  - Task 2: `crate::vault::material::{open, public_key_data, Material, OpenError}`
  - Task 4: `crate::agent::session::{SignAuthority, SignRequest}`
  - Task 5: `crate::agent::peer::Program`
  - Task 1: `crate::vault::store::AgentSettings`; Task 7: `SlotSource::Vault { fingerprint, public_key, has_passphrase }`
- Produces:
  - `crate::agent::broker::{PASSPHRASE_ATTEMPTS = 3, WRONG_PASSPHRASE = "That passphrase didn't work.", passphrase_account(slot_id) -> String /* "vault:passphrase:<slot id>" */}`
  - `#[derive(Clone, Debug, PartialEq, Eq)] pub struct VaultKey { pub slot_id: String, pub name: String, pub fingerprint: String, pub public_key: String, pub has_passphrase: bool }`
  - `pub fn vault_keys(state: &SyncStateV2) -> Vec<VaultKey>`
  - `pub fn host_name_in(known_hosts: &str, host_key: &KeyData) -> Option<String>`
  - `pub trait AgentHost: Send + Sync { fn keys(&self) -> Vec<VaultKey>; fn private_key(&self, slot_id: &str) -> Result<Option<Zeroizing<String>>, AppError>; fn settings(&self) -> AgentSettings; fn keychain(&self) -> &dyn Keychain; fn now_ms(&self) -> u64; fn host_name(&self, host_key: &KeyData) -> Option<String>; fn ask(&self, request: AgentApprovalRequest) -> Option<AgentApprovalAnswer>; }`
  - `#[derive(Clone, Debug, PartialEq, Eq)] pub struct Grant { pub slot_id: String }`
  - `#[derive(Default)] pub struct Broker` with `identities(&self, host: &dyn AgentHost, grant: Option<&Grant>) -> Vec<(KeyData, String)>`, `sign(&self, host: &dyn AgentHost, request: &SignRequest, program: Option<&Program>, grant: Option<&Grant>) -> Option<Vec<u8>>`, `clear(&self)`
  - `pub struct Connection<'a> { pub broker: &'a Broker, pub host: &'a dyn AgentHost, pub program: Option<Program>, pub grant: Option<Grant> }` implementing `SignAuthority`
  - `AgentRuntime.broker: broker::Broker`

Rules (spec §5.2, §5.3, §5.5, §5.6):
- Forwarded requests are refused before anything else; a key that is not one of this computer's vault keys is refused; a grant serves only its own key.
- Approval key = key fingerprint × host key fingerprint × program identity. No bound host or no recognised program → ask, never remember. `always_ask` → ask, never remember. Per-key protection is `KeyProtection::default()` in Plan 1 (`keyprefs` is Plan 2).
- Identical rememberable requests in flight share one prompt: the first asks, the others wait for its answer (allowed → they unlock and sign without a prompt; denied → refused).
- Passphrase: memory → none needed → keychain (`vault:passphrase:<slot id>`; a wrong one is deleted) → the approval window's field or an unlock-only prompt (`preapproved: true`), three attempts in all. A passphrase remembered in the keychain is used per signature and the opened key is not kept; otherwise the opened key stays in memory for the remember window, extended by later remembered approvals of the same key.
- An approval is remembered only after the key opened.
- A grant (Connect's one-shot channel) never shows the approval and is never remembered; it may still show the unlock-only prompt.
- Host shown in the window: the known_hosts name; without one, the host key's fingerprint; no bound host → None ("an unknown host").

- [ ] **Step 1: Write the failing tests**

Add `pub mod broker;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/broker.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::slot_rules::test_keys;
    use crate::sync::testkit::MemKeychain;
    use std::collections::VecDeque;
    use std::sync::atomic::AtomicU64;

    const ID: &str = "0123456789abcdef0123456789abcdef";
    const ENC_ID: &str = "fedcba9876543210fedcba9876543210";

    struct FakeHost {
        keys: Vec<VaultKey>,
        private: HashMap<String, String>,
        settings: AgentSettings,
        keychain: MemKeychain,
        now: AtomicU64,
        names: HashMap<String, String>,
        answers: Mutex<VecDeque<Option<AgentApprovalAnswer>>>,
        asked: Mutex<Vec<AgentApprovalRequest>>,
        /// `ask` waits for this gate before it answers (the parallel test closes it).
        gate: Mutex<bool>,
        opened: Condvar,
    }

    impl FakeHost {
        fn new() -> Self {
            let key = |slot_id: &str, name: &str, public: &str, fingerprint: &str, has_passphrase: bool| VaultKey {
                slot_id: slot_id.into(),
                name: name.into(),
                fingerprint: fingerprint.into(),
                public_key: public.into(),
                has_passphrase,
            };
            FakeHost {
                keys: vec![
                    key(ID, "id_mac", test_keys::PLAIN_PUBLIC, test_keys::PLAIN_FINGERPRINT, false),
                    key(ENC_ID, "id_enc", test_keys::ENC_PUBLIC, test_keys::ENC_FINGERPRINT, true),
                ],
                private: HashMap::from([(ID.to_string(), test_keys::plain()), (ENC_ID.to_string(), test_keys::encrypted())]),
                settings: AgentSettings::default(),
                keychain: MemKeychain::default(),
                now: AtomicU64::new(1_000),
                names: HashMap::new(),
                answers: Mutex::new(VecDeque::new()),
                asked: Mutex::new(Vec::new()),
                gate: Mutex::new(true),
                opened: Condvar::new(),
            }
        }

        fn answer(&self, answer: Option<AgentApprovalAnswer>) {
            self.answers.lock().unwrap().push_back(answer);
        }

        fn asked(&self) -> Vec<AgentApprovalRequest> {
            self.asked.lock().unwrap().clone()
        }

        fn advance(&self, ms: u64) {
            self.now.fetch_add(ms, Ordering::SeqCst);
        }
    }

    impl AgentHost for FakeHost {
        fn keys(&self) -> Vec<VaultKey> {
            self.keys.clone()
        }
        fn private_key(&self, slot_id: &str) -> Result<Option<Zeroizing<String>>, AppError> {
            Ok(self.private.get(slot_id).cloned().map(Zeroizing::new))
        }
        fn settings(&self) -> AgentSettings {
            self.settings.clone()
        }
        fn keychain(&self) -> &dyn Keychain {
            &self.keychain
        }
        fn now_ms(&self) -> u64 {
            self.now.load(Ordering::SeqCst)
        }
        fn host_name(&self, host_key: &KeyData) -> Option<String> {
            self.names.get(&host_key.fingerprint(HashAlg::Sha256).to_string()).cloned()
        }
        fn ask(&self, request: AgentApprovalRequest) -> Option<AgentApprovalAnswer> {
            self.asked.lock().unwrap().push(request);
            let open = self.gate.lock().unwrap();
            drop(self.opened.wait_while(open, |open| !*open).unwrap());
            self.answers.lock().unwrap().pop_front().flatten()
        }
    }

    fn allow(remember: bool) -> Option<AgentApprovalAnswer> {
        Some(AgentApprovalAnswer { allow: true, remember, ..Default::default() })
    }

    fn with_passphrase(passphrase: &str, remember_passphrase: bool) -> Option<AgentApprovalAnswer> {
        Some(AgentApprovalAnswer { allow: true, remember: true, passphrase: Some(passphrase.into()), remember_passphrase })
    }

    fn program(identity: &str) -> Program {
        Program { chain: vec![identity.into(), "ssh".into()], identity: identity.into() }
    }

    fn host_key() -> KeyData {
        material::public_key_data(test_keys::ECDSA_PUBLIC).unwrap()
    }

    fn request(public: &str, host: Option<KeyData>) -> SignRequest {
        SignRequest {
            key: material::public_key_data(public).unwrap(),
            data: b"to sign".to_vec(),
            flags: 0,
            user: Some("root".into()),
            host_key: host,
            forwarded: false,
        }
    }

    fn verifies(public: &str, blob: &[u8]) -> bool {
        use signature::Verifier;
        use ssh_encoding::Decode;
        let signature = ssh_key::Signature::decode(&mut &blob[..]).unwrap();
        material::public_key_data(public).unwrap().verify(b"to sign", &signature).is_ok()
    }

    #[test]
    fn vault_keys_come_from_vault_slots_only() {
        let mut state = SyncStateV2::fresh("mac").unwrap();
        let slot = |file: &str, source: Option<SlotSource>| crate::sync::state_v2::LocalSlot {
            file_name: file.into(),
            source,
            last_error: None,
            asked: false,
            payload: None,
            uploaded_fingerprint: None,
            parked: false,
            learned_in: None,
            copy_from_another_account: false,
        };
        let vault = SlotSource::Vault {
            fingerprint: test_keys::PLAIN_FINGERPRINT.into(),
            public_key: test_keys::PLAIN_PUBLIC.into(),
            has_passphrase: false,
        };
        state.key_slots.insert(ID.into(), slot("id_mac-01234567", Some(vault)));
        state.key_slots.insert(ENC_ID.into(), slot("other-fedcba98", Some(SlotSource::SyncedCopy { fingerprint: "SHA256:x".into() })));
        let keys = vault_keys(&state);
        assert_eq!(keys.len(), 1);
        assert_eq!(keys[0].slot_id, ID);
        assert_eq!(keys[0].name, "id_mac-01234567", "no payload: the slot file name");
        assert_eq!(keys[0].public_key, test_keys::PLAIN_PUBLIC);
    }

    #[test]
    fn identities_list_the_vault_keys_and_a_grant_lists_only_its_own() {
        let broker = Broker::default();
        let host = FakeHost::new();
        let all = broker.identities(&host, None);
        assert_eq!(all.iter().map(|(_, name)| name.as_str()).collect::<Vec<_>>(), vec!["id_mac", "id_enc"]);
        let one = broker.identities(&host, Some(&Grant { slot_id: ENC_ID.into() }));
        assert_eq!(one.len(), 1);
        assert_eq!(one[0].0, material::public_key_data(test_keys::ENC_PUBLIC).unwrap());
    }

    #[test]
    fn a_remembered_approval_is_reused_for_the_same_program_and_host_only() {
        let broker = Broker::default();
        let host = FakeHost::new();
        let claude = program("claude");
        host.answer(allow(true));
        let blob = broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&claude), None).unwrap();
        assert!(verifies(test_keys::PLAIN_PUBLIC, &blob));
        let first = &host.asked()[0];
        assert!(first.rememberable && !first.needs_passphrase && !first.preapproved);
        assert_eq!(first.program_chain, vec!["claude", "ssh"]);
        assert_eq!(first.user.as_deref(), Some("root"));
        assert_eq!(first.remember_minutes, 240);
        assert_eq!(first.host_fingerprint, Some(host_key().fingerprint(HashAlg::Sha256).to_string()));

        assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&claude), None).is_some());
        assert_eq!(host.asked().len(), 1, "remembered: no second prompt");

        host.answer(allow(false));
        assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("iterm2")), None).is_some());
        assert_eq!(host.asked().len(), 2, "another program asks again");

        host.advance(240 * 60_000);
        host.answer(allow(false));
        broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&claude), None);
        assert_eq!(host.asked().len(), 3, "expired after the remember window");
    }

    #[test]
    fn an_unknown_host_or_program_always_asks_and_is_never_remembered() {
        let broker = Broker::default();
        let host = FakeHost::new();
        for _ in 0..2 {
            host.answer(allow(true));
            assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, None), Some(&program("claude")), None).is_some());
        }
        for _ in 0..2 {
            host.answer(allow(true));
            assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), None, None).is_some());
        }
        let asked = host.asked();
        assert_eq!(asked.len(), 4);
        assert!(asked.iter().all(|r| !r.rememberable));
        assert_eq!(asked[0].host, None);
        assert!(asked[2].program_chain.is_empty());
    }

    #[test]
    fn this_computers_always_ask_setting_is_never_remembered() {
        let broker = Broker::default();
        let mut host = FakeHost::new();
        host.settings.always_ask = true;
        for _ in 0..2 {
            host.answer(allow(true));
            broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        }
        assert_eq!(host.asked().len(), 2);
        assert!(!host.asked()[0].rememberable);
    }

    #[test]
    fn deny_timeout_forwarding_and_unknown_keys_refuse() {
        let broker = Broker::default();
        let host = FakeHost::new();
        let claude = program("claude");
        host.answer(Some(AgentApprovalAnswer { allow: false, remember: true, ..Default::default() }));
        assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&claude), None).is_none());
        host.answer(None);
        assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&claude), None).is_none(), "timed out");
        assert_eq!(host.asked().len(), 2, "a denial is not remembered");

        let mut forwarded = request(test_keys::PLAIN_PUBLIC, Some(host_key()));
        forwarded.forwarded = true;
        assert!(broker.sign(&host, &forwarded, Some(&claude), None).is_none());
        assert!(broker.sign(&host, &request(test_keys::ECDSA_PUBLIC, Some(host_key())), Some(&claude), None).is_none(), "not a vault key");
        assert_eq!(host.asked().len(), 2, "neither asks");
    }

    #[test]
    fn the_host_shows_its_known_hosts_name_or_its_fingerprint() {
        let broker = Broker::default();
        let mut host = FakeHost::new();
        host.answer(allow(false));
        broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("a")), None);
        assert_eq!(host.asked()[0].host, Some(host_key().fingerprint(HashAlg::Sha256).to_string()));
        host.names.insert(host_key().fingerprint(HashAlg::Sha256).to_string(), "web".into());
        host.answer(allow(false));
        broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("b")), None);
        assert_eq!(host.asked()[1].host.as_deref(), Some("web"));
    }

    #[test]
    fn known_hosts_names_skip_hashed_names_patterns_and_markers() {
        let key = host_key();
        let line = |names: &str| format!("{names} {}\n", test_keys::ECDSA_PUBLIC);
        assert_eq!(host_name_in(&line("web,10.0.0.5"), &key).as_deref(), Some("web"));
        assert_eq!(host_name_in(&line("[web]:2222"), &key).as_deref(), Some("web:2222"));
        assert_eq!(host_name_in(&line("|1|c2FsdA==|aGFzaA==,lab"), &key).as_deref(), Some("lab"));
        assert_eq!(host_name_in(&line("|1|c2FsdA==|aGFzaA=="), &key), None);
        assert_eq!(host_name_in(&line("*.lab,!x"), &key), None);
        assert_eq!(host_name_in(&format!("@cert-authority *.lab {}\n# web\n", test_keys::ECDSA_PUBLIC), &key), None);
        assert_eq!(host_name_in(&format!("web {}\n", test_keys::PLAIN_PUBLIC), &key), None, "another key");
    }

    #[test]
    fn a_passphrase_is_asked_with_the_approval_and_kept_in_memory_for_the_remember_window() {
        let broker = Broker::default();
        let host = FakeHost::new();
        host.answer(with_passphrase("test-passphrase", false));
        let blob = broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        assert!(verifies(test_keys::ENC_PUBLIC, &blob));
        assert!(host.asked()[0].needs_passphrase);
        assert_eq!(host.keychain.entry(&passphrase_account(ENC_ID)), None, "not remembered on this computer");

        host.answer(allow(false));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("iterm2")), None).unwrap();
        assert!(!host.asked()[1].needs_passphrase, "still open in memory");

        host.advance(240 * 60_000);
        host.answer(with_passphrase("test-passphrase", false));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("iterm2")), None).unwrap();
        assert!(host.asked()[2].needs_passphrase, "the opened key is dropped when the window ends");
    }

    #[test]
    fn three_wrong_passphrases_refuse_and_a_right_retry_signs() {
        let broker = Broker::default();
        let host = FakeHost::new();
        for _ in 0..3 {
            host.answer(with_passphrase("wrong", false));
        }
        assert!(broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).is_none());
        let asked = host.asked();
        assert_eq!(asked.len(), 3);
        assert!(asked[1].preapproved && asked[1].needs_passphrase && !asked[1].rememberable);
        assert_eq!(asked[1].passphrase_error.as_deref(), Some(WRONG_PASSPHRASE));

        host.answer(with_passphrase("wrong", false));
        host.answer(with_passphrase("test-passphrase", false));
        assert!(broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).is_some());
        assert!(!host.asked()[3].preapproved, "the failed attempt was not remembered as an approval");
    }

    #[test]
    fn a_remembered_passphrase_lives_in_the_keychain_not_in_memory() {
        let broker = Broker::default();
        let host = FakeHost::new();
        host.answer(with_passphrase("test-passphrase", true));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        assert_eq!(host.keychain.entry(&passphrase_account(ENC_ID)).as_deref(), Some("test-passphrase"));

        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        assert_eq!(host.asked().len(), 1, "approval remembered, passphrase from the keychain");

        host.keychain.delete(&passphrase_account(ENC_ID)).unwrap();
        host.answer(with_passphrase("test-passphrase", false));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        let unlock = &host.asked()[1];
        assert!(unlock.preapproved && unlock.needs_passphrase, "the opened key was never kept in memory");
    }

    #[test]
    fn a_stale_remembered_passphrase_is_forgotten_and_asked_again() {
        let broker = Broker::default();
        let host = FakeHost::new();
        host.keychain.set(&passphrase_account(ENC_ID), "old").unwrap();
        host.answer(allow(false));
        host.answer(with_passphrase("test-passphrase", false));
        assert!(broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).is_some());
        let asked = host.asked();
        assert!(!asked[0].needs_passphrase, "the keychain had one");
        assert!(asked[1].preapproved && asked[1].needs_passphrase && asked[1].passphrase_error.is_none());
        assert_eq!(host.keychain.entry(&passphrase_account(ENC_ID)), None, "the wrong one is gone");
    }

    #[test]
    fn a_connect_grant_skips_the_approval_and_asks_only_for_a_passphrase() {
        let broker = Broker::default();
        let host = FakeHost::new();
        let grant = Grant { slot_id: ID.into() };
        assert!(broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), None, Some(&grant)).is_some());
        assert!(host.asked().is_empty());
        assert!(broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), None, Some(&grant)).is_none(), "only the granted key");

        let grant = Grant { slot_id: ENC_ID.into() };
        host.answer(with_passphrase("test-passphrase", false));
        assert!(broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), None, Some(&grant)).is_some());
        let asked = host.asked();
        assert!(asked[0].preapproved && asked[0].needs_passphrase && !asked[0].rememberable);

        host.answer(allow(false));
        broker.sign(&host, &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("claude")), None);
        assert_eq!(host.asked().len(), 2, "a grant is not a remembered approval");
    }

    #[test]
    fn identical_requests_at_the_same_time_share_one_prompt() {
        let broker = Arc::new(Broker::default());
        let host = Arc::new(FakeHost::new());
        *host.gate.lock().unwrap() = false;
        host.answer(allow(false));
        let sign = |broker: Arc<Broker>, host: Arc<FakeHost>| {
            std::thread::spawn(move || {
                broker.sign(host.as_ref(), &request(test_keys::PLAIN_PUBLIC, Some(host_key())), Some(&program("git")), None)
            })
        };
        let first = sign(Arc::clone(&broker), Arc::clone(&host));
        while host.asked().is_empty() {
            std::thread::sleep(Duration::from_millis(5));
        }
        let others: Vec<_> = (0..3).map(|_| sign(Arc::clone(&broker), Arc::clone(&host))).collect();
        while broker.waiting() < 3 {
            std::thread::sleep(Duration::from_millis(5));
        }
        *host.gate.lock().unwrap() = true;
        host.opened.notify_all();
        assert!(first.join().unwrap().is_some());
        for other in others {
            assert!(other.join().unwrap().is_some(), "the shared answer allowed it");
        }
        assert_eq!(host.asked().len(), 1, "one prompt for all four");
    }

    #[test]
    fn a_very_long_user_name_is_shortened_for_the_window() {
        let broker = Broker::default();
        let host = FakeHost::new();
        host.answer(allow(false));
        let mut long = request(test_keys::PLAIN_PUBLIC, Some(host_key()));
        long.user = Some("u".repeat(5000));
        broker.sign(&host, &long, Some(&program("claude")), None);
        let shown = host.asked()[0].user.clone().unwrap();
        assert_eq!(shown.chars().count(), 257);
        assert!(shown.ends_with('…'));
    }

    #[test]
    fn clear_forgets_approvals_and_opened_keys() {
        let broker = Broker::default();
        let host = FakeHost::new();
        host.answer(with_passphrase("test-passphrase", false));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        broker.clear();
        host.answer(with_passphrase("test-passphrase", false));
        broker.sign(&host, &request(test_keys::ENC_PUBLIC, Some(host_key())), Some(&program("claude")), None).unwrap();
        let again = &host.asked()[1];
        assert!(!again.preapproved && again.needs_passphrase, "asks for the approval and the passphrase again");
    }
}
```

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib agent::broker -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`Broker`, `AgentHost`, … not found).

- [ ] **Step 3: Implement**

Put above the tests in `src-tauri/src/agent/broker.rs`:

```rust
//! agent 的決定(金鑰保管庫 spec §5.2、§5.3、§5.5、§5.6):列出哪些金鑰、簽不簽、要不要問、passphrase 與解開的私鑰。不碰 Tauri:金鑰清單、
//! 保管庫、keychain、時鐘、known_hosts 與核准視窗都經 `AgentHost`(production 是 `agent::AppAgentHost`)。

use std::collections::HashMap;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::{Arc, Condvar, Mutex, PoisonError};
use std::time::Duration;

use ssh_key::public::KeyData;
use ssh_key::HashAlg;
use zeroize::Zeroizing;

use crate::agent::approval::{remember_minutes, verdict, ApprovalCache, ApprovalKey, KeyProtection, Verdict};
use crate::agent::peer::Program;
use crate::agent::prompt::{AgentApprovalAnswer, AgentApprovalRequest, APPROVAL_TIMEOUT};
use crate::agent::session::{SignAuthority, SignRequest};
use crate::error::AppError;
use crate::sync::env::Keychain;
use crate::sync::state_v2::{SlotSource, SyncStateV2};
use crate::vault::material::{self, Material, OpenError};
use crate::vault::store::AgentSettings;

/// 輸錯幾次就拒絕這次請求(spec §5.5)。
pub const PASSPHRASE_ATTEMPTS: usize = 3;
pub const WRONG_PASSPHRASE: &str = "That passphrase didn't work.";

/// 記在這台的 passphrase 在 keychain 的 account(spec §4.3)。
pub fn passphrase_account(slot_id: &str) -> String {
    format!("vault:passphrase:{slot_id}")
}

/// 這台只在 SSHelter 的一把金鑰(同步狀態裡 `SlotSource::Vault` 的插槽):列出金鑰、比對簽章請求都不必開保管庫。
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct VaultKey {
    pub slot_id: String,
    /// 顯示名稱:插槽的名稱,沒有就用插槽檔名。
    pub name: String,
    pub fingerprint: String,
    pub public_key: String,
    pub has_passphrase: bool,
}

/// 同步狀態裡這台只在 SSHelter 的金鑰(依插槽 id)。
pub fn vault_keys(state: &SyncStateV2) -> Vec<VaultKey> {
    state
        .key_slots
        .iter()
        .filter_map(|(id, local)| match &local.source {
            Some(SlotSource::Vault { fingerprint, public_key, has_passphrase }) => Some(VaultKey {
                slot_id: id.clone(),
                name: local
                    .payload
                    .as_ref()
                    .map(|p| p.name.clone())
                    .filter(|name| !name.is_empty())
                    .unwrap_or_else(|| local.file_name.clone()),
                fingerprint: fingerprint.clone(),
                public_key: public_key.clone(),
                has_passphrase: *has_passphrase,
            }),
            _ => None,
        })
        .collect()
}

/// known_hosts 的內容裡,這把主機金鑰第一個能顯示的名稱(`[host]:port` 顯示成 `host:port`)。雜湊過的名稱(`|1|…`)、萬用字元與否定的
/// pattern 不算;`@cert-authority`、`@revoked` 的行不算。
pub fn host_name_in(known_hosts: &str, host_key: &KeyData) -> Option<String> {
    for line in known_hosts.lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') || line.starts_with('@') {
            continue;
        }
        let mut fields = line.split_whitespace();
        let (Some(names), Some(kind), Some(blob)) = (fields.next(), fields.next(), fields.next()) else { continue };
        if material::public_key_data(&format!("{kind} {blob}")).as_ref() != Some(host_key) {
            continue;
        }
        let shown = names.split(',').find(|name| !name.starts_with('|') && !name.starts_with('!') && !name.contains(['*', '?']));
        if let Some(name) = shown {
            return Some(match name.strip_prefix('[').and_then(|rest| rest.split_once("]:")) {
                Some((host, port)) => format!("{host}:{port}"),
                None => name.to_string(),
            });
        }
    }
    None
}

/// agent 與外界的邊界(production:`agent::AppAgentHost`;測試:替身)。
pub trait AgentHost: Send + Sync {
    /// 這台只在 SSHelter 的金鑰。
    fn keys(&self) -> Vec<VaultKey>;
    /// 保管庫裡這把金鑰的私鑰原文(有 passphrase 的仍是加密狀態)。
    fn private_key(&self, slot_id: &str) -> Result<Option<Zeroizing<String>>, AppError>;
    /// 這台的 agent 設定(保管庫檔頭);讀不到用預設值。
    fn settings(&self) -> AgentSettings;
    fn keychain(&self) -> &dyn Keychain;
    fn now_ms(&self) -> u64;
    /// 顯示用的主機名稱(known_hosts)。
    fn host_name(&self, host_key: &KeyData) -> Option<String>;
    /// 問使用者(核准視窗,最多 60 秒);逾時或沒有回答 → None。
    fn ask(&self, request: AgentApprovalRequest) -> Option<AgentApprovalAnswer>;
}

/// Connect 的一次性通道(spec §5.6):只提供這把金鑰,已經核准。
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct Grant {
    pub slot_id: String,
}

/// 解開的私鑰(沒記住 passphrase 的金鑰):留到記住的時間結束(spec §5.5)。
struct Unlocked {
    material: Arc<Material>,
    expires_ms: u64,
}

/// 正在問的「金鑰 × 主機 × 程式」:同時到的相同請求等這一個答案(spec §5.3)。
#[derive(Default)]
struct Pending {
    answer: Mutex<Option<bool>>,
    done: Condvar,
    followers: AtomicUsize,
}

impl Pending {
    fn finish(&self, allowed: bool) {
        *self.answer.lock().unwrap_or_else(PoisonError::into_inner) = Some(allowed);
        self.done.notify_all();
    }

    /// 等第一個請求的答案;等太久(視窗本身最多 60 秒)當成拒絕。
    fn wait(&self) -> bool {
        self.followers.fetch_add(1, Ordering::SeqCst);
        let answer = self.answer.lock().unwrap_or_else(PoisonError::into_inner);
        let (answer, _) = self
            .done
            .wait_timeout_while(answer, APPROVAL_TIMEOUT + Duration::from_secs(5), |answer| answer.is_none())
            .unwrap_or_else(PoisonError::into_inner);
        answer.unwrap_or(false)
    }
}

/// 第一個請求問完(或 panic)時,讓等它的請求拿到答案,並從 `asking` 拿掉。
struct First<'a> {
    broker: &'a Broker,
    key: ApprovalKey,
    pending: Arc<Pending>,
    allowed: bool,
}

impl Drop for First<'_> {
    fn drop(&mut self) {
        self.broker.asking.lock().unwrap_or_else(PoisonError::into_inner).remove(&self.key);
        self.pending.finish(self.allowed);
    }
}

/// agent 在記憶體裡的狀態(`AgentRuntime::broker`):記住的核准、解開的私鑰、正在問的請求。螢幕鎖定(Plan 3)時 `clear`;SSHelter 結束時
/// 跟著行程消失。
#[derive(Default)]
pub struct Broker {
    approvals: Mutex<ApprovalCache>,
    unlocked: Mutex<HashMap<String, Unlocked>>,
    asking: Mutex<HashMap<ApprovalKey, Arc<Pending>>>,
}

/// 視窗裡顯示的使用者名稱最多幾個字元:它來自 ssh 送來的資料,可能很長。
const MAX_SHOWN_CHARS: usize = 256;

/// 最多 `max` 個字元,超過的部分換成「…」。
fn shorten(text: &str, max: usize) -> String {
    if text.chars().count() <= max {
        return text.to_string();
    }
    let mut out: String = text.chars().take(max).collect();
    out.push('…');
    out
}

/// 一個簽章請求要顯示的一切。
struct Ask<'a> {
    key: &'a VaultKey,
    request: &'a SignRequest,
    program: Option<&'a Program>,
    host_fingerprint: Option<String>,
    host_display: Option<String>,
    minutes: u32,
}

impl Ask<'_> {
    fn prompt(&self, preapproved: bool, needs_passphrase: bool, rememberable: bool, passphrase_error: Option<String>) -> AgentApprovalRequest {
        AgentApprovalRequest {
            id: String::new(),
            key_name: self.key.name.clone(),
            key_fingerprint: self.key.fingerprint.clone(),
            program_chain: self.program.map(|p| p.chain.clone()).unwrap_or_default(),
            user: self.request.user.as_deref().map(|user| shorten(user, MAX_SHOWN_CHARS)),
            host: self.host_display.clone(),
            host_fingerprint: self.host_fingerprint.clone(),
            rememberable,
            remember_minutes: self.minutes,
            needs_passphrase,
            passphrase_error,
            preapproved,
        }
    }
}

impl Broker {
    /// 列出的金鑰:這台只在 SSHelter 的金鑰;一次性通道只列它那一把。列出不必核准(公鑰不是祕密)。
    pub fn identities(&self, host: &dyn AgentHost, grant: Option<&Grant>) -> Vec<(KeyData, String)> {
        host.keys()
            .into_iter()
            .filter(|key| grant.is_none_or(|g| g.slot_id == key.slot_id))
            .filter_map(|key| material::public_key_data(&key.public_key).map(|data| (data, key.name)))
            .collect()
    }

    /// 簽或不簽:回傳 signature blob,不簽 → None。可能等核准視窗。
    pub fn sign(&self, host: &dyn AgentHost, request: &SignRequest, program: Option<&Program>, grant: Option<&Grant>) -> Option<Vec<u8>> {
        // 轉送到遠端的 agent:一律拒絕(spec §3、§5.2)。
        if request.forwarded {
            return None;
        }
        let keys = host.keys();
        let key = keys.iter().find(|k| material::public_key_data(&k.public_key).as_ref() == Some(&request.key))?;
        if grant.is_some_and(|g| g.slot_id != key.slot_id) {
            return None;
        }
        let settings = host.settings();
        let host_fingerprint = request.host_key.as_ref().map(|k| k.fingerprint(HashAlg::Sha256).to_string());
        let host_display = request
            .host_key
            .as_ref()
            .zip(host_fingerprint.as_ref())
            .map(|(k, fingerprint)| host.host_name(k).unwrap_or_else(|| fingerprint.clone()));
        let ask = Ask { key, request, program, host_fingerprint, host_display, minutes: remember_minutes(&settings) };
        let material = if grant.is_some() {
            // Connect:已經核准,不跳核准視窗、不算進記住的核准;只可能要 passphrase。
            self.unlock(host, &ask, None)?
        } else {
            self.approve(host, &ask, &settings)?
        };
        material.sign(&request.data, request.flags).ok()
    }

    /// 忘掉記住的核准與解開的私鑰(螢幕鎖定;spec §5.3)。
    pub fn clear(&self) {
        self.approvals.lock().unwrap().clear();
        self.unlocked.lock().unwrap().clear();
    }

    /// 核准這次請求並解開私鑰;沒有允許(拒絕、逾時、passphrase 錯三次)→ None。
    fn approve(&self, host: &dyn AgentHost, ask: &Ask, settings: &AgentSettings) -> Option<Arc<Material>> {
        let approval_key = match (&ask.host_fingerprint, ask.program) {
            (Some(host_fingerprint), Some(program)) => Some(ApprovalKey {
                key_fingerprint: ask.key.fingerprint.clone(),
                host_fingerprint: host_fingerprint.clone(),
                program: program.identity.clone(),
            }),
            // 未知的主機或認不出的程式:問,而且不記住(spec §5.3、§5.4)。
            _ => None,
        };
        let now = host.now_ms();
        let remembered = approval_key.as_ref().is_some_and(|k| self.approvals.lock().unwrap().is_remembered(k, now));
        match verdict(KeyProtection::default(), settings, approval_key.is_some(), remembered) {
            Verdict::Remembered => self.unlock(host, ask, None),
            Verdict::Ask { rememberable: false } => self.ask_and_unlock(host, ask, None),
            Verdict::Ask { rememberable: true } => match approval_key {
                Some(approval_key) => self.ask_once(host, ask, approval_key),
                None => self.ask_and_unlock(host, ask, None),
            },
        }
    }

    /// 同一個「金鑰 × 主機 × 程式」同時只問一次:第一個請求問,其他的等它的答案(允許就各自解開、簽章)。
    fn ask_once(&self, host: &dyn AgentHost, ask: &Ask, approval_key: ApprovalKey) -> Option<Arc<Material>> {
        let (pending, first) = {
            let mut asking = self.asking.lock().unwrap();
            match asking.get(&approval_key) {
                Some(pending) => (Arc::clone(pending), false),
                None => {
                    let pending = Arc::new(Pending::default());
                    asking.insert(approval_key.clone(), Arc::clone(&pending));
                    (pending, true)
                }
            }
        };
        if !first {
            return if pending.wait() { self.unlock(host, ask, None) } else { None };
        }
        let mut first = First { broker: self, key: approval_key, pending, allowed: false };
        // 上一個相同的請求可能在這個請求查過之後才記住:再查一次。
        let material = if self.approvals.lock().unwrap().is_remembered(&first.key, host.now_ms()) {
            self.unlock(host, ask, None)
        } else {
            self.ask_and_unlock(host, ask, Some(&first.key))
        };
        first.allowed = material.is_some();
        material
    }

    /// 跳核准視窗,需要 passphrase 就一起問。允許了才解開私鑰;要記住的,解開之後才記住。
    fn ask_and_unlock(&self, host: &dyn AgentHost, ask: &Ask, approval_key: Option<&ApprovalKey>) -> Option<Arc<Material>> {
        let needs_passphrase = ask.key.has_passphrase
            && self.cached(&ask.key.slot_id, host.now_ms()).is_none()
            && !matches!(host.keychain().get(&passphrase_account(&ask.key.slot_id)), Ok(Some(_)));
        let answer = host.ask(ask.prompt(false, needs_passphrase, approval_key.is_some(), None))?;
        if !answer.allow {
            return None;
        }
        let supplied = answer
            .passphrase
            .filter(|_| needs_passphrase)
            .map(|passphrase| (Zeroizing::new(passphrase), answer.remember_passphrase));
        let material = self.unlock(host, ask, supplied)?;
        if let (Some(key), true) = (approval_key, answer.remember) {
            self.remember(key.clone(), &ask.key.slot_id, host.now_ms(), ask.minutes);
        }
        Some(material)
    }

    /// 解開這把金鑰:記憶體裡解開的 → 不需要 passphrase → keychain 記住的 passphrase(不對就刪掉)→ 核准視窗帶回來的(`supplied`)或
    /// 再問(`preapproved`:只要 passphrase),一共三次。
    fn unlock(&self, host: &dyn AgentHost, ask: &Ask, mut supplied: Option<(Zeroizing<String>, bool)>) -> Option<Arc<Material>> {
        let slot_id = &ask.key.slot_id;
        if let Some(material) = self.cached(slot_id, host.now_ms()) {
            return Some(material);
        }
        let text = host.private_key(slot_id).ok().flatten()?;
        match material::open(&text, None) {
            Ok(material) => return Some(Arc::new(material)),
            Err(OpenError::NeedsPassphrase) => {}
            Err(_) => return None,
        }
        let account = passphrase_account(slot_id);
        if supplied.is_none() {
            if let Ok(Some(saved)) = host.keychain().get(&account) {
                let saved = Zeroizing::new(saved);
                match material::open(&text, Some(&saved)) {
                    Ok(material) => return Some(Arc::new(material)),
                    Err(OpenError::WrongPassphrase) => {
                        let _ = host.keychain().delete(&account);
                    }
                    Err(_) => return None,
                }
            }
        }
        let mut error = None;
        for _ in 0..PASSPHRASE_ATTEMPTS {
            let (passphrase, remember) = match supplied.take() {
                Some(given) => given,
                None => {
                    let answer = host.ask(ask.prompt(true, true, false, error.take()))?;
                    if !answer.allow {
                        return None;
                    }
                    (Zeroizing::new(answer.passphrase.unwrap_or_default()), answer.remember_passphrase)
                }
            };
            match material::open(&text, Some(&passphrase)) {
                Ok(material) => {
                    let material = Arc::new(material);
                    // 記住了:每次簽章時才用它解開,不留解開的私鑰(spec §5.5)。存不進 keychain 就留在記憶體。
                    let saved = remember && host.keychain().set(&account, &passphrase).is_ok();
                    if !saved {
                        let expires_ms = host.now_ms().saturating_add(u64::from(ask.minutes) * 60_000);
                        self.unlocked.lock().unwrap().insert(slot_id.clone(), Unlocked { material: Arc::clone(&material), expires_ms });
                    }
                    return Some(material);
                }
                Err(OpenError::WrongPassphrase) => error = Some(WRONG_PASSPHRASE.to_string()),
                Err(_) => return None,
            }
        }
        None
    }

    fn cached(&self, slot_id: &str, now_ms: u64) -> Option<Arc<Material>> {
        let mut unlocked = self.unlocked.lock().unwrap();
        unlocked.retain(|_, u| u.expires_ms > now_ms);
        unlocked.get(slot_id).map(|u| Arc::clone(&u.material))
    }

    /// 記住核准;這把金鑰解開的私鑰至少留到這個核准到期(spec §5.5)。
    fn remember(&self, key: ApprovalKey, slot_id: &str, now_ms: u64, minutes: u32) {
        self.approvals.lock().unwrap().remember(key, now_ms, minutes);
        let until = now_ms.saturating_add(u64::from(minutes) * 60_000);
        if let Some(unlocked) = self.unlocked.lock().unwrap().get_mut(slot_id) {
            unlocked.expires_ms = unlocked.expires_ms.max(until);
        }
    }

    /// 正在等別人答案的請求數(測試用)。
    #[cfg(test)]
    fn waiting(&self) -> usize {
        self.asking.lock().unwrap().values().map(|p| p.followers.load(Ordering::SeqCst)).sum()
    }
}

/// 一條連線的 `SignAuthority`:連上的程式(連線時辨識一次,`peer`)與一次性通道的授權。
pub struct Connection<'a> {
    pub broker: &'a Broker,
    pub host: &'a dyn AgentHost,
    pub program: Option<Program>,
    pub grant: Option<Grant>,
}

impl SignAuthority for Connection<'_> {
    fn identities(&self) -> Vec<(KeyData, String)> {
        self.broker.identities(self.host, self.grant.as_ref())
    }

    fn sign(&self, request: &SignRequest) -> Option<Vec<u8>> {
        self.broker.sign(self.host, request, self.program.as_ref(), self.grant.as_ref())
    }
}
```

In `src-tauri/src/agent/mod.rs`, add `pub mod broker;` and the field `pub broker: broker::Broker,` to `AgentRuntime` (it derives `Default`).

If the compiler says `Material` is not `Send`/`Sync` (needed because `AgentRuntime` lives in Tauri's managed state), stop and report it instead of wrapping it in something unsafe.

- [ ] **Step 4: Run the tests to verify they pass**

Run: same command as Step 2.
Expected: PASS (16 tests). The parallel test must pass 20 runs in a row: `for i in $(seq 20); do cargo test --offline --lib agent::broker::tests::identical -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain -q || break; done`.

- [ ] **Step 5: Run the full Rust suite and commit**

```bash
git add src-tauri/src/agent
git commit -m "feat(agent): decide which keys to offer, when to ask and how to unlock them"
```

---

### Task 9: The agent's endpoint and start-up

**Files:**
- Modify: `src-tauri/Cargo.toml` (`windows-sys` features `Win32_System_Pipes`, `Win32_System_IO`)
- Create: `src-tauri/src/agent/server.rs`, `src-tauri/src/agent/pipe_windows.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`agent_dir`, `identity_agent_value`, `AgentStatus`, `AppAgentHost`, `start`)
- Modify: `src-tauri/src/sync/slot_files_windows.rs` (`current_user_token` becomes `pub(crate)`)
- Modify: `src-tauri/src/lib.rs` (start the agent in `setup`)

**Interfaces:**
- Consumes: Task 4 `crate::agent::session::serve`; Task 5 `crate::agent::peer::{process_chain, identify}`; Task 6 `crate::agent::prompt::{TauriPromptSurface, APPROVAL_TIMEOUT}`, `AgentRuntime`; Task 8 `crate::agent::broker::{AgentHost, Broker, Connection, VaultKey, vault_keys, host_name_in}`; Task 1 `crate::vault::store::{vault_path, with_vault, AgentSettings}`; `crate::sync::engine::with_env`; `crate::sync::slot_files::ensure_keys_dir`.
- Produces:
  - `crate::agent::agent_dir(home: &Path) -> PathBuf` (`<home>/.ssh/sshelter/agent`)
  - `#[cfg(unix)] crate::agent::SOCKET_VALUE: &str = "~/.ssh/sshelter/agent/sock"`
  - `crate::agent::identity_agent_value() -> Result<String, AppError>` (Unix: `SOCKET_VALUE`; Windows: `//./pipe/<pipe_name()>`)
  - `#[derive(Clone, Debug, Default, PartialEq, Eq)] pub enum AgentStatus { #[default] NotStarted, Running, OtherInstance, Failed(String) }`; `AgentRuntime.status: Mutex<AgentStatus>`
  - `crate::agent::AppAgentHost { pub app: tauri::AppHandle }` implementing `AgentHost`
  - `crate::agent::start(app: &tauri::AppHandle)`
  - `crate::agent::server::{MAX_CONNECTIONS = 64, Stream, Handler, Started { Running, OtherInstance }, take_lock, dispatch, listen_unix}`; `#[cfg(unix)] pub(crate) fn check_socket_path(&Path) -> Result<(), AppError>`; `#[cfg(unix)] pub(crate) fn peer(&UnixStream) -> Result<Option<u32>, ()>`; `#[cfg(unix)] pub(crate) const IDLE_TIMEOUT: Duration` (5 minutes); `#[cfg(test)] server::testing::{NoKeys, serving}`
  - `#[cfg(windows)] crate::agent::pipe_windows::{pipe_name() -> io::Result<String>, listen(dir, name, handle) -> Result<Started, AppError>}`; `pub(crate) fn accept(&OwnedHandle) -> io::Result<Option<u32>>`; private `create_instance(path, security, first, max_instances)`, `wide_pipe_path(name)`, `OwnerOnly` (Task 12 adds a one-shot pipe next to them)

Rules (spec §4.4, §5.1, §5.7, §11):
- One agent per user: whoever holds `<agent dir>/lock` (`File::try_lock`, released by the OS when the process dies) serves; a second SSHelter returns `OtherInstance` and touches nothing. The lock is taken before anything at the endpoint is changed.
- Unix: the socket path is checked against the platform limit (104 bytes on macOS, 108 on Linux, including the NUL) before anything else; a leftover `sock` is removed only by the lock holder; the socket is 0600 inside the 0700 directory; a peer whose effective UID differs is closed at once; the peer PID comes from `LOCAL_PEEREPID` (falling back to `LOCAL_PEERPID`) on macOS and `SO_PEERCRED` on Linux.
- Windows: `\\.\pipe\sshelter-agent-<first 16 hex of SHA-256 of the user's SID string>`; every instance gets the owner-only DACL and `PIPE_REJECT_REMOTE_CLIENTS`; the first uses `FILE_FLAG_FIRST_PIPE_INSTANCE`, so a name someone else already holds is an error (spec §11); the next instance is created before a connection is served; the client PID comes from `GetNamedPipeClientProcessId`.
- At most 64 connections at once; more are closed. Each connection runs on its own thread: the program is identified once (Task 5), then `session::serve` runs with a `Connection` (Task 8).
- `start` also runs a thread that calls `Broker::expire(now)` once a minute (Task 8's review: opened keys and remembered approvals are dropped when their window ends, not at the next request).
- The agent starts in `run_app`'s `setup` for both the normal app and `--mcp-host` (both open windows). The `--mcp` stdio adapter never builds Tauri and never starts it. A failure is stored in `AgentRuntime.status` and logged; SSHelter keeps working.

- [ ] **Step 1: Add the Windows features and expose the token helper**

In `src-tauri/Cargo.toml`, add `"Win32_System_IO"` and `"Win32_System_Pipes"` to the `windows-sys` feature list (`ConnectNamedPipe` takes an `OVERLAPPED` pointer, which windows-sys gates behind `Win32_System_IO`). In `src-tauri/src/sync/slot_files_windows.rs`, change `fn current_user_token()` to `pub(crate) fn current_user_token()`.

- [ ] **Step 2: Write the failing Unix tests**

Add `pub mod server;` and `#[cfg(windows)] pub mod pipe_windows;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/server.rs` with the shared test helpers and the Unix tests first:

```rust
/// 兩個平台的端點測試共用:不提供任何金鑰的 authority,以及把對方 PID 送出來再服務連線的 handler。
#[cfg(test)]
pub(crate) mod testing {
    use std::sync::mpsc::Sender;
    use std::sync::{Arc, Mutex};

    use ssh_key::public::KeyData;

    use super::Handler;
    use crate::agent::session::{serve, SignAuthority, SignRequest};

    pub struct NoKeys;

    impl SignAuthority for NoKeys {
        fn identities(&self) -> Vec<(KeyData, String)> {
            Vec::new()
        }
        fn sign(&self, _request: &SignRequest) -> Option<Vec<u8>> {
            None
        }
    }

    pub fn serving(pids: Sender<Option<u32>>) -> Handler {
        let pids = Mutex::new(pids);
        Arc::new(move |mut stream, pid| {
            let _ = pids.lock().unwrap().send(pid);
            let _ = serve(&mut stream, &NoKeys);
        })
    }
}

#[cfg(all(test, unix))]
mod tests {
    use super::testing::serving;
    use super::*;
    use crate::agent::protocol::{read_frame, write_frame, SSH_AGENTC_REQUEST_IDENTITIES, SSH_AGENT_IDENTITIES_ANSWER};
    use std::os::unix::fs::PermissionsExt;
    use std::os::unix::net::{UnixListener, UnixStream};
    use std::sync::mpsc;
    use std::time::Duration;

    /// macOS 的暫存目錄路徑很長,socket 路徑有 104 bytes 的上限:測試用 /tmp 底下的短目錄。
    fn short_dir() -> tempfile::TempDir {
        tempfile::Builder::new().prefix("sa").tempdir_in("/tmp").unwrap()
    }

    fn identities(sock: &Path) -> u8 {
        let mut stream = UnixStream::connect(sock).unwrap();
        write_frame(&mut stream, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();
        read_frame(&mut stream).unwrap().unwrap()[0]
    }

    #[test]
    fn serves_owner_only_and_hands_over_the_peer_pid() {
        let dir = short_dir();
        let agent = dir.path().join("agent");
        let (tx, rx) = mpsc::channel();
        assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::Running);
        assert_eq!(identities(&agent.join("sock")), SSH_AGENT_IDENTITIES_ANSWER);
        assert_eq!(rx.recv_timeout(Duration::from_secs(5)).unwrap(), Some(std::process::id()));
        assert_eq!(std::fs::metadata(&agent).unwrap().permissions().mode() & 0o777, 0o700);
        assert_eq!(std::fs::metadata(agent.join("sock")).unwrap().permissions().mode() & 0o777, 0o600);
    }

    #[test]
    fn a_second_agent_does_not_start_and_the_first_keeps_serving() {
        let dir = short_dir();
        let agent = dir.path().join("agent");
        let (tx, _rx) = mpsc::channel();
        assert_eq!(listen_unix(&agent, serving(tx.clone())).unwrap(), Started::Running);
        assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::OtherInstance);
        assert_eq!(identities(&agent.join("sock")), SSH_AGENT_IDENTITIES_ANSWER, "the first one still answers");
    }

    #[test]
    fn a_stale_socket_left_by_a_crash_is_replaced() {
        let dir = short_dir();
        let agent = dir.path().join("agent");
        std::fs::create_dir_all(&agent).unwrap();
        drop(UnixListener::bind(agent.join("sock")).unwrap());
        assert!(UnixStream::connect(agent.join("sock")).is_err(), "nobody answers on it");
        let (tx, _rx) = mpsc::channel();
        assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::Running);
        assert_eq!(identities(&agent.join("sock")), SSH_AGENT_IDENTITIES_ANSWER);
    }

    #[test]
    fn a_socket_path_over_the_limit_is_refused_with_a_clear_message() {
        let dir = short_dir();
        let agent = dir.path().join("a".repeat(120));
        let (tx, _rx) = mpsc::channel();
        let err = listen_unix(&agent, serving(tx)).unwrap_err().to_string();
        assert!(err.contains("too long"), "{err}");
        assert!(!agent.exists(), "nothing was created");
    }
}
```

- [ ] **Step 3: Run them to verify they fail**

Run: `cargo test --offline --lib agent::server -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`listen_unix`, `Started`, `Handler` not found).

- [ ] **Step 4: Implement `server.rs`**

Put above the test modules:

```rust
//! agent 的端點(金鑰保管庫 spec §4.4、§5.1):Unix socket,或 Windows 的 named pipe(`pipe_windows`)。一個使用者只有一個 agent:先拿到
//! `<agent 目錄>/lock` 的 SSHelter 提供,持有到行程結束(當掉也由系統釋放),另一個 SSHelter 不開、什麼都不碰。每條連線一條執行緒(同 MCP bridge)。

use std::fs::{File, OpenOptions, TryLockError};
use std::path::Path;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::Arc;

use crate::error::AppError;
use crate::sync::slot_files;

/// 同時處理的連線上限;超過的直接關掉(同一使用者的程式可以一直連上來)。
pub const MAX_CONNECTIONS: usize = 64;

/// 一條連線的串流。
#[cfg(unix)]
pub type Stream = std::os::unix::net::UnixStream;
#[cfg(windows)]
pub type Stream = std::fs::File;

/// 處理一條連線(在連線自己的執行緒裡):串流與對方的 PID(拿不到 → None)。
pub type Handler = Arc<dyn Fn(Stream, Option<u32>) + Send + Sync>;

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Started {
    Running,
    /// 另一個 SSHelter 已經在提供 agent。
    OtherInstance,
}

/// 拿 `dir/lock` 的鎖;別的 SSHelter 拿著 → None。`dir` 一併建好(只有目前使用者能存取)。
pub(crate) fn take_lock(dir: &Path) -> Result<Option<File>, AppError> {
    slot_files::ensure_keys_dir(dir)?;
    let file = OpenOptions::new().create(true).truncate(false).write(true).open(dir.join("lock"))?;
    match file.try_lock() {
        Ok(()) => Ok(Some(file)),
        Err(TryLockError::WouldBlock) => Ok(None),
        Err(TryLockError::Error(e)) => Err(e.into()),
    }
}

/// 交給連線自己的執行緒;超過上限就關掉。
pub(crate) fn dispatch(stream: Stream, pid: Option<u32>, active: &Arc<AtomicUsize>, handle: &Handler) {
    struct Slot(Arc<AtomicUsize>);
    impl Drop for Slot {
        fn drop(&mut self) {
            self.0.fetch_sub(1, Ordering::SeqCst);
        }
    }
    if active.fetch_add(1, Ordering::SeqCst) >= MAX_CONNECTIONS {
        active.fetch_sub(1, Ordering::SeqCst);
        return;
    }
    let slot = Slot(Arc::clone(active));
    let handle = Arc::clone(handle);
    // spawn 失敗時 closure(連同 `slot` 與串流)被丟掉:計數照樣減回去,連線關掉。
    let _ = std::thread::Builder::new().name("sshelter-agent-connection".to_string()).spawn(move || {
        let _slot = slot;
        handle(stream, pid);
    });
}

/// Unix socket 路徑的上限(含結尾的 NUL):macOS 104、Linux 108(spec §4.4、§15)。
#[cfg(unix)]
const SUN_PATH_MAX: usize = if cfg!(target_os = "macos") { 104 } else { 108 };

/// 一條連線兩則訊息之間最多閒置多久(等核准視窗時沒有在讀,不算):ssh 認證完就關掉它的 agent 連線,閒置的連線不該一直佔著
/// `MAX_CONNECTIONS` 的名額。Windows 的 pipe(`File`)沒有讀取逾時,不設。
#[cfg(unix)]
pub(crate) const IDLE_TIMEOUT: std::time::Duration = std::time::Duration::from_secs(300);

/// socket 路徑放不放得進 `sun_path`;放不進就說清楚(`bind` 自己的錯誤訊息看不出原因)。
#[cfg(unix)]
pub(crate) fn check_socket_path(sock: &Path) -> Result<(), AppError> {
    let length = sock.as_os_str().len();
    if length + 1 > SUN_PATH_MAX {
        return Err(AppError::Other(format!(
            "The agent's socket path is too long ({length} bytes; the limit is {}): {}",
            SUN_PATH_MAX - 1,
            sock.display()
        )));
    }
    Ok(())
}

/// 在 `dir`(`~/.ssh/sshelter/agent`)開 `sock`,接受同一使用者的連線,每條交給 `handle`。另一個 SSHelter 已經在提供 → `OtherInstance`;
/// 沒人接聽的舊 `sock`(上次當掉留下的)由拿到鎖的這一個換掉。
#[cfg(unix)]
pub fn listen_unix(dir: &Path, handle: Handler) -> Result<Started, AppError> {
    use std::os::unix::fs::PermissionsExt;
    use std::os::unix::net::UnixListener;

    let sock = dir.join("sock");
    check_socket_path(&sock)?;
    let Some(lock) = take_lock(dir)? else { return Ok(Started::OtherInstance) };
    match std::fs::remove_file(&sock) {
        Ok(()) => {}
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => {}
        Err(e) => return Err(e.into()),
    }
    let listener = UnixListener::bind(&sock)?;
    std::fs::set_permissions(&sock, std::fs::Permissions::from_mode(0o600))?;
    std::thread::Builder::new().name("sshelter-agent".to_string()).spawn(move || {
        // 鎖跟著 listener 持有到行程結束。
        let _lock = lock;
        let active = Arc::new(AtomicUsize::new(0));
        for stream in listener.incoming() {
            let stream = match stream {
                Ok(stream) => stream,
                Err(_) => {
                    // 例如檔案描述元用完:稍等再接,不要空轉。
                    std::thread::sleep(std::time::Duration::from_millis(50));
                    continue;
                }
            };
            // 不是同一個使用者:直接關掉。
            let Ok(pid) = peer(&stream) else { continue };
            let _ = stream.set_read_timeout(Some(IDLE_TIMEOUT));
            dispatch(stream, pid, &active, &handle);
        }
    })?;
    Ok(Started::Running)
}

/// 對方是同一個使用者(有效 UID)時回傳它的 PID(拿不到 → None);不是 → Err。
#[cfg(target_os = "macos")]
pub(crate) fn peer(stream: &std::os::unix::net::UnixStream) -> Result<Option<u32>, ()> {
    use std::os::fd::AsRawFd;
    let fd = stream.as_raw_fd();
    let mut uid: libc::uid_t = 0;
    let mut gid: libc::gid_t = 0;
    // SAFETY: getpeereid writes two integers; geteuid has no preconditions.
    if unsafe { libc::getpeereid(fd, &mut uid, &mut gid) } != 0 || uid != unsafe { libc::geteuid() } {
        return Err(());
    }
    let local_pid = |option: libc::c_int| {
        let mut pid: libc::pid_t = 0;
        let mut len = std::mem::size_of::<libc::pid_t>() as libc::socklen_t;
        // SAFETY: the buffer is one pid_t and `len` says so.
        let rc = unsafe { libc::getsockopt(fd, libc::SOL_LOCAL, option, &mut pid as *mut _ as *mut libc::c_void, &mut len) };
        (rc == 0 && pid > 0).then_some(pid as u32)
    };
    Ok(local_pid(libc::LOCAL_PEEREPID).or_else(|| local_pid(libc::LOCAL_PEERPID)))
}

#[cfg(target_os = "linux")]
pub(crate) fn peer(stream: &std::os::unix::net::UnixStream) -> Result<Option<u32>, ()> {
    use std::os::fd::AsRawFd;
    // SAFETY: `ucred` is plain old data; getsockopt fills at most `len` bytes.
    let mut cred: libc::ucred = unsafe { std::mem::zeroed() };
    let mut len = std::mem::size_of::<libc::ucred>() as libc::socklen_t;
    let rc = unsafe {
        libc::getsockopt(stream.as_raw_fd(), libc::SOL_SOCKET, libc::SO_PEERCRED, &mut cred as *mut _ as *mut libc::c_void, &mut len)
    };
    if rc != 0 || cred.uid != unsafe { libc::geteuid() } {
        return Err(());
    }
    Ok(u32::try_from(cred.pid).ok().filter(|pid| *pid > 0))
}

/// 其他 Unix:認不出對方,一律關掉。
#[cfg(all(unix, not(any(target_os = "macos", target_os = "linux"))))]
pub(crate) fn peer(_stream: &std::os::unix::net::UnixStream) -> Result<Option<u32>, ()> {
    Err(())
}
```

- [ ] **Step 5: Run the Unix tests to verify they pass**

Run: same command as Step 3.
Expected: PASS (4 tests).

- [ ] **Step 6: Implement the Windows pipe, with its tests**

Create `src-tauri/src/agent/pipe_windows.rs`:

```rust
//! Windows 的 agent 端點(金鑰保管庫 spec §4.4、§5.1):自己的 named pipe `\\.\pipe\sshelter-agent-<使用者 SID 字串的 SHA-256 前 16 個十六進位>`。
//! 每個 instance 的 DACL 只給目前使用者、拒絕遠端連線;第一個 instance 帶 `FILE_FLAG_FIRST_PIPE_INSTANCE`:名稱已經被別的程式佔用就開不起來
//! (spec §11;另一個 SSHelter 先被 `server::take_lock` 擋下)。只依賴 std 與 windows-sys。

use std::ffi::c_void;
use std::fs::File;
use std::io;
use std::os::windows::io::{AsRawHandle, FromRawHandle, OwnedHandle};
use std::path::Path;
use std::ptr;
use std::sync::atomic::AtomicUsize;
use std::sync::Arc;
use std::time::Duration;

use sha2::{Digest, Sha256};
use windows_sys::Win32::Foundation::{LocalFree, ERROR_PIPE_CONNECTED, ERROR_SUCCESS, HANDLE, INVALID_HANDLE_VALUE};
use windows_sys::Win32::Security::Authorization::{
    ConvertSidToStringSidW, SetEntriesInAclW, EXPLICIT_ACCESS_W, NO_MULTIPLE_TRUSTEE, SET_ACCESS, TRUSTEE_IS_SID, TRUSTEE_IS_USER, TRUSTEE_W,
};
use windows_sys::Win32::Security::{
    InitializeSecurityDescriptor, SetSecurityDescriptorDacl, ACL, NO_INHERITANCE, SECURITY_ATTRIBUTES, SECURITY_DESCRIPTOR, TOKEN_USER,
};
use windows_sys::Win32::Storage::FileSystem::{FILE_ALL_ACCESS, FILE_FLAG_FIRST_PIPE_INSTANCE, PIPE_ACCESS_DUPLEX};
use windows_sys::Win32::System::Pipes::{
    ConnectNamedPipe, CreateNamedPipeW, GetNamedPipeClientProcessId, PIPE_READMODE_BYTE, PIPE_REJECT_REMOTE_CLIENTS, PIPE_TYPE_BYTE,
    PIPE_UNLIMITED_INSTANCES, PIPE_WAIT,
};
use windows_sys::Win32::System::SystemServices::SECURITY_DESCRIPTOR_REVISION;

use crate::agent::server::{dispatch, take_lock, Handler, Started};
use crate::error::AppError;
use crate::sync::slot_files_windows::current_user_token;

/// 每個 pipe instance 共用的 SECURITY_ATTRIBUTES:DACL 只有一條「目前使用者:完全控制」。
struct OwnerOnly {
    /// `TOKEN_USER`;ACL 裡的 SID 指進這裡。
    _token: Vec<u64>,
    acl: *mut ACL,
    _descriptor: Box<SECURITY_DESCRIPTOR>,
    attributes: SECURITY_ATTRIBUTES,
}

// 指標只指向這個結構自己擁有的記憶體(token 緩衝區、LocalAlloc 的 ACL、Box 的描述元);整個結構一起搬到 listener 的執行緒,只在那裡使用。
unsafe impl Send for OwnerOnly {}

impl Drop for OwnerOnly {
    fn drop(&mut self) {
        // SAFETY: `acl` came from SetEntriesInAclW (LocalAlloc) and is freed once.
        unsafe { LocalFree(self.acl as *mut c_void) };
    }
}

impl OwnerOnly {
    fn new() -> io::Result<Self> {
        let token = current_user_token()?;
        // SAFETY: `token` holds an 8-byte aligned TOKEN_USER whose SID lives in `token`, which this struct keeps alive.
        unsafe {
            let user = &*(token.as_ptr() as *const TOKEN_USER);
            let access = EXPLICIT_ACCESS_W {
                grfAccessPermissions: FILE_ALL_ACCESS,
                grfAccessMode: SET_ACCESS,
                grfInheritance: NO_INHERITANCE,
                Trustee: TRUSTEE_W {
                    pMultipleTrustee: ptr::null_mut(),
                    MultipleTrusteeOperation: NO_MULTIPLE_TRUSTEE,
                    TrusteeForm: TRUSTEE_IS_SID,
                    TrusteeType: TRUSTEE_IS_USER,
                    ptstrName: user.User.Sid as *mut u16,
                },
            };
            let mut acl: *mut ACL = ptr::null_mut();
            let status = SetEntriesInAclW(1, &access, ptr::null(), &mut acl);
            if status != ERROR_SUCCESS {
                return Err(io::Error::from_raw_os_error(status as i32));
            }
            let mut descriptor: Box<SECURITY_DESCRIPTOR> = Box::new(std::mem::zeroed());
            let pointer = &mut *descriptor as *mut SECURITY_DESCRIPTOR as *mut c_void;
            if InitializeSecurityDescriptor(pointer, SECURITY_DESCRIPTOR_REVISION) == 0 || SetSecurityDescriptorDacl(pointer, 1, acl, 0) == 0 {
                let error = io::Error::last_os_error();
                LocalFree(acl as *mut c_void);
                return Err(error);
            }
            let attributes = SECURITY_ATTRIBUTES {
                nLength: std::mem::size_of::<SECURITY_ATTRIBUTES>() as u32,
                lpSecurityDescriptor: pointer,
                bInheritHandle: 0,
            };
            Ok(Self { _token: token, acl, _descriptor: descriptor, attributes })
        }
    }
}

/// 目前使用者 SID 的字串(`S-1-5-21-…`)。
fn user_sid() -> io::Result<String> {
    let token = current_user_token()?;
    // SAFETY: as in `OwnerOnly::new`; ConvertSidToStringSidW returns a LocalAlloc'ed NUL-terminated string, freed here.
    unsafe {
        let user = &*(token.as_ptr() as *const TOKEN_USER);
        let mut text: *mut u16 = ptr::null_mut();
        if ConvertSidToStringSidW(user.User.Sid, &mut text) == 0 {
            return Err(io::Error::last_os_error());
        }
        let length = (0..).take_while(|&i| *text.add(i) != 0).count();
        let sid = String::from_utf16_lossy(std::slice::from_raw_parts(text, length));
        LocalFree(text as *mut c_void);
        Ok(sid)
    }
}

/// `sshelter-agent-<SID 字串的 SHA-256 前 16 個十六進位>`(spec §4.4)。
pub fn pipe_name() -> io::Result<String> {
    let digest = Sha256::digest(user_sid()?.as_bytes());
    Ok(format!("sshelter-agent-{}", digest.iter().take(8).map(|b| format!("{b:02x}")).collect::<String>()))
}

fn wide_pipe_path(name: &str) -> Vec<u16> {
    format!(r"\\.\pipe\{name}").encode_utf16().chain(std::iter::once(0)).collect()
}

/// 建一個 pipe instance。`max_instances`:agent 是 `PIPE_UNLIMITED_INSTANCES`,Connect 的一次性 pipe 是 1。
fn create_instance(path: &[u16], security: &OwnerOnly, first: bool, max_instances: u32) -> io::Result<OwnedHandle> {
    let open_mode = PIPE_ACCESS_DUPLEX | if first { FILE_FLAG_FIRST_PIPE_INSTANCE } else { 0 };
    let pipe_mode = PIPE_TYPE_BYTE | PIPE_READMODE_BYTE | PIPE_WAIT | PIPE_REJECT_REMOTE_CLIENTS;
    // SAFETY: `path` is NUL-terminated; the security attributes outlive the call (the system copies them).
    let handle =
        unsafe { CreateNamedPipeW(path.as_ptr(), open_mode, pipe_mode, max_instances, 65536, 65536, 0, &security.attributes) };
    if handle == INVALID_HANDLE_VALUE {
        return Err(io::Error::last_os_error());
    }
    // SAFETY: a fresh handle that nothing else owns.
    Ok(unsafe { OwnedHandle::from_raw_handle(handle as _) })
}

/// 等一個連線;連上了回傳對方的 PID(拿不到 → None)。
pub(crate) fn accept(pipe: &OwnedHandle) -> io::Result<Option<u32>> {
    let handle = pipe.as_raw_handle() as HANDLE;
    // SAFETY: a valid pipe handle in synchronous mode (no OVERLAPPED).
    if unsafe { ConnectNamedPipe(handle, ptr::null_mut()) } == 0 {
        let error = io::Error::last_os_error();
        // 對方在 CreateNamedPipeW 與 ConnectNamedPipe 之間就連上了:也算連上。
        if error.raw_os_error() != Some(ERROR_PIPE_CONNECTED as i32) {
            return Err(error);
        }
    }
    let mut pid = 0u32;
    // SAFETY: writes one u32.
    let known = unsafe { GetNamedPipeClientProcessId(handle, &mut pid) } != 0;
    Ok(known.then_some(pid))
}

/// 在 `dir` 拿鎖,開 `\\.\pipe\<name>`,每條連線交給 `handle`。另一個 SSHelter 拿著鎖 → `OtherInstance`;名稱被別的程式佔用 → 錯誤。
pub fn listen(dir: &Path, name: &str, handle: Handler) -> Result<Started, AppError> {
    let Some(lock) = take_lock(dir)? else { return Ok(Started::OtherInstance) };
    let security = OwnerOnly::new()?;
    let path = wide_pipe_path(name);
    let first = create_instance(&path, &security, true, PIPE_UNLIMITED_INSTANCES)
        .map_err(|e| AppError::Other(format!("Another program is using SSHelter's agent pipe ({e})")))?;
    std::thread::Builder::new().name("sshelter-agent".to_string()).spawn(move || {
        let _lock = lock;
        let active = Arc::new(AtomicUsize::new(0));
        let mut current = first;
        loop {
            let connected = accept(&current);
            // 先建好下一個 instance:這條連線處理的時候,別的程式才連得上。
            let next = loop {
                match create_instance(&path, &security, false, PIPE_UNLIMITED_INSTANCES) {
                    Ok(next) => break next,
                    Err(e) => {
                        eprintln!("[agent] cannot create a pipe instance: {e}");
                        std::thread::sleep(Duration::from_secs(1));
                    }
                }
            };
            let served = std::mem::replace(&mut current, next);
            match connected {
                Ok(pid) => dispatch(File::from(served), pid, &active, &handle),
                Err(e) => eprintln!("[agent] a pipe connection failed: {e}"),
            }
        }
    })?;
    Ok(Started::Running)
}

#[cfg(test)]
mod tests {
    use super::*;
    use crate::agent::protocol::{read_frame, write_frame, SSH_AGENTC_REQUEST_IDENTITIES, SSH_AGENT_IDENTITIES_ANSWER};
    use crate::agent::server::testing::serving;
    use std::sync::mpsc;

    #[test]
    fn the_pipe_name_is_stable_hex_from_the_user() {
        let name = pipe_name().unwrap();
        assert_eq!(name, pipe_name().unwrap());
        let hex = name.strip_prefix("sshelter-agent-").unwrap();
        assert_eq!(hex.len(), 16);
        assert!(hex.chars().all(|c| c.is_ascii_hexdigit()));
    }

    #[test]
    fn serves_a_client_and_a_second_agent_does_not_start() {
        let dir = tempfile::tempdir().unwrap();
        let agent = dir.path().join("agent");
        let name = format!("sshelter-test-{}", std::process::id());
        let (tx, rx) = mpsc::channel();
        assert_eq!(listen(&agent, &name, serving(tx.clone())).unwrap(), Started::Running);
        let mut client = std::fs::OpenOptions::new().read(true).write(true).open(format!(r"\\.\pipe\{name}")).unwrap();
        write_frame(&mut client, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();
        assert_eq!(read_frame(&mut client).unwrap().unwrap()[0], SSH_AGENT_IDENTITIES_ANSWER);
        assert_eq!(rx.recv_timeout(Duration::from_secs(5)).unwrap(), Some(std::process::id()));
        assert_eq!(listen(&agent, &name, serving(tx)).unwrap(), Started::OtherInstance);
    }

    #[test]
    fn a_pipe_name_someone_else_holds_is_an_error() {
        let dir = tempfile::tempdir().unwrap();
        let name = format!("sshelter-squat-{}", std::process::id());
        let security = OwnerOnly::new().unwrap();
        let _squatter = create_instance(&wide_pipe_path(&name), &security, true, PIPE_UNLIMITED_INSTANCES).unwrap();
        let (tx, _rx) = mpsc::channel();
        let err = listen(&dir.path().join("agent"), &name, serving(tx)).unwrap_err().to_string();
        assert!(err.contains("Another program"), "{err}");
    }
}
```

Type-check it from macOS in a scratch crate in the session scratchpad (the whole crate is not checked for Windows from macOS): include the real `pipe_windows.rs`, `server.rs` and `slot_files_windows.rs` with `#[path]`, stub the crate-internal items they use (`AppError` with `From<io::Error>`, `slot_files::ensure_keys_dir`, `session::serve`, the protocol frame helpers used by the tests), depend on `windows-sys` 0.61 with the same features, `sha2`, `ssh-key` 0.6.7 and `tempfile`, and run `cargo check --offline --tests --target x86_64-pc-windows-msvc`. Prove the check is real once (a deliberate type error under `cfg(windows)` is reported). windows-sys 0.61 is the authority on exact names and modules: if a name above lives elsewhere, import it from where the compiler points and say so in the report. The Windows CI job (Task 13) runs these tests for real.

- [ ] **Step 7: Wire the agent into the app**

In `src-tauri/src/agent/mod.rs`, add:

```rust
use std::path::{Path, PathBuf};
use std::sync::Mutex;

use ssh_key::public::KeyData;
use tauri::Manager;
use zeroize::Zeroizing;

use crate::error::AppError;
use crate::state::AppState;
use crate::sync::env::{Clock, Keychain, OsKeychain, SystemClock};
use crate::vault::store::{vault_path, with_vault, AgentSettings};

/// `IdentityAgent` 在 `agent/config` 裡的值(spec §4.4、§6;ssh 自己展開 `~`)。
#[cfg(unix)]
pub const SOCKET_VALUE: &str = "~/.ssh/sshelter/agent/sock";

/// agent 的目錄 `<home>/.ssh/sshelter/agent`(0700):`config`、`sock`、`lock`、Connect 的 `run/`。
pub fn agent_dir(home: &Path) -> PathBuf {
    home.join(".ssh").join("sshelter").join("agent")
}

/// 寫進 `agent/config` 的 `IdentityAgent` 值。Windows 的 pipe 寫成正斜線(Win32-OpenSSH 8.9 起反斜線的寫法會失敗,spec §4.4)。
pub fn identity_agent_value() -> Result<String, AppError> {
    #[cfg(windows)]
    {
        Ok(format!("//./pipe/{}", pipe_windows::pipe_name()?))
    }
    #[cfg(not(windows))]
    {
        Ok(SOCKET_VALUE.to_string())
    }
}

/// agent 有沒有在提供(spec §11)。
#[derive(Clone, Debug, Default, PartialEq, Eq)]
pub enum AgentStatus {
    #[default]
    NotStarted,
    Running,
    /// 另一個 SSHelter 在提供。
    OtherInstance,
    /// 開不起來(路徑太長、權限、pipe 名稱被佔)。
    Failed(String),
}
```

and extend `AgentRuntime`:

```rust
/// agent 的執行期狀態(`AppState::agent`)。
#[derive(Default)]
pub struct AgentRuntime {
    pub prompts: prompt::PromptHub,
    pub broker: broker::Broker,
    pub status: Mutex<AgentStatus>,
}
```

Then the production host and the start function:

```rust
/// production 的 `AgentHost`:同步狀態的金鑰、保管庫、OS keychain、系統時鐘、known_hosts 與核准視窗。
pub struct AppAgentHost {
    pub app: tauri::AppHandle,
}

impl broker::AgentHost for AppAgentHost {
    fn keys(&self) -> Vec<broker::VaultKey> {
        let state = self.app.state::<AppState>();
        let core = state.sync.core.lock().unwrap();
        core.state.as_ref().map(broker::vault_keys).unwrap_or_default()
    }

    fn private_key(&self, slot_id: &str) -> Result<Option<Zeroizing<String>>, AppError> {
        let result = crate::sync::engine::with_env(&self.app, |env| {
            with_vault(env.runtime, &vault_path(&env.state_path), env.keychain, env.now(), |vault| vault.get(slot_id))
                .map(|entry| entry.map(|entry| Zeroizing::new(entry.private_key.clone())))
                .map_err(AppError::from)
        })
        .and_then(|inner| inner);
        // 使用者按了允許,金鑰卻拿不出來(keychain 鎖著、保管庫讀不懂):broker 只會拒絕,原因記在這裡。
        if let Err(e) = &result {
            eprintln!("[agent] cannot read the key from SSHelter's vault: {e}");
        }
        result
    }

    fn settings(&self) -> AgentSettings {
        crate::sync::engine::with_env(&self.app, |env| {
            with_vault(env.runtime, &vault_path(&env.state_path), env.keychain, env.now(), |vault| Ok(vault.settings().clone()))
        })
        .ok()
        .and_then(Result::ok)
        .unwrap_or_default()
    }

    fn keychain(&self) -> &dyn Keychain {
        &OsKeychain
    }

    fn now_ms(&self) -> u64 {
        SystemClock.now_ms()
    }

    fn host_name(&self, host_key: &KeyData) -> Option<String> {
        let mut files = Vec::new();
        if let Ok(ssh_dir) = crate::keys::ssh_dir() {
            files.push(ssh_dir.join("known_hosts"));
        }
        #[cfg(unix)]
        files.push(PathBuf::from("/etc/ssh/ssh_known_hosts"));
        files.iter().filter_map(|path| std::fs::read_to_string(path).ok()).find_map(|text| broker::host_name_in(&text, host_key))
    }

    fn ask(&self, request: prompt::AgentApprovalRequest) -> Option<prompt::AgentApprovalAnswer> {
        let surface = prompt::TauriPromptSurface { app: self.app.clone() };
        self.app.state::<AppState>().agent.prompts.ask(&surface, request, prompt::APPROVAL_TIMEOUT)
    }
}

/// 開 agent(SSHelter 的視窗程式啟動時,含 `--mcp-host`;`--mcp` 的 stdio 轉接不建 Tauri,不會到這裡)。開不起來只記下原因(spec §11),
/// SSHelter 其他功能照常。
pub fn start(app: &tauri::AppHandle) {
    // 記住的核准與解開的私鑰到期就丟掉(spec §5.5),不等下一個請求;agent 開不起來也照做(Connect 的一次性通道仍會用到 broker)。
    let janitor = app.clone();
    let _ = std::thread::Builder::new().name("sshelter-agent-expire".to_string()).spawn(move || loop {
        std::thread::sleep(std::time::Duration::from_secs(60));
        janitor.state::<AppState>().agent.broker.expire(SystemClock.now_ms());
    });
    let status = match listen(app) {
        Ok(server::Started::Running) => AgentStatus::Running,
        Ok(server::Started::OtherInstance) => AgentStatus::OtherInstance,
        Err(e) => {
            eprintln!("[agent] not started: {e}");
            AgentStatus::Failed(e.to_string())
        }
    };
    *app.state::<AppState>().agent.status.lock().unwrap() = status;
}

fn listen(app: &tauri::AppHandle) -> Result<server::Started, AppError> {
    let ssh_dir = crate::keys::ssh_dir()?;
    let home = ssh_dir.parent().ok_or_else(|| AppError::Other("cannot determine home directory".to_string()))?;
    let dir = agent_dir(home);
    let app = app.clone();
    let handle: server::Handler = std::sync::Arc::new(move |mut stream, pid| {
        let program = pid.and_then(|pid| peer::identify(&peer::process_chain(pid)));
        let host = AppAgentHost { app: app.clone() };
        let state = app.state::<AppState>();
        let connection = broker::Connection { broker: &state.agent.broker, host: &host, program, grant: None };
        if let Err(e) = session::serve(&mut stream, &connection) {
            eprintln!("[agent] connection closed: {e}");
        }
    });
    #[cfg(unix)]
    {
        server::listen_unix(&dir, handle)
    }
    #[cfg(windows)]
    {
        pipe_windows::listen(&dir, &pipe_windows::pipe_name()?, handle)
    }
}
```

In `src-tauri/src/lib.rs` `setup`, after `sync::engine::initialize(app.handle())?;`, add:

```rust
            // SSHelter 的 SSH agent(金鑰保管庫 spec §5.1):開不起來只記在 `AgentRuntime::status`,不擋啟動。
            agent::start(app.handle());
```

- [ ] **Step 8: Run the tests, type-check Windows again, run the full suite, commit**

Run: `cargo test --offline --lib agent:: -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`, then the Windows check from Step 6 again, then the full Rust suite. (`agent/mod.rs` depends on Tauri and cannot go into the scratch crate: keep its Windows arms exactly the two calls shown — `pipe_windows::pipe_name()` and `pipe_windows::listen(...)` — and the Windows CI job compiles them.)

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/agent src-tauri/src/sync/slot_files_windows.rs src-tauri/src/lib.rs
git commit -m "feat(agent): serve the agent on a socket or named pipe owned by one SSHelter"
```
(`Cargo.lock` changes only if the build resolves something new; add it only when `git status` shows it.)

---

### Task 10: `agent/config`, the Include line, and keeping them out of sync and the host list

**Files:**
- Create: `src-tauri/src/agent/wiring.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`pub mod wiring;`)
- Modify: `src-tauri/src/sync/hosts_file.rs` (`sync_include_index`)
- Modify: `src-tauri/src/config/include.rs` (`load_recursive` skips the agent Include)
- Modify: `src-tauri/src/config/intel.rs` (lint rule 3 and `key_hygiene`)
- Modify: `src-tauri/src/sync/round.rs` (refresh after step 6b), `src-tauri/src/sync/slots.rs` (refresh after `set_delivery`)

**Interfaces:**
- Consumes: `crate::agent::{agent_dir, identity_agent_value}` (Task 9), `crate::sync::slots::vault_slot_files` (Task 7), `crate::config::commands::persist_file(doc, idx, backed_up, retention)`, `crate::config::parser::parse_file`, `crate::config::serialize::serialize_items`, `crate::sync::slot_rules::{resolve_identity_value, IdentityTarget, slot_file_of_value}`, `crate::sync::slot_files::ensure_keys_dir`, `crate::fsutil::atomic_write`.
- Produces:
  - `crate::agent::wiring::{INCLUDE_TOKEN = "~/.ssh/sshelter/agent/config", HEADER = "# Managed by SSHelter. Changes here are overwritten."}`
  - `agent_config_path(home: &Path) -> PathBuf` (`<agent_dir>/config`)
  - `is_agent_include_token(token: &str) -> bool`, `is_agent_include(item: &Item) -> bool`
  - `vault_host_patterns(doc: &SshConfigDoc, vault_files: &BTreeSet<String>, home: &Path) -> Vec<Vec<String>>`
  - `render(patterns: &[Vec<String>], endpoint: &str) -> String`
  - `ensure_include_first(items: &mut Vec<Item>, trailing_newline: &mut bool) -> bool`
  - `#[derive(Clone, Copy, Debug, PartialEq, Eq)] pub enum WiringStatus { NotNeeded, Ready, IncludeMissing }`
  - `refresh(doc: &mut SshConfigDoc, backed_up: &mut HashSet<PathBuf>, retention: Option<usize>, home: &Path, vault_files: &BTreeSet<String>, endpoint: &str) -> Result<WiringStatus, AppError>`
  - `refresh_env(env: &SyncEnv) -> Result<WiringStatus, AppError>`

Rules (spec §6):
- `agent/config` lists only Host blocks whose `IdentityFile` resolves to a slot whose file is in `vault_files`; each block's pattern list is copied as-is (negations included), once.
- The Include is item 0 of the main config. It is added the first time `agent/config` is created. If `agent/config` already exists and the Include is gone, the user removed it: report `IncludeMissing` and do not add it back.
- A CRLF file gets a CRLF Include line; an empty file gets a trailing newline; nothing else in the file changes.
- Only the default root `<home>/.ssh/config` gets the Include; any other loaded root reports `IncludeMissing`.

- [ ] **Step 1: Write the failing tests**

Add `pub mod wiring;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/wiring.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::parser::parse_file;
    use crate::config::serialize::serialize_items;

    const SOCK: &str = "~/.ssh/sshelter/agent/sock";

    fn with_include(text: &str) -> String {
        let (mut items, mut trailing) = parse_file(text);
        assert!(ensure_include_first(&mut items, &mut trailing));
        serialize_items(&items, trailing)
    }

    #[test]
    fn the_include_goes_first_and_nothing_else_changes() {
        assert_eq!(with_include(""), "Include ~/.ssh/sshelter/agent/config\n");
        assert_eq!(with_include("# mine\nHost a\n  HostName x\n"), "Include ~/.ssh/sshelter/agent/config\n# mine\nHost a\n  HostName x\n");
        assert_eq!(with_include("Host a\n  HostName x"), "Include ~/.ssh/sshelter/agent/config\nHost a\n  HostName x");
        assert_eq!(with_include("# c\r\nHost a\r\n"), "Include ~/.ssh/sshelter/agent/config\r\n# c\r\nHost a\r\n");
    }

    #[test]
    fn an_include_already_first_is_left_alone_and_others_move_to_the_top() {
        let (mut items, mut trailing) = parse_file("Include ~/.ssh/sshelter/agent/config\nHost a\n");
        assert!(!ensure_include_first(&mut items, &mut trailing));
        let (mut items, mut trailing) = parse_file("Host a\n  HostName x\nInclude ~/.ssh/sshelter/agent/config\n");
        assert!(ensure_include_first(&mut items, &mut trailing));
        assert_eq!(serialize_items(&items, trailing), "Include ~/.ssh/sshelter/agent/config\nHost a\n  HostName x\n");
    }

    #[test]
    fn agent_config_lists_only_hosts_on_vault_slots_with_their_patterns() {
        let home = Path::new("/h");
        let (items, trailing) = parse_file(
            "Host web\n  IdentityFile ~/.ssh/sshelter/keys/id_mac-11111111\n\
             Host *.lab !bastion.lab\n  IdentityFile ~/.ssh/sshelter/keys/id_mac-11111111\n\
             Host plain\n  IdentityFile ~/.ssh/id_rsa\n\
             Host other\n  IdentityFile ~/.ssh/sshelter/keys/other-22222222\n\
             Host off\n  #IdentityFile ~/.ssh/sshelter/keys/id_mac-11111111\n\
             Host web\n  IdentityFile ~/.ssh/sshelter/keys/id_mac-11111111\n",
        );
        let doc = SshConfigDoc {
            files: vec![crate::config::model::ConfigFile {
                path: home.join(".ssh/config"),
                items,
                trailing_newline: trailing,
                fingerprint: Default::default(),
            }],
        };
        let vault = BTreeSet::from(["id_mac-11111111".to_string()]);
        let patterns = vault_host_patterns(&doc, &vault, home);
        assert_eq!(patterns, vec![vec!["web".to_string()], vec!["*.lab".to_string(), "!bastion.lab".to_string()]]);
        assert_eq!(
            render(&patterns, SOCK),
            "# Managed by SSHelter. Changes here are overwritten.\n\
             Host web\n  IdentityAgent ~/.ssh/sshelter/agent/sock\n  IdentitiesOnly yes\n\
             Host *.lab !bastion.lab\n  IdentityAgent ~/.ssh/sshelter/agent/sock\n  IdentitiesOnly yes\n"
        );
    }

    #[test]
    fn the_agent_include_token_is_recognised_exactly() {
        assert!(is_agent_include_token("~/.ssh/sshelter/agent/config"));
        assert!(!is_agent_include_token("~/.ssh/sshelter/agent/config.bak"));
        assert!(!is_agent_include_token("~/.ssh/sshelter/work-11111111.config"));
    }
}
```

Check the field names of `ConfigFile` and whether its fingerprint type implements `Default`; if it does not, build the doc by writing the text to a temp `config` and loading it with `crate::config::include::load_doc`.

Add these integration tests to the `tests` module of `src-tauri/src/sync/slots.rs` (they use `TestDevice`):

```rust
    #[test]
    fn moving_a_key_into_the_vault_wires_its_hosts_to_the_agent() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let home = home(&b);
        let config = std::fs::read_to_string(crate::agent::wiring::agent_config_path(&home)).unwrap();
        assert!(config.starts_with(crate::agent::wiring::HEADER));
        assert!(config.contains("Host web\n  IdentityAgent "));
        let main = std::fs::read_to_string(b.main_path()).unwrap();
        assert!(main.starts_with("Include ~/.ssh/sshelter/agent/config\n"), "{main}");

        // A sync round keeps both Includes, ours first, and the loader never sees agent/config.
        settle(&b);
        settle(&b);
        let main = std::fs::read_to_string(b.main_path()).unwrap();
        let lines: Vec<&str> = main.lines().collect();
        assert_eq!(lines[0], "Include ~/.ssh/sshelter/agent/config");
        assert!(lines[1].starts_with("Include ~/.ssh/sshelter/"), "the sync Include comes right after ours: {main}");
        b.reload();
        let doc = b.doc.lock().unwrap();
        assert!(doc.as_ref().unwrap().files.iter().all(|f| f.path != crate::agent::wiring::agent_config_path(&home)));
    }

    #[test]
    fn a_removed_include_is_reported_and_not_added_back() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        set_delivery(&b.env(), &id, true).unwrap();
        let main = std::fs::read_to_string(b.main_path()).unwrap();
        std::fs::write(b.main_path(), main.replacen("Include ~/.ssh/sshelter/agent/config\n", "", 1)).unwrap();
        b.reload();
        assert_eq!(crate::agent::wiring::refresh_env(&b.env()).unwrap(), crate::agent::wiring::WiringStatus::IncludeMissing);
        assert!(!std::fs::read_to_string(b.main_path()).unwrap().contains("sshelter/agent/config"));
    }
```

Add to the lint tests in `src-tauri/src/config/intel.rs` (follow the existing temp-home pattern of the SP3 lint tests there):

```rust
    #[test]
    fn a_vault_slot_with_only_its_pub_is_not_a_missing_identity_file() {
        let dir = tempfile::tempdir().unwrap();
        let keys = dir.path().join(".ssh/sshelter/keys");
        std::fs::create_dir_all(&keys).unwrap();
        std::fs::write(keys.join("vaulted-11111111.pub"), "ssh-ed25519 AAAA test\n").unwrap();
        let text = format!(
            "Host v\n  IdentityFile {0}/vaulted-11111111\nHost m\n  IdentityFile {0}/missing-22222222\n",
            keys.display()
        );
        let (items, trailing) = crate::config::parser::parse_file(&text);
        let doc = SshConfigDoc {
            files: vec![crate::config::model::ConfigFile { path: dir.path().join("config"), items, trailing_newline: trailing, fingerprint: Default::default() }],
        };
        let issues = lint(&doc, &BTreeSet::new());
        assert!(issues.iter().all(|i| !(i.rule == "missing-identity-file" && i.alias == "v")), "{issues:?}");
        assert!(issues.iter().any(|i| i.rule == "missing-identity-file" && i.alias == "m"));
    }
```

(These absolute paths are not slot values, so for this test also make `slot_file_of_value` irrelevant: the rule below checks the sibling `.pub` of any path whose own file is missing **and** whose value is a slot path **or** lives under `<home>/.ssh/sshelter/keys/`. If the existing lint tests build values as `~/.ssh/sshelter/keys/...`, use that form instead and drop the absolute-path variant.)

- [ ] **Step 2: Run them to verify they fail**

Run: `cargo test --offline --lib -- agent::wiring sync::slots config::intel --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors in `agent::wiring`; the new slots and intel tests fail.

- [ ] **Step 3: Implement `wiring.rs`**

Put above the tests:

```rust
//! `ssh` 怎麼找到 SSHelter 的 agent(金鑰保管庫 spec §6):`~/.ssh/sshelter/agent/config` 只列用到「只在 SSHelter」插槽的主機,
//! 每個 Host 區塊設 `IdentityAgent` 與 `IdentitiesOnly yes`;`~/.ssh/config` 的第一行 Include 它(ssh_config 取第一個符合的值)。
//! 這個檔案在子目錄裡:同步把 `~/.ssh/sshelter/<名稱>.config` 當成 space 檔,子目錄裡的不碰(`hosts_file::is_our_include_token`)。
//! SSHelter 讀取 config 時略過它(`config::include::load_recursive`),主機清單才不會重複。

use std::collections::{BTreeSet, HashSet};
use std::path::{Path, PathBuf};

use crate::agent::agent_dir;
use crate::config::commands::persist_file;
use crate::config::model::{Item, SshConfigDoc};
use crate::config::parser::parse_file;
use crate::error::AppError;
use crate::sync::env::SyncEnv;
use crate::sync::slot_files;
use crate::sync::slot_rules::{resolve_identity_value, IdentityTarget};

pub const INCLUDE_TOKEN: &str = "~/.ssh/sshelter/agent/config";
pub const HEADER: &str = "# Managed by SSHelter. Changes here are overwritten.";

pub fn agent_config_path(home: &Path) -> PathBuf {
    agent_dir(home).join("config")
}

pub fn is_agent_include_token(token: &str) -> bool {
    token == INCLUDE_TOKEN
}

/// 生效中的 top-level `Include`,而且含有 agent 的 token。
pub fn is_agent_include(item: &Item) -> bool {
    matches!(item, Item::Directive(d) if d.key == "include" && !d.serializes_as_comment() && d.value.split_whitespace().any(is_agent_include_token))
}

/// 用到「只在 SSHelter」插槽(`vault_files` 是插槽檔名)的 Host 區塊的 pattern,依出現順序,重複的只留一次。註解掉的 Host 與 `IdentityFile`
/// 不算;`Match` 區塊不支援(spec §6)。
pub fn vault_host_patterns(doc: &SshConfigDoc, vault_files: &BTreeSet<String>, home: &Path) -> Vec<Vec<String>> {
    let mut out: Vec<Vec<String>> = Vec::new();
    for file in &doc.files {
        for item in &file.items {
            let Item::Host(host) = item else { continue };
            if host.header.serializes_as_comment() {
                continue;
            }
            let uses_vault = host.body.iter().any(|line| {
                matches!(line, Item::Directive(d) if d.key == "identityfile"
                    && !d.serializes_as_comment()
                    && matches!(resolve_identity_value(&d.value, home), IdentityTarget::Slot(f) if vault_files.contains(&f)))
            });
            if uses_vault && !out.contains(&host.patterns) {
                out.push(host.patterns.clone());
            }
        }
    }
    out
}

/// `agent/config` 的內容。`endpoint` = `IdentityAgent` 的值(`crate::agent::identity_agent_value`)。
pub fn render(patterns: &[Vec<String>], endpoint: &str) -> String {
    let mut out = format!("{HEADER}\n");
    for host in patterns {
        out.push_str(&format!("Host {}\n  IdentityAgent {endpoint}\n  IdentitiesOnly yes\n", host.join(" ")));
    }
    out
}

/// 把 agent 的 Include 放在 `items` 的第 0 項:已經在第 0 項就不動(回傳 false);其他位置的一併移除再插到最前面。CRLF 的檔案插入 CRLF 的一行;
/// 空檔案加上結尾換行。
pub fn ensure_include_first(items: &mut Vec<Item>, trailing_newline: &mut bool) -> bool {
    let first = items.first().is_some_and(is_agent_include);
    let elsewhere = items.iter().skip(1).any(is_agent_include);
    if first && !elsewhere {
        return false;
    }
    let crlf = items.iter().any(|item| match item {
        Item::Blank(s) | Item::Comment(s) => s.ends_with('\r'),
        Item::Directive(d) => d.raw.ends_with('\r'),
        Item::Host(h) => h.header.raw.ends_with('\r'),
        Item::Match(m) => m.header.raw.ends_with('\r'),
    });
    let was_empty = items.is_empty();
    items.retain(|item| !is_agent_include(item));
    let (mut line, _) = parse_file(&format!("Include {INCLUDE_TOKEN}{}\n", if crlf { "\r" } else { "" }));
    items.insert(0, line.remove(0));
    if was_empty {
        *trailing_newline = true;
    }
    true
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum WiringStatus {
    /// 這台沒有任何主機用到只在 SSHelter 的金鑰,也從沒寫過 `agent/config`。
    NotNeeded,
    Ready,
    /// `agent/config` 在,`~/.ssh/config` 卻沒有它的 Include(使用者拿掉了,或載入的不是預設的 config):不自動加回(spec §6、§11)。
    IncludeMissing,
}

/// 重寫 `agent/config`(內容有變才寫),第一次寫時在主 config 的第一行加上 Include。呼叫端持有 doc 與 backed_up 的鎖。
pub fn refresh(
    doc: &mut SshConfigDoc,
    backed_up: &mut HashSet<PathBuf>,
    retention: Option<usize>,
    home: &Path,
    vault_files: &BTreeSet<String>,
    endpoint: &str,
) -> Result<WiringStatus, AppError> {
    let patterns = vault_host_patterns(doc, vault_files, home);
    let config_path = agent_config_path(home);
    let first_time = !config_path.exists();
    if patterns.is_empty() && first_time {
        return Ok(WiringStatus::NotNeeded);
    }
    let content = render(&patterns, endpoint);
    if std::fs::read_to_string(&config_path).ok().as_deref() != Some(content.as_str()) {
        slot_files::ensure_keys_dir(&agent_dir(home))?;
        crate::fsutil::atomic_write(&config_path, content.as_bytes(), 0o600)?;
    }
    let default_root = home.join(".ssh").join("config");
    let Some(main) = doc.files.first() else { return Ok(WiringStatus::IncludeMissing) };
    if main.path != default_root {
        return Ok(WiringStatus::IncludeMissing);
    }
    let present = main.items.iter().any(is_agent_include);
    if !present && !first_time {
        return Ok(WiringStatus::IncludeMissing);
    }
    put_include_first(doc, backed_up, retention)?;
    Ok(WiringStatus::Ready)
}

/// 把 agent 的 Include 放在主 config(`doc.files[0]`)的第 0 項並存檔;已經在那裡就什麼都不做。存檔失敗時記憶體裡的 doc 復原。
fn put_include_first(doc: &mut SshConfigDoc, backed_up: &mut HashSet<PathBuf>, retention: Option<usize>) -> Result<(), AppError> {
    let mut items = doc.files[0].items.clone();
    let mut trailing = doc.files[0].trailing_newline;
    if !ensure_include_first(&mut items, &mut trailing) {
        return Ok(());
    }
    let saved = (std::mem::replace(&mut doc.files[0].items, items), doc.files[0].trailing_newline);
    doc.files[0].trailing_newline = trailing;
    if let Err(e) = persist_file(doc, 0, backed_up, retention) {
        doc.files[0].items = saved.0;
        doc.files[0].trailing_newline = saved.1;
        return Err(e);
    }
    Ok(())
}

/// 同步執行緒與命令用:取保管庫的插槽檔名(只短暫拿 core 鎖),再拿 doc 與 backed_up 的鎖呼叫 `refresh`。config 還沒載入就什麼都不做。
pub fn refresh_env(env: &SyncEnv) -> Result<WiringStatus, AppError> {
    let Some(home) = env.ssh_dir.parent().map(Path::to_path_buf) else { return Ok(WiringStatus::NotNeeded) };
    let vault_files = env
        .runtime
        .core
        .lock()
        .unwrap()
        .state
        .as_ref()
        .map(crate::sync::slots::vault_slot_files)
        .unwrap_or_default();
    let endpoint = crate::agent::identity_agent_value()?;
    let mut doc_lock = env.doc.lock().unwrap();
    let Some(doc) = doc_lock.as_mut() else { return Ok(WiringStatus::NotNeeded) };
    let mut backed_up = env.backed_up.lock().unwrap();
    refresh(doc, &mut backed_up, env.retention(), &home, &vault_files, &endpoint)
}
```

In tests, `TestDevice::main_path()` is `<home>/.ssh/config`, so the default-root check holds there.

- [ ] **Step 4: Keep sync's Include after ours and keep `agent/config` out of the loaded doc**

In `src-tauri/src/sync/hosts_file.rs`, change `sync_include_index`:

```rust
/// 同步 Include 的位置:前導註解/空行與 SSHelter agent 的 Include(`agent::wiring`,必須是第一行)之後、其他任何項目之前。
fn sync_include_index(items: &[Item]) -> usize {
    items
        .iter()
        .position(|i| !matches!(i, Item::Blank(_) | Item::Comment(_)) && !crate::agent::wiring::is_agent_include(i))
        .unwrap_or(items.len())
}
```

(Keep the existing doc comment's explanation of first-obtained-wins; just add the agent Include to it.)

In `src-tauri/src/config/include.rs`, inside `for token in pattern_str.split_whitespace() {`, add as the first statement:

```rust
            // SSHelter 自己產生的 agent 設定(金鑰保管庫 spec §6):不是使用者的 config,讀進來會讓主機重複出現。
            if crate::agent::wiring::is_agent_include_token(token) {
                continue;
            }
```

- [ ] **Step 5: Lint and key hygiene accept a vault slot**

In `src-tauri/src/config/intel.rs` rule 3, push the `missing-identity-file` issue only when the key is really missing:

```rust
                            let path = Path::new(expanded.as_ref());
                            // 只在 SSHelter 的插槽(金鑰保管庫 spec §6):插槽路徑上沒有私鑰,只有 `.pub`,金鑰由 SSHelter 的 agent 提供。
                            let vault_slot = crate::sync::slot_rules::slot_file_of_value(&d.value).is_some()
                                && crate::sync::slot_rules::public_path(path).is_file();
                            if !path.exists() && !vault_slot {
                                // ... the existing issues.push(...) unchanged ...
                            }
```

In `key_hygiene`, compute `exists` the same way: `Path::new(&expanded).exists() || (slot_file_of_value(&d.value).is_some() && public_path(Path::new(&expanded)).is_file())`.

If Step 1's intel test used absolute paths, rewrite its values as `~/.ssh/sshelter/keys/<file>` and point `HOME` at the temp dir the way the existing SP3 lint tests do, so `slot_file_of_value` recognises them.

- [ ] **Step 6: Refresh after a delivery change and after each round**

In `src-tauri/src/sync/slots.rs` `set_delivery`, right before `env.events.wake();`:

```rust
    if let Err(e) = crate::agent::wiring::refresh_env(env) {
        eprintln!("[agent] could not update the agent config: {e}");
    }
```

In `src-tauri/src/sync/round.rs`, after the whole step 6b block (after its `commit` and notices), add:

```rust
    // 6c. agent 的設定(金鑰保管庫 spec §6):主機或插槽變了就重寫 `agent/config`;第一次需要時把 Include 放在主 config 的第一行。
    if let Err(e) = crate::agent::wiring::refresh_env(env) {
        eprintln!("[agent] could not update the agent config: {e}");
    }
```

Make sure no lock is held at that point (step 6b's `commit` has returned).

- [ ] **Step 7: Run the tests to verify they pass**

Run: `cargo test --offline --lib -- agent::wiring sync:: config:: --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS, including all existing sync and config tests (in particular the `hosts_file` Include tests).

- [ ] **Step 8: Run the full Rust suite and commit**

```bash
git add src-tauri/src/agent src-tauri/src/sync/hosts_file.rs src-tauri/src/config/include.rs src-tauri/src/config/intel.rs src-tauri/src/sync/round.rs src-tauri/src/sync/slots.rs
git commit -m "feat(agent): point hosts on vault keys at SSHelter's agent through a managed Include"
```

---

### Task 11: "Only in SSHelter" in the Keys dialog

**Files:**
- Modify: `src-tauri/src/agent/wiring.rs` (`status`, `restore_include`), `src-tauri/src/agent/mod.rs` (`AgentProblem`, `problem`, commands), `src-tauri/src/lib.rs` (register commands)
- Generated: `src/bindings/AgentProblem.ts` (written by `cargo test`; commit it)
- Modify: `src/lib/agent.ts`, `src/lib/agent.test.ts`, `src/lib/key-slots.ts`, `src/lib/key-slots.test.ts`, `src/lib/sync.ts`, `src/components/KeySlotsSection.tsx`, `src/components/KeySlotsSection.test.tsx`

**Interfaces:**
- Consumes: Task 9 `AgentStatus`, `AgentRuntime.status`; Task 7 `sync_key_set_delivery(slot_id, vault)`, `SyncKeySlotView.in_vault`; Task 10 `wiring::{WiringStatus, agent_config_path, is_agent_include, put_include_first}`.
- Produces:
  - `crate::agent::wiring::status(doc: &SshConfigDoc, home: &Path, vault_files: &BTreeSet<String>) -> WiringStatus`
  - `crate::agent::wiring::restore_include(doc: &mut SshConfigDoc, backed_up: &mut HashSet<PathBuf>, retention: Option<usize>, home: &Path) -> Result<(), AppError>`
  - `#[serde(tag = "kind", rename_all = "snake_case")] pub enum AgentProblem { NotRunning { reason: String }, IncludeMissing }` (ts-rs), `pub fn problem(status: &AgentStatus, wiring: WiringStatus) -> Option<AgentProblem>`
  - Commands `agent_problem() -> Result<Option<AgentProblem>, AppError>`, `agent_fix_include() -> Result<Option<AgentProblem>, AppError>`
  - TS: `agentProblemKey = ["config", "agentProblem"]`, `useAgentProblem(enabled)`, `useFixAgentInclude()`, `agentProblemText(problem)` in `src/lib/agent.ts`; `deliveryAction(slot): "vault" | "file" | null`, `deliveryLine(slot): string | null` in `src/lib/key-slots.ts`; `keyArgs.delivery`, `useKeySetDelivery()` in `src/lib/sync.ts`; `KeepFileConfirm`, `AgentProblemLine` and the `"vault" | "file"` slot actions in `KeySlotsSection.tsx`.

Behavior (spec §6, §7.3, §7.4, §11, §14 item 1):
- A slot row offers `Only in SSHelter` when this computer has the key in a file and hosts use it here (`status.kind === "ready"`, `in_account`), and `Keep a file` whenever the slot is in the vault (always a way back out).
- Delete copy on a vault row says what it deletes: "The key {name} kept in SSHelter on this computer is deleted. If it is your only copy, it is gone. Other computers aren't affected." (the vault entry may be the last copy; Task 7 review).
- A vault row offers neither `Pick a key…`/`Change…` nor `Use the synced key`: the backend refuses both with "This key is only in SSHelter. Choose Keep a file first." (Task 7 ruling: the vault key may be the last copy). `slotActions` turns `pick` and `useSynced` off for `in_vault` slots; the status line still says when a synced key is available.
- `Keep a file` asks first: "Keep {name} as a file?" / "Any program on this computer can use the file without asking." / `Cancel` / `Keep a file`. `Only in SSHelter` needs no confirm; the first time any slot moves in, the success toast adds "Hosts that use it connect only while SSHelter is open. In Settings, turn on Launch at login and Keep running in menu bar when window closes." (spec §5.7; the hidden launch at login is Plan 3).
- A vault slot's row says "Only in SSHelter on this computer — programs ask before they use it".
- Above the rows, while any slot is in the vault: "SSHelter's agent isn't running: {reason}" when it failed to start, else "Hosts that use keys in SSHelter can't reach its agent." with a `Fix` button that puts the Include back first in `~/.ssh/config` (spec §6: never re-added on its own). Another SSHelter providing the agent is not a problem.
- The problem query lives under `["config", …]`, so every overview mutation and config change refreshes it.

- [ ] **Step 1: Write the failing Rust tests**

In `src-tauri/src/agent/wiring.rs`'s `tests` module, add:

```rust
    fn loaded(home: &Path, text: &str) -> SshConfigDoc {
        let main = home.join(".ssh").join("config");
        std::fs::create_dir_all(main.parent().unwrap()).unwrap();
        std::fs::write(&main, text).unwrap();
        crate::config::include::with_test_home(home, || crate::config::commands::load_doc_migrated(&main)).unwrap()
    }

    #[test]
    fn status_tells_whether_ssh_reaches_the_agent() {
        let home = tempfile::tempdir().unwrap();
        let none = BTreeSet::new();
        let doc = loaded(home.path(), "Host a\n  HostName x\n");
        assert_eq!(status(&doc, home.path(), &none), WiringStatus::NotNeeded);
        // A host on a vault key, but the first Include add never succeeded (agent/config is only written after it).
        let vault = BTreeSet::from(["id_mac-11111111".to_string()]);
        let on_vault = loaded(home.path(), "Host a\n  IdentityFile ~/.ssh/sshelter/keys/id_mac-11111111\n");
        assert_eq!(status(&on_vault, home.path(), &vault), WiringStatus::IncludeMissing, "needed but not wired yet");
        std::fs::create_dir_all(crate::agent::agent_dir(home.path())).unwrap();
        std::fs::write(agent_config_path(home.path()), format!("{HEADER}\n")).unwrap();
        assert_eq!(status(&doc, home.path(), &none), WiringStatus::IncludeMissing);
        let doc = loaded(home.path(), "Include ~/.ssh/sshelter/agent/config\nHost a\n");
        assert_eq!(status(&doc, home.path(), &none), WiringStatus::Ready);
    }

    #[test]
    fn fix_puts_the_include_back_first_once_and_saves() {
        let home = tempfile::tempdir().unwrap();
        let mut doc = loaded(home.path(), "# mine\nHost a\n  HostName x\n");
        let mut backed_up = HashSet::new();
        restore_include(&mut doc, &mut backed_up, None, home.path()).unwrap();
        let main = home.path().join(".ssh").join("config");
        assert_eq!(std::fs::read_to_string(&main).unwrap(), "Include ~/.ssh/sshelter/agent/config\n# mine\nHost a\n  HostName x\n");
        restore_include(&mut doc, &mut backed_up, None, home.path()).unwrap();
        assert_eq!(std::fs::read_to_string(&main).unwrap().matches("sshelter/agent/config").count(), 1);
    }
```

In `src-tauri/src/agent/mod.rs`, add a test module:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::agent::wiring::WiringStatus;

    #[test]
    fn a_failed_agent_comes_first_and_another_instance_is_fine() {
        assert_eq!(
            problem(&AgentStatus::Failed("path too long".into()), WiringStatus::IncludeMissing),
            Some(AgentProblem::NotRunning { reason: "path too long".into() })
        );
        assert_eq!(problem(&AgentStatus::Running, WiringStatus::IncludeMissing), Some(AgentProblem::IncludeMissing));
        assert_eq!(problem(&AgentStatus::OtherInstance, WiringStatus::Ready), None);
        assert_eq!(problem(&AgentStatus::Running, WiringStatus::NotNeeded), None);
    }
}
```

Run: `cargo test --offline --lib agent:: -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors (`status`, `restore_include`, `problem`, `AgentProblem` not found).

- [ ] **Step 2: Implement the Rust side**

In `src-tauri/src/agent/wiring.rs`, add after `refresh_env`:

```rust
/// 這台的 `ssh` 接不接得到 agent(畫面提示用;spec §6、§11):沒有主機用到只在 SSHelter 的金鑰、也從沒寫過 `agent/config` → NotNeeded;主 config
/// 是預設的 `~/.ssh/config` 而且有 agent 的 Include → Ready;其他(使用者拿掉了,或第一次一直加不進去 —— `agent/config` 要等 Include 放好才寫)→ IncludeMissing。
pub fn status(doc: &SshConfigDoc, home: &Path, vault_files: &BTreeSet<String>) -> WiringStatus {
    let needed = agent_config_path(home).exists() || !vault_host_patterns(doc, vault_files, home).is_empty();
    if !needed {
        return WiringStatus::NotNeeded;
    }
    let default_root = home.join(".ssh").join("config");
    match doc.files.first() {
        Some(main) if main.path == default_root && main.items.iter().any(is_agent_include) => WiringStatus::Ready,
        _ => WiringStatus::IncludeMissing,
    }
}

/// 使用者按 Fix:把 Include 放回 `~/.ssh/config` 的第一行(spec §6:不會自己加回去)。載入的主 config 不是預設的那一個 → 錯誤。
pub fn restore_include(
    doc: &mut SshConfigDoc,
    backed_up: &mut HashSet<PathBuf>,
    retention: Option<usize>,
    home: &Path,
) -> Result<(), AppError> {
    let default_root = home.join(".ssh").join("config");
    if doc.files.first().map(|f| f.path.as_path()) != Some(default_root.as_path()) {
        return Err(AppError::Other("SSHelter isn't using ~/.ssh/config, so it can't add the line there.".to_string()));
    }
    put_include_first(doc, backed_up, retention)
}
```

In `src-tauri/src/agent/mod.rs`, add (`serde::Serialize` import as needed):

```rust
/// 「Keys used by synced hosts」上方的提示(spec §6、§11)。
#[derive(Clone, Debug, PartialEq, Eq, serde::Serialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum AgentProblem {
    /// agent 開不起來:用保管庫金鑰的主機暫時連不上。
    NotRunning { reason: String },
    /// `agent/config` 在,`~/.ssh/config` 卻沒有它的 Include。
    IncludeMissing,
}

/// 要顯示的提示:agent 開不起來優先,其次是 Include 不見了。另一個 SSHelter 在提供不算問題。
pub fn problem(status: &AgentStatus, wiring: wiring::WiringStatus) -> Option<AgentProblem> {
    if let AgentStatus::Failed(reason) = status {
        return Some(AgentProblem::NotRunning { reason: reason.clone() });
    }
    (wiring == wiring::WiringStatus::IncludeMissing).then_some(AgentProblem::IncludeMissing)
}

/// 使用者的家目錄(`~/.ssh` 的上一層)。
pub(crate) fn home_dir() -> Result<PathBuf, AppError> {
    let ssh_dir = crate::keys::ssh_dir()?;
    ssh_dir.parent().map(Path::to_path_buf).ok_or_else(|| AppError::Other("cannot determine home directory".to_string()))
}

fn current_problem(state: &AppState) -> Result<Option<AgentProblem>, AppError> {
    let home = home_dir()?;
    let wiring = {
        // doc → core,同 `config/commands.rs` 的順序。
        let doc_lock = state.doc.lock().unwrap();
        let vault_files =
            state.sync.core.lock().unwrap().state.as_ref().map(crate::sync::slots::vault_slot_files).unwrap_or_default();
        doc_lock.as_ref().map_or(wiring::WiringStatus::NotNeeded, |doc| wiring::status(doc, &home, &vault_files))
    };
    let status = state.agent.status.lock().unwrap().clone();
    Ok(problem(&status, wiring))
}

#[tauri::command]
pub fn agent_problem(state: tauri::State<AppState>) -> Result<Option<AgentProblem>, AppError> {
    current_problem(&state)
}

/// Fix:把 Include 放回 `~/.ssh/config` 的第一行,回傳之後的提示。
#[tauri::command]
pub fn agent_fix_include(state: tauri::State<AppState>) -> Result<Option<AgentProblem>, AppError> {
    let home = home_dir()?;
    {
        let mut doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("The SSH config isn't loaded yet.".to_string()))?;
        let mut backed_up = state.backed_up.lock().unwrap();
        let retention = *state.backup_retention.lock().unwrap();
        wiring::restore_include(doc, &mut backed_up, retention, &home)?;
    }
    current_problem(&state)
}
```

Use the same lock order as the existing config commands (doc → backed_up → backup_retention); check one of them (for example `config_save_host`) and follow it if it differs. Register `agent::{agent_problem, agent_fix_include}` in `lib.rs` next to `agent_pending` / `agent_resolve`.

Run the Rust tests again (Step 1 command). Expected: PASS; `src/bindings/AgentProblem.ts` is written.

- [ ] **Step 3: Write the failing frontend tests**

Append to `src/lib/agent.test.ts`:

```ts
import { agentProblemText } from "@/lib/agent";

describe("the agent problem line", () => {
  it("says why hosts on vault keys can't connect", () => {
    expect(agentProblemText({ kind: "not_running", reason: "path too long" })).toBe("SSHelter's agent isn't running: path too long");
    expect(agentProblemText({ kind: "include_missing" })).toBe("Hosts that use keys in SSHelter can't reach its agent.");
  });
});
```

(Merge the import into the file's existing `@/lib/agent` import.)

Append to `src/lib/key-slots.test.ts` (import `deliveryAction`, `deliveryLine`, `slotActions` and `keySlot` the way the file already imports its helpers and fixtures):

```ts
describe("a vault row's actions", () => {
  it("hides Pick and Use the synced key, which the backend refuses for a vault key", () => {
    const actions = slotActions(keySlot({ in_vault: true, status: { kind: "synced_available", file: "/f" } }));
    expect(actions.pick).toBeNull();
    expect(actions.useSynced).toBe(false);
    expect(actions.stopSyncing).toBe(true);
    expect(slotActions(keySlot({ in_vault: true, mode: "own", fingerprint: null })).syncThis).toBe(true);
  });
});

describe("where this computer keeps a slot's key", () => {
  it("offers Only in SSHelter for a ready file and Keep a file for a vault key", () => {
    expect(deliveryAction(keySlot())).toBe("vault");
    expect(deliveryAction(keySlot({ in_vault: true }))).toBe("file");
    expect(deliveryAction(keySlot({ in_vault: true, in_account: false, status: { kind: "not_in_use", file: "/f" } }))).toBe("file");
    expect(deliveryAction(keySlot({ status: { kind: "needs_key", waiting_for_sync: false } }))).toBeNull();
    expect(deliveryAction(keySlot({ in_account: false }))).toBeNull();
  });

  it("says when the key is only in SSHelter", () => {
    expect(deliveryLine(keySlot({ in_vault: true }))).toBe("Only in SSHelter on this computer — programs ask before they use it");
    expect(deliveryLine(keySlot())).toBeNull();
  });
});
```

Append to `src/components/KeySlotsSection.test.tsx` (add `KeepFileConfirm` and `AgentProblemLine` to the import from `./KeySlotsSection`, and `agentProblemKey` from `@/lib/agent`):

```tsx
describe("keeping a slot's key only in SSHelter", () => {
  it("offers the move each way", () => {
    expect(row(keySlot())).toContain(">Only in SSHelter<");
    const vault = row(keySlot({ in_vault: true }));
    expect(vault).toContain(">Keep a file<");
    expect(vault).not.toContain(">Only in SSHelter<");
    expect(lines(vault)).toContain("Only in SSHelter on this computer — programs ask before they use it");
  });

  it("asks before the key becomes a file any program can use", () => {
    let kept = 0;
    const tree = KeepFileConfirm({ slot: keySlot({ name: SPOOFED_NAME }), open: true, onCancel: () => {}, onConfirm: () => kept++ });
    expect(textIn(elementsOf(tree, AlertDialogTitle))).toBe(`Keep ${SPOOFED_NAME_SHOWN} as a file?`);
    expect(elementsOf(tree, AlertDialogDescription).map(textIn)).toEqual(["Any program on this computer can use the file without asking."]);
    expect(elementsOf(tree, AlertDialogCancel).map(textIn)).toEqual(["Cancel"]);
    const [action] = elementsOf(tree, AlertDialogAction);
    expect(textIn(action)).toBe("Keep a file");
    action.props.onClick!();
    expect(kept).toBe(1);
  });

  it("shows why the agent can't be reached, with Fix for a removed Include", () => {
    const missing = renderToStaticMarkup(<AgentProblemLine problem={{ kind: "include_missing" }} busy={false} onFix={() => {}} />);
    expect(text(missing)).toContain("Hosts that use keys in SSHelter can't reach its agent.");
    expect(missing).toContain(">Fix<");
    const failed = renderToStaticMarkup(<AgentProblemLine problem={{ kind: "not_running", reason: "path too long" }} busy={false} onFix={() => {}} />);
    expect(text(failed)).toContain("SSHelter's agent isn't running: path too long");
    expect(failed).not.toContain(">Fix<");
  });

  it("warns that deleting a vault key may delete the only copy", () => {
    const tree = DeleteCopyConfirm({ slot: keySlot({ in_vault: true, status: { kind: "not_in_use", file: "/f" } }), open: true, onCancel: () => {}, onConfirm: () => {} });
    expect(elementsOf(tree, AlertDialogDescription).map(textIn)).toEqual([
      "The key id_mac kept in SSHelter on this computer is deleted. If it is your only copy, it is gone. Other computers aren't affected.",
    ]);
  });

  it("shows the problem above the rows only while a key is in the vault", () => {
    const render = (slot: SyncKeySlotView) => {
      const queryClient = new QueryClient();
      queryClient.setQueryData(syncOverviewKey, overview({ key_slots: [slot] }));
      queryClient.setQueryData(agentProblemKey, { kind: "include_missing" });
      return renderToStaticMarkup(
        <QueryClientProvider client={queryClient}>
          <KeySlotsSection />
        </QueryClientProvider>,
      );
    };
    expect(text(render(keySlot({ in_vault: true })))).toContain("can't reach its agent");
    expect(text(render(keySlot()))).not.toContain("can't reach its agent");
  });
});
```

Run: `./node_modules/.bin/vitest run src/lib/agent.test.ts src/lib/key-slots.test.ts src/components/KeySlotsSection.test.tsx`
Expected: FAIL (the new exports do not exist).

- [ ] **Step 4: Implement the frontend**

Append to `src/lib/agent.ts`:

```ts
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { AgentProblem } from "@/bindings/AgentProblem";
import { errorMessage } from "@/lib/sync";

/** Under ["config"], so every sync overview mutation and config change refreshes it. */
export const agentProblemKey = ["config", "agentProblem"] as const;

export function useAgentProblem(enabled: boolean) {
  return useQuery<AgentProblem | null>({
    queryKey: agentProblemKey,
    queryFn: () => tauriInvoke<AgentProblem | null>("agent_problem"),
    enabled,
  });
}

/** Fix: put the Include line back first in ~/.ssh/config (key vault spec §6). */
export function useFixAgentInclude() {
  const queryClient = useQueryClient();
  return useMutation<AgentProblem | null, unknown, void>({
    mutationFn: () => tauriInvoke<AgentProblem | null>("agent_fix_include"),
    onSuccess: (problem) => {
      queryClient.setQueryData(agentProblemKey, problem);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => toast.error("Could not fix ~/.ssh/config", { description: errorMessage(error) }),
  });
}

export function agentProblemText(problem: AgentProblem): string {
  switch (problem.kind) {
    case "not_running":
      return `SSHelter's agent isn't running: ${problem.reason}`;
    case "include_missing":
      return "Hosts that use keys in SSHelter can't reach its agent.";
  }
}
```

(Move the new imports to the top of the file with the others. If importing `errorMessage` from `@/lib/sync` creates an import cycle, copy the two-line helper `src/lib/mcp.ts` uses instead.)

Append to `src/lib/key-slots.ts`:

```ts
/**
 * "Only in SSHelter" / "Keep a file" (key vault spec §4.3, §7.3): where this computer's copy can move. A file this computer
 * uses can move into the vault; a vault key can always go back to a file.
 */
export function deliveryAction(slot: SyncKeySlotView): "vault" | "file" | null {
  if (slot.in_vault) return "file";
  return slot.in_account && slot.status.kind === "ready" ? "vault" : null;
}

export function deliveryLine(slot: SyncKeySlotView): string | null {
  return slot.in_vault ? "Only in SSHelter on this computer — programs ask before they use it" : null;
}
```

In `src/lib/sync.ts`, add to `keyArgs`:

```ts
  delivery: (v: { slotId: string; vault: boolean }) => ({ slotId: v.slotId, vault: v.vault }),
```

and after `useKeyDeleteCopy`:

```ts
/** It moves the private key between the slot file and SSHelter's vault before the state changes: re-read everything on failure. */
export function useKeySetDelivery() {
  return useOverviewMutation("sync_key_set_delivery", "Could not change where the key is kept", keyArgs.delivery, true);
}
```

In `src/components/KeySlotsSection.tsx`:

1. `export type SlotAction = "sync" | "stop" | "pick" | "useSynced" | "syncNew" | "delete" | "vault" | "file";`
2. In `KeySlotRow`, compute `const delivery = deliveryAction(slot);` and `const kept = deliveryLine(slot);`, render `{kept && <p className="text-xs text-muted-foreground">{kept}</p>}` after the status line, and add the buttons before `Stop syncing`:

```tsx
        {delivery === "vault" && button("vault", "Only in SSHelter")}
        {delivery === "file" && button("file", "Keep a file")}
```

3. Add the two components:

```tsx
/** The confirm before a vault key becomes a file again: any program can use a file without asking. No hooks: exported for the tests. */
export function KeepFileConfirm({
  slot,
  open,
  onCancel,
  onConfirm,
}: {
  slot: SyncKeySlotView | null;
  open: boolean;
  onCancel: () => void;
  onConfirm: () => void;
}) {
  return (
    <AlertDialog open={open} onOpenChange={(next) => !next && onCancel()}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Keep {slot ? revealHidden(slot.name) : ""} as a file?</AlertDialogTitle>
          <AlertDialogDescription>Any program on this computer can use the file without asking.</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={onConfirm}>Keep a file</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}

/** Why ssh can't reach SSHelter's agent (key vault spec §6, §11), with Fix when the Include line was removed. Exported for the tests. */
export function AgentProblemLine({ problem, busy, onFix }: { problem: AgentProblem; busy: boolean; onFix: () => void }) {
  return (
    <div className="flex items-center justify-between gap-3 px-1">
      <p className={cn("text-xs", TONE_TEXT.error)}>{agentProblemText(problem)}</p>
      {problem.kind === "include_missing" && (
        <Button type="button" size="sm" variant="outline" className="h-7 shrink-0" disabled={busy} onClick={onFix}>
          Fix
        </Button>
      )}
    </div>
  );
}
```

`KeepFileConfirm`'s title is one string in the tests: write it as a template literal (`{`Keep ${…} as a file?`}`) if the JSX above splits it into several text nodes and `textIn` then does not match.

In `DeleteCopyConfirm`, use that text when `slot?.in_vault` (keep today's text otherwise):

```tsx
          <AlertDialogDescription>
            {slot?.in_vault
              ? `The key ${revealHidden(slot.name)} kept in SSHelter on this computer is deleted. If it is your only copy, it is gone. Other computers aren't affected.`
              : `The copy of ${slot ? revealHidden(slot.name) : ""} on this computer is deleted. Other computers aren't affected.`}
          </AlertDialogDescription>
```

(The existing test for the non-vault text must still pass unchanged; if it compares one text node, the template literal keeps it one string.)

4. In `KeySlotsSection`: add `const delivery = useKeySetDelivery();`, `const fix = useFixAgentInclude();`, `const [keeping, setKeeping] = useState<SyncKeySlotView | null>(null);` and `const shownKeeping = useLastNonNull(keeping);` with the other hooks; after `const slots = …` and **before** `if (slots.length === 0) return null;` add

```tsx
  const anyInVault = slots.some((s) => s.in_vault);
  const problemQuery = useAgentProblem(anyInVault);
  const problem = anyInVault ? (problemQuery.data ?? null) : null;
```

include `delivery.isPending || fix.isPending` in `busy`, and handle the two actions in `act`:

```tsx
      case "vault":
        delivery.mutate(
          { slotId: slot.id, vault: true },
          {
            onSuccess: () =>
              toast.success(
                `${name} is now only in SSHelter on this computer`,
                anyInVault ? undefined : { description: "Hosts that use it connect only while SSHelter is open. In Settings, turn on Launch at login and Keep running in menu bar when window closes." },
              ),
          },
        );
        break;
      case "file":
        // A file can be used by any program without asking: confirm first (`KeepFileConfirm`).
        setKeeping(slot);
        break;
```

Render `{problem && <AgentProblemLine problem={problem} busy={busy} onFix={() => fix.mutate()} />}` right under the section title, and the confirm next to the others:

```tsx
      <KeepFileConfirm
        slot={shownKeeping}
        open={keeping !== null}
        onCancel={() => setKeeping(null)}
        onConfirm={() => {
          if (keeping) {
            const name = revealHidden(keeping.name);
            delivery.mutate({ slotId: keeping.id, vault: false }, { onSuccess: () => toast.success(`${name} is kept as a file on this computer`) });
          }
          setKeeping(null);
        }}
      />
```

Import `AgentProblem` (type), `agentProblemText`, `useAgentProblem`, `useFixAgentInclude`, `deliveryAction`, `deliveryLine` and `useKeySetDelivery`.

- [ ] **Step 5: Run the frontend checks**

Run: `./node_modules/.bin/vitest run src/lib/agent.test.ts src/lib/key-slots.test.ts src/components/KeySlotsSection.test.tsx`, then `./node_modules/.bin/tsc --noEmit`, `./node_modules/.bin/vitest run` and `./node_modules/.bin/vite build`.
Expected: all pass.

- [ ] **Step 6: Run the full Rust suite and commit**

```bash
git add src-tauri/src/agent src-tauri/src/lib.rs src/bindings/AgentProblem.ts src/lib/agent.ts src/lib/agent.test.ts src/lib/key-slots.ts src/lib/key-slots.test.ts src/lib/sync.ts src/components/KeySlotsSection.tsx src/components/KeySlotsSection.test.tsx
git commit -m "feat(keys): let a synced key live only in SSHelter on this computer"
```

---

### Task 12: Connect through a one-shot channel

**Files:**
- Create: `src-tauri/src/agent/oneshot.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`pub mod oneshot;`), `src-tauri/src/agent/peer.rs` (`executable_base`), `src-tauri/src/agent/pipe_windows.rs` (`one_shot`)
- Modify: `src-tauri/src/connect.rs` (`ssh_argv`; `connect_launch` takes the `AppHandle` and connects vault hosts through a channel), `src-tauri/src/tray.rs` (quick connect does the same, off the menu thread)

**Interfaces:**
- Consumes: Task 5 `peer::{proc_info (private), file_name (private)}`; Task 8 `broker::{Connection, Grant}`; Task 9 `server::{Stream, check_socket_path, peer}`, `pipe_windows::{accept, create_instance, wide_pipe_path, OwnerOnly}`, `AppAgentHost`, `agent_dir`; Task 11 `crate::agent::home_dir`; `crate::config::intel::effective_config(alias, None) -> Result<Vec<(String, String)>, AppError>`; `crate::sync::slot_rules::{resolve_identity_value, IdentityTarget}`; `crate::connect::{build_launch, build_launch_command, launch, validate_alias, detect_terminals}`.
- Produces:
  - `crate::agent::oneshot::{ONE_SHOT_TIMEOUT = 60 s, Channel { name, #[cfg(unix)] path }, Channel::identity_agent(&self) -> String, vault_slot_of(identity_files: &[String], state: &SyncStateV2, home: &Path) -> Option<String>, ssh_options(&Channel) -> Vec<String>, open(run_dir: &Path, serve: Box<dyn FnOnce(Stream, Option<u32>) + Send>, timeout: Duration) -> Result<Channel, AppError>, prepare(app: &AppHandle, alias: &str) -> Result<Option<Vec<String>>, AppError>}`
  - `crate::agent::peer::executable_base(pid: u32) -> Option<String>`
  - `#[cfg(windows)] crate::agent::pipe_windows::one_shot(name: &str) -> io::Result<OwnedHandle>`
  - `crate::connect::ssh_argv(options: &[String], alias: &str) -> Vec<String>`

Rules (spec §5.6; ledger rulings):
- A host goes through a channel when one of its `IdentityFile` values (in `ssh -G` order) names a vault slot; the first such slot is granted. Every other host connects exactly as before (password auto-fill included); a host on a vault key never uses password auto-fill.
- The channel is `~/.ssh/sshelter/agent/run/<8 hex>` (0600 in a 0700 directory) or `\\.\pipe\sshelter-connect-<32 hex>` (one instance, owner-only DACL, remote clients refused). The first same-user connection is taken and the endpoint is removed at once; nobody connecting within 60 seconds closes it. Only the granted key is listed and signed for, without an approval window; an encrypted key may still show the unlock-only prompt (ruling: the passphrase is asked when ssh asks, not before the terminal opens).
- Defense in depth: when the connecting process's executable is known and is not `ssh`, it is refused (only the `ssh` SSHelter just launched should use the grant).
- The terminal runs `ssh -o IdentityAgent=<channel> -o ForwardAgent=no <alias>`; on Unix the value is `~/.ssh/sshelter/agent/run/<name>` (ssh expands `~` itself, so a home path with spaces never enters the command line).
- Touch ID / Windows Hello before Connect is Plan 3.

- [ ] **Step 1: Write the failing tests**

Add `pub mod oneshot;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/oneshot.rs` with the tests first:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::agent::protocol::{read_frame, write_frame, SSH_AGENTC_REQUEST_IDENTITIES, SSH_AGENT_IDENTITIES_ANSWER};
    use crate::agent::server::testing::NoKeys;
    use crate::sync::state_v2::LocalSlot;
    use std::sync::mpsc;

    fn slot(file: &str, source: Option<SlotSource>) -> LocalSlot {
        LocalSlot {
            file_name: file.into(),
            source,
            last_error: None,
            asked: false,
            payload: None,
            uploaded_fingerprint: None,
            parked: false,
            learned_in: None,
            copy_from_another_account: false,
        }
    }

    #[test]
    fn the_first_identity_file_on_a_vault_slot_picks_the_key() {
        let home = Path::new("/h");
        let mut state = SyncStateV2::fresh("mac").unwrap();
        let vault = SlotSource::Vault { fingerprint: "SHA256:v".into(), public_key: "ssh-ed25519 AAAA".into(), has_passphrase: false };
        state.key_slots.insert("a".repeat(32), slot("id_mac-aaaaaaaa", Some(vault)));
        state.key_slots.insert("b".repeat(32), slot("work-bbbbbbbb", Some(SlotSource::SyncedCopy { fingerprint: "SHA256:w".into() })));
        let files = |values: &[&str]| values.iter().map(|v| v.to_string()).collect::<Vec<_>>();
        assert_eq!(
            vault_slot_of(&files(&["~/.ssh/id_rsa", "~/.ssh/sshelter/keys/id_mac-aaaaaaaa"]), &state, home),
            Some("a".repeat(32))
        );
        assert_eq!(vault_slot_of(&files(&["%d/.ssh/sshelter/keys/id_mac-aaaaaaaa"]), &state, home), Some("a".repeat(32)));
        assert_eq!(vault_slot_of(&files(&["~/.ssh/sshelter/keys/work-bbbbbbbb"]), &state, home), None, "a file slot is not a vault key");
        assert_eq!(vault_slot_of(&files(&["~/.ssh/id_rsa"]), &state, home), None);
    }

    #[cfg(unix)]
    #[test]
    fn ssh_gets_the_channel_with_tilde_and_no_forwarding() {
        let channel = Channel { name: "0a1b2c3d".into(), path: PathBuf::from("/x/run/0a1b2c3d") };
        assert_eq!(
            ssh_options(&channel),
            vec!["-o", "IdentityAgent=~/.ssh/sshelter/agent/run/0a1b2c3d", "-o", "ForwardAgent=no"]
        );
    }

    #[cfg(windows)]
    #[test]
    fn ssh_gets_the_channel_as_a_forward_slash_pipe_and_no_forwarding() {
        let channel = Channel { name: "sshelter-connect-00".into() };
        assert_eq!(ssh_options(&channel), vec!["-o", "IdentityAgent=//./pipe/sshelter-connect-00", "-o", "ForwardAgent=no"]);
    }

    fn serving(pids: mpsc::Sender<Option<u32>>) -> Box<dyn FnOnce(Stream, Option<u32>) + Send> {
        Box::new(move |mut stream, pid| {
            let _ = pids.send(pid);
            let _ = crate::agent::session::serve(&mut stream, &NoKeys);
        })
    }

    #[cfg(unix)]
    fn short_dir() -> tempfile::TempDir {
        tempfile::Builder::new().prefix("so").tempdir_in("/tmp").unwrap()
    }

    #[cfg(unix)]
    #[test]
    fn the_first_connection_is_served_and_the_socket_disappears() {
        use std::os::unix::net::UnixStream;
        let dir = short_dir();
        let (tx, rx) = mpsc::channel();
        let channel = open(&dir.path().join("run"), serving(tx), Duration::from_secs(5)).unwrap();
        assert_eq!(channel.name.len(), 8);
        let mut stream = UnixStream::connect(&channel.path).unwrap();
        write_frame(&mut stream, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();
        assert_eq!(read_frame(&mut stream).unwrap().unwrap()[0], SSH_AGENT_IDENTITIES_ANSWER);
        assert_eq!(rx.recv_timeout(Duration::from_secs(5)).unwrap(), Some(std::process::id()));
        assert!(!channel.path.exists(), "removed as soon as the first connection arrived");
        assert!(UnixStream::connect(&channel.path).is_err(), "no second connection");
    }

    #[cfg(unix)]
    #[test]
    fn a_channel_nobody_uses_closes_after_the_timeout() {
        let dir = short_dir();
        let (tx, rx) = mpsc::channel();
        let channel = open(&dir.path().join("run"), serving(tx), Duration::from_millis(100)).unwrap();
        let deadline = std::time::Instant::now() + Duration::from_secs(5);
        while channel.path.exists() && std::time::Instant::now() < deadline {
            std::thread::sleep(Duration::from_millis(20));
        }
        assert!(!channel.path.exists());
        assert!(rx.recv_timeout(Duration::from_millis(200)).is_err(), "nothing was served");
    }

    #[cfg(windows)]
    #[test]
    fn the_first_pipe_client_is_served_and_nobody_else() {
        let dir = tempfile::tempdir().unwrap();
        let (tx, rx) = mpsc::channel();
        let channel = open(&dir.path().join("run"), serving(tx), Duration::from_secs(5)).unwrap();
        let path = format!(r"\\.\pipe\{}", channel.name);
        let mut client = std::fs::OpenOptions::new().read(true).write(true).open(&path).unwrap();
        write_frame(&mut client, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();
        assert_eq!(read_frame(&mut client).unwrap().unwrap()[0], SSH_AGENT_IDENTITIES_ANSWER);
        assert_eq!(rx.recv_timeout(Duration::from_secs(5)).unwrap(), Some(std::process::id()));
        assert!(std::fs::OpenOptions::new().read(true).write(true).open(&path).is_err(), "one instance only");
    }
}
```

In the Windows test the client is this test process, so `serve` sees its own PID: that is the expected value. (Production's `serve` closure, not `open`, decides whom to refuse.)

Add to `src-tauri/src/agent/peer.rs`'s tests:

```rust
    #[cfg(unix)]
    #[test]
    fn the_executable_base_name_of_a_live_process_and_of_a_gone_one() {
        let base = executable_base(std::process::id()).unwrap();
        assert!(!base.is_empty() && !base.contains('/'));
        assert_eq!(executable_base(u32::MAX - 7), None);
    }
```

Add to `src-tauri/src/connect.rs`'s tests:

```rust
    #[test]
    fn ssh_argv_puts_the_options_before_the_alias() {
        let options = vec!["-o".to_string(), "ForwardAgent=no".to_string()];
        assert_eq!(ssh_argv(&options, "web"), vec!["ssh", "-o", "ForwardAgent=no", "web"]);
        assert_eq!(ssh_argv(&[], "web"), vec!["ssh", "web"]);
    }
```

Run: `cargo test --offline --lib -- agent::oneshot agent::peer connect:: --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: compile errors.

- [ ] **Step 2: Implement `oneshot.rs`**

Put above the tests:

```rust
//! Connect 的一次性通道(金鑰保管庫 spec §5.6):從 SSHelter 按 Connect(或系統匣的快速連線)連到用「只在 SSHelter」金鑰的主機時,開一個只給這次
//! 連線的 agent 端點,在終端機執行 `ssh -o IdentityAgent=<通道> -o ForwardAgent=no <主機>`。通道只接受同一使用者的第一個連線,60 秒內沒人連
//! 就關掉,連線結束就關掉;裡面只提供這台主機的金鑰,而且已經核准(`broker::Grant`:不跳核准視窗、不算進記住的核准)。

use std::path::Path;
#[cfg(unix)]
use std::path::PathBuf;
use std::time::Duration;

use tauri::Manager;

use crate::agent::broker::{Connection, Grant};
use crate::agent::server::Stream;
use crate::agent::{agent_dir, home_dir, peer, session, AppAgentHost};
use crate::error::AppError;
use crate::state::AppState;
use crate::sync::slot_rules::{resolve_identity_value, IdentityTarget};
use crate::sync::state_v2::{SlotSource, SyncStateV2};

pub const ONE_SHOT_TIMEOUT: Duration = Duration::from_secs(60);

/// 一個開著、等第一個連線的通道。
#[derive(Debug)]
pub struct Channel {
    /// Unix:`run/` 裡的 socket 檔名;Windows:pipe 名稱。
    pub name: String,
    /// socket 的路徑(測試用它連)。
    #[cfg(unix)]
    pub path: PathBuf,
}

impl Channel {
    /// `ssh -o IdentityAgent=` 的值:Unix 用 `~/…`(ssh 自己展開,命令裡不放可能含空白的家目錄路徑),Windows 用正斜線的 pipe 路徑。
    pub fn identity_agent(&self) -> String {
        #[cfg(windows)]
        {
            format!("//./pipe/{}", self.name)
        }
        #[cfg(not(windows))]
        {
            format!("~/.ssh/sshelter/agent/run/{}", self.name)
        }
    }
}

/// `identity_files`(`ssh -G` 的 `identityfile` 值,依序;`~`、`%d` 不展開)裡第一個指到這台「只在 SSHelter」插槽的,回傳插槽 id。
pub fn vault_slot_of(identity_files: &[String], state: &SyncStateV2, home: &Path) -> Option<String> {
    identity_files.iter().find_map(|value| {
        let IdentityTarget::Slot(file) = resolve_identity_value(value, home) else { return None };
        state
            .key_slots
            .iter()
            .find(|(_, local)| local.file_name == file && matches!(local.source, Some(SlotSource::Vault { .. })))
            .map(|(id, _)| id.clone())
    })
}

/// ssh 要多帶的選項(放在主機名稱之前)。
pub fn ssh_options(channel: &Channel) -> Vec<String> {
    vec!["-o".into(), format!("IdentityAgent={}", channel.identity_agent()), "-o".into(), "ForwardAgent=no".into()]
}

fn random_hex(bytes: usize) -> Result<String, AppError> {
    let mut buf = vec![0u8; bytes];
    getrandom::fill(&mut buf).map_err(|e| AppError::Other(format!("cannot draw a channel name: {e}")))?;
    Ok(buf.iter().map(|b| format!("{b:02x}")).collect())
}

/// 在 `run_dir`(`~/.ssh/sshelter/agent/run`)開一個 socket,背景等第一個同一使用者的連線並交給 `serve`;`timeout` 內沒人連就關掉。
/// 第一個連線一到 socket 檔就移除,不會有第二個連線。
#[cfg(unix)]
pub fn open(run_dir: &Path, serve: Box<dyn FnOnce(Stream, Option<u32>) + Send>, timeout: Duration) -> Result<Channel, AppError> {
    use std::os::unix::fs::PermissionsExt;
    use std::os::unix::net::UnixListener;

    crate::sync::slot_files::ensure_keys_dir(run_dir)?;
    let name = random_hex(4)?;
    let path = run_dir.join(&name);
    crate::agent::server::check_socket_path(&path)?;
    let listener = UnixListener::bind(&path)?;
    std::fs::set_permissions(&path, std::fs::Permissions::from_mode(0o600))?;
    listener.set_nonblocking(true)?;
    let socket = path.clone();
    std::thread::Builder::new().name("sshelter-agent-connect".to_string()).spawn(move || {
        let deadline = std::time::Instant::now() + timeout;
        let accepted = loop {
            match listener.accept() {
                Ok((stream, _)) => break Some(stream),
                Err(e)
                    if matches!(e.kind(), std::io::ErrorKind::WouldBlock | std::io::ErrorKind::Interrupted)
                        && std::time::Instant::now() < deadline =>
                {
                    std::thread::sleep(Duration::from_millis(50));
                }
                Err(_) => break None,
            }
        };
        drop(listener);
        let _ = std::fs::remove_file(&socket);
        if let Some(stream) = accepted {
            // macOS 的 accept 沿用 listener 的 non-blocking;不是同一個使用者就不服務。
            if stream.set_nonblocking(false).is_ok() {
                if let Ok(pid) = crate::agent::server::peer(&stream) {
                    let _ = stream.set_read_timeout(Some(crate::agent::server::IDLE_TIMEOUT));
                    serve(stream, pid);
                }
            }
        }
    })?;
    Ok(Channel { name, path })
}

/// Windows:一次性的 pipe(只有一個 instance)。`ConnectNamedPipe` 沒有逾時,時間到還沒人連就自己連一次讓等待結束(認得出是自己,不服務)。
#[cfg(windows)]
pub fn open(_run_dir: &Path, serve: Box<dyn FnOnce(Stream, Option<u32>) + Send>, timeout: Duration) -> Result<Channel, AppError> {
    use std::sync::atomic::{AtomicBool, Ordering};
    use std::sync::Arc;

    let name = format!("sshelter-connect-{}", random_hex(16)?);
    let pipe = crate::agent::pipe_windows::one_shot(&name)?;
    let client_path = format!(r"\\.\pipe\{name}");
    std::thread::Builder::new().name("sshelter-agent-connect".to_string()).spawn(move || {
        let connected = Arc::new(AtomicBool::new(false));
        let woken = Arc::new(AtomicBool::new(false));
        {
            let (connected, woken) = (Arc::clone(&connected), Arc::clone(&woken));
            std::thread::spawn(move || {
                std::thread::sleep(timeout);
                if !connected.load(Ordering::SeqCst) {
                    woken.store(true, Ordering::SeqCst);
                    let _ = std::fs::OpenOptions::new().read(true).write(true).open(&client_path);
                }
            });
        }
        let accepted = crate::agent::pipe_windows::accept(&pipe);
        connected.store(true, Ordering::SeqCst);
        if let Ok(pid) = accepted {
            if !woken.load(Ordering::SeqCst) {
                serve(std::fs::File::from(pipe), pid);
            }
        }
    })?;
    Ok(Channel { name })
}

/// Connect 之前:這台主機用的是「只在 SSHelter」的金鑰 → 開一次性通道,回傳 ssh 要多帶的選項;不是(或這台沒有任何這種金鑰)→ None。
/// 會執行 `ssh -G`:不要在主執行緒呼叫。
pub fn prepare(app: &tauri::AppHandle, alias: &str) -> Result<Option<Vec<String>>, AppError> {
    let state = app.state::<AppState>();
    let any_vault = state
        .sync
        .core
        .lock()
        .unwrap()
        .state
        .as_ref()
        .is_some_and(|s| s.key_slots.values().any(|l| matches!(l.source, Some(SlotSource::Vault { .. }))));
    if !any_vault {
        return Ok(None);
    }
    let identity_files: Vec<String> = crate::config::intel::effective_config(alias, None)?
        .into_iter()
        .filter(|(key, _)| key == "identityfile")
        .map(|(_, value)| value)
        .collect();
    let home = home_dir()?;
    let slot_id = state.sync.core.lock().unwrap().state.as_ref().and_then(|s| vault_slot_of(&identity_files, s, &home));
    let Some(slot_id) = slot_id else { return Ok(None) };
    let app = app.clone();
    let serve: Box<dyn FnOnce(Stream, Option<u32>) + Send> = Box::new(move |mut stream, pid| {
        // 只服務剛在終端機啟動的 ssh(認得出程式的時候):同一使用者的其他程式搶先連上來,拿不到這個已經核准的授權。
        if pid.and_then(peer::executable_base).is_some_and(|base| base != "ssh") {
            return;
        }
        let state = app.state::<AppState>();
        let host = AppAgentHost { app: app.clone() };
        let connection = Connection { broker: &state.agent.broker, host: &host, program: None, grant: Some(Grant { slot_id }) };
        if let Err(e) = session::serve(&mut stream, &connection) {
            eprintln!("[agent] connect channel closed: {e}");
        }
    });
    let channel = open(&agent_dir(&home).join("run"), serve, ONE_SHOT_TIMEOUT)?;
    Ok(Some(ssh_options(&channel)))
}
```

`ssh -G` prints `identityfile` values as written (`~/…`, `%d/…`, not expanded; checked with OpenSSH 10.3p1), which is what `resolve_identity_value` expects.

In `src-tauri/src/agent/peer.rs`, add after `process_chain`:

```rust
/// `pid` 的執行檔名稱(小寫、去掉 `.exe`);讀不到(程序已經結束)→ None。
pub fn executable_base(pid: u32) -> Option<String> {
    proc_info(pid)?.path.map(|path| file_name(&path).to_ascii_lowercase())
}
```

In `src-tauri/src/agent/pipe_windows.rs`, add after `accept`:

```rust
/// Connect 的一次性 pipe:只有一個 instance,名稱已經存在就失敗(`FILE_FLAG_FIRST_PIPE_INSTANCE`)。描述元在建立時複製,用完即丟。
pub(crate) fn one_shot(name: &str) -> io::Result<OwnedHandle> {
    let security = OwnerOnly::new()?;
    create_instance(&wide_pipe_path(name), &security, true, 1)
}
```

- [ ] **Step 3: Use it from Connect and the tray**

In `src-tauri/src/connect.rs`, add:

```rust
/// `ssh <options…> <alias>`。
pub fn ssh_argv(options: &[String], alias: &str) -> Vec<String> {
    std::iter::once("ssh".to_string()).chain(options.iter().cloned()).chain(std::iter::once(alias.to_string())).collect()
}
```

Give `connect_launch` the app handle (Tauri injects it; the frontend call is unchanged) and choose the launch:

```rust
#[tauri::command(async)]
pub fn connect_launch(
    app: tauri::AppHandle,
    state: tauri::State<crate::state::AppState>,
    alias: String,
    terminal_override: Option<String>,
    new_tab: Option<bool>,
) -> Result<(), AppError> {
```

and replace the `let spec = match password_autofill_env(…) { … };` with:

```rust
    // 用「只在 SSHelter」金鑰的主機經一次性通道連(金鑰保管庫 spec §5.6);其他主機照舊(含密碼自動填入)。
    let spec = match crate::agent::oneshot::prepare(&app, &alias)? {
        Some(options) => build_launch_command(&terminal_id, &ssh_argv(&options, &alias), new_tab)?,
        None => match password_autofill_env(&state, &alias) {
            Some(env_pairs) => build_autofill_launch(&terminal_id, &alias, new_tab, &env_pairs)?,
            None => build_launch(&terminal_id, &alias, new_tab)?,
        },
    };
```

In `src-tauri/src/tray.rs`, run the quick connect off the menu thread and release the doc lock before `prepare` (it runs `ssh -G`; the lock order stays doc → core):

```rust
            if let Some(alias) = other.strip_prefix("connect:") {
                // Off the menu thread: a host on a vault key runs `ssh -G` first (key vault spec §5.6).
                let (app, alias) = (app.clone(), alias.to_string());
                std::thread::spawn(move || {
                    if let Err(e) = quick_connect(&app, &alias) {
                        eprintln!("[tray] quick-connect '{alias}' failed: {e}");
                    }
                });
            }
```

```rust
/// Validate, then launch with the first detected terminal: a host on a vault key through a one-shot channel.
fn quick_connect(app: &tauri::AppHandle, alias: &str) -> Result<(), crate::error::AppError> {
    use crate::error::AppError;

    let state = app.state::<crate::state::AppState>();
    {
        let doc_lock = state.doc.lock().unwrap();
        let doc = doc_lock.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        crate::connect::validate_alias(doc, alias)?;
    }

    let terminal = crate::connect::detect_terminals()
        .into_iter()
        .next()
        .ok_or_else(|| AppError::Other("no terminal found".to_string()))?;

    let spec = match crate::agent::oneshot::prepare(app, alias)? {
        Some(options) => crate::connect::build_launch_command(&terminal.id, &crate::connect::ssh_argv(&options, alias), false)?,
        None => crate::connect::build_launch(&terminal.id, alias, false)?,
    };
    crate::connect::launch(&spec)
}
```

- [ ] **Step 4: Run the tests, type-check Windows, run the full suite, commit**

Run: the Step 1 command (expect PASS), then the scratch-crate Windows check from Task 9 Step 6 with `oneshot.rs`'s Windows branch added (stub `prepare`'s Tauri parts out of the scratch crate, or include only `open` and `Channel`), then the full Rust suite. Then `./node_modules/.bin/tsc --noEmit` from the repo root (the frontend call to `connect_launch` is unchanged; this just confirms nothing else moved).

```bash
git add src-tauri/src/agent src-tauri/src/connect.rs src-tauri/src/tray.rs
git commit -m "feat(agent): connect hosts on vault keys through a one-shot agent channel"
```

---

### Task 13: Real OpenSSH tests, Windows CI, manual checklist

**Files:**
- Create: `src-tauri/src/agent/openssh_tests.rs`
- Modify: `src-tauri/src/agent/mod.rs` (`#[cfg(test)] mod openssh_tests;`)
- Modify: `.github/workflows/test-windows.yml`
- Create: `docs/superpowers/plans/2026-10-07-key-vault-agent-manual-verification.md`

**Interfaces:**
- Consumes: Tasks 2–12 (`broker::{AgentHost, Broker, Connection, Grant, VaultKey}`, `server::{Handler, Stream, listen_unix}`, `pipe_windows::listen`, `oneshot::open`, `peer::{identify, process_chain}`, `session::serve`, `test_keys::{PLAIN_*, ECDSA_*, RSA_*, plain, ecdsa, rsa}`).
- Produces: tests and documents only.

Spec §12: the agent is tested with the real OpenSSH tools and no sshd — `ssh-add -L` lists, `ssh-keygen -Y sign` signs through the agent and `ssh-keygen -Y verify` checks the signature — on macOS and on Windows CI (Windows uses its own pipe). The tools read only `SSH_AUTH_SOCK` and files in the test directory; nothing touches the real `~/.ssh`, and nothing starts an sshd (an sshd with agent forwarding once created a socket in the real `~/.ssh` during the spike).

- [ ] **Step 1: Write the integration tests**

Add `#[cfg(test)] mod openssh_tests;` to `src-tauri/src/agent/mod.rs`. Create `src-tauri/src/agent/openssh_tests.rs`:

```rust
//! 用真的 OpenSSH 工具測 agent(金鑰保管庫 spec §12):`ssh-add -L` 列出、`ssh-keygen -Y sign` 經 agent 簽章、`ssh-keygen -Y verify` 驗證。
//! 不需要 sshd,也不碰真的 `~/.ssh`:工具只讀 `SSH_AUTH_SOCK` 與測試目錄裡的檔案。工具不在 PATH 上就略過;CI 設
//! `SSHELTER_REQUIRE_OPENSSH=1` 時改成失敗。

use std::collections::HashMap;
use std::process::Command;
use std::sync::{Arc, Mutex};
use std::time::Duration;

use ssh_key::public::KeyData;
use zeroize::Zeroizing;

use crate::agent::broker::{AgentHost, Broker, Connection, Grant, VaultKey};
use crate::agent::prompt::{AgentApprovalAnswer, AgentApprovalRequest};
use crate::agent::server::{Handler, Stream};
use crate::agent::{oneshot, peer, session};
use crate::error::AppError;
use crate::sync::env::{Clock, Keychain, SystemClock};
use crate::sync::slot_rules::test_keys;
use crate::sync::testkit::MemKeychain;
use crate::vault::store::AgentSettings;

const IDS: [&str; 3] = ["11111111111111111111111111111111", "22222222222222222222222222222222", "33333333333333333333333333333333"];

/// 三把測試金鑰,每次都允許(並記下問了什麼)。
struct TestHost {
    keys: Vec<VaultKey>,
    private: HashMap<String, String>,
    keychain: MemKeychain,
    asked: Mutex<Vec<AgentApprovalRequest>>,
}

impl TestHost {
    fn new() -> Self {
        let key = |id: &str, name: &str, public: &str, fingerprint: &str| VaultKey {
            slot_id: id.into(),
            name: name.into(),
            fingerprint: fingerprint.into(),
            public_key: public.into(),
            has_passphrase: false,
        };
        TestHost {
            keys: vec![
                key(IDS[0], "ed25519", test_keys::PLAIN_PUBLIC, test_keys::PLAIN_FINGERPRINT),
                key(IDS[1], "ecdsa", test_keys::ECDSA_PUBLIC, test_keys::ECDSA_FINGERPRINT),
                key(IDS[2], "rsa", test_keys::RSA_PUBLIC, test_keys::RSA_FINGERPRINT),
            ],
            private: HashMap::from([
                (IDS[0].to_string(), test_keys::plain()),
                (IDS[1].to_string(), test_keys::ecdsa()),
                (IDS[2].to_string(), test_keys::rsa()),
            ]),
            keychain: MemKeychain::default(),
            asked: Mutex::new(Vec::new()),
        }
    }
}

impl AgentHost for TestHost {
    fn keys(&self) -> Vec<VaultKey> {
        self.keys.clone()
    }
    fn private_key(&self, slot_id: &str) -> Result<Option<Zeroizing<String>>, AppError> {
        Ok(self.private.get(slot_id).cloned().map(Zeroizing::new))
    }
    fn settings(&self) -> AgentSettings {
        AgentSettings::default()
    }
    fn keychain(&self) -> &dyn Keychain {
        &self.keychain
    }
    fn now_ms(&self) -> u64 {
        SystemClock.now_ms()
    }
    fn host_name(&self, _host_key: &KeyData) -> Option<String> {
        None
    }
    fn ask(&self, request: AgentApprovalRequest) -> Option<AgentApprovalAnswer> {
        self.asked.lock().unwrap().push(request);
        Some(AgentApprovalAnswer { allow: true, ..Default::default() })
    }
}

/// OpenSSH 的工具在不在 PATH 上;`SSHELTER_REQUIRE_OPENSSH` 有設時不在就失敗。
fn have(tool: &str) -> bool {
    let found = Command::new(tool).arg("-?").output().is_ok();
    if !found {
        assert!(std::env::var_os("SSHELTER_REQUIRE_OPENSSH").is_none(), "{tool} is not on PATH");
        eprintln!("skipped: {tool} is not on PATH");
    }
    found
}

/// macOS 的暫存目錄路徑太長,放不下 socket:Unix 用 /tmp 底下的短目錄。
fn scratch() -> tempfile::TempDir {
    #[cfg(unix)]
    {
        tempfile::Builder::new().prefix("so").tempdir_in("/tmp").unwrap()
    }
    #[cfg(windows)]
    {
        tempfile::tempdir().unwrap()
    }
}

struct Running {
    dir: tempfile::TempDir,
    /// `SSH_AUTH_SOCK` 的值。
    endpoint: String,
    host: Arc<TestHost>,
    broker: Arc<Broker>,
}

/// 在暫存目錄開一個 agent(Windows 用這個測試自己的 pipe 名稱)。
fn start() -> Running {
    let dir = scratch();
    let host = Arc::new(TestHost::new());
    let broker = Arc::new(Broker::default());
    let handler: Handler = {
        let (host, broker) = (Arc::clone(&host), Arc::clone(&broker));
        Arc::new(move |mut stream: Stream, pid: Option<u32>| {
            let program = pid.and_then(|pid| peer::identify(&peer::process_chain(pid)));
            let connection = Connection { broker: &broker, host: host.as_ref(), program, grant: None };
            let _ = session::serve(&mut stream, &connection);
        })
    };
    let agent_dir = dir.path().join("agent");
    #[cfg(unix)]
    let endpoint = {
        crate::agent::server::listen_unix(&agent_dir, handler).unwrap();
        agent_dir.join("sock").display().to_string()
    };
    #[cfg(windows)]
    let endpoint = {
        use std::sync::atomic::{AtomicUsize, Ordering};
        static NEXT: AtomicUsize = AtomicUsize::new(0);
        let name = format!("sshelter-ossh-{}-{}", std::process::id(), NEXT.fetch_add(1, Ordering::SeqCst));
        crate::agent::pipe_windows::listen(&agent_dir, &name, handler).unwrap();
        format!(r"\\.\pipe\{name}")
    };
    Running { dir, endpoint, host, broker }
}

fn stderr(out: &std::process::Output) -> String {
    String::from_utf8_lossy(&out.stderr).into_owned()
}

#[test]
fn ssh_add_lists_the_vault_keys_without_asking() {
    if !have("ssh-add") {
        return;
    }
    let agent = start();
    let out = Command::new("ssh-add").arg("-L").env("SSH_AUTH_SOCK", &agent.endpoint).output().unwrap();
    assert!(out.status.success(), "{}", stderr(&out));
    let listed = String::from_utf8_lossy(&out.stdout);
    for public in [test_keys::PLAIN_PUBLIC, test_keys::ECDSA_PUBLIC, test_keys::RSA_PUBLIC] {
        assert!(listed.contains(public), "{public} missing from {listed}");
    }
    assert!(agent.host.asked.lock().unwrap().is_empty(), "listing never asks");
}

#[test]
fn ssh_keygen_signs_through_the_agent_and_the_signature_verifies() {
    if !have("ssh-keygen") {
        return;
    }
    let agent = start();
    let files = agent.dir.path();
    let data = files.join("data.txt");
    std::fs::write(&data, b"sign me").unwrap();
    let signature = files.join("data.txt.sig");
    for (name, public) in [("ed25519", test_keys::PLAIN_PUBLIC), ("ecdsa", test_keys::ECDSA_PUBLIC), ("rsa", test_keys::RSA_PUBLIC)] {
        let public_file = files.join(format!("{name}.pub"));
        std::fs::write(&public_file, format!("{public} {name}\n")).unwrap();
        let _ = std::fs::remove_file(&signature);
        let out = Command::new("ssh-keygen")
            .args(["-Y", "sign", "-n", "test", "-f"])
            .arg(&public_file)
            .arg(&data)
            .env("SSH_AUTH_SOCK", &agent.endpoint)
            .output()
            .unwrap();
        assert!(out.status.success(), "{name}: {}", stderr(&out));
        let signers = files.join("allowed_signers");
        std::fs::write(&signers, format!("test@sshelter {public}\n")).unwrap();
        let verify = Command::new("ssh-keygen")
            .args(["-Y", "verify", "-n", "test", "-I", "test@sshelter", "-f"])
            .arg(&signers)
            .arg("-s")
            .arg(&signature)
            .stdin(std::fs::File::open(&data).unwrap())
            .output()
            .unwrap();
        assert!(verify.status.success(), "{name}: {}", stderr(&verify));
    }
    let asked = agent.host.asked.lock().unwrap();
    assert_eq!(asked.len(), 3, "each signature was approved");
    assert!(asked.iter().all(|r| r.host.is_none() && !r.rememberable), "an SSHSIG request has no host and is never remembered");
}

#[test]
fn a_one_shot_channel_lists_only_its_key_and_only_once() {
    if !have("ssh-add") {
        return;
    }
    let agent = start();
    let serve: Box<dyn FnOnce(Stream, Option<u32>) + Send> = {
        let (host, broker) = (Arc::clone(&agent.host), Arc::clone(&agent.broker));
        Box::new(move |mut stream, _pid| {
            let grant = Some(Grant { slot_id: IDS[1].into() });
            let connection = Connection { broker: &broker, host: host.as_ref(), program: None, grant };
            let _ = session::serve(&mut stream, &connection);
        })
    };
    // `late` 不會被呼叫(10 秒內就連上了);`channel` 要留到兩次 ssh-add 都跑完:沒有 `keep` 就丟掉會取消通道。
    let channel =
        oneshot::open(&agent.dir.path().join("run"), serve, Box::new(|| {}), Duration::from_secs(10), Duration::from_secs(10)).unwrap();
    #[cfg(unix)]
    let endpoint = channel.path.display().to_string();
    #[cfg(windows)]
    let endpoint = format!(r"\\.\pipe\{}", channel.name);
    let out = Command::new("ssh-add").arg("-L").env("SSH_AUTH_SOCK", &endpoint).output().unwrap();
    assert!(out.status.success(), "{}", stderr(&out));
    let listed = String::from_utf8_lossy(&out.stdout);
    assert!(listed.contains(test_keys::ECDSA_PUBLIC), "{listed}");
    assert!(!listed.contains(test_keys::PLAIN_PUBLIC), "only the granted key: {listed}");
    let again = Command::new("ssh-add").arg("-L").env("SSH_AUTH_SOCK", &endpoint).output().unwrap();
    assert!(!again.status.success(), "the channel is gone after its first connection");
}
```

- [ ] **Step 2: Run them**

Run: `cargo test --offline --lib agent::openssh_tests -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS (3 tests) with macOS's OpenSSH (`ssh-add -L` and `ssh-keygen -Y` exist in 10.3p1). If one fails, the failure is in the agent (these tests add no production code): report it with the tool's stderr instead of weakening the test.

- [ ] **Step 3: Run the vault and agent tests on Windows CI**

Edit `.github/workflows/test-windows.yml`: update the header comment to say it also runs the vault and agent tests (the named pipe, its owner-only DACL, Windows program identification, and the real Win32-OpenSSH tools), and replace the last step with:

```yaml
      # The platform-dependent tests: identity path parsing (slot_rules), the slot files, links and DACL (slot_files,
      # slot_files_windows), the key vault (vault::) and the SSH agent (agent::: the named pipe, program identification and
      # the real ssh-add / ssh-keygen against the agent). The rest of the suite runs on macOS; parts of it use the system keychain.
      - name: Key slot, vault and agent tests
        env:
          SSHELTER_REQUIRE_OPENSSH: "1"
        # Quoted: an unquoted `vault:: agent::` is read as a YAML mapping and the workflow would not load.
        run: "cargo test --lib -- sync::slot_rules sync::slot_files vault:: agent::"
```

This job runs on `push` to `main`, on pull requests and on `workflow_dispatch`; nothing here pushes. The run happens when the branch is pushed with the user's authorization.

- [ ] **Step 4: Write the manual checklist**

> After execution: the live list is `docs/superpowers/plans/2026-10-07-key-vault-agent-manual-verification.md`. The final fix round
> added items 8, 17 and 21 (32 items); the copy below is the list as Task 13 first wrote it.

Create `docs/superpowers/plans/2026-10-07-key-vault-agent-manual-verification.md`:

```markdown
# Key vault and agent, plan 1 — manual verification

Run on a Mac and on a Windows computer (the user's Windows has OpenSSH 9.5), with a beta build of this branch. Use a test
server or a throwaway host entry; never a production key you can't replace. Record each item as pass / fail with a note.

## Setup

1. Two computers in one sync account, a synced key slot used by a host (`web`) that both computers can reach.
2. In Keys → Keys used by synced hosts, the slot shows Ready on both.

## Moving a key into SSHelter

3. Click Only in SSHelter on one computer. The toast says the key is now only in SSHelter and, the first time, points at Launch at login and Keep running in menu bar when window closes.
4. The slot directory `~/.ssh/sshelter/keys/` keeps only `<file>.pub` for that slot; the private key file is gone.
5. `~/.ssh/config` starts with `Include ~/.ssh/sshelter/agent/config`; nothing else in the file changed (compare with the backup).
6. `~/.ssh/sshelter/agent/config` starts with `# Managed by SSHelter. Changes here are overwritten.` and lists `Host web` with
   `IdentityAgent` (`~/.ssh/sshelter/agent/sock` on the Mac, `//./pipe/sshelter-agent-<hex>` on Windows) and `IdentitiesOnly yes`.
7. The host list in SSHelter does not show `web` twice.

## Approvals from a terminal

8. In Terminal (Windows Terminal on Windows), `ssh web`: the approval window appears on top, also while SSHelter is hidden in the tray.
   Title "Allow Terminal to use <key>?" (Windows: "Allow WindowsTerminal to use <key>?"), the chain line (for example
   "Terminal → login → zsh → ssh"), `<user>@web` (the known_hosts name), the key fingerprint, "Remember for 4 hours" checked.
9. Allow: the session opens. `exit`, `ssh web` again: no window.
10. From another terminal app (iTerm2, VS Code's terminal): the window asks again (another program).
11. Deny: ssh does not log in with this key. No answer for 60 seconds: same, and the window closes.
12. Claude Code (or another AI tool) runs `ssh web`: the window names the tool's app first in the title and the chain; remembering
    it does not let Terminal skip the window.
13. `git fetch` in a repository whose remote host uses this key, with several remotes at once (`git fetch --all`): one window, and every
    fetch succeeds after one Allow.
14. `ssh -A web`, then on `web` run `ssh other-host-using-the-same-key`: refused without a window (forwarded request).
15. `ssh-keygen -Y sign -f ~/.ssh/sshelter/keys/<file>.pub -n test somefile`: the window says "an unknown host" and offers no
    Remember.

## Passphrases

16. Repeat 3–9 with a key that has a passphrase: the first window has a Passphrase field. A wrong passphrase shows
    "That passphrase didn't work." and asks again; three wrong ones refuse.
17. Without "Remember on this computer": a second program within 4 hours gets the window without the passphrase field.
18. With "Remember on this computer": after quitting and reopening SSHelter, the window has no passphrase field.

## Connect from SSHelter

19. Connect on `web` (main window and tray quick connect): the terminal runs
    `ssh -o IdentityAgent=… -o ForwardAgent=no web` and logs in without an approval window.
20. With a passphrase key that is not remembered: "Unlock <key> to connect to <user>@web" with Cancel and Unlock.
21. Connect on a host whose key is a normal file: unchanged (password auto-fill still works where it did).
22. Connect on a host you haven't connected to before (or remove its line from `known_hosts` first), leave ssh's fingerprint
    question open for more than a minute, then answer yes: ssh can't use the key ("communication with agent failed") and
    SSHelter shows "Connect to web again"; Connect again logs in.
23. With `ControlMaster auto` and `ControlPersist` set for `web`: a second Connect while the first session is open logs in through
    the master, and no "Connect again" message appears, then or 10 minutes later.

## When SSHelter isn't there, or things break

24. Quit SSHelter, `ssh web` from Terminal: ssh cannot use the key (it says so); reopen SSHelter and it works again.
25. Start a second SSHelter (`--mcp-host` while the app runs): the first keeps answering; no error in the second.
26. Remove the Include line from `~/.ssh/config` by hand: Keys shows "Hosts that use keys in SSHelter can't reach its agent." with
    Fix. Fix puts the line back first; a sync round does not add it back on its own before you press Fix.
27. Keep a file: the confirm says any program can use the file without asking; afterwards the private key is back in the slot, the
    host drops out of `agent/config`, and `ssh web` works without a window.
28. Lose the vault: quit SSHelter, rename `vault.json` in SSHelter's data folder (next to `sync-state.json`), start SSHelter. After a sync
    round the synced key is back in the vault (Only in SSHelter still works); a Connect on `web` before that round finishes opens
    no terminal and says the key isn't in SSHelter on this computer. Repeat with a key that is not synced: the slot asks for
    a key again.

## Windows only

29. The pipe `\\.\pipe\sshelter-agent-<hex>` exists only while SSHelter runs; `ssh web` from PowerShell gets the window
    ("WindowsTerminal → pwsh → ssh").
```

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/agent .github/workflows/test-windows.yml docs/superpowers/plans/2026-10-07-key-vault-agent-manual-verification.md
git commit -m "test(agent): exercise the agent with the real OpenSSH tools and on Windows CI"
```
