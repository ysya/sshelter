# Sync v2 — B3b(引擎接線:v1 升級、接上 app、更換同步碼、搬移精靈;B4 handoff)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 B3a 的引擎核心之上完成 Sync v2 的引擎:v1 狀態自動升級到 v2 帳戶(spec §7.6)、把 app 從 v1 引擎切換到
v2(Tauri 外殼、背景執行緒、存檔 hook、視窗焦點、狀態 DTO、commands 與事件、跨 space 搬移、搬移精靈),退役 v1 引擎、
更換同步碼(spec §7.5)、搬移精靈的「一個來源檔建立一個 space」,並在文末列出 B4 前端需要的全部介面。

**Architecture:** 引擎本體(`round`、`account`、`spaces`、`upgrade`、`rotation`、`migrate`)都只依賴 `SyncEnv`,以
`testkit` 與 `FakeRelay` 做多台裝置的決定性測試;`engine.rs` 縮成 Tauri 外殼 —— 行程間同步鎖、啟動(載入 v2 狀態,
或把已加入的 v1 狀態排給背景執行緒升級)、背景執行緒(`round::sync_once` + `round::next_delay`)、存檔 hook、
`TauriEvents`,以及包裝引擎函式的 commands(`spawn_blocking`;改結構的動作全程持有 lifecycle 鎖)。v1 的同步引擎、
v1 的 relay client、v1 狀態的存取在 Task 2 一起移除,之後只剩 v1 升級會讀 v1 的狀態檔與 `hosts.config`。

**Tech Stack:** Rust 2021、Tauri 2;只用既有相依 —— `serde` / `serde_json`、`reqwest 0.13`(blocking,只經
`RelayClient`)、`thiserror 2`;dev:`tempfile 3`、`ts-rs 10`、`mockito 1`(既有的 relay client 測試)。

**Spec:** `docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`(§7.2 搬移、§7.5、§7.6、§8 的後端、§9、§13 I3/I4)。
前篇:`docs/superpowers/plans/2026-10-02-sync-v2-b3a-engine-core.md`(B3a 的四個 task 必須已完成);B2 與 relay 的介面
見 `docs/superpowers/plans/2026-10-02-sync-v2-b2-core.md`、`docs/superpowers/plans/2026-10-02-sync-v2-b1-relay.md`。

## Global Constraints

- 兩個會寫入真實 keychain 的既有測試一律略過:`secrets::tests::round_trip_set_get_delete` 與 `askpass::tests::env_secret_takes_priority_over_keychain`。本計畫寫的「N passed」都是兩個都略過時的數字。
- Rust 註解用繁體中文(沿用 `src-tauri/src/sync/`);識別字、錯誤訊息、UI 字串、commit 訊息用英文;Conventional Commits。
- 只 `git add` 每個 task 列出的路徑(含該 task 由 ts-rs 產生的 `src/bindings/*.ts`);不得 stage `Cargo.lock`、
  `.superpowers/`、`relay/` 或其他無關檔案。
- 每個 task 結束時 `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain` 必須全綠才 commit
  (那兩個既有的測試會讀寫真正的 OS keychain,一律略過;下文的「跑測試」都是這個指令)。測試不連網、不碰真正的 keychain 與家目錄:引擎一律經
  `SyncEnv` 注入外界(`testkit::TestDevice`、`MemKeychain`、`FakeRelay`、`TestClock`);`engine.rs` 的啟動邏輯以注入
  keychain 的純函式 `startup` 測試;Tauri 外殼本身(`initialize`、`worker_loop`、commands)不做單元測試。
- 不新增 crate,`Cargo.toml` 不變。repo 的 `Cargo.lock` 裡 `sshelter` 自己的版本還是 `0.15.1`,第一次 `cargo test`
  會改掉那一行 —— 不屬於 B3,不要 stage,也不要為它另開 commit。
- 鎖:順序固定 lifecycle → doc → backed_up → core;網路 I/O 一律不持有 doc 或 core 鎖;Tauri command 一律在
  `spawn_blocking` 裡呼叫引擎(`reqwest::blocking` 不能在 tokio runtime 內呼叫);`SyncEvents` 只在放掉所有鎖之後呼叫
  (`TauriEvents` 會同步等待主執行緒重建 tray)。
- 局部提交與 generation(spec §7.1、§12 #8):一輪裡的提交只改自己的區段並比 generation(`runtime::commit`);生命週期與
  結構性變更一律換 generation(`runtime::mutate`,或在 doc → core 鎖內自己換)。只改提示清單的 `sync_dismiss_notice`
  不換 generation(在途的輪次不必重跑、不多查一次 relay)。
- 祕密(同步碼、權杖、`enc_key`、`spacekey` payload)永不進 log、錯誤訊息、`Debug` 輸出、事件 payload 或狀態檔明文。
  同步碼只經 `sync_create_account` / `sync_show_words` 的回傳值交給 UI(前端只放在元件狀態裡,不進查詢快取);新同步碼
  在更換完成前只在 keychain 的 `sync:mnemonic-next`。
- ts-rs:匯出型別的 u64 欄位一律 `#[cfg_attr(test, ts(type = "number"))]`(`Option<u64>` 用 `"number | null"`)。
- **B1(relay)最終審查的裁定**(spec §6.2、§6.4、§9、§13 I3/I4),B3a 已在引擎裡實作,本計畫負責接線並不得破壞:
  1. 依焦點輪詢:`WindowEvent::Focused` → `engine::window_focused` → `SyncRuntime::set_focused`;回到前景喚醒一輪。
     存檔 hook 與每個 Sync command 都記下操作時間(`note_activity`);背景執行緒的等待時間一律取 `round::next_delay`。
     (最終修正之後:回到前景與存檔 hook 是隱含的喚醒,在 `429` / `5xx` / keychain 的退避期間先記下、等退避結束才跑;`sync_now`、
     commands 與引擎自己的後續動作照舊馬上跑 —— spec §6.4。)
  2. 批次 `5xx` 第二次起逐條 `GET`、指數退避;`429` 絕不立刻重試(B3a `round::pull_all`)。
  3. push `409 frozen`:停止上傳、保留 dirty、`SyncOverview::frozen` 讓 UI 顯示「sync code changed」,不重試;使用者
     輸入新同步碼(`sync_rejoin_account`)之後才繼續。
  4. 凍結的 chain 被 `DELETE` 之後仍是凍結(`PUT` 回 200、push 回 409):v1 升級與更換同步碼都不把 `PUT` 成功當成
     「chain 可寫」;更換同步碼刪除舊 space chain 之後,舊裝置的上傳仍被擋下(Task 3 的測試)。
  5. 功能偵測:沒有 `freeze` → `sync_change_sync_code` 回 `rotation::NO_FREEZE_MESSAGE`、`SyncRelayView.freeze = false`
     (UI 停用並說明要先更新 relay);沒有 `pull-batch` → `SyncRelayView.batch_pull = false`(UI 提示「relay 可以更新」)。
- 前端在 B4 之前必須仍能編譯:Task 2 移除 v1 的 Rust DTO,但 `src/bindings/SyncStatus.ts`、`src/bindings/SyncDevice.ts`
  保留在 repo(不再由 ts-rs 產生),由 B4 換掉 Sync UI 時刪除。Task 2 之後現有 Sync pane 呼叫的 v1 命令不存在、
  `sync://status` 的 payload 變成 `SyncOverview` —— 這段期間 Sync pane 不能用,其他功能(編輯 config、連線、tray)照常;
  B4 要改的地方列在文末「B4 handoff」。
- UI 一律稱「sync code」(B4 計畫的 A2):Task 2 起後端的錯誤訊息也一樣,`crypto::normalize_mnemonic` / `seed_of` /
  `generate_mnemonic` 不再說「recovery phrase」或「recovery words」。
- 數值照抄 spec:space0「Synced」/`synced`;v1 不能進 space0 的區塊另存 `~/.ssh/sshelter-v1-kept.config`;v1 狀態備份
  `sync-state.v1-backup.json`;keychain `sync:mnemonic` / `sync:mnemonic-next`;建立 chain 被限流時暫停 1 小時後接續;
  第一輪在啟動時就跑(`engine::Wait::launch`;最終修正拿掉了原本的 `FIRST_DELAY`),之後的間隔一律取 `round::next_delay`;
  同步鎖 `sync.lock`(約 2 秒內重試 10 次)。
- 狀態檔的相容性(B2 審查 M4):狀態檔裡任何一個 enum 讀到不認得的 variant,整份檔案就讀不懂 —— 從 beta 降回正式版
  就會丟掉同步狀態。`RotationStep` 等狀態檔裡的 enum 要加 variant 時必須升狀態檔的 `version`;`notices` 例外(逐則讀、
  略過不認得的種類,B3a Task 1)。
- **Task 1 實際的介面**(已執行,`103cdeb`、`1e841cf`,審查後的修正):`merge::push_outgoing(relay, chain_id, token, outgoing)`
  回 `Pushed`、不再是 `Result`;`Pushed` 多了 `pub error: Option<RelayError>`,不再 derive `Clone` / `PartialEq`。它在第一個
  失敗的批次停下,之前被接受的照樣保留,所以 `frozen` 也可能和非空的 `accepted` 一起出現。呼叫端一律:先套用 `accepted`
  (`apply_pushed_space` / `apply_pushed_account`)→ 再看 `frozen` → 最後把 `error` 當成以前的 `Err`(限流、relay `5xx`、
  其他錯誤回傳)。另外:`FakeRelay::set_push_quota(Option<u32>)`(前 n 次上傳照常,之後 `413`);回應的筆數對不上 →
  `error = BadResponse`;relay 歷史倒退時,待核准與被拒絕版本的 seq 也歸零,原本乾淨的記錄加進 `republish`;
  `put_space_key` 在 `space_keys.chain_id != space_id` 時回錯誤。`sync::merge` 26 個測試、`sync::state_v2` 13 個。
- **B3a 的介面**(Task 2–4 與最終修正都已執行,`22d49cc`…`5c98f25`):`spaces::approve` /
  `spaces::reject` 收使用者看過的 `(alias, digest)`(`spaces::review_digest`:那一版內容的指紋)、回
  `Reviewed { applied, changed }`(待核准的已換成別的內容就略過、列在 `changed`,較新的版本留在清單上)—— 所以 `sync_approve` /
  `sync_reject` 收 `approvals: ReviewedVersion[]`(`{ alias, digest }`)、回 `ReviewOutcome { applied, changed, overview }`,
  `PendingApprovalView` 帶 `digest`(Task 2;序號與版本號都可能指到別的內容,所以以內容認)。`prepare_files` /
  `reconcile_space_files` 碰到主 config 被外部改過(`Conflict`)時重載 doc、在鎖都放掉之後發 `applied(0)`,同步輪次把它當成
  「馬上重跑」(B3a Task 4)。`rename_space` / `delete_space` 的檔案那一半碰到 `Conflict` 時回 Ok 並 `wake()`;`leave_account`
  在更換同步碼過了可以取消的階段時拒絕(B3b 最終修正之後的規則見文末「最終修正(已執行)」);刪除帳戶也刪掉 `chain_deletes` 裡
  排隊的 chain。`round::run_round` 回
  `RoundOutcome { frozen, markers, backoff }`(B3a Task 4 審查後):`markers` = 拉帳戶時看到的更換標記(`mark_frozen_chains` =
  false 時只回報、不記進狀態),`backoff` = 這一輪被限流或 relay 出錯、不要立刻重跑;某個 space 的上傳被 relay 拒絕(儲存
  額度滿了等)記成那個 space 的 `last_error`,其他 space 照常。Task 3 的第 2 步依此讓步與等待。最終修正之後還有:
  `leave_account(delete_remote = true)` 在 `frozen` 時只在這台離開、不刪 relay,回 `Err(LEAVE_REPLACED_MESSAGE)`(command
  原樣回傳);`create_account` / `join_account` 先把 v1 留下、主 config 的 Include 還讀著的 `~/.ssh/sshelter/hosts.config` 搬到
  `~/.ssh/sshelter-local/` 並留下 `left_account` 提示;每個以退避收尾的一輪都把 `failed_rounds` 加一、寫 `last_error`,`/v1/info` 的 `429` 也讓這一輪以退避
  結束;`note_activity` 直接存新的時間。
- **Task 1 實際的做法**(已執行,`8ffd36f`、`f95fed6`、`1ee11de`,兩輪審查後的修正;報告在
  `.superpowers/sdd/2026-10-02-sync-v2-b3b-engine-wiring/task-1-report.md`):升級先讀帳戶、只 `PUT` 還不存在的 chain(已刪除的
  space0 不重建,帳戶已被凍結時不建 space0 的 chain);v1 狀態在檔案階段之前就備份;主 config 的兩個 Include 修改一次寫入(失敗就
  還原),任何時刻 ssh 讀得到的主機都在某個被列出的檔案裡;重跑時認得前一次的 space0 檔;v1 的 `baseline_established` 帶進
  space0;`Conflict` 重載 doc 並發 `applied(0)`;升級的 `429` / `5xx` 算進 `failed_rounds`;檔案階段前與切換前再比一次
  `legacy` 與 generation(使用者在升級中離開 = superseded,不留錯誤);`split_v1_file` 回兩個值;測試用的 `TEST_HOOKS`
  (`#[cfg(test)]`)。`account::leave_account` 放棄卡住的升級時,把 `hosts.config` 與主 config 裡我們的 Include 現在讀得到的
  每個檔案一起搬到 `~/.ssh/sshelter-local/`(`abandoned_files`)。第二輪(`1ee11de`):升級的來源由主 config 有效的 Include 決定
  (`plan_sources`),清單沒列的殘留檔備份後移除、絕不併入;v1 使用者自己放在 `~/.ssh/sshelter/`、被我們的 Include 列到的檔案改成
  本機檔案(與離開共用 `account::listed_our_files`),`SyncNotice::Upgraded` 多了 `moved_files: Vec<String>`(`#[serde(default)]`,
  `src/bindings/SyncNotice.ts` 已重新產生);離開在 doc 鎖內決定要不要放棄升級,兩條路搬走的檔案都列進 `LeftAccount`,放棄時清掉
  `last_error` 與 `failed_rounds`;切換之後的失敗重載 doc 並發 `applied(0)`;檔案階段前先檢查 doc 是否過時;測試用的
  `account::BEFORE_LEAVE`、`TEST_HOOKS.fail_removal`。之後的 task 只用到 `upgrade::shell_state`(簽章不變)。
- **Task 3 實際的做法**(已執行,`7ce26a2`、`8a8c961`,兩輪審查後的修正 `8b11fe5`…`6ca1323`;報告在
  `.superpowers/sdd/2026-10-02-sync-v2-b3b-engine-wiring/task-3-report.md`):
  - `8a8c961`:第 3 步寫標記撞到 `Frozen` 時先讀舊帳戶,只有上面沒有這台自己的標記才讓步 —— 中斷之後重試不會讓給自己(否則舊帳戶
    凍結、卻沒有人拿著新碼)。
  - 第 3–7 步的 relay 錯誤分類:`429` / `5xx` 經 `round::note_backoff`(改成 `pub(crate)`)算失敗的一輪、`last_error` 是
    `RATE_LIMITED_MESSAGE` / `RELAY_TROUBLE_MESSAGE`、回 Ok 不立刻重跑;儲存上限、不合規格的請求、讀不懂的回答算失敗的一輪並回 Err
    (原因在 `last_error`);連不上不算;建立 chain 的 `429` 暫停也算失敗的一輪;一步做完就清掉 `failed_rounds` 與 `last_error`。
  - keychain:`rotation::finish_interrupted_switch(keychain, chain_id) -> Option<InterruptedSwitch>`(`pub struct InterruptedSwitch
    { pub keys: ChainKeys, pub promoted: bool }`;keychain 寫不進去時 `promoted = false`、暫存的碼留著)。新增
    `rotation::retry_pending_swap(env)` 與 `runtime::SyncCore.swap_pending`(`round::sync_once` 每一輪最後重試;啟動時一樣設定):
    換不進 keychain 時 `last_error` 是 `SWAP_PENDING_MESSAGE`、同步照常;這段期間 `start_rotation` 回 `SWAP_PENDING_BLOCKS_MESSAGE`。
    新碼在 Freezing 之前就讀(不見了或對不上 → 停在 LocalChangesSent,還能取消);`start_rotation` 在 `mutate` 裡再檢查一次。
  - 重新加入:先 `saves_allowed`(另一個行程不會先動 keychain 或檔案);沒有任何勾選的 space 被新帳戶接續 → `NOT_A_SUCCESSOR_MESSAGE`
    (別的帳戶的碼,或換過兩次以上);驗證之後、切換之前把輸入的碼暫存在 `sync:mnemonic-next`(中斷由啟動補完;切換之前失敗就刪掉)。
  - 新帳戶沒有接續的勾選 space(重新加入與第 7 步切換)改成本機檔案,留下 `LeftAccount { kept_files }`;改不成時什麼都不換(重新
    加入:「…; nothing was changed — try again」;切換:算失敗的一輪,「…; SSHelter retries the sync code change by itself」)。切換時
    主 config 被外部改過(`Conflict`)= 重載後馬上重跑,不算失敗。第 7 步讀舊帳戶時 `404`(chain 已過期)= 沒有別台的標記。
  - `account::show_words` 優先回傳推導得出這個帳戶的暫存新碼(`pub(crate) account::staged_code_for`);`account::keep_files_local`
    改成 `pub(crate)`;測試用的 `MemKeychain::fail_writes_to`、`FakeRelay::expire`。
  - Task 3 結束時(兩個 keychain 測試都略過)`696 passed`:`sync::rotation` 37、`sync::engine` 9、`sync::account` 25。
- **Task 4 實際的做法**(已執行,`e950c05` 與本計畫一致,兩輪審查後的修正 `c89aa46`、`e164a3e`;報告在
  `.superpowers/sdd/2026-10-02-sync-v2-b3b-engine-wiring/task-4-report.md`):`move_into_new_spaces` 一組失敗不再中斷整批(那一組的
  每台都列為失敗,其他組照做);同一台主機(以 alias,或同一個區塊的另一個名稱)出現在兩組時只搬一次,後面的列為「listed in more
  than one group」;一組裡沒有任何能搬的主機就不建立 space(先在 doc 鎖內做搬移會做的拒絕);建立 space 之前比對 doc 與磁碟,過時
  就重載(`applied(0)`),重載失敗時那一組列為「the config changed on disk and could not be reloaded: <error>」、不建立 space;
  `unmovable_hosts` 只判斷搬移真的會拿的那個區塊。Task 4 之後 `708 passed`(`sync::migrate` 25)。
- 基準:四個 task 與 B3b 的最終修正都已在 repo 執行(HEAD `d469bc0`;兩個 keychain 測試都略過時 `740 passed`)。B4 以這一版
  為基準。

## Review Focus

1. **更換同步碼期間,另一台離線改了主機**:它的上傳被凍結擋下;使用者在那台輸入新同步碼之後,未上傳的修改以原時間戳
   帶進新帳戶、照 LWW 上傳,不會遺失;快照也包含別台已上傳、這台還在等待核准的記錄(Task 3
   `other_devices_freeze_and_rejoin_with_their_unsent_edits`、
   `the_snapshot_holds_changes_and_held_records_the_rotating_device_never_applied`)。
2. **兩台同時更換同步碼**:先凍結的那次勝出,另一台在第 2 步拉帳戶時看到對方的標記、或寫標記時發現已被凍結,就讓步、改走
   「輸入新同步碼」(不會停在第 2 步);兩個標記都寫成時不刪舊 space chain(另一次更換可能還要複製),並提示「另一台電腦也
   更換了同步碼」(Task 3 `a_device_that_loses_the_race_yields_to_the_other_rotation`、
   `a_marker_seen_while_this_device_still_sends_its_changes_yields_at_once`、`concurrent_markers_keep_the_old_chains_and_tell_the_device`)。
3. **在切換的最後一步當掉或被關掉**(狀態已換成新帳戶、keychain 還是舊碼),或建立 chain 被限流:重啟時以
   `sync:mnemonic-next` 補完;限流時暫停到下個小時自動接續;凍結之後不能取消(Task 3
   `startup_finishes_a_sync_code_switch_that_was_interrupted`(加在 `engine.rs` 的測試裡)、
   `a_rotation_is_cancellable_only_before_freezing_and_resumes_after_a_pause`)。
4. **兩台同時從 v1 升級、升級中斷後重跑、或 space0 已在別台被改名 / 刪除**:兩台得到同一個 space0、沒有衝突事件;
   重跑結果相同;改名過的沿用、刪除的不重建(v1 的主機留在本機檔案)(Task 1
   `two_devices_upgrading_at_once_end_up_in_the_same_space0_without_conflicts`、`an_upgrade_can_run_again_with_the_same_result`、
   `a_space0_renamed_elsewhere_is_kept_and_a_deleted_one_is_not_recreated`、`a_failed_upgrade_leaves_v1_untouched_and_is_retried`)。
5. **把不能同步的區塊搬進 space**(含 `Include`、帶引號的 keyword、wildcard、目標已有同名主機、目標 space 第一輪還沒
   完成、這個行程沒有同步引擎):逐台拒絕並說明原因,其他主機照搬;精靈事先列出不能搬的主機(Task 2
   `moving_into_a_space_refuses_what_cannot_sync_without_halting_the_batch`、`names_already_in_the_target_space_are_refused`、
   `moves_into_a_space_need_this_process_engine_and_a_finished_first_sync`;Task 4
   `one_space_per_source_file_creates_the_spaces_and_lists_what_cannot_move`)。

## 對 spec 的解讀(實作時的決定)

1. **v1 的 `Include` 區塊另存在哪**(§7.6 第 3 步「保留在另存的本機檔案」):`~/.ssh/sshelter-v1-kept.config`,並在主
   config 我們那一行 Include 之後加 `Include ~/.ssh/sshelter-v1-kept.config` —— 升級前這些區塊就生效,不能因升級而
   消失。它不在 `~/.ssh/sshelter/` 底下,所以不是「我們的」token,`ensure_include` 不會收走。`SyncNotice::Upgraded`
   列出檔案與主機,以及搬成本機檔案的使用者檔案(`moved_files`)。
2. **帳戶上已經有 space0**(別台先升級):沿用(可能已被改名,檔名用它目前的 slug),不再寫一份蓋掉它;space0 已被別台
   刪除:不重建,v1 檔案的全部內容留在本機另存檔。
3. **v1 帶進 space0 的已同步記錄**(§7.6 第 3 步「標為 dirty」):照原 metadata 重新上傳;它們輸給較新的遠端版本不是
   「本機修改被覆蓋」(`SpaceState.republish`),不發 `sync://conflict`。v1 本來就 dirty 的記錄照常算衝突。
4. **升級在哪裡做**:背景執行緒的第一輪(`round::sync_once`),不擋 app 啟動;完成前 core 放一個未加入的 v2 外殼給 UI
   看,會改狀態的命令回 `UPGRADING_MESSAGE`;失敗時 v1 狀態檔與 `hosts.config` 原封不動、錯誤顯示在狀態列,下一輪
   再試(§9)。升級一直做不完(例如 keychain 沒有同步碼)時,`sync_leave_account` 等於放棄升級、換成未加入的 v2 狀態
   (`hosts.config` 與主 config 裡我們的 Include 讀得到的檔案一起改成本機檔案)。
   沒有加入的 v1(在 v1 已離開)啟動時直接換成未加入的 v2 狀態、`hosts.config` 不動;之後建立或加入帳戶時,主 config 的 Include
   還讀它就由 B3a 搬到 `~/.ssh/sshelter-local/`(`left_account` 提示)。已加入的 v1 由升級移除 `hosts.config`,不會留給建立 / 加入。
5. **更換同步碼的取消**(§7.5「刪除已建立的新 chain」):新 chain 在第 5 步才建立,第 3 步之前取消時沒有 chain 要刪,
   只清 `sync:mnemonic-next` 與 `rotation`。
6. **兩台同時更換**(§7.5):第 2 步拉帳戶時看到別台的標記,或寫標記之前發現舊帳戶已被別台凍結 → 這台讓步(清掉進度與
   暫存的新碼,記下 `frozen`,改走「輸入新同步碼」);兩個標記都寫成時,第 6 步不刪舊 space chain,留給 relay 閒置過期清除。
   第 2 步的一輪以退避收尾(被限流、relay 出錯)時不立刻重跑,只有往下走了才立刻跑下一步;某個 space 的上傳被 relay 拒絕
   (例如儲存額度滿了)時錯誤留在那個 space 上,第 2 步不等它,它的 dirty 記錄在切換時帶進新 space。
7. **第 1 步之後才建立的 space**(§7.5「每個 space」):凍結之後以 relay 上的舊帳戶列出 space,第 5 步補上新位置與金鑰,
   一樣複製。
8. **第 7 步的中斷**:狀態已換成新帳戶、keychain 還是舊碼時,啟動流程以 `sync:mnemonic-next` 推導出狀態裡的帳戶就用它取代
   `sync:mnemonic`(`rotation::finish_interrupted_switch`)。
9. **更換後的檔名**:新 space 依 `previous_id` 沿用舊檔名(檔名裡的 8 碼仍是舊 id),不改名、不動 Include;勾選、待核准
   項目、被拒絕的版本一併帶過去,seq 與 cursor 歸零、基線已建立(第一輪是一般 LWW,不是基線輪)。
10. **建立 chain 被限流**(§6.6 每 IP 每小時 20 次):第 5 步暫停 1 小時(`SyncRotationView.paused_until_ms`),之後自動
    接續;已建立、已複製的不重做。
11. **重新加入的檢查**(§7.5「其他電腦」):只有 `frozen` 的裝置能用;輸入的必須是新帳戶的同步碼(舊碼直接拒絕)、新帳戶
    必須存在且沒有又被更換;對不到新 space 的勾選 space 不帶過去。
12. **跨 space 搬移**(§7.2):沿用 `config_move_host`(先寫目標、再從來源移除);目標是這台勾選的 space 檔時才套搬移規則
    (同步引擎在這個行程、目標 space 已完成第一輪、區塊沒有 wildcard / `Include` / 帶引號的 keyword、目標沒有同名主機)。
13. **「一個來源檔建立一個 space」**(§7.2「以來源檔名作為 space 名稱(檔名 alias 優先)」):UI 產生名稱(套用使用者
    在 sidebar 設的檔名 alias,沒有才用 `tag_for_file`),送 `NewSpaceGroup { name, aliases }`;後端每組先建立 space
    (建立者勾選,新的空 chain 不需要基線輪)再搬;建立失敗的那一組,主機全部列為失敗,其他組照常。
14. **不能搬的主機**(§7.2「精靈列出原因」、§8):`sync_unmovable_hosts` 列出不在任何 space 檔裡、所有 pattern 都具名、
    卻含 `Include` 或帶引號 keyword 的區塊;wildcard 區塊不是主機,不列出。
15. **刪除帳戶**(§7.3):`sync_leave_account(delete_remote: true)`;「是不是最後一台」由 UI 依 `SyncOverview.devices`
    判斷,後端不擋。同步碼已在別台更換(`frozen`)時後端只在這台離開、不刪 relay(回 `LEAVE_REPLACED_MESSAGE`),UI 那時不提供
    刪除帳戶。
16. **狀態推送**:`sync://status` 的 payload 是完整的 `SyncOverview`(取代 v1 的 `SyncStatus`);提示存在
    `SyncOverview.notices`,`sync_dismiss_notice(index)` 清掉。
17. **同步碼的文案**(B4 計畫的 A2):v1 的 UI 稱「recovery phrase」,v2 一律稱「sync code」;v1 在 Task 2 退役,所以
    `crypto` 的錯誤訊息在同一個 task 改成「a sync code has 24 words (got N)」、「invalid sync code: …」、「cannot generate
    a sync code: …」(錯誤內容只有字數或位置,不含使用者輸入的字)。

## 檔案結構

| 檔案 | 動作 | 責任 |
|---|---|---|
| `src-tauri/src/sync/upgrade.rs` | 新增(Task 1) | v1 → v2 升級(spec §7.6):推導帳戶與 space0、記錄搬移、檔案與 Include、狀態備份與切換 |
| `src-tauri/src/sync/round.rs` | 修改(Task 1、3) | `sync_once` 先做 v1 升級(Task 1)、`rotation` 存在時改由更換流程推進(Task 3) |
| `src-tauri/src/sync/engine.rs` | 改寫(Task 2);修改(Task 3、4) | Tauri 外殼:同步鎖、啟動、背景執行緒、存檔 hook、視窗焦點、`TauriEvents`、commands |
| `src-tauri/src/sync/dto.rs` | 改寫(Task 2) | `SyncOverview` 與子型別、`PendingApprovalView`、`ReviewedVersion`、`ReviewOutcome`、`overview()`、`pending_approvals()` |
| `src-tauri/src/sync/migrate.rs` | 改寫(Task 2);修改(Task 4) | 搬進 space(精靈與 sidebar 拖曳共用的規則)、跨檔同名主機;Task 4 加一個來源檔一個 space、不能搬的主機 |
| `src-tauri/src/sync/reconcile.rs` | 改寫(Task 2) | 只留記錄的加解密編碼與 `HostEffect`(v1 的合併規則已由 `merge` 取代) |
| `src-tauri/src/sync/relay.rs` | 修改(Task 2) | 移除 v1 的 `LegacyRelayClient` |
| `src-tauri/src/sync/state.rs` | 修改(Task 2) | v1 狀態只剩讀取用的型別、`state_path`、內建 relay、keychain account 名稱 |
| `src-tauri/src/sync/state_v2.rs` | 修改(Task 2) | `RotationStep` 匯出 TS;移除直接呼叫 `secrets` 的 `sync:mnemonic-next` helper(改經 `Keychain`) |
| `src-tauri/src/sync/hosts_file.rs` | 修改(Task 2) | 移除 v1 的 `INCLUDE_VALUE`、`ensure_managed_file` |
| `src-tauri/src/sync/crypto.rs` | 修改(Task 2) | 錯誤訊息改稱「sync code」 |
| `src-tauri/src/config/commands.rs` | 修改(Task 2) | `config_load` 依所有勾選的 space 檔決定是否喚醒;`config_move_host` 跨 space 搬移的規則 |
| `src-tauri/src/state.rs` | 修改(Task 2) | `AppState::sync` 改成 `runtime::SyncRuntime` |
| `src-tauri/src/lib.rs` | 修改(Task 2、3、4) | 註冊 commands;`WindowEvent::Focused` → `engine::window_focused` |
| `src-tauri/src/sync/rotation.rs` | 新增(Task 3) | 更換同步碼 1–7 步、取消、中斷接續、其他電腦重新加入 |
| `src-tauri/src/sync/mod.rs` | 修改(Task 1、2、3) | 註冊模組、更新說明 |
| `src/bindings/*.ts` | 新增 / 修改(Task 2、4,ts-rs 產生) | `SyncOverview` 等狀態 DTO、`RotationStep`、`NewSpaceGroup`;`DuplicateAlias` 多了欄位說明 |

Task 依序執行:Task 2 的啟動流程用到 Task 1 的 `upgrade::shell_state`;Task 3 的 commands 與啟動補完寫在 Task 2 的
`engine.rs` 上;Task 4 的 command 也加在 `engine.rs`。

四個 task 與最終修正都已在 repo 執行(Task 1 `8ffd36f`、`f95fed6`、`1ee11de`;Task 2 `66cc73f`;Task 3 `7ce26a2`、`8a8c961`…
`6ca1323`;Task 4 `e950c05`、`c89aa46`、`e164a3e`;最終修正 `b675943`…`d469bc0`),之後(兩個 keychain 測試都略過)是 `740 passed`。Task 2–4 的程式碼執行之前都在 repo `1ee11de` 的拷貝上逐 task 重播過:每個 task 先確認新
測試編譯失敗,再確認 `cargo test` 全綠(701 → 656 → 669 → 671;Task 2 移除 v1 引擎的測試,數量因此下降),最後的樹與參考實作逐位元組
相同;前端在 Task 2 之後以 `tsc --noEmit` 確認仍能編譯。實際執行後:Task 3 之後 696、Task 4 之後 708、最終修正之後 740。

---

### Task 1: 從 v1 升級(spec §7.6)

> **已執行**(repo `8ffd36f`;兩輪審查後的修正 `f95fed6`、`1ee11de`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的地方見 Global Constraints 的「Task 1 實際的做法」。第一輪之後是 `683 passed`,第二輪之後(兩個 keychain 測試都略過)是 `701 passed`(`sync::upgrade` 38、`sync::account` 25、`sync::state_v2` 14);下面 Step 裡的數字是原本計畫的。

v2 app 啟動時讀到已加入的 `version: 1` 狀態檔、keychain 有同步碼 → 背景執行緒的第一輪先做升級(`round::sync_once` 在
`SyncCore.legacy` 存在時呼叫 `upgrade::upgrade_v1`)。可重複執行、結果相同,兩台同時升級也安全:帳戶與 space0 都由同步碼
推導,space0 的 `space` payload 是確定值;帳戶上已經有 space0 就沿用,已被刪除就不重建。v1 快取的 host 記錄原封不動
(含 version、時間戳、device_id、tombstone)搬進 space0、全部標 dirty;原本已同步的記錄記在 `republish`,輸給較新的遠端
版本時不算衝突。檔案照 §4.3 的順序:先寫好 space0 檔(`hosts.config` 去掉含 `Include` 或帶引號 keyword 的區塊)→ 另存
不能同步的區塊並 Include → 主 config 的 Include 換成新清單 → 備份並移除 `hosts.config`;最後備份 v1 狀態檔、寫成 v2。
升級失敗時什麼都不寫,下一輪再試。

**Files:**
- Create: `src-tauri/src/sync/upgrade.rs`
- Modify: `src-tauri/src/sync/round.rs`(`sync_once` 開頭的 v1 升級)
- Modify: `src-tauri/src/sync/mod.rs`

**Interfaces:**
- Consumes(B3a):`account::{put_new_space, selected_ids}`、`files::write_include`、`merge::{merge_account, plan_device,
  put_account_record, selected_include_tokens, space_deleted_by, space_entry}`、`runtime::save_core`、`SyncCore.legacy`、
  `round::sync_once`、`testkit::*`、`FakeRelay`。
- Consumes(B2):`crypto::{derive_keys, derive_account, derive_space0}`;`record::{record_key, HostPayload, LocalRecord,
  MetaPayload, RecordKind, SpacePayload, ACCOUNT_META_ID, SPACE0_NAME}`(`SpacePayload::space0()`);
  `hosts_file::{managed_path, forbidden_directive, validate_host_text, is_our_include_token}`;
  `space_files::{add_space_file, space_file_path, space_file_name}`;`state::{SyncState, MNEMONIC_ACCOUNT}`;
  `state_v2::{back_up_legacy, FreezeInfo, SyncNotice}`;`fsutil::{atomic_write, backup}`。
- Produces:`pub const KEPT_FILE: &str = "sshelter-v1-kept.config"`、`pub const KEPT_INCLUDE: &str = "~/.ssh/sshelter-v1-kept.config"`;
  `pub fn shell_state(v1: &LegacyState) -> SyncStateV2`(升級前給 UI 看的 v2 外殼:沿用 v1 的裝置身分與 relay);
  `pub fn upgrade_v1(env: &SyncEnv, v1: &LegacyState) -> Result<bool, AppError>`(doc 還沒載入 → `Ok(false)`、什麼都不做);
  `round::sync_once` 在 `SyncCore.legacy` 存在時先升級(失敗 → `last_error`、下一輪再試)。

- [ ] **Step 1: 寫失敗的測試:`src-tauri/src/sync/upgrade.rs`**

測試以 v1 的狀態與 `hosts.config` 布置裝置(`legacy_device`),驗證兩台同時升級、重跑、失敗不動 v1、space0 已被改名 / 刪除。

建立 `src-tauri/src/sync/upgrade.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! 從 v1 升級(spec §7.6):v2 app 啟動時讀到已加入的 `version: 1` 狀態檔、keychain 有同步碼 → 背景執行緒做一次
//! 升級。可重複執行、結果相同,兩台同時升級也安全:帳戶與 space0 都由同步碼推導(同一個 chain、同一把金鑰),
//! space0 的 `space` payload 是確定值;帳戶上已經有 space0 就沿用(可能已被改名),不再寫一份蓋掉它。升級失敗時
//! v1 狀態檔與 `hosts.config` 原封不動,下一輪再試。

use std::collections::BTreeSet;

use crate::config::model::Item;
use crate::config::parser::parse_file;
use crate::config::serialize::serialize_items;
use crate::error::AppError;
use crate::fsutil;
use crate::sync::account::{put_new_space, selected_ids};
use crate::sync::crypto;
use crate::sync::env::SyncEnv;
use crate::sync::files::write_include;
use crate::sync::hosts_file::{self, forbidden_directive, validate_host_text};
use crate::sync::merge::{merge_account, plan_device, put_account_record, selected_include_tokens, space_deleted_by, space_entry};
use crate::sync::record::{
    record_key, HostPayload, LocalRecord, MetaPayload, RecordKind, SpacePayload, ACCOUNT_META_ID, SPACE0_NAME,
};
use crate::sync::runtime::save_core;
use crate::sync::space_files::{self, space_file_name};
use crate::sync::state::{SyncState as LegacyState, MNEMONIC_ACCOUNT};
use crate::sync::state_v2::{self, AccountState, FreezeInfo, SpaceState, SyncNotice, SyncStateV2};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::env::Keychain;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::hosts_file::blocks_of;
    use crate::sync::merge::space_entries;
    use crate::sync::record::{HostPayload, Record};
    use crate::sync::round::sync_once;
    use crate::sync::round::tests::settle;
    use crate::sync::spaces::{delete_space, rename_space};
    use crate::sync::state::SyncState;
    use crate::sync::testkit::{TestClock, TestDevice, RELAY_URL};
    use std::sync::Arc;

    const WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art";

    /// v1 的狀態檔(`version: 1`)。
    fn write_v1(path: &std::path::Path, v1: &SyncState) {
        std::fs::create_dir_all(path.parent().unwrap()).unwrap();
        std::fs::write(path, serde_json::to_vec_pretty(v1).unwrap()).unwrap();
    }

    const MAIN: &str = "# main\nInclude ~/.ssh/sshelter/hosts.config\nHost local\n";

    /// 一台已加入 v1 chain 的裝置:`hosts.config` 的每個區塊都是 v1 快取裡已同步的記錄(兩台的 metadata 相同,
    /// 就像經過 v1 同步),狀態檔是 v1,keychain 有同步碼,core 等著升級。
    fn v1_device(name: &str, relay: &Arc<FakeRelay>, clock: &Arc<TestClock>, hosts: &str) -> (TestDevice, SyncState) {
        let d = TestDevice::with_main_config(name, relay, clock, MAIN);
        let dir = d.ssh_dir().join("sshelter");
        std::fs::create_dir_all(&dir).unwrap();
        std::fs::write(dir.join("hosts.config"), hosts).unwrap();
        d.reload();
        d.keychain.set(MNEMONIC_ACCOUNT, WORDS).unwrap();
        let mut v1 = SyncState::fresh(name).unwrap();
        v1.device_id = format!("{name:0<32}");
        v1.relay_url = RELAY_URL.to_string();
        v1.chain_id = Some(crypto::derive_keys(WORDS).unwrap().chain_id);
        v1.baseline_established = true;
        for (i, block) in blocks_of(&parse_file(hosts).0).into_iter().enumerate() {
            let record = Record {
                kind: RecordKind::Host,
                id: block.alias.clone(),
                version: 1,
                updated_at_ms: 1_600_000_000_000 + i as u64,
                device_id: "v1-origin".into(),
                deleted: false,
                payload: serde_json::to_value(HostPayload { schema: 1, text: block.text }).unwrap(),
            };
            v1.records.insert(record_key(RecordKind::Host, &block.alias), LocalRecord { record, seq: i as u64 + 1, dirty: false });
        }
        write_v1(&d.env().state_path, &v1);
        {
            let mut core = d.runtime.core.lock().unwrap();
            core.state = Some(shell_state(&v1));
            core.legacy = Some(v1.clone());
        }
        (d, v1)
    }

    fn space0() -> String {
        crypto::derive_space0(WORDS).unwrap().chain_id
    }

    #[test]
    fn an_upgrade_moves_the_v1_hosts_into_space0_and_keeps_include_blocks_local() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let hosts = "Host web\n  HostName 10.0.0.1\nHost jump\n  Include ~/.ssh/jump.config\nHost db\n";
        let (d, v1) = v1_device("a", &relay, &clock, hosts);
        assert!(upgrade_v1(&d.env(), &v1).unwrap());
        let id = space0();
        let file = format!("synced-{}.config", &id[..8]);
        assert_eq!(d.read(&d.ssh_dir().join("sshelter").join(&file)), "Host web\n  HostName 10.0.0.1\nHost db\n");
        assert_eq!(d.read(&d.ssh_dir().join(KEPT_FILE)), "Host jump\n  Include ~/.ssh/jump.config\n");
        assert_eq!(d.main_config(), format!("# main\nInclude ~/.ssh/sshelter/{file}\nInclude {KEPT_INCLUDE}\nHost local\n"));
        assert!(!hosts_file::managed_path(&d.ssh_dir()).exists(), "hosts.config is backed up and removed");
        // 狀態檔已是 v2;v1 的原檔另存。
        let env = d.env();
        assert!(matches!(state_v2::load(&env.state_path).unwrap(), state_v2::LoadedState::Current(_)));
        let backup = env.state_path.with_file_name(state_v2::LEGACY_BACKUP_FILE);
        assert!(matches!(state_v2::load(&backup).unwrap(), state_v2::LoadedState::Legacy(_)));
        let s = d.state();
        assert_eq!(s.legacy_v1_backup.as_deref(), Some(state_v2::LEGACY_BACKUP_FILE));
        assert_eq!(s.device_id, v1.device_id);
        let sp = &s.spaces[&id];
        assert_eq!(sp.records.keys().cloned().collect::<Vec<_>>(), vec!["host:db".to_string(), "host:web".to_string()]);
        assert!(sp.records.values().all(|l| l.dirty && l.record.device_id == "v1-origin"), "v1 metadata is kept");
        assert_eq!(sp.republish.len(), 2);
        let entries = space_entries(s.account.as_ref().unwrap());
        assert_eq!(entries.iter().map(|e| (e.name.as_str(), e.created_at_ms)).collect::<Vec<_>>(), vec![("Synced", 0)]);
        assert_eq!(s.notices, vec![SyncNotice::Upgraded { kept_file: Some(d.ssh_dir().join(KEPT_FILE).to_string_lossy().into_owned()), kept_hosts: vec!["jump".into()] }]);
        assert!(d.runtime.core.lock().unwrap().legacy.is_none());
        // 一般輪次以 LWW 上傳。
        settle(&d);
        assert_eq!(relay.rows(&id).len(), 2);
    }

    #[test]
    fn two_devices_upgrading_at_once_end_up_in_the_same_space0_without_conflicts() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let hosts = "Host web\n  HostName 10.0.0.1\nHost db\n";
        let (a, v1a) = v1_device("a", &relay, &clock, hosts);
        let (b, v1b) = v1_device("b", &relay, &clock, hosts);
        assert!(upgrade_v1(&a.env(), &v1a).unwrap());
        assert!(upgrade_v1(&b.env(), &v1b).unwrap());
        for _ in 0..2 {
            settle(&a);
            settle(&b);
        }
        let id = space0();
        assert!(a.state().spaces.contains_key(&id) && b.state().spaces.contains_key(&id));
        assert_eq!(space_entries(a.state().account.as_ref().unwrap()).len(), 1, "one space0, not two");
        assert!(a.events.conflicts.lock().unwrap().is_empty() && b.events.conflicts.lock().unwrap().is_empty());
        assert!(a.state().spaces[&id].records.values().all(|l| !l.dirty));
        assert!(b.state().spaces[&id].records.values().all(|l| !l.dirty));
        // 之後照常同步。
        a.save_in_app(&a.space_path(&id), "Host web\n  HostName 10.0.0.2\nHost db\n");
        settle(&a);
        settle(&b);
        assert_eq!(b.read(&b.space_path(&id)), "Host web\n  HostName 10.0.0.2\nHost db\n");
    }

    #[test]
    fn an_upgrade_can_run_again_with_the_same_result() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let hosts = "Host web\nHost jump\n  Include ~/.ssh/jump.config\n";
        let (d, v1) = v1_device("a", &relay, &clock, hosts);
        assert!(upgrade_v1(&d.env(), &v1).unwrap());
        let (main, kept) = (d.main_config(), d.read(&d.ssh_dir().join(KEPT_FILE)));
        let space_file = d.read(&d.space_path(&space0()));
        // 狀態還沒寫成 v2 就中斷了:下次啟動又讀到 v1 狀態,`hosts.config` 已經移除。
        write_v1(&d.env().state_path, &v1);
        d.runtime.core.lock().unwrap().legacy = Some(v1.clone());
        assert!(upgrade_v1(&d.env(), &v1).unwrap());
        assert_eq!(d.main_config(), main);
        assert_eq!(d.read(&d.ssh_dir().join(KEPT_FILE)), kept);
        assert_eq!(d.read(&d.space_path(&space0())), space_file);
        assert_eq!(d.state().spaces.keys().cloned().collect::<Vec<_>>(), vec![space0()]);
        assert_eq!(space_entries(d.state().account.as_ref().unwrap()).len(), 1);
    }

    #[test]
    fn a_failed_upgrade_leaves_v1_untouched_and_is_retried() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let (d, _) = v1_device("a", &relay, &clock, "Host web\n");
        relay.set_offline(true);
        assert!(sync_once(&d.env()).is_err());
        assert!(d.state().last_error.unwrap().starts_with("SSHelter could not upgrade this device's sync yet"));
        assert!(hosts_file::managed_path(&d.ssh_dir()).exists());
        assert_eq!(d.main_config(), MAIN);
        assert!(matches!(state_v2::load(&d.env().state_path).unwrap(), state_v2::LoadedState::Legacy(_)));
        // 升級完成前不接受會改狀態的命令。
        assert!(crate::sync::account::create_account(&d.env(), "A").is_err());
        assert_eq!(d.keychain.entry(MNEMONIC_ACCOUNT).as_deref(), Some(WORDS), "the sync code is untouched");
        relay.set_offline(false);
        sync_once(&d.env()).unwrap();
        assert!(d.state().joined());
        settle(&d);
        assert_eq!(relay.rows(&space0()).len(), 1);
    }

    #[test]
    fn a_space0_renamed_elsewhere_is_kept_and_a_deleted_one_is_not_recreated() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let (a, v1a) = v1_device("a", &relay, &clock, "Host web\n");
        upgrade_v1(&a.env(), &v1a).unwrap();
        settle(&a);
        rename_space(&a.env(), &space0(), "Servers").unwrap();
        settle(&a);
        let (b, v1b) = v1_device("b", &relay, &clock, "Host web\n");
        upgrade_v1(&b.env(), &v1b).unwrap();
        settle(&b);
        assert_eq!(space_entries(b.state().account.as_ref().unwrap())[0].name, "Servers", "B did not rename it back");
        assert!(b.space_path(&space0()).ends_with(format!("servers-{}.config", &space0()[..8])));
        delete_space(&a.env(), &space0()).unwrap();
        settle(&a);
        let (c, v1c) = v1_device("c", &relay, &clock, "Host web\nHost mine\n");
        upgrade_v1(&c.env(), &v1c).unwrap();
        let s = c.state();
        assert!(s.spaces.is_empty(), "a deleted space0 is not recreated");
        assert_eq!(c.read(&c.ssh_dir().join(KEPT_FILE)), "Host web\nHost mine\n");
        assert_eq!(c.main_config(), format!("# main\nInclude {KEPT_INCLUDE}\nHost local\n"));
        assert!(matches!(&s.notices[..], [SyncNotice::Upgraded { kept_hosts, .. }] if kept_hosts == &vec!["mine".to_string(), "web".to_string()]));
    }
}
```

- [ ] **Step 2: 更新 `src-tauri/src/sync/mod.rs`**

把 `src-tauri/src/sync/mod.rs` 整個換成:

```rust
//! Sync chain: Brave 式免帳號端對端同步。各子模組單一責任、皆可單元測試:
//! - `crypto`: 助記詞、金鑰派生、記錄加密
//! - `record`: 記錄模型與 LWW 合併(Task 2)
//! - `hosts_file`: 受管同步檔的區塊操作(Task 3)
//! - `planner`: 本機變更偵測(Task 1)
//! - `state`: 本機同步狀態持久化(Task 4)
//! - `relay`: 中繼 HTTP client(Task 5)
//! - `reconcile`: 一輪同步的三段純函式(plan_local → pull_merge → push_dirty)
//! - `engine`: 背景同步執行緒、`SyncCore`、存檔當下規劃、套用+發布交易、Tauri commands
//! - `space_files`: Sync v2 的 space 檔命名、Include 清單順序與建立 / 移除 / 改名的順序規則
//! - `approval`: Sync v2 危險設定的核准簽章
//! - `state_v2`: Sync v2 的本機狀態(`version: 2`)與 v1 狀態檔的偵測
//! - `merge`: Sync v2 帳戶與 space 區段的本機 diff、合併、上傳(純函式)
//! - `fake_relay`(只在測試):記憶體假 relay
//! - `dto`: Sync v2 給前端的事件與狀態形狀
//! - `runtime`: Sync v2 的 `SyncCore` 與局部提交
//! - `env`: Sync v2 引擎與外界的邊界(keychain、relay、事件、時鐘)
//! - `files`: Sync v2 space 檔的準備、讀取、套用 + 發布交易、存檔 hook
//! - `testkit`(只在測試):測試裝置
//! - `account`: Sync v2 帳戶生命週期與 relay 設定
//! - `spaces`: Sync v2 space 操作與核准
//! - `round`: Sync v2 的一輪同步
//! - `upgrade`: 從 v1 升級

pub mod account;
pub mod approval;
pub mod crypto;
pub mod dto;
pub mod engine;
pub mod env;
#[cfg(test)]
pub mod fake_relay;
pub mod files;
pub mod hosts_file;
pub mod merge;
pub mod migrate;
pub mod planner;
pub mod reconcile;
pub mod record;
pub mod relay;
pub mod round;
pub mod runtime;
pub mod space_files;
pub mod spaces;
pub mod state;
pub mod state_v2;
#[cfg(test)]
pub mod testkit;
pub mod upgrade;
```

- [ ] **Step 3: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0425]: cannot find value `KEPT_FILE` in this scope
--> src/sync/upgrade.rs:100:45
```

- [ ] **Step 4: 實作 `src-tauri/src/sync/upgrade.rs`**

重點:`PUT` 兩條 chain 之後先讀帳戶現況(可能已被別台升級、改名、刪除 space0,或已更換同步碼 —— `PUT` 回 200 不代表可寫);檔案操作全在 doc 鎖內、照 §4.3 的順序;狀態最後才換,core 換 generation。

`src-tauri/src/sync/upgrade.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// v1 同步檔裡不能進 space0 的區塊(含 `Include` 或帶引號的 keyword)另存的本機檔案(`~/.ssh/` 底下,不在
/// `~/.ssh/sshelter/` 裡 —— 不是「我們的」Include token,之後也不會被 `ensure_include` 收走)。
pub const KEPT_FILE: &str = "sshelter-v1-kept.config";
pub const KEPT_INCLUDE: &str = "~/.ssh/sshelter-v1-kept.config";

/// 升級前,core 裡給 UI 看的 v2 外殼:沿用 v1 的裝置身分與 relay,還沒有帳戶。
pub fn shell_state(v1: &LegacyState) -> SyncStateV2 {
    let mut s = SyncStateV2::fresh(&v1.device_name).expect("a fresh state only draws a device id");
    s.device_id = v1.device_id.clone();
    s.relay_url = v1.relay_url.clone();
    s.phrase_cleanup_pending = v1.phrase_cleanup_pending;
    s
}

/// 文字裡每個 Host 區塊的第一個 pattern。
fn host_aliases(text: &str) -> Vec<String> {
    parse_file(text).0.iter().filter_map(|i| match i {
        Item::Host(h) => h.patterns.first().cloned(),
        _ => None,
    }).collect()
}

/// v1 同步檔的內容拆成兩份:進 space0 的(其他一切)與留在本機的(含 `Include` 或帶引號 keyword 的 top-level
/// 項目,spec §7.6 第 3 步)。回傳(space0 的 bytes, 留在本機的 bytes, 留下的主機 alias)。
fn split_v1_file(text: &str) -> (String, String, Vec<String>) {
    let (items, trailing_newline) = parse_file(text);
    let (mut synced, mut kept) = (Vec::new(), Vec::new());
    for item in items {
        if forbidden_directive(std::slice::from_ref(&item)).is_some() {
            kept.push(item);
        } else {
            synced.push(item);
        }
    }
    let kept_text = if kept.is_empty() { String::new() } else { serialize_items(&kept, true) };
    let kept_hosts = host_aliases(&kept_text);
    (serialize_items(&synced, trailing_newline || synced.is_empty()), kept_text, kept_hosts)
}

/// 主 config 裡、我們那一行 Include 之後,放一行 `Include ~/.ssh/sshelter-v1-kept.config`(已經有就不動)。
fn ensure_kept_include(items: &mut Vec<Item>) -> bool {
    let present = items.iter().any(|i| {
        matches!(i, Item::Directive(d) if d.key == "include" && d.enabled && d.value.split_whitespace().any(|t| t == KEPT_INCLUDE))
    });
    if present {
        return false;
    }
    let ours = items.iter().position(|i| {
        matches!(i, Item::Directive(d) if d.key == "include" && d.enabled && d.value.split_whitespace().any(hosts_file::is_our_include_token))
    });
    let at = match ours {
        Some(i) => i + 1,
        None => items.iter().position(|i| !matches!(i, Item::Blank(_) | Item::Comment(_))).unwrap_or(items.len()),
    };
    items.insert(at, Item::Directive(crate::config::model::Directive::new("Include", KEPT_INCLUDE, "")));
    true
}

/// 升級本體(spec §7.6 的 1–5 步)。成功時 core 換成 v2 狀態(`legacy` 清掉、換 generation)、狀態檔寫成 v2(v1 備份
/// 為 `sync-state.v1-backup.json`),回傳 true。doc 還沒載入 → 什麼都不做、回傳 false(config 載入時會喚醒下一輪)。
/// 網路在鎖外;檔案在 doc 鎖內、照 §4.3 的順序。
pub fn upgrade_v1(env: &SyncEnv, v1: &LegacyState) -> Result<bool, AppError> {
    if env.doc.lock().unwrap().is_none() {
        return Ok(false);
    }
    let v1_chain = v1.chain_id.clone().ok_or_else(|| AppError::Other("the v1 sync state has not joined a chain".to_string()))?;
    let words = env
        .keychain
        .get(MNEMONIC_ACCOUNT)?
        .ok_or_else(|| AppError::Other("the sync code is missing from the keychain; leave and join again".to_string()))?;
    if crypto::derive_keys(&words).map(|k| k.chain_id).ok().as_deref() != Some(v1_chain.as_str()) {
        return Err(AppError::Other(
            "the sync code in the keychain belongs to a different sync chain; leave and join again".to_string(),
        ));
    }
    let account_keys = crypto::derive_account(&words)?;
    let space0 = crypto::derive_space0(&words)?;

    // 1. 兩條 chain(冪等),再讀帳戶現況:別台可能已經升級(甚至改名、刪掉 space0,或已更換同步碼)。`PUT` 回 200
    //    只表示 chain 已存在(可能已凍結),不代表能寫入:能不能寫以讀到的更換標記與之後 push 的結果為準。
    let relay = env.relay(&v1.relay_url)?;
    relay.create_chain(&account_keys.chain_id, &account_keys.auth_token)?;
    relay.create_chain(&space0.chain_id, &space0.auth_token)?;
    let pulled = relay.pull(&account_keys.chain_id, &account_keys.auth_token, 0)?;
    let merged = merge_account(&AccountState::new(&account_keys.chain_id), &account_keys, &pulled);
    let now = env.now();
    let mut account = merged.section;
    account.baseline_established = true;
    if !merged.markers.is_empty() {
        account.frozen = Some(FreezeInfo { detected_at_ms: now, markers: merged.markers });
    }
    if !account.records.contains_key(&record_key(RecordKind::Meta, ACCOUNT_META_ID)) {
        put_account_record(
            &mut account,
            RecordKind::Meta,
            ACCOUNT_META_ID,
            serde_json::to_value(MetaPayload::account(env!("CARGO_PKG_VERSION"))).expect("MetaPayload serializes"),
            false,
            &v1.device_id,
            now,
        );
    }

    // 2. space0 的記錄:帳戶上還沒有才寫(spec §5.2 的確定值);已被刪除就不重建,v1 的主機改留在本機。
    let space0_gone = space_deleted_by(&account, &account_keys, &space0.chain_id).is_some();
    if space_entry(&account, &space0.chain_id).is_none() && !space0_gone {
        put_new_space(&mut account, &account_keys, &space0, &SpacePayload::space0(), &v1.device_id, now)?;
    }
    let slug = space_entry(&account, &space0.chain_id).map(|e| e.slug).unwrap_or_else(|| SPACE0_NAME.to_lowercase());
    let file_name = space_file_name(&slug, &space0.chain_id)?;

    // 3. v1 快取裡的 host 記錄 → space0(保留 version、時間戳、device_id、tombstone 與 dirty;全部標 dirty 以 LWW
    //    上傳)。含 `Include` 的區塊不進 space0;v1 `sealed` 裡的 key / password 捨棄。
    let mut space = SpaceState::new(&file_name);
    space.baseline_established = true; // 一般輪次以 LWW 合併,不是基線輪(spec §7.6)
    if !space0_gone {
        for local in v1.records.values().filter(|l| l.record.kind == RecordKind::Host) {
            let record = &local.record;
            let allowed = record.deleted
                || serde_json::from_value::<HostPayload>(record.payload.clone())
                    .is_ok_and(|p| validate_host_text(&record.id, &p.text).is_ok());
            if !allowed {
                continue;
            }
            let key = record_key(RecordKind::Host, &record.id);
            if !local.dirty {
                space.republish.insert(key.clone());
            }
            space.records.insert(key, LocalRecord { record: record.clone(), seq: 0, dirty: true });
        }
    }

    // 4. 檔案(doc 鎖內,§4.3 的順序):space0 檔 = `hosts.config` 去掉不能同步的區塊(先寫好)→ 留在本機的區塊另存並
    //    Include → 主 config 的 Include 換成新清單(v1 的 token 一併收掉)→ 備份並移除 `hosts.config`。
    let hosts_path = hosts_file::managed_path(&env.ssh_dir);
    let kept_path = env.ssh_dir.join(KEPT_FILE);
    let (kept_hosts, kept_file) = {
        let mut doc_lock = env.doc.lock().unwrap();
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        let mut backed_up = env.backed_up.lock().unwrap();
        let retention = env.retention();
        let v1_text = match std::fs::read_to_string(&hosts_path) {
            Ok(t) => Some(t),
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => None,
            Err(e) => return Err(AppError::Io(e)),
        };
        let (space_text, kept_text, hosts) = match &v1_text {
            // space0 已在別台刪除:整份 v1 內容都留在本機。
            Some(t) if space0_gone => (String::new(), t.clone(), host_aliases(t)),
            Some(t) => split_v1_file(t),
            None => (String::new(), String::new(), Vec::new()),
        };
        let mut kept_file = None;
        if !kept_text.is_empty() {
            if kept_path.try_exists()? {
                fsutil::backup(&kept_path)?;
            }
            fsutil::atomic_write(&kept_path, kept_text.as_bytes(), 0o600)?;
            kept_file = Some(kept_path.to_string_lossy().into_owned());
        } else if kept_path.try_exists()? {
            // 前一次中斷的升級已經另存過(`hosts.config` 已移除):沿用。
            kept_file = Some(kept_path.to_string_lossy().into_owned());
        }
        let mut spaces = std::collections::BTreeMap::new();
        if !space0_gone {
            spaces.insert(space0.chain_id.clone(), space.clone());
        }
        let tokens = selected_include_tokens(Some(&account), &spaces)?;
        let write_lists = |doc: &mut crate::config::model::SshConfigDoc, backed_up: &mut std::collections::HashSet<std::path::PathBuf>| -> Result<(), AppError> {
            write_include(doc, backed_up, retention, &tokens)?;
            if kept_file.is_some() && ensure_kept_include(&mut doc.files[0].items) {
                crate::config::commands::persist_file(doc, 0, backed_up, retention)?;
            }
            Ok(())
        };
        if space0_gone {
            write_lists(doc, &mut backed_up)?;
        } else {
            // `hosts.config` 還在就以它為準(重跑時結果相同);已經移除(前一次升級中斷在後面)就沿用 space0 檔。
            let path = space_files::space_file_path(&env.ssh_dir, &file_name)?;
            if v1_text.is_some() && path.try_exists()? {
                fsutil::backup(&path)?;
            }
            let initial = v1_text.as_ref().map(|_| space_text.as_bytes());
            space_files::add_space_file(&env.ssh_dir, &file_name, initial, || write_lists(doc, &mut backed_up))?;
        }
        if v1_text.is_some() {
            fsutil::backup(&hosts_path)?;
            std::fs::remove_file(&hosts_path)?;
        }
        drop(backed_up);
        let main = doc.files[0].path.clone();
        *doc_lock = Some(env.load_doc(&main)?);
        (hosts, kept_file)
    };

    // 5. 狀態:v1 狀態檔先備份,再寫成 v2;core 換成 v2 狀態與帳戶金鑰。
    let mut state = shell_state(v1);
    let backup = state_v2::back_up_legacy(&env.state_path)?;
    state.legacy_v1_backup = Some(backup);
    if !space0_gone {
        state.spaces.insert(space0.chain_id.clone(), space);
    }
    let ids = selected_ids(&state);
    plan_device(&mut account, &v1.device_id, &v1.device_name, env.platform, &ids, now);
    state.account = Some(account);
    let kept_hosts: Vec<String> = kept_hosts.into_iter().filter(|h| !h.is_empty()).collect::<BTreeSet<_>>().into_iter().collect();
    let notice = SyncNotice::Upgraded { kept_file, kept_hosts };
    state.notices.push(notice.clone());
    {
        let _doc = env.doc.lock().unwrap();
        let mut core = env.runtime.core.lock().unwrap();
        core.generation += 1;
        core.state = Some(state);
        core.legacy = None;
        core.account_keys = Some(account_keys);
        save_core(&mut core, &env.state_path)?;
    }
    env.events.notice(&notice);
    env.events.applied(0);
    env.events.wake();
    Ok(true)
}
```

- [ ] **Step 5: 修改 `src-tauri/src/sync/round.rs`**

`sync_once` 在任何一般輪次之前先處理 `legacy`。

`src-tauri/src/sync/round.rs`:把

```rust
    if env.runtime.syncing.swap(true, Ordering::SeqCst) {
        return Ok(()); // 已在同步中
    }
    // 先補存上次沒寫進磁碟的狀態;然後 generation / 狀態 / 金鑰一次快照(同一把鎖)。
```

換成:

```rust
    if env.runtime.syncing.swap(true, Ordering::SeqCst) {
        return Ok(()); // 已在同步中
    }
    // v1 升級先做(spec §7.6);失敗時保留 v1 狀態與檔案,錯誤顯示在狀態列(不寫狀態檔),下一輪再試(spec §9)。
    let legacy = env.runtime.core.lock().unwrap().legacy.clone();
    if let Some(v1) = legacy {
        let result = crate::sync::upgrade::upgrade_v1(env, &v1);
        env.runtime.syncing.store(false, Ordering::SeqCst);
        if let Err(e) = &result {
            if let Some(s) = env.runtime.core.lock().unwrap().state.as_mut() {
                s.last_error = Some(format!("SSHelter could not upgrade this device's sync yet: {e}"));
            }
        }
        env.events.status();
        return result.map(|_| ());
    }
    // 先補存上次沒寫進磁碟的狀態;然後 generation / 狀態 / 金鑰一次快照(同一把鎖)。
```

- [ ] **Step 6: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 662 passed; 0 failed`(task 開始前 657)。數量有變的模組:`sync::upgrade` 5(新)。非測試建置會有一長串 `dead_code` 類的 warning(`is never used` 之類):B2 留下的,加上本計畫新增、要到 B3b Task 2 才接上的項目;B3b Task 2 之後只剩既有的 `set_host_enabled`。這是預期的,不要加 `#[allow(dead_code)]`;不得有其他種類的 warning。

- [ ] **Step 7: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/upgrade.rs
git add src-tauri/src/sync/round.rs
git add src-tauri/src/sync/mod.rs
git commit -m "feat(sync): upgrade a v1 sync state to the v2 account"
```

---

### Task 2: 接上 app:Tauri 外殼、狀態 DTO、commands 與事件、搬進 space;退役 v1 引擎

> **已執行**(repo `66cc73f`,與本節的程式碼一致;審查通過,Minor 都留到最終修正)。下面保留原本的步驟作為紀錄,不要再執行。執行後(兩個 keychain 測試都略過)是 `656 passed`。

這是切換的 task:app 從 v1 引擎改跑 v2,v1 的引擎、合併規則、relay client、狀態存取一起移除(只留 v1 升級要讀的
型別)。`engine.rs` 整個改寫成 Tauri 外殼:行程間同步鎖(同 v1)、啟動(`startup` 純函式:v2 → 推導帳戶金鑰;已加入的
v1 → 排給背景執行緒升級;讀不懂的狀態檔搬到旁邊)、背景執行緒(`round::sync_once` + `round::next_delay`)、存檔 hook
(`note_file_written` → `files::note_written`)、`window_focused`、`TauriEvents`(`sync://status` 帶完整的
`SyncOverview`)與全部 commands。`dto.rs` 改寫成完整的狀態 DTO。`migrate.rs` 改寫:搬移精靈「搬進一個 space」與
sidebar 拖曳(`config_move_host`)共用同一套拒絕規則,跨檔同名主機改看所有勾選的 space 檔。這個 task 之後 Sync pane
的 v1 命令不存在(B4 換掉 UI),其餘功能照常。

**Files:**
- Modify(整個改寫):`src-tauri/src/sync/engine.rs`
- Modify(整個改寫):`src-tauri/src/sync/dto.rs`
- Modify(整個改寫):`src-tauri/src/sync/migrate.rs`
- Modify(整個改寫):`src-tauri/src/sync/reconcile.rs`
- Modify: `src-tauri/src/sync/state_v2.rs`、`src-tauri/src/sync/relay.rs`、`src-tauri/src/sync/state.rs`、
  `src-tauri/src/sync/hosts_file.rs`、`src-tauri/src/sync/crypto.rs`(錯誤訊息改稱「sync code」)、`src-tauri/src/sync/mod.rs`
- Modify: `src-tauri/src/config/commands.rs`(`config_load`、`load_wakes_sync`、`config_move_host`)
- Modify: `src-tauri/src/state.rs`、`src-tauri/src/lib.rs`
- Generated: `src/bindings/SyncOverview.ts`、`SyncSpaceView.ts`、`SyncDeviceView.ts`、`SyncRelayView.ts`、
  `SyncFrozenView.ts`、`SyncRotationView.ts`、`RotationStep.ts`、`PendingApprovalView.ts`、`ReviewedVersion.ts`、
  `ReviewOutcome.ts`;修改 `DuplicateAlias.ts`
  (欄位說明)。`SyncStatus.ts`、`SyncDevice.ts` 不再產生,但**保留**(現有前端還 import 它們)。

**Interfaces:**
- Consumes(B3a、Task 1):`round::{sync_once, next_delay}`、`account::*`、`spaces::*`(`approve` / `reject` 收 `(alias, digest)`、`review_digest`、
  回 `Reviewed`)、`files::{note_written, space_path}`、
  `runtime::{SyncRuntime, save_core, mutate}`、`env::{SyncEnv, OsKeychain, HttpRelays, SystemClock, Keychain, Clock,
  SyncEvents}`、`merge::{devices, space_deleted_by, space_entries, space_entry}`、`upgrade::shell_state`、
  `space_files::stray_space_files`、`hosts_file::{blocks_of, first_alias, forbidden_directive, is_syncable_block}`、
  `approval::ApprovalSignature`、`state_v2::{load, LoadedState}`、`relay::{FEATURE_FREEZE, FEATURE_PULL_BATCH}`。
- Produces(`engine`):`pub fn initialize(app: &AppHandle) -> Result<(), AppError>`;`pub fn wake()`;
  `pub fn engine_active() -> bool`;`pub fn window_focused(app: &AppHandle, focused: bool)`;
  `pub fn note_file_written(path: &Path, items: &[Item])`(`persist_file` 呼叫);
  `pub(crate) fn with_env<T>(app: &AppHandle, f: impl FnOnce(&SyncEnv) -> T) -> Result<T, AppError>`;
  `pub(crate) fn join_error(e: tauri::Error) -> AppError`;`pub(crate) const ANOTHER_ENGINE_MESSAGE: &str`;
  commands(簽章見文末 B4 handoff):`sync_overview`、`sync_now`、`sync_create_account`、`sync_join_account`、
  `sync_leave_account`、`sync_show_words`、`sync_set_relay_url`、`sync_check_relay`、`sync_set_device_name`、
  `sync_forget_device`、`sync_create_space`、`sync_rename_space`、`sync_delete_space`、`sync_select_space`、
  `sync_unselect_space`、`sync_rebuild_space`、`sync_pending_approvals`、`sync_approve`、`sync_reject`、
  `sync_dismiss_notice`、`sync_move_hosts_to_space`。
- Produces(`dto`):`SyncConflict`、`ApprovalNotice`(同 B3a)、`SyncRelayView`、`SyncFrozenView`、`SyncRotationView`、
  `SyncDeviceView`、`SyncSpaceView`、`SyncOverview`、`PendingApprovalView`(帶 `digest`)、`ReviewedVersion`、`ReviewOutcome`;
  `pub fn overview(env: &SyncEnv) -> Result<SyncOverview, AppError>`;
  `pub fn pending_approvals(env: &SyncEnv) -> Result<Vec<PendingApprovalView>, AppError>`。
- Produces(`migrate`):`pub const WAIT_FOR_FIRST_SYNC: &str`;`DuplicateAlias`、`ShadowedAction`、`MigrationFailure`、
  `MigrationReport`(TS 形狀不變);`pub fn tag_for_file(path: &Path) -> String`;`pub fn refuse_wildcard(doc, alias)`、
  `pub fn refuse_forbidden(doc, alias)`、`pub fn refuse_already_synced(doc, target: &Path, alias)`、
  `pub fn refuse_while_sync_inactive(active: bool, sync: &SyncRuntime)`、`pub fn refuse_before_first_sync(sync: &SyncRuntime, space_id: &str)`
  —— 全部 `-> Result<(), AppError>`;`pub fn managed_defines(doc: &SshConfigDoc, target: &Path, name: &str) -> bool`;
  `pub fn selected_space_files(sync: &SyncRuntime, ssh_dir: &Path) -> Vec<(String, PathBuf)>`;
  `pub fn duplicate_aliases(doc: &SshConfigDoc, spaces: &[PathBuf]) -> Vec<DuplicateAlias>`;
  `pub fn resolve_shadowed(doc: &mut SshConfigDoc, alias: &str, file: &str, action: ShadowedAction, spaces: &[PathBuf]) -> Result<usize, AppError>`;
  `pub fn move_hosts_into_space(env: &SyncEnv, active: bool, aliases: Vec<String>, space_id: &str, tag_by_file: bool) -> Result<MigrationReport, AppError>`;
  commands `sync_duplicate_aliases`、`sync_resolve_shadowed`(名稱與參數不變)。
- Produces(其他):`AppState::sync: runtime::SyncRuntime`;`reconcile` 只剩 `HostEffect`、`encode`、`decode`;`crypto` 的錯誤
  訊息:`a sync code has 24 words (got N)`、`invalid sync code: …`、`cannot generate a sync code: …`。
- 移除(之後沒有任何呼叫端):`engine::{SyncCore, SyncRuntime, SyncStatus, SyncDevice, status_from,
  reset_hosts_for_rematerialize, sync_once, sync_status, sync_create_chain, sync_join_chain, sync_leave_chain}`(v1 引擎);
  `migrate::sync_migrate_hosts`;`reconcile::{Relay, Merged, Pushed, plan_local, pull_merge, push_dirty, own_device_record,
  unpushed_host_effects}`;`relay::LegacyRelayClient`;`state::{load, save, store_mnemonic, load_mnemonic, clear_mnemonic}`;
  `state_v2::{store_next_mnemonic, load_next_mnemonic, clear_next_mnemonic}`;`hosts_file::{INCLUDE_VALUE, ensure_managed_file}`。

- [ ] **Step 1: 改寫 `src-tauri/src/sync/reconcile.rs` 的測試**

v1 的合併規則與它的測試一併移除;留下 `encode` / `decode` 的往返測試。

把 `src-tauri/src/sync/reconcile.rs` 整個換成下面的內容 —— 新的 module 註解、`use` 與測試;原本的實作與測試全部移除,新的實作在後面的步驟加入:

```rust
//! 記錄的加解密編碼(spec §5.3)與遠端 host 記錄要套到檔案的效果。一輪同步的合併規則在 `merge`。

use crate::error::AppError;
use crate::sync::crypto::{self, ChainKeys, Sealed};
use crate::sync::record::{Envelope, Record, RecordKind};
use crate::sync::relay::PushItem;

#[cfg(test)]
mod tests {
    use super::*;

    fn host(alias: &str) -> Record {
        Record {
            kind: RecordKind::Host,
            id: alias.to_string(),
            version: 1,
            updated_at_ms: 5,
            device_id: "dev-a".into(),
            deleted: false,
            payload: serde_json::json!({ "schema": 1, "text": format!("Host {alias}\n") }),
        }
    }

    fn envelope(item: PushItem, kind: &str) -> Envelope {
        Envelope { id_hash: item.id_hash, kind: kind.to_string(), seq: 1, nonce: item.nonce, ciphertext: item.ciphertext, deleted: item.deleted }
    }

    #[test]
    fn records_round_trip_and_only_their_own_key_and_identity_open_them() {
        let keys = ChainKeys::generate().unwrap();
        let item = encode(&keys, &host("web"), 3).unwrap();
        assert_eq!((item.kind.as_str(), item.base_seq), ("host", 3));
        assert!(!item.ciphertext.contains("Host web"));
        assert_eq!(decode(&keys, &envelope(item.clone(), "host")).unwrap(), host("web"));
        assert!(decode(&ChainKeys::generate().unwrap(), &envelope(item.clone(), "host")).is_err(), "another chain's key");
        assert!(decode(&keys, &envelope(item.clone(), "meta")).is_err(), "the kind is bound to the envelope");
        assert!(decode(&keys, &envelope(item, "future")).is_err(), "unknown kinds are never decoded");
    }
}
```

- [ ] **Step 2: `src-tauri/src/sync/state_v2.rs` 的測試**

v1 偵測的測試改用 `serde_json` 直接寫出 v1 狀態檔(`state::save` 已移除)。

`src-tauri/src/sync/state_v2.rs`:把

```rust
        v1.chain_id = Some("ab".repeat(32));
        v1.cursor_seq = 9;
        crate::sync::state::save(&path, &v1).unwrap();
        match load(&path).unwrap() {
            LoadedState::Legacy(back) => assert_eq!(*back, v1),
```

換成:

```rust
        v1.chain_id = Some("ab".repeat(32));
        v1.cursor_seq = 9;
        std::fs::write(&path, serde_json::to_vec_pretty(&v1).unwrap()).unwrap();
        match load(&path).unwrap() {
            LoadedState::Legacy(back) => assert_eq!(*back, v1),
```

- [ ] **Step 3: `src-tauri/src/sync/state.rs` 的測試**

v1 狀態的 load / save 測試移除(v1 狀態只剩 `state_v2::load` 讀它)。

`src-tauri/src/sync/state.rs`:刪除從下面這段開始

```rust
    #[test]
    fn sealed_envelopes_survive_save_and_load_without_being_decoded() {
```

到下面這段為止的整段程式碼(含這兩段本身,共 56 行):

```rust
        assert_eq!(mode, 0o600);
    }

```

- [ ] **Step 4: `src-tauri/src/sync/relay.rs` 的測試**

移除 `LegacyRelayClient` 的測試;URL 規則的測試改測 `RelayClient`。

`src-tauri/src/sync/relay.rs`:刪除從下面這段開始

```rust
    #[test]
    fn create_chain_sends_bearer_and_accepts_2xx() {
```

到下面這段為止的整段程式碼(含這兩段本身,共 99 行):

```rust
        assert!(started.elapsed() < Duration::from_secs(10));
    }

```

`src-tauri/src/sync/relay.rs`:把

```rust
    #[test]
    fn rejects_relay_url_without_scheme() {
        assert!(LegacyRelayClient::new("sync.example.com", "tok").is_err());
        assert!(LegacyRelayClient::new("", "tok").is_err());
        assert!(RelayClient::new("sync.example.com").is_err());
    }

    #[test]
    fn rejects_plain_http_except_loopback() {
        // bearer token 有整條 chain 的權限:非 loopback 一律要 https。
        assert!(LegacyRelayClient::new("http://sync.example.com", "tok").is_err());
        assert!(LegacyRelayClient::new("http://10.0.0.5:8787", "tok").is_err());
        assert!(LegacyRelayClient::new("ftp://sync.example.com", "tok").is_err());
        assert!(LegacyRelayClient::new("https://sync.example.com", "tok").is_ok());
        assert!(LegacyRelayClient::new("http://127.0.0.1:8787", "tok").is_ok());
        assert!(LegacyRelayClient::new("http://localhost:8787/", "tok").is_ok());
        assert!(LegacyRelayClient::new("http://[::1]:8787", "tok").is_ok());
        assert!(RelayClient::new("http://sync.example.com").is_err());
        assert!(RelayClient::new("https://sync.example.com").is_ok());
    }

```

換成:

```rust
    #[test]
    fn rejects_relay_url_without_scheme() {
        assert!(RelayClient::new("sync.example.com").is_err());
        assert!(RelayClient::new("").is_err());
    }

    #[test]
    fn rejects_plain_http_except_loopback() {
        // 權杖有整條 chain 的權限:非 loopback 一律要 https。
        assert!(RelayClient::new("http://sync.example.com").is_err());
        assert!(RelayClient::new("http://10.0.0.5:8787").is_err());
        assert!(RelayClient::new("ftp://sync.example.com").is_err());
        assert!(RelayClient::new("https://sync.example.com").is_ok());
        assert!(RelayClient::new("http://127.0.0.1:8787").is_ok());
        assert!(RelayClient::new("http://localhost:8787/").is_ok());
        assert!(RelayClient::new("http://[::1]:8787").is_ok());
    }

```

- [ ] **Step 5: `src-tauri/src/sync/hosts_file.rs` 的測試**

v1 的 `INCLUDE_VALUE` 改成測試裡的常數;`ensure_managed_file` 的測試移除。

`src-tauri/src/sync/hosts_file.rs`:把

```rust
    use super::*;

    /// v1 引擎傳給 `ensure_include` 的清單。
    fn v1() -> Vec<String> {
```

換成:

```rust
    use super::*;

    /// v1 寫進主 config 的 Include 值(v1 的同步檔)。
    const INCLUDE_VALUE: &str = "~/.ssh/sshelter/hosts.config";

    /// v1 引擎傳給 `ensure_include` 的清單。
    fn v1() -> Vec<String> {
```

`src-tauri/src/sync/hosts_file.rs`:把

```rust
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

```

換成:

```rust
        let p = managed_path(Path::new("/home/f/.ssh"));
        assert_eq!(p, Path::new("/home/f/.ssh").join("sshelter").join("hosts.config"));
    }

```

- [ ] **Step 6: `src-tauri/src/sync/crypto.rs` 的測試**

錯誤訊息釘住新的文案(UI 一律稱「sync code」),也確認不會帶出使用者輸入的字。

`src-tauri/src/sync/crypto.rs`:把

```rust
        let messy = format!("  {}\n", WORDS.to_uppercase().replace(' ', "   "));
        assert_eq!(normalize_mnemonic(&messy).unwrap(), WORDS);
        assert!(normalize_mnemonic("abandon abandon").is_err());
        let bad = WORDS.replacen("art", "zzzz", 1);
        assert!(normalize_mnemonic(&bad).is_err());
        let wrong_checksum = WORDS.replacen("art", "abandon", 1);
        assert!(normalize_mnemonic(&wrong_checksum).is_err());
    }

```

換成:

```rust
        let messy = format!("  {}\n", WORDS.to_uppercase().replace(' ', "   "));
        assert_eq!(normalize_mnemonic(&messy).unwrap(), WORDS);
        // UI 一律稱「sync code」:錯誤訊息也是。
        assert_eq!(normalize_mnemonic("abandon abandon").unwrap_err().to_string(), "a sync code has 24 words (got 2)");
        let bad = WORDS.replacen("art", "zzzz", 1);
        let err = normalize_mnemonic(&bad).unwrap_err().to_string();
        assert!(err.starts_with("invalid sync code: ") && !err.contains("zzzz"), "{err}");
        let wrong_checksum = WORDS.replacen("art", "abandon", 1);
        assert!(normalize_mnemonic(&wrong_checksum).unwrap_err().to_string().starts_with("invalid sync code: "));
    }

```

- [ ] **Step 7: 改寫 `src-tauri/src/sync/dto.rs` 的測試**

把 `src-tauri/src/sync/dto.rs` 整個換成下面的內容 —— 新的 module 註解、`use` 與測試;原本的實作與測試全部移除,新的實作在後面的步驟加入:

```rust
//! Sync v2 給前端的資料形狀(ts-rs 匯出到 `src/bindings/`):事件 payload、Settings → Sync 的狀態(`SyncOverview`)
//! 與待核准清單。u64 一律以 `number` 匯出。狀態從 core 的快照組出,不持有任何鎖做 I/O。

use std::collections::BTreeMap;

use serde::{Deserialize, Serialize};

use crate::error::AppError;
use crate::sync::approval::ApprovalSignature;
use crate::sync::env::SyncEnv;
use crate::sync::files::space_path;
use crate::sync::hosts_file::blocks_of;
use crate::sync::merge::{devices, space_deleted_by, space_entries};
use crate::sync::record::RecordKind;
use crate::sync::relay::{FEATURE_FREEZE, FEATURE_PULL_BATCH};
use crate::sync::space_files::stray_space_files;
use crate::sync::spaces::review_digest;
use crate::sync::state_v2::{RotationStep, SyncNotice};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::round::tests::{pair, settle};
    use crate::sync::spaces::create_space;

    #[test]
    fn the_overview_lists_spaces_devices_and_waiting_approvals() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        a.save_in_app(&a.space_path(&personal), "Host web\n  ProxyCommand nc %h 22\n");
        settle(&a);
        settle(&b);
        let o = overview(&b.env()).unwrap();
        assert!(o.joined && !o.upgrading && o.frozen.is_none());
        assert_eq!(o.account_short.as_deref().map(str::len), Some(8));
        assert_eq!(o.devices.len(), 2);
        assert!(o.devices.iter().any(|d| d.is_this && d.name == "MacBook-B" && d.spaces == vec![personal.clone()]));
        assert_eq!(o.spaces.iter().map(|v| v.name.as_str()).collect::<Vec<_>>(), vec!["Personal", "Work"]);
        let p = &o.spaces[0];
        assert!(p.selected && p.file_path.as_deref().is_some_and(|f| f.ends_with(".config")));
        assert_eq!((p.hosts, p.approvals), (Some(0), 1), "the ProxyCommand host waits for approval");
        assert_eq!(p.synced_on.len(), 2);
        let w = o.spaces.iter().find(|v| v.id == work).unwrap();
        assert!(!w.selected && w.hosts.is_none() && w.file_name.is_none());
        assert_eq!(o.approvals_waiting, 1);
        assert!(o.relay.as_ref().is_some_and(|r| r.batch_pull && r.freeze && r.version.is_some()));
        let pending = pending_approvals(&b.env()).unwrap();
        assert_eq!(pending.len(), 1);
        assert_eq!((pending[0].alias.as_str(), pending[0].from_device.as_str()), ("web", "MacBook-A"));
        assert_eq!(pending[0].current_text, None);
        assert_eq!(pending[0].incoming.gated[0].keyword, "proxycommand");
        // 對話框顯示的版本:核准 / 拒絕時連同 alias 送回的就是它的內容指紋。
        let state = b.state();
        assert_eq!(pending[0].digest, review_digest(&state.spaces[&personal].pending_approvals["web"]));
        // 不在清單上的檔案只提示。
        std::fs::write(b.ssh_dir().join("sshelter").join("old-12345678.config"), "").unwrap();
        assert_eq!(overview(&b.env()).unwrap().stray_files, vec!["old-12345678.config".to_string()]);
    }
}
```

- [ ] **Step 8: 改寫 `src-tauri/src/sync/engine.rs` 的測試**

啟動、狀態檔讀不懂、同步鎖的測試沿用 v1 的語意(改測新的 `startup` 與 `acquire_engine_lock_with`);v1 引擎的其他測試一併移除(它們測的行為已由 B3a 的 v2 測試涵蓋)。

把 `src-tauri/src/sync/engine.rs` 整個換成下面的內容 —— 新的 module 註解、`use` 與測試;原本的實作與測試全部移除,新的實作在後面的步驟加入:

```rust
//! 同步引擎的 Tauri 外殼(Sync v2 spec §7):行程間同步鎖、啟動(載入 v2 狀態,或等待 v1 升級)、背景執行緒(輪詢
//! 間隔與 `429` 退避,spec §6.4)、存檔 hook、事件與 Tauri commands。引擎本體在 `round`、`account`、`spaces`、`upgrade`
//! (以 `SyncEnv` 注入外界、可單元測試);這裡只把 `AppHandle` 組成 `SyncEnv`。網路一律在同步執行緒或
//! `spawn_blocking` 裡;鎖順序固定:lifecycle → doc → backed_up → core。

use std::fs::{File, OpenOptions, TryLockError};
use std::path::Path;
use std::sync::atomic::{AtomicBool, Ordering};
use std::sync::mpsc::{self, Receiver, RecvTimeoutError, Sender};
use std::sync::{Mutex, OnceLock};
use std::time::Duration;

use tauri::{AppHandle, Emitter, Manager};

use crate::config::model::Item;
use crate::error::AppError;
use crate::fsutil;
use crate::state::AppState;
use crate::sync::account::{self, account_keys_from_keychain};
use crate::sync::crypto::ChainKeys;
use crate::sync::dto::{self, ApprovalNotice, PendingApprovalView, ReviewOutcome, ReviewedVersion, SyncConflict, SyncOverview};
use crate::sync::env::{Clock, HttpRelays, Keychain, OsKeychain, SyncEnv, SyncEvents, SystemClock};
use crate::sync::files;
use crate::sync::round;
use crate::sync::runtime::save_core;
use crate::sync::spaces;
use crate::sync::state::{self as v1_state, SyncState as LegacyState, MNEMONIC_ACCOUNT};
use crate::sync::state_v2::{self, LoadedState, SyncNotice, SyncStateV2};
use crate::sync::upgrade;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::testkit::MemKeychain;

    const WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art";

    #[test]
    fn startup_loads_v2_keys_and_queues_a_joined_v1_state_for_the_upgrade() {
        let keychain = MemKeychain::default();
        keychain.set(MNEMONIC_ACCOUNT, WORDS).unwrap();
        let account = crate::sync::crypto::derive_account(WORDS).unwrap();
        let mut v2 = SyncStateV2::fresh("Box").unwrap();
        v2.account = Some(crate::sync::state_v2::AccountState::new(&account.chain_id));
        let start = startup(Ok(LoadedState::Current(Box::new(v2.clone()))), &keychain, "Box").unwrap();
        assert_eq!(start.keys.unwrap().chain_id, account.chain_id);
        assert!(start.legacy.is_none() && !start.save_now);
        // keychain 裡沒有同步碼:不推導金鑰,說明放進 last_error。
        let empty = MemKeychain::default();
        let start = startup(Ok(LoadedState::Current(Box::new(v2))), &empty, "Box").unwrap();
        assert!(start.keys.is_none());
        assert_eq!(start.state.last_error.as_deref(), Some("the sync code is missing from the keychain; leave and join again"));
        // 已加入的 v1:等升級;core 放未加入的外殼(沿用裝置身分)。
        let mut v1 = LegacyState::fresh("Old").unwrap();
        v1.chain_id = Some("ab".repeat(32));
        let start = startup(Ok(LoadedState::Legacy(Box::new(v1.clone()))), &keychain, "Box").unwrap();
        assert!(start.legacy.is_some() && !start.state.joined());
        assert_eq!((start.state.device_id.as_str(), start.state.device_name.as_str()), (v1.device_id.as_str(), "Old"));
        // 沒加入的 v1:直接換成 v2,啟動時寫一次。
        v1.chain_id = None;
        let start = startup(Ok(LoadedState::Legacy(Box::new(v1))), &keychain, "Box").unwrap();
        assert!(start.legacy.is_none() && start.save_now);
        let start = startup(Err(AppError::Other("sync state is unreadable: boom".into())), &keychain, "Box").unwrap();
        assert_eq!(start.state.last_error.as_deref(), Some("sync state is unreadable: boom"));
    }

    #[test]
    fn an_unreadable_state_file_is_set_aside_instead_of_overwritten() {
        // 只用暫存目錄,絕不碰真正的 app data。
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        std::fs::write(&path, b"{ not json").unwrap();
        let error = AppError::Other("sync state is unreadable: boom".to_string());
        let (message, blocked) = unreadable_state_outcome(&path, 1234, &error);
        assert_eq!(message, "sync state is unreadable: boom; the old file was kept as sync-state.unreadable-1234.json");
        assert!(blocked.is_none(), "once it is set aside, saving a fresh state is safe");
        assert!(!path.exists(), "a fresh state saved later can no longer overwrite it");
        assert_eq!(std::fs::read(dir.path().join("sync-state.unreadable-1234.json")).unwrap(), b"{ not json");
        let (failed, blocked) = unreadable_state_outcome(&path, 1235, &error);
        assert!(failed.starts_with("sync state is unreadable: boom; could not set the old file aside: "), "got: {failed}");
        assert!(failed.ends_with("; the sync state file was left in place — restart SSHelter to retry"), "got: {failed}");
        assert_eq!(blocked.as_deref(), Some(failed.as_str()));
    }

    #[test]
    fn a_state_file_that_cannot_be_read_right_now_stays_in_place_and_blocks_saving() {
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        std::fs::write(&path, b"{ \"version\": 2 }").unwrap();
        let io = AppError::Io(std::io::Error::other("device busy"));
        let (message, blocked) = unreadable_state_outcome(&path, 1234, &io);
        assert_eq!(message, "io error: device busy; the sync state file was left in place — restart SSHelter to retry");
        assert_eq!(blocked.as_deref(), Some(message.as_str()));
        assert!(path.exists(), "a transient read error must not move a healthy state aside");
    }

    #[test]
    fn only_one_handle_can_hold_the_sync_lock() {
        let dir = tempfile::tempdir().unwrap();
        let first = acquire_engine_lock(dir.path()).expect("the first process takes the lock");
        assert!(dir.path().join("sync.lock").is_file());
        let mut pauses = 0;
        assert_eq!(
            acquire_engine_lock_with(dir.path(), ENGINE_LOCK_ATTEMPTS, || pauses += 1).unwrap_err(),
            "Sync is running in another SSHelter process — quit it to use sync here",
            "a second holder is refused while the first one lives"
        );
        assert_eq!(pauses, ENGINE_LOCK_ATTEMPTS - 1, "it retried before giving up");
        drop(first);
        assert!(acquire_engine_lock(dir.path()).is_ok(), "released once the holder goes away");
    }

    #[test]
    fn a_sync_lock_released_while_retrying_is_taken() {
        // 用明確的 `unlock()`:別的測試同時在 spawn 子行程,子行程在 exec 之前會短暫握著這個 handle 的複本。
        let dir = tempfile::tempdir().unwrap();
        let old_process = acquire_engine_lock(dir.path()).unwrap();
        let mut pauses = 0;
        let taken = acquire_engine_lock_with(dir.path(), ENGINE_LOCK_ATTEMPTS, || {
            pauses += 1;
            if pauses == 3 {
                old_process.unlock().unwrap();
            }
        });
        assert!(taken.is_ok(), "the lock is taken on the attempt after the old process let go");
        assert_eq!(pauses, 3);
    }

    #[test]
    fn a_sync_lock_that_cannot_be_opened_reports_the_error() {
        let dir = tempfile::tempdir().unwrap();
        let not_a_dir = dir.path().join("not-a-dir");
        std::fs::write(&not_a_dir, b"").unwrap();
        let message =
            acquire_engine_lock_with(&not_a_dir, ENGINE_LOCK_ATTEMPTS, || panic!("only a held lock is retried")).unwrap_err();
        assert!(message.starts_with("sync is off in this SSHelter process: the sync lock could not be taken ("), "got: {message}");
        assert!(message.ends_with("); restart SSHelter to retry"), "got: {message}");
        assert_eq!(acquire_engine_lock(&not_a_dir).unwrap_err(), message);
    }
}
```

- [ ] **Step 9: 改寫 `src-tauri/src/sync/migrate.rs` 的測試**

v1 的搬移測試改寫成搬進 space 的版本(含兩台裝置的端到端情境)。

把 `src-tauri/src/sync/migrate.rs` 整個換成下面的內容 —— 新的 module 註解、`use` 與測試;原本的實作與測試全部移除,新的實作在後面的步驟加入:

```rust
//! 主機搬進 space(spec §7.2:搬移精靈「搬進一個 space」與側邊欄拖曳,沿用 v1 的搬移管線)、跨檔案的同名主機
//! (spec §4.3:ssh 用 Include 清單中排在前面的那份)、以及**以檔案路徑定位**的處理(既有
//! `config_rename_host`/`config_remove_host` 以第一個命中為準,同名時會誤中排在前面的那份,不能用)。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};

use serde::{Deserialize, Serialize};
use tauri::{AppHandle, Manager};

use crate::sync::env::SyncEnv;

use crate::config::commands::{load_doc_migrated, move_host, persist_file, validate_host_patterns};
use crate::config::dto::parse_tags;
use crate::config::edit::{find_host_mut, set_host_patterns, set_tags};
use crate::config::include::find_host_file_index;
use crate::config::model::{Item, SshConfigDoc};
use crate::error::AppError;
use crate::state::AppState;
use crate::sync::engine::ANOTHER_ENGINE_MESSAGE;
use crate::sync::hosts_file::{first_alias, forbidden_directive, is_syncable_block};
use crate::sync::merge::space_entry;
use crate::sync::runtime::SyncRuntime;
use crate::sync::space_files::space_file_path;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::include::load_doc;
    use crate::config::serialize::serialize_items;
    use crate::sync::state_v2::{SpaceState, SyncStateV2};

    /// 主 config Include 一個 space 檔;兩邊都有 `web`,主 config 另有 `local-only`。
    fn fixture() -> (tempfile::TempDir, PathBuf, PathBuf) {
        let dir = tempfile::tempdir().unwrap();
        let space = dir.path().join("work-3fa2c1d9.config");
        std::fs::write(&space, "Host web\n  HostName 1\nHost only-synced\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {}\nHost web\n  HostName 2\nHost local-only\n", space.display())).unwrap();
        (dir, main, space)
    }

    #[test]
    fn tag_for_file_strips_extensions_and_normalizes() {
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/homelab.config")), "homelab");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config.d/Work Stuff.conf")), "work-stuff");
        assert_eq!(tag_for_file(Path::new("/h/.ssh/config")), "config");
    }

    #[test]
    fn shadowed_copies_are_every_other_definition_of_an_alias_a_space_file_wins() {
        let dir = tempfile::tempdir().unwrap();
        let (work, home) = (dir.path().join("work-3fa2c1d9.config"), dir.path().join("home-8b01e4aa.config"));
        std::fs::write(&work, "Host web\n  HostName 1\n").unwrap();
        std::fs::write(&home, "Host web\n  HostName 2\nHost db\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {} {}\nHost web\nHost db\nHost lone\n", home.display(), work.display())).unwrap();
        let doc = load_doc(&main).unwrap();
        // Include 順序:home 在 work 前面 —— home 的 web 勝出,work 與主 config 的被遮蔽。
        let dups = duplicate_aliases(&doc, &[home.clone(), work.clone()]);
        let pairs: Vec<(String, String)> = dups.iter().map(|d| (d.alias.clone(), d.local_file.clone())).collect();
        assert!(pairs.contains(&("web".into(), work.to_string_lossy().into_owned())));
        assert!(pairs.contains(&("web".into(), main.to_string_lossy().into_owned())));
        assert!(pairs.contains(&("db".into(), main.to_string_lossy().into_owned())));
        assert_eq!(dups.len(), 3);
    }

    #[test]
    fn resolving_a_shadowed_host_never_touches_the_copy_ssh_uses() {
        let (_dir, main, space) = fixture();
        let main_str = main.to_string_lossy().into_owned();
        let spaces = vec![space.clone()];
        let mut doc = load_doc(&main).unwrap();
        let idx = resolve_shadowed(&mut doc, "web", &main_str, ShadowedAction::Rename, &spaces).unwrap();
        assert_eq!(idx, 0);
        let main_text = serialize_items(&doc.files[0].items, true);
        assert!(main_text.contains("Host web-local\n  HostName 2\n"));
        assert!(duplicate_aliases(&doc, &spaces).is_empty());
        let mut fresh = load_doc(&main).unwrap();
        assert!(resolve_shadowed(&mut fresh, "web", &space.to_string_lossy(), ShadowedAction::Remove, &spaces).is_err());
        assert_eq!(resolve_shadowed(&mut fresh, "web", &main_str, ShadowedAction::Remove, &spaces).unwrap(), 0);
        assert!(!serialize_items(&fresh.files[0].items, true).contains("Host web\n"));
        assert!(resolve_shadowed(&mut fresh, "ghost", &main_str, ShadowedAction::Remove, &spaces).is_err());
    }

    #[test]
    fn a_failed_shadow_resolution_reloads_the_doc_so_it_is_never_ahead_of_disk() {
        let (_dir, main, space) = fixture();
        let main_str = main.to_string_lossy().into_owned();
        let mut slot = Some(load_doc(&main).unwrap());
        let err = resolve_shadowed_and_persist(&mut slot, "web", &main_str, ShadowedAction::Rename, &[space], |_, _| {
            Err(AppError::Other("disk is full".to_string()))
        })
        .unwrap_err();
        assert_eq!(err.to_string(), "disk is full");
        let main_text = serialize_items(&slot.as_ref().unwrap().files[0].items, true);
        assert!(main_text.contains("Host web\n  HostName 2\n") && !main_text.contains("web-local"));
    }

    #[test]
    fn blocks_that_cannot_sync_are_refused_by_the_block_move_host_would_pick() {
        let dir = tempfile::tempdir().unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, "Host other web *.internal\n  User ops\nHost web\nHost jump\n  Include ~/.ssh/j.config\nHost db\n").unwrap();
        let doc = load_doc(&main).unwrap();
        assert!(refuse_wildcard(&doc, "web").is_err());
        assert!(refuse_wildcard(&doc, "db").is_ok());
        assert_eq!(
            refuse_forbidden(&doc, "jump").unwrap_err().to_string(),
            "host 'jump' contains an Include line, which synced hosts cannot use; keep it in a local file"
        );
        assert!(refuse_forbidden(&doc, "db").is_ok());
    }

    #[test]
    fn names_already_in_the_target_space_are_refused() {
        let dir = tempfile::tempdir().unwrap();
        let space = dir.path().join("work-3fa2c1d9.config");
        std::fs::write(&space, "Host web-1 web\nHost bastion\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {}\nHost web\nHost bastion jump\nHost db\n", space.display())).unwrap();
        let doc = load_doc(&main).unwrap();
        assert_eq!(
            refuse_already_synced(&doc, &space, "web").unwrap_err().to_string(),
            "'web' is already used by a host in that space — remove or rename it in the local host 'web' first"
        );
        assert_eq!(
            refuse_already_synced(&doc, &space, "jump").unwrap_err().to_string(),
            "'bastion' is already in that space — resolve the duplicate instead"
        );
        assert!(refuse_already_synced(&doc, &space, "db").is_ok());
    }

    #[test]
    fn moving_into_a_space_refuses_what_cannot_sync_without_halting_the_batch() {
        let dir = tempfile::tempdir().unwrap();
        let space = dir.path().join("work-3fa2c1d9.config");
        std::fs::write(&space, "").unwrap();
        let homelab = dir.path().join("homelab.config");
        std::fs::write(&homelab, "Host b\n  HostName 2\n").unwrap();
        let main = dir.path().join("config");
        std::fs::write(
            &main,
            format!("Include {}\nInclude {}\nHost a\n  HostName 1\nHost jump\n  Include ~/.ssh/j.config\n", space.display(), homelab.display()),
        )
        .unwrap();
        let mut doc = load_doc(&main).unwrap();
        let mut backed_up = std::collections::HashSet::new();
        let (report, needs_reload) = migrate_hosts(
            &mut doc,
            vec!["a".into(), "jump".into(), "b".into()],
            true,
            &space.to_string_lossy(),
            |doc, idx| persist_file(doc, idx, &mut backed_up, None),
        );
        assert!(!needs_reload);
        assert_eq!(report.moved, vec!["a".to_string(), "b".to_string()]);
        assert_eq!(report.failed.len(), 1);
        assert!(report.failed[0].error.contains("Include"));
        assert_eq!(report.tagged, 1, "only the host from an included file is tagged");
        let synced = std::fs::read_to_string(&space).unwrap();
        assert!(synced.contains("Host a\n") && synced.contains("Host b\n") && !synced.contains("jump"));
    }

    #[test]
    fn a_persist_failure_halts_the_batch_and_flags_a_reload() {
        let dir = tempfile::tempdir().unwrap();
        let space = dir.path().join("work-3fa2c1d9.config");
        std::fs::write(&space, "").unwrap();
        let main = dir.path().join("config");
        std::fs::write(&main, format!("Include {}\nHost a\nHost b\nHost c\n", space.display())).unwrap();
        let mut doc = load_doc(&main).unwrap();
        let mut backed_up = std::collections::HashSet::new();
        let mut calls = 0u32;
        let (report, needs_reload) = migrate_hosts(&mut doc, vec!["a".into(), "b".into(), "c".into()], false, &space.to_string_lossy(), |doc, idx| {
            calls += 1;
            if calls == 3 {
                Err(AppError::Other("disk is full".to_string()))
            } else {
                persist_file(doc, idx, &mut backed_up, None)
            }
        });
        assert!(needs_reload);
        assert_eq!(report.moved, vec!["a".to_string()]);
        assert_eq!(report.failed[1].error, "not attempted: an earlier move failed");
    }

    #[test]
    fn moving_local_hosts_into_a_space_syncs_them_to_the_other_devices() {
        use crate::sync::account::{create_account, join_account};
        use crate::sync::fake_relay::FakeRelay;
        use crate::sync::round::tests::settle;
        use crate::sync::spaces::select_space;
        use crate::sync::testkit::{TestClock, TestDevice};
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::with_main_config("a", &relay, &clock, "# main\nHost web\n  HostName 10.0.0.1\nHost jump\n  Include ~/.ssh/j.config\n");
        let words = create_account(&a.env(), "MacBook-A").unwrap();
        let personal = a.state().spaces.keys().next().unwrap().clone();
        assert!(move_hosts_into_space(&a.env(), false, vec!["web".into()], &personal, false).is_err(), "no engine in this process");
        let report = move_hosts_into_space(&a.env(), true, vec!["web".into(), "jump".into()], &personal, false).unwrap();
        assert_eq!(report.moved, vec!["web".to_string()]);
        assert!(report.failed[0].error.contains("Include"));
        assert_eq!(a.read(&a.space_path(&personal)).trim_end(), "Host web\n  HostName 10.0.0.1");
        assert!(!a.main_config().contains("Host web"));
        settle(&a);
        let b = TestDevice::new("b", &relay, &clock);
        join_account(&b.env(), &words, "MacBook-B").unwrap();
        select_space(&b.env(), &personal).unwrap();
        settle(&b);
        assert_eq!(b.read(&b.space_path(&personal)), "Host web\n  HostName 10.0.0.1\n");
    }

    fn runtime_with(space_id: &str, baseline: bool) -> SyncRuntime {
        let runtime = SyncRuntime::default();
        let mut s = SyncStateV2::fresh("Box").unwrap();
        s.account = Some(crate::sync::state_v2::AccountState::new(&"a".repeat(64)));
        let mut space = SpaceState::new(&format!("work-{}.config", &space_id[..8]));
        space.baseline_established = baseline;
        s.spaces.insert(space_id.to_string(), space);
        runtime.core.lock().unwrap().state = Some(s);
        runtime
    }

    #[test]
    fn moves_into_a_space_need_this_process_engine_and_a_finished_first_sync() {
        let id = "3fa2c1d9".repeat(8);
        let runtime = runtime_with(&id, false);
        assert_eq!(refuse_while_sync_inactive(false, &runtime).unwrap_err().to_string(), ANOTHER_ENGINE_MESSAGE);
        runtime.core.lock().unwrap().save_blocked = Some("sync is off in this SSHelter process: boom".into());
        assert_eq!(refuse_while_sync_inactive(false, &runtime).unwrap_err().to_string(), "sync is off in this SSHelter process: boom");
        assert!(refuse_while_sync_inactive(true, &runtime).is_ok());
        assert_eq!(refuse_before_first_sync(&runtime, &id).unwrap_err().to_string(), WAIT_FOR_FIRST_SYNC);
        assert!(refuse_before_first_sync(&runtime_with(&id, true), &id).is_ok());
        let files = selected_space_files(&runtime, Path::new("/home/f/.ssh"));
        assert_eq!(files, vec![(id.clone(), PathBuf::from("/home/f/.ssh/sshelter/work-3fa2c1d9.config"))]);
    }
}
```

- [ ] **Step 10: `src-tauri/src/config/commands.rs` 的測試**

`load_wakes_sync` 改收勾選的 space 檔清單。

`src-tauri/src/config/commands.rs`:把

```rust
        let main = write_config(&dir, "config", &format!("Include {}\nHost local\n", managed.display()));
        let first = load_doc(&main).unwrap();
        assert!(load_wakes_sync(None, &first, Some(&managed)), "the first load wakes the engine");
        // 什麼都沒變(例如引擎寫檔之後、前端因 sync://applied 重新載入):不喚醒。
        let same = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&first), &same, Some(&managed)), "an identical reload does not");
        // 只有別的檔案變了:不喚醒。
        std::fs::write(&main, format!("Include {}\nHost local\n  User me\n", managed.display())).unwrap();
        let main_edited = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&same), &main_edited, Some(&managed)), "other files do not matter");
        // 受管檔在 app 以外被改了(例如被清空):喚醒。
        std::fs::write(&managed, "").unwrap();
        let emptied = load_doc(&main).unwrap();
        assert!(load_wakes_sync(Some(&main_edited), &emptied, Some(&managed)), "a changed fingerprint wakes it");
        // 拿不到受管檔路徑:只有第一次載入喚醒。
        assert!(load_wakes_sync(None, &emptied, None));
```

換成:

```rust
        let main = write_config(&dir, "config", &format!("Include {}\nHost local\n", managed.display()));
        let first = load_doc(&main).unwrap();
        assert!(load_wakes_sync(None, &first, Some(std::slice::from_ref(&managed))), "the first load wakes the engine");
        // 什麼都沒變(例如引擎寫檔之後、前端因 sync://applied 重新載入):不喚醒。
        let same = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&first), &same, Some(std::slice::from_ref(&managed))), "an identical reload does not");
        // 只有別的檔案變了:不喚醒。
        std::fs::write(&main, format!("Include {}\nHost local\n  User me\n", managed.display())).unwrap();
        let main_edited = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&same), &main_edited, Some(std::slice::from_ref(&managed))), "other files do not matter");
        // 受管檔在 app 以外被改了(例如被清空):喚醒。
        std::fs::write(&managed, "").unwrap();
        let emptied = load_doc(&main).unwrap();
        assert!(load_wakes_sync(Some(&main_edited), &emptied, Some(std::slice::from_ref(&managed))), "a changed fingerprint wakes it");
        // 拿不到受管檔路徑:只有第一次載入喚醒。
        assert!(load_wakes_sync(None, &emptied, None));
```

`src-tauri/src/config/commands.rs`:把

```rust
        std::fs::write(&managed, "Host web\n").unwrap();
        let with = load_doc(&main).unwrap();
        assert!(load_wakes_sync(Some(&without), &with, Some(&managed)), "appeared");
        assert!(load_wakes_sync(Some(&with), &without, Some(&managed)), "disappeared");
        // 存在卻載入不了(例如另存成 UTF-16):`load_doc` 略過它,引擎每一輪都重載 doc 並發 sync://applied。
        // 前端因此重新載入時不能再喚醒 —— 否則就是沒有間隔的迴圈。
```

換成:

```rust
        std::fs::write(&managed, "Host web\n").unwrap();
        let with = load_doc(&main).unwrap();
        assert!(load_wakes_sync(Some(&without), &with, Some(std::slice::from_ref(&managed))), "appeared");
        assert!(load_wakes_sync(Some(&with), &without, Some(std::slice::from_ref(&managed))), "disappeared");
        // 存在卻載入不了(例如另存成 UTF-16):`load_doc` 略過它,引擎每一輪都重載 doc 並發 sync://applied。
        // 前端因此重新載入時不能再喚醒 —— 否則就是沒有間隔的迴圈。
```

`src-tauri/src/config/commands.rs`:把

```rust
        assert!(unloadable.files.iter().all(|f| f.path != managed), "load_doc skips a non-UTF-8 include");
        let reloaded = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&unloadable), &reloaded, Some(&managed)));
    }
}
```

換成:

```rust
        assert!(unloadable.files.iter().all(|f| f.path != managed), "load_doc skips a non-UTF-8 include");
        let reloaded = load_doc(&main).unwrap();
        assert!(!load_wakes_sync(Some(&unloadable), &reloaded, Some(std::slice::from_ref(&managed))));
    }
}
```

- [ ] **Step 11: 更新 `src-tauri/src/sync/mod.rs`**

把 `src-tauri/src/sync/mod.rs` 整個換成:

```rust
//! Sync(Sync v2 spaces spec):端對端加密的多 space 同步。各子模組單一責任、皆可單元測試:
//! - `crypto`: 同步碼、金鑰推導、記錄加密
//! - `record`: 記錄模型與 LWW 合併
//! - `planner`: 本機變更偵測
//! - `hosts_file`: 同步檔的區塊操作、主 config 的 Include 清單、禁用的 directive
//! - `space_files`: space 檔命名、Include 清單順序與建立 / 移除 / 改名的順序規則
//! - `approval`: 危險設定的核准簽章
//! - `relay`: relay HTTP client(`RelayApi`)與輪詢間隔
//! - `reconcile`: 記錄的加解密編碼與套到檔案的效果
//! - `merge`: 帳戶與 space 區段的本機 diff、合併、上傳(純函式)
//! - `state`: v1 狀態(只為了升級)、狀態檔路徑、同步碼的 keychain account
//! - `state_v2`: 本機狀態(`version: 2`)與 v1 狀態檔的偵測
//! - `runtime`: `SyncCore`(generation / 狀態 / 帳戶金鑰)與局部提交
//! - `env`: 引擎與外界的邊界(keychain、relay、事件、時鐘)
//! - `files`: space 檔的準備、讀取、套用 + 發布交易、存檔 hook
//! - `account`: 帳戶生命週期與 relay 設定
//! - `spaces`: space 操作與核准
//! - `round`: 一輪同步
//! - `upgrade`: 從 v1 升級
//! - `migrate`: 主機搬進 space、跨檔案的同名主機
//! - `dto`: 給前端的事件與狀態形狀
//! - `engine`: Tauri 外殼(同步鎖、啟動、背景執行緒、存檔 hook、commands)
//! - `fake_relay`、`testkit`(只在測試):記憶體假 relay 與測試裝置

pub mod account;
pub mod approval;
pub mod crypto;
pub mod dto;
pub mod engine;
pub mod env;
#[cfg(test)]
pub mod fake_relay;
pub mod files;
pub mod hosts_file;
pub mod merge;
pub mod migrate;
pub mod planner;
pub mod reconcile;
pub mod record;
pub mod relay;
pub mod round;
pub mod runtime;
pub mod space_files;
pub mod spaces;
pub mod state;
pub mod state_v2;
#[cfg(test)]
pub mod testkit;
pub mod upgrade;
```

- [ ] **Step 12: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0432]: unresolved imports `crate::sync::dto::ApprovalNotice`, `crate::sync::dto::PendingApprovalView`, `crate::sync::dto::ReviewOutcome`, `crate::sync::dto::ReviewedVersion`, `crate::sync::dto::SyncConflict`, `crate::sync::dto::SyncOverview`
--> src/sync/engine.rs:21:30
```

- [ ] **Step 13: 實作 `src-tauri/src/sync/reconcile.rs`**

只留 `HostEffect` 與記錄的加解密編碼。

`src-tauri/src/sync/reconcile.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 遠端 host 記錄要套到 space 檔的效果。沒有第三種:格式不支援的記錄只會被略過。
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
    let kind = RecordKind::parse(&env.kind).ok_or_else(|| AppError::Other(format!("unknown record kind '{}'", env.kind)))?;
    let sealed = Sealed { id_hash: env.id_hash.clone(), nonce: env.nonce.clone(), ciphertext: env.ciphertext.clone() };
    let plaintext = crypto::open(keys, &env.kind, &env.id_hash, &sealed)?;
    let record: Record =
        serde_json::from_slice(&plaintext).map_err(|e| AppError::Other(format!("record payload is unreadable: {e}")))?;
    if record.kind != kind || crypto::id_hash(keys, record.kind.as_str(), &record.id) != env.id_hash {
        return Err(AppError::Other("record identity does not match its envelope".to_string()));
    }
    Ok(record)
}
```

- [ ] **Step 14: 修改 `src-tauri/src/sync/state_v2.rs`**

`RotationStep` 匯出 TS;`sync:mnemonic-next` 改經 `Keychain` 存取,刪掉直接呼叫 `secrets` 的 helper。

`src-tauri/src/sync/state_v2.rs`:把

```rust
use crate::error::AppError;
use crate::fsutil;
use crate::secrets;
use crate::sync::approval::ApprovalSignature;
use crate::sync::crypto::ChainKeys;
```

換成:

```rust
use crate::error::AppError;
use crate::fsutil;
use crate::sync::approval::ApprovalSignature;
use crate::sync::crypto::ChainKeys;
```

`src-tauri/src/sync/state_v2.rs`:把

```rust
/// 更換同步碼走到哪一步(spec §7.5 的編號)。
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum RotationStep {
```

換成:

```rust
/// 更換同步碼走到哪一步(spec §7.5 的編號)。
#[derive(Clone, Copy, Debug, PartialEq, Eq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
#[serde(rename_all = "snake_case")]
pub enum RotationStep {
```

`src-tauri/src/sync/state_v2.rs`:刪除從下面這段開始

```rust
pub fn store_next_mnemonic(words: &str) -> Result<(), AppError> {
```

到下面這段為止的整段程式碼(含這兩段本身,共 12 行):

```rust
    secrets::delete(NEXT_MNEMONIC_ACCOUNT)
}

```

- [ ] **Step 15: 修改 `src-tauri/src/sync/state.rs`**

v1 狀態只剩讀取用的型別、`state_path`、內建 relay 與 keychain account 名稱。

`src-tauri/src/sync/state.rs`:把

```rust
//! 本機同步狀態(`sync-state.json`,0600)與助記詞的 keychain 保管。
//! 助記詞永不落成純文字檔;派生值只在記憶體。

use std::collections::BTreeMap;
use std::path::{Path, PathBuf};

use serde::{Deserialize, Serialize};
```

換成:

```rust
//! v1 的本機同步狀態(`sync-state.json` 的 `version: 1` 格式):只用來讀取並升級(Sync v2 spec §7.6;讀檔在
//! `state_v2::load`)。另有兩版共用的狀態檔路徑、內建 relay 與同步碼的 keychain account。

use std::collections::BTreeMap;
use std::path::PathBuf;

use serde::{Deserialize, Serialize};
```

`src-tauri/src/sync/state.rs`:把

```rust
use crate::error::AppError;
use crate::fsutil;
use crate::secrets;
use crate::sync::record::{Envelope, LocalRecord, SCHEMA_VERSION};

```

換成:

```rust
use crate::error::AppError;
use crate::fsutil;
use crate::sync::record::{Envelope, LocalRecord, SCHEMA_VERSION};

```

`src-tauri/src/sync/state.rs`:刪除從下面這段開始

```rust
pub fn load(path: &Path) -> Result<Option<SyncState>, AppError> {
```

到下面這段為止的整段程式碼(含這兩段本身,共 45 行):

```rust
    secrets::delete(MNEMONIC_ACCOUNT)
}

```

- [ ] **Step 16: 修改 `src-tauri/src/sync/relay.rs`**

移除 v1 的 `LegacyRelayClient`(v1 引擎已不存在)。

`src-tauri/src/sync/relay.rs`:把

```rust
//! 中繼 HTTP client(v1 spec §5、Sync v2 spec §6)。只搬密文;所有錯誤都映射成錯誤值,絕不 panic。
//! 使用 blocking client:同步引擎跑在自己的 std 執行緒。**不可在 tokio runtime 內呼叫**
//! (`reqwest::blocking` 會 panic);Tauri command 要用 `tauri::async_runtime::spawn_blocking`。
//! - `RelayClient`(v2,spec §6.4):不綁權杖,每次呼叫帶入該 chain 的權杖;批次查詢、凍結、版本資訊;錯誤是
//!   型別化的 `RelayError`。引擎經由 `RelayApi` trait 使用它(測試以記憶體假中繼實作同一個 trait)。
//! - `LegacyRelayClient`(v1):一個 client 綁一個權杖,只給 v1 引擎用;v1 引擎換掉之後一起移除。

use std::collections::HashSet;
```

換成:

```rust
//! 中繼 HTTP client(Sync v2 spec §6)。只搬密文;所有錯誤都映射成錯誤值,絕不 panic。
//! 使用 blocking client:同步引擎跑在自己的 std 執行緒。**不可在 tokio runtime 內呼叫**
//! (`reqwest::blocking` 會 panic);Tauri command 要用 `tauri::async_runtime::spawn_blocking`。
//! `RelayClient`(spec §6.4)不綁權杖,每次呼叫帶入該 chain 的權杖;批次查詢、凍結、版本資訊;錯誤是型別化的
//! `RelayError`。引擎經由 `RelayApi` trait 使用它(測試以記憶體假中繼 `fake_relay` 實作同一個 trait)。

use std::collections::HashSet;
```

`src-tauri/src/sync/relay.rs`:刪除從下面這段開始

```rust
/// v1 client:一個 client 綁一個權杖,錯誤是 `AppError`(行為與 Phase A 相同)。只給 v1 引擎用。
```

到下面這段為止的整段程式碼(含這兩段本身,共 88 行):

```rust
    }
}

```

- [ ] **Step 17: 修改 `src-tauri/src/sync/hosts_file.rs`**

v1 的同步檔只剩 v1 升級會讀(`managed_path`)。

`src-tauri/src/sync/hosts_file.rs`:把

```rust
use crate::config::serialize::{render_directive, serialize_items};
use crate::error::AppError;
use crate::fsutil;
use crate::sync::space_files;

/// 寫進主 config 的 Include 值;`~` 在 macOS/Linux/Windows OpenSSH 皆可解析。
pub const INCLUDE_VALUE: &str = "~/.ssh/sshelter/hosts.config";

pub fn managed_path(ssh_dir: &Path) -> PathBuf {
    ssh_dir.join("sshelter").join("hosts.config")
```

換成:

```rust
use crate::config::serialize::{render_directive, serialize_items};
use crate::error::AppError;
use crate::sync::space_files;

/// v1 的同步檔 `~/.ssh/sshelter/hosts.config`:只剩 v1 升級會讀它(spec §7.6)。
pub fn managed_path(ssh_dir: &Path) -> PathBuf {
    ssh_dir.join("sshelter").join("hosts.config")
```

`src-tauri/src/sync/hosts_file.rs`:刪除從下面這段開始

```rust
/// 建立 `~/.ssh/sshelter/`(0700)與空的 `hosts.config`(0600);已存在則不動內容。
```

到下面這段為止的整段程式碼(含這兩段本身,共 13 行):

```rust
}

```

- [ ] **Step 18: 修改 `src-tauri/src/sync/crypto.rs`**

v1 退役之後,錯誤訊息改稱「sync code」。

`src-tauri/src/sync/crypto.rs`:把

```rust
pub fn generate_mnemonic() -> Result<String, AppError> {
    let m = Mnemonic::generate_in(Language::English, 24)
        .map_err(|e| AppError::Other(format!("cannot generate recovery words: {e}")))?;
    Ok(m.to_string())
}
```

換成:

```rust
pub fn generate_mnemonic() -> Result<String, AppError> {
    let m = Mnemonic::generate_in(Language::English, 24)
        .map_err(|e| AppError::Other(format!("cannot generate a sync code: {e}")))?;
    Ok(m.to_string())
}
```

`src-tauri/src/sync/crypto.rs`:把

```rust
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
```

換成:

```rust
    let words: Vec<String> = input.split_whitespace().map(|w| w.to_lowercase()).collect();
    if words.len() != 24 {
        return Err(AppError::Other(format!("a sync code has 24 words (got {})", words.len())));
    }
    let joined = words.join(" ");
    Mnemonic::parse_in(Language::English, &joined).map_err(|e| AppError::Other(format!("invalid sync code: {e}")))?;
    Ok(joined)
}
```

`src-tauri/src/sync/crypto.rs`:把

```rust
    let normalized = normalize_mnemonic(mnemonic)?;
    let m = Mnemonic::parse_in(Language::English, &normalized)
        .map_err(|e| AppError::Other(format!("invalid recovery phrase: {e}")))?;
    Ok(m.to_seed(""))
}
```

換成:

```rust
    let normalized = normalize_mnemonic(mnemonic)?;
    let m = Mnemonic::parse_in(Language::English, &normalized)
        .map_err(|e| AppError::Other(format!("invalid sync code: {e}")))?;
    Ok(m.to_seed(""))
}
```

- [ ] **Step 19: 實作 `src-tauri/src/sync/dto.rs`**

`overview` 只短暫持有 core 鎖拿快照,讀目錄(`stray_space_files`、主機數)在鎖外。

`PendingApprovalView.digest`(`spaces::review_digest`)是審核對話框顯示的版本;`ReviewedVersion` / `ReviewOutcome` 是 `sync_approve` / `sync_reject` 的參數與回傳(B3a 的 `spaces::approve` / `reject` 只認使用者看過的 `(alias, digest)`)。

`src-tauri/src/sync/dto.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// `sync://conflict` 的一項(spec §7.1 第 8 步):這台未上傳的修改被別台較新的版本取代。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncConflict {
    pub space_id: String,
    pub space_name: String,
    pub aliases: Vec<String>,
}

/// `sync://approval` 的一項(spec §7.4):這一輪新保留、等待核准的主機。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct ApprovalNotice {
    pub space_id: String,
    pub space_name: String,
    pub aliases: Vec<String>,
}

/// 最近一次 `GET /v1/info`(spec §6.4、§8)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncRelayView {
    pub url: String,
    /// relay 回報的版本;None = 舊版 relay(沒有 `GET /v1/info`)。
    pub version: Option<String>,
    /// 有批次查詢;false → 逐條查詢,UI 提示「relay 可以更新」。
    pub batch_pull: bool,
    /// 有凍結;false → 「更換同步碼」停用並說明要先更新 relay。
    pub freeze: bool,
}

/// 這台偵測到同步碼已被更換(spec §7.5):請使用者輸入新同步碼。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncFrozenView {
    #[cfg_attr(test, ts(type = "number"))]
    pub detected_at_ms: u64,
    /// 更換了同步碼的裝置名稱;空 = 還只知道 relay 拒絕了上傳。兩個以上 = 多台同時更換,輸入其中一組即可。
    pub by_devices: Vec<String>,
}

/// 這台正在更換同步碼(spec §7.5)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncRotationView {
    pub step: RotationStep,
    /// 只有第 3 步(凍結)之前可以取消。
    pub cancellable: bool,
    /// 建立 chain 被限流:暫停到這個時間後自動接續。
    #[cfg_attr(test, ts(type = "number | null"))]
    pub paused_until_ms: Option<u64>,
}

/// 帳戶裡的一台裝置(spec §8)。Forget 只是從清單移除,不是撤權。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncDeviceView {
    pub id: String,
    pub name: String,
    pub platform: String,
    #[cfg_attr(test, ts(type = "number"))]
    pub joined_at_ms: u64,
    #[cfg_attr(test, ts(type = "number"))]
    pub last_seen_ms: u64,
    pub is_this: bool,
    /// 這台裝置勾選的 space id。
    pub spaces: Vec<String>,
}

/// 帳戶裡的一個 space(spec §8),依 Include 清單的順序。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncSpaceView {
    pub id: String,
    pub name: String,
    /// 這台有勾選(同步下來成為一個檔案)。
    pub selected: bool,
    /// 勾選時的檔名與完整路徑(`~/.ssh/sshelter/<slug>-<id8>.config`)。
    pub file_name: Option<String>,
    pub file_path: Option<String>,
    /// 勾選時的主機數;沒勾選的 space 這台不知道。
    #[cfg_attr(test, ts(type = "number | null"))]
    pub hosts: Option<u64>,
    #[cfg_attr(test, ts(type = "number"))]
    pub pending_uploads: u64,
    #[cfg_attr(test, ts(type = "number"))]
    pub approvals: u64,
    /// 剛勾選、第一輪(基線輪)還沒完成:這段期間不要搬主機進來。
    pub first_sync_pending: bool,
    /// relay 上的 chain 不見了、帳戶卻仍有它(spec §9):提供「重建」或「刪除」。
    pub missing: bool,
    /// 只屬於這個 space 的錯誤(違反不變式、寫入失敗)。
    pub last_error: Option<String>,
    #[cfg_attr(test, ts(type = "number"))]
    pub created_at_ms: u64,
    /// 勾選了這個 space 的裝置名稱。
    pub synced_on: Vec<String>,
}

/// Settings → Sync 的全部狀態(`sync_overview` 與 `sync://status`)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct SyncOverview {
    pub joined: bool,
    /// 帳戶 chain id 的前 8 個 hex,只作辨識。
    pub account_short: Option<String>,
    pub device_id: String,
    pub device_name: String,
    pub relay_url: String,
    pub relay: Option<SyncRelayView>,
    #[cfg_attr(test, ts(type = "number | null"))]
    pub last_sync_ms: Option<u64>,
    pub last_error: Option<String>,
    /// 帳戶用了比這版新的格式:只讀,不上傳。
    pub read_only: bool,
    /// v1 升級還沒完成(spec §7.6)。
    pub upgrading: bool,
    pub frozen: Option<SyncFrozenView>,
    pub rotation: Option<SyncRotationView>,
    pub devices: Vec<SyncDeviceView>,
    pub spaces: Vec<SyncSpaceView>,
    #[cfg_attr(test, ts(type = "number"))]
    pub pending_uploads: u64,
    #[cfg_attr(test, ts(type = "number"))]
    pub approvals_waiting: u64,
    /// `~/.ssh/sshelter/` 裡不在 Include 清單上的 `.config` 檔:OpenSSH 不讀,只提示(spec §4.3)。
    pub stray_files: Vec<String>,
    /// 等使用者看過的提示,`sync_dismiss_notice(index)` 清掉。
    pub notices: Vec<SyncNotice>,
    /// 離開帳戶時同步碼刪不掉:顯示警示與重試。
    pub phrase_cleanup_pending: bool,
}

/// 一筆等待核准的主機(`sync_pending_approvals`;spec §7.4、§8 的審核對話框)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct PendingApprovalView {
    pub space_id: String,
    pub space_name: String,
    pub alias: String,
    /// 這一版的內容指紋(`spaces::review_digest`):核准 / 拒絕時連同 `alias` 送回(`ReviewedVersion`)—— 只處理使用者看過的
    /// 這一版。序號與版本號都可能指到別的內容(relay 的歷史倒退、兩台寫出同一個版本號),所以以內容認。
    pub digest: String,
    /// 要套用的完整區塊。
    pub text: String,
    /// 目前 space 檔裡的區塊;None = 這台還沒有這台主機。
    pub current_text: Option<String>,
    /// 目前區塊與新區塊的核准簽章(UI 標出受管制的行與差異)。
    pub applied: ApprovalSignature,
    pub incoming: ApprovalSignature,
    pub from_device: String,
    #[cfg_attr(test, ts(type = "number"))]
    pub updated_at_ms: u64,
}

/// 審核對話框送回的一筆(`sync_approve` / `sync_reject`):使用者看過的那一版 —— `PendingApprovalView` 的 `alias` 與 `digest`。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct ReviewedVersion {
    pub alias: String,
    pub digest: String,
}

/// `sync_approve` / `sync_reject` 的結果(spec §7.4)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct ReviewOutcome {
    /// 實際處理了幾台:核准 = 套用的主機數,拒絕 = 丟棄的版本數。
    #[cfg_attr(test, ts(type = "number"))]
    pub applied: u64,
    /// 略過的主機:使用者看過的版本已經不是待核准的那一版(被較新的取代、已處理或已不在清單上)。較新的版本留在清單上;
    /// UI 顯示「已變更,請重新確認」並重新讀 `sync_pending_approvals`。
    pub changed: Vec<String>,
    /// 動作之後的最新狀態(同其他 command 回傳的 `SyncOverview`)。
    pub overview: SyncOverview,
}

/// 組出 `SyncOverview`(spec §8)。只短暫持有 core 鎖;讀目錄在鎖外。
pub fn overview(env: &SyncEnv) -> Result<SyncOverview, AppError> {
    let (s, keys, upgrading) = {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        (s, core.account_keys.clone(), core.legacy.is_some())
    };
    let account = s.account.as_ref();
    let device_list: Vec<SyncDeviceView> = account
        .map(devices)
        .unwrap_or_default()
        .into_iter()
        .map(|(id, p)| SyncDeviceView {
            is_this: id == s.device_id,
            id,
            name: p.name,
            platform: p.platform,
            joined_at_ms: p.joined_at_ms,
            last_seen_ms: p.last_seen_ms,
            spaces: p.spaces,
        })
        .collect();
    let synced_on = |space_id: &str| -> Vec<String> {
        device_list.iter().filter(|d| d.spaces.iter().any(|x| x == space_id)).map(|d| d.name.clone()).collect()
    };
    let mut spaces: Vec<SyncSpaceView> = Vec::new();
    let entries = account.map(space_entries).unwrap_or_default();
    for entry in &entries {
        let deleted = entry.deleted || match (account, keys.as_ref()) {
            (Some(a), Some(k)) => space_deleted_by(a, k, &entry.id).is_some(),
            _ => false,
        };
        if deleted {
            continue;
        }
        let local = s.spaces.get(&entry.id).filter(|sp| sp.selected);
        spaces.push(SyncSpaceView {
            id: entry.id.clone(),
            name: entry.name.clone(),
            selected: local.is_some(),
            file_name: local.map(|sp| sp.file_name.clone()),
            file_path: local.and_then(|sp| space_path(env, &sp.file_name).ok()).map(|p| p.to_string_lossy().into_owned()),
            hosts: local.map(|sp| sp.records.values().filter(|l| l.record.kind == RecordKind::Host && !l.record.deleted).count() as u64),
            pending_uploads: local.map(|sp| sp.records.values().filter(|l| l.dirty).count() as u64).unwrap_or(0),
            approvals: local.map(|sp| sp.pending_approvals.len() as u64).unwrap_or(0),
            first_sync_pending: local.is_some_and(|sp| !sp.baseline_established),
            missing: local.is_some_and(|sp| sp.missing),
            last_error: local.and_then(|sp| sp.last_error.clone()),
            created_at_ms: entry.created_at_ms,
            synced_on: synced_on(&entry.id),
        });
    }
    let listed: Vec<String> = s.spaces.values().filter(|sp| sp.selected).map(|sp| sp.file_name.clone()).collect();
    let stray_files = if s.joined() { stray_space_files(&env.ssh_dir, &listed).unwrap_or_default() } else { Vec::new() };
    let account_dirty = account
        .map(|a| a.records.values().filter(|l| l.dirty).count() + a.sealed.values().filter(|x| x.dirty).count())
        .unwrap_or(0) as u64;
    Ok(SyncOverview {
        joined: s.joined(),
        account_short: account.map(|a| a.chain_id.chars().take(8).collect()),
        device_id: s.device_id.clone(),
        device_name: s.device_name.clone(),
        relay_url: s.relay_url.clone(),
        relay: s.relay_features.as_ref().filter(|f| f.url == s.relay_url).map(|f| SyncRelayView {
            url: f.url.clone(),
            version: f.version.clone(),
            batch_pull: f.supports(FEATURE_PULL_BATCH),
            freeze: f.supports(FEATURE_FREEZE),
        }),
        last_sync_ms: s.last_sync_ms,
        last_error: s.last_error.clone(),
        read_only: s.read_only(),
        upgrading,
        frozen: s.frozen().map(|f| SyncFrozenView {
            detected_at_ms: f.detected_at_ms,
            by_devices: f.markers.iter().map(|m| m.by_device_name.clone()).collect(),
        }),
        rotation: s.rotation.as_ref().map(|r| SyncRotationView {
            step: r.step,
            cancellable: r.cancellable(),
            paused_until_ms: r.paused_until_ms,
        }),
        pending_uploads: account_dirty + spaces.iter().map(|v| v.pending_uploads).sum::<u64>(),
        approvals_waiting: spaces.iter().map(|v| v.approvals).sum(),
        devices: device_list,
        spaces,
        stray_files,
        notices: s.notices.clone(),
        phrase_cleanup_pending: s.phrase_cleanup_pending,
    })
}

/// 等待核准的主機(spec §7.4 的審核對話框):每一筆附上目前 space 檔裡的區塊。
pub fn pending_approvals(env: &SyncEnv) -> Result<Vec<PendingApprovalView>, AppError> {
    let s = env.runtime.core.lock().unwrap().state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    let names: BTreeMap<String, String> =
        s.account.as_ref().map(space_entries).unwrap_or_default().into_iter().map(|e| (e.id, e.name)).collect();
    let mut current: BTreeMap<(String, String), String> = BTreeMap::new();
    {
        let doc_lock = env.doc.lock().unwrap();
        if let Some(doc) = doc_lock.as_ref() {
            for (id, sp) in s.spaces.iter().filter(|(_, sp)| !sp.pending_approvals.is_empty()) {
                let Ok(path) = space_path(env, &sp.file_name) else { continue };
                if let Some(file) = doc.files.iter().find(|f| f.path == path) {
                    for block in blocks_of(&file.items) {
                        current.insert((id.clone(), block.alias), block.text);
                    }
                }
            }
        }
    }
    let mut out = Vec::new();
    for (id, sp) in &s.spaces {
        for (alias, p) in &sp.pending_approvals {
            out.push(PendingApprovalView {
                space_id: id.clone(),
                space_name: names.get(id).cloned().unwrap_or_else(|| id.chars().take(8).collect()),
                alias: alias.clone(),
                digest: review_digest(p),
                text: p.text.clone(),
                current_text: current.get(&(id.clone(), alias.clone())).cloned(),
                applied: p.applied.clone(),
                incoming: p.incoming.clone(),
                from_device: p.from_device.clone(),
                updated_at_ms: p.record.updated_at_ms,
            });
        }
    }
    Ok(out)
}
```

- [ ] **Step 20: 實作 `src-tauri/src/sync/engine.rs`**

外殼:`with_env` 把 `AppState` 組成 `SyncEnv`;`run` 在 `spawn_blocking` 裡記下操作時間、視需要持有 lifecycle 鎖,完成後發 `sync://status`;`sync_dismiss_notice` 不換 generation。

`sync_approve` / `sync_reject` 收 `approvals: Vec<ReviewedVersion>`、回 `ReviewOutcome`(`run_review`):`changed` 非空時 UI 請使用者重新確認(B4 handoff)。

`src-tauri/src/sync/engine.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 第一輪之前的等待;之後依 `round::next_delay`(spec §6.4)。
const FIRST_DELAY: Duration = Duration::from_secs(45);
/// app data 目錄裡的行程間同步鎖:一個 OS 使用者同時只有一個行程跑同步引擎。
const ENGINE_LOCK_FILE: &str = "sync.lock";
pub(crate) const ANOTHER_ENGINE_MESSAGE: &str = "Sync is running in another SSHelter process — quit it to use sync here";

/// 喚醒背景執行緒的通道;由存檔 hook 與 commands 共用,故放全域。
static WAKER: OnceLock<Mutex<Option<Sender<()>>>> = OnceLock::new();
/// 行程間同步鎖的 handle:鎖跟著這個 File,所以要活到行程結束。
static ENGINE_LOCK: OnceLock<File> = OnceLock::new();
/// 這個行程取得了同步鎖、跑著同步引擎。false(別的行程持有鎖、或單元測試)時存檔 hook 什麼都不做。
static ENGINE_ACTIVE: AtomicBool = AtomicBool::new(false);
/// `persist_file` 只有路徑與 doc、沒有 AppHandle;存檔 hook 靠它組出 `SyncEnv`。只在 `initialize` 設定。
static APP: OnceLock<AppHandle> = OnceLock::new();

/// 這個行程是否跑著同步引擎。搬進 space 檔的命令用它拒絕沒有引擎的行程(`migrate::refuse_while_sync_inactive`)。
pub fn engine_active() -> bool {
    ENGINE_ACTIVE.load(Ordering::SeqCst)
}

pub fn wake() {
    if let Some(slot) = WAKER.get() {
        if let Some(tx) = slot.lock().unwrap().as_ref() {
            let _ = tx.send(());
        }
    }
}

/// 主視窗到前景 / 離開前景(`WindowEvent::Focused`):決定輪詢間隔(`relay::next_poll_delay`);回到前景立刻同步一輪。
pub fn window_focused(app: &AppHandle, focused: bool) {
    let state = app.state::<AppState>();
    state.sync.set_focused(focused, SystemClock.now_ms());
    if focused {
        wake();
    }
}

/// production 的事件:Tauri events + tray。**呼叫端不持有任何鎖**(建立 tray menu 會同步等待主執行緒)。
struct TauriEvents {
    app: AppHandle,
}

impl SyncEvents for TauriEvents {
    fn status(&self) {
        if let Ok(Ok(overview)) = with_env(&self.app, dto::overview) {
            let _ = self.app.emit("sync://status", &overview);
        }
    }

    fn applied(&self, hosts: usize) {
        let aliases = {
            let state = self.app.state::<AppState>();
            let doc_lock = state.doc.lock().unwrap();
            doc_lock.as_ref().map(crate::tray::tray_aliases)
        };
        if let Some(aliases) = aliases {
            let _ = crate::tray::rebuild_tray(&self.app, &aliases);
        }
        let _ = self.app.emit("sync://applied", &hosts);
    }

    fn conflict(&self, conflicts: &[SyncConflict]) {
        let _ = self.app.emit("sync://conflict", conflicts);
    }

    fn approval(&self, waiting: &[ApprovalNotice]) {
        let _ = self.app.emit("sync://approval", waiting);
    }

    fn notice(&self, notice: &SyncNotice) {
        let _ = self.app.emit("sync://notice", notice);
    }

    fn wake(&self) {
        wake();
    }
}

/// 以 app 的狀態、OS keychain、HTTP relay 與系統時鐘組出 `SyncEnv`。
pub(crate) fn with_env<T>(app: &AppHandle, f: impl FnOnce(&SyncEnv) -> T) -> Result<T, AppError> {
    let state = app.state::<AppState>();
    let events = TauriEvents { app: app.clone() };
    let env = SyncEnv {
        doc: &state.doc,
        backed_up: &state.backed_up,
        retention: &state.backup_retention,
        runtime: &state.sync,
        ssh_dir: crate::keys::ssh_dir()?,
        state_path: v1_state::state_path()?,
        home: None,
        keychain: &OsKeychain,
        relays: &HttpRelays,
        events: &events,
        clock: &SystemClock,
        platform: std::env::consts::OS,
    };
    Ok(f(&env))
}

/// `persist_file` 寫完任何檔案後呼叫(呼叫端持有 doc 鎖)。space 檔的 app 編輯在存檔當下規劃(`files::note_written`)。
/// 同步引擎在別的行程、或單元測試裡:什麼都不做。
pub fn note_file_written(path: &Path, items: &[Item]) {
    if !ENGINE_ACTIVE.load(Ordering::SeqCst) {
        return;
    }
    let Some(app) = APP.get() else { return };
    let _ = with_env(app, |env| files::note_written(env, path, items));
}

/// 一輪之前先把已經排隊的喚醒全部收掉(一次搬 60 台主機會排上百則);等待時間依 `round::next_delay`。
fn worker_loop(app: AppHandle, rx: Receiver<()>) {
    let mut delay = FIRST_DELAY;
    loop {
        match rx.recv_timeout(delay) {
            Ok(()) | Err(RecvTimeoutError::Timeout) => {
                while rx.try_recv().is_ok() {}
                if let Ok(next) = with_env(&app, |env| {
                    let _ = round::sync_once(env);
                    round::next_delay(env)
                }) {
                    delay = next;
                }
            }
            Err(RecvTimeoutError::Disconnected) => return,
        }
    }
}

/// 狀態檔讀不懂(損毀,或更新版 SSHelter 寫的)時,把它搬到同一目錄的 `sync-state.unreadable-<ms>.json`:
/// 之後的任何存檔都會把新狀態寫到原路徑,不搬就會蓋掉它。回傳新檔名;搬不動回傳 I/O 錯誤。
fn set_aside_unreadable_state(path: &Path, timestamp_ms: u64) -> std::io::Result<String> {
    let name = format!("sync-state.unreadable-{timestamp_ms}.json");
    std::fs::rename(path, path.with_file_name(&name))?;
    Ok(name)
}

/// 狀態檔還留在原路徑時附在說明後面的提示;同一段完整說明也放進 `save_blocked`。
const STATE_LEFT_IN_PLACE: &str = "; the sync state file was left in place — restart SSHelter to retry";

/// 啟動時 `state_v2::load` 失敗:回傳(要放進 `last_error` 的說明, `save_blocked`)。
/// - 內容錯誤(`AppError::Other`:讀不懂的 JSON、更新版的格式)→ 搬到旁邊保留,之後照常存新狀態。搬不動 →
///   這個 session 不寫狀態。
/// - 其他(I/O 錯誤)可能只是暫時的 → 檔案留在原地,這個 session 不寫狀態。
fn unreadable_state_outcome(path: &Path, timestamp_ms: u64, error: &AppError) -> (String, Option<String>) {
    let left_in_place = |message: String| {
        let message = format!("{message}{STATE_LEFT_IN_PLACE}");
        (message.clone(), Some(message))
    };
    match error {
        AppError::Other(_) => match set_aside_unreadable_state(path, timestamp_ms) {
            Ok(name) => (format!("{error}; the old file was kept as {name}"), None),
            Err(e) => left_in_place(format!("{error}; could not set the old file aside: {e}")),
        },
        _ => left_in_place(error.to_string()),
    }
}

/// 啟動時的狀態(`startup`):給 core 的狀態、帳戶金鑰、等待升級的 v1 狀態,以及要不要立刻寫一次狀態檔。
#[derive(Debug)]
struct Startup {
    state: SyncStateV2,
    keys: Option<ChainKeys>,
    legacy: Option<LegacyState>,
    /// 未加入的 v1 狀態直接換成 v2:啟動時寫一次。
    save_now: bool,
}

/// 依讀到的狀態檔決定啟動狀態(純邏輯,keychain 注入)。
/// - v2:已加入就從 keychain 推導帳戶金鑰,失敗時說明放進 `last_error`(同 v1 的分類)。
/// - v1 且已加入:等背景執行緒升級(spec §7.6),core 先放一個未加入的 v2 外殼。
/// - v1 且未加入:直接換成 v2。
/// - 沒有檔案 / 讀不懂:全新狀態(讀不懂的說明放進 `last_error`)。
fn startup(loaded: Result<LoadedState, AppError>, keychain: &dyn Keychain, device_name: &str) -> Result<Startup, AppError> {
    Ok(match loaded {
        Ok(LoadedState::Current(s)) => {
            let mut s = *s;
            let mut keys = None;
            if let Some(chain) = s.account.as_ref().map(|a| a.chain_id.clone()) {
                match account_keys_from_keychain(keychain.get(MNEMONIC_ACCOUNT), &chain) {
                    Ok(k) => keys = Some(k),
                    Err(message) => s.last_error = Some(message),
                }
            }
            Startup { state: s, keys, legacy: None, save_now: false }
        }
        Ok(LoadedState::Legacy(v1)) if v1.joined() => {
            Startup { state: upgrade::shell_state(&v1), keys: None, legacy: Some(*v1), save_now: false }
        }
        Ok(LoadedState::Legacy(v1)) => Startup { state: upgrade::shell_state(&v1), keys: None, legacy: None, save_now: true },
        Ok(LoadedState::Missing) => {
            Startup { state: SyncStateV2::fresh(device_name)?, keys: None, legacy: None, save_now: false }
        }
        Err(e) => {
            let mut s = SyncStateV2::fresh(device_name)?;
            s.last_error = Some(e.to_string());
            Startup { state: s, keys: None, legacy: None, save_now: false }
        }
    })
}

/// 取不到同步鎖時要顯示的說明。
fn engine_lock_error(e: &dyn std::fmt::Display) -> String {
    format!("sync is off in this SSHelter process: the sync lock could not be taken ({e}); restart SSHelter to retry")
}

/// 同步鎖被別的行程拿著時最多試幾次、每次之間等多久(合計約 2 秒;app 內更新時舊行程還沒結束)。
const ENGINE_LOCK_ATTEMPTS: u32 = 10;
const ENGINE_LOCK_RETRY_DELAY: Duration = Duration::from_millis(200);

/// 取得 `<dir>/sync.lock` 的獨占鎖。回傳的 File 要一直活著。別的行程一直持有 → `ANOTHER_ENGINE_MESSAGE`。
fn acquire_engine_lock(dir: &Path) -> Result<File, String> {
    acquire_engine_lock_with(dir, ENGINE_LOCK_ATTEMPTS, || std::thread::sleep(ENGINE_LOCK_RETRY_DELAY))
}

/// `acquire_engine_lock` 的本體:最多試 `attempts` 次,兩次之間呼叫 `pause`(測試注入,不必真的等)。
fn acquire_engine_lock_with(dir: &Path, attempts: u32, mut pause: impl FnMut()) -> Result<File, String> {
    fsutil::ensure_dir_secure(dir).map_err(|e| engine_lock_error(&e))?;
    let file = OpenOptions::new()
        .read(true)
        .write(true)
        .create(true)
        .truncate(false)
        .open(dir.join(ENGINE_LOCK_FILE))
        .map_err(|e| engine_lock_error(&e))?;
    let mut attempt = 1;
    loop {
        match file.try_lock() {
            Ok(()) => return Ok(file),
            Err(TryLockError::WouldBlock) if attempt < attempts => {
                attempt += 1;
                pause();
            }
            Err(TryLockError::WouldBlock) => return Err(ANOTHER_ENGINE_MESSAGE.to_string()),
            Err(TryLockError::Error(e)) => return Err(engine_lock_error(&e)),
        }
    }
}

/// 啟動:取得同步鎖,載入狀態(v2;v1 交給背景執行緒升級),推導帳戶金鑰,開背景執行緒。狀態壞掉不阻擋 app 啟動
/// (`unreadable_state_outcome`)。拿不到同步鎖的行程只讀一份狀態給 UI 看:不搬狀態檔、不讀 keychain、不開背景
/// 執行緒、存檔 hook 不動作,`save_blocked` 讓所有會寫狀態的命令一律拒絕。
pub fn initialize(app: &AppHandle) -> Result<(), AppError> {
    let _ = APP.set(app.clone());
    let state = app.state::<AppState>();
    let lock = fsutil::app_data_root().map_err(|e| engine_lock_error(&e)).and_then(|root| acquire_engine_lock(&root));
    match lock {
        Ok(file) => {
            let _ = ENGINE_LOCK.set(file);
            ENGINE_ACTIVE.store(true, Ordering::SeqCst);
        }
        Err(reason) => {
            // 檔案屬於持有鎖的那個行程,這裡絕不搬動或改寫它。
            let mut shown = match v1_state::state_path().and_then(|path| state_v2::load(&path)) {
                Ok(LoadedState::Current(s)) => *s,
                Ok(LoadedState::Legacy(v1)) => upgrade::shell_state(&v1),
                _ => SyncStateV2::fresh(&default_device_name())?,
            };
            shown.last_error = Some(reason.clone());
            let mut core = state.sync.core.lock().unwrap();
            core.state = Some(shown);
            core.save_blocked = Some(reason);
            return Ok(());
        }
    }
    let (loaded, save_blocked) = match v1_state::state_path() {
        Ok(path) => match state_v2::load(&path) {
            Ok(loaded) => (Ok(loaded), None),
            Err(e) => {
                let (message, blocked) = unreadable_state_outcome(&path, SystemClock.now_ms(), &e);
                (Err(AppError::Other(message)), blocked)
            }
        },
        Err(e) => (Err(e), None),
    };
    let start = startup(loaded, &OsKeychain, &default_device_name())?;
    {
        let mut core = state.sync.core.lock().unwrap();
        core.state = Some(start.state);
        core.account_keys = start.keys;
        core.legacy = start.legacy;
        core.save_blocked = save_blocked;
        if start.save_now {
            if let Ok(path) = v1_state::state_path() {
                let _ = save_core(&mut core, &path);
            }
        }
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

/// spawn_blocking 的 JoinHandle 錯誤 → AppError(`sync::migrate` 的 command 共用)。
pub(crate) fn join_error(e: tauri::Error) -> AppError {
    AppError::Other(format!("sync task failed: {e}"))
}

/// 在 `spawn_blocking` 裡跑一個引擎動作(`reqwest::blocking` 不能在 tokio runtime 內呼叫;等鎖與寫磁碟也不在主執行
/// 緒上)。`lifecycle` = 全程持有 lifecycle 鎖(帳戶與 space 結構的變更)。動作完成、鎖都放掉之後發 `sync://status`。
async fn run<T: Send + 'static>(
    app: AppHandle,
    lifecycle: bool,
    f: impl FnOnce(&SyncEnv) -> Result<T, AppError> + Send + 'static,
) -> Result<T, AppError> {
    let handle = app.clone();
    let result = tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        state.sync.note_activity(SystemClock.now_ms());
        let _guard = lifecycle.then(|| state.sync.lifecycle.lock().unwrap());
        with_env(&handle, f)?
    })
    .await
    .map_err(join_error)?;
    let _ = with_env(&app, |env| env.events.status());
    result
}

/// 動作完成後的最新狀態。
async fn run_then_overview(
    app: AppHandle,
    lifecycle: bool,
    f: impl FnOnce(&SyncEnv) -> Result<(), AppError> + Send + 'static,
) -> Result<SyncOverview, AppError> {
    run(app, lifecycle, move |env| {
        f(env)?;
        dto::overview(env)
    })
    .await
}

#[tauri::command]
pub async fn sync_overview(app: AppHandle) -> Result<SyncOverview, AppError> {
    tauri::async_runtime::spawn_blocking(move || with_env(&app, dto::overview)?).await.map_err(join_error)?
}

/// 立刻同步一輪(前端在視窗回到前景時也會呼叫);算一次操作。
#[tauri::command]
pub fn sync_now(app: AppHandle) -> Result<(), AppError> {
    app.state::<AppState>().sync.note_activity(SystemClock.now_ms());
    wake();
    Ok(())
}

/// 建立帳戶(spec §7.3):回傳同步碼(呼叫端只放在元件狀態裡,不進查詢快取)。
#[tauri::command]
pub async fn sync_create_account(app: AppHandle, device_name: String) -> Result<String, AppError> {
    run(app, true, move |env| account::create_account(env, &device_name)).await
}

#[tauri::command]
pub async fn sync_join_account(app: AppHandle, words: String, device_name: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| account::join_account(env, &words, &device_name)).await
}

/// 離開帳戶:這台的 space 檔搬到 `~/.ssh/sshelter-local/`、主 config 改以一般的 Include 引入,ssh 照常可用(留下
/// `SyncNotice::LeftAccount`;搬不過去就什麼都不改、回錯誤)。`delete_remote`(這台是最後一台時)一併刪除帳戶與所有
/// space 的 chain。未加入時只重試 keychain 清理。
#[tauri::command]
pub async fn sync_leave_account(app: AppHandle, delete_remote: bool) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| account::leave_account(env, delete_remote)).await
}

#[tauri::command]
pub async fn sync_show_words(app: AppHandle) -> Result<String, AppError> {
    run(app, false, account::show_words).await
}

#[tauri::command]
pub async fn sync_set_relay_url(app: AppHandle, url: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| account::set_relay_url(env, &url)).await
}

/// 重新查一次 `GET /v1/info`(使用者更新 relay 之後)。
#[tauri::command]
pub async fn sync_check_relay(app: AppHandle) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, |env| account::check_relay(env).map(|_| ())).await
}

#[tauri::command]
pub async fn sync_set_device_name(app: AppHandle, name: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| account::set_device_name(env, &name)).await
}

/// 只把裝置從清單移除,**不是撤權**(要撤銷遺失的電腦請更換同步碼);UI 文案必須如此說明。
#[tauri::command]
pub async fn sync_forget_device(app: AppHandle, device_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, false, move |env| account::forget_device(env, &device_id)).await
}

#[tauri::command]
pub async fn sync_create_space(app: AppHandle, name: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::create_space(env, &name).map(|_| ())).await
}

#[tauri::command]
pub async fn sync_rename_space(app: AppHandle, space_id: String, name: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::rename_space(env, &space_id, &name)).await
}

#[tauri::command]
pub async fn sync_delete_space(app: AppHandle, space_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::delete_space(env, &space_id)).await
}

#[tauri::command]
pub async fn sync_select_space(app: AppHandle, space_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::select_space(env, &space_id)).await
}

#[tauri::command]
pub async fn sync_unselect_space(app: AppHandle, space_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::unselect_space(env, &space_id)).await
}

/// relay 上的 chain 不見了、帳戶仍有這個 space(spec §9):用這台的內容重建。
#[tauri::command]
pub async fn sync_rebuild_space(app: AppHandle, space_id: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| spaces::rebuild_space(env, &space_id)).await
}

#[tauri::command]
pub async fn sync_pending_approvals(app: AppHandle) -> Result<Vec<PendingApprovalView>, AppError> {
    run(app, false, dto::pending_approvals).await
}

/// 核准、拒絕之後附上最新狀態;`changed` 是使用者看過、卻已經不是待核准那一版的主機。
async fn run_review(
    app: AppHandle,
    f: impl FnOnce(&SyncEnv) -> Result<spaces::Reviewed, AppError> + Send + 'static,
) -> Result<ReviewOutcome, AppError> {
    run(app, true, move |env| {
        let reviewed = f(env)?;
        Ok(ReviewOutcome { applied: reviewed.applied as u64, changed: reviewed.changed, overview: dto::overview(env)? })
    })
    .await
}

fn reviewed_versions(approvals: &[ReviewedVersion]) -> Vec<(String, String)> {
    approvals.iter().map(|v| (v.alias.clone(), v.digest.clone())).collect()
}

/// 核准(spec §7.4;「全部核准」= 傳入整個清單)。`approvals` 是對話框顯示的版本(`PendingApprovalView` 的 `alias` 與
/// `digest`):只套用清單上仍是那一版的;較新的版本留在清單上、列在回傳的 `changed`(UI:已變更,請重新確認)。
#[tauri::command]
pub async fn sync_approve(app: AppHandle, space_id: String, approvals: Vec<ReviewedVersion>) -> Result<ReviewOutcome, AppError> {
    run_review(app, move |env| spaces::approve(env, &space_id, &reviewed_versions(&approvals))).await
}

/// 拒絕(spec §7.4):丟棄對話框顯示的版本(同 `sync_approve` 的 `approvals` 與 `changed`),本機維持原狀。
#[tauri::command]
pub async fn sync_reject(app: AppHandle, space_id: String, approvals: Vec<ReviewedVersion>) -> Result<ReviewOutcome, AppError> {
    run_review(app, move |env| spaces::reject(env, &space_id, &reviewed_versions(&approvals))).await
}

/// 搬移精靈「搬進一個 space」(spec §7.2)。
#[tauri::command]
pub async fn sync_move_hosts_to_space(
    app: AppHandle,
    aliases: Vec<String>,
    space_id: String,
    tag_by_file: bool,
) -> Result<crate::sync::migrate::MigrationReport, AppError> {
    run(app, false, move |env| crate::sync::migrate::move_hosts_into_space(env, engine_active(), aliases, &space_id, tag_by_file)).await
}

/// 使用者看過了一則提示(`SyncOverview::notices` 的 index)。
#[tauri::command]
pub async fn sync_dismiss_notice(app: AppHandle, index: usize) -> Result<SyncOverview, AppError> {
    // 看過提示不是結構性變更:只改 `notices`、不換 generation —— 在途的輪次照常提交(它們只在尾端加提示),不必為了
    // 一則提示整輪重跑、多查一次 relay。
    run_then_overview(app, false, move |env| {
        let mut core = env.runtime.core.lock().unwrap();
        if let Some(reason) = &core.save_blocked {
            return Err(AppError::Other(reason.clone()));
        }
        let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        if index >= s.notices.len() {
            return Err(AppError::NotFound("that notice is already gone".to_string()));
        }
        s.notices.remove(index);
        save_core(&mut core, &env.state_path)
    })
    .await
}
```

- [ ] **Step 21: 實作 `src-tauri/src/sync/migrate.rs`**

搬進 space 的拒絕規則(精靈與 `config_move_host` 共用)全部在任何改動之前檢查;一台失敗不中斷整批,寫檔失敗才停下並要求重載。

`src-tauri/src/sync/migrate.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 目標 space 剛勾選、基線輪還沒跑完時搬進去的拒絕訊息。
pub const WAIT_FOR_FIRST_SYNC: &str = "wait for the first sync of this space to finish before moving hosts into it";

#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct DuplicateAlias {
    pub alias: String,
    /// 被遮蔽(ssh 不會用)的那份所在的檔案。
    pub local_file: String,
}

#[derive(Clone, Copy, Debug, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum ShadowedAction {
    /// 被遮蔽的那份改名 `<alias>-local`(保留它的定義)。
    Rename,
    /// 移除被遮蔽的那份(改用 ssh 正在用的那份)。
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
    let stem = path.file_name().map(|s| s.to_string_lossy().into_owned()).unwrap_or_default();
    let stem = stem.strip_suffix(".config").or_else(|| stem.strip_suffix(".conf")).unwrap_or(&stem).to_lowercase();
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

/// `move_host` 會搬的那個區塊:`find_host_file_index` → 該檔案裡「任一 pattern 相符」的第一個區塊。
fn block_to_move<'a>(doc: &'a SshConfigDoc, alias: &str) -> Option<&'a Item> {
    let idx = find_host_file_index(doc, alias)?;
    doc.files[idx].items.iter().find(|i| matches!(i, Item::Host(h) if h.patterns.iter().any(|p| p == alias)))
}

/// 搬進 space 前的資格檢查:那個區塊的所有 pattern 都必須具名。找不到 alias 交給 `move_host` 回報。
pub fn refuse_wildcard(doc: &SshConfigDoc, alias: &str) -> Result<(), AppError> {
    match block_to_move(doc, alias) {
        Some(Item::Host(h)) if !is_syncable_block(&h.patterns) => Err(AppError::Other(format!(
            "host '{alias}' belongs to a block with wildcard patterns and cannot be synced"
        ))),
        _ => Ok(()),
    }
}

/// 區塊含 `Include` 或帶引號的 keyword:不能搬進 space(spec §7.2、§7.4)—— 搬進去那個 space 就違反不變式而暫停。
pub fn refuse_forbidden(doc: &SshConfigDoc, alias: &str) -> Result<(), AppError> {
    match block_to_move(doc, alias).and_then(|block| forbidden_directive(std::slice::from_ref(block))) {
        Some(f) => Err(AppError::Other(format!(
            "host '{alias}' contains {}, which synced hosts cannot use; keep it in a local file",
            f.describe()
        ))),
        None => Ok(()),
    }
}

/// `target` 檔案裡已有 Host 區塊定義了 `name`(該區塊任一 pattern 等於它)。
pub fn managed_defines(doc: &SshConfigDoc, target: &Path, name: &str) -> bool {
    doc.files
        .iter()
        .filter(|f| f.path == target)
        .flat_map(|f| f.items.iter())
        .any(|i| matches!(i, Item::Host(h) if h.patterns.iter().any(|p| p == name)))
}

fn managed_first_alias(doc: &SshConfigDoc, target: &Path, name: &str) -> bool {
    doc.files.iter().filter(|f| f.path == target).flat_map(|f| f.items.iter()).any(|i| first_alias(i) == Some(name))
}

/// 搬進目標 space 檔之前的重複檢查(同 v1):目標檔已經定義了要搬的區塊的**任一** pattern 就拒絕 —— 同一個檔案裡
/// 重複的 alias 會讓那個 space 違反不變式而暫停。不同 space 之間同名是允許的(spec §4.3,UI 另外標示)。
pub fn refuse_already_synced(doc: &SshConfigDoc, target: &Path, alias: &str) -> Result<(), AppError> {
    let moving: &[String] = match block_to_move(doc, alias) {
        Some(Item::Host(h)) => h.patterns.as_slice(),
        _ => &[],
    };
    let Some(name) = std::iter::once(alias).chain(moving.iter().map(String::as_str)).find(|name| managed_defines(doc, target, name))
    else {
        return Ok(());
    };
    let first = moving.first().map_or(alias, String::as_str);
    if name == first && managed_first_alias(doc, target, name) {
        Err(AppError::Other(format!("'{name}' is already in that space — resolve the duplicate instead")))
    } else {
        Err(AppError::Other(format!(
            "'{name}' is already used by a host in that space — remove or rename it in the local host '{first}' first"
        )))
    }
}

/// 這個行程沒有跑同步引擎時,拒絕任何「搬進 space」的動作(這個行程的同步狀態只是啟動時的快照)。訊息沿用
/// `save_blocked`;沒有記下原因(單元測試)時退回 `ANOTHER_ENGINE_MESSAGE`。呼叫端可能持有 doc 鎖。
pub fn refuse_while_sync_inactive(active: bool, sync: &SyncRuntime) -> Result<(), AppError> {
    if active {
        return Ok(());
    }
    let reason = sync.core.lock().unwrap().save_blocked.clone();
    Err(AppError::Other(reason.unwrap_or_else(|| ANOTHER_ENGINE_MESSAGE.to_string())))
}

/// 目標 space 剛勾選、基線輪還沒跑完就拒絕搬進去:搬進去的主機會被基線輪以 chain 為準直接覆蓋。
pub fn refuse_before_first_sync(sync: &SyncRuntime, space_id: &str) -> Result<(), AppError> {
    let pending = sync
        .core
        .lock()
        .unwrap()
        .state
        .as_ref()
        .and_then(|s| s.spaces.get(space_id))
        .is_some_and(|sp| !sp.baseline_established);
    if pending {
        return Err(AppError::Other(WAIT_FOR_FIRST_SYNC.to_string()));
    }
    Ok(())
}

/// 這台勾選的 space 檔(space id, 路徑),依 Include 清單的順序(名稱不分大小寫 → 名稱 → id,spec §4.3)。呼叫端可能
/// 持有 doc 鎖,這裡只短暫拿 core 鎖。
pub fn selected_space_files(sync: &SyncRuntime, ssh_dir: &Path) -> Vec<(String, PathBuf)> {
    let core = sync.core.lock().unwrap();
    let Some(s) = core.state.as_ref() else { return Vec::new() };
    let mut out: Vec<(String, String, PathBuf)> = s
        .spaces
        .iter()
        .filter(|(_, sp)| sp.selected)
        .filter_map(|(id, sp)| {
            let name = s.account.as_ref().and_then(|a| space_entry(a, id)).map(|e| e.name).unwrap_or_else(|| id.clone());
            Some((id.clone(), name, space_file_path(ssh_dir, &sp.file_name).ok()?))
        })
        .collect();
    out.sort_by(|a, b| a.1.to_lowercase().cmp(&b.1.to_lowercase()).then_with(|| a.1.cmp(&b.1)).then_with(|| a.0.cmp(&b.0)));
    out.into_iter().map(|(id, _, path)| (id, path)).collect()
}

/// 每個同步的 alias 由哪個 space 檔勝出:`spaces` 依 Include 順序,第一個定義它(第一個 pattern)的檔案。
fn winners(doc: &SshConfigDoc, spaces: &[PathBuf]) -> BTreeMap<String, PathBuf> {
    let mut out = BTreeMap::new();
    for path in spaces {
        for file in doc.files.iter().filter(|f| &f.path == path) {
            for alias in file.items.iter().filter_map(first_alias) {
                out.entry(alias.to_string()).or_insert_with(|| path.clone());
            }
        }
    }
    out
}

/// 被遮蔽的主機(spec §4.3):alias 定義在某個 space 檔、又出現在其他任何檔案(其他 space 檔或本機檔)。ssh 用 Include
/// 清單中排在前面的 space 檔那份(Include 在主 config 最頂端);其他那幾份列出來,`local_file` = 它們所在的檔案。
pub fn duplicate_aliases(doc: &SshConfigDoc, spaces: &[PathBuf]) -> Vec<DuplicateAlias> {
    let winners = winners(doc, spaces);
    let mut out = Vec::new();
    for file in &doc.files {
        for alias in file.items.iter().filter_map(first_alias) {
            if winners.get(alias).is_some_and(|w| *w != file.path) {
                out.push(DuplicateAlias { alias: alias.to_string(), local_file: file.path.to_string_lossy().into_owned() });
            }
        }
    }
    out
}

/// 處理一筆被遮蔽的主機:以 `file`(完整路徑)定位那個檔案裡第一個 pattern 等於 `alias` 的區塊。回傳改動的檔案
/// 索引(呼叫端負責 `persist_file`)。拒絕改 ssh 正在用的那份。被遮蔽的那份若在另一個 space 檔,改動照常同步。
pub fn resolve_shadowed(
    doc: &mut SshConfigDoc,
    alias: &str,
    file: &str,
    action: ShadowedAction,
    spaces: &[PathBuf],
) -> Result<usize, AppError> {
    let idx = doc
        .files
        .iter()
        .position(|f| f.path.to_string_lossy() == file)
        .ok_or_else(|| AppError::NotFound(format!("file '{file}' is not loaded")))?;
    if winners(doc, spaces).get(alias).is_some_and(|w| *w == doc.files[idx].path) {
        return Err(AppError::Other("refusing to change the copy ssh uses; pick the shadowed file".to_string()));
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

fn space_paths(sync: &SyncRuntime) -> Result<Vec<PathBuf>, AppError> {
    let ssh_dir = crate::keys::ssh_dir()?;
    Ok(selected_space_files(sync, &ssh_dir).into_iter().map(|(_, p)| p).collect())
}

#[tauri::command]
pub async fn sync_duplicate_aliases(app: AppHandle) -> Result<Vec<DuplicateAlias>, AppError> {
    let handle = app.clone();
    tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        let guard = state.doc.lock().unwrap();
        let doc = guard.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        Ok(duplicate_aliases(doc, &space_paths(&state.sync)?))
    })
    .await
    .map_err(crate::sync::engine::join_error)?
}

#[tauri::command]
pub async fn sync_resolve_shadowed(app: AppHandle, alias: String, file: String, action: ShadowedAction) -> Result<Vec<DuplicateAlias>, AppError> {
    let handle = app.clone();
    tauri::async_runtime::spawn_blocking(move || {
        let state = handle.state::<AppState>();
        let mut doc_lock = state.doc.lock().unwrap();
        let mut backed_up = state.backed_up.lock().unwrap();
        let retention = *state.backup_retention.lock().unwrap();
        let spaces = space_paths(&state.sync)?;
        resolve_shadowed_and_persist(&mut doc_lock, &alias, &file, action, &spaces, |doc, idx| {
            persist_file(doc, idx, &mut backed_up, retention)
        })
    })
    .await
    .map_err(crate::sync::engine::join_error)?
}

/// `sync_resolve_shadowed` 的改動與寫檔(`persist` 由呼叫端注入)。寫入失敗時從磁碟重載(重載也失敗就作廢 doc),
/// 再回傳原本的錯誤 —— doc 不能比磁碟新。
fn resolve_shadowed_and_persist(
    slot: &mut Option<SshConfigDoc>,
    alias: &str,
    file: &str,
    action: ShadowedAction,
    spaces: &[PathBuf],
    mut persist: impl FnMut(&mut SshConfigDoc, usize) -> Result<(), AppError>,
) -> Result<Vec<DuplicateAlias>, AppError> {
    let doc = slot.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let main_path = doc.files[0].path.clone();
    let idx = resolve_shadowed(doc, alias, file, action, spaces)?;
    if let Err(e) = persist(doc, idx) {
        *slot = load_doc_migrated(&main_path).ok();
        return Err(e);
    }
    Ok(duplicate_aliases(doc, spaces))
}

/// 批次搬進一個 space 檔的核心迴圈(同 v1,可直接單元測試;`persist` 由呼叫端注入)。target、source、tag 三個寫入
/// 任一失敗就停止整批並請呼叫端重載。搬移前就拒絕的(wildcard、`Include`、目標檔已有同名)doc 沒動過,不停批次。
fn migrate_hosts(
    doc: &mut SshConfigDoc,
    aliases: Vec<String>,
    tag_by_file: bool,
    target: &str,
    mut persist: impl FnMut(&mut SshConfigDoc, usize) -> Result<(), AppError>,
) -> (MigrationReport, bool) {
    let mut report = MigrationReport { moved: Vec::new(), failed: Vec::new(), tagged: 0 };
    let mut halted = false;
    for alias in aliases {
        if halted {
            report.failed.push(MigrationFailure { alias, error: "not attempted: an earlier move failed".to_string() });
            continue;
        }
        let refused = refuse_wildcard(doc, &alias)
            .and_then(|()| refuse_forbidden(doc, &alias))
            .and_then(|()| refuse_already_synced(doc, Path::new(target), &alias));
        if let Err(e) = refused {
            report.failed.push(MigrationFailure { alias, error: e.to_string() });
            continue;
        }
        let source_tag = find_host_file_index(doc, &alias).filter(|&i| i != 0).map(|i| tag_for_file(&doc.files[i].path));
        match move_host(doc, &alias, target) {
            Ok((src, tgt)) => {
                if let Err(e) = persist(doc, tgt).and_then(|_| persist(doc, src)) {
                    report.failed.push(MigrationFailure { alias, error: e.to_string() });
                    halted = true;
                    continue;
                }
                if let (true, Some(tag)) = (tag_by_file, source_tag) {
                    let mut tags = find_host_mut(&mut doc.files[tgt].items, &alias).map(|h| parse_tags(&h.body)).unwrap_or_default();
                    if !tags.contains(&tag) {
                        tags.push(tag);
                        if let Some(host) = find_host_mut(&mut doc.files[tgt].items, &alias) {
                            set_tags(host, &tags);
                        }
                        match persist(doc, tgt) {
                            Ok(()) => report.tagged += 1,
                            Err(_) => halted = true,
                        }
                    }
                }
                report.moved.push(alias);
            }
            Err(e) => report.failed.push(MigrationFailure { alias, error: e.to_string() }),
        }
    }
    (report, halted)
}

/// 搬移精靈「搬進一個 space」(spec §7.2)的本體:逐台搬進這台勾選的 `space_id` 的檔案(目標先寫、再從來源移除;兩個
/// 檔案各自經存檔 hook 產生記錄)。`tag_by_file` 時把 Include 檔的檔名加成 tag。`active` = 這個行程跑著同步引擎
/// (`engine::engine_active`)。沒有引擎、目標 space 沒勾選或第一輪還沒完成時,在任何改動之前整批拒絕。
pub fn move_hosts_into_space(
    env: &SyncEnv,
    active: bool,
    aliases: Vec<String>,
    space_id: &str,
    tag_by_file: bool,
) -> Result<MigrationReport, AppError> {
    let report = {
        let mut doc_lock = env.doc.lock().unwrap();
        let mut backed_up = env.backed_up.lock().unwrap();
        let retention = env.retention();
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        refuse_while_sync_inactive(active, env.runtime)?;
        let (_, target) = selected_space_files(env.runtime, &env.ssh_dir)
            .into_iter()
            .find(|(id, _)| id == space_id)
            .ok_or_else(|| AppError::Other("that space is not synced on this device".to_string()))?;
        refuse_before_first_sync(env.runtime, space_id)?;
        if !doc.files.iter().any(|f| f.path == target) {
            return Err(AppError::Other("the space file is not loaded yet; try again in a moment".to_string()));
        }
        let main_path = doc.files[0].path.clone();
        let target = target.to_string_lossy().into_owned();
        let (report, needs_reload) =
            migrate_hosts(doc, aliases, tag_by_file, &target, |doc, idx| persist_file(doc, idx, &mut backed_up, retention));
        if needs_reload {
            // 有寫入失敗:in-memory doc 可能已經比磁碟新。重載讓兩邊一致;重載也失敗就整份作廢。
            drop(backed_up);
            *doc_lock = env.load_doc(&main_path).ok();
        }
        report
    };
    env.events.applied(0);
    env.events.wake();
    Ok(report)
}
```

- [ ] **Step 22: 修改 `src-tauri/src/config/commands.rs`**

`config_load`:任何一個勾選的 space 檔在新舊 doc 裡不同才喚醒;`config_move_host`:目標是勾選的 space 檔時套用搬進 space 的規則(spec §7.2 跨 space 搬移也走這裡,先寫目標、再從來源移除)。

`src-tauri/src/config/commands.rs`:把

```rust
    let _ = crate::tray::rebuild_tray(&app, &aliases);

    // 同步引擎用的受管檔路徑(與 `sync::engine` 相同);拿不到家目錄時只在第一次載入喚醒。
    let managed = crate::keys::ssh_dir()
        .ok()
        .map(|dir| crate::sync::hosts_file::managed_path(&dir));
    let wake = {
        let mut doc_lock = state.doc.lock().unwrap();
```

換成:

```rust
    let _ = crate::tray::rebuild_tray(&app, &aliases);

    // 這台勾選的 space 檔(與同步引擎相同);拿不到家目錄時只在第一次載入喚醒。
    let managed: Option<Vec<PathBuf>> = crate::keys::ssh_dir().ok().map(|dir| {
        crate::sync::migrate::selected_space_files(&state.sync, &dir).into_iter().map(|(_, path)| path).collect()
    });
    let wake = {
        let mut doc_lock = state.doc.lock().unwrap();
```

`src-tauri/src/config/commands.rs`:把

```rust
/// `config_load` 換上新的 doc 之後要不要喚醒同步引擎(純函式):
/// - 之前沒有 doc(第一次載入,或寫入失敗後被作廢):要 —— 引擎的輪次在 doc 是 None 時都安靜跳過。
/// - 受管檔(`managed`,與引擎同一個路徑)在新舊 doc 裡的有無或指紋不同:要 —— 這次載入帶進了 app 以外的
///   修改(例如被外部工具清空)。存檔當下的規劃(`note_file_written`)拿整個檔案去比快取,要讓同步輪次先看到
///   這次載入(例如先從 chain 重新長出被清空的檔案),下一次 app 存檔才不會對著過期的快取把每一台主機都規劃成
///   刪除。
/// - 其他情況不喚醒。前端在每次 `sync://applied` 之後都會重新載入,而引擎每次整份重載 doc(即使什麼都沒套用)
///   都會發 `sync://applied`:例如 hosts.config 存在卻載入不了(非 UTF-8、讀不到、不是一般檔案)時,
```

換成:

```rust
/// `config_load` 換上新的 doc 之後要不要喚醒同步引擎(純函式):
/// - 之前沒有 doc(第一次載入,或寫入失敗後被作廢):要 —— 引擎的輪次在 doc 是 None 時都安靜跳過。
/// - 任何一個勾選的 space 檔(`managed`,與引擎同一組路徑)在新舊 doc 裡的有無或指紋不同:要 —— 這次載入帶進了
///   app 以外的修改(例如被外部工具清空)。存檔當下的規劃(`note_file_written`)拿整個檔案去比快取,要讓同步輪次
///   先看到這次載入(例如先從 chain 重新長出被清空的檔案),下一次 app 存檔才不會對著過期的快取把每一台主機都
///   規劃成刪除。
/// - 其他情況不喚醒。前端在每次 `sync://applied` 之後都會重新載入,而引擎每次整份重載 doc(即使什麼都沒套用)
///   都會發 `sync://applied`:例如 hosts.config 存在卻載入不了(非 UTF-8、讀不到、不是一般檔案)時,
```

`src-tauri/src/config/commands.rs`:把

```rust
    previous: Option<&crate::config::model::SshConfigDoc>,
    next: &crate::config::model::SshConfigDoc,
    managed: Option<&Path>,
) -> bool {
    let Some(previous) = previous else { return true };
    let Some(managed) = managed else { return false };
    let fingerprint = |doc: &crate::config::model::SshConfigDoc| {
        doc.files.iter().find(|f| f.path == managed).map(|f| f.fingerprint.clone())
    };
    fingerprint(previous) != fingerprint(next)
}

```

換成:

```rust
    previous: Option<&crate::config::model::SshConfigDoc>,
    next: &crate::config::model::SshConfigDoc,
    managed: Option<&[PathBuf]>,
) -> bool {
    let Some(previous) = previous else { return true };
    let Some(managed) = managed else { return false };
    let fingerprints = |doc: &crate::config::model::SshConfigDoc| {
        managed
            .iter()
            .map(|path| doc.files.iter().find(|f| &f.path == path).map(|f| f.fingerprint.clone()))
            .collect::<Vec<_>>()
    };
    fingerprints(previous) != fingerprints(next)
}

```

`src-tauri/src/config/commands.rs`:把

```rust
        .as_ref()
        .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    // 拖進 sidebar 的 Synced 群組(目標是已載入的同步檔):與遷移精靈同一套規則,都在任何改動之前 —— 這個行程
    // 沒有同步引擎(別的 SSHelter 行程持有同步鎖,這裡的狀態只是啟動時的快照)就拒絕;剛 Join、第一輪同步還沒
    // 完成就拒絕;要搬的區塊有任何名字已經在同步檔裡也拒絕(重複會讓整條同步停下)。鎖順序 doc → backed_up → core。
    let managed = crate::keys::ssh_dir()
        .ok()
        .map(|dir| crate::sync::hosts_file::managed_path(&dir));
    if let Some(managed) = managed.filter(|managed| {
        doc.files
            .iter()
            .any(|f| &f.path == managed && f.path.to_string_lossy() == target_file.as_str())
    }) {
        crate::sync::migrate::refuse_while_sync_inactive(crate::sync::engine::engine_active(), &state.sync)?;
        crate::sync::migrate::refuse_before_first_sync(&state.sync)?;
        crate::sync::migrate::refuse_already_synced(doc, &managed, &alias)?;
    }
    move_host_and_persist(&mut doc_lock, &alias, &target_file, |doc, idx| {
```

換成:

```rust
        .as_ref()
        .ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    // 拖進 sidebar 的某個 space 群組(目標是已載入的、這台勾選的 space 檔;spec §7.2 跨 space 搬移也走這裡):與搬移
    // 精靈同一套規則,都在任何改動之前 —— 這個行程沒有同步引擎就拒絕;目標 space 第一輪同步還沒完成就拒絕;區塊含
    // wildcard、`Include` 或帶引號的 keyword 就拒絕;要搬的區塊有任何名字已經在目標檔裡也拒絕(重複會讓那個 space
    // 停下)。`move_host_and_persist` 先寫目標檔、再從來源移除。鎖順序 doc → backed_up → core。
    let target = crate::keys::ssh_dir().ok().and_then(|dir| {
        crate::sync::migrate::selected_space_files(&state.sync, &dir)
            .into_iter()
            .find(|(_, path)| path.to_string_lossy() == target_file.as_str() && doc.files.iter().any(|f| &f.path == path))
    });
    if let Some((space_id, path)) = target {
        crate::sync::migrate::refuse_while_sync_inactive(crate::sync::engine::engine_active(), &state.sync)?;
        crate::sync::migrate::refuse_before_first_sync(&state.sync, &space_id)?;
        crate::sync::migrate::refuse_wildcard(doc, &alias)?;
        crate::sync::migrate::refuse_forbidden(doc, &alias)?;
        crate::sync::migrate::refuse_already_synced(doc, &path, &alias)?;
    }
    move_host_and_persist(&mut doc_lock, &alias, &target_file, |doc, idx| {
```

- [ ] **Step 23: 修改 `src-tauri/src/state.rs`**

`AppState::sync` 換成 B3a 的 `runtime::SyncRuntime`。

`src-tauri/src/state.rs`:把

```rust
    /// Local MCP bridge policy, pending approvals, and recent audit events.
    pub mcp: crate::mcp::McpRuntime,
    /// 同步執行期狀態:generation/狀態/金鑰同一把鎖(`SyncCore`)、lifecycle 互斥鎖、同步中旗標。
    pub sync: crate::sync::engine::SyncRuntime,
}

```

換成:

```rust
    /// Local MCP bridge policy, pending approvals, and recent audit events.
    pub mcp: crate::mcp::McpRuntime,
    /// 同步執行期狀態:generation/狀態/帳戶金鑰同一把鎖(`SyncCore`)、lifecycle 互斥鎖、同步中旗標。
    pub sync: crate::sync::runtime::SyncRuntime,
}

```

`src-tauri/src/state.rs`:把

```rust
            close_to_tray: AtomicBool::new(false),
            mcp: crate::mcp::McpRuntime::default(),
            sync: crate::sync::engine::SyncRuntime::default(),
        }
    }
```

換成:

```rust
            close_to_tray: AtomicBool::new(false),
            mcp: crate::mcp::McpRuntime::default(),
            sync: crate::sync::runtime::SyncRuntime::default(),
        }
    }
```

- [ ] **Step 24: 修改 `src-tauri/src/lib.rs`**

註冊 v2 commands、移除 v1 commands;視窗焦點事件接到 `engine::window_focused`。

`src-tauri/src/lib.rs`:把

```rust
use settings_io::{settings_export, settings_import};
use sync::engine::{
    sync_create_chain, sync_forget_device, sync_join_chain, sync_leave_chain, sync_now,
    sync_set_device_name, sync_set_relay_url, sync_show_words, sync_status,
};
use sync::migrate::{sync_duplicate_aliases, sync_migrate_hosts, sync_resolve_shadowed};
use tauri::Manager;
use tray::tray_set_visible;
```

換成:

```rust
use settings_io::{settings_export, settings_import};
use sync::engine::{
    sync_approve, sync_check_relay, sync_create_account, sync_create_space, sync_delete_space,
    sync_dismiss_notice, sync_forget_device, sync_join_account, sync_leave_account,
    sync_move_hosts_to_space, sync_now, sync_overview, sync_pending_approvals, sync_rebuild_space,
    sync_reject, sync_rename_space, sync_select_space, sync_set_device_name, sync_set_relay_url,
    sync_show_words, sync_unselect_space,
};
use sync::migrate::{sync_duplicate_aliases, sync_resolve_shadowed};
use tauri::Manager;
use tray::tray_set_visible;
```

`src-tauri/src/lib.rs`:把

```rust
        })
        .on_window_event(|window, event| {
            if let tauri::WindowEvent::CloseRequested { api, .. } = event {
                let state = window.app_handle().state::<state::AppState>();
```

換成:

```rust
        })
        .on_window_event(|window, event| {
            // 視窗在前景時以一般間隔輪詢、回到前景立刻同步一輪;不在前景時省 relay 的配額(Sync v2)。
            if let tauri::WindowEvent::Focused(focused) = event {
                sync::engine::window_focused(window.app_handle(), *focused);
            }
            if let tauri::WindowEvent::CloseRequested { api, .. } = event {
                let state = window.app_handle().state::<state::AppState>();
```

`src-tauri/src/lib.rs`:把

```rust
            mcp_set_host_allowed,
            mcp_resolve_request,
            sync_status,
            sync_now,
            sync_create_chain,
            sync_join_chain,
            sync_show_words,
            sync_leave_chain,
            sync_set_relay_url,
            sync_set_device_name,
            sync_forget_device,
            sync_migrate_hosts,
            sync_duplicate_aliases,
            sync_resolve_shadowed,
```

換成:

```rust
            mcp_set_host_allowed,
            mcp_resolve_request,
            sync_overview,
            sync_now,
            sync_create_account,
            sync_join_account,
            sync_leave_account,
            sync_show_words,
            sync_set_relay_url,
            sync_check_relay,
            sync_set_device_name,
            sync_forget_device,
            sync_create_space,
            sync_rename_space,
            sync_delete_space,
            sync_select_space,
            sync_unselect_space,
            sync_rebuild_space,
            sync_pending_approvals,
            sync_approve,
            sync_reject,
            sync_dismiss_notice,
            sync_move_hosts_to_space,
            sync_duplicate_aliases,
            sync_resolve_shadowed,
```

- [ ] **Step 25: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 656 passed; 0 failed`(task 開始前 701)。數量有變的模組:`sync::dto` 2 → 12、`sync::engine` 24 → 6、`sync::hosts_file` 24 → 23、`sync::migrate` 18 → 13、`sync::reconcile` 21 → 1、`sync::relay` 26 → 19、`sync::state` 9 → 4、`sync::state_v2` 14 → 15。warning 只有 3 個 `dead_code`:`rotation_meta_id`、`freeze_chain`、`set_host_enabled` —— `rotation_meta_id`、`freeze_chain` 由 Task 3 的更換同步碼使用,`set_host_enabled` 是既有的。不得有其他 warning。

- [ ] **Step 26: 確認前端仍能編譯**

Run: `pnpm exec tsc --noEmit`(在 repo 根目錄)
Expected: 沒有錯誤 —— `src/bindings/SyncStatus.ts`、`SyncDevice.ts` 仍在,現有的 Sync UI 照樣型別正確(它呼叫的 v1
命令在執行時已不存在,由 B4 換掉)。

- [ ] **Step 27: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/reconcile.rs
git add src-tauri/src/sync/state_v2.rs
git add src-tauri/src/sync/state.rs
git add src-tauri/src/sync/relay.rs
git add src-tauri/src/sync/hosts_file.rs
git add src-tauri/src/sync/crypto.rs
git add src-tauri/src/sync/dto.rs
git add src-tauri/src/sync/engine.rs
git add src-tauri/src/sync/migrate.rs
git add src-tauri/src/config/commands.rs
git add src-tauri/src/state.rs
git add src-tauri/src/lib.rs
git add src-tauri/src/sync/mod.rs
git add src/bindings/DuplicateAlias.ts
git add src/bindings/PendingApprovalView.ts
git add src/bindings/RotationStep.ts
git add src/bindings/SyncDeviceView.ts
git add src/bindings/SyncFrozenView.ts
git add src/bindings/SyncOverview.ts
git add src/bindings/SyncRelayView.ts
git add src/bindings/SyncRotationView.ts
git add src/bindings/SyncSpaceView.ts
git add src/bindings/ReviewOutcome.ts
git add src/bindings/ReviewedVersion.ts
git commit -m "feat(sync): switch the app to the v2 engine and retire v1 sync"
```

---

### Task 3: 更換同步碼(spec §7.5)

> **已執行**(repo `7ce26a2` 與本節一致;`8a8c961` 修正本節自己的缺陷 —— 第 3 步重試時不再讓給這台自己的標記;兩輪審查後的修正 `8b11fe5`、`f1d462c`、`9872b9a`、`6416176`、`9251d17`、`6ca1323`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的介面與行為見 Global Constraints 的「Task 3 實際的做法」。執行後(兩個 keychain 測試都略過)是 `696 passed`(`sync::rotation` 37、`sync::engine` 9、`sync::account` 25);下面 Step 裡的數字是原本計畫的。

可中斷、可接續的長時間操作,進度在 `SyncStateV2::rotation`,新同步碼暫存在 keychain `sync:mnemonic-next`。第 1 步在
command 裡做完(`start_rotation`:relay 要有 `freeze`;產生新碼、推導新帳戶、為帳戶內**每個** space 產生新位置與金鑰,
以新帳戶金鑰加密後寫進 `rotation`);之後背景執行緒每一輪推進一步(`drive_rotation`,`round::sync_once` 在 `rotation`
存在時改呼叫它),每一步完成就持久化:2 送出本機修改(一般輪次,撞到凍結只回報;看到別台的標記就讓步;被限流或 relay 出錯時等下一次輪詢)→ 3 寫標記並凍結舊帳戶與每個舊 space
chain(**寫標記之前**就存成 `Freezing`,從此不能取消;已被別台凍結就讓步)→ 4 從凍結的 relay 取完整快照 → 5 建立新 chain
並複製(保留 version、時間戳、device_id 與 tombstone;`429` 暫停 1 小時)→ 6 刪除舊 space chain(有別台的標記就不刪)
→ 7 切換(狀態依 `previous_id` 帶過勾選、檔名、待核准;keychain 換成新碼;提示保存新同步碼)。其他電腦在 `frozen` 之後以
`rejoin_account` 輸入新同步碼。啟動時補完第 7 步的中斷。

**Files:**
- Create: `src-tauri/src/sync/rotation.rs`
- Modify: `src-tauri/src/sync/round.rs`(`sync_once` 在 `rotation` 存在時改由 `drive_rotation` 推進)
- Modify: `src-tauri/src/sync/engine.rs`(`startup` 補完中斷的切換;3 個 commands;啟動補完的測試)
- Modify: `src-tauri/src/lib.rs`、`src-tauri/src/sync/mod.rs`

**Interfaces:**
- Consumes(B3a、Task 1–2):`round::{run_round, sync_once, next_delay, RoundOutcome}`(`markers` / `backoff` 見 Global
  Constraints)、`round::tests::{pair, settle, rotate_elsewhere}`、
  `account::{account_ready, check_relay, selected_ids, NO_ACCOUNT_MESSAGE}`、`merge::{account_outgoing, merge_account,
  plan_device, push_outgoing, put_account_record, space_deleted_by, space_entries, space_keys, SpaceEntry}`(`push_outgoing`
  回 `Pushed`,見 Global Constraints)、
  `runtime::{mutate, save_core, snapshot}`、`spaces::create_space`(測試)、`engine::startup`、`engine::run_then_overview`。
- Consumes(B2):`crypto::{generate_mnemonic, normalize_mnemonic, derive_account, ChainKeys}`;`reconcile::{decode, encode}`;
  `record::{rotation_meta_id, MetaPayload, Record, RecordKind, RotationMarkerPayload, SpaceKeyPayload, SpacePayload,
  ACCOUNT_META_ID, SCHEMA_VERSION}`;`relay::{PushOutcome, RelayApi, RelayError, FEATURE_FREEZE}`;
  `state_v2::{RotationProgress, RotationStep, RotatedSpace, SealedRecord, FreezeInfo, NEXT_MNEMONIC_ACCOUNT}`;
  `state::MNEMONIC_ACCOUNT`。
- Produces(`rotation`):`pub const NO_FREEZE_MESSAGE: &str`、`pub const NOT_CANCELLABLE_MESSAGE: &str`;
  `pub fn start_rotation(env: &SyncEnv) -> Result<(), AppError>`;`pub fn cancel_rotation(env: &SyncEnv) -> Result<(), AppError>`;
  `pub fn drive_rotation(env: &SyncEnv, generation: u64, s: SyncStateV2, keys: ChainKeys) -> Result<(), AppError>`;
  `pub fn finish_interrupted_switch(keychain: &dyn Keychain, chain_id: &str) -> Option<ChainKeys>`;
  `pub fn rejoin_account(env: &SyncEnv, words: &str) -> Result<(), AppError>`。
- Produces(commands):`sync_change_sync_code() -> SyncOverview`、`sync_cancel_sync_code_change() -> SyncOverview`、
  `sync_rejoin_account(words: String) -> SyncOverview`。

- [ ] **Step 1: 寫失敗的測試:`src-tauri/src/sync/rotation.rs`**

情境測試:兩台(或三台)裝置共用一個 `FakeRelay`;`tick` 推進一步、`finish` 推進到完成。涵蓋每一步、取消、限流暫停後接續、讓步、同時更換、凍結期間別台的修改、快照裡的待核准記錄、relay 沒有 `freeze`。

`a_marker_seen_while_this_device_still_sends_its_changes_yields_at_once`、`a_rate_limited_step_two_waits_for_the_next_poll_instead_of_rerunning_at_once`、`a_step_two_round_that_cannot_send_anything_does_not_rerun_at_once`、`a_space_over_its_storage_limit_does_not_hold_up_the_change` 釘住第 2 步依 B3a Task 4 的 `RoundOutcome`:看到標記就讓步、退避時不立刻重跑、沒有往下走就不 `wake`、儲存額度滿的 space 不擋住更換。

建立 `src-tauri/src/sync/rotation.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! 更換同步碼(spec §7.5):可中斷、可接續的長時間操作,進度在 `SyncStateV2::rotation`,新同步碼暫存在 keychain
//! `sync:mnemonic-next`。背景執行緒每一輪推進一步(`drive_rotation`),每一步完成就持久化,重啟後接著做:
//! 1 準備(`start_rotation`)→ 2 送出本機修改 → 3 標記與凍結 → 4 從凍結的 relay 取完整快照 → 5 建立與複製 →
//! 6 刪除舊 space chain → 7 切換。第 3 步之前可以取消;之後其他電腦已被擋下,只能做完。其他電腦偵測到更換後
//! (`frozen`)以新同步碼重新加入(`rejoin_account`):依 `previous_id` 保留勾選、檔名、待核准項目,未上傳的修改沿用原
//! 時間戳帶進新 space,第一輪以一般 LWW 合併。

use std::collections::BTreeMap;

use crate::error::AppError;
use crate::sync::account::{account_ready, check_relay, selected_ids, NO_ACCOUNT_MESSAGE};
use crate::sync::crypto::{self, ChainKeys};
use crate::sync::env::SyncEnv;
use crate::sync::merge::{
    account_outgoing, merge_account, plan_device, push_outgoing, put_account_record, space_deleted_by, space_entries,
    space_keys, SpaceEntry,
};
use crate::sync::reconcile::{decode, encode};
use crate::sync::record::{
    rotation_meta_id, MetaPayload, Record, RecordKind, RotationMarkerPayload, SpaceKeyPayload, SpacePayload,
    ACCOUNT_META_ID, SCHEMA_VERSION,
};
use crate::sync::relay::{PushOutcome, RelayApi, RelayError, FEATURE_FREEZE};
use crate::sync::runtime::{mutate, save_core};
use crate::sync::state::MNEMONIC_ACCOUNT;
use crate::sync::state_v2::{
    AccountState, FreezeInfo, RotatedSpace, RotationProgress, RotationStep, SealedRecord, SpaceState, SyncNotice,
    SyncStateV2, NEXT_MNEMONIC_ACCOUNT,
};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::merge::space_entries;
    use crate::sync::record::rotation_meta_id;
    use crate::sync::round::{next_delay, sync_once};
    use crate::sync::round::tests::{pair, settle};
    use crate::sync::spaces::create_space;
    use crate::sync::testkit::TestDevice;

    fn step(d: &TestDevice) -> Option<RotationStep> {
        d.state().rotation.map(|r| r.step)
    }

    /// 推進一步(背景執行緒的一輪)。
    fn tick(d: &TestDevice) {
        let _ = sync_once(&d.env());
    }

    /// 推進到更換完成,再跑到沒有要立刻重跑的輪次。
    fn finish(d: &TestDevice) {
        for _ in 0..30 {
            if d.state().rotation.is_none() {
                settle(d);
                return;
            }
            tick(d);
        }
        panic!("the rotation never finished: {:?}", step(d));
    }

    fn new_code(d: &TestDevice) -> String {
        d.keychain.entry(NEXT_MNEMONIC_ACCOUNT).expect("the new sync code waits in the keychain")
    }

    fn entry_named(d: &TestDevice, name: &str) -> SpaceEntry {
        space_entries(d.state().account.as_ref().unwrap()).into_iter().find(|e| e.name == name && !e.deleted).unwrap()
    }

    #[test]
    fn changing_the_sync_code_moves_every_space_to_a_new_account() {
        let (relay, _clock, a, _b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        let work = create_space(&a.env(), "Work").unwrap();
        a.save_in_app(&a.space_path(&work), "Host db\n");
        settle(&a);
        let old_account = a.state().account.unwrap().chain_id;
        let personal_file = a.space_path(&personal);
        start_rotation(&a.env()).unwrap();
        let next = new_code(&a);
        assert_eq!(step(&a), Some(RotationStep::Prepared));
        assert_eq!(a.state().rotation.unwrap().spaces.len(), 2, "every space, selected or not");
        finish(&a);
        let s = a.state();
        assert_eq!(a.keychain.entry(MNEMONIC_ACCOUNT), Some(next.clone()));
        assert!(a.keychain.entry(NEXT_MNEMONIC_ACCOUNT).is_none());
        assert_eq!(s.account.as_ref().unwrap().chain_id, crypto::derive_account(&next).unwrap().chain_id);
        let new_personal = entry_named(&a, "Personal");
        assert_eq!(new_personal.previous_id.as_deref(), Some(personal.as_str()));
        assert_eq!(entry_named(&a, "Work").previous_id.as_deref(), Some(work.as_str()));
        assert_eq!(a.space_path(&new_personal.id), personal_file, "the file name is kept");
        assert_eq!(a.read(&personal_file), "Host web\n");
        assert!(s.notices.contains(&SyncNotice::NewSyncCode));
        // relay:舊帳戶凍結並帶著標記;舊 space chain 的記錄已刪除、但仍然凍結;新 chain 有複製過去的記錄。
        assert!(relay.is_frozen(&old_account));
        for old in [&personal, &work] {
            assert!(relay.rows(old).is_empty() && relay.is_frozen(old));
        }
        assert!(!relay.rows(&new_personal.id).is_empty());
        assert!(!relay.rows(&entry_named(&a, "Work").id).is_empty(), "a space this device does not sync is copied too");
        // 之後照常同步到新 chain。
        a.save_in_app(&personal_file, "Host web\n  User x\n");
        settle(&a);
        assert!(a.state().spaces[&new_personal.id].records.values().all(|l| !l.dirty));
    }

    #[test]
    fn other_devices_freeze_and_rejoin_with_their_unsent_edits() {
        let (relay, _clock, a, b, words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        settle(&b);
        start_rotation(&a.env()).unwrap();
        let next = new_code(&a);
        finish(&a);
        // B 改了 web 還沒上傳,下一輪就發現帳戶已被更換:不上傳、記下 frozen。
        let file = b.space_path(&personal);
        b.save_in_app(&file, "Host web\n  User b\n");
        relay.clear_calls();
        settle(&b);
        let frozen = b.state().frozen().cloned().unwrap();
        assert_eq!(frozen.markers.iter().map(|m| m.by_device_name.as_str()).collect::<Vec<_>>(), vec!["MacBook-A"]);
        assert!(!relay.calls().iter().any(|c| c.starts_with("push:")));
        // 已刪除的舊 space chain 仍然凍結:就算有電腦沒先拉帳戶就推送,也寫不進去。
        let old_space = {
            let core = b.runtime.core.lock().unwrap();
            space_keys(core.state.as_ref().unwrap().account.as_ref().unwrap(), core.account_keys.as_ref().unwrap(), &personal).unwrap()
        };
        let stale = encode(&old_space, &Record { kind: RecordKind::Host, id: "x".into(), version: 1, updated_at_ms: 1, device_id: "z".into(), deleted: false, payload: serde_json::json!({ "schema": 1, "text": "Host x\n" }) }, 0).unwrap();
        assert_eq!(relay.push(&old_space.chain_id, &old_space.auth_token, &[stale]).unwrap(), PushOutcome::Frozen);
        assert!(rejoin_account(&b.env(), &words).is_err(), "the old code is refused");
        rejoin_account(&b.env(), &next).unwrap();
        let s = b.state();
        assert!(s.frozen().is_none());
        let new_personal = entry_named(&b, "Personal").id;
        assert_eq!(b.space_path(&new_personal), file, "selection and file name follow previous_id");
        assert!(s.spaces[&new_personal].records["host:web"].dirty, "the unsent edit comes along with its timestamp");
        assert_eq!(b.keychain.entry(MNEMONIC_ACCOUNT), Some(next));
        settle(&b);
        settle(&a);
        assert_eq!(a.read(&a.space_path(&new_personal)), "Host web\n  User b\n");
    }

    #[test]
    fn the_snapshot_holds_changes_and_held_records_the_rotating_device_never_applied() {
        let (relay, _clock, a, b, _words, personal) = pair();
        // B 先推一台要核准的主機:A 保留它(待核准)。
        b.save_in_app(&b.space_path(&personal), "Host jump\n  ProxyCommand nc %h 22\n");
        settle(&b);
        start_rotation(&a.env()).unwrap();
        tick(&a); // 第 2 步:一般輪次(A 拉到 jump,保留待核准)
        assert_eq!(step(&a), Some(RotationStep::LocalChangesSent));
        assert!(a.state().spaces[&personal].pending_approvals.contains_key("jump"));
        // 凍結之前 B 又推了一台:A 從來沒拉到它,快照照樣包含它。
        b.save_in_app(&b.space_path(&personal), "Host jump\n  ProxyCommand nc %h 22\nHost db\n");
        settle(&b);
        tick(&a); // 第 3 步:標記與凍結
        assert_eq!(step(&a), Some(RotationStep::Copying));
        // 凍結之後 B 的修改被擋下。
        b.save_in_app(&b.space_path(&personal), "Host jump\n  ProxyCommand nc %h 22\nHost db\nHost late\n");
        settle(&b);
        assert!(b.state().frozen().is_some());
        finish(&a);
        let new_personal = entry_named(&a, "Personal").id;
        assert!(a.read(&a.space_path(&new_personal)).contains("Host db"), "the change A never pulled was copied");
        assert!(a.state().spaces[&new_personal].pending_approvals.contains_key("jump"), "the held record is still waiting");
        assert!(relay.rows(&new_personal).len() >= 2);
        // B 重新加入:被擋下的修改送進新帳戶。
        rejoin_account(&b.env(), &a.keychain.entry(MNEMONIC_ACCOUNT).unwrap()).unwrap();
        settle(&b);
        settle(&a);
        assert!(a.read(&a.space_path(&new_personal)).contains("Host late"));
    }

    #[test]
    fn a_space_created_elsewhere_during_the_rotation_is_frozen_and_copied_too() {
        let (relay, _clock, a, b, _words, _personal) = pair();
        start_rotation(&a.env()).unwrap();
        tick(&a); // 第 2 步之後,A 不會再拉帳戶
        let lab = create_space(&b.env(), "Lab").unwrap();
        b.save_in_app(&b.space_path(&lab), "Host gpu\n");
        settle(&b);
        tick(&a); // 第 3 步
        assert!(relay.is_frozen(&lab), "a space this device never pulled is frozen as well");
        finish(&a);
        let copied = entry_named(&a, "Lab");
        assert_eq!(copied.previous_id.as_deref(), Some(lab.as_str()));
        assert_eq!(relay.rows(&copied.id).len(), 1);
    }

    #[test]
    fn a_rotation_is_cancellable_only_before_freezing_and_resumes_after_a_pause() {
        let (relay, clock, a, _b, _words, _personal) = pair();
        start_rotation(&a.env()).unwrap();
        cancel_rotation(&a.env()).unwrap();
        assert!(a.state().rotation.is_none() && a.keychain.entry(NEXT_MNEMONIC_ACCOUNT).is_none());
        start_rotation(&a.env()).unwrap();
        tick(&a);
        assert_eq!(step(&a), Some(RotationStep::LocalChangesSent));
        cancel_rotation(&a.env()).unwrap();
        start_rotation(&a.env()).unwrap();
        tick(&a);
        tick(&a);
        assert_eq!(step(&a), Some(RotationStep::Copying));
        assert_eq!(cancel_rotation(&a.env()).unwrap_err().to_string(), NOT_CANCELLABLE_MESSAGE);
        // 建立 chain 被限流:暫停到下個小時,期間什麼都不做,之後自動接續。
        relay.fail_creates_with_429(1);
        tick(&a);
        let paused = a.state().rotation.unwrap();
        assert_eq!(paused.step, RotationStep::Copying);
        assert!(paused.paused_until_ms.is_some());
        relay.clear_calls();
        tick(&a);
        assert!(relay.calls().is_empty(), "nothing happens while paused");
        clock.advance(2 * CREATE_PAUSE_MS);
        finish(&a);
        assert!(a.state().notices.contains(&SyncNotice::NewSyncCode));
    }

    #[test]
    fn a_device_that_loses_the_race_yields_to_the_other_rotation() {
        let (_relay, _clock, a, b, _words, _personal) = pair();
        start_rotation(&a.env()).unwrap();
        start_rotation(&b.env()).unwrap();
        tick(&a);
        tick(&b);
        tick(&a); // A 寫標記並凍結
        tick(&b); // B 的標記寫不進去:讓給 A
        let s = b.state();
        assert!(s.rotation.is_none());
        assert_eq!(s.frozen().unwrap().markers[0].by_device_name, "MacBook-A");
        assert!(b.keychain.entry(NEXT_MNEMONIC_ACCOUNT).is_none());
        let next = new_code(&a);
        finish(&a);
        rejoin_account(&b.env(), &next).unwrap();
        assert!(b.state().frozen().is_none());
    }

    #[test]
    fn a_marker_seen_while_this_device_still_sends_its_changes_yields_at_once() {
        let (relay, _clock, a, b, _words, personal) = pair();
        // A 開始更換時還有沒上傳的修改:第 2 步要先送出它們。
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        start_rotation(&a.env()).unwrap();
        // B 先做完了第 3 步(寫標記、凍結):A 的第 2 步拉帳戶時就看到 B 的標記。
        start_rotation(&b.env()).unwrap();
        tick(&b);
        tick(&b);
        assert_eq!(step(&b), Some(RotationStep::Copying));
        relay.clear_calls();
        tick(&a);
        // A 還沒寫標記、沒凍結任何東西:讓給 B(不會停在第 2 步),修改留著等輸入 B 的新同步碼。
        let s = a.state();
        assert!(s.rotation.is_none() && a.keychain.entry(NEXT_MNEMONIC_ACCOUNT).is_none());
        assert_eq!(s.frozen().unwrap().markers[0].by_device_name, "MacBook-B");
        assert!(s.spaces[&personal].records["host:web"].dirty);
        assert!(!relay.calls().iter().any(|c| c.starts_with("push:")), "nothing was uploaded after the marker was seen");
        let next = new_code(&b);
        finish(&b);
        rejoin_account(&a.env(), &next).unwrap();
        settle(&a);
        settle(&b);
        let new_personal = entry_named(&b, "Personal").id;
        assert_eq!(b.read(&b.space_path(&new_personal)), "Host web\n", "A's unsent change arrives in the new account");
    }

    #[test]
    fn a_rate_limited_step_two_waits_for_the_next_poll_instead_of_rerunning_at_once() {
        let (relay, _clock, a, _b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        start_rotation(&a.env()).unwrap();
        relay.fail_batches_with_429(3);
        for round in 1..=3 {
            let wakes = a.events.wakes();
            relay.clear_calls();
            tick(&a);
            assert_eq!(step(&a), Some(RotationStep::Prepared), "round {round}");
            assert_eq!(a.events.wakes(), wakes, "round {round}: no immediate rerun after a 429");
            assert_eq!(relay.calls().len(), 1, "round {round}: one rejected batch and nothing else: {:?}", relay.calls());
        }
        // 背景執行緒等的是退避後的間隔(連續 3 輪:6 分鐘)。
        assert!(next_delay(&a.env()).as_secs() >= 360, "{:?}", next_delay(&a.env()));
        // 限流解除之後照常往下走。
        tick(&a);
        assert_eq!(step(&a), Some(RotationStep::LocalChangesSent));
        assert!(a.state().spaces[&personal].records.values().all(|l| !l.dirty));
    }

    #[test]
    fn a_step_two_round_that_cannot_send_anything_does_not_rerun_at_once() {
        let (_relay, _clock, a, _b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        start_rotation(&a.env()).unwrap();
        // 另一台把帳戶升到這版讀不懂的格式:這台變成唯讀、送不出任何東西 —— 第 2 步不能往下走,也不能一直立刻重跑。
        a.runtime.core.lock().unwrap().state.as_mut().unwrap().account.as_mut().unwrap().remote_schema_version = Some(u32::MAX);
        let wakes = a.events.wakes();
        tick(&a);
        assert_eq!(step(&a), Some(RotationStep::Prepared));
        assert_eq!(a.events.wakes(), wakes, "the step did not advance: wait for the next poll");
        assert!(a.state().spaces[&personal].records["host:web"].dirty);
    }

    #[test]
    fn a_space_over_its_storage_limit_does_not_hold_up_the_change() {
        let (relay, _clock, a, _b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        start_rotation(&a.env()).unwrap();
        relay.set_push_quota(Some(0));
        let wakes = a.events.wakes();
        tick(&a);
        // 這個 space 的記錄送不出去:錯誤記在 space 上(使用者看得到),第 2 步照樣做完 —— 它們跟著切換帶進新 space。
        let message = a.state().spaces[&personal].last_error.clone().unwrap();
        assert!(message.contains("\"Personal\"") && message.contains("storage limit"), "{message}");
        assert_eq!(step(&a), Some(RotationStep::LocalChangesSent));
        assert_eq!(a.events.wakes(), wakes, "the round ended in backoff: the next step waits for the next poll");
        relay.set_push_quota(None);
        finish(&a);
        let new_personal = entry_named(&a, "Personal").id;
        assert_eq!(a.read(&a.space_path(&new_personal)), "Host web\n");
        let s = a.state();
        assert!(s.spaces[&new_personal].records.values().all(|l| !l.dirty), "uploaded to the new chain");
        assert!(s.spaces[&new_personal].last_error.is_none());
        assert!(!relay.rows(&new_personal).is_empty());
    }

    #[test]
    fn concurrent_markers_keep_the_old_chains_and_tell_the_device() {
        let (relay, _clock, a, _b, _words, personal) = pair();
        start_rotation(&a.env()).unwrap();
        tick(&a);
        // 另一台(Y)在 A 凍結之前也寫了標記。
        let keys = a.runtime.core.lock().unwrap().account_keys.clone().unwrap();
        let mut y = AccountState::new(&keys.chain_id);
        let marker = RotationMarkerPayload { rotated_at_ms: 1, by_device_id: "dev-y".into(), by_device_name: "MacBook-Y".into() };
        put_account_record(&mut y, RecordKind::Meta, &rotation_meta_id("dev-y"), serde_json::to_value(marker).unwrap(), false, "dev-y", 1);
        assert!(push_outgoing(relay.as_ref(), &keys.chain_id, &keys.auth_token, &account_outgoing(&y, &keys).unwrap()).error.is_none());
        finish(&a);
        assert!(relay.exists(&personal), "another rotation may still copy the old chains");
        assert!(a.state().notices.contains(&SyncNotice::OtherRotation { devices: vec!["MacBook-Y".into()] }));
    }

    #[test]
    fn the_relay_must_be_able_to_freeze() {
        let (relay, _clock, a, _b, _words, _personal) = pair();
        relay.set_legacy(true);
        a.runtime.core.lock().unwrap().state.as_mut().unwrap().relay_features = None;
        assert_eq!(start_rotation(&a.env()).unwrap_err().to_string(), NO_FREEZE_MESSAGE);
        assert!(a.state().rotation.is_none());
        assert!(a.keychain.entry(NEXT_MNEMONIC_ACCOUNT).is_none());
    }
}
```

- [ ] **Step 2: `src-tauri/src/sync/engine.rs` 的測試**

啟動時補完中斷的第 7 步。

`src-tauri/src/sync/engine.rs`:把

```rust

    #[test]
    fn an_unreadable_state_file_is_set_aside_instead_of_overwritten() {
        // 只用暫存目錄,絕不碰真正的 app data。
```

換成:

```rust

    #[test]
    fn startup_finishes_a_sync_code_switch_that_was_interrupted() {
        // 狀態已換成新帳戶,keychain 還是舊碼、新碼還在 `sync:mnemonic-next`。
        let new_words = crate::sync::crypto::generate_mnemonic().unwrap();
        let keychain = MemKeychain::default();
        keychain.set(MNEMONIC_ACCOUNT, WORDS).unwrap();
        keychain.set(crate::sync::state_v2::NEXT_MNEMONIC_ACCOUNT, &new_words).unwrap();
        let account = crate::sync::crypto::derive_account(&new_words).unwrap();
        let mut v2 = SyncStateV2::fresh("Box").unwrap();
        v2.account = Some(crate::sync::state_v2::AccountState::new(&account.chain_id));
        let start = startup(Ok(LoadedState::Current(Box::new(v2))), &keychain, "Box").unwrap();
        assert_eq!(start.keys.unwrap().chain_id, account.chain_id);
        assert!(start.state.last_error.is_none());
        assert_eq!(keychain.entry(MNEMONIC_ACCOUNT), Some(new_words));
        assert_eq!(keychain.entry(crate::sync::state_v2::NEXT_MNEMONIC_ACCOUNT), None);
    }

    #[test]
    fn an_unreadable_state_file_is_set_aside_instead_of_overwritten() {
        // 只用暫存目錄,絕不碰真正的 app data。
```

- [ ] **Step 3: 更新 `src-tauri/src/sync/mod.rs`**

把 `src-tauri/src/sync/mod.rs` 整個換成:

```rust
//! Sync(Sync v2 spaces spec):端對端加密的多 space 同步。各子模組單一責任、皆可單元測試:
//! - `crypto`: 同步碼、金鑰推導、記錄加密
//! - `record`: 記錄模型與 LWW 合併
//! - `planner`: 本機變更偵測
//! - `hosts_file`: 同步檔的區塊操作、主 config 的 Include 清單、禁用的 directive
//! - `space_files`: space 檔命名、Include 清單順序與建立 / 移除 / 改名的順序規則
//! - `approval`: 危險設定的核准簽章
//! - `relay`: relay HTTP client(`RelayApi`)與輪詢間隔
//! - `reconcile`: 記錄的加解密編碼與套到檔案的效果
//! - `merge`: 帳戶與 space 區段的本機 diff、合併、上傳(純函式)
//! - `state`: v1 狀態(只為了升級)、狀態檔路徑、同步碼的 keychain account
//! - `state_v2`: 本機狀態(`version: 2`)與 v1 狀態檔的偵測
//! - `runtime`: `SyncCore`(generation / 狀態 / 帳戶金鑰)與局部提交
//! - `env`: 引擎與外界的邊界(keychain、relay、事件、時鐘)
//! - `files`: space 檔的準備、讀取、套用 + 發布交易、存檔 hook
//! - `account`: 帳戶生命週期與 relay 設定
//! - `spaces`: space 操作與核准
//! - `round`: 一輪同步
//! - `upgrade`: 從 v1 升級
//! - `rotation`: 更換同步碼
//! - `migrate`: 主機搬進 space、跨檔案的同名主機
//! - `dto`: 給前端的事件與狀態形狀
//! - `engine`: Tauri 外殼(同步鎖、啟動、背景執行緒、存檔 hook、commands)
//! - `fake_relay`、`testkit`(只在測試):記憶體假 relay 與測試裝置

pub mod account;
pub mod approval;
pub mod crypto;
pub mod dto;
pub mod engine;
pub mod env;
#[cfg(test)]
pub mod fake_relay;
pub mod files;
pub mod hosts_file;
pub mod merge;
pub mod migrate;
pub mod planner;
pub mod reconcile;
pub mod record;
pub mod relay;
pub mod rotation;
pub mod round;
pub mod runtime;
pub mod space_files;
pub mod spaces;
pub mod state;
pub mod state_v2;
#[cfg(test)]
pub mod testkit;
pub mod upgrade;
```

- [ ] **Step 4: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0425]: cannot find value `NOT_CANCELLABLE_MESSAGE` in this scope
--> src/sync/rotation.rs:205:72
```

- [ ] **Step 5: 實作 `src-tauri/src/sync/rotation.rs`**

重點:進度只由背景執行緒推進(`update_rotation` 不換 generation;取消在 core 鎖內比對步驟);`freeze` 從 relay 上的舊帳戶列出 space;`copy` 每完成一個 space 就記下(重跑時已寫過的列回 conflict,視為已複製);`install_new_account` 在 doc 鎖內以**最新**的狀態帶過 space,再換 keychain。

複製與新帳戶的上傳依 Task 1 實際的介面:`push_outgoing` 回的 `Pushed` 帶著 `error` 時就回傳那個錯誤(同以前的 `?`)。

第 2 步(`send_local_changes`)依 `RoundOutcome`:`markers` 非空就 `yield_to_other_rotation`(它改成直接收標記;第 3 步寫不進標記時才另外拉帳戶取得);往下走了、而且不是 `backoff` 才 `wake`。

`src-tauri/src/sync/rotation.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
pub const NO_FREEZE_MESSAGE: &str = "this relay cannot change the sync code yet; update the relay first";
pub const NOT_CANCELLABLE_MESSAGE: &str = "changing the sync code can no longer be cancelled; it will finish on its own";
const NEXT_CODE_MISSING_MESSAGE: &str = "the new sync code is missing from the keychain; unlock the keychain and restart SSHelter";
/// 建立 chain 被限流(每 IP 每小時 20 次)時暫停多久再接續(spec §6.6、§7.5)。
const CREATE_PAUSE_MS: u64 = 60 * 60 * 1000;

/// 只改 `rotation` 的進度(不換 generation:進度只由背景執行緒推進,取消在 core 鎖內比對步驟)。`f` 回 false 表示這次
/// 轉換不成立(已被取消或步驟不符),什麼都不改。
fn update_rotation(env: &SyncEnv, f: impl FnOnce(&mut RotationProgress) -> bool) -> Result<bool, AppError> {
    let mut core = env.runtime.core.lock().unwrap();
    let Some(rotation) = core.state.as_mut().and_then(|s| s.rotation.as_mut()) else { return Ok(false) };
    if !f(rotation) {
        return Ok(false);
    }
    save_core(&mut core, &env.state_path)?;
    Ok(true)
}

/// 新帳戶的 `spacekey` 記錄(id = 新 space id),以**新帳戶金鑰**加密。
fn new_space_key(new_account: &ChainKeys, space: &ChainKeys, device_id: &str, now_ms: u64) -> Result<SealedRecord, AppError> {
    let record = Record {
        kind: RecordKind::SpaceKey,
        id: space.chain_id.clone(),
        version: 1,
        updated_at_ms: now_ms,
        device_id: device_id.to_string(),
        deleted: false,
        payload: serde_json::to_value(SpaceKeyPayload::from_keys(space)).expect("SpaceKeyPayload serializes"),
    };
    SealedRecord::seal(new_account, &record, 0)
}

/// 帳戶裡還在、而且解得開金鑰的 space。
fn live_spaces(account: &AccountState, keys: &ChainKeys) -> Vec<SpaceEntry> {
    space_entries(account)
        .into_iter()
        .filter(|e| !e.deleted && space_deleted_by(account, keys, &e.id).is_none() && space_keys(account, keys, &e.id).is_some())
        .collect()
}

/// 第 1 步(spec §7.5):relay 要支援凍結 → 產生新同步碼(存 `sync:mnemonic-next`)→ 推導新帳戶 → 為帳戶內**每個**
/// space(含這台沒勾選的)產生新的 chain id、權杖、金鑰,以新帳戶金鑰加密後寫進 `rotation`。背景執行緒接著做。
pub fn start_rotation(env: &SyncEnv) -> Result<(), AppError> {
    let (s, keys) = {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        account_ready(&s, core.account_keys.as_ref())?;
        (s, core.account_keys.clone().expect("checked by account_ready"))
    };
    let supported = match s.relay_features.as_ref().filter(|f| f.url == s.relay_url) {
        Some(f) => f.supports(FEATURE_FREEZE),
        None => check_relay(env)?.supports(FEATURE_FREEZE),
    };
    if !supported {
        return Err(AppError::Other(NO_FREEZE_MESSAGE.to_string()));
    }
    let words = crypto::generate_mnemonic()?;
    let new_account = crypto::derive_account(&words)?;
    let now = env.now();
    let mut progress = RotationProgress::new(&new_account.chain_id, now);
    for entry in live_spaces(s.account.as_ref().expect("joined"), &keys) {
        let space = ChainKeys::generate()?;
        let sealed_key = new_space_key(&new_account, &space, &s.device_id, now)?;
        progress.spaces.insert(entry.id, RotatedSpace { new_space_id: space.chain_id, sealed_key });
    }
    env.keychain.set(NEXT_MNEMONIC_ACCOUNT, &words)?;
    mutate(env, |s| {
        if s.rotation.is_some() {
            return Err(AppError::Other("the sync code is already being changed".to_string()));
        }
        s.rotation = Some(progress);
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 取消(spec §7.5):只能在第 3 步之前(還沒寫標記、沒凍結任何東西)。新 chain 在第 5 步才建立,所以這時還沒有要刪的
/// chain;清掉暫存的同步碼,回到原狀。
pub fn cancel_rotation(env: &SyncEnv) -> Result<(), AppError> {
    mutate(env, |s| match s.rotation.as_ref() {
        None => Err(AppError::Other("the sync code is not being changed".to_string())),
        Some(r) if !r.cancellable() => Err(AppError::Other(NOT_CANCELLABLE_MESSAGE.to_string())),
        Some(_) => {
            s.rotation = None;
            Ok(())
        }
    })?;
    env.keychain.delete(NEXT_MNEMONIC_ACCOUNT)?;
    env.events.wake();
    Ok(())
}

/// 新同步碼:keychain 的 `sync:mnemonic-next`;推導出的帳戶必須就是這次更換的新帳戶。
fn new_words(env: &SyncEnv, rotation: &RotationProgress) -> Result<(String, ChainKeys), AppError> {
    let words = env.keychain.get(NEXT_MNEMONIC_ACCOUNT)?.ok_or_else(|| AppError::Other(NEXT_CODE_MISSING_MESSAGE.to_string()))?;
    let keys = crypto::derive_account(&words).map_err(|_| AppError::Other(NEXT_CODE_MISSING_MESSAGE.to_string()))?;
    if keys.chain_id != rotation.new_account_chain_id {
        return Err(AppError::Other(NEXT_CODE_MISSING_MESSAGE.to_string()));
    }
    Ok((words, keys))
}

/// 這台還有能上傳、還沒上傳的記錄(帳戶與勾選的 space)。暫停中的 space(違反不變式、chain 不見了、儲存額度滿了或上傳被
/// relay 拒絕 —— 都記在那個 space 的 `last_error`,使用者看得到)送不出去,不擋住更換:它們的 dirty 記錄在切換時照樣帶進
/// 新 space,第一輪再上傳。
fn has_dirty(s: &SyncStateV2) -> bool {
    s.account.as_ref().is_some_and(|a| a.records.values().any(|l| l.dirty) || a.sealed.values().any(|x| x.dirty))
        || s.spaces
            .values()
            .any(|sp| sp.selected && sp.last_error.is_none() && !sp.missing && sp.records.values().any(|l| l.dirty))
}

/// 背景執行緒在 `rotation` 存在時呼叫(取代一般輪次):推進一步。每一步完成就持久化並要求立刻再跑一輪(第 2 步見
/// `send_local_changes`)。
pub fn drive_rotation(env: &SyncEnv, generation: u64, s: SyncStateV2, keys: ChainKeys) -> Result<(), AppError> {
    let Some(rotation) = s.rotation.clone() else { return Ok(()) };
    if rotation.paused_until_ms.is_some_and(|until| env.now() < until) {
        return Ok(());
    }
    let relay = env.relay(&s.relay_url)?;
    match rotation.step {
        RotationStep::Prepared => return send_local_changes(env, generation, s, keys),
        RotationStep::LocalChangesSent => {
            // **寫標記之前**就存成 Freezing:從這裡起不能取消(與取消在同一把鎖內比對步驟)。
            if !update_rotation(env, |r| {
                let ok = r.step == RotationStep::LocalChangesSent;
                if ok {
                    r.step = RotationStep::Freezing;
                }
                ok
            })? {
                return Ok(());
            }
            freeze(env, &s, &keys, relay.as_ref())?;
        }
        RotationStep::Freezing => freeze(env, &s, &keys, relay.as_ref())?,
        RotationStep::Copying => copy(env, &s, &keys, &rotation, relay.as_ref())?,
        RotationStep::Deleting => delete_old(env, &s, &keys, &rotation, relay.as_ref())?,
        RotationStep::Switching => switch(env, &s, &keys, &rotation, relay.as_ref())?,
    }
    env.events.wake();
    Ok(())
}

/// 第 2 步:一般輪次把這台的 dirty 記錄送出(`mark_frozen_chains` = false:這台自己正在更換,撞到凍結不記進狀態)。
/// - 拉帳戶時看到別台的更換標記(`RoundOutcome::markers`):對方先開始了,這台還沒寫標記、也還沒凍結任何東西 —— 直接讓給
///   那一次更換(同第 3 步寫不進標記時)。
/// - 撞到凍結的 chain(`frozen`)也算做完:別台已經凍結了,只是送不出去;第 3 步寫標記時會發現並讓步。
/// - 沒有能送的 dirty 記錄了(`has_dirty`)才往下走,而且只有往下走了才要求立刻再跑;以退避收尾的一輪(`backoff`:被限流、
///   relay 出錯)絕不立刻重跑 —— 背景執行緒等 `next_delay`,不會一直對 relay 發請求。
fn send_local_changes(env: &SyncEnv, generation: u64, s: SyncStateV2, keys: ChainKeys) -> Result<(), AppError> {
    let outcome = crate::sync::round::run_round(env, generation, s, keys, false)?;
    if !outcome.markers.is_empty() {
        return yield_to_other_rotation(env, outcome.markers);
    }
    let latest = crate::sync::runtime::snapshot(env).ok_or_else(crate::sync::runtime::superseded)?;
    let advanced = (outcome.frozen || !has_dirty(&latest))
        && update_rotation(env, |r| {
            let ok = r.step == RotationStep::Prepared;
            if ok {
                r.step = RotationStep::LocalChangesSent;
            }
            ok
        })?;
    if advanced && !outcome.backoff {
        env.events.wake();
    }
    Ok(())
}

/// 第 3 步:舊帳戶 chain 寫入 `meta` `rotation:<device_id>` → 凍結舊帳戶 chain → 凍結每個舊 space chain(含這台沒勾選
/// 的、別台剛建立的)。重跑時都是冪等的。舊帳戶在寫標記之前已被別台凍結(另一台先更換了同步碼、這台的標記寫不進去):這台還沒有
/// 擋下任何人 —— 讓給那一次更換:清掉進度與暫存的同步碼,這台記下 `frozen`,請使用者輸入對方的新同步碼。
fn freeze(env: &SyncEnv, s: &SyncStateV2, keys: &ChainKeys, relay: &dyn RelayApi) -> Result<(), AppError> {
    let now = env.now();
    let marker = Record {
        kind: RecordKind::Meta,
        id: rotation_meta_id(&s.device_id),
        version: 1,
        updated_at_ms: now,
        device_id: s.device_id.clone(),
        deleted: false,
        payload: serde_json::to_value(RotationMarkerPayload {
            rotated_at_ms: now,
            by_device_id: s.device_id.clone(),
            by_device_name: s.device_name.clone(),
        })
        .expect("RotationMarkerPayload serializes"),
    };
    match relay.push(&keys.chain_id, &keys.auth_token, &[encode(keys, &marker, 0)?])? {
        // 已有一份(上一次中斷前寫的,id 只有這台會用)也算寫好了。
        PushOutcome::Applied(_) => {}
        PushOutcome::Frozen => return yield_to_other_rotation(env, old_account_snapshot(keys, relay)?.markers),
    }
    relay.freeze_chain(&keys.chain_id, &keys.auth_token)?;
    // 帳戶凍結之後內容就不會再變:以 relay 上的帳戶列出 space(含別台剛建立、這台還沒拉到的)。
    let account = old_account_snapshot(keys, relay)?.section;
    for entry in space_entries(&account) {
        if let Some(space) = space_keys(&account, keys, &entry.id) {
            match relay.freeze_chain(&space.chain_id, &space.auth_token) {
                Ok(()) | Err(RelayError::NotFound) => {}
                Err(e) => return Err(e.into()),
            }
        }
    }
    update_rotation(env, |r| {
        r.step = RotationStep::Copying;
        true
    })?;
    Ok(())
}

/// 另一台先更換了同步碼(第 2 步拉帳戶時看到它的標記,或第 3 步寫不進這台的標記):放棄這一次(還沒凍結任何東西),改成
/// 「其他電腦」的流程 —— 清掉進度與暫存的新同步碼,記下 `frozen` 與 `markers`,請使用者輸入對方的新同步碼。
fn yield_to_other_rotation(env: &SyncEnv, markers: Vec<RotationMarkerPayload>) -> Result<(), AppError> {
    let now = env.now();
    {
        let _doc = env.doc.lock().unwrap();
        let mut core = env.runtime.core.lock().unwrap();
        core.generation += 1;
        if let Some(s) = core.state.as_mut() {
            s.rotation = None;
            if let Some(a) = s.account.as_mut() {
                a.frozen = Some(FreezeInfo { detected_at_ms: now, markers });
            }
        }
        save_core(&mut core, &env.state_path)?;
    }
    let _ = env.keychain.delete(NEXT_MNEMONIC_ACCOUNT);
    Ok(())
}

/// 舊帳戶在 relay 上的完整內容(凍結之後就是最終內容,spec §7.5 第 4 步)。
fn old_account_snapshot(keys: &ChainKeys, relay: &dyn RelayApi) -> Result<crate::sync::merge::AccountMerged, AppError> {
    Ok(merge_account(&AccountState::new(&keys.chain_id), keys, &relay.pull(&keys.chain_id, &keys.auth_token, 0)?))
}

/// 第 4、5 步:從凍結的 relay 取完整快照(來源是 relay,不是本機快取 —— 包含別台已上傳的修改與這台尚待核准的記錄)
/// → `PUT` 新帳戶與每個新 space chain(`429` 就暫停到下個小時)→ 每個 space 的記錄以新金鑰重新加密上傳(保留
/// version、updated_at_ms、device_id 與 tombstone)→ 新帳戶寫入 `space`(含 `previous_id`)、`spacekey`、這台的
/// `device` 與帳戶 `meta`。每完成一個 space 就記下;重跑時已寫過的列回 conflict,視為已複製。
fn copy(env: &SyncEnv, s: &SyncStateV2, keys: &ChainKeys, rotation: &RotationProgress, relay: &dyn RelayApi) -> Result<(), AppError> {
    let (_, new_account) = new_words(env, rotation)?;
    let old = old_account_snapshot(keys, relay)?.section;
    let live = live_spaces(&old, keys);
    let now = env.now();
    // 第 1 步之後才建立的 space:現在補上它的新位置與金鑰(先存下來,中斷也不會換一組)。
    let mut rotation = rotation.clone();
    for entry in &live {
        if !rotation.spaces.contains_key(&entry.id) {
            let space = ChainKeys::generate()?;
            let rotated = RotatedSpace { new_space_id: space.chain_id.clone(), sealed_key: new_space_key(&new_account, &space, &s.device_id, now)? };
            rotation.spaces.insert(entry.id.clone(), rotated.clone());
            let id = entry.id.clone();
            update_rotation(env, move |r| {
                r.spaces.entry(id).or_insert(rotated);
                true
            })?;
        }
    }
    let new_keys = |old_id: &str| -> Result<ChainKeys, AppError> {
        let rotated = &rotation.spaces[old_id];
        let record = rotated.sealed_key.open(&new_account)?;
        serde_json::from_value::<SpaceKeyPayload>(record.payload)
            .map_err(|_| AppError::Other("a new space key is unreadable".to_string()))?
            .to_keys(&rotated.new_space_id)
    };
    let mut chains: Vec<ChainKeys> = vec![new_account.clone()];
    for entry in &live {
        chains.push(new_keys(&entry.id)?);
    }
    for chain in chains {
        if rotation.created.contains(&chain.chain_id) {
            continue;
        }
        match relay.create_chain(&chain.chain_id, &chain.auth_token) {
            Ok(()) => {
                let id = chain.chain_id.clone();
                update_rotation(env, move |r| {
                    r.created.insert(id);
                    r.paused_until_ms = None;
                    true
                })?;
            }
            Err(RelayError::RateLimited) => {
                let until = now + CREATE_PAUSE_MS;
                update_rotation(env, move |r| {
                    r.paused_until_ms = Some(until);
                    true
                })?;
                return Ok(());
            }
            Err(e) => return Err(e.into()),
        }
    }
    for entry in &live {
        if rotation.copied.contains(&entry.id) {
            continue;
        }
        let old_keys = space_keys(&old, keys, &entry.id).expect("live spaces have keys");
        let target = new_keys(&entry.id)?;
        let pulled = match relay.pull(&old_keys.chain_id, &old_keys.auth_token, 0) {
            Ok(p) => p,
            // 舊 chain 已經不在(閒置過期,或另一次更換刪掉了):沒有內容可複製。
            Err(RelayError::NotFound) => crate::sync::relay::PullResponse { records: Vec::new(), latest_seq: 0 },
            Err(e) => return Err(e.into()),
        };
        let outgoing = pulled
            .records
            .iter()
            .filter_map(|env| decode(&old_keys, env).ok())
            .filter(|r| r.kind == RecordKind::Host)
            .map(|r| {
                Ok(crate::sync::merge::Outgoing {
                    key: crate::sync::record::record_key(RecordKind::Host, &r.id),
                    item: encode(&target, &r, 0)?,
                    version: r.version,
                    updated_at_ms: r.updated_at_ms,
                })
            })
            .collect::<Result<Vec<_>, AppError>>()?;
        if let Some(e) = push_outgoing(relay, &target.chain_id, &target.auth_token, &outgoing).error {
            return Err(e.into());
        }
        let id = entry.id.clone();
        update_rotation(env, move |r| {
            r.copied.insert(id);
            true
        })?;
    }
    // 新帳戶的記錄。
    let mut section = AccountState::new(&new_account.chain_id);
    put_account_record(
        &mut section,
        RecordKind::Meta,
        ACCOUNT_META_ID,
        serde_json::to_value(MetaPayload::account(env!("CARGO_PKG_VERSION"))).expect("MetaPayload serializes"),
        false,
        &s.device_id,
        now,
    );
    let mut selected_new = Vec::new();
    for entry in &live {
        let rotated = &rotation.spaces[&entry.id];
        let payload = SpacePayload {
            schema: SCHEMA_VERSION,
            name: entry.name.clone(),
            slug: entry.slug.clone(),
            created_at_ms: entry.created_at_ms,
            previous_id: Some(entry.id.clone()),
        };
        put_account_record(&mut section, RecordKind::Space, &rotated.new_space_id, serde_json::to_value(payload).expect("SpacePayload serializes"), false, &s.device_id, now);
        section.sealed.insert(rotated.sealed_key.key(), rotated.sealed_key.clone());
        if s.spaces.get(&entry.id).is_some_and(|sp| sp.selected) {
            selected_new.push(rotated.new_space_id.clone());
        }
    }
    selected_new.sort();
    plan_device(&mut section, &s.device_id, &s.device_name, env.platform, &selected_new, now);
    let outgoing = account_outgoing(&section, &new_account)?;
    if let Some(e) = push_outgoing(relay, &new_account.chain_id, &new_account.auth_token, &outgoing).error {
        return Err(e.into());
    }
    update_rotation(env, |r| {
        r.step = RotationStep::Deleting;
        true
    })?;
    Ok(())
}

/// 第 6 步:`DELETE` 每個舊 space chain(舊帳戶 chain 保留:凍結、帶著標記,閒置 180 天後清除)。舊帳戶上有**別台**的
/// 更換標記(兩台同時更換)時不刪:另一次更換可能還要從這些 chain 複製,留給 relay 過期清除。
fn delete_old(env: &SyncEnv, s: &SyncStateV2, keys: &ChainKeys, rotation: &RotationProgress, relay: &dyn RelayApi) -> Result<(), AppError> {
    let old = old_account_snapshot(keys, relay)?;
    let concurrent = old.markers.iter().any(|m| m.by_device_id != s.device_id);
    if !concurrent {
        for old_id in rotation.spaces.keys().filter(|id| !rotation.deleted.contains(*id)) {
            if let Some(space) = space_keys(&old.section, keys, old_id) {
                match relay.delete_chain(&space.chain_id, &space.auth_token) {
                    Ok(()) | Err(RelayError::NotFound) => {}
                    Err(e) => return Err(e.into()),
                }
            }
            let id = old_id.clone();
            update_rotation(env, move |r| {
                r.deleted.insert(id);
                true
            })?;
        }
    }
    update_rotation(env, |r| {
        r.step = RotationStep::Switching;
        true
    })?;
    Ok(())
}

/// 依 `previous_id` 把這台勾選的 space 帶進新帳戶(spec §7.5「保留勾選、檔名、待核准項目」):記錄的 seq 屬於舊 chain
/// → 歸零;cursor 歸零、基線已建立(第一輪以一般 LWW 合併,未上傳的修改照原時間戳上傳)。對不到新 space 的不帶。
fn carry_spaces(old: &BTreeMap<String, SpaceState>, mapping: &BTreeMap<String, String>) -> BTreeMap<String, SpaceState> {
    let mut out = BTreeMap::new();
    for (old_id, sp) in old.iter().filter(|(_, sp)| sp.selected) {
        let Some(new_id) = mapping.get(old_id) else { continue };
        let mut sp = sp.clone();
        sp.cursor_seq = 0;
        sp.baseline_established = true;
        sp.missing = false;
        sp.last_error = None;
        for local in sp.records.values_mut() {
            local.seq = 0;
        }
        for pending in sp.pending_approvals.values_mut() {
            pending.seq = 0;
        }
        for declined in sp.declined.values_mut() {
            declined.seq = 0;
        }
        out.insert(new_id.clone(), sp);
    }
    out
}

/// 換成新帳戶(第 7 步與其他電腦的重新加入共用):在 doc 鎖內、以**最新**的狀態(含期間存檔當下規劃的修改)依
/// `mapping`(舊 space id → 新 space id)帶過勾選的 space,一次換掉並存檔;再把 keychain 的同步碼換成新碼。在兩者
/// 之間中斷時,啟動流程以 `sync:mnemonic-next` 補完(`engine::startup`)。
fn install_new_account(
    env: &SyncEnv,
    words: &str,
    keys: ChainKeys,
    account: AccountState,
    mapping: &BTreeMap<String, String>,
    notices: Vec<SyncNotice>,
) -> Result<(), AppError> {
    let now = env.now();
    {
        let _doc = env.doc.lock().unwrap();
        let mut core = env.runtime.core.lock().unwrap();
        core.generation += 1;
        core.conflict_streak = 0;
        core.failed_rounds = 0;
        core.batch_failures = 0;
        let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        s.spaces = carry_spaces(&s.spaces, mapping);
        s.account = Some(account);
        s.rotation = None;
        s.last_error = None;
        s.notices.extend(notices.iter().cloned());
        let ids = selected_ids(s);
        let (me, name) = (s.device_id.clone(), s.device_name.clone());
        plan_device(s.account.as_mut().expect("just set"), &me, &name, env.platform, &ids, now);
        core.account_keys = Some(keys);
        save_core(&mut core, &env.state_path)?;
    }
    env.keychain.set(MNEMONIC_ACCOUNT, words)?;
    env.keychain.delete(NEXT_MNEMONIC_ACCOUNT)?;
    for notice in &notices {
        env.events.notice(notice);
    }
    Ok(())
}

/// 第 7 步:讀新帳戶(這台剛寫的)→ 狀態改用新帳戶(依 `previous_id` 保留勾選、檔名、待核准項目)→ keychain 的新碼取代
/// 舊碼 → 提示保存新同步碼。再讀一次舊帳戶 chain:有別台的更換標記就提示「另一台電腦也更換了同步碼」。
fn switch(env: &SyncEnv, s: &SyncStateV2, keys: &ChainKeys, rotation: &RotationProgress, relay: &dyn RelayApi) -> Result<(), AppError> {
    let (words, new_account) = new_words(env, rotation)?;
    let pulled = relay.pull(&new_account.chain_id, &new_account.auth_token, 0)?;
    let mut section = merge_account(&AccountState::new(&new_account.chain_id), &new_account, &pulled).section;
    section.baseline_established = true;
    let mapping: BTreeMap<String, String> = rotation
        .spaces
        .iter()
        .filter(|(old, _)| rotation.copied.contains(*old))
        .map(|(old, r)| (old.clone(), r.new_space_id.clone()))
        .collect();
    let mut notices = vec![SyncNotice::NewSyncCode];
    let others: Vec<String> = old_account_snapshot(keys, relay)
        .map(|old| old.markers.into_iter().filter(|m| m.by_device_id != s.device_id).map(|m| m.by_device_name).collect())
        .unwrap_or_default();
    if !others.is_empty() {
        notices.push(SyncNotice::OtherRotation { devices: others });
    }
    install_new_account(env, &words, new_account, section, &mapping, notices)?;
    env.events.applied(0);
    Ok(())
}

/// 啟動時 keychain 的同步碼推導不出狀態裡的帳戶:若暫存的新同步碼(`sync:mnemonic-next`)推導得出,代表第 7 步在
/// 「狀態已切換、keychain 還沒換」之間中斷 —— 以新碼取代舊碼、清掉暫存,回傳帳戶金鑰。
pub fn finish_interrupted_switch(keychain: &dyn crate::sync::env::Keychain, chain_id: &str) -> Option<ChainKeys> {
    let words = keychain.get(NEXT_MNEMONIC_ACCOUNT).ok()??;
    let keys = crypto::derive_account(&words).ok().filter(|k| k.chain_id == chain_id)?;
    keychain.set(MNEMONIC_ACCOUNT, &words).ok()?;
    let _ = keychain.delete(NEXT_MNEMONIC_ACCOUNT);
    Some(keys)
}

/// 其他電腦(spec §7.5):這台已偵測到同步碼被更換(`frozen`),使用者輸入新同步碼 → 驗證新帳戶存在、沒有又被更換
/// → 依 `previous_id` 保留勾選、檔名、待核准項目;本機尚未上傳的 dirty 記錄沿用原時間戳帶進新 space(第一輪是一般
/// LWW 合併,不是基線輪),所以被凍結擋下的修改不會遺失。
pub fn rejoin_account(env: &SyncEnv, words: &str) -> Result<(), AppError> {
    let words = crypto::normalize_mnemonic(words)?;
    let s = crate::sync::runtime::snapshot(env).ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    let old_chain = s.account.as_ref().map(|a| a.chain_id.clone());
    if s.frozen().is_none() {
        return Err(AppError::Other("the sync code of this account has not been changed".to_string()));
    }
    let new_account = crypto::derive_account(&words)?;
    if Some(&new_account.chain_id) == old_chain.as_ref() {
        return Err(AppError::Other("that is the old sync code; enter the new one from the device that changed it".to_string()));
    }
    let relay = env.relay(&s.relay_url)?;
    let pulled = match relay.pull(&new_account.chain_id, &new_account.auth_token, 0) {
        Ok(p) => p,
        Err(RelayError::NotFound) => return Err(AppError::NotFound(NO_ACCOUNT_MESSAGE.to_string())),
        Err(e) => return Err(e.into()),
    };
    let merged = merge_account(&AccountState::new(&new_account.chain_id), &new_account, &pulled);
    if let Some(marker) = merged.markers.first() {
        return Err(AppError::Other(format!(
            "this sync code was changed as well, on {}; enter the newest sync code",
            marker.by_device_name
        )));
    }
    let mut section = merged.section;
    section.baseline_established = true;
    let mapping: BTreeMap<String, String> = space_entries(&section)
        .into_iter()
        .filter(|e| !e.deleted)
        .filter_map(|e| e.previous_id.map(|old| (old, e.id)))
        .collect();
    install_new_account(env, &words, new_account, section, &mapping, Vec::new())?;
    env.events.applied(0);
    env.events.wake();
    Ok(())
}
```

- [ ] **Step 6: 修改 `src-tauri/src/sync/engine.rs`**

`startup`:keychain 的同步碼推導不出狀態裡的帳戶時,先試 `finish_interrupted_switch`;再加 3 個 commands。

`src-tauri/src/sync/engine.rs`:把

```rust
                match account_keys_from_keychain(keychain.get(MNEMONIC_ACCOUNT), &chain) {
                    Ok(k) => keys = Some(k),
                    Err(message) => s.last_error = Some(message),
                }
            }
```

換成:

```rust
                match account_keys_from_keychain(keychain.get(MNEMONIC_ACCOUNT), &chain) {
                    Ok(k) => keys = Some(k),
                    // 更換同步碼的第 7 步在「狀態已換成新帳戶、keychain 還是舊碼」之間中斷:以暫存的新碼補完。
                    Err(message) => match crate::sync::rotation::finish_interrupted_switch(keychain, &chain) {
                        Some(k) => keys = Some(k),
                        None => s.last_error = Some(message),
                    },
                }
            }
```

`src-tauri/src/sync/engine.rs`:把

```rust
}

/// 使用者看過了一則提示(`SyncOverview::notices` 的 index)。
#[tauri::command]
```

換成:

```rust
}

/// 更換同步碼(spec §7.5):第 1 步在這裡做完,之後由背景執行緒逐步推進;進度在 `SyncOverview::rotation`,完成時
/// 留下 `SyncNotice::NewSyncCode`(UI 以 `sync_show_words` 顯示新同步碼)。
#[tauri::command]
pub async fn sync_change_sync_code(app: AppHandle) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, crate::sync::rotation::start_rotation).await
}

/// 取消更換同步碼:只能在凍結之前(`SyncRotationView::cancellable`)。
#[tauri::command]
pub async fn sync_cancel_sync_code_change(app: AppHandle) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, crate::sync::rotation::cancel_rotation).await
}

/// 同步碼已在別台更換(`SyncOverview::frozen`):輸入新同步碼重新加入,保留勾選、檔名與未上傳的修改。
#[tauri::command]
pub async fn sync_rejoin_account(app: AppHandle, words: String) -> Result<SyncOverview, AppError> {
    run_then_overview(app, true, move |env| crate::sync::rotation::rejoin_account(env, &words)).await
}

/// 使用者看過了一則提示(`SyncOverview::notices` 的 index)。
#[tauri::command]
```

- [ ] **Step 7: 修改 `src-tauri/src/sync/round.rs`**

這台正在更換(且沒有被別台凍結)時,背景執行緒的一輪交給 `drive_rotation`。

`src-tauri/src/sync/round.rs`:把

```rust
        // 狀態還寫不進磁碟:不在未落盤的狀態上做任何網路操作。
        (_, Some(e)) => Err(e),
        (Some((s, keys)), None) => run_round(env, generation, s, keys, true).map(|_| ()),
        (None, None) => Ok(()),
```

換成:

```rust
        // 狀態還寫不進磁碟:不在未落盤的狀態上做任何網路操作。
        (_, Some(e)) => Err(e),
        // 這台正在更換同步碼:由更換流程推進(第 2 步裡面會跑一般輪次)。
        (Some((s, keys)), None) if s.rotation.is_some() && s.frozen().is_none() => {
            crate::sync::rotation::drive_rotation(env, generation, s, keys)
        }
        (Some((s, keys)), None) => run_round(env, generation, s, keys, true).map(|_| ()),
        (None, None) => Ok(()),
```

- [ ] **Step 8: 修改 `src-tauri/src/lib.rs`**

註冊 3 個 commands。

`src-tauri/src/lib.rs`:把

```rust
use settings_io::{settings_export, settings_import};
use sync::engine::{
    sync_approve, sync_check_relay, sync_create_account, sync_create_space, sync_delete_space,
    sync_dismiss_notice, sync_forget_device, sync_join_account, sync_leave_account,
    sync_move_hosts_to_space, sync_now, sync_overview, sync_pending_approvals, sync_rebuild_space,
    sync_reject, sync_rename_space, sync_select_space, sync_set_device_name, sync_set_relay_url,
    sync_show_words, sync_unselect_space,
};
```

換成:

```rust
use settings_io::{settings_export, settings_import};
use sync::engine::{
    sync_approve, sync_cancel_sync_code_change, sync_change_sync_code, sync_check_relay,
    sync_create_account, sync_create_space, sync_delete_space, sync_dismiss_notice,
    sync_forget_device, sync_join_account, sync_leave_account, sync_move_hosts_to_space, sync_now,
    sync_overview, sync_pending_approvals, sync_rebuild_space, sync_reject, sync_rejoin_account,
    sync_rename_space, sync_select_space, sync_set_device_name, sync_set_relay_url,
    sync_show_words, sync_unselect_space,
};
```

`src-tauri/src/lib.rs`:把

```rust
            sync_dismiss_notice,
            sync_move_hosts_to_space,
            sync_duplicate_aliases,
            sync_resolve_shadowed,
```

換成:

```rust
            sync_dismiss_notice,
            sync_move_hosts_to_space,
            sync_change_sync_code,
            sync_cancel_sync_code_change,
            sync_rejoin_account,
            sync_duplicate_aliases,
            sync_resolve_shadowed,
```

- [ ] **Step 9: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 669 passed; 0 failed`(task 開始前 656)。數量有變的模組:`sync::engine` 6 → 7、`sync::rotation` 12(新)。warning 只剩既有的 `config::edit::set_host_enabled` 未使用(與本計畫無關)。

- [ ] **Step 10: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/rotation.rs
git add src-tauri/src/sync/engine.rs
git add src-tauri/src/sync/round.rs
git add src-tauri/src/lib.rs
git add src-tauri/src/sync/mod.rs
git commit -m "feat(sync): change the sync code with frozen chains and rejoin by previous id"
```

---

### Task 4: 搬移精靈:一個來源檔建立一個 space、列出不能搬的主機

> **已執行**(repo `e950c05` 與本節一致;兩輪審查後的修正 `c89aa46`、`e164a3e`;之後 B3b 的最終修正又改了部分規則與文字)。下面保留原本的步驟作為紀錄,不要再執行;實際的做法見 Global Constraints 的「Task 4 實際的做法」與文末「最終修正(已執行)」。Task 4 之後(兩個 keychain 測試都略過)是 `708 passed`(`sync::migrate` 25);下面 Step 裡的數字是原本計畫的。

spec §7.2「從自己的 config 檔搬入」的選項「一個來源檔建立一個 space」與 §8「列出不能搬入的區塊與原因」。UI 依來源
檔分組、產生名稱(檔名 alias 優先),送 `NewSpaceGroup` 清單;後端每一組先建立 space(建立者勾選,新的空 chain 不需要
基線輪,所以可以馬上搬),再用 Task 2 的 `move_hosts_into_space` 搬進去。建立失敗(例如重名)的那一組,主機全部列為失敗,
其他組照常。`unmovable_hosts` 讓精靈事先列出含 `Include` 或帶引號 keyword 的主機。

**Files:**
- Modify: `src-tauri/src/sync/migrate.rs`(`NewSpaceGroup`、`move_into_new_spaces`、`unmovable_hosts` 與測試)
- Modify: `src-tauri/src/sync/engine.rs`(2 個 commands)
- Modify: `src-tauri/src/lib.rs`
- Generated: `src/bindings/NewSpaceGroup.ts`

**Interfaces:**
- Consumes(Task 2):`migrate::{move_hosts_into_space, refuse_while_sync_inactive, selected_space_files, MigrationReport,
  MigrationFailure}`、`engine::{run, engine_active}`;B3a `spaces::create_space`;B2 `hosts_file::{forbidden_directive,
  is_syncable_block}`。
- Produces:`pub struct NewSpaceGroup { pub name: String, pub aliases: Vec<String> }`(ts-rs);
  `pub fn move_into_new_spaces(env: &SyncEnv, active: bool, groups: Vec<NewSpaceGroup>, tag_by_file: bool) -> Result<MigrationReport, AppError>`;
  `pub fn unmovable_hosts(env: &SyncEnv) -> Result<Vec<MigrationFailure>, AppError>`;commands
  `sync_move_files_to_new_spaces(groups: Vec<NewSpaceGroup>, tag_by_file: bool) -> MigrationReport`、
  `sync_unmovable_hosts() -> Vec<MigrationFailure>`。

- [ ] **Step 1: `src-tauri/src/sync/migrate.rs` 的測試**

一台裝置:主 config 裡有一般主機、含 `Include` 的主機與 wildcard;另一個被 Include 的檔案有兩台主機。

`src-tauri/src/sync/migrate.rs`:把

```rust
    }

    fn runtime_with(space_id: &str, baseline: bool) -> SyncRuntime {
        let runtime = SyncRuntime::default();
```

換成:

```rust
    }

    #[test]
    fn one_space_per_source_file_creates_the_spaces_and_lists_what_cannot_move() {
        use crate::sync::account::create_account;
        use crate::sync::fake_relay::FakeRelay;
        use crate::sync::merge::space_entries;
        use crate::sync::testkit::{TestClock, TestDevice};
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::with_main_config(
            "a",
            &relay,
            &clock,
            "# main\nInclude ~/.ssh/homelab.config\nHost web\nHost jump\n  Include ~/.ssh/j.config\nHost *\n  User me\n",
        );
        std::fs::write(a.ssh_dir().join("homelab.config"), "Host nas\nHost pi\n").unwrap();
        a.reload();
        create_account(&a.env(), "MacBook-A").unwrap();
        let unmovable = unmovable_hosts(&a.env()).unwrap();
        assert_eq!(unmovable.len(), 1, "wildcard blocks are not hosts and are not listed");
        assert_eq!((unmovable[0].alias.as_str(), unmovable[0].error.as_str()), ("jump", "contains an Include line, which synced hosts cannot use"));
        let groups = vec![
            NewSpaceGroup { name: "homelab".into(), aliases: vec!["nas".into(), "pi".into()] },
            NewSpaceGroup { name: "config".into(), aliases: vec!["web".into(), "jump".into()] },
            NewSpaceGroup { name: "Personal".into(), aliases: vec!["ghost".into()] },
        ];
        let report = move_into_new_spaces(&a.env(), true, groups, true).unwrap();
        assert_eq!(report.moved, vec!["nas".to_string(), "pi".to_string(), "web".to_string()]);
        assert_eq!(report.failed.iter().map(|f| f.alias.as_str()).collect::<Vec<_>>(), vec!["jump", "ghost"]);
        assert_eq!(report.failed[1].error, "a space named 'Personal' already exists");
        assert_eq!(report.tagged, 2, "hosts from an included file get its name as a tag");
        let names: Vec<String> = space_entries(a.state().account.as_ref().unwrap()).into_iter().map(|e| e.name).collect();
        assert_eq!(names, vec!["config".to_string(), "homelab".to_string(), "Personal".to_string()]);
        let homelab = space_entries(a.state().account.as_ref().unwrap()).into_iter().find(|e| e.name == "homelab").unwrap().id;
        let text = a.read(&a.space_path(&homelab));
        assert!(text.contains("Host nas") && text.contains("Host pi") && text.contains("#tags:homelab"), "{text}");
        assert_eq!(a.read(&a.ssh_dir().join("homelab.config")).trim(), "", "moved out of the source file");
    }

    fn runtime_with(space_id: &str, baseline: bool) -> SyncRuntime {
        let runtime = SyncRuntime::default();
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0422]: cannot find struct, variant or union type `NewSpaceGroup` in this scope
--> src/sync/migrate.rs:610:13
```

- [ ] **Step 3: 修改 `src-tauri/src/sync/migrate.rs`**

每一組先建立 space 再搬;`unmovable_hosts` 只看不在任何勾選 space 檔裡的檔案。

`src-tauri/src/sync/migrate.rs`:把

```rust
}

#[cfg(test)]
mod tests {
```

換成:

```rust
}

/// 「一個來源檔建立一個 space」(spec §7.2)的一組:新 space 的名稱(UI 以來源檔名產生,檔名 alias 優先)與要搬進去的主機。
#[derive(Clone, Debug, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct NewSpaceGroup {
    pub name: String,
    pub aliases: Vec<String>,
}

/// 搬移精靈「一個來源檔建立一個 space」(spec §7.2):每一組先建立 space(建立者勾選,新的空 chain 不需要基線輪)、
/// 再把主機搬進去。建立失敗的那一組,主機全部列為失敗(原因相同);其他組照常。
pub fn move_into_new_spaces(env: &SyncEnv, active: bool, groups: Vec<NewSpaceGroup>, tag_by_file: bool) -> Result<MigrationReport, AppError> {
    refuse_while_sync_inactive(active, env.runtime)?;
    let mut report = MigrationReport { moved: Vec::new(), failed: Vec::new(), tagged: 0 };
    for group in groups {
        match crate::sync::spaces::create_space(env, &group.name) {
            Ok(space_id) => {
                let part = move_hosts_into_space(env, active, group.aliases, &space_id, tag_by_file)?;
                report.moved.extend(part.moved);
                report.failed.extend(part.failed);
                report.tagged += part.tagged;
            }
            Err(e) => {
                let error = e.to_string();
                report.failed.extend(group.aliases.into_iter().map(|alias| MigrationFailure { alias, error: error.clone() }));
            }
        }
    }
    Ok(report)
}

/// 搬移精靈列出的、不能搬進 space 的本機主機與原因(spec §7.2、§8):不在任何勾選 space 檔裡、所有 pattern 都具名、
/// 卻含 `Include` 或帶引號 keyword 的區塊。
pub fn unmovable_hosts(env: &SyncEnv) -> Result<Vec<MigrationFailure>, AppError> {
    let spaces: Vec<PathBuf> = selected_space_files(env.runtime, &env.ssh_dir).into_iter().map(|(_, p)| p).collect();
    let doc_lock = env.doc.lock().unwrap();
    let doc = doc_lock.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let mut out = Vec::new();
    for file in doc.files.iter().filter(|f| !spaces.contains(&f.path)) {
        for item in &file.items {
            let Item::Host(h) = item else { continue };
            if !is_syncable_block(&h.patterns) {
                continue;
            }
            if let (Some(alias), Some(f)) = (h.patterns.first(), forbidden_directive(std::slice::from_ref(item))) {
                out.push(MigrationFailure {
                    alias: alias.clone(),
                    error: format!("contains {}, which synced hosts cannot use", f.describe()),
                });
            }
        }
    }
    Ok(out)
}

#[cfg(test)]
mod tests {
```

- [ ] **Step 4: 修改 `src-tauri/src/sync/engine.rs`**

建立 space 是結構變更,`sync_move_files_to_new_spaces` 全程持有 lifecycle 鎖。

`src-tauri/src/sync/engine.rs`:把

```rust
}

/// 更換同步碼(spec §7.5):第 1 步在這裡做完,之後由背景執行緒逐步推進;進度在 `SyncOverview::rotation`,完成時
/// 留下 `SyncNotice::NewSyncCode`(UI 以 `sync_show_words` 顯示新同步碼)。
```

換成:

```rust
}

/// 搬移精靈「一個來源檔建立一個 space」(spec §7.2)。
#[tauri::command]
pub async fn sync_move_files_to_new_spaces(
    app: AppHandle,
    groups: Vec<crate::sync::migrate::NewSpaceGroup>,
    tag_by_file: bool,
) -> Result<crate::sync::migrate::MigrationReport, AppError> {
    run(app, true, move |env| crate::sync::migrate::move_into_new_spaces(env, engine_active(), groups, tag_by_file)).await
}

/// 搬移精靈列出的、不能搬進 space 的主機與原因。
#[tauri::command]
pub async fn sync_unmovable_hosts(app: AppHandle) -> Result<Vec<crate::sync::migrate::MigrationFailure>, AppError> {
    run(app, false, crate::sync::migrate::unmovable_hosts).await
}

/// 更換同步碼(spec §7.5):第 1 步在這裡做完,之後由背景執行緒逐步推進;進度在 `SyncOverview::rotation`,完成時
/// 留下 `SyncNotice::NewSyncCode`(UI 以 `sync_show_words` 顯示新同步碼)。
```

- [ ] **Step 5: 修改 `src-tauri/src/lib.rs`**

註冊 2 個 commands。

`src-tauri/src/lib.rs`:把

```rust
    sync_approve, sync_cancel_sync_code_change, sync_change_sync_code, sync_check_relay,
    sync_create_account, sync_create_space, sync_delete_space, sync_dismiss_notice,
    sync_forget_device, sync_join_account, sync_leave_account, sync_move_hosts_to_space, sync_now,
    sync_overview, sync_pending_approvals, sync_rebuild_space, sync_reject, sync_rejoin_account,
    sync_rename_space, sync_select_space, sync_set_device_name, sync_set_relay_url,
    sync_show_words, sync_unselect_space,
};
use sync::migrate::{sync_duplicate_aliases, sync_resolve_shadowed};
```

換成:

```rust
    sync_approve, sync_cancel_sync_code_change, sync_change_sync_code, sync_check_relay,
    sync_create_account, sync_create_space, sync_delete_space, sync_dismiss_notice,
    sync_forget_device, sync_join_account, sync_leave_account, sync_move_files_to_new_spaces,
    sync_move_hosts_to_space, sync_now, sync_overview, sync_pending_approvals, sync_rebuild_space,
    sync_reject, sync_rejoin_account, sync_rename_space, sync_select_space, sync_set_device_name,
    sync_set_relay_url, sync_show_words, sync_unmovable_hosts, sync_unselect_space,
};
use sync::migrate::{sync_duplicate_aliases, sync_resolve_shadowed};
```

`src-tauri/src/lib.rs`:把

```rust
            sync_dismiss_notice,
            sync_move_hosts_to_space,
            sync_change_sync_code,
            sync_cancel_sync_code_change,
```

換成:

```rust
            sync_dismiss_notice,
            sync_move_hosts_to_space,
            sync_move_files_to_new_spaces,
            sync_unmovable_hosts,
            sync_change_sync_code,
            sync_cancel_sync_code_change,
```

- [ ] **Step 6: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 698 passed; 0 failed`(task 開始前 696)。數量有變的模組:`sync::migrate` 13 → 15。warning 只剩既有的 `config::edit::set_host_enabled` 未使用(與本計畫無關)。

- [ ] **Step 7: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/migrate.rs
git add src-tauri/src/sync/engine.rs
git add src-tauri/src/lib.rs
git add src/bindings/NewSpaceGroup.ts
git commit -m "feat(sync): create one space per source file and list hosts that cannot move"
```

---

## 最終修正(已執行)

B3b 的整體審查(`.superpowers/sdd/2026-10-02-sync-v2-b3b-engine-wiring/final-review.md`)之後的一次修正:`b675943`、`a43e521`、
`472d92b`、`3de5ad4`、`8971f96`、`a2b77b9`,以及複審之後的 `d469bc0`(報告與 Addendum 在同一個資料夾的 `final-fix-report.md`)。
spec 同步改在 `a11d0c3`、`01ce436`、`76a426c`、`83e2e6d`。之後(兩個 keychain 測試都略過)是 **`740 passed`**:`sync::account` 30、
`sync::engine` 14、`sync::hosts_file` 24、`sync::migrate` 31、`sync::rotation` 41、`sync::upgrade` 46、`config::commands` 45、
`fsutil` 15;warning 只剩既有的 `set_host_enabled`。沒有新增或移除任何 command、DTO 欄位、事件、提示種類或 binding;使用者看得到
的文字與行為變化列在 B4 handoff。

- **離開與更換同步碼**:`leave_account` 在 doc 鎖內、任何檔案或 relay 動作之前決定進行中的更換 —— 還能取消(凍結之前)就在同一把
  鎖內取消(同 `cancel_rotation`;背景的下一步因此不會凍結);過了凍結就拒絕(`LEAVE_ROTATING_MESSAGE`),例外是暫存的新碼不見了
  或不能用(這次更換永遠做不完):照常離開(檔案留在本機、`left_account`,relay 上什麼都不刪,`deleteRemote` 也一樣),再回 Err,
  說明出路是建立新的同步帳戶(處理方式同 `LEAVE_REPLACED_MESSAGE`)。取消了更換、之後檔案卻搬不過去時,錯誤會說更換已取消。各步驟
  讀新碼共用 `rotation::read_new_code`;第 3–7 步的 `last_error` 不再叫使用者重開 SSHelter;`NOT_A_SUCCESSOR_MESSAGE` 改成說實話的版本。
- **升級與 Include 的歸屬**(spec §4.3、§7.6):我們的 Include token 只算 `~/.ssh/sshelter/` 直接底下的 `.config`(含 glob),
  更深或在外面的路徑是使用者自己的 —— 不收也不搬;主 config 沒列 `KEPT_INCLUDE` 時,kept 檔備份後以這一次的內容重寫(沒有要留的
  就移除,`Upgraded.kept_file` = None);放棄升級時,沒被主 config 列出的 `hosts.config` 備份後移除、不列進 `LeftAccount`;
  `persist_file` 寫成功之後重讀指紋失敗,改用寫進去的內容算的指紋(不再回報失敗);升級切換時重查 `/v1/info`。
- **喚醒與退避**(spec §6.4):存檔 hook 與回到前景是隱含喚醒(`wake_implicit`),`429` / `5xx` / keychain 的退避期間先記下、
  等退避結束才跑(`engine::Wait`);`sync_now`、commands、`config_load` 偵測到的外部修改與引擎自己的後續動作照舊馬上跑。第一輪在
  啟動時就跑(拿掉 `FIRST_DELAY`)。keychain 的錯誤(升級、更換同步碼的各步、`retry_pending_swap`)也算失敗的一輪而退避
  (`SyncCore::swap_failures` 讓換碼重試的間隔加倍到 15 分鐘)。`sync://status` 在放掉 lifecycle 之後才發。
- **搬移精靈與拖曳**(spec §7.2):拖曳(`config_move_host`)與精靈共用同一套拒絕(`refuse_move_batch`、`refuse_move_of`);Include
  的順序只有一個比較器(`space_files::include_order`);建立 space 第一次被限流就停,之後的組不再問 relay;標記寫不進去、已經在那個
  space、建立了 space 卻搬不進去都有新的說明(見 B4 handoff)。
- **清理**:過時的註解;v1 狀態只剩升級與啟動要讀的(`SyncState::fresh`、`read_only` 等改成只在測試建置)。

---

## B4 handoff

B3 完成後後端的全部介面。命令的參數名稱在前端一律是 camelCase(Tauri 的預設轉換,例如 `space_id` → `spaceId`);
「→ Overview」表示回傳動作完成後的 `SyncOverview`(同時也會發一次 `sync://status`);核准與拒絕回傳 `ReviewOutcome`(裡面也有
`overview`)。錯誤一律是字串訊息(英文,可直接
顯示)。

**Commands**

| 命令 | 參數 | 回傳 | 語意 |
|---|---|---|---|
| `sync_overview` | — | `SyncOverview` | Settings → Sync 的全部狀態 |
| `sync_now` | — | `void` | 記下一次操作並立刻跑一輪(不受退避影響)。回到前景與存檔的喚醒由後端處理,在 `429` / `5xx` / keychain 的退避期間會等到退避結束 —— **前端不要在 focus 時呼叫它**(會繞過退避) |
| `sync_create_account` | `deviceName` | `string` | 建立帳戶與預設 space「Personal」並勾選;回傳 24 詞同步碼(只放元件狀態,不進查詢快取)。v1 留下的 `~/.ssh/sshelter/hosts.config`(主 config 的 Include 還讀它時)先搬到 `~/.ssh/sshelter-local/`(主機照常可用)並留下 `left_account` 提示 |
| `sync_join_account` | `words`, `deviceName` | Overview | 加入帳戶,不勾選任何 space;接著讓使用者勾選。同步碼驗證過之後,v1 留下的 `hosts.config` 同 `sync_create_account` 先搬走(打錯同步碼什麼都不動)。帳戶的同步碼已被更換(帳戶上有更換標記)時拒絕「this sync code was changed on <裝置>; enter the new sync code」 |
| `sync_leave_account` | `deleteRemote` | Overview | 離開:這台的 space 檔(放棄卡住的 v1 升級時是主 config 讀得到的 `hosts.config`,加上我們的 Include 現在讀得到的每個檔案;沒被列出的 `hosts.config` 備份後移除、不列進提示)搬到 `~/.ssh/sshelter-local/`,主 config 原地改成一般的 Include,ssh 照常可用,留下 `left_account` 提示;搬不過去 → 錯誤「could not keep this device's synced files as local files (…); nothing was changed — try leaving again」,什麼都沒變。`deleteRemote` = 一併刪除帳戶與所有 space 的 chain(含刪掉的 space 還排著要刪的 chain;只在這台是裝置清單上的最後一台時提供);relay 那一半做完、檔案卻搬不過去 → 錯誤「the sync account was deleted from the relay, but this computer's files could not be kept as local files (…); try leaving again」(仍是已加入,再離開一次只重做檔案)。`deleteRemote` 而同步碼已在別台更換(`frozen`)時:relay 上什麼都不刪、只在這台離開,回錯誤「left the sync account on this computer, but did not delete it from the relay: its sync code was changed on another device, and the computers still on the old code learn the change from it」—— 這台已經離開了:照常顯示訊息、重讀 overview;`frozen` 時 UI 不提供「刪除帳戶」。更換同步碼期間**不要停用離開**(DTO 看不出暫存的新碼還在不在;加確認,拒絕時直接顯示後端的訊息):還能取消時(凍結之前)離開會先取消這次更換(之後檔案搬不過去時錯誤是「could not keep this device's synced files as local files (…); the sync code change in progress was cancelled, nothing else was changed — try leaving again」);過了凍結就拒絕「a sync code change is in progress; let it finish (it resumes on its own) before this computer leaves」,例外是暫存的新碼不見了或不能用(這次更換永遠做不完):照常離開(檔案留在本機、`left_account` 提示,relay 上什麼都不刪,`deleteRemote` 也一樣)並回錯誤「left the sync account on this computer, but its sync code change could not be finished (the new sync code was missing from the keychain), so the old sync account can no longer be joined — create a new sync account on one computer, and on the other computers leave the old account (their synced files stay as local files) and join the new one」(`deleteRemote` 時後面再加「. The sync account was not deleted from the relay」)—— 同 `LEAVE_REPLACED_MESSAGE`:這台已經離開,顯示訊息並重讀 overview。未加入時只重試 keychain 清理 |
| `sync_show_words` | — | `string` | 顯示同步碼。更換同步碼(或重新加入)之後 keychain 還沒收下新碼時,回傳暫存的新碼 —— 它就是這個帳戶現在的碼 |
| `sync_set_relay_url` | `url` | Overview | 只能在未加入時;會重查 `GET /v1/info` |
| `sync_check_relay` | — | Overview | 重查 `GET /v1/info`(使用者更新 relay 之後) |
| `sync_set_device_name` | `name` | Overview | |
| `sync_forget_device` | `deviceId` | Overview | 只從清單移除,**不是撤權**(文案同 v1;撤權要更換同步碼) |
| `sync_create_space` | `name` | Overview | 建立並勾選(名稱 1–64 字元、帳戶內不分大小寫不重複) |
| `sync_rename_space` | `spaceId`, `name` | Overview | 目標檔名已存在 → 保留舊檔名並留下 `rename_blocked` 提示。主 config 在 app 外被改過時照樣成功(帳戶記錄已存檔),檔案那一半由下一輪做完 |
| `sync_delete_space` | `spaceId` | Overview | 先確認;其他電腦會移除檔案並看到 `space_deleted` 提示。主 config 在 app 外被改過時同 `sync_rename_space` |
| `sync_select_space` | `spaceId` | Overview | 建立檔案並列進 Include;第一輪(基線輪)完成前 `first_sync_pending` |
| `sync_unselect_space` | `spaceId` | Overview | 先確認;移出 Include、備份並刪除這台的檔案;relay 與其他電腦不受影響 |
| `sync_rebuild_space` | `spaceId` | Overview | `SyncSpaceView.missing` 時:用這台的內容重建 chain |
| `sync_pending_approvals` | — | `PendingApprovalView[]` | 等待核准的主機(完整區塊、目前區塊、兩個簽章,以及這一版的內容指紋 `digest`) |
| `sync_approve` | `spaceId`, `approvals: ReviewedVersion[]` | `ReviewOutcome` | 核准對話框**顯示的版本**:每筆 `{ alias, digest }` 取自 `PendingApprovalView`(對話框以 `digest` 認版本:同一台主機換了內容,`digest` 就不同)(「全部核准」= 傳入整個清單)。只套用清單上仍是那一版的;`changed` 非空 = 那幾台在對話框開著時換成了較新的版本(還在清單上)→ 顯示「已變更,請重新確認」並重讀 `sync_pending_approvals`。空清單什麼都不做 |
| `sync_reject` | `spaceId`, `approvals: ReviewedVersion[]` | `ReviewOutcome` | 拒絕顯示的版本:丟棄,本機維持原狀,不推送;`changed` 同 `sync_approve` |
| `sync_dismiss_notice` | `index` | Overview | 清掉 `SyncOverview.notices[index]` |
| `sync_move_hosts_to_space` | `aliases`, `spaceId`, `tagByFile` | `MigrationReport` | 搬移精靈「搬進一個 space」;逐台回報失敗原因(`MigrationFailure.error`,與拖曳 `config_move_host` 同一套拒絕、同樣的文字)。要搬的區塊已經在那個 space:「'web' is already in that space」,或「'web' is already in that space — it is another name of the host 'a'」;那個 space 已有同名的另一個區塊:「'web' is already in that space — resolve the duplicate instead」/「'web' is already used by a host in that space — remove or rename it in the local host 'a' first」;標記寫不進去:「moved, but its tag could not be saved: <cause>」(那台只列在 `failed`),之後的主機「not attempted: an earlier move failed」 |
| `sync_move_files_to_new_spaces` | `groups: NewSpaceGroup[]`, `tagByFile` | `MigrationReport` | 「一個來源檔建立一個 space」:每組先建立 space 再搬(拒絕的文字同 `sync_move_hosts_to_space`);一組失敗不影響其他組。沒有任何能搬的主機的組不建立 space(各台列出原因);同一台主機(或同一個區塊的另一個名稱)出現在兩組時只搬第一次,之後的列為「listed in more than one group」(UI 送出前最好先去掉重複);建立前發現 config 在磁碟上被改過就重載,重載失敗時那一組是「the config changed on disk and could not be reloaded: <error>」、不建立 space;建立了 space 卻搬不進去:「the space '<name>' was created, but the hosts could not be moved into it (<cause>) — move them into that space instead of creating it again」;建立 space 第一次被限流就停:那一組與之後每一組都是「the relay is rate-limiting new spaces from this network, so no more are created now; try again in about an hour」(事先就被拒絕的主機保留自己的原因) |
| `sync_unmovable_hosts` | — | `MigrationFailure[]` | 精靈事先列出的、不能搬的主機與原因 |
| `sync_change_sync_code` | — | Overview | 更換同步碼(relay 要有 `freeze`,否則回錯誤);進度在 `rotation`。上一次換碼的新碼還沒存進 keychain 時拒絕:「the new sync code is not saved to the keychain yet; SSHelter finishes that by itself, at the latest the next time it starts — change the sync code again after that」。第 3–7 步被限流或 relay 出錯時退避,`last_error` 是一般輪次的限流 / relay 出錯訊息;relay 拒絕的請求(例如儲存上限)與切換時檔案改不成本機檔案(「could not keep this device's synced files as local files (…); SSHelter retries the sync code change by itself」)的原因也在 `last_error`,都由背景執行緒自己重試;新碼的問題也在 `last_error`,背景每一輪都重讀 keychain、自動重試(keychain 的錯誤一樣退避):凍結之前新碼不見了「the new sync code is missing from the keychain, so this sync code change cannot continue; cancel it and start again」(還能取消);凍結之後不見了「the new sync code is missing from the keychain, so this sync code change can never finish and the old sync account can no longer be joined — leave the sync account on this computer, then create a new sync account on one computer; the other computers leave the old account and join the new one」(這時離開會放行,見 `sync_leave_account`);讀不到 keychain「could not read the new sync code from the keychain (<error>); unlock the keychain — SSHelter tries again on every sync」。切換完成、keychain 卻還沒收下新碼時 `last_error` 是「the keychain did not accept the new sync code yet; SSHelter keeps the new code and retries, and syncing continues meanwhile」,同步照常 |
| `sync_cancel_sync_code_change` | — | Overview | 只在 `rotation.cancellable` 時 |
| `sync_rejoin_account` | `words` | Overview | `frozen` 時輸入新同步碼;保留勾選、檔名與未上傳的修改;新帳戶沒有接續的勾選 space 改成本機檔案(`left_account` 提示)。拒絕時什麼都不變:沒有被換過碼「the sync code of this account has not been changed」;輸入舊碼「that is the old sync code; enter the new one from the device that changed it」;新帳戶不存在;新碼又被換過「this sync code was changed as well, on <裝置>; enter the newest sync code」;沒有任何勾選的 space 被它接續「none of the spaces this computer syncs continue in that sync account; if it is the newest sync code, leave the sync account — your synced files stay as local files that ssh keeps reading — and join with it」;檔案改不成本機檔案「could not keep this device's synced files as local files (…); nothing was changed — try again」;這個行程沒有同步引擎(另一個 SSHelter 拿著同步鎖)時也拒絕 |
| `sync_duplicate_aliases` | — | `DuplicateAlias[]` | 同一個 alias 在不同檔案(含不同 space 檔)被定義;列出 ssh 不會用的那份 |
| `sync_resolve_shadowed` | `alias`, `file`, `action: "rename" \| "remove"` | `DuplicateAlias[]` | 以檔案路徑定位處理被遮蔽的那份 |

既有的 `config_move_host` 現在也是**跨 space 搬移**:目標是勾選的 space 檔時套用搬進 space 的規則(先寫目標、再從來源
移除),錯誤訊息直接顯示;拒絕的規則與文字和精靈相同(見 `sync_move_hosts_to_space`)。

**Events**

| 事件 | payload | 語意 |
|---|---|---|
| `sync://status` | `SyncOverview` | 狀態變了(取代 v1 的 `SyncStatus`) |
| `sync://applied` | `number` | 引擎寫了 space 檔(套用的主機數)或整份重載了 doc(0):重新載入 config 與同名主機清單 |
| `sync://conflict` | `SyncConflict[]` | 這台未上傳的修改被別台較新的版本取代(每個 space 一項,帶 space 名稱) |
| `sync://approval` | `ApprovalNotice[]` | 這一輪新保留、等待核准的主機:跳出通知、打開審核對話框 |
| `sync://notice` | `SyncNotice` | 新的提示(同時存在 `SyncOverview.notices`) |

**Binding types**(`src/bindings/`)

| 型別 | 用途 |
|---|---|
| `SyncOverview` | 全部狀態:`joined`、`account_short`、裝置、relay URL 與 `relay`、`last_sync_ms`、`last_error`、`read_only`、`upgrading`、`frozen`、`rotation`、`devices`、`spaces`、`pending_uploads`、`approvals_waiting`、`stray_files`、`notices`、`phrase_cleanup_pending` |
| `SyncSpaceView` | 一個 space(依 Include 順序):名稱、勾選、檔名與路徑、主機數、待上傳、待核准、`first_sync_pending`、`missing`、`last_error`(也會是上傳被 relay 拒絕的原因,例如這個 space 超過 relay 的儲存上限;訊息帶 space 名稱,直接顯示)、建立時間、`synced_on` |
| `SyncDeviceView` | 裝置:名稱、平台、加入與最後上線時間、`is_this`、勾選的 space |
| `SyncRelayView` | relay 版本、`batch_pull`(false → 提示「relay 可以更新」)、`freeze`(false → 停用更換同步碼) |
| `SyncFrozenView` | 同步碼已在別台更換:`by_devices`(空 = 只知道上傳被拒;兩個以上 = 多台同時更換) |
| `SyncRotationView`、`RotationStep` | 更換同步碼的進度、能否取消、限流暫停到何時 |
| `SyncNotice` | 提示:`upgraded`(含另存檔與主機 —— 重跑時另存檔若已沒有要留的,`kept_file` 是 null —— 以及 `moved_files`:使用者自己放在 `~/.ssh/sshelter/`、被主 config 列著的檔案搬到 `~/.ssh/sshelter-local/` 之後的新路徑,通常是空陣列)、`space_deleted`、`rename_blocked`(同一個目標只出現一次)、`left_account { kept_files }`(這台改成本機檔案的檔案的新路徑:離開帳戶、更換同步碼的切換、重新加入時新帳戶沒有接續的 space)、`new_sync_code`(以 `sync_show_words` 顯示新碼)、`other_rotation` |
| `SyncConflict`、`ApprovalNotice` | 事件 payload |
| `PendingApprovalView`、`ApprovalSignature`、`GatedDirective` | 審核對話框:完整區塊、目前區塊、簽章差異(標出受管制的行)、這一版的 `digest` |
| `ReviewedVersion`、`ReviewOutcome` | 核准 / 拒絕送回的 `{ alias, digest }`;結果 `applied`(處理了幾台)、`changed`(已變更、請重新確認的 alias)、`overview` |
| `MigrationReport`、`MigrationFailure`、`NewSpaceGroup`、`DuplicateAlias` | 搬移精靈與同名主機 |

**B4 必須修改的現有前端**(B3b Task 2 之後這些呼叫在執行時會失敗):

- `src/lib/sync.ts`:`useSyncStatus` 的 `sync_status` → `sync_overview`(型別 `SyncOverview`);`useLeaveChain` 的
  `sync_leave_chain` → `sync_leave_account`;`createChain` 的 `sync_create_chain` → `sync_create_account`;`joinChain` 的
  `sync_join_chain` → `sync_join_account`(回傳 `SyncOverview`);`useMigrateHosts` 的 `sync_migrate_hosts` →
  `sync_move_hosts_to_space`(多一個 `spaceId`);所有 `useStatusMutation` 的回傳型別改成 `SyncOverview`。
- `src/App.tsx`:`sync://status` 的 payload 改成 `SyncOverview`;`sync://conflict` 的 payload 從 `string[]` 改成
  `SyncConflict[]`(toast 帶 space 名稱);新增 `sync://approval`、`sync://notice` 的 listener;拿掉 `focus` → `sync_now`
  (後端已由 `WindowEvent::Focused` 處理,而且在退避期間會等到退避結束;前端在 focus 時呼叫 `sync_now` 會繞過退避、多用 relay 配額)。
- `src/components/SyncPane.tsx`、`src/components/SyncMigrationDialog.tsx`、`src/lib/sync-migration.ts`:改用
  `SyncOverview`(`managed_file` → 各 space 的 `file_path`;`first_sync_pending` → 各 space 的同名欄位;`chain_short` →
  `account_short`;`pending` → `pending_uploads`;`hosts_in_sync` → 各 space 的 `hosts`);sidebar 的「Synced」檔名 alias
  改成每個勾選的 space 一個(`SyncSpaceView.name` → `file_path`)。
- 刪除不再產生的 `src/bindings/SyncStatus.ts`、`src/bindings/SyncDevice.ts`。
- 提示多了 `left_account { kept_files }`:`noticeMessage` 之類依 `kind` 分支的程式要處理它(例如「Left the sync account —
  your hosts stay in these files and keep working: …」),離開的對話框照 spec 說明檔案會留下來當一般的本機檔案。
- `rename_blocked` 在同一個目標仍被擋時不再重複加回或重發 `sync://notice`;前端「每則提示只 toast 一次」的保護可以留著,
  不再是必要的。
- 加入 / 重新加入時同步碼格式不對的錯誤改稱「sync code」:`a sync code has 24 words (got N)`、`invalid sync code: …`。

---

## Self-review(已執行)

- **Spec 覆蓋**:§7.6 第 1–6 步 → Task 1(第 6 步 v1 chain 不再讀寫:升級後沒有任何程式碼碰它);§7.5 第 1–7 步、
  取消、其他電腦、兩台同時更換 → Task 3;§7.2 跨 space 搬移與「搬進一個 space」→ Task 2,「一個來源檔建立一個 space」
  與「列出不能搬的區塊」→ Task 4;§7.3 的 commands → Task 2;§6.4 的焦點輪詢接線 → Task 2(`window_focused`、存檔
  hook、`round::next_delay`);§8 的後端(帳戶、relay 版本、裝置、spaces、等待核准、更換進度、未加入、側邊欄需要的
  檔案路徑、精靈、v1 升級說明)→ Task 2 的 `SyncOverview` 與 Task 3、4 的 commands;§9 的「更換同步碼中斷」→ Task 3、
  「v1 升級失敗」→ Task 1、「keychain 讀取失敗 / 狀態寫不進磁碟 / 讀不懂的狀態檔」→ Task 2 的 `startup` 與
  `unreadable_state_outcome`。B4 要的介面列在「B4 handoff」。
- **Placeholder 掃描**:每個程式步驟都是完整程式碼或精確的 edit;沒有 TBD、TODO 或「同 Task N」。
- **型別一致**:`upgrade::shell_state`(Task 1)→ Task 2 的 `startup`;`SyncOverview` 的 `rotation: Option<SyncRotationView>`
  與 `RotationStep`(Task 2)→ Task 3 推進的 `SyncStateV2::rotation`;`finish_interrupted_switch`(Task 3)→ Task 3 對
  `engine::startup` 的修改;`MigrationReport` / `MigrationFailure`(Task 2)→ Task 4;command 名稱在 `engine.rs`、
  `lib.rs` 與 B4 handoff 一致。
- **Review Focus**:五項各有測試,寫在對應 task 裡(見上方清單的測試名稱)。
- **驗證**:四個 task 與最終修正都已在 repo 執行(`740 passed`);Task 2–4 執行之前依序套用在 repo `1ee11de` 的拷貝上,每個 task
  先確認新測試編譯失敗,再確認全綠(701 → 656 → 669 → 671);Task 4 在 `6ca1323` 上照本計畫執行(696 → 698,anchor 已對照確認);最後的樹與參考實作逐位元組相同;每個 commit 的路徑清單涵蓋該 task 的全部變更。Task 2 之後
  前端 `tsc --noEmit` 通過;Task 4 之後 `cargo clippy --all-targets` 沒有新的警告(只剩既有的 7 個,v1 `migrate.rs` 原本的
  那一個隨改寫消失;上一版驗證的,這一版的修改沒有重跑 clippy),`cargo test` 的 warning 只剩既有的 `set_host_enabled`。
