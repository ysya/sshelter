# SSHelter SP3 — 金鑰插槽與金鑰同步(設計)

日期:2026-10-05。前置:Sync v2 SP1(`2026-10-02-sync-v2-spaces-design.md`,0.17.0-3 起以 beta 發佈)。
同類產品的調查與 OpenSSH 行為的查證見 §13。

## 0. 定位

同步的主機用 `IdentityFile` 指到「寫下它的那台電腦」上的金鑰路徑。另一台電腦上那個路徑不存在,或是另一把金鑰,
主機就連不上。使用者的電腦通常各有不同檔名的金鑰(例如 Mac 的 `id_mac`、Windows 的 `id_win`)。

SP1 的 spec 原本把 SP3 定為「每台裝置一把金鑰 + 自動部署/撤銷公鑰,或 SSHelter 當 SSH agent」,並把私鑰同步留給
SP4。本文取代那個規劃:

- **自動部署/撤銷公鑰:否決。** 等於工具替使用者決定金鑰的管理方式,還要改每台伺服器的 `authorized_keys`。
- **SSHelter 當 SSH agent:否決。** 工程量最大;ssh 要靠 app 執行中才能用;Windows 上會和內建的 OpenSSH agent 搶同一個
  pipe。
- **私鑰同步從 SP4 提前到本文**,改成每把金鑰由使用者在它所在的電腦上決定。SP4 只剩「把 space 分享給別人」。

2026-10-05 定下的路線(§14):本文(金鑰插槽)→ SSHelter 自己的 SSH agent → 內建終端機(可選)。後兩者各自另寫 spec;
本文的資料格式與插槽路徑要讓 agent 階段直接沿用。

## 1. 目標與非目標

**目標**
- 同步的主機在每台電腦上都能連線,而主機的設定文字在每台電腦上相同。
- 一把私鑰要不要離開它所在的電腦,由使用者在那台電腦上、在主機開始同步的當下決定,每把金鑰只問一次。
- 選擇同步的金鑰:新電腦加入、勾選 space 之後就能直接連線,不用改伺服器。
- 選擇不同步的金鑰:其他電腦各自指定一次本機的金鑰,之後不再問;伺服器不動。
- 私鑰永遠不會被遠端刪除。

**非目標**
- 修改伺服器(部署或撤銷公鑰)。
- SSHelter 當 SSH agent。
- 同步 passphrase。
- 插槽建立之後改名。
- `CertificateFile`(SSH 憑證)、PKCS#11,以及 `IdentityFile` 指向 `.pub`、私鑰由 agent 提供的用法。
- 自己電腦之間的加密邊界(同 SP1:同一帳戶的每台電腦都能解開帳戶裡的所有記錄)。

## 2. 已定案的決策

| 決策 | 選擇 | 理由 / 被否決的選項 |
|---|---|---|
| 主機怎麼指到金鑰 | **金鑰插槽**:`IdentityFile ~/.ssh/sshelter/keys/<name>-<id8>`;插槽裡放什麼由每台電腦各自決定 | 否決「保留原路徑,在每台另寫一份設定覆寫」:設定裡不存在的 `IdentityFile`,OpenSSH 每次連線都以 INFO 印出 `no such identity`(§13) |
| 私鑰的預設 | 不移動。主機開始同步時,每把金鑰跳出一次「Sync key / Keep on this computer」 | 同類「以路徑引用金鑰」的產品都要使用者逐把同意;「預設同步」只出現在使用者主動把金鑰交給 app 保管庫的產品(§13) |
| 金鑰記錄放哪裡 | 帳戶 chain | 被多個 space 用到的金鑰只存一份;SP4 分享 space 時,金鑰不會跟著被分享 |
| 插槽檔名 | `<name>-<插槽 id 前 8 字元>` | 同 space 檔名(SP1 §4.3):不需要跨裝置協調名稱唯一,並發建立不會撞名 |
| 本機的選擇與同步的金鑰衝突時 | 本機挑的優先;同步的金鑰只放進空的插槽 | 不默默換掉使用者在這台電腦做的決定 |
| 停止同步 | 之後的新電腦收不到;已經有副本的電腦保留副本 | 私鑰不遠端刪除:被刪的那份可能是最後一份 |
| passphrase | 不同步;私鑰檔原樣同步 | 同 Panic Sync;有 passphrase 的私鑰在同步後仍受它保護 |
| Windows 上連到本機金鑰 | hard link,不行時複製 | symlink 需要系統管理員權限或開發者模式;hard link 不需要,而且和原檔共用權限設定 |

## 3. 安全模型

延續 SP1 §3,新增:

- 選擇同步的私鑰和主機一樣,以帳戶金鑰端對端加密:同步碼與每一台已加入的電腦都能解開。
- 本機狀態檔只保存它的密文 envelope(`RecordKind::is_secret` 已涵蓋 `key`,`refuse_plaintext_secrets` 不變)。解開
  只為了寫入插槽檔;內容不進 log、toast、錯誤訊息,也不經 IPC 回傳給前端。
- 私鑰只在需要它的電腦上寫成檔案(§6.2);其他電腦只保存密文,與 `spacekey` 相同。UI 不得暗示其他電腦取不到它。

| 情況 | 能做 | 不能做 |
|---|---|---|
| relay 被入侵 | 看到 `key` 記錄的密文與大小 | 讀取私鑰 |
| 電腦被偷(更換同步碼前) | 取得所有選擇同步的私鑰 | 取得「Keep on this computer」的私鑰(從未離開原本的電腦) |
| 電腦被偷(更換同步碼後) | 保有被偷當時已同步的私鑰(更換流程提醒到伺服器換掉,§6.6) | 取得之後才同步的私鑰 |
| 帳戶裡的惡意成員 | 推送一把金鑰,並讓主機指到它(只改變連線用哪把金鑰,不洩漏你的私鑰) | 覆蓋已經有內容的插槽;把檔案寫到 `~/.ssh/sshelter/keys/` 以外;讓公鑰和記錄上的指紋不符的私鑰落地 |

## 4. 資料模型

### 4.1 帳戶 chain 新增的種類

| kind | id | payload |
|---|---|---|
| `keyslot` | 插槽 id(隨機 16 bytes,32 字元小寫 hex) | `{ schema: 1, name, mode, origin_device_id, created_at_ms, public_key, fingerprint, key_type, has_passphrase }`。`mode` 為 `"synced"` 或 `"own"`;`mode = "synced"` 時後四項必填,`"own"` 時為 `null`(每台電腦的金鑰不同)。刪除 = tombstone |
| `key` | 插槽 id | `{ schema: 1, private_key }`:私鑰檔內容原樣(沿用 v1 預留的 `key` 種類,`is_secret`)。只在 `mode = "synced"` 時存在;停止同步 = tombstone |

- `name`:`^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$`,不得以 `.pub` 結尾。預設取金鑰的檔名,不合規的字元換成 `-`,去掉頭尾 `-`;
  結果是空字串時用 `key`。
- `device` payload 加上 `slots: Vec<{ slot_id, fingerprint, synced_copy }>`(serde default 空,空時不寫出):這台的插槽裡
  目前是哪把金鑰,以及它是不是同步來的副本。用來在 Keys 對話框顯示每台電腦的狀態,以及停止同步時列出還有副本的電腦。
  只含公開資訊;每台只寫自己的 `device` 記錄,所以舊版電腦改寫自己的記錄時不會影響別台的 `slots`。
  **不能沿用既有的 `keys` 欄位**:SP1 的 `DevicePayload.keys` 是 `Vec<String>`(一律寫空),SP1 電腦讀到物件陣列會解析失敗,
  而 `valid_account_record` 會把整筆 `device` 記錄丟掉。
- 舊版(SP1)的 `merge_account` 只解開 `device`、`meta`、`space`、`spacekey`,其他種類以密文原樣存進 `sealed`
  (`sync/merge.rs`):`keyslot` 與 `key` 在舊版電腦上會被保留,不處理、不刪除。
- SP3 的合併:`keyslot` 解開進 `records`,照一般記錄做 LWW。`key` 存在 `sealed`,但和 `spacekey` 一樣在記憶體解開比較
  (`record::merge`),本機較新的不被拉到的舊版本蓋掉;relay 回滾時也和 `spacekey` 一樣重設。
- 大小:relay 每筆密文上限 65,536 字元(約 49 KB 明文,`relay/src/index.ts`)。`private_key` 超過 16 KiB 的金鑰不提供同步
  (RSA 4096 的私鑰約 3.3 KB)。

### 4.2 本機檔案

- 目錄 `~/.ssh/sshelter/keys/`:Unix 0700;Windows 為只有目前使用者、不繼承的 DACL。
- 每個插槽兩個檔案:`<name>-<id8>`(私鑰,或指向私鑰的連結)與 `<name>-<id8>.pub`(公鑰,一般檔案)。

| 這台電腦的情況 | 插槽裡放的 |
|---|---|
| 建立插槽的電腦 | 連結到原本的金鑰檔(Unix:symlink;Windows:hard link,不行時複製) |
| 其他電腦,`mode = "synced"`,插槽空著而且這台需要它 | 解開的私鑰檔(Unix 0600;Windows 先建立帶 DACL 的空檔再寫入內容) |
| 其他電腦,使用者在這台挑了本機金鑰 | 連結到挑的那把(同第一列) |

- 寫入一律原子:暫存檔 → 設好權限 → rename。插槽路徑上已經有檔案時,只有本機狀態記錄為 SSHelter 建立的才會被替換。
- 插槽不再被任何主機使用,或 `keyslot` 被刪除:移除連結。同步來的副本與複製出來的檔案不刪,Keys 對話框標成
  「Not in use」並提供刪除按鈕。
- 離開同步帳戶:插槽目錄原樣保留;搬到 `~/.ssh/sshelter-local/` 的主機檔仍然指著它,照常能連線。

### 4.3 本機狀態

`sync-state.json` 新增欄位(serde default,舊檔讀得進來):

```text
key_slots: { <slot id>: { file_name,
                          source: Linked { path, fingerprint, link: Symlink | HardLink | Copy }
                                | SyncedCopy { fingerprint },
                          uploaded_fingerprint,
                          learned_in,
                          last_error } }
```

- 只含公開資訊與本機路徑,沒有祕密。
- 用來判斷「插槽檔是 SSHelter 建立的」,以及比對指紋(§6.5)。
- `uploaded_fingerprint`:這台的使用者在這裡選擇同步(上傳)的那把金鑰的指紋,是本機的同意記錄,帳戶裡的變更不會設定它。補寫 `key`
  (§6.6)時,連到本機金鑰的插槽只認它;建立或加入另一個帳戶時清掉。
- `learned_in`:這筆記錄是在哪個帳戶 chain 學到的;在被更換同步碼的帳戶學到的記錄,更換時換成新帳戶的:更換的那台一定換,其他電腦
  輸入新同步碼時,要新帳戶接續了舊帳戶的 space 才換(§6.6)。只有在目前帳戶學到的插槽會被補寫(§6.6)。

## 5. 主機改寫(建立插槽時)

- **範圍**:這台電腦上、勾選的 space 檔案裡,啟用中的 `IdentityFile`。它的值解析後必須是這台電腦上存在的私鑰檔,
  而且不是插槽路徑。
- **解析**:看得懂 `~`、`%d`、絕對路徑,以及 Windows 的反斜線路徑。以實際檔案為準,同一個檔案的不同寫法視為同一把金鑰。
  其他 token(`%h`、`%r`、`${VAR}` 等)和指向 `.pub` 的值不處理,對話框把它們列在「Can't set up automatically」並說明原因。
- **改寫**:只把那一行的值換成 `~/.ssh/sshelter/keys/<name>-<id8>`;縮排、關鍵字寫法、分隔符號和區塊裡的其他行都不動。
  用 SSHelter 既有的無損編輯完成,改寫結果照一般主機修改同步出去。
- 同名主機有不只一份、被 SP1 FA3 規則鎖住的,不改寫,對話框說明原因。
- 沒有 `IdentityFile`、用預設金鑰的主機:不處理。

## 6. 同步引擎

### 6.1 建立插槽(金鑰所在的電腦)

1. 這台電腦已經有指到同一個檔案的插槽,或帳戶裡已經有同指紋的 `mode = "synced"` 插槽:直接沿用,不再詢問
   (這把金鑰已經決定過了)。這台還沒有它的本機插槽時先做第 3 步,然後做第 4、5 步。
2. 寫入 `keyslot`(`mode` 依使用者的選擇);選「Sync key」時同時寫入 `key`。
3. 建立本機插槽(連結到原檔)與 `.pub`,寫入 `key_slots`。
4. 改寫主機(§5)。插槽檔一定先就位,主機才改寫,所以改寫後的主機立刻能用。
5. 更新這台的 `device.slots`。

### 6.2 接收(其他電腦)

每一輪合併帳戶 chain 之後:

- `keyslot` 解開進 `records`(不是祕密)。`key` 保持密封,只在要落地時於記憶體解開。
- 一個插槽是這台「**需要**」的:這台勾選的 space 檔裡,有啟用中的 `IdentityFile` 等於該插槽的路徑。
- 需要、插槽空著、`mode = "synced"`、有 `key` 記錄 → 驗證後落地。
- 需要、插槽空著、其他情況 → 狀態為「Needs a key on this computer」(§7.3)。
- 插槽裡已經有這台的連結或副本 → 不動(本機優先)。
- **驗證**:
  - 名稱符合 §4.1 的規則。
  - `private_key` 不超過 16 KiB,而且是 OpenSSH 或 PEM 格式的私鑰。
  - 從私鑰讀出的公鑰,指紋必須等於 `keyslot.fingerprint`。讀不出公鑰的(有 passphrase 的舊式 PEM)拒收。
  - 驗證失敗就不寫檔,插槽狀態顯示原因;訊息不含金鑰內容。

### 6.3 改成同步、停止同步

- **改成同步**(`own` → `synced`):只能在插槽裡讀得到私鑰的電腦上做。寫入 `key`,`keyslot` 補上公鑰與指紋,
  `mode = "synced"`。其他電腦上本機挑的仍然優先。
- **停止同步**:`key` 寫 tombstone,`keyslot` 改成 `mode = "own"`、清空公鑰欄位。已經有副本的電腦保留副本並繼續使用。

### 6.4 落地之後

更新 `key_slots` 與這台的 `device.slots`。

### 6.5 指紋檢查

在啟動與每輪同步時(視窗重新取得焦點會跑一輪),對 `key_slots` 裡每個連結比對來源檔的指紋;打開 Keys 對話框不另外檢查,
顯示最近一輪的結果(app 開著時換掉金鑰檔,要等下一輪才更新):

- 來源檔換成另一把金鑰:
  - Unix 的 symlink 本來就跟著路徑走。
  - hard link 或複製的,重新連結或重新複製,並在狀態列告知插槽已更新。
  - 如果這是 `mode = "synced"` 的插槽,而這台是上傳了這個插槽目前同步金鑰的電腦:**不自動上傳新金鑰**。插槽狀態顯示「這台的金鑰換了,
    其他電腦還是上一把」,提供「Sync the new key」。
- 來源檔不見了:狀態為「The key this slot points to is gone」。

### 6.6 更換同步碼(SP1 §7.5)

- 第 5 步也複製 `keyslot` 與 `key`:以新帳戶金鑰重新加密,保留 version、updated_at_ms、device_id 與 tombstone。
- 第 7 步的完成畫面列出同步過的金鑰,並提醒:如果是因為電腦遺失才更換,請到伺服器換掉這些金鑰;SSHelter 不會自動做。
- 沒有 SP3 的舊版電腦執行更換時,不會複製這兩種記錄。SP3 電腦發現 `key_slots` 裡的插槽在帳戶裡不存在、而勾選的 space 裡還有主機
  用到它時,重新寫入 `keyslot`;`synced` 的插槽,這台握有那把私鑰時一併重新寫入 `key`。「握有」只有兩種:這台的使用者在這裡選過同步
  這把金鑰(`uploaded_fingerprint`),或這台的同步副本(位元組來自帳戶,路徑上的檔案仍是記錄裡的那一把)。使用者選擇留在這台的私鑰,
  即使帳戶記錄被改成 `synced` 也不補傳。所以只要還有一台握有它、勾選的 space 裡也有主機用到它的 SP3 電腦輸入新同步碼,同步過的金鑰就會
  回到帳戶裡。
- 只補寫在這個帳戶(或更換同步碼之前的同一個帳戶)學到的插槽(`learned_in`)。離開之後建立或加入了另一個帳戶時,留給
  `~/.ssh/sshelter-local/` 主機的插槽記錄不寫進新帳戶,私鑰也不,那些主機之後被搬進新帳戶的 space 也一樣;這台在之前的帳戶裡同步
  金鑰的選擇也不帶過來。用同一個同步碼直接重新加入同一個帳戶(中間沒有建立或加入別的帳戶)時照常保留。輸入的「新同步碼」若不是
  接續這個帳戶的(沒有任何 space 接續舊帳戶的 space),視同加入另一個帳戶。
- 更換同步碼進行中、或這台錯過了更換(已凍結)時,建立插槽、Sync this key 與 Stop syncing 都會被拒絕(與 space 操作相同的訊息),
  等更換完成再做。

### 6.7 新電腦

加入帳戶或勾選 space 之後,插槽照 §6.2 處理:`synced` 的自動落地。需要本機金鑰的,在加入流程(或勾選 space)的最後
一步列出「Keys for this computer」,可以當場挑,也可以略過;略過的留在 Settings 的提示列(§7.3)。

## 7. 前端

### 7.1 「Sync key」對話框

- **觸發**:勾選的 space 檔裡出現「指到本機私鑰、還沒有插槽」的 `IdentityFile`,而且是這台電腦造成的:
  - Move hosts into a space 精靈完成時;
  - 在 space 裡新增主機,或修改同步主機的 `IdentityFile`(主機編輯器儲存、Deploy key 寫入)之後;
  - 升級到 SP3 後第一次啟動、既有的同步主機已經指到本機金鑰時。同一個對話框多一個「Later」,選了之後改成 Settings 的
    提示列。
- **內容**:每把金鑰一列,列出用到它的主機、插槽名稱(預設為檔名,可以在這裡改)、有沒有 passphrase、會被改寫的行。
  兩個按鈕:「Sync key」(主要)與「Keep on this computer」。
- 不能同步的金鑰(超過 16 KiB、有 passphrase 的舊式 PEM、不是 OpenSSH 或 PEM 格式)只提供「Keep on this computer」,
  並說明原因。接收端的驗證(§6.2)在這裡先做一次。
- 關閉視窗或按「Later」:不改任何東西;這些金鑰會出現在 Settings 的提示列(§7.3),之後從那裡或 Keys 對話框設定。
- 「Can't set up automatically」的值(§5)另列,附上原因。

### 7.2 Keys 對話框

新增「Keys used by synced hosts」區塊,每個插槽一列:

- 名稱;`synced` 的顯示指紋,`own` 的顯示「Each computer uses its own key」;用到它的主機數;
- 這台電腦用的檔案與狀態;各台電腦的狀態(來自 `device.slots`);
- 動作:Sync this key、Stop syncing、Pick a key on this computer、Change、Delete(只對 Not in use 的副本)。
- 「Pick a key on this computer」列出這台 `~/.ssh` 裡的私鑰(Keys 對話框現有的清單),也可以選其他位置的檔案。

### 7.3 狀態提示

- Settings → Sync 的提示列:
  - 「N keys used by synced hosts aren't set up」(金鑰所在的電腦還沒決定);
  - 「N keys need a key on this computer — Pick…」。
- 側邊欄:用到「這台缺金鑰」插槽的主機加一個小標記,tooltip 說明。
- linter 的「missing identity file」對插槽路徑改用插槽的說明,而不是只說檔案不存在。

### 7.4 其他文案

- Leave 對話框加一句:「Keys in ~/.ssh/sshelter/keys stay on this computer.」
- Change sync code:開始前與完成時提到同步過的金鑰(§6.6)。
- Forget:不變(仍然不是撤銷)。

## 8. 平台

- **Unix**:目錄 0700、私鑰 0600、`.pub` 0644;symlink 用絕對路徑。
- **Windows**:
  - 私鑰檔與目錄以「只有目前使用者、不繼承」的 DACL 建立,先建立帶權限的空檔再寫入內容,不留下可被讀取的空窗。
    使用 `windows-sys`(已經是間接相依,Cargo.lock 裡有現成版本)。Win32-OpenSSH 要求私鑰只屬於使用者、其他人不得存取,
    否則拒用(§13)。
  - 連結用 `CreateHardLinkW`:NTFS、同一個磁碟,不需特殊權限。失敗時(不同磁碟、FAT/exFAT)改成複製,套用同樣的 DACL。
- **路徑正規化(既有問題)**:SSHelter 寫入 `IdentityFile` 時,目前只把含 `/.ssh/` 的絕對路徑轉成 `~/.ssh/...`
  (`src/lib/identity-file.ts`)。Windows 上 home 底下的反斜線路徑(`C:\Users\<name>\.ssh\...`)也要轉成 `~/.ssh/...`。

## 9. 錯誤處理

| 情況 | 行為 |
|---|---|
| 落地失敗(磁碟、權限) | 插槽狀態顯示原因,下一輪重試;不影響其他插槽與主機同步 |
| 指紋不符、格式不對、超過大小 | 不寫檔;插槽狀態顯示「The synced key didn't match and was not written」 |
| 插槽路徑上已經有不是 SSHelter 建立的檔案 | 不覆蓋;狀態說明,請使用者把它移走 |
| 來源檔不見 | 狀態為「The key this slot points to is gone」 |
| 主機被 FA3 規則鎖住 | 對話框列出並說明,不改寫 |
| 更換同步碼進行中、或這台錯過了更換(已凍結) | 建立插槽、Sync this key、Stop syncing 一律拒絕(不讀檔、不寫記錄),訊息與 space 操作相同(「finish or cancel changing the sync code first」、「the sync code was changed on another device; enter the new sync code first」),等更換完成再做(§6.6) |
| 帳戶 chain 超過 relay 額度 | 沿用 SP1 §9 |

## 10. 測試

- **Rust 單元測試**:
  - 名稱與檔名規則;`keyslot`、`key` 的編解碼。
  - `key` 永遠不以明文進狀態檔。
  - 「需要」的判斷與落地條件;本機優先。
  - 驗證拒收:指紋不符、超過大小、有 passphrase 的舊式 PEM、名稱含路徑字元。
  - Unix 的檔案權限;停止同步不刪副本;指紋檢查與重新連結;上傳了這個插槽目前同步金鑰的電腦換了金鑰時不自動上傳。
  - 更換同步碼會複製兩種記錄;被舊版更換後的補寫。
  - 主機改寫無損(各種 `IdentityFile` 寫法、被 FA3 鎖住的主機)。
- **Windows**:DACL 與 hard link 的測試(`#[cfg(windows)]`)。新增一個在 `windows-latest` 上執行這些測試的 CI job
  (目前 CI 沒有在任何平台執行 app 的測試)。
- **前端**:
  - 對話框的觸發條件與內容(哪些金鑰、哪些行、無法自動設定的原因)。
  - Keys 對話框的各種狀態;提示列與側邊欄標記。
  - 反斜線路徑的正規化。
- **手動清單(Mac + Windows 實機)**:同步一把金鑰後在另一台直接連線;每台用自己的金鑰,在另一台挑一次;停止同步;
  本機優先;更換同步碼時的提醒;舊版電腦上的提示。

## 11. 發佈

以 beta 發佈。release notes 提醒每台電腦都要更新:舊版電腦上,指到插槽的主機連不上。

## 12. 對 SP1 spec 的影響

- SP1 §0 的子專案表:SP3 改為本文;SP4 只剩分享。
- SP1 §7.5 第 5 步的複製清單加上 `keyslot`、`key`(§6.6)。

## 13. 查證與調查(2026-10-05)

**OpenSSH 的行為**
- 設定裡不存在的 `IdentityFile`:`sshconnect2.c` 的 `load_identity_file` 在 `stat` 失敗時記錄 `no such identity`,
  `userprovided` 為真時用 INFO 等級。`readconf.c` 處理 `IdentityFile` 時傳入 `flags & SSHCONF_USERCONF`,也就是使用者
  設定檔裡的 `IdentityFile` 都算 `userprovided`。
  - https://github.com/openssh/openssh-portable/blob/master/sshconnect2.c
  - https://github.com/openssh/openssh-portable/blob/master/readconf.c
- Win32-OpenSSH 對使用者私鑰的要求:屬於使用者本人,其他使用者不得存取;權限太寬時顯示「UNPROTECTED PRIVATE KEY FILE」
  並無法使用。https://github.com/PowerShell/Win32-OpenSSH/wiki/Security-protection-of-various-files-in-Win32-OpenSSH

**同類產品**(完整的比較表與各產品來源見 `2026-10-05-sp3-key-sync-research.md`;以下為結論)
- 預設同步私鑰的,都是金鑰存在 app 保管庫的產品:Termius(預設同步,`Sync keys and identities` 可全域關閉,只管
  Personal vault)、1Password(隨 vault 同步,各台另外開 agent)、Bitwarden(隨 vault 同步,沒有逐把開關)、
  Panic Prompt(all-or-nothing,passphrase 不同步)。
  - https://docs.termius.com/keychain/sync-of-keys-and-passwords
  - https://www.1password.dev/ssh/agent
  - https://bitwarden.com/help/about-ssh/
  - https://help.panic.com/sync/sync-data-types/
- 以路徑引用金鑰的產品,預設不移動私鑰,要逐把明確同意:XPipe(key 檔「always require confirmation」)、Tabby(加 key 時
  選 Filesystem 或 Vault)、Royal TS(預設存路徑,可改成嵌入)。
  - https://docs.xpipe.io/guide/ssh-auth
  - https://github.com/Eugeny/tabby
  - https://docs.royalapps.com/r2023/scripting/objects/organization/royalcredential.html
- 沒有產品在接收端詢問。金鑰留在原本電腦時,常見的解法是「邏輯名稱 + 每台不同步的對照」:Royal TS 依名稱指派
  credential、Tabby 的 ssh-keymap 外掛、1Password 讓 `IdentityFile` 指向 `.pub` 再由 agent 提供私鑰。
  - https://github.com/mathys-lopinto/tabby-ssh-keymap
  - https://www.1password.dev/ssh/agent/advanced

## 14. 後續階段(2026-10-05 決定,各自另寫 spec)

1. **本文:金鑰插槽。** 先讓同步的主機在每台電腦上都能連線。使用者若選「Keep on this computer」,沒有私鑰離開電腦。
2. **SSHelter 自己的 SSH agent。**
   - 金鑰放進 SSHelter 的保管庫,用同一條帳戶 chain 加密同步(沿用本文的 `keyslot`、`key` 記錄)。
   - SSHelter 提供 agent(Unix socket;Windows named pipe),每次使用都要核准。
   - 插槽只放 `.pub`,主機的 `IdentityFile` 不變,每台電腦的本機設定加上 `IdentityAgent`。
   - MCP 的 `run` 也走這個 agent。私鑰只在 SSHelter 手裡時,電腦上的其他程式(包括 AI 工具)不經 SSHelter 核准就用不了
     這些金鑰;現在的 MCP 核准擋不住其他程式直接執行 `ssh`(README「AI Access (MCP)」)。
   - 代價:SSHelter 沒在執行時,這些金鑰無法使用;要處理 Windows 上系統 agent 的 pipe。
3. **內建終端機(可選)。** 用 xterm.js 加上 SSHelter 自己的 SSH 連線,直接用 agent 的金鑰。Ghostty 的圖形介面目前沒有
   Windows 版;可以嵌入的 `libghostty-vt` 只負責解析與畫面狀態,繪製要自己做(查證於 2026-10-05)。
   - https://mitchellh.com/writing/libghostty-is-coming
   - https://github.com/ghostty-org/ghostty/discussions/2563

**本文要為第 2 階段保留的條件**
- `keyslot`/`key` 記錄只描述金鑰本身,不寫死「落地成檔案」:第 2 階段加一個「由 agent 提供」的交付方式時,
  資料格式不需要改。
- 主機的 `IdentityFile` 一律指到插槽路徑;第 2 階段把插槽檔換成 `.pub`(OpenSSH 會依 `.pub` 向 agent 要對應的私鑰)。

