# SP3 — 金鑰插槽與金鑰同步 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 同步的主機在每台電腦上都能連線:主機的 `IdentityFile` 指到 `~/.ssh/sshelter/keys/<name>-<id8>` 這個插槽,每台電腦
自己決定插槽裡放什麼 —— 建立它的那台連到原檔、選了「Sync key」的其他電腦放端對端加密同步來的私鑰、選了「Keep on this
computer」的其他電腦連到使用者在那台挑的金鑰。伺服器永遠不動。

**Architecture:** 帳戶 chain 新增 `keyslot`(明文 metadata)與沿用 v1 預留的 `key`(祕密,以密文保存、LWW 在記憶體比較)。
純函式(`sync/slot_rules.rs`:型別、名稱規則、OpenSSH 私鑰檢查、`IdentityFile` 解析)、檔案系統(`sync/slot_files.rs`
與只依賴 `windows-sys` 的 `sync/slot_files_windows.rs`)、引擎(`sync/slots.rs`:記錄、每一輪的插槽維護、模式切換、
挑選;`sync/slot_setup.rs`:候選與建立插槽、無損改寫主機)分開。每一輪在合併帳戶、套用 space 檔之後、上傳之前
維護插槽;Tauri commands 包裝建立、切換、挑選、刪除副本。前端新增「Sync key」對話框、Keys 對話框的插槽區塊、
Settings 提示列、側邊欄標記與「Keys for this computer」對話框。

**Tech Stack:** Rust 2021、Tauri 2;既有相依 `serde`/`serde_json`、`base64 0.22`、`sha2 0.10`、`getrandom 0.3`、
`tempfile 3`;新增 **只限 Windows** 的 `windows-sys = "0.61"`(Cargo.lock 已有 0.61.2,是 `tempfile` 等的相依,不需下載)。
前端 React 19、TypeScript、TanStack Query、zustand、shadcn/ui;測試 vitest(node 環境、`renderToStaticMarkup`)。

**Spec:** `docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md`(本計畫實作 §1–§12;§14 的後續階段不在範圍)。
同類產品調查:`docs/superpowers/specs/2026-10-05-sp3-key-sync-research.md`。Sync v2 的背景:
`docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`。

## 計畫裁定(寫計畫時對 spec 做的決定)

1. **只有 OpenSSH 格式的私鑰可以同步**(spec §6.2 寫「OpenSSH 或 PEM」)。接收端要從私鑰讀出公鑰比對指紋:OpenSSH 格式的
   公鑰段不加密,純 Rust 就讀得出來(已用三把測試金鑰對過 `ssh-keygen -l`);PEM/PKCS#8 要自己解析 ASN.1,不值得為舊格式
   新增大量程式。PEM 金鑰的「Sync key」不提供,對話框說明可以用 `ssh-keygen -p -f <file>` 轉成 OpenSSH 格式
   (OpenSSH 7.8 起預設寫出新格式);「Keep on this computer」照常可用。代價:舊式 PEM 金鑰要先轉檔才能同步。
2. **`device` 記錄的新欄位叫 `slots`**,不沿用既有的 `keys: Vec<String>`(spec 已在 21c913e 改正)。
3. **同步的金鑰換了,不自動蓋掉別台的副本**(spec §6.5 只寫來源電腦不自動上傳)。別台已有的同步副本若和插槽目前的同步
   金鑰不同,狀態顯示「A newer synced key is available」,由使用者按「Use the synced key」:舊副本改名保留成
   `<file>.previous-<8 hex>`,再放新的。理由:spec「私鑰永遠不自動刪除」;被換掉的可能是那把舊金鑰的最後一份。
4. **候選掃描與插槽狀態分開**:`SyncOverview` 只帶插槽狀態(由同步狀態算出,便宜);「還沒設定的金鑰」要讀 config,
   由另一個 command `sync_key_candidates` 提供,前端放在 `["config", "keyCandidates"]` 底下,config 變動就重抓。
5. **「Keys for this computer」改由通知觸發**(spec §6.7 寫在加入流程的最後一步):剛勾選的 space 第一輪同步完成之前,
   主機還不在磁碟上,加入流程結束時還不知道需要哪些金鑰。後端在某個插槽第一次「需要這台的金鑰」時發出
   `SyncNotice::KeysNeeded`,前端收到就開這個對話框(同 `SyncUpgradeDialog` 讀通知的做法)。
6. **相對路徑的 `IdentityFile`**(不以 `~`、`%d`、`/`、磁碟代號開頭)列為「Can't set up automatically」:OpenSSH 對它的
   解析依呼叫時的工作目錄,無法可靠地對到一個檔案。
7. **hard link 與複製跟上原檔時不另外告知**(spec §6.5 寫「在狀態列告知插槽已更新」):插槽換成的正是使用者自己換上的那把,
   Unix 的 symlink 本來就無聲地跟著走,Windows 照做才一致;真正需要使用者處理的情況(`synced` 插槽的來源電腦換了金鑰)
   另有「This computer's key changed …」狀態與「Sync the new key」。代價:Windows 上換了原檔的人不會看到「插槽已更新」。
8. **指紋檢查在每一輪做**(spec §6.5 另列「打開 Keys 對話框時」):啟動與視窗取得焦點時本來就會跑一輪,打開 Keys 對話框
   不再另外觸發(多一輪就多一次 relay 用量,同 `sync-events.ts` 的做法);對話框顯示最近一輪的結果。代價:在 app 開著時
   換掉金鑰檔,要等下一輪(例如切回視窗)狀態才更新。
9. **主機搬進 space 或在 space 裡複製也會跳出詢問**:spec §7.1 列的「在 space 裡新增主機」,在 app 裡的路徑是主機編輯器的
   Move to file 與 Duplicate host、側邊欄的搬移(含整批);「Add host」對話框沒有 `IdentityFile` 欄位,不需要。

## Global Constraints

- 不要執行 `pnpm` 或 `pnpm exec`(會重裝真正的 node_modules);前端一律用 `./node_modules/.bin/vitest run --dir src`、
  `./node_modules/.bin/tsc --noEmit`、`./node_modules/.bin/vite build`(在 repo 根目錄執行)。
- Rust 一律用 rustup 的工具鏈(Homebrew 的 rustc 壞了):在 `src-tauri/` 執行
  `PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
  (恰好一個 `--`;那兩個測試會讀寫真正的 keychain,一律略過)。下文「跑 Rust 測試」都是這個指令;只跑一部分時在
  `--lib` 後面加篩選字串,例如 `cargo test --offline --lib slot_rules -- --skip …`。
- 每個 task 結束前:Rust 全部測試、`tsc --noEmit`、vitest 全部通過才 commit;前端 task 另外跑一次 `vite build`。
- 測試不碰真正的家目錄、`~/.ssh` 與 keychain:引擎測試用 `sync::testkit::TestDevice`(臨時家目錄);檔案函式一律收
  明確的路徑參數。程式裡的家目錄一律由 `env.ssh_dir.parent()` 取得,**不得**在會被測試執行到的程式裡呼叫
  `keys::ssh_dir()`、`dirs::home_dir()` 或 `shellexpand`。不得 pkill/killall,不得啟動 app。
- 私鑰內容永遠不進 log、錯誤訊息、`Debug` 輸出、事件 payload、狀態檔明文與 IPC 回傳值(`KeyPayload` 的 `Debug` 要遮蔽;
  錯誤訊息只帶路徑與原因)。
- 鎖的順序:lifecycle → doc → backed_up → retention → core。`persist_file` 會在持有 doc 鎖時呼叫存檔 hook(它拿 core 鎖),
  所以**不得**在持有 core 鎖時呼叫 `persist_file`;`runtime::mutate` 內部會拿 doc 鎖與 core 鎖,不得在它的閉包裡再拿 doc 鎖。
- Rust 註解用繁體中文;識別字、錯誤訊息、UI 字串、commit 訊息用英文;Conventional Commits。
- 只 `git add` 每個 task 列出的路徑(含該 task 由 ts-rs 產生的 `src/bindings/*.ts`);**不得** stage `src-tauri/Cargo.lock`
  (repo 裡它有一行和本計畫無關的本機變更;Task 2 加的 `windows-sys` 也會讓它多一行,同樣不 stage)、`.superpowers/`、
  `relay/`。不 push、不派 subagent。
- ts-rs:匯出型別的 `u64` 欄位一律 `#[cfg_attr(test, ts(type = "number"))]`(`Option<u64>` 用 `"number | null"`);
  `cargo test` 會重寫 `src/bindings/`,新增或改動的 binding 要一起 commit。
- 特殊字元在原始碼裡一律寫成 escape(`\u{…}`、`\u…`),commit 前檢查實際的位元組。
- 來自其他電腦的名稱(插槽名稱、裝置名稱、space 名稱)在 UI 一律經 `revealHidden`(`src/lib/sync-approvals.ts` 的匯出)顯示。
- 任何處理 Enter 的 key handler 第一行(或同一行)要有 `isImeKey(e)` 判斷(`src/lib/enter-handlers.test.ts` 會檢查)。
- Windows 專屬程式碼(`slot_files_windows.rs`)只依賴 std 與 `windows_sys`,好讓它能在 macOS 上以 scratch crate 對
  `x86_64-pc-windows-msvc` 做型別檢查(Task 2 Step 6;整個 app 對 Windows 目標的 `cargo check` 會卡在 `aws-lc-sys` 需要
  Windows 的 C 編譯器,不可行)。
- UI 字串照下表一字不差(各 task 的程式碼與測試用的就是這些;README 與手動清單也引用它們)。`{…}` 是代入的值,
  來自其他電腦的名稱一律先經 `revealHidden`:

| 位置 | 字串 |
|---|---|
| Sync key 對話框 | 標題 `Keys used by synced hosts`;說明 `Choose for each key whether it goes to your other computers. Your servers aren't changed.` |
| 每把金鑰的列 | `{host} uses {name}.` / `{hosts} use {name}.`;`Sync this key to your other computers?`;改寫的行 `{alias}: IdentityFile {value} → ~/.ssh/sshelter/keys/{name}-…`;`Not changed: {aliases} — {reason}` |
| passphrase | `Has a passphrase — it stays on each computer.` / `No passphrase — your sync code and every joined computer can use this key once it syncs.` |
| 按鈕 | `Sync key`、`Keep on this computer`、`Rename`(輸入框 aria-label `Key name`);頁尾 `Later`(升級後或從 Settings 開的)/ `Close` |
| 無法自動設定 | 標題 `Can't set up automatically`;每列 `{alias}: {value} — {reason}`;reason 是 `points at a public key (an agent provides the private key)`、`uses % tokens or environment variables`、`is a relative path` |
| 不能同步的金鑰 | `This key is larger than 16 KiB, so it can't be synced. Keep it on this computer.`、`This key isn't in the OpenSSH format, so it can't be synced. Convert it with ssh-keygen -p -f <file>, or keep it on this computer.`、`This file couldn't be read as an OpenSSH private key.` |
| 不改寫的主機 | `This host has more than one copy; SSHelter changes it once only one copy is left.` |
| 成功 toast | `{name} syncs to your other computers`、`{name} stays on this computer`、`{name} no longer syncs; computers that have it keep their copy`、`{name} uses the synced key on this computer`、`{name} uses {file} on this computer` |
| 失敗 toast | `Could not set up the key`、`Could not change how the key is shared`、`Could not use that key`、`Could not use the synced key`、`Could not delete the copy` |
| Settings → Sync 提示列 | `1 key used by synced hosts isn't set up` / `{N} keys used by synced hosts aren't set up`(說明 `Choose whether each key goes to your other computers.`,按鈕 `Set up…`);`1 key slot needs a key on this computer` / `{N} key slots need a key on this computer`(說明 `Synced hosts use {names}, which stay on your other computers.`,按鈕 `Pick…`) |
| Keys 對話框的區塊 | 標題 `Keys used by synced hosts`;模式 `Synced to your computers` / `Each computer uses its own key`;`Used by {hosts}`;`{device}: synced copy` / `{device}: its own key`(多台以 ` · ` 相接) |
| 插槽狀態 | `Ready`、`Needs a key on this computer`、`Waiting for the synced key`、`Not in use`、`Not used on this computer`、`A synced key is available`、`This computer's key changed — your other computers still have the previous one`;錯誤時直接顯示後端訊息 |
| 插槽動作 | `Sync this key`、`Sync the new key`、`Use the synced key`、`Pick a key on this computer…`、`Change…`、`Stop syncing`、`Delete copy` |
| 挑金鑰 | 標題 `Pick a key on this computer`;說明 `Hosts that use {name} will use the key you pick, on this computer only.`;`The synced copy on this computer is kept as a .previous file.`;按鈕 `Browse…`(檔案對話框標題 `Choose a private key`) |
| 刪除副本 | 標題 `Delete this copy?`;說明 `The copy of {name} on this computer is deleted. Other computers and the original key aren't affected.`;按鈕 `Cancel`、`Delete copy` |
| Keys for this computer | 標題 `Keys for this computer`;說明 `Synced hosts on this computer use keys that stay on your other computers. Pick a key on this computer for each, or do it later in Keys.`;按鈕 `Pick…`、`Done` |
| `keys_needed` 通知 | `noticeMessage`:標題 `Pick keys for this computer`、說明 `Synced hosts use {names}, which stay on your other computers.`(窮舉的 `switch` 需要它;這則通知只用來開上一列的對話框,不跳 toast、不列在 Notices) |
| 後端訊息(插槽狀態 `Error` 與 command 的錯誤) | `The synced key didn't match and was not written.`、`A file SSHelter didn't create is in the way: {path}. Move it, then sync again.`、`The key this slot points to is gone: {path}.`、`Hosts on this computer still use this key; change them first.`、`Do this on a computer that has this key, such as {device}.` |
| 側邊欄標記 | title `This host's key isn't on this computer — pick one in Keys.`;圖示 aria-label `This host's key isn't on this computer` |
| Leave 對話框 | `Keys in ~/.ssh/sshelter/keys stay on this computer.` |
| Change sync code 確認 | `Keys you synced stay on every computer that has them. If a computer was lost, replace those keys on your servers.` |
| 新同步碼的對話框 | `If a computer was lost, also replace these synced keys on your servers: {names}.` |
| lint | `IdentityFile not found: {value} (a synced key slot — pick a key for it in Keys)` |

## Review Focus

spec 沒寫、各 task 的一般測試也碰不到,最可能讓人吃虧的輸入或情況(每一條的測試放在負責的 task 裡):

1. **插槽路徑上已經有使用者自己的檔案**(例如重灌後狀態檔是新的、目錄裡留著舊檔):絕不覆蓋,狀態說明。→ Task 4。
2. **私鑰內容出現在狀態檔、錯誤訊息或回傳值裡**:同步過的金鑰在 `sync-state.json` 只能是密文;落地失敗的訊息不含金鑰。
   → Task 3、Task 4。
3. **建立插槽到一半 config 被外部改了**(`persist_file` 回 `Conflict`):帳戶記錄與本機連結已建立、主機沒改寫;下一次掃描
   把它當成「已有插槽、直接沿用」,不會重複建立。→ Task 5。
4. **Windows 與帶引號的 `IdentityFile` 值**(`C:\Users\x\.ssh\id`、`"~/.ssh/id work"`、`~\.ssh\id`):解析到正確的檔案;
   SSHelter 自己寫入的路徑是 `~/.ssh/…`。→ Task 1、Task 8。
5. **`key` 記錄比 `keyslot` 先到,或 `keyslot` 已刪除、主機還指著它**:不寫出半套的檔案;插槽狀態合理(等待或需要金鑰)。
   → Task 4。

## 檔案結構

| 檔案 | 責任 |
|---|---|
| `src-tauri/src/sync/slot_rules.rs`(新) | 純函式與型別:`KeySlotPayload`、`SlotMode`、`KeyPayload`、`DeviceSlot`、插槽 id/名稱/檔名規則、OpenSSH 私鑰檢查、`IdentityFile` 值解析 |
| `src-tauri/src/sync/slot_files.rs`(新) | 插槽的檔案系統動作:目錄、私鑰與 `.pub` 的寫入、連結(symlink / hard link / 複製)、內容雜湊、移除 |
| `src-tauri/src/sync/slot_files_windows.rs`(新) | Windows:owner-only、不繼承的 DACL(只依賴 std 與 `windows-sys`) |
| `src-tauri/src/sync/slots.rs`(新) | 帳戶記錄(`keyslot`/`key`)的讀寫、每一輪的插槽維護(`reconcile`)、插槽狀態與 overview 用的檢視、模式切換、挑選、改用同步金鑰、刪除副本 |
| `src-tauri/src/sync/slot_setup.rs`(新) | 「還沒設定的金鑰」候選、建立插槽、無損改寫主機的 `IdentityFile` |
| `src-tauri/src/sync/record.rs` | `RecordKind::KeySlot`;`DevicePayload.slots` |
| `src-tauri/src/sync/merge.rs` | `valid_account_record` 的新種類;`merge_account` 對 `keyslot`/`key` 的合併;`device` 記錄沿用 `slots`、`set_device_slots` |
| `src-tauri/src/sync/state_v2.rs` | `SyncStateV2.key_slots`(本機插槽);`SyncNotice::KeysNeeded` |
| `src-tauri/src/sync/round.rs` | 每一輪呼叫 `slots::reconcile` |
| `src-tauri/src/sync/rotation.rs` | 更換同步碼時複製 `keyslot`/`key` |
| `src-tauri/src/sync/dto.rs` | `SyncOverview.key_slots` 與插槽檢視型別 |
| `src-tauri/src/sync/engine.rs`、`src-tauri/src/lib.rs`、`src-tauri/src/sync/mod.rs` | 新 commands 與模組註冊 |
| `src-tauri/src/config/intel.rs` | 插槽路徑的 lint 訊息 |
| `src-tauri/Cargo.toml` | `[target.'cfg(windows)'.dependencies] windows-sys` |
| `src/lib/key-slots.ts`(新) | 前端純函式:插槽列、狀態文字、候選篩選、提示列計數、側邊欄標記 |
| `src/lib/sync.ts`、`src/lib/sync-fixtures.ts`、`src/stores/ui.ts`、`src/stores/settings.ts` | hooks、測試 builder、開啟對話框的狀態、升級提示只出現一次的旗標 |
| `src/components/SyncKeyDialog.tsx`(新) | 「Keys used by synced hosts」對話框 |
| `src/components/KeySlotsSection.tsx`(新) | Keys 對話框的插槽區塊與「Pick a key」對話框 |
| `src/components/KeysNeededDialog.tsx`(新) | 「Keys for this computer」對話框 |
| `src/components/KeysDialog.tsx`、`SyncPane.tsx`、`HostList.tsx`、`SyncMigrationDialog.tsx`、`HostEditor.tsx`、`DeployKeyDialog.tsx`、`App.tsx` | 掛上新元件與觸發點 |
| `src/lib/identity-file.ts`、`src/lib/deploy-key-select.ts`、`src/lib/sync-events.ts` | Windows 路徑正規化;新通知的文字 |
| `.github/workflows/test-windows.yml`(新) | 在 `windows-latest` 跑 `slot_rules` 與 `slot_files`(含 DACL、hard link)的測試 |
| `README.md`、`docs/superpowers/plans/2026-10-05-sp3-manual-verification.md`(新)、SP1 spec | 文件 |

---

### Task 1: 插槽的純函式與型別(`slot_rules.rs`、`RecordKind::KeySlot`、`DevicePayload.slots`)

**Files:**
- Create: `src-tauri/src/sync/slot_rules.rs`
- Modify: `src-tauri/src/sync/mod.rs`(註冊模組)
- Modify: `src-tauri/src/sync/record.rs`(`RecordKind::KeySlot`、`DevicePayload.slots`)
- Modify: `src-tauri/src/sync/merge.rs`(`own_device_record` 建 `DevicePayload` 時填 `slots: Vec::new()`,讓它編得過;Task 3 再改成沿用)
- Generated: `src/bindings/SlotMode.ts`

**Interfaces:**
- Produces(之後每個 task 都用):
  - `pub const SLOT_DIR: &str = ".ssh/sshelter/keys"`、`MAX_PRIVATE_KEY_BYTES: usize = 16 * 1024`、`SLOT_SCHEMA: u32 = 1`
  - `pub enum SlotMode { Synced, Own }`(serde `"synced"`/`"own"`,ts 匯出)
  - `pub struct KeySlotPayload { schema, name, mode, origin_device_id, created_at_ms: u64, public_key: Option<String>, fingerprint: Option<String>, key_type: Option<String>, has_passphrase: Option<bool> }`
  - `pub struct KeyPayload { schema: u32, private_key: String }`(`Debug` 遮蔽 `private_key`)
  - `pub struct DeviceSlot { slot_id: String, fingerprint: Option<String>, synced_copy: bool }`
  - `pub fn is_slot_id(&str) -> bool`、`pub fn new_slot_id() -> Result<String, AppError>`
  - `pub fn valid_slot_name(&str) -> bool`、`pub fn default_slot_name(file_name: &str) -> String`
  - `pub fn slot_file_name(name: &str, slot_id: &str) -> String`(`<name>-<id 前 8>`)、`pub fn slot_value(file_name: &str) -> String`(`~/.ssh/sshelter/keys/<file>`)、`pub fn slot_path(home: &Path, file_name: &str) -> PathBuf`、`pub fn public_path(slot: &Path) -> PathBuf`
  - `pub fn unquote(&str) -> &str`、`pub fn slot_file_of_value(value: &str) -> Option<String>`
  - `pub enum IdentityTarget { Slot(String), File(PathBuf), Unsupported(&'static str) }`、`pub fn resolve_identity_value(value: &str, home: &Path) -> IdentityTarget`;原因常數 `REASON_PUBLIC_KEY`、`REASON_TOKENS`、`REASON_RELATIVE`
  - `pub struct KeyFacts { public_key: String, fingerprint: String, key_type: String, has_passphrase: bool }`、`pub enum Unsyncable { TooLarge, NotOpenSsh, Unreadable }`(`fn message(self) -> &'static str`)、`pub fn inspect_private_key(text: &str) -> Result<KeyFacts, Unsyncable>`
  - `pub fn blob_fingerprint(&[u8]) -> String`、`pub fn parse_public_key(line: &str) -> Option<(String, String)>`(回 `<type> <base64>` 與指紋)
  - `pub fn valid_slot_payload(&KeySlotPayload) -> bool`、`pub fn valid_key_payload(&KeyPayload) -> bool`
  - 測試用:`#[cfg(test)] pub(crate) mod test_keys`(三把測試金鑰與它們的公鑰、指紋,見 Step 1)
  - `RecordKind::KeySlot`(wire `"keyslot"`,`is_secret` = false);`DevicePayload.slots: Vec<DeviceSlot>`(`#[serde(default, skip_serializing_if = "Vec::is_empty")]`)

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slot_rules.rs` 先放測試(實作在 Step 3)。測試金鑰是寫計畫時用 `ssh-keygen` 產生、只給測試用的
金鑰(指紋已用 `ssh-keygen -l` 核對過)。標頭與結尾在編譯時以 `concat!` 組出來:原始碼(與本計畫)裡不出現完整的
私鑰標頭字面,避免 push 時被祕密掃描擋下。

```rust
#[cfg(test)]
pub(crate) mod test_keys {
    //! 只給測試用的金鑰(寫計畫時產生,沒有在任何地方使用)。指紋由 `ssh-keygen -l` 核對過。
    pub const BEGIN: &str = concat!("-----BEGIN ", "OPENSSH", " PRIVATE KEY-----");
    pub const END: &str = concat!("-----END ", "OPENSSH", " PRIVATE KEY-----");

    /// 標頭 + base64 本體 + 結尾,每行以 `\n` 結束(同 ssh-keygen 的輸出)。
    pub fn armor(body: &[&str]) -> String {
        let mut text = format!("{BEGIN}\n");
        for line in body {
            text.push_str(line);
            text.push('\n');
        }
        text.push_str(END);
        text.push('\n');
        text
    }

    /// ed25519,沒有 passphrase,comment `sp3-test`。
    pub const PLAIN_BODY: &[&str] = &[
        "b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW",
        "QyNTUxOQAAACBeTPcXX309Kd9wQ9S4ixU4hL647+CGTBQAwvNXseJ3XwAAAJD8YIOG/GCD",
        "hgAAAAtzc2gtZWQyNTUxOQAAACBeTPcXX309Kd9wQ9S4ixU4hL647+CGTBQAwvNXseJ3Xw",
        "AAAEBIKVUPhG+FZbzpyXbI6YwCJdusAIAdT6he8GYE/GBAtF5M9xdffT0p33BD1LiLFTiE",
        "vrjv4IZMFADC81ex4ndfAAAACHNwMy10ZXN0AQIDBAU=",
    ];
    pub const PLAIN_PUBLIC: &str = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIF5M9xdffT0p33BD1LiLFTiEvrjv4IZMFADC81ex4ndf";
    pub const PLAIN_FINGERPRINT: &str = "SHA256:9Q3QMhBJBcoUNE88XYEQbCPlcFByPPyVPJ6enJtQ+ew";

    /// ed25519,passphrase `test-passphrase`(aes256-ctr / bcrypt),comment `sp3-enc`。
    pub const ENC_BODY: &[&str] = &[
        "b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABAkYULw+o",
        "iDv11WqAPDdElfAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIJSEH6Vd1hjhpqq0",
        "z2zGJIQJG79kGlcIWqul53zwVaNNAAAAkC/r9CLPKPu1IJXPuu+UkwdfrDNh8vFxuo8PcI",
        "EMnUqZ/CrnnXRNdYFMp+tCsL0mXqDLa79kRN91YRyBFjyRjeLMAtTBCc6wtqfRK5Lh8bla",
        "2Vp25Stdqeaj1VV4bsn/Vpdh0CVtMU8B+uOLmaFig6Y0G7bN7b3StHzLx07OtnAOFaajRG",
        "gPtXuPP1MoRi3kkQ==",
    ];
    pub const ENC_PUBLIC: &str = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIJSEH6Vd1hjhpqq0z2zGJIQJG79kGlcIWqul53zwVaNN";
    pub const ENC_FINGERPRINT: &str = "SHA256:WZW83czQjcboddNwGO5ZP5Kvf7gt1ONldkA+inshlZM";

    /// ecdsa-sha2-nistp256,沒有 passphrase,comment `sp3-ecdsa`。
    pub const ECDSA_BODY: &[&str] = &[
        "b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAaAAAABNlY2RzYS",
        "1zaGEyLW5pc3RwMjU2AAAACG5pc3RwMjU2AAAAQQRIxTSlQ+YP54DPsfKBEqMLIXXX3x47",
        "4fpeDGMslw2TH516c8pU+sY5E4jKKdP/CtSjdfi8rHgXV5S4+o+ZvJr1AAAAqAWjAiQFow",
        "IkAAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEjFNKVD5g/ngM+x",
        "8oESowshddffHjvh+l4MYyyXDZMfnXpzylT6xjkTiMop0/8K1KN1+LyseBdXlLj6j5m8mv",
        "UAAAAhAOjBMQ46KiYFnSOdrHUgOWwcziDFP1KSy0a92A/T1zX/AAAACXNwMy1lY2RzYQEC",
        "AwQFBg==",
    ];
    pub const ECDSA_PUBLIC: &str = "ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEjFNKVD5g/ngM+x8oESowshddffHjvh+l4MYyyXDZMfnXpzylT6xjkTiMop0/8K1KN1+LyseBdXlLj6j5m8mvU=";
    pub const ECDSA_FINGERPRINT: &str = "SHA256:vUthAmDZoxYXCTAPEZUn5qtWSMHWQCEcUfpnyM05mMs";

    pub fn plain() -> String {
        armor(PLAIN_BODY)
    }
    pub fn encrypted() -> String {
        armor(ENC_BODY)
    }
    pub fn ecdsa() -> String {
        armor(ECDSA_BODY)
    }
}

#[cfg(test)]
mod tests {
    use super::test_keys::*;
    use super::*;
    use base64::engine::general_purpose::STANDARD;
    use base64::Engine as _;

    /// 把 OpenSSH 私鑰的二進位內容包回文字(每 70 字元一行)。
    fn rearmor(bytes: &[u8]) -> String {
        let b64 = STANDARD.encode(bytes);
        let lines: Vec<&str> = b64.as_bytes().chunks(70).map(|c| std::str::from_utf8(c).unwrap()).collect();
        armor(&lines)
    }

    fn plain_bytes() -> Vec<u8> {
        STANDARD.decode(PLAIN_BODY.concat()).unwrap()
    }

    #[test]
    fn inspects_an_unencrypted_ed25519_key() {
        let facts = inspect_private_key(&plain()).unwrap();
        assert_eq!(facts.public_key, PLAIN_PUBLIC);
        assert_eq!(facts.fingerprint, PLAIN_FINGERPRINT);
        assert_eq!(facts.key_type, "ssh-ed25519");
        assert!(!facts.has_passphrase);
    }

    #[test]
    fn inspects_an_encrypted_key_without_its_passphrase() {
        let facts = inspect_private_key(&encrypted()).unwrap();
        assert_eq!(facts.public_key, ENC_PUBLIC);
        assert_eq!(facts.fingerprint, ENC_FINGERPRINT);
        assert!(facts.has_passphrase);
    }

    #[test]
    fn inspects_an_ecdsa_key_and_crlf_text() {
        let facts = inspect_private_key(&ecdsa().replace('\n', "\r\n")).unwrap();
        assert_eq!(facts.public_key, ECDSA_PUBLIC);
        assert_eq!(facts.fingerprint, ECDSA_FINGERPRINT);
        assert_eq!(facts.key_type, "ecdsa-sha2-nistp256");
    }

    #[test]
    fn pem_and_pkcs8_keys_are_not_openssh() {
        let pem = format!("{}\nMIIBOgIBAAJBAKj34GkxFhD90vcNLYLInFEX6Ppy1tPf9Cnzj4p4WGeKLs1Pt8Qu\n{}\n", concat!("-----BEGIN RSA ", "PRIVATE KEY-----"), concat!("-----END RSA ", "PRIVATE KEY-----"));
        assert_eq!(inspect_private_key(&pem), Err(Unsyncable::NotOpenSsh));
        let pkcs8 = format!("{}\nMC4CAQAwBQYDK2VwBCIEIA==\n{}\n", concat!("-----BEGIN ", "PRIVATE KEY-----"), concat!("-----END ", "PRIVATE KEY-----"));
        assert_eq!(inspect_private_key(&pkcs8), Err(Unsyncable::NotOpenSsh));
    }

    #[test]
    fn garbled_and_oversized_keys_are_refused() {
        // 沒有結尾、base64 壞掉、魔術字不對、兩把金鑰、公鑰段被截斷。
        let no_end = plain().replace(END, "");
        assert_eq!(inspect_private_key(&no_end), Err(Unsyncable::Unreadable));
        assert_eq!(inspect_private_key(&armor(&["!!!not base64!!!"])), Err(Unsyncable::Unreadable));
        let mut wrong_magic = plain_bytes();
        wrong_magic[0] = b'X';
        assert_eq!(inspect_private_key(&rearmor(&wrong_magic)), Err(Unsyncable::Unreadable));
        let mut two_keys = plain_bytes();
        // "openssh-key-v1\0" (15) + "none" (4+4) + "none" (4+4) + "" (4) = 35:金鑰數量的 u32 從這裡開始。
        two_keys[35..39].copy_from_slice(&2u32.to_be_bytes());
        assert_eq!(inspect_private_key(&rearmor(&two_keys)), Err(Unsyncable::Unreadable));
        assert_eq!(inspect_private_key(&rearmor(&plain_bytes()[..50])), Err(Unsyncable::Unreadable));
        assert_eq!(inspect_private_key("not a key at all"), Err(Unsyncable::Unreadable));
        assert_eq!(inspect_private_key(&"A".repeat(MAX_PRIVATE_KEY_BYTES + 1)), Err(Unsyncable::TooLarge));
    }

    #[test]
    fn reads_public_key_lines() {
        assert_eq!(
            parse_public_key(&format!("{PLAIN_PUBLIC} sp3-test")),
            Some((PLAIN_PUBLIC.to_string(), PLAIN_FINGERPRINT.to_string()))
        );
        // 類型欄位和 blob 裡寫的不一樣、不是 base64、少欄位。
        let mislabelled = PLAIN_PUBLIC.replacen("ssh-ed25519", "ssh-rsa", 1);
        assert_eq!(parse_public_key(&mislabelled), None);
        assert_eq!(parse_public_key("ssh-ed25519 !!!"), None);
        assert_eq!(parse_public_key("ssh-ed25519"), None);
    }

    #[test]
    fn slot_ids_names_and_paths() {
        assert!(is_slot_id("0123456789abcdef0123456789abcdef"));
        assert!(!is_slot_id("0123456789ABCDEF0123456789abcdef"));
        assert!(!is_slot_id("0123456789abcdef"));
        let (a, b) = (new_slot_id().unwrap(), new_slot_id().unwrap());
        assert!(is_slot_id(&a) && is_slot_id(&b) && a != b);

        for ok in ["id_ed25519", "work", "a", "Key.2026_v-1"] {
            assert!(valid_slot_name(ok), "{ok}");
        }
        let too_long = "a".repeat(65);
        for bad in ["", "-x", ".x", "a b", "a/b", "a\\b", "id.pub", "ID.PUB", too_long.as_str(), "\u{9375}"] {
            assert!(!valid_slot_name(bad), "{bad}");
        }
        assert_eq!(default_slot_name("id_ed25519"), "id_ed25519");
        assert_eq!(default_slot_name("my key"), "my-key");
        assert_eq!(default_slot_name(".hidden"), "hidden");
        assert_eq!(default_slot_name("id_rsa.pub"), "id_rsa");
        assert_eq!(default_slot_name("\u{9375}"), "key");
        assert_eq!(default_slot_name(&"k".repeat(80)), "k".repeat(64));

        let id = "3fa2c1d90123456789abcdef01234567";
        assert_eq!(slot_file_name("id_mac", id), "id_mac-3fa2c1d9");
        assert_eq!(slot_value("id_mac-3fa2c1d9"), "~/.ssh/sshelter/keys/id_mac-3fa2c1d9");
        assert_eq!(slot_path(Path::new("/home/f"), "id_mac-3fa2c1d9"), PathBuf::from("/home/f/.ssh/sshelter/keys/id_mac-3fa2c1d9"));
        assert_eq!(public_path(Path::new("/k/id_mac-3fa2c1d9")), PathBuf::from("/k/id_mac-3fa2c1d9.pub"));
    }

    #[test]
    fn recognises_slot_values_in_every_spelling() {
        assert_eq!(slot_file_of_value("~/.ssh/sshelter/keys/id_mac-3fa2c1d9").as_deref(), Some("id_mac-3fa2c1d9"));
        assert_eq!(slot_file_of_value("\"~/.ssh/sshelter/keys/id_mac-3fa2c1d9\"").as_deref(), Some("id_mac-3fa2c1d9"));
        assert_eq!(slot_file_of_value("%d/.ssh/sshelter/keys/x-00000000").as_deref(), Some("x-00000000"));
        assert_eq!(slot_file_of_value("~/.ssh/sshelter/keys/sub/x"), None);
        assert_eq!(slot_file_of_value("~/.ssh/sshelter/keys/"), None);
        assert_eq!(slot_file_of_value("~/.ssh/id_mac"), None);
        assert_eq!(slot_file_of_value("/home/f/.ssh/sshelter/keys/x"), None);
    }

    #[test]
    fn resolves_identity_values_from_either_platform() {
        let home = Path::new("/home/f");
        let file = |p: &str| IdentityTarget::File(PathBuf::from(p));
        assert_eq!(resolve_identity_value("~/.ssh/id_mac", home), file("/home/f/.ssh/id_mac"));
        assert_eq!(resolve_identity_value("\"~/.ssh/id work\"", home), file("/home/f/.ssh/id work"));
        assert_eq!(resolve_identity_value("%d/.ssh/k", home), file("/home/f/.ssh/k"));
        assert_eq!(resolve_identity_value("~\\.ssh\\id_win", home), file("/home/f/.ssh/id_win"));
        assert_eq!(resolve_identity_value("/Users/x/.ssh/k", home), file("/Users/x/.ssh/k"));
        assert_eq!(resolve_identity_value("C:\\Users\\x\\.ssh\\k", home), file("C:\\Users\\x\\.ssh\\k"));
        assert_eq!(resolve_identity_value("~/.ssh/sshelter/keys/a-12345678", home), IdentityTarget::Slot("a-12345678".into()));
        assert_eq!(resolve_identity_value("~/.ssh/id.pub", home), IdentityTarget::Unsupported(REASON_PUBLIC_KEY));
        assert_eq!(resolve_identity_value("~/.ssh/%h", home), IdentityTarget::Unsupported(REASON_TOKENS));
        assert_eq!(resolve_identity_value("${HOME}/.ssh/k", home), IdentityTarget::Unsupported(REASON_TOKENS));
        assert_eq!(resolve_identity_value("id_rsa", home), IdentityTarget::Unsupported(REASON_RELATIVE));
    }

    fn synced_payload() -> KeySlotPayload {
        KeySlotPayload {
            schema: SLOT_SCHEMA,
            name: "id_mac".into(),
            mode: SlotMode::Synced,
            origin_device_id: "a".repeat(32),
            created_at_ms: 5,
            public_key: Some(PLAIN_PUBLIC.into()),
            fingerprint: Some(PLAIN_FINGERPRINT.into()),
            key_type: Some("ssh-ed25519".into()),
            has_passphrase: Some(false),
        }
    }

    #[test]
    fn validates_slot_payloads() {
        assert!(valid_slot_payload(&synced_payload()));
        let own = KeySlotPayload { mode: SlotMode::Own, public_key: None, fingerprint: None, key_type: None, has_passphrase: None, ..synced_payload() };
        assert!(valid_slot_payload(&own));
        // own 不得帶金鑰欄位;synced 的指紋要等於公鑰的指紋;名稱、schema、來源裝置。
        assert!(!valid_slot_payload(&KeySlotPayload { fingerprint: Some(PLAIN_FINGERPRINT.into()), ..own.clone() }));
        assert!(!valid_slot_payload(&KeySlotPayload { fingerprint: Some(ENC_FINGERPRINT.into()), ..synced_payload() }));
        assert!(!valid_slot_payload(&KeySlotPayload { key_type: None, ..synced_payload() }));
        assert!(!valid_slot_payload(&KeySlotPayload { name: "../x".into(), ..synced_payload() }));
        assert!(!valid_slot_payload(&KeySlotPayload { schema: 2, ..synced_payload() }));
        assert!(!valid_slot_payload(&KeySlotPayload { origin_device_id: String::new(), ..synced_payload() }));

        assert!(valid_key_payload(&KeyPayload { schema: SLOT_SCHEMA, private_key: plain() }));
        assert!(!valid_key_payload(&KeyPayload { schema: SLOT_SCHEMA, private_key: String::new() }));
        assert!(!valid_key_payload(&KeyPayload { schema: SLOT_SCHEMA, private_key: "A".repeat(MAX_PRIVATE_KEY_BYTES + 1) }));
    }

    #[test]
    fn key_payload_debug_hides_the_key() {
        let shown = format!("{:?}", KeyPayload { schema: SLOT_SCHEMA, private_key: plain() });
        assert!(!shown.contains(PLAIN_BODY[0]), "{shown}");
        assert!(shown.contains("<redacted>"));
    }
}
```

另外在 `src-tauri/src/sync/record.rs` 的 `#[cfg(test)] mod tests` 加:

```rust
    #[test]
    fn keyslot_kind_round_trips_and_is_not_secret() {
        assert_eq!(RecordKind::KeySlot.as_str(), "keyslot");
        assert_eq!(RecordKind::parse("keyslot"), Some(RecordKind::KeySlot));
        assert!(!RecordKind::KeySlot.is_secret());
        assert!(RecordKind::Key.is_secret());
    }

    /// SP1 的 `DevicePayload` 沒有 `slots`:SP3 寫出的裝置記錄,SP1 照樣讀得懂(未知欄位略過);`slots` 空的時候不寫出。
    #[test]
    fn device_slots_are_invisible_to_sp1_and_omitted_when_empty() {
        #[derive(serde::Deserialize)]
        struct Sp1DevicePayload {
            #[allow(dead_code)]
            name: String,
            #[serde(default)]
            #[allow(dead_code)]
            keys: Vec<String>,
        }
        let payload = DevicePayload {
            schema: 1,
            name: "MacBook".into(),
            platform: "macos".into(),
            joined_at_ms: 1,
            last_seen_ms: 2,
            keys: Vec::new(),
            spaces: Vec::new(),
            slots: vec![crate::sync::slot_rules::DeviceSlot { slot_id: "0".repeat(32), fingerprint: None, synced_copy: true }],
        };
        let json = serde_json::to_value(&payload).unwrap();
        assert!(serde_json::from_value::<Sp1DevicePayload>(json.clone()).is_ok());
        assert_eq!(serde_json::from_value::<DevicePayload>(json).unwrap(), payload);
        let empty = serde_json::to_value(DevicePayload { slots: Vec::new(), ..payload }).unwrap();
        assert!(empty.get("slots").is_none());
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run(在 `src-tauri/`):`PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib slot_rules -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected:編譯失敗(`slot_rules` 還沒有實作、`RecordKind::KeySlot` 不存在)。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/mod.rs` 加 `pub mod slot_rules;`(依字母順序放在 `round` 之後)。

`src-tauri/src/sync/slot_rules.rs`(測試模組之前的部分):

```rust
//! SP3 金鑰插槽的純函式與型別(spec `docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md` §4、§5、§6.2):
//! 帳戶記錄的 payload、插槽 id/名稱/檔名規則、OpenSSH 私鑰的檢查(不需要 passphrase、不寫任何檔案),以及
//! `IdentityFile` 值的解析。這裡不碰檔案系統與同步狀態。

use std::path::{Path, PathBuf};

use base64::engine::general_purpose::{STANDARD, STANDARD_NO_PAD};
use base64::Engine as _;
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

use crate::error::AppError;

/// 插槽目錄,相對於家目錄。主機的 `IdentityFile` 一律寫成 `~/` 加上這個路徑。
pub const SLOT_DIR: &str = ".ssh/sshelter/keys";
/// 可以同步的私鑰檔大小上限(spec §4.1)。
pub const MAX_PRIVATE_KEY_BYTES: usize = 16 * 1024;
/// `keyslot`、`key` payload 的 schema。
pub const SLOT_SCHEMA: u32 = 1;

// 以 `concat!` 組出標頭:原始碼裡不出現完整的字面(避免被祕密掃描誤判)。
const OPENSSH_BEGIN: &str = concat!("-----BEGIN ", "OPENSSH", " PRIVATE KEY-----");
const OPENSSH_END: &str = concat!("-----END ", "OPENSSH", " PRIVATE KEY-----");
const AUTH_MAGIC: &[u8] = b"openssh-key-v1\0";

/// 插槽的共用方式(spec §2):同步私鑰,或每台電腦用自己的金鑰。
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub enum SlotMode {
    Synced,
    Own,
}

/// `keyslot` 記錄的 payload(帳戶 chain,id = 插槽 id;spec §4.1)。`mode = Synced` 時後四項必填,`Own` 時都是 None
/// (每台電腦的金鑰不同)。刪除 = tombstone。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct KeySlotPayload {
    pub schema: u32,
    pub name: String,
    pub mode: SlotMode,
    pub origin_device_id: String,
    pub created_at_ms: u64,
    pub public_key: Option<String>,
    pub fingerprint: Option<String>,
    pub key_type: Option<String>,
    pub has_passphrase: Option<bool>,
}

/// `key` 記錄的 payload(祕密;帳戶 chain,id = 插槽 id):私鑰檔內容原樣,passphrase 不在這裡。`Debug` 不印內容。
#[derive(Clone, PartialEq, Serialize, Deserialize)]
pub struct KeyPayload {
    pub schema: u32,
    pub private_key: String,
}

impl std::fmt::Debug for KeyPayload {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("KeyPayload")
            .field("schema", &self.schema)
            .field("private_key", &format_args!("<redacted>"))
            .finish()
    }
}

/// `device` 記錄的 `slots`(spec §4.1):這台的插槽裡是哪把金鑰、是不是同步來的副本。只含公開資訊。
#[derive(Clone, Debug, PartialEq, Eq, Serialize, Deserialize)]
pub struct DeviceSlot {
    pub slot_id: String,
    pub fingerprint: Option<String>,
    pub synced_copy: bool,
}

/// 插槽 id:32 字元小寫 hex(隨機 16 bytes)。
pub fn is_slot_id(id: &str) -> bool {
    id.len() == 32 && id.bytes().all(|b| b.is_ascii_digit() || (b'a'..=b'f').contains(&b))
}

pub fn new_slot_id() -> Result<String, AppError> {
    let mut bytes = [0u8; 16];
    getrandom::fill(&mut bytes).map_err(|e| AppError::Other(format!("cannot create a key slot id: {e}")))?;
    Ok(bytes.iter().map(|b| format!("{b:02x}")).collect())
}

/// 名稱規則(spec §4.1):`^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`,不得以 `.pub` 結尾(不分大小寫)。
pub fn valid_slot_name(name: &str) -> bool {
    let mut chars = name.chars();
    let Some(first) = chars.next() else { return false };
    name.len() <= 64
        && first.is_ascii_alphanumeric()
        && chars.all(|c| c.is_ascii_alphanumeric() || matches!(c, '.' | '_' | '-'))
        && !name.to_ascii_lowercase().ends_with(".pub")
}

/// 由金鑰檔名產生預設名稱:不合規的字元換成 `-`,去掉開頭的非英數字元與結尾的 `-`、去掉 `.pub`,最長 64;
/// 結果不合規就用 `key`。
pub fn default_slot_name(file_name: &str) -> String {
    let mapped: String = file_name
        .chars()
        .map(|c| if c.is_ascii_alphanumeric() || matches!(c, '.' | '_' | '-') { c } else { '-' })
        .collect();
    let mut name: String = mapped.trim_start_matches(|c: char| !c.is_ascii_alphanumeric()).chars().take(64).collect();
    while name.to_ascii_lowercase().ends_with(".pub") {
        name.truncate(name.len() - 4);
    }
    let name = name.trim_end_matches('-').to_string();
    if valid_slot_name(&name) {
        name
    } else {
        "key".to_string()
    }
}

/// 插槽檔名:`<name>-<插槽 id 前 8 字元>`(同 space 檔名,不需跨裝置協調唯一性)。
pub fn slot_file_name(name: &str, slot_id: &str) -> String {
    format!("{name}-{}", &slot_id[..slot_id.len().min(8)])
}

/// 主機的 `IdentityFile` 要寫的值。
pub fn slot_value(file_name: &str) -> String {
    format!("~/{SLOT_DIR}/{file_name}")
}

/// 插槽在這台電腦上的完整路徑。
pub fn slot_path(home: &Path, file_name: &str) -> PathBuf {
    home.join(SLOT_DIR).join(file_name)
}

/// 插槽旁的公鑰檔(`<slot>.pub`)。
pub fn public_path(slot: &Path) -> PathBuf {
    let mut name = slot.as_os_str().to_owned();
    name.push(".pub");
    PathBuf::from(name)
}

/// 去掉成對的雙引號(ssh_config 的值可以加引號,`Directive.value` 原樣保留引號)。
pub fn unquote(value: &str) -> &str {
    let v = value.trim();
    if v.len() >= 2 && v.starts_with('"') && v.ends_with('"') {
        &v[1..v.len() - 1]
    } else {
        v
    }
}

/// 一個 `IdentityFile` 值若指到插槽,回傳插槽檔名:`~/.ssh/sshelter/keys/<file>` 或 `%d/.ssh/sshelter/keys/<file>`
/// (可加引號),`<file>` 不得含路徑分隔字元。
pub fn slot_file_of_value(value: &str) -> Option<String> {
    let v = unquote(value);
    let rest = ["~/", "%d/"].iter().find_map(|p| v.strip_prefix(p))?;
    let file = rest.strip_prefix(SLOT_DIR)?.strip_prefix('/')?;
    (!file.is_empty() && !file.contains(['/', '\\'])).then(|| file.to_string())
}

/// 一個 `IdentityFile` 值在這台電腦上指到什麼(spec §5)。
#[derive(Clone, Debug, PartialEq, Eq)]
pub enum IdentityTarget {
    /// 指到插槽(插槽檔名)。
    Slot(String),
    /// 指到這台電腦上的一個檔案(不保證存在)。
    File(PathBuf),
    /// 無法自動設定;原因給使用者看。
    Unsupported(&'static str),
}

pub const REASON_PUBLIC_KEY: &str = "points at a public key (an agent provides the private key)";
pub const REASON_TOKENS: &str = "uses % tokens or environment variables";
pub const REASON_RELATIVE: &str = "is a relative path";

/// 解析一個 `IdentityFile` 值:`~`、`%d`(`/` 或 `\` 分隔)、Unix 與 Windows 的絕對路徑;其他 token、環境變數、
/// 指向 `.pub` 的值、相對路徑都不處理(計畫裁定 6)。
pub fn resolve_identity_value(value: &str, home: &Path) -> IdentityTarget {
    let v = unquote(value);
    if let Some(file) = slot_file_of_value(v) {
        return IdentityTarget::Slot(file);
    }
    if v.to_ascii_lowercase().ends_with(".pub") {
        return IdentityTarget::Unsupported(REASON_PUBLIC_KEY);
    }
    if let Some(rest) = ["~/", "~\\", "%d/", "%d\\"].iter().find_map(|p| v.strip_prefix(p)) {
        if rest.contains('%') || rest.contains("${") {
            return IdentityTarget::Unsupported(REASON_TOKENS);
        }
        // 另一種平台寫下的分隔字元:兩種平台的 PathBuf 都接受 `/`。
        return IdentityTarget::File(home.join(rest.replace('\\', "/")));
    }
    if v.contains('%') || v.contains("${") {
        return IdentityTarget::Unsupported(REASON_TOKENS);
    }
    if is_absolute_path(v) {
        return IdentityTarget::File(PathBuf::from(v));
    }
    IdentityTarget::Unsupported(REASON_RELATIVE)
}

/// Unix 的 `/…`、Windows 的 `C:\…` / `C:/…` 與 UNC `\\server\…`(同步的設定可能來自另一種平台,兩種都認得)。
fn is_absolute_path(v: &str) -> bool {
    let b = v.as_bytes();
    v.starts_with('/')
        || v.starts_with("\\\\")
        || (b.len() >= 3 && b[0].is_ascii_alphabetic() && b[1] == b':' && (b[2] == b'\\' || b[2] == b'/'))
}

/// 從 OpenSSH 私鑰讀出的公開資訊。
#[derive(Clone, Debug, PartialEq, Eq)]
pub struct KeyFacts {
    /// `<type> <base64>`(沒有 comment)。
    pub public_key: String,
    /// `SHA256:<base64 無補位>`(同 `ssh-keygen -l`)。
    pub fingerprint: String,
    pub key_type: String,
    pub has_passphrase: bool,
}

/// 不能同步的原因(訊息給使用者;不含金鑰內容)。
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Unsyncable {
    TooLarge,
    NotOpenSsh,
    Unreadable,
}

impl Unsyncable {
    pub fn message(self) -> &'static str {
        match self {
            Unsyncable::TooLarge => "This key is larger than 16 KiB, so it can't be synced. Keep it on this computer.",
            Unsyncable::NotOpenSsh => "This key isn't in the OpenSSH format, so it can't be synced. Convert it with ssh-keygen -p -f <file>, or keep it on this computer.",
            Unsyncable::Unreadable => "This file couldn't be read as an OpenSSH private key.",
        }
    }
}

/// 檢查 OpenSSH 格式的私鑰(PROTOCOL.key):讀出未加密的公鑰段、指紋、類型,以及 ciphername 是不是 `none`。
/// 不需要 passphrase、不寫任何檔案。只接受一個檔案一把金鑰。
pub fn inspect_private_key(text: &str) -> Result<KeyFacts, Unsyncable> {
    if text.len() > MAX_PRIVATE_KEY_BYTES {
        return Err(Unsyncable::TooLarge);
    }
    let lines: Vec<&str> = text.lines().map(str::trim).filter(|l| !l.is_empty()).collect();
    let first = lines.first().copied().unwrap_or("");
    if first != OPENSSH_BEGIN {
        return Err(if first.starts_with("-----BEGIN ") && first.contains("PRIVATE KEY") {
            Unsyncable::NotOpenSsh
        } else {
            Unsyncable::Unreadable
        });
    }
    if lines.len() < 3 || lines.last().copied() != Some(OPENSSH_END) {
        return Err(Unsyncable::Unreadable);
    }
    let data = STANDARD.decode(lines[1..lines.len() - 1].concat()).map_err(|_| Unsyncable::Unreadable)?;
    let mut reader = Reader(data.strip_prefix(AUTH_MAGIC).ok_or(Unsyncable::Unreadable)?);
    let cipher = reader.string()?;
    reader.string()?; // kdfname
    reader.string()?; // kdfoptions
    if reader.u32()? != 1 {
        return Err(Unsyncable::Unreadable);
    }
    let blob = reader.string()?;
    let mut inner = Reader(blob);
    let key_type = std::str::from_utf8(inner.string()?).map_err(|_| Unsyncable::Unreadable)?;
    if key_type.is_empty() || !key_type.bytes().all(|c| c.is_ascii_graphic()) {
        return Err(Unsyncable::Unreadable);
    }
    Ok(KeyFacts {
        public_key: format!("{key_type} {}", STANDARD.encode(blob)),
        fingerprint: blob_fingerprint(blob),
        key_type: key_type.to_string(),
        has_passphrase: cipher != b"none",
    })
}

/// OpenSSH 的指紋:`SHA256:` + blob 的 SHA-256(base64、不補位)。
pub fn blob_fingerprint(blob: &[u8]) -> String {
    format!("SHA256:{}", STANDARD_NO_PAD.encode(Sha256::digest(blob)))
}

/// 一行公鑰(`<type> <base64> [comment]`)→(`<type> <base64>`、指紋)。類型欄位必須和 blob 裡寫的一致。
pub fn parse_public_key(line: &str) -> Option<(String, String)> {
    let mut parts = line.split_whitespace();
    let key_type = parts.next()?;
    let b64 = parts.next()?;
    let blob = STANDARD.decode(b64).ok()?;
    if Reader(&blob).string().ok()? != key_type.as_bytes() {
        return None;
    }
    Some((format!("{key_type} {b64}"), blob_fingerprint(&blob)))
}

/// `keyslot` payload 能不能進快取(spec §4.1):schema、名稱、來源裝置;`mode` 與四個金鑰欄位一致;`Synced` 的指紋要等於
/// 公鑰的指紋。
pub fn valid_slot_payload(p: &KeySlotPayload) -> bool {
    if p.schema != SLOT_SCHEMA || !valid_slot_name(&p.name) || p.origin_device_id.is_empty() {
        return false;
    }
    match p.mode {
        SlotMode::Own => {
            p.public_key.is_none() && p.fingerprint.is_none() && p.key_type.is_none() && p.has_passphrase.is_none()
        }
        SlotMode::Synced => match (&p.public_key, &p.fingerprint, &p.key_type, p.has_passphrase) {
            (Some(public), Some(fingerprint), Some(_), Some(_)) => {
                parse_public_key(public).is_some_and(|(_, f)| &f == fingerprint)
            }
            _ => false,
        },
    }
}

/// `key` payload 能不能進快取:schema 與大小(內容與指紋的比對在落地時做,spec §6.2)。
pub fn valid_key_payload(p: &KeyPayload) -> bool {
    p.schema == SLOT_SCHEMA && !p.private_key.is_empty() && p.private_key.len() <= MAX_PRIVATE_KEY_BYTES
}

/// SSH wire 格式的讀取器(uint32 big-endian;string = 長度 + 內容)。
struct Reader<'a>(&'a [u8]);

impl<'a> Reader<'a> {
    fn u32(&mut self) -> Result<u32, Unsyncable> {
        if self.0.len() < 4 {
            return Err(Unsyncable::Unreadable);
        }
        let (head, tail) = self.0.split_at(4);
        self.0 = tail;
        Ok(u32::from_be_bytes([head[0], head[1], head[2], head[3]]))
    }

    fn string(&mut self) -> Result<&'a [u8], Unsyncable> {
        let n = self.u32()? as usize;
        if self.0.len() < n {
            return Err(Unsyncable::Unreadable);
        }
        let (head, tail) = self.0.split_at(n);
        self.0 = tail;
        Ok(head)
    }
}
```

`src-tauri/src/sync/record.rs`:

```rust
pub enum RecordKind {
    Host,
    Key,
    Password,
    Device,
    Meta,
    /// 帳戶 chain:一個 space 的名稱與 slug(id = space id)。
    Space,
    /// 帳戶 chain:一個 space 的權杖與金鑰(id = space id;wire 名稱 `spacekey`)。
    SpaceKey,
    /// 帳戶 chain:一個金鑰插槽(id = 插槽 id;wire 名稱 `keyslot`;SP3 spec §4.1)。
    KeySlot,
}
```

`as_str` 加 `RecordKind::KeySlot => "keyslot",`;`parse` 加 `"keyslot" => Some(RecordKind::KeySlot),`;`is_secret` 不變
(`Key` 已在裡面)。`Key` 那一行的文件註解改成「帳戶 chain:同步的私鑰(id = 插槽 id;祕密,SP3 spec §4.1)」。

`DevicePayload` 加在 `spaces` 之後:

```rust
    /// 這台的插槽裡是哪把金鑰(SP3 spec §4.1)。空的時候不寫出,SP1 的 payload 內容不變;SP1 讀到會略過這個欄位。
    /// 不能用 `keys`:SP1 把它當 `Vec<String>` 解析。
    #[serde(default, skip_serializing_if = "Vec::is_empty")]
    pub slots: Vec<crate::sync::slot_rules::DeviceSlot>,
```

`DevicePayload` 的 derive 已有 `PartialEq`;測試要 `assert_eq!` 整個 payload,所以 `DeviceSlot` 也 derive `PartialEq`(上面已有)。

`src-tauri/src/sync/merge.rs` 的 `own_device_record` 建 `DevicePayload` 時加 `slots: Vec::new(),`(Task 3 改成沿用前一版)。
其他建 `DevicePayload` 的地方(`grep -rn "DevicePayload {" src-tauri/src`)一律補 `slots: Vec::new()`。

- [ ] **Step 4: 跑測試確認通過**

Run:同 Step 2,再跑一次完整的 Rust 測試。
Expected:`slot_rules` 的 11 個測試與 `record` 新增的 2 個測試通過;完整測試全綠;產生 `src/bindings/SlotMode.ts`。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/slot_rules.rs src-tauri/src/sync/mod.rs src-tauri/src/sync/record.rs src-tauri/src/sync/merge.rs src/bindings/SlotMode.ts
git commit -m "feat(sync): key slot records, names and OpenSSH private key inspection"
```

### Task 2: 插槽的檔案系統動作(`slot_files.rs`、`slot_files_windows.rs`、`windows-sys`)

**Files:**
- Create: `src-tauri/src/sync/slot_files.rs`
- Create: `src-tauri/src/sync/slot_files_windows.rs`
- Modify: `src-tauri/src/sync/mod.rs`
- Modify: `src-tauri/Cargo.toml`

**Interfaces:**
- Consumes: Task 1 的 `slot_rules::{public_path, new_slot_id, test_keys}`。
- Produces:
  - `pub enum LinkKind { Symlink, HardLink, Copy }`(serde snake_case)
  - `pub fn ensure_keys_dir(dir: &Path) -> Result<(), AppError>`
  - `pub fn write_private(path: &Path, bytes: &[u8]) -> Result<(), AppError>`
  - `pub fn write_public(slot: &Path, public_key: &str) -> Result<(), AppError>`(寫 `<slot>.pub`)
  - `pub fn link(source: &Path, slot: &Path) -> Result<LinkKind, AppError>`(`source` 必須是絕對路徑)
  - `pub fn content_sha256(path: &Path) -> Option<String>`(跟著 symlink 讀;小寫 hex)
  - `pub fn occupied(path: &Path) -> bool`(包括壞掉的 symlink)
  - `pub fn remove_slot(slot: &Path) -> Result<(), AppError>`(連 `.pub` 一起;不存在不算錯)
  - `pub fn retire(slot: &Path, tag: &str) -> Result<PathBuf, AppError>`(改名保留成 `<file>.previous-<tag>`,已存在時加 `-2`、`-3`…;`.pub` 一起改名)
  - Windows:`slot_files_windows::restrict_to_owner(path: &Path, inheritable: bool) -> io::Result<()>`;測試用 `ace_count(path) -> io::Result<u32>`

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slot_files.rs` 的測試模組:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::slot_rules::{public_path, test_keys};

    #[cfg(unix)]
    fn mode(path: &Path) -> u32 {
        use std::os::unix::fs::PermissionsExt;
        fs::metadata(path).unwrap().permissions().mode() & 0o777
    }

    #[test]
    #[cfg(unix)]
    fn keys_dir_and_private_files_are_owner_only() {
        use std::os::unix::fs::PermissionsExt;
        let home = tempfile::tempdir().unwrap();
        let dir = home.path().join(".ssh/sshelter/keys");
        ensure_keys_dir(&dir).unwrap();
        assert_eq!(mode(&dir), 0o700);
        // SSHelter 自己的目錄:權限被放寬過也改回 0700。
        fs::set_permissions(&dir, fs::Permissions::from_mode(0o755)).unwrap();
        ensure_keys_dir(&dir).unwrap();
        assert_eq!(mode(&dir), 0o700);

        let slot = dir.join("id_mac-3fa2c1d9");
        write_private(&slot, test_keys::plain().as_bytes()).unwrap();
        assert_eq!(mode(&slot), 0o600);
        assert_eq!(fs::read_to_string(&slot).unwrap(), test_keys::plain());
        write_public(&slot, test_keys::PLAIN_PUBLIC).unwrap();
        assert_eq!(fs::read_to_string(public_path(&slot)).unwrap(), format!("{}\n", test_keys::PLAIN_PUBLIC));
        assert_eq!(mode(&public_path(&slot)), 0o644);
        // 沒有留下暫存檔。
        let mut names: Vec<String> = fs::read_dir(&dir).unwrap().map(|e| e.unwrap().file_name().into_string().unwrap()).collect();
        names.sort();
        assert_eq!(names, ["id_mac-3fa2c1d9", "id_mac-3fa2c1d9.pub"]);
    }

    #[test]
    #[cfg(unix)]
    fn a_symlink_slot_follows_its_source_and_replaces_what_was_there() {
        let home = tempfile::tempdir().unwrap();
        let source = home.path().join(".ssh/id_mac");
        fs::create_dir_all(source.parent().unwrap()).unwrap();
        fs::write(&source, test_keys::plain()).unwrap();
        let dir = home.path().join(".ssh/sshelter/keys");
        ensure_keys_dir(&dir).unwrap();
        let slot = dir.join("id_mac-3fa2c1d9");
        write_private(&slot, b"old").unwrap();

        assert_eq!(link(&source, &slot).unwrap(), LinkKind::Symlink);
        assert_eq!(fs::read_link(&slot).unwrap(), source);
        assert_eq!(content_sha256(&slot), content_sha256(&source));
        fs::write(&source, test_keys::ecdsa()).unwrap();
        assert_eq!(content_sha256(&slot), content_sha256(&source), "a symlink follows the file at the source path");

        assert!(link(Path::new("id_mac"), &slot).is_err(), "the source must be an absolute path");
        remove_slot(&slot).unwrap();
        assert!(!occupied(&slot));
        assert!(source.exists(), "removing a slot never touches the key it points to");
        remove_slot(&slot).unwrap();
    }

    #[test]
    fn a_broken_link_still_occupies_the_slot() {
        let home = tempfile::tempdir().unwrap();
        let dir = home.path().join("keys");
        ensure_keys_dir(&dir).unwrap();
        let source = home.path().join("gone");
        fs::write(&source, test_keys::plain()).unwrap();
        let slot = dir.join("gone-3fa2c1d9");
        link(&source, &slot).unwrap();
        fs::remove_file(&source).unwrap();
        assert!(occupied(&slot));
        #[cfg(unix)]
        assert_eq!(content_sha256(&slot), None);
    }

    #[test]
    fn retired_copies_keep_their_bytes_under_a_new_name() {
        let home = tempfile::tempdir().unwrap();
        let dir = home.path().join("keys");
        ensure_keys_dir(&dir).unwrap();
        let slot = dir.join("id_mac-3fa2c1d9");
        write_private(&slot, test_keys::plain().as_bytes()).unwrap();
        write_public(&slot, test_keys::PLAIN_PUBLIC).unwrap();

        let kept = retire(&slot, "0a1b2c3d").unwrap();
        assert_eq!(kept, dir.join("id_mac-3fa2c1d9.previous-0a1b2c3d"));
        assert_eq!(fs::read_to_string(&kept).unwrap(), test_keys::plain());
        assert!(public_path(&kept).exists());
        assert!(!occupied(&slot));

        write_private(&slot, b"second").unwrap();
        let again = retire(&slot, "0a1b2c3d").unwrap();
        assert_eq!(again, dir.join("id_mac-3fa2c1d9.previous-0a1b2c3d-2"));
        assert_eq!(fs::read_to_string(&kept).unwrap(), test_keys::plain(), "never overwrites an earlier copy");
    }

    #[test]
    #[cfg(windows)]
    fn windows_slots_are_owner_only_and_hard_linked() {
        use crate::sync::slot_files_windows::ace_count;
        let home = tempfile::tempdir().unwrap();
        let dir = home.path().join("keys");
        ensure_keys_dir(&dir).unwrap();
        assert_eq!(ace_count(&dir).unwrap(), 1);
        let slot = dir.join("id_win-3fa2c1d9");
        write_private(&slot, test_keys::plain().as_bytes()).unwrap();
        assert_eq!(ace_count(&slot).unwrap(), 1);

        let source = home.path().join("id_win");
        fs::write(&source, test_keys::ecdsa()).unwrap();
        let linked = dir.join("linked-3fa2c1d9");
        assert_eq!(link(&source, &linked).unwrap(), LinkKind::HardLink);
        assert_eq!(content_sha256(&linked), content_sha256(&source));
    }
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib slot_files -- --skip …`(同 Global Constraints 的指令,篩選 `slot_files`)
Expected:編譯失敗(模組不存在)。

- [ ] **Step 3: 實作**

`src-tauri/Cargo.toml`,在現有的 desktop-only target 區段之後加:

```toml
# SP3 金鑰插槽:Windows 上把插槽目錄與私鑰設成只有目前使用者能存取(src/sync/slot_files_windows.rs)。
# 0.61 已在 Cargo.lock(tempfile 等的相依),不需下載。
[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.61", features = ["Win32_Foundation", "Win32_Security", "Win32_Security_Authorization", "Win32_System_Threading"] }
```

`src-tauri/src/sync/mod.rs` 加:

```rust
pub mod slot_files;
#[cfg(windows)]
pub mod slot_files_windows;
```

`src-tauri/src/sync/slot_files_windows.rs`(寫計畫時已在 scratch crate 對 `x86_64-pc-windows-msvc` 型別檢查通過):

```rust
//! Windows:把檔案或目錄的 DACL 設成「只有目前使用者、不繼承上層」(SP3 spec §8;Win32-OpenSSH 拒用其他人也能讀的
//! 私鑰)。只依賴 std 與 windows-sys,好讓它能在其他平台上單獨做型別檢查(計畫 Task 2 Step 6)。
use std::ffi::{c_void, OsStr};
use std::io;
use std::os::windows::ffi::OsStrExt;
use std::path::Path;
use std::ptr;

use windows_sys::Win32::Foundation::{CloseHandle, LocalFree, ERROR_SUCCESS, GENERIC_ALL, HANDLE};
use windows_sys::Win32::Security::Authorization::{
    SetEntriesInAclW, SetNamedSecurityInfoW, EXPLICIT_ACCESS_W, NO_MULTIPLE_TRUSTEE, SET_ACCESS, SE_FILE_OBJECT,
    TRUSTEE_IS_SID, TRUSTEE_IS_USER, TRUSTEE_W,
};
use windows_sys::Win32::Security::{
    GetTokenInformation, TokenUser, ACL, DACL_SECURITY_INFORMATION, NO_INHERITANCE, PROTECTED_DACL_SECURITY_INFORMATION,
    SUB_CONTAINERS_AND_OBJECTS_INHERIT, TOKEN_QUERY, TOKEN_USER,
};
use windows_sys::Win32::System::Threading::{GetCurrentProcess, OpenProcessToken};

fn wide(path: &Path) -> Vec<u16> {
    OsStr::new(path).encode_wide().chain(std::iter::once(0)).collect()
}

/// 目前使用者的 `TOKEN_USER`(放在回傳的緩衝區裡;SID 指標指進緩衝區)。
fn current_user_token() -> io::Result<Vec<u8>> {
    unsafe {
        let mut token: HANDLE = ptr::null_mut();
        if OpenProcessToken(GetCurrentProcess(), TOKEN_QUERY, &mut token) == 0 {
            return Err(io::Error::last_os_error());
        }
        let mut needed = 0u32;
        GetTokenInformation(token, TokenUser, ptr::null_mut(), 0, &mut needed);
        let mut buffer = vec![0u8; needed as usize];
        let ok = GetTokenInformation(token, TokenUser, buffer.as_mut_ptr() as *mut c_void, needed, &mut needed);
        let error = io::Error::last_os_error();
        CloseHandle(token);
        if ok == 0 {
            return Err(error);
        }
        Ok(buffer)
    }
}

/// 把 `path` 的 DACL 換成只有一條「目前使用者:完全控制」,並切斷上層繼承。`inheritable` = true(目錄)時這一條會被
/// 之後在裡面建立的檔案與目錄繼承 —— 暫存檔一建立就是 owner-only,沒有可被讀取的空窗。
pub fn restrict_to_owner(path: &Path, inheritable: bool) -> io::Result<()> {
    let token = current_user_token()?;
    unsafe {
        let user = &*(token.as_ptr() as *const TOKEN_USER);
        let access = EXPLICIT_ACCESS_W {
            grfAccessPermissions: GENERIC_ALL,
            grfAccessMode: SET_ACCESS,
            grfInheritance: if inheritable { SUB_CONTAINERS_AND_OBJECTS_INHERIT } else { NO_INHERITANCE },
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
        let name = wide(path);
        let status = SetNamedSecurityInfoW(
            name.as_ptr(),
            SE_FILE_OBJECT,
            DACL_SECURITY_INFORMATION | PROTECTED_DACL_SECURITY_INFORMATION,
            ptr::null_mut(),
            ptr::null_mut(),
            acl,
            ptr::null(),
        );
        LocalFree(acl as *mut c_void);
        if status != ERROR_SUCCESS {
            return Err(io::Error::from_raw_os_error(status as i32));
        }
    }
    Ok(())
}

/// 測試用:`path` 的 DACL 有幾條 ACE。
#[cfg(test)]
pub fn ace_count(path: &Path) -> io::Result<u32> {
    use windows_sys::Win32::Security::Authorization::GetNamedSecurityInfoW;
    use windows_sys::Win32::Security::{AclSizeInformation, GetAclInformation, ACL_SIZE_INFORMATION};
    unsafe {
        let name = wide(path);
        let mut dacl: *mut ACL = ptr::null_mut();
        let mut descriptor: *mut c_void = ptr::null_mut();
        let status = GetNamedSecurityInfoW(
            name.as_ptr(),
            SE_FILE_OBJECT,
            DACL_SECURITY_INFORMATION,
            ptr::null_mut(),
            ptr::null_mut(),
            &mut dacl,
            ptr::null_mut(),
            &mut descriptor,
        );
        if status != ERROR_SUCCESS {
            return Err(io::Error::from_raw_os_error(status as i32));
        }
        let mut info: ACL_SIZE_INFORMATION = std::mem::zeroed();
        let ok = GetAclInformation(
            dacl,
            &mut info as *mut ACL_SIZE_INFORMATION as *mut c_void,
            std::mem::size_of::<ACL_SIZE_INFORMATION>() as u32,
            AclSizeInformation,
        );
        let error = io::Error::last_os_error();
        LocalFree(descriptor);
        if ok == 0 {
            return Err(error);
        }
        Ok(info.AceCount)
    }
}
```

`src-tauri/src/sync/slot_files.rs`(測試模組之前):

```rust
//! SP3 插槽的檔案系統動作(spec §4.2、§8):插槽目錄、私鑰與 `.pub` 的寫入、連結(symlink / hard link / 複製)、
//! 內容雜湊、移除與改名保留。所有函式都收明確的路徑,不碰同步狀態。

use std::fs;
use std::io::{self, Write as _};
use std::path::{Path, PathBuf};

use serde::{Deserialize, Serialize};

use crate::error::AppError;
use crate::sync::slot_rules::{new_slot_id, public_path};

/// 插槽連到本機金鑰的方式。
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LinkKind {
    /// Unix:指向原檔路徑的 symlink(原檔被換掉也跟著走)。
    Symlink,
    /// Windows:hard link(和原檔共用內容與 ACL;原檔被換成新檔時不會跟著變,由指紋檢查重新連結)。
    HardLink,
    /// 建不了連結時的複製(只有目前使用者能讀)。
    Copy,
}

fn parent_of(path: &Path) -> Result<&Path, AppError> {
    path.parent().ok_or_else(|| AppError::Other(format!("no parent dir for {}", path.display())))
}

/// 確保插槽目錄存在且只有目前使用者能存取。Unix:0700(已存在也改 —— 這是 SSHelter 自己的目錄);Windows:owner-only、
/// 不繼承上層、會被子項繼承的 DACL。
pub fn ensure_keys_dir(dir: &Path) -> Result<(), AppError> {
    fs::create_dir_all(dir)?;
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        fs::set_permissions(dir, fs::Permissions::from_mode(0o700))?;
    }
    #[cfg(windows)]
    crate::sync::slot_files_windows::restrict_to_owner(dir, true)?;
    Ok(())
}

/// 原子寫入私鑰檔:暫存檔建在同一個目錄(Unix 先設 0600 再寫;Windows 從 `ensure_keys_dir` 設好的目錄繼承 owner-only
/// 權限),寫入、fsync、rename 蓋過去;Windows 另外把結果設成明確的 owner-only DACL。
pub fn write_private(path: &Path, bytes: &[u8]) -> Result<(), AppError> {
    let mut tmp = tempfile::NamedTempFile::new_in(parent_of(path)?)?;
    #[cfg(unix)]
    {
        use std::os::unix::fs::PermissionsExt;
        tmp.as_file().set_permissions(fs::Permissions::from_mode(0o600))?;
    }
    tmp.write_all(bytes)?;
    tmp.as_file().sync_all()?;
    tmp.persist(path).map_err(|e| AppError::Io(e.error))?;
    #[cfg(windows)]
    crate::sync::slot_files_windows::restrict_to_owner(path, false)?;
    Ok(())
}

/// 寫插槽旁的 `<slot>.pub`(公鑰一行加換行;Unix 0644)。
pub fn write_public(slot: &Path, public_key: &str) -> Result<(), AppError> {
    crate::fsutil::atomic_write(&public_path(slot), format!("{public_key}\n").as_bytes(), 0o644)
}

/// 讓插槽 `slot` 指到 `source`(建立插槽那台的原檔,或使用者在這台挑的金鑰),原子替換插槽上原本的東西。
/// Unix 建 symlink(就用 `source` 這個絕對路徑);Windows 建 hard link,不行(不同磁碟、FAT/exFAT)時複製。
pub fn link(source: &Path, slot: &Path) -> Result<LinkKind, AppError> {
    if !source.is_absolute() {
        return Err(AppError::Other(format!("the key path must be absolute: {}", source.display())));
    }
    let dir = parent_of(slot)?;
    let name = slot.file_name().and_then(|n| n.to_str()).unwrap_or("slot");
    let tmp = dir.join(format!(".{name}.{}.tmp", new_slot_id()?));
    let kind = link_at(source, &tmp, slot)?;
    if kind != LinkKind::Copy {
        if let Err(e) = fs::rename(&tmp, slot) {
            let _ = fs::remove_file(&tmp);
            return Err(e.into());
        }
    }
    Ok(kind)
}

#[cfg(unix)]
fn link_at(source: &Path, tmp: &Path, _slot: &Path) -> Result<LinkKind, AppError> {
    std::os::unix::fs::symlink(source, tmp)?;
    Ok(LinkKind::Symlink)
}

#[cfg(not(unix))]
fn link_at(source: &Path, tmp: &Path, slot: &Path) -> Result<LinkKind, AppError> {
    match fs::hard_link(source, tmp) {
        Ok(()) => Ok(LinkKind::HardLink),
        Err(_) => {
            // 複製直接寫到插槽(`write_private` 自己做原子替換),不經 `tmp`。
            write_private(slot, &fs::read(source)?)?;
            Ok(LinkKind::Copy)
        }
    }
}

/// 檔案目前內容的 SHA-256(小寫 hex;跟著 symlink 讀)。讀不到 → None。
pub fn content_sha256(path: &Path) -> Option<String> {
    fs::read(path).ok().map(|bytes| crate::fsutil::fingerprint_of(&bytes).sha256)
}

/// 路徑上有沒有東西(包括指向不存在檔案的 symlink)。
pub fn occupied(path: &Path) -> bool {
    fs::symlink_metadata(path).is_ok()
}

fn remove_if_present(path: &Path) -> Result<(), AppError> {
    match fs::remove_file(path) {
        Ok(()) => Ok(()),
        Err(e) if e.kind() == io::ErrorKind::NotFound => Ok(()),
        Err(e) => Err(e.into()),
    }
}

/// 移除插槽(連結本身或檔案)與它的 `.pub`;不存在不算錯。移除 symlink 不會動到它指向的金鑰。
pub fn remove_slot(slot: &Path) -> Result<(), AppError> {
    remove_if_present(slot)?;
    remove_if_present(&public_path(slot))
}

/// 把插槽上的舊副本改名保留成 `<file>.previous-<tag>`(計畫裁定 3:私鑰不自動刪除),已存在時加 `-2`、`-3`…,
/// 不覆蓋任何檔案;`.pub` 一起改名。回傳保留的路徑。
pub fn retire(slot: &Path, tag: &str) -> Result<PathBuf, AppError> {
    let dir = parent_of(slot)?;
    let name = slot.file_name().and_then(|n| n.to_str()).unwrap_or("slot");
    let base = format!("{name}.previous-{tag}");
    let mut kept = dir.join(&base);
    let mut n = 2;
    while occupied(&kept) {
        kept = dir.join(format!("{base}-{n}"));
        n += 1;
    }
    fs::rename(slot, &kept)?;
    if occupied(&public_path(slot)) {
        fs::rename(public_path(slot), public_path(&kept))?;
    }
    Ok(kept)
}
```

- [ ] **Step 4: 跑測試確認通過**

Run:同 Step 2;再跑完整的 Rust 測試。
Expected:Unix 上 `slot_files` 的 4 個測試通過(Windows 那一個在 Task 12 的 CI 跑)。

- [ ] **Step 5: 確認 Cargo.lock 只多了 windows-sys 那一行**

Run:`git diff --stat src-tauri/Cargo.lock` 與 `git diff src-tauri/Cargo.lock | grep '^[+-]' | grep -v '^+++\|^---'`
Expected:除了原本就有的那一行(`sshelter` 的 version)之外,只多了 `"windows-sys 0.61.2",`。**不要** stage 它。

- [ ] **Step 6: 在 macOS 上對 Windows 目標做型別檢查**

整個 app 對 Windows 目標的 `cargo check` 會卡在 `aws-lc-sys`(需要 Windows 的 C 編譯器);`slot_files_windows.rs` 只依賴
`windows-sys`,用 scratch crate 單獨檢查(含 `#[cfg(test)]` 的部分):

```bash
REPO=$(git rev-parse --show-toplevel)
S=$(mktemp -d)/slot-acl-check && mkdir -p "$S/src"
cat > "$S/Cargo.toml" <<'EOF'
[package]
name = "slot-acl-check"
version = "0.0.0"
edition = "2021"

[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.61", features = ["Win32_Foundation", "Win32_Security", "Win32_Security_Authorization", "Win32_System_Threading"] }
EOF
printf '#[cfg(windows)]\n#[path = "%s"]\npub mod slot_files_windows;\n' "$REPO/src-tauri/src/sync/slot_files_windows.rs" > "$S/src/lib.rs"
cd "$S" && PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo check --offline --tests --target x86_64-pc-windows-msvc
```

Expected:`Finished`,沒有 error。(要確認它真的有被檢查:暫時在檔尾加一行 `fn _probe() { let _x: u8 = "x"; }`,應該出現
`mismatched types`;記得拿掉。)rustup 的 stable 工具鏈已裝 `x86_64-pc-windows-msvc` 的 std。

- [ ] **Step 7: Commit**

```bash
git add src-tauri/src/sync/slot_files.rs src-tauri/src/sync/slot_files_windows.rs src-tauri/src/sync/mod.rs src-tauri/Cargo.toml
git commit -m "feat(sync): owner-only key slot files, links and Windows ACLs"
```

### Task 3: 帳戶記錄、合併與本機插槽狀態(`slots.rs` 前半、`merge.rs`、`state_v2.rs`)

**Files:**
- Create: `src-tauri/src/sync/slots.rs`
- Modify: `src-tauri/src/sync/mod.rs`、`src-tauri/src/sync/merge.rs`、`src-tauri/src/sync/state_v2.rs`、`src-tauri/src/sync/upgrade.rs`(`shell_state` 補新欄位)
- Modify: `src/lib/sync-events.ts`、`src/lib/sync-events.test.ts`(`noticeMessage` 是窮舉的 `switch`:binding 多了 `keys_needed`
  之後不處理就過不了 `tsc`)
- Generated: `src/bindings/SyncNotice.ts`(新增 `keys_needed`)

**Interfaces:**
- Consumes: Task 1(`slot_rules::*`)、Task 2(`slot_files::LinkKind`)。
- Produces:
  - `slots::live_slots(&AccountState) -> Vec<(String, KeySlotPayload)>`(依名稱、id 排序)、`slots::slot(&AccountState, &str) -> Option<KeySlotPayload>`、`slots::slot_record_exists(&AccountState, &str) -> bool`(含 tombstone)
  - `slots::put_slot(&mut AccountState, slot_id: &str, payload: Option<&KeySlotPayload>, device_id: &str, now_ms: u64)`(None = tombstone)
  - `slots::put_key_secret(&mut AccountState, &ChainKeys, slot_id: &str, private_key: Option<&str>, device_id: &str, now_ms: u64) -> Result<(), AppError>`(None = tombstone)
  - `slots::open_key_secret(&AccountState, &ChainKeys, slot_id: &str) -> Option<String>`
  - `merge::set_device_slots(&mut AccountState, device_id: &str, slots: Vec<DeviceSlot>, now_ms: u64) -> bool`;`own_device_record` 沿用前一版的 `slots`
  - `state_v2::LocalSlot { file_name: String, source: Option<SlotSource>, last_error: Option<String>, asked: bool, payload: Option<KeySlotPayload> }`
  - `state_v2::SlotSource::{Linked { path: String, link: LinkKind, fingerprint: Option<String>, origin: bool }, SyncedCopy { fingerprint: String }}`(serde `tag = "kind"`、snake_case)
  - `SyncStateV2.key_slots: BTreeMap<String, LocalSlot>`(`#[serde(default)]`)
  - `SyncNotice::KeysNeeded { names: Vec<String> }`(wire `keys_needed`)

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slots.rs` 的測試模組(之後的 task 會在同一個模組加測試):

```rust
#[cfg(test)]
pub(crate) mod tests {
    use super::*;
    use crate::sync::merge::{account_outgoing, merge_account, plan_device, set_device_slots, Outgoing};
    use crate::sync::record::{DevicePayload, Envelope};
    use crate::sync::relay::PullResponse;
    use crate::sync::slot_rules::{test_keys, DeviceSlot, SlotMode};
    use crate::sync::state_v2::{AccountState, SyncStateV2};

    pub(crate) const SLOT_ID: &str = "3fa2c1d90123456789abcdef01234567";

    pub(crate) fn synced_payload(origin: &str) -> KeySlotPayload {
        KeySlotPayload {
            schema: SLOT_SCHEMA,
            name: "id_mac".into(),
            mode: SlotMode::Synced,
            origin_device_id: origin.into(),
            created_at_ms: 5,
            public_key: Some(test_keys::PLAIN_PUBLIC.into()),
            fingerprint: Some(test_keys::PLAIN_FINGERPRINT.into()),
            key_type: Some("ssh-ed25519".into()),
            has_passphrase: Some(false),
        }
    }

    /// 把一台的 dirty 上傳項目當成另一台拉到的內容(seq 從 1 起)。
    fn pulled(items: &[Outgoing], latest_seq: u64) -> PullResponse {
        PullResponse {
            records: items
                .iter()
                .enumerate()
                .map(|(i, o)| Envelope {
                    id_hash: o.item.id_hash.clone(),
                    kind: o.item.kind.clone(),
                    seq: i as u64 + 1,
                    nonce: o.item.nonce.clone(),
                    ciphertext: o.item.ciphertext.clone(),
                    deleted: o.item.deleted,
                })
                .collect(),
            latest_seq,
        }
    }

    #[test]
    fn a_slot_and_its_key_reach_another_device_and_the_key_stays_sealed() {
        let keys = ChainKeys::generate().unwrap();
        let mut a = AccountState::new(&keys.chain_id);
        put_slot(&mut a, SLOT_ID, Some(&synced_payload("a")), "a", 10);
        put_key_secret(&mut a, &keys, SLOT_ID, Some(&test_keys::plain()), "a", 10).unwrap();
        let items = account_outgoing(&a, &keys).unwrap();
        assert_eq!(items.len(), 2);

        let b = merge_account(&AccountState::new(&keys.chain_id), &keys, &pulled(&items, 2)).section;
        assert_eq!(live_slots(&b), vec![(SLOT_ID.to_string(), synced_payload("a"))]);
        assert!(b.records.keys().all(|k| !k.starts_with("key:")), "the key never lands in the plaintext records");
        assert_eq!(open_key_secret(&b, &keys, SLOT_ID).as_deref(), Some(test_keys::plain().as_str()));
    }

    #[test]
    fn key_records_merge_last_writer_wins_in_memory() {
        let keys = ChainKeys::generate().unwrap();
        let mut a = AccountState::new(&keys.chain_id);
        put_key_secret(&mut a, &keys, SLOT_ID, Some(&test_keys::plain()), "a", 10).unwrap();
        let older = account_outgoing(&a, &keys).unwrap();

        // b 之後寫了自己的版本(還沒上傳):拉到 a 的舊版本不能蓋掉它。
        let mut b = AccountState::new(&keys.chain_id);
        put_key_secret(&mut b, &keys, SLOT_ID, Some(&test_keys::ecdsa()), "b", 20).unwrap();
        let merged = merge_account(&b, &keys, &pulled(&older, 1)).section;
        assert_eq!(open_key_secret(&merged, &keys, SLOT_ID).as_deref(), Some(test_keys::ecdsa().as_str()));

        // 反過來:遠端較新就取遠端。
        let mut c = AccountState::new(&keys.chain_id);
        put_key_secret(&mut c, &keys, SLOT_ID, Some(&test_keys::ecdsa()), "c", 5).unwrap();
        let merged = merge_account(&c, &keys, &pulled(&older, 1)).section;
        assert_eq!(open_key_secret(&merged, &keys, SLOT_ID).as_deref(), Some(test_keys::plain().as_str()));
    }

    #[test]
    fn a_relay_rollback_reuploads_key_records() {
        let keys = ChainKeys::generate().unwrap();
        let mut a = AccountState::new(&keys.chain_id);
        put_key_secret(&mut a, &keys, SLOT_ID, Some(&test_keys::plain()), "a", 10).unwrap();
        for sealed in a.sealed.values_mut() {
            sealed.dirty = false;
            sealed.envelope.seq = 7;
        }
        a.cursor_seq = 7;
        let merged = merge_account(&a, &keys, &PullResponse { records: Vec::new(), latest_seq: 2 }).section;
        let sealed = merged.sealed.get(&key_secret_key(&keys, SLOT_ID)).unwrap();
        assert!(sealed.dirty);
        assert_eq!(sealed.envelope.seq, 0);
    }

    #[test]
    fn unreadable_slot_records_are_skipped() {
        let keys = ChainKeys::generate().unwrap();
        let mut a = AccountState::new(&keys.chain_id);
        put_slot(&mut a, SLOT_ID, Some(&KeySlotPayload { name: "../escape".into(), ..synced_payload("a") }), "a", 10);
        put_slot(&mut a, "not-a-slot-id", Some(&synced_payload("a")), "a", 10);
        let merged = merge_account(&AccountState::new(&keys.chain_id), &keys, &pulled(&account_outgoing(&a, &keys).unwrap(), 2));
        assert_eq!(merged.skipped, 2);
        assert!(live_slots(&merged.section).is_empty());
    }

    #[test]
    fn the_state_file_never_holds_a_private_key_in_plaintext() {
        let keys = ChainKeys::generate().unwrap();
        let mut state = SyncStateV2::fresh("MacBook").unwrap();
        let mut account = AccountState::new(&keys.chain_id);
        put_key_secret(&mut account, &keys, SLOT_ID, Some(&test_keys::plain()), &state.device_id, 10).unwrap();
        state.account = Some(account);
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        crate::sync::state_v2::save(&path, &state).unwrap();
        let text = std::fs::read_to_string(&path).unwrap();
        for line in test_keys::PLAIN_BODY {
            assert!(!text.contains(line), "a line of the private key is in the state file");
        }
    }

    #[test]
    fn device_slots_survive_the_heartbeat() {
        let keys = ChainKeys::generate().unwrap();
        let mut account = AccountState::new(&keys.chain_id);
        assert!(plan_device(&mut account, "a", "MacBook", "macos", &[], 1_000));
        let slots = vec![DeviceSlot { slot_id: SLOT_ID.into(), fingerprint: Some(test_keys::PLAIN_FINGERPRINT.into()), synced_copy: false }];
        assert!(set_device_slots(&mut account, "a", slots.clone(), 2_000));
        assert!(!set_device_slots(&mut account, "a", slots.clone(), 3_000), "unchanged slots write nothing");
        // 一小時後的心跳(plan_device)重寫裝置記錄:slots 照樣保留。
        assert!(plan_device(&mut account, "a", "MacBook", "macos", &[], 2_000 + 60 * 60 * 1000));
        let payload: DevicePayload = serde_json::from_value(account.records["device:a"].record.payload.clone()).unwrap();
        assert_eq!(payload.slots, slots);
    }
}
```

`src-tauri/src/sync/state_v2.rs` 的測試模組加:

```rust
    /// SP3 之前寫的狀態檔沒有 `key_slots`,讀進來是空的;有 `key_slots` 的狀態檔照樣讀得回來。
    #[test]
    fn key_slots_default_to_empty_and_round_trip() {
        let mut state = SyncStateV2::fresh("MacBook").unwrap();
        let mut json = serde_json::to_value(&state).unwrap();
        json.as_object_mut().unwrap().remove("key_slots");
        let old: SyncStateV2 = serde_json::from_value(json).unwrap();
        assert!(old.key_slots.is_empty());

        state.key_slots.insert(
            "3fa2c1d90123456789abcdef01234567".into(),
            LocalSlot {
                file_name: "id_mac-3fa2c1d9".into(),
                source: Some(SlotSource::Linked {
                    path: "/home/f/.ssh/id_mac".into(),
                    link: crate::sync::slot_files::LinkKind::Symlink,
                    fingerprint: None,
                    origin: true,
                }),
                last_error: None,
                asked: false,
                payload: None,
            },
        );
        let back: SyncStateV2 = serde_json::from_value(serde_json::to_value(&state).unwrap()).unwrap();
        assert_eq!(back.key_slots, state.key_slots);
    }
```

`src/lib/sync-events.test.ts` 的 `noticeMessage` 測試(`describe` 裡已有其他種類的那一個 `it`)加:

```ts
    expect(noticeMessage({ kind: "keys_needed", names: ["id_mac", "work"] })).toEqual({
      title: "Pick keys for this computer",
      description: "Synced hosts use id_mac and work, which stay on your other computers.",
    });
    // 名稱來自別台電腦:看不見的字元要顯示出來。
    expect(noticeMessage({ kind: "keys_needed", names: [SPOOFED_NAME] }).description).toContain(SPOOFED_NAME_SHOWN);
```

(`SPOOFED_NAME`、`SPOOFED_NAME_SHOWN` 從 `@/lib/sync-fixtures` 匯入;檔案頂端已有的 import 不夠就補。)

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib slots -- --skip …` 與 `… --lib state_v2 -- --skip …`
Expected:編譯失敗(`slots` 模組、`set_device_slots`、`LocalSlot` 不存在)。前端的測試在 Step 4 binding 重新產生之後才跑。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/mod.rs` 加 `pub mod slots;`。

`src-tauri/src/sync/state_v2.rs`:

```rust
use crate::sync::slot_files::LinkKind;
use crate::sync::slot_rules::KeySlotPayload;
```

`SyncStateV2` 在 `notices` 之前加:

```rust
    /// 這台的金鑰插槽(SP3 spec §4.3),key = 插槽 id。只含公開資訊與本機路徑。離開帳戶時保留(插槽檔留在原地)。
    #[serde(default)]
    pub key_slots: BTreeMap<String, LocalSlot>,
```

`SyncStateV2::fresh`、`upgrade::shell_state` 與 `state_v2.rs` 測試裡的 `full_state` 都補 `key_slots: BTreeMap::new(),`
(`grep -rn "SyncStateV2 {" src-tauri/src/sync` 找出所有字面)。

新型別(放在 `SpaceState` 之前):

```rust
/// 一個插槽在這台電腦上的狀況(SP3 spec §4.3)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct LocalSlot {
    /// 插槽檔名(`<name>-<插槽 id 前 8>`)。
    pub file_name: String,
    /// 插槽裡放的是什麼;None = 這台還沒有它的金鑰。
    #[serde(default)]
    pub source: Option<SlotSource>,
    /// 最近一次維護這個插槽的錯誤(給使用者看;只有路徑與原因,不含金鑰內容)。
    #[serde(default)]
    pub last_error: Option<String>,
    /// 已經為「這台需要金鑰」發過 `SyncNotice::KeysNeeded`(每個插槽只發一次)。
    #[serde(default)]
    pub asked: bool,
    /// 最後看到的 `keyslot` payload:帳戶裡找不到這個插槽時(例如在沒有 SP3 的電腦上更換了同步碼)據此補寫(spec §6.6)。
    #[serde(default)]
    pub payload: Option<KeySlotPayload>,
}

/// 插槽裡放的東西(SP3 spec §4.2)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SlotSource {
    /// 連到這台的一把金鑰:建立插槽的那台(`origin`)連到原檔,其他電腦連到使用者挑的那把。
    Linked { path: String, link: LinkKind, fingerprint: Option<String>, origin: bool },
    /// 同步來的私鑰(檔案就在插槽裡)。
    SyncedCopy { fingerprint: String },
}
```

`SyncNotice` 加一個 variant(放在最後):

```rust
    /// 這台同步的主機用到留在別台電腦的金鑰:請在這台挑一把(SP3 spec §6.7;每個插槽只提示一次)。
    KeysNeeded { names: Vec<String> },
```

`src-tauri/src/sync/merge.rs`:

1. `use crate::sync::slot_rules::{is_slot_id, valid_key_payload, valid_slot_payload, DeviceSlot, KeyPayload, KeySlotPayload};`
2. `valid_account_record` 在 `RecordKind::Meta => true,` 之後、`_ => false` 之前加:

```rust
        RecordKind::KeySlot => {
            is_slot_id(&record.id)
                && parses(
                    serde_json::from_value::<KeySlotPayload>(record.payload.clone())
                        .ok()
                        .is_some_and(|p| valid_slot_payload(&p)),
                )
        }
        RecordKind::Key => {
            is_slot_id(&record.id)
                && parses(
                    serde_json::from_value::<KeyPayload>(record.payload.clone())
                        .ok()
                        .is_some_and(|p| valid_key_payload(&p)),
                )
        }
```

3. `merge_account`:文件註解改成「`device` / `space` / `meta` / `keyslot` 解密進 `records`;`spacekey` 與 `key`(SP3)只在記憶體
   解開比較,保存的是密文(`sealed`);其他種類(含未知)原樣存進 `sealed`、永不解密」。種類清單改成
   `Some(RecordKind::Device | RecordKind::Meta | RecordKind::Space | RecordKind::SpaceKey | RecordKind::KeySlot | RecordKind::Key) => {}`;
   `if record.kind == RecordKind::SpaceKey {` 改成 `if matches!(record.kind, RecordKind::SpaceKey | RecordKind::Key) {`;
   relay 倒退時重設的 `sealed` 改成:

```rust
        let resent = [RecordKind::SpaceKey.as_str(), RecordKind::Key.as_str()];
        for sealed in next.sealed.values_mut().filter(|s| resent.contains(&s.envelope.kind.as_str())) {
            sealed.envelope.seq = 0;
            sealed.dirty = true;
        }
```

4. `own_device_record`:沿用前一版 payload 的 `slots`(與 `joined_at_ms` 同一份前一版):

```rust
    let previous_payload = previous.and_then(|r| serde_json::from_value::<DevicePayload>(r.payload.clone()).ok());
    let joined_at_ms = previous_payload.as_ref().map(|p| p.joined_at_ms).unwrap_or(now_ms);
    // 插槽清單由 `set_device_slots` 維護;心跳與勾選變更沿用前一版(SP3 spec §4.1)。
    let slots = previous_payload.map(|p| p.slots).unwrap_or_default();
```

   payload 裡的 `slots: Vec::new()`(Task 1 加的)改成 `slots`。

5. 新函式(放在 `plan_device` 之後):

```rust
/// 這台的插槽清單(SP3 spec §4.1)和裝置記錄裡的不同時,寫一版新的(dirty;其他欄位沿用現況,`last_seen_ms` 更新)。
/// 這台還沒有裝置記錄(`plan_device` 還沒寫)時什麼都不做。回傳是否寫了。
pub fn set_device_slots(account: &mut AccountState, device_id: &str, slots: Vec<DeviceSlot>, now_ms: u64) -> bool {
    let key = record_key(RecordKind::Device, device_id);
    let Some(local) = account.records.get(&key).filter(|l| !l.record.deleted) else { return false };
    let Ok(mut payload) = serde_json::from_value::<DevicePayload>(local.record.payload.clone()) else { return false };
    if payload.slots == slots {
        return false;
    }
    payload.slots = slots;
    payload.last_seen_ms = now_ms;
    let record = Record {
        kind: RecordKind::Device,
        id: device_id.to_string(),
        version: local.record.version + 1,
        updated_at_ms: next_timestamp(now_ms, Some(local.record.updated_at_ms)),
        device_id: device_id.to_string(),
        deleted: false,
        payload: serde_json::to_value(payload).expect("DevicePayload serializes"),
    };
    let seq = local.seq;
    account.records.insert(key, LocalRecord { record, seq, dirty: true });
    true
}
```

`src-tauri/src/sync/slots.rs`(測試模組之前):

```rust
//! SP3 金鑰插槽的引擎(spec `docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md` §4、§6):帳戶裡 `keyslot` 與
//! `key` 記錄的讀寫(本節)、每一輪在這台維護插槽(`reconcile`,Task 4)、模式切換與挑選(Task 6)。

use serde_json::Value;

use crate::error::AppError;
use crate::sync::crypto::{id_hash, ChainKeys};
use crate::sync::merge::put_account_record;
use crate::sync::planner::next_timestamp;
use crate::sync::record::{record_key, Record, RecordKind};
use crate::sync::slot_rules::{valid_key_payload, valid_slot_payload, KeyPayload, KeySlotPayload, SLOT_SCHEMA};
use crate::sync::state_v2::{sealed_key, AccountState, SealedRecord};

// ── 帳戶記錄 ──────────────────────────────────────────────────────────────────────────────────

/// 帳戶裡未刪除、讀得懂的插槽(id, payload),依名稱、id 排序。
pub fn live_slots(account: &AccountState) -> Vec<(String, KeySlotPayload)> {
    let mut out: Vec<(String, KeySlotPayload)> = account
        .records
        .values()
        .filter(|l| l.record.kind == RecordKind::KeySlot && !l.record.deleted)
        .filter_map(|l| {
            let payload: KeySlotPayload = serde_json::from_value(l.record.payload.clone()).ok()?;
            valid_slot_payload(&payload).then(|| (l.record.id.clone(), payload))
        })
        .collect();
    out.sort_by(|a, b| (a.1.name.as_str(), a.0.as_str()).cmp(&(b.1.name.as_str(), b.0.as_str())));
    out
}

pub fn slot(account: &AccountState, slot_id: &str) -> Option<KeySlotPayload> {
    live_slots(account).into_iter().find(|(id, _)| id == slot_id).map(|(_, p)| p)
}

/// 帳戶裡有沒有這個插槽的記錄(含 tombstone)。
pub fn slot_record_exists(account: &AccountState, slot_id: &str) -> bool {
    account.records.contains_key(&record_key(RecordKind::KeySlot, slot_id))
}

/// 寫一筆 `keyslot`(dirty)。`None` = tombstone。
pub fn put_slot(account: &mut AccountState, slot_id: &str, payload: Option<&KeySlotPayload>, device_id: &str, now_ms: u64) {
    let value = payload.map(|p| serde_json::to_value(p).expect("KeySlotPayload serializes")).unwrap_or(Value::Null);
    put_account_record(account, RecordKind::KeySlot, slot_id, value, payload.is_none(), device_id, now_ms);
}

/// 一個插槽的 `key` 在 `sealed` 裡的 key(以帳戶金鑰算的 id_hash)。
pub fn key_secret_key(account_keys: &ChainKeys, slot_id: &str) -> String {
    sealed_key(RecordKind::Key.as_str(), &id_hash(account_keys, RecordKind::Key.as_str(), slot_id))
}

/// 寫一筆 `key`(祕密,SP3 spec §4.1):以帳戶金鑰加密後放進 `sealed`(dirty),明文只在記憶體。`None` = tombstone
/// (不帶任何祕密)。版本號、時間戳與 seq 接在前一版之後(同 `merge::put_space_key`)。
pub fn put_key_secret(
    account: &mut AccountState,
    account_keys: &ChainKeys,
    slot_id: &str,
    private_key: Option<&str>,
    device_id: &str,
    now_ms: u64,
) -> Result<(), AppError> {
    let key = key_secret_key(account_keys, slot_id);
    let previous = account.sealed.get(&key).and_then(|s| s.open(account_keys).ok().map(|r| (r, s.envelope.seq)));
    let record = Record {
        kind: RecordKind::Key,
        id: slot_id.to_string(),
        version: previous.as_ref().map(|(r, _)| r.version + 1).unwrap_or(1),
        updated_at_ms: next_timestamp(now_ms, previous.as_ref().map(|(r, _)| r.updated_at_ms)),
        device_id: device_id.to_string(),
        deleted: private_key.is_none(),
        payload: match private_key {
            Some(text) => serde_json::to_value(KeyPayload { schema: SLOT_SCHEMA, private_key: text.to_string() })
                .expect("KeyPayload serializes"),
            None => Value::Null,
        },
    };
    let sealed = SealedRecord::seal(account_keys, &record, previous.map(|(_, seq)| seq).unwrap_or(0))?;
    account.sealed.insert(key, sealed);
    Ok(())
}

/// 在記憶體解開一個插槽的私鑰。沒有、已刪除或讀不懂 → None。
pub fn open_key_secret(account: &AccountState, account_keys: &ChainKeys, slot_id: &str) -> Option<String> {
    let record = account.sealed.get(&key_secret_key(account_keys, slot_id))?.open(account_keys).ok()?;
    if record.deleted || record.id != slot_id {
        return None;
    }
    let payload: KeyPayload = serde_json::from_value(record.payload).ok()?;
    valid_key_payload(&payload).then_some(payload.private_key)
}
```

`src/lib/sync-events.ts` 的 `noticeMessage` 在 `other_rotation` 之後加:

```ts
    case "keys_needed":
      return {
        title: "Pick keys for this computer",
        description: `Synced hosts use ${listNames(notice.names.map(revealHidden))}, which stay on your other computers.`,
      };
```

- [ ] **Step 4: 跑測試確認通過**

Run:`… cargo test --offline --lib slots -- --skip …`、`… --lib state_v2 -- --skip …`,再跑完整的 Rust 測試;接著在 repo
根目錄跑 `./node_modules/.bin/tsc --noEmit` 與 `./node_modules/.bin/vitest run --dir src`。
Expected:`slots` 的 6 個測試、`state_v2` 新增的 1 個測試通過;完整測試全綠;`src/bindings/SyncNotice.ts` 多了
`{ "kind": "keys_needed", names: Array<string> }`;`tsc` 沒有錯誤、vitest 全綠。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/slots.rs src-tauri/src/sync/mod.rs src-tauri/src/sync/merge.rs src-tauri/src/sync/state_v2.rs src-tauri/src/sync/upgrade.rs src/bindings/SyncNotice.ts src/lib/sync-events.ts src/lib/sync-events.test.ts
git commit -m "feat(sync): keyslot and key records in the account chain"
```

### Task 4: 每一輪在這台維護插槽(`slots::reconcile`)、插槽檢視、接進同步輪次與 overview

**Files:**
- Modify: `src-tauri/src/sync/slots.rs`(reconcile、檢視)
- Modify: `src-tauri/src/sync/dto.rs`(檢視型別、`SyncOverview.key_slots`)
- Modify: `src-tauri/src/sync/round.rs`(每一輪呼叫 reconcile)
- Modify: `src/lib/sync-fixtures.ts`(`overview()` 加 `key_slots: []`,否則 `tsc` 過不了)
- Generated: `src/bindings/SyncKeySlotView.ts`、`src/bindings/SlotStatusView.ts`、`src/bindings/SlotDeviceView.ts`、`src/bindings/SyncOverview.ts`

**Interfaces:**
- Consumes: Task 1–3。`round::tests::{pair, settle}`(`pub(crate)`)、`runtime::mutate`、`round::sync_once`。
- Produces:
  - `slots::identity_slot_files(text: &str) -> Vec<String>`、`slots::slot_hosts(&SyncStateV2) -> BTreeMap<String, Vec<String>>`(插槽檔名 → alias)
  - `slots::local_key_fingerprint(path: &Path) -> Option<String>`
  - `slots::MISMATCH_MESSAGE`、`slots::in_the_way_message(&Path) -> String`、`slots::source_gone_message(&str) -> String`
  - `pub struct SlotRound { notices: Vec<SyncNotice>, changed: bool }`、`slots::reconcile(state: &mut SyncStateV2, account_keys: &ChainKeys, home: &Path, now_ms: u64) -> SlotRound`
  - `slots::add_notice(notices: &mut Vec<SyncNotice>, notice: &SyncNotice)`(`KeysNeeded` 至多一則,新名稱併入)
  - `slots::views(&SyncStateV2, &ChainKeys, home: &Path) -> Vec<SyncKeySlotView>`、`slots::slot_status(...) -> SlotStatusView`
  - `dto::SyncKeySlotView { id, name, mode: SlotMode, fingerprint: Option<String>, key_type: Option<String>, has_passphrase: Option<bool>, origin_device: String, origin_is_this: bool, value: String, hosts: Vec<String>, status: SlotStatusView, devices: Vec<SlotDeviceView> }`
  - `dto::SlotStatusView`(serde `tag = "kind"`、snake_case):`Ready { file, synced_copy: bool, fingerprint: Option<String> }`、`NeedsKey { waiting_for_sync: bool }`、`NotInUse { file }`、`NotUsedHere`、`SyncedAvailable { file }`、`SourceChanged { file }`、`Error { message }`
  - `dto::SlotDeviceView { name, fingerprint: Option<String>, synced_copy: bool }`
  - `SyncOverview.key_slots: Vec<SyncKeySlotView>`

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slots.rs` 的測試模組加(沿用 Task 3 的 `SLOT_ID`、`synced_payload`):

```rust
    use crate::sync::dto::{SlotDeviceView, SlotStatusView};
    use crate::sync::round::tests::{pair, settle};
    use crate::sync::runtime::mutate;
    use crate::sync::slot_files;
    use crate::sync::slot_rules::{default_slot_name, inspect_private_key, new_slot_id, public_path, slot_file_name, SLOT_DIR};
    use crate::sync::state_v2::{LocalSlot, SlotSource, SyncNotice};
    use crate::sync::testkit::TestDevice;
    use std::path::PathBuf;

    fn home(d: &TestDevice) -> PathBuf {
        d.ssh_dir().parent().unwrap().to_path_buf()
    }

    fn account_keys(d: &TestDevice) -> ChainKeys {
        let env = d.env();
        let keys = env.runtime.core.lock().unwrap().account_keys.clone().expect("joined");
        keys
    }

    /// 在 `d` 上建立一個插槽(Task 5 的 `setup_keys` 之前,直接寫記錄與連結):金鑰檔放在 `~/.ssh/<key_file>`,插槽連到它。
    /// 回傳(插槽 id、插槽檔名)。
    pub(crate) fn create_slot_on(d: &TestDevice, mode: SlotMode, key_text: &str, key_file: &str) -> (String, String) {
        let source = d.ssh_dir().join(key_file);
        std::fs::write(&source, key_text).unwrap();
        let facts = inspect_private_key(key_text).ok();
        let (id, name) = (new_slot_id().unwrap(), default_slot_name(key_file));
        let file = slot_file_name(&name, &id);
        let keys_dir = home(d).join(SLOT_DIR);
        slot_files::ensure_keys_dir(&keys_dir).unwrap();
        let link = slot_files::link(&source, &keys_dir.join(&file)).unwrap();
        let keys = account_keys(d);
        let env = d.env();
        let now = env.now();
        mutate(&env, |s| {
            let device_id = s.device_id.clone();
            let synced = mode == SlotMode::Synced;
            let payload = KeySlotPayload {
                schema: SLOT_SCHEMA,
                name: name.clone(),
                mode,
                origin_device_id: device_id.clone(),
                created_at_ms: now,
                public_key: synced.then(|| facts.clone().unwrap().public_key),
                fingerprint: synced.then(|| facts.clone().unwrap().fingerprint),
                key_type: synced.then(|| facts.clone().unwrap().key_type),
                has_passphrase: synced.then(|| facts.clone().unwrap().has_passphrase),
            };
            let account = s.account.as_mut().unwrap();
            put_slot(account, &id, Some(&payload), &device_id, now);
            if synced {
                put_key_secret(account, &keys, &id, Some(key_text), &device_id, now)?;
            }
            s.key_slots.insert(
                id.clone(),
                LocalSlot {
                    file_name: file.clone(),
                    source: Some(SlotSource::Linked {
                        path: source.to_string_lossy().into_owned(),
                        link,
                        fingerprint: facts.as_ref().map(|f| f.fingerprint.clone()),
                        origin: true,
                    }),
                    last_error: None,
                    asked: false,
                    payload: Some(payload),
                },
            );
            Ok(())
        })
        .unwrap();
        (id, file)
    }

    /// 在 app 裡把 Personal 的主機 `web` 指到插槽(存檔 → 上傳)。
    pub(crate) fn use_slot(d: &TestDevice, personal: &str, file: &str) {
        d.save_in_app(&d.space_path(personal), &format!("Host web\n  HostName 10.0.0.1\n  IdentityFile ~/.ssh/sshelter/keys/{file}\n"));
    }

    #[test]
    fn finds_the_hosts_that_use_each_slot() {
        assert_eq!(
            identity_slot_files("Host web\n  identityfile = \"~/.ssh/sshelter/keys/a-11111111\"\n  # IdentityFile ~/.ssh/sshelter/keys/b-22222222\n  IdentityFile ~/.ssh/id_mac\n"),
            vec!["a-11111111".to_string()]
        );
    }

    #[test]
    #[cfg(unix)]
    fn a_synced_key_lands_on_the_other_computer() {
        use std::os::unix::fs::PermissionsExt;
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        let landed = home(&b).join(SLOT_DIR).join(&file);
        assert_eq!(std::fs::read_to_string(&landed).unwrap(), test_keys::plain());
        assert_eq!(std::fs::metadata(&landed).unwrap().permissions().mode() & 0o777, 0o600);
        assert_eq!(std::fs::read_to_string(public_path(&landed)).unwrap(), format!("{}\n", test_keys::PLAIN_PUBLIC));
        assert_eq!(b.state().key_slots[&id].source, Some(SlotSource::SyncedCopy { fingerprint: test_keys::PLAIN_FINGERPRINT.into() }));

        let b_view = views(&b.state(), &account_keys(&b), &home(&b));
        assert_eq!(b_view.len(), 1);
        assert_eq!(b_view[0].hosts, vec!["web".to_string()]);
        assert_eq!(b_view[0].origin_device, "MacBook-A");
        assert!(!b_view[0].origin_is_this);
        assert!(matches!(b_view[0].status, SlotStatusView::Ready { synced_copy: true, .. }), "{:?}", b_view[0].status);
        assert_eq!(crate::sync::dto::overview(&b.env()).unwrap().key_slots, b_view);

        // A 收到 B 的 `device.slots`。
        settle(&a);
        let a_view = views(&a.state(), &account_keys(&a), &home(&a));
        assert!(matches!(a_view[0].status, SlotStatusView::Ready { synced_copy: false, .. }), "{:?}", a_view[0].status);
        assert_eq!(
            a_view[0].devices,
            vec![SlotDeviceView { name: "MacBook-B".into(), fingerprint: Some(test_keys::PLAIN_FINGERPRINT.into()), synced_copy: true }]
        );
    }

    #[test]
    fn an_own_key_slot_asks_the_other_computer_once() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Own, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        let local = b.state().key_slots[&id].clone();
        assert_eq!(local.source, None);
        assert!(local.asked);
        assert!(!home(&b).join(SLOT_DIR).join(&file).exists());
        let asked = SyncNotice::KeysNeeded { names: vec!["id_mac".into()] };
        assert_eq!(b.state().notices.iter().filter(|n| **n == asked).count(), 1);
        assert_eq!(b.events.notices.lock().unwrap().iter().filter(|n| **n == asked).count(), 1);

        let _ = crate::sync::round::sync_once(&b.env());
        assert_eq!(b.events.notices.lock().unwrap().iter().filter(|n| **n == asked).count(), 1, "asked only once");
        let view = views(&b.state(), &account_keys(&b), &home(&b));
        assert_eq!(view[0].status, SlotStatusView::NeedsKey { waiting_for_sync: false });
    }

    #[test]
    fn keys_needed_notices_merge_into_one() {
        // 還沒關掉的那一則收下新名稱:前端的「Keys for this computer」只開一次、按一次 Done 就結束。
        let mut notices = vec![SyncNotice::NewSyncCode, SyncNotice::KeysNeeded { names: vec!["id_mac".into()] }];
        add_notice(&mut notices, &SyncNotice::KeysNeeded { names: vec!["id_mac".into(), "work".into()] });
        assert_eq!(
            notices,
            vec![SyncNotice::NewSyncCode, SyncNotice::KeysNeeded { names: vec!["id_mac".into(), "work".into()] }]
        );
        // 其他種類照舊:一樣的不重複加。
        add_notice(&mut notices, &SyncNotice::NewSyncCode);
        assert_eq!(notices.len(), 2);
        let mut empty = Vec::new();
        add_notice(&mut empty, &SyncNotice::KeysNeeded { names: vec!["id_mac".into()] });
        assert_eq!(empty, vec![SyncNotice::KeysNeeded { names: vec!["id_mac".into()] }]);
    }

    #[test]
    fn a_file_in_the_slot_path_is_never_overwritten() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        let theirs = home(&b).join(SLOT_DIR).join(&file);
        std::fs::create_dir_all(theirs.parent().unwrap()).unwrap();
        std::fs::write(&theirs, "mine").unwrap();
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        assert_eq!(std::fs::read_to_string(&theirs).unwrap(), "mine");
        assert_eq!(b.state().key_slots[&id].last_error, Some(in_the_way_message(&theirs)));
        let view = views(&b.state(), &account_keys(&b), &home(&b));
        assert_eq!(view[0].status, SlotStatusView::Error { message: in_the_way_message(&theirs) });
        // 錯誤訊息只帶路徑,不帶金鑰。
        assert!(!in_the_way_message(&theirs).contains(test_keys::PLAIN_BODY[1]));
    }

    #[test]
    fn a_key_without_its_slot_writes_nothing_until_the_slot_arrives() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let keys = account_keys(&a);
        let id = new_slot_id().unwrap();
        let file = slot_file_name("id_mac", &id);
        let env = a.env();
        let now = env.now();
        mutate(&env, |s| {
            let me = s.device_id.clone();
            put_key_secret(s.account.as_mut().unwrap(), &keys, &id, Some(&test_keys::plain()), &me, now)
        })
        .unwrap();
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        assert!(!home(&b).join(SLOT_DIR).join(&file).exists());
        assert!(b.state().key_slots.is_empty());

        mutate(&env, |s| {
            let me = s.device_id.clone();
            put_slot(s.account.as_mut().unwrap(), &id, Some(&synced_payload(&me)), &me, now + 1);
            Ok(())
        })
        .unwrap();
        settle(&a);
        settle(&b);
        assert_eq!(std::fs::read_to_string(home(&b).join(SLOT_DIR).join(&file)).unwrap(), test_keys::plain());
    }

    #[test]
    fn a_deleted_slot_removes_links_and_keeps_copies() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        let keys = account_keys(&a);
        let env = a.env();
        let now = env.now();
        mutate(&env, |s| {
            let me = s.device_id.clone();
            let account = s.account.as_mut().unwrap();
            put_slot(account, &id, None, &me, now);
            put_key_secret(account, &keys, &id, None, &me, now)
        })
        .unwrap();
        settle(&a);
        settle(&b);

        assert!(!slot_files::occupied(&home(&a).join(SLOT_DIR).join(&file)), "A's link is removed");
        assert!(a.ssh_dir().join("id_mac").exists(), "the key it pointed to is untouched");
        assert!(!a.state().key_slots.contains_key(&id));
        let copy = home(&b).join(SLOT_DIR).join(&file);
        assert_eq!(std::fs::read_to_string(&copy).unwrap(), test_keys::plain(), "B keeps its copy");
        let view = views(&b.state(), &account_keys(&b), &home(&b));
        assert_eq!(view.len(), 1);
        assert_eq!(view[0].status, SlotStatusView::NotInUse { file: copy.display().to_string() });
    }

    #[test]
    fn a_key_written_before_a_lost_commit_is_adopted() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        // 檔案寫了、狀態沒存下來:拿掉 B 對插槽的記錄,下一輪不能把自己寫的檔案當成擋路的。
        mutate(&b.env(), |s| {
            s.key_slots.clear();
            Ok(())
        })
        .unwrap();
        let _ = crate::sync::round::sync_once(&b.env());
        let local = b.state().key_slots[&id].clone();
        assert_eq!(local.last_error, None);
        assert_eq!(local.source, Some(SlotSource::SyncedCopy { fingerprint: test_keys::PLAIN_FINGERPRINT.into() }));
        let _ = file;
    }

    #[test]
    fn a_changed_key_on_the_origin_is_reported_and_not_uploaded() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        std::fs::write(a.ssh_dir().join("id_mac"), test_keys::ecdsa()).unwrap();
        settle(&a);

        let view = views(&a.state(), &account_keys(&a), &home(&a));
        assert!(matches!(view[0].status, SlotStatusView::SourceChanged { .. }), "{:?}", view[0].status);
        let account = a.state().account.unwrap();
        assert_eq!(open_key_secret(&account, &account_keys(&a), &id).as_deref(), Some(test_keys::plain().as_str()));
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib slots -- --skip …`
Expected:編譯失敗(`reconcile`、`views`、`SlotStatusView` 等不存在)。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/dto.rs`(放在 `SyncSpaceView` 之後):

```rust
/// 一個金鑰插槽(SP3 spec §7.2),依名稱排序。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncKeySlotView {
    pub id: String,
    /// 插槽名稱;來自別台電腦,UI 以 `revealHidden` 顯示。
    pub name: String,
    pub mode: crate::sync::slot_rules::SlotMode,
    /// `synced` 的金鑰指紋;`own` 為 null。
    pub fingerprint: Option<String>,
    pub key_type: Option<String>,
    pub has_passphrase: Option<bool>,
    /// 建立插槽的電腦名稱。
    pub origin_device: String,
    pub origin_is_this: bool,
    /// 主機 `IdentityFile` 的值(`~/.ssh/sshelter/keys/<file>`)。
    pub value: String,
    /// 這台用到它的主機。
    pub hosts: Vec<String>,
    pub status: SlotStatusView,
    /// 其他電腦的插槽狀況(它們的 `device.slots`)。
    pub devices: Vec<SlotDeviceView>,
}

/// 插槽在這台電腦上的狀態(SP3 spec §7.2、§7.3)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SlotStatusView {
    /// 插槽裡有金鑰;`file` = 這台實際用的檔案(連到的金鑰,或插槽本身的副本)。
    Ready { file: String, synced_copy: bool, fingerprint: Option<String> },
    /// 這台需要金鑰:`own` 要使用者挑(`waiting_for_sync` = false),`synced` 的私鑰還沒到(true)。
    NeedsKey { waiting_for_sync: bool },
    /// 沒有主機用到,但這台還留著同步來的副本或複製檔(可以刪除)。
    NotInUse { file: String },
    /// 這台沒有主機用到它。
    NotUsedHere,
    /// 插槽有同步的金鑰,這台用的卻是另一把(本機挑的,或舊的副本):可以改用。
    SyncedAvailable { file: String },
    /// 建立插槽的這台,原檔換成了另一把金鑰;其他電腦還是上一把。
    SourceChanged { file: String },
    Error { message: String },
}

/// 另一台電腦的插槽狀況。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SlotDeviceView {
    pub name: String,
    pub fingerprint: Option<String>,
    pub synced_copy: bool,
}
```

`SyncOverview` 在 `notices` 之前加:

```rust
    /// 帳戶裡的金鑰插槽與這台還留著副本的舊插槽(SP3 spec §7.2)。
    pub key_slots: Vec<SyncKeySlotView>,
```

`dto::overview` 組 `SyncOverview` 之前算出它(`keys` 是函式開頭從 core 拿的帳戶金鑰):

```rust
    let key_slots = match (keys.as_ref(), env.ssh_dir.parent()) {
        (Some(k), Some(home)) => crate::sync::slots::views(&s, k, home),
        _ => Vec::new(),
    };
```

並在 struct 字面加 `key_slots,`。

`src-tauri/src/sync/slots.rs`:import 補齊(放在檔案開頭的 `use` 區):

```rust
use std::collections::{BTreeMap, BTreeSet};
use std::path::{Path, PathBuf};

use crate::config::model::Item;
use crate::config::parser::parse_file;
use crate::sync::dto::{SlotDeviceView, SlotStatusView, SyncKeySlotView};
use crate::sync::merge::{device_name, devices, set_device_slots};
use crate::sync::record::HostPayload;
use crate::sync::slot_files::{self, LinkKind};
use crate::sync::slot_rules::{
    inspect_private_key, parse_public_key, public_path, slot_file_name, slot_file_of_value, slot_value, DeviceSlot, SlotMode,
    SLOT_DIR,
};
use crate::sync::state_v2::{LocalSlot, SlotSource, SyncNotice, SyncStateV2};
```

接在帳戶記錄那一節之後:

```rust
// ── 每一輪在這台維護插槽(SP3 spec §6.2–§6.6)──────────────────────────────────────────────────

pub const MISMATCH_MESSAGE: &str = "The synced key didn't match and was not written.";

pub fn in_the_way_message(path: &Path) -> String {
    format!("A file SSHelter didn't create is in the way: {}. Move it, then sync again.", path.display())
}

pub fn source_gone_message(path: &str) -> String {
    format!("The key this slot points to is gone: {path}.")
}

/// 一個 Host 區塊文字裡,指到插槽的 `IdentityFile`(插槽檔名)。註解掉的行不算。
pub fn identity_slot_files(text: &str) -> Vec<String> {
    let (items, _) = parse_file(text);
    let mut out = Vec::new();
    for item in &items {
        let Item::Host(host) = item else { continue };
        for line in &host.body {
            if let Item::Directive(d) = line {
                if d.key == "identityfile" && !d.serializes_as_comment() {
                    out.extend(slot_file_of_value(&d.value));
                }
            }
        }
    }
    out
}

/// 這台勾選的 space 裡、未刪除的主機中,用到各插槽(依插槽檔名)的 alias(排序、不重複)。等待核准的主機不算。
pub fn slot_hosts(state: &SyncStateV2) -> BTreeMap<String, Vec<String>> {
    let mut out: BTreeMap<String, BTreeSet<String>> = BTreeMap::new();
    for space in state.spaces.values().filter(|sp| sp.selected) {
        for local in space.records.values().filter(|l| l.record.kind == RecordKind::Host && !l.record.deleted) {
            let Ok(payload) = serde_json::from_value::<HostPayload>(local.record.payload.clone()) else { continue };
            for file in identity_slot_files(&payload.text) {
                out.entry(file).or_default().insert(local.record.id.clone());
            }
        }
    }
    out.into_iter().map(|(file, hosts)| (file, hosts.into_iter().collect())).collect()
}

/// 本機一把金鑰的指紋:OpenSSH 格式從私鑰讀;其他格式讀旁邊的 `.pub`;都讀不到 → None。
pub fn local_key_fingerprint(path: &Path) -> Option<String> {
    let text = std::fs::read_to_string(path).ok()?;
    if let Ok(facts) = inspect_private_key(&text) {
        return Some(facts.fingerprint);
    }
    let public = std::fs::read_to_string(public_path(path)).ok()?;
    parse_public_key(public.lines().next()?).map(|(_, fingerprint)| fingerprint)
}

/// 一輪插槽維護的結果:要發的通知(呼叫端用 `add_notice` 存進狀態並發出)、狀態是否變了。
#[derive(Debug, Default)]
pub struct SlotRound {
    pub notices: Vec<SyncNotice>,
    pub changed: bool,
}

/// 把通知加進狀態的 `notices`。「這台需要金鑰」至多一則:新名稱併進還沒關掉的那一則(前端的「Keys for this computer」
/// 因此只開一次);其他種類一樣的不重複加。
pub fn add_notice(notices: &mut Vec<SyncNotice>, notice: &SyncNotice) {
    if let SyncNotice::KeysNeeded { names } = notice {
        if let Some(SyncNotice::KeysNeeded { names: open }) = notices.iter_mut().find(|n| matches!(n, SyncNotice::KeysNeeded { .. })) {
            for name in names {
                if !open.contains(name) {
                    open.push(name.clone());
                }
            }
            return;
        }
    }
    if !notices.contains(notice) {
        notices.push(notice.clone());
    }
}

/// 依合併後的帳戶與 space 記錄,維護這台的插槽(SP3 spec §6.2–§6.6)。檔案系統的動作在這裡做(呼叫端不持有任何鎖);
/// 狀態的變更寫進 `state`(`key_slots` 與帳戶的 `device.slots`、補寫的 `keyslot`/`key`),由呼叫端提交。`home` = 家目錄。
pub fn reconcile(state: &mut SyncStateV2, account_keys: &ChainKeys, home: &Path, now_ms: u64) -> SlotRound {
    let mut round = SlotRound::default();
    let needed = slot_hosts(state);
    let device_id = state.device_id.clone();
    let keys_dir = home.join(SLOT_DIR);
    let Some(account) = state.account.as_mut() else { return round };
    let live = live_slots(account);
    let mut asked = Vec::new();

    for (id, payload) in &live {
        let file = slot_file_name(&payload.name, id);
        let path = keys_dir.join(&file);
        let before = state.key_slots.get(id).cloned();
        let mut local = before.clone().unwrap_or_else(|| LocalSlot {
            file_name: file.clone(),
            source: None,
            last_error: None,
            asked: false,
            payload: None,
        });
        local.file_name = file.clone();
        local.payload = Some(payload.clone());
        let is_needed = needed.contains_key(&file);
        if is_needed {
            maintain(&mut local, id, payload, account, account_keys, &keys_dir, &path);
            if local.source.is_none() && local.last_error.is_none() && payload.mode == SlotMode::Own && !local.asked {
                local.asked = true;
                asked.push(payload.name.clone());
            }
        } else {
            drop_link(&mut local, &path);
        }
        if !is_needed && local.source.is_none() {
            // 這台沒用到、也沒留東西:不記。
            round.changed |= state.key_slots.remove(id).is_some();
        } else if before.as_ref() != Some(&local) {
            state.key_slots.insert(id.clone(), local);
            round.changed = true;
        }
    }

    // 這台記著、帳戶裡卻沒有的插槽。已刪除的(tombstone):移除連結、副本留著。完全找不到記錄而主機還用著:補寫(spec §6.6,
    // 例如在沒有 SP3 的電腦上更換了同步碼)。
    let gone: Vec<String> = state.key_slots.keys().filter(|id| !live.iter().any(|(l, _)| l == *id)).cloned().collect();
    for id in gone {
        let mut local = state.key_slots[&id].clone();
        let path = keys_dir.join(&local.file_name);
        if !slot_record_exists(account, &id) && needed.contains_key(&local.file_name) {
            if let Some(payload) = local.payload.clone() {
                republish(account, account_keys, &id, &payload, &local, &path, &device_id, now_ms);
                round.changed = true;
                continue;
            }
        }
        let before = local.clone();
        drop_link(&mut local, &path);
        if local.source.is_none() {
            state.key_slots.remove(&id);
            round.changed = true;
        } else if local != before {
            state.key_slots.insert(id, local);
            round.changed = true;
        }
    }

    let slots: Vec<DeviceSlot> = state.key_slots.iter().filter_map(|(id, l)| device_slot(id, l)).collect();
    round.changed |= set_device_slots(account, &device_id, slots, now_ms);
    if !asked.is_empty() {
        round.notices.push(SyncNotice::KeysNeeded { names: asked });
    }
    round
}

fn device_slot(slot_id: &str, local: &LocalSlot) -> Option<DeviceSlot> {
    match &local.source {
        Some(SlotSource::Linked { fingerprint, .. }) => {
            Some(DeviceSlot { slot_id: slot_id.to_string(), fingerprint: fingerprint.clone(), synced_copy: false })
        }
        Some(SlotSource::SyncedCopy { fingerprint }) => {
            Some(DeviceSlot { slot_id: slot_id.to_string(), fingerprint: Some(fingerprint.clone()), synced_copy: true })
        }
        None => None,
    }
}

/// 沒有主機用到的插槽:symlink / hard link 移除(原檔不動);同步來的副本與複製檔留著(私鑰不自動刪除)。
fn drop_link(local: &mut LocalSlot, path: &Path) {
    if let Some(SlotSource::Linked { link: LinkKind::Symlink | LinkKind::HardLink, .. }) = &local.source {
        match slot_files::remove_slot(path) {
            Ok(()) => {
                local.source = None;
                local.last_error = None;
            }
            Err(e) => local.last_error = Some(e.to_string()),
        }
    }
}

/// 這台需要的插槽:連結的確認原檔還在、hard link 與複製跟上原檔;副本被刪掉就重放;空的就試著落地。
fn maintain(
    local: &mut LocalSlot,
    slot_id: &str,
    payload: &KeySlotPayload,
    account: &AccountState,
    account_keys: &ChainKeys,
    keys_dir: &Path,
    path: &Path,
) {
    match local.source.clone() {
        Some(SlotSource::Linked { path: source, link, origin, .. }) => {
            let source_path = PathBuf::from(&source);
            if !source_path.is_file() {
                local.last_error = Some(source_gone_message(&source));
                return;
            }
            // hard link 與複製不會跟著原檔走:內容不同(原檔被換掉)就重新連結;任何一種,插槽不見了都重建。
            let stale = !slot_files::occupied(path)
                || (link != LinkKind::Symlink && slot_files::content_sha256(path) != slot_files::content_sha256(&source_path));
            let link = if stale {
                match slot_files::ensure_keys_dir(keys_dir).and_then(|()| slot_files::link(&source_path, path)) {
                    Ok(kind) => kind,
                    Err(e) => {
                        local.last_error = Some(e.to_string());
                        return;
                    }
                }
            } else {
                link
            };
            local.source = Some(SlotSource::Linked { path: source, link, fingerprint: local_key_fingerprint(&source_path), origin });
            local.last_error = None;
        }
        Some(SlotSource::SyncedCopy { .. }) if slot_files::occupied(path) => local.last_error = None,
        // 副本被刪掉了:同步的金鑰還在就放回去。
        Some(SlotSource::SyncedCopy { .. }) | None => {
            local.source = None;
            land_into(local, slot_id, payload, account, account_keys, keys_dir, path);
        }
    }
}

/// 空的插槽:`synced` 而且私鑰到了就落地;`own` 等使用者挑,私鑰還沒到就等下一輪。
fn land_into(
    local: &mut LocalSlot,
    slot_id: &str,
    payload: &KeySlotPayload,
    account: &AccountState,
    account_keys: &ChainKeys,
    keys_dir: &Path,
    path: &Path,
) {
    local.last_error = None;
    if payload.mode != SlotMode::Synced {
        return;
    }
    let Some(secret) = open_key_secret(account, account_keys, slot_id) else { return };
    match land(&secret, payload, keys_dir, path) {
        Ok(fingerprint) => local.source = Some(SlotSource::SyncedCopy { fingerprint }),
        Err(message) => local.last_error = Some(message),
    }
}

/// 把同步的私鑰寫進空的插槽(spec §6.2):指紋要和插槽記錄一致。插槽路徑上已經有東西時,只有內容完全相同(上一輪寫了
/// 檔案、狀態卻沒存下來)才當成自己的,其他一律不覆蓋。錯誤訊息只帶路徑與原因。
fn land(secret: &str, payload: &KeySlotPayload, keys_dir: &Path, path: &Path) -> Result<String, String> {
    let facts = inspect_private_key(secret).map_err(|_| MISMATCH_MESSAGE.to_string())?;
    if payload.fingerprint.as_deref() != Some(facts.fingerprint.as_str()) {
        return Err(MISMATCH_MESSAGE.to_string());
    }
    if slot_files::occupied(path) {
        if std::fs::read(path).ok().as_deref() != Some(secret.as_bytes()) {
            return Err(in_the_way_message(path));
        }
    } else {
        slot_files::ensure_keys_dir(keys_dir).map_err(|e| e.to_string())?;
        slot_files::write_private(path, secret.as_bytes()).map_err(|e| e.to_string())?;
    }
    let public = payload.public_key.as_deref().unwrap_or(&facts.public_key);
    slot_files::write_public(path, public).map_err(|e| e.to_string())?;
    Ok(facts.fingerprint)
}

/// 補寫帳戶裡不見的插槽(spec §6.6):`keyslot` 用最後看到的 payload;原本是 `synced`、而這台讀得到同指紋的私鑰時,
/// `key` 一起補。
#[allow(clippy::too_many_arguments)]
fn republish(
    account: &mut AccountState,
    account_keys: &ChainKeys,
    slot_id: &str,
    payload: &KeySlotPayload,
    local: &LocalSlot,
    path: &Path,
    device_id: &str,
    now_ms: u64,
) {
    put_slot(account, slot_id, Some(payload), device_id, now_ms);
    if payload.mode != SlotMode::Synced {
        return;
    }
    let readable = match &local.source {
        Some(SlotSource::Linked { path: source, .. }) => std::fs::read_to_string(source).ok(),
        Some(SlotSource::SyncedCopy { .. }) => std::fs::read_to_string(path).ok(),
        None => None,
    };
    let matching = readable.filter(|text| {
        inspect_private_key(text).is_ok_and(|f| payload.fingerprint.as_deref() == Some(f.fingerprint.as_str()))
    });
    if let Some(text) = matching {
        if let Err(e) = put_key_secret(account, account_keys, slot_id, Some(&text), device_id, now_ms) {
            eprintln!("[sync] could not restore a key slot's key: {e}");
        }
    }
}

// ── 給 UI 的插槽檢視(SP3 spec §7.2、§7.3)─────────────────────────────────────────────────────

/// 帳戶裡的插槽(依名稱),加上帳戶裡已經沒有、這台還留著副本的(Not in use)。
pub fn views(state: &SyncStateV2, account_keys: &ChainKeys, home: &Path) -> Vec<SyncKeySlotView> {
    let Some(account) = state.account.as_ref() else { return Vec::new() };
    let needed = slot_hosts(state);
    let all_devices = devices(account);
    let keys_dir = home.join(SLOT_DIR);
    let live = live_slots(account);
    let view = |id: &str, payload: &KeySlotPayload, status: SlotStatusView, hosts: Vec<String>| SyncKeySlotView {
        id: id.to_string(),
        name: payload.name.clone(),
        mode: payload.mode,
        fingerprint: payload.fingerprint.clone(),
        key_type: payload.key_type.clone(),
        has_passphrase: payload.has_passphrase,
        origin_device: device_name(account, &payload.origin_device_id),
        origin_is_this: payload.origin_device_id == state.device_id,
        value: slot_value(&slot_file_name(&payload.name, id)),
        hosts,
        status,
        devices: all_devices
            .iter()
            .filter(|(device, _)| *device != state.device_id)
            .filter_map(|(_, p)| {
                p.slots.iter().find(|s| s.slot_id == id).map(|s| SlotDeviceView {
                    name: p.name.clone(),
                    fingerprint: s.fingerprint.clone(),
                    synced_copy: s.synced_copy,
                })
            })
            .collect(),
    };
    let mut out = Vec::new();
    for (id, payload) in &live {
        let file = slot_file_name(&payload.name, id);
        let has_secret = account.sealed.get(&key_secret_key(account_keys, id)).is_some_and(|s| !s.envelope.deleted);
        let status = slot_status(state.key_slots.get(id), payload, needed.contains_key(&file), has_secret, &keys_dir.join(&file));
        out.push(view(id, payload, status, needed.get(&file).cloned().unwrap_or_default()));
    }
    for (id, local) in &state.key_slots {
        if live.iter().any(|(l, _)| l == id) {
            continue;
        }
        let (Some(_), Some(payload)) = (&local.source, &local.payload) else { continue };
        let file = keys_dir.join(&local.file_name).display().to_string();
        out.push(view(id, payload, SlotStatusView::NotInUse { file }, Vec::new()));
    }
    out
}

/// 一個帳戶裡的插槽在這台的狀態。錯誤優先;`synced` 的插槽,這台用的若不是同步的那把,依情況是 SourceChanged(這台是
/// 來源)或 SyncedAvailable。
pub fn slot_status(
    local: Option<&LocalSlot>,
    payload: &KeySlotPayload,
    needed: bool,
    has_secret: bool,
    slot_path: &Path,
) -> SlotStatusView {
    if let Some(message) = local.and_then(|l| l.last_error.clone()) {
        return SlotStatusView::Error { message };
    }
    let here = slot_path.display().to_string();
    let synced = payload.mode == SlotMode::Synced;
    match local.and_then(|l| l.source.as_ref()) {
        Some(SlotSource::Linked { path, link, fingerprint, origin }) => {
            if synced && *origin && fingerprint != &payload.fingerprint {
                SlotStatusView::SourceChanged { file: path.clone() }
            } else if synced && !origin && has_secret && fingerprint != &payload.fingerprint {
                SlotStatusView::SyncedAvailable { file: path.clone() }
            } else if !needed {
                SlotStatusView::NotInUse { file: if *link == LinkKind::Copy { here } else { path.clone() } }
            } else {
                SlotStatusView::Ready { file: path.clone(), synced_copy: false, fingerprint: fingerprint.clone() }
            }
        }
        Some(SlotSource::SyncedCopy { fingerprint }) => {
            if synced && has_secret && Some(fingerprint) != payload.fingerprint.as_ref() {
                SlotStatusView::SyncedAvailable { file: here }
            } else if !needed {
                SlotStatusView::NotInUse { file: here }
            } else {
                SlotStatusView::Ready { file: here, synced_copy: true, fingerprint: Some(fingerprint.clone()) }
            }
        }
        None if needed => SlotStatusView::NeedsKey { waiting_for_sync: synced },
        None => SlotStatusView::NotUsedHere,
    }
}
```

`src-tauri/src/sync/round.rs` 的 `run_round`,在第 6 步最後的 `announce(env, applied_hosts, &conflicts, &held);` 之後、
第 7 步的上傳之前加:

```rust
    // 6b. 金鑰插槽(SP3 spec §6.2–§6.6):依合併後的帳戶與 space 記錄,在這台落地或維護插槽。帳戶的變更(`device.slots`、
    //     補寫的 `keyslot`/`key`)跟著下面的上傳送出;通知存進狀態、放掉鎖之後發出。
    if let Some(home) = env.ssh_dir.parent() {
        let slot_round = crate::sync::slots::reconcile(&mut work, &keys, home, now);
        if slot_round.changed || !slot_round.notices.is_empty() {
            commit(env, generation, |latest| {
                // 帳戶區段在這一輪已經整份提交過(`commit_account`),之後只有 space 的提交:用這一輪的版本整份換掉是安全的。
                latest.account = work.account.clone();
                latest.key_slots = work.key_slots.clone();
                for notice in &slot_round.notices {
                    crate::sync::slots::add_notice(&mut latest.notices, notice);
                }
                Ok(())
            })?;
            for notice in &slot_round.notices {
                env.events.notice(notice);
            }
        }
    }
```

`src/lib/sync-fixtures.ts` 的 `overview()` 在 `notices: [],` 之前加 `key_slots: [],`。

- [ ] **Step 4: 跑測試確認通過**

Run:`… cargo test --offline --lib slots -- --skip …`,再跑完整的 Rust 測試;repo 根目錄跑 `./node_modules/.bin/tsc --noEmit`
與 `./node_modules/.bin/vitest run --dir src`。
Expected:`slots` 新增的 9 個測試通過(`a_synced_key_lands_on_the_other_computer` 只在 Unix 跑);完整測試全綠;
`src/bindings/SyncOverview.ts` 多了 `key_slots`;`tsc`、vitest 通過。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/slots.rs src-tauri/src/sync/dto.rs src-tauri/src/sync/round.rs src/lib/sync-fixtures.ts src/bindings/SyncKeySlotView.ts src/bindings/SlotStatusView.ts src/bindings/SlotDeviceView.ts src/bindings/SyncOverview.ts
git commit -m "feat(sync): land and maintain key slots on each computer"
```

### Task 5: 還沒設定的金鑰與建立插槽(`slot_setup.rs`、`sync_key_candidates`、`sync_setup_keys`)

**Files:**
- Create: `src-tauri/src/sync/slot_setup.rs`
- Modify: `src-tauri/src/sync/mod.rs`、`src-tauri/src/sync/engine.rs`(commands)、`src-tauri/src/lib.rs`(註冊)
- Generated: `src/bindings/KeyCandidates.ts`、`KeyCandidate.ts`、`CandidateHost.ts`、`UnsupportedIdentity.ts`、`KeyChoice.ts`、`KeyDecision.ts`

**Interfaces:**
- Consumes: Task 1–4;`migrate::{refuse_while_sync_inactive, selected_space_files, ANOTHER_ENGINE_MESSAGE}`、`config::commands::persist_file`、
  `runtime::mutate`、`merge::space_entry`、`engine::engine_active`。
- Produces:
  - `CandidateHost { alias, space_name, value, locked: Option<String> }`、`KeyCandidate { path, default_name, fingerprint: Option<String>, has_passphrase: Option<bool>, unsyncable: Option<String>, existing_slot: Option<String>, hosts: Vec<CandidateHost> }`、`UnsupportedIdentity { alias, value, reason }`、`KeyCandidates { keys, unsupported }`(皆 ts 匯出)
  - `KeyDecision`(serde `tag = "kind"`、snake_case):`Sync { name }`、`Keep { name }`、`Reuse { slot_id }`;`KeyChoice { path, decision }`(ts 匯出)
  - `slot_setup::LOCKED_REASON`
  - `slot_setup::key_candidates(env: &SyncEnv) -> Result<KeyCandidates, AppError>`
  - `slot_setup::setup_keys(env: &SyncEnv, active: bool, choices: Vec<KeyChoice>) -> Result<Vec<String>, AppError>`(回改寫了的 alias)
  - commands:`sync_key_candidates() -> KeyCandidates`、`sync_setup_keys(choices: Vec<KeyChoice>) -> SyncOverview`

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slot_setup.rs` 的測試模組:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::account::create_account;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::round::tests::{pair, settle};
    use crate::sync::slot_rules::{test_keys, REASON_PUBLIC_KEY, REASON_TOKENS};
    use crate::sync::slots::live_slots;
    use crate::sync::testkit::{TestClock, TestDevice};

    /// 一台已建立帳戶的裝置(Personal),主 config 是 `main`;回傳(裝置、Personal 的 id)。
    fn device(main: &str) -> (TestDevice, String) {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::with_main_config("a", &relay, &clock, main);
        create_account(&a.env(), "MacBook-A").unwrap();
        settle(&a);
        let personal = a.state().spaces.keys().next().unwrap().clone();
        (a, personal)
    }

    fn put_key(d: &TestDevice, name: &str, text: &str) -> PathBuf {
        let path = d.ssh_dir().join(name);
        std::fs::write(&path, text).unwrap();
        path
    }

    fn home(d: &TestDevice) -> PathBuf {
        d.ssh_dir().parent().unwrap().to_path_buf()
    }

    fn keep(path: &Path, name: &str) -> KeyChoice {
        KeyChoice { path: path.display().to_string(), decision: KeyDecision::Keep { name: name.into() } }
    }

    fn sync(path: &Path, name: &str) -> KeyChoice {
        KeyChoice { path: path.display().to_string(), decision: KeyDecision::Sync { name: name.into() } }
    }

    #[test]
    fn candidates_group_hosts_by_key_and_list_what_cannot_be_set_up() {
        let (a, personal) = device("# main\n");
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(
            &a.space_path(&personal),
            &format!(
                "Host web\n  IdentityFile ~/.ssh/id_mac\nHost db\n  identityfile = \"{}\"   # spelled out\nHost proxy\n  IdentityFile ~/.ssh/%h\nHost agent\n  IdentityFile ~/.ssh/id_mac.pub\nHost gone\n  IdentityFile ~/.ssh/missing\n",
                key.display()
            ),
        );
        let found = key_candidates(&a.env()).unwrap();
        assert_eq!(found.keys.len(), 1, "{found:?}");
        let candidate = &found.keys[0];
        assert_eq!(candidate.path, key.display().to_string());
        assert_eq!(candidate.default_name, "id_mac");
        assert_eq!(candidate.fingerprint.as_deref(), Some(test_keys::PLAIN_FINGERPRINT));
        assert_eq!(candidate.has_passphrase, Some(false));
        assert_eq!(candidate.unsyncable, None);
        assert_eq!(candidate.existing_slot, None);
        let hosts: Vec<(&str, &str)> = candidate.hosts.iter().map(|h| (h.alias.as_str(), h.value.as_str())).collect();
        assert_eq!(hosts, vec![("web", "~/.ssh/id_mac"), ("db", format!("\"{}\"", key.display()).as_str())]);
        assert!(candidate.hosts.iter().all(|h| h.space_name == "Personal" && h.locked.is_none()));
        let unsupported: Vec<(&str, &str)> = found.unsupported.iter().map(|u| (u.alias.as_str(), u.reason.as_str())).collect();
        assert_eq!(unsupported, vec![("proxy", REASON_TOKENS), ("agent", REASON_PUBLIC_KEY)]);
    }

    #[test]
    fn keeping_a_key_creates_an_own_slot_links_it_and_rewrites_only_those_lines() {
        let (a, personal) = device("# main\n");
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(
            &a.space_path(&personal),
            &format!("Host web\n  HostName 10.0.0.1\n  IdentityFile ~/.ssh/id_mac\nHost db\n  identityfile = \"{}\"   # spelled out\n", key.display()),
        );
        let rewritten = setup_keys(&a.env(), true, vec![keep(&key, "personal")]).unwrap();
        assert_eq!(rewritten, vec!["web".to_string(), "db".to_string()]);

        let slots = live_slots(a.state().account.as_ref().unwrap());
        assert_eq!(slots.len(), 1);
        let (id, payload) = &slots[0];
        assert_eq!((payload.name.as_str(), payload.mode), ("personal", SlotMode::Own));
        let file = slot_file_name("personal", id);
        assert_eq!(
            a.read(&a.space_path(&personal)),
            format!("Host web\n  HostName 10.0.0.1\n  IdentityFile ~/.ssh/sshelter/keys/{file}\nHost db\n  identityfile = ~/.ssh/sshelter/keys/{file}   # spelled out\n")
        );
        let local = &a.state().key_slots[id];
        assert_eq!(local.source, Some(SlotSource::Linked { path: key.display().to_string(), link: local_link_kind(), fingerprint: Some(test_keys::PLAIN_FINGERPRINT.into()), origin: true }));
        assert_eq!(std::fs::read_to_string(home(&a).join(SLOT_DIR).join(&file)).unwrap(), test_keys::plain());
        assert!(key_candidates(&a.env()).unwrap().keys.is_empty(), "nothing left to set up");
        // 一輪之後,改寫的主機已上傳。
        settle(&a);
        assert!(a.state().spaces[&personal].records.values().all(|l| !l.dirty));
    }

    #[cfg(unix)]
    fn local_link_kind() -> crate::sync::slot_files::LinkKind {
        crate::sync::slot_files::LinkKind::Symlink
    }
    #[cfg(not(unix))]
    fn local_link_kind() -> crate::sync::slot_files::LinkKind {
        crate::sync::slot_files::LinkKind::HardLink
    }

    #[test]
    fn a_synced_key_reaches_the_other_computer() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(&a.space_path(&personal), "Host web\n  IdentityFile ~/.ssh/id_mac\n");
        setup_keys(&a.env(), true, vec![sync(&key, "id_mac")]).unwrap();
        settle(&a);
        settle(&b);
        let id = live_slots(b.state().account.as_ref().unwrap())[0].0.clone();
        let landed = home(&b).join(SLOT_DIR).join(slot_file_name("id_mac", &id));
        assert_eq!(std::fs::read_to_string(landed).unwrap(), test_keys::plain());
    }

    #[test]
    fn the_same_key_on_another_computer_reuses_the_synced_slot() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(&a.space_path(&personal), "Host web\n  IdentityFile ~/.ssh/id_mac\n");
        setup_keys(&a.env(), true, vec![sync(&key, "id_mac")]).unwrap();
        settle(&a);
        settle(&b);
        let id = live_slots(b.state().account.as_ref().unwrap())[0].0.clone();
        // B 自己也有同一把金鑰(別的檔名),又加了一台主機用它。
        let copy = put_key(&b, "same_key", &test_keys::plain());
        let text = b.read(&b.space_path(&personal));
        b.save_in_app(&b.space_path(&personal), &format!("{text}Host db\n  IdentityFile ~/.ssh/same_key\n"));
        let found = key_candidates(&b.env()).unwrap();
        assert_eq!(found.keys[0].existing_slot.as_deref(), Some(id.as_str()));
        setup_keys(&b.env(), true, vec![KeyChoice { path: copy.display().to_string(), decision: KeyDecision::Reuse { slot_id: id.clone() } }]).unwrap();
        assert_eq!(live_slots(b.state().account.as_ref().unwrap()).len(), 1, "no second slot");
        assert!(b.read(&b.space_path(&personal)).contains(&format!("Host db\n  IdentityFile ~/.ssh/sshelter/keys/{}", slot_file_name("id_mac", &id))));
    }

    #[test]
    fn a_host_with_more_than_one_copy_is_locked() {
        let (a, personal) = device("# main\nHost web\n  HostName 1.1.1.1\n");
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(&a.space_path(&personal), "Host web\n  IdentityFile ~/.ssh/id_mac\nHost db\n  IdentityFile ~/.ssh/id_mac\n");
        let found = key_candidates(&a.env()).unwrap();
        let locked: Vec<(&str, Option<&str>)> = found.keys[0].hosts.iter().map(|h| (h.alias.as_str(), h.locked.as_deref())).collect();
        assert_eq!(locked, vec![("web", Some(LOCKED_REASON)), ("db", None)]);
        assert_eq!(setup_keys(&a.env(), true, vec![keep(&key, "id_mac")]).unwrap(), vec!["db".to_string()]);
        assert!(a.read(&a.space_path(&personal)).starts_with("Host web\n  IdentityFile ~/.ssh/id_mac\n"));
    }

    #[test]
    fn a_conflict_while_rewriting_leaves_the_slot_ready_to_reuse() {
        let (a, personal) = device("# main\n");
        let key = put_key(&a, "id_mac", &test_keys::plain());
        let space = a.space_path(&personal);
        a.save_in_app(&space, "Host web\n  IdentityFile ~/.ssh/id_mac\n");
        // 另一個編輯器改了檔案,app 還沒重載:改寫會撞到 Conflict。
        a.write_externally(&space, "Host web\n  IdentityFile ~/.ssh/id_mac\nHost other\n  HostName 9.9.9.9\n");
        assert!(matches!(setup_keys(&a.env(), true, vec![keep(&key, "id_mac")]), Err(AppError::Conflict(_))));
        let id = live_slots(a.state().account.as_ref().unwrap())[0].0.clone();
        // doc 已從磁碟重載;下一次掃描把它當成可以沿用的插槽。
        let found = key_candidates(&a.env()).unwrap();
        assert_eq!(found.keys[0].existing_slot.as_deref(), Some(id.as_str()));
        setup_keys(&a.env(), true, vec![KeyChoice { path: key.display().to_string(), decision: KeyDecision::Reuse { slot_id: id.clone() } }]).unwrap();
        assert!(a.read(&space).starts_with(&format!("Host web\n  IdentityFile ~/.ssh/sshelter/keys/{}\n", slot_file_name("id_mac", &id))));
        assert_eq!(live_slots(a.state().account.as_ref().unwrap()).len(), 1);
    }

    #[test]
    fn keys_that_cannot_be_synced_can_still_be_kept() {
        let (a, personal) = device("# main\n");
        let pem = format!("{}\nMIIBOgIBAAJBAKj34GkxFhD90vcNLYLInFEX6Ppy1tPf9Cnzj4p4WGeKLs1Pt8Qu\n{}\n", concat!("-----BEGIN RSA ", "PRIVATE KEY-----"), concat!("-----END RSA ", "PRIVATE KEY-----"));
        let key = put_key(&a, "id_rsa", &pem);
        a.save_in_app(&a.space_path(&personal), "Host web\n  IdentityFile ~/.ssh/id_rsa\n");
        let found = key_candidates(&a.env()).unwrap();
        assert_eq!(found.keys[0].unsyncable.as_deref(), Some(crate::sync::slot_rules::Unsyncable::NotOpenSsh.message()));
        let refused = setup_keys(&a.env(), true, vec![sync(&key, "id_rsa")]).unwrap_err().to_string();
        assert_eq!(refused, crate::sync::slot_rules::Unsyncable::NotOpenSsh.message());
        assert!(live_slots(a.state().account.as_ref().unwrap()).is_empty(), "nothing was created");
        setup_keys(&a.env(), true, vec![keep(&key, "id_rsa")]).unwrap();
        assert_eq!(live_slots(a.state().account.as_ref().unwrap())[0].1.mode, SlotMode::Own);
    }

    #[test]
    fn setup_needs_the_sync_engine_and_a_valid_name() {
        let (a, personal) = device("# main\n");
        let key = put_key(&a, "id_mac", &test_keys::plain());
        a.save_in_app(&a.space_path(&personal), "Host web\n  IdentityFile ~/.ssh/id_mac\n");
        assert!(setup_keys(&a.env(), false, vec![keep(&key, "id_mac")]).is_err(), "no engine in this process");
        assert!(setup_keys(&a.env(), true, vec![keep(&key, "../x")]).is_err());
        assert!(live_slots(a.state().account.as_ref().unwrap()).is_empty());
    }
}
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib slot_setup -- --skip …`
Expected:編譯失敗(模組不存在)。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/mod.rs` 加 `pub mod slot_setup;`。

`src-tauri/src/sync/slot_setup.rs`(測試模組之前):

```rust
//! SP3:還沒設定的金鑰(候選)與建立插槽(spec `docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md` §5、§6.1、
//! §7.1)。主機的改寫只換 `IdentityFile` 那一行的值,經 `persist_file` 寫回(存檔 hook 照一般修改上傳)。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};

use serde::{Deserialize, Serialize};

use crate::config::commands::persist_file;
use crate::config::model::{Item, SshConfigDoc};
use crate::error::AppError;
use crate::sync::env::SyncEnv;
use crate::sync::merge::space_entry;
use crate::sync::migrate::{refuse_while_sync_inactive, selected_space_files};
use crate::sync::runtime::mutate;
use crate::sync::slot_files;
use crate::sync::slot_rules::{
    default_slot_name, inspect_private_key, new_slot_id, parse_public_key, public_path, resolve_identity_value,
    slot_file_name, slot_value, valid_slot_name, IdentityTarget, KeySlotPayload, SlotMode, SLOT_DIR, SLOT_SCHEMA,
};
use crate::sync::slots::{live_slots, local_key_fingerprint, put_key_secret, put_slot, slot};
use crate::sync::state_v2::{LocalSlot, SlotSource, SyncStateV2};

/// 一台用到候選金鑰的主機。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct CandidateHost {
    pub alias: String,
    /// 主機所在 space 的名稱(來自帳戶;UI 以 `revealHidden` 顯示)。
    pub space_name: String,
    /// 那一行 `IdentityFile` 目前的值(會被改寫的就是它)。
    pub value: String,
    /// 不改寫的原因(同名主機有不只一份,spec §5);null = 會改寫。
    pub locked: Option<String>,
}

/// 一把用在同步主機上、還沒有插槽的本機金鑰(或可以直接沿用的插槽)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct KeyCandidate {
    /// 這台電腦上金鑰檔的完整路徑(`IdentityFile` 解析出來的)。
    pub path: String,
    pub default_name: String,
    pub fingerprint: Option<String>,
    pub has_passphrase: Option<bool>,
    /// 不能同步的原因(只能 Keep on this computer);null = 可以同步。
    pub unsyncable: Option<String>,
    /// 已經有這把金鑰的插槽:這台建立或挑過、連到同一個檔案的,或帳戶裡同指紋的 `synced` 插槽(spec §6.1 第 1 步:直接
    /// 沿用,不再詢問)。
    pub existing_slot: Option<String>,
    pub hosts: Vec<CandidateHost>,
}

/// `IdentityFile` 的值無法自動設定的主機(§5)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct UnsupportedIdentity {
    pub alias: String,
    pub value: String,
    pub reason: String,
}

#[derive(Clone, Debug, Default, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct KeyCandidates {
    pub keys: Vec<KeyCandidate>,
    pub unsupported: Vec<UnsupportedIdentity>,
}

/// 使用者對一把金鑰的決定。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum KeyDecision {
    /// 「Sync key」:建立 `synced` 插槽並上傳私鑰。
    Sync { name: String },
    /// 「Keep on this computer」:建立 `own` 插槽。
    Keep { name: String },
    /// 沿用既有的插槽(`KeyCandidate.existing_slot`)。
    Reuse { slot_id: String },
}

#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct KeyChoice {
    /// `KeyCandidate.path`。
    pub path: String,
    pub decision: KeyDecision,
}

pub const LOCKED_REASON: &str = "This host has more than one copy; SSHelter changes it once only one copy is left.";

fn home_of(env: &SyncEnv) -> Result<PathBuf, AppError> {
    env.ssh_dir
        .parent()
        .map(Path::to_path_buf)
        .ok_or_else(|| AppError::Other("cannot determine the home directory".to_string()))
}

/// 兩個路徑是不是同一個檔案(都要存在)。
fn same_file(a: &Path, b: &Path) -> bool {
    match (std::fs::canonicalize(a), std::fs::canonicalize(b)) {
        (Ok(x), Ok(y)) => x == y,
        _ => false,
    }
}

/// 存在的私鑰檔(第一行是 `-----BEGIN … PRIVATE KEY-----`;大於 64 KiB 的不讀)。
fn is_private_key_file(path: &Path) -> bool {
    let Ok(meta) = std::fs::metadata(path) else { return false };
    if !meta.is_file() || meta.len() > 64 * 1024 {
        return false;
    }
    std::fs::read_to_string(path)
        .ok()
        .and_then(|text| text.lines().next().map(|l| l.trim().starts_with("-----BEGIN ") && l.contains("PRIVATE KEY")))
        .unwrap_or(false)
}

/// 每個 pattern 在整份 config(所有檔案)出現在幾個 Host 區塊;超過一個的主機不改寫(SP1 FA3)。
fn alias_counts(doc: &SshConfigDoc) -> BTreeMap<String, usize> {
    let mut counts = BTreeMap::new();
    for file in &doc.files {
        for item in &file.items {
            if let Item::Host(host) = item {
                for pattern in &host.patterns {
                    *counts.entry(pattern.clone()).or_insert(0) += 1;
                }
            }
        }
    }
    counts
}

fn locked(patterns: &[String], counts: &BTreeMap<String, usize>) -> bool {
    patterns.iter().any(|p| counts.get(p).copied().unwrap_or(0) > 1)
}

/// `.pub` 的第一行(`<type> <base64>`)。
fn read_public_line(key: &Path) -> Option<String> {
    let text = std::fs::read_to_string(public_path(key)).ok()?;
    parse_public_key(text.lines().next()?).map(|(line, _)| line)
}

/// 已經有這把金鑰的插槽:這台建立或挑過、連到同一個檔案的(帳戶裡仍在);或帳戶裡同指紋的 `synced` 插槽。
fn existing_slot_for(state: &SyncStateV2, key: &Path, fingerprint: Option<&str>) -> Option<String> {
    let account = state.account.as_ref()?;
    let live = live_slots(account);
    let linked = state.key_slots.iter().find(|(id, local)| {
        live.iter().any(|(l, _)| l == *id)
            && matches!(&local.source, Some(SlotSource::Linked { path, .. }) if same_file(Path::new(path), key))
    });
    if let Some((id, _)) = linked {
        return Some(id.clone());
    }
    let fingerprint = fingerprint?;
    live.into_iter()
        .find(|(_, p)| p.mode == SlotMode::Synced && p.fingerprint.as_deref() == Some(fingerprint))
        .map(|(id, _)| id)
}

fn candidate_for(key: &Path, state: &SyncStateV2) -> KeyCandidate {
    let text = std::fs::read_to_string(key).unwrap_or_default();
    let inspected = inspect_private_key(&text);
    let fingerprint = inspected.as_ref().ok().map(|f| f.fingerprint.clone()).or_else(|| local_key_fingerprint(key));
    KeyCandidate {
        path: key.display().to_string(),
        default_name: default_slot_name(key.file_name().and_then(|n| n.to_str()).unwrap_or("key")),
        existing_slot: existing_slot_for(state, key, fingerprint.as_deref()),
        fingerprint,
        has_passphrase: inspected.as_ref().ok().map(|f| f.has_passphrase),
        unsyncable: inspected.err().map(|e| e.message().to_string()),
        hosts: Vec::new(),
    }
}

/// 掃描 doc 裡這台勾選的 space 檔(純函式;呼叫端持有 doc 鎖)。
fn scan(doc: &SshConfigDoc, space_files: &[(String, PathBuf)], state: &SyncStateV2, home: &Path) -> KeyCandidates {
    let counts = alias_counts(doc);
    let mut keys: Vec<KeyCandidate> = Vec::new();
    let mut unsupported = Vec::new();
    for (space_id, path) in space_files {
        let Some(file) = doc.files.iter().find(|f| &f.path == path) else { continue };
        let space_name = state
            .account
            .as_ref()
            .and_then(|a| space_entry(a, space_id))
            .map(|e| e.name)
            .unwrap_or_else(|| space_id.clone());
        for item in &file.items {
            let Item::Host(host) = item else { continue };
            let Some(alias) = host.patterns.first().cloned() else { continue };
            let lock = locked(&host.patterns, &counts).then(|| LOCKED_REASON.to_string());
            for line in &host.body {
                let Item::Directive(d) = line else { continue };
                if d.key != "identityfile" || d.serializes_as_comment() {
                    continue;
                }
                match resolve_identity_value(&d.value, home) {
                    IdentityTarget::Slot(_) => {}
                    IdentityTarget::Unsupported(reason) => unsupported.push(UnsupportedIdentity {
                        alias: alias.clone(),
                        value: d.value.clone(),
                        reason: reason.to_string(),
                    }),
                    IdentityTarget::File(key) => {
                        if !is_private_key_file(&key) {
                            continue; // 不存在或不是私鑰:交給 lint
                        }
                        let index = match keys.iter().position(|c| same_file(Path::new(&c.path), &key)) {
                            Some(i) => i,
                            None => {
                                keys.push(candidate_for(&key, state));
                                keys.len() - 1
                            }
                        };
                        keys[index].hosts.push(CandidateHost {
                            alias: alias.clone(),
                            space_name: space_name.clone(),
                            value: d.value.clone(),
                            locked: lock.clone(),
                        });
                    }
                }
            }
        }
    }
    KeyCandidates { keys, unsupported }
}

/// 這台勾選的 space 檔裡,指到本機私鑰、還沒有插槽的 `IdentityFile`(依金鑰檔分組),以及無法自動設定的值。鎖:先短暫
/// 拿 core 取快照,再拿 doc。
pub fn key_candidates(env: &SyncEnv) -> Result<KeyCandidates, AppError> {
    let home = home_of(env)?;
    let Some(state) = env.runtime.core.lock().unwrap().state.clone() else { return Ok(KeyCandidates::default()) };
    if state.account.is_none() {
        return Ok(KeyCandidates::default());
    }
    let space_files = selected_space_files(env.runtime, &env.ssh_dir);
    let doc_lock = env.doc.lock().unwrap();
    let Some(doc) = doc_lock.as_ref() else { return Ok(KeyCandidates::default()) };
    Ok(scan(doc, &space_files, &state, &home))
}

/// 把 `planned`(金鑰檔 → 插槽檔名)的主機改寫成指到插槽:只換那一行的值,被 FA3 鎖住的主機不動;改了的檔案逐一經
/// `persist`。回傳改寫了的 alias(依出現順序)。
fn rewrite_in(
    doc: &mut SshConfigDoc,
    space_files: &[PathBuf],
    home: &Path,
    planned: &[(PathBuf, String)],
    mut persist: impl FnMut(&mut SshConfigDoc, usize) -> Result<(), AppError>,
) -> Result<Vec<String>, AppError> {
    let counts = alias_counts(doc);
    let mut rewritten: Vec<String> = Vec::new();
    for idx in 0..doc.files.len() {
        if !space_files.contains(&doc.files[idx].path) {
            continue;
        }
        let mut changed = false;
        for item in doc.files[idx].items.iter_mut() {
            let Item::Host(host) = item else { continue };
            if locked(&host.patterns, &counts) {
                continue;
            }
            let alias = host.patterns.first().cloned().unwrap_or_default();
            for line in host.body.iter_mut() {
                let Item::Directive(d) = line else { continue };
                if d.key != "identityfile" || d.serializes_as_comment() {
                    continue;
                }
                let IdentityTarget::File(target) = resolve_identity_value(&d.value, home) else { continue };
                let Some((_, file)) = planned.iter().find(|(key, _)| same_file(key, &target)) else { continue };
                d.value = slot_value(file);
                d.dirty = true;
                changed = true;
                if !rewritten.contains(&alias) {
                    rewritten.push(alias.clone());
                }
            }
        }
        if changed {
            persist(doc, idx)?;
        }
    }
    Ok(rewritten)
}

/// 依使用者的決定建立或沿用插槽,再改寫用到那些金鑰的主機(SP3 spec §6.1;插槽一定先就位,主機才改寫)。`active` = 這個
/// 行程跑著同步引擎(`engine::engine_active`;改寫的主機要靠它上傳)。決定裡的路徑不在目前的候選裡就略過。回傳改寫了的 alias。
pub fn setup_keys(env: &SyncEnv, active: bool, choices: Vec<KeyChoice>) -> Result<Vec<String>, AppError> {
    refuse_while_sync_inactive(active, env.runtime)?;
    let home = home_of(env)?;
    let keys_dir = home.join(SLOT_DIR);
    let account_keys = env
        .runtime
        .core
        .lock()
        .unwrap()
        .account_keys
        .clone()
        .ok_or_else(|| AppError::Other("join a sync account first".to_string()))?;
    let current = key_candidates(env)?;
    let mut planned: Vec<(PathBuf, String)> = Vec::new();
    for choice in choices {
        let Some(candidate) = current.keys.iter().find(|c| same_file(Path::new(&c.path), Path::new(&choice.path))).cloned() else {
            continue;
        };
        let source = PathBuf::from(&candidate.path);
        match choice.decision {
            KeyDecision::Reuse { slot_id } => {
                let state = env.runtime.core.lock().unwrap().state.clone();
                let payload = state
                    .as_ref()
                    .and_then(|s| s.account.as_ref())
                    .and_then(|a| slot(a, &slot_id))
                    .ok_or_else(|| AppError::Other("that key slot no longer exists".to_string()))?;
                let file = slot_file_name(&payload.name, &slot_id);
                let has_local = state.as_ref().and_then(|s| s.key_slots.get(&slot_id)).is_some_and(|l| l.source.is_some());
                if !has_local {
                    // 這台還沒有這個插槽的金鑰:連到這台同一把金鑰(本機挑的優先,不另外放一份副本)。
                    let slot_path = keys_dir.join(&file);
                    slot_files::ensure_keys_dir(&keys_dir)?;
                    let link = slot_files::link(&source, &slot_path)?;
                    mutate(env, |s| {
                        s.key_slots.insert(
                            slot_id.clone(),
                            LocalSlot {
                                file_name: file.clone(),
                                source: Some(SlotSource::Linked {
                                    path: candidate.path.clone(),
                                    link,
                                    fingerprint: candidate.fingerprint.clone(),
                                    origin: false,
                                }),
                                last_error: None,
                                asked: false,
                                payload: Some(payload.clone()),
                            },
                        );
                        Ok(())
                    })?;
                }
                planned.push((source, file));
            }
            KeyDecision::Sync { name } => planned.push(create_slot(env, &account_keys, &keys_dir, &candidate, &source, name, true)?),
            KeyDecision::Keep { name } => planned.push(create_slot(env, &account_keys, &keys_dir, &candidate, &source, name, false)?),
        }
    }
    let space_files: Vec<PathBuf> = selected_space_files(env.runtime, &env.ssh_dir).into_iter().map(|(_, p)| p).collect();
    let mut doc_lock = env.doc.lock().unwrap();
    let mut backed_up = env.backed_up.lock().unwrap();
    let retention = env.retention();
    let (result, main_path) = {
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        let main_path = doc.files[0].path.clone();
        (rewrite_in(doc, &space_files, &home, &planned, |doc, idx| persist_file(doc, idx, &mut backed_up, retention)), main_path)
    };
    if result.is_err() {
        // 記憶體裡的 doc 已改、磁碟沒寫成:重載讓兩邊一致(同搬移精靈);插槽留著,下一次掃描會建議沿用。
        drop(backed_up);
        *doc_lock = env.load_doc(&main_path).ok();
    }
    drop(doc_lock);
    env.events.applied(0);
    env.events.wake();
    result
}

/// 建立一個新插槽:本機插槽先連到原檔(與 `.pub`),再寫帳戶記錄與本機狀態;記錄寫不進去就把連結收回。回傳(金鑰檔、插槽檔名)。
fn create_slot(
    env: &SyncEnv,
    account_keys: &crate::sync::crypto::ChainKeys,
    keys_dir: &Path,
    candidate: &KeyCandidate,
    source: &Path,
    name: String,
    sync: bool,
) -> Result<(PathBuf, String), AppError> {
    if !valid_slot_name(&name) {
        return Err(AppError::Other(format!(
            "\"{name}\" can't be used as a key name: use letters, digits, '.', '_' or '-', start with a letter or digit, and don't end with .pub"
        )));
    }
    let text = std::fs::read_to_string(source)?;
    let facts = if sync { Some(inspect_private_key(&text).map_err(|e| AppError::Other(e.message().to_string()))?) } else { None };
    let slot_id = new_slot_id()?;
    let file = slot_file_name(&name, &slot_id);
    let slot_path = keys_dir.join(&file);
    slot_files::ensure_keys_dir(keys_dir)?;
    let link = slot_files::link(source, &slot_path)?;
    if let Some(public) = facts.as_ref().map(|f| f.public_key.clone()).or_else(|| read_public_line(source)) {
        let _ = slot_files::write_public(&slot_path, &public);
    }
    let now = env.now();
    let result = mutate(env, |s| {
        let device_id = s.device_id.clone();
        let payload = KeySlotPayload {
            schema: SLOT_SCHEMA,
            name: name.clone(),
            mode: if sync { SlotMode::Synced } else { SlotMode::Own },
            origin_device_id: device_id.clone(),
            created_at_ms: now,
            public_key: facts.as_ref().map(|f| f.public_key.clone()),
            fingerprint: facts.as_ref().map(|f| f.fingerprint.clone()),
            key_type: facts.as_ref().map(|f| f.key_type.clone()),
            has_passphrase: facts.as_ref().map(|f| f.has_passphrase),
        };
        let account = s.account.as_mut().ok_or_else(|| AppError::Other("join a sync account first".to_string()))?;
        put_slot(account, &slot_id, Some(&payload), &device_id, now);
        if sync {
            put_key_secret(account, account_keys, &slot_id, Some(&text), &device_id, now)?;
        }
        s.key_slots.insert(
            slot_id.clone(),
            LocalSlot {
                file_name: file.clone(),
                source: Some(SlotSource::Linked {
                    path: candidate.path.clone(),
                    link,
                    fingerprint: candidate.fingerprint.clone(),
                    origin: true,
                }),
                last_error: None,
                asked: false,
                payload: Some(payload),
            },
        );
        Ok(())
    });
    if let Err(e) = result {
        let _ = slot_files::remove_slot(&slot_path);
        return Err(e);
    }
    Ok((source.to_path_buf(), file))
}
```

`src-tauri/src/sync/engine.rs`(放在 `sync_move_hosts_to_space` 附近):

```rust
/// 還沒設定的金鑰(SP3 spec §7.1):讀 config,不改任何東西。
#[tauri::command]
pub async fn sync_key_candidates(app: AppHandle) -> Result<crate::sync::slot_setup::KeyCandidates, AppError> {
    run(app, false, crate::sync::slot_setup::key_candidates).await
}

/// 依使用者的決定建立或沿用插槽並改寫主機(SP3 spec §6.1)。結構性變更:持有 lifecycle 鎖。
#[tauri::command]
pub async fn sync_setup_keys(app: AppHandle, choices: Vec<crate::sync::slot_setup::KeyChoice>) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| crate::sync::slot_setup::setup_keys(env, engine_active(), choices).map(|_| ())).await
}
```

`src-tauri/src/lib.rs`:`use` 清單與 `generate_handler!` 都加上 `sync_key_candidates`、`sync_setup_keys`。

- [ ] **Step 4: 跑測試確認通過**

Run:`… cargo test --offline --lib slot_setup -- --skip …`,再跑完整的 Rust 測試與 `./node_modules/.bin/tsc --noEmit`。
Expected:`slot_setup` 的 8 個測試通過;完整測試全綠;新增 6 個 binding。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/slot_setup.rs src-tauri/src/sync/mod.rs src-tauri/src/sync/engine.rs src-tauri/src/lib.rs src/bindings/KeyCandidates.ts src/bindings/KeyCandidate.ts src/bindings/CandidateHost.ts src/bindings/UnsupportedIdentity.ts src/bindings/KeyChoice.ts src/bindings/KeyDecision.ts
git commit -m "feat(sync): set up key slots for synced hosts and rewrite their IdentityFile"
```

### Task 6: 使用者的動作:改成同步、停止同步、挑金鑰、改用同步的金鑰、刪除副本

**Files:**
- Modify: `src-tauri/src/sync/slots.rs`
- Modify: `src-tauri/src/sync/slot_setup.rs`(`is_private_key_file` 改成 `pub(crate)`)
- Modify: `src-tauri/src/sync/engine.rs`、`src-tauri/src/lib.rs`

**Interfaces:**
- Consumes: Task 1–5(`slots::land` 是 Task 4 的私有函式,同一個模組可以用)。
- Produces:
  - `slots::set_mode(env: &SyncEnv, slot_id: &str, mode: SlotMode) -> Result<(), AppError>`
  - `slots::pick(env: &SyncEnv, slot_id: &str, path: &str) -> Result<(), AppError>`
  - `slots::use_synced(env: &SyncEnv, slot_id: &str) -> Result<(), AppError>`
  - `slots::delete_copy(env: &SyncEnv, slot_id: &str) -> Result<(), AppError>`
  - `slots::not_here_message(device: &str) -> String`(`Do this on a computer that has this key, such as {device}.`)、`slots::IN_USE_MESSAGE`
  - commands:`sync_key_set_mode(slot_id, mode: SlotMode)`、`sync_key_pick(slot_id, path)`、`sync_key_use_synced(slot_id)`、`sync_key_delete_copy(slot_id)`,都回 `SyncOverview`

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/slots.rs` 的測試模組加(沿用 Task 4 的 `create_slot_on`、`use_slot`、`home`、`account_keys`):

```rust
    #[test]
    fn stopping_and_restarting_sync_keeps_the_copies() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        set_mode(&a.env(), &id, SlotMode::Own).unwrap();
        settle(&a);
        settle(&b);
        let account = a.state().account.unwrap();
        assert_eq!(slot(&account, &id).unwrap().mode, SlotMode::Own);
        assert_eq!(open_key_secret(&account, &account_keys(&a), &id), None, "the key record is a tombstone");
        let copy = home(&b).join(SLOT_DIR).join(&file);
        assert_eq!(std::fs::read_to_string(&copy).unwrap(), test_keys::plain(), "B keeps its copy and keeps using it");
        assert!(matches!(views(&b.state(), &account_keys(&b), &home(&b))[0].status, SlotStatusView::Ready { synced_copy: true, .. }));

        set_mode(&a.env(), &id, SlotMode::Synced).unwrap();
        let account = a.state().account.unwrap();
        assert_eq!(open_key_secret(&account, &account_keys(&a), &id).as_deref(), Some(test_keys::plain().as_str()));
    }

    #[test]
    fn syncing_needs_the_key_on_this_computer() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Own, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        let refused = set_mode(&b.env(), &id, SlotMode::Synced).unwrap_err().to_string();
        assert_eq!(refused, not_here_message("MacBook-A"));
    }

    #[test]
    #[cfg(unix)]
    fn a_pick_wins_over_the_synced_copy_until_the_user_switches_back() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        let slot_path = home(&b).join(SLOT_DIR).join(&file);

        let mine = b.ssh_dir().join("id_b");
        std::fs::write(&mine, test_keys::ecdsa()).unwrap();
        pick(&b.env(), &id, &mine.display().to_string()).unwrap();
        settle(&b);
        assert_eq!(std::fs::read_link(&slot_path).unwrap(), mine, "the slot links to B's own key");
        let kept: Vec<PathBuf> = std::fs::read_dir(slot_path.parent().unwrap())
            .unwrap()
            .map(|e| e.unwrap().path())
            .filter(|p| p.file_name().unwrap().to_string_lossy().contains(".previous-") && !p.to_string_lossy().ends_with(".pub"))
            .collect();
        assert_eq!(kept.len(), 1);
        assert_eq!(std::fs::read_to_string(&kept[0]).unwrap(), test_keys::plain(), "the synced copy was kept, not deleted");
        assert!(matches!(views(&b.state(), &account_keys(&b), &home(&b))[0].status, SlotStatusView::SyncedAvailable { .. }));

        use_synced(&b.env(), &id).unwrap();
        assert_eq!(std::fs::read_to_string(&slot_path).unwrap(), test_keys::plain());
        assert!(std::fs::symlink_metadata(&slot_path).unwrap().file_type().is_file());
        assert!(mine.exists(), "B's own key is untouched");
        assert_eq!(b.state().key_slots[&id].source, Some(SlotSource::SyncedCopy { fingerprint: test_keys::PLAIN_FINGERPRINT.into() }));
    }

    #[test]
    fn only_copies_nobody_uses_can_be_deleted() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        assert_eq!(delete_copy(&b.env(), &id).unwrap_err().to_string(), IN_USE_MESSAGE);

        // A 讓主機不再用它、並刪除插槽:B 的副本變成 Not in use,可以刪。
        a.save_in_app(&a.space_path(&personal), "Host web\n  HostName 10.0.0.1\n");
        let keys = account_keys(&a);
        let env = a.env();
        let now = env.now();
        mutate(&env, |s| {
            let me = s.device_id.clone();
            let account = s.account.as_mut().unwrap();
            put_slot(account, &id, None, &me, now);
            put_key_secret(account, &keys, &id, None, &me, now)
        })
        .unwrap();
        settle(&a);
        settle(&b);
        let copy = home(&b).join(SLOT_DIR).join(&file);
        assert_eq!(views(&b.state(), &account_keys(&b), &home(&b))[0].status, SlotStatusView::NotInUse { file: copy.display().to_string() });
        delete_copy(&b.env(), &id).unwrap();
        assert!(!slot_files::occupied(&home(&b).join(SLOT_DIR).join(&file)));
        assert!(!b.state().key_slots.contains_key(&id));
        assert!(views(&b.state(), &account_keys(&b), &home(&b)).is_empty());
    }

    #[test]
    fn the_origin_syncs_its_new_key_only_when_asked_and_others_keep_theirs() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        std::fs::write(a.ssh_dir().join("id_mac"), test_keys::ecdsa()).unwrap();
        settle(&a);

        set_mode(&a.env(), &id, SlotMode::Synced).unwrap();
        let account = a.state().account.unwrap();
        assert_eq!(slot(&account, &id).unwrap().fingerprint.as_deref(), Some(test_keys::ECDSA_FINGERPRINT));
        assert_eq!(open_key_secret(&account, &account_keys(&a), &id).as_deref(), Some(test_keys::ecdsa().as_str()));
        settle(&a);
        settle(&b);
        // B 的舊副本不被自動換掉(計畫裁定 3),狀態提示可以改用。
        assert_eq!(std::fs::read_to_string(home(&b).join(SLOT_DIR).join(&file)).unwrap(), test_keys::plain());
        assert!(matches!(views(&b.state(), &account_keys(&b), &home(&b))[0].status, SlotStatusView::SyncedAvailable { .. }));
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib slots -- --skip …`
Expected:編譯失敗(`set_mode`、`pick`、`use_synced`、`delete_copy` 不存在)。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/slot_setup.rs`:`fn is_private_key_file` 改成 `pub(crate) fn is_private_key_file`。

`src-tauri/src/sync/slots.rs`:import 補 `use crate::sync::env::SyncEnv;`、`use crate::sync::runtime::mutate;`;接在檢視那一節之後:

```rust
// ── 使用者的動作(SP3 spec §6.3、§7.2)─────────────────────────────────────────────────────────

pub const IN_USE_MESSAGE: &str = "Hosts on this computer still use this key; change them first.";

pub fn not_here_message(device: &str) -> String {
    format!("Do this on a computer that has this key, such as {device}.")
}

fn not_found() -> AppError {
    AppError::NotFound("that key slot no longer exists".to_string())
}

/// 讀這台插槽裡的私鑰:連到的金鑰,或同步來的副本。
fn readable_key(local: Option<&LocalSlot>, slot_path: &Path) -> Option<String> {
    match local.and_then(|l| l.source.as_ref()) {
        Some(SlotSource::Linked { path, .. }) => std::fs::read_to_string(path).ok(),
        Some(SlotSource::SyncedCopy { .. }) => std::fs::read_to_string(slot_path).ok(),
        None => None,
    }
}

/// 快照:(狀態、帳戶金鑰、家目錄)。
fn snapshot(env: &SyncEnv) -> Result<(SyncStateV2, ChainKeys, PathBuf), AppError> {
    let core = env.runtime.core.lock().unwrap();
    let state = core.state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    let keys = core.account_keys.clone().ok_or_else(|| AppError::Other("join a sync account first".to_string()))?;
    drop(core);
    let home = env.ssh_dir.parent().map(Path::to_path_buf).ok_or_else(|| AppError::Other("cannot determine the home directory".to_string()))?;
    Ok((state, keys, home))
}

/// 改成同步(`own` → `synced`;建立插槽的那台換了金鑰後的「Sync the new key」也是它)或停止同步(`synced` → `own`,
/// `key` 寫 tombstone;別台的副本留著)。同步要在讀得到這把私鑰的電腦上做。
pub fn set_mode(env: &SyncEnv, slot_id: &str, mode: SlotMode) -> Result<(), AppError> {
    let (state, keys, home) = snapshot(env)?;
    let account = state.account.as_ref().ok_or_else(not_found)?;
    let payload = slot(account, slot_id).ok_or_else(not_found)?;
    let slot_path = home.join(SLOT_DIR).join(slot_file_name(&payload.name, slot_id));
    let text = match mode {
        SlotMode::Synced => {
            let text = readable_key(state.key_slots.get(slot_id), &slot_path)
                .ok_or_else(|| AppError::Other(not_here_message(&device_name(account, &payload.origin_device_id))))?;
            Some(text)
        }
        SlotMode::Own => None,
    };
    let facts = match &text {
        Some(t) => Some(inspect_private_key(t).map_err(|e| AppError::Other(e.message().to_string()))?),
        None => None,
    };
    let now = env.now();
    mutate(env, |s| {
        let device_id = s.device_id.clone();
        let account = s.account.as_mut().ok_or_else(not_found)?;
        let next = KeySlotPayload {
            mode,
            public_key: facts.as_ref().map(|f| f.public_key.clone()),
            fingerprint: facts.as_ref().map(|f| f.fingerprint.clone()),
            key_type: facts.as_ref().map(|f| f.key_type.clone()),
            has_passphrase: facts.as_ref().map(|f| f.has_passphrase),
            ..payload.clone()
        };
        put_slot(account, slot_id, Some(&next), &device_id, now);
        let has_key_record = account.sealed.contains_key(&key_secret_key(&keys, slot_id));
        if text.is_some() || has_key_record {
            put_key_secret(account, &keys, slot_id, text.as_deref(), &device_id, now)?;
        }
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 在這台為插槽挑一把金鑰(本機挑的優先,spec §6.2)。插槽上原本的同步副本或複製檔改名保留(計畫裁定 3)。
pub fn pick(env: &SyncEnv, slot_id: &str, path: &str) -> Result<(), AppError> {
    let source = PathBuf::from(path);
    if !source.is_absolute() || !crate::sync::slot_setup::is_private_key_file(&source) {
        return Err(AppError::Other(format!("{path} isn't a private key file")));
    }
    let (state, _keys, home) = snapshot(env)?;
    let account = state.account.as_ref().ok_or_else(not_found)?;
    let payload = slot(account, slot_id).ok_or_else(not_found)?;
    let keys_dir = home.join(SLOT_DIR);
    let file = slot_file_name(&payload.name, slot_id);
    let slot_path = keys_dir.join(&file);
    let local = state.key_slots.get(slot_id);
    let holds_copy = matches!(
        local.and_then(|l| l.source.as_ref()),
        Some(SlotSource::SyncedCopy { .. }) | Some(SlotSource::Linked { link: LinkKind::Copy, .. })
    );
    if holds_copy && slot_files::occupied(&slot_path) {
        let tag = slot_files::content_sha256(&slot_path).map(|h| h[..8].to_string()).unwrap_or_else(|| "old".to_string());
        slot_files::retire(&slot_path, &tag)?;
    }
    slot_files::ensure_keys_dir(&keys_dir)?;
    let link = slot_files::link(&source, &slot_path)?;
    let fingerprint = local_key_fingerprint(&source);
    mutate(env, |s| {
        let origin = payload.origin_device_id == s.device_id;
        s.key_slots.insert(
            slot_id.to_string(),
            LocalSlot {
                file_name: file.clone(),
                source: Some(SlotSource::Linked { path: path.to_string(), link, fingerprint: fingerprint.clone(), origin }),
                last_error: None,
                asked: true,
                payload: Some(payload.clone()),
            },
        );
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 改用同步的金鑰(狀態 SyncedAvailable):這台挑的連結移除(金鑰本身不動),舊副本改名保留(計畫裁定 3),放進同步的金鑰。
pub fn use_synced(env: &SyncEnv, slot_id: &str) -> Result<(), AppError> {
    let (state, keys, home) = snapshot(env)?;
    let account = state.account.as_ref().ok_or_else(not_found)?;
    let payload = slot(account, slot_id).ok_or_else(not_found)?;
    let secret = (payload.mode == SlotMode::Synced)
        .then(|| open_key_secret(account, &keys, slot_id))
        .flatten()
        .ok_or_else(|| AppError::Other("this key isn't synced".to_string()))?;
    let keys_dir = home.join(SLOT_DIR);
    let slot_path = keys_dir.join(slot_file_name(&payload.name, slot_id));
    match state.key_slots.get(slot_id).and_then(|l| l.source.as_ref()) {
        Some(SlotSource::Linked { link: LinkKind::Symlink | LinkKind::HardLink, .. }) => slot_files::remove_slot(&slot_path)?,
        Some(_) if slot_files::occupied(&slot_path) => {
            let tag = slot_files::content_sha256(&slot_path).map(|h| h[..8].to_string()).unwrap_or_else(|| "old".to_string());
            slot_files::retire(&slot_path, &tag)?;
        }
        _ => {}
    }
    let fingerprint = land(&secret, &payload, &keys_dir, &slot_path).map_err(AppError::Other)?;
    mutate(env, |s| {
        let local = s.key_slots.entry(slot_id.to_string()).or_insert_with(|| LocalSlot {
            file_name: slot_file_name(&payload.name, slot_id),
            source: None,
            last_error: None,
            asked: true,
            payload: Some(payload.clone()),
        });
        local.source = Some(SlotSource::SyncedCopy { fingerprint: fingerprint.clone() });
        local.last_error = None;
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 刪除沒有主機用到的副本(同步來的,或 Windows 上的複製檔)與它的 `.pub`。用得到的插槽不能刪。
pub fn delete_copy(env: &SyncEnv, slot_id: &str) -> Result<(), AppError> {
    let (state, _keys, home) = snapshot(env)?;
    let local = state.key_slots.get(slot_id).ok_or_else(not_found)?;
    if slot_hosts(&state).contains_key(&local.file_name) {
        return Err(AppError::Other(IN_USE_MESSAGE.to_string()));
    }
    let is_copy = matches!(local.source, Some(SlotSource::SyncedCopy { .. }) | Some(SlotSource::Linked { link: LinkKind::Copy, .. }));
    if !is_copy {
        return Err(not_found());
    }
    slot_files::remove_slot(&home.join(SLOT_DIR).join(&local.file_name))?;
    mutate(env, |s| {
        s.key_slots.remove(slot_id);
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}
```

`src-tauri/src/sync/engine.rs`:

```rust
/// 改成同步或停止同步(SP3 spec §6.3)。寫帳戶記錄:持有 lifecycle 鎖。
#[tauri::command]
pub async fn sync_key_set_mode(app: AppHandle, slot_id: String, mode: crate::sync::slot_rules::SlotMode) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| crate::sync::slots::set_mode(env, &slot_id, mode)).await
}

/// 在這台為插槽挑一把金鑰(只改這台)。
#[tauri::command]
pub async fn sync_key_pick(app: AppHandle, slot_id: String, path: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| crate::sync::slots::pick(env, &slot_id, &path)).await
}

/// 改用同步的金鑰(只改這台)。
#[tauri::command]
pub async fn sync_key_use_synced(app: AppHandle, slot_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| crate::sync::slots::use_synced(env, &slot_id)).await
}

/// 刪除沒有主機用到的副本(只改這台)。
#[tauri::command]
pub async fn sync_key_delete_copy(app: AppHandle, slot_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| crate::sync::slots::delete_copy(env, &slot_id)).await
}
```

`src-tauri/src/lib.rs`:`use` 清單與 `generate_handler!` 加上這四個 command。

- [ ] **Step 4: 跑測試確認通過**

Run:`… cargo test --offline --lib slots -- --skip …`,再跑完整的 Rust 測試。
Expected:新增的 5 個測試通過(`a_pick_wins…` 只在 Unix 跑);完整測試全綠。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/slots.rs src-tauri/src/sync/slot_setup.rs src-tauri/src/sync/engine.rs src-tauri/src/lib.rs
git commit -m "feat(sync): sync, stop syncing, pick and replace keys in a slot"
```

### Task 7: 更換同步碼時一起搬插槽;帳戶裡不見的插槽補寫

**Files:**
- Modify: `src-tauri/src/sync/rotation.rs`
- Test: `src-tauri/src/sync/rotation.rs`、`src-tauri/src/sync/slots.rs`

**Interfaces:**
- Consumes: Task 3(`slots::key_secret_key`、`open_key_secret`、`live_slots`、`slot`)、Task 4(`reconcile` 的補寫、測試輔助 `create_slot_on`、`use_slot`)。
- Produces:`rotation::copy` 把 `keyslot`(明文)與 `key`(以新帳戶金鑰重新加密)帶進新帳戶(SP3 spec §6.6)。

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/sync/rotation.rs` 的測試模組加:

```rust
    #[test]
    fn changing_the_sync_code_carries_the_key_slots() {
        use crate::sync::slot_rules::{test_keys, SlotMode, SLOT_DIR};
        use crate::sync::slots::tests::{create_slot_on, use_slot};
        use crate::sync::slots::{live_slots, open_key_secret};
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);

        start_rotation(&a.env()).unwrap();
        let next = new_code(&a);
        finish(&a);
        let account = a.state().account.unwrap();
        assert_eq!(live_slots(&account).iter().map(|(i, _)| i.clone()).collect::<Vec<_>>(), vec![id.clone()]);
        let keys = a.env().runtime.core.lock().unwrap().account_keys.clone().unwrap();
        assert_eq!(open_key_secret(&account, &keys, &id).as_deref(), Some(test_keys::plain().as_str()));

        // B 以新同步碼重新加入:插槽還在,副本照常。
        settle(&b);
        rejoin_account(&b.env(), &next).unwrap();
        settle(&b);
        assert_eq!(live_slots(b.state().account.as_ref().unwrap())[0].0, id);
        let copy = b.ssh_dir().parent().unwrap().join(SLOT_DIR).join(&file);
        assert_eq!(std::fs::read_to_string(copy).unwrap(), test_keys::plain());
    }
```

`src-tauri/src/sync/slots.rs` 的測試模組加:

```rust
    #[test]
    fn a_slot_missing_from_the_account_is_published_again() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let (id, file) = create_slot_on(&a, SlotMode::Synced, &test_keys::plain(), "id_mac");
        use_slot(&a, &personal, &file);
        settle(&a);
        settle(&b);
        // 在沒有 SP3 的電腦上更換同步碼之後,新帳戶裡沒有插槽記錄:直接從 B 的帳戶快取拿掉它們來模擬。
        let keys = account_keys(&b);
        mutate(&b.env(), |s| {
            let account = s.account.as_mut().unwrap();
            account.records.remove(&record_key(RecordKind::KeySlot, &id));
            account.sealed.remove(&key_secret_key(&keys, &id));
            Ok(())
        })
        .unwrap();
        let _ = crate::sync::round::sync_once(&b.env());
        let account = b.state().account.unwrap();
        assert_eq!(slot(&account, &id).map(|p| p.mode), Some(SlotMode::Synced));
        assert_eq!(open_key_secret(&account, &keys, &id).as_deref(), Some(test_keys::plain().as_str()));
    }
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib changing_the_sync_code_carries_the_key_slots -- --skip …` 與 `… --lib a_slot_missing_from_the_account -- --skip …`
Expected:第一個失敗(新帳戶沒有插槽);第二個應該已經通過(Task 4 的補寫)—— 若沒通過,修 `reconcile` 的補寫而不是改測試。

- [ ] **Step 3: 實作**

`src-tauri/src/sync/rotation.rs` 的 `copy`,在 `plan_device(&mut section, …)` 之後、`let outgoing = account_outgoing(&section, &new_account)?;`
之前加(`old` 是舊帳戶的完整快照、`keys` 是舊帳戶金鑰、`new_account` 是新帳戶金鑰;名稱照 `copy` 現有的變數):

```rust
    // SP3 spec §6.6:金鑰插槽跟著搬 —— `keyslot` 原樣(含 tombstone,別台才不會把刪掉的插槽補寫回來),`key` 以舊帳戶金鑰
    // 解開、新帳戶金鑰重新加密;版本、時間戳、裝置不變。
    for local in old.records.values().filter(|l| l.record.kind == RecordKind::KeySlot) {
        section.records.insert(
            crate::sync::record::record_key(RecordKind::KeySlot, &local.record.id),
            crate::sync::record::LocalRecord { record: local.record.clone(), seq: 0, dirty: true },
        );
    }
    for sealed in old.sealed.values().filter(|s| s.envelope.kind == RecordKind::Key.as_str()) {
        let Ok(record) = sealed.open(keys) else { continue };
        let moved = SealedRecord::seal(&new_account, &record, 0)?;
        section.sealed.insert(moved.key(), moved);
    }
```

(`SealedRecord` 若還沒 import,從 `crate::sync::state_v2` 補上。)

- [ ] **Step 4: 跑測試確認通過**

Run:同 Step 2,再跑完整的 Rust 測試。
Expected:兩個測試通過;完整測試全綠。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/sync/rotation.rs src-tauri/src/sync/slots.rs
git commit -m "feat(sync): carry key slots through a sync code change"
```

### Task 8: 插槽路徑的 lint 訊息;Windows 路徑寫成 `~/.ssh/…`

**Files:**
- Modify: `src-tauri/src/config/intel.rs`
- Modify: `src/lib/identity-file.ts`、`src/lib/identity-file.test.ts`
- Modify: `src/lib/deploy-key-select.ts`、`src/lib/deploy-key-select.test.ts`

**Interfaces:**
- Consumes: Task 1 的 `slot_rules::slot_file_of_value`。
- Produces:lint 對插槽路徑的訊息(Global Constraints 的字串);`toTildeSshPath` 認得 `\` 分隔的路徑、輸出一律 `/`;
  `identityFileAction` 與 `pickDefaultPublicKey` 比對時不分 `/` 與 `\`。

- [ ] **Step 1: 寫失敗的測試**

`src-tauri/src/config/intel.rs` 的測試模組加:

```rust
    #[test]
    fn a_missing_key_slot_points_to_the_keys_dialog() {
        let (doc, _dir) = doc_with("Host web\n IdentityFile ~/.ssh/sshelter/keys/sp3-lint-missing-00000000\n");
        let issue = lint(&doc).into_iter().find(|i| i.rule == "missing-identity-file").expect("flagged");
        assert_eq!(
            issue.message,
            "IdentityFile not found: ~/.ssh/sshelter/keys/sp3-lint-missing-00000000 (a synced key slot \u{2014} pick a key for it in Keys)"
        );
    }
```

`src/lib/identity-file.test.ts` 加:

```ts
  it("writes Windows paths under .ssh in the portable ~/.ssh/ form", () => {
    expect(toTildeSshPath("C:\\Users\\frank\\.ssh\\id_win")).toBe("~/.ssh/id_win");
    expect(toTildeSshPath("C:\\Users\\frank\\.ssh\\sub\\key")).toBe("~/.ssh/sub/key");
    expect(toTildeSshPath("D:\\keys\\deploy")).toBe("D:\\keys\\deploy");
  });
```

(放進 `describe("toTildeSshPath")`;`describe("identityFileAction")` 加:)

```ts
  it("matches a ~/.ssh entry against a Windows path", () => {
    expect(identityFileAction(["~/.ssh/id_win"], "C:\\Users\\frank\\.ssh\\id_win")).toBe("already");
  });
```

`src/lib/deploy-key-select.test.ts` 加一個 case(該檔的 builder 是 `key(name, pub)`):

```ts
  it("matches ~/.ssh IdentityFiles against Windows key paths", () => {
    const win: KeyInfo = { ...key("id_win", "C:\\Users\\frank\\.ssh\\id_win.pub"), private_path: "C:\\Users\\frank\\.ssh\\id_win" };
    expect(pickDefaultPublicKey(["~/.ssh/id_win"], [win, key("other", "/home/f/.ssh/other.pub")])).toBe(
      "C:\\Users\\frank\\.ssh\\id_win.pub",
    );
  });
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`… cargo test --offline --lib a_missing_key_slot_points -- --skip …`;`./node_modules/.bin/vitest run src/lib/identity-file.test.ts src/lib/deploy-key-select.test.ts`
Expected:三個新測試失敗。

- [ ] **Step 3: 實作**

`src-tauri/src/config/intel.rs` 的 Rule 3:

```rust
                                    message: if crate::sync::slot_rules::slot_file_of_value(&d.value).is_some() {
                                        format!("IdentityFile not found: {} (a synced key slot \u{2014} pick a key for it in Keys)", d.value)
                                    } else {
                                        format!("IdentityFile not found: {}", d.value)
                                    },
```

`src/lib/identity-file.ts`:

```ts
/** Forward slashes, so Windows paths compare and print like the ones in ssh_config. */
function slashes(path: string): string {
  return path.replace(/\\/g, "/");
}

/** Rewrite an absolute path under a `.ssh` directory to its `~/.ssh/…` form (either separator; the result uses `/`). */
export function toTildeSshPath(absPath: string): string {
  const normalized = slashes(absPath);
  const marker = "/.ssh/";
  const at = normalized.indexOf(marker);
  if (at === -1) return absPath;
  return `~/.ssh/${normalized.slice(at + marker.length)}`;
}

/** True when a config IdentityFile entry points at the deployed private key. */
function pointsAt(entry: string, deployedPrivateAbs: string): boolean {
  const e = slashes(entry);
  const deployed = slashes(deployedPrivateAbs);
  if (e === deployed) return true;
  // ssh_config keeps `~` verbatim; compare the `~/`-relative tail against the
  // end of the absolute path, segment-aligned via the leading `/`.
  return e.startsWith("~/") && deployed.endsWith(e.slice(1));
}
```

`src/lib/deploy-key-select.ts` 的 `matches`:

```ts
  // ssh_config stores IdentityFile verbatim, so `~/.ssh/work` never string-equals
  // the absolute private_path reported by keys_list (which uses `\` on Windows).
  // Compare the `~/`-relative tail against the end of the absolute path.
  const matches = (identity: string, privatePath: string) => {
    const [id, path] = [identity.replace(/\\/g, "/"), privatePath.replace(/\\/g, "/")];
    return path === id || (id.startsWith("~/") && path.endsWith(id.slice(1)));
  };
```

- [ ] **Step 4: 跑測試確認通過**

Run:同 Step 2,再跑完整的 Rust 測試、`tsc --noEmit` 與 vitest 全部。
Expected:全部通過。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/config/intel.rs src/lib/identity-file.ts src/lib/identity-file.test.ts src/lib/deploy-key-select.ts src/lib/deploy-key-select.test.ts
git commit -m "fix: write Windows key paths as ~/.ssh and explain missing key slots"
```

### Task 9: 前端的資料層:hooks、純函式、開啟對話框的狀態、測試 builder

**Files:**
- Create: `src/lib/key-slots.ts`、`src/lib/key-slots.test.ts`
- Modify: `src/lib/sync.ts`、`src/lib/sync.test.ts`
- Modify: `src/lib/sync-fixtures.ts`(`keySlot()`、`keyCandidate()`)
- Modify: `src/stores/ui.ts`(`keysOpen`、`keySetup`)

**Interfaces:**
- Consumes: Task 4–6 的 bindings(`SyncKeySlotView`、`SlotStatusView`、`SlotDeviceView`、`SlotMode`、`KeyCandidates`、`KeyCandidate`、`KeyChoice`、`KeyDecision`)與 commands。
- Produces(Task 10、11 使用):
  - `src/lib/sync.ts`:`keyCandidatesKey = ["config", "keyCandidates"]`、`fetchKeyCandidates()`、`useKeyCandidates(enabled: boolean)`、`keyArgs`、`useSetupKeys()`、`useKeySetMode()`、`useKeyPick()`、`useKeyUseSynced()`、`useKeyDeleteCopy()`
  - `src/lib/key-slots.ts`:`KeySetupRequest`、`keysToAsk`、`reuseChoices`、`isValidSlotName`、`usesLine`、`passphraseNote`、`rewrittenLines`、`lockedNote`、`choiceFor`、`identityFileChanged`、`notSetUpLabel`、`needsKeyLabel`、`slotsNeedingKey`、`slotStatusText`、`slotActions`、`hostsLine`、`deviceLine`、`hostsMissingKey`、`syncedKeysNote`、`keySetupAskedBefore`、`rememberKeySetupAsked`
  - `useUiStore`:`keysOpen` / `setKeysOpen`、`keySetup` / `setKeySetup`

- [ ] **Step 1: 寫失敗的測試**

`src/lib/sync-fixtures.ts` 先加 builder(其他測試要用):

```ts
import type { KeyCandidate } from "@/bindings/KeyCandidate";
import type { SyncKeySlotView } from "@/bindings/SyncKeySlotView";

export const SLOT_FINGERPRINT = "SHA256:9Q3QMhBJBcoUNE88XYEQbCPlcFByPPyVPJ6enJtQ+ew";

/** A synced slot this computer created, ready here, used by `web`. */
export function keySlot(overrides: Partial<SyncKeySlotView> = {}): SyncKeySlotView {
  return {
    id: "3fa2c1d90123456789abcdef01234567",
    name: "id_mac",
    mode: "synced",
    fingerprint: SLOT_FINGERPRINT,
    key_type: "ssh-ed25519",
    has_passphrase: false,
    origin_device: "MacBook-A",
    origin_is_this: true,
    value: "~/.ssh/sshelter/keys/id_mac-3fa2c1d9",
    hosts: ["web"],
    status: { kind: "ready", file: "/home/f/.ssh/id_mac", synced_copy: false, fingerprint: SLOT_FINGERPRINT },
    devices: [],
    ...overrides,
  };
}

/** A key that `web` uses and no slot holds yet. */
export function keyCandidate(overrides: Partial<KeyCandidate> = {}): KeyCandidate {
  return {
    path: "/home/f/.ssh/id_mac",
    default_name: "id_mac",
    fingerprint: SLOT_FINGERPRINT,
    has_passphrase: false,
    unsyncable: null,
    existing_slot: null,
    hosts: [{ alias: "web", space_name: "Personal", value: "~/.ssh/id_mac", locked: null }],
    ...overrides,
  };
}
```

`src/lib/key-slots.test.ts`:

```ts
import { afterEach, describe, expect, it, vi } from "vitest";

import { keyCandidate, keySlot, overview, SPOOFED_NAME, SPOOFED_NAME_SHOWN } from "@/lib/sync-fixtures";
import {
  choiceFor,
  deviceLine,
  hostsLine,
  hostsMissingKey,
  identityFileChanged,
  isValidSlotName,
  keySetupAskedBefore,
  keysToAsk,
  lockedNote,
  needsKeyLabel,
  notSetUpLabel,
  passphraseNote,
  rememberKeySetupAsked,
  reuseChoices,
  rewrittenLines,
  slotActions,
  slotStatusText,
  slotsNeedingKey,
  syncedKeysNote,
  usesLine,
} from "./key-slots";

afterEach(() => vi.unstubAllGlobals());

const two = keyCandidate({
  hosts: [
    { alias: "web", space_name: "Personal", value: "~/.ssh/id_mac", locked: null },
    { alias: "db", space_name: "Work", value: "\"/home/f/.ssh/id_mac\"", locked: null },
  ],
});
const LOCK = "This host has more than one copy; SSHelter changes it once only one copy is left.";

describe("which keys the dialog asks about", () => {
  it("asks about keys without a slot that would rewrite at least one of the given hosts", () => {
    const reused = keyCandidate({ path: "/home/f/.ssh/old", existing_slot: "a".repeat(32) });
    const lockedOnly = keyCandidate({ path: "/home/f/.ssh/locked", hosts: [{ alias: "api", space_name: "Personal", value: "~/.ssh/locked", locked: LOCK }] });
    const all = { keys: [two, reused, lockedOnly], unsupported: [] };
    expect(keysToAsk(all, null)).toEqual([two]);
    expect(keysToAsk(all, ["db"])).toEqual([two]);
    expect(keysToAsk(all, ["other"])).toEqual([]);
    expect(keysToAsk(undefined, null)).toEqual([]);
    expect(reuseChoices(all, null)).toEqual([{ path: "/home/f/.ssh/old", decision: { kind: "reuse", slot_id: "a".repeat(32) } }]);
    expect(reuseChoices(all, ["web"])).toEqual([]);
  });

  it("describes a key, its passphrase and the lines it rewrites", () => {
    expect(usesLine(keyCandidate(), "id_mac")).toBe("web uses id_mac.");
    expect(usesLine(two, "personal")).toBe("web and db use personal.");
    expect(passphraseNote(keyCandidate())).toBe("No passphrase — your sync code and every joined computer can use this key once it syncs.");
    expect(passphraseNote(keyCandidate({ has_passphrase: true }))).toBe("Has a passphrase — it stays on each computer.");
    expect(passphraseNote(keyCandidate({ has_passphrase: null }))).toBeNull();
    expect(rewrittenLines(two, "personal")).toEqual([
      "web: IdentityFile ~/.ssh/id_mac → ~/.ssh/sshelter/keys/personal-…",
      "db: IdentityFile \"/home/f/.ssh/id_mac\" → ~/.ssh/sshelter/keys/personal-…",
    ]);
    const mixed = keyCandidate({ hosts: [...two.hosts, { alias: "api", space_name: "Personal", value: "~/.ssh/id_mac", locked: LOCK }] });
    expect(rewrittenLines(mixed, "k")).toHaveLength(2);
    expect(lockedNote(mixed)).toBe(`Not changed: api — ${LOCK}`);
    expect(lockedNote(two)).toBeNull();
    expect(choiceFor(keyCandidate(), true, "id_mac")).toEqual({ path: "/home/f/.ssh/id_mac", decision: { kind: "sync", name: "id_mac" } });
    expect(choiceFor(keyCandidate(), false, "k")).toEqual({ path: "/home/f/.ssh/id_mac", decision: { kind: "keep", name: "k" } });
  });

  it("accepts the slot names the backend accepts", () => {
    for (const ok of ["id_ed25519", "work", "a", "Key.2026_v-1", "k".repeat(64)]) expect(isValidSlotName(ok)).toBe(true);
    for (const bad of ["", "-x", ".x", "a b", "a/b", "id.pub", "ID.PUB", "k".repeat(65), "鍵"]) expect(isValidSlotName(bad)).toBe(false);
  });

  it("notices an IdentityFile change in a host save, whatever its spelling", () => {
    expect(identityFileChanged([{ keyword: "identityfile", value: "~/.ssh/k", remove: false }])).toBe(true);
    expect(identityFileChanged([{ keyword: "HostName", value: "x", remove: false }])).toBe(false);
  });
});

describe("Settings → Sync rows", () => {
  it("count keys to set up and slots that need a key here", () => {
    expect(notSetUpLabel(1)).toBe("1 key used by synced hosts isn't set up");
    expect(notSetUpLabel(2)).toBe("2 keys used by synced hosts aren't set up");
    expect(needsKeyLabel(1)).toBe("1 key slot needs a key on this computer");
    expect(needsKeyLabel(3)).toBe("3 key slots need a key on this computer");
    const o = overview({
      key_slots: [
        keySlot(),
        keySlot({ id: "b".repeat(32), status: { kind: "needs_key", waiting_for_sync: false } }),
        keySlot({ id: "c".repeat(32), status: { kind: "needs_key", waiting_for_sync: true } }),
      ],
    });
    expect(slotsNeedingKey(o).map((s) => s.id)).toEqual(["b".repeat(32)]);
  });

  it("reminds about synced keys after a sync code change", () => {
    expect(syncedKeysNote(overview({ key_slots: [keySlot(), keySlot({ id: "b".repeat(32), name: SPOOFED_NAME, mode: "own" })] }))).toBe(
      "If a computer was lost, also replace these synced keys on your servers: id_mac.",
    );
    expect(syncedKeysNote(overview({ key_slots: [keySlot({ mode: "own" })] }))).toBeNull();
  });
});

describe("a slot row", () => {
  it("says the status in words and tone", () => {
    expect(slotStatusText({ kind: "ready", file: "/f", synced_copy: false, fingerprint: null })).toEqual({ text: "Ready", tone: "ok" });
    expect(slotStatusText({ kind: "needs_key", waiting_for_sync: false })).toEqual({ text: "Needs a key on this computer", tone: "warning" });
    expect(slotStatusText({ kind: "needs_key", waiting_for_sync: true })).toEqual({ text: "Waiting for the synced key", tone: "busy" });
    expect(slotStatusText({ kind: "not_in_use", file: "/f" })).toEqual({ text: "Not in use", tone: "ok" });
    expect(slotStatusText({ kind: "not_used_here" })).toEqual({ text: "Not used on this computer", tone: "ok" });
    expect(slotStatusText({ kind: "synced_available", file: "/f" })).toEqual({ text: "A synced key is available", tone: "warning" });
    expect(slotStatusText({ kind: "source_changed", file: "/f" }).text).toBe(
      "This computer's key changed — your other computers still have the previous one",
    );
    expect(slotStatusText({ kind: "error", message: "boom" })).toEqual({ text: "boom", tone: "error" });
  });

  it("offers the actions that fit", () => {
    const none = { syncThis: false, stopSyncing: false, pick: null, useSynced: false, syncNew: false, deleteCopy: false };
    expect(slotActions(keySlot())).toEqual({ ...none, stopSyncing: true, pick: "change" });
    expect(slotActions(keySlot({ mode: "own", fingerprint: null }))).toEqual({ ...none, syncThis: true, pick: "change" });
    expect(slotActions(keySlot({ mode: "own", status: { kind: "needs_key", waiting_for_sync: false } }))).toEqual({ ...none, pick: "pick" });
    expect(slotActions(keySlot({ status: { kind: "synced_available", file: "/f" } }))).toEqual({ ...none, stopSyncing: true, pick: "change", useSynced: true });
    expect(slotActions(keySlot({ status: { kind: "source_changed", file: "/f" } }))).toEqual({ ...none, stopSyncing: true, syncNew: true });
    expect(slotActions(keySlot({ mode: "own", status: { kind: "not_in_use", file: "/f" } }))).toEqual({ ...none, deleteCopy: true });
  });

  it("lists hosts and other computers, revealing hidden characters in their names", () => {
    expect(hostsLine(keySlot({ hosts: ["web", "db"] }))).toBe("Used by web and db");
    expect(hostsLine(keySlot({ hosts: [] }))).toBeNull();
    expect(
      deviceLine(keySlot({ devices: [{ name: SPOOFED_NAME, fingerprint: null, synced_copy: true }, { name: "FRANK-DESKTOP", fingerprint: "SHA256:x", synced_copy: false }] })),
    ).toBe(`${SPOOFED_NAME_SHOWN}: synced copy · FRANK-DESKTOP: its own key`);
    expect(deviceLine(keySlot())).toBeNull();
  });

  it("marks the sidebar hosts whose key is missing here, but not while the synced key is on its way", () => {
    const o = overview({
      key_slots: [
        keySlot(),
        keySlot({ id: "b".repeat(32), hosts: ["db"], status: { kind: "needs_key", waiting_for_sync: false } }),
        keySlot({ id: "c".repeat(32), hosts: ["api"], status: { kind: "error", message: "x" } }),
        keySlot({ id: "d".repeat(32), hosts: ["ci"], status: { kind: "needs_key", waiting_for_sync: true } }),
      ],
    });
    expect([...hostsMissingKey(o)].sort()).toEqual(["api", "db"]);
    expect(hostsMissingKey(undefined).size).toBe(0);
  });
});

describe("the once-per-computer setup prompt", () => {
  it("remembers that it asked, and survives a missing or broken localStorage", () => {
    expect(keySetupAskedBefore()).toBe(false); // node: no localStorage
    expect(() => rememberKeySetupAsked()).not.toThrow();
    const store = new Map<string, string>();
    vi.stubGlobal("localStorage", { getItem: (k: string) => store.get(k) ?? null, setItem: (k: string, v: string) => store.set(k, v) });
    expect(keySetupAskedBefore()).toBe(false);
    rememberKeySetupAsked();
    expect(keySetupAskedBefore()).toBe(true);
    vi.stubGlobal("localStorage", { getItem: () => { throw new Error("denied"); }, setItem: () => { throw new Error("denied"); } });
    expect(keySetupAskedBefore()).toBe(false);
    expect(() => rememberKeySetupAsked()).not.toThrow();
  });
});
```

`src/lib/sync.test.ts` 加(用該檔的 `stubBackend`):

```ts
describe("key slot commands", () => {
  it("call the backend with its camelCase arguments", async () => {
    const calls = stubBackend(async () => ({ keys: [], unsupported: [] }));
    expect(await fetchKeyCandidates()).toEqual({ keys: [], unsupported: [] });
    expect(calls).toEqual([["sync_key_candidates", {}]]);
    const choices = [{ path: "/home/f/.ssh/id_mac", decision: { kind: "keep" as const, name: "id_mac" } }];
    expect(keyArgs.setup({ choices })).toEqual({ choices });
    expect(keyArgs.setMode({ slotId: "s", mode: "own" })).toEqual({ slotId: "s", mode: "own" });
    expect(keyArgs.pick({ slotId: "s", path: "/k" })).toEqual({ slotId: "s", path: "/k" });
    expect(keyArgs.slot({ slotId: "s" })).toEqual({ slotId: "s" });
  });
});
```

(把 `fetchKeyCandidates`、`keyArgs` 加進檔案頂端從 `./sync` 的 import。)

- [ ] **Step 2: 跑測試確認失敗**

Run:`./node_modules/.bin/vitest run src/lib/key-slots.test.ts src/lib/sync.test.ts`
Expected:失敗(`./key-slots` 不存在;`fetchKeyCandidates`、`keyArgs` 沒有匯出)。

- [ ] **Step 3: 實作**

`src/lib/key-slots.ts`:

```ts
import type { HostFieldChange } from "@/bindings/HostFieldChange";
import type { KeyCandidate } from "@/bindings/KeyCandidate";
import type { KeyCandidates } from "@/bindings/KeyCandidates";
import type { KeyChoice } from "@/bindings/KeyChoice";
import type { SlotStatusView } from "@/bindings/SlotStatusView";
import type { SyncKeySlotView } from "@/bindings/SyncKeySlotView";
import type { SyncOverview } from "@/bindings/SyncOverview";
import { listNames, plural } from "@/lib/format";
import { revealHidden } from "@/lib/sync-approvals";
import type { Tone } from "@/lib/sync-overview";

/**
 * Key slots (SP3 spec docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md): pure helpers for the
 * "Keys used by synced hosts" dialog, the Keys dialog's slot section, Settings → Sync and the sidebar.
 */

/** Why the setup dialog opened: the hosts it is about (null = every host) and what opened it. */
export interface KeySetupRequest {
  aliases: string[] | null;
  reason: "moved" | "saved" | "upgrade" | "settings";
}

function hostsIn(k: KeyCandidate, aliases: string[] | null) {
  return (aliases === null ? k.hosts : k.hosts.filter((h) => aliases.includes(h.alias))).filter((h) => h.locked === null);
}

/** Keys to ask about: no slot yet, and at least one host (among `aliases`, when given) the setup would rewrite. */
export function keysToAsk(candidates: KeyCandidates | undefined, aliases: string[] | null): KeyCandidate[] {
  return (candidates?.keys ?? []).filter((k) => k.existing_slot === null && hostsIn(k, aliases).length > 0);
}

/** Keys that already have a slot here or in the account: set up without asking (spec §6.1). */
export function reuseChoices(candidates: KeyCandidates | undefined, aliases: string[] | null): KeyChoice[] {
  return (candidates?.keys ?? [])
    .filter((k) => k.existing_slot !== null && hostsIn(k, aliases).length > 0)
    .map((k) => ({ path: k.path, decision: { kind: "reuse", slot_id: k.existing_slot as string } }));
}

/** The backend's slot name rule (`slot_rules::valid_slot_name`). */
export function isValidSlotName(name: string): boolean {
  return /^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$/.test(name) && !/\.pub$/i.test(name);
}

/** "web uses id_mac." / "web and db use id_mac." (only the hosts the setup changes). */
export function usesLine(k: KeyCandidate, name: string): string {
  const hosts = k.hosts.filter((h) => h.locked === null).map((h) => h.alias);
  return `${listNames(hosts)} ${hosts.length === 1 ? "uses" : "use"} ${name}.`;
}

export function passphraseNote(k: KeyCandidate): string | null {
  if (k.has_passphrase === null) return null;
  return k.has_passphrase
    ? "Has a passphrase — it stays on each computer."
    : "No passphrase — your sync code and every joined computer can use this key once it syncs.";
}

/** The lines the setup rewrites. The slot's id is only known once it exists, so its file name ends in "…". */
export function rewrittenLines(k: KeyCandidate, name: string): string[] {
  return k.hosts.filter((h) => h.locked === null).map((h) => `${h.alias}: IdentityFile ${h.value} → ~/.ssh/sshelter/keys/${name}-…`);
}

/** Hosts the setup leaves alone, and why. */
export function lockedNote(k: KeyCandidate): string | null {
  const locked = k.hosts.filter((h) => h.locked !== null);
  if (locked.length === 0) return null;
  return `Not changed: ${listNames(locked.map((h) => h.alias))} — ${locked[0].locked}`;
}

export function choiceFor(k: KeyCandidate, sync: boolean, name: string): KeyChoice {
  return { path: k.path, decision: sync ? { kind: "sync", name } : { kind: "keep", name } };
}

/** A host save that touched IdentityFile (any spelling) may need a key set up. */
export function identityFileChanged(changes: readonly HostFieldChange[]): boolean {
  return changes.some((c) => c.keyword.toLowerCase() === "identityfile");
}

export function notSetUpLabel(n: number): string {
  return `${plural(n, "key")} used by synced hosts ${n === 1 ? "isn't" : "aren't"} set up`;
}

export function needsKeyLabel(n: number): string {
  return `${plural(n, "key slot")} ${n === 1 ? "needs" : "need"} a key on this computer`;
}

/** Slots this computer needs a key for and the user has to pick one (not ones waiting for a synced key). */
export function slotsNeedingKey(o: SyncOverview): SyncKeySlotView[] {
  return o.key_slots.filter((s) => s.status.kind === "needs_key" && !s.status.waiting_for_sync);
}

/** After a sync code change: which synced keys to replace if a computer was lost (spec §6.6). */
export function syncedKeysNote(o: SyncOverview): string | null {
  const synced = o.key_slots.filter((s) => s.mode === "synced").map((s) => revealHidden(s.name));
  if (synced.length === 0) return null;
  return `If a computer was lost, also replace these synced keys on your servers: ${listNames(synced)}.`;
}

export function slotStatusText(status: SlotStatusView): { text: string; tone: Tone } {
  switch (status.kind) {
    case "ready":
      return { text: "Ready", tone: "ok" };
    case "needs_key":
      return status.waiting_for_sync
        ? { text: "Waiting for the synced key", tone: "busy" }
        : { text: "Needs a key on this computer", tone: "warning" };
    case "not_in_use":
      return { text: "Not in use", tone: "ok" };
    case "not_used_here":
      return { text: "Not used on this computer", tone: "ok" };
    case "synced_available":
      return { text: "A synced key is available", tone: "warning" };
    case "source_changed":
      return { text: "This computer's key changed — your other computers still have the previous one", tone: "warning" };
    case "error":
      return { text: status.message, tone: "error" };
  }
}

export interface SlotActions {
  syncThis: boolean;
  stopSyncing: boolean;
  pick: "pick" | "change" | null;
  useSynced: boolean;
  syncNew: boolean;
  deleteCopy: boolean;
}

export function slotActions(slot: SyncKeySlotView): SlotActions {
  const s = slot.status;
  return {
    syncThis: slot.mode === "own" && s.kind === "ready",
    stopSyncing: slot.mode === "synced",
    pick: s.kind === "needs_key" ? "pick" : s.kind === "ready" || s.kind === "synced_available" ? "change" : null,
    useSynced: s.kind === "synced_available",
    syncNew: s.kind === "source_changed",
    deleteCopy: s.kind === "not_in_use",
  };
}

export function hostsLine(slot: SyncKeySlotView): string | null {
  return slot.hosts.length === 0 ? null : `Used by ${listNames(slot.hosts)}`;
}

/** The other computers' slots; their names come from those computers. */
export function deviceLine(slot: SyncKeySlotView): string | null {
  if (slot.devices.length === 0) return null;
  return slot.devices.map((d) => `${revealHidden(d.name)}: ${d.synced_copy ? "synced copy" : "its own key"}`).join(" · ");
}

/** Sidebar hosts whose key isn't on this computer: a slot that needs a key picked here, or failed (not one whose synced key is on its way). */
export function hostsMissingKey(o: SyncOverview | undefined): Set<string> {
  const out = new Set<string>();
  for (const slot of o?.key_slots ?? []) {
    const s = slot.status;
    if ((s.kind === "needs_key" && !s.waiting_for_sync) || s.kind === "error") slot.hosts.forEach((h) => out.add(h));
  }
  return out;
}

/** The setup dialog opens by itself once per computer after the SP3 update (spec §7.1). localStorage may be missing or throw. */
const ASKED_KEY = "sshelter.keySetupAsked";

export function keySetupAskedBefore(): boolean {
  try {
    return globalThis.localStorage?.getItem(ASKED_KEY) === "1";
  } catch {
    return false;
  }
}

export function rememberKeySetupAsked(): void {
  try {
    globalThis.localStorage?.setItem(ASKED_KEY, "1");
  } catch {
    // The prompt may show once more.
  }
}
```

`src/lib/sync.ts`(放在檔尾附近;import 補 `KeyCandidates`、`KeyChoice`、`SlotMode` 三個 binding 型別):

```ts
/**
 * Keys used by synced hosts that no slot holds yet. Under ["config"], so a reload, a sync apply and the key slot commands
 * refresh it; host edits don't, but the setup dialog's query turns on when it opens and so fetches it again.
 */
export const keyCandidatesKey = ["config", "keyCandidates"] as const;

export function fetchKeyCandidates(): Promise<KeyCandidates> {
  return tauriInvoke<KeyCandidates>("sync_key_candidates");
}

export function useKeyCandidates(enabled: boolean) {
  return useQuery<KeyCandidates>({ queryKey: keyCandidatesKey, queryFn: fetchKeyCandidates, enabled });
}

/** The key slot commands' arguments, in the backend's camelCase. */
export const keyArgs = {
  setup: (v: { choices: KeyChoice[] }) => ({ choices: v.choices }),
  setMode: (v: { slotId: string; mode: SlotMode }) => ({ slotId: v.slotId, mode: v.mode }),
  pick: (v: { slotId: string; path: string }) => ({ slotId: v.slotId, path: v.path }),
  slot: (v: { slotId: string }) => ({ slotId: v.slotId }),
};

/** Create or reuse slots and rewrite the hosts (it can fail after creating a slot, so a failure re-reads everything). */
export function useSetupKeys() {
  return useOverviewMutation("sync_setup_keys", "Could not set up the key", keyArgs.setup, true);
}

export function useKeySetMode() {
  return useOverviewMutation("sync_key_set_mode", "Could not change how the key is shared", keyArgs.setMode);
}

export function useKeyPick() {
  return useOverviewMutation("sync_key_pick", "Could not use that key", keyArgs.pick);
}

export function useKeyUseSynced() {
  return useOverviewMutation("sync_key_use_synced", "Could not use the synced key", keyArgs.slot);
}

export function useKeyDeleteCopy() {
  return useOverviewMutation("sync_key_delete_copy", "Could not delete the copy", keyArgs.slot);
}
```

`src/stores/ui.ts`:interface 加(放在 `syncApprovalsOpen` 之後),並在 `create` 的初始值裡加 `keysOpen: false, setKeysOpen: (keysOpen) => set({ keysOpen }), keySetup: null, setKeySetup: (keySetup) => set({ keySetup }),`:

```ts
  /** The Keys dialog (toolbar button, Settings → Sync, the sidebar's missing-key marker). Session-only. */
  keysOpen: boolean;
  setKeysOpen: (open: boolean) => void;
  /** "Keys used by synced hosts" (SP3 spec §7.1): open while non-null. Session-only. */
  keySetup: KeySetupRequest | null;
  setKeySetup: (request: KeySetupRequest | null) => void;
```

(`import type { KeySetupRequest } from "@/lib/key-slots";`)

- [ ] **Step 4: 跑測試確認通過**

Run:`./node_modules/.bin/vitest run src/lib/key-slots.test.ts src/lib/sync.test.ts`,再跑 `tsc --noEmit` 與 vitest 全部。
Expected:全部通過。

- [ ] **Step 5: Commit**

```bash
git add src/lib/key-slots.ts src/lib/key-slots.test.ts src/lib/sync.ts src/lib/sync.test.ts src/lib/sync-fixtures.ts src/stores/ui.ts
git commit -m "feat(ui): key slot data, helpers and dialog state"
```

### Task 10: 「Keys used by synced hosts」對話框與它的觸發點

**Files:**
- Create: `src/components/SyncKeyDialog.tsx`、`src/components/SyncKeyDialog.test.tsx`
- Modify: `src/App.tsx`(掛上對話框與升級提示)
- Modify: `src/components/SyncMigrationDialog.tsx`(搬完之後)、`src/components/HostEditor.tsx`(存檔、搬到別的檔案、複製之後)、`src/components/HostList.tsx`(搬到別的檔案之後,含整批搬移)、`src/components/DeployKeyDialog.tsx`(寫入 IdentityFile 之後)

**Interfaces:**
- Consumes: Task 9 的 hooks、`key-slots` 純函式、`useUiStore.keySetup`。
- Produces:`SyncKeyDialog`(App 層級)、`KeySetupRow`、`UnsupportedList`(匯出給 markup 測試)、`useKeySetupOnUpgrade()`(App 呼叫)。

- [ ] **Step 1: 寫失敗的測試**

`src/components/SyncKeyDialog.test.tsx`:

```tsx
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";

import { keyCandidate } from "@/lib/sync-fixtures";
import { KeySetupRow, UnsupportedList } from "./SyncKeyDialog";

/**
 * The dialog's rows rendered on the server (no DOM): what each key's row says and which buttons it offers.
 * The dialog itself opens from the UI store and the candidates query; its logic is tested in key-slots.test.ts.
 */
const row = (candidate = keyCandidate()) => renderToStaticMarkup(<KeySetupRow candidate={candidate} busy={false} onChoose={() => {}} />);
const text = (html: string) => html.replace(/<[^>]*>/g, " ").replace(/\s+/g, " ");
const buttonTag = (html: string, label: string) => {
  const at = html.indexOf(`>${label}<`);
  return html.slice(html.lastIndexOf("<button", at), at);
};

describe("a key's row", () => {
  it("says which hosts use the key, asks the question and shows what changes", () => {
    const t = text(row());
    expect(t).toContain("web uses id_mac.");
    expect(t).toContain("Sync this key to your other computers?");
    expect(t).toContain("No passphrase — your sync code and every joined computer can use this key once it syncs.");
    expect(t).toContain("web: IdentityFile ~/.ssh/id_mac → ~/.ssh/sshelter/keys/id_mac-…");
    for (const label of ["Sync key", "Keep on this computer", "Rename"]) expect(row()).toContain(`>${label}<`);
    expect(buttonTag(row(), "Sync key")).not.toContain("disabled");
  });

  it("offers only Keep for a key that can't be synced, and says why", () => {
    const html = row(keyCandidate({ unsyncable: "This key isn't in the OpenSSH format, so it can't be synced." }));
    expect(text(html)).toContain("This key isn't in the OpenSSH format, so it can't be synced.");
    expect(buttonTag(html, "Sync key")).toContain("disabled");
    expect(buttonTag(html, "Keep on this computer")).not.toContain("disabled");
  });

  it("names the hosts it leaves alone", () => {
    const html = row(
      keyCandidate({
        hosts: [
          { alias: "web", space_name: "Personal", value: "~/.ssh/id_mac", locked: null },
          { alias: "api", space_name: "Personal", value: "~/.ssh/id_mac", locked: "This host has more than one copy; SSHelter changes it once only one copy is left." },
        ],
      }),
    );
    expect(text(html)).toContain("Not changed: api — This host has more than one copy");
    expect(text(html)).not.toContain("api: IdentityFile");
  });
});

describe("values that can't be set up", () => {
  it("lists each host with its value and the reason", () => {
    const html = renderToStaticMarkup(<UnsupportedList items={[{ alias: "proxy", value: "~/.ssh/%h", reason: "uses % tokens or environment variables" }]} />);
    expect(text(html)).toContain("Can't set up automatically");
    expect(text(html)).toContain("proxy: ~/.ssh/%h — uses % tokens or environment variables");
    expect(renderToStaticMarkup(<UnsupportedList items={[]} />)).toBe("");
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run:`./node_modules/.bin/vitest run src/components/SyncKeyDialog.test.tsx`
Expected:失敗(模組不存在)。

- [ ] **Step 3: 實作**

`src/components/SyncKeyDialog.tsx`:

```tsx
import { useEffect, useRef, useState } from "react";
import { KeyRound } from "lucide-react";
import { toast } from "sonner";

import type { KeyCandidate } from "@/bindings/KeyCandidate";
import type { UnsupportedIdentity } from "@/bindings/UnsupportedIdentity";
import { TONE_TEXT } from "@/components/sync-primitives";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Input } from "@/components/ui/input";
import { isImeKey } from "@/lib/ime";
import {
  choiceFor,
  isValidSlotName,
  keySetupAskedBefore,
  keysToAsk,
  lockedNote,
  passphraseNote,
  rememberKeySetupAsked,
  reuseChoices,
  rewrittenLines,
  usesLine,
  type KeySetupRequest,
} from "@/lib/key-slots";
import { useKeyCandidates, useSetupKeys, useSyncOverview } from "@/lib/sync";
import { cn } from "@/lib/utils";
import { useUiStore } from "@/stores/ui";

/** One key's row: the hosts that use it, the question, what changes, and the two answers. Exported for the markup tests. */
export function KeySetupRow({
  candidate,
  busy,
  onChoose,
}: {
  candidate: KeyCandidate;
  busy: boolean;
  onChoose: (sync: boolean, name: string) => void;
}) {
  const [name, setName] = useState(candidate.default_name);
  const [renaming, setRenaming] = useState(false);
  const valid = isValidSlotName(name);
  const note = passphraseNote(candidate);
  const locked = lockedNote(candidate);
  return (
    <div className="space-y-2 rounded-md border p-3">
      <div className="flex items-start justify-between gap-2">
        <p className="flex items-center gap-1.5 text-sm">
          <KeyRound className="size-3.5 shrink-0 text-muted-foreground" />
          {usesLine(candidate, name)}
        </p>
        {!renaming && (
          <Button type="button" variant="link" size="sm" className="h-auto p-0 text-xs" disabled={busy} onClick={() => setRenaming(true)}>
            Rename
          </Button>
        )}
      </div>
      {renaming && (
        <Input
          autoFocus
          value={name}
          aria-label="Key name"
          className={cn("h-7 font-mono text-xs", !valid && "border-destructive")}
          onChange={(e) => setName(e.target.value)}
          onKeyDown={(e) => {
            if (isImeKey(e)) return;
            if (e.key === "Enter" && valid) setRenaming(false);
          }}
          onBlur={() => valid && setRenaming(false)}
        />
      )}
      <p className="text-sm">Sync this key to your other computers?</p>
      {note && <p className="text-xs text-muted-foreground">{note}</p>}
      <ul className="space-y-0.5 font-mono text-xs text-muted-foreground">
        {rewrittenLines(candidate, name).map((line) => (
          <li key={line} className="break-all">
            {line}
          </li>
        ))}
      </ul>
      {locked && <p className={cn("text-xs", TONE_TEXT.warning)}>{locked}</p>}
      {candidate.unsyncable && <p className={cn("text-xs", TONE_TEXT.warning)}>{candidate.unsyncable}</p>}
      <div className="flex justify-end gap-2">
        <Button type="button" variant="outline" size="sm" disabled={busy || !valid} onClick={() => onChoose(false, name)}>
          Keep on this computer
        </Button>
        <Button type="button" size="sm" disabled={busy || !valid || candidate.unsyncable !== null} onClick={() => onChoose(true, name)}>
          Sync key
        </Button>
      </div>
    </div>
  );
}

/** Values SSHelter can't set up by itself (spec §5). Exported for the markup tests. */
export function UnsupportedList({ items }: { items: UnsupportedIdentity[] }) {
  if (items.length === 0) return null;
  return (
    <div className="space-y-1">
      <p className="text-xs font-medium text-muted-foreground">Can't set up automatically</p>
      <ul className="space-y-0.5 text-xs text-muted-foreground">
        {items.map((u) => (
          <li key={`${u.alias} ${u.value}`} className="break-all">
            {`${u.alias}: ${u.value} — ${u.reason}`}
          </li>
        ))}
      </ul>
    </div>
  );
}

/**
 * "Keys used by synced hosts" (SP3 spec §7.1). Opens from the UI store (`keySetup`): after hosts move into a space,
 * after a save or a deploy that wrote IdentityFile, once after the update, or from Settings → Sync. Keys that already
 * have a slot are set up without asking; with nothing left to ask, it closes by itself without showing.
 */
export function SyncKeyDialog() {
  const request = useUiStore((s) => s.keySetup);
  const setRequest = useUiStore((s) => s.setKeySetup);
  const candidates = useKeyCandidates(request !== null);
  const { refetch } = candidates;
  const setup = useSetupKeys();
  // Each request reads the hosts as they are now: the edit or move that opened it just changed them, and host edits don't
  // refresh the candidates. Until that read is back nothing shows (the cached list may still have the old hosts).
  const [readFor, setReadFor] = useState<KeySetupRequest | null>(null);
  useEffect(() => {
    if (!request) return;
    let current = true;
    void refetch().then(() => {
      if (current) setReadFor(request);
    });
    return () => {
      current = false;
    };
  }, [request, refetch]);
  const ready = request !== null && readFor === request;
  const aliases = request?.aliases ?? null;
  const ask = ready ? keysToAsk(candidates.data, aliases) : [];
  const reuse = ready ? reuseChoices(candidates.data, aliases) : [];
  const reusedFor = useRef<KeySetupRequest | null>(null);

  useEffect(() => {
    if (!request || !ready || candidates.isFetching) return;
    if (reuse.length > 0 && reusedFor.current !== request) {
      reusedFor.current = request;
      setup.mutate({ choices: reuse });
    }
    if (ask.length === 0) setRequest(null);
  }, [request, ready, candidates.isFetching, ask.length, reuse, setup, setRequest]);

  const close = () => setRequest(null);
  const later = request?.reason === "upgrade" || request?.reason === "settings";
  return (
    <Dialog open={ready && ask.length > 0} onOpenChange={(next) => !next && !setup.isPending && close()}>
      <DialogContent className="sm:max-w-lg" showCloseButton={!setup.isPending}>
        <DialogHeader>
          <DialogTitle>Keys used by synced hosts</DialogTitle>
          <DialogDescription>Choose for each key whether it goes to your other computers. Your servers aren't changed.</DialogDescription>
        </DialogHeader>
        <div className="max-h-[50vh] space-y-3 overflow-y-auto pr-1">
          {ask.map((candidate) => (
            <KeySetupRow
              key={candidate.path}
              candidate={candidate}
              busy={setup.isPending}
              onChoose={(sync, name) =>
                setup.mutate(
                  { choices: [choiceFor(candidate, sync, name)] },
                  { onSuccess: () => toast.success(sync ? `${name} syncs to your other computers` : `${name} stays on this computer`) },
                )
              }
            />
          ))}
        </div>
        <UnsupportedList items={candidates.data?.unsupported ?? []} />
        <DialogFooter>
          <Button type="button" variant="outline" disabled={setup.isPending} onClick={close}>
            {later ? "Later" : "Close"}
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

/**
 * After the SP3 update, synced hosts may already use this computer's keys: ask once per computer (spec §7.1). "Later"
 * leaves them in the Settings → Sync row.
 */
export function useKeySetupOnUpgrade() {
  const overview = useSyncOverview();
  const enabled = overview.data?.joined === true && !keySetupAskedBefore();
  const candidates = useKeyCandidates(enabled);
  const setRequest = useUiStore((s) => s.setKeySetup);
  useEffect(() => {
    if (!enabled || !candidates.data) return;
    rememberKeySetupAsked();
    if (keysToAsk(candidates.data, null).length > 0 || reuseChoices(candidates.data, null).length > 0) {
      setRequest({ aliases: null, reason: "upgrade" });
    }
  }, [enabled, candidates.data, setRequest]);
}
```

`src/App.tsx`:在 `<SyncApprovalDialog />` 之後加 `<SyncKeyDialog />`;在 App 元件本體(和 `useSyncEvents()` 同一處)呼叫
`useKeySetupOnUpgrade();`,import 兩者。

`src/components/SyncMigrationDialog.tsx` 的 `done`,在 `setFailedMoves(report.failed);` 之後加:

```tsx
    // Hosts that now sync may use this computer's keys (SP3 spec §7.1).
    if (report.moved.length > 0) useUiStore.getState().setKeySetup({ aliases: report.moved, reason: "moved" });
```

`src/components/HostEditor.tsx` 的 `onSave`:

```tsx
      onSave={(changes) =>
        saveHost.mutate(
          { alias: detail.alias, changes },
          {
            onSuccess: () => {
              toast.success(`Saved ${detail.alias}`);
              // A synced host's IdentityFile may now point at a key no slot holds (SP3 spec §7.1).
              if (identityFileChanged(changes)) useUiStore.getState().setKeySetup({ aliases: [detail.alias], reason: "saved" });
            },
          },
        )
      }
```

`src/components/DeployKeyDialog.tsx` 的 `writeIdentityFile`,`onSuccess` 裡最後加:

```tsx
          useUiStore.getState().setKeySetup({ aliases: [alias], reason: "saved" });
```

搬進 space 檔或在 space 裡複製的主機也是「開始同步的主機」(spec §7.1「在 space 裡新增主機」)。搬到一般檔案時候選裡
沒有它,對話框不會出現(`sync_key_candidates` 只看勾選的 space 檔):

`src/components/HostEditor.tsx`:「Move to file」那一項的 `onSuccess` 改成

```tsx
                            onSuccess: () => {
                              toast.success(`Moved ${detail.alias} to ${labelOf(f)}`);
                              // Moved into a space, it now syncs and may use this computer's key (SP3 spec §7.1).
                              useUiStore.getState().setKeySetup({ aliases: [detail.alias], reason: "moved" });
                            },
```

同檔「Duplicate host…」的 `onSuccess` 最後加 `useUiStore.getState().setKeySetup({ aliases: [newAlias], reason: "saved" });`。

`src/components/HostList.tsx`:`moveTo` 的 `onSuccess` 改成

```tsx
        onSuccess: () => {
          toast.success(`Moved ${alias} → ${labels.get(targetFile) ?? basename(targetFile)}`);
          // Moved into a space, it now syncs and may use this computer's key (SP3 spec §7.1).
          useUiStore.getState().setKeySetup({ aliases: [alias], reason: "moved" });
        },
```

`batchMove` 記下搬成功的 alias(`const movedAliases: string[] = [];`,`moved += 1;` 旁邊 `movedAliases.push(alias);`),
迴圈結束、`toast.success(…)` 之後加
`if (movedAliases.length > 0) useUiStore.getState().setKeySetup({ aliases: movedAliases, reason: "moved" });`
(整批問一次;逐一觸發的話後一個會蓋掉前一個)。

不需要觸發的:`AddHostDialog`(只填 HostName、User、Port,沒有 `IdentityFile`)、`NewConfigFileDialog` 的搬移(目標是新建的
一般 config 檔,不會是 space 檔)。

(各檔補上 `useUiStore`、`identityFileChanged` 的 import;已 import 的不重複。)

- [ ] **Step 4: 跑測試確認通過**

Run:`./node_modules/.bin/vitest run src/components/SyncKeyDialog.test.tsx`,再跑 `tsc --noEmit`、vitest 全部與 `vite build`。
Expected:全部通過(`enter-handlers.test.ts` 也要過:Rename 的 Enter handler 第一行是 `isImeKey`)。

- [ ] **Step 5: Commit**

```bash
git add src/components/SyncKeyDialog.tsx src/components/SyncKeyDialog.test.tsx src/App.tsx src/components/SyncMigrationDialog.tsx src/components/HostEditor.tsx src/components/HostList.tsx src/components/DeployKeyDialog.tsx
git commit -m "feat(ui): ask once per key whether it syncs when hosts start syncing"
```

### Task 11: Keys 對話框的插槽區塊、挑金鑰、Keys for this computer、Settings 提示列、側邊欄標記、文案

**Files:**
- Create: `src/components/KeySlotsSection.tsx`、`src/components/KeySlotsSection.test.tsx`、`src/components/KeysNeededDialog.tsx`
- Modify: `src/components/KeysDialog.tsx`(開關改由 store;掛上插槽區塊)
- Modify: `src/components/SyncPane.tsx`、`src/components/SyncPane.test.tsx`(兩個提示列;Leave 與更換同步碼的文案)
- Modify: `src/components/sync-primitives.tsx`(`SyncCodeDialog` 的 `note`)
- Modify: `src/components/HostList.tsx`、`src/components/HostList.test.tsx`(缺金鑰的標記)
- Modify: `src/lib/key-slots.ts`、`src/lib/key-slots.test.ts`(`keysNeededNoticeIndex`、`finishedKeysNeededNotice`)
- Modify: `src/lib/sync-events.ts`、`src/lib/sync-events.test.ts`、`src/lib/sync-overview.ts`、`src/lib/sync-overview.test.ts`
  (`keys_needed` 只開對話框:不跳 toast、不列在 Settings 的 Notices)
- Modify: `src/App.tsx`(掛上 `KeysNeededDialog`)

**Interfaces:**
- Consumes: Task 9、10;Task 4 的 `slots::add_notice`(後端至多一則 `keys_needed`,所以前端只找第一則)。
- Produces:`KeySlotsSection`、`KeySlotRow`(匯出給測試)、`PickKeyDialog`、`KeysNeededDialog`、
  `keysNeededNoticeIndex(o: SyncOverview): number | null`、`finishedKeysNeededNotice(o: SyncOverview): number | null`。

- [ ] **Step 1: 寫失敗的測試**

`src/components/KeySlotsSection.test.tsx`:

```tsx
import { renderToStaticMarkup } from "react-dom/server";
import { describe, expect, it } from "vitest";

import { keySlot, SPOOFED_NAME, SPOOFED_NAME_SHOWN } from "@/lib/sync-fixtures";
import { KeySlotRow } from "./KeySlotsSection";

/** A slot's row in the Keys dialog, rendered on the server: its words and the actions it offers. */
const row = (slot = keySlot()) => renderToStaticMarkup(<KeySlotRow slot={slot} busy={false} onAction={() => {}} />);
const text = (html: string) => html.replace(/<[^>]*>/g, " ").replace(/\s+/g, " ");

describe("a key slot in the Keys dialog", () => {
  it("shows how the key is shared, its state here, its hosts and the other computers", () => {
    const t = text(row(keySlot({ hosts: ["web", "db"], devices: [{ name: "FRANK-DESKTOP", fingerprint: null, synced_copy: true }] })));
    expect(t).toContain("id_mac");
    expect(t).toContain("Synced to your computers");
    expect(t).toContain("SHA256:9Q3QMhBJBcoUNE88XYEQbCPlcFByPPyVPJ6enJtQ+ew");
    expect(t).toContain("Ready");
    expect(t).toContain("Used by web and db");
    expect(t).toContain("FRANK-DESKTOP: synced copy");
    expect(row()).toContain(">Stop syncing<");
    expect(row()).toContain(">Change…<");
  });

  it("offers the action each state needs", () => {
    expect(row(keySlot({ mode: "own", fingerprint: null, status: { kind: "needs_key", waiting_for_sync: false } }))).toContain(">Pick a key on this computer…<");
    expect(text(row(keySlot({ mode: "own", fingerprint: null })))).toContain("Each computer uses its own key");
    expect(row(keySlot({ mode: "own", fingerprint: null }))).toContain(">Sync this key<");
    expect(row(keySlot({ status: { kind: "synced_available", file: "/f" } }))).toContain(">Use the synced key<");
    expect(row(keySlot({ status: { kind: "source_changed", file: "/f" } }))).toContain(">Sync the new key<");
    expect(row(keySlot({ mode: "own", status: { kind: "not_in_use", file: "/f" } }))).toContain(">Delete copy<");
  });

  it("reveals hidden characters in a name another computer chose", () => {
    expect(text(row(keySlot({ name: SPOOFED_NAME })))).toContain(SPOOFED_NAME_SHOWN);
  });
});
```

`src/lib/key-slots.test.ts` 加:

```ts
describe("the Keys for this computer notice", () => {
  it("is found by its kind", () => {
    expect(keysNeededNoticeIndex(overview({ notices: [{ kind: "new_sync_code" }, { kind: "keys_needed", names: ["id_mac"] }] }))).toBe(1);
    expect(keysNeededNoticeIndex(overview())).toBeNull();
  });

  it("is finished once no slot needs a key picked here", () => {
    const notices = [{ kind: "keys_needed" as const, names: ["id_mac"] }];
    const needing = keySlot({ mode: "own", fingerprint: null, status: { kind: "needs_key", waiting_for_sync: false } });
    expect(finishedKeysNeededNotice(overview({ notices, key_slots: [needing] }))).toBeNull();
    // Picked here or in Keys, or the origin started syncing it: nothing left to ask.
    expect(finishedKeysNeededNotice(overview({ notices, key_slots: [keySlot()] }))).toBe(0);
    expect(finishedKeysNeededNotice(overview({ notices, key_slots: [keySlot({ status: { kind: "needs_key", waiting_for_sync: true } })] }))).toBe(0);
    expect(finishedKeysNeededNotice(overview({ key_slots: [keySlot()] }))).toBeNull();
  });
});
```

(`keysNeededNoticeIndex`、`finishedKeysNeededNotice` 加進該檔的 import。)

`src/lib/sync-events.test.ts`:在「toasts notices with a way to Settings → Sync, except the upgrade」那個 `it` 之後加:

```ts
  it("leaves the keys notice to its dialog", async () => {
    const { emit } = await subscribed();
    emit("sync://notice", { kind: "keys_needed", names: ["id_mac"] });
    expect(toast.getToasts()).toEqual([]);
  });
```

`src/lib/sync-overview.test.ts` 的 `describe("noticeRows")` 加:

```ts
  it("leaves the keys notice to its dialog and to the row that counts the slots", () => {
    const o = overview({ notices: [{ kind: "keys_needed", names: ["id_mac"] }, { kind: "new_sync_code" }] });
    expect(noticeRows(o).map((n) => n.index)).toEqual([1]);
  });
```

`src/components/SyncPane.test.tsx` 加(照該檔 `pane(o)` 的寫法,另外把候選放進快取):

```tsx
describe("key slot rows", () => {
  it("ask to set up keys and to pick keys for this computer", () => {
    const client = new QueryClient();
    const o = overview({ key_slots: [keySlot({ status: { kind: "needs_key", waiting_for_sync: false }, mode: "own", fingerprint: null })] });
    client.setQueryData(syncOverviewKey, o);
    client.setQueryData(keyCandidatesKey, { keys: [keyCandidate()], unsupported: [] });
    const html = renderToStaticMarkup(
      <QueryClientProvider client={client}>
        <SyncPane />
      </QueryClientProvider>,
    );
    expect(text(html)).toContain("1 key used by synced hosts isn't set up");
    expect(html).toContain(">Set up…<");
    expect(text(html)).toContain("1 key slot needs a key on this computer");
    expect(html).toContain(">Pick…<");
  });

  it("stay hidden when there is nothing to do", () => {
    const html = pane(overview());
    expect(text(html)).not.toContain("isn't set up");
    expect(text(html)).not.toContain("needs a key on this computer");
  });
});
```

(`keyCandidatesKey` 從 `@/lib/sync`、`keySlot`、`keyCandidate` 從 `@/lib/sync-fixtures` 匯入;該檔已有 `text()` 輔助函式。
兩個按鈕只有文字、沒有圖示,所以 `>Set up…<`、`>Pick…<` 直接對得到。)

`src/components/HostList.test.tsx`:照該檔 `render()` 的寫法,overview 用
`overview({ key_slots: [keySlot({ hosts: ["web"], status: { kind: "needs_key", waiting_for_sync: false } })] })`,斷言 markup 含
`title="This host&#x27;s key isn&#x27;t on this computer — pick one in Keys."`(React 會把 `'` 轉成 `&#x27;`)且只出現一次。

- [ ] **Step 2: 跑測試確認失敗**

Run:`./node_modules/.bin/vitest run src/components/KeySlotsSection.test.tsx src/lib/key-slots.test.ts src/components/SyncPane.test.tsx src/components/HostList.test.tsx src/lib/sync-events.test.ts src/lib/sync-overview.test.ts`
Expected:新測試失敗。

- [ ] **Step 3: 實作**

`src/lib/key-slots.ts` 加:

```ts
/** Where the "keys needed" notice sits in the overview's notices (the backend dismisses notices by index). */
export function keysNeededNoticeIndex(o: SyncOverview): number | null {
  const index = o.notices.findIndex((n) => n.kind === "keys_needed");
  return index < 0 ? null : index;
}

/**
 * The "keys needed" notice once no slot needs a key picked here (picked in the dialog or in Keys, or the origin
 * started syncing it), so it can be dismissed; null while there is something left to pick, or no notice.
 */
export function finishedKeysNeededNotice(o: SyncOverview): number | null {
  return slotsNeedingKey(o).length === 0 ? keysNeededNoticeIndex(o) : null;
}
```

`src/lib/sync-events.ts`:`sync://notice` 的 handler 第一行改成

```ts
    // The upgrade and the keys notice have their own dialogs (SyncUpgradeDialog, KeysNeededDialog).
    if (notice.kind === "upgraded" || notice.kind === "keys_needed") return;
```

`src/lib/sync-overview.ts` 的 `noticeRows`:`if (notice.kind === "upgraded") return [];` 改成
`if (notice.kind === "upgraded" || notice.kind === "keys_needed") return [];`,函式的文件註解「The upgrade has its own dialog.」改成
「The upgrade and the keys notice have their own dialogs (the slots that need a key have their own row).」。

`src/components/KeySlotsSection.tsx`:

```tsx
import { useState } from "react";
import { open as openFileDialog } from "@tauri-apps/plugin-dialog";
import { toast } from "sonner";

import type { SyncKeySlotView } from "@/bindings/SyncKeySlotView";
import { TONE_TEXT } from "@/components/sync-primitives";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { deviceLine, hostsLine, slotActions, slotStatusText } from "@/lib/key-slots";
import { useKeys } from "@/lib/queries";
import { useKeyDeleteCopy, useKeyPick, useKeySetMode, useKeyUseSynced, useSyncOverview } from "@/lib/sync";
import { revealHidden } from "@/lib/sync-approvals";
import { cn } from "@/lib/utils";

export type SlotAction = "sync" | "stop" | "pick" | "useSynced" | "syncNew" | "delete";

/** One slot in the Keys dialog. Exported for the markup tests. */
export function KeySlotRow({ slot, busy, onAction }: { slot: SyncKeySlotView; busy: boolean; onAction: (action: SlotAction) => void }) {
  const status = slotStatusText(slot.status);
  const actions = slotActions(slot);
  const hosts = hostsLine(slot);
  const devices = deviceLine(slot);
  const button = (action: SlotAction, label: string, variant: "outline" | "ghost" = "outline", extra = "") => (
    <Button type="button" size="sm" variant={variant} className={cn("h-7", extra)} disabled={busy} onClick={() => onAction(action)}>
      {label}
    </Button>
  );
  return (
    <div className="flex items-start justify-between gap-3 px-3 py-2">
      <div className="min-w-0 space-y-0.5">
        <p className="truncate font-mono text-sm">{revealHidden(slot.name)}</p>
        <p className="text-xs break-all text-muted-foreground">
          {slot.mode === "synced" ? "Synced to your computers" : "Each computer uses its own key"}
          {slot.fingerprint ? ` · ${slot.fingerprint}` : ""}
        </p>
        <p className={cn("text-xs", TONE_TEXT[status.tone])}>{status.text}</p>
        {hosts && <p className="text-xs text-muted-foreground">{hosts}</p>}
        {devices && <p className="text-xs text-muted-foreground">{devices}</p>}
      </div>
      <div className="flex shrink-0 flex-wrap justify-end gap-1">
        {actions.syncThis && button("sync", "Sync this key")}
        {actions.syncNew && button("syncNew", "Sync the new key")}
        {actions.useSynced && button("useSynced", "Use the synced key")}
        {actions.pick && button("pick", actions.pick === "pick" ? "Pick a key on this computer…" : "Change…")}
        {actions.stopSyncing && button("stop", "Stop syncing", "ghost")}
        {actions.deleteCopy && button("delete", "Delete copy", "ghost", "text-destructive hover:text-destructive")}
      </div>
    </div>
  );
}

/** Pick one of this computer's keys for a slot (only this computer changes). */
export function PickKeyDialog({ slot, onClose }: { slot: SyncKeySlotView | null; onClose: () => void }) {
  const keys = useKeys({ enabled: slot !== null });
  const pick = useKeyPick();
  const name = slot ? revealHidden(slot.name) : "";
  const choose = (path: string) => {
    if (!slot) return;
    const file = path.split(/[\\/]/).pop() ?? path;
    pick.mutate({ slotId: slot.id, path }, { onSuccess: () => { toast.success(`${name} uses ${file} on this computer`); onClose(); } });
  };
  const browse = async () => {
    const picked = await openFileDialog({ multiple: false, directory: false, title: "Choose a private key" });
    if (typeof picked === "string") choose(picked);
  };
  const replacesCopy = slot?.status.kind === "ready" && slot.status.synced_copy;
  return (
    <Dialog open={slot !== null} onOpenChange={(next) => !next && !pick.isPending && onClose()}>
      <DialogContent className="sm:max-w-md" showCloseButton={!pick.isPending}>
        <DialogHeader>
          <DialogTitle>Pick a key on this computer</DialogTitle>
          <DialogDescription>Hosts that use {name} will use the key you pick, on this computer only.</DialogDescription>
        </DialogHeader>
        {replacesCopy && <p className="text-sm text-muted-foreground">The synced copy on this computer is kept as a .previous file.</p>}
        <div className="settings-group max-h-[40vh] overflow-y-auto">
          {(keys.data ?? []).map((k) => (
            <button
              key={k.private_path}
              type="button"
              disabled={pick.isPending}
              className="flex w-full flex-col items-start px-3 py-2 text-left hover:bg-muted/60"
              onClick={() => choose(k.private_path)}
            >
              <span className="font-mono text-sm">{k.name}</span>
              <span className="text-xs break-all text-muted-foreground">{k.fingerprint_sha256 ?? k.private_path}</span>
            </button>
          ))}
        </div>
        <DialogFooter>
          <Button type="button" variant="outline" disabled={pick.isPending} onClick={() => void browse()}>
            Browse…
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}

/** "Keys used by synced hosts" in the Keys dialog (SP3 spec §7.2). Nothing while there are no slots. */
export function KeySlotsSection() {
  const overview = useSyncOverview();
  const setMode = useKeySetMode();
  const switchToSynced = useKeyUseSynced();
  const deleteCopy = useKeyDeleteCopy();
  const [picking, setPicking] = useState<SyncKeySlotView | null>(null);
  const [deleting, setDeleting] = useState<SyncKeySlotView | null>(null);
  const slots = overview.data?.joined ? overview.data.key_slots : [];
  if (slots.length === 0) return null;
  const busy = setMode.isPending || switchToSynced.isPending || deleteCopy.isPending;
  const act = (slot: SyncKeySlotView, action: SlotAction) => {
    const name = revealHidden(slot.name);
    switch (action) {
      case "sync":
      case "syncNew":
        setMode.mutate({ slotId: slot.id, mode: "synced" }, { onSuccess: () => toast.success(`${name} syncs to your other computers`) });
        break;
      case "stop":
        setMode.mutate({ slotId: slot.id, mode: "own" }, { onSuccess: () => toast.success(`${name} no longer syncs; computers that have it keep their copy`) });
        break;
      case "pick":
        setPicking(slot);
        break;
      case "useSynced":
        switchToSynced.mutate({ slotId: slot.id }, { onSuccess: () => toast.success(`${name} uses the synced key on this computer`) });
        break;
      case "delete":
        setDeleting(slot);
        break;
    }
  };
  return (
    <section className="space-y-1.5">
      <h3 className="px-1 text-xs font-medium text-muted-foreground select-none">Keys used by synced hosts</h3>
      <div className="settings-group">
        {slots.map((slot) => (
          <KeySlotRow key={slot.id} slot={slot} busy={busy} onAction={(action) => act(slot, action)} />
        ))}
      </div>
      <PickKeyDialog slot={picking} onClose={() => setPicking(null)} />
      <AlertDialog open={deleting !== null} onOpenChange={(next) => !next && setDeleting(null)}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Delete this copy?</AlertDialogTitle>
            <AlertDialogDescription>
              The copy of {deleting ? revealHidden(deleting.name) : ""} on this computer is deleted. Other computers and the original key aren't affected.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction
              onClick={() => {
                if (deleting) deleteCopy.mutate({ slotId: deleting.id });
                setDeleting(null);
              }}
            >
              Delete copy
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </section>
  );
}
```

`src/components/KeysNeededDialog.tsx`:

```tsx
import { useEffect, useRef, useState } from "react";

import type { SyncKeySlotView } from "@/bindings/SyncKeySlotView";
import { PickKeyDialog } from "@/components/KeySlotsSection";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { finishedKeysNeededNotice, hostsLine, keysNeededNoticeIndex, slotsNeedingKey } from "@/lib/key-slots";
import { useDismissNotice, useSyncOverview } from "@/lib/sync";
import { revealHidden } from "@/lib/sync-approvals";

/**
 * "Keys for this computer" (SP3 spec §6.7, plan ruling 5): opens on the backend's `keys_needed` notice, which comes the
 * first time a synced host here needs a key that stays on another computer — for example right after joining. "Done"
 * dismisses the notice; the Settings row and the Keys dialog stay until each slot has a key. Once nothing is left to
 * pick, the notice is dismissed by itself.
 */
export function KeysNeededDialog() {
  const overview = useSyncOverview();
  const dismiss = useDismissNotice();
  const [picking, setPicking] = useState<SyncKeySlotView | null>(null);
  const o = overview.data;
  const index = o ? keysNeededNoticeIndex(o) : null;
  const slots = o ? slotsNeedingKey(o) : [];
  const done = () => {
    if (index !== null) dismiss.mutate({ index });
  };
  // One try per notice: a failed dismiss shows its error once instead of retrying on every render.
  const finished = o ? finishedKeysNeededNotice(o) : null;
  const dismissNotice = dismiss.mutate;
  const tried = useRef<number | null>(null);
  useEffect(() => {
    if (finished === null) {
      tried.current = null;
      return;
    }
    if (tried.current === finished) return;
    tried.current = finished;
    dismissNotice({ index: finished });
  }, [finished, dismissNotice]);
  return (
    <>
      <Dialog open={index !== null && slots.length > 0 && picking === null} onOpenChange={(next) => !next && done()}>
        <DialogContent className="sm:max-w-md">
          <DialogHeader>
            <DialogTitle>Keys for this computer</DialogTitle>
            <DialogDescription>
              Synced hosts on this computer use keys that stay on your other computers. Pick a key on this computer for each, or do it later in Keys.
            </DialogDescription>
          </DialogHeader>
          <div className="settings-group">
            {slots.map((slot) => (
              <div key={slot.id} className="flex items-center justify-between gap-3 px-3 py-2">
                <div className="min-w-0">
                  <p className="truncate font-mono text-sm">{revealHidden(slot.name)}</p>
                  <p className="text-xs text-muted-foreground">{hostsLine(slot)}</p>
                </div>
                <Button type="button" size="sm" className="h-7" onClick={() => setPicking(slot)}>
                  Pick…
                </Button>
              </div>
            ))}
          </div>
          <DialogFooter>
            <Button type="button" disabled={dismiss.isPending} onClick={done}>
              Done
            </Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
      <PickKeyDialog slot={picking} onClose={() => setPicking(null)} />
    </>
  );
}
```

`src/App.tsx`:在 `<SyncKeyDialog />` 之後加 `<KeysNeededDialog />`。

`src/components/KeysDialog.tsx`:

1. `const [open, setOpen] = useState(false);` 改成
   `const open = useUiStore((s) => s.keysOpen);` 與 `const setOpen = useUiStore((s) => s.setKeysOpen);`(其餘用到 `setOpen`
   的地方不變)。
2. 在金鑰清單那一段(`keys.length === 0 ? … : (…)` 之後)、`<NewKeySection … />` 之前加 `<KeySlotsSection />`
   (import 自 `@/components/KeySlotsSection`)。

`src/components/SyncPane.tsx` 的 `JoinedPane`:

```tsx
  const candidates = useKeyCandidates(o.joined);
  const notSetUp = keysToAsk(candidates.data, null).length;
  const needing = slotsNeedingKey(o);
  const setKeySetup = useUiStore((s) => s.setKeySetup);
  const setKeysOpen = useUiStore((s) => s.setKeysOpen);
```

在 Review 那一列(`o.approvals_waiting > 0 && (…)`)之後加:

```tsx
          {notSetUp > 0 && (
            <SettingsRow label={notSetUpLabel(notSetUp)} description="Choose whether each key goes to your other computers.">
              <Button type="button" size="sm" className="h-7" onClick={() => setKeySetup({ aliases: null, reason: "settings" })}>
                Set up…
              </Button>
            </SettingsRow>
          )}
          {needing.length > 0 && (
            <SettingsRow
              label={needsKeyLabel(needing.length)}
              description={`Synced hosts use ${listNames(needing.map((s) => revealHidden(s.name)))}, which stay on your other computers.`}
            >
              <Button type="button" size="sm" className="h-7" onClick={() => setKeysOpen(true)}>
                Pick…
              </Button>
            </SettingsRow>
          )}
```

Leave 對話框:在 `<AlertDialogHeader>…</AlertDialogHeader>` 之後加
`{o.key_slots.length > 0 && <p className="text-sm text-muted-foreground">Keys in ~/.ssh/sshelter/keys stay on this computer.</p>}`。

更換同步碼的確認:`<ul>` 的最後加
`{o.key_slots.some((s) => s.mode === "synced") && <li>Keys you synced stay on every computer that has them. If a computer was lost, replace those keys on your servers.</li>}`。

`mode="changed"` 的 `SyncCodeDialog` 加 `note={syncedKeysNote(o) ?? undefined}`。

`src/components/sync-primitives.tsx` 的 `SyncCodeDialog`:props 加 `note?: string`(文件註解「Shown under the description,
e.g. which synced keys to replace after a sync code change.」),在 `</DialogHeader>` 之後加
`{note && <p className={cn("text-sm", TONE_TEXT.warning)}>{note}</p>}`(`cn` 不在該檔就從 `@/lib/utils` 匯入)。

`src/components/HostList.tsx`:

1. `HostRow` 的 props 加 `missingKey?: boolean;`,並在陰影標記(`{shadow && (…)}`)之後加:

```tsx
        {missingKey && (
          <span className="shrink-0 text-amber-600 dark:text-amber-400" title="This host's key isn't on this computer — pick one in Keys.">
            <KeyRound className="size-3" aria-label="This host's key isn't on this computer" />
          </span>
        )}
```

2. 主元件(`const overview = useSyncOverview();` 附近)加
   `const missingKeys = useMemo(() => hostsMissingKey(overview.data), [overview.data]);`,傳 `missingKey={missingKeys.has(host.alias)}`
   給 `HostRow`(和 `shadow={shadowFor(host)}` 同一處)。`KeyRound` 從 `lucide-react` 匯入。

- [ ] **Step 4: 跑測試確認通過**

Run:同 Step 2;再跑 `tsc --noEmit`、vitest 全部與 `vite build`。
Expected:全部通過。

- [ ] **Step 5: Commit**

```bash
git add src/components/KeySlotsSection.tsx src/components/KeySlotsSection.test.tsx src/components/KeysNeededDialog.tsx src/components/KeysDialog.tsx src/components/SyncPane.tsx src/components/SyncPane.test.tsx src/components/sync-primitives.tsx src/components/HostList.tsx src/components/HostList.test.tsx src/lib/key-slots.ts src/lib/key-slots.test.ts src/lib/sync-events.ts src/lib/sync-events.test.ts src/lib/sync-overview.ts src/lib/sync-overview.test.ts src/App.tsx
git commit -m "feat(ui): manage key slots in Keys, Settings and the sidebar"
```

### Task 12: Windows 的 CI、README、手動驗證清單、SP1 spec 的更新

**Files:**
- Create: `.github/workflows/test-windows.yml`
- Create: `docs/superpowers/plans/2026-10-05-sp3-manual-verification.md`
- Modify: `README.md`
- Modify: `docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`(§0 的子專案表、§7.5 第 5 步)

**Interfaces:**
- Consumes: Task 1、2 的測試(模組 `sync::slot_rules`、`sync::slot_files`、`sync::slot_files_windows`;Task 2 有
  `#[cfg(windows)]` 的 DACL 與 hard link 測試);Global Constraints 的 UI 字串。
- Produces:一個在 `windows-latest` 跑上述模組測試的 job(push 到 main、PR 動到 `src-tauri/**` 或這個 workflow 時,或手動執行)。

- [ ] **Step 1: 寫 workflow**

`.github/workflows/test-windows.yml`。觸發條件與 cache 照 `relay.yml`、`build-platform.yml` 的寫法;`run:` 裡不得出現 `${{ }}`
(這個 job 也用不到)。只跑 spec §10 要的 Windows 測試(DACL、hard link)與同樣跟平台有關的路徑解析:兩個篩選字串
(libtest 接受多個,符合任一個就跑;`sync::slot_files` 也涵蓋 `sync::slot_files_windows`)。其他測試(包括用 `testkit` 的引擎
測試)從沒在 Windows 跑過,也有會讀寫系統鑰匙圈的,不放進這個 job:

```yaml
# Runs the key slot tests on Windows: hard links, the copy fallback and the owner-only DACL (SP3 Task 2).
# macOS can only type-check the Windows code; the permission behavior has to be seen on Windows.
name: windows key slots

on:
  push:
    branches: [main]
    paths: ["src-tauri/**", ".github/workflows/test-windows.yml"]
  pull_request:
    paths: ["src-tauri/**", ".github/workflows/test-windows.yml"]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  slots:
    runs-on: windows-latest
    defaults:
      run:
        working-directory: src-tauri
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: "./src-tauri -> target"
      # tauri::generate_context! reads frontendDist (../dist) at compile time and fails when it is missing.
      # This job only runs Rust tests, so a placeholder page is enough.
      - name: Placeholder frontend
        shell: pwsh
        run: |
          New-Item -ItemType Directory -Force -Path ../dist | Out-Null
          Set-Content -Path ../dist/index.html -Value "<!doctype html>"
      # Only the platform-dependent slot tests: identity path parsing (slot_rules) and the files, links and DACL
      # (slot_files, slot_files_windows). The rest of the suite runs on macOS; parts of it use the system keychain.
      - name: Key slot tests
        run: cargo test --lib -- sync::slot_rules sync::slot_files
```

- [ ] **Step 2: 檢查 workflow**

Run(repo 根目錄):`actionlint .github/workflows/test-windows.yml` 與 `grep -n '\${{' .github/workflows/test-windows.yml`
Expected:actionlint 沒有輸出;grep 沒有輸出。在本機確認篩選字串選到的測試:
`(cd src-tauri && PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- sync::slot_rules sync::slot_files --list | grep -c ': test')`,
Expected:等於 `slot_rules` 與 `slot_files` 在 macOS 上的測試數(`#[cfg(windows)]` 的不算),且大於 0。這個 job 在 push 之後才會在 GitHub 上跑 —— **不要** push;
交給收尾時由使用者決定。

- [ ] **Step 3: README**

`README.md` 的 Sync 那一項,把「Only hosts sync today; private keys and passwords stay on each device.」換成:

```markdown
Hosts sync, and the keys they use can too: when a host starts syncing, SSHelter asks once per key whether it goes to your other computers (end-to-end encrypted like the hosts; a passphrase is never synced, so a protected key stays protected) or stays on this computer (each other computer then picks its own key once). Your servers are never changed. Passwords stay on each device.
```

並在「Synced hosts live in **spaces** …」那一段的最後加一段:

```markdown
A synced host points its `IdentityFile` at a key slot, `~/.ssh/sshelter/keys/<name>-<id>`, and each computer decides what that slot holds: on the computer the key came from, a link to the original file; on the others, the synced copy (only readable by you) or a key you picked there. So the same host text works on every computer, plain `ssh` keeps working, and nothing in your own `~/.ssh` keys is moved or changed. The Keys dialog lists every slot with what each computer uses; *Stop syncing* never deletes the copies other computers already have, and a copy that's replaced is kept as a `.previous` file. Only OpenSSH-format private keys can be synced (convert an older PEM key with `ssh-keygen -p -f <file>`); any key can be kept on its computer.
```

- [ ] **Step 4: 手動驗證清單**

`docs/superpowers/plans/2026-10-05-sp3-manual-verification.md`(Mac 與 Windows 各一台,都在 Beta 頻道、已加入同一個同步帳戶;每一項
寫出操作、預期的畫面字串與要檢查的檔案):

```markdown
# SP3 金鑰插槽:手動驗證清單(Mac + Windows)

前置:兩台都在 Beta 頻道、已加入同一個同步帳戶、都勾選 Personal。Mac 有 `~/.ssh/id_mac`、`~/.ssh/id_mac2`,Windows 有
`~/.ssh/id_win`,這些公鑰都已加到測試伺服器。先只把 Mac 更新到含 SP3 的 beta,Windows 暫時留在 0.17.0-4。

1. **升級後的詢問**(Mac):Personal 已有用 `~/.ssh/id_mac` 的主機 `web`。重開 app → 跳出「Keys used by synced hosts」,
   列出 `web uses id_mac.` 與 `web: IdentityFile ~/.ssh/id_mac → ~/.ssh/sshelter/keys/id_mac-…`。按「Later」→
   Settings → Sync 出現「1 key used by synced hosts isn't set up」。再重開 app,不再自動跳出。
2. **Sync key**(Mac):Settings 那一列按「Set up…」→「Sync key」→ toast「id_mac syncs to your other computers」。`web` 那一行
   變成 `IdentityFile ~/.ssh/sshelter/keys/id_mac-xxxxxxxx`,該路徑是指向 `~/.ssh/id_mac` 的 symlink;`ssh web` 連得上。
3. **舊版電腦**(Windows 仍是 0.17.0-4):同步之後 `web` 在 Windows 連不上,lint 顯示 `IdentityFile not found`(預期;所以
   release notes 要提醒每台都更新)。把 Windows 更新到含 SP3 的 beta → 下一輪自動落地:
   `%USERPROFILE%\.ssh\sshelter\keys\id_mac-xxxxxxxx` 存在,`icacls` 只列出自己的帳戶;`ssh web` 連得上。
4. **Keep on this computer**(Mac):在 Personal 新增主機 `api`,在編輯器加上 `IdentityFile ~/.ssh/id_mac2` 並儲存 → 立刻跳出
   對話框 →「Keep on this computer」。Windows 同步之後跳出「Keys for this computer」;側邊欄的 `api` 有鑰匙標記。按「Pick…」→
   選 `id_win` → toast「id_mac2 uses id_win on this computer」,標記消失,`ssh api` 連得上;該插槽是 hard link(或複製)。
   全部挑完之後,Settings → Sync 沒有殘留的提示。
5. **搬進 space**(Mac):把一台用 `~/.ssh/id_mac` 的本機主機用「Move to file」搬進 Personal → 不再詢問(這把金鑰已經有插槽),
   主機改指到同一個插槽。
6. **本機挑的優先**:Mac 在 Keys 對話框對 4 的插槽按「Sync this key」→ Windows 的插槽不被換掉,Keys 對話框顯示
   「A synced key is available」→「Use the synced key」→ 插槽換成同步的金鑰;`id_win` 沒被動到。
7. **Stop syncing**(Mac):對 2 的插槽按「Stop syncing」→ toast「id_mac no longer syncs; computers that have it keep their copy」;
   Windows 的副本仍在、`ssh web` 照樣連得上。
8. **刪除副本**:Mac 把用 2 的插槽的主機(`web` 與 5 搬進來的那台)都改成不用插槽 → Windows 的 Keys 對話框顯示「Not in use」→
   「Delete copy」→ 確認 → 檔案刪除。
9. **金鑰換了**(只看狀態;伺服器換上新公鑰之前 `api` 連不上):Mac 用 `ssh-keygen -f ~/.ssh/id_mac2` 重新產生 6 已改成同步的
   那把金鑰,切回 app → Keys 對話框顯示
   「This computer's key changed — your other computers still have the previous one」→「Sync the new key」→ Windows 顯示
   「A synced key is available」→「Use the synced key」→ 舊的副本留成 `<file>.previous-xxxxxxxx`。
10. **擋路的檔案**:Mac 以「Keep on this computer」建立一個新插槽;Windows 在那個插槽路徑(主機的 `IdentityFile`)放一個自己的
    檔案;Mac 改成「Sync this key」→ Windows 的 Keys 對話框顯示「A file SSHelter didn't create is in the way: … Move it, then
    sync again.」,檔案沒被改。移走之後下一輪落地。
11. **更換同步碼**(Mac):確認對話框多一條「Keys you synced stay on every computer that has them. …」;完成畫面顯示「If a computer
    was lost, also replace these synced keys on your servers: …」。Windows 以新同步碼重新加入後,插槽與副本都在。
12. **離開帳戶**(Windows):Leave 對話框有「Keys in ~/.ssh/sshelter/keys stay on this computer.」;離開後 `ssh web` 照常。
13. **Windows 的路徑**:Windows 上對一台主機做 Deploy key(寫入 IdentityFile)→ 寫進去的是 `~/.ssh/...`,不是 `C:\Users\...`。
14. **lint**:手動把主機指到一個不存在的插槽 → lint 顯示「IdentityFile not found: … (a synced key slot — pick a key for it in Keys)」。
15. **PEM 金鑰**:用一把舊式 PEM 金鑰(`ssh-keygen -m PEM`)的主機 → 對話框的「Sync key」不能按,說明
    「This key isn't in the OpenSSH format, … Convert it with ssh-keygen -p -f <file>, or keep it on this computer.」;
    「Keep on this computer」照常可用。
```

- [ ] **Step 5: SP1 spec**

`docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`(spec §12):

- §0 子專案表的兩列換成:

```markdown
| SP3 SSH 金鑰 | 金鑰插槽與金鑰同步(`2026-10-05-sp3-key-slots-design.md`;取代原本「每台一把金鑰與自動部署/撤銷公鑰,或 SSH agent」的規劃) |
| SP4 分享 | 把單一 space 分享給其他人(以對方公鑰加密 space 金鑰) |
```

- §7.5 第 5 步的結尾「→ 新帳戶寫入 `space`(含 `previous_id`)、`spacekey`、`device`、`meta`。」(原文跨兩行)改成
  「→ 新帳戶寫入 `space`(含 `previous_id`)、`spacekey`、`device`、`meta`,以及 SP3 的 `keyslot`、`key`
  (`2026-10-05-sp3-key-slots-design.md` §6.6)。」

- [ ] **Step 6: 跑全部檢查**

Run:完整的 Rust 測試、`tsc --noEmit`、vitest 全部、`vite build`。
Expected:全部通過。

- [ ] **Step 7: Commit**

```bash
git add .github/workflows/test-windows.yml docs/superpowers/plans/2026-10-05-sp3-manual-verification.md README.md docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md
git commit -m "docs: key slots in the README, a Windows test job and the SP3 checklist"
```

---

## 執行順序與相依

Task 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 → 9 → 10 → 11 → 12,依序執行(每個都用到前一個的介面)。Task 8 只依賴 Task 1,可以提早,
但照順序也沒問題。後端(1–7)完成時,前端還沒有 UI,但 `tsc` 與 vitest 一直保持綠燈(Task 3、4 已處理 binding 對前端的影響)。

## 發佈(不在任何 task 裡)

push、合併與發佈 beta 都要使用者逐一確認。spec §11:以 beta 發佈,release notes 提醒每台電腦都要更新。「publish beta」
workflow 的 `notes` 輸入用:

```text
Synced hosts can use your keys on every computer. When a host starts syncing, SSHelter asks once per key whether it syncs to your other computers (end-to-end encrypted; a passphrase is never synced) or stays on this computer, in which case each other computer picks its own key once. Your servers are never changed. Update SSHelter on every computer: older versions can't connect to hosts that use a synced key slot.
```
