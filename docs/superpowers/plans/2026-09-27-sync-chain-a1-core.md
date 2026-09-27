# Sync Chain — Phase A1(Rust 核心:crypto / records / hosts file / state / relay client)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立 sync chain 的純 Rust 基礎:助記詞/金鑰派生/記錄加密、記錄模型與 LWW 合併、受管同步檔的區塊操作、本機同步狀態持久化、以及中繼 HTTP client —— 全部可單元測試、不含背景執行緒與 UI。

**Architecture:** 新增 `src-tauri/src/sync/` 模組樹,每個檔案一個責任;所有函式為純函式或只碰明確傳入的路徑,網路層以 `mockito` 假伺服器測試。後續 A2(relay Worker)與 A3(engine + UI)只依賴本計畫定義的介面。

**Tech Stack:** Rust 2021、`bip39 2`(rand)、`hkdf 0.12` + `sha2 0.10`、`hmac 0.12`、`chacha20poly1305 0.10`、`reqwest 0.13`(blocking/json;與 lockfile 既有的 `reqwest 0.13.4` 共用同一份,預設 rustls)、`mockito 1`(dev)。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md`(§3 資料模型、§4 密碼學、§5 中繼 API、§6 狀態檔)

## Global Constraints

- Rust 端註解沿用既有 `deploy.rs`/`mcp.rs` 慣例(繁中);識別字、commit、UI 文案一律英文;Conventional Commits。
- 不新增 unix-only API 而不加 `#[cfg(unix)]`;CI 跑 macOS/Linux/Windows 三平台,`cargo test` 必須全綠。
- 私鑰、助記詞、密碼**永不**進 log、error message、toast。
- 本機落地權限:狀態檔 0600(`fsutil::atomic_write(path, bytes, 0o600)`)、目錄 0700(`fsutil::ensure_dir_secure`)。
- 每個 task 結束都要 `cd src-tauri && cargo test` 綠燈後才 commit;commit 只含該 task 的檔案。
- 加密參數固定(spec §4 的 byte-level 定義):XChaCha20-Poly1305、24-byte 隨機 nonce、AAD = `chain_id + "\n" + kind + "\n" + id_hash`(綁 `id_hash`,**不是**明文 id);`id_hash = hex(HMAC-SHA256(enc_key, kind + "\n" + id))`;HKDF-SHA256 salt = `b"sshelter-sync-v1"`,info = `sshelter/v1/chain-id` / `sshelter/v1/auth` / `sshelter/v1/enc`。已知答案向量釘在 Task 1 測試,不得改動。
- relay URL 只接受 `https://`;`http://` 只允許 loopback host(`127.0.0.1`、`localhost`、`[::1]`);HTTP client 不跟隨 redirect(bearer token 有整條 chain 的權限)。
- `sync-state.json` 只保存 host/device/meta 的明文記錄;其他種類(key/password/未知)只以原始密文 envelope 存在 `sealed`,絕不解密進狀態檔。
- 受管檔中含 wildcard 字元(`*`、`?`、`!`)的 Host 區塊是裝置本地結構:不擷取、不套用、不刪除。

## Review Focus

1. 助記詞輸入含大寫、多餘空白、換行、全形空白 —— 應正規化後接受;錯字/字數錯要回可讀錯誤(Task 1 `normalize_mnemonic` 測試)。
2. 受管同步檔被手改成含頂部註解、空行、甚至 `Host *` —— `blocks_of` 只取具名 Host 區塊(wildcard 略過),`apply_host_text`/`remove_host_block` 不得動到其他項目或 wildcard 區塊(Task 3 測試)。
3. 中繼不可達(連線拒絕)—— client 必須在 connect timeout 內回 `Err`,不可掛住(Task 5 測試)。
4. 中繼回傳非預期 JSON / 5xx —— 必須映射成 `AppError::Other` 而非 panic(Task 5 測試)。
5. 狀態檔損毀或版本較新 —— `load` 回可辨識錯誤,呼叫端可視為「未加入」(Task 4 測試)。
6. 使用者把 relay URL 填成 `http://sync.example.com` —— 必須拒絕(bearer token 會明文外洩);`http://127.0.0.1:8787` 仍可用(Task 5 測試)。
7. 主 config 已有別的 `Include` 在前 —— 同步 Include 仍要插在它們之前,否則 spec §10 的遮蔽承諾不成立(Task 3 測試)。

---

### Task 1: 相依與 `sync::crypto`(助記詞、派生、記錄加密)

**Files:**
- Modify: `src-tauri/Cargo.toml`(`[dependencies]` 與 `[dev-dependencies]`)
- Create: `src-tauri/src/sync/mod.rs`
- Create: `src-tauri/src/sync/crypto.rs`
- Modify: `src-tauri/src/lib.rs`(宣告 `mod sync;`)

**Interfaces:**
- Produces:
  - `pub struct ChainKeys { pub chain_id: String, pub auth_token: String, enc_key: [u8; 32] }`
  - `pub fn generate_mnemonic() -> Result<String, AppError>`(24 個英文字,空白分隔)
  - `pub fn normalize_mnemonic(input: &str) -> Result<String, AppError>`
  - `pub fn derive_keys(mnemonic: &str) -> Result<ChainKeys, AppError>`
  - `pub fn id_hash(keys: &ChainKeys, kind: &str, id: &str) -> String`(64 hex)
  - `pub struct Sealed { pub id_hash: String, pub nonce: String, pub ciphertext: String }`(nonce/ciphertext 皆 base64)
  - `pub fn seal(keys: &ChainKeys, kind: &str, id: &str, plaintext: &[u8]) -> Result<Sealed, AppError>`(內部算 `id_hash`,AAD 綁 `chain_id + kind + id_hash`)
  - `pub fn open(keys: &ChainKeys, kind: &str, id_hash: &str, sealed: &Sealed) -> Result<Vec<u8>, AppError>`(接收端只有 `id_hash`,沒有明文 id —— 解密後再由呼叫端驗證明文 id 的 hash 相符)

- [ ] **Step 1: 加相依**

在 `src-tauri/Cargo.toml` 的 `[dependencies]` 末尾加入:

```toml
bip39 = { version = "2", features = ["rand"] }
hkdf = "0.12"
hmac = "0.12"
chacha20poly1305 = "0.10"
reqwest = { version = "0.13", features = ["blocking", "json"] }
```

(`reqwest 0.13` 的預設 features 已含 rustls(`default-tls`)、`http2`、`system-proxy`;lockfile 裡已經有 tauri 相依帶進來的 `reqwest 0.13.4`,指定 `0.13` 會合併成同一份而不是再編一份 0.12。)

在 `[dev-dependencies]` 加入:

```toml
mockito = "1"
```

Run: `cd src-tauri && cargo fetch && cargo tree -i reqwest --depth 0`
Expected: 解析成功,無版本衝突(`sha2 0.10` 與 `hkdf 0.12`/`hmac 0.12` 相容);`cargo tree` 只列出一個 `reqwest v0.13.x`。

- [ ] **Step 2: 建立模組骨架**

建立 `src-tauri/src/sync/mod.rs`:

```rust
//! Sync chain:Brave 式免帳號端對端同步。各子模組單一責任、皆可單元測試:
//! - `crypto`:助記詞、金鑰派生、記錄加密
//! - `record`:記錄模型與 LWW 合併(Task 2)
//! - `hosts_file`:受管同步檔的區塊操作(Task 3)
//! - `state`:本機同步狀態持久化(Task 4)
//! - `relay`:中繼 HTTP client(Task 5)

pub mod crypto;
```

在 `src-tauri/src/lib.rs` 的其他 `mod` 宣告旁加入 `mod sync;`(檔案頂部 `mod askpass;` 之後即可)。

- [ ] **Step 3: 寫失敗的測試**

建立 `src-tauri/src/sync/crypto.rs`,先只放測試模組與 `use`:

```rust
//! 助記詞與加密。安全模型見 spec §2/§4:助記詞是唯一祕密,所有派生值可重算。

use base64::engine::general_purpose::STANDARD as B64;
use base64::Engine as _;
use bip39::{Language, Mnemonic};
use chacha20poly1305::aead::{Aead, KeyInit, Payload};
use chacha20poly1305::{Key, XChaCha20Poly1305, XNonce};
use hkdf::Hkdf;
use hmac::{Hmac, Mac};
use sha2::Sha256;

use crate::error::AppError;

#[cfg(test)]
mod tests {
    use super::*;

    const WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art";

    #[test]
    fn generated_mnemonic_has_24_valid_words() {
        let words = generate_mnemonic().unwrap();
        assert_eq!(words.split_whitespace().count(), 24);
        assert_eq!(normalize_mnemonic(&words).unwrap(), words);
    }

    #[test]
    fn normalize_accepts_messy_input_and_rejects_bad_words() {
        let messy = format!("  {}\n", WORDS.to_uppercase().replace(' ', "   "));
        assert_eq!(normalize_mnemonic(&messy).unwrap(), WORDS);
        assert!(normalize_mnemonic("abandon abandon").is_err());
        let bad = WORDS.replacen("art", "zzzz", 1);
        assert!(normalize_mnemonic(&bad).is_err());
        let wrong_checksum = WORDS.replacen("art", "abandon", 1);
        assert!(normalize_mnemonic(&wrong_checksum).is_err());
    }

    #[test]
    fn derivation_is_deterministic_and_domain_separated() {
        let a = derive_keys(WORDS).unwrap();
        let b = derive_keys(WORDS).unwrap();
        assert_eq!(a.chain_id, b.chain_id);
        assert_eq!(a.auth_token, b.auth_token);
        assert_eq!(a.chain_id.len(), 64);
        assert_eq!(a.auth_token.len(), 64);
        assert_ne!(a.chain_id, a.auth_token);
        let other = generate_mnemonic().unwrap();
        assert_ne!(derive_keys(&other).unwrap().chain_id, a.chain_id);
    }

    #[test]
    fn derivation_matches_pinned_vectors() {
        // spec §4 的已知答案向量(PBKDF2-HMAC-SHA512 → HKDF-SHA256 → hex)。
        // 改了 salt、info、分隔符或編碼,這裡就會炸 —— 這是互通性的護欄,不得更新數值遷就實作。
        let keys = derive_keys(WORDS).unwrap();
        assert_eq!(keys.chain_id, "4eb6631d45882eb3a0c2541383de4263fc056a1bc851718d66d2f664ae77bf4c");
        assert_eq!(keys.auth_token, "8bef1a101b57ae1888a071b3e559b4253eb06572ff4fd021b5ef1ab97747e543");
        assert_eq!(
            id_hash(&keys, "host", "web-1"),
            "41fc9eb6d727b31a34350586bcee7619a7983fd8d695e7bebde44e51e937113a"
        );
    }

    #[test]
    fn id_hash_hides_the_id_but_is_stable() {
        let keys = derive_keys(WORDS).unwrap();
        let h1 = id_hash(&keys, "host", "web-1");
        assert_eq!(h1, id_hash(&keys, "host", "web-1"));
        assert_ne!(h1, id_hash(&keys, "key", "web-1"));
        assert_eq!(h1.len(), 64);
        assert!(!h1.contains("web"));
    }

    #[test]
    fn seal_open_round_trip_and_aad_binding() {
        let keys = derive_keys(WORDS).unwrap();
        let sealed = seal(&keys, "host", "web-1", b"Host web-1\n  HostName 10.0.0.9\n").unwrap();
        assert_eq!(sealed.id_hash, id_hash(&keys, "host", "web-1"));
        assert_eq!(
            open(&keys, "host", &sealed.id_hash, &sealed).unwrap(),
            b"Host web-1\n  HostName 10.0.0.9\n"
        );
        // 同一密文換 id_hash / kind 都要失敗(AAD 綁定)。
        assert!(open(&keys, "host", &id_hash(&keys, "host", "db-1"), &sealed).is_err());
        assert!(open(&keys, "key", &sealed.id_hash, &sealed).is_err());
        // 別的 chain 開不了。
        let other = derive_keys(&generate_mnemonic().unwrap()).unwrap();
        assert!(open(&other, "host", &sealed.id_hash, &sealed).is_err());
        // 每次 nonce 不同。
        let again = seal(&keys, "host", "web-1", b"x").unwrap();
        assert_ne!(again.nonce, sealed.nonce);
    }

    #[test]
    fn open_rejects_corrupt_base64() {
        let keys = derive_keys(WORDS).unwrap();
        let sealed = Sealed { id_hash: "00".repeat(32), nonce: "!!".to_string(), ciphertext: "!!".to_string() };
        assert!(open(&keys, "host", &sealed.id_hash, &sealed).is_err());
    }
}
```

- [ ] **Step 4: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::crypto 2>&1 | tail -5`
Expected: 編譯錯誤 —— `cannot find function generate_mnemonic`(紅燈)。

- [ ] **Step 5: 實作**

在 `crypto.rs` 的 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
const HKDF_SALT: &[u8] = b"sshelter-sync-v1";
const NONCE_LEN: usize = 24;

/// 從助記詞派生出的一組 chain 金鑰。`enc_key` 不公開:只能透過 `seal`/`open` 使用。
#[derive(Clone)]
pub struct ChainKeys {
    /// 可公開的 chain 識別(hex)。
    pub chain_id: String,
    /// 中繼的 bearer token(hex);中繼只存它的 SHA-256。
    pub auth_token: String,
    enc_key: [u8; 32],
}

impl std::fmt::Debug for ChainKeys {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        // 絕不印出 enc_key 或 token。
        f.debug_struct("ChainKeys").field("chain_id", &self.chain_id).finish_non_exhaustive()
    }
}

/// 24 個英文字的 BIP39 助記詞(256-bit entropy)。
pub fn generate_mnemonic() -> Result<String, AppError> {
    let m = Mnemonic::generate_in(Language::English, 24)
        .map_err(|e| AppError::Other(format!("cannot generate recovery words: {e}")))?;
    Ok(m.to_string())
}

/// 正規化使用者輸入:小寫、單一空白;字數與 BIP39 checksum 不對就拒絕。
pub fn normalize_mnemonic(input: &str) -> Result<String, AppError> {
    let words: Vec<String> = input.split_whitespace().map(|w| w.to_lowercase()).collect();
    if words.len() != 24 {
        return Err(AppError::Other(format!(
            "recovery phrase must be 24 words (got {})",
            words.len()
        )));
    }
    let joined = words.join(" ");
    Mnemonic::parse_in(Language::English, &joined)
        .map_err(|e| AppError::Other(format!("invalid recovery phrase: {e}")))?;
    Ok(joined)
}

fn hkdf_expand(hk: &Hkdf<Sha256>, info: &[u8]) -> [u8; 32] {
    let mut out = [0u8; 32];
    hk.expand(info, &mut out).expect("32 bytes is a valid HKDF-SHA256 output length");
    out
}

fn hex(bytes: &[u8]) -> String {
    bytes.iter().map(|b| format!("{b:02x}")).collect()
}

/// 助記詞 → seed(BIP39,空 passphrase)→ HKDF 三個用途分離的 32-byte 值。
pub fn derive_keys(mnemonic: &str) -> Result<ChainKeys, AppError> {
    let normalized = normalize_mnemonic(mnemonic)?;
    let m = Mnemonic::parse_in(Language::English, &normalized)
        .map_err(|e| AppError::Other(format!("invalid recovery phrase: {e}")))?;
    let seed = m.to_seed("");
    let hk = Hkdf::<Sha256>::new(Some(HKDF_SALT), &seed);
    Ok(ChainKeys {
        chain_id: hex(&hkdf_expand(&hk, b"sshelter/v1/chain-id")),
        auth_token: hex(&hkdf_expand(&hk, b"sshelter/v1/auth")),
        enc_key: hkdf_expand(&hk, b"sshelter/v1/enc"),
    })
}

/// 中繼看到的記錄識別:HMAC-SHA256(enc_key, kind || "\n" || id),連 alias 都不外洩。
pub fn id_hash(keys: &ChainKeys, kind: &str, id: &str) -> String {
    // `chacha20poly1305::aead::KeyInit` 與 `hmac::Mac` 都提供 `new_from_slice`;
    // 不指名 trait 會是 E0034(multiple applicable items in scope)。
    let mut mac = <Hmac<Sha256> as Mac>::new_from_slice(&keys.enc_key)
        .expect("HMAC accepts any key length");
    mac.update(kind.as_bytes());
    mac.update(b"\n");
    mac.update(id.as_bytes());
    hex(&mac.finalize().into_bytes())
}

/// 加密後的記錄本體。`id_hash` 是中繼看到的識別;nonce/ciphertext 為 base64。
#[derive(Clone, Debug, PartialEq, serde::Serialize, serde::Deserialize)]
pub struct Sealed {
    pub id_hash: String,
    pub nonce: String,
    pub ciphertext: String,
}

/// AAD 綁 chain 與 id_hash(不是明文 id):接收端只有 id_hash 也能驗證。
fn aad(keys: &ChainKeys, kind: &str, id_hash: &str) -> Vec<u8> {
    format!("{}\n{}\n{}", keys.chain_id, kind, id_hash).into_bytes()
}

pub fn seal(keys: &ChainKeys, kind: &str, id: &str, plaintext: &[u8]) -> Result<Sealed, AppError> {
    let mut nonce = [0u8; NONCE_LEN];
    getrandom::fill(&mut nonce).map_err(|e| AppError::Other(format!("cannot draw nonce: {e}")))?;
    let hash = id_hash(keys, kind, id);
    let cipher = XChaCha20Poly1305::new(Key::from_slice(&keys.enc_key));
    let aad = aad(keys, kind, &hash);
    let ciphertext = cipher
        .encrypt(XNonce::from_slice(&nonce), Payload { msg: plaintext, aad: &aad })
        .map_err(|_| AppError::Other("encryption failed".to_string()))?;
    Ok(Sealed {
        id_hash: hash,
        nonce: B64.encode(nonce),
        ciphertext: B64.encode(ciphertext),
    })
}

pub fn open(keys: &ChainKeys, kind: &str, id_hash: &str, sealed: &Sealed) -> Result<Vec<u8>, AppError> {
    let nonce = B64
        .decode(&sealed.nonce)
        .map_err(|_| AppError::Other("record nonce is not valid base64".to_string()))?;
    if nonce.len() != NONCE_LEN {
        return Err(AppError::Other("record nonce has the wrong length".to_string()));
    }
    let ciphertext = B64
        .decode(&sealed.ciphertext)
        .map_err(|_| AppError::Other("record ciphertext is not valid base64".to_string()))?;
    let cipher = XChaCha20Poly1305::new(Key::from_slice(&keys.enc_key));
    let aad = aad(keys, kind, id_hash);
    cipher
        .decrypt(XNonce::from_slice(&nonce), Payload { msg: &ciphertext, aad: &aad })
        .map_err(|_| AppError::Other("record cannot be decrypted with this chain".to_string()))
}
```

- [ ] **Step 6: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::crypto 2>&1 | tail -5`
Expected: `7 passed`。`derivation_matches_pinned_vectors` 若失敗,錯的是實作(對照 spec §4 逐項檢查 salt/info/`\n` 分隔/hex 小寫),不是向量。

- [ ] **Step 7: Commit**

```bash
git add src-tauri/Cargo.toml src-tauri/Cargo.lock src-tauri/src/lib.rs src-tauri/src/sync/mod.rs src-tauri/src/sync/crypto.rs
git commit -m "feat(sync): mnemonic, key derivation and record encryption"
```

---

### Task 2: `sync::record` —— 記錄模型與 LWW 合併

**Files:**
- Create: `src-tauri/src/sync/record.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod record;`)

**Interfaces:**
- Produces:
  - `pub enum RecordKind { Host, Key, Password, Device, Meta }`(serde 小寫字串;`as_str()`)
  - `pub struct Record { pub kind: RecordKind, pub id: String, pub version: u64, pub updated_at_ms: u64, pub device_id: String, pub deleted: bool, pub payload: serde_json::Value }`
  - `pub struct LocalRecord { pub record: Record, pub seq: u64, pub dirty: bool }`
  - `pub fn record_key(kind: RecordKind, id: &str) -> String`(`"host:web-1"`)
  - `pub enum MergeOutcome { KeepLocal, TakeRemote, RemoteWinsOverDirtyLocal }`
  - `pub fn merge(local: Option<&LocalRecord>, remote: &Record) -> MergeOutcome`
  - `pub struct HostPayload { pub schema: u32, pub text: String }`
  - `pub struct DevicePayload { pub schema: u32, pub name: String, pub platform: String, pub joined_at_ms: u64, pub last_seen_ms: u64, pub keys: Vec<String> }`
  - `pub struct MetaPayload { pub schema_version: u32, pub created_by_app_version: String }`
  - `pub const SCHEMA_VERSION: u32 = 1;`
  - `pub struct Envelope { pub id_hash: String, pub kind: String, pub seq: u64, pub nonce: String, pub ciphertext: String, pub deleted: bool }`(serde `rename_all = "camelCase"`:wire 欄位 `idHash`…;Task 4 的 `SyncState.sealed` 與 Task 5 的 client 共用)

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/record.rs`:

```rust
//! 記錄模型與合併規則(spec §3.2、§6)。純資料,不碰 I/O。

use serde::{Deserialize, Serialize};
use serde_json::Value;

#[cfg(test)]
mod tests {
    use super::*;

    fn rec(id: &str, updated: u64, device: &str, deleted: bool) -> Record {
        Record {
            kind: RecordKind::Host,
            id: id.to_string(),
            version: 1,
            updated_at_ms: updated,
            device_id: device.to_string(),
            deleted,
            payload: serde_json::json!({ "schema": 1, "text": "Host x\n" }),
        }
    }

    fn local(record: Record, dirty: bool) -> LocalRecord {
        LocalRecord { record, seq: 3, dirty }
    }

    #[test]
    fn kind_round_trips_as_lowercase_strings() {
        assert_eq!(serde_json::to_string(&RecordKind::Password).unwrap(), "\"password\"");
        assert_eq!(RecordKind::Meta.as_str(), "meta");
        assert_eq!(record_key(RecordKind::Host, "web-1"), "host:web-1");
    }

    #[test]
    fn remote_is_taken_when_nothing_is_local() {
        assert_eq!(merge(None, &rec("a", 10, "dev-b", false)), MergeOutcome::TakeRemote);
    }

    #[test]
    fn newer_remote_wins_and_flags_dirty_local_loss() {
        let mine = local(rec("a", 10, "dev-a", false), false);
        assert_eq!(merge(Some(&mine), &rec("a", 11, "dev-b", false)), MergeOutcome::TakeRemote);
        let dirty = local(rec("a", 10, "dev-a", false), true);
        assert_eq!(
            merge(Some(&dirty), &rec("a", 11, "dev-b", false)),
            MergeOutcome::RemoteWinsOverDirtyLocal
        );
    }

    #[test]
    fn older_remote_never_overwrites() {
        let mine = local(rec("a", 10, "dev-a", false), true);
        assert_eq!(merge(Some(&mine), &rec("a", 9, "dev-b", false)), MergeOutcome::KeepLocal);
    }

    #[test]
    fn equal_timestamps_break_ties_by_device_id_then_tombstone() {
        let mine = local(rec("a", 10, "dev-b", false), false);
        // 字典序小的裝置贏。
        assert_eq!(merge(Some(&mine), &rec("a", 10, "dev-a", false)), MergeOutcome::TakeRemote);
        assert_eq!(merge(Some(&mine), &rec("a", 10, "dev-c", false)), MergeOutcome::KeepLocal);
        // 同時間 tombstone 優先於修改,不論裝置。
        assert_eq!(merge(Some(&mine), &rec("a", 10, "dev-z", true)), MergeOutcome::TakeRemote);
        let mine_deleted = local(rec("a", 10, "dev-z", true), false);
        assert_eq!(merge(Some(&mine_deleted), &rec("a", 10, "dev-a", false)), MergeOutcome::KeepLocal);
    }

    #[test]
    fn payloads_serialize_with_schema() {
        let p = HostPayload { schema: SCHEMA_VERSION, text: "Host a\n".into() };
        let v = serde_json::to_value(&p).unwrap();
        assert_eq!(v["schema"], 1);
        let back: HostPayload = serde_json::from_value(v).unwrap();
        assert_eq!(back.text, "Host a\n");
    }

    #[test]
    fn envelope_uses_camel_case_wire_names() {
        let env = Envelope {
            id_hash: "h".into(),
            kind: "host".into(),
            seq: 3,
            nonce: "n".into(),
            ciphertext: "c".into(),
            deleted: false,
        };
        let json = serde_json::to_string(&env).unwrap();
        assert!(json.contains("\"idHash\":\"h\""), "got {json}");
        assert!(!json.contains("id_hash"));
        assert_eq!(serde_json::from_str::<Envelope>(&json).unwrap(), env);
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::record 2>&1 | tail -5`
Expected: 編譯錯誤(`Record` 未定義)。

- [ ] **Step 3: 實作**

在 `record.rs` 的 `use` 之後、tests 之前加入:

```rust
pub const SCHEMA_VERSION: u32 = 1;

#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum RecordKind {
    Host,
    Key,
    Password,
    Device,
    Meta,
}

impl RecordKind {
    pub fn as_str(self) -> &'static str {
        match self {
            RecordKind::Host => "host",
            RecordKind::Key => "key",
            RecordKind::Password => "password",
            RecordKind::Device => "device",
            RecordKind::Meta => "meta",
        }
    }

    pub fn parse(s: &str) -> Option<Self> {
        match s {
            "host" => Some(RecordKind::Host),
            "key" => Some(RecordKind::Key),
            "password" => Some(RecordKind::Password),
            "device" => Some(RecordKind::Device),
            "meta" => Some(RecordKind::Meta),
            _ => None,
        }
    }
}

/// 一筆解密後的記錄。`payload` 依 kind 對應 `HostPayload` 等結構。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct Record {
    pub kind: RecordKind,
    pub id: String,
    pub version: u64,
    pub updated_at_ms: u64,
    pub device_id: String,
    pub deleted: bool,
    pub payload: Value,
}

/// 本機快取的記錄:附上最後看到的中繼序號與是否尚未上傳。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct LocalRecord {
    pub record: Record,
    pub seq: u64,
    pub dirty: bool,
}

pub fn record_key(kind: RecordKind, id: &str) -> String {
    format!("{}:{}", kind.as_str(), id)
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum MergeOutcome {
    KeepLocal,
    TakeRemote,
    /// 遠端較新且本機有未上傳修改 —— 呼叫端應通知使用者。
    RemoteWinsOverDirtyLocal,
}

/// 記錄層級 LWW:時間戳大者勝;同時間 tombstone 勝,再比 device_id 字典序小者勝。
pub fn merge(local: Option<&LocalRecord>, remote: &Record) -> MergeOutcome {
    let Some(local) = local else {
        return MergeOutcome::TakeRemote;
    };
    let mine = &local.record;
    let remote_wins = match remote.updated_at_ms.cmp(&mine.updated_at_ms) {
        std::cmp::Ordering::Greater => true,
        std::cmp::Ordering::Less => false,
        std::cmp::Ordering::Equal => match (remote.deleted, mine.deleted) {
            (true, false) => true,
            (false, true) => false,
            _ => remote.device_id < mine.device_id,
        },
    };
    if !remote_wins {
        MergeOutcome::KeepLocal
    } else if local.dirty {
        MergeOutcome::RemoteWinsOverDirtyLocal
    } else {
        MergeOutcome::TakeRemote
    }
}

/// 中繼往返的密文信封(wire 格式 camelCase,對應 relay Worker 與 Task 5 的 client)。
/// 也是 `SyncState.sealed` 保留「本版不處理的種類」時的原樣儲存格式。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct Envelope {
    pub id_hash: String,
    pub kind: String,
    pub seq: u64,
    pub nonce: String,
    pub ciphertext: String,
    pub deleted: bool,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct HostPayload {
    pub schema: u32,
    /// 整個 Host 區塊的原始文字(lossless)。
    pub text: String,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct DevicePayload {
    pub schema: u32,
    pub name: String,
    pub platform: String,
    pub joined_at_ms: u64,
    pub last_seen_ms: u64,
    /// 這台裝置持有的同步金鑰 id(Phase B 才會填)。
    #[serde(default)]
    pub keys: Vec<String>,
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct MetaPayload {
    pub schema_version: u32,
    pub created_by_app_version: String,
}
```

並在 `sync/mod.rs` 加 `pub mod record;`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::record 2>&1 | tail -5`
Expected: `7 passed`。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/record.rs
git commit -m "feat(sync): record model, wire envelope and last-writer-wins merge"
```

---

### Task 3: `sync::hosts_file` —— 受管同步檔的區塊操作

**Files:**
- Create: `src-tauri/src/sync/hosts_file.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod hosts_file;`)

**Interfaces:**
- Consumes: `crate::config::model::{Item, HostBlock}`(`Item` 變體:`Blank(String)`、`Comment(String)`、`Directive(Directive)`、`Host(HostBlock)`、`Match(MatchBlock)`)、`crate::config::parser::parse_file(&str) -> (Vec<Item>, bool)`、`crate::config::serialize::serialize_items(&[Item], bool) -> String`、`crate::config::model::Directive::new(keyword, value, indent)`、`crate::fsutil::{ensure_dir_secure, atomic_write}`
- Produces:
  - `pub const INCLUDE_VALUE: &str = "~/.ssh/sshelter/hosts.config";`
  - `pub fn managed_path(ssh_dir: &Path) -> PathBuf`(`<ssh_dir>/sshelter/hosts.config`)
  - `pub fn ensure_managed_file(ssh_dir: &Path) -> Result<PathBuf, AppError>`
  - `pub fn ensure_include(items: &mut Vec<Item>) -> bool`(改了回 true;位置 = 前導註解/空行之後、其他任何項目之前 —— **不是** `newfile::include_insert_index`;已存在但不在最頂端就搬上去,多路徑 Include 只抽走我們的 token)
  - `pub fn is_syncable_alias(alias: &str) -> bool`(非空且不含 `*`、`?`、`!`)
  - `pub fn is_syncable_block(patterns: &[String]) -> bool`(非空且**所有** pattern 都 `is_syncable_alias`;`Host web *.internal` 整個區塊不同步)
  - `pub struct HostBlockText { pub alias: String, pub text: String }`
  - `pub fn blocks_of(items: &[Item]) -> Vec<HostBlockText>`(只取 `is_syncable_block` 的 Host 區塊)
  - `pub fn validate_host_text(alias: &str, text: &str) -> Result<(), AppError>`(恰好一個 Host 區塊、第一個 pattern 等於 alias、所有 pattern 皆 syncable;A3 在合併前用它把壞記錄擋在快取外)
  - `pub fn apply_host_text(items: &mut Vec<Item>, alias: &str, text: &str) -> Result<bool, AppError>`(內容改變才回 true;wildcard alias、本地同名區塊含 wildcard、文字不合法都回 Err)
  - `pub fn remove_host_block(items: &mut Vec<Item>, alias: &str) -> bool`(wildcard alias 或本地區塊含 wildcard 一律 false)

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/hosts_file.rs`:

```rust
//! 受管同步檔 `~/.ssh/sshelter/hosts.config`:同步範圍就是這個檔案裡的 Host 區塊。
//! 區塊以原始文字為單位進出(lossless),其他項目(註解、空行、wildcard)原封不動。

use std::path::{Path, PathBuf};

use crate::config::model::{Directive, Item};
use crate::config::parser::parse_file;
use crate::config::serialize::serialize_items;
use crate::error::AppError;
use crate::fsutil;

#[cfg(test)]
mod tests {
    use super::*;

    const FILE: &str = "# synced by sshelter\n\nHost web-1\n  HostName 10.0.0.9\n  #tags: prod, web\n\nHost db-1\n  HostName 10.0.0.10\n";
    const WITH_WILDCARD: &str = "Host *\n  ServerAliveInterval 30\n\nHost web-1\n  HostName 10.0.0.9\n\nHost *.internal !bad.internal\n  User ops\n";

    #[test]
    fn managed_path_lives_under_ssh_dir() {
        let p = managed_path(Path::new("/home/f/.ssh"));
        assert_eq!(p, Path::new("/home/f/.ssh").join("sshelter").join("hosts.config"));
    }

    #[test]
    fn ensure_managed_file_creates_dir_and_empty_file_once() {
        let dir = tempfile::tempdir().unwrap();
        let p = ensure_managed_file(dir.path()).unwrap();
        assert!(p.is_file());
        assert_eq!(std::fs::read_to_string(&p).unwrap(), "");
        std::fs::write(&p, "Host keep\n").unwrap();
        ensure_managed_file(dir.path()).unwrap();
        assert_eq!(std::fs::read_to_string(&p).unwrap(), "Host keep\n");
    }

    #[test]
    fn ensure_include_goes_to_the_very_top_and_is_idempotent() {
        // 前導註解/空行之後、既有 Include 與全域指令之前:同步檔必須是 ssh 第一個讀到的定義
        // (first-obtained-wins),spec §10 的「同步主機遮蔽本地同名主機」才成立。
        let (mut items, _) = parse_file("# main\n\nInclude ~/.ssh/other.config\nAddKeysToAgent yes\nHost a\n  HostName 1\n");
        assert!(ensure_include(&mut items));
        assert!(!ensure_include(&mut items));
        let text = serialize_items(&items, true);
        assert_eq!(
            text,
            format!("# main\n\nInclude {INCLUDE_VALUE}\nInclude ~/.ssh/other.config\nAddKeysToAgent yes\nHost a\n  HostName 1\n")
        );
        // 空檔:就是第一行。
        let (mut empty, _) = parse_file("");
        assert!(ensure_include(&mut empty));
        assert_eq!(serialize_items(&empty, true), format!("Include {INCLUDE_VALUE}\n"));
    }

    #[test]
    fn ensure_include_moves_an_existing_include_to_the_top() {
        // 舊版插法(最後一個 Include 之後)或使用者搬動過:搬到最頂端,其他行原封不動。
        let (mut items, _) = parse_file("Include ~/.ssh/other.config\nInclude ~/.ssh/sshelter/hosts.config\nHost a\n");
        assert!(ensure_include(&mut items));
        assert_eq!(serialize_items(&items, true), format!("Include {INCLUDE_VALUE}\nInclude ~/.ssh/other.config\nHost a\n"));
        assert!(!ensure_include(&mut items));
        // 多路徑 Include:只抽走我們的 token,其他路徑留在原地。
        let (mut items, _) = parse_file("# c\nAddKeysToAgent yes\nInclude ~/.ssh/a.config ~/.ssh/sshelter/hosts.config\nHost a\n");
        assert!(ensure_include(&mut items));
        assert_eq!(
            serialize_items(&items, true),
            format!("# c\nInclude {INCLUDE_VALUE}\nAddKeysToAgent yes\nInclude ~/.ssh/a.config\nHost a\n")
        );
        assert!(!ensure_include(&mut items));
        // 已在首行、但排在別的路徑後面(`Include a ours`):ssh 會先讀 a,所以仍要正規化成獨立一行。
        let (mut items, _) = parse_file("Include ~/.ssh/a.config ~/.ssh/sshelter/hosts.config\nHost a\n");
        assert!(ensure_include(&mut items));
        assert_eq!(serialize_items(&items, true), format!("Include {INCLUDE_VALUE}\nInclude ~/.ssh/a.config\nHost a\n"));
        assert!(!ensure_include(&mut items));
    }

    #[test]
    fn wildcard_blocks_are_neither_extracted_nor_touched() {
        let (mut items, _) = parse_file(WITH_WILDCARD);
        // 先綁定 blocks_of 的結果再借用,否則暫時 Vec 在敘述結束就釋放(E0716)。
        let blocks = blocks_of(&items);
        let aliases: Vec<&str> = blocks.iter().map(|b| b.alias.as_str()).collect();
        assert_eq!(aliases, vec!["web-1"]);
        // 遠端送來 wildcard alias 的記錄:拒絕、不套用。
        assert!(apply_host_text(&mut items, "*", "Host *\n  User root\n").is_err());
        assert!(apply_host_text(&mut items, "*.internal", "Host *.internal\n").is_err());
        // tombstone 也刪不到 wildcard 區塊。
        assert!(!remove_host_block(&mut items, "*"));
        assert!(!remove_host_block(&mut items, "*.internal"));
        assert_eq!(serialize_items(&items, true), WITH_WILDCARD);
        assert!(!is_syncable_alias("*"));
        assert!(!is_syncable_alias("web?"));
        assert!(!is_syncable_alias("!web"));
        assert!(!is_syncable_alias(""));
        assert!(is_syncable_alias("web-1"));
        // 混合 pattern(`Host web *.internal`、`Host web !prod`):第一個 pattern 具名也不算,整個區塊本地。
        assert!(!is_syncable_block(&["web".to_string(), "*.internal".to_string()]));
        assert!(!is_syncable_block(&["web".to_string(), "!prod".to_string()]));
        assert!(!is_syncable_block(&[]));
        assert!(is_syncable_block(&["web".to_string(), "web.example.com".to_string()]));
        const MIXED: &str = "Host web *.internal\n  User ops\n\nHost db\n  HostName 1\n";
        let (mut items, _) = parse_file(MIXED);
        let blocks = blocks_of(&items);
        assert_eq!(blocks.iter().map(|b| b.alias.as_str()).collect::<Vec<_>>(), vec!["db"]);
        // 遠端的 `web` 記錄不能替換、也不能在旁邊附加第二個 `Host web`;tombstone 也刪不到它。
        assert!(apply_host_text(&mut items, "web", "Host web\n  User root\n").is_err());
        assert!(!remove_host_block(&mut items, "web"));
        assert_eq!(serialize_items(&items, true), MIXED);
        // 遠端記錄的文字本身含 wildcard pattern:拒絕。
        assert!(validate_host_text("web", "Host web *.internal\n").is_err());
        assert!(validate_host_text("web", "Host web\n  User root\n").is_ok());
    }

    #[test]
    fn blocks_of_extracts_every_host_block_verbatim_and_skips_the_rest() {
        let (items, _) = parse_file(FILE);
        let blocks = blocks_of(&items);
        assert_eq!(blocks.len(), 2);
        assert_eq!(blocks[0].alias, "web-1");
        assert!(blocks[0].text.contains("#tags: prod, web"));
        assert_eq!(blocks[1].alias, "db-1");
        assert_eq!(blocks[1].text, "Host db-1\n  HostName 10.0.0.10\n");
    }

    #[test]
    fn apply_replaces_one_block_and_leaves_neighbors_byte_identical() {
        let (mut items, _) = parse_file(FILE);
        let changed = apply_host_text(&mut items, "web-1", "Host web-1\n  HostName 10.0.0.99\n\n").unwrap();
        assert!(changed);
        let text = serialize_items(&items, true);
        assert!(text.starts_with("# synced by sshelter\n\nHost web-1\n  HostName 10.0.0.99\n"));
        assert!(text.ends_with("Host db-1\n  HostName 10.0.0.10\n"));
        // 相同內容再套一次 → 無變更。
        assert!(!apply_host_text(&mut items, "web-1", "Host web-1\n  HostName 10.0.0.99\n\n").unwrap());
    }

    #[test]
    fn apply_appends_when_missing_and_rejects_non_host_text() {
        let (mut items, _) = parse_file(FILE);
        assert!(apply_host_text(&mut items, "new", "Host new\n  User root\n").unwrap());
        assert!(serialize_items(&items, true).ends_with("Host new\n  User root\n"));
        assert!(apply_host_text(&mut items, "bad", "# just a comment\n").is_err());
        assert!(apply_host_text(&mut items, "mismatch", "Host other\n").is_err());
        // 同一套規則的純檢查(A3 在合併前用它):
        assert!(validate_host_text("bad", "# just a comment\n").is_err());
        assert!(validate_host_text("mismatch", "Host other\n").is_err());
        assert!(validate_host_text("two", "Host two\nHost three\n").is_err());
        assert!(validate_host_text("new", "Host new\n  User root\n").is_ok());
    }

    #[test]
    fn remove_deletes_only_that_block() {
        let (mut items, _) = parse_file(FILE);
        assert!(remove_host_block(&mut items, "web-1"));
        assert!(!remove_host_block(&mut items, "web-1"));
        let text = serialize_items(&items, true);
        assert!(!text.contains("web-1"));
        assert!(text.contains("Host db-1"));
        assert!(text.starts_with("# synced by sshelter\n"));
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::hosts_file 2>&1 | tail -5`
Expected: 編譯錯誤(`managed_path` 未定義)。

- [ ] **Step 3: 實作**

在 `use` 之後、tests 之前加入:

```rust
/// 寫進主 config 的 Include 值;`~` 在 macOS/Linux/Windows OpenSSH 皆可解析。
pub const INCLUDE_VALUE: &str = "~/.ssh/sshelter/hosts.config";

pub fn managed_path(ssh_dir: &Path) -> PathBuf {
    ssh_dir.join("sshelter").join("hosts.config")
}

/// 建立 `~/.ssh/sshelter/`(0700)與空的 `hosts.config`(0600);已存在則不動內容。
pub fn ensure_managed_file(ssh_dir: &Path) -> Result<PathBuf, AppError> {
    let path = managed_path(ssh_dir);
    let dir = path.parent().expect("managed path always has a parent");
    fsutil::ensure_dir_secure(dir)?;
    if !path.exists() {
        fsutil::atomic_write(&path, b"", 0o600)?;
    }
    Ok(path)
}

fn is_our_include(item: &Item) -> bool {
    matches!(item, Item::Directive(d) if d.key == "include" && d.enabled && d.value.split_whitespace().any(|t| t == INCLUDE_VALUE))
}

/// 同步 Include 的位置:前導註解/空行之後、其他任何項目(既有 Include、全域指令、Host/Match)之前。
/// 刻意不用 `newfile::include_insert_index`(它插在最後一個 Include **之後**):ssh 是
/// first-obtained-wins,同步檔必須是第一個被讀到的定義,spec §10 的遮蔽承諾才成立。
fn sync_include_index(items: &[Item]) -> usize {
    items
        .iter()
        .position(|i| !matches!(i, Item::Blank(_) | Item::Comment(_)))
        .unwrap_or(items.len())
}

/// 主 config 的同步 Include 必須在最頂端(見 `sync_include_index`):沒有就插入;已存在但不在
/// 最頂端(舊版插法、使用者搬動)就搬上去 —— 多路徑的 `Include a b` 只抽走我們的 token。
/// 回傳是否改了 items。
pub fn ensure_include(items: &mut Vec<Item>) -> bool {
    let top = sync_include_index(items);
    // 「已經正確」= 在最頂端、而且那一行只有我們這一個路徑。多路徑的 `Include a ours` 就算在首行,
    // ssh 也會先讀 a —— 一律正規化成獨立的一行。
    let exact = |item: &Item| matches!(item, Item::Directive(d) if d.key == "include" && d.enabled && d.value.trim() == INCLUDE_VALUE);
    match items.iter().position(is_our_include) {
        Some(pos) if pos == top && exact(&items[pos]) => false,
        Some(pos) => {
            let leftover: Vec<String> = match &items[pos] {
                Item::Directive(d) => d
                    .value
                    .split_whitespace()
                    .filter(|t| *t != INCLUDE_VALUE)
                    .map(str::to_string)
                    .collect(),
                _ => Vec::new(),
            };
            if leftover.is_empty() {
                items.remove(pos);
            } else if let Item::Directive(d) = &mut items[pos] {
                d.value = leftover.join(" ");
                d.dirty = true;
            }
            // `pos >= top`(Include 是 Directive,不可能在前導註解區裡):移除或就地改寫都不影響 top。
            items.insert(top, Item::Directive(Directive::new("Include", INCLUDE_VALUE, "")));
            true
        }
        None => {
            items.insert(top, Item::Directive(Directive::new("Include", INCLUDE_VALUE, "")));
            true
        }
    }
}

/// 具名 pattern 才同步;含 wildcard/否定字元的 pattern(`*`、`*.internal`、`!x`)是裝置本地的 config 結構。
pub fn is_syncable_alias(alias: &str) -> bool {
    !alias.is_empty() && !alias.contains(['*', '?', '!'])
}

/// 整個區塊的同步資格:非空且**所有** pattern 都具名。`Host web *.internal` 的第一個 pattern 具名,
/// 但它是 wildcard 規則的一部分 —— 整個區塊不擷取、不上傳、不遷入、不套用、不刪除。
pub fn is_syncable_block(patterns: &[String]) -> bool {
    !patterns.is_empty() && patterns.iter().all(|p| is_syncable_alias(p))
}

fn first_alias(item: &Item) -> Option<&str> {
    match item {
        Item::Host(h) => h.patterns.first().map(String::as_str),
        _ => None,
    }
}

/// 第一個 pattern 等於 `alias` 的具名 Host 區塊位置。alias 本身是 wildcard、或本地那個同名區塊含
/// wildcard pattern(不能替換它、也不能在旁邊再附加一個 `Host alias`)→ Err;找不到 → Ok(None)。
fn syncable_host_position(items: &[Item], alias: &str) -> Result<Option<usize>, AppError> {
    if !is_syncable_alias(alias) {
        return Err(AppError::Other(format!("'{alias}' is a wildcard pattern; wildcard blocks are never synced")));
    }
    match items.iter().position(|i| first_alias(i) == Some(alias)) {
        Some(p) => match &items[p] {
            Item::Host(h) if is_syncable_block(&h.patterns) => Ok(Some(p)),
            _ => Err(AppError::Other(format!("local block '{alias}' has wildcard patterns and stays local"))),
        },
        None => Ok(None),
    }
}

#[derive(Clone, Debug, PartialEq)]
pub struct HostBlockText {
    pub alias: String,
    /// 區塊原始文字,含 header 與 body 所有行(含 `#tags:` 與尾隨空行)。
    pub text: String,
}

fn block_text(item: &Item) -> String {
    serialize_items(std::slice::from_ref(item), true)
}

/// 只取可同步的 Host 區塊(`is_syncable_block`);alias = 第一個 pattern。註解、空行、Match 與
/// 含 wildcard 的區塊一律略過。
pub fn blocks_of(items: &[Item]) -> Vec<HostBlockText> {
    items
        .iter()
        .filter_map(|item| match item {
            Item::Host(h) if is_syncable_block(&h.patterns) => h
                .patterns
                .first()
                .map(|alias| HostBlockText { alias: alias.clone(), text: block_text(item) }),
            _ => None,
        })
        .collect()
}

fn parse_single_host(alias: &str, text: &str) -> Result<Item, AppError> {
    let (parsed, _) = parse_file(text);
    let mut hosts = parsed.into_iter().filter(|i| matches!(i, Item::Host(_)));
    let host = hosts
        .next()
        .ok_or_else(|| AppError::Other(format!("synced record for '{alias}' has no Host block")))?;
    if hosts.next().is_some() {
        return Err(AppError::Other(format!("synced record for '{alias}' has more than one Host block")));
    }
    match &host {
        Item::Host(h) if h.patterns.first().map(String::as_str) != Some(alias) => {
            Err(AppError::Other(format!("synced record for '{alias}' names a different host")))
        }
        Item::Host(h) if !is_syncable_block(&h.patterns) => {
            Err(AppError::Other(format!("synced record for '{alias}' contains wildcard patterns")))
        }
        _ => Ok(host),
    }
}

/// `apply_host_text` 對文字的全部要求,拆成純檢查:恰好一個 Host 區塊、第一個 pattern 等於 alias、
/// 所有 pattern 皆具名。A3 在合併前用它把壞掉的遠端記錄擋在快取外(否則套用失敗的記錄會在下一輪
/// 被當成本機刪除而產生 tombstone)。
pub fn validate_host_text(alias: &str, text: &str) -> Result<(), AppError> {
    if !is_syncable_alias(alias) {
        return Err(AppError::Other(format!("'{alias}' is a wildcard pattern; wildcard blocks are never synced")));
    }
    parse_single_host(alias, text).map(|_| ())
}

/// 以原始文字替換(或附加)一個具名 Host 區塊。回傳是否真的改了內容;wildcard alias、本地同名區塊
/// 含 wildcard、文字不合法都回 Err。
pub fn apply_host_text(items: &mut Vec<Item>, alias: &str, text: &str) -> Result<bool, AppError> {
    let pos = syncable_host_position(items, alias)?;
    let incoming = parse_single_host(alias, text)?;
    match pos {
        Some(p) => {
            if block_text(&items[p]) == block_text(&incoming) {
                return Ok(false);
            }
            items[p] = incoming;
            Ok(true)
        }
        None => {
            items.push(incoming);
            Ok(true)
        }
    }
}

/// 移除一個具名 Host 區塊;wildcard alias 或本地區塊含 wildcard 一律 false(tombstone 刪不到本地結構)。
pub fn remove_host_block(items: &mut Vec<Item>, alias: &str) -> bool {
    match syncable_host_position(items, alias) {
        Ok(Some(p)) => {
            items.remove(p);
            true
        }
        _ => false,
    }
}
```

並在 `sync/mod.rs` 加 `pub mod hosts_file;`。

> 注意:`serialize_items` 的第二個參數是「檔尾是否補換行」;區塊文字一律以換行結尾,
> 所以比較與輸出都用 `true`。若 `Directive::new` 產生的 Include 行序列化格式與測試
> 預期不同(例如分隔符),以既有 `newfile.rs` 測試對 Include 行的期望為準調整測試字串,
> 不要改 `Directive::new`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::hosts_file 2>&1 | tail -5`
Expected: `9 passed`。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/hosts_file.rs
git commit -m "feat(sync): managed hosts file block operations"
```

---

### Task 4: `sync::state` —— 本機同步狀態與助記詞保管

**Files:**
- Modify: `src-tauri/src/fsutil.rs`(新增 `pub fn app_data_root()`)
- Modify: `src-tauri/src/mcp.rs:245-249`(改用 `fsutil::app_data_root()`)
- Create: `src-tauri/src/sync/state.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod state;`)

**Interfaces:**
- Consumes: `crate::secrets::{get, set, delete}`(account 字串;`delete` 對不存在的項目回 `Ok(())`,見 secrets.rs)、`crate::sync::record::{LocalRecord, Envelope, SCHEMA_VERSION}`
- Produces:
  - `pub const STATE_VERSION: u32 = 1;`
  - `pub const DEFAULT_RELAY_URL: &str`(= `option_env!("SSHELTER_RELAY_URL")` 或 `"http://127.0.0.1:8787"`)
  - `pub struct SyncState { pub version: u32, pub chain_id: Option<String>, pub device_id: String, pub device_name: String, pub relay_url: String, pub cursor_seq: u64, pub password_sync: bool, pub remote_schema_version: Option<u32>, pub baseline_established: bool, pub phrase_cleanup_pending: bool, pub records: BTreeMap<String, LocalRecord>, pub sealed: BTreeMap<String, Envelope>, pub last_sync_ms: Option<u64>, pub last_error: Option<String> }`
    - `baseline_established`:剛 Create/Join 後為 false,A3 的基線輪成功後設 true(spec §6);`phrase_cleanup_pending`:Leave 時 keychain 刪不掉 → true(持久化,重啟後仍顯示重試)。
    - `records`:只放本版會處理的種類(host/device/meta)的明文快取;`sealed`(key = `"{kind}:{id_hash}"`):本版不處理的種類(key/password/未知)的原始密文 envelope,絕不解密。
  - `impl SyncState { pub fn fresh(device_name: &str) -> Result<Self, AppError>; pub fn joined(&self) -> bool; pub fn read_only(&self) -> bool }`(`read_only` = `remote_schema_version > SCHEMA_VERSION`)
  - `pub fn state_path() -> Result<PathBuf, AppError>`
  - `pub fn load(path: &Path) -> Result<Option<SyncState>, AppError>`(檔案不存在 → `Ok(None)`)
  - `pub fn save(path: &Path, state: &SyncState) -> Result<(), AppError>`
  - `pub const MNEMONIC_ACCOUNT: &str = "sync:mnemonic";`
  - `pub fn store_mnemonic(words: &str) -> Result<(), AppError>` / `pub fn load_mnemonic() -> Result<Option<String>, AppError>` / `pub fn clear_mnemonic() -> Result<(), AppError>`

- [ ] **Step 1: 把 app data root 抽到 fsutil**

在 `src-tauri/src/fsutil.rs` 加入(放在 `ensure_dir_secure` 之前):

```rust
/// 應用程式本機資料根目錄(MCP policy、sync state 等皆放這裡)。
pub fn app_data_root() -> Result<PathBuf, AppError> {
    dirs::data_local_dir()
        .ok_or_else(|| AppError::Other("cannot determine local data directory".to_string()))
        .map(|p| p.join("org.homelab.sshelter"))
}
```

(確認檔案頂部已 `use std::path::PathBuf;`,沒有就加。)

把 `src-tauri/src/mcp.rs` 裡的私有 `fn app_data_root()` 整段刪除,兩處呼叫改為
`crate::fsutil::app_data_root()`。

Run: `cd src-tauri && cargo test mcp:: 2>&1 | tail -3`
Expected: 既有 MCP 測試全綠。

- [ ] **Step 2: 寫失敗的測試**

建立 `src-tauri/src/sync/state.rs`:

```rust
//! 本機同步狀態(`sync-state.json`,0600)與助記詞的 keychain 保管。
//! 助記詞永不落成純文字檔;派生值只在記憶體。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};

use serde::{Deserialize, Serialize};

use crate::error::AppError;
use crate::fsutil;
use crate::secrets;
use crate::sync::record::{Envelope, LocalRecord, SCHEMA_VERSION};

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn fresh_state_is_not_joined_and_has_a_device_id() {
        let s = SyncState::fresh("MacBook").unwrap();
        assert!(!s.joined());
        assert!(!s.read_only());
        assert_eq!(s.device_id.len(), 32);
        assert_eq!(s.device_name, "MacBook");
        assert_eq!(s.version, STATE_VERSION);
        assert_eq!(s.relay_url, DEFAULT_RELAY_URL);
        assert!(s.sealed.is_empty());
        assert!(!s.baseline_established);
        assert!(!s.phrase_cleanup_pending);
    }

    #[test]
    fn read_only_follows_the_persisted_remote_schema_version() {
        let mut s = SyncState::fresh("A").unwrap();
        s.remote_schema_version = Some(SCHEMA_VERSION);
        assert!(!s.read_only());
        s.remote_schema_version = Some(SCHEMA_VERSION + 1);
        assert!(s.read_only());
    }

    #[test]
    fn sealed_envelopes_survive_save_and_load_without_being_decoded() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        let mut s = SyncState::fresh("A").unwrap();
        s.sealed.insert(
            "password:ff".to_string(),
            Envelope { id_hash: "ff".into(), kind: "password".into(), seq: 4, nonce: "n".into(), ciphertext: "c".into(), deleted: false },
        );
        save(&path, &s).unwrap();
        let text = std::fs::read_to_string(&path).unwrap();
        assert!(text.contains("\"ciphertext\": \"c\""), "envelope is stored verbatim: {text}");
        assert_eq!(load(&path).unwrap().unwrap().sealed, s.sealed);
    }

    #[test]
    fn save_then_load_round_trips() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        let mut s = SyncState::fresh("A").unwrap();
        s.chain_id = Some("ab".repeat(32));
        s.cursor_seq = 7;
        save(&path, &s).unwrap();
        let back = load(&path).unwrap().expect("state exists");
        assert_eq!(back, s);
        assert!(back.joined());
    }

    #[test]
    fn missing_file_means_not_joined() {
        let dir = tempfile::tempdir().unwrap();
        assert!(load(&dir.path().join("nope.json")).unwrap().is_none());
    }

    #[test]
    fn corrupt_or_newer_state_is_an_error_not_a_panic() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        std::fs::write(&path, b"{ not json").unwrap();
        assert!(load(&path).is_err());
        std::fs::write(&path, format!("{{\"version\": {} }}", STATE_VERSION + 1)).unwrap();
        let err = load(&path).unwrap_err();
        assert!(err.to_string().contains("newer"), "got: {err}");
    }

    #[cfg(unix)]
    #[test]
    fn state_file_is_private() {
        use std::os::unix::fs::PermissionsExt;
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        save(&path, &SyncState::fresh("A").unwrap()).unwrap();
        let mode = std::fs::metadata(&path).unwrap().permissions().mode() & 0o777;
        assert_eq!(mode, 0o600);
    }
}
```

- [ ] **Step 3: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::state 2>&1 | tail -5`
Expected: 編譯錯誤(`SyncState` 未定義)。

- [ ] **Step 4: 實作**

在 `use` 之後、tests 之前加入:

```rust
pub const STATE_VERSION: u32 = 1;

/// 建置時由 CI 以 `SSHELTER_RELAY_URL` 注入正式中繼;本機開發預設指向 `wrangler dev`。
pub const DEFAULT_RELAY_URL: &str = match option_env!("SSHELTER_RELAY_URL") {
    Some(url) => url,
    None => "http://127.0.0.1:8787",
};

pub const MNEMONIC_ACCOUNT: &str = "sync:mnemonic";

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct SyncState {
    pub version: u32,
    /// None = 尚未建立/加入 chain。
    pub chain_id: Option<String>,
    pub device_id: String,
    pub device_name: String,
    pub relay_url: String,
    /// 已套用到本機的最大中繼序號。
    pub cursor_seq: u64,
    /// 本機是否參與密碼同步(Phase B 使用;A 只持久化)。
    pub password_sync: bool,
    /// chain 的 `meta.schema_version`(收到後持久化);比 `SCHEMA_VERSION` 新 → 唯讀模式。
    #[serde(default)]
    pub remote_schema_version: Option<u32>,
    /// 剛 Create/Join 後 false:第一輪是「以 chain 為準」的基線輪,不做本機 diff(spec §6)。
    #[serde(default)]
    pub baseline_established: bool,
    /// Leave 時 keychain 裡的助記詞刪不掉:持久化這個待辦,重啟後仍顯示警示與重試。
    #[serde(default)]
    pub phrase_cleanup_pending: bool,
    /// 明文快取,key = `record_key(kind, id)`;只放本版會處理的種類(host/device/meta)。
    pub records: BTreeMap<String, LocalRecord>,
    /// 本版不處理的種類(key/password/未知 kind)的原始密文 envelope,key = `"{kind}:{id_hash}"`。
    /// 絕不解密進這裡:祕密只能落在 keychain 或 `~/.ssh/<name>`(spec §2)。
    #[serde(default)]
    pub sealed: BTreeMap<String, Envelope>,
    pub last_sync_ms: Option<u64>,
    pub last_error: Option<String>,
}

impl SyncState {
    pub fn fresh(device_name: &str) -> Result<Self, AppError> {
        let mut bytes = [0u8; 16];
        getrandom::fill(&mut bytes)
            .map_err(|e| AppError::Other(format!("cannot create device id: {e}")))?;
        Ok(Self {
            version: STATE_VERSION,
            chain_id: None,
            device_id: bytes.iter().map(|b| format!("{b:02x}")).collect(),
            device_name: device_name.to_string(),
            relay_url: DEFAULT_RELAY_URL.to_string(),
            cursor_seq: 0,
            password_sync: false,
            remote_schema_version: None,
            baseline_established: false,
            phrase_cleanup_pending: false,
            records: BTreeMap::new(),
            sealed: BTreeMap::new(),
            last_sync_ms: None,
            last_error: None,
        })
    }

    pub fn joined(&self) -> bool {
        self.chain_id.is_some()
    }

    /// chain 用了比本 app 新的格式:只套用可理解的記錄、不上傳(spec §10)。
    pub fn read_only(&self) -> bool {
        self.remote_schema_version.is_some_and(|v| v > SCHEMA_VERSION)
    }
}

pub fn state_path() -> Result<PathBuf, AppError> {
    Ok(fsutil::app_data_root()?.join("sync-state.json"))
}

pub fn load(path: &Path) -> Result<Option<SyncState>, AppError> {
    let bytes = match std::fs::read(path) {
        Ok(b) => b,
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => return Ok(None),
        Err(e) => return Err(AppError::Io(e)),
    };
    // 先只看版本,避免新版欄位讓整個反序列化失敗時給出誤導訊息。
    #[derive(Deserialize)]
    struct Probe {
        version: u32,
    }
    let probe: Probe = serde_json::from_slice(&bytes)
        .map_err(|e| AppError::Other(format!("sync state is unreadable: {e}")))?;
    if probe.version > STATE_VERSION {
        return Err(AppError::Other(format!(
            "sync state was written by a newer SSHelter (version {}); update the app",
            probe.version
        )));
    }
    let state: SyncState = serde_json::from_slice(&bytes)
        .map_err(|e| AppError::Other(format!("sync state is unreadable: {e}")))?;
    Ok(Some(state))
}

pub fn save(path: &Path, state: &SyncState) -> Result<(), AppError> {
    if let Some(dir) = path.parent() {
        fsutil::ensure_dir_secure(dir)?;
    }
    let bytes = serde_json::to_vec_pretty(state)
        .map_err(|e| AppError::Other(format!("cannot serialize sync state: {e}")))?;
    fsutil::atomic_write(path, &bytes, 0o600)
}

pub fn store_mnemonic(words: &str) -> Result<(), AppError> {
    secrets::set(MNEMONIC_ACCOUNT, words)
}

pub fn load_mnemonic() -> Result<Option<String>, AppError> {
    secrets::get(MNEMONIC_ACCOUNT)
}

pub fn clear_mnemonic() -> Result<(), AppError> {
    secrets::delete(MNEMONIC_ACCOUNT)
}
```

並在 `sync/mod.rs` 加 `pub mod state;`。

> `secrets::delete` 對不存在的項目回 `Ok(())`(secrets.rs 已如此,清理路徑可重入),所以
> `clear_mnemonic` 直接轉發即可;其他錯誤(keychain 不可用、拒絕存取)必須往上傳 ——
> A3 的 Leave 據此回報「recovery phrase still in keychain」,不得吞掉。

- [ ] **Step 5: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::state 2>&1 | tail -5`
Expected: 全綠(unix 上 7 個、Windows 上 6 個)。

- [ ] **Step 6: Commit**

```bash
git add src-tauri/src/fsutil.rs src-tauri/src/mcp.rs src-tauri/src/sync/mod.rs src-tauri/src/sync/state.rs
git commit -m "feat(sync): persist local sync state and keep the mnemonic in the keychain"
```

---

### Task 5: `sync::relay` —— 中繼 HTTP client

**Files:**
- Create: `src-tauri/src/sync/relay.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod relay;`)

**Interfaces:**
- Consumes: `crate::sync::record::Envelope`(Task 2;relay.rs 以 `pub use` 重新匯出,A3 可從 `relay::Envelope` 取用)
- Produces(wire 格式為 camelCase JSON,對應 A2 的 Worker):
  - `pub use crate::sync::record::Envelope;`
  - `pub struct PushItem { pub id_hash: String, pub kind: String, pub nonce: String, pub ciphertext: String, pub deleted: bool, pub base_seq: u64 }`
  - `pub enum PushResult { Accepted { seq: u64 }, Conflict { current: Envelope } }`
  - `pub struct PullResponse { pub records: Vec<Envelope>, pub latest_seq: u64 }`
  - `pub struct RelayClient` with `pub fn validate_url(base_url: &str) -> Result<String, AppError>`(純驗證:只接受 `https://`,`http://` 僅限 loopback host;回傳正規化 URL)、`pub fn new(base_url: &str, auth_token: &str) -> Result<Self, AppError>`(先 `validate_url`;不跟隨 redirect)、`pub fn create_chain(&self, chain_id: &str) -> Result<(), AppError>`、`pub fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError>`、`pub fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError>`、`pub fn delete_chain(&self, chain_id: &str) -> Result<(), AppError>`
  - 呼叫端契約:**只能在非 tokio 執行緒呼叫**(同步執行緒或 `tauri::async_runtime::spawn_blocking`);`reqwest::blocking` 在 async runtime 內會 panic。

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/relay.rs`:

```rust
//! 中繼 HTTP client(spec §5)。只搬密文;所有錯誤映射成 `AppError`,絕不 panic。
//! 使用 blocking client:同步引擎跑在自己的 std 執行緒。**不可在 tokio runtime 內呼叫**
//! (`reqwest::blocking` 會 panic);Tauri command 要用 `tauri::async_runtime::spawn_blocking`。

use std::time::Duration;

use serde::{Deserialize, Serialize};

use crate::error::AppError;
pub use crate::sync::record::Envelope;

#[cfg(test)]
mod tests {
    use super::*;

    fn item(id: &str, base: u64) -> PushItem {
        PushItem {
            id_hash: id.to_string(),
            kind: "host".to_string(),
            nonce: "bm9uY2U=".to_string(),
            ciphertext: "Y2lwaGVy".to_string(),
            deleted: false,
            base_seq: base,
        }
    }

    #[test]
    fn create_chain_sends_bearer_and_accepts_2xx() {
        let mut server = mockito::Server::new();
        let m = server
            .mock("PUT", "/v1/chains/abc")
            .match_header("authorization", "Bearer tok")
            .with_status(201)
            .with_body("{}")
            .create();
        let client = RelayClient::new(&server.url(), "tok").unwrap();
        client.create_chain("abc").unwrap();
        m.assert();
    }

    #[test]
    fn push_parses_accepted_and_conflict_results() {
        let mut server = mockito::Server::new();
        server
            .mock("POST", "/v1/chains/abc/records")
            .match_header("authorization", "Bearer tok")
            .with_status(200)
            .with_header("content-type", "application/json")
            .with_body(
                r#"{"results":[{"status":"ok","seq":12},{"status":"conflict","current":{"idHash":"h2","kind":"host","seq":9,"nonce":"n","ciphertext":"c","deleted":true}}],"latestSeq":12}"#,
            )
            .create();
        let client = RelayClient::new(&server.url(), "tok").unwrap();
        let results = client.push("abc", &[item("h1", 0), item("h2", 3)]).unwrap();
        assert_eq!(results.len(), 2);
        assert!(matches!(results[0], PushResult::Accepted { seq: 12 }));
        match &results[1] {
            PushResult::Conflict { current } => {
                assert_eq!(current.id_hash, "h2");
                assert_eq!(current.seq, 9);
                assert!(current.deleted);
            }
            other => panic!("expected conflict, got {other:?}"),
        }
    }

    #[test]
    fn pull_sends_since_and_parses_envelopes() {
        let mut server = mockito::Server::new();
        server
            .mock("GET", "/v1/chains/abc/records?since=5")
            .with_status(200)
            .with_header("content-type", "application/json")
            .with_body(r#"{"records":[{"idHash":"h1","kind":"host","seq":6,"nonce":"n","ciphertext":"c","deleted":false}],"latestSeq":6}"#)
            .create();
        let client = RelayClient::new(&server.url(), "tok").unwrap();
        let resp = client.pull("abc", 5).unwrap();
        assert_eq!(resp.latest_seq, 6);
        assert_eq!(resp.records.len(), 1);
        assert_eq!(resp.records[0].kind, "host");
    }

    #[test]
    fn unknown_chain_or_bad_token_is_not_found() {
        let mut server = mockito::Server::new();
        server.mock("GET", "/v1/chains/abc/records?since=0").with_status(404).create();
        let client = RelayClient::new(&server.url(), "tok").unwrap();
        assert!(matches!(client.pull("abc", 0), Err(AppError::NotFound(_))));
    }

    #[test]
    fn server_errors_and_garbage_bodies_become_errors_not_panics() {
        let mut server = mockito::Server::new();
        server.mock("GET", "/v1/chains/a/records?since=0").with_status(500).with_body("boom").create();
        server.mock("GET", "/v1/chains/b/records?since=0").with_status(200).with_body("not json").create();
        let client = RelayClient::new(&server.url(), "tok").unwrap();
        assert!(client.pull("a", 0).is_err());
        assert!(client.pull("b", 0).is_err());
    }

    #[test]
    fn unreachable_relay_fails_fast() {
        // 127.0.0.1:9 幾乎不會有人聽;connect timeout 2s 內必須回錯。
        let client = RelayClient::new("http://127.0.0.1:9", "tok").unwrap();
        let started = std::time::Instant::now();
        assert!(client.pull("abc", 0).is_err());
        assert!(started.elapsed() < Duration::from_secs(10));
    }

    #[test]
    fn rejects_relay_url_without_scheme() {
        assert!(RelayClient::new("sync.example.com", "tok").is_err());
        assert!(RelayClient::new("", "tok").is_err());
    }

    #[test]
    fn rejects_plain_http_except_loopback() {
        // bearer token 有整條 chain 的權限:非 loopback 一律要 https。
        assert!(RelayClient::new("http://sync.example.com", "tok").is_err());
        assert!(RelayClient::new("http://10.0.0.5:8787", "tok").is_err());
        assert!(RelayClient::new("ftp://sync.example.com", "tok").is_err());
        assert!(RelayClient::new("https://sync.example.com", "tok").is_ok());
        assert!(RelayClient::new("http://127.0.0.1:8787", "tok").is_ok());
        assert!(RelayClient::new("http://localhost:8787/", "tok").is_ok());
        assert!(RelayClient::new("http://[::1]:8787", "tok").is_ok());
    }

    #[test]
    fn validate_url_normalizes_without_building_a_client() {
        assert_eq!(RelayClient::validate_url(" https://relay.example.com/ ").unwrap(), "https://relay.example.com");
        assert_eq!(RelayClient::validate_url("https://relay.example.com/prefix/").unwrap(), "https://relay.example.com/prefix");
        assert!(RelayClient::validate_url("http://relay.example.com").is_err());
        // endpoint 用字串接在後面:query / fragment / userinfo 都拒絕。
        assert!(RelayClient::validate_url("https://relay.example.com/#x").is_err());
        assert!(RelayClient::validate_url("https://relay.example.com/?a=1").is_err());
        assert!(RelayClient::validate_url("https://user:pw@relay.example.com").is_err());
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::relay 2>&1 | tail -5`
Expected: 編譯錯誤(`RelayClient` 未定義)。

- [ ] **Step 3: 實作**

在 `use` 之後、tests 之前加入:

```rust
const CONNECT_TIMEOUT: Duration = Duration::from_secs(5);
const REQUEST_TIMEOUT: Duration = Duration::from_secs(20);

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct PushItem {
    pub id_hash: String,
    pub kind: String,
    pub nonce: String,
    pub ciphertext: String,
    pub deleted: bool,
    /// 本機最後看到的該記錄序號(新記錄為 0);中繼較新則回 conflict。
    pub base_seq: u64,
}

#[derive(Clone, Debug, PartialEq)]
pub enum PushResult {
    Accepted { seq: u64 },
    Conflict { current: Envelope },
}

#[derive(Deserialize)]
#[serde(tag = "status", rename_all = "lowercase")]
enum WirePushResult {
    Ok { seq: u64 },
    Conflict { current: Envelope },
}

#[derive(Deserialize)]
#[serde(rename_all = "camelCase")]
struct WirePushResponse {
    results: Vec<WirePushResult>,
}

#[derive(Clone, Debug, PartialEq, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct PullResponse {
    pub records: Vec<Envelope>,
    pub latest_seq: u64,
}

pub struct RelayClient {
    base_url: String,
    auth_token: String,
    http: reqwest::blocking::Client,
}

impl RelayClient {
    /// 只接受 `https://`;`http://` 僅限 loopback host(本機 `wrangler dev`)。回傳去掉尾斜線的正規化
    /// URL。純驗證、不建 client:A3 的 `sync_set_relay_url` 在 tokio 執行緒上也能用。bearer token 有
    /// 整條 chain 的讀寫刪權限,明文送出等於把 chain 交給網路上的任何人。
    pub fn validate_url(base_url: &str) -> Result<String, AppError> {
        let trimmed = base_url.trim().trim_end_matches('/');
        let url = reqwest::Url::parse(trimmed)
            .map_err(|_| AppError::Other("relay URL must be a full URL such as https://relay.example.com".to_string()))?;
        // `Url::host_str` 對 IPv6 回含中括號的 "[::1]"。
        let loopback = matches!(url.host_str(), Some("127.0.0.1") | Some("localhost") | Some("[::1]"));
        match url.scheme() {
            "https" => {}
            "http" if loopback => {}
            "http" => {
                return Err(AppError::Other(
                    "relay URL must use https:// (plain http is only allowed for 127.0.0.1 / localhost)".to_string(),
                ))
            }
            _ => return Err(AppError::Other("relay URL must start with https://".to_string())),
        }
        // endpoint 是用字串接在 base 後面的:query/fragment 會把 `/v1/...` 吞掉,userinfo 沒有理由出現。
        if url.query().is_some() || url.fragment().is_some() || !url.username().is_empty() || url.password().is_some() {
            return Err(AppError::Other("relay URL must not contain a query, fragment or credentials".to_string()));
        }
        Ok(trimmed.to_string())
    }

    pub fn new(base_url: &str, auth_token: &str) -> Result<Self, AppError> {
        let base_url = Self::validate_url(base_url)?;
        let http = reqwest::blocking::Client::builder()
            .user_agent(concat!("sshelter/", env!("CARGO_PKG_VERSION")))
            .connect_timeout(CONNECT_TIMEOUT)
            .timeout(REQUEST_TIMEOUT)
            // 不跟隨 redirect:避免 https → http 降級把 bearer token 送上明文連線。
            .redirect(reqwest::redirect::Policy::none())
            .build()
            .map_err(|e| AppError::Other(format!("cannot build HTTP client: {e}")))?;
        Ok(Self {
            base_url,
            auth_token: auth_token.to_string(),
            http,
        })
    }

    fn url(&self, path: &str) -> String {
        format!("{}{}", self.base_url, path)
    }

    /// 統一狀態碼映射:404 = chain 不存在或 token 不符(中繼刻意不區分)。
    fn check(resp: reqwest::blocking::Response) -> Result<reqwest::blocking::Response, AppError> {
        let status = resp.status();
        if status.is_success() {
            return Ok(resp);
        }
        let code = status.as_u16();
        match code {
            404 => Err(AppError::NotFound("sync chain not found on the relay (or wrong recovery phrase)".to_string())),
            413 => Err(AppError::Other("relay refused the upload: chain storage quota exceeded".to_string())),
            429 => Err(AppError::Other("relay is rate-limiting this chain; try again later".to_string())),
            _ => Err(AppError::Other(format!("relay returned HTTP {code}"))),
        }
    }

    fn send_err(e: reqwest::Error) -> AppError {
        // reqwest 錯誤訊息可能含 URL,但不含 token(token 在 header)。
        AppError::Other(format!("cannot reach the sync relay: {e}"))
    }

    pub fn create_chain(&self, chain_id: &str) -> Result<(), AppError> {
        let resp = self
            .http
            .put(self.url(&format!("/v1/chains/{chain_id}")))
            .bearer_auth(&self.auth_token)
            .json(&serde_json::json!({}))
            .send()
            .map_err(Self::send_err)?;
        Self::check(resp).map(|_| ())
    }

    pub fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError> {
        let resp = self
            .http
            .post(self.url(&format!("/v1/chains/{chain_id}/records")))
            .bearer_auth(&self.auth_token)
            .json(items)
            .send()
            .map_err(Self::send_err)?;
        let body: WirePushResponse = Self::check(resp)?
            .json()
            .map_err(|e| AppError::Other(format!("relay sent an unreadable push response: {e}")))?;
        Ok(body
            .results
            .into_iter()
            .map(|r| match r {
                WirePushResult::Ok { seq } => PushResult::Accepted { seq },
                WirePushResult::Conflict { current } => PushResult::Conflict { current },
            })
            .collect())
    }

    pub fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError> {
        let resp = self
            .http
            .get(self.url(&format!("/v1/chains/{chain_id}/records?since={since}")))
            .bearer_auth(&self.auth_token)
            .send()
            .map_err(Self::send_err)?;
        Self::check(resp)?
            .json()
            .map_err(|e| AppError::Other(format!("relay sent an unreadable pull response: {e}")))
    }

    pub fn delete_chain(&self, chain_id: &str) -> Result<(), AppError> {
        let resp = self
            .http
            .delete(self.url(&format!("/v1/chains/{chain_id}")))
            .bearer_auth(&self.auth_token)
            .send()
            .map_err(Self::send_err)?;
        Self::check(resp).map(|_| ())
    }
}
```

並在 `sync/mod.rs` 加 `pub mod relay;`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::relay 2>&1 | tail -5`
Expected: `9 passed`(mockito 綁 `127.0.0.1` 的隨機 port —— 屬 loopback,所以 `http://` 可用;不需網路)。

- [ ] **Step 5: 全套測試與 Commit**

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result'`
Expected: 全綠、無 warning 以外的輸出。

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/relay.rs
git commit -m "feat(sync): relay HTTP client with conflict-aware push"
```

---

## Self-review(已執行)

- **Spec 覆蓋**:§4 密碼學(含已知答案向量)→ Task 1;§3.2 記錄、§4 envelope 與 §6 合併規則 → Task 2;§3.1 受管檔、Include 置頂、wildcard 政策 → Task 3;§6 狀態檔(`records`/`sealed`/`remote_schema_version`)、助記詞 keychain(§4)→ Task 4;§5 中繼 API 與 §2 的 HTTPS/不跟隨 redirect → Task 5。背景執行緒、套用流程、UI、migration wizard 在 A3;Worker 在 A2。
- **型別一致**:`Envelope` 定義在 Task 2(`record.rs`),Task 4 的 `sealed` 與 Task 5 的 client(`pub use`)共用;`PushItem` 欄位名與 A2 Worker 的 wire 格式(camelCase)一致;`LocalRecord.seq` 對應 `PushItem.base_seq`;`record_key` 作為 `SyncState.records` 的 key。
- **Review Focus 對應**:1 → Task 1 `normalize_accepts_messy_input_and_rejects_bad_words`;2 → Task 3 `blocks_of_…skips_the_rest`、`apply_replaces_one_block…` 與 `wildcard_blocks_are_neither_extracted_nor_touched`;3 → Task 5 `unreachable_relay_fails_fast`;4 → Task 5 `server_errors_and_garbage_bodies…`;5 → Task 4 `corrupt_or_newer_state…`;6 → Task 5 `rejects_plain_http_except_loopback`;7 → Task 3 `ensure_include_goes_to_the_very_top_and_is_idempotent`。
- **Codex review(2026-09-27)已納入**:HMAC trait 歧義(finding 15)、AAD/協定 byte-level 定義與向量(16)、wildcard 政策(17)、http relay URL(3)、祕密不進狀態檔(2,`sealed`)、Include 置頂(13)、`secrets::delete` 行為(24)、reqwest 版本與 lockfile 對齊。
