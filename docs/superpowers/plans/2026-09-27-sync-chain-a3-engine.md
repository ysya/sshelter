# Sync Chain — Phase A3(同步引擎:planner / reconcile / runtime / commands / migration)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 A1 的基礎串成會動的同步:偵測本機變更(逐區塊時間戳)→ 持久化 dirty → pull/merge → 在指紋與 generation 守衛下套用到受管檔 → 分批 push(純函式、以假中繼測試)→ 背景執行緒與 Tauri commands(lifecycle 互斥)→ 主機遷入、重複 alias 偵測與檔案定位的處理。完成後 A4 只需要接 UI。

**Architecture:** `planner`(本機 diff)與 `reconcile`(`plan_local` / `pull_merge` / `push_dirty` 三段純函式,對 `Relay` trait)以記憶體假中繼模擬多台裝置測試;`engine` 負責一輪的順序(spec §6)、`SyncCore`(generation/狀態/金鑰同一把鎖)、doc 鎖與檔案指紋守衛、逐區塊修改時間(`BlockTracker`)、套用失敗回饋快取、狀態持久化與事件;lifecycle commands 在 `tauri::async_runtime::spawn_blocking` 裡跑並全程持有 lifecycle 鎖(`reqwest::blocking` 不能在 tokio runtime 內跑)。背景執行緒用 `mpsc::recv_timeout(45s)` 等喚醒。鎖順序固定:lifecycle → doc → core。

**Tech Stack:** Rust、Tauri 2(`AppHandle`、`Emitter`、`Manager`、`async_runtime::spawn_blocking`)、ts-rs(bindings)、A1 全部模組。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §6(引擎:基線輪、一輪的順序、cursor、generation 與 lifecycle 互斥、sealed、唯讀)、§3.1(受管檔)、§10(migration、shadowed alias)

## Global Constraints

- 同 A1:繁中註解、英文識別字/UI 文案、Conventional Commits、三平台 `cargo test` 全綠、祕密不進 log。
- 引擎**絕不**在鎖住 `AppState.doc` 或 `sync.core` 的情況下做網路 I/O;`RelayClient` 只在同步執行緒(std thread)或 `spawn_blocking` 裡建立與使用。
- **cursor 只隨 pull 的 `latestSeq` 前進;push 回來的 seq 只更新該筆 `LocalRecord.seq`。**
- 本機 diff 產生的 dirty 記錄在任何網路操作之前先持久化;記錄的 `updated_at_ms` 是**逐區塊**的:app 存檔的區塊 = 存檔當下(`note_file_written` 記錄),外部編輯的區塊 = gather 時的檔案 mtime;心跳才用現在時間;重試沿用原版本/時間戳。
- 剛 Join 的第一輪是**基線輪**(`baseline_established = false`;Create 直接 true):不做本機 diff,先 pull 並以 chain 為準套用(含移除 chain 上已 tombstone、本機卻還留著的區塊),成功後才設 true 並立刻再跑一輪。
- 套用遠端效果前必須比對 gather 時的檔案指紋(in-memory 與磁碟都比);不同就整輪丟棄(cursor 不前進)並立刻再跑一輪。**套不上的 alias 不進快取**(退回合併前的本機版本或移除)並記入 `last_error`。持久化失敗先退回舊區塊、再從磁碟重載。
- `SyncCore { generation, state, keys }` 在同一把鎖裡,永遠一起快照、一起替換;`apply_effects` 在 doc 鎖內、寫檔前比 generation,`commit_state` 也比。Create/Join/Leave 全程持有 lifecycle 鎖(含前置檢查、網路、keychain);換 generation/狀態/金鑰時再持 doc 鎖。只動狀態的命令用 `mutate_state`(同一臨界區 +1 並改狀態)。
- 遠端 host 記錄只有三種結果:`Upsert`(文字先通過 `hosts_file::validate_host_text`:只含一個 Host 區塊)/ `Delete`(只有驗證過的 `deleted = true`)/ 略過(**不進快取**);格式不支援、文字不合法或 wildcard **絕不**當成刪除。
- 本版不處理的種類(key/password/未知)以原始 envelope 存進 `state.sealed`,不解密。
- 唯讀模式每輪從 `state.remote_schema_version` 判斷(持久化),不是只看本輪。
- Join 只用 `GET …/records?since=0` 驗證,404 → 「no sync chain matches this recovery phrase」,絕不 PUT;只有 Create 用 PUT。
- 任何同步失敗只寫 `state.last_error` 並發事件;本機操作永不被阻擋。
- 所有對 `hosts.config` 的寫入走既有 `persist_file`(備份 + 指紋衝突守衛)。
- 新 u64 欄位在 ts-rs 上一律 `#[cfg_attr(test, ts(type = "number"))]`(沿用 `mcp.rs` 做法)。

## Review Focus

1. 裝置時鐘落後 —— 本機修改的 `updated_at_ms` 必須 ≥ 該記錄前一版 +1,否則永遠輸給遠端(Task 1 測試)。
2. 中繼上有解不開的 envelope(別的 chain 殘留、損毀)—— 跳過並記錄,不可讓整輪同步失敗或 panic(Task 2 測試)。
3. 兩台同時改同一 host —— 時間戳大者勝;輸的一方若是未上傳的本機修改,`conflicts` 必須列出 alias(Task 2 測試)。
4. 自己 push 拿到的 seq 比別台剛寫入的大 —— cursor 不能跳過別台的記錄(Task 2 `cursor_only_follows_the_pull_watermark`)。
5. pull 之後、push 之前別台又推了同一筆 —— push 回 conflict,本輪不處理、記錄保持 dirty,下一輪 pull 合併(Task 2 `push_conflict_is_deferred_to_the_next_round`)。
6. 遠端 host 記錄 payload 解析失敗但 `deleted = false` —— 不得刪掉本機區塊(Task 2 `unparseable_host_payload_is_skipped_not_deleted`)。
7. 受管檔在 app 未察覺時被手改、或在網路期間被 UI 存檔 —— 同步前先比指紋重載;套用前再比一次,變了就丟棄本輪(Task 3 `apply_effects`;手動驗證)。
8. 一次遷入 450 台主機 —— push 必須分批(Task 2 `pushes_are_batched_by_count`)。
9. 遠端記錄的 payload 結構正確但文字不是合法區塊 —— 合併前就擋掉、不進快取,下一輪不得因此產生 tombstone(Task 2 `invalid_host_text_is_skipped_and_not_cached`)。
10. Leave 後 `hosts.config` 保留舊主機、別台刪了其中一台、再 Join —— 基線輪以 chain 為準,舊區塊不得復活遠端的刪除(Task 3 `run_round` 基線分支;手動驗證)。
11. 同一台上先改 `web`(t100)、再改另一台主機(t300),worker 才來收 —— `web` 的時間戳必須是 t100,不是整檔的 t300(Task 1 `each_changed_block_carries_its_own_change_time`、Task 3 `block_tracker_stamps_only_what_changed`)。
12. 遠端合法的 `Host web` 撞到本機 `Host web *.internal` 而套不上 —— 不得進快取,下一輪不得因此 tombstone 掉別台的 `web`(Task 3 `drop_unapplied`;`a_broken_effect_does_not_stop_the_others` 回傳失敗 alias)。

---

### Task 1: `sync::planner` —— 本機變更偵測(逐區塊時間)

**Files:**
- Create: `src-tauri/src/sync/planner.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod planner;`)

**Interfaces:**
- Consumes: `record::{Record, RecordKind, LocalRecord, HostPayload, record_key, SCHEMA_VERSION}`、`hosts_file::HostBlockText`
- Produces:
  - `pub fn next_timestamp(now_ms: u64, previous: Option<u64>) -> u64`
  - `pub fn detect_local_changes(cached: &BTreeMap<String, LocalRecord>, current: &[HostBlockText], device_id: &str, changed_at: impl Fn(&str) -> u64) -> Vec<Record>`(`changed_at(alias)` = 該區塊的修改時間;消失的區塊也用它決定 tombstone 時間)

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/planner.rs`:

```rust
//! 本機變更偵測:把受管檔目前的 Host 區塊與快取記錄比對,產生要上傳的新版記錄。
//! 時間戳逐區塊(由呼叫端提供),不是一個整檔的時間。

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
        let out = detect_local_changes(&BTreeMap::new(), &[block("a", "Host a\n")], "dev-a", |_| 10);
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
        assert!(detect_local_changes(&map, &[block("a", "Host a\n")], "dev-a", |_| 60).is_empty());
        let out = detect_local_changes(&map, &[block("a", "Host a\n  User x\n")], "dev-a", |_| 40);
        assert_eq!(out[0].version, 4);
        assert_eq!(out[0].updated_at_ms, 51, "clock skew must not lose to the previous version");
    }

    #[test]
    fn missing_block_becomes_a_tombstone_only_once() {
        let mut map = BTreeMap::new();
        let (k, v) = cached("gone", "Host gone\n", 1, 5, false);
        map.insert(k, v);
        let out = detect_local_changes(&map, &[], "dev-a", |_| 9);
        assert_eq!(out.len(), 1);
        assert!(out[0].deleted);
        assert_eq!(out[0].version, 2);
        assert_eq!(out[0].updated_at_ms, 9);
        // 已是 tombstone 就不再重複產生。
        let (k, v) = cached("gone", "", 2, 9, true);
        let mut map2 = BTreeMap::new();
        map2.insert(k, v);
        assert!(detect_local_changes(&map2, &[], "dev-a", |_| 20).is_empty());
    }

    #[test]
    fn a_block_that_reappears_after_a_tombstone_is_a_new_version() {
        let mut map = BTreeMap::new();
        let (k, v) = cached("back", "", 2, 9, true);
        map.insert(k, v);
        let out = detect_local_changes(&map, &[block("back", "Host back\n")], "dev-a", |_| 30);
        assert_eq!(out.len(), 1);
        assert!(!out[0].deleted);
        assert_eq!(out[0].version, 3);
    }

    #[test]
    fn other_kinds_in_the_cache_are_ignored() {
        let mut map = BTreeMap::new();
        let (k, mut v) = cached("dev", "", 1, 5, false);
        v.record.kind = RecordKind::Device;
        map.insert(k, v);
        // device 記錄不是主機,消失的區塊清單裡不能出現它。
        assert!(detect_local_changes(&map, &[], "dev-a", |_| 9).is_empty());
    }

    #[test]
    fn each_changed_block_carries_its_own_change_time() {
        // 先改 a(t100)、後改 b(t300),worker 才來收:a 不能被套上 b 的時間。
        let mut map = BTreeMap::new();
        let (k, v) = cached("a", "Host a\n", 1, 10, false);
        map.insert(k, v);
        let (k, v) = cached("b", "Host b\n", 1, 10, false);
        map.insert(k, v);
        let mut out = detect_local_changes(
            &map,
            &[block("a", "Host a\n  User x\n"), block("b", "Host b\n  User y\n")],
            "dev-a",
            |alias| if alias == "a" { 100 } else { 300 },
        );
        out.sort_by(|x, y| x.id.cmp(&y.id));
        assert_eq!(out[0].updated_at_ms, 100);
        assert_eq!(out[1].updated_at_ms, 300);
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
/// `changed_at(alias)` 是該區塊的修改時間(app 存檔當下或外部編輯的檔案 mtime),由呼叫端提供。
pub fn detect_local_changes(
    cached: &BTreeMap<String, LocalRecord>,
    current: &[HostBlockText],
    device_id: &str,
    changed_at: impl Fn(&str) -> u64,
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
            updated_at_ms: next_timestamp(changed_at(&block.alias), previous.map(|r| r.updated_at_ms)),
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
            updated_at_ms: next_timestamp(changed_at(&r.id), Some(r.updated_at_ms)),
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
Expected: `7 passed`。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/planner.rs
git commit -m "feat(sync): detect local host block changes with per-block change times"
```

---

### Task 2: `sync::reconcile` —— 一輪同步的三段純函式(假中繼測試)

**Files:**
- Create: `src-tauri/src/sync/reconcile.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod reconcile;`)

**Interfaces:**
- Consumes: `crypto::{ChainKeys, Sealed, seal, open, id_hash}`、`record::*`(含 `Envelope`)、`relay::{PushItem, PushResult, PullResponse, RelayClient}`、`state::SyncState`(含 `sealed`、`remote_schema_version`、`read_only()`)、`planner::{detect_local_changes, next_timestamp}`、`hosts_file::{HostBlockText, is_syncable_alias, validate_host_text}`
- Produces:
  - `pub trait Relay { fn pull(&self, chain_id: &str, since: u64) -> Result<PullResponse, AppError>; fn push(&self, chain_id: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError>; }`(`RelayClient` 實作它)
  - `pub enum HostEffect { Upsert { alias: String, text: String }, Delete { alias: String } }` + `pub fn alias(&self) -> &str`
  - `pub struct Merged { pub state: SyncState, pub host_effects: Vec<HostEffect>, pub conflicts: Vec<String>, pub skipped: u32 }`(`state` = 合併後的新狀態,含 `cursor_seq = latestSeq`;**套用成功後**才取代 runtime 的狀態)
  - `pub struct Pushed { pub accepted: usize, pub conflicts: usize }`
  - `pub fn own_device_record(state: &SyncState, now_ms: u64, platform: &str) -> Record`
  - `pub fn plan_local(state: &mut SyncState, current: &[HostBlockText], changed_at: impl Fn(&str) -> u64, now_ms: u64, platform: &str) -> usize`(寫入 dirty 記錄與心跳;記錄時間戳用 `changed_at(alias)`,心跳用 `now_ms`;回傳新增/更新的筆數)
  - `pub fn pull_merge(state: &SyncState, keys: &ChainKeys, relay: &dyn Relay) -> Result<Merged, AppError>`(純:不改傳入的 state)
  - `pub fn push_dirty(state: &mut SyncState, keys: &ChainKeys, relay: &dyn Relay) -> Result<Pushed, AppError>`(分批;唯讀模式直接回 0;accepted 只更新該筆 seq/dirty,**不動 cursor**;conflict 只計數)
  - `pub fn encode(keys, record, base_seq) -> Result<PushItem, AppError>`、`pub fn decode(keys, env) -> Result<Record, AppError>`

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/reconcile.rs`:

```rust
//! 一輪同步拆成三段純函式(spec §6):`plan_local`(本機 diff + 心跳,呼叫端立刻持久化)→
//! `pull_merge`(拉取、解密、LWW 合併;回傳「合併後的狀態」與要套到檔案的效果,呼叫端在指紋
//! 守衛下套用成功後才採用)→ `push_dirty`(分批上傳;只更新每筆的 seq,絕不動 cursor)。
//! 不碰檔案、不碰 Tauri;中繼以 trait 注入,測試用記憶體假中繼模擬多台裝置。

use std::collections::BTreeMap;

use crate::error::AppError;
use crate::sync::crypto::{self, ChainKeys, Sealed};
use crate::sync::hosts_file::{is_syncable_alias, validate_host_text, HostBlockText};
use crate::sync::planner::{detect_local_changes, next_timestamp};
use crate::sync::record::{
    merge, record_key, DevicePayload, Envelope, HostPayload, LocalRecord, MergeOutcome, MetaPayload, Record, RecordKind,
    SCHEMA_VERSION,
};
use crate::sync::relay::{PullResponse, PushItem, PushResult};
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
        push_calls: RefCell<u32>,
    }

    impl Relay for FakeRelay {
        fn pull(&self, _chain: &str, since: u64) -> Result<PullResponse, AppError> {
            let mut records: Vec<Envelope> = self.rows.borrow().values().filter(|e| e.seq > since).cloned().collect();
            records.sort_by_key(|e| e.seq);
            Ok(PullResponse { records, latest_seq: *self.latest.borrow() })
        }

        fn push(&self, _chain: &str, items: &[PushItem]) -> Result<Vec<PushResult>, AppError> {
            *self.push_calls.borrow_mut() += 1;
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

    fn upsert(alias: &str, text: &str) -> HostEffect {
        HostEffect::Upsert { alias: alias.to_string(), text: text.to_string() }
    }

    /// 模擬引擎一輪(不碰檔案):plan_local → pull_merge → 採用合併狀態 → push_dirty。
    fn round(state: &mut SyncState, relay: &FakeRelay, blocks: &[HostBlockText], now: u64) -> (Merged, Pushed) {
        let k = keys();
        plan_local(state, blocks, |_| now, now, "test");
        let merged = pull_merge(state, &k, relay).unwrap();
        *state = merged.state.clone();
        let pushed = push_dirty(state, &k, relay).unwrap();
        (merged, pushed)
    }

    fn host_record(id: &str, payload: serde_json::Value, deleted: bool) -> Record {
        Record { kind: RecordKind::Host, id: id.into(), version: 1, updated_at_ms: 1, device_id: "z".into(), deleted, payload }
    }

    #[test]
    fn host_created_on_a_arrives_on_b() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        let (m, p) = round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        assert!(m.host_effects.is_empty());
        assert_eq!(p.accepted, 2, "host + device heartbeat");
        assert!(!a.records["host:web"].dirty);
        let (m, _) = round(&mut b, &relay, &[], 200);
        assert_eq!(m.host_effects, vec![upsert("web", "Host web\n")]);
        assert!(b.records.contains_key("host:web"));
        assert!(b.cursor_seq > 0);
    }

    #[test]
    fn edits_and_deletes_flow_both_ways() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        round(&mut b, &relay, &[], 200);
        // B 修改後 A 收到新文字;A 刪除後 B 收到移除。
        round(&mut b, &relay, &[block("web", "Host web\n  User x\n")], 300);
        let (m, _) = round(&mut a, &relay, &[block("web", "Host web\n")], 400);
        assert_eq!(m.host_effects, vec![upsert("web", "Host web\n  User x\n")]);
        let (m, _) = round(&mut a, &relay, &[], 500);
        assert!(m.host_effects.is_empty());
        let (m, _) = round(&mut b, &relay, &[block("web", "Host web\n  User x\n")], 600);
        assert_eq!(m.host_effects, vec![HostEffect::Delete { alias: "web".into() }]);
    }

    #[test]
    fn concurrent_edit_is_resolved_by_timestamp_and_reported() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        round(&mut b, &relay, &[], 150);
        // 兩邊離線各改一次:A 較早(200)、B 較晚(300)。A 先推;B 這輪 pull 到 A 的版本但自己較新 → B 勝。
        round(&mut a, &relay, &[block("web", "Host web\n  User a\n")], 200);
        let (mb, _) = round(&mut b, &relay, &[block("web", "Host web\n  User b\n")], 300);
        assert!(mb.host_effects.is_empty(), "B keeps its own newer text");
        assert!(mb.conflicts.is_empty());
        let (ma, _) = round(&mut a, &relay, &[block("web", "Host web\n  User a\n")], 400);
        assert_eq!(ma.host_effects, vec![upsert("web", "Host web\n  User b\n")]);
        // C 離線改(時間 460),同時 B 又改(時間 900)並先推:C 的未上傳修改被較新的遠端蓋掉 → conflicts。
        let mut c = device("c");
        round(&mut c, &relay, &[], 450);
        round(&mut b, &relay, &[block("web", "Host web\n  User bb\n")], 900);
        let (mc, _) = round(&mut c, &relay, &[block("web", "Host web\n  User c\n")], 460);
        assert_eq!(mc.conflicts, vec!["web".to_string()]);
        assert_eq!(mc.host_effects, vec![upsert("web", "Host web\n  User bb\n")]);
        assert!(!c.records["host:web"].dirty);
    }

    #[test]
    fn push_conflict_is_deferred_to_the_next_round() {
        let relay = FakeRelay::default();
        let k = keys();
        let mut a = device("a");
        let mut b = device("b");
        round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        round(&mut b, &relay, &[], 150);
        // B:pull 之後、push 之前,A 又推了新版(時間 400)。
        plan_local(&mut b, &[block("web", "Host web\n  User b\n")], |_| 300, 300, "test");
        let merged = pull_merge(&b, &k, &relay).unwrap();
        b = merged.state;
        round(&mut a, &relay, &[block("web", "Host web\n  User a\n")], 400);
        let pushed = push_dirty(&mut b, &k, &relay).unwrap();
        assert_eq!(pushed.conflicts, 1);
        assert!(b.records["host:web"].dirty, "stays dirty until the next round merges the relay version");
        // 下一輪:A 的版本(400)較新 → B 收到效果、列入 conflicts、不再 dirty。
        let (m, _) = round(&mut b, &relay, &[block("web", "Host web\n  User b\n")], 500);
        assert_eq!(m.conflicts, vec!["web".to_string()]);
        assert_eq!(m.host_effects, vec![upsert("web", "Host web\n  User a\n")]);
        assert!(!b.records["host:web"].dirty);
    }

    #[test]
    fn cursor_only_follows_the_pull_watermark() {
        let relay = FakeRelay::default();
        let k = keys();
        let mut a = device("a");
        let mut b = device("b");
        round(&mut a, &relay, &[], 100);
        let cursor_before = a.cursor_seq;
        round(&mut b, &relay, &[block("x", "Host x\n"), block("y", "Host y\n")], 200);
        // A 推自己的變更:拿到比 B 的記錄更大的 seq,但 cursor 不能跳過 B 的記錄。
        plan_local(&mut a, &[block("z", "Host z\n")], |_| 300, 300, "test");
        push_dirty(&mut a, &k, &relay).unwrap();
        assert_eq!(a.cursor_seq, cursor_before);
        let (m, _) = round(&mut a, &relay, &[block("z", "Host z\n")], 400);
        let mut aliases: Vec<&str> = m.host_effects.iter().map(|e| e.alias()).collect();
        aliases.sort();
        assert_eq!(aliases, vec!["x", "y"]);
    }

    #[test]
    fn undecryptable_envelopes_are_skipped_not_fatal() {
        let relay = FakeRelay::default();
        relay.rows.borrow_mut().insert(
            "ff".repeat(32),
            Envelope { id_hash: "ff".repeat(32), kind: "host".into(), seq: 1, nonce: "!!".into(), ciphertext: "!!".into(), deleted: false },
        );
        *relay.latest.borrow_mut() = 1;
        let a = device("a");
        let m = pull_merge(&a, &keys(), &relay).unwrap();
        assert_eq!(m.skipped, 1);
        assert_eq!(m.state.cursor_seq, 1);
        assert!(m.host_effects.is_empty());
    }

    #[test]
    fn unsupported_kinds_are_kept_sealed_not_decoded() {
        let relay = FakeRelay::default();
        let k = keys();
        // 一筆合法加密的 password 記錄(Phase A 不處理)與一筆未知種類。
        let secret = Record { kind: RecordKind::Password, id: "web".into(), version: 1, updated_at_ms: 1, device_id: "z".into(), deleted: false, payload: serde_json::json!("hunter2") };
        relay.push("x", &[encode(&k, &secret, 0).unwrap()]).unwrap();
        relay.rows.borrow_mut().insert(
            "ee".repeat(32),
            Envelope { id_hash: "ee".repeat(32), kind: "future".into(), seq: 2, nonce: "n".into(), ciphertext: "c".into(), deleted: false },
        );
        *relay.latest.borrow_mut() = 2;
        let a = device("a");
        let m = pull_merge(&a, &k, &relay).unwrap();
        assert_eq!(m.skipped, 0);
        assert_eq!(m.state.sealed.len(), 2);
        assert!(m.state.sealed.keys().any(|key| key.starts_with("password:")));
        assert!(m.state.sealed.contains_key(&format!("future:{}", "ee".repeat(32))));
        assert!(m.state.records.values().all(|l| l.record.kind != RecordKind::Password));
        let text = serde_json::to_string(&m.state).unwrap();
        assert!(!text.contains("hunter2"), "secret payload must never reach the state");
        assert_eq!(m.state.cursor_seq, 2);
    }

    #[test]
    fn unparseable_host_payload_is_skipped_not_deleted() {
        let relay = FakeRelay::default();
        let k = keys();
        let mut a = device("a");
        round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        // 合法加密、但 payload 缺 text 的 host 記錄,時間比本機新。
        let mut broken = host_record("web", serde_json::json!({ "schema": 1 }), false);
        broken.updated_at_ms = 999;
        relay.push("x", &[encode(&k, &broken, a.records["host:web"].seq).unwrap()]).unwrap();
        let m = pull_merge(&a, &k, &relay).unwrap();
        assert!(m.host_effects.is_empty(), "never turns a bad payload into a delete");
        assert_eq!(m.skipped, 1);
        assert_eq!(m.state.records["host:web"], a.records["host:web"], "local cache untouched");
    }

    #[test]
    fn wildcard_records_from_remote_are_skipped() {
        let relay = FakeRelay::default();
        let k = keys();
        let star = host_record("*", serde_json::json!({ "schema": 1, "text": "Host *\n  User root\n" }), false);
        relay.push("x", &[encode(&k, &star, 0).unwrap()]).unwrap();
        let a = device("a");
        let m = pull_merge(&a, &k, &relay).unwrap();
        assert!(m.host_effects.is_empty());
        assert_eq!(m.skipped, 1);
    }

    #[test]
    fn invalid_host_text_is_skipped_and_not_cached() {
        let relay = FakeRelay::default();
        let k = keys();
        // payload 結構正確、但文字不是合法區塊 / alias 不符 / 夾帶 Match:合併前就擋掉,不進快取、不產生效果。
        let comment = host_record("web", serde_json::json!({ "schema": 1, "text": "# just a comment\n" }), false);
        let mismatch = host_record("db", serde_json::json!({ "schema": 1, "text": "Host other\n" }), false);
        let extra = host_record("app", serde_json::json!({ "schema": 1, "text": "Host app\n  User a\nMatch all\n  User b\n" }), false);
        relay
            .push("x", &[encode(&k, &comment, 0).unwrap(), encode(&k, &mismatch, 0).unwrap(), encode(&k, &extra, 0).unwrap()])
            .unwrap();
        let a = device("a");
        let m = pull_merge(&a, &k, &relay).unwrap();
        assert!(m.host_effects.is_empty());
        assert_eq!(m.skipped, 3);
        assert!(!m.state.records.contains_key("host:web"));
        assert!(!m.state.records.contains_key("host:db"));
        assert!(!m.state.records.contains_key("host:app"));
        // 下一輪的本機 diff 因此不會把它們當成「快取有、檔案沒有」而產生 tombstone。
        let mut s = m.state;
        assert_eq!(plan_local(&mut s, &[], |_| 100, 100, "test"), 1, "only the device heartbeat");
    }

    #[test]
    fn newer_schema_puts_this_device_in_read_only_mode_persistently() {
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
        relay.push("x", &[encode(&k, &meta, 0).unwrap()]).unwrap();
        let mut a = device("a");
        let (m, p) = round(&mut a, &relay, &[block("web", "Host web\n")], 100);
        assert!(m.state.read_only());
        assert_eq!(p.accepted, 0, "nothing is pushed in read-only mode");
        assert!(a.records["host:web"].dirty);
        // 下一輪 pull 不再帶 meta,仍然唯讀(從持久化的 remote_schema_version 判斷)。
        let (m, p) = round(&mut a, &relay, &[block("web", "Host web\n")], 200);
        assert!(m.state.read_only());
        assert_eq!(p.accepted, 0);
        assert!(a.records["host:web"].dirty);
    }

    #[test]
    fn device_heartbeat_is_uploaded_and_visible_to_others() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let mut b = device("b");
        round(&mut a, &relay, &[], 100);
        round(&mut b, &relay, &[], 200);
        let seen: Vec<_> = b.records.values().filter(|l| l.record.kind == RecordKind::Device).map(|l| l.record.id.clone()).collect();
        assert!(seen.contains(&a.device_id));
        assert!(seen.contains(&b.device_id));
    }

    #[test]
    fn pushes_are_batched_by_count() {
        let relay = FakeRelay::default();
        let mut a = device("a");
        let blocks: Vec<HostBlockText> = (0..450).map(|i| block(&format!("h{i}"), &format!("Host h{i}\n"))).collect();
        let (_, p) = round(&mut a, &relay, &blocks, 100);
        assert_eq!(p.accepted, 451, "450 hosts + device heartbeat");
        assert_eq!(*relay.push_calls.borrow(), 3, "200 + 200 + 51");
        assert!(a.records.values().all(|l| !l.dirty));
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::reconcile 2>&1 | tail -5`
Expected: 編譯錯誤(`Relay`/`plan_local` 未定義)。

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

/// 遠端 host 記錄要套到受管檔的效果。沒有第三種:格式不支援的記錄只會被略過。
#[derive(Clone, Debug, PartialEq)]
pub enum HostEffect {
    Upsert { alias: String, text: String },
    /// 只有驗證過的 `deleted = true` 才會產生。
    Delete { alias: String },
}

impl HostEffect {
    pub fn alias(&self) -> &str {
        match self {
            HostEffect::Upsert { alias, .. } | HostEffect::Delete { alias } => alias,
        }
    }
}

/// `pull_merge` 的結果。`state` 是合併後的完整新狀態(含 `cursor_seq`);呼叫端在指紋守衛下
/// 把 `host_effects` 寫進檔案成功後才用它取代 runtime 的狀態 —— 失敗就整份丟棄,下一輪重拉。
#[derive(Clone, Debug)]
pub struct Merged {
    pub state: SyncState,
    pub host_effects: Vec<HostEffect>,
    /// 本機未上傳的修改被較新的遠端蓋掉的 alias。
    pub conflicts: Vec<String>,
    /// 解不開/身分不符/格式不支援/wildcard 而略過的 envelope 數。
    pub skipped: u32,
}

#[derive(Clone, Copy, Debug, Default, PartialEq)]
pub struct Pushed {
    pub accepted: usize,
    /// 中繼比本機新的筆數:本輪不處理,呼叫端應立刻再跑一輪(pull 會拿到)。
    pub conflicts: usize,
}

const HEARTBEAT_MS: u64 = 60 * 60 * 1000;
const PUSH_BATCH_ITEMS: usize = 200;
const PUSH_BATCH_BYTES: usize = 512 * 1024;
/// 本版會解密進 `records` 的種類;其他一律原樣留在 `sealed`。
const HANDLED_KINDS: [RecordKind; 3] = [RecordKind::Host, RecordKind::Device, RecordKind::Meta];

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
        updated_at_ms: next_timestamp(now_ms, previous.map(|r| r.updated_at_ms)),
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

/// 第一段:本機 diff → dirty 記錄(保留既有 seq 當 base_seq),加上至多每小時一次的裝置心跳。
/// `changed_at(alias)` 是該區塊的修改時間(app 存檔當下或外部編輯的檔案 mtime),記錄的時間戳用它;
/// 心跳才用 `now_ms`。呼叫端必須在任何網路操作前把 state 持久化:重試沿用同一版本與時間戳。
pub fn plan_local(
    state: &mut SyncState,
    current: &[HostBlockText],
    changed_at: impl Fn(&str) -> u64,
    now_ms: u64,
    platform: &str,
) -> usize {
    let mut changed = 0;
    for record in detect_local_changes(&state.records, current, &state.device_id, changed_at) {
        let key = record_key(record.kind, &record.id);
        let seq = state.records.get(&key).map(|l| l.seq).unwrap_or(0);
        state.records.insert(key, LocalRecord { record, seq, dirty: true });
        changed += 1;
    }
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
        changed += 1;
    }
    changed
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

/// 遠端 host 記錄 → 效果。tombstone → Delete;文字通過 `validate_host_text`(只含一個具名 Host 區塊、
/// alias 相符、無 wildcard)→ Upsert;其他 → None(略過,且不進快取 —— 否則套用失敗的記錄會在
/// 下一輪被當成「快取有、檔案沒有」而產生 tombstone,把別台的主機刪掉)。
fn host_effect(record: &Record) -> Option<HostEffect> {
    if !is_syncable_alias(&record.id) {
        return None;
    }
    if record.deleted {
        return Some(HostEffect::Delete { alias: record.id.clone() });
    }
    let text = serde_json::from_value::<HostPayload>(record.payload.clone()).ok()?.text;
    validate_host_text(&record.id, &text).ok()?;
    Some(HostEffect::Upsert { alias: record.id.clone(), text })
}

#[derive(Default)]
struct Accumulator {
    host_effects: Vec<HostEffect>,
    conflicts: Vec<String>,
    skipped: u32,
}

/// 把一筆遠端記錄合併進狀態副本;host 記錄的效果與衝突收進 accumulator。
fn take_remote(state: &mut SyncState, record: Record, seq: u64, acc: &mut Accumulator) {
    let key = record_key(record.kind, &record.id);
    match merge(state.records.get(&key), &record) {
        MergeOutcome::KeepLocal => {
            // 本機較新:把 seq 更新到中繼現況,下次 push 才不會再撞 conflict。
            if let Some(local) = state.records.get_mut(&key) {
                local.seq = seq;
            }
        }
        outcome => {
            if record.kind == RecordKind::Host {
                match host_effect(&record) {
                    Some(effect) => {
                        if outcome == MergeOutcome::RemoteWinsOverDirtyLocal {
                            acc.conflicts.push(record.id.clone());
                        }
                        acc.host_effects.push(effect);
                    }
                    None => {
                        // 格式不支援 / 文字不合法 / wildcard:不套用、不覆蓋本機快取、絕不當成刪除。
                        acc.skipped += 1;
                        return;
                    }
                }
            }
            if record.kind == RecordKind::Meta {
                if let Ok(meta) = serde_json::from_value::<MetaPayload>(record.payload.clone()) {
                    state.remote_schema_version = Some(meta.schema_version);
                }
            }
            state.records.insert(key, LocalRecord { record, seq, dirty: false });
        }
    }
}

/// 第二段:pull → 解密 → LWW 合併。純函式:回傳合併後的新狀態,不改傳入的 state。
pub fn pull_merge(state: &SyncState, keys: &ChainKeys, relay: &dyn Relay) -> Result<Merged, AppError> {
    let chain_id = state.chain_id.clone().ok_or_else(|| AppError::Other("not in a sync chain".to_string()))?;
    let pulled = relay.pull(&chain_id, state.cursor_seq)?;
    let mut next = state.clone();
    let mut acc = Accumulator::default();
    for env in &pulled.records {
        match RecordKind::parse(&env.kind) {
            Some(kind) if HANDLED_KINDS.contains(&kind) => match decode(keys, env) {
                Ok(record) => take_remote(&mut next, record, env.seq, &mut acc),
                Err(_) => acc.skipped += 1,
            },
            _ => {
                // 本版不處理的種類(key/password/未知):原樣保留、永不解密(spec §2/§6)。
                next.sealed.insert(format!("{}:{}", env.kind, env.id_hash), env.clone());
            }
        }
    }
    // cursor 只跟 pull 的 watermark 走。
    next.cursor_seq = next.cursor_seq.max(pulled.latest_seq);
    Ok(Merged { state: next, host_effects: acc.host_effects, conflicts: acc.conflicts, skipped: acc.skipped })
}

fn send_batch(
    state: &mut SyncState,
    relay: &dyn Relay,
    chain_id: &str,
    batch: &[(String, PushItem)],
    pushed: &mut Pushed,
) -> Result<(), AppError> {
    let items: Vec<PushItem> = batch.iter().map(|(_, item)| item.clone()).collect();
    let results = relay.push(chain_id, &items)?;
    if results.len() != items.len() {
        return Err(AppError::Other("relay answered with the wrong number of results".to_string()));
    }
    for ((key, _), result) in batch.iter().zip(results) {
        match result {
            PushResult::Accepted { seq } => {
                if let Some(local) = state.records.get_mut(key) {
                    local.seq = seq;
                    local.dirty = false;
                }
                pushed.accepted += 1;
            }
            // 中繼比本機新:保持 dirty,下一輪 pull 會拿到中繼版本再合併(spec §6 第 5 步)。
            PushResult::Conflict { .. } => pushed.conflicts += 1,
        }
    }
    Ok(())
}

/// 第三段:分批上傳 dirty 記錄。accepted 只更新該筆 seq/dirty,**絕不推進 cursor**;唯讀模式不上傳。
pub fn push_dirty(state: &mut SyncState, keys: &ChainKeys, relay: &dyn Relay) -> Result<Pushed, AppError> {
    let chain_id = state.chain_id.clone().ok_or_else(|| AppError::Other("not in a sync chain".to_string()))?;
    let mut pushed = Pushed::default();
    if state.read_only() {
        return Ok(pushed);
    }
    let dirty_keys: Vec<String> = state.records.iter().filter(|(_, l)| l.dirty).map(|(k, _)| k.clone()).collect();
    let mut batch: Vec<(String, PushItem)> = Vec::new();
    let mut batch_bytes = 0usize;
    for key in dirty_keys {
        let Some(local) = state.records.get(&key) else { continue };
        let item = encode(keys, &local.record, local.seq)?;
        let size = item.ciphertext.len() + item.nonce.len();
        if !batch.is_empty() && (batch.len() >= PUSH_BATCH_ITEMS || batch_bytes + size > PUSH_BATCH_BYTES) {
            send_batch(state, relay, &chain_id, &batch, &mut pushed)?;
            batch.clear();
            batch_bytes = 0;
        }
        batch_bytes += size;
        batch.push((key, item));
    }
    if !batch.is_empty() {
        send_batch(state, relay, &chain_id, &batch, &mut pushed)?;
    }
    Ok(pushed)
}
```

並在 `sync/mod.rs` 加 `pub mod reconcile;`。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::reconcile 2>&1 | tail -5`
Expected: `13 passed`。若 `concurrent_edit…` 失敗,先確認 FakeRelay 的 conflict 判斷是 `current.seq > item.base_seq`(與 Worker 相同),再檢查 `take_remote` 的 `KeepLocal` 分支有更新 seq。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/reconcile.rs
git commit -m "feat(sync): plan/pull-merge/push round with sealed retention and batched uploads"
```

---

### Task 3: `sync::engine` —— 執行緒、SyncCore、指紋/generation 守衛、逐區塊時間、Tauri commands

**Files:**
- Create: `src-tauri/src/sync/engine.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod engine;`)
- Modify: `src-tauri/src/state.rs`(加 `pub sync: crate::sync::engine::SyncRuntime`)
- Modify: `src-tauri/src/config/commands.rs`(`load_doc_migrated` 改 `pub(crate)`;`persist_file` 成功後呼叫 `crate::sync::engine::note_file_written(&path)`)
- Modify: `src-tauri/src/lib.rs`(setup 呼叫 `sync::engine::initialize`;註冊 commands)

**Interfaces:**
- Consumes: A1 全部(含 `RelayClient::validate_url` —— 純驗證、不建 client)、Task 1–2、`crate::keys::ssh_dir()`、`crate::config::commands::{persist_file, load_doc_migrated}`、`crate::config::parser::parse_file`、`crate::config::model::Item`、`crate::fsutil::{Fingerprint, has_changed}`、`crate::tray::{tray_aliases, rebuild_tray}`
- Produces(ts-rs 匯出到 `src/bindings/`):
  - `pub struct SyncDevice { pub id: String, pub name: String, pub platform: String, pub joined_at_ms: u64, pub last_seen_ms: u64, pub is_this: bool }`
  - `pub struct SyncStatus { pub joined: bool, pub chain_short: Option<String>, pub device_id: String, pub device_name: String, pub relay_url: String, pub last_sync_ms: Option<u64>, pub last_error: Option<String>, pub pending: u64, pub read_only: bool, pub devices: Vec<SyncDevice>, pub managed_file: String, pub hosts_in_sync: u64, pub phrase_cleanup_pending: bool }`
  - `pub struct SyncCore { pub generation: u64, pub state: Option<SyncState>, pub keys: Option<ChainKeys> }`、`pub struct SyncRuntime { pub core: Mutex<SyncCore>, … }`、`pub struct BlockTracker`(逐區塊修改時間;`observe`/`changed_at`)
  - commands:`sync_status`、`sync_create_chain(device_name) -> String`(async)、`sync_join_chain(words, device_name) -> SyncStatus`(async)、`sync_show_words() -> String`、`sync_leave_chain(delete_remote: bool) -> SyncStatus`(async;未加入時只重試清 keychain)、`sync_now`、`sync_set_relay_url(url)`、`sync_set_device_name(name)`、`sync_forget_device(device_id)`
  - `pub fn initialize(app: &AppHandle) -> Result<(), AppError>`、`pub fn note_file_written(path: &Path)`、`pub fn wake()`
  - 事件:`sync://status`(payload `SyncStatus`)、`sync://conflict`(payload `Vec<String>` alias)、`sync://applied`(payload `usize`,本輪寫進受管檔的效果數;前端據此讓 config query cache 失效)

- [ ] **Step 1: 寫失敗的測試(可離線測的部分)**

建立 `src-tauri/src/sync/engine.rs`(先放 `use` 與測試):

```rust
//! 同步引擎:背景執行緒 + 一輪的順序(spec §6)+ `SyncCore`(generation/狀態/金鑰同一把鎖)+ 逐區塊
//! 修改時間 + 在指紋與 generation 守衛下把 reconcile 的效果套回 in-memory doc / 磁碟 + Tauri commands。
//! 網路與檔案的邊界:網路只在同步執行緒或 `spawn_blocking` 裡、不持有任何鎖;套用時才短暫鎖 doc。
//! 鎖順序固定:lifecycle → doc → core。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::mpsc::{self, Receiver, RecvTimeoutError, Sender};
use std::sync::{Mutex, OnceLock};
use std::time::{Duration, SystemTime, UNIX_EPOCH};

use serde::{Deserialize, Serialize};
use tauri::{AppHandle, Emitter, Manager};

use crate::config::commands::{load_doc_migrated, persist_file};
use crate::config::model::Item;
use crate::error::AppError;
use crate::fsutil::Fingerprint;
use crate::state::AppState;
use crate::sync::crypto::{self, ChainKeys};
use crate::sync::hosts_file::{self, HostBlockText};
use crate::sync::reconcile::{self, HostEffect};
use crate::sync::record::{record_key, DevicePayload, LocalRecord, MetaPayload, Record, RecordKind, SCHEMA_VERSION};
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
            HostEffect::Upsert { alias: "a".into(), text: "Host a\n  User y\n\n".into() },
            HostEffect::Delete { alias: "b".into() },
            HostEffect::Upsert { alias: "c".into(), text: "Host c\n".into() },
        ];
        let (changed, failed) = apply_effects_to_items(&mut items, &effects);
        assert!(changed);
        assert!(failed.is_empty());
        assert_eq!(serialize_items(&items, true), "Host a\n  User y\n\nHost c\n");
        assert_eq!(apply_effects_to_items(&mut items, &[]), (false, Vec::new()));
    }

    #[test]
    fn a_broken_effect_does_not_stop_the_others_and_is_reported_by_alias() {
        // 本機 `Host web *.internal` 擋住遠端合法的 `Host web`:回報 alias,呼叫端據此不讓它進快取。
        let (mut items, _) = parse_file("Host a\nHost web *.internal\n  User ops\n");
        let effects = vec![
            HostEffect::Upsert { alias: "web".into(), text: "Host web\n  User root\n".into() },
            HostEffect::Upsert { alias: "bad".into(), text: "# not a host\n".into() },
            HostEffect::Upsert { alias: "ok".into(), text: "Host ok\n".into() },
        ];
        let (changed, failed) = apply_effects_to_items(&mut items, &effects);
        assert!(changed);
        assert_eq!(failed, vec!["web".to_string(), "bad".to_string()]);
        let text = serialize_items(&items, true);
        assert!(text.contains("Host ok"));
        assert!(text.contains("Host web *.internal\n  User ops\n"), "local wildcard block untouched");
        assert!(!text.contains("User root"));
    }

    #[test]
    fn status_reflects_state_without_a_chain() {
        let mut s = SyncState::fresh("Box").unwrap();
        s.phrase_cleanup_pending = true; // Leave 時 keychain 刪不掉:持久化在狀態裡
        let status = status_from(&s, "/tmp/hosts.config");
        assert!(!status.joined);
        assert_eq!(status.device_name, "Box");
        assert_eq!(status.pending, 0);
        assert!(status.devices.is_empty());
        assert!(status.phrase_cleanup_pending);
        assert!(!status.read_only);
    }

    #[test]
    fn status_lists_devices_pending_counts_and_read_only() {
        let mut s = SyncState::fresh("Box").unwrap();
        s.chain_id = Some("ab".repeat(32));
        s.remote_schema_version = Some(SCHEMA_VERSION + 1);
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
        let status = status_from(&s, "/tmp/hosts.config");
        assert!(status.joined);
        assert_eq!(status.chain_short.as_deref(), Some("abababab"));
        assert_eq!(status.pending, 2);
        assert_eq!(status.hosts_in_sync, 1);
        assert_eq!(status.devices.len(), 1);
        assert!(status.devices[0].is_this);
        assert!(status.read_only);
        assert!(!status.phrase_cleanup_pending);
    }

    #[test]
    fn block_tracker_stamps_only_what_changed() {
        let mut t = BlockTracker::default();
        let a = HostBlockText { alias: "a".into(), text: "Host a\n".into() };
        let b = HostBlockText { alias: "b".into(), text: "Host b\n".into() };
        t.observe(&[a.clone(), b.clone()], 100);
        assert_eq!(t.changed_at("a"), Some(100));
        assert_eq!(t.changed_at("b"), Some(100));
        // 只有 b 變了:a 保留 100,b 變 200。
        let b2 = HostBlockText { alias: "b".into(), text: "Host b\n  User x\n".into() };
        t.observe(&[a.clone(), b2], 200);
        assert_eq!(t.changed_at("a"), Some(100));
        assert_eq!(t.changed_at("b"), Some(200));
        // 消失的 alias 也記時間(tombstone 用);沒動的仍不變。
        t.observe(&[a], 300);
        assert_eq!(t.changed_at("b"), Some(300));
        assert_eq!(t.changed_at("a"), Some(100));
        assert_eq!(t.changed_at("zzz"), None);
    }

    #[test]
    fn unapplied_aliases_are_rolled_back_to_the_pre_merge_cache() {
        let before = SyncState::fresh("A").unwrap();
        let mut before = before;
        let old = Record { kind: RecordKind::Host, id: "web".into(), version: 1, updated_at_ms: 1, device_id: "me".into(), deleted: false, payload: serde_json::json!({ "schema": 1, "text": "Host web\n" }) };
        before.records.insert(record_key(RecordKind::Host, "web"), LocalRecord { record: old.clone(), seq: 1, dirty: false });
        let mut next = before.clone();
        let mut newer = old.clone();
        newer.updated_at_ms = 9;
        next.records.insert(record_key(RecordKind::Host, "web"), LocalRecord { record: newer, seq: 5, dirty: false });
        next.records.insert(record_key(RecordKind::Host, "fresh"), LocalRecord { record: Record { id: "fresh".into(), ..old.clone() }, seq: 6, dirty: false });
        let message = drop_unapplied(&mut next, &before, &["web".to_string(), "fresh".to_string()]);
        assert!(message.is_some());
        // web 退回合併前的本機版本;fresh 本機原本沒有 → 移除。下一輪的 diff 因此不會產生 tombstone。
        assert_eq!(next.records["host:web"].record.updated_at_ms, 1);
        assert!(!next.records.contains_key("host:fresh"));
        assert!(drop_unapplied(&mut next, &before, &[]).is_none());
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::engine 2>&1 | tail -5`
Expected: 編譯錯誤(`apply_effects_to_items`/`status_from`/`BlockTracker` 未定義)。

- [ ] **Step 3: 實作 —— 型別、runtime、純函式**

在 `use` 之後、tests 之前:

```rust
const SYNC_INTERVAL: Duration = Duration::from_secs(45);
const SUPERSEDED: &str = "sync round superseded by a newer chain state";
const READ_ONLY_MESSAGE: &str = "this sync chain uses a newer format; update SSHelter to keep syncing";

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
    /// Leave 之後助記詞還留在 keychain(刪除失敗):UI 顯示警示與重試。
    pub phrase_cleanup_pending: bool,
}

/// generation / 狀態 / 金鑰永遠一起快照、一起替換(spec §6):分開鎖會出現「新 generation + 舊狀態」。
/// Create/Join/Leave 換 generation 時 +1;只動狀態的命令也 +1(`mutate_state`)。
pub struct SyncCore {
    pub generation: u64,
    pub state: Option<SyncState>,
    pub keys: Option<ChainKeys>,
}

/// Tauri 管理的同步執行期狀態。
pub struct SyncRuntime {
    pub core: Mutex<SyncCore>,
    /// Create/Join/Leave 全程互斥(含前置檢查、網路等待、keychain 讀寫;都在 spawn_blocking 裡):
    /// 舊 Leave 不可能刪掉新 Join 剛存的助記詞,兩個 Join 也不可能同時通過檢查。順序:lifecycle → doc → core。
    lifecycle: Mutex<()>,
    syncing: AtomicBool,
}

impl Default for SyncRuntime {
    fn default() -> Self {
        Self {
            core: Mutex::new(SyncCore { generation: 0, state: None, keys: None }),
            lifecycle: Mutex::new(()),
            syncing: AtomicBool::new(false),
        }
    }
}

/// 逐區塊的修改時間(spec §6):app 存檔時(`note_file_written`)用存檔當下;外部編輯在 gather 時用檔案
/// mtime(近似)。純資料、可測試;放全域是因為 `persist_file` 只有路徑、沒有 AppHandle。
#[derive(Default)]
pub struct BlockTracker {
    snapshot: BTreeMap<String, String>,
    changed_at: BTreeMap<String, u64>,
}

impl BlockTracker {
    /// 觀察目前的區塊:文字變了 / 新出現 / 消失的 alias 記為 `stamp_ms`;沒變的保留原時間。
    pub fn observe(&mut self, blocks: &[HostBlockText], stamp_ms: u64) {
        let current: BTreeMap<String, String> = blocks.iter().map(|b| (b.alias.clone(), b.text.clone())).collect();
        for (alias, text) in &current {
            if self.snapshot.get(alias) != Some(text) {
                self.changed_at.insert(alias.clone(), stamp_ms);
            }
        }
        for alias in self.snapshot.keys() {
            if !current.contains_key(alias) {
                self.changed_at.insert(alias.clone(), stamp_ms);
            }
        }
        self.snapshot = current;
    }

    pub fn changed_at(&self, alias: &str) -> Option<u64> {
        self.changed_at.get(alias).copied()
    }
}

static TRACKER: OnceLock<Mutex<BlockTracker>> = OnceLock::new();

fn tracker() -> &'static Mutex<BlockTracker> {
    TRACKER.get_or_init(|| Mutex::new(BlockTracker::default()))
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

fn now_ms() -> u64 {
    SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_millis() as u64).unwrap_or(0)
}

/// `persist_file` 寫完任何檔案後呼叫:只有受管同步檔才會觸發。存檔當下就逐區塊記下修改時間 ——
/// 呼叫端(config commands)正持有 doc 鎖,這裡不能再鎖 doc,所以直接讀剛寫好的檔案。
pub fn note_file_written(path: &Path) {
    let Ok(ssh_dir) = crate::keys::ssh_dir() else { return };
    if path != hosts_file::managed_path(&ssh_dir) {
        return;
    }
    if let Ok(text) = std::fs::read_to_string(path) {
        let (items, _) = crate::config::parser::parse_file(&text);
        tracker().lock().unwrap().observe(&hosts_file::blocks_of(&items), now_ms());
    }
    wake();
}

fn superseded() -> AppError {
    AppError::Other(SUPERSEDED.to_string())
}

/// 把 reconcile 的效果套到區塊列表。回傳(是否改了任何東西, 套不上的 alias);壞掉的效果不中斷其他效果。
pub fn apply_effects_to_items(items: &mut Vec<Item>, effects: &[HostEffect]) -> (bool, Vec<String>) {
    let mut changed = false;
    let mut failed = Vec::new();
    for effect in effects {
        let result = match effect {
            HostEffect::Upsert { alias, text } => hosts_file::apply_host_text(items, alias, text),
            HostEffect::Delete { alias } => Ok(hosts_file::remove_host_block(items, alias)),
        };
        match result {
            Ok(c) => changed |= c,
            Err(_) => failed.push(effect.alias().to_string()),
        }
    }
    (changed, failed)
}

/// 套不上的 alias 不得進快取(spec §6):記錄退回合併前的本機版本(本機原本沒有就移除)。
/// 否則下一輪 diff 會把它當成「快取有、檔案沒有」而產生 tombstone,把別台的主機刪掉。回傳要放進
/// `last_error` 的訊息(只有數量,不含主機名)。
fn drop_unapplied(next: &mut SyncState, before: &SyncState, failed: &[String]) -> Option<String> {
    if failed.is_empty() {
        return None;
    }
    for alias in failed {
        let key = record_key(RecordKind::Host, alias);
        match before.records.get(&key) {
            Some(previous) => {
                next.records.insert(key, previous.clone());
            }
            None => {
                next.records.remove(&key);
            }
        }
    }
    Some(format!(
        "{} synced host(s) could not be applied: a local block with the same name contains wildcard patterns",
        failed.len()
    ))
}

pub fn status_from(state: &SyncState, managed_file: &str) -> SyncStatus {
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
        read_only: state.read_only(),
        devices,
        managed_file: managed_file.to_string(),
        hosts_in_sync: state.records.values().filter(|l| l.record.kind == RecordKind::Host && !l.record.deleted).count() as u64,
        phrase_cleanup_pending: state.phrase_cleanup_pending,
    }
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test sync::engine 2>&1 | tail -5`
Expected: `6 passed`。

- [ ] **Step 5: 實作 —— 啟動、執行緒、一輪的順序與檔案邊界**

接在 `status_from` 之後加入:

```rust
fn managed_path() -> Result<PathBuf, AppError> {
    Ok(hosts_file::managed_path(&crate::keys::ssh_dir()?))
}

/// 確保受管檔存在、主 config 有 Include(置頂)、且 doc 已載入受管檔。回傳受管檔路徑。
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

/// gather 的結果:區塊 + 當時的檔案指紋(套用前要再比一次)+ 檔案 mtime(外部編輯的時間戳)。
struct Gathered {
    blocks: Vec<HostBlockText>,
    fingerprint: Fingerprint,
    modified_ms: u64,
}

/// 取出受管檔目前的區塊。磁碟若已被手改(指紋不同)先重載,避免用過期的 in-memory 內容。
fn gather_blocks(app: &AppHandle, managed: &Path) -> Result<Gathered, AppError> {
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
    // 外部編輯的時間戳 = 檔案 mtime(整檔的近似值);拿不到就退回現在。
    let modified_ms = std::fs::metadata(managed)
        .and_then(|m| m.modified())
        .ok()
        .and_then(|t| t.duration_since(UNIX_EPOCH).ok())
        .map(|d| d.as_millis() as u64)
        .unwrap_or_else(now_ms);
    let blocks = hosts_file::blocks_of(&doc.files[idx].items);
    // app 自己存檔的區塊已在 note_file_written 記過時間、文字沒變就保留;只有外部編輯改到的區塊會拿到 mtime。
    tracker().lock().unwrap().observe(&blocks, modified_ms);
    Ok(Gathered { blocks, fingerprint: doc.files[idx].fingerprint.clone(), modified_ms })
}

enum Applied {
    /// 套用完成;`wrote` = 真的寫了檔案;`failed` = 套不上的 alias,呼叫端不得讓它們進快取。
    Done { wrote: bool, failed: Vec<String> },
    /// 受管檔在 gather 之後變過(UI 存檔、外部編輯或 persist 的 Conflict):本輪作廢。
    FileChanged,
}

/// 在指紋與 generation 守衛下把效果寫進受管檔(經 `persist_file`)。效果先套到 items 的副本;
/// 持久化失敗時先退回舊區塊、再從磁碟重載,不留下與磁碟不一致的 doc。
fn apply_effects(
    app: &AppHandle,
    managed: &Path,
    generation: u64,
    gathered: &Fingerprint,
    effects: &[HostEffect],
) -> Result<Applied, AppError> {
    if effects.is_empty() {
        return Ok(Applied::Done { wrote: false, failed: Vec::new() });
    }
    let state = app.state::<AppState>();
    let mut doc_lock = state.doc.lock().unwrap();
    // lifecycle 命令在 doc 鎖內換 generation/狀態/金鑰:拿到 doc 鎖後再比,舊 keys 的合併結果就不可能被寫進檔案。
    if state.sync.core.lock().unwrap().generation != generation {
        return Err(superseded());
    }
    let mut backed_up = state.backed_up.lock().unwrap();
    let retention = *state.backup_retention.lock().unwrap();
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let idx = doc
        .files
        .iter()
        .position(|f| f.path == managed)
        .ok_or_else(|| AppError::Other("synced hosts file is not loaded".to_string()))?;
    // in-memory 指紋不同 = app 自己在網路期間存過檔;磁碟指紋不同 = 外部編輯。兩者都丟棄本輪。
    if doc.files[idx].fingerprint != *gathered
        || crate::fsutil::has_changed(managed, &doc.files[idx].fingerprint).unwrap_or(true)
    {
        return Ok(Applied::FileChanged);
    }
    let mut items = doc.files[idx].items.clone();
    let (changed, failed) = apply_effects_to_items(&mut items, effects);
    if !failed.is_empty() {
        // 只印數量,不印內容。
        eprintln!("[sync] {} remote host record(s) could not be applied to the local file", failed.len());
    }
    if !changed {
        return Ok(Applied::Done { wrote: false, failed });
    }
    let original = std::mem::replace(&mut doc.files[idx].items, items);
    if let Err(e) = persist_file(doc, idx, &mut backed_up, retention) {
        // 任何失敗:先把 in-memory 區塊退回舊版,再從磁碟重載讓兩邊一致 —— 磁碟可能已是新內容
        // (atomic_write 成功但 fingerprint 失敗),也可能仍是舊內容。重載也失敗就只保留舊區塊並回報。
        doc.files[idx].items = original;
        let main_path = doc.files[0].path.clone();
        if let Err(reload) = load_doc_migrated(&main_path).map(|fresh| *doc_lock = Some(fresh)) {
            return Err(AppError::Other(format!("{e}; reloading the config afterwards also failed: {reload}")));
        }
        return match e {
            AppError::Conflict(_) => Ok(Applied::FileChanged),
            other => Err(other),
        };
    }
    let aliases = crate::tray::tray_aliases(doc);
    let _ = crate::tray::rebuild_tray(app, &aliases);
    Ok(Applied::Done { wrote: true, failed })
}

/// 持久化 core 裡的狀態(呼叫端已持有 core 鎖)。
fn persist_core(core: &SyncCore) -> Result<(), AppError> {
    if let Some(s) = core.state.as_ref() {
        sync_state::save(&sync_state::state_path()?, s)?;
    }
    Ok(())
}

fn save_state(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    let core = state.sync.core.lock().unwrap();
    persist_core(&core)
}

fn emit_status(app: &AppHandle) {
    let state = app.state::<AppState>();
    let core = state.sync.core.lock().unwrap();
    if let Some(s) = core.state.as_ref() {
        let managed = managed_path().map(|p| p.to_string_lossy().into_owned()).unwrap_or_default();
        let _ = app.emit("sync://status", &status_from(s, &managed));
    }
}

/// 同步輪次回寫狀態的唯一入口:generation 變了(lifecycle 或狀態命令搶先)就丟棄,絕不覆蓋。
fn commit_state(app: &AppHandle, generation: u64, s: &SyncState) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    let mut core = state.sync.core.lock().unwrap();
    if core.generation != generation {
        return Err(superseded());
    }
    core.state = Some(s.clone());
    persist_core(&core)
}

/// 一輪(spec §6 的順序)。回 `SUPERSEDED` 表示被 lifecycle / 狀態命令搶先,不是錯誤。
fn run_round(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    // generation / 狀態 / 金鑰一次快照(同一把鎖):不可能拿到「新 generation + 舊狀態」。
    let (generation, mut s, keys) = {
        let core = state.sync.core.lock().unwrap();
        match (core.state.as_ref(), core.keys.as_ref()) {
            (Some(s), Some(k)) if s.joined() => (core.generation, s.clone(), k.clone()),
            _ => return Ok(()),
        }
    };
    let managed = ensure_managed_loaded(app)?;
    let gathered = gather_blocks(app, &managed)?;
    let now = now_ms();
    let platform = std::env::consts::OS;
    let relay = RelayClient::new(&s.relay_url, &keys.auth_token)?;

    // 0. 基線輪(剛 Join):不做本機 diff,先以 chain 為準套用 —— chain 上已 tombstone、本機同步檔卻還
    //    留著的區塊會被移除(先備份)。否則 Leave 後保留的舊區塊會以「現在」的時間戳復活遠端的刪除。
    //    成功後立刻再跑一輪,本機獨有的區塊才當新主機上傳。
    if !s.baseline_established {
        let merged = reconcile::pull_merge(&s, &keys, &relay)?;
        return match apply_effects(app, &managed, generation, &gathered.fingerprint, &merged.host_effects)? {
            Applied::FileChanged => {
                wake();
                Ok(())
            }
            Applied::Done { wrote, failed } => {
                let mut next = merged.state;
                next.last_error = drop_unapplied(&mut next, &s, &failed);
                next.baseline_established = true;
                next.last_sync_ms = Some(now);
                commit_state(app, generation, &next)?;
                if wrote {
                    let _ = app.emit("sync://applied", &merged.host_effects.len());
                }
                wake();
                Ok(())
            }
        };
    }

    // 1. 本機變更(逐區塊時間:app 存檔當下 / 外部編輯的 mtime)先持久化。
    let changed = {
        let times = tracker().lock().unwrap();
        let fallback = gathered.modified_ms.min(now);
        reconcile::plan_local(&mut s, &gathered.blocks, |alias| times.changed_at(alias).unwrap_or(fallback), now, platform)
    };
    if changed > 0 {
        commit_state(app, generation, &s)?;
    }

    // 2. 網路:不持有任何鎖。
    let merged = reconcile::pull_merge(&s, &keys, &relay)?;

    // 3. 套用:受管檔若在網路期間變了,整輪 pull/merge 丟棄(cursor 不前進),立刻再跑一輪。
    let unapplied = match apply_effects(app, &managed, generation, &gathered.fingerprint, &merged.host_effects)? {
        Applied::FileChanged => {
            wake();
            return Ok(());
        }
        Applied::Done { wrote, failed } => {
            let before = std::mem::replace(&mut s, merged.state);
            let message = drop_unapplied(&mut s, &before, &failed);
            commit_state(app, generation, &s)?;
            if wrote {
                let _ = app.emit("sync://applied", &merged.host_effects.len());
            }
            if !merged.conflicts.is_empty() {
                let _ = app.emit("sync://conflict", &merged.conflicts);
            }
            message
        }
    };

    // 4. 推送(唯讀模式內部略過)。部分成功也要持久化,所以先 commit 再看結果。
    let pushed = reconcile::push_dirty(&mut s, &keys, &relay);
    s.last_sync_ms = Some(now);
    if pushed.is_ok() {
        s.last_error = if s.read_only() { Some(READ_ONLY_MESSAGE.to_string()) } else { unapplied };
    }
    commit_state(app, generation, &s)?;
    if pushed?.conflicts > 0 {
        wake(); // 下一輪 pull 會拿到中繼版本再合併。
    }
    Ok(())
}

/// 一輪同步。錯誤寫進 last_error 並發事件,永不 panic;被搶先(SUPERSEDED)不算錯誤。
pub fn sync_once(app: &AppHandle) -> Result<(), AppError> {
    let state = app.state::<AppState>();
    if state.sync.syncing.swap(true, Ordering::SeqCst) {
        return Ok(()); // 已在同步中
    }
    let result = run_round(app);
    state.sync.syncing.store(false, Ordering::SeqCst);
    match &result {
        Err(AppError::Other(m)) if m == SUPERSEDED => {}
        Err(e) => {
            let mut core = state.sync.core.lock().unwrap();
            if let Some(s) = core.state.as_mut() {
                s.last_error = Some(e.to_string());
            }
            let _ = persist_core(&core);
        }
        Ok(()) => {}
    }
    emit_status(app);
    result
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
    let mut loaded = match sync_state::state_path().and_then(|p| sync_state::load(&p)) {
        Ok(Some(s)) => s,
        Ok(None) => SyncState::fresh(&default_device_name())?,
        Err(e) => {
            let mut s = SyncState::fresh(&default_device_name())?;
            s.last_error = Some(e.to_string());
            s
        }
    };
    let mut keys = None;
    if loaded.joined() {
        match sync_state::load_mnemonic() {
            Ok(Some(words)) => keys = crypto::derive_keys(&words).ok(),
            _ => loaded.last_error = Some("recovery phrase is missing from the keychain; leave and rejoin the chain".to_string()),
        }
    }
    {
        let mut core = state.sync.core.lock().unwrap();
        core.state = Some(loaded);
        core.keys = keys;
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
    let mut core = state.sync.core.lock().unwrap();
    let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    f(s)
}

/// 只動狀態的命令:在同一個臨界區「換 generation + 改狀態 + 持久化」。在途輪次的整份狀態副本
/// 會因 generation 不同而被丟棄,不會蓋掉這裡的修改。
fn mutate_state<T>(app: &AppHandle, f: impl FnOnce(&mut SyncState) -> Result<T, AppError>) -> Result<T, AppError> {
    let state = app.state::<AppState>();
    let mut core = state.sync.core.lock().unwrap();
    core.generation += 1;
    let out = {
        let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        f(s)?
    };
    persist_core(&core)?;
    Ok(out)
}

fn current_status(app: &AppHandle) -> Result<SyncStatus, AppError> {
    let managed = managed_path()?.to_string_lossy().into_owned();
    with_state(app, |s| Ok(status_from(s, &managed)))
}

/// spawn_blocking 的 JoinHandle 錯誤(執行緒被取消等)→ AppError。
fn join_error(e: tauri::Error) -> AppError {
    AppError::Other(format!("sync task failed: {e}"))
}

#[derive(Clone, Copy)]
enum ChainEntry {
    Create,
    Join,
}

/// 建立或加入 chain 的共同流程。**呼叫端已持有 lifecycle 鎖**(全程,含網路與 keychain)。
/// 驗證/建立 → 存助記詞 → 在 doc 鎖內、同一個 core 臨界區換 generation/狀態/金鑰 → 準備受管檔 → 喚醒。
fn enter_chain(app: &AppHandle, words: &str, device_name: &str, mode: ChainEntry) -> Result<SyncStatus, AppError> {
    let keys = crypto::derive_keys(words)?;
    let relay_url = with_state(app, |s| Ok(s.relay_url.clone()))?;
    let relay = RelayClient::new(&relay_url, &keys.auth_token)?;
    match mode {
        // 只有 Create 用 PUT(冪等建立)。
        ChainEntry::Create => relay.create_chain(&keys.chain_id)?,
        // Join 只驗證:404 = 這組助記詞沒有對應的 chain。絕不 PUT —— 否則任何 checksum 正確的
        // 助記詞都會靜靜建出一條新 chain,而不是回報錯誤。
        ChainEntry::Join => {
            relay.pull(&keys.chain_id, 0).map_err(|e| match e {
                AppError::NotFound(_) => AppError::NotFound(
                    "no sync chain matches this recovery phrase (check the words and the relay URL)".to_string(),
                ),
                other => other,
            })?;
        }
    }
    sync_state::store_mnemonic(words)?;
    let now = now_ms();
    let state = app.state::<AppState>();
    {
        // doc 鎖:與 apply_effects 的寫檔互斥;core 鎖:generation/狀態/金鑰一次換掉,不會有「新 generation +
        // 舊狀態」的交錯(spec §6)。
        let _doc = state.doc.lock().unwrap();
        let mut core = state.sync.core.lock().unwrap();
        core.generation += 1;
        let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        s.chain_id = Some(keys.chain_id.clone());
        s.device_name = device_name.trim().to_string();
        s.cursor_seq = 0;
        s.records.clear();
        s.sealed.clear();
        s.remote_schema_version = None;
        // Create 的 chain 是空的,基線輪沒有意義;Join 要先以 chain 為準(spec §6 基線輪)。
        s.baseline_established = matches!(mode, ChainEntry::Create);
        s.phrase_cleanup_pending = false;
        s.last_sync_ms = None;
        s.last_error = None;
        let me = reconcile::own_device_record(s, now, std::env::consts::OS);
        s.records.insert(record_key(RecordKind::Device, &s.device_id), LocalRecord { record: me, seq: 0, dirty: true });
        if matches!(mode, ChainEntry::Create) {
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
        core.keys = Some(keys);
        persist_core(&core)?;
    }
    ensure_managed_loaded(app)?;
    wake();
    current_status(app)
}

/// 離開 chain。**呼叫端已持有 lifecycle 鎖**。先作廢在途輪次並清掉 chain 狀態(確保停止同步),再刪
/// keychain;keychain 刪不掉要回報並持久化重試旗標,不得宣稱已清乾淨。未加入時只做 keychain 重試。
fn leave_chain(app: &AppHandle, delete_remote: bool) -> Result<SyncStatus, AppError> {
    let state = app.state::<AppState>();
    let (joined, chain, keys, relay_url) = {
        let core = state.sync.core.lock().unwrap();
        let s = core.state.as_ref();
        (
            s.is_some_and(|s| s.joined()),
            s.and_then(|s| s.chain_id.clone()),
            core.keys.clone(),
            s.map(|s| s.relay_url.clone()).unwrap_or_default(),
        )
    };
    if joined {
        if delete_remote {
            if let (Some(chain), Some(keys)) = (chain, keys) {
                RelayClient::new(&relay_url, &keys.auth_token)?.delete_chain(&chain)?;
            }
        }
        // doc 鎖內、同一個 core 臨界區:在途的舊輪次不可能再寫檔、也不可能把已離開的 chain 放回來。
        let _doc = state.doc.lock().unwrap();
        let mut core = state.sync.core.lock().unwrap();
        core.generation += 1;
        core.keys = None;
        if let Some(s) = core.state.as_mut() {
            s.chain_id = None;
            s.cursor_seq = 0;
            s.records.clear();
            s.sealed.clear();
            s.remote_schema_version = None;
            s.baseline_established = false;
            s.last_sync_ms = None;
            s.last_error = None;
        }
        persist_core(&core)?;
    }
    // keychain 清理(仍在 lifecycle 鎖內:新 Join 不可能穿插)。結果持久化,重啟後警示與重試入口仍在。
    let cleared = sync_state::clear_mnemonic();
    with_state(app, |s| {
        s.phrase_cleanup_pending = cleared.is_err();
        Ok(())
    })?;
    save_state(app)?;
    emit_status(app);
    if let Err(e) = cleared {
        return Err(AppError::Other(format!(
            "left the sync chain, but the recovery phrase could not be removed from the keychain ({e}); use \"Remove phrase\" to retry"
        )));
    }
    current_status(app)
}

#[tauri::command]
pub fn sync_status(app: AppHandle) -> Result<SyncStatus, AppError> {
    current_status(&app)
}

/// 網路在 spawn_blocking 裡(`reqwest::blocking` 不能在 tokio runtime 內呼叫);lifecycle 鎖涵蓋前置檢查。
#[tauri::command]
pub async fn sync_create_chain(app: AppHandle, device_name: String) -> Result<String, AppError> {
    let handle = app.clone();
    tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        let _lifecycle = state.sync.lifecycle.lock().unwrap();
        if with_state(&handle, |s| Ok(s.joined()))? {
            return Err(AppError::Other("already in a sync chain; leave it first".to_string()));
        }
        let words = crypto::generate_mnemonic()?;
        enter_chain(&handle, &words, &device_name, ChainEntry::Create)?;
        Ok(words)
    })
    .await
    .map_err(join_error)?
}

#[tauri::command]
pub async fn sync_join_chain(app: AppHandle, words: String, device_name: String) -> Result<SyncStatus, AppError> {
    let normalized = crypto::normalize_mnemonic(&words)?;
    let handle = app.clone();
    tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        let _lifecycle = state.sync.lifecycle.lock().unwrap();
        if with_state(&handle, |s| Ok(s.joined()))? {
            return Err(AppError::Other("already in a sync chain; leave it first".to_string()));
        }
        enter_chain(&handle, &normalized, &device_name, ChainEntry::Join)
    })
    .await
    .map_err(join_error)?
}

#[tauri::command]
pub fn sync_show_words(app: AppHandle) -> Result<String, AppError> {
    if !with_state(&app, |s| Ok(s.joined()))? {
        return Err(AppError::Other("not in a sync chain".to_string()));
    }
    sync_state::load_mnemonic()?.ok_or_else(|| AppError::Other("recovery phrase is not in the keychain".to_string()))
}

#[tauri::command]
pub async fn sync_leave_chain(app: AppHandle, delete_remote: bool) -> Result<SyncStatus, AppError> {
    let handle = app.clone();
    tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        let _lifecycle = state.sync.lifecycle.lock().unwrap();
        leave_chain(&handle, delete_remote)
    })
    .await
    .map_err(join_error)?
}

#[tauri::command]
pub fn sync_now(app: AppHandle) -> Result<(), AppError> {
    wake();
    let _ = app;
    Ok(())
}

#[tauri::command]
pub fn sync_set_relay_url(app: AppHandle, url: String) -> Result<SyncStatus, AppError> {
    let normalized = RelayClient::validate_url(&url)?; // 純驗證,不建 client
    mutate_state(&app, |s| {
        s.relay_url = normalized;
        s.last_error = None;
        Ok(())
    })?;
    wake();
    current_status(&app)
}

#[tauri::command]
pub fn sync_set_device_name(app: AppHandle, name: String) -> Result<SyncStatus, AppError> {
    let name = name.trim().to_string();
    if name.is_empty() {
        return Err(AppError::Other("device name cannot be empty".to_string()));
    }
    let now = now_ms();
    mutate_state(&app, |s| {
        s.device_name = name;
        if s.joined() {
            let me = reconcile::own_device_record(s, now, std::env::consts::OS);
            let key = record_key(RecordKind::Device, &s.device_id);
            let seq = s.records.get(&key).map(|l| l.seq).unwrap_or(0);
            s.records.insert(key, LocalRecord { record: me, seq, dirty: true });
        }
        Ok(())
    })?;
    wake();
    current_status(&app)
}

/// 只把裝置從清單移除(tombstone 它的 device 記錄)。**不是撤權**:它若還有助記詞就會繼續同步
/// (spec §2);UI 文案必須如此說明。
#[tauri::command]
pub fn sync_forget_device(app: AppHandle, device_id: String) -> Result<SyncStatus, AppError> {
    let now = now_ms();
    mutate_state(&app, |s| {
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
    wake();
    current_status(&app)
}
```

- [ ] **Step 7: 接線**

1. `src-tauri/src/sync/mod.rs` 加 `pub mod engine;`。
2. `src-tauri/src/state.rs`:`AppState` 加欄位 `pub sync: crate::sync::engine::SyncRuntime,`,`Default` 加 `sync: crate::sync::engine::SyncRuntime::default(),`。
3. `src-tauri/src/config/commands.rs`:`fn load_doc_migrated` 改為 `pub(crate) fn load_doc_migrated`;在 `persist_file` 成功寫入後(函式回傳 `Ok(())` 之前)加 `crate::sync::engine::note_file_written(&path);`(`path` 為該函式開頭 clone 的檔案路徑)。
4. `src-tauri/src/lib.rs`:`use sync::engine::{sync_create_chain, sync_forget_device, sync_join_chain, sync_leave_chain, sync_now, sync_set_device_name, sync_set_relay_url, sync_show_words, sync_status};`,`generate_handler!` 清單加入這九個;`.setup` 內 `mcp::initialize(...)?;` 之後加 `sync::engine::initialize(app.handle())?;`。

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result'` 並 `ls ../src/bindings/ | grep -i sync`
Expected: 全綠;`SyncStatus.ts`、`SyncDevice.ts` 已生成(`SyncStatus.ts` 含 `phrase_cleanup_pending: boolean`)。

- [ ] **Step 8: 手動 smoke(需 A2 的 relay 在本機跑)**

```bash
cd relay && npm run dev        # 127.0.0.1:8787
pnpm tauri dev                 # 另一個終端
```

在 devtools console 執行:
- `await window.__TAURI__.core.invoke("sync_create_chain", { deviceName: "dev" })` → 回 24 詞(不得 panic —— 這條路徑驗證 spawn_blocking 包住了 blocking reqwest);`invoke("sync_status")` → `joined: true`、`devices` 含本機;`~/.ssh/config` **第一個非註解行**是 `Include ~/.ssh/sshelter/hosts.config`。
- 把任一 host 用既有 Move 搬進 `hosts.config` 後幾秒內 `pending` 歸零。
- `invoke("sync_leave_chain", { deleteRemote: false })` 後,用另一組**合法但不同**的 24 詞 `invoke("sync_join_chain", …)` → 錯誤訊息含 "no sync chain matches",且 relay 的 `.wrangler/state` 沒有多出 chain。
- 同步期間(關掉 relay 讓 pull 卡在 timeout)在 UI 改一台同步主機並存檔 → relay 回來後那筆修改仍在(本輪被 FileChanged 丟棄後重跑)。
- 基線輪:Leave(不刪 relay)→ 用另一個 OS 使用者/裝置刪掉一台同步主機 → 本機 Join 回來 → 那台主機從本機 `hosts.config` 消失(備份在 backups 目錄),本機獨有的主機則在下一輪上傳。
- 並行 lifecycle:快速連按兩次 Join(devtools 同時 `invoke` 兩次)→ 只有一個成功,另一個回 "already in a sync chain";Leave 後立刻 Join → 重啟 app 後 `sync_show_words` 仍能回同一組詞(舊 Leave 沒有刪掉新 Join 的助記詞)。

- [ ] **Step 9: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/engine.rs src-tauri/src/state.rs src-tauri/src/config/commands.rs src-tauri/src/lib.rs src/bindings/SyncStatus.ts src/bindings/SyncDevice.ts
git commit -m "feat(sync): background engine with atomic core snapshots, lifecycle mutex and per-block change times"
```

---

### Task 4: 主機遷入、重複 alias 偵測與檔案定位的處理

**Files:**
- Create: `src-tauri/src/sync/migrate.rs`
- Modify: `src-tauri/src/sync/mod.rs`(加 `pub mod migrate;`)
- Modify: `src-tauri/src/config/dto.rs`(`fn parse_tags` → `pub(crate) fn parse_tags`)
- Modify: `src-tauri/src/lib.rs`(註冊 `sync_migrate_hosts`、`sync_duplicate_aliases`、`sync_resolve_shadowed`)

**Interfaces:**
- Consumes: `crate::config::commands::{move_host, persist_file, validate_host_patterns}`、`crate::config::edit::{find_host_mut, set_tags, set_host_patterns}`、`crate::config::dto::parse_tags`、`crate::config::include::find_host_file_index`、`crate::sync::hosts_file::is_syncable_block`
- Produces:
  - `pub fn tag_for_file(path: &Path) -> String`
  - `pub fn refuse_wildcard(doc: &SshConfigDoc, alias: &str) -> Result<(), AppError>`(遷入前:以**與 `move_host` 相同的定位規則**找到會被搬的區塊,任一 pattern 含 wildcard → Err)
  - `pub fn duplicate_aliases(doc: &SshConfigDoc, managed: &Path) -> Vec<DuplicateAlias>`
  - `pub struct DuplicateAlias { pub alias: String, pub local_file: String }`(ts-rs)
  - `pub enum ShadowedAction { Rename, Remove }`(serde 小寫:`"rename" | "remove"`)
  - `pub fn resolve_shadowed(doc: &mut SshConfigDoc, alias: &str, file: &str, action: ShadowedAction, managed: &Path) -> Result<usize, AppError>`(回傳改動的檔案索引;**以檔案路徑定位**,拒絕碰同步檔那份)
  - `pub struct MigrationFailure { pub alias: String, pub error: String }`、`pub struct MigrationReport { pub moved: Vec<String>, pub failed: Vec<MigrationFailure>, pub tagged: u64 }`(ts-rs)
  - commands:`sync_migrate_hosts(aliases: Vec<String>, tag_by_file: bool) -> MigrationReport`、`sync_duplicate_aliases() -> Vec<DuplicateAlias>`、`sync_resolve_shadowed(alias: String, file: String, action: ShadowedAction) -> Vec<DuplicateAlias>`(處理後回傳剩餘的重複清單)

- [ ] **Step 1: 寫失敗的測試**

建立 `src-tauri/src/sync/migrate.rs`:

```rust
//! 既有主機遷入受管同步檔(spec §10):批次 move + 以原檔名上 tag;加入 chain 後「同步檔與本地同名
//! 主機」的偵測;以及**以檔案路徑定位**的處理(既有 `config_rename_host`/`config_remove_host`
//! 以第一個命中為準,會依 Include 順序誤中同步檔那份,不能用)。

use std::path::Path;

use serde::{Deserialize, Serialize};
use tauri::{AppHandle, Manager};

use crate::config::commands::{move_host, persist_file, validate_host_patterns};
use crate::config::dto::parse_tags;
use crate::config::edit::{find_host_mut, set_host_patterns, set_tags};
use crate::config::include::find_host_file_index;
use crate::config::model::{Item, SshConfigDoc};
use crate::error::AppError;
use crate::state::AppState;
use crate::sync::hosts_file::is_syncable_block;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::include::load_doc;
    use crate::config::serialize::serialize_items;

    /// 主 config Include 受管檔;兩邊都有 `web`,主 config 另有 `local-only`。
    fn fixture() -> (tempfile::TempDir, std::path::PathBuf, std::path::PathBuf) {
        let dir = tempfile::tempdir().unwrap();
        let managed = dir.path().join("hosts.config");
        std::fs::write(&managed, "Host web\n  HostName 1\nHost only-synced\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {}\nHost web\n  HostName 2\nHost local-only\n", managed.display())).unwrap();
        (dir, main, managed)
    }

    #[test]
    fn tag_for_file_strips_extensions_and_normalizes() {
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/homelab.config")), "homelab");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/Work Stuff.conf")), "work-stuff");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config")), "config");
    }

    #[test]
    fn duplicates_are_aliases_defined_both_in_managed_and_elsewhere() {
        let (_dir, main, managed) = fixture();
        let doc = load_doc(&main).unwrap();
        let dups = duplicate_aliases(&doc, &managed);
        assert_eq!(dups.len(), 1);
        assert_eq!(dups[0].alias, "web");
        assert!(dups[0].local_file.ends_with("config"));
    }

    #[test]
    fn migration_refuses_the_block_move_host_would_actually_pick() {
        let dir = tempfile::tempdir().unwrap();
        let main = dir.path().join("config");
        // `move_host` 挑的是「任一 pattern 相符」的第一個區塊:遷移 `web` 會搬到含 wildcard 的第一個,不是
        // 後面那個乾淨的 `Host web`。資格檢查必須用同一條定位規則。
        std::fs::write(&main, "Host other web *.internal\n  User ops\nHost web\nHost db\n").unwrap();
        let doc = load_doc(&main).unwrap();
        assert!(refuse_wildcard(&doc, "web").is_err());
        assert!(refuse_wildcard(&doc, "other").is_err());
        assert!(refuse_wildcard(&doc, "db").is_ok());
        assert!(refuse_wildcard(&doc, "ghost").is_ok(), "move_host reports unknown aliases itself");
    }

    #[test]
    fn resolve_shadowed_only_touches_the_named_local_file() {
        let (_dir, main, managed) = fixture();
        let main_str = main.to_string_lossy().into_owned();
        let mut doc = load_doc(&main).unwrap();
        // rename:主 config 那份改成 web-local,同步檔那份原封不動。
        let idx = resolve_shadowed(&mut doc, "web", &main_str, ShadowedAction::Rename, &managed).unwrap();
        assert_eq!(idx, 0);
        let main_text = serialize_items(&doc.files[0].items, true);
        assert!(main_text.contains("Host web-local\n  HostName 2\n"));
        assert!(!main_text.contains("Host web\n"));
        let managed_idx = doc.files.iter().position(|f| f.path == managed).unwrap();
        assert!(serialize_items(&doc.files[managed_idx].items, true).contains("Host web\n  HostName 1\n"));
        assert!(duplicate_aliases(&doc, &managed).is_empty());
        // 再 rename 一次會撞名(web-local 已存在)→ 錯,且不能改成同步檔。
        let mut doc2 = load_doc(&main).unwrap();
        std::fs::write(&main, format!("Include {}\nHost web\nHost web-local\n", managed.display())).unwrap();
        let mut doc3 = load_doc(&main).unwrap();
        assert!(resolve_shadowed(&mut doc3, "web", &main_str, ShadowedAction::Rename, &managed).is_err());
        assert!(resolve_shadowed(&mut doc2, "web", &managed.to_string_lossy(), ShadowedAction::Remove, &managed).is_err());
        // remove:只刪主 config 那份。
        let removed = resolve_shadowed(&mut doc2, "web", &main_str, ShadowedAction::Remove, &managed).unwrap();
        assert_eq!(removed, 0);
        assert!(!serialize_items(&doc2.files[0].items, true).contains("Host web\n"));
        assert!(serialize_items(&doc2.files[managed_idx].items, true).contains("Host web\n"));
        // 不存在的檔案 / alias。
        assert!(resolve_shadowed(&mut doc2, "web", "/nope/config", ShadowedAction::Remove, &managed).is_err());
        assert!(resolve_shadowed(&mut doc2, "ghost", &main_str, ShadowedAction::Remove, &managed).is_err());
    }
}
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `cd src-tauri && cargo test sync::migrate 2>&1 | tail -5`
Expected: 編譯錯誤。

- [ ] **Step 3: 實作**

先把 `src-tauri/src/config/dto.rs` 的 `fn parse_tags(body: &[Item]) -> Vec<String>` 改成 `pub(crate) fn parse_tags(...)`。

在 `migrate.rs` 的 `use` 之後、tests 之前:

```rust
#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct DuplicateAlias {
    pub alias: String,
    pub local_file: String,
}

#[derive(Clone, Copy, Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum ShadowedAction {
    /// 本地那份改名 `<alias>-local`(保留本地定義)。
    Rename,
    /// 移除本地那份(改用同步版)。
    Remove,
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

fn first_alias(item: &Item) -> Option<&str> {
    match item {
        Item::Host(h) => h.patterns.first().map(String::as_str),
        _ => None,
    }
}

/// 遷入前的資格檢查,用**與 `move_host` 相同的定位規則**(`find_host_file_index` → 該檔案裡「任一 pattern
/// 相符」的第一個區塊):那個區塊的所有 pattern 都必須具名。找不到 alias 交給 `move_host` 回報。
pub fn refuse_wildcard(doc: &SshConfigDoc, alias: &str) -> Result<(), AppError> {
    let Some(idx) = find_host_file_index(doc, alias) else { return Ok(()) };
    let block = doc.files[idx]
        .items
        .iter()
        .find(|i| matches!(i, Item::Host(h) if h.patterns.iter().any(|p| p == alias)));
    match block {
        Some(Item::Host(h)) if !is_syncable_block(&h.patterns) => Err(AppError::Other(format!(
            "host '{alias}' belongs to a block with wildcard patterns and cannot be synced"
        ))),
        _ => Ok(()),
    }
}

/// 同步檔與其他任何檔案都定義了的 alias(Include 置頂 → 同步檔那份的選項優先,本地那份仍會補上其餘選項)。
pub fn duplicate_aliases(doc: &SshConfigDoc, managed: &Path) -> Vec<DuplicateAlias> {
    let synced: Vec<&str> = doc
        .files
        .iter()
        .filter(|f| f.path == managed)
        .flat_map(|f| f.items.iter())
        .filter_map(first_alias)
        .collect();
    let mut out = Vec::new();
    for file in doc.files.iter().filter(|f| f.path != managed) {
        for alias in file.items.iter().filter_map(first_alias) {
            if synced.contains(&alias) {
                out.push(DuplicateAlias { alias: alias.to_string(), local_file: file.path.to_string_lossy().into_owned() });
            }
        }
    }
    out
}

/// 處理一筆被遮蔽的本地主機:以 `file`(完整路徑)定位那個檔案裡第一個 pattern 等於 `alias` 的
/// 區塊。回傳改動的檔案索引(呼叫端負責 `persist_file`)。拒絕碰同步檔那份。
pub fn resolve_shadowed(
    doc: &mut SshConfigDoc,
    alias: &str,
    file: &str,
    action: ShadowedAction,
    managed: &Path,
) -> Result<usize, AppError> {
    let idx = doc
        .files
        .iter()
        .position(|f| f.path.to_string_lossy() == file)
        .ok_or_else(|| AppError::NotFound(format!("file '{file}' is not loaded")))?;
    if doc.files[idx].path == managed {
        return Err(AppError::Other("refusing to change the synced copy; pick the local file".to_string()));
    }
    let pos = doc.files[idx]
        .items
        .iter()
        .position(|i| first_alias(i) == Some(alias))
        .ok_or_else(|| AppError::NotFound(format!("host '{alias}' is not defined in '{file}'")))?;
    match action {
        ShadowedAction::Remove => {
            doc.files[idx].items.remove(pos);
        }
        ShadowedAction::Rename => {
            let new_alias = format!("{alias}-local");
            validate_host_patterns(std::slice::from_ref(&new_alias))?;
            let taken = doc.files.iter().flat_map(|f| f.items.iter()).any(|i| first_alias(i) == Some(new_alias.as_str()));
            if taken {
                return Err(AppError::Other(format!("host '{new_alias}' already exists; rename it in the editor instead")));
            }
            if let Item::Host(h) = &mut doc.files[idx].items[pos] {
                let mut patterns = h.patterns.clone();
                patterns[0] = new_alias;
                set_host_patterns(h, &patterns);
            }
        }
    }
    Ok(idx)
}

fn managed_path() -> Result<std::path::PathBuf, AppError> {
    Ok(crate::sync::hosts_file::managed_path(&crate::keys::ssh_dir()?))
}

#[tauri::command]
pub fn sync_duplicate_aliases(app: AppHandle) -> Result<Vec<DuplicateAlias>, AppError> {
    let managed = managed_path()?;
    let state = app.state::<AppState>();
    let guard = state.doc.lock().unwrap();
    let doc = guard.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    Ok(duplicate_aliases(doc, &managed))
}

#[tauri::command]
pub fn sync_resolve_shadowed(app: AppHandle, alias: String, file: String, action: ShadowedAction) -> Result<Vec<DuplicateAlias>, AppError> {
    let managed = managed_path()?;
    let state = app.state::<AppState>();
    let mut doc_lock = state.doc.lock().unwrap();
    let mut backed_up = state.backed_up.lock().unwrap();
    let retention = *state.backup_retention.lock().unwrap();
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let idx = resolve_shadowed(doc, &alias, &file, action, &managed)?;
    persist_file(doc, idx, &mut backed_up, retention)?;
    Ok(duplicate_aliases(doc, &managed))
}

/// 逐台搬進同步檔;每台獨立成功/失敗。`tag_by_file` 時把原檔名加成 tag(已有同名 tag 不重複)。
#[tauri::command]
pub fn sync_migrate_hosts(app: AppHandle, aliases: Vec<String>, tag_by_file: bool) -> Result<MigrationReport, AppError> {
    let managed = managed_path()?;
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
        if let Err(e) = refuse_wildcard(doc, &alias) {
            report.failed.push(MigrationFailure { alias, error: e.to_string() });
            continue;
        }
        let source_tag = find_host_file_index(doc, &alias).map(|i| tag_for_file(&doc.files[i].path));
        match move_host(doc, &alias, &managed_str) {
            Ok((src, tgt)) => {
                if let Err(e) = persist_file(doc, tgt, &mut backed_up, retention)
                    .and_then(|_| persist_file(doc, src, &mut backed_up, retention))
                {
                    report.failed.push(MigrationFailure { alias, error: e.to_string() });
                    continue;
                }
                if let (true, Some(tag)) = (tag_by_file, source_tag) {
                    // 先讀搬進去那個區塊自己的 tags(可變借用在這行結束),再取可變借用寫回:
                    // 兩段借用不重疊,才不會 E0502;也不用 host_summaries(它取第一個命中,遮蔽時會錯)。
                    let mut tags = find_host_mut(&mut doc.files[tgt].items, &alias)
                        .map(|h| parse_tags(&h.body))
                        .unwrap_or_default();
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

並在 `sync/mod.rs` 加 `pub mod migrate;`;`lib.rs` 的 `use` 與 `generate_handler!` 加入 `sync_migrate_hosts`、`sync_duplicate_aliases`、`sync_resolve_shadowed`(路徑 `sync::migrate::…`)。

- [ ] **Step 4: 執行測試確認通過**

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result' && ls ../src/bindings/ | grep -iE 'Migration|Duplicate'`
Expected: 全綠(`sync::migrate` 4 個);`MigrationReport.ts`、`MigrationFailure.ts`、`DuplicateAlias.ts` 生成。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/mod.rs src-tauri/src/sync/migrate.rs src-tauri/src/config/dto.rs src-tauri/src/lib.rs src/bindings/MigrationReport.ts src/bindings/MigrationFailure.ts src/bindings/DuplicateAlias.ts
git commit -m "feat(sync): migrate hosts into the synced file, detect and resolve shadowed aliases by file"
```

---

## Self-review(已執行)

- **Spec 覆蓋**:§6 基線輪(Join;Create 直接 true)、一輪的順序(gather 指紋與 mtime → plan_local 逐區塊時間並持久化 → pull_merge → 指紋/generation 守衛套用、套不上的不進快取 → push 分批 → 事件)→ Task 2 + Task 3;cursor 只隨 pull(Task 2 測試);`SyncCore` 同一把鎖的原子快照與 lifecycle 互斥(Task 3);sealed 保留與唯讀持久化(Task 2 測試);Upsert/Delete/略過三態(Task 2 測試,含夾帶 Match 的文字);Create 用 PUT、Join 用 GET(Task 3 `enter_chain`);Leave 的 keychain 失敗回報與持久化旗標(Task 3 `leave_chain`);網路只在 std thread / spawn_blocking(Task 3);§10 遷入 wizard 的主機/tag 部分、遷入拒絕 wildcard 區塊(與 `move_host` 同一定位規則)、檔案定位的 shadowed 處理(Task 4);`sync://applied` 事件與 tray 重建(Task 3 `apply_effects`)。
- **型別一致**:`HostEffect`/`Merged`/`Pushed` 由 Task 2 定義、Task 3 使用;`plan_local(state, current, changed_at, now_ms, platform)` 與 `detect_local_changes(…, changed_at)` 的閉包簽章一致;`Envelope` 來自 A1 `record.rs`;`SyncState.sealed`/`remote_schema_version`/`baseline_established`/`phrase_cleanup_pending`/`read_only()` 來自 A1 Task 4;`RelayClient::validate_url` 來自 A1 Task 5;`status_from(state, managed_file)` 兩個參數,`phrase_cleanup_pending` 從狀態讀;A4 的 hooks 對應 commands:`sync_status`、`sync_create_chain`、`sync_join_chain`、`sync_show_words`、`sync_leave_chain`、`sync_now`、`sync_set_relay_url`、`sync_set_device_name`、`sync_forget_device`、`sync_migrate_hosts`、`sync_duplicate_aliases`、`sync_resolve_shadowed`。
- **鎖順序**:lifecycle → doc → core。`enter_chain`/`leave_chain` 在 lifecycle 內先做網路(不持其他鎖),再 doc → core;`apply_effects` doc → core(只短暫讀 generation);`commit_state`/`mutate_state`/`with_state`/`emit_status` 只鎖 core;`gather_blocks`/`ensure_managed_loaded`/`sync_migrate_hosts` 只鎖 doc;`note_file_written` 在呼叫端持有 doc 鎖時只鎖 TRACKER(獨立的全域鎖,不與其他鎖巢狀)。沒有任何路徑以相反順序取鎖。
- **Review Focus 對應**:1 → Task 1 `timestamps_never_go_backwards` + `unchanged_block…bumps_version`;2 → Task 2 `undecryptable_envelopes_are_skipped_not_fatal`;3 → Task 2 `concurrent_edit_is_resolved…`;4 → Task 2 `cursor_only_follows_the_pull_watermark`;5 → Task 2 `push_conflict_is_deferred_to_the_next_round`;6 → Task 2 `unparseable_host_payload_is_skipped_not_deleted`;7 → Task 3 `apply_effects`(指紋雙重比對 + Conflict → FileChanged;手動 smoke);8 → Task 2 `pushes_are_batched_by_count`;9 → Task 2 `invalid_host_text_is_skipped_and_not_cached`;10 → Task 3 基線分支(手動 smoke);11 → Task 1 `each_changed_block_carries_its_own_change_time` + Task 3 `block_tracker_stamps_only_what_changed`;12 → Task 3 `a_broken_effect…is_reported_by_alias` + `unapplied_aliases_are_rolled_back…`。
- **Codex review 三輪已納入**:cursor(5)、唯讀持久化與 sealed(6)、格式不支援不當成刪除(7)、離線 dirty 先持久化(8)、網路期間本機編輯(9)、Leave 覆蓋(10)、Join 用 GET(21)、E0502(22)、分批(23)、keychain 錯誤回報與持久化(24、M10)、blocking reqwest 進 spawn_blocking、shadowed 檔案定位(12)、`sync://applied`(25);第二輪:generation 守衛(H3)、基線輪(H4)、合併前驗證(H5)、mtime(M9)、持久化失敗還原(M8)、遷入拒絕 wildcard(M3);第三輪:`SyncCore` 原子快照(R3-H1)、lifecycle 鎖(R3-H2)、套不上的 alias 不進快取(R3-H3、7、H5)、`refuse_wildcard` 與 `move_host` 同一定位規則(R3-H4、17、M3)、持久化失敗先退回再重載(R3-M4)、逐區塊修改時間(R3-M5、8、M9)、Create 基線例外與 `status_from` 參數的文字同步(R3-L1)。
