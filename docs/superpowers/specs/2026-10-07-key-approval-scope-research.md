# SSH 金鑰的「核准／保護」設定放在哪裡：每台裝置各自設定，還是跟著 vault 同步？

- 研究日期：2026-10-06。下列 URL 的存取日期都是 2026-10-06。
- GitHub 原始碼以 commit SHA permalink 引用：
  - bitwarden/clients `787fc24`
  - keepassxreboot/keepassxc `9e0f57a`
  - maxgoedjen/secretive `9edc879`
  - Eugeny/tabby `4004cc5`
- developer.1password.com 的 SSH 文件現在會 301 轉址到 www.1password.dev。這是 1Password 自家網域發出的轉址，下文引用轉址後的網址。
- 只有一處直接引文（1Password 員工的發言），其餘都是改寫。UI 標籤和程式識別字照原文列出，因為它們是名稱，不算引文。

## 一、總表

| 產品 | 設定 | 範圍 | 來源 |
|---|---|---|---|
| 1Password | Settings › Developer › SSH agent 進階設定：**Ask approval for each new**（application〔預設〕／application and terminal session／request），以及 **Remember key approval**（until 1Password locks〔預設〕／until quits／4、12、24 小時） | **per device**（app 偏好設定；官方文件沒有明講，是推論，見 §二.1）。已核准的 session 存在該機 agent 的記憶體 | 1P-1、1P-2、1P-6 |
| 1Password | 用來通過核准提示的驗證方式：Touch ID、Apple Watch、Windows Hello、帳號密碼（位於 Settings › Security） | **per device**（官方明言 unlock／auto-lock 設定不在裝置間同步） | 1P-1、1P-4、1P-5 |
| 1Password | 核准提示上的 **Approve for all applications** 勾選框 | 每次提示、每把 key 各自決定，只在該機的 agent session 期間有效 | 1P-1 |
| 1Password | `agent.toml`（決定哪些 key 交給 agent、提供順序） | **per device**，官方明言不同步 | 1P-3 |
| 1Password | 每把 key 各自的核准設定 | **不存在**，只有社群 feature request | 1P-7、1P-8 |
| 1Password | SSH Bookmarks（在 SSH Key item 的欄位裡放 `ssh://` URL） | per key、存在 vault → **會同步**（但這不是核准設定） | 1P-9 |
| Bitwarden | **Ask for authorization when using SSH agent**：Always〔預設〕／Never／Remember until vault is locked | **per device × per account**：desktop app 本機 disk state（`UserKeyDefinition`），不在 vault 裡 | BW-1、BW-2 |
| Bitwarden | **Enable SSH agent** | **per device**，同一台機器上所有帳號共用 | BW-1、BW-4 |
| Bitwarden | 選「Remember until vault is locked」後記住的核准 | 只在記憶體。以 SSH key item × 目的主機 host key fingerprint（或 local／forwarded）區分；vault 鎖定或切換帳號就清空 | BW-3 |
| KeePassXC | 每筆 entry 的 **Require user confirmation when this key is used**、**Remove key from agent after N seconds**、解鎖時加入／鎖定時移除 | **per key-in-database**：以 XML 附件 `KeeAgent.settings` 存在 entry 裡，跟著 .kdbx 檔到其他裝置 | KX-1、KX-2 |
| KeePassXC | Enable SSH Agent integration、Pageant／OpenSSH、`SSH_AUTH_SOCK` override | **per device**（本機 `keepassxc.ini`） | KX-3 |
| KeeAgent（KeePass 2 外掛，KeePassXC 沿用它的附件格式） | 全域 **Always require confirmation…**（覆蓋每筆 entry 的 confirm） | 全域選項：per installation。entry 選項：存在資料庫 | KX-4 |
| Secretive | 建立 key 時選保護等級：不需驗證／user presence（Touch ID、Apple Watch 或密碼）／只接受建立當下的生物辨識組合 | per key，建立時寫死在 Secure Enclave 的 access control（`ThisDeviceOnly`）→ **天生 device-bound**，事後不能改 | SE-1、SE-2 |
| Secretive | 通知上的「暫時免再驗證」：1 分／5 分／1 小時／24 小時 | per key、只在記憶體、該機有效 | SE-3 |
| Termius | Biometric keys（Secure Enclave、TPM＋Windows Hello、Android Keystore），每次使用都由 OS 要求生物辨識 | **device-bound**，不同步 | TM-1 |
| Termius | 產生 FIDO2 key 時的 **Require User Presence**／**Require PIN code** | per key，產生時決定、由硬體 token 執行；non-resident key 會同步 | TM-1 |
| Termius | Host 的 **Agent Forwarding** 開關 | per host、存在 vault → 會同步（但不是核准提示） | TM-3 |
| Tabby | 解鎖 vault 對話框的 **Remember for…**（不記住，或 1 分鐘～7 天） | **per device**：選項存在 renderer 的 `localStorage`；passphrase 只放記憶體；不在 config sync 的內容裡 | TB-1、TB-2 |
| OpenSSH | `ssh-add -c`／`-t`、`AddKeysToAgent confirm` | **per agent session**：加入 key 時附上 constraint，由正在執行的 agent 持有；`ssh_config` 是本機檔案 | OS-1、OS-2 |
| OpenSSH FIDO（`*-sk`） | `verify-required`／`no-touch-required` | per key（產生時寫入 key，由 token 和 sshd 執行） | OS-3 |
| GnuPG agent | `sshcontrol` 裡每把 key 的 `confirm` flag | per key，但存在本機檔案 | OT-6 |
| Apple／Windows／Android 平台 | Touch ID／Face ID、Windows Hello、Keystore 的使用者驗證 | 生物辨識資料和 Hello 憑證都是 **device-local**。passkey 本身可以同步，但授權一律在本機做 | OT-1 到 OT-5 |

## 二、各產品註記

### 1. 1Password SSH agent

- **1P-1** https://www.1password.dev/ssh/agent/authorization
  - 這兩個設定位於 Settings › Developer 的 SSH agent 進階設定。
  - 選 request 時沒有 remember 選項。
  - 核准提示使用畫面上顯示的方式（Touch ID、Windows Hello 或帳號密碼），實際方式取決於裝置、OS 版本和 1Password 設定；1Password 鎖定時要先解鎖。
  - 換句話說，**用什麼方式核准，取決於該台裝置的 unlock 設定**。
- **1P-2** https://www.1password.dev/ssh/agent/security
  - 核准後會在 key 和 process 之間建立 session。
  - 選固定時數時，核准會留在 agent 記憶體，即使 1Password 已鎖定也一樣，屆時只需要解鎖 1Password。
  - Local storage 段：eligible key 的公鑰（以及可選的 item 標題）以未加密方式存在本機磁碟。
- **1P-3** https://www.1password.dev/ssh/agent/config
  - `agent.toml` 存在本機，不同步到 1Password 伺服器。
  - 多台工作站請自行用 Git 之類的方式同步，或每台各用一份設定。
- **1P-4** https://support.1password.com/unlock-auto-lock/ （Published 2026-09-11）
  - 官方明言 unlock 和 auto-lock 設定不會在裝置間同步，每台可以各自設定。
  - 這是我找到唯一一則官方文件直接談「設定是否同步」。
- **1P-5** https://support.1password.com/touch-id-apple-watch-security-mac/
  - Touch ID 解鎖用的 secret 以 Secure Enclave 內的金鑰加密，整個流程在本機進行。
- **1P-6** https://www.1password.community/developers-69/ssh-feature-questions-19037
  - 1Password 員工 floris_1P（2022-02-18）說，新機器要在每台裝置的偏好設定裡各自開啟 SSH agent，因為這個設定是 "local (by design!)"。
  - 注意：他講的是「開啟 agent」這個開關。可設定的核准模型要到 2022-07 的 beta 才加入（同一員工 2022-07-20 在 https://www.1password.community/developers-69/1password-asking-for-permission-each-time-19323/index3.html 的回覆）。
- **1P-7** https://www.1password.community/developers-69/more-control-over-ssh-key-approval-settings-20381
  - 2024-06-17 有使用者要求 per vault／per key 的核准設定，因為目前是全域設定。沒有官方回覆。
- **1P-8** https://www.1password.community/developers-69/feature-request-per-application-authorization-policy-for-the-ssh-agent-25530
  - 2026-09-09 有使用者想把 AI coding agent 限制在「每次都要核准」，指出現在只有單一全域設定。
  - 1Password 員工 2026-10-05 回覆，已轉給團隊考慮。
- **1P-9** https://www.1password.dev/ssh/bookmarks 、https://www.1password.dev/ssh/manage-keys
  - Bookmarks 是在 SSH Key item 加上 `ssh://` 自訂欄位。
  - SSH Key item 本身只有私鑰、公鑰、指紋等欄位，**沒有核准相關欄位**。

### 2. Bitwarden SSH agent

- **BW-1** 原始碼 `apps/desktop/src/platform/services/desktop-settings.service.ts`
  - https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/apps/desktop/src/platform/services/desktop-settings.service.ts#L49-L60
  - https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/apps/desktop/src/platform/services/desktop-settings.service.ts#L109-L116
  - `SSH_AGENT_PROMPT_BEHAVIOR` 是 `UserKeyDefinition(DESKTOP_SETTINGS_DISK, "sshAgentRememberAuthorizations")`，透過 `getActive` 取值（per account），未設定時預設 `Always`。
  - `SSH_AGENT_ENABLED` 是 `KeyDefinition`，透過 `getGlobal` 取值（同機所有帳號共用）。
  - 選項 enum 在 https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/apps/desktop/src/autofill/models/ssh-agent-setting.ts
- **BW-2** state 儲存位置
  - `DESKTOP_SETTINGS_DISK` 是 `StateDefinition("desktopSettings", "disk")`：https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/libs/state/src/core/state-definitions.ts#L148
  - `disk` 的定義是 app 重啟後仍保留的 client 端 state：https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/libs/storage-core/src/storage-location.ts
  - 結論：這是**本機偏好設定**，不是 vault 或帳號層級的設定。
- **BW-3** `apps/desktop/src/autofill/services/ssh-agent.service.ts`
  - 記住的核准存在 `authorizedHosts: Map<cipherId, Set<…>>`（#L59）。
  - vault 不在 unlocked 狀態或切換帳號時清空（#L189-L224）。
  - 判斷邏輯在 `needsAuthorization`（#L524）。
  - https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/apps/desktop/src/autofill/services/ssh-agent.service.ts
  - 核准對話框只有 Authorize／Deny，**沒有生物辨識**：https://github.com/bitwarden/clients/blob/787fc24d30871463b29369ba2d68bad733d1675d/apps/desktop/src/autofill/components/approve-ssh-request.html
  - 生物辨識只用在解鎖 vault：vault 鎖住時，先要求解鎖，再要求授權。
- **BW-4** 官方說明文件
  - https://bitwarden.com/help/ssh-agent/ ：在 Settings 開啟 agent 並調整授權設定。
  - https://bitwarden.com/help/about-ssh/ ：Enable SSH Agent 是跨帳號的全域設定；SSH key item 只有 name、private key、public key、fingerprint。
  - https://bitwarden.com/help/biometrics/ ：生物辨識由 OS 原生 API 在本機驗證。

### 3. KeePassXC（確認你的假設：設定存在資料庫、跟著 .kdbx 走）

- **KX-1** `src/sshagent/KeeAgentSettings.cpp`
  - https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/src/sshagent/KeeAgentSettings.cpp#L290-L377
  - `toXml()` 寫出 `EntrySettings`，內容包含 `UseConfirmConstraintWhenAdding`、`UseLifetimeConstraintWhenAdding`、`LifetimeConstraintDuration`、`AddAtDatabaseOpen`、`RemoveAtDatabaseClose`。
  - `toEntry()` 把這段 XML 以附件 `KeeAgent.settings` 存進 entry；全部是預設值時就移除附件。
  - UI 標籤見 https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/src/gui/entry/EditEntryWidgetSSHAgent.ui
  - User guide 也說明 entry 用來存放 SSH Agent 設定和 key 檔：https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/docs/topics/SSHAgent.adoc#L162-L176
- **KX-2** 實際執行面（`src/sshagent/SSHAgent.cpp`）
  - https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/src/sshagent/SSHAgent.cpp#L284-L304
  - KeePassXC 把 key 加進**本機系統 agent** 時，附上 `SSH_AGENT_CONSTRAIN_CONFIRM` 和 `SSH_AGENT_CONSTRAIN_LIFETIME`。
  - 所以設定雖然跟著資料庫走，確認視窗卻由**每台機器自己的** agent 或 askpass 顯示。
- **KX-3** `src/core/Config.cpp`
  - https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/src/core/Config.cpp#L195-L200
  - https://github.com/keepassxreboot/keepassxc/blob/9e0f57a4a4c6c629fa6d0a593acb7d089b1d95cd/src/core/Config.cpp#L569-L613
  - `SSHAgent/Enabled`、`UseOpenSSH`、`UsePageant` 存在 keepassxc.ini；`AuthSockOverride` 存在 local ini。兩者都是本機設定檔。
  - KeePassXC 沒有全域的「一律確認」選項。
- **KX-4** https://keeagent.readthedocs.io/en/latest/usage/options.html
  - KeeAgent 文件明講：全域選項存在 KeePass 設定檔，只影響單一安裝；entry 選項存在資料庫檔。
  - 全域的 Always require confirmation 會覆蓋 entry 的 Use confirm constraint。

### 4. Secretive（Secure Enclave，天生 device-bound）

- **SE-1** `SecureEnclaveStore.swift` 和 `CreationOptions.swift`
  - https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/Sources/Packages/Sources/SecureEnclaveSecretKit/SecureEnclaveStore.swift#L92-L110
  - `notRequired`／`presenceRequired`／`biometryCurrent` 分別對應 `.privateKeyUsage`、`.userPresence`、`.biometryCurrentSet`，accessibility 為 `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`。
  - 原始碼註解說，記錄下來的驗證屬性只是建立時的描述，改它不會改變 key 的實際要求：https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/Sources/Packages/Sources/SecretKit/Types/CreationOptions.swift#L8
- **SE-2** README 和 FAQ
  - https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/README.md#L58-L60
  - https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/FAQ.md#L3-L5
  - key 無法匯出、備份或搬到新機；換新 Mac 就重新建一組 key。
- **SE-3** 暫時免再驗證
  - https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/Sources/SecretAgent/Notifier.swift#L26-L31
  - https://github.com/maxgoedjen/secretive/blob/9edc8799009a9d4fb30b45c13689d19e7c31a3c7/Sources/Packages/Sources/SecretAgentKit/AuthenticationHandler.swift#L26-L31
  - 同檔 #L98：以 per-secret 的 context 存在記憶體，用 monotonic clock 判斷過期，並設定 `LAContext.touchIDAuthenticationAllowableReuseDuration`。

### 5. Termius

- **TM-1** https://docs.termius.com/keychain/ssh-keys-and-certificates
  - Biometric keys 存在硬體隔離區，因為私鑰取不出來，所以不同步。
  - 每次使用，由 macOS／iOS、Windows、Android 跳出生物辨識。
  - FIDO2 key 產生時可選 Require User Presence／Require PIN code；Termius 產生的 non-resident FIDO2 key 會跨裝置同步。
- **TM-2** SSH ID
  - https://docs.termius.com/ssh-id-passkeys-for-ssh/what-is-ssh-id
  - https://docs.termius.com/ssh-id-passkeys-for-ssh/ssh-id-security
  - 每台裝置各自產生 device-bound key，只有公鑰會同步。
- **TM-3** Host 設定與同步
  - https://docs.termius.com/organize-and-connect-to-hosts/connecting-to-a-server ：Agent Forwarding 是 Host 詳細頁裡的開關，Termius 使用內建 agent。
  - https://docs.termius.com/getting-started/learn-about-vaults ：Vault 會同步 Hosts、Keys 等資料。
  - https://docs.termius.com/keychain/sync-of-keys-and-passwords ：可以關掉 Personal vault 的憑證同步。
- 沒找到任何 per-key 或 per-host 的「每次詢問」或「記住 N 分鐘」核准設定。

### 6. Tabby vault

- **TB-1** 解鎖對話框
  - https://github.com/Eugeny/tabby/blob/4004cc51e95574658c96270322df8234d9d253ab/tabby-core/src/components/unlockVaultModal.component.ts#L11-L33
  - https://github.com/Eugeny/tabby/blob/4004cc51e95574658c96270322df8234d9d253ab/tabby-core/src/components/unlockVaultModal.component.pug#L23
  - 可選 1、5、15、60、1440、10080 分鐘；選擇寫入 `window.localStorage.vaultRememberPassphraseFor`，預設 1 分鐘。
  - UI 說明 master passphrase 只放記憶體，關掉 Tabby 就要重新解鎖。
  - https://github.com/Eugeny/tabby/blob/4004cc51e95574658c96270322df8234d9d253ab/tabby-core/src/services/vault.service.ts#L178-L193 ：時間到時用 `setTimeout` 清掉 passphrase。
- **TB-2** Config sync
  - https://github.com/Eugeny/tabby/blob/4004cc51e95574658c96270322df8234d9d253ab/tabby-settings/src/services/configSync.service.ts#L18
  - https://github.com/Eugeny/tabby/blob/4004cc51e95574658c96270322df8234d9d253ab/tabby-settings/src/services/configSync.service.ts#L87-L153
  - 上傳的是 config YAML，去掉 `configSync` 區塊；`hotkeys`、`appearance`、`vault` 是可選的同步部分。
  - 加密後的 vault 可以同步，但「Remember for」的選擇不在 config 裡，所以**不會同步**。

### 7. OpenSSH（macOS `ssh-add -c`）

- **OS-1** `ssh-add`
  - 在本機 macOS 27.0.1（OpenSSH_10.3p1）查 man ssh-add：`-c` 表示每次使用前都要經 ssh-askpass 確認，`-t` 設定 lifetime。線上版：https://man.openbsd.org/ssh-add.1
  - 觀察：這台 Mac 的標準路徑上沒有 ssh-askpass。
- **OS-2** `ssh_config` 和 agent 協定
  - `AddKeysToAgent confirm` 會在 ssh 自動加入 key 時套用同樣的確認。它寫在本機 `ssh_config`，可以按 Host 設定：https://man.openbsd.org/ssh_config.5#AddKeysToAgent
  - 協定層：constraint 是加入 key 時附上的（`SSH_AGENT_CONSTRAIN_CONFIRM`／`_LIFETIME`），由持有 key 的 agent 執行。RFC 9987（2026-05）§5.2.7：https://www.rfc-editor.org/rfc/rfc9987
  - 所以這是 agent session 的屬性：agent 重啟就消失，也沒有任何同步機制。
- **OS-3** FIDO
  - `ssh-keygen -O verify-required`／`no-touch-required` 在產生 key 時寫入。
  - sshd 可以在 `authorized_keys` 加上對應選項，要求或放寬這兩項檢查（man ssh-keygen(1)、sshd(8)）。

### 8. 其他相關（通則：生物辨識只在本機）

- **OT-1** Apple
  - https://support.apple.com/en-us/105095 、https://support.apple.com/en-us/102381 ：Touch ID 和 Face ID 的資料不離開裝置，也不備份到 iCloud。
  - https://support.apple.com/en-us/102195 ：passkey 經 iCloud Keychain（端對端加密）同步，但每次使用都在該台裝置上用 Touch ID／Face ID 授權。
- **OT-2** https://developer.apple.com/documentation/security/restricting-keychain-item-accessibility
  - 屬性結尾是 `ThisDeviceOnly` 的 Keychain 項目，不會 migrate 到別台裝置。
- **OT-3** macOS 內建 Secure Enclave SSH key（本機 man sc_auth、ssh-keychain(8)）
  - `sc_auth create-ctk-identity -l <label> -k p-256-ne -t bio|none` 在建立時決定私鑰保護方式（`-ne` 表示不可匯出），再透過 `/usr/lib/ssh-keychain.dylib` 提供給 ssh。
- **OT-4** Windows Hello
  - https://learn.microsoft.com/windows/apps/develop/security/windows-hello ：每個 Hello 綁定特定使用者和裝置，不跨裝置同步。
  - https://learn.microsoft.com/windows/security/identity-protection/hello-for-business/faq ：PIN 綁定設定它的那台裝置，要在多台使用就得每台各自設定。
  - https://learn.microsoft.com/windows/security/identity-protection/hello-for-business/how-it-works ：生物辨識資料只存在本機，不會 roam。
- **OT-5** https://developer.android.com/privacy-and-security/keystore
  - key 的使用授權（包括「需要使用者驗證」）在產生或匯入時指定，之後不能改。
- **OT-6** https://www.gnupg.org/documentation/manuals/gnupg/Agent-Configuration.html
  - `sshcontrol` 每行 keygrip 後面可以加 TTL 和 `confirm` flag，`ssh-add -c` 會自動設定；這是本機檔案。
  - 文件也說 `sshcontrol` 已被 key 檔的 Use-for-ssh 屬性取代。

## 三、Pattern

**事實**（以上都有出處）

1. 全域的核准政策（「每次問」或「記住 N 時間」），在查到的每個產品裡都是 **app／裝置偏好設定**：
   - 1Password：Developer 設定；同一區的 agent 開關被員工稱為 local；unlock 設定官方明言不同步。
   - Bitwarden：本機 disk state。
   - Tabby：`localStorage`。
   - KeeAgent：全域選項只影響單一安裝。
2. 「已記住的核准」一律只存在該台機器上**正在執行的 agent 或 app 的記憶體**：1Password agent、Bitwarden `authorizedHosts`、Secretive context、Tabby passphrase、OpenSSH constraint 都是如此。
3. 會跟著 key 走的 per-key 保護只有兩種形式：
   - (a) 存在資料庫或 item 裡：KeePassXC／KeeAgent 的 `KeeAgent.settings`。1Password 只把 Bookmarks 的 host URL 放進 item，核准設定不放。
   - (b) 寫進 key 本身，由硬體或伺服器執行：FIDO2 的 user presence 和 `verify-required`，以及 Termius 的 FIDO2 選項。
   - (a) 雖然同步，confirm 仍由每台機器本機的 agent 執行。
4. 生物辨識這道關卡一律 **device-local**：
   - Touch ID／Face ID 資料、Windows Hello 憑證和 PIN、Secure Enclave／TPM／Keystore 裡的 key，都不會離開裝置。
   - 保護等級在 key 建立時就固定：Secretive、Android Keystore、`sc_auth` 都是如此。
   - Passkey 是「秘密同步、授權在本機」的典型例子。
5. 1Password 沒有 per-key 核准設定；社群在 2024 和 2026 年都有人提出要求。

**分析**（我的推論，不是來源原文）

- 實務上可以分成兩層：
  - 「這把 key 至少要什麼保護」：key 的屬性，可以隨 vault 同步。會隨 vault 同步的 per-key 核准設定只出現在 KeePassXC／KeeAgent；其他 per-key 保護都綁在 key 本身或硬體上（Secretive、FIDO2）。
  - 「這台電腦用什麼方式核准、記多久」：裝置政策，大家都做成 per device，並且不同步。
- 對 SSHelter 的意涵：
  - 如果希望設定跟著 key slot 走，比較一致的做法是只同步一個「最低要求」（例如 always ask 或 require user presence），各裝置再對應到自己的能力（Touch ID、Windows Hello、OS 密碼或確認視窗）。
  - 「remember for N hours」和「用 Touch ID」則保留在各裝置。
  - KeeAgent 的「本機全域下限＋per-key 設定」可以參考：本機下限只能讓保護更嚴，同步下來的 per-key 設定不能讓它變寬。
- 風險：
  - 同步過來的「require Touch ID」到了沒有生物辨識的裝置，需要明確的 fallback。1Password 的核准方式本來就會依裝置不同而不同（1P-1）；Termius 也說明各平台都不允許關掉生物辨識的 PIN／密碼 fallback（TM-2）。
  - 同步的放寬設定，可能讓某台裝置上的變更悄悄削弱其他裝置的保護。

## 四、找不到或未驗證的部分

- **1Password Developer 設定是否同步**：官方文件沒有明講 **Ask approval for each new** 和 **Remember key approval** 會不會同步。
  - 「per device」是根據三點推得：員工 2022 年說 agent 開關是 local；官方說 unlock 設定不同步；核准存在 agent 記憶體。
  - 沒有用兩台裝置實測。
- **1Password per-key 核准設定**：1Password 是閉源軟體，「不存在」只能從文件和社群請求推定。
- **1Password Business policy**：沒找到能否用 policy 強制 SSH 核准設定。Developer permissions policy 沒有列出細項，MDM 文件也沒有 SSH 相關的 key。
- **Bitwarden**：help 頁沒有用文字說明這個設定的範圍，結論來自原始碼。我沒有窮舉是否有其他路徑會把 desktop 設定上傳到伺服器。
- **Termius**：沒找到 per-key／per-host 的核准或記憶設定，也沒查到 app lock（PIN 或生物辨識鎖）設定的範圍。
- **Tabby**：沒有官方的 vault 文件，結論只依據 master 分支的原始碼。
- **GnuPG**：新式 key 檔的 Use-for-ssh 屬性是否也帶 `confirm`，因為無法取得 `keyformat.txt` 而未確認。
- **其他 agent**：Strongbox、Proton Pass 等 SSH agent 沒查到相關文件。
- **macOS 缺 ssh-askpass**：沒找到官方說明 `ssh-add -c` 在沒有 ssh-askpass 時會怎樣；只觀察到本機標準路徑上沒有 ssh-askpass。
