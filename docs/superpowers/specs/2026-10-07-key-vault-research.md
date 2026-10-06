# SSH 金鑰管理與 agent 授權：同類產品怎麼設計（調查報告）

調查日期：2026-10-06。所有連結都在這一天讀取；頁面本身有日期或版本的，另外標在旁邊。

**範圍**：金鑰管理的 UX（Termius Keychain 一類的功能）、自帶 SSH agent 的設計、每次使用的核准、agent 怎麼知道「誰在要求」與「要連去哪台」。產品之間怎麼同步金鑰，已在 `docs/superpowers/specs/2026-10-05-sp3-key-sync-research.md` 整理，本文不重複。

**讀法**
- 表格和第 7 節的內容都是**有來源的事實**。來源順序：官方文件，其次是官方 repo 的原始碼（標出 commit 或 tag），再來是官方論壇，最後才是第三方（會註明）。
- 標 **【分析】** 的是我的推論，不是來源內容。第 9 節全部是分析。
- 「未記載」表示在讀過的官方來源裡找不到，不做推測。
- 引文保留英文原文，只引幾個字；其餘都改寫。
- 抓取方式：
  - Termius 文件讀 GitBook 提供的 `.md` 版本；1Password、Bitwarden、KeePassXC、PuTTY、Microsoft Learn 抓 HTML 後轉成文字；原始碼取固定的 commit。
  - `termius.com/documentation/*` 的舊頁面已經 404。
  - Bitwarden 的授權選項在官方頁面上只有截圖，所以改讀原始碼。
  - 沒有遇到擋抓取的網站。

**背景**：SSHelter 的下一階段要做到這幾件事：
- key 存進 SSHelter 的加密 vault，經帳號 chain 同步。
- SSHelter 跑自己的 agent（Unix socket／Windows named pipe）。
- 每次使用 key 都要核准。
- slot 檔只放 `.pub`，host 加上 `IdentityAgent`。
- MCP `run` 也走這個 agent。

本文回答這些設計問題：key 物件該存什麼、介面怎麼排、核准的粒度與記憶方式、agent 怎麼辨識請求者與目的 host、key 很多時怎麼辦、Windows 的 pipe 怎麼處理、App 鎖定或沒開時怎麼辦。

---

## 1. 總表 A：Termius Keychain（桌面版）

docs.termius.com 的頁面沒有標日期。桌面版 changelog 最新是 10.1.3（2026-09-30）。

| 項目 | 現況 | 來源 |
|---|---|---|
| Keychain 的定位 | 存 username、password、SSH key、certificate、identity。host 和 group 是**參照**同一個物件，不是複製一份，所以改一次，所有參照它的 host 都會生效 | what-is-termius |
| Key 物件存什麼 | label、private key、public key、certificate、passphrase。passphrase 可以不存：開 `Save passphrase` 就不再詢問，不存就每次連線都問 | glossary、ssh-keys-and-certificates |
| 產生 | Ed25519、ECDSA、RSA、ML-DSA。ML-DSA 從 9.37.0（2026-02-09）開始支援，user key 有 44、65、87 三級，需要 server 也支援。RSA 和 ECDSA 有哪些長度可選，官方未記載；第三方教學（GridPane，2023-06-05 更新）提到有 `Key size` 選單，可選 4096 | ssh-keys、post-quantum-cryptography、changelog/desktop、GridPane |
| 匯入 | 貼上 private key；拖放或選擇 key 檔；在 Hosts 的 Import 選 `~/.ssh`，可一次帶入 ssh_config、known_hosts 和 key 檔（`IdentityFile` 只有官網下載版能匯入）；支援 PuTTY 的 PPK3 格式（7.23.2）；可從 FIDO2 裝置匯入 resident key（7.44.0） | ssh-keys、import-existing-hosts、changelog |
| 匯出 | 文件只寫了 `Export to host`：選一台 host，可以改 `SSH keys location` 和 `Filename`，按下 `Export and Attach` 之後，公鑰會附加到 `authorized_keys`，key 也同時連到這台 host。桌面版能不能匯出到檔案、複製 public key、匯出 private key：未記載（iOS 4.3.8 版，2019-05-16，有「經剪貼簿匯出 public key」） | ssh-keys、changelog/ios |
| Identity | 一組 username、password、key（也可以是 certificate 或 FIDO2 key）的組合，可以連到多台 host 或多個 group。在 host 的 `Username` 欄位挑選。identity 必須和 host 在同一個 vault，可以用 `Credentials from` 指定從哪個 vault 取 | identities、glossary |
| host 怎麼指定 key | 在 host 的 Credentials 區塊按 `+ SSH ID, Key, Certificate, FIDO2`，挑一個物件。官方明確說：不能填本機 private key 的路徑；key 沒連結時，也不會自動把所有 key 試一遍（"Termius does not work that way"）。key 必須先匯入、再連結到 host，否則認證會失敗 | i-cant-connect-to-a-host |
| Certificate | certificate 存在 key 物件裡面（private key 和 cert 一起貼上）。host 選擇 `Certificate` 這個方式來使用。9.34.8（2025-12-22）起免費 | ssh-keys、changelog |
| FIDO2 | 可以在 App 裡產生 non-resident key，選項有 `Require User Presence`、`Require PIN code`、passphrase。server 要 OpenSSH 8.4 以上 | ssh-keys |
| Biometric key | macOS 和 iOS 存在 Secure Enclave；Windows 用 Windows Hello（TPM，8.11.0 起）；Android 用 Keystore。這種 key 不能匯出、也不會同步，每次簽章都由作業系統要求生物辨識 | ssh-keys、changelog |
| 刪除還有 host 在用的 key | 未記載。唯一相關的是：把 host 搬到或複製到別的 vault 時，連結著的 key 和 identity 要選 Copy 或 Move；選 Move 會把它們從原 vault 移除，並解除和原 vault 裡其他物件的連結 | team-vaults |
| 介面怎麼排 | 9.0.0（2024-07-11）把原本放在 Settings 裡的 Keychain 和 Known Hosts 移到 Vaults 分頁。Vaults 底下有 Hosts、Keychain、Port Forwarding、Snippets、Known Hosts 幾個畫面；這幾個畫面從 7.16.0（2021-07-10）起都支援多選。Keychain 畫面的入口有：`New key`（貼上或匯入）、key 下拉選單裡的 `Generate key` 和 `New Identity`、`Certificate`、`FIDO2`、`Touch ID` 或 `Windows Hello`。在 key 上按右鍵有 `Export to host`；點選一把 key 會打開 `Key Details` | changelog、Termius blog（2024-06-27）、ssh-keys、identities |
| 和 `~/.ssh`、系統 agent 的關係 | 文件只寫了「從 `~/.ssh` 讀取並匯入」。agent forwarding 用的是 Termius "its own built-in SSH agent"，官方明說不是作業系統的 agent；連線後會把認證用的 key 放進這個 agent。沒有記載任何對外開放的 socket。1Password 的相容性表把 Termius 列為不支援任何 SSH agent、只用內建的金鑰管理 | connecting-to-a-server、1Password compatibility |
| 系統的 `ssh` 能不能用 Termius 的 key | 未記載任何管道：沒有 agent socket，也沒有桌面版匯出 private key 的說明。舊的 termius-cli 已經 archived | 同上、github.com/termius/termius-cli |

## 2. 總表 B：其他產品的金鑰物件與管理介面

| 產品 | key 物件存什麼 | 產生 | 匯入 | 匯出 | 管理介面 |
|---|---|---|---|---|---|
| **1Password** | SSH Key item：private key、public key、fingerprint、key type。item 可以放在任何 vault，但 agent 預設只用 Personal、Private、Employee 這幾個 vault | Ed25519（預設）、RSA 2048／3072／4096；CLI 預設產生 Ed25519 | 選檔、拖放、貼上。支援 PKCS#1、PKCS#8、OpenSSH 格式。有 passphrase 的 key 只在匯入時輸入一次解開，之後改由 1Password 加密保護。不支援 DSA、ECDSA、PuTTY `.ppk`，也不支援 public exponent 小於 65537 的 RSA | private key：可匯出成 OpenSSH 或 PKCS#8（原本是 PKCS#1 的也能匯出 PKCS#1）；可加 passphrase（只限 OpenSSH 格式）或明文；可複製或下載。public key：可複製、下載，或用瀏覽器擴充自動填入 | Developer › View SSH Agent：列出 agent 可用的 key。Activity 分頁記錄每次使用的指令、時間、背景請求和用到的 key；紀錄在裝置上加密、會定期清除 |
| **Bitwarden** | SSH key item：name、private key、public key、fingerprint。可以放 folder、寫 notes、加 custom fields、開 master password re-prompt。key 本身建立後就不能修改 | 只能產生 Ed25519 | 只能從剪貼簿匯入，格式要是 OpenSSH 或 PKCS#8，不支援 PuTTY | 瀏覽器擴充可以自動填入 public key。help 頁沒寫怎麼匯出 private key | 和一般 vault item 一樣；agent 沒有獨立的狀態頁面（未記載） |
| **Secretive** | secret：name、key type、保護等級（Notify、Require Authentication、Current Biometrics）、key attribution（附在 public key 尾端的註解） | Secure Enclave：256-bit EC；macOS 26 起多了 ML-DSA-65、ML-DSA-87。smart card：EC256、EC384、RSA2048 | 不行。Secure Enclave 的設計就是不能匯入、不能匯出 | 同左。public key 有固定的檔案路徑，可以 Reveal in Finder | 主 App 加上一個獨立的 SecretAgent process。Integrations 視窗提供 shell、SSH、git 的設定片段 |
| **KeePassXC** | 資料庫 entry：key 檔放在附件或外部檔案，passphrase 放在 password 欄。SSH Agent 分頁顯示 public key、fingerprint、comment，都能複製 | 2.8.0-beta1（2026-09-23）起內建產生器：Ed25519、RSA 2048／3072（預設）／4096、ECDSA 256／384／521 | 只接受 OpenSSH 格式；PuTTYgen 產生的 key 要先轉檔 | — | 在 entry 上按右鍵，可以手動把 key 加進 agent 或移除；快捷鍵是 Ctrl+H 和 Ctrl+Shift+H |
| **Tabby** | 見前一份調查：vault 裡存 key 檔的內容（`vault://`），或只存路徑（`file://`） | — | — | — | Vault 用 master passphrase 加密（PBKDF2-SHA512 跑 10 萬次、AES-256-CBC） |

## 3. 總表 C：agent 放在哪裡、鎖定或沒開時、key 很多時

| 產品 | 有自己的 agent 嗎 | macOS／Linux 的位置 | Windows | 鎖定時 | App 沒開時 | key 很多時 |
|---|---|---|---|---|---|---|
| **1Password** | 有 | macOS：`~/Library/Group Containers/2BUA8C4S2C.com.1password/t/agent.sock`，可以自己建 symlink `~/.1password/agent.sock`。Linux：`~/.1password/agent.sock`。Flatpak 和 Snap 版不能用 agent | 接管 `\\.\pipe\openssh-ssh-agent`。使用前要把 OpenSSH Authentication Agent 服務停掉，並設成 Disabled | agent 繼續執行，有請求時要求解鎖。鎖定期間不把 private key 留在記憶體，只保留核准紀錄。public key（以及可選的 key 名稱）以明文存在磁碟上，這樣鎖定時也能顯示提示 | 結束 1Password 時，agent 和所有 agent session 一起結束。官方建議開「Start at login」，並讓 App 常駐在選單列或系統匣 | 預設提供所有符合條件的 key。`agent.toml` 可以依 item、vault、account 挑選 key 並排順序（這個檔只在本機、不同步，改了立刻生效）。也可以用 `.pub` 搭配 `IdentitiesOnly yes`，或用 SSH Bookmarks |
| **Bitwarden** | 有（用 Rust 寫） | macOS 的 .dmg 版和 Linux：`~/.bitwarden-ssh-agent.sock`。Mac App Store 版：`~/Library/Containers/com.bitwarden.desktop/Data/.bitwarden-ssh-agent.sock`。Snap 和 Flatpak 各有自己的路徑。原始碼裡還有：可以用 `BITWARDEN_SSH_AUTH_SOCK` 改路徑、socket 權限設為 0600、啟動時刪掉殘留的舊 socket | 接管 `\\.\pipe\openssh-ssh-agent`；要先停用系統服務 | 登出時 agent 不執行。鎖定時：help 頁的表格說 list 照常可用；sign 在第一次解鎖前會要求解鎖，之後的鎖定則是先解鎖、再依設定授權。原始碼註解說，鎖定時 key 仍留在加密的記憶體 keystore 裡，所以 `ssh-add -L` 照常可用 | 連不上。help 的疑難排解寫：出現 connection refused 就表示 agent 沒在跑 | 沒有依 host 選 key 的機制。help 說 agent 會把 key 依序試過，key 多了可能認證失敗，變通方法是在 ssh config 用 `IdentityFile` 指定 |
| **Secretive** | 有（獨立的 SecretAgent process） | `~/Library/Containers/com.maxgoedjen.Secretive.SecretAgent/Data/socket.ssh` | — | 看每把 key 的保護等級：Require Authentication 的 key 每次都要 Touch ID、Apple Watch 或密碼 | SecretAgent 是獨立的 process，關掉主 App 不受影響。FAQ 提到，即使移除了 App，SecretAgent 也可能一直執行到你結束它或重開機 | 每把 key 在磁碟上都有一個 `.pub`，用 `IdentityFile` 指定 |
| **KeePassXC** | 沒有，它把 key 加進系統的 agent | 用 `SSH_AUTH_SOCK`，也可以手動覆寫 | 用 Pageant、Windows OpenSSH agent（這時服務必須**開著**），或兩者都用 | 可以設定「資料庫鎖定或關閉時，把 key 從 agent 移除」 | 已加入的 key 留在系統 agent 裡；只有設了「關閉時移除」或 lifetime 才會被清掉 | — |
| **Tabby** | 沒有 | 用設定裡的 socket 路徑或 `SSH_AUTH_SOCK` | 只在 Windows 有 Agent type 設定：Automatic（先找 OpenSSH 的 pipe，再找 Pageant）、Pageant、Named pipe（可以填 pipe 路徑） | Vault 解鎖後，在 Tabby 開著時可以記住 1、5、15、60 分鐘、1 天或 7 天 | — | profile 有設 private key 時，先用 `<key>.pub` 指定 agent 裡的那一把；不行再試 agent 裡的全部 key |
| **PuTTY Pageant** | 有 | `--unix` 可以開一個 Unix socket（給 WSL1 用） | 每個使用者有自己的 pipe：`\\.\pipe\pageant.<user>.<hash>`。`--openssh-config` 會寫出一行 `IdentityAgent` 設定，讓 Windows 的 ssh 用 `Include` 引入 | 可以加入「仍然加密」的 key，第一次使用時才問 passphrase；解開之後就一直保持解開，可以手動 re-encrypt | — | — |
| **Windows OpenSSH ssh-agent** | 有（以 LocalSystem 執行的服務，預設是 disabled） | — | 固定使用 `\\.\pipe\openssh-ssh-agent`。pipe 的權限設定不讓一般使用者建立同名的 pipe instance | 沒有鎖定的概念。key 用 DPAPI 加密後，存在使用者的 registry（`SOFTWARE\OpenSSH\Agent\Keys`），重開機後還在 | 服務停用時，這個 pipe 不存在 | — |
| **OpenSSH ssh-agent**（對照組） | 有 | 新版預設在 `$HOME/.ssh/agent/s.*`；加 `-T` 改用 `$TMPDIR/ssh-XXXXXXXXXX/agent.<ppid>` | — | 可用 `ssh-add -x` 鎖定 | — | 用 `ssh-add -h` 限制目的地之後，每條連線拿到的清單只列出允許給它的 key |
| **Termius** | 只有給自己 forwarding 用的內部 agent | 沒有對外的 socket | — | — | — | 不會把 key 全部試一遍；host 必須連結 key |

## 4. 總表 D：核准、辨識請求者、得知目的 host

| 產品 | 核准的粒度（預設，以及可選的） | 核准記住多久 | 怎麼辨識請求者 | 知不知道要連去哪台 | 提示框顯示什麼 | 轉送（forwarding） |
|---|---|---|---|---|---|---|
| **1Password** | 預設：每把 key 對每個新 App（含它的子 process）問一次。可改成再細分到每個 terminal 分頁（包括 IDE 內建的 terminal），或每個請求都問。提示框上可以勾 `Approve for all applications`，等於讓同一個 OS 使用者的所有 process 都能用這把 key，這時只剩 socket 或 pipe 的權限在把關 | 預設記到 1Password 鎖定為止。可改成記到結束 1Password，或 4、12、24 小時；選時間制的話，鎖定後核准仍然有效，但還是要解鎖才拿得到 key | 官方說法：提示會顯示是哪個 App 或 process 在要求；核准綁在 process 上，可以是 terminal 視窗或分頁、IDE、GUI App。實作方式未記載。官方論壇的員工（2022-06-10）說，每個分頁分開授權是預期中的行為 | 未記載 | App 和 key。key 預設顯示截短的 fingerprint；要開 `Display key names` 才會顯示名稱 | 只支援 Mac 和 Linux。在轉送的 session 裡核准某把 key 之後，遠端同一 OS 使用者的 process 都能用它；其他 key 照樣要核准 |
| **Bitwarden** | `Ask for authorization when using SSH agent` 有三個選項：Always（原始碼裡的預設值）、Never、Remember until vault is locked | 選 Remember 時，以「key × 目的 host 的 host key 指紋 × 本機或轉送」為單位記住。沒有 session-bind 的本機簽章（例如 git 簽章）一律歸到「local」。vault 鎖定或切換帳號時清空 | 用 socket 的 peer PID 查 process 名稱：macOS 用 `proc_pidpath`，其他平台用 sysinfo；Windows 用 `GetNamedPipeClientProcessId` 取 PID。查不到時顯示 Unknown application | 知道：驗證 session-bind 的簽章後，取出 host key 的 SHA-256 指紋和 is_forwarding。指紋只用來記住核准，不會顯示給使用者 | 標題是「Confirm SSH key usage」，內容是哪個 App、哪把 key，以及用途：登入 server、簽訊息或簽 git commit（從 SSHSIG 的 namespace 判斷） | 轉送來的請求會多一個「Agent Forwarding」警告；記住的核准也和本機的分開存 |
| **Secretive** | 建立 key 時選保護等級：Notify（Mac 解鎖時不用驗證，但每次使用都會通知）、Require Authentication（每次都要 Touch ID、Apple Watch 或密碼）、Current Biometrics（指紋設定一改，key 就不能用了） | 需要驗證的 key 通過之後，通知上可以選 `Leave Unlocked` 1 分鐘、5 分鐘、1 小時、24 小時，或選 Do Not Unlock | 從 peer PID 一路往上找，直到第一個 GUI App，記下它的名稱、路徑、圖示，以及程式碼簽章是否有效 | 4.0（2026-09-21）起知道：解析被簽資料裡 `publickey-hostbound-v00` 帶的 host key，再查 `~/.ssh/known_hosts` 換成主機名稱，查不到就顯示「unknown host」。session-bind 已經會解析，但目前一律回 failure，原始碼註解寫著這功能 "disabled until forward enforcement is handled" | 「從某 App 連到 `user@host`，使用某把 key」。git 簽章則顯示 namespace。好幾個請求同時等待核准時，可以一次整批核准 | FAQ：在遠端每次用到 key，都要先經過 Secretive 驗證 |
| **KeePassXC** | 每把 key 可以設 `Require user confirmation`（加上 confirm constraint），以及「幾秒後從 agent 移除」（lifetime constraint） | 由 lifetime 的秒數決定 | 交給系統 agent 處理 | — | 由 agent 決定；OpenSSH 的 agent 用 ssh-askpass 跳視窗 | — |
| **OpenSSH ssh-agent** | 用 `ssh-add -c` 加 key 的話，每次使用都用 ssh-askpass 確認 | `ssh-add -t` 設 key 的 lifetime；也可以在啟動 agent 時用 `-t` 設預設值 | 只檢查 peer 的 euid 是不是同一個使用者 | 知道（session-bind 和 hostbound），但只用在 `-h` 的 destination constraint | 「Allow use of key …?」加上 fingerprint。原始碼裡有個 TODO：轉送路徑和目的 host 還沒顯示在 askpass 視窗上 | 用 `-h` 可以逐 hop 限制 |
| **Windows OpenSSH ssh-agent** | 沒有核准機制。8.1 版會忽略 `ssh-add -c`、`-t`；9.5 版改成直接拒絕 | key 永久保存 | 用 client 的 token 判斷是哪個使用者 | 9.5 版會驗證並記錄 session-bind，但不支援 destination constraint | — | — |

## 5. 總表 E：協定事實（RFC 9987、OpenSSH 的 PROTOCOL 和 PROTOCOL.agent）

| 主題 | 事實 | 來源 |
|---|---|---|
| 標準化狀態 | agent 協定在 2026 年 5 月發布為 RFC 9987（Standards Track，作者 D. Miller）。它的前身是 draft-miller-ssh-agent，之後成為工作小組草案 draft-ietf-sshm-ssh-agent。OpenSSH 的 PROTOCOL.agent 現在只記錄 OpenSSH 自己的擴充，開頭就直接指向 RFC 9987。session-bind 這些擴充**不在** RFC 裡 | RFC 9987、IETF datatracker、PROTOCOL.agent（rev 1.26，2026-06-02） |
| 連線端點與存取控制 | Unix 上用 Unix domain socket，Windows 上用 named pipe，位置通常放在 `SSH_AUTH_SOCK`。能連上 agent 通常就能用裡面的 key，所以 agent 必須只開放給擁有者：Unix 靠 socket 的檔案權限，或檢查 peer credential（例如 SO_PEERCRED）；Windows 則在建立 pipe 時附上 security descriptor | RFC 9987 §6、§10 |
| Key constraint | 有三種：lifetime（代碼 1）、confirm（代碼 2）、extension（代碼 255，名稱格式是 `name@domain`）。agent 遇到不認得或不支援的 constraint，必須拒絕整個加入請求。這是刻意設計成出錯時保持安全（fail safe） | RFC 9987 §5.2.7、§8.2 |
| Lock 和 unlock | 用一組 passphrase 鎖住 agent。鎖住期間至少要暫停所有簽章。agent 應該防範暴力猜測，例如延遲回應、暫時拒絕，或刪掉 key | RFC 9987 §5.7、§10 |
| Extension 機制 | 用 `SSH_AGENTC_EXTENSION` 加上一個名稱。agent 不支援的 extension，回一個空的 `SSH_AGENT_FAILURE`。另外有一個可選的 `query`，可以列出 agent 支援哪些 extension。extension 本身執行失敗時，要回 `SSH_AGENT_EXTENSION_FAILURE`，才能和「不支援」區分開來 | RFC 9987 §5.8 |
| 簽章 flag | `SSH_AGENT_RSA_SHA2_256`（0x02）和 `SSH_AGENT_RSA_SHA2_512`（0x04），只適用於 `ssh-rsa` key，要求 agent 回 `rsa-sha2-256` 或 `rsa-sha2-512` 簽章。agent 不支援請求的 flag 時，必須回 failure。OpenSSH 8.8 起，預設停用以 SHA-1 做的 RSA 簽章 | RFC 9987 §5.6.1、§8.3；OpenSSH 8.8 release notes |
| session-bind@openssh.com 的內容 | 包含 server 的 host key、session identifier（第一次 key exchange 算出的 exchange hash）、server 用 host key 對 session id 做的簽章，以及 `is_forwarding`。agent 收到後要驗證簽章、拒絕重複的 session id、拒絕把已經用於認證的連線再綁一次，並在連線存在期間記住這些資訊，給 destination constraint 使用 | PROTOCOL.agent §1 |
| session-bind 什麼時候送 | 它是 ssh 連上 agent 後送的**第一個**訊息。用於認證的連線，會先送 session-bind（is_forwarding=0），才去要 key 清單。轉送 agent 時，每開一條 channel 就送一次（is_forwarding=1）。agent 不支援的話，ssh 只在 debug2 記一行，照常繼續 | sshconnect2.c `get_agent_identities`、clientloop.c、agent-restrict.html |
| hostbound 公鑰認證 | 認證方法 `publickey-hostbound-v00@openssh.com` 會在 userauth 請求裡多帶 server 的 host key，而整個請求都在被簽的資料裡。server 用 EXT_INFO 宣告 `publickey-hostbound@openssh.com`；client 看到就應該優先使用這個方法 | PROTOCOL §3.1；sshconnect2.c |
| 從被簽資料能看出什麼 | userauth 的被簽資料裡有 session id、目的 username 和公鑰；用 hostbound 方法時，還多了 server 的 host key。OpenSSH 的 ssh-agent 會解析這些資料，並檢查 host key 是否和最近一次綁定的相同 | ssh-agent.c `parse_userauth_request`；agent-restrict.html |
| restrict-destination-v00 | 用 `ssh-add -h` 逐 hop 限制 key 能用在哪裡。ssh-add 會用 known_hosts 把主機名稱換成 host key。之後在每條連線上，列清單、簽章、刪除都會依綁定來檢查：不允許的 key 連清單上都不會出現。這個限制只對 user authentication 有效，不能用在 ssh-keygen 的簽章上 | PROTOCOL.agent §2；ssh-add(1)；ssh-agent.c `process_request_identities`；agent-restrict.html |
| 這套機制的限制 | agent、ssh、ssh-add 都要支援；轉送時，遠端的 client 和 server 也要支援。惡意的中繼主機可以把路徑「拉長」，但最終目的地無法偽造，第一跳也可以信任。設計文件把這件事比喻成「把 key 委託給某台 host」 | agent-restrict.html（2022-01-10）；ssh-add(1) |
| 舊的 confirm 為什麼不夠 | agent-restrict 的作者指出，舊的 confirm 視窗看不到目的 host，也看不到轉送路徑，"somewhat easy to phish"：攻擊者只要抓準使用者正在連線的時機，對方就可能按下同意 | agent-restrict.html |
| 轉送的風險 | 在遠端能繞過 socket 檔案權限的人，可以透過轉送用你的 key 去認證（但拿不到 key 本身）。官方建議改用 ProxyJump。RFC 要求：實作不應該預設開啟轉送；agent 應該對轉送進來的連線加上額外的控制，否則使用者只剩「全部開放」或「完全不轉送」兩種選擇 | ssh(1)、ssh_config(5)、RFC 9987 §10、agent-restrict.html |
| IdentityAgent | 會覆寫 `SSH_AUTH_SOCK`。設成 `none` 就停用 agent；也可以寫字串 `SSH_AUTH_SOCK`，或用 `$變數`；支援 `~` 和各種 tokens | ssh_config(5) |
| 用 `.pub` 當 IdentityFile | IdentityFile 可以指向 public key，實際用的是 agent 裡對應的 private key。`IdentitiesOnly yes` 讓 ssh 只用設定裡指定的 identity。agent 裡如果有和設定檔相同的 key，會排在最前面 | ssh_config(5)；sshconnect2.c `pubkey_prepare` |
| server 端的限制 | `MaxAuthTries` 預設 6 次；`LoginGraceTime` 預設 120 秒。`PerSourcePenalties` 從 9.8 起預設開啟：認證失敗後斷線罰 5 秒、超過 LoginGraceTime 罰 10 秒；罰的時間累積到 15 秒才開始拒絕連線，最多累積到 10 分鐘 | sshd_config(5)；OpenSSH 9.8 release notes |

## 6. 總表 F：誰會送 session-bind、誰會用它

「不送」的判斷方法：用 2026-10-06 時各專案預設分支（或註明的 commit）的原始碼，搜尋 `session-bind` 字串，找不到就算不送。

| client | 會不會送 | 依據 |
|---|---|---|
| OpenSSH `ssh` 8.9 以上（2022-02-23） | 送 | 8.9 release notes；sshconnect2.c |
| macOS 內建的 ssh | 送。本機觀察：macOS 27.0.1 的版本是 `OpenSSH_10.3p1` | 本機執行 `ssh -V`（不是官方文件） |
| Ubuntu 22.04、24.04 | 送（分別是 8.9p1、9.6p1） | Launchpad |
| RHEL 9 | 版本是 8.7p1，比 8.9 舊。Red Hat 有沒有 backport 這個功能，未查證 | RHSA-2024:4312 |
| Win32-OpenSSH（Microsoft 維護的 fork） | tag v8.1.0.0、v8.6.0.0 沒有；v8.9.0.0 起有（v9.5.0.0 也有），client 會送、agent 也會處理 | PowerShell/openssh-portable 各個 tag 的 authfd.c、keyagent-request.c |
| Windows 內建的 OpenSSH | 安裝媒體上的版本「例如 7.7p1 或 8.1p1」，這些不送。Microsoft 有一篇疑難排解文章，把 2024-10-08 起各版 Windows 的更新列為「OpenSSH Version 9.5.2.1」相關更新，涵蓋 Windows 10 22H2、Windows 11 21H2 到 24H2、Server 2019 到 2025。【分析】如果這些更新確實把內建 client 升到 9.5，就會送。各版 Windows 實際的 `ssh -V` 對照，沒有找到官方表格 | Microsoft Learn 兩篇文章；Win32-OpenSSH issue #1949 的 log 裡，8.9p1 有出現 `bound agent to hostkey` |
| PuTTY 和 Plink（含 Pageant） | 不送，也不處理 | git.tartarus.org putty 37af671（2026-08-26） |
| Go 的 `golang.org/x/crypto/ssh` | 不送 | GitHub golang/crypto |
| russh（Tabby 用的函式庫） | 不送 | GitHub Eugeny/russh |
| libssh、paramiko | 不送 | git.libssh.org、GitHub paramiko |

| agent | 怎麼用 session-bind |
|---|---|
| OpenSSH ssh-agent | 驗證簽章並記錄，用在 destination constraint，也用來過濾回給 client 的 key 清單 |
| Windows ssh-agent 9.5 | 驗證簽章並記錄；但不支援 destination constraint |
| Bitwarden | 驗證 host key 是 Ed25519 或 RSA 的簽章；ECDSA host key 還不支援，驗證會失敗。取出指紋，用來記住核准和顯示轉送警告 |
| Secretive 4.0 | 會解析，但回 failure；改從 hostbound 的被簽資料取 host key |
| 1Password | 未記載 |

---

## 7. 各產品筆記與來源

### 7.1 Termius
- Key、Identity、Keychain 的定義：https://docs.termius.com/getting-started/glossary 、https://docs.termius.com/getting-started/what-is-termius
- 產生、匯入、Export to host、certificate、FIDO2、biometric key：https://docs.termius.com/keychain/ssh-keys-and-certificates
- Identity 和「`Credentials from` 指定 vault」：https://docs.termius.com/keychain/identities
- agent forwarding 用內建 agent：https://docs.termius.com/organize-and-connect-to-hosts/connecting-to-a-server 。同一頁也寫到，Mac App Store 和 Snap 版因為 sandbox，沒有 local terminal。
- `~/.ssh` 匯入，以及「`IdentityFile` 只有官網下載版能匯入」：https://docs.termius.com/getting-started/import-existing-hosts
- 「key 必須匯入並連結，不吃路徑、也不會試全部」：https://docs.termius.com/help-center/troubleshooting/i-cant-connect-to-a-host 。官方給的理由是：要跨桌面和手機都一致，資料就必須存在 vault 裡。
- ML-DSA 的 44、65、87 三級：https://docs.termius.com/security/post-quantum-cryptography
- 搬移或複製時怎麼處理連結的物件：https://docs.termius.com/team-collaboration/team-vaults
- 版本與日期：https://docs.termius.com/changelog/desktop
  - 7.28.0（2021-12-13）：可產生 Ed25519、ECDSA、RSA。
  - 7.36.0（2022-03-23）：新增 Export to host。
  - 7.37.0（2022-03-29）：Secure Enclave key。
  - 9.0.0（2024-07-11）：Keychain 移到 Vaults 分頁。
  - 9.37.0（2026-02-09）：ML-DSA。
  - 10.1.0（2026-09-21）：Agent Forwarding 改為免費。
- iOS 版的舊紀錄：https://docs.termius.com/changelog/ios （4.3.8，2019-05-16，經剪貼簿匯出 public key）
- 介面調整的理由：https://termius.com/blog/termius-x （2024-06-27）。Keychain 和 Known Hosts 為了讓介面保持清爽，在 Settings 裡放了好幾年；後來改版，把所有資料集中到 Vaults 分頁。
- 和 agent 的相容性：https://www.1password.dev/ssh/agent/compatibility 。Termius 在 Mac、Windows、Linux 都被標成不支援任何 SSH agent。
- 第三方：key size 選單（GridPane，2023-06-05 更新）：https://gridpane.com/kb/generate-an-ssh-key-with-termius/
- termius-cli 已經 archived（最後 push 是 2024-07-16）：https://github.com/termius/termius-cli

### 7.2 1Password SSH agent
- 總覽：預設使用哪些 vault；合格的 key 只有 Ed25519 和 RSA，而且必須是 active item（已封存或刪除的不算）；超過 6 把 key 會有問題。https://www.1password.dev/ssh/agent
- 核准的三種粒度、三種記憶方式、`Approve for all applications`，以及背景請求怎麼壓下（請求來源不在前景時，不跳提示，只在選單列或系統匣圖示上加一個點，點開才看到「SSH request waiting」）。https://www.1password.dev/ssh/agent/authorization
- 授權模型：
  - 核准綁在 process 上，涵蓋它的子 process。
  - 鎖定時不把 private key 留在記憶體，只留核准紀錄。
  - public key 以明文存在磁碟上；`Display key names` 預設關閉，關閉時提示只顯示截短的 fingerprint。
  - 和 OpenSSH agent 的差別是：沒有 ssh-add 那種「加入或移除 key」的概念。
  - 來源：https://www.1password.dev/ssh/agent/security
- 6 把 key 的上限：
  - OpenSSH server 的 `MaxAuthTries` 預設 6 次，agent 提供第 7 把 key 時，會出現 `Too many authentication failures`。
  - 解法是下載 `.pub`，搭配 `IdentityFile` 和 `IdentitiesOnly yes`；官方提醒有些 SSH client 不支援在 `IdentityFile` 放 public key。
  - 漸進遷移的寫法：個別 host 用 `IdentityAgent`；或用 `Host *` 預設走 1Password，再對例外的 host 寫 `IdentityAgent none`。
  - Windows：官方說 Microsoft OpenSSH 只聽固定的 pipe，所以無法依 host 切換 agent。7.8 節有反例。
  - 來源：https://www.1password.dev/ssh/agent/advanced
- `agent.toml`：
  - 欄位有 `[[ssh-keys]]` 加上 item、vault、account（像 AND 條件一樣組合）。
  - 一個條目符合多把 key 時，依建立時間由舊到新排列；條目的順序就是提供給 server 的順序。
  - 檔案位置：Windows 在 `%LOCALAPPDATA%/1Password/config/ssh/agent.toml`；Mac 和 Linux 在 `~/.config/1Password/ssh/agent.toml`（會先找 `XDG_CONFIG_HOME`）。
  - 自己手動建立的空檔案也會覆蓋預設行為，等於一把 key 都不提供。
  - 來源：https://www.1password.dev/ssh/agent/config
- 位置、開啟方式、Windows 要停用的服務、Git for Windows 要把 `core.sshCommand` 改成 `C:/Windows/System32/OpenSSH/ssh.exe`：https://www.1password.dev/ssh/get-started
- 轉送：
  - 只支援 Mac 和 Linux（Windows 改用 WSL integration）。
  - 官方建議只對信任的特定 host 開啟。
  - 遠端工作站上可以用 `Match host * exec "test -z $SSH_TTY"`，讓 `IdentityAgent` 只在本機 shell 生效，從 SSH 登入時就改用轉送過來的 socket。
  - 來源：https://www.1password.dev/ssh/agent/forwarding
- 產生、匯入、匯出、Activity log：https://www.1password.dev/ssh/manage-keys
- 相容性表：Microsoft OpenSSH 支援 Windows pipe，也支援用 `.pub` 當 `IdentityFile`；`ssh-add` 只能列出 key，不能加入或刪除，也不能 lock 或 unlock。https://www.1password.dev/ssh/agent/compatibility
- 社群（非官方文件）：員工 floris_1P 在 2022-06-10 說，每個 terminal 分頁分開授權是預期中的行為。https://www.1password.community/developers-69/1password-asking-for-permission-each-time-19323/index2.html
- 頁面都沒有標日期。

### 7.3 Bitwarden SSH agent
- 開啟方式、各平台的 socket 位置、Windows 要停用服務、各種 vault 狀態下的行為表、轉送時的提示：https://bitwarden.com/help/ssh-agent/
- key 欄位、只能產生 Ed25519、只能從剪貼簿匯入、key 本身不能修改，以及 agent 的限制：
  - 不能用 ssh-add 管理 key。
  - 沒有依 host 選 key 的機制。
  - 和系統原生的 agent 並存時，可能悄悄改用原生 agent；授權設定是 Never 或 Remember 時，畫面上看不出是哪個 agent 處理了請求。
  - 來源：https://bitwarden.com/help/about-ssh/
- 原始碼（bitwarden/clients @ `d1c95e72`，2026-10-06）：
  - 架構：Rust 寫的 server 透過 napi 和 Electron 連接；`AuthPolicy`（決定要不要核准）和 `ApprovalRequester`（去問 UI）分成兩個介面，方便測試。https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/ssh_agent/README.md
  - session-bind：驗證 Ed25519 和 RSA（rsa-sha2-256／512）的 host key 簽章；ECDSA 會記一行 warning 然後失敗；is_forwarding 一旦設為真，之後的 bind 不能再把它取消。https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/ssh_agent/src/server/session_bind.rs
  - 辨識請求者（PID 換成 process 名稱）：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/ssh_agent/src/server/peer_info.rs
  - Windows 固定使用 `\\.\pipe\openssh-ssh-agent`，用 `GetNamedPipeClientProcessId` 取 PID：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/ssh_agent/src/server/listener/windows.rs
  - Unix 的 socket 權限是 0600，可用 `BITWARDEN_SSH_AUTH_SOCK` 覆寫路徑：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/ssh_agent/src/server/listener/unix.rs
  - 記住核准的 cache key（`local`、`local:<指紋>`、`forwarded:<指紋>`）、等待解鎖的逾時 60 秒、收到 vault 裡已不存在的 key 的請求時直接拒絕：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/src/autofill/services/ssh-agent.service.ts
  - 預設值是 Always：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/src/platform/services/desktop-settings.service.ts
  - 等核准回應的逾時 60 秒：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/desktop_native/napi/src/sshagent.rs
  - 介面字串（三個授權選項、Confirm SSH key usage、Agent Forwarding 警告、三種用途）：https://github.com/bitwarden/clients/blob/d1c95e72be9ab5e9e05417f7aa5ba259fbf27e50/apps/desktop/src/locales/en/messages.json

### 7.4 Secretive（macOS）
- Secure Enclave 不能匯出、新 Mac 要重新產生 key、可以要求 Touch ID 或 Watch、每次使用都通知、支援 smart card：https://github.com/maxgoedjen/secretive （README）
- 只支援 256-bit EC；指定 key 的方法是用 2.2 版起的「Public Key Path」，搭配 `IdentityFile`；可以轉送：https://github.com/maxgoedjen/secretive/blob/main/FAQ.md
- 4.0.0（2026-09-21）：
  - 新功能：pending requests 介面、「request usage attribution」（標示請求是誰發的、要做什麼）、SSH certificate 介面、OpenSSH extension 解析、從 known_hosts 查主機名稱。
  - release：https://github.com/maxgoedjen/secretive/releases/tag/v4.0.0
  - 整批核准的 PR：https://github.com/maxgoedjen/secretive/pull/821
- 原始碼（main，2026-10-01）：
  - 收到 session-bind 一律回 failure；從 hostbound 的 payload 取 host key：`Sources/Packages/Sources/SecretAgentKit/Agent.swift`、`Sources/Packages/Sources/SSHProtocolKit/SSHAgentInputParser.swift`
  - 往上找到 GUI App 並檢查程式碼簽章：`Sources/Packages/Sources/SecretAgentKit/SigningRequestTracer.swift`
  - Leave Unlocked 的四種時長：`Sources/SecretAgent/Notifier.swift`
  - known_hosts 的查表：只讀格式剛好三欄的行，以 host key 對應第一欄：`Sources/SecretAgentHostsfileReader/SecretAgentHostsfileReader.swift`
  - socket 和 PublicKeys 的路徑：`Sources/Packages/Sources/Common/URLs.swift`
  - 介面字串：`Sources/Packages/Resources/Localizable.xcstrings`
- shell 設定（`SSH_AUTH_SOCK` 的路徑）：https://github.com/maxgoedjen/secretive-config-instructions/blob/main/shells/zsh.md

### 7.5 KeePassXC
- 使用手冊的 SSH Agent 一節：
  - KeePassXC 不提供 agent，只當 OpenSSH 相容 agent 的 client。
  - Windows 上支援 Pageant 和 OpenSSH，預設用 Pageant；用 OpenSSH 時要把服務設成 Automatic 並啟動。
  - gpg-agent 不相容，因為它不支援移除 key。
  - 來源：https://keepassxc.org/docs/KeePassXC_UserGuide
- 每把 key 的選項（解鎖時加入、鎖定時移除、`Require user confirmation`、幾秒後移除）：https://github.com/keepassxreboot/keepassxc/blob/develop/src/gui/entry/EditEntryWidgetSSHAgent.ui
- 把 key 加入 agent 時帶上 lifetime 和 confirm constraint；agent 不支援時，顯示「…is not supported by the agent」；Windows 上連 `\\.\pipe\openssh-ssh-agent` 或 Pageant：https://github.com/keepassxreboot/keepassxc/blob/develop/src/sshagent/SSHAgent.cpp
- 內建產生 key：2.8.0-beta1 的 release notes（#7215），以及 `src/sshagent/OpenSSHKeyGenDialog.cpp`

### 7.6 Tabby
- 只當 agent 的 client：
  - Windows 的 Automatic 會先檢查 OpenSSH 的 pipe，再找 Pageant。
  - 走 agent 認證時，先載入 `<key>.pub` 指定那一把，再加一個「試全部」的備案。
  - 對方要求轉送、但 profile 沒開轉送時，拒絕對方開的 agent channel。
  - 來源：https://github.com/Eugeny/tabby/blob/master/tabby-ssh/src/session/ssh.ts
- Agent type 設定只在 Windows 出現：https://github.com/Eugeny/tabby/blob/master/tabby-ssh/src/components/sshSettingsTab.component.pug
- Vault 的加密方式（PBKDF2 和 AES）、記住 passphrase 的選項（1、5、15、60 分鐘、1 天、7 天、不記住）：https://github.com/Eugeny/tabby/blob/master/tabby-core/src/services/vault.service.ts 、`tabby-core/src/components/unlockVaultModal.component.ts`
- 版本：最新 release 是 v1.0.237（2026-09-25）。

### 7.7 PuTTY Pageant
- 手冊（PuTTY 0.85）：
  - 9.3.3 節：用 `--openssh-config` 產生給 Windows ssh 的設定檔，再用 `Include` 引入。這招只適用 Windows 自帶的 OpenSSH；Git for Windows 附的 ssh 不懂 named pipe。
  - 9.5 節：可以加入加密狀態的 key，第一次使用才問 passphrase；之後可以 re-encrypt。
  - 9.6 節：轉送的風險。
  - 來源：https://the.earth.li/~sgtatham/putty/latest/htmldoc/Chapter9.html
- 原始碼（git.tartarus.org/simon/putty，37af671）：
  - pipe 名稱是 `\\.\pipe\pageant.<username>.<混淆過的字串>`：`windows/utils/agent_named_pipe_name.c`
  - 寫設定檔時把 `\` 一律換成 `/`，註解說有些版本的 Windows OpenSSH 比較吃 `/`，而且已知沒有版本會拒絕 `/`：`windows/pageant.c`

### 7.8 Windows OpenSSH：內建 agent，以及第三方 agent 怎麼共存
- Microsoft 文件：
  - ssh-agent 服務預設是 disabled。
  - 官方範例是用 `ssh-add` 把 key 存進 agent 之後，建議刪掉本機的 private key 檔，並說用 ECDSA 這類演算法時，key 無法從 agent 取回。
  - 來源：https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement
- Microsoft 疑難排解：ssh-agent 存的 private key 會一直留在 registry，跨 session 都在，建議定期稽核和清除。https://learn.microsoft.com/troubleshoot/windows-server/system-management-components/open-client-can-not-connect-server
- 內建版本「例如 7.7p1 或 8.1p1」常常落後，只會隨 Windows Update 更新：https://learn.microsoft.com/troubleshoot/windows-server/system-management-components/upgrade-in-box-openssh-to-latest-openssh-release
- OpenSSH 9.5.2.1 的更新清單（2024-10-08 起）：https://learn.microsoft.com/troubleshoot/windows-server/system-management-components/error-1053-1067-7034-after-update-openssh-doesnt-start
- Win32-OpenSSH wiki：
  - Various Considerations：Windows 的 ssh-agent 只支援 `-l`、`-L`、`-d`、`-D`，會忽略 `-c`、`-t`，而且會 "persistently and permanently stores" 使用者的 key。https://github.com/PowerShell/Win32-OpenSSH/wiki/Various-Considerations
  - 設計文件：agent 是 LocalSystem 服務，故意聽一個固定的 IPC 位置，用來防止被劫持或冒充。https://github.com/PowerShell/Win32-OpenSSH/wiki/About-Win32-OpenSSH-and-Design-Details
- 原始碼（PowerShell/openssh-portable，分支 `latestw_all`，以及各 tag）：
  - pipe 的 DACL：SYSTEM 和 Administrators 有完整權限；Authenticated Users 有讀寫權限，但**沒有** `FILE_CREATE_PIPE_INSTANCE`；並設了 `PIPE_REJECT_REMOTE_CLIENTS`。`contrib/win32/win32compat/ssh-agent/agent.c`
  - key 寫進 registry 前先用 DPAPI 加密。9.5 版只接受 `sk-provider` 這種 constraint extension；lifetime 和 confirm 會被當成未知的 constraint 而拒絕；8.1 版則是根本不解析 constraint。session-bind 的處理從 v8.9.0.0 開始有。`contrib/win32/win32compat/ssh-agent/keyagent-request.c`（比對了 `v8.1.0.0`、`v9.5.0.0`、`latestw_all`）
  - 沒有設 `SSH_AUTH_SOCK` 時，預設指向 `\\.\pipe\openssh-ssh-agent`；有設就用設定的值：`contrib/win32/win32compat/wmain_common.c`
  - AF_UNIX 的 connect 是用 `CreateFileW` 去開那個路徑來模擬的，所以 `SSH_AUTH_SOCK` 或 `IdentityAgent` 指向別的 pipe 也行得通：`contrib/win32/win32compat/fileio.c`
- `IdentityAgent` 指向自訂 pipe 的實例：Win32-OpenSSH 8.9 起，用反斜線寫的 `\\.\pipe\…` 會失敗，改成 `//./pipe/…` 或把反斜線加倍就好。Pageant 0.79 起產生的設定檔就是用斜線。https://github.com/PowerShell/Win32-OpenSSH/issues/1949 （2022-06-02 開，仍未關閉）
- 1Password 和 Bitwarden 都要求先停用 OpenSSH Authentication Agent 服務，才能接管預設的 pipe（見 7.2、7.3）；KeePassXC 用 OpenSSH 時則需要這個服務開著（見 7.5）。

### 7.9 協定與 OpenSSH 原始碼
- RFC 9987（2026 年 5 月）：https://www.rfc-editor.org/rfc/rfc9987 ；草案的演變：https://datatracker.ietf.org/doc/rfc9987/
- PROTOCOL.agent（rev 1.26，2026-06-02）：https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.agent
- PROTOCOL §3.1 hostbound（rev 1.60，2026-02-09）：https://github.com/openssh/openssh-portable/blob/master/PROTOCOL
- agent-restrict 設計文件（2022-01-10）：https://www.openssh.com/agent-restrict.html 。文件也提到一個被否決的替代方案：agent 開多個 socket，再用 `IdentityAgent` 和 `ForwardAgent` 分配 key；否決的理由是要大量手動設定，而且沒有密碼學上的保證。
- Release notes：8.8（停用 SHA-1 的 RSA 簽章）https://www.openssh.com/txt/release-8.8 ；8.9（agent 限制，2022-02-23）https://www.openssh.com/txt/release-8.9 ；9.8（PerSourcePenalties）https://www.openssh.com/txt/release-9.8
- Man page：https://man.openbsd.org/ssh_config 、https://man.openbsd.org/ssh-add 、https://man.openbsd.org/ssh-agent （`SSH_AUTH_SOCK` 只有本人能存取，但 "easily abused by root" 或同一使用者的其他 process）、https://man.openbsd.org/ssh 、https://man.openbsd.org/sshd_config
- 原始碼（openssh-portable master）：
  - `sshconnect2.c`：`get_agent_identities` 先 bind 再要清單；`pubkey_prepare` 的 key 排序和 `IdentitiesOnly`；hostbound 的選擇；`load_identity_file`。
  - `clientloop.c`：轉送時的 bind。
  - `ssh-agent.c`：`confirm_key`、`process_request_identities`、`parse_userauth_request`，以及檢查 peer euid。
  - `authfile.c` 的 `sshkey_load_public`：依序試原檔名、加上 `.pub` 的檔名、最後從 private key 檔取出 public key。
- 各 client 的支援依據：
  - https://launchpad.net/ubuntu/jammy/+source/openssh
  - https://launchpad.net/ubuntu/noble/+source/openssh
  - https://access.redhat.com/errata/RHSA-2024:4312
  - https://github.com/golang/crypto
  - https://github.com/Eugeny/russh
  - https://git.libssh.org/projects/libssh.git
  - https://github.com/paramiko/paramiko

---

## 8. 觀察到的模式（只根據上面的來源）

1. **Keychain 都是「物件 + 參照」的模型。**
   - Termius 的 host 連結的是 key 物件；不接受路徑，也不會把 key 全部試一遍。
   - 1Password 和 Bitwarden 的 key 是 vault item。
   - Secretive 替每把 key 在磁碟上放一個 `.pub`，給 `IdentityFile` 指。
   - Termius 另外有 Identity（username、password、key 的組合），讓多台 host 共用一組認證資料。
2. **在 Windows 上搶同一條 pipe 的有兩派。**
   - 1Password 和 Bitwarden 接管預設的 `\\.\pipe\openssh-ssh-agent`，所以必須停用系統服務。
   - KeePassXC 走 OpenSSH 時，需要這個服務開著。【分析】所以兩派在預設 pipe 上無法並存。
   - Pageant 走第三條路：用自己的、每個使用者一條的 pipe，再產生 `IdentityAgent` 設定。
   - Windows 系統服務對這條 pipe 設了權限，一般使用者無法插入同名的 pipe instance。
3. **「誰在要求」的辨識粒度差很多。**
   - 1Password 細到 App 和 terminal 分頁。
   - Secretive 細到 GUI App，還會檢查程式碼簽章。
   - Bitwarden 只拿連上 socket 的那個 process 的名稱。【分析】從 terminal 或 git 發出的請求，連上 socket 的是 `ssh` 本身，所以顯示的通常就只是 `ssh`。
4. **「要連去哪」很少被用在介面上。**
   - Secretive 4 會顯示 `user@host`，靠的是 hostbound 的 payload 加上 known_hosts。
   - Bitwarden 用 session-bind 的指紋，但只拿來記住核准，不顯示。
   - 1Password 未記載。OpenSSH 自己的 confirm 視窗也還不會顯示。
5. **記住核准有三種方式。**
   - 依 App 或 session 記（1Password）。
   - 依「key × host」記（Bitwarden）。
   - 依「key × 時間」記（Secretive；1Password 的時間制選項也算）。
   - 預設值兩極：Bitwarden 預設每次都問；1Password 預設每個 App 只問一次，一直記到鎖定。
6. **鎖定時，agent socket 都還在。**
   - 1Password 鎖定時不把 private key 留在記憶體，但把 public key 存在磁碟上，好在鎖定時顯示提示。
   - Bitwarden 把 key 留在加密的記憶體裡：list 照常可用，sign 要先解鎖；解鎖和核准各有 60 秒逾時。
7. **key 太多的問題，幾乎都靠 client 端的設定解決。**
   - 做法有：`.pub` 搭配 `IdentitiesOnly`、`agent.toml` 排順序、Tabby 的 `.pub`、Termius 乾脆禁止「全部試一遍」。
   - 只有 OpenSSH 自己的 destination constraint，會在 agent 端依連線過濾 key 清單。
8. **會分辨「用途」的有兩家。** Bitwarden 和 Secretive 會解析 SSHSIG，區分「登入 server」和「簽 git commit」。
9. **刪除 key 的處理。**
   - 沒有任何產品記載「刪除一把還有 host 在用的 key」時要提示什麼。
   - agent 類的產品一刪就不再提供那把 key：1Password 只提供 active item；Bitwarden 在 vault 變更時換掉整個 keystore，也會拒絕已經不在 vault 裡的 key。
10. **系統 agent 的 confirm 功能有限制。**
    - Windows 內建的 agent 在 8.1 版忽略 `-c`、`-t`，9.5 版直接拒絕。
    - OpenSSH 的 confirm 視窗只顯示 key，不顯示目的地。

## 9. 對 SSHelter 的啟示（以下全是分析，不是來源內容）

1. **Windows 不要接管預設的 pipe，改學 Pageant。**
   - 每個使用者用自己的 pipe（名稱含使用者 SID 或雜湊）。建立時：
     - 設 `FILE_FLAG_FIRST_PIPE_INSTANCE`；
     - DACL 只開放給自己；
     - 加上 `PIPE_REJECT_REMOTE_CLIENTS`；
     - 用 client token 確認連進來的是同一個使用者。
   - 在 SSHelter 管理的 host 區塊裡寫 `IdentityAgent "//./pipe/…"`（用斜線）。
   - 好處：
     - 不必停用系統服務，也不用管理員權限；
     - 不會和 KeePassXC、1Password、Bitwarden 搶 pipe；
     - 不會讓使用者原本存在 Windows agent registry 裡的 key 突然消失（agent 協定本來就無法把 key 匯出，SSHelter 也匯入不了這些 key）。
   - SP3 spec 否決 agent 的理由之一是「會在 Windows 搶 pipe」，照這個做法就不成立了。
   - 限制：只有 Windows 內建的 `ssh.exe` 懂 named pipe，Git for Windows 附的 ssh 不懂，所以 SSHelter 啟動 ssh 時要用 System32 那一個。
2. **host 用哪把 key，由設定決定；agent 再用 session-bind 補強。**
   - SSHelter 本來就會寫 `IdentityFile <slot .pub>`。再加上 `IdentitiesOnly yes` 和 `IdentityAgent`，就不會撞到 MaxAuthTries，也不會把使用者其他 agent 裡的 key 混進來。
   - 因為 session-bind 會在「要 key 清單」之前送到，agent 可以只回傳綁定這台 host 的 key（OpenSSH 對 destination-constrained key 就是這樣做的）。前提是 SSHelter 記得每台 host 的 host key。
   - 舊的 client（PuTTY、Go、8.9 之前的 OpenSSH）沒有 session-bind，就只能靠設定。
3. **核准以「key × 目的 host × 本機或轉送」為單位記住，請求者另外顯示。**
   - Bitwarden 的 cache key 可以直接參考。
   - 提示框要顯示：
     - 誰：從 peer PID 往上找 GUI App 或 terminal，可以附上程式碼簽章；
     - 哪把 key；
     - `user@alias`：username 從被簽資料取，host key 對應到 SSHelter 的 host；
     - 用途：從 SSHSIG namespace 判斷；
     - 是不是轉送。
   - 收不到 session-bind 時，就退回「每次都問」。
   - 「預設每次都問」有 Bitwarden 這個前例。可以再提供「這台 host 記到鎖定或 N 分鐘」。
4. **MCP `run` 用一次性的 agent 端點。**
   - SSHelter 自己啟動 ssh，所以可以替每個已經核准的指令開一個臨時的 socket 或 pipe（`-o IdentityAgent=…`）。這個端點只替預期的 host key 自動簽章，指令結束就關掉。
   - 結果：MCP 只需要問一次，也就是現有的指令核准。其他 process（包括自己去跑 `ssh` 的 AI 工具）走一般端點，照樣會跳出核准。
   - OpenSSH 否決「多個 socket」的原因是要手動設定，但 SSHelter 會自動產生設定，所以這個缺點不存在。
5. **處理鎖定和沒開的情況。**
   - socket 一直保持存在；list public key 不需要解鎖；簽章時才要求解鎖。
   - agent 的核准發生在認證進行中，所以解鎖加核准的逾時要比 server 的 `LoginGraceTime` 120 秒短（Bitwarden 用 60 秒）。SSHelter MCP 現有的指令核准等 120 秒，但那是在 ssh 啟動之前，不受這個限制；agent 的核准若沿用 120 秒，就會和 `LoginGraceTime` 一樣長，server 可能先斷線。
   - SSHelter 沒開時，寫了 `IdentityAgent` 的 host 不會退回系統的 agent。照 OpenSSH 原始碼，如果 server 接受那個公鑰，ssh 還會試著從 `.pub` 的路徑載入 private key 並印出 `Load key` 錯誤，最後改用其他認證方式。
   - 介面上要說明「SSHelter 必須開著」，並提供開機自動啟動、常駐選單列。
6. **協定要做到的事。**
   - RSA SHA-2 flag 是必須的。
   - 驗證 session-bind 的簽章時，要支援 ECDSA host key（Bitwarden 還沒做）。
   - 不認得的 extension 回空的 failure。
   - 拒絕透過 ssh-add 加入或移除 key（1Password 和 Bitwarden 都不支援）。
   - Unix socket 設 0600，並檢查 peer 的 euid。
   - 如果真的要接受 constraint，就照 RFC 的 fail-safe 規則來。
7. **Keychain 的介面。**
   - 做成獨立的頂層區塊（Termius 正是把它從設定裡搬出來）。
   - 畫面是清單加詳細面板，面板顯示：類型、fingerprint、複製 public key、「被 N 台 host 使用」、Export to host（SSHelter 已經有 deploy）、每台電腦的狀態。
   - 可以考慮做 Identity 物件。
   - 做一頁活動紀錄（時間、App、host、key），對 MCP 和 AI 工具的情境特別有用。
   - 刪除時列出會受影響的 host。目前沒有產品記載這一點，正好補上。
8. **轉送。**
   - MCP 已經設了 `ForwardAgent=no`。
   - 使用者自己的 host 開了 ForwardAgent 的話，學 Bitwarden 顯示警告，並把核准和本機的分開記。
   - RFC 也要求 agent 對轉送連線多加控制。

## 10. 查不到或未記載的項目

- **Termius：**
  - RSA 和 ECDSA 的長度選項（官方文件沒有寫）。
  - 刪除一把還有 host 在用的 key 會發生什麼事。
  - 桌面版怎麼複製 public key、怎麼匯出 private key。
  - `Key Details` 有哪些欄位。
  - 內建 agent 是否對 local terminal 開放。
  - docs.termius.com 的頁面都沒有日期。
- **1Password：**
  - 辨識 process 和 terminal 分頁的實作方式。
  - 是否使用 session-bind，或在提示框顯示目的 host。
  - 核准提示有沒有逾時。
  - 鎖定時 list 會怎樣。
  - Windows 的系統服務佔著 pipe 時，會出現什麼錯誤。
- **Bitwarden：**
  - 怎麼匯出 private key（help 頁沒寫）。
  - help 頁和原始碼 README，對「第一次解鎖前收到 list 請求」的描述不一致：help 說可以 list，README 說所有請求都會先要求解鎖。
- **Windows：**
  - 各版 Windows 實際內建的 `ssh -V` 對照（Microsoft 沒有表格）。
  - 2024-10 的更新是不是真的把 Windows 10 的內建 client 升到 9.5：文章只說這些更新「和 OpenSSH 9.5.2.1 有關」。
- **RHEL 9：** 8.7p1 是否 backport 了 session-bind。
- **Secretive：** 開了 HashKnownHosts（known_hosts 裡的主機名稱被雜湊）時會顯示什麼。原始碼看起來是直接取第一欄，但沒有實測。
- **其他函式庫：** 只比對了預設分支的原始碼，沒有逐一確認各個 release 版本。
