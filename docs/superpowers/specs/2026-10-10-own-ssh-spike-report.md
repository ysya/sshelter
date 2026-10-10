# 自有 SSH 用戶端:第 0 期試驗報告

- 日期:2026-10-10
- 對應:`docs/superpowers/specs/2026-10-10-own-ssh-client-design.md` 第 13 節第 0 期、第 16 節;計畫 `docs/superpowers/plans/2026-10-10-own-ssh-phase0-spike.md`
- 環境:macOS(OpenSSH 10.3p1、Apple silicon)、russh `=0.64.1`、app 的 `ssh-key` 0.6.7;Windows 是 GitHub Actions 的 `windows-latest`。
- **`spike/russh/` 是拋棄式的試驗程式,不是產品程式碼。** 第 1 期要在 `ssh/russh_engine.rs` 裡、`SshEngine` 介面後面重寫一遍;這裡的檔案一個也不會複製進 `src-tauri/`。這個分支動到 `src-tauri/` 的只有:第 21 項的 `ipc/` 搬移(行為不變)和第 22 項的一行 `pub mod vault;`(讓試驗 crate 用得到保管庫;保管庫最後怎麼對外公開,第 1 期再決定);另有兩個 CI 檔(新的 `spike-windows.yml`,以及 `test-windows.yml` 的過濾加上 `ipc::`)。
- 證據來源:在 `spike/russh/` 執行 `cargo test --offline -- --include-ignored --nocapture` 的輸出;下表的通過/失敗與 `FACT` 值都來自那一次。測試名稱可以直接在 `spike/russh/tests/` 與 `spike/russh/src/` 找到。第 2、21、22 項的 `lock.*`、`ipc.*`、`app.*` 是同一次之後收集的(兩份 `Cargo.lock`、`src-tauri/` 的 lib 測試、`git diff`)。那次執行時機器有背景負載(load average 5.14 / 5.97 / 5.86,18 核),環境是 macOS、OpenSSH 10.3p1、russh 0.64.1;**所有計時與競態類的數字只代表這個環境**。第 14 項那種每次執行結果不同的觀察,另外附上整次 `--test-threads=1` 重跑(第二欄)和重複執行的次數,見該項的細節與附錄;主 log 的值沒有被取代。Windows 的值只來自「Windows」一節引用的 GitHub Actions 紀錄:本機 log 裡的 `windows.*` 是對 macOS 臨時 sshd 的預演,不是 Windows 的事實。引用的任務報告(`task-1-report.md` 到 `task-5-report.md`、`task-4-windows-run.md`、`progress.md`)在本機工作區 `.superpowers/sdd/2026-10-10-own-ssh-phase0-spike/`,沒有納入 git;引用處已把用到的數字寫在文中。

## 結論

| 條件 | 狀態 |
|---|---|
| 兩代 `ssh-key` 並存、編譯、互相認得公鑰 | PASS |
| 保管庫簽的簽章被 sshd 接受(Ed25519、ECDSA P-256、RSA 3072),錯的金鑰被拒 | PASS |
| exec、PTY shell、主機金鑰、keepalive、兩層跳板的基本行為 | PASS |
| Windows 實機連線與 exec | PASS(`spike windows` 通過) |

**判定:可行(go)** —— 條件都過了。規格先照下面「規格裡不成立的假設」的表修改,再開第 1 期。

判斷依據(逐項見下面的表):

- 四個條件都過:兩代 `ssh-key` 並存且互認(第 1 項);保管庫簽的三種金鑰被真的 sshd 接受、錯的金鑰被拒(第 3、4 項);exec、PTY shell、主機金鑰、keepalive、兩層跳板在臨時 sshd 上的行為都量到了(第 6 到 20 項);russh 連得上 Windows runner 的 OpenSSH server、登入、exec(「Windows」一節)。
- 「規格裡不成立的假設」沒有一列推翻設計,它們是實作必須遵守的合約。影響最大的三個:主機金鑰確認期間連線可能已經死了,引擎要看連線自己的存活狀態、不能看錯誤種類,並在釘住之後重連(第 14 項);拒絕 SHA-1 的 RSA 登入簽章是引擎自己的檢查,不是 russh 的設定(第 5 項);內層跳板的斷線原因只能從外層取得(第 19 項)。
- 代價:把 russh 加進 app,`Cargo.lock` 多 92 個套件、3 個預覽版,`zeroize` 被強制升到 1.9.1(第 2 項)。
- 還沒做完的(不在上面四個條件裡,第 1 期的計畫要排進去):密碼與 keyboard-interactive 的成功路徑、Windows 的 PTY、Linux、app 本身加上 russh 之後的 Windows build(「沒驗證到的」)。

## 每個問題的結果

| # | 問題 | 結果 | 證據 |
|---|---|---|---|
| 1 | app 的 `ssh-key` 0.6.7 與 russh 釘的 `ssh-key` 0.7.0-rc 能在同一個 build 並存嗎? | PASS | 附錄的 `cargo tree -i ssh-key`;`coexistence` 的三個測試 |
| 2 | 把 russh 加進 app,`src-tauri/Cargo.lock` 會多什麼? | 新增 92 個套件(34 個新 crate 加 58 個既有 crate 的第二版本;預覽版 3 個:pkcs1 0.8.0-rc.4、rsa 0.10.0-rc.18、ssh-key 0.7.0-rc.11;`zeroize` 1.8.2 → 1.9.1 是唯一被換掉的版本。Task 1 實測;`lock_delta.py` 的原始 `new_crates`/`prerelease_crates` 是 35/5,含試驗套件本身與 wasi build-metadata 的誤判,不要直接引用);app 既有的套件被換版本:1 個;已有的套件多出第二個版本:58 個 | 附錄的 lock delta(和重算的結果相同;本任務另用獨立的腳本從兩份 `Cargo.lock` 再算一次:34 + 58 = 92、預覽版 3 個、`zeroize` 是唯一被換掉的版本,結果相同);`aws-lc-rs` 只有一個版本(附錄) |
| 3 | 保管庫的 `Material::sign` 能接成 russh 的 `Signer` 嗎(Ed25519、ECDSA P-256、RSA 3072)? | PASS;RSA 在線上用的演算法:rsa-sha2-512(russh 給的 hash_alg:Some(Sha512)) | `vault_signer`;簽章格式見計畫 Task 2 |
| 4 | sshd 真的在驗保管庫簽的東西嗎(負向對照) | PASS | `vault_signer` |
| 5 | 只講 `ssh-rsa`(SHA-1)的舊伺服器(Review Focus 1) | PASS(測試斷言的是現狀:預設設定連得上、用 SHA-1 簽。這正是引擎要擋掉的,見規格表 §6.3 一列);russh 給的 hash_alg:None;線上演算法:ssh-rsa | `vault_signer` |
| 6 | exec:結束碼、stdout、stderr | PASS | `exec` |
| 7 | exec:被訊號殺掉(Review Focus 3) | PASS;exit_status = None,exit_signal = Some("KILL") | `exec` |
| 8 | exec:很長的輸出(Review Focus 5) | PASS(150.4 MiB/s,loopback、debug build,只說明 32 MiB 不會卡住,不是效能數字);讀到上限就關 channel,連線仍可用:PASS(下一個 exec 8 ms) | `exec` |
| 9 | exec 的逾時 | PASS:russh 沒有指令逾時,引擎自己用 `tokio::time::timeout` 加 `Channel::close`,連線之後仍可用 | `exec` |
| 10 | shell:PTY、提示、指令、結束碼 | PASS;第一個提示在 163 ms 內出現 | `shell` |
| 11 | shell:視窗大小(開 PTY 時給的、執行中改的、shell 就緒前改的:Review Focus 4) | PASS;連 PTY 都還沒有就送 window-change:最後大小 24 80,保留了嗎:false | `shell` |
| 12 | 密碼與 keyboard-interactive | 拒絕的路徑:PASS;`none` 認證列出的方法:MethodSet([PublicKey])。**成功的路徑沒有驗證:scratch sshd 沒有 PAM,要在真實主機上測** | `auth_methods` |
| 13 | 主機金鑰:接受、拒絕、釘住、比對 | PASS;被拒時 russh 回的錯誤:UnknownKey | `host_key` |
| 14 | 主機金鑰確認要等人:等得了嗎 | PASS;答案比伺服器的 `LoginGraceTime` 慢(3 秒的寬限、6 秒後才答。**這份 log 是伺服器恰好沒有切斷連線的那一種結果;同一個觀察在本任務的 17 次執行裡,13 次是連線已死、4 次沒被切斷(含這一次),是競態,見本表下方「第 14 項的細節」**):connect 回 connected,緊接著的第一次呼叫 Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false }),200 ms 後 Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false }),登入 Success,handler 記到的斷線 None(對照組、預設寬限時間:第一次呼叫 Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false }),登入 Success);等答案時連線被切斷,第一次呼叫的面貌(51 回合):{"empty_failure": 17, "inconsistent": 3, "recv_error": 17, "send_error": 14},之後 `is_closed()`:{true: 51},`Handle` future:{"Err(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 50, "Err(IO(Os { code: 54, kind: ConnectionReset, message: \"Connection reset by peer\" }))": 1};sshd 真正掉線的時間(`LoginGraceTime 1`,秒):3.61 s、3.05 s、1.71 s、1.40 s、2.65 s | `host_key` |
| 15 | 沒有共同的主機金鑰演算法(Review Focus 2) | PASS;錯誤:NoCommonAlgo { kind: Key, ours: ["rsa-sha2-256"], theirs: ["ssh-ed25519"] } | `host_key` |
| 16 | keepalive | PASS:interval 1 秒、max 2,3044 ms 後察覺,原因 Error(KeepaliveTimeout);有回應的連線不會被關:PASS | `keepalive` |
| 17 | 連線在指令執行中被切斷(Review Focus 3) | PASS:0 ms 內結束,原因 Error(IO(Custom { kind: UnexpectedEof, error: "early eof" })) | `keepalive` |
| 18 | 握手一直沒有回應 | PASS:russh 的 `Config` 沒有連線或握手逾時,`connect` 沒有上限(測試在我們自己的 2 秒逾時取消它) | `keepalive` |
| 19 | 兩層跳板(direct-tcpip → `into_stream` → `connect_stream`) | PASS;連到沒人聽的埠:ChannelOpenFailure(ConnectFailed);關掉第一跳後,第二跳的 `Handle` future:Err(IO(Custom { kind: BrokenPipe, error: "channel closed" }))、`is_closed()`:true、handler 的 `disconnected`:None(russh 不會替內層跳板呼叫它;原因要看外層自己的紀錄:Error(Disconnect)) | `jump` |
| 20 | 每個 channel 都有讀取工作 | PASS;沒人讀的 channel 旁邊的 exec 在 8 秒內回應了嗎:false(之後開始讀,3 秒內收到 498892800 bytes) | `channels` |
| 21 | `ipc/` 搬移(不改行為) | `ipc::` 與 `agent::` 共列 168 tests, 0 benchmarks;整個 lib 測試:test result: ok. 1417 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 97.23s(搬移前 `agent::` 單獨是 167 個〔Task 5 報告〕,多出來的 1 個是 `ipc/` 的守門測試,沒有任何測試消失);`ipc/peer.rs`、`ipc/pipe_windows.rs`、`ipc/server.rs` 與 `sync/slot_files_windows.rs`(含測試程式碼)用 `x86_64-pc-windows-msvc` 在本機型別檢查通過,故意弄錯會被抓到;`agent/` 自己的 `cfg(windows)` 幾行(`mod.rs`、`oneshot.rs`、`openssh_tests.rs`)不在這個本機檢查裡,只靠閱讀,由下面 Windows 的實際執行補上;`test-windows.yml` 的過濾加了 `ipc::`。**搬移後的測試已在 Windows 上實際跑過**:run 38059199511(`windows key slots`,`workflow_dispatch`,headSha 8e68ade,新過濾 `sync::slot_rules sync::slot_files vault:: ipc:: agent::`)`test result: ok. 217 passed; 0 failed; 0 ignored; 0 measured; 1139 filtered out; finished in 12.21s`,其中 28 個是 `ipc::` 測試;比搬移前的基準(run 38056602396,headSha d6d9017,舊過濾,216 passed)正好多一個守門測試、沒有測試消失,見「Windows」一節 | 計畫 Task 5;`git log` 裡的 `refactor(ipc)`;`gh run view 38059199511`(搬移後)、`gh run view 38056602396`(搬移前的基準) |
| 22 | 為了試驗動到 app 的哪裡 | `src-tauri/` 只有:`lib.rs` 的 `mod vault;` 改成 `pub mod vault;`(一行;`cargo check --lib` 的警告和改之前一樣,只有原本就有的 `set_host_enabled` 那一個;副作用:`vault/` 裡沒用到的 `pub` 項目不再被死碼警告抓到,今天沒有東西被藏起來)、`lib.rs` 的 `mod ipc;`,以及第 21 項的搬移;`vault/` 一個字沒動。`.github/workflows/` 動了 `test-windows.yml`(過濾)並新增 `spike-windows.yml`;`src-tauri/Cargo.toml` 與 `Cargo.lock` 對 main 的差異:0 行 | `git diff` |

### 第 14 項的細節:答案太慢時,伺服器切不切斷連線是競態

設定:伺服器 `LoginGraceTime 3`,host key 的答案 6 秒後才到(`observe_an_answer_slower_than_the_servers_login_grace_time`)。`connect` 每次都回 Ok(`host_key.slow_answer.outcome = connected`)。接著做第一次呼叫(`authenticate_none`)、等 200 ms 再呼叫一次、再登入。結果不是固定的:

| 來源 | 執行數 | 連線沒被切斷(兩次呼叫都是正常的拒絕,登入成功) | 第一次呼叫就是空的拒絕,200 ms 後 `SendError`,登入失敗 | 第一次呼叫還是正常的拒絕,200 ms 後 `SendError` |
|---|---|---|---|---|
| 主 log(整次平行執行;上表用的那一次) | 1 | 1 | 0 | 0 |
| 第二欄(整次 `--test-threads=1`) | 1 | 0 | 1 | 0 |
| 單獨重跑這個觀察(`--test-threads=1`) | 10 | 2 | 8 | 0 |
| `host_key` 整個 binary 平行重跑(`--include-ignored`) | 5 | 1 | 4 | 0 |
| 本任務合計 | 17 | 4 | 13 | 0 |
| Task 3 修正輪的重跑(取自 `task-3-report.md`,不是本任務的 log) | 8 | 0 | 5 | 3 |

- 預設寬限時間(120 秒)的對照組:本任務 17 次執行登入全部成功,所以失敗來自寬限時間,不是 6 秒的延遲。
- 為什麼是競態:同樣這幾次執行裡,用原始 TCP 量 sshd 切斷「握手後就不出聲的連線」要多久(`observe_when_sshd_drops_a_connection_that_stays_quiet`):`LoginGraceTime 1` 落在 1.11 到 4.84 秒(35 個樣本),`LoginGraceTime 3` 落在 3.10 到 6.93 秒(21 個樣本,其中 4 個 ≥ 6 秒)。寬限時間的執行很粗:答案比切斷早到,連線就活下來。活下來的比例(4/17)和切斷時間 ≥ 6 秒的比例(4/21)是同一個量級,符合這個解釋,但那個探針是另一種情境,不是證明。為什麼這麼粗沒有查清楚(OpenSSH 10.3 會為每條連線重新執行 `sshd-session`/`sshd-auth`,`LogLevel DEBUG1` 下記為 `Timeout before authentication`)。其他版本的 sshd 沒量過。
- 與寬限時間無關、可以重現的版本:用 proxy 在確認視窗開著的 0.3 秒時切斷 TCP、0.6 秒時回答「信任」。一般測試 `a_connect_that_returned_ok_on_a_dead_session_is_found_out_by_the_first_call` 釘住它:`authenticate_none` 與 keyboard-interactive 的第一次呼叫必須是死掉的樣子,`is_closed()` 為 true,`Handle` future 以 `Err` 結束。ignored 的 51 回合觀察(三種呼叫各 17 回合)在本任務的 7 次執行共 357 回合(每種呼叫 119 回合)裡:`connect` 從沒失敗;第一次呼叫後 `is_closed()` 一律 true(357/357);`Handle` future 一律 `Err`(355 次 `UnexpectedEof "early eof"`、2 次 `ConnectionReset`;handler 記到的原因分布相同);第一次呼叫的樣子:`authenticate_none` 119 次都是空的拒絕、keyboard-interactive 119 次都是 `RecvError`、`best_supported_rsa_hash` 92 次 `SendError` 加 27 次 `Inconsistent`,沒有一次正常回答。Task 3 的複審重跑同一個觀察時,在 17 回合裡看過 1 次 `best_supported_rsa_hash` 正常回答(`Ok(Some(Some(Sha512)))`;記在 `progress.md`,不是本任務的 log),所以它不能當存活探針。讀原始碼的推測:它向 session 要本地已有的 `server-sig-algs`(client/mod.rs:767-795),session 還沒發現連線死了就答得出來;沒有另外驗證。
- 結論(對引擎):`connect` 回 Ok 不代表連線活著;第一次呼叫成功也不代表;任何錯誤的種類都不能當證據。連線是否活著只看連線自己的狀態(`Handle` future 是否結束、`is_closed()`、handler 記下的 `disconnected`);使用者答完、釘住之後若連線已死就重連。伺服器的寬限時間引擎無從得知,所以限制確認視窗的時間不是解法。

## 規格裡不成立的假設

每一列都用上面的證據核對過。證據支持規格說法的列已移到下一節「核對過、沒問題」;留在這裡的,是規格的說法不成立、或不完整到會改變第 1 期做法的。「原本的說法」引規格現行的句子;§12 與 §15 的兩列是例外,引的是試驗開始前就已改掉的舊句子(標了 commit),留著是因為試驗給了它們證據。標「文件」的列來自 russh 0.64.1 的原始碼,這份試驗的測試沒有推翻它。

| 規格位置 | 原本的說法 | 實測 | 建議的規格修改 |
|---|---|---|---|
| §12(bfd5b35 之前的版本) | 整合測試「沿用 `agent/openssh_tests.rs` 起 sshd 的模式」 | 那個檔案從來沒起過 sshd(它只用 `ssh-keygen`、`ssh-add`)。試驗自己建了一套:臨時目錄、空閒埠、測試用主機金鑰與 authorized_keys、一般使用者執行 `sshd -D -e -f`(`spike/russh/src/harness.rs`,`harness_smoke` 兩個測試證明它能用)。另外量到一個會咬人的地方:OpenSSH 9.8 起預設開 `PerSourcePenalties`,同一個來源位址(測試全是 127.0.0.1)造成夠多次未認證就斷線(`noauth` 1 秒、`grace-exceeded` 10 秒,累計到 15 秒就開始執行;預設值已對照本機的 `man sshd_config`),sshd 之後直接不回橫幅地拒絕這個來源;第一次量 `LoginGraceTime` 的探針就撞上了(同一個 sshd 的第三條連線沒有橫幅)。 | 規格已在 bfd5b35 改正。第 1 期把這套搬進 app 的測試支援程式碼,並照做:每個測試自己起一個 sshd、不共用(一般測試今天就是這樣,只造成幾次未認證斷線);非得共用同一個 sshd 的長迴圈或計時觀察,設定 `PerSourcePenalties no`——這個關鍵字 OpenSSH 9.8 以前不認得、`sshd -t` 會拒絕設定,所以不能放進預設設定。 |
| §15、§6.3(文件) | §15 現行寫法:`Signer::auth_sign(&AgentIdentity, Vec<u8>)` 收待簽緩衝區,要回傳「原緩衝區 + u32 長度 + 簽章 blob」;`authenticate_publickey_with(user, PublicKey, hash_alg, &mut impl Signer)` | 合約是對的:三種金鑰都被真的 sshd 接受(`an_ed25519_key_in_the_vault_logs_in`, `an_ecdsa_p256_key_in_the_vault_logs_in`, `an_rsa_3072_key_in_the_vault_logs_in_with_a_sha2_signature`),錯的金鑰被拒(`a_signature_from_another_key_is_refused`)。簽名少了兩樣,實際是 `fn auth_sign(&mut self, key: &AgentIdentity, hash_alg: Option<HashAlg>, to_sign: Vec<u8>) -> impl Future<Output = Result<Vec<u8>, Self::Error>> + Send`:有 `&mut self` 和 `hash_alg`,非同步,回傳 `Result`。`to_sign` 開頭是 `string(session id)`;回傳必須是 `to_sign` 原樣接上 `u32 長度 ‖ 簽章 blob`(`Material::sign` 回的正是那個 blob),russh 之後把 session id 那段切掉再送(client/encrypted.rs:271-307)。RSA 用哪個雜湊由呼叫者傳的 `hash_alg` 決定:`Some(Sha512)` = flag 4、`Some(Sha256)` = 2、`None` = 0 即 `ssh-rsa`;其他金鑰型別忽略它。 | §15 的簽名補上 `&mut self` 與 `hash_alg`;§6.1 的 `AuthSource::sign` 一節寫上這個合約,並註明 RSA 的雜湊是引擎傳進來的(拒絕 SHA-1 見下面 §6.3 一列)。 |
| §15(8b78077 加的一行;4c545ec 已改) | 「russh 的預設不會帶進新的加密堆疊;待驗證的只剩 `ssh-key` 兩個版本並存」 | TLS 後端確實沒有新增(`aws-lc-rs` 1.18.1 與 app 共用同一個版本,只有一個);但 russh 帶進第二代 RustCrypto 預覽版(`ssh-key =0.7.0-rc.11`、`rsa =0.10.0-rc.18`、`p256`/`p384`/`p521` 0.14、`ed25519-dalek` 3、`curve25519-dalek` 5、`ecdsa` 0.17、`elliptic-curve` 0.14、`crypto-bigint` 0.7、`ml-kem` 0.3 等):`Cargo.lock` 新增 92 個套件(34 個新 crate、58 個既有 crate 的第二版本),其中 3 個是預覽版(pkcs1 0.8.0-rc.4、rsa 0.10.0-rc.18、ssh-key 0.7.0-rc.11);另外 `zeroize` 1.8.2 被強制升到 1.9.1(`ssh-key 0.7.0-rc.11` 的 manifest 要 `zeroize 1.9`;app 的 `zeroize = "1"` 允許,不用改 manifest;保管庫的 `generate.rs`、`export.rs`、`store.rs` 用它)。兩代 `ssh-key` 並存、編譯、互認公鑰與指紋(第 1 項)。 | 現行 §15 已寫「TLS 那一層不會多一套;但 russh 會帶進第二組 RustCrypto 的 rc 版本」,方向對了;補上數量(92 個套件、3 個預覽版)與 `zeroize` 的升版;`cargo audit` 與升版要看兩代;第 1 期把 russh 放進 `src-tauri/Cargo.lock` 之後,重跑整個 lib 測試,特別是 `vault::`。 |
| §6.3(4c545ec 之後的版本) | 「把 `ssh-rsa`(SHA-1)從主機金鑰與簽章演算法清單拿掉(russh 的預設清單含它…)」 | 在 russh 裡這是兩件不同的事,不是同一份清單。(1) 主機金鑰:`Preferred::DEFAULT.key` 的最後一項是 `Algorithm::Rsa { hash: None }`,也就是 `ssh-rsa`(SHA-1)(FACT `host_key.russh_default_key_list`)。預設設定因此連得上只講 `ssh-rsa` 的舊伺服器,登入時用 SHA-1 簽章(第 5 項:russh 給的 `hash_alg` 是 None,線上演算法 ssh-rsa;`an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1` 通過)。把那一項從 `Config::preferred.key` 拿掉就夠,已驗證(`a_host_key_list_without_ssh_rsa_has_nothing_in_common_with_a_server_that_only_has_ssh_rsa`):只講 `ssh-rsa` 的伺服器得到 `NoCommonAlgo { kind: Key, ours: [六個名稱], theirs: ["ssh-rsa"] }`,兩份清單都在錯誤裡,可以直接組成使用者看的訊息(FACT `host_key.no_common_algorithm.ssh_rsa_only_server.error`);對照組證明是這份清單造成的:預設清單連得上同一台,短清單連得上一般伺服器。(2) 使用者認證的簽章:russh 沒有「簽章演算法清單」。RSA 用哪個雜湊,是呼叫者傳給 `authenticate_publickey_with` 的 `hash_alg` 引數;russh 原樣轉給 `Signer::auth_sign`、不檢查(client/mod.rs:515、:536);慣例是採用 `best_supported_rsa_hash()` 的答案,只講 `ssh-rsa` 的伺服器它回 Some(None),於是簽 SHA-1。`Preferred` 管不到這件事。 | §6.3 拆成兩條:(a) `Config::preferred.key` = `Preferred::DEFAULT.key` 去掉 `Rsa { hash: None }`;沒有共同演算法時,用 `NoCommonAlgo` 的 `ours` 與 `theirs` 組出錯誤訊息(已驗證);(b) 拒絕 SHA-1 的 RSA 登入簽章是引擎自己的檢查:RSA 金鑰且 `hash_alg` 為 `None` 就不簽、回明確的錯誤。(b) 還沒有測試——試驗只釘住「預設行為是會簽 SHA-1」,第 1 期要加一個會擋下來的測試。 |
| §6.2 第 5 步 | 「keepalive:每 `keepalive_secs` 一次,連續 `keepalive_max_missed` 次沒回應就斷線並通知擁有者」 | russh 的 `keepalive_interval` 是「這麼久沒收到任何東西才送一次」,`keepalive_max` 是未回應的探測數,超過才斷:收到任何資料就把未回應數歸零(client/mod.rs:1507-1514);`keepalive_max = 0` 表示永不因此斷線;認證完成之後才計時、才送(:1426-1429)。預設 `keepalive_interval: None`,也就是關閉。實測 interval 1 秒、max 2 的連線在 3044 ms 後才被判定斷線,原因是 `KeepaliveTimeout`,也就是 (max + 1) × interval;有回應的連線不會被關(`keepalive_closes_a_silent_link_and_reports_keepalive_timeout`, `a_link_that_answers_keeps_the_session_alive_across_several_intervals`,第 16 項)。 | 引擎要自己設 `keepalive_interval`(預設是關的);設定名對上 russh 的語意:察覺斷線要 `(keepalive_max_missed + 1) × keepalive_secs`,§11 的 `no response for {n} seconds` 用這個值;keepalive 管不到認證之前,那一段由 connect 的逾時和連線存活檢查負責(見下面兩列)。 |
| §6.2 第 2 步 | 「連第一跳(TCP + 連線逾時)」 | russh 的 `Config` 沒有連線或握手逾時(`inactivity_timeout` 與 `keepalive_interval` 預設都是 `None`);握手一直沒有回應時,`connect` 沒有上限(第 18 項:測試在我們自己的 2 秒逾時取消它,`handshake_stall.waited_ms = 2002`)。 | 引擎用 `tokio::time::timeout` 包住整個 `connect`(含握手),時間取 host 記錄的 `connect_timeout_secs`;規格的「連線逾時」要涵蓋握手,不能只是 TCP 連線的逾時。 |
| §6.2 第 3 步 | 「`Pinned` 繼續;`Unknown` → broker 在 app 視窗要求確認…;`Mismatch` → 警告視窗…」 | `check_server_key` 只能回 true/false;「沒看過」和「不符」被拒時是同一個錯誤(UnknownKey),russh 不會告訴引擎為什麼。更大的問題在等人的那段時間:russh 在 handler 的 `check_server_key` 回傳之前不處理這條連線(它在 session 迴圈的讀取分支裡 inline `await`,client/mod.rs:2018;`select!` 從 :1380 開始,那條分支裡的 `reply` 等待在 :1402),所以使用者看確認視窗的整段時間沒有人在讀 socket,伺服器那邊的 `LoginGraceTime`(預設 120 秒)照樣在跑。後果:答案太慢、或連線在等答案時被切斷,`connect` 仍然回 Ok(7 次執行共 357 個切斷回合,`connect` 從沒失敗),當下 `is_closed()` 是 false;失敗到之後的呼叫才浮現,而且**沒有單一的樣子**:`authenticate_none` 是空的 `AuthResult::Failure`(`remaining_methods` 為空,會被誤認成認證失敗)、keyboard-interactive 是 `Err(RecvError)`(偶爾 `SendError`,Task 3)、`best_supported_rsa_hash` 是 `SendError` 或 `Inconsistent`,甚至可能正常回答(Task 3 複審在 17 回合裡看過 1 次,本任務的 119 回合沒重現),所以 `login_with_key_file` 的第一步 `best_supported_rsa_hash` 不能當存活探針。對真的伺服器(`LoginGraceTime 3`、6 秒後才答)結果是競態,因為這台 sshd 執行寬限時間很粗(`LoginGraceTime 1` 實測 1.11 到 4.84 秒才切):本任務 17 次執行,13 次連線已死(第一次呼叫就是空的拒絕、200 ms 後 `SendError`、登入失敗)、4 次整個沒被切斷;Task 3 的 8 次重跑,5 次已死、3 次第一次呼叫還是正常的拒絕(列出 `PublicKey`)但 200 ms 後就死了。所以第一次呼叫得到正常的拒絕,不代表連線還活著(細節見「第 14 項的細節」)。 | 引擎自己記下拒絕的原因(決定回 `HostKeyMismatch` 還是使用者取消)。連線是否還活著看連線自己的狀態,不看錯誤的 variant(否則斷線會被誤報成認證失敗):`Handle` future 是否已結束、`is_closed()`、或 handler 記下的 `disconnected`(單一跳可靠;內層跳板見下一列)。`Handle` future 在連線被切斷時是 `Err`,在伺服器正式送 SSH_MSG_DISCONNECT 時是 `Ok(())`(讀原始碼 client/mod.rs:1314-1319,沒有測試實際觀察到),所以判準是「結束了」,不是「Err」。使用者答完、釘住之後若連線已死,就重連一次(已釘住的金鑰第二次不會再問)。伺服器的寬限時間引擎無從得知,所以「限制確認視窗的時間」不是解法。 |
| §6.2 第 2 步(跳板)、§7.1 `event{disconnected{reason}}` | (規格沒明說,是第 0 期計畫的預期)每一跳都是一個 russh session,斷線的原因由它自己的 `Handler::disconnected` 報告;外層關掉,內層也跟著結束 | 內層確實跟著結束,但 russh 從來不為內層跳板呼叫 `Handler::disconnected`:不論是 `disconnect()` 關掉第一跳,還是切斷第一跳底下的 TCP,第二跳的 `Handle` future 立刻以 `Err(IO(Custom { kind: BrokenPipe, error: "channel closed" }))` 結束(FACT `jump.close_first_hop.second_hop_end`)、`is_closed()` 為 true,handler 的 `disconnected` 是 None。那個 `BrokenPipe` 是次要的錯誤,不是原因:`Session::run` 先對內層串流做 `shutdown`(client/mod.rs:1313),內層串流就是外層的 channel,外層已經沒了,shutdown 失敗、`?` 先返回,走不到 :1317 的 `handler.disconnected`。真正的原因只在外層自己的紀錄裡:關掉的是 Error(Disconnect),切斷的是 `UnexpectedEof` "early eof"(FACT `jump.cut_first_hop.first_hop_reason`)。另外,只丟掉外層的 `Handle` 不會結束任何一跳:內層的 channel 還握著 session 的 `Sender`,內層照常 exec(`jump.drop_first_handle.exec_on_second_hop`);要明確 `disconnect()` 才會一起結束,兩邊的 `Handle` 都丟掉時外層才結束(推論自 0.64.1 與一種 transport,升版要重查)。 | §6.2:跳板鏈的斷線原因取自出事那一跳(最外層有紀錄的那個)自己的紀錄;內層的 `Handle` future 與 `is_closed()` 只回答「結束了」。broker 的 `disconnected{reason}` 因此要從外層補。連線物件要明確擁有整條鏈的生命週期,不靠丟掉 `Handle` 來關。 |
| §7.1 `exec_result{exit_code, …}` | 結果有結束碼 | 指令被訊號殺掉時沒有結束碼(exit_status = None,exit_signal = Some("KILL"));連線被切斷時也沒有(第 17 項)。 | `exit_code` 改成可空,加 `signal` 欄位;MCP 的 `ssh_exec` 要回「被 KILL 終止」或「連線中斷」,不能回 0。 |
| §7.3 視窗大小 | 「視窗大小變更 → `window_change`」(§7.3;§6.1 的 `window_change(cols, rows)` 沒說 shell 還沒就緒時怎麼辦) | PTY 建好之後送的有效(第 11 項);連 PTY 都還沒有時送的,最後大小是 24 80(保留了嗎:false)。 | broker 記住最後一次的大小,shell 就緒後再送一次。 |

## 核對過、沒問題

- §6.3「每個 channel 都有專屬的讀取工作(russh 已知:沒人讀的 channel 會卡住整條連線…)」:`a_second_channel_works_while_the_first_is_drained_in_the_background` 通過(有讀取工作時,旁邊的 channel 照常運作);沒人讀的那個 channel 讓旁邊的 exec 8 秒內沒有回應(`channels.unread_neighbour.answered_within_8s = false`),開始讀之後連線恢復(第 20 項)。規格的紀律照舊,不用改;可以把「實測 8 秒內沒有回應」補進 §6.3。
- §6.1 `Session::exec(command, timeout)` 的 `timeout`:russh 的 `Channel::exec` 沒有逾時,逾時由引擎自己做,用 `tokio::time::timeout` 加 `Channel::close`,連線之後仍可用(`a_command_timeout_is_ours_to_enforce_and_the_connection_survives_it`,第 9 項)。規格本來就把 `timeout` 放在引擎的介面上,沒有說 russh 會幫忙,所以不是錯的假設,只是實作要自己做。
- 規格自己列的待試驗項目(§6.3、§16):`ssh-key` 0.6.7 與 0.7.0-rc 並存、編譯、互認公鑰與指紋(`both_ssh_key_versions_parse_the_same_public_keys_and_agree_on_the_fingerprints`, `the_lock_file_holds_ssh_key_0_6_7_and_a_0_7_release_candidate`, `the_app_library_is_linked_into_the_same_binary`,第 1 項);`Signer` 接 `Material::sign`,三種金鑰都被 sshd 接受(第 3、4 項;簽名細節見上一節);Windows 實機連線加 exec(「Windows」一節)。
- §6.2 第 2 步的兩層跳板(direct-tcpip → `into_stream` → `connect_stream`):`a_two_hop_jump_runs_a_command_on_the_second_host` 通過;第二跳自己檢查主機金鑰、被拒時第一跳不受影響(`the_second_hop_checks_its_own_host_key_and_the_first_hop_survives_a_refusal`);連到沒人聽的埠乾淨地失敗(`a_jump_to_a_closed_port_fails_cleanly`,第 19 項)。斷線的行為不同,見上一節。

## 第 1 期要小心的實測發現(不是規格寫錯)

- **Windows 的 exec 輸出是 CRLF。** Windows OpenSSH 回來的 `echo` 輸出是 `"spike-windows-ok\r\n"`,macOS 臨時 sshd 是 `"spike-windows-ok\n"`(「Windows」一節)。引擎不能假設 `\n`;MCP 的 `ssh_exec` 照原樣回位元組,SSHelter 自己要切行的地方兩種都要收。
- **russh 的預設幾乎都是「不設限」。** `Config::default()`:`keepalive_interval: None`(keepalive 關閉)、`inactivity_timeout: None`、沒有連線或握手逾時;keepalive 要認證完成後才計時(client/mod.rs:1426-1429)。所以未認證的那一段(連線、握手、host key 確認、登入)沒有任何自己會觸發的上限:引擎必須自己加逾時,並設好 keepalive(規格表 §6.2 第 2、5 步)。
- **所有計時與競態的數字只在 macOS、OpenSSH 10.3p1、russh 0.64.1 量過。** sshd 的 `LoginGraceTime` 執行方式和 `PerSourcePenalties` 的預設都跟版本有關;Linux 與其他版本的 sshd 沒量過(「沒驗證到的」)。

## 用到的 russh 0.64.1 介面

下面這些名稱都是從 docs.rs 讀來的,Task 2 到 Task 3 逐一編譯過。與 docs.rs 不同、編譯時改過的名稱:無(Task 1 到 Task 3 的報告都記下沒有任何名稱需要依編譯器修改;這份清單的每個名稱在試驗程式裡至少用到一次,所以都編譯過)。

- 簽章:`russh::Signer::auth_sign(&mut self, &AgentIdentity, Option<HashAlg>, Vec<u8>) -> impl Future<Output = Result<Vec<u8>, Self::Error>> + Send`(`type Error: From<russh::SendError>`;預設沒開 `async-trait`,實作直接寫 `async fn`)、`russh::keys::agent::AgentIdentity::public_key()`、`Handle::authenticate_publickey_with(user, PublicKey, Option<HashAlg>, &mut S)`、`Handle::best_supported_rsa_hash()`
- 連線:`client::{connect, connect_stream, Config { keepalive_interval, keepalive_max, preferred }, Handler::{check_server_key(&PublicKeyOrCertificate), disconnected(DisconnectReason)}}`、`russh::keys::{PublicKey, HashAlg, Algorithm, PrivateKeyWithHashAlg, PublicKeyOrCertificate, load_secret_key}`、`Preferred::DEFAULT`
- 認證:`Handle::{authenticate_publickey, authenticate_password, authenticate_none, authenticate_keyboard_interactive_start, authenticate_keyboard_interactive_respond}`、`client::{AuthResult, KeyboardInteractiveAuthResponse, Prompt}`、`MethodKind`
- channel:`Handle::{channel_open_session, channel_open_direct_tcpip, disconnect, is_closed}`、`Channel::{exec, request_pty, request_shell, window_change, data, wait, close, into_stream}`、`ChannelMsg::{Data, ExtendedData, ExitStatus, ExitSignal, Failure}`、`Disconnect::ByApplication`
- 錯誤:`russh::Error::{UnknownKey, KeepaliveTimeout, NoCommonAlgo, ChannelOpenFailure, Keys}`、`SendError`

## 沒驗證到的(要在真實環境補)

- 密碼與 keyboard-interactive 的**成功**路徑:scratch sshd 沒有 PAM。`auth::login_password` 與 `auth::login_keyboard_interactive` 只跑到伺服器拒絕。第 1 期的整合測試要在有 PAM 的主機或容器上補。
- Windows 上的 PTY/shell(ConPTY)與視窗大小;`sshelter connect` 在 Windows Terminal 與傳統 console 的行為(第 3 期)。
- Linux:這份試驗只在 macOS 跑過;RHEL/Fedora 的 crypto policy 會讓第 5 項失敗(用 `SPIKE_SKIP_SHA1=1` 略過)。
- 長時間連線(數小時)、rekey(`Limits` 預設 1 GiB 或 3600 秒)、大量並行 channel。
- `cargo audit` 對 russh 兩代 RustCrypto 的結果。
- (已補)搬移後的 `ipc::` 測試與 `agent/` 裡 `cfg(windows)` 的幾行在 Windows 上實際跑過:run 38059199511(headSha 8e68ade,過濾 `sync::slot_rules sync::slot_files vault:: ipc:: agent::`)`test result: ok. 217 passed; 0 failed; 0 ignored; 0 measured; 1139 filtered out; finished in 12.21s`,見第 21 項與「Windows」一節。原本列在這裡,是因為 run 38056602396 只是搬移前的基準(它跑的是 d6d9017)。
- app 本身加上 russh 之後的 Windows build。`spike windows` 只建試驗 crate(`--no-default-features`,不含 app 函式庫);本機也沒法預先檢查,因為 `aws-lc-sys` 的 build script 需要 Windows SDK 的標頭。第 1 期把 russh 放進 `src-tauri/` 之後,第一次 Windows CI 才會知道。
- app 自己的測試在加了 russh 的 `src-tauri/Cargo.lock` 之下的結果,尤其是用到 `zeroize` 的 `vault::`(1.8.2 → 1.9.1)。試驗只證明 app 函式庫在試驗自己的 lock 下編得過、保管庫的簽章在那個 lock 下被 sshd 接受,沒有用那份 lock 跑 app 的整個測試。
- 沒有 `server-sig-algs` 的伺服器(很舊的 OpenSSH、老路由器):讀原始碼,`best_supported_rsa_hash()` 這時回 `Ok(None)`(client/mod.rs:767-795)。這台 sshd 一定回答 ext-info,產生不出這種伺服器,測試沒有涵蓋。
- sshd 的其他版本:`LoginGraceTime` 的執行方式(這台 OpenSSH 10.3p1 執行得很粗,原因沒查清楚)和 `PerSourcePenalties` 都跟版本有關,只在 macOS 的 OpenSSH 10.3p1 量過。
- 測試用的 `run_within`(`spike/russh/src/harness.rs`):逾時殺掉子行程之後還會等讀管線的執行緒結束;若有孫行程抓著管線,30 秒的上限會被拉長。目前用到的四個工具(`ssh-keygen`、`sshd -t`、`id -un`、`ssh`)不會;第 1 期把它搬進 app 的測試支援程式碼時再處理(Task 3 複審的 Minor 2,延後)。

## 附錄

### `cargo tree -i ssh-key`

```
$ cargo tree --offline -i ssh-key@0.6.7
ssh-key v0.6.7
└── sshelter v0.16.0 (/Users/ysya/project/sideproj/sshelter/src-tauri)
    └── russh-spike v0.0.0 (/Users/ysya/project/sideproj/sshelter/spike/russh)
[dev-dependencies]
└── russh-spike v0.0.0 (/Users/ysya/project/sideproj/sshelter/spike/russh)

$ cargo tree --offline -i ssh-key@0.7.0-rc.11
ssh-key v0.7.0-rc.11
└── russh v0.64.1
    └── russh-spike v0.0.0 (/Users/ysya/project/sideproj/sshelter/spike/russh)
```


### `cargo tree -i aws-lc-rs`

```
aws-lc-rs v1.18.1
├── russh v0.64.1
│   └── russh-spike v0.0.0 (/Users/ysya/project/sideproj/sshelter/spike/russh)
├── rustls v0.23.40
│   ├── hyper-rustls v0.27.9
│   │   └── reqwest v0.13.4
│   │       ├── sshelter v0.16.0 (/Users/ysya/project/sideproj/sshelter/src-tauri)
│   │       │   └── russh-spike v0.0.0 (/Users/ysya/project/sideproj/sshelter/spike/russh)
│   │       └── tauri-plugin-updater v2.10.1
│   │           └── sshelter v0.16.0 (/Users/ysya/project/sideproj/sshelter/src-tauri) (*)
│   ├── reqwest v0.13.4 (*)
│   ├── rustls-platform-verifier v0.7.0
│   │   └── reqwest v0.13.4 (*)
│   ├── tauri-plugin-updater v2.10.1 (*)
│   └── tokio-rustls v0.26.4
│       ├── hyper-rustls v0.27.9 (*)
│       └── reqwest v0.13.4 (*)
└── rustls-webpki v0.103.13
    └── rustls v0.23.40 (*)
```


### Cargo.lock 差異(試驗的 lock 對 app 的 lock)

(下面是 Task 1 存下來的檔案,和用 `lock_delta.py` 對兩份 `Cargo.lock` 重算的結果逐位元相同。三個原始數字不能直接引用:`lock.new_crates = 35` 含試驗套件自己,實際是 34;`lock.prerelease_crates = 5` 含兩個 wasi 的 build-metadata 誤判,實際是 3(pkcs1、rsa、ssh-key);`lock.dropped_from_the_app_lock = 10` 少算了只被 `mockito` 用的 `rand`、`rand_chacha`、`rand_core` 0.9,實際是 13 個只在 app 的 lock 裡的開發相依套件。)

```
FACT lock.new_crates = 35
FACT lock.second_versions_of_crates_the_app_has = 58
FACT lock.app_versions_replaced = 1
FACT lock.dropped_from_the_app_lock = 10
FACT lock.prerelease_crates = 5

## New crates (35)
argon2 0.6.0
blake2 0.11.0
cmov 0.5.4
cpubits 0.1.1
crypto-primes 0.7.2
ctutils 0.4.3
data-encoding 2.11.1
delegate 0.13.5
des 0.9.0
enum_dispatch 0.3.13
futures 0.3.32
hex-literal 1.1.0
hybrid-array 0.4.15
keccak 0.2.2
kem 0.3.0
md5 0.8.1
ml-kem 0.3.2
module-lattice 0.2.3
pageant 0.2.4
password-hash 0.6.1
phc 0.6.1
pkcs5 0.8.1
primefield 0.14.0
russh 0.64.1
russh-cryptovec 0.62.0
russh-spike 0.0.0
russh-util 0.52.0
salsa20 0.11.0
scrypt 0.12.0
serdect 0.4.3
sha3 0.11.0
sha3 0.12.0
sponge-cursor 0.1.0
tokio-macros 2.7.2
wnaf 0.14.1

## A second version of a crate the app already has (58)
aead: app has ['0.5.2'], spike adds ['0.6.1']
aes-gcm: app has ['0.10.3'], spike adds ['0.11.1']
aes: app has ['0.8.4'], spike adds ['0.9.3']
base16ct: app has ['0.2.0'], spike adds ['1.0.0']
bcrypt-pbkdf: app has ['0.10.0'], spike adds ['0.11.0']
block-buffer: app has ['0.10.4'], spike adds ['0.12.1']
block-padding: app has ['0.3.3'], spike adds ['0.4.2']
blowfish: app has ['0.9.1'], spike adds ['0.10.0']
cbc: app has ['0.1.2'], spike adds ['0.2.1']
cipher: app has ['0.4.4'], spike adds ['0.5.2']
const-oid: app has ['0.9.6'], spike adds ['0.10.2']
crypto-bigint: app has ['0.5.5'], spike adds ['0.7.5']
crypto-common: app has ['0.1.7'], spike adds ['0.2.2']
ctr: app has ['0.9.2'], spike adds ['0.10.1']
curve25519-dalek: app has ['4.1.3'], spike adds ['5.0.0']
der: app has ['0.7.10'], spike adds ['0.8.2']
digest: app has ['0.10.7'], spike adds ['0.11.3']
ecdsa: app has ['0.16.9'], spike adds ['0.17.0']
ed25519-dalek: app has ['2.2.0'], spike adds ['3.0.0']
ed25519: app has ['2.2.3'], spike adds ['3.0.0']
elliptic-curve: app has ['0.13.8'], spike adds ['0.14.1']
ff: app has ['0.13.1'], spike adds ['0.14.0']
fiat-crypto: app has ['0.2.9'], spike adds ['0.3.0']
generic-array: app has ['0.14.7'], spike adds ['1.4.5']
ghash: app has ['0.5.1'], spike adds ['0.6.0']
group: app has ['0.13.0'], spike adds ['0.14.0']
hkdf: app has ['0.12.4'], spike adds ['0.13.0']
hmac: app has ['0.12.1'], spike adds ['0.13.0']
inout: app has ['0.1.4'], spike adds ['0.2.2']
num-bigint: app has ['0.4.8'], spike adds ['0.5.1']
p256: app has ['0.13.2'], spike adds ['0.14.0']
p384: app has ['0.13.1'], spike adds ['0.14.0']
p521: app has ['0.13.3'], spike adds ['0.14.0']
pbkdf2: app has ['0.12.2'], spike adds ['0.13.0']
pem-rfc7468: app has ['0.7.0'], spike adds ['1.0.0']
pkcs1: app has ['0.7.5'], spike adds ['0.8.0-rc.4']
pkcs8: app has ['0.10.2'], spike adds ['0.11.0']
poly1305: app has ['0.8.0'], spike adds ['0.9.1']
polyval: app has ['0.6.2'], spike adds ['0.7.3']
primeorder: app has ['0.13.6'], spike adds ['0.14.0']
rfc6979: app has ['0.4.0'], spike adds ['0.6.0']
rsa: app has ['0.9.10'], spike adds ['0.10.0-rc.18']
sec1: app has ['0.7.3'], spike adds ['0.8.1']
sha1: app has ['0.10.7'], spike adds ['0.11.0']
sha2: app has ['0.10.9'], spike adds ['0.11.0']
signature: app has ['2.2.0'], spike adds ['3.0.0']
spki: app has ['0.7.3'], spike adds ['0.8.1']
ssh-cipher: app has ['0.2.0'], spike adds ['0.3.0']
ssh-encoding: app has ['0.2.0'], spike adds ['0.3.0']
ssh-key: app has ['0.6.7'], spike adds ['0.7.0-rc.11']
syn: app has ['1.0.109', '2.0.117'], spike adds ['3.0.7']
universal-hash: app has ['0.5.1'], spike adds ['0.6.1']
untrusted: app has ['0.9.0'], spike adds ['0.7.1']
windows-collections: app has ['0.2.0'], spike adds ['0.3.2']
windows-future: app has ['0.2.1'], spike adds ['0.3.2']
windows-numerics: app has ['0.2.0'], spike adds ['0.3.1']
windows-threading: app has ['0.1.0'], spike adds ['0.2.1']
windows: app has ['0.61.3'], spike adds ['0.62.2']

## App versions REPLACED by another version (this would change src-tauri/Cargo.lock) (1)
zeroize: app has ['1.8.2'], spike has ['1.9.1']

## In the app lock only (dev-dependencies are not resolved through a path dependency) (10)
assert-json-diff 2.0.2
colored 3.1.1
httpdate 1.0.3
mockito 1.7.2
ryu 1.0.23
serde_urlencoded 0.7.1
similar 2.7.0
termcolor 1.4.1
ts-rs 10.1.0
ts-rs-macros 10.1.0

## Pre-release crates in the spike lock (5)
pkcs1 0.8.0-rc.4
rsa 0.10.0-rc.18
ssh-key 0.7.0-rc.11
wasi 0.11.1+wasi-snapshot-preview1
wasip3 0.4.0+wasi-0.3.0-rc-2026-01-06
```


### 所有測試

本次執行只有一行 `skipped:`,在 `connect_log_in_and_exec_against_the_server_in_the_environment`(沒設 `SPIKE_SSH_PORT`、`SPIKE_SSH_USER`、`SPIKE_SSH_KEY`),所以它在表上的 PASS 在這台機器上只是「沒有失敗」,真的跑是在 Windows 的工作裡。其餘測試都對 `/usr/sbin/sshd`(OpenSSH 10.3p1)起的臨時 sshd 真的跑過,沒有任何一個因為找不到 sshd 而略過。

| test | outcome |
|---|---|
| `a_command_killed_by_a_signal_reports_the_signal_and_no_exit_status` | PASS |
| `a_command_timeout_is_ours_to_enforce_and_the_connection_survives_it` | PASS |
| `a_connect_that_returned_ok_on_a_dead_session_is_found_out_by_the_first_call` | PASS |
| `a_connection_cut_during_an_exec_ends_it_without_an_exit_status` | PASS |
| `a_handshake_that_never_gets_an_answer_needs_our_own_timeout` | PASS |
| `a_host_key_list_without_ssh_rsa_has_nothing_in_common_with_a_server_that_only_has_ssh_rsa` | PASS |
| `a_jump_to_a_closed_port_fails_cleanly` | PASS |
| `a_key_that_is_not_in_authorized_keys_is_refused` | PASS |
| `a_link_that_answers_keeps_the_session_alive_across_several_intervals` | PASS |
| `a_password_is_refused_by_a_key_only_server_and_the_refusal_lists_what_remains` | PASS |
| `a_pinned_fingerprint_that_matches_connects_and_one_that_differs_is_refused` | PASS |
| `a_pty_shell_prints_a_prompt_runs_a_command_and_reports_its_exit_status` | PASS |
| `a_reader_that_stops_at_a_cap_and_closes_the_channel_leaves_the_connection_usable` | PASS |
| `a_second_channel_works_while_the_first_is_drained_in_the_background` | PASS |
| `a_signature_from_another_key_is_refused` | PASS |
| `a_tool_that_cannot_be_started_is_an_error_that_names_it` | PASS |
| `a_tool_that_outlives_its_deadline_is_killed_and_named` | PASS |
| `a_two_hop_jump_runs_a_command_on_the_second_host` | PASS |
| `a_window_change_sent_before_the_shell_is_ready_is_not_lost` | PASS |
| `a_window_change_while_the_shell_runs_is_applied` | PASS |
| `accepting_the_key_connects_and_the_handler_saw_the_servers_fingerprint` | PASS |
| `an_ecdsa_p256_key_in_the_vault_logs_in` | PASS |
| `an_ed25519_key_in_the_vault_logs_in` | PASS |
| `an_rsa_3072_key_in_the_vault_logs_in_with_a_sha2_signature` | PASS |
| `an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1` | PASS |
| `authenticate_none_lists_the_methods_the_server_offers` | PASS |
| `blackhole_holds_the_bytes_until_forward_releases_them` | PASS |
| `both_ssh_key_versions_parse_the_same_public_keys_and_agree_on_the_fingerprints` | PASS |
| `closing_the_first_hop_ends_the_second` | PASS |
| `connect_log_in_and_exec_against_the_server_in_the_environment` | PASS |
| `cut_closes_open_connections_and_new_ones` | PASS |
| `exec_returns_stdout_stderr_and_the_exit_code` | PASS |
| `forward_passes_bytes_both_ways` | PASS |
| `keepalive_closes_a_silent_link_and_reports_keepalive_timeout` | PASS |
| `keyboard_interactive_is_refused_by_a_key_only_server` | PASS |
| `no_common_host_key_algorithm_names_both_lists` | PASS |
| `observe_a_window_change_sent_before_the_pty_request` | PASS |
| `observe_an_answer_slower_than_the_servers_login_grace_time` | PASS |
| `observe_an_unread_channel_next_to_a_working_one` | PASS |
| `observe_how_the_second_hop_learns_that_the_first_is_gone` | PASS |
| `observe_the_first_call_after_a_link_was_cut_during_the_host_key_prompt` | PASS |
| `observe_when_sshd_drops_a_connection_that_stays_quiet` | PASS |
| `only_the_answer_counts_not_the_echoed_command` | PASS |
| `output_far_larger_than_a_pipe_arrives_complete_and_does_not_deadlock` | PASS |
| `rejecting_the_key_fails_the_connect_with_unknown_key` | PASS |
| `stdin_is_closed_so_a_tool_that_reads_it_does_not_wait` | PASS |
| `the_algorithm_is_the_first_ssh_string_of_the_blob` | PASS |
| `the_answer_can_arrive_seconds_later_and_the_connection_goes_on` | PASS |
| `the_app_library_is_linked_into_the_same_binary` | PASS |
| `the_exit_code_and_both_streams_come_back_like_output_does` | PASS |
| `the_lock_file_holds_ssh_key_0_6_7_and_a_0_7_release_candidate` | PASS |
| `the_same_flow_against_a_scratch_sshd` | PASS |
| `the_second_hop_checks_its_own_host_key_and_the_first_hop_survives_a_refusal` | PASS |
| `the_size_given_with_the_pty_request_is_the_initial_size` | PASS |
| `the_system_ssh_client_logs_in_to_the_scratch_sshd_and_gets_the_exit_code` | PASS |
| `thirty_two_mebibytes_of_output_arrive_complete` | PASS |

### 所有 FACT
```
app.cargo_files_changed = 0
auth.none.remaining_methods = MethodSet([PublicKey])
auth.password.remaining_methods = MethodSet([PublicKey])
channels.unread_neighbour.answered_within_8s = false
channels.unread_neighbour.bytes_read_once_a_reader_started = 498892800
cut_during_exec.ended_after_ms = 0
cut_during_exec.reason = Error(IO(Custom { kind: UnexpectedEof, error: "early eof" }))
exec.capped_reader.next_exec_ms = 8
exec.killed_by_signal.exit_signal = Some("KILL")
exec.killed_by_signal.exit_status = None
exec.long_output.bytes = 33554432
exec.long_output.mib_per_second = 150.4
exec.long_output.seconds = 0.21
exec.timeout.waited_ms = 1001
handshake_stall.waited_ms = 2002
host_key.ask.algorithm = ssh-ed25519
host_key.cut_during_prompt.connect_failed = {}
host_key.cut_during_prompt.example.authenticate_keyboard_interactive_start.recv_error = Err(RecvError)
host_key.cut_during_prompt.example.authenticate_none.empty_failure = Ok(Failure { remaining_methods: MethodSet([]), partial_success: false })
host_key.cut_during_prompt.example.best_supported_rsa_hash.inconsistent = Err(Inconsistent)
host_key.cut_during_prompt.example.best_supported_rsa_hash.send_error = Err(SendError)
host_key.cut_during_prompt.first_call.authenticate_keyboard_interactive_start = {"recv_error": 17}
host_key.cut_during_prompt.first_call.authenticate_none = {"empty_failure": 17}
host_key.cut_during_prompt.first_call.best_supported_rsa_hash = {"inconsistent": 3, "send_error": 14}
host_key.cut_during_prompt.first_call.total = {"empty_failure": 17, "inconsistent": 3, "recv_error": 17, "send_error": 14}
host_key.cut_during_prompt.handle_future = {"Err(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 50, "Err(IO(Os { code: 54, kind: ConnectionReset, message: \"Connection reset by peer\" }))": 1}
host_key.cut_during_prompt.is_closed_after_the_first_call = {true: 51}
host_key.cut_during_prompt.recorded_reason = {"Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 50, "Error(IO(Os { code: 54, kind: ConnectionReset, message: \"Connection reset by peer\" }))": 1}
host_key.cut_during_prompt.rounds = 51
host_key.grace_enforcement.login_grace_1.seconds_until_dropped = 3.61 s | 3.05 s | 1.71 s | 1.40 s | 2.65 s
host_key.grace_enforcement.login_grace_3.seconds_until_dropped = 3.21 s | 4.98 s | 3.57 s
host_key.link_cut.authenticate_keyboard_interactive_start.first_call = Err(RecvError)
host_key.link_cut.authenticate_keyboard_interactive_start.recorded_reason = Some("Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))")
host_key.link_cut.authenticate_none.first_call = Ok(Failure { remaining_methods: MethodSet([]), partial_success: false })
host_key.link_cut.authenticate_none.recorded_reason = Some("Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))")
host_key.list_without_ssh_rsa = [Ed25519, Ecdsa { curve: NistP256 }, Ecdsa { curve: NistP384 }, Ecdsa { curve: NistP521 }, Rsa { hash: Some(Sha512) }, Rsa { hash: Some(Sha256) }]
host_key.no_common_algorithm.error = NoCommonAlgo { kind: Key, ours: ["rsa-sha2-256"], theirs: ["ssh-ed25519"] }
host_key.no_common_algorithm.kind = Key
host_key.no_common_algorithm.ssh_rsa_only_server.error = NoCommonAlgo { kind: Key, ours: ["ssh-ed25519", "ecdsa-sha2-nistp256", "ecdsa-sha2-nistp384", "ecdsa-sha2-nistp521", "rsa-sha2-512", "rsa-sha2-256"], theirs: ["ssh-rsa"] }
host_key.reject.error = UnknownKey
host_key.russh_default_key_list = [Ed25519, Ecdsa { curve: NistP256 }, Ecdsa { curve: NistP384 }, Ecdsa { curve: NistP521 }, Rsa { hash: Some(Sha512) }, Rsa { hash: Some(Sha256) }, Rsa { hash: None }]
host_key.slow_answer.after_ms = 6023
host_key.slow_answer.call_200ms_later = Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false })
host_key.slow_answer.disconnect_seen = None
host_key.slow_answer.first_call_after_connect = Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false })
host_key.slow_answer.handle_closed_right_after_connect = false
host_key.slow_answer.login_after = Success
host_key.slow_answer.outcome = connected
host_key.slow_answer_control_default_grace.after_ms = 6023
host_key.slow_answer_control_default_grace.call_200ms_later = Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false })
host_key.slow_answer_control_default_grace.disconnect_seen = None
host_key.slow_answer_control_default_grace.first_call_after_connect = Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false })
host_key.slow_answer_control_default_grace.handle_closed_right_after_connect = false
host_key.slow_answer_control_default_grace.login_after = Success
host_key.slow_answer_control_default_grace.outcome = connected
ipc.lib_suite = test result: ok. 1417 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 97.23s
ipc.tests_listed = 168 tests, 0 benchmarks
jump.close_first_hop.first_hop_reason = Error(Disconnect)
jump.close_first_hop.second_hop_end = Err(IO(Custom { kind: BrokenPipe, error: "channel closed" }))
jump.close_first_hop.second_hop_is_closed = true
jump.close_first_hop.second_hop_reason = None
jump.closed_port.after_ms = 0
jump.closed_port.error = ChannelOpenFailure(ConnectFailed)
jump.cut_first_hop.first_hop_reason = Some("Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))")
jump.cut_first_hop.second_hop_end = Err(IO(Custom { kind: BrokenPipe, error: "channel closed" }))
jump.cut_first_hop.second_hop_end_after_ms = 0
jump.cut_first_hop.second_hop_reason = None
jump.drop_both_handles.first_hop_reason = Some("Error(Disconnect)")
jump.drop_first_handle.exec_on_second_hop = Ok("second-works\n")
jump.drop_first_handle.first_hop_reason = None
jump.drop_first_handle.second_hop_closed = false
keepalive.interval_1s_max_2.noticed_after_ms = 3044
keepalive.interval_1s_max_2.reason = Error(KeepaliveTimeout)
lock.app_versions_replaced = 1
lock.dropped_from_the_app_lock = 10
lock.new_crates = 35
lock.prerelease_crates = 5
lock.second_versions_of_crates_the_app_has = 58
shell.first_prompt_ms = 163
shell.window_change_before_pty_request.final_size = 24 80
shell.window_change_before_pty_request.was_kept = false
signer.rsa.algorithm_on_the_wire = rsa-sha2-512
signer.rsa.best_supported_rsa_hash_raw = Some(Some(Sha512))
signer.rsa.hash_alg_offered_by_russh = Some(Sha512)
signer.sha1_only.algorithm_on_the_wire = ssh-rsa
signer.sha1_only.best_supported_rsa_hash_raw = Some(None)
signer.sha1_only.hash_alg_offered_by_russh = None
windows.echo_stdout = "spike-windows-ok\n"   # macOS 預演(臨時 sshd)的值,不是 Windows 的事實;Windows 的值在「Windows」一節
windows.host_key = SHA256:v7RhQgJwm1aWgkYQEc44ltdOhqdKXbUstrOWzsM6tlk   # macOS 預演(臨時 sshd)的值,不是 Windows 的事實;Windows 的值在「Windows」一節
```

### 第二欄:同一份測試用 `--test-threads=1` 整次重跑

指令與主 log 相同,只多了 `--test-threads=1`:`cargo test --offline -- --include-ignored --nocapture --test-threads=1`(在 `spike/russh/`),149 秒(主 log 那次 81 秒,含重新編譯)。結果:13 行 `test result` 全是 `ok`,0 failed,各 binary 的通過與 ignored 個數與主 log 完全相同。**主 log 沒有被取代**:上面所有表格的值都來自主 log;下面只列兩次的 FACT 不同的地方(其餘 64 個 FACT 完全相同;`windows.host_key` 每次執行都不同,略過)。計時差異是機器負載與平行度造成的;第 14 項那幾個 `slow_answer` 欄位是競態的兩種結果,不是不穩定的測試。

| FACT | 主 log(平行) | 第二欄(`--test-threads=1`) |
|---|---|---|
| `channels.unread_neighbour.bytes_read_once_a_reader_started` | 498892800 | 506658816 |
| `exec.long_output.mib_per_second` | 150.4 | 150.9 |
| `exec.timeout.waited_ms` | 1001 | 1002 |
| `handshake_stall.waited_ms` | 2002 | 2001 |
| `host_key.cut_during_prompt.handle_future` | {"Err(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 50, "Err(IO(Os { code: 54, kind: ConnectionReset, message: \"Connection reset by pe… | {"Err(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 51} |
| `host_key.cut_during_prompt.recorded_reason` | {"Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 50, "Error(IO(Os { code: 54, kind: ConnectionReset, message: \"Connection reset b… | {"Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))": 51} |
| `host_key.grace_enforcement.login_grace_1.seconds_until_dropped` | 3.61 s, 3.05 s, 1.71 s, 1.40 s, 2.65 s | 4.72 s, 3.14 s, 3.81 s, 4.84 s, 3.84 s |
| `host_key.grace_enforcement.login_grace_3.seconds_until_dropped` | 3.21 s, 4.98 s, 3.57 s | 6.64 s, 5.81 s, 5.46 s |
| `host_key.slow_answer.after_ms` | 6023 | 6024 |
| `host_key.slow_answer.call_200ms_later` | Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false }) | Err(SendError) |
| `host_key.slow_answer.disconnect_seen` | None | Some("Error(IO(Custom { kind: UnexpectedEof, error: \"early eof\" }))") |
| `host_key.slow_answer.first_call_after_connect` | Ok(Failure { remaining_methods: MethodSet([PublicKey]), partial_success: false }) | Ok(Failure { remaining_methods: MethodSet([]), partial_success: false }) |
| `host_key.slow_answer.login_after` | Success | failed: SendError |
| `host_key.slow_answer_control_default_grace.after_ms` | 6023 | 6024 |
| `keepalive.interval_1s_max_2.noticed_after_ms` | 3044 | 3040 |
| `shell.first_prompt_ms` | 163 | 105 |

## Windows

### `spike windows`:russh 連 Windows OpenSSH server、登入、exec

- 執行:https://github.com/ysya/sshelter/actions/runs/38054666697(workflow `spike windows`,job `russh`,事件 `push`,分支 `next/own-ssh`,commit d6d9017,2026-10-10T13:09:51Z 開始);結論:**success**,第一次就過,沒有改過 PowerShell。GitHub 的 run metadata(`gh run view`)與 `.superpowers/sdd/2026-10-10-own-ssh-phase0-spike/task-4-windows-run.md` 的記錄一致。
- 步驟都是 success:checkout → toolchain → rust-cache → 安裝並啟動 OpenSSH server → 為 runner 使用者授權一把拋棄式金鑰 → OpenSSH 用戶端登入(`client-ok`)→ 用 russh 連線並 exec;最後一步「印出 OpenSSH server 的 log」只在失敗時才跑,所以略過。這些 log 行與步驟已和 `gh run view 38054666697 --log` 逐行核對過,一致。
- 試驗 crate 的程式庫(harness、fixture、client、proxy、shell、auth)第一次在 `windows-latest` 上以 `--no-default-features --locked` 編譯,沒有警告。這個 build 不含 app 函式庫(沒有 Tauri)。
- 這次執行的 log,逐字(取自 `task-4-windows-run.md`):

```
client-ok
FACT windows.host_key = SHA256:W1Rh/k/tWxgJFSwvzf/E3mzaJvCIXpNtUl/e1bYMRCo
FACT windows.echo_stdout = "spike-windows-ok\r\n"
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.19s
```

- 測試做了什麼(`spike/russh/tests/windows_exec.rs`):connect、公鑰登入、`echo`(輸出 `contains` 檢查,結束碼 `Some(0)`)、`exit 3`(結束碼 `Some(3)`);這些斷言在測試裡,沒有印出來。`1 passed` 是預期的:macOS 預演的測試用 `#[cfg(not(windows))]` 擋掉,Windows 的測試檔只剩環境驅動的那一個。
- **發現:Windows OpenSSH 的 exec 輸出以 CRLF 結尾**(`"spike-windows-ok\r\n"`),macOS 預演是 `"spike-windows-ok\n"`。試驗的斷言用 `contains`,所以過了;引擎不能假設 `\n`。
- 沒量到的:主機金鑰的演算法(這個測試只印指紋);Windows 上的 PTY/shell。
- 本機 log 裡的 `windows.host_key` 與 `windows.echo_stdout` 是 macOS 預演(臨時 sshd)的值,不是 Windows 的事實;Windows 的值是上面這兩行。

### `test-windows.yml`:搬移 `ipc/` 之後的 Windows 測試

- 搬移前的基準:https://github.com/ysya/sshelter/actions/runs/38056602396(workflow `windows key slots`,job `slots`,事件 `workflow_dispatch`,分支 `next/own-ssh`);結論:success;`test result: ok. 216 passed; 0 failed; 0 ignored; 0 measured; 1139 filtered out; finished in 25.65s`。
- **這一次跑的不是搬移後的程式。** GitHub 回報這次執行的 commit 是 d6d9017(`headSha`),也就是當時遠端 `next/own-ssh` 的最前面;307307a(`refactor(ipc)`)當時只在本機,還沒 push。log 裡執行的指令是舊的過濾 `cargo test --lib -- sync::slot_rules sync::slot_files vault:: agent::`(沒有 `ipc::`),d6d9017 裡也還沒有 `src-tauri/src/ipc/`,`server.rs`、`peer.rs`、`pipe_windows.rs` 還在 `agent/` 下。(查證:`gh run view 38056602396 --json headSha,...`、`gh run view 38056602396 --log` 的 `Run cargo test` 那一行、`git ls-tree d6d9017 -- src-tauri/src/ipc`。)
- 所以它證明的是:搬移之前的 `agent::`(含現在在 `ipc/` 的那些測試)、`vault::`、`sync::slot_rules`、`sync::slot_files` 在 `windows-latest` 上全過——搬移前的基準。它不證明搬移後的程式。
- 搬移後的實際執行:https://github.com/ysya/sshelter/actions/runs/38059199511(workflow `windows key slots`,job `slots`,事件 `workflow_dispatch`,分支 `next/own-ssh`,headSha **8e68ade**〔含 307307a 的搬移〕,2026-10-10T14:20:14Z 建立);結論:**success**。步驟「Key slot, vault and agent tests」執行的是新的過濾 `cargo test --lib -- sync::slot_rules sync::slot_files vault:: ipc:: agent::`,結果(逐字取自該次執行的 log):`test result: ok. 217 passed; 0 failed; 0 ignored; 0 measured; 1139 filtered out; finished in 12.21s`。
- 217 = 搬移前基準的 216 + `ipc` 的守門測試。核對方式:把兩次 log 的測試名稱比對,基準裡的 `agent::{server,peer,pipe_windows}::` 對應成 `ipc::…` 之後,新的一次只多 `ipc::tests::the_ipc_module_does_not_reach_into_the_agent` 一個,基準的測試沒有任何一個消失;217 個全是 ok。其中 28 個是 `ipc::` 測試(`ipc::peer` 22、`ipc::pipe_windows` 5〔Windows 才有的具名管道測試〕、守門測試 1),另有 121 個 `agent::` 測試全過。所以搬移後的 `ipc::` 測試與 `agent/` 裡 `cfg(windows)` 的幾行,在 `windows-latest` 上編得過、相關測試全過。
