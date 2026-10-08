# SSHelter 金鑰保管庫與 SSH agent(金鑰路線第 2 階段)設計

- 日期:2026-10-07
- 狀態:待使用者審閱(2026-10-07 修訂:金鑰一律在 SSHelter、Keychain 的排版、計畫 2 切成 2a 與 2b)
- 前置:SP3 金鑰插槽(`docs/superpowers/specs/2026-10-05-sp3-key-slots-design.md`,已在 beta 0.17.0-5)
- 調查:`docs/superpowers/specs/2026-10-07-key-vault-research.md`(金鑰管理與 agent 核准)、
  `docs/superpowers/specs/2026-10-07-key-approval-scope-research.md`(核准設定放在哪裡)

## 0. 定位

這是 SP3 spec §14 的第 2 階段:把金鑰管理獨立出來,做成像 Termius 的 Keychain。和 §14 原本的規劃相比,有三處依使用者決定(2026-10-06)改變:

- **AI 只走 agent**:AI 工具用自己的 `ssh`,經 SSHelter 的 agent 取用金鑰,由核准視窗把關。MCP 的 `run` 移除,MCP 只留唯讀工具(§9)。
- **只接給用到保管庫金鑰的主機**:不在每台電腦把所有主機都指到 agent,只有用到「只在 SSHelter」金鑰的主機才設 `IdentityAgent`(§6)。
- **介面參考 Termius**:側邊欄的 Keychain、金鑰物件、Export to host、主機連結金鑰、不把所有金鑰試一遍(§7)。
- **金鑰一律在 SSHelter**(2026-10-07):SSHelter 管的私鑰只放在保管庫、只經 agent 提供;要檔案就匯出(Export private key),
  匯出之後那個檔案不再由 SSHelter 管。拿掉每台電腦「另存成檔案」的選項,概念從三種放法變成一種。

和 Termius 的根本差異:Termius 有自己的 SSH 連線程式,金鑰只在 Termius 裡用;SSHelter 用系統的 `ssh`,所以由 SSHelter 自己的 agent 提供金鑰。內建終端機仍是第 3 階段。

## 1. 目標與非目標

**目標**
1. 金鑰是 SSHelter 管理的物件(Keychain),主機連結金鑰,不再是 `~/.ssh` 裡的一個路徑。
2. SSHelter 管的私鑰只在 SSHelter 的保管庫。終端機、AI 工具、`git` 的 `ssh` 經 SSHelter 的 agent 使用,每次都經過核准規則。
3. AI 工具不經使用者核准,用不到保管庫裡的金鑰。這補上 README「AI Access (MCP)」寫的限制:其他程式可以直接執行 `ssh`。
4. 不破壞 SP3:`keyslot`、`key` 的格式不變;新增的一種記錄,舊版原樣保存(§4.2)。
5. 每把金鑰要不要同步、保護到什麼程度由使用者決定;需要檔案就匯出一份。

**非目標(這一階段不做)**
- Identity(使用者名稱加金鑰的組合)、憑證(certificate)、FIDO2、Secure Enclave/TPM 金鑰。
- 內建終端機與 SSH 連線程式(第 3 階段)。
- Linux 的系統驗證(Linux 一律點選核准)。
- 把保管庫的金鑰轉送(agent forwarding)到遠端主機。
- 不讀 `~/.ssh/config` 的程式(例如部分 IDE 內建的 ssh、PuTTY)使用保管庫的金鑰。
- 動到伺服器(SP3 的原則不變)。

## 2. 已定案的決策

| # | 決策 | 來源 |
|---|---|---|
| 1 | 私鑰放在保管庫,由 agent 提供(「每台可另存成檔案」已由 #11 取代) | 使用者,2026-10-06 |
| 2 | 核准的單位是「金鑰 × 主機 × 發出請求的程式」,問一次後記住一段時間(預設 4 小時),螢幕鎖定或 SSHelter 結束就清除;每把金鑰可設成每次都問 | 使用者,2026-10-06 |
| 3 | 核准預設點選;每把金鑰可要求 Touch ID/Windows Hello;Linux 點選 | 使用者,2026-10-06 |
| 4 | passphrase 保留原狀,第一次使用時問,可記在這台的系統 Keychain,永不同步 | 使用者,2026-10-06 |
| 5 | 把既有的金鑰檔加入保管庫時,每次問:搬進來(移除原檔)或同時保留檔案 | 使用者,2026-10-06 |
| 6 | Keychain 放在側邊欄,和 Hosts 並列;排法同 Hosts(清單 + 明細),取 Termius 的原則 | 使用者,2026-10-06、10-07 |
| 7 | agent 只接給用到「只在 SSHelter」金鑰的主機 | 使用者,2026-10-06 |
| 8 | AI 存取只走 agent;移除 MCP `run`;MCP 保留唯讀工具 | 使用者,2026-10-06 |
| 9 | 每把金鑰的保護跟著金鑰同步(KeePassXC 的做法);每台電腦只能再加嚴;記住的核准與預設時長每台各自 | 使用者依調查結果同意,2026-10-06 |
| 10 | `src-tauri/Cargo.lock` 從此和套件變動一起 commit | 使用者,2026-10-07 |
| 11 | 金鑰一律在 SSHelter:同步的金鑰到每台都進保管庫;拿掉「Keep a file」;要檔案用 Export private key(匯出後不再管);`~/.ssh` 的金鑰檔只提供匯入 | 使用者,2026-10-07 |
| 12 | 已經在用的檔案(SP3 留下的)更新後不自動搬,Keychain 提示,按一次全部搬進保管庫;你自己的原始檔不刪 | 使用者,2026-10-07 |
| 13 | 計畫 2 切成 2a(Keychain 頁與金鑰一律在 SSHelter)與 2b(把其他金鑰帶進來) | 使用者,2026-10-07 |

## 3. 安全模型

| 威脅 | 能做到的 | 做不到的 |
|---|---|---|
| AI 工具或其他程式自己執行 `ssh` | 要用保管庫的金鑰必須經過核准;視窗顯示是哪個程式在要、要連哪台 | 核准之後,那段期間它在那台主機上能做任何事,SSHelter 看不到指令。敏感的金鑰要設「每次都問」或要求 Touch ID |
| 同一使用者身分的惡意程式 | 仍要核准;要求 Touch ID/Windows Hello 的金鑰需要本人在場 | 可以連上 agent、冒用程式名稱(程式辨識是推測,§5.4),也可以在記住的期間內借用同一個「金鑰 × 主機 × 程式」的核准;在 Windows、Linux 上可能讀取 SSHelter 的記憶體 |
| 電腦被偷(沒有登入) | 保管庫檔是加密的,解鎖金鑰在系統 Keychain,受登入保護 | — |
| 匯出的檔案、你自己在 `~/.ssh` 的金鑰檔 | 匯出時清楚標示「任何程式都能使用,SSHelter 不再管它」 | 任何程式不經核准就能用 |
| 遠端主機(agent forwarding) | 轉送過來的請求一律拒絕(session-bind 標示為轉送) | — |
| 同步 | 沿用 SP3:端對端加密;passphrase 永不同步 | 沿用 SP3 §3 |

## 4. 資料模型

### 4.1 保管庫(本機)

- 位置:`fsutil::app_data_root()`(和 `sync-state.json` 同一處)的 `vault.json`。寫入一律原子(暫存檔 → 設好權限 → rename),
  檔案只有擁有者能讀寫(Unix 0600;Windows 只給目前使用者的 DACL,同 SP3 的插槽檔)。
- 加密:每一筆以 XChaCha20-Poly1305 個別加密,AAD 是 `sshelter-vault-v1` 加上插槽 id;金鑰是 32 bytes 的隨機值,
  存在系統 Keychain(service `SSHelter`)裡這個保管庫自己的 account `vault:key:<16 個 hex>`,名稱記在檔頭的 `key_account`(格式第 2 版)。
  新的金鑰一律放在新的 account,不寫到已經有名字的 account 上:系統 Keychain 誤報「沒有這筆」時,搬到旁邊的保管庫檔仍解得開。
  計畫 1 寫的保管庫檔沿用 account `vault:key`(檔頭沒有 `key_account`,格式第 1 版),舊版照常開啟。
  沒有系統 Keychain 可用時,保管庫停用,同步的金鑰暫時寫成檔案(§11)。
- 一筆的內容:私鑰原文(OpenSSH 格式,有 passphrase 的仍是加密狀態)、指紋、公鑰、來源(產生、匯入、同步)、加入時間。
- 檔頭(明文、不含祕密):格式版本、金鑰所在的 account 名稱(`key_account`),以及這台電腦的 agent 設定(記住多久、這台一律每次都問)。
- 讀不懂的保管庫檔:不覆寫,搬到旁邊保留(同 Sync v2 對讀不懂的狀態檔的處理),保管庫進入錯誤狀態(§11)。

### 4.2 帳戶 chain

- `keyslot`、`key` 不變(SP3 §4.1):同步的金鑰照舊經 `key` 記錄傳遞,到了這台再放進保管庫。
- **新增 `keyprefs`**(id = 插槽 id):`{schema, label, ask_every_time, require_user_presence}`,和其他記錄一樣 LWW。
  - `label`:顯示名稱;空字串就用插槽名稱。改名只改它,插槽檔名不變,不必改寫任何主機。
  - `ask_every_time`、`require_user_presence`:這把金鑰的保護,跟著金鑰到每台電腦。
- 舊版(0.17.0-5 以前)遇到不認得的種類,會把密文原樣保存,不解密、不刪除、不改寫(`merge.rs` 的 `merge_account`)。
  所以舊版的電腦不會弄丟它。
- 換碼:新版的換碼一併複製 `keyprefs`。舊版的換碼不會複製它;新版發現帳戶裡少了,就補寫回去。補寫的規則同 SP3 §6.6:
  只補在這個帳戶學到的插槽(`LocalSlot::learned_in`)。

### 4.3 本機狀態

- `LocalSlot`(`sync-state.json` 的 `key_slots`)的來源多一種:`SlotSource::Vault { fingerprint }`,表示私鑰在保管庫裡,
  插槽目錄只放 `.pub`。新版的插槽正常都是 `Vault`:
  - 同步的金鑰到了這台,直接放進保管庫(不再寫成 `SyncedCopy`)。
  - `own` 插槽在這台挑金鑰(或換一把)時,把那把金鑰複製進保管庫;你原本的檔案原封不動,之後由你自己管。
  - 在保管庫的插槽上換金鑰(挑別把、改用同步的那把):新的取代;舊的在保管庫裡改存成 retired 項目,不自動刪除
    (同 SP3「私鑰不自動刪除」;2b 的 Keychain 列出它們,可匯出或刪除)。
  - SP3 的 `Linked`、`SyncedCopy` 只出現在三種情況:更新前留下的(§8)、保管庫不能用時同步金鑰的暫時落地(§11)、舊版的電腦。
- 沒加入同步帳戶也能用:保管庫的金鑰在 `key_slots` 裡有一筆本機記錄(不在任何帳戶)。之後同步的主機用到它時,
  沿用 SP3 的 Sync key 對話框把它收編進帳戶(`slot_setup::adopt_slot`)。
- 記在這台的 passphrase:系統 Keychain(account `vault:passphrase:<插槽 id>`),永不同步。
- 這台電腦只能更嚴的設定(保管庫檔頭):「這台一律每次都問」、記住多久。
- 記住的核准只在記憶體(§5.3)。

### 4.4 本機檔案

- 插槽目錄 `~/.ssh/sshelter/keys/`:只放 `<檔名>.pub`,沒有 `<檔名>` 本身(更新前留下的與保管庫不能用時的暫時檔案例外,§4.3)。
  實測(§15):`IdentityFile` 指到 `<檔名>`、只有 `<檔名>.pub` 時,`ssh` 會讀 `.pub` 並向 agent 要對應的私鑰。
- agent 的目錄 `~/.ssh/sshelter/agent/`(0700):產生的設定檔 `config`(§6)、socket `sock`、Connect 的一次性通道 `run/`。
  不用 `~/.ssh/sshelter/<名稱>.config`:同步把那種路徑當成 space 檔(`hosts_file::is_our_include_token`),
  每一輪會改寫它的 Include,離開帳戶時還會把它搬到 `sshelter-local`;子目錄裡的檔案不受影響。
- agent 的位置:
  - macOS、Linux:`~/.ssh/sshelter/agent/sock`(目錄 0700、socket 0600)。Unix socket 的路徑有長度上限(macOS 104 字元),
    啟動時檢查,太長就進入錯誤狀態。
  - Windows:`\\.\pipe\sshelter-agent-<使用者 SID 的雜湊>`;設定檔裡寫成 `//./pipe/sshelter-agent-<…>`(Win32-OpenSSH 8.9 起,
    反斜線的寫法會失敗)。
- Connect 用的一次性通道(§5.6):`~/.ssh/sshelter/agent/run/<隨機>`;Windows `\\.\pipe\sshelter-connect-<隨機>`。

## 5. agent

### 5.1 端點與存取

- agent 跑在 SSHelter 的視窗程式裡,隨程式啟動。同一個使用者只有一個:已經有 SSHelter 在提供,就不再開。
- 沒有視窗的 MCP stdio 轉接(`--mcp`)不開 agent:它不建立 Tauri,核准需要視窗。`--mcp-host` 是完整的視窗程式(核准中心),照常開 agent(同一個使用者只會有一個在提供)。
- Unix:接受連線後檢查對方的 UID 等於自己(`getpeereid`)。
- Windows:pipe 的 DACL 只給目前使用者;設 `PIPE_REJECT_REMOTE_CLIENTS`;用 `FILE_FLAG_FIRST_PIPE_INSTANCE` 防止名稱被搶先佔用。

### 5.2 協定

依 RFC 9987(2026-05,Standards Track)與 OpenSSH 的 PROTOCOL.agent:

- 支援:列出金鑰(只列這台「只在 SSHelter」的金鑰)、簽章:Ed25519、ECDSA(P-256/384/521)、RSA(依請求的 flag 用 SHA-256/512;
  沒帶 flag 的 SHA-1 照請求簽,同 OpenSSH 的 agent)。`ssh-key` 0.6.7 的 RSA 簽章有錯(組私鑰時把 `p` 傳了兩次),RSA 改用 `rsa`
  套件從 n、e、d、p、q 組私鑰再簽(§15)。協定自己實作,不用 `ssh-agent-lib`(§15)。訊息上限 256 KiB(同 OpenSSH)。
- 擴充 `session-bind@openssh.com`:驗證伺服器對 session id 的簽章,記下這條連線的主機金鑰,以及是否為轉送;簽章驗證失敗回 28
  (EXTENSION_FAILURE)。連線上只要有一次轉送的 bind,這條連線的簽章請求一律拒絕。被簽的 userauth 資料裡的 session id 要等於最後一次
  bind 的 session id,hostbound 方法帶的主機金鑰也要相同,否則主機算未知。
- 其他擴充回 failure(RFC 的要求)。
- 加入、移除金鑰,以及 lock、unlock:一律 failure。金鑰只能從 Keychain 進來。

### 5.3 核准

收到簽章請求時,先確定:

- 金鑰。
- 主機:session-bind 的主機金鑰;對照 `known_hosts` 找出主機名稱來顯示。沒有 session-bind 就是「未知的主機」。
- 使用者名稱:從被簽的 userauth 資料取出。
- 程式:§5.4。

規則:

1. 這把金鑰設了「每次都問」,或這台設了「一律每次都問」,或主機未知:問,而且不記住。
2. 否則,記憶體裡有沒過期的「金鑰指紋 × 主機金鑰指紋 × 程式」核准:直接簽。
3. 否則:問。允許時預設記住;記住多久依這台的設定,選項是 15 分鐘、1 小時、4 小時(預設)、12 小時。

其他:

- 要求 Touch ID/Windows Hello 的金鑰:按下允許後再做系統驗證;失敗或取消就拒絕。這台電腦沒有可用的系統驗證(例如沒設定
  Windows Hello、或是 Linux)時退回點選,視窗註明原因。
- 視窗 60 秒沒有回應就拒絕(伺服器預設 120 秒內沒完成認證就斷線)。
- 同時多個請求:排隊,一次一個視窗;相同「金鑰 × 主機 × 程式」的請求共用同一個答案。
- 螢幕鎖定、登出、SSHelter 結束:清除全部記住的核准,以及記憶體裡解開的私鑰。

### 5.4 程式辨識

從連上 agent 的 `ssh` 往上找父程序(macOS 用 libc 的 `proc_pidinfo`、`proc_pidpath`,不用 `sysinfo`:它的執行檔路徑可能是相對路徑或
symlink 名稱,§15)。「程式」= 第一個不是 `ssh`、`ssh-keygen`、shell、login、env、sudo 的程式;「App」= 程序鏈裡最外層的應用程式
(macOS 只認 `<X>.app/Contents/MacOS/<執行檔>` 這種 bundle 主程式,不然 `/usr/bin/git` 實際執行的 Xcode 裡的 git 會顯示成 Xcode)。
核准視窗的標題用 App,名稱鏈完整列出。

- macOS:`LOCAL_PEERPID`,再用 `proc_pidpath` 與父程序資訊往上找。
- Linux:`SO_PEERCRED`,再讀 `/proc/<pid>/exe` 與 `/proc/<pid>/stat`。
- Windows:`GetNamedPipeClientProcessId`、`QueryFullProcessImageNameW`,父程序用 `CreateToolhelp32Snapshot`。

識別值是「App 的執行檔路徑 + 程式的執行檔路徑」;程式是直譯器(例如 `node`、`python`)時再加上它執行的腳本路徑(從程序的命令列取得)。
加上 App,是因為 Claude Code 跑的 `git` 和你在終端機跑的 `git` 是同一個執行檔,只看程式會共用記住的核准。完整的命令列不保存、也不顯示。
認不出程式時(程序已經結束)顯示「an unknown program」,而且不記住。顯示成名稱鏈(例如「claude → zsh → ssh」)。
這只是推測:同一使用者的程式可以偽造,所以只用來分組記住核准和顯示給使用者,不當成安全保證。

### 5.5 passphrase

- 有 passphrase 的金鑰,第一次需要時,核准視窗多一個 passphrase 欄,可勾「Remember on this computer」(存進系統 Keychain)。
- 沒記住:解開的私鑰只在記憶體,保留到這把金鑰的記住期間結束(或螢幕鎖定、SSHelter 結束),之後清除。
- 記住了:每次簽章時才取出 passphrase 解開,簽完立即清除。
- 輸錯三次,拒絕這次請求。
- 只支援 OpenSSH 格式的加密私鑰。有 passphrase 的舊式 PEM 無法加入保管庫,沿用 SP3 的說明(用 `ssh-keygen -p` 轉換)。

### 5.6 從 SSHelter 連線(Connect)

這台主機用的金鑰「只在 SSHelter」時:

1. 這把金鑰要求系統驗證:先驗證,失敗就不連。
2. 有 passphrase、這台沒記住:先在 SSHelter 問。
3. 開一個一次性通道(§4.4),在終端機執行 `ssh -o IdentityAgent=<通道> -o ForwardAgent=no <主機>`(沿用 `connect.rs` 的參數組法與各終端機)。
4. 通道只接受同一使用者的第一個連線,60 秒逾時,只提供這台主機的金鑰,認證完成就關閉。視為已核准:不跳視窗,也不算進記住的核准。
   60 秒之後才連上來的 ssh 拿不到金鑰:通道留到開啟後 10 分鐘,只為了讓畫面請使用者再按一次 Connect。`ssh` 只在第一次試公鑰認證時才連
   agent(重用 ControlMaster 的連線、先用密碼登入就不會連),所以一直沒人連就安靜關掉,不說什麼。

系統匣的快速連線同上。以密碼登入、由 SSHelter 自動填密碼的連線(`connect.rs`)不受影響。

### 5.7 生命週期

- SSHelter 在執行時,agent 才在;SSHelter 沒開時,用保管庫金鑰的主機連不上,`ssh` 顯示 `no such identity`(§15 實測)。
- 第一把金鑰進保管庫時(同步進來、搬遷、在這台挑金鑰),SSHelter 建議開啟「開機自動啟動(只在系統匣,不開視窗)」與「關閉視窗時保持執行」;
  兩個都還沒開時,Keychain 上方保留這個提示,直到開啟或按掉。金鑰一律在 SSHelter 之後,SSHelter 沒開就等於這些主機都連不上。
  目前開機啟動會開視窗;要新增「隱藏啟動」的參數。

## 6. 接到 `ssh` 的設定

- SSHelter 產生 `~/.ssh/sshelter/agent/config`,內容只有這台用到保管庫金鑰的主機。原本每個 Host 區塊的 pattern
  照抄成一個 `Host` 行:

  ```text
  # Managed by SSHelter. Changes here are overwritten.
  Host web
    IdentityAgent ~/.ssh/sshelter/agent/sock
    IdentitiesOnly yes
  Host *.lab !bastion.lab
    IdentityAgent ~/.ssh/sshelter/agent/sock
    IdentitiesOnly yes
  ```

- `~/.ssh/config` 的第一行,加一行 SSHelter 管理的 `Include ~/.ssh/sshelter/agent/config`(寫入前照現有的存檔規則備份)。
  ssh_config 取第一個符合的值:任何 Host 區塊之前的全域設定等於套用到所有主機,space 檔的 Include 也可能排在前面,
  所以這一行必須在所有內容之前,這兩個設定才會以這裡為準。同步維護自己的 Include 時跳過這一行,排在它後面(`hosts_file::sync_include_index`)。
- SSHelter 讀取 config 時略過這個產生的檔案(它不是使用者的設定):不然主機清單會重複列出這些主機,主機頁也可能打開到產生的那份。
- lint:只在 SSHelter 的插槽(只有 `.pub`)不算「IdentityFile not found」。
- `IdentitiesOnly yes`:只提供這台主機自己的那把金鑰,不把 agent 裡的金鑰全試一遍(Termius 也不這麼做),也避免撞到伺服器的嘗試次數上限。
- 何時重寫:主機的 `IdentityFile` 變了、金鑰的提供方式變了、插槽改名、space 檔同步進來(主機增減)。
- 使用者刪掉 Include 那一行:不自動加回;Keychain 顯示「Hosts that use keys in SSHelter can't reach its agent」與「Fix」。
- `Match` 區塊裡的 `IdentityFile`:不支援,列在「Can't set up automatically」。
- 不在 SSHelter 設定裡的主機(例如 github.com):Keychain 的「Add a host for this key…」建立一筆主機
  (`Host github.com`、`User git`、`IdentityFile <插槽>`),之後自動列進 `agent/config`。
- `IdentityAgent` 在同步的 space 檔裡仍是需要核准的指令(SP1);`agent/config` 是本機檔,不同步。

## 7. Keychain 介面(參考 Termius)

### 7.1 位置

- 側邊欄最上方加「Hosts｜Keychain」切換,記住上次選的(和側邊欄寬度一樣存起來,重開也記得)。選 Keychain 時,側邊欄是金鑰清單,主區塊是選到的金鑰的明細,
  和 Hosts 的「主機清單 + 主機編輯器」同一種排法(使用者 2026-10-07 選定)。Termius 的導覽列 + 卡片格狀 + 右側面板在工具視窗的寬度下會擠,
  卡片也放不下「金鑰放在哪、誰能用」的狀態;Termius 的金鑰只在 Termius 裡用,SSHelter 的金鑰給系統的 `ssh`、git、AI 工具用,這些狀態是必要的。
- 取 Termius 的原則,不照抄排版:金鑰是集中在一處的獨立物件,而且只在 SSHelter 裡(要檔案就匯出,§7.3.2);主機連結金鑰;
  Export to host 一步完成(裝公鑰並連結,§7.3.1);在原地新增與產生,不疊對話框。
- 工具列的鑰匙按鈕切到 Keychain,不再開對話框。舊的 Keys 對話框與其中的「Keys used by synced hosts」區塊移除,內容併進 Keychain。
  SP3 的兩個流程對話框保留(主機開始同步時問金鑰要不要同步、「Pick keys for this computer」),它們的挑金鑰與打開都帶到 Keychain 裡對應的金鑰;
  「Pick keys for this computer」的說明改成「…or do it later in Keychain」。Settings → Sync 的「Pick…」關掉設定、切到 Keychain,選好第一把需要挑金鑰的。

### 7.2 清單

上方依序(有才顯示):SSHelter 的 agent 的問題與「Fix」(§6、§11);搬遷提示「{N} keys can move into SSHelter」與「Move」(§8);
開機自動啟動的提示(§5.7);Windows 的 Git 提示(§10)。再來是搜尋(名稱、類型、指紋)。

1. **In SSHelter**:所有插槽(SP3 的 `synced` 與 `own`;金鑰在保管庫,或標「File for now」還沒搬)。一個清單,依名稱排序,需要處理的排最前面
   (這台還沒有金鑰、錯誤)。列上:名稱、類型、標記:
   - 「Synced」(`synced` 插槽)或「Own key on each computer」(`own` 插槽)。
   - 需要處理的:「Needs a key」(這台還沒有)、錯誤、「Not in your sync account」(帳戶裡已經沒有、這台還留著,只能刪除沒有主機用到的副本)、
     「File for now」(更新前留下的檔案,或保管庫不能用時的暫時檔案,§4.3)。
   - 計畫 2b 再加:「This computer only」(不在帳戶的本機金鑰)、要求 Touch ID/Windows Hello、每次都問、需要 passphrase。
2. **Other key files in ~/.ssh**(收合,在最下面):不在任何插槽裡的金鑰檔。它們照舊能用,SSHelter 不管;載入系統 ssh-agent 的標
   「in ssh-agent」。標題旁有「Generate a key file…」(沿用現有的產生功能;2b 改成直接產生進保管庫之後移除)。計畫 2b 再加「Import」。

系統 ssh-agent 的「agent: N keys」那行拿掉。

### 7.3 明細

- 每一種都有:名稱、類型、指紋、「Copy public key」、「Export to host…」(§7.3.1)、用到它的主機(點一下切到 Hosts 並選到那台)。
- SSHelter 的金鑰再加「Export private key…」(§7.3.2)。
- 同步的金鑰(`synced`):
  - 其他電腦:各台在 SSHelter、用檔案(舊版或保管庫不能用),或還沒拿到。
  - SP3 的狀態訊息與動作(Use the synced key、Sync the new key、Stop syncing、Delete copy…;SP3 §6.3、§7.2)。在保管庫的插槽上
    也能直接做(§4.3),不再要求先改回檔案。
- 每台各自的金鑰(`own`):這台用的是哪把(在保管庫裡,從哪個檔案複製來的)、「Pick a key on this computer」/「Change…」、
  「Sync to your computers」。
- 其他金鑰檔:路徑、是否在 ssh-agent 裡。不提供刪除(§7.6:使用者自己的原始檔一律不碰)。
- 計畫 2b 再加:
  - 改名(只改 `label`)、建立時間、有沒有 passphrase。
  - 保護(跟著金鑰同步):「Ask every time」、「Require Touch ID」/「Require Windows Hello」。
  - passphrase:「Remembered on this computer」與「Forget」。
  - Add a host for this key…(§6)、Delete(§7.6)。

#### 7.3.1 Export to host…

Termius 的「Export and Attach」:

1. 選一台主機(只列真的主機,不列 `Host *` 之類的萬用區塊)。
2. 沿用現有的 Deploy 流程(App 內或終端機),把公鑰裝到那台主機的 `authorized_keys`。
3. 成功之後,把那台主機的 `IdentityFile` 換成這把金鑰:取代主機區塊裡原本的 `IdentityFile` 行,不動 `Host *` 等繼承來的設定。
   確認畫面寫明「{host} will use {key} instead of {old keys}」(原本沒有 `IdentityFile` 時只寫「{host} will use {key}」)。
   值寫插槽路徑(SSHelter 的金鑰)或 `~/` 開頭的檔案路徑(其他金鑰檔)。
4. 主機在同步的 space 裡:改完之後照 SP3 的規則處理這把金鑰(問要不要同步;插槽的金鑰本來就跨電腦通用)。
5. 裝公鑰失敗就不改 `IdentityFile`。

#### 7.3.2 Export private key…

需要檔案時的唯一出口(取代「Keep a file」):

1. 確認畫面:「Any program can use the exported file without asking, and SSHelter won't keep track of it.」
2. 存檔對話框選位置,預設檔名是金鑰名稱;不能存到 `~/.ssh/sshelter/` 底下(SSHelter 管理的目錄)。
3. 有 passphrase 的金鑰照原樣匯出(仍是加密的)。沒有 passphrase 的可選擇加一個(OpenSSH 格式,同 `ssh-keygen -p` 的預設加密)。
4. 檔案只有擁有者能讀寫(Unix 0600;Windows 只給目前使用者)。
5. 匯出不改變任何東西:保管庫裡的那把照舊,主機照舊用 SSHelter 的金鑰。

### 7.4 核准視窗

獨立的小視窗,永遠在最上層;SSHelter 縮在系統匣時也會出現。

- 標題:「Allow {程式} to use {金鑰名稱}?」;內文:程式的名稱鏈、`{user}@{host}`(或「an unknown host」)。
- 「Remember for 4 hours」(依這台的設定;「每次都問」的金鑰與未知主機沒有這個選項)。
- 需要 passphrase 時,多一個輸入欄與「Remember on this computer」。
- 「Deny」、「Allow」;要求系統驗證的金鑰,按 Allow 之後跳出 Touch ID/Windows Hello。
- 一次只顯示最早的請求。新的請求出現後 0.7 秒內 Allow(Connect 的 Unlock)不能按,答完之後要等下一個請求真的換上來:
  連點兩下不會答到使用者沒看到的下一個請求(可能是別的程式、別的主機,而且預設會記住)。Deny 不必等這 0.7 秒。

### 7.5 主機編輯器與新增金鑰

- 主機的 `IdentityFile` 可以從 Keychain 挑一把金鑰(填入插槽路徑),仍可手打路徑(Termius 的「主機連結金鑰」)。
- 「New key」:貼上私鑰、拖放或選擇檔案、從 `~/.ssh` 匯入(「Other key files」的「Import」)。「Generate key」:Ed25519(預設)、
  RSA 3072/4096、ECDSA P-256,可設 passphrase。都直接進保管庫,不留檔案;「Generate a key file…」同時移除。
- 不在帳戶的本機金鑰(§4.3「沒加入同步帳戶也能用」)在這時加入,清單標「This computer only」。
- 把既有的金鑰檔加入保管庫時,問:
  - 「Move into SSHelter」:匯入並核對指紋之後,移除原檔;確認視窗寫明「保管庫將是這把金鑰在這台電腦上唯一的一份,用 Export 可以再拿回檔案」。
  - 「Keep the file too」:原檔留著,畫面標示「這個檔案任何程式都能不經核准使用」。
- 建立「只在這台」的金鑰時,提醒匯出一份備份。

### 7.6 刪除

- 有主機在用:先列出那些主機,確認後才刪。
- 已同步的金鑰:在其他電腦上先保留,標成「Deleted on another computer」,要在那台再按一次刪除。避免一台按錯就讓每台都失去金鑰
  (同 SP3 對副本的處理)。
- 使用者自己的原始檔一律不碰。

## 8. 從 SP3(0.17.0-5)搬遷

- 更新後不自動改任何東西:主機照舊用現在的檔案連線。之後同步進來的新金鑰直接進保管庫(§4.3)。
- 這台有 `SyncedCopy` 或 `Linked` 的插槽時,Keychain 上方提示「{N} keys can move into SSHelter」與「Move」,按一次全部搬進保管庫:
  - SSHelter 自己放的副本(`SyncedCopy`,包括保管庫不能用時的暫時檔案):換成 `.pub`,私鑰放進保管庫。
  - 連到你自己檔案的(`Linked`):只拿掉插槽裡的連結;你的原始檔原封不動,之後由你自己管。
  - 一把搬不進去:其他照搬,那把留著檔案,列上顯示原因。
- 第一把金鑰進保管庫時:寫入 `agent/config` 與 Include(§6),並建議開機自動啟動(§5.7)。
- 不在插槽裡的 `~/.ssh` 金鑰:列在「Other key files in ~/.ssh」,照舊能用;2b 起可以匯入。

## 9. MCP

- 移除 `run`,連同它的核准視窗、允許主機清單與指令紀錄。MCP 保留 `list_hosts` 與 `get_effective_config`(不碰金鑰)。
- 還在呼叫 `run` 的用戶端:回錯誤,說明已移除,改用 `ssh` 加上 SSHelter 的核准。
- Settings 的 AI Access 頁改寫:AI 工具用自己的 `ssh`,經 SSHelter 的 agent 取用保管庫的金鑰,每次經過核准視窗。
- README「AI Access (MCP)」改寫:保管庫裡的金鑰,AI 不經核准就用不到;匯出的檔案與你自己在 `~/.ssh` 的金鑰檔仍然可以被任何程式直接使用。

## 10. 平台

| | macOS | Windows | Linux |
|---|---|---|---|
| agent 端點 | Unix socket | 自己的 named pipe(不碰系統服務) | Unix socket |
| 系統驗證 | Touch ID(LocalAuthentication,可退回登入密碼) | Windows Hello(UserConsentVerifier) | 無,點選 |
| 鎖定偵測 | 螢幕鎖定通知 | session lock 通知 | logind 的 Lock 訊號;拿不到就只在結束時清除 |
| 程式辨識 | §5.4 | §5.4 | §5.4 |

- Windows:Git for Windows 附的 `ssh` 不懂 named pipe(調查報告 §7.8),連不到 SSHelter 的 agent。Keychain 偵測到這台有保管庫金鑰、
  而 `git config --global core.sshCommand` 不是 `C:/Windows/System32/OpenSSH/ssh.exe` 時,顯示一行提示與要執行的指令;不自動改 git 的設定。
- 主機身分(session-bind)需要 OpenSSH 8.9 以上。macOS 內建 10.3p1(§15 實測);使用者的 Windows 是 9.5(使用者回報,2026-10-07)。
  更舊的 `ssh` 仍能簽章,但主機一律是「未知」,核准不記住。

## 11. 錯誤處理

| 狀況 | 行為 |
|---|---|
| agent 開不起來(路徑太長、權限、pipe 名稱被佔) | Keychain 顯示錯誤;用保管庫金鑰的主機暫時連不上;你自己的金鑰檔不受影響 |
| Include 那一行被刪 | 不自動加回;顯示提示與「Fix」(§6) |
| 保管庫檔讀不懂、或系統 Keychain 裡保管庫的金鑰不見 | 不覆寫,搬到旁邊保留;同步過的金鑰從帳戶重新取回;只在這台的金鑰只能靠匯出的備份 |
| 系統 Keychain 不可用 | 保管庫停用;同步的金鑰暫時寫成檔案,列上標「File for now」;保管庫恢復之後,用搬遷提示的「Move」搬回保管庫(§8) |
| 搬遷時一把搬不進去 | 其他照搬;那把留著檔案,列上顯示原因 |
| Windows 上 Git 用自己附的 `ssh` | 連不到 agent;Keychain 提示改 `core.sshCommand`(§10) |
| 核准逾時、拒絕、passphrase 錯三次、系統驗證取消 | 拒絕這次請求;`ssh` 繼續試別的方式或失敗 |
| 這台的保管庫沒有這台主機要用的金鑰 | Connect 不啟動,說明原因 |
| 一次性通道的 60 秒過了,`ssh` 才來要金鑰 | 不提供;畫面請使用者再按一次 Connect(§5.6) |
| 在自己的終端機連用保管庫金鑰的主機,而 SSHelter 沒開 | `ssh` 顯示 `no such identity`;主機頁提示「This host's key is in SSHelter; open SSHelter to connect」 |

## 12. 測試

- Rust 單元測試:保管庫的加解密與原子寫入;agent 協定(列出、各種簽章、session-bind 解析與驗證、轉送拒絕、不支援的請求回 failure);
  核准快取(三元組、到期、鎖定清除、每次都問、未知主機);`agent/config` 的產生與 Include 的插入(含和同步的 Include 並存、讀取時略過);程序樹辨識(模擬資料);
  同步的金鑰直接落地進保管庫(與保管庫不能用時的暫時檔案);在保管庫插槽上換金鑰時舊的改存成 retired;搬遷;Export private key;
  `keyprefs` 的合併與補寫(2b)。
- 用真的 OpenSSH 測 agent,不需要 sshd:`ssh-add -L` 列出;`ssh-keygen -Y sign` 經 agent 簽章、`ssh-keygen -Y verify` 驗證。
  macOS、Linux 與 Windows 的 CI 都能跑(Windows CI 用自己的 pipe)。
- 前端 vitest:Keychain 清單的排序與標記、上方的提示、明細的動作、核准視窗的文字。
- 手動驗收清單(Mac 與 Windows 各一次):Connect、Claude Code 自己跑 `ssh` 時的核准、Touch ID/Windows Hello、passphrase、
  搬遷(一鍵 Move)、Export private key、Windows 的 Git 提示、SSHelter 沒開時的訊息、`git` 經「Add a host for this key」(2b)。

## 13. 發佈

- 以 beta 發佈。2a 的 release notes 說明:同步的金鑰一律放在 SSHelter(要檔案就 Export private key)、更新後按一次「Move」
  把現有的同步金鑰搬進來、SSHelter 要在執行(建議開機自動啟動)、第一把金鑰進保管庫時會改 `~/.ssh/config`(加一行 Include)、
  Windows 的 Git 要改用系統的 OpenSSH。MCP `run` 的移除在計畫 3 的 release notes。
- 混用版本:新版的電腦把同步的金鑰放進保管庫,舊版的電腦照舊寫成檔案;`keyprefs` 舊版原樣保存。不要求每台同時更新。
- 降級:新版寫的狀態(`SlotSource::Vault`)舊版讀不懂,會把狀態檔搬到旁邊並離開帳戶,同 0.17.0-5 的降級說明;用同步碼重新加入後,
  舊版照舊把同步的金鑰寫成檔案。保管庫檔舊版不讀;只在保管庫、帳戶裡沒有的金鑰,降級前要先 Export private key。

## 14. 實作分段

一份 spec,分三份計畫(同 Sync v2):

1. 保管庫與 agent 核心:保管庫檔、agent 協定、socket/pipe、核准規則與快取、核准視窗、passphrase、程式辨識、`agent/config` 與 Include、
   Connect,以及現有「Keys used by synced hosts」裡每個同步插槽的「Only in SSHelter」切換。做完就能實際試用:把一把同步的金鑰改成只在
   SSHelter,從終端機 `ssh`,看到核准視窗。
2. 分兩段(使用者 2026-10-07 決定):
   - **2a Keychain 頁與金鑰一律在 SSHelter**:§7.1–§7.3.2 的頁面、清單與明細、Export to host、Export private key;同步的金鑰直接落地進
     保管庫(保管庫不能用時暫時寫成檔案);`own` 插槽挑金鑰時複製進保管庫;在保管庫插槽上換金鑰(舊的改存成 retired,不再要求先改回檔案);
     拿掉「Keep a file」;搬遷提示與一鍵「Move」(§8);開機自動啟動與 Windows 的 Git 提示;移除舊的 Keys 對話框。
     後端補:落地進保管庫、匯出私鑰、retired 項目、金鑰檔被哪些主機用到、Export 之後寫入主機的 `IdentityFile`。
     「Generate a key file…」暫時留在「Other key files」。做完就能以 beta 發佈。
     一鍵「Move」也會搬停止同步之後留在這台的副本(帳戶裡沒有那把),搬完保管庫就是它唯一的一份;所以計畫 1 留下的
     `vault:key` 帳戶帶世代(`docs/superpowers/plans/2026-10-07-key-vault-agent-1-followups.md`)在 2a 做:系統 Keychain 誤報
     「沒有這筆」時,新的金鑰不會蓋掉舊的,搬到旁邊的保管庫檔仍然解得開。
   - **2b 把其他金鑰帶進來**:New key(貼上、檔案、從 `~/.ssh` 匯入,Move 或 Keep the original file,決策 #5)、產生直接進保管庫
     (移除「Generate a key file…」)、不在帳戶的本機金鑰、主機編輯器挑金鑰、Add a host for this key、改名與保護(`keyprefs`)、
     passphrase 的記住與 Forget、Delete(§7.6)、列出 retired 與孤兒項目和搬到旁邊的保管庫檔(可匯出或刪除)。
3. 移除 MCP `run`、文件、平台收尾(Windows、Touch ID/Windows Hello、鎖定偵測、隱藏啟動)。

## 15. 查證紀錄

- **本機實測**(2026-10-06,macOS 內建 OpenSSH 10.3p1,scratch 的 sshd 與 ssh-agent,不碰使用者的 `~/.ssh`):
  - 插槽檔只放公鑰、agent 在:經 agent 登入成功(`explicit agent`)。
  - 只有 `<檔名>.pub`、沒有 `<檔名>`、agent 在:同樣成功。
  - agent 不在、插槽檔放公鑰:`Load key "...": invalid format`;只有 `.pub`:`no such identity: ...`。所以採用「只放 `.pub`」。
  - `ssh` 送出 session-bind(`bound agent to hostkey`)。
  - socket 路徑超過上限時,ssh-agent 開不起來(`too long for Unix domain socket`)。
- **Windows**:使用者機器的 OpenSSH 是 9.5(2026-10-07)。Win32-OpenSSH 8.9 起支援 session-bind,`IdentityAgent` 可指向自訂的 pipe
  (寫成正斜線)(調查報告 §7.8)。
- **舊版保存未知種類**:`merge_account` 對不認得的種類只保存密文(`src-tauri/src/sync/merge.rs`)。
- **協定**:RFC 9987(2026-05,Standards Track),session-bind 不在 RFC 內,見 OpenSSH 的 PROTOCOL 與 PROTOCOL.agent。
- **套件實驗**(2026-10-07,scratch,OpenSSH 10.3p1;報告在 session scratchpad 的 `agent-crate-spike/REPORT.md`):
  - 採用 `ssh-key` 0.6.7(features `crypto`、`encryption`)與 `ssh-encoding` 0.2(0.3 只配 ssh-key 0.7 預發布版,混用編譯失敗)。
  - 不用 `ssh-agent-lib` 0.6.0:不能回 SHA-1 RSA 簽章、未知訊息直接斷線(違反 RFC 9987)、沒有長度上限、Windows pipe 不能設 DACL、
    accept 出錯整個 agent 停掉、debug log 記下整個請求。自己實作約 150 行,已用 `ssh-add -L`、`ssh-keygen -Y sign/verify`、`ssh` 登入
    驗證 Ed25519、ECDSA P-256/384/521、RSA 3072。
  - `ssh-key` 0.6.7 的 RSA 簽章壞掉(`[p, p]`);改用 `rsa` 0.9 自己組私鑰,已驗證 flag 0/2/4。`rsa` 0.9 有 RUSTSEC-2023-0071(Marvin),
    本機 agent 每次簽章都要核准,被量測時間的機會小。
  - 加密私鑰:aes-ctr/cbc/gcm 與 chacha20-poly1305 都能解;舊式 PEM 與 PKCS#8 讀不了。密碼錯和 cipher 不支援是同一個錯誤,
    所以先檢查 cipher。
  - macOS:peer 的 PID 由 `LOCAL_PEEREPID` 取得;Windows:`GetNamedPipeClientProcessId`;Windows 的程式與 pipe DACL 已型別檢查。
  - Touch ID 要用 `DeviceOwnerAuthentication`(可退回登入密碼);Windows Hello 的桌面版 Interop 需要 Windows 11。

## 16. 計畫開始前要驗證的

- `ssh-key`/`ssh-agent-lib` 的版本相容、session-bind 的解析、RSA SHA-2 flag、加密私鑰支援哪些 cipher。
- Windows:pipe 的 DACL、`GetNamedPipeClientProcessId`、`IdentityAgent "//./pipe/…"` 在 9.5 的實際行為。
- Touch ID、Windows Hello 與 Tauri 的整合方式;各平台的鎖定偵測。
- `ssh-keygen -Y sign` 經 agent 簽章可以當整合測試(各平台)。

## 17. 後續(不在本文)

- 第 3 階段:內建終端機與 SSH 連線程式。
- Identity、憑證、FIDO2、Secure Enclave/TPM 金鑰(Termius 的 biometric key)。
- Linux 的系統驗證。
