# SSH 金鑰同步：同類產品怎麼設計（調查報告）

調查日期：2026-10-05。來源以官方文件、help center、release notes、官方 repo 的 README／原始碼為主，每項主張都附 URL。「未記載」表示在讀過的官方來源裡找不到，不做推測。引文保留英文原文，每個來源最多一句、少於 15 字。

**問題背景**：SSHelter 同步的 host 用 `IdentityFile` 路徑引用私鑰，但私鑰只在建立它的電腦上。要決定的是：同步過來的 host 用到沒同步的 key 時，預設要 (A) 每把 key 問一次、(B) 自動同步但可關閉，還是 (C) 只同步使用者明確挑選的 key。

---

## 1. 總表 A：會在使用者自己的裝置間同步 host 的產品

| 產品 | 私鑰存放位置 | **預設：私鑰在自己的裝置間會不會同步** | 使用者怎麼控制 | host 怎麼引用 key | 全新裝置會遇到什麼 | 官方給的理由或警告 |
|---|---|---|---|---|---|---|
| **Termius** | 匯入或產生到 Termius Keychain（存 key 內容，不存路徑），放在 Personal 或 Team vault，端對端加密 | **預設同步**。credentials 包括 usernames、passwords、SSH keys、identities | 全域開關 Settings > Account > `Sync keys and identities`，只管 Personal vault 的 credentials，hosts 照樣同步。Team vault 的 host 可以要求成員改用自己 Personal vault 的憑證。Biometric key 永遠不同步 | 在 host 上直接選 Keychain 裡的 key 或 identity 物件 | 登入後輸入 encryption password 解密，hosts 和 keys 一起到。關掉 credentials sync 時，其他裝置只收到 hosts，每台都要手動補 credentials | 有些企業規定 credentials 連加密形式都不能同步。關掉同步後一登出，本機 credentials 就被清掉，無法復原 |
| **1Password SSH agent** | vault 裡的 SSH Key item，端對端加密，私鑰不會變成磁碟上的檔案 | **預設同步**（vault item 跟著帳號到每台裝置）。但 agent 預設只提供 Personal／Private／Employee vault 裡的 key | 每台桌面 App 都要在 Settings > Developer 開「Use the SSH agent」。本機 `agent.toml`（不同步）依 vault／item／account 篩選並排序。使用時會跳授權提示，預設是每把 key 對每個新應用程式各問一次，核准維持到 1Password 鎖定 | `~/.ssh/config` 的 `IdentityFile` 指向從 1Password 下載的 **.pub**，再加 `IdentitiesOnly yes`。或用 SSH Bookmarks（預設關閉）自動產生 `~/.ssh/1Password/config` | 第一次登入要 Secret Key 加帳號密碼，或用已登入裝置的「Set Up Another Device」QR code。key 會自己出現，但仍要在這台開 agent、設 `IdentityAgent`，`agent.toml` 要自己搬 | 私鑰不離開 1Password，沒經過同意不會被使用。授權以「哪個 process 要用哪把 key」為單位 |
| **Bitwarden SSH agent** | vault 裡的 SSH key item。各種客戶端都看得到，但只有桌面 App 能當 agent | **預設同步**（vault 跨裝置） | 桌面 Settings > `Enable SSH agent`。授權設定 `Ask for authorization when using SSH agent` 有 Always／Never／Remember until vault is locked 三個選項（2025.5.0 起），預設值文件沒寫。**沒有逐把 key 的開關**。2025.9.0 起可把 SSH key 放進組織 collection | agent 會把 vault 裡的 key 一把一把試，無法依 host 指定。官方建議的變通是在 ssh config 用 `IdentityFile` | 登入桌面 App，vault 同步下來後，啟用 agent，再讓作業系統或 SSH client 改用它的 socket | key 太多可能導致驗證失敗。和原生 agent 並存時可能悄悄改用原生 agent，建議把本機 key 移出 `~/.ssh`。要共用到組織前先考慮改用每人各自的 key |
| **Tabby** | 加 key 時，如果已經設定 Vault，會跳出「Select file storage」讓你選：Filesystem（存 `file://` 路徑）或 Vault（把 key 檔內容存進加密 vault，用 `vault://id` 引用） | 路徑型 key **只同步路徑**。Vault 型 key 在 config sync 開啟後**預設同步**（`Sync Vault` 預設開） | **每把 key 加入時選存放位置**。Config sync > Advanced 有 `Sync Vault` 全域開關；整份 config 被 vault 加密時不能部分同步。config sync 需要 Tabby Web 和 token，自動同步預設關閉 | SSH profile 的 `privateKeys` 清單，內容是 `file://` 或 `vault://` | 填入 sync host 和 token 後下載 config。vault 要用 master passphrase 解鎖。`file://` 路徑在新機器上不存在就失效，官方 README 收錄的社群外掛 ssh-keymap 專門處理這件事 | 官方沒寫理由。上述外掛把「名稱 → 路徑」對照檔設計成永遠不同步 |
| **Panic Prompt 3（Panic Sync）** | Prompt 內的 Keys。Secure Enclave key 另外處理 | **預設同步**：開了 Panic Sync 就同步 Servers、Passwords、Keys、Clips | **不能選**：官方說是 all-or-nothing。key 的 passphrase 不同步，Secure Enclave key 也不同步 | 在 server 設定裡選 key | 登入 Panic Sync 後 key 會到，但每台都要重新輸入 passphrase。Secure Enclave key 要在新裝置重新產生，再把 public key 加到各主機 | 「基於安全考量」不同步 passphrase。端對端加密 |
| **Royal TS / TSX** | Credential 的 Private Key File 欄位：`PrivateKeyMode` 0 是路徑（預設），1 是 Embedded（key 內容嵌進文件） | **預設不同步私鑰**（預設存路徑；沒有內建雲端同步，靠把文件放在共享資料夾或 Dropbox） | 每個 credential 自己選路徑或嵌入。可以「依名稱指派 credential」，讓每個人在自己的個人文件裡放同名 credential | connection 引用 credential（直接指定、從上層繼承，或依名稱） | 開啟同步來的文件。嵌入的 key 直接能用；路徑型要在同一路徑放好 key；依名稱引用的要在個人文件建同名 credential | 依名稱指派可以讓共享文件不含個人憑證。建立含密碼的 credential 時，強烈建議替文件加密 |
| **Blink Shell (iOS)** | iOS Keychain（由 Secure Enclave 加密）、Secure Enclave key、WebAuthn passkey、實體 security key | Hosts 透過 iCloud 同步。一般私鑰會不會同步：**未記載**。passkey 透過 iCloud Keychain 同步。Secure Enclave key 和實體 key 無法匯出 | 未記載 | host 依 key 名稱引用（預設 `id_rsa`） | hosts 經 iCloud 送到；key 要怎麼處理未記載 | 同步 host 時不存密碼這類關鍵資料 |
| **XPipe** | key 檔用路徑引用，可選擇加入 git vault（加密後 commit） | **預設不同步**，每個 key 檔都要明確加入並確認 | 檔案選擇器旁的 git 按鈕，按下後跳確認視窗（**逐把 key**）。identity 分 local（不能同步）和 synced（裡面有 key 檔時，key 檔也必須同步）。連線分類也要明確標記才會同步 | connection 引用 identity 或 key 檔路徑 | clone repo 後提供 vault passphrase 或金鑰。別人的 personal identity 會顯示為 Unknown | key 檔一律要確認，不會自動加入。repo 必須設為 private |

## 2. 總表 B：其他找得到資料的產品（重點：是否刻意排除私鑰）

| 產品 | 預設 | 控制方式與重點 |
|---|---|---|
| **Devolutions Remote Desktop Manager** | 每個 entry 自己選 | 私鑰來源有三種：File (local)、Embedded data（存進 RDM，可共用）、My personal SSH key（每個使用者自己的 key） |
| **NetShell (iOS)** | **自動**經 iCloud Keychain 同步私鑰，App 內沒有設定 | known_hosts **刻意不同步**，因為信任主機是每台裝置自己做的決定 |
| **Secure ShellFish** | servers 經 iCloud Keychain 同步。匯入的 key 是否同步：**未記載**。Secure Enclave key 綁定裝置、不能匯出 | 「Install Key」工具能一次把 public key 裝到多台主機；主機只收密碼時，可以請另一台裝置協助 |
| **Xshell** | 沒有帳號同步。整個 User Data Folder（含 sessions 和 host／user keys）可以改放到雲端硬碟 | Master Password 加密 session 裡的密碼和 user key passphrase。匯出的 session 要兩台設同一組 Master Password，裡面的密碼才能用 |
| **SecureCRT** | 沒有內建同步，靠 Export/Import Settings（7.3 以後）或複製設定資料夾 | 使用 personal data folder 時，帳密和自動登入資訊不會被匯出。跨平台時，identity 等本機路徑可能要改 |
| **VS Code Settings Sync** | 同步項目清單裡沒有 SSH config 或 keys | machine 範圍的設定不同步，登入資訊存在作業系統 keychain。不算刻意排除，而是本來就不在範圍內 |
| **JetBrains Backup and Sync** | 文件列出的同步類別沒有 SSH configurations 或 keys（**未記載**） | — |
| **WindTerm** | 只存路徑，沒有官方同步文件 | 社群提案 #576（2022 年，仍開著）：設定可以同步，但 key 必須在新電腦明確匯入，並改用 fingerprint 而不是路徑來選 key。這不是官方決策 |

## 3. 總表 C：「金鑰不離開裝置」陣營

| 方案 | key 在哪裡 | 新裝置怎麼開始用 |
|---|---|---|
| **Secretive (macOS)** | Secure Enclave 或 smart card，設計上不能匯出、不能備份 | 替新 Mac 產生一組新 key，再把 public key 加到伺服器。可設定使用前要 Touch ID 或 Apple Watch，使用時會通知 |
| **FIDO2 實體金鑰（OpenSSH 8.2 起的 `-sk` key）** | 私鑰在 token 裡，磁碟上的檔案只放 key handle | non-resident key 要把 handle 檔搬過去（Termius 會替你同步 handle）。resident key 用 `ssh-keygen -K` 或 `ssh-add -K` 直接從 token 取回 |
| **Termius SSH.ID** | 每台登入的裝置自動產生一把綁定該裝置的 key | public key 集中發佈在 `sshid.io/<handle>`，用 `curl … >> authorized_keys` 裝到伺服器。登出就失效；要換 key 就登出再登入 |
| **Termius biometric／Prompt、ShellFish、Blink 的 Secure Enclave key** | 平台的安全硬體（Secure Enclave、TPM、Android Keystore） | 每台各自產生 key，各自把 public key 裝到主機 |
| **Tailscale SSH** | 不用 SSH key，靠 tailnet 身分加上自動產生的 WireGuard node key | 新裝置登入 tailnet，再由 ACL／grants 授權。check mode 會要求定期重新驗證身分（預設 12 小時） |
| **Teleport** | 短效 SSH 憑證（預設 TTL 12 小時），存在 `~/.tsh` 和 agent | 安裝 `tsh`，執行 `tsh login`（SSO）即可 |

---

## 4. 各產品筆記與來源

### 4.1 Termius
- **預設同步，可用全域開關關閉**。credentials 指 usernames、passwords、SSH keys、identities。關閉的位置是 Settings > Account > `Sync keys and identities`，**只影響 Personal vault**，hosts、groups、snippets 照樣同步；有帳號就無法停止所有同步。關閉後其他裝置收不到 credentials，要每台手動補。登出會清掉本機資料；同步關閉時連 credentials 一起清掉，Termius 無法幫忙救回。免費的 Starter 方案本來就不同步。官方說關閉同步是為了這類企業需求："strict requirements for credentials to be kept locally and never synced"。開關是以裝置還是帳號為單位沒有寫清楚；Team vault 那段說每個成員都要各自關閉。https://docs.termius.com/keychain/sync-of-keys-and-passwords
- **Team vault**：host 可以選擇和團隊共用憑證，或要求成員改用自己 Personal vault 的憑證（同上頁）。identity 必須和 host 在同一個 vault 才選得到。https://docs.termius.com/keychain/identities
- **key 怎麼存、host 怎麼引用**：用貼上或從檔案匯入，存的是 key 內容；host 在 Credentials 區塊直接選「SSH ID, Key, Certificate, FIDO2」。
  - Biometric key（Secure Enclave、Windows Hello／TPM、Android Keystore）"they are not synchronized"。
  - FIDO2 是 non-resident key，App 會跨裝置同步 handle，但每次連線都要插實體金鑰。
  - 來源：https://docs.termius.com/keychain/ssh-keys-and-certificates 。Biometric key 部落格文章（2020-03-26）：https://termius.com/blog/ssh-to-a-server-with-face-id-or-touch-id
- **新裝置**：要輸入 encryption password 才能解密 Personal vault，這組密碼不會上傳、也不會跟資料存在一起；裝置上的金鑰放在作業系統的 keychain／Keystore。https://docs.termius.com/security/encryption-overview
- **SSH.ID**：每台登入的裝置自動加入你的 SSH.ID 並產生自己的 key，私鑰不離開裝置；public key 用 `curl -fs https://sshid.io/<handle> >> ~/.ssh/authorized_keys` 裝到伺服器；登出 key 就失效，也能從 Devices 頁遠端登出。https://docs.termius.com/ssh-id-passkeys-for-ssh/what-is-ssh-id 、https://docs.termius.com/ssh-id-passkeys-for-ssh/setup-and-usage
- 版本或日期：文件頁沒有標示。

### 4.2 1Password SSH agent
- **預設的 key 範圍**：agent 會提供內建 Personal、Private 或 Employee vault 裡每一把符合條件的 key；共用或自訂 vault 要用 `agent.toml` 明確加入。https://www.1password.dev/ssh/agent
- **agent.toml 只在本機**：不會同步到 1Password 伺服器，官方建議需要的話自己用 Git 之類同步，或替每台機器寫不同設定（"isn't synced to the 1Password servers"）。另外，這個檔案在磁碟上沒有加密，所以官方建議用 ID 代替名稱。https://www.1password.dev/ssh/agent/config
- **開啟方式**：在桌面 App 的 Settings > Developer 開「Use the SSH agent」，再在 `~/.ssh/config` 設 `IdentityAgent`；Windows 不必額外設定。https://www.1password.dev/ssh/get-started
- **授權**：預設每把 key 對每個新應用程式各問一次，核准維持到 1Password 鎖定。可以改成每個 terminal session 或每次請求都問，記住時間可選到鎖定、到結束 App，或 4／12／24 小時。https://www.1password.dev/ssh/agent/authorization 。安全模型的說法是私鑰不離開 1Password、不存成本機檔案，"never used without your consent"。https://www.1password.dev/ssh/agent/security
- **host 對應 key**：`IdentityFile` 指向從 1Password 下載的 public key，加上 `IdentitiesOnly yes`；官方提醒有些 SSH client 不支援在 `IdentityFile` 放 public key。https://www.1password.dev/ssh/agent/advanced
- **SSH Bookmarks（beta，預設關閉）**：
  - 開關在 Settings > Developer > Advanced 的「Generate SSH config files with bookmarked hosts」。
  - 開啟後產生 `~/.ssh/1Password/config`，裡面是 `Match Host` 區塊，另外每個 bookmark 有一個用 fingerprint 命名的 `.pub`；使用者在 `~/.ssh/config` 加 `Include` 引入。
  - 磁碟上只有未加密的 host URL 和 public key，私鑰仍留在 1Password。
  - bookmark 是 SSH Key item 上的 `ssh://` 欄位。
  - 來源：https://www.1password.dev/ssh/bookmarks
- **新裝置**：vault 變更會自動出現在每台裝置（support 頁 2026-08-25）。第一次在新裝置登入要 Secret Key，或從已登入的裝置用「Set Up Another Device」（2026-08-28）。https://support.1password.com/sync/ 、https://support.1password.com/secret-key/
- 未記載：「Use the SSH agent」這個設定本身會不會跨裝置同步；Bookmarks 產生的 config 在第二台電腦會怎樣。

### 4.3 Bitwarden SSH agent
- **存放**：SSH key item 有 key name、private key、public key、fingerprint 四個欄位，桌面 App、網頁、瀏覽器擴充、行動 App 都支援。產生 key 只能是 Ed25519；匯入要 OpenSSH 或 PKCS#8 格式，不支援 PuTTY 格式。https://bitwarden.com/help/about-ssh/
- **控制**：
  - 沒有 `ssh-add` 式的管理；agent 能用哪些 key，完全看 vault 裡存了什麼。
  - 沒有依 host 選 key 的機制（"no mechanism to specify which key should be used"），key 多了可能驗證失敗，變通方式是 `IdentityFile`。
  - 和原生 agent 並存時，若授權設定是 Never 或 Remember，介面看不出是哪個 agent 處理了請求，官方建議把本機 key 放到 `~/.ssh` 以外。
  - 共用到組織前，建議先參考 SSH key best practices，因為多數服務都支援每人各自的 key。
  - 來源同上頁。
- **開啟與鎖定狀態**：在桌面 Settings 開「Enable SSH agent」，再設定授權。登出時 agent 不執行；鎖定時會先要求解鎖再授權。https://bitwarden.com/help/ssh-agent/
- **版本**（release notes）：
  - 2025.1.1：推出 SSH agent。
  - Desktop 2025.3.2：改善 agent forwarding。
  - Desktop 2025.5.0：新增 SSH approval settings。
  - 2025.9.0：可在組織 collection 存放和共用 SSH key。
  - 2026.7.0：agent 更新。
  - 2026.9.0：瀏覽器擴充可自動填入 SSH key 的 public key。
  - 來源：https://bitwarden.com/help/releasenotes/ 。三個授權選項來自官方 PR #13995（2025-05-05 merge）：https://github.com/bitwarden/clients/pull/13995 ；預設值沒有記載。

### 4.4 Tabby
- **加 key 的流程**：SSH profile 的「Add a private key」會呼叫 `selectAndStoreFile`。只有在 Vault 設好時，才會跳「Select file storage」讓你選 `Filesystem`（存 `file://` 路徑）或 `Vault`（把檔案內容 base64 存進 vault，回傳 `vault://<id>`）。
  - https://github.com/Eugeny/tabby/blob/master/tabby-ssh/src/components/sshProfileSettings.component.ts
  - https://github.com/Eugeny/tabby/blob/master/tabby-core/src/services/fileProviders.service.ts
  - https://github.com/Eugeny/tabby/blob/master/tabby-core/src/services/vault.service.ts
- **Config sync**：需要 Tabby Web 服務和 secret sync token。Advanced 分頁有 `Sync hotkeys`、`Sync window settings`、`Sync Vault` 三個開關，預設都開；`auto`（每分鐘自動同步）預設關。config 整份被 vault 加密時不能部分同步。上傳時沒勾選的部分會保留遠端版本，下載時保留本機版本。
  - https://github.com/Eugeny/tabby/blob/master/tabby-settings/src/components/configSyncSettingsTab.component.pug
  - https://github.com/Eugeny/tabby/blob/master/tabby-settings/src/config.ts
  - https://github.com/Eugeny/tabby/blob/master/tabby-settings/src/services/configSync.service.ts
- **Vault 介面**：Vault 是一個永遠加密的容器，用來放密碼、private key passphrase 和檔案；還有「Encrypt config file」選項。https://github.com/Eugeny/tabby/blob/master/tabby-settings/src/components/vaultSettingsTab.component.pug
- **路徑在別台電腦失效的問題**：官方 README 收錄社群外掛 ssh-keymap，讓同步的 profile 用名稱引用 key；每台機器有一份永遠不同步的對照檔，把名稱轉成本機路徑。https://github.com/Eugeny/tabby （README），外掛：https://github.com/mathys-lopinto/tabby-ssh-keymap
- 版本：讀的是 master 原始碼；最新 release 是 v1.0.237（2026-09-25）。

### 4.5 Panic Prompt 3／Panic Sync
- **同步範圍**：Prompt 會同步連線需要的一切，包括 Servers、Passwords、Keys、Clips。被問到能否只同步部分資料時，官方回答 "Panic Sync is all-or-nothing"，並說以後想改進。https://help.panic.com/prompt/prompt-sync/ （2025-07-23）
- **passphrase**：基於安全考量，Nova、Transmit、Prompt 裡跟 SSH key 一起存的 passphrase 目前不同步。Prompt 3 只和 Prompt 同步。https://help.panic.com/sync/sync-data-types/ （2025-07-30）
- **加密**：端對端加密。新裝置登入時，密碼經 PBKDF2 推導出金鑰，用它解開伺服器送來的 master keys，再存進裝置的 Keychain。https://help.panic.com/sync/panic-sync-common-qs/ （2025-07-30）
- **Secure Enclave key**：server 設定會同步，但 key 本身不會；新裝置要重新產生並把 public key 加到主機；同一時間只能有一把，移除後無法復原。https://help.panic.com/prompt/prompt-secure-enclave/ （2023-11-17）

### 4.6 Royal TS／TSX
- **Credential 頁**：Private Key File 分頁只有「路徑」和「passphrase」兩個欄位。依名稱指派 credential 時，共享文件不含個人憑證（"no credentials are stored with the connections"）；每個人在自己受保護的個人文件裡定義同名 credential 即可。含密碼時強烈建議加密並設密碼保護文件。https://docs.royalapps.com/r2023/royalts/reference/organization/credential.html （r2023，Royal TS V7）
- **嵌入模式**：`PrivateKeyMode` 0 是 Path to file（預設），1 是 Embedded；`PrivateKeyContent` 存嵌入的內容。https://docs.royalapps.com/r2023/scripting/objects/organization/royalcredential.html
- **功能清單**：支援把私鑰檔嵌入文件、從文件匯出；每個使用者可以為共享連線指定自己的 credentials；多人編輯與合併同步不需要資料庫，只要把檔案放在共享資料夾或 Dropbox。https://www.royalapps.com/ts/win/features-all
- **TSX**：文件格式和 Royal TS 等各平台互通，同樣支援 Dropbox 同步與依名稱指派 credential。https://www.royalapps.com/ts/mac/features

### 4.7 Blink Shell
- **一般 key**：私鑰存在 iOS Keychain，內容由 Secure Enclave 加密；Secure Enclave key 無法被任何 App 或人取出。https://docs.blink.sh/basics/ssh-keys
- **host 引用 key**：依名稱，預設用 `id_rsa`；密碼若有存，放在 Secure Enclave。https://docs.blink.sh/basics/hosts
- **iCloud 同步 Hosts**：從 v3.021.2 開始（"No critical data like passwords is saved."）。https://github.com/blinksh/blink/blob/main/CHANGELOG.md 。App Store 描述也只寫了 host 透過 iCloud 同步。https://apps.apple.com/us/app/blink-shell-build-code/id1594898306
- **WebAuthn**：passkey 存在 iCloud Keychain，會同步到其他 Apple 裝置；實體 security key 的私鑰永遠取不出；伺服器需要 OpenSSH 8.2 以上。https://docs.blink.sh/advanced/webauthn
- 未記載：一般私鑰會不會透過 iCloud 同步。2017 年有使用者在 issue #235 提到同步私鑰有安全上的困難，沒有官方回覆。https://github.com/blinksh/blink/issues/235

### 4.8 XPipe
- **key 檔同步**：開啟 git sync 後，key 檔旁會多一個同步按鈕，按下會跳確認。key 檔 "not added automatically to a repository and always require confirmation"。https://docs.xpipe.io/guide/ssh-auth
- **git vault**：加入的檔案以加密形式 commit。**連線本身也預設不同步**：一開始沒有任何連線分類會同步，官方說這樣是為了讓使用者明確控制要同步哪些連線，所以遠端 repo 起初是空的，要把分類的「Sync with git repository」設成 Yes 才會同步。新安裝要 clone repo 並提供 vault passphrase 或金鑰；repo 必須是 private。https://docs.xpipe.io/guide/sync
- **安全文件的說法**：私鑰也可以完全放在 XPipe 外面，只要執行時讀得到就好；把 key 檔同步，是為了不必擔心其他系統少了 key 檔。https://docs.xpipe.io/reference/security
- **identity**：local identity 不能同步。synced identity 裡如果有 key 檔，key 檔也必須同步。team vault 裡分 personal 和 global identity，別人看不到的會顯示為 Unknown。可以把 local identity 轉成 synced。https://docs.xpipe.io/guide/identities
- **加密**：AES-128-GCM，用 Argon2 推導金鑰；金鑰可以是自動產生的 vault key 檔，或自訂 passphrase。https://docs.xpipe.io/reference/security
- 版本：docs 是 master 版本；GitHub 最新 release 為 24.5（2026-10-04）。

### 4.9 其他
- **Devolutions RDM**：SSH key entry 的私鑰來源有 File (local)、Embedded data、My personal SSH key。https://docs.devolutions.net/rdm/knowledge-base/knowledge-base-articles/entry-settings/ssh-key
- **NetShell**：
  - 私鑰透過 iCloud Keychain 端對端加密同步，App 內沒有設定、也不用建帳號。
  - 在同一 iCloud 帳號下重裝 App，key 會自動回來。
  - known_hosts 刻意只留在本機（"trust-on-first-use is a decision tied to the device that made it"）。
  - 來源：https://netshellssh.com/docs/key-sync
- **Secure ShellFish**：
  - servers 在有 iCloud Keychain 時跨裝置共用。https://secureshellfish.app/privacy.html
  - Secure Enclave key 綁定裝置、無法取出；也支援 YubiKey 和短效憑證。
  - Install Key 一次把 key 加到多台主機的 `authorized_keys`，主機只收密碼時可以請另一台裝置協助。
  - 來源：https://secureshellfish.app/help/ssh-keys
- **Xshell**：
  - User Data Folder 放 session、log、host／user keys 和各種設定檔；要同步到雲端硬碟時可以改這個資料夾的位置（支援文章，2025-12-31 更新）。https://netsarang.atlassian.net/wiki/spaces/ENSUP/pages/177471571/Changing+the+User+Data+Folder+-+Xshell
  - Master Password、把 session 複製到另一台的流程：見 Xshell 8 手冊。https://cdn.netsarang.net/docs/Xshell8_manual.pdf
- **SecureCRT**：FAQ F014 說明，使用 personal data folder 時，帳密和自動登入資訊不會被複製；跨平台時本機資料夾位置（包括 identity）可能要改。https://www.vandyke.com/products/securecrt/faq/index.html
- **VS Code Settings Sync**：同步的是 settings、快捷鍵、snippets、tasks、MCP servers、UI state、extensions、profiles、prompts；machine 和 machine-overridable 範圍的設定不同步。（2026-09-30）https://code.visualstudio.com/docs/configure/settings-sync
- **JetBrains Backup and Sync**：類別清單沒有 SSH configurations 或 keys，而且官方說清單不完整。（IntelliJ IDEA 2026.2，2026-08-17）https://www.jetbrains.com/help/idea/sharing-your-ide-settings.html
- **WindTerm**：社群提案 issue #576（2022-03-04，仍開著）。https://github.com/kingToolbox/WindTerm/issues/576

### 4.10 「金鑰不離開裝置」陣營
- **Secretive**：Secure Enclave 裡的 key 不能匯出、不能備份；新 Mac 的做法是 "just create a new set of secrets specific to that Mac"。https://github.com/maxgoedjen/secretive
- **OpenSSH 8.2**（2020-02-14）：新增 `ecdsa-sk` 和 `ed25519-sk`。私鑰檔裡只有 key handle。resident key 可以在新機器用 `ssh-keygen -K` 或 `ssh-add -K` 取回。https://www.openssh.org/txt/release-8.2
- **Tailscale SSH**：用 tailnet 身分和 WireGuard key 取代 SSH key 的分發與管理；check mode 會要求重新驗證身分。（2026-01-05 驗證）https://tailscale.com/kb/1193/tailscale-ssh
- **Teleport**：`tsh login` 取得短效憑證，存到 `~/.tsh` 和 agent，預設 TTL 12 小時；新電腦只要安裝 tsh 再登入。（文件範例版本 18.11.2）https://goteleport.com/docs/connect-your-client/teleport-clients/tsh/

---

## 5. 觀察到的模式

1. **「key 歸誰保管」決定預設值。**
   - **vault 型**：key 已經被匯入 App 的加密 vault，包括 Termius、1Password、Bitwarden、Panic、NetShell、Tabby 的 Vault 選項。這類**預設同步**，靠端對端加密加上使用時授權（1Password、Bitwarden）來把關。關閉方式多半是全域開關（Termius、Tabby），甚至沒有（Panic、Bitwarden、NetShell）。
   - **路徑型**：引用磁碟上的 key 檔，包括 Royal 預設、Tabby Filesystem、XPipe、SecureCRT、Xshell、WindTerm、IDE。這類**預設不搬私鑰**，要使用者對單一 key 做明確動作：XPipe 按 git 按鈕再確認、Tabby 選 Vault、Royal 和 RDM 選 Embedded。
2. **在我讀過的文件裡，沒有任何產品在「收到同步來的 host、發現本機沒有它的 key」時跳出「問一次」作為預設流程。**最接近「每把 key 問一次」的是在**來源端**：Tabby 加 key 時問存放位置、XPipe 每個 key 檔要確認。
3. **常見解法是讓 host 引用一個「邏輯 key」，再由每台裝置自己解析：**
   - Royal：依名稱指派 credential。
   - Termius：Team vault 的 host 要求成員用自己的憑證。
   - RDM：My personal SSH key。
   - Tabby：ssh-keymap 外掛（`sshkey://name` 加上本機、不同步的對照檔）。
   - 1Password：`IdentityFile` 指向 `.pub`，私鑰由 agent 提供。
   - WindTerm：提案改用 fingerprint。
4. **綁定硬體的 key 一律不同步。**新裝置的處理方式是「每台產生自己的 key，再發佈 public key」：SSH.ID 的 URL、ShellFish 的 Install Key、Prompt 的 SE key、Secretive。或者改用身分系統，完全不用 key：Tailscale、Teleport。
5. **密碼、passphrase 和私鑰常常分開處理：**
   - Panic 同步 key，但不同步 passphrase。
   - Xshell 用 Master Password 保護 session 密碼。
   - SecureCRT 的 personal data folder 讓帳密不被匯出。
   - Blink 同步 host 時不存密碼。
6. **常見警告：**
   - 企業政策可能禁止同步（Termius）。
   - 關閉同步後登出會遺失 key（Termius）。
   - 和原生 agent 並存時可能悄悄 fallback（Bitwarden）。
   - key 太多會讓 SSH 驗證失敗，要用 IdentitiesOnly 或 IdentityFile 指定（1Password、Bitwarden）。
   - 同步用的 git repo 必須是 private（XPipe）。

## 6. 意外發現

- **Panic** 同步私鑰本體，卻刻意不同步 passphrase；而且完全不能選要同步哪些資料（all-or-nothing）。
- **1Password** 說私鑰「不離開 1Password」，但 key 其實隨 vault 到每台裝置。真正限縮使用範圍的是**每台機器各自**的 agent 開關、只存在本機且不同步的 `agent.toml`，以及使用時的授權提示。共用 vault 的 key 必須在每台機器另外加入。
- **Bitwarden** agent 會把 vault 裡所有 key 依序拿去試，沒有逐把開關，也不能依 host 選 key。
- **Termius** 關閉 credentials 同步後，一登出本機 key 就被清掉而且救不回來；「不同步」反而比「同步」更容易遺失 key。
- **XPipe** 是唯一規定每個 key 檔都「必須確認」才同步的產品，而且連 host（連線分類）都預設不同步。
- **NetShell** 自動同步私鑰，卻刻意不同步 known_hosts，因為信任主機是每台裝置自己的決定。
- **Termius SSH.ID** 和 **ShellFish Install Key** 把新裝置的設定變成「自動產生新 key，再批次把 public key 裝到主機」；ShellFish 還能請另一台已登入的裝置協助。
- **Tabby** 官方 README 收錄的外掛，正好是為了解決「同步的 profile 引用絕對路徑，到別台就壞」這個和 SSHelter 一模一樣的問題。

## 7. 對 SSHelter 的啟示（以下是我的分析，不是來源內容）

- SSHelter 用 `IdentityFile` 引用 `~/.ssh` 裡的檔案，key 原本不在 SSHelter 的 vault 裡，所以屬於**路徑型**。同類產品（XPipe、Tabby Filesystem、Royal、RDM）的預設都是**不搬私鑰、要逐把明確同意**，XPipe 甚至每次都要確認。「自動同步、可關閉」只出現在 key 一開始就由 App vault 保管的產品，那些使用者是主動把 key 交給 vault 的。
- 不論最後選哪個預設，以下做法都有前例：
  - **(a) 逐把 key 的明確同意**：XPipe 的確認視窗、Tabby 的存放位置選擇。
  - **(b) 讓同步的 host 帶著 key 的 fingerprint 或邏輯名稱**，讓沒有那把 key 的裝置可以改指定自己的 key，而不是留下一條壞掉的路徑：Royal 依名稱、Tabby ssh-keymap、WindTerm 提案、1Password 的 `.pub` 加 agent。
  - **(c) 新裝置提供「產生本機新 key，用已同步的 host 清單批次安裝 public key」**：SSH.ID、ShellFish。
  - **(d) 如果同步私鑰，passphrase 不要一起同步（Panic 的做法），並在介面上說明端對端加密。**
  - **(e) 硬體綁定的 key（Secure Enclave、FIDO）本來就不能同步**，介面要能把它和「沒選擇同步」區分開來。

## 8. 查不到或未記載的項目

- Termius：`Sync keys and identities` 是以裝置還是帳號為單位；文件頁沒有日期。
- 1Password：「Use the SSH agent」設定本身會不會同步；Bookmarks 產生的 config 在第二台電腦的行為。
- Bitwarden：授權設定的預設值（help 頁沒寫）。
- Blink、Secure ShellFish：一般匯入的私鑰會不會透過 iCloud 同步。
- JetBrains：SSH configurations 是否同步。
- Warp：找不到和 SSH key 同步有關的官方文件，因此未納入。
- 沒有找到任何產品記載「同步的 host 引用了缺少的 key 時，在接收端提示使用者」的流程。
