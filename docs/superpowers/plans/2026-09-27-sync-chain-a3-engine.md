# Sync Chain — Phase A3(同步引擎:planner / reconcile / runtime / commands / migration)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 A1 的基礎串成會動的同步:偵測本機變更 → 拉取/合併/推送(純函式、以假中繼測試)→ 背景執行緒與 Tauri commands → 主機遷入與重複 alias 偵測。完成後 A4 只需要接 UI。

**Architecture:** `planner`(本機 diff)與 `reconcile`(對 `Relay` trait 的一輪同步)都是純函式,以記憶體假中繼做兩台裝置的模擬測試;`engine` 只負責:讀取 in-memory doc 的區塊、呼叫 reconcile、把 `HostEffect` 套回 doc 並 `persist_file`、存狀態、發事件。背景執行緒用 `mpsc::recv_timeout(45s)` 等喚醒。

**Tech Stack:** Rust、Tauri 2(`AppHandle`、`Emitter`、`Manager`)、ts-rs(bindings)、A1 全部模組。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §6(引擎)、§3.1(受管檔)、§10(migration)

## Global Constraints

- 同 A1:繁中註解、英文識別字/UI 文案、Conventional Commits、三平台 `cargo test` 全綠、祕密不進 log。
- 引擎**絕不**在鎖住 `AppState.doc` 的情況下做網路 I/O(reconcile 只吃已複製出來的區塊)。
- 任何同步失敗只寫 `state.last_error` 並發事件;本機操作永不被阻擋。
- 所有對 `hosts.config` 的寫入走既有 `persist_file`(備份 + 指紋衝突守衛)。
- 新 u64 欄位在 ts-rs 上一律 `#[cfg_attr(test, ts(type = "number"))]`(沿用 `mcp.rs` 做法)。

## Review Focus

1. 裝置時鐘落後 —— 本機修改的 `updated_at_ms` 必須 ≥ 該記錄前一版 +1,否則永遠輸給遠端(Task 1 測試)。
2. 中繼上有解不開的 envelope(別的 chain 殘留、損毀)—— 跳過並記錄,不可讓整輪同步失敗或 panic(Task 2 測試)。
3. 兩台同時改同一 host —— 時間戳大者勝;輸的一方若是未上傳的本機修改,`conflicts` 必須列出 alias(Task 2 測試)。
4. 受管檔在 app 未察覺時被手改 —— 同步前先比指紋、重載,不能用過期的 in-memory 區塊蓋掉(Task 3 測試:`gather_blocks` 指紋重載)。
5. `hosts.config` 的 `persist_file` 回 `Conflict`(磁碟已變)—— 重載一次再套用,仍失敗才回報(Task 3 測試)。

---

### Task 1: `sync::planner` —— 本機變更偵測

**Files:**
- Create: `src-tauri/src/sync/planner.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod planner;`)

**Interfaces:**
- Consumes: `record::{Record, RecordKind, LocalRecord, HostPayload, record_key, SCHEMA_VERSION}`、`hosts_file::HostBlockText`
- Produces:
  - `pub fn next_timestamp(now_ms: u64, previous: Option<u64>) -> u64`
  - `pub fn detect_local_changes(cached: &BTreeMap<String, LocalRecord>, current: &[HostBlockText], device_id: &str, now_ms: u64) -> Vec<Record>`

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/planner.rs`:

```rust
//! 本機變更偵測:把受管檔目前的 Host 區塊與快取記錄比對,產生要上傳的新版記錄。

use std::collections::BTreeMap;

use serde_json::Value;

use crate::sync::hosts_file::HostBlockText;
use crate::sync::record::{record_key, HostPayload, LocalRecord, Record, RecordKind, SCHEMA_VERSION};

#[cfg(test)]
mod tests {
    use super::*;

    fn block(alias: &str, text: &str) -> HostBlockText {
        HostBlockText { alias: alias.to_string(), text: text.to_string() }
    }

    fn cached(alias: &str, text: &str, version: u64, updated: u64, deleted: bool) -> (String, LocalRecord) {
        let record = Record {
            kind: RecordKind::Host,
            id: alias.to_string(),
            version,
            updated_at_ms: updated,
            device_id: "dev-a".to_string(),
            deleted,
            payload: serde_json::to_value(HostPayload { schema: SCHEMA_VERSION, text: text.to_string() }).unwrap(),
        };
        (record_key(RecordKind::Host, alias), LocalRecord { record, seq: 1, dirty: false })
    }

    #[test]
    fn timestamps_never_go_backwards() {
        assert_eq!(next_timestamp(100, None), 100);
        assert_eq!(next_timestamp(100, Some(50)), 100);
        // 時鐘落後:必須比前一版大。
        assert_eq!(next_timestamp(100, Some(100)), 101);
        assert_eq!(next_timestamp(100, Some(500)), 501);
    }

    #[test]
    fn new_block_becomes_a_version_one_record() {
        let out = detect_local_changes(&BTreeMap::new(), &[block("a", "Host a\n")], "dev-a", 10);
        assert_eq!(out.len(), 1);
        assert_eq!(out[0].id, "a");
        assert_eq!(out[0].version, 1);
        assert_eq!(out[0].updated_at_ms, 10);
        assert!(!out[0].deleted);
        assert_eq!(out[0].payload["text"], "Host a\n");
    }

    #[test]
    fn unchanged_block_produces_nothing_and_changed_block_bumps_version() {
        let mut map = BTreeMap::new();
        let (k, v) = cached("a", "Host a\n", 3, 50, false);
        map.insert(k, v);
        assert!(detect_local_changes(&map, &[block("a", "Host a\n")], "dev-a", 60).is_empty());
        let out = detect_local_changes(&map, &[block("a", "Host a\n  User x\n")], "dev-a", 40);
        assert_eq!(out[0].version, 4);
        assert_eq!(out[0].updated_at_ms, 51, "clock skew must not lose to the previous version");
    }

    #[test]
    fn missing_block_becomes_a_tombstone_only_once() {
        let mut map = BTreeMap::new();
        let (k, v) = cached("gone", "Host gone\n", 1, 5, false);
        map.insert(k, v);
        let out = detect_local_changes(&map, &[], "dev-a", 9);
        assert_eq!(out.len(), 1);
        assert!(out[0].deleted);
        assert_eq!(out[0].version, 2);
        // 已是 tombstone 就不再重複產生。
        let (k, v) = cached("gone", "", 2, 9, true);
        let mut map2 = BTreeMap::new();
        map2.insert(k, v);
        assert!(detect_local_changes(&map2, &[], "dev-a", 20).is_empty());
    }

    #[test]
    fn a_block_that_reappears_after_a_tombstone_is_a_new_version() {
        let mut map = BTreeMap::new();
        let (k, v) = cached("back", "", 2, 9, true);
        map.insert(k, v);
        let out = detect_local_changes(&map, &[block("back", "Host back\n")], "dev-a", 30);
        assert_eq!(out.len(), 1);
        assert!(!out[0].deleted);
        assert_eq!(out[0].version, 3);
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::planner 2>&1 | tail -5`
Expected: 編譯錯誤(`next_timestamp` 未定義)。

- [ ] **Step 3: 實作**

在 `use` 之後、tests 之前:

```rust
/// 時間戳單調:裝置時鐘落後時仍必須大於前一版,否則 LWW 永遠輸。
pub fn next_timestamp(now_ms: u64, previous: Option<u64>) -> u64 {
    match previous {
        Some(p) if p >= now_ms => p + 1,
        _ => now_ms,
    }
}

fn host_text(payload: &Value) -> Option<String> {
    serde_json::from_value::<HostPayload>(payload.clone()).ok().map(|p| p.text)
}

/// 比對受管檔目前的區塊與快取:新增/修改 → 新版記錄;消失 → tombstone(僅一次)。
pub fn detect_local_changes(
    cached: &BTreeMap<String, LocalRecord>,
    current: &[HostBlockText],
    device_id: &str,
    now_ms: u64,
) -> Vec<Record> {
    let mut out = Vec::new();

    for block in current {
        let key = record_key(RecordKind::Host, &block.alias);
        let previous = cached.get(&key).map(|l| &l.record);
        let unchanged = previous
            .filter(|r| !r.deleted)
            .and_then(|r| host_text(&r.payload))
            .map(|t| t == block.text)
            .unwrap_or(false);
        if unchanged {
            continue;
        }
        out.push(Record {
            kind: RecordKind::Host,
            id: block.alias.clone(),
            version: previous.map(|r| r.version + 1).unwrap_or(1),
            updated_at_ms: next_timestamp(now_ms, previous.map(|r| r.updated_at_ms)),
            device_id: device_id.to_string(),
            deleted: false,
            payload: serde_json::to_value(HostPayload { schema: SCHEMA_VERSION, text: block.text.clone() })
                .expect("HostPayload serializes"),
        });
    }

    for local in cached.values() {
        let r = &local.record;
        if r.kind != RecordKind::Host || r.deleted {
            continue;
        }
        if current.iter().any(|b| b.alias == r.id) {
            continue;
        }
        out.push(Record {
            kind: RecordKind::Host,
            id: r.id.clone(),
            version: r.version + 1,
            updated_at_ms: next_timestamp(now_ms, Some(r.updated_at_ms)),
            device_id: device_id.to_string(),
            deleted: true,
            payload: Value::Null,
        });
    }

    out
}
```

並在 `sync/mod.rs` 加 `pub mod planner;`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::planner 2>&1 | tail -5`
Expected: `5 passed`。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/planner.rs
git commit -m "feat(sync): detect local host block changes"
```

---

### Task 2: `sync::reconcile` —— 一輪同步(純函式,假中繼測試)

**Files:**
- Create: `src-tauri/src/sync/reconcile.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod reconcile;`)
- Modify: `src-tauri/src/sync/relay.rs`(為 `RelayClient` 實作 `Relay` trait —— trait 定義在 reconcile.rs)

**Interfaces:**
- Consumes: `crypto::{ChainKeys, Sealed, seal, open, id_hash}`、`record::*`、`relay::{Envelope, PushItem, PushResult, PullResponse, RelayClient}`、`state::SyncState`、`planner::detect_local_changes`、`hosts_file::HostBlockText`
- Produces:
  - `pub trait Relay { fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError>; fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError>; }`
  - `pub struct HostEffect { pub alias: String, pub text: Option<String> }`(`None` = 移除區塊)
  - `pub struct Reconciled { pub host_effects: Vec<HostEffect>, pub conflicts: Vec<String>, pub read_only: bool, pub skipped: u32 }`
  - `pub fn reconcile(state: &mut SyncState, keys: &ChainKeys, relay: &dyn Relay, current: &[HostBlockText], now_ms: u64, platform: &str) -> Result<Reconciled, AppError>`
  - `pub fn own_device_record(state: &SyncState, now_ms: u64, platform: &str) -> Record`

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/reconcile.rs`:

```rust
//! 一輪同步:本機 diff → pull/merge → push(含衝突重合併)→ 裝置心跳。
//! 不碰檔案、不碰 Tauri;中繼以 trait 注入,測試用記憶體假中繼模擬兩台裝置。

use std::collections::BTreeMap;

use crate::error::AppError;
use crate::sync::crypto::{self, ChainKeys, Sealed};
use crate::sync::hosts_file::HostBlockText;
use crate::sync::planner::detect_local_changes;
use crate::sync::record::{
    merge, record_key, DevicePayload, HostPayload, LocalRecord, MergeOutcome, MetaPayload, Record, RecordKind,
    SCHEMA_VERSION,
};
use crate::sync::relay::{Envelope, PullResponse, PushItem, PushResult};
use crate::sync::state::SyncState;

#[cfg(test)]
mod tests {
    use super::*;
    use std::cell::RefCell;

    const WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art";

    /// 記憶體假中繼,語意與 relay Worker 相同:單調 seq、stale base_seq → conflict。
    #[derive(Default)]
    struct FakeRelay {
        rows: RefCell<BTreeMap<String, Envelope>>,
        latest: RefCell<u64>,
    }

    impl Relay for FakeRelay {
        fn pull(&self, _chain: &str, since: u64) -> Result<PullResponse, AppError> {
            let mut records: Vec<Envelope> = self.rows.borrow().values().filter(|e| e.seq > since).cloned().collect();
            records.sort_by_key(|e| e.seq);
            Ok(PullResponse { records, latest_seq: *self.latest.borrow() })
        }

        fn push(&self, _chain: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError> {
            let mut out = Vec::new();
            for item in items {
                let mut rows = self.rows.borrow_mut();
                if let Some(current) = rows.get(&item.id_hash) {
                    if current.seq > item.base_seq {
                        out.push(PushResult::Conflict { current: current.clone() });
                        continue;
                    }
                }
                *self.latest.borrow_mut() += 1;
                let seq = *self.latest.borrow();
                rows.insert(
                    item.id_hash.clone(),
                    Envelope {
                        id_hash: item.id_hash.clone(),
                        kind: item.kind.clone(),
                        seq,
                        nonce: item.nonce.clone(),
                        ciphertext: item.ciphertext.clone(),
                        deleted: item.deleted,
                    },
                );
                out.push(PushResult::Accepted { seq });
            }
            Ok(out)
        }
    }

    fn keys() -> ChainKeys {
        crypto::derive_keys(WORDS).unwrap()
    }

    fn device(name: &str) -> SyncState {
        let mut s = SyncState::fresh(name).unwrap();
        s.chain_id = Some(keys().chain_id);
        s.device_id = format!("{:0<32}", name); // 固定、可預期的字典序
        s
    }

    fn block(alias: &str, text: &str) -> HostBlockText {
        HostBlockText { alias: alias.to_string(), text: text.to_string() }
    }

    fn texts(effects: &[HostEffect]) -> Vec<(String, Option<String>)> {
        effects.iter().map(|e| (e.alias.clone(), e.text.clone())).collect()
    }

    #[test]
    fn host_created_on_a_arrives_on_b() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        let r = reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n")], 100, "macos").unwrap();
        assert!(r.host_effects.is_empty());
        assert!(!a.records["host:web"].dirty);
        let r = reconcile(&mut b, &keys(), &relay, &[], 200, "windows").unwrap();
        assert_eq!(texts(&r.host_effects), vec![("web".to_string(), Some("Host web\n".to_string()))]);
        assert!(b.records.contains_key("host:web"));
        assert!(b.cursor_seq > 0);
    }

    #[test]
    fn edits_and_deletes_flow_both_ways() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n")], 100, "macos").unwrap();
        reconcile(&mut b, &keys(), &relay, &[], 200, "linux").unwrap();
        // B 修改後 A 收到新文字;A 刪除後 B 收到移除。
        reconcile(&mut b, &keys(), &relay, &[block("web", "Host web\n  User x\n")], 300, "linux").unwrap();
        let r = reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n")], 400, "macos").unwrap();
        assert_eq!(texts(&r.host_effects), vec![("web".to_string(), Some("Host web\n  User x\n".to_string()))]);
        let r = reconcile(&mut a, &keys(), &relay, &[], 500, "macos").unwrap();
        assert!(r.host_effects.is_empty());
        let r = reconcile(&mut b, &keys(), &relay, &[block("web", "Host web\n  User x\n")], 600, "linux").unwrap();
        assert_eq!(texts(&r.host_effects), vec![("web".to_string(), None)]);
    }

    #[test]
    fn concurrent_edit_is_resolved_by_timestamp_and_reported() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n")], 100, "macos").unwrap();
        reconcile(&mut b, &keys(), &relay, &[], 150, "linux").unwrap();
        // 兩邊離線各改一次:A 較早(200)、B 較晚(300)。A 先推,B 推時撞 conflict,但 B 較新 → B 勝。
        reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n  User a\n")], 200, "macos").unwrap();
        let rb = reconcile(&mut b, &keys(), &relay, &[block("web", "Host web\n  User b\n")], 300, "linux").unwrap();
        assert!(rb.host_effects.is_empty(), "B keeps its own newer text");
        assert!(rb.conflicts.is_empty());
        let ra = reconcile(&mut a, &keys(), &relay, &[block("web", "Host web\n  User a\n")], 400, "macos").unwrap();
        assert_eq!(texts(&ra.host_effects), vec![("web".to_string(), Some("Host web\n  User b\n".to_string()))]);
        // A 的未上傳修改被較新的遠端蓋掉 → 要列在 conflicts。
        let mut c = device("c");
        reconcile(&mut c, &keys(), &relay, &[], 450, "linux").unwrap();
        // C 離線改(時間 460),同時 B 又改(時間 900)並先推。
        reconcile(&mut b, &keys(), &relay, &[block("web", "Host web\n  User bb\n")], 900, "linux").unwrap();
        let rc = reconcile(&mut c, &keys(), &relay, &[block("web", "Host web\n  User c\n")], 460, "linux").unwrap();
        assert_eq!(rc.conflicts, vec!["web".to_string()]);
        assert_eq!(texts(&rc.host_effects), vec![("web".to_string(), Some("Host web\n  User bb\n".to_string()))]);
    }

    #[test]
    fn undecryptable_envelopes_are_skipped_not_fatal() {
        let relay = FakeRelay::default();
        relay.rows.borrow_mut().insert(
            "ff".repeat(32),
            Envelope { id_hash: "ff".repeat(32), kind: "host".into(), seq: 1, nonce: "!!".into(), ciphertext: "!!".into(), deleted: false },
        );
        *relay.latest.borrow_mut() = 1;
        let mut a = device("a");
        let r = reconcile(&mut a, &keys(), &relay, &[], 100, "macos").unwrap();
        assert_eq!(r.skipped, 1);
        assert_eq!(a.cursor_seq, 1);
    }

    #[test]
    fn newer_schema_puts_this_device_in_read_only_mode() {
        let relay = FakeRelay::default();
        let k = keys();
        let meta = Record {
            kind: RecordKind::Meta,
            id: "chain".into(),
            version: 1,
            updated_at_ms: 1,
            device_id: "z".into(),
            deleted: false,
            payload: serde_json::to_value(MetaPayload { schema_version: SCHEMA_VERSION + 1, created_by_app_version: "9.9.9".into() }).unwrap(),
        };
        let item = encode(&k, &meta, 0).unwrap();
        relay.push("x", &[item]).unwrap();
        let mut a = device("a");
        let r = reconcile(&mut a, &k, &relay, &[block("web", "Host web\n")], 100, "macos").unwrap();
        assert!(r.read_only);
        assert!(a.records["host:web"].dirty, "nothing is pushed in read-only mode");
        assert!(a.last_error.as_deref().unwrap_or("").contains("update"));
    }

    #[test]
    fn device_heartbeat_is_uploaded_and_visible_to_others() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        reconcile(&mut a, &keys(), &relay, &[], 100, "macos").unwrap();
        reconcile(&mut b, &keys(), &relay, &[], 200, "windows").unwrap();
        let seen: Vec<_> = b.records.values().filter(|l| l.record.kind == RecordKind::Device).map(|l| l.record.id.clone()).collect();
        assert!(seen.contains(&a.device_id));
        assert!(seen.contains(&b.device_id));
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::reconcile 2>&1 | tail -5`
Expected: 編譯錯誤(`Relay`/`reconcile` 未定義)。

- [ ] **Step 3: 實作**

在 `use` 之後、tests 之前:

```rust
/// 中繼抽象:引擎只依賴 pull/push,測試以記憶體實作模擬多台裝置。
pub trait Relay {
    fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError>;
    fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError>;
}

impl Relay for crate::sync::relay::RelayClient {
    fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError> {
        crate::sync::relay::RelayClient::pull(self, chain_id, since)
    }
    fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError> {
        crate::sync::relay::RelayClient::push(self, chain_id, items)
    }
}

#[derive(Clone, Debug, PartialEq)]
pub struct HostEffect {
    pub alias: String,
    /// `None` = 移除該區塊。
    pub text: Option<String>,
}

#[derive(Clone, Debug, Default, PartialEq)]
pub struct Reconciled {
    pub host_effects: Vec<HostEffect>,
    /// 本機未上傳的修改被較新的遠端蓋掉的 alias。
    pub conflicts: Vec<String>,
    /// chain 的 schema 比本 app 新:只套用、不上傳。
    pub read_only: bool,
    /// 解不開/格式不符而跳過的 envelope 數。
    pub skipped: u32,
}

const HEARTBEAT_MS: u64 = 60 * 60 * 1000;

pub fn own_device_record(state: &SyncState, now_ms: u64, platform: &str) -> Record {
    let key = record_key(RecordKind::Device, &state.device_id);
    let previous = state.records.get(&key).map(|l| &l.record);
    let joined_at_ms = previous
        .and_then(|r| serde_json::from_value::<DevicePayload>(r.payload.clone()).ok())
        .map(|p| p.joined_at_ms)
        .unwrap_or(now_ms);
    Record {
        kind: RecordKind::Device,
        id: state.device_id.clone(),
        version: previous.map(|r| r.version + 1).unwrap_or(1),
        updated_at_ms: crate::sync::planner::next_timestamp(now_ms, previous.map(|r| r.updated_at_ms)),
        device_id: state.device_id.clone(),
        deleted: false,
        payload: serde_json::to_value(DevicePayload {
            schema: SCHEMA_VERSION,
            name: state.device_name.clone(),
            platform: platform.to_string(),
            joined_at_ms,
            last_seen_ms: now_ms,
            keys: Vec::new(),
        })
        .expect("DevicePayload serializes"),
    }
}

/// 記錄 → 上傳項目(整筆記錄 JSON 加密;`deleted` 明文供中繼算配額)。
pub fn encode(keys: &ChainKeys, record: &Record, base_seq: u64) -> Result<PushItem, AppError> {
    let plaintext = serde_json::to_vec(record).map_err(|e| AppError::Other(format!("cannot serialize record: {e}")))?;
    let sealed = crypto::seal(keys, record.kind.as_str(), &record.id, &plaintext)?;
    Ok(PushItem {
        id_hash: sealed.id_hash,
        kind: record.kind.as_str().to_string(),
        nonce: sealed.nonce,
        ciphertext: sealed.ciphertext,
        deleted: record.deleted,
        base_seq,
    })
}

/// envelope → 記錄;kind 與 id_hash 都必須和明文相符,否則視為損毀。
pub fn decode(keys: &ChainKeys, env: &Envelope) -> Result<Record, AppError> {
    let kind = RecordKind::parse(&env.kind)
        .ok_or_else(|| AppError::Other(format!("unknown record kind '{}'", env.kind)))?;
    let sealed = Sealed { id_hash: env.id_hash.clone(), nonce: env.nonce.clone(), ciphertext: env.ciphertext.clone() };
    let plaintext = crypto::open(keys, &env.kind, &env.id_hash, &sealed)?;
    let record: Record = serde_json::from_slice(&plaintext)
        .map_err(|e| AppError::Other(format!("record payload is unreadable: {e}")))?;
    if record.kind != kind || crypto::id_hash(keys, record.kind.as_str(), &record.id) != env.id_hash {
        return Err(AppError::Other("record identity does not match its envelope".to_string()));
    }
    Ok(record)
}

fn host_text(record: &Record) -> Option<String> {
    serde_json::from_value::<HostPayload>(record.payload.clone()).ok().map(|p| p.text)
}

/// 把一筆遠端記錄合併進本機狀態;回傳需要套到檔案的效果。
fn take_remote(state: &mut SyncState, record: Record, seq: u64, out: &mut Reconciled) {
    let key = record_key(record.kind, &record.id);
    match merge(state.records.get(&key), &record) {
        MergeOutcome::KeepLocal => {
            // 本機較新:把 seq 更新到中繼現況,下次 push 才不會再撞 conflict。
            if let Some(local) = state.records.get_mut(&key) {
                local.seq = seq;
            }
        }
        outcome => {
            if outcome == MergeOutcome::RemoteWinsOverDirtyLocal && record.kind == RecordKind::Host {
                out.conflicts.push(record.id.clone());
            }
            if record.kind == RecordKind::Host {
                out.host_effects.push(HostEffect {
                    alias: record.id.clone(),
                    text: if record.deleted { None } else { host_text(&record) },
                });
            }
            if record.kind == RecordKind::Meta {
                if let Ok(meta) = serde_json::from_value::<MetaPayload>(record.payload.clone()) {
                    if meta.schema_version > SCHEMA_VERSION {
                        out.read_only = true;
                        state.last_error = Some(format!(
                            "this sync chain uses a newer format (schema {}); update SSHelter to keep syncing",
                            meta.schema_version
                        ));
                    }
                }
            }
            state.records.insert(key, LocalRecord { record, seq, dirty: false });
        }
    }
}

pub fn reconcile(
    state: &mut SyncState,
    keys: &ChainKeys,
    relay: &dyn Relay,
    current: &[HostBlockText],
    now_ms: u64,
    platform: &str,
) -> Result<Reconciled, AppError> {
    let chain_id = state.chain_id.clone().ok_or_else(|| AppError::Other("not in a sync chain".to_string()))?;
    let mut out = Reconciled::default();

    // 1. 本機變更 → dirty 記錄(保留既有 seq 當 base_seq)。
    for record in detect_local_changes(&state.records, current, &state.device_id, now_ms) {
        let key = record_key(record.kind, &record.id);
        let seq = state.records.get(&key).map(|l| l.seq).unwrap_or(0);
        state.records.insert(key, LocalRecord { record, seq, dirty: true });
    }

    // 2. 裝置心跳(至多每小時一次,避免無謂上傳)。
    let device_key = record_key(RecordKind::Device, &state.device_id);
    let stale = state
        .records
        .get(&device_key)
        .and_then(|l| serde_json::from_value::<DevicePayload>(l.record.payload.clone()).ok())
        .map(|p| now_ms.saturating_sub(p.last_seen_ms) >= HEARTBEAT_MS)
        .unwrap_or(true);
    if stale {
        let record = own_device_record(state, now_ms, platform);
        let seq = state.records.get(&device_key).map(|l| l.seq).unwrap_or(0);
        state.records.insert(device_key, LocalRecord { record, seq, dirty: true });
    }

    // 3. 拉取並合併。
    let pulled = relay.pull(&chain_id, state.cursor_seq)?;
    for env in &pulled.records {
        match decode(keys, env) {
            Ok(record) => take_remote(state, record, env.seq, &mut out),
            Err(_) => out.skipped += 1,
        }
    }
    state.cursor_seq = state.cursor_seq.max(pulled.latest_seq);

    // 4. 推送(唯讀模式下略過)。
    if !out.read_only {
        let dirty: Vec<(String, PushItem)> = state
            .records
            .iter()
            .filter(|(_, l)| l.dirty)
            .map(|(key, l)| encode(keys, &l.record, l.seq).map(|item| (key.clone(), item)))
            .collect::<Result<_, _>>()?;
        if !dirty.is_empty() {
            let items: Vec<PushItem> = dirty.iter().map(|(_, i)| i.clone()).collect();
            let results = relay.push(&chain_id, &items)?;
            for ((key, _), result) in dirty.iter().zip(results) {
                match result {
                    PushResult::Accepted { seq } => {
                        if let Some(local) = state.records.get_mut(key) {
                            local.seq = seq;
                            local.dirty = false;
                        }
                        state.cursor_seq = state.cursor_seq.max(seq);
                    }
                    PushResult::Conflict { current } => match decode(keys, &current) {
                        Ok(remote) => take_remote(state, remote, current.seq, &mut out),
                        Err(_) => out.skipped += 1,
                    },
                }
            }
        }
    }

    state.last_sync_ms = Some(now_ms);
    if !out.read_only {
        state.last_error = None;
    }
    Ok(out)
}
```

並在 `sync/mod.rs` 加 `pub mod reconcile;`。

> 注意 `state.cursor_seq = max(cursor, accepted seq)`:自己推上去的記錄不需要再拉回來。
> 但若同一輪別台也推了更大的 seq,下一輪 pull 仍會抓到(cursor 只前進到自己看過的最大值)。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::reconcile 2>&1 | tail -5`
Expected: `6 passed`。若 `concurrent_edit…` 失敗,先確認 FakeRelay 的 conflict 判斷是 `current.seq > item.base_seq`(與 Worker 相同),再檢查 `take_remote` 的 `KeepLocal` 分支有更新 seq。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/reconcile.rs
git commit -m "feat(sync): one-round reconcile with conflict handling and device heartbeat"
```

---

### Task 3: `sync::engine` —— 執行緒、檔案套用、Tauri commands

**Files:**
- Create: `src-tauri/src/sync/engine.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod engine;`)
- Modify: `src-tauri/src/state.rs`(加 `pub sync: crate::sync::engine::SyncRuntime`)
- Modify: `src-tauri/src/config/commands.rs`(`load_doc_migrated` 改 `pub(crate)`;`persist_file` 成功後呼叫 `crate::sync::engine::note_file_written(&path)`)
- Modify: `src-tauri/src/lib.rs`(setup 呼叫 `sync::engine::initialize`;註冊 commands)

**Interfaces:**
- Consumes: A1 全部、Task 1–2、`crate::keys::ssh_dir()`、`crate::config::commands::{persist_file, load_doc_migrated}`、`crate::config::model::SshConfigDoc`
- Produces(ts-rs 匯出到 `src/bindings/`):
  - `pub struct SyncDevice { pub id: String, pub name: String, pub platform: String, pub joined_at_ms: u64, pub last_seen_ms: u64, pub is_this: bool }`
  - `pub struct SyncStatus { pub joined: bool, pub chain_short: Option<String>, pub device_id: String, pub device_name: String, pub relay_url: String, pub last_sync_ms: Option<u64>, pub last_error: Option<String>, pub pending: u64, pub read_only: bool, pub devices: Vec<SyncDevice>, pub managed_file: String, pub hosts_in_sync: u64 }`
  - commands:`sync_status`、`sync_create_chain(device_name) -> String`、`sync_join_chain(words, device_name) -> SyncStatus`、`sync_show_words() -> String`、`sync_leave_chain(delete_remote: bool)`、`sync_now`、`sync_set_relay_url(url)`、`sync_set_device_name(name)`、`sync_remove_device(device_id)`
  - `pub fn initialize(app: &AppHandle) -> Result<(), AppError>`、`pub fn note_file_written(path: &Path)`、`pub fn wake()`
  - 事件:`sync://status`(payload `SyncStatus`)、`sync://conflict`(payload `Vec<String>` alias)

- [ ] **Step 1: 寫失敗的測試(可離線測的部分)**

建立 `src-tauri/src/sync/engine.rs`(先放 `use` 與測試):

```rust
//! 同步引擎:背景執行緒 + 把 reconcile 的結果套回 in-memory doc / 磁碟 + Tauri commands。
//! 網路與檔案的邊界:reconcile 只吃複製出來的區塊(不持有 doc 鎖);套用時才短暫鎖 doc。

use std::path::{Path, PathBuf};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::mpsc::{self, Receiver, RecvTimeoutError, Sender};
use std::sync::{Mutex, OnceLock};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

use serde::{Deserialize, Serialize};
use tauri::{AppHandle, Emitter, Manager};

use crate::config::commands::{load_doc_migrated, persist_file};
use crate::config::model::SshConfigDoc;
use crate::error::AppError;
use crate::state::AppState;
use crate::sync::crypto::{self, ChainKeys};
use crate::sync::hosts_file::{self, HostBlockText};
use crate::sync::reconcile::{self, HostEffect, Reconciled};
use crate::sync::record::{record_key, DevicePayload, MetaPayload, Record, RecordKind, LocalRecord, SCHEMA_VERSION};
use crate::sync::relay::RelayClient;
use crate::sync::state::{self as sync_state, SyncState};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::parser::parse_file;
    use crate::config::serialize::serialize_items;

    #[test]
    fn effects_are_applied_in_order_and_report_whether_anything_changed() {
        let (mut items, _) = parse_file("Host a\n  User x\n\nHost b\n");
        let effects = vec![
            HostEffect { alias: "a".into(), text: Some("Host a\n  User y\n\n".into()) },
            HostEffect { alias: "b".into(), text: None },
            HostEffect { alias: "c".into(), text: Some("Host c\n".into()) },
        ];
        assert!(apply_effects_to_items(&mut items, &effects).unwrap());
        assert_eq!(serialize_items(&items, true), "Host a\n  User y\n\nHost c\n");
        assert!(!apply_effects_to_items(&mut items, &[]).unwrap());
    }

    #[test]
    fn a_broken_effect_does_not_stop_the_others() {
        let (mut items, _) = parse_file("Host a\n");
        let effects = vec![
            HostEffect { alias: "bad".into(), text: Some("# not a host\n".into()) },
            HostEffect { alias: "ok".into(), text: Some("Host ok\n".into()) },
        ];
        // 壞的一筆回錯,但好的一筆已套用。
        assert!(apply_effects_to_items(&mut items, &effects).is_err());
        assert!(serialize_items(&items, true).contains("Host ok"));
    }

    #[test]
    fn status_reflects_state_without_a_chain() {
        let s = SyncState::fresh("Box").unwrap();
        let status = status_from(&s, "/tmp/hosts.config", false);
        assert!(!status.joined);
        assert_eq!(status.device_name, "Box");
        assert_eq!(status.pending, 0);
        assert!(status.devices.is_empty());
    }

    #[test]
    fn status_lists_devices_and_pending_counts() {
        let mut s = SyncState::fresh("Box").unwrap();
        s.chain_id = Some("ab".repeat(32));
        let me = reconcile::own_device_record(&s, 5, "macos");
        s.records.insert(record_key(RecordKind::Device, &s.device_id), LocalRecord { record: me, seq: 1, dirty: true });
        let host = Record {
            kind: RecordKind::Host,
            id: "web".into(),
            version: 1,
            updated_at_ms: 5,
            device_id: s.device_id.clone(),
            deleted: false,
            payload: serde_json::json!({ "schema": 1, "text": "Host web\n" }),
        };
        s.records.insert(record_key(RecordKind::Host, "web"), LocalRecord { record: host, seq: 0, dirty: true });
        let status = status_from(&s, "/tmp/hosts.config", false);
        assert!(status.joined);
        assert_eq!(status.chain_short.as_deref(), Some("abababab"));
        assert_eq!(status.pending, 2);
        assert_eq!(status.hosts_in_sync, 1);
        assert_eq!(status.devices.len(), 1);
        assert!(status.devices[0].is_this);
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::engine 2>&1 | tail -5`
Expected: 編譯錯誤(`apply_effects_to_items`/`status_from` 未定義)。

- [ ] **Step 3: 實作 —— 型別、狀態、純函式**

在 `use` 之後、tests 之前:

```rust
const SYNC_INTERVAL: Duration = Duration::from_secs(45);

#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncDevice {
    pub id: String,
    pub name: String,
    pub platform: String,
    #[cfg_attr(test, ts(type = "number"))]
    pub joined_at_ms: u64,
    #[cfg_attr(test, ts(type = "number"))]
    pub last_seen_ms: u64,
    pub is_this: bool,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncStatus {
    pub joined: bool,
    /// chain id 前 8 個 hex,只作辨識用。
    pub chain_short: Option<String>,
    pub device_id: String,
    pub device_name: String,
    pub relay_url: String,
    #[cfg_attr(test, ts(type = "number | null"))]
    pub last_sync_ms: Option<u64>,
    pub last_error: Option<String>,
    /// 尚未上傳的記錄數。
    #[cfg_attr(test, ts(type = "number"))]
    pub pending: u64,
    pub read_only: bool,
    pub devices: Vec<SyncDevice>,
    pub managed_file: String,
    #[cfg_attr(test, ts(type = "number"))]
    pub hosts_in_sync: u64,
}

/// Tauri 管理的同步執行期狀態。
pub struct SyncRuntime {
    pub state: Mutex<Option<SyncState>>,
    keys: Mutex<Option<ChainKeys>>,
    read_only: AtomicBool,
    syncing: AtomicBool,
}

impl Default for SyncRuntime {
    fn default() -> Self {
        Self {
            state: Mutex::new(None),
            keys: Mutex::new(None),
            read_only: AtomicBool::new(false),
            syncing: AtomicBool::new(false),
        }
    }
}

/// 喚醒背景執行緒的通道;由 `persist_file` 與 commands 共用,故放全域。
static WAKER: OnceLock<Mutex<Option<Sender<()>>>> = OnceLock::new();

pub fn wake() {
    if let Some(slot) = WAKER.get() {
        if let Some(tx) = slot.lock().unwrap().as_ref() {
            let _ = tx.send(());
        }
    }
}

/// `persist_file` 寫完任何檔案後呼叫:只有受管同步檔才會觸發同步。
pub fn note_file_written(path: &Path) {
    if let Ok(ssh_dir) = crate::keys::ssh_dir() {
        if path == hosts_file::managed_path(&ssh_dir) {
            wake();
        }
    }
}

fn now_ms() -> u64 {
    SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_millis() as u64).unwrap_or(0)
}

/// 把 reconcile 的效果套到區塊列表;回傳是否改了任何東西。壞掉的效果回錯但不中斷其他效果。
pub fn apply_effects_to_items(items: &mut Vec<crate::config::model::Item>, effects: &[HostEffect]) -> Result<bool, AppError> {
    let mut changed = false;
    let mut first_error: Option<AppError> = None;
    for effect in effects {
        let result = match &effect.text {
            Some(text) => hosts_file::apply_host_text(items, &effect.alias, text),
            None => Ok(hosts_file::remove_host_block(items, &effect.alias)),
        };
        match result {
            Ok(c) => changed |= c,
            Err(e) => {
                if first_error.is_none() {
                    first_error = Some(e);
                }
            }
        }
    }
    match first_error {
        Some(e) => Err(e),
        None => Ok(changed),
    }
}

pub fn status_from(state: &SyncState, managed_file: &str, read_only: bool) -> SyncStatus {
    let devices = state
        .records
        .values()
        .filter(|l| l.record.kind == RecordKind::Device && !l.record.deleted)
        .filter_map(|l| {
            let p: DevicePayload = serde_json::from_value(l.record.payload.clone()).ok()?;
            Some(SyncDevice {
                id: l.record.id.clone(),
                name: p.name,
                platform: p.platform,
                joined_at_ms: p.joined_at_ms,
                last_seen_ms: p.last_seen_ms,
                is_this: l.record.id == state.device_id,
            })
        })
        .collect();
    SyncStatus {
        joined: state.joined(),
        chain_short: state.chain_id.as_ref().map(|c| c.chars().take(8).collect()),
        device_id: state.device_id.clone(),
        device_name: state.device_name.clone(),
        relay_url: state.relay_url.clone(),
        last_sync_ms: state.last_sync_ms,
        last_error: state.last_error.clone(),
        pending: state.records.values().filter(|l| l.dirty).count() as u64,
        read_only,
        devices,
        managed_file: managed_file.to_string(),
        hosts_in_sync: state.records.values().filter(|l| l.record.kind == RecordKind::Host && !l.record.deleted).count() as u64,
    }
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::engine 2>&1 | tail -5`
Expected: `4 passed`。

- [ ] **Step 5: 實作 —— 啟動、執行緒、一輪同步的檔案邊界**

接在 `status_from` 之後加入:

```rust
fn managed_path() -> Result<PathBuf, AppError> {
    Ok(hosts_file::managed_path(&crate::keys::ssh_dir()?))
}

/// 確保受管檔存在、主 config 有 Include、且 doc 已載入受管檔。回傳受管檔路徑。
fn ensure_managed_loaded(app: &AppHandle) -> Result<PathBuf, AppError> {
    let ssh_dir = crate::keys::ssh_dir()?;
    let managed = hosts_file::ensure_managed_file(&ssh_dir)?;
    let state = app.state::<AppState>();
    let mut doc_lock = state.doc.lock().unwrap();
    let mut backed_up = state.backed_up.lock().unwrap();
    let retention = *state.backup_retention.lock().unwrap();
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let main_path = doc.files[0].path.clone();
    if hosts_file::ensure_include(&mut doc.files[0].items) {
        persist_file(doc, 0, &mut backed_up, retention)?;
    }
    if !doc.files.iter().any(|f| f.path == managed) {
        *doc_lock = Some(load_doc_migrated(&main_path)?);
    }
    Ok(managed)
}

/// 取出受管檔目前的區塊。磁碟若已被手改(指紋不同)先重載,避免用過期的 in-memory 內容。
fn gather_blocks(app: &AppHandle, managed: &Path) -> Result<Vec<HostBlockText>, AppError> {
    let state = app.state::<AppState>();
    let mut doc_lock = state.doc.lock().unwrap();
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let idx = doc
        .files
        .iter()
        .position(|f| f.path == managed)
        .ok_or_else(|| AppError::Other("synced hosts file is not loaded".to_string()))?;
    if crate::fsutil::has_changed(managed, &doc.files[idx].fingerprint).unwrap_or(true) {
        let main_path = doc.files[0].path.clone();
        *doc_lock = Some(load_doc_migrated(&main_path)?);
    }
    let doc = doc_lock.as_ref().expect("just loaded");
    let idx = doc.files.iter().position(|f| f.path == managed).ok_or_else(|| AppError::Other("synced hosts file vanished".to_string()))?;
    Ok(hosts_file::blocks_of(&doc.files[idx].items))
}

/// 套用效果並持久化;磁碟衝突時重載一次再試。
fn apply_effects(app: &AppHandle, managed: &Path, effects: &[HostEffect]) -> Result<(), AppError> {
    if effects.is_empty() {
        return Ok(());
    }
    let state = app.state::<AppState>();
    for attempt in 0..2 {
        let mut doc_lock = state.doc.lock().unwrap();
        let mut backed_up = state.backed_up.lock().unwrap();
        let retention = *state.backup_retention.lock().unwrap();
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        let idx = doc.files.iter().position(|f| f.path == managed).ok_or_else(|| AppError::Other("synced hosts file is not loaded".to_string()))?;
        let changed = apply_effects_to_items(&mut doc.files[idx].items, effects)?;
        if !changed {
            return Ok(());
        }
        match persist_file(doc, idx, &mut backed_up, retention) {
            Ok(()) => return Ok(()),
            Err(AppError::Conflict(_)) if attempt == 0 => {
                let main_path = doc.files[0].path.clone();
                *doc_lock = Some(load_doc_migrated(&main_path)?);
            }
            Err(e) => return Err(e),
        }
    }
    Err(AppError::Other("synced hosts file keeps changing on disk; try again".to_string()))
}

fn emit_status(app: &AppHandle) {
    let state = app.state::<AppState>();
    let guard = state.sync.state.lock().unwrap();
    if let Some(s) = guard.as_ref() {
        let managed = managed_path().map(|p| p.to_string_lossy().into_owned()).unwrap_or_default();
        let status = status_from(s, &managed, state.sync.read_only.load(Ordering::Relaxed));
        let _ = app.emit("sync://status", &status);
    }
}

fn save_state(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    let guard = state.sync.state.lock().unwrap();
    if let Some(s) = guard.as_ref() {
        sync_state::save(&sync_state::state_path()?, s)?;
    }
    Ok(())
}

/// 一輪同步。錯誤寫進 last_error 並發事件,永不 panic。
pub fn sync_once(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    if state.sync.syncing.swap(true, Ordering::SeqCst) {
        return Ok(()); // 已在同步中
    }
    let result = (|| -> Result<Reconciled, AppError> {
        let (mut s, keys) = {
            let guard = state.sync.state.lock().unwrap();
            let k = state.sync.keys.lock().unwrap();
            match (guard.as_ref(), k.as_ref()) {
                (Some(s), Some(k)) if s.joined() => (s.clone(), k.clone()),
                _ => return Ok(Reconciled::default()),
            }
        };
        let managed = ensure_managed_loaded(app)?;
        let blocks = gather_blocks(app, &managed)?;
        let relay = RelayClient::new(&s.relay_url, &keys.auth_token)?;
        let platform = std::env::consts::OS;
        // 不持有任何鎖做網路 I/O。
        let out = reconcile::reconcile(&mut s, &keys, &relay, &blocks, now_ms(), platform)?;
        apply_effects(app, &managed, &out.host_effects)?;
        *state.sync.state.lock().unwrap() = Some(s);
        state.sync.read_only.store(out.read_only, Ordering::Relaxed);
        Ok(out)
    })();
    state.sync.syncing.store(false, Ordering::SeqCst);

    match &result {
        Ok(out) if !out.conflicts.is_empty() => {
            let _ = app.emit("sync://conflict", &out.conflicts);
        }
        Err(e) => {
            if let Some(s) = state.sync.state.lock().unwrap().as_mut() {
                s.last_error = Some(e.to_string());
            }
        }
        _ => {}
    }
    let _ = save_state(app);
    emit_status(app);
    result.map(|_| ())
}

fn worker_loop(app: AppHandle, rx: Receiver<()>) {
    loop {
        match rx.recv_timeout(SYNC_INTERVAL) {
            Ok(()) | Err(RecvTimeoutError::Timeout) => {
                let _ = sync_once(&app);
            }
            Err(RecvTimeoutError::Disconnected) => return,
        }
    }
}

/// 啟動:載入狀態與助記詞,派生金鑰,開背景執行緒。狀態損毀不阻擋 app 啟動。
pub fn initialize(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    let loaded = match sync_state::state_path().and_then(|p| sync_state::load(&p)) {
        Ok(Some(s)) => s,
        Ok(None) => SyncState::fresh(&default_device_name())?,
        Err(e) => {
            let mut s = SyncState::fresh(&default_device_name())?;
            s.last_error = Some(e.to_string());
            s
        }
    };
    if loaded.joined() {
        match sync_state::load_mnemonic() {
            Ok(Some(words)) => *state.sync.keys.lock().unwrap() = crypto::derive_keys(&words).ok(),
            _ => {
                let mut s = loaded.clone();
                s.last_error = Some("recovery phrase is missing from the keychain; leave and rejoin the chain".to_string());
                *state.sync.state.lock().unwrap() = Some(s);
            }
        }
    }
    if state.sync.state.lock().unwrap().is_none() {
        *state.sync.state.lock().unwrap() = Some(loaded);
    }
    let (tx, rx) = mpsc::channel();
    *WAKER.get_or_init(|| Mutex::new(None)).lock().unwrap() = Some(tx);
    let handle = app.clone();
    std::thread::Builder::new()
        .name("sshelter-sync".into())
        .spawn(move || worker_loop(handle, rx))
        .map_err(AppError::Io)?;
    wake();
    Ok(())
}

fn default_device_name() -> String {
    std::env::var("COMPUTERNAME")
        .or_else(|_| std::env::var("HOSTNAME"))
        .ok()
        .filter(|s| !s.trim().is_empty())
        .unwrap_or_else(|| format!("SSHelter on {}", std::env::consts::OS))
}
```

> macOS 沒有 `HOSTNAME` 環境變數時會落到 `SSHelter on macos`;A4 的 UI 允許改名,足夠。

- [ ] **Step 6: 實作 —— Tauri commands**

接著加入:

```rust
fn with_state<T>(app: &AppHandle, f: impl FnOnce(&mut SyncState) -> Result<T, AppError>) -> Result<T, AppError> {
    let state = app.state::<AppState>();
    let mut guard = state.sync.state.lock().unwrap();
    let s = guard.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    f(s)
}

fn current_status(app: &AppHandle) -> Result<SyncStatus, AppError> {
    let state = app.state::<AppState>();
    let read_only = state.sync.read_only.load(Ordering::Relaxed);
    let managed = managed_path()?.to_string_lossy().into_owned();
    with_state(app, |s| Ok(status_from(s, &managed, read_only)))
}

/// 建立或加入 chain 的共同尾段:存助記詞、寫狀態、種下 device/meta 記錄、準備受管檔。
fn enter_chain(app: &AppHandle, words: &str, device_name: &str, is_creator: bool) -> Result<SyncStatus, AppError> {
    let keys = crypto::derive_keys(words)?;
    let relay_url = with_state(app, |s| Ok(s.relay_url.clone()))?;
    let relay = RelayClient::new(&relay_url, &keys.auth_token)?;
    // PUT 冪等:建立者建立;加入者驗證 token(錯的助記詞 → 404)。
    relay.create_chain(&keys.chain_id)?;
    sync_state::store_mnemonic(words)?;
    let now = now_ms();
    with_state(app, |s| {
        s.chain_id = Some(keys.chain_id.clone());
        s.device_name = device_name.trim().to_string();
        s.cursor_seq = 0;
        s.records.clear();
        s.last_error = None;
        let me = reconcile::own_device_record(s, now, std::env::consts::OS);
        s.records.insert(record_key(RecordKind::Device, &s.device_id), LocalRecord { record: me, seq: 0, dirty: true });
        if is_creator {
            let meta = Record {
                kind: RecordKind::Meta,
                id: "chain".to_string(),
                version: 1,
                updated_at_ms: now,
                device_id: s.device_id.clone(),
                deleted: false,
                payload: serde_json::to_value(MetaPayload {
                    schema_version: SCHEMA_VERSION,
                    created_by_app_version: env!("CARGO_PKG_VERSION").to_string(),
                })
                .expect("MetaPayload serializes"),
            };
            s.records.insert(record_key(RecordKind::Meta, "chain"), LocalRecord { record: meta, seq: 0, dirty: true });
        }
        Ok(())
    })?;
    *app.state::<AppState>().sync.keys.lock().unwrap() = Some(keys);
    ensure_managed_loaded(app)?;
    save_state(app)?;
    wake();
    current_status(app)
}

#[tauri::command]
pub fn sync_status(app: AppHandle) -> Result<SyncStatus, AppError> {
    current_status(&app)
}

#[tauri::command(async)]
pub fn sync_create_chain(app: AppHandle, device_name: String) -> Result<String, AppError> {
    if with_state(&app, |s| Ok(s.joined()))? {
        return Err(AppError::Other("already in a sync chain; leave it first".to_string()));
    }
    let words = crypto::generate_mnemonic()?;
    enter_chain(&app, &words, &device_name, true)?;
    Ok(words)
}

#[tauri::command(async)]
pub fn sync_join_chain(app: AppHandle, words: String, device_name: String) -> Result<SyncStatus, AppError> {
    if with_state(&app, |s| Ok(s.joined()))? {
        return Err(AppError::Other("already in a sync chain; leave it first".to_string()));
    }
    let normalized = crypto::normalize_mnemonic(&words)?;
    enter_chain(&app, &normalized, &device_name, false)
}

#[tauri::command]
pub fn sync_show_words(app: AppHandle) -> Result<String, AppError> {
    if !with_state(&app, |s| Ok(s.joined()))? {
        return Err(AppError::Other("not in a sync chain".to_string()));
    }
    sync_state::load_mnemonic()?.ok_or_else(|| AppError::Other("recovery phrase is not in the keychain".to_string()))
}

#[tauri::command(async)]
pub fn sync_leave_chain(app: AppHandle, delete_remote: bool) -> Result<SyncStatus, AppError> {
    let state = app.state::<AppState>();
    if delete_remote {
        let (chain, keys) = {
            let s = state.sync.state.lock().unwrap();
            let k = state.sync.keys.lock().unwrap();
            (s.as_ref().and_then(|s| s.chain_id.clone()), k.clone())
        };
        if let (Some(chain), Some(keys)) = (chain, keys) {
            let relay_url = with_state(&app, |s| Ok(s.relay_url.clone()))?;
            RelayClient::new(&relay_url, &keys.auth_token)?.delete_chain(&chain)?;
        }
    }
    let _ = sync_state::clear_mnemonic();
    *state.sync.keys.lock().unwrap() = None;
    state.sync.read_only.store(false, Ordering::Relaxed);
    with_state(&app, |s| {
        s.chain_id = None;
        s.cursor_seq = 0;
        s.records.clear();
        s.last_sync_ms = None;
        s.last_error = None;
        Ok(())
    })?;
    save_state(&app)?;
    emit_status(&app);
    current_status(&app)
}

#[tauri::command]
pub fn sync_now(app: AppHandle) -> Result<(), AppError> {
    wake();
    let _ = app;
    Ok(())
}

#[tauri::command]
pub fn sync_set_relay_url(app: AppHandle, url: String) -> Result<SyncStatus, AppError> {
    RelayClient::new(&url, "probe")?; // 只驗證格式
    with_state(&app, |s| {
        s.relay_url = url.trim().trim_end_matches('/').to_string();
        Ok(())
    })?;
    save_state(&app)?;
    current_status(&app)
}

#[tauri::command]
pub fn sync_set_device_name(app: AppHandle, name: String) -> Result<SyncStatus, AppError> {
    let name = name.trim().to_string();
    if name.is_empty() {
        return Err(AppError::Other("device name cannot be empty".to_string()));
    }
    let now = now_ms();
    with_state(&app, |s| {
        s.device_name = name;
        if s.joined() {
            let me = reconcile::own_device_record(s, now, std::env::consts::OS);
            let key = record_key(RecordKind::Device, &s.device_id);
            let seq = s.records.get(&key).map(|l| l.seq).unwrap_or(0);
            s.records.insert(key, LocalRecord { record: me, seq, dirty: true });
        }
        Ok(())
    })?;
    save_state(&app)?;
    wake();
    current_status(&app)
}

#[tauri::command]
pub fn sync_remove_device(app: AppHandle, device_id: String) -> Result<SyncStatus, AppError> {
    let now = now_ms();
    with_state(&app, |s| {
        if device_id == s.device_id {
            return Err(AppError::Other("use Leave chain to remove this device".to_string()));
        }
        let key = record_key(RecordKind::Device, &device_id);
        let Some(local) = s.records.get(&key).cloned() else {
            return Err(AppError::NotFound(format!("device {device_id} is not in this chain")));
        };
        let mut record = local.record;
        record.version += 1;
        record.updated_at_ms = crate::sync::planner::next_timestamp(now, Some(record.updated_at_ms));
        record.device_id = s.device_id.clone();
        record.deleted = true;
        s.records.insert(key, LocalRecord { record, seq: local.seq, dirty: true });
        Ok(())
    })?;
    save_state(&app)?;
    wake();
    current_status(&app)
}
```

- [ ] **Step 7: 接線**

1. `src-tauri/src/sync/mod.rs` 加 `pub mod engine;`。
2. `src-tauri/src/state.rs`:`AppState` 加欄位 `pub sync: crate::sync::engine::SyncRuntime,`,`Default` 加 `sync: crate::sync::engine::SyncRuntime::default(),`。
3. `src-tauri/src/config/commands.rs`:`fn load_doc_migrated` 改為 `pub(crate) fn load_doc_migrated`;在 `persist_file` 成功寫入後(函式回傳 `Ok(())` 之前)加 `crate::sync::engine::note_file_written(&path);`(`path` 為該函式開頭 clone 的檔案路徑)。
4. `src-tauri/src/lib.rs`:`use sync::engine::{sync_create_chain, sync_join_chain, sync_leave_chain, sync_now, sync_remove_device, sync_set_device_name, sync_set_relay_url, sync_show_words, sync_status};`,`generate_handler!` 清單加入這九個;`.setup` 內 `mcp::initialize(...)?;` 之後加 `sync::engine::initialize(app.handle())?;`。

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result'` 並 `ls ../src/bindings/ | grep -i sync`
Expected: 全綠;`SyncStatus.ts`、`SyncDevice.ts` 已生成。

- [ ] **Step 8: 手動 smoke(需 A2 的 relay 在本機跑)**

```bash
cd relay && npm run dev        # 127.0.0.1:8787
pnpm tauri dev                 # 另一個終端
```

在 devtools console 執行:`await window.__TAURI__.core.invoke("sync_create_chain", { deviceName: "dev" })` → 回 24 詞;`invoke("sync_status")` → `joined: true`、`devices` 含本機;`~/.ssh/config` 頂部出現 `Include ~/.ssh/sshelter/hosts.config`;把任一 host 用既有 Move 搬進 `hosts.config` 後幾秒內 `pending` 歸零。

- [ ] **Step 9: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/engine.rs src-tauri/src/state.rs src-tauri/src/config/commands.rs src-tauri/src/lib.rs src/bindings/SyncStatus.ts src/bindings/SyncDevice.ts
git commit -m "feat(sync): background engine, chain lifecycle commands and managed-file wiring"
```

---

### Task 4: 主機遷入與重複 alias 偵測

**Files:**
- Create: `src-tauri/src/sync/migrate.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod migrate;`)
- Modify: `src-tauri/src/lib.rs`(註冊 `sync_migrate_hosts`、`sync_duplicate_aliases`)

**Interfaces:**
- Consumes: `crate::config::commands::{move_host, persist_file}`、`crate::config::edit::set_tags`、`crate::config::edit::find_host_mut`、`crate::config::dto::host_summaries`
- Produces:
  - `pub fn tag_for_file(path: &Path) -> String`
  - `pub fn duplicate_aliases(doc: &SshConfigDoc, managed: &Path) -> Vec<DuplicateAlias>`
  - `pub struct DuplicateAlias { pub alias: String, pub local_file: String }`(ts-rs)
  - `pub struct MigrationFailure { pub alias: String, pub error: String }`、`pub struct MigrationReport { pub moved: Vec<String>, pub failed: Vec<MigrationFailure>, pub tagged: u64 }`(ts-rs)
  - commands:`sync_migrate_hosts(aliases: Vec<String>, tag_by_file: bool) -> MigrationReport`、`sync_duplicate_aliases() -> Vec<DuplicateAlias>`

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/migrate.rs`:

```rust
//! 既有主機遷入受管同步檔(spec §10):批次 move + 以原檔名上 tag;以及加入 chain 後
//! 「同步檔遮蔽本地同名主機」的偵測。

use std::path::Path;

use serde::{Deserialize, Serialize};
use tauri::{AppHandle, Manager};

use crate::config::commands::{move_host, persist_file};
use crate::config::edit::{find_host_mut, set_tags};
use crate::config::model::{Item, SshConfigDoc};
use crate::error::AppError;
use crate::state::AppState;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::include::load_doc;

    #[test]
    fn tag_for_file_strips_extensions_and_normalizes() {
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/homelab.config")), "homelab");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/Work Stuff.conf")), "work-stuff");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config")), "config");
    }

    #[test]
    fn duplicates_are_aliases_defined_both_in_managed_and_elsewhere() {
        let dir = tempfile::tempdir().unwrap();
        let managed = dir.path().join("hosts.config");
        std::fs::write(&managed, "Host web\n  HostName 1\nHost only-synced\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {}\nHost web\n  HostName 2\nHost local-only\n", managed.display())).unwrap();
        let doc = load_doc(&main).unwrap();
        let dups = duplicate_aliases(&doc, &managed);
        assert_eq!(dups.len(), 1);
        assert_eq!(dups[0].alias, "web");
        assert!(dups[0].local_file.ends_with("config"));
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::migrate 2>&1 | tail -5`
Expected: 編譯錯誤。

- [ ] **Step 3: 實作**

在 `use` 之後、tests 之前:

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct DuplicateAlias {
    pub alias: String,
    pub local_file: String,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct MigrationFailure {
    pub alias: String,
    pub error: String,
}

#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct MigrationReport {
    pub moved: Vec<String>,
    pub failed: Vec<MigrationFailure>,
    #[cfg_attr(test, ts(type = "number"))]
    pub tagged: u64,
}

/// `homelab.config` → `homelab`;非 `[a-z0-9_-]` 一律成 `-`。
pub fn tag_for_file(path: &Path) -> String {
    let stem = path
        .file_name()
        .map(|s| s.to_string_lossy().into_owned())
        .unwrap_or_default();
    let stem = stem
        .strip_suffix(".config")
        .or_else(|| stem.strip_suffix(".conf"))
        .unwrap_or(&stem)
        .to_lowercase();
    let mut out = String::new();
    for ch in stem.chars() {
        if ch.is_ascii_alphanumeric() || ch == '_' || ch == '-' {
            out.push(ch);
        } else if !out.ends_with('-') {
            out.push('-');
        }
    }
    out.trim_matches('-').to_string()
}

/// 同步檔與其他任何檔案都定義了的 alias(Include 在頂部 → 同步檔遮蔽本地定義)。
pub fn duplicate_aliases(doc: &SshConfigDoc, managed: &Path) -> Vec<DuplicateAlias> {
    let synced: Vec<String> = doc
        .files
        .iter()
        .filter(|f| f.path == managed)
        .flat_map(|f| f.items.iter())
        .filter_map(|i| match i {
            Item::Host(h) => h.patterns.first().cloned(),
            _ => None,
        })
        .collect();
    let mut out = Vec::new();
    for file in doc.files.iter().filter(|f| f.path != managed) {
        for item in &file.items {
            if let Item::Host(h) = item {
                if let Some(alias) = h.patterns.first() {
                    if synced.contains(alias) {
                        out.push(DuplicateAlias { alias: alias.clone(), local_file: file.path.to_string_lossy().into_owned() });
                    }
                }
            }
        }
    }
    out
}

#[tauri::command]
pub fn sync_duplicate_aliases(app: AppHandle) -> Result<Vec<DuplicateAlias>, AppError> {
    let managed = crate::sync::hosts_file::managed_path(&crate::keys::ssh_dir()?);
    let state = app.state::<AppState>();
    let guard = state.doc.lock().unwrap();
    let doc = guard.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    Ok(duplicate_aliases(doc, &managed))
}

/// 逐台搬進同步檔;每台獨立成功/失敗。`tag_by_file` 時把原檔名加成 tag(已有同名 tag 不重複)。
#[tauri::command]
pub fn sync_migrate_hosts(app: AppHandle, aliases: Vec<String>, tag_by_file: bool) -> Result<MigrationReport, AppError> {
    let managed = crate::sync::hosts_file::managed_path(&crate::keys::ssh_dir()?);
    let managed_str = managed.to_string_lossy().into_owned();
    let state = app.state::<AppState>();
    let mut doc_lock = state.doc.lock().unwrap();
    let mut backed_up = state.backed_up.lock().unwrap();
    let retention = *state.backup_retention.lock().unwrap();
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    if !doc.files.iter().any(|f| f.path == managed) {
        return Err(AppError::Other("synced hosts file is not loaded; create or join a chain first".to_string()));
    }

    let mut report = MigrationReport { moved: Vec::new(), failed: Vec::new(), tagged: 0 };
    for alias in aliases {
        let source_tag = crate::config::include::find_host_file_index(doc, &alias)
            .map(|i| tag_for_file(&doc.files[i].path));
        match move_host(doc, &alias, &managed_str) {
            Ok((src, tgt)) => {
                if let Err(e) = persist_file(doc, tgt, &mut backed_up, retention).and_then(|_| persist_file(doc, src, &mut backed_up, retention)) {
                    report.failed.push(MigrationFailure { alias, error: e.to_string() });
                    continue;
                }
                if tag_by_file {
                    if let (Some(tag), Some(host)) = (source_tag, find_host_mut(&mut doc.files[tgt].items, &alias)) {
                        let mut tags = crate::config::dto::host_summaries(doc)
                            .into_iter()
                            .find(|h| h.alias == alias)
                            .map(|h| h.tags)
                            .unwrap_or_default();
                        let _ = host; // find_host_mut 借用結束後再取 host_summaries;見下方重新查找
                        if !tags.contains(&tag) {
                            tags.push(tag);
                            if let Some(host) = find_host_mut(&mut doc.files[tgt].items, &alias) {
                                set_tags(host, &tags);
                            }
                            if persist_file(doc, tgt, &mut backed_up, retention).is_ok() {
                                report.tagged += 1;
                            }
                        }
                    }
                }
                report.moved.push(alias);
            }
            Err(e) => report.failed.push(MigrationFailure { alias, error: e.to_string() }),
        }
    }
    drop(doc_lock);
    crate::sync::engine::wake();
    Ok(report)
}
```

> 上面 tag 區段的借用寫法若編譯不過(`find_host_mut` 的可變借用與 `host_summaries(doc)` 的不可變借用重疊),改成:先算 `tags`(用 `host_summaries(doc)`),再 `if let Some(host) = find_host_mut(...) { set_tags(host, &tags); }`,兩段不重疊即可;把多餘的 `let _ = host;` 行刪掉。

並在 `sync/mod.rs` 加 `pub mod migrate;`;`lib.rs` 的 `use` 與 `generate_handler!` 加入 `sync_migrate_hosts`、`sync_duplicate_aliases`(路徑 `sync::migrate::…`)。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result' && ls ../src/bindings/ | grep -iE 'Migration|Duplicate'`
Expected: 全綠;`MigrationReport.ts`、`MigrationFailure.ts`、`DuplicateAlias.ts` 生成。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/migrate.rs src-tauri/src/lib.rs src/bindings/MigrationReport.ts src/bindings/MigrationFailure.ts src/bindings/DuplicateAlias.ts
git commit -m "feat(sync): migrate existing hosts into the synced file and detect shadowed aliases"
```

---

## Self-review(已執行)

- **Spec 覆蓋**:§6 觸發時機(啟動/45s/本地變更喚醒 → Task 3;視窗焦點觸發由 A4 前端呼叫 `sync_now`)、合併規則與衝突事件(Task 2)、套用到本機(Task 3 `apply_effects` 走 `persist_file`)、從本機產生記錄(Task 1 + Task 3 `gather_blocks` 指紋重載)、離線不阻擋(Task 3 錯誤只寫 last_error)、裝置身分與 Leave(Task 3);§10 遷入 wizard 的主機/tag 部分與重複 alias 偵測(Task 4;金鑰/密碼遷入在 Phase B)。
- **型別一致**:`HostEffect`/`Reconciled` 由 Task 2 定義、Task 3 使用;`own_device_record` 在 Task 2 定義、Task 3 的 `enter_chain`/`sync_set_device_name` 使用;`encode`/`decode` 只在 reconcile 內部與其測試使用;A1 的 `Sealed.id_hash` 與本計畫 `decode` 的比對一致。
- **Review Focus 對應**:1 → Task 1 `timestamps_never_go_backwards` + `unchanged_block…bumps_version`;2 → Task 2 `undecryptable_envelopes_are_skipped_not_fatal`;3 → Task 2 `concurrent_edit_is_resolved…`;4 → Task 3 `gather_blocks` 的指紋重載(手動 smoke 驗證:手改 hosts.config 後 45 秒內 pending 上升);5 → Task 3 `apply_effects` 的 Conflict 重試(以程式碼審查為主,無自動測試 —— 需要注入磁碟變更,列為 A4 的手動驗證項)。
