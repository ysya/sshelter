# Spike 驗證結果：SSH_ASKPASS 密碼部署 + macOS Keychain

執行日期：2026-07-29
對應計畫：`docs/superpowers/plans/2026-07-29-key-deploy-password.md` Task 0
對應規格：`docs/superpowers/specs/2026-07-29-key-deploy-password-design.md` 「待驗證假設」

本 task 不寫任何 production code，僅實測三個假設中的假設 1 與假設 3（假設 2 依規格留給 Task 3，需要先有 helper 模式程式碼才能驗證）。

## 環境調整（相對 task-0-brief.md 的偏離）

- **Docker host port**：brief 原文用 `-p 2222:2222`。實測前用 `lsof -nP -iTCP:2222 -sTCP:LISTEN` 確認 host 的 2222（與 2223）已被本機既有的 OrbStack `vpn-gateway` / `vpn-gateway-tor` 容器占用。改用 **host port 2299 對應容器內部 2222**（`-p 2299:2222`），下游所有 `ssh -p` 一律改成 `-p 2299`。image、tag、所有環境變數（`PASSWORD_ACCESS`、`USER_NAME`、`USER_PASSWORD`、`PUID`、`PGID`）與 brief 完全一致，未替換 image。
- 其餘所有指令（含 `-o UserKnownHostsFile=...`、`-o StrictHostKeyChecking=...` 等）逐字照抄 brief，未增刪任何 flag。未曾寫入 `~/.ssh/config` 或使用者真正的 `~/.ssh/known_hosts`。

---

## Step 1：啟動拋棄式 sshd

實際執行指令：

```bash
docker run -d --name sshelter-spike -p 2299:2222 \
  -e PASSWORD_ACCESS=true -e USER_NAME=spike -e USER_PASSWORD=hunter2 \
  -e PUID=1000 -e PGID=1000 \
  linuxserver/openssh-server
sleep 10 && docker logs sshelter-spike | tail -5
```

輸出（image 拉取略，容器啟動後 log 尾五行）：

```
sshd is listening on port 2222
User/password ssh access is enabled.
[custom-init] No custom files found, skipping...
[ls.io-init] done.
```

容器就緒。

## Step 2：假 askpass

依 brief 逐字建立 `$SP/askpass.sh`（`$SP` = `/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike`），內容：

```sh
#!/bin/sh
printf '%s\n' "PROMPT>>>$1<<<" >> "$SPIKE_LOG"
printf '%s\n' 'hunter2'
```

已 `chmod +x`，並以 `od -c` 逐位元組核對內容與 brief 一致。

## Step 3：假設 1 驗證 —— `force` 是否攔截 password 認證提示

實際執行指令（僅 `-p 2222` → `-p 2299` 一處變動，其餘逐字照抄）：

```bash
SP=/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike
rm -f "$SP/prompts.log"
SPIKE_LOG="$SP/prompts.log" \
SSH_ASKPASS="$SP/askpass.sh" \
SSH_ASKPASS_REQUIRE=force \
  ssh -T -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
      -o NumberOfPasswordPrompts=1 -o PreferredAuthentications=password \
      -p 2299 spike@localhost 'echo DEPLOY_OK'
echo "exit=$?"
cat "$SP/prompts.log"
```

**終端機原始輸出（完整、未刪節）：**

```
Warning: Permanently added '[localhost]:2299' (ED25519) to the list of known hosts.
DEPLOY_OK
exit=0
```

**`prompts.log` 原始內容（完整、未刪節）：**

```
PROMPT>>>spike@localhost's password: <<<
```

`od -c` 位元組核對（證明 `password:` 後只有一個空格，緊接 `<<<`）：

```
0000000    P   R   O   M   P   T   >   >   >   s   p   i   k   e   @   l
0000020    o   c   a   l   h   o   s   t   '   s       p   a   s   s   w
0000040    o   r   d   :       <   <   <  \n
```

### 結論（Verdict）——這是本次 spike 最重要的輸出

**假設 1 成立：YES。** `SSH_ASKPASS_REQUIRE=force` 確實讓 ssh 在 password 認證時，完全不在終端機互動要求輸入密碼，而是把提示文字（`spike@localhost's password: `）交給 askpass helper 取得密碼，一次性完成、`exit=0`、遠端指令 `echo DEPLOY_OK` 正確執行。終端機唯一出現的訊息是 `StrictHostKeyChecking=no` 造成的「加入 known_hosts」告知訊息，**不是**互動式提示，過程中沒有任何一行要求使用者輸入。`prompts.log` 僅有一行，形狀與 brief 預期的 `PROMPT>>>spike@localhost's password: <<<` 完全吻合。

**`SSHELTER_REAL_PASSWORD_PROMPT`（供 Task 2 白名單測試「接受」用，逐位元組，含結尾恰一個空格）：**

```
spike@localhost's password: ␠
```
（`␠` 為本文件標記結尾空格用的可視符號，非提示字串本身的一部分；真正字串到 `password:` 後的那一個半形空格為止，之後直接是 argv 結尾，見上方 `od -c`。）

## Step 4：host key 提示字串（給 Task 2 白名單測試「拒絕」用）

實際執行指令（同樣僅调整 port）：

```bash
SP=/private/tmp/claude-501/-Users-ysya-project-homelab-ssheditor/spike
rm -f "$SP/prompts.log"
SPIKE_LOG="$SP/prompts.log" \
SSH_ASKPASS="$SP/askpass.sh" \
SSH_ASKPASS_REQUIRE=force \
  ssh -T -o StrictHostKeyChecking=ask -o UserKnownHostsFile="$SP/empty_known_hosts" \
      -p 2299 spike@localhost 'echo X'
```

**非預期但重要的發現：這條指令不會自行結束，會無限迴圈。** 原因（已由 `prompts.log` 內容證實）：`SSH_ASKPASS_REQUIRE=force` 不只攔截 password 提示，**連 host key 信任確認的 yes/no/fingerprint 提示也一併透過 askpass helper 取得答案**。但 Step 2 的假 askpass 不論收到什麼提示文字，一律固定回覆 `hunter2` —— 這對 yes/no 問題不是合法答案，於是 ssh 不斷重新提出「Please type 'yes', 'no' or the fingerprint: 」，askpass 又固定回 `hunter2`，形成無終止的重試迴圈（此重問沒有類似 `NumberOfPasswordPrompts` 的次數上限）。實測中此迴圈在被人工中止前已寫入 `prompts.log` 達 **2.5MB / 44,657 行**（其中 44,653 行是重複的 reprompt 訊息），已人工中止該行程並清空巨大 log，不影響下方擷取到的真實提示字串。

**這不影響假設 1 的結論**——假設 1 明確只針對「password 認證提示」，Step 3 已乾淨驗證。這是一個額外發現：`force` 對「所有」askpass 風格的輸入（不只密碼）都會生效，這也印證了 spec 原先要做白名單機制的必要性：正式的 askpass 替身必須先判斷提示文字是否為「password:」形狀，不符合白名單就必須拒答，否則會像本次實測一樣對 host-key 提示做出無意義、無限迴圈的回應。

`prompts.log` 第一段（真正的 host key 提示，`od -c` 位元組核對，確認結尾 `?` 後只有一個空格）：

```
0000000    P   R   O   M   P   T   >   >   >   T   h   e       a   u   t
0000020    h   e   n   t   i   c   i   t   y       o   f       h   o   s
0000040    t       '   [   l   o   c   a   l   h   o   s   t   ]   :   2
0000060    2   9   9       (   [   :   :   1   ]   :   2   2   9   9   )
0000100    '       c   a   n   '   t       b   e       e   s   t   a   b
0000120    l   i   s   h   e   d   .  \n   E   D   2   5   5   1   9    
0000140    k   e   y       f   i   n   g   e   r   p   r   i   n   t    
0000160    i   s   :       S   H   A   2   5   6   :   w   c   P   z   O
0000200    F   /   /   C   E   0   Q   p   Q   3   D   i   k   A   +   x
0000220    +   o   b   B   S   n   S   P   2   D   M   b   e   v   j   E
0000240    i   S   n   o   u   Q  \n   T   h   i   s       k   e   y    
0000260    i   s       n   o   t       k   n   o   w   n       b   y    
0000300    a   n   y       o   t   h   e   r       n   a   m   e   s   .
0000320   \n   A   r   e       y   o   u       s   u   r   e       y   o
0000340    u       w   a   n   t       t   o       c   o   n   t   i   n
0000360    u   e       c   o   n   n   e   c   t   i   n   g       (   y
0000400    e   s   /   n   o   /   [   f   i   n   g   e   r   p   r   i
0000420    n   t   ]   )   ?       <   <   <  \n
```

**`SSHELTER_REAL_HOSTKEY_PROMPT`（供 Task 2 白名單測試「拒絕」用，逐位元組，此為單一多行字串，含結尾恰一個空格）：**

```
The authenticity of host '[localhost]:2299 ([::1]:2299)' can't be established.
ED25519 key fingerprint is: SHA256:wcPzOF//CE0QpQ3DikA+x+obBSnSP2DMbevjEiSnouQ
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? ␠
```
（同上，`␠` 只是本文件用來標出結尾空格的標記，不是字串內容；真實字串在問號後有一個半形空格作結，見上方 `od -c` 最後一行。）

> **給 Task 2 的提醒**：上面字串裡的 `[localhost]:2299` 是本次 spike 用的臨時 host/port，實際部署目標的 host/port 會不同。白名單測試「拒絕」這個字串時，應驗證的是整體**形狀**（`The authenticity of host ... can't be established.` … `Are you sure you want to continue connecting (yes/no/[fingerprint])? `），而不是把 `localhost:2299` 這幾個字硬編碼進測試斷言裡當作必要條件。

## Step 5：假設 3 驗證 —— macOS 未簽章 app 存取 keychain 的提示行為

**未能在本次 sandbox 執行環境完成實測 —— 這是誠實的「無法驗證」，不是「不成立」。**

依 brief 執行第一條指令時：

```bash
security add-generic-password -s SSHelter-spike -a probe -w hunter2
```

被 Claude Code 的 auto-mode 權限分類器直接擋下（未執行），訊息為：「Permission for this action was denied by the Claude Code auto mode classifier.」。診斷後確認**只有寫入動作被擋**——同一 service/account 的唯讀查詢並未被擋：

```bash
$ security find-generic-password -s SSHelter-spike -a probe -w
security: SecKeychainSearchCopyNext: The specified item could not be found in the keychain.
find_exit=44
```

（`44` = item not found，證實 add 從未執行、keychain 內無殘留，無需清理。）

因此無法在本次會話中實際觀察「讀回時是否跳出允許存取鑰匙圈對話框」。額外記錄一項結構性限制供後續參考：即使該指令被允許執行，若 macOS 真的彈出「允許存取鑰匙圈」GUI 對話框，本 agent 是在非互動的背景 shell 中執行，**無法看到或點擊該對話框**——這代表此假設的真正驗證，無論如何都需要一個人坐在真正互動式終端機前執行並親眼觀察，而不只是取得指令執行權限就夠。

補充唯讀背景資訊（不是本假設的直接證據，但佐證 spec 所述「未公證的未簽章 build」的前提）：

```
$ codesign -dv /Applications/SSHelter.app
Executable=/Applications/SSHelter.app/Contents/MacOS/sshelter
Identifier=sshelter-47a2748a49a76c8f
Format=app bundle with Mach-O universal (x86_64 arm64)
CodeDirectory v=20400 size=109208 flags=0x20002(adhoc,linker-signed) hashes=3409+0 location=embedded
Signature=adhoc
Info.plist=not bound
TeamIdentifier=not set
Sealed Resources=none
Internal requirements=none

$ spctl -a -vv /Applications/SSHelter.app
/Applications/SSHelter.app: rejected
source=no usable signature
```

已安裝的 SSHelter.app 目前確實是 ad-hoc 簽章、無 TeamIdentifier，`spctl` 判定 rejected——與 spec 假設情境（未公證未簽章 build）相符，但這只是背景資訊，不能取代真正對 keychain 存取提示行為的實測。

### 後續更新（Task 1，2026-07-29）：假設 3 已有第一手證據

Task 1 為 `secrets.rs` 寫的 `round_trip_set_get_delete` 測試，實際執行時提供了本次 spike 當下拿不到的第一手觀察（單獨隔離執行）：

```
$ cargo test secrets::tests::round_trip_set_get_delete -- --nocapture --exact
running 1 test
test secrets::tests::round_trip_set_get_delete ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 199 filtered out; finished in 0.21s
```

輸出中沒有出現測試裡「no credential store」的 skip 訊息，代表 `available()` 回傳 `true`，且真的走了 `set` → `get` → `delete` → 再讀回確認 `None` → 再 `delete` 一次（驗證可重入）這條真實路徑，不是提早 return 的分支。整個 `cargo test`（含編譯）數秒內完成，測試本體只花 0.21 秒，過程中沒有任何停頓、卡住或需要人工介入的跡象。

**觀察結論：這次以 `cargo test` 產生的（同樣未簽章/ad-hoc）測試執行檔存取 SSHelter 自建的 keychain 項目時，沒有跳出「允許存取鑰匙圈」的 GUI 對話框，執行也沒有被卡住等待互動。**

保留的限制（誠實記錄，不誇大這次觀察的涵蓋範圍）：
1. 這次驗證的是 `cargo test` 產生的測試執行檔，不是 Task 0 原本要驗證的「已打包、ad-hoc 簽章的 `SSHelter.app`」本體。兩者雖然都是未簽章/ad-hoc，但簽章身分不必然相同，keychain 的 ACL 判斷(「這是同一個 app 嗎」)也可能因此不同——這是強力佐證，不是對已打包 app 的逐位元組驗證。
2. 這次是同一個 process 建立、讀取、刪除同一筆項目，屬於「自己存取自己剛建立的項目」的情境，不是「不同簽章身分存取既有項目」的情境（後者更接近使用者升級/重裝 app 後、舊 app 建立的既有 keychain 項目被新 app 存取的實際場景）。
3. 沒有嘗試用任何方式抑制或繞過對話框——本來就沒有出現，符合「若跳出對話框不得抑制或繞過」的要求。

### 結論（Verdict）

**Task 0 當下**：無法驗證（BLOCKED），非「成立」也非「不成立」——寫入指令被權限分類器擋下，且背景 shell 本來就看不到 GUI 對話框。

**Task 1 更新**（見上方「後續更新」）：`round_trip_set_get_delete` 測試提供了第一手證據——`available()` 為 `true`、真正碰了 keychain、0.21 秒內完成、全程無停頓，沒有觀察到鑰匙圈存取對話框。**在「同一 process 建立並存取自己項目」與「`cargo test` 執行檔」這兩個限定條件下，假設 3 傾向不成立（不會跳出阻塞式對話框）**；但「已打包 SSHelter.app 存取」與「不同簽章身分存取既有項目」這兩種更貼近實際使用情境的狀況仍未實測，不宜視為假設 3 全面成立。

---

## Task 3（2026-07-30）：假設 2 驗證 —— 打包後的 bundle 自我啟動為 helper

對應 brief：`.superpowers/sdd/2026-07-29-key-deploy-password/task-3-brief.md`。程式碼變更：`src-tauri/src/main.rs` 在任何 Tauri 呼叫之前攔截 `SSHELTER_ASKPASS=1` 並呼叫 `sshelter_lib::askpass::run()`。

### 主要論證：型別系統保證（編譯期,不是實測)

**決定性的證據不是底下任何一次實測,而是編譯期就能確認的控制流程。** `askpass::run()` 的型別是 `-> !`;內部每一個分支——白名單拒絕、沒有密碼、寫入失敗、成功寫出密碼——最終都以 `std::process::exit(...)` 結尾,沒有任何一條路徑會 `return`。因此 `main()` 裡緊接在 `askpass::run()` 呼叫之後的 `sshelter_lib::run()`（也就是唯一會呼叫 `tauri::Builder::default()....run(...)` 的地方,整個程式裡唯一能建立視窗的程式碼),在 `SSHELTER_ASKPASS` 這個分支上**於控制流程上保證不可能被執行到**。這不是「這次剛好沒撞到」的經驗觀察,而是型別系統與程式碼結構本身就排除了這個可能性——任何一次執行、任何環境下都成立,不需要靠實測去「發現」。

下面 Step 3、4 的實測與 GUI 檢查訊號,回答的是一個**更窄的殘留問題**：上面的保證只涵蓋「我們自己的 Rust 程式碼會不會呼叫到 Tauri 初始化」,但打包（bundling）這個動作本身,或作業系統啟動一個 `.app` bundle 內執行檔的方式,會不會在我們的程式碼控制流程之外,另外引入某種副作用（例如 Info.plist 的某個 key、Launch Services 對 bundle 執行檔的預設處理)？下面的內容都是針對這個殘留問題的檢查,不是重新論證控制流程本身——控制流程的論證在這裡就已經完成。

### Step 3：debug 執行檔（未打包）

```bash
cd src-tauri
BIN=./target/debug/sshelter

SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 \
  "$BIN" "Are you sure you want to continue connecting (yes/no/[fingerprint])? " \
  >/tmp/reject.stdout 2>/tmp/reject.stderr
echo "rejected exit=$?"
```

輸出：`rejected exit=1`；`/tmp/reject.stdout` 為 0 bytes；`/tmp/reject.stderr` 為 `[sshelter-askpass] refused: "Are you sure you want to continue connecting (yes/no/[fingerprint])? "`。

```bash
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 \
  "$BIN" "spike@localhost's password: " \
  >/tmp/accept.stdout 2>/tmp/accept.stderr
echo "accepted exit=$?"
```

輸出：`accepted exit=0`；`od -c /tmp/accept.stdout` 顯示恰為 `h u n t e r 2 \n`（8 bytes）；`/tmp/accept.stderr` 為 0 bytes。兩次結果都與白名單設計相符，且完全複現 brief Step 3 的期望。

### Step 4：打包後的 bundle

```bash
pnpm tauri build --debug
```

`.app` 本體成功打包：`Bundling SSHelter.app (.../target/debug/bundle/macos/SSHelter.app)`。**後續的 DMG 打包步驟失敗**（`bundle_dmg.sh` 執行失敗，`[ELIFECYCLE] Command failed with exit code 1`）——研判是 `bundle_dmg.sh` 內部用 AppleScript 操控 Finder 排版視窗圖示，在這次無互動桌面 session 的環境下無法完成；這與 Step 4 要驗證的目標（`.app` 內執行檔的 helper 行為）無關，`.app` bundle 本身在 DMG 步驟失敗前已完整產出，不影響本假設的驗證。

執行 brief 指定的萬用字元探測：

```bash
APP=$(ls src-tauri/target/debug/bundle/macos/SSHelter.app/Contents/MacOS/*)
```

只比對到唯一一個檔案：`src-tauri/target/debug/bundle/macos/SSHelter.app/Contents/MacOS/sshelter`（`file` 確認為 `Mach-O 64-bit executable arm64`；`codesign -dv` 確認 `flags=0x20002(adhoc,linker-signed)`、`Signature=adhoc`，與已安裝的正式版一致，非本假設重點但供對照）。

**偏離 brief 逐字指令之處**：brief Step 4 範例用 `"Password: "` 且預期 `exit=0`。但 Task 2 已把白名單收緊為「無前綴時只接受 client 產生的固定形狀」，`askpass.rs` 的 `accepts_kbdint_password_prompt_with_client_prefix` 測試明確斷言 `!prompt_is_answerable("Password: ")`——裸的 `Password: ` 現在是**刻意拒絕**的形狀，不再是合法的 accept 範例。依控制者指示改用與 Step 3 相同、目前確實會被接受/拒絕的兩個字串重跑：

```bash
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 "$APP" "spike@localhost's password: "
# bundle accept exit=0；stdout od -c 恰為 h u n t e r 2 \n（8 bytes）；stderr 0 bytes

SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 "$APP" "Are you sure you want to continue connecting (yes/no/[fingerprint])? "
# bundle reject exit=1；stdout 0 bytes；stderr: [sshelter-askpass] refused: "..."
```

兩者與 debug 執行檔（未打包）的行為逐位元組一致。

### 次要檢查：實測訊號（針對上面的殘留問題,不是重新論證控制流程）

**方法**：本 agent 執行於非互動的背景 shell，無法用肉眼確認「有沒有視窗閃現」或「Dock 有沒有跳圖示」。改用以下幾項可在非互動環境驗證、且彼此獨立的訊號：

1. **執行時間**：`time (SSHELTER_ASKPASS=1 ... "$APP" "spike@localhost's password: " >/dev/null 2>&1)` → `0.00s user 0.00s system 74% cpu 0.006 total`。**6 毫秒這個絕對數字本身就足以支撐論點,不需要靠比較**：這支 app 掛載 8 個 Tauri plugin,任何一次真正的 AppKit/WebView 初始化都不可能在 6 毫秒內完成。（先前這裡曾把 `lsappinfo` 回報的、同一支 app 正常 GUI 啟動的 `launch to checkin time: 10.9872 seconds` 拿來相除,算出「約 1800 倍」的比值,已經拿掉——那個 11 秒數字來自**另一個、已經在跑兩天的已安裝 process**,`launch to checkin` 是 macOS 生命週期的一個里程碑事件,不是 `time` 量的 fork/exec 到結束 wall clock,兩者既不是同一個 process,也不是同一種量測基準；11 秒對這支 app 而言本身也偏慢,很可能只是登入時的資源競爭,拿來當分母只會製造這次量測撐不起的假精確度。6ms 這個數字單獨站得住,不需要那個比值。）
2. **Launch Services 註冊（`lsappinfo list`）**：分別在測試前、測試中（背景執行時緊接著 20 次高頻 `ps` 輪詢）、測試後各取一次快照，比對 `grep -i sshelter` 的結果。三次快照完全相同,只有已安裝、本來就在執行中的 `/Applications/SSHelter.app`（checkin time 為 2+ 天前，對應本機一直開著、縮到系統匣的正式版，PID 660）——我們測試用的 `.../target/debug/bundle/macos/SSHelter.app` 從未出現任何一筆新註冊。**補充說明其論證力道**：單純直接執行 bundle 內的 Mach-O（不透過 `open`／Finder）本身並不保證不會註冊 Launch Services——只要程式碼真的跑到 `NSApplicationMain`（Tauri/tao 事件迴圈的底層），一般仍會正常取得 Dock 圖示與 LS 註冊。因此「完全沒有新註冊」這件事,對應的正是「程式碼在到達那段初始化之前就已經 `exit()`」，而不是「直接執行 bundle 執行檔」這個啟動方式本身的副作用。
3. **process 存活時間（`ps` 輪詢）**：背景啟動後緊接 20 次幾乎無間隔的 `ps -p <pid>`輪詢，只在 1/20 次輪詢中捕捉到該 process 存在,其餘 19 次已經結束——與「立即印出答案並結束」一致，不是長駐等待事件迴圈的行為。
4. **無殘留 process／無 crash report**：測試後 `ps aux | grep sshelter` 只剩下本來就在跑的正式版（PID 660，`Mon05AM` 就啟動，與本次測試無關）；`~/Library/Logs/DiagnosticReports` 過去 10 分鐘內無任何 `sshelter` 相關的當機報告（若真的初始化一半又崩潰，跳出的當機對話框也算一種「GUI」，因此一併排除）；`lsappinfo front` 回報的最前景 app ASN 與上述任何 SSHelter 相關 ASN 皆不同。

### 嘗試更直接的驗證：lldb 對 `_NSApplicationMain` 設中斷點（審查意見 Finding 3，2026-07-30）

比上面四項訊號更直接的驗證方式：直接對 bundle 執行檔在 AppKit 的 `_NSApplicationMain`（Tauri/tao 事件迴圈實際呼叫 Cocoa 的進入點）設中斷點,在 `SSHELTER_ASKPASS=1` 分支下執行——若中斷點完全不觸發、process 正常結束,就是不靠時間或註冊表推論的直接證據。實際指令：

```bash
APP=$(ls src-tauri/target/debug/bundle/macos/SSHelter.app/Contents/MacOS/*)
SSHELTER_ASKPASS=1 SSHELTER_ASKPASS_SECRET=hunter2 lldb -b \
  -o 'breakpoint set --name _NSApplicationMain' \
  -o 'run' \
  -o 'continue' \
  -- "$APP" "spike@localhost's password: "
```

`breakpoint set`順利完成，回報 `Breakpoint 1: no locations (pending)`（預期行為——AppKit 尚未載入，中斷點延後解析）。但接著 `run` 之後整個指令卡住，不再有任何新輸出，直到背景工作的 9 分鐘上限前被主動中止。診斷過程：

- `ps -p <pid>` 顯示目標 process（`sshelter ... spike@localhost's password:`）處於 `T`（stopped）狀態超過 3 分鐘不變。
- `lldb` 本身的累積 CPU 時間在這段期間完全沒有增加（前後皆為 `0:06.85`，`%CPU` 為 0）——不是還在運算，是真的被卡住，不是「符號解析很慢」。
- `/usr/bin/log show --last 5m --predicate 'process == "debugserver"'` 讀 `debugserver` 自己的 log，最後一行停在：

  ```
  [LaunchAttach] (96280) about to task_for_pid(96279)
  ```

  之後 5 分鐘內沒有再寫入任何一行。`task_for_pid()` 正是 macOS 上取得除錯控制權那個受安全機制（SIP／`get-task-allow`／Developer Mode 授權）閘門管制的系統呼叫——與審查意見預先設想的「lldb 被 SIP、codesign/get-task-allow、或 developer-mode 提示擋下」完全吻合。

**依審查意見的指示，沒有嘗試繞過**（例如執行 `DevToolsSecurity -enable` 這類會變更系統層級安全設定、且超出本 repo 範圍的操作），而是直接停止該背景工作、確認清乾淨其產生的三個 process（`lldb`、`debugserver`、被除錯的目標 process 皆已終止，只剩下本來就在跑的正式版 PID 660，未受影響）。

**這個更直接的驗證方法在本次環境中被 macOS 的除錯授權機制擋下，沒有得到「中斷點有沒有觸發」的直接答案。** 如實記錄被擋下的事實，不因此升級或降級「傾向成立（YES）」的判斷——維持原本以型別系統保證（見上方主要論證）為主、四項執行期訊號為輔的結構。

**另一個做不到、誠實聲明放棄的嘗試**：曾嘗試用 `osascript -e 'tell application "System Events" to get name of every process'` 想直接列舉當下所有 GUI process/視窗，但這個呼叫觸發了 macOS 的 Automation 權限對話框（要求允許本 shell 控制 System Events），在非互動 session 中無法點擊，導致該指令掛住直到 120 秒逾時被強制終止（已確認終止後沒有殘留的 `osascript` 行程；`System Events.app` 是 macOS 自身隨附的自動化服務，並非我們的程式碼產生的副作用）。此後放棄任何需要 Accessibility/Automation 權限的檢查方式,只採用上述四項不需要額外權限的訊號。**因此「肉眼確認畫面上沒有任何視窗閃現」這一半,本次無法百分之百驗證**——真正蓋棺論定需要一個人坐在互動式 session 前實際觀察螢幕,而不只是取得 shell 執行權限。上述四項訊號（尤其是控制流程上的靜態保證 + 6ms 執行時間 + 零 Launch Services 註冊）已足以高度支持「沒有初始化 GUI」，但不等同人眼觀察的直接證據。

### 結論（Verdict）

**假設 2：傾向成立（YES，但有一項無法窮盡驗證的子項）。** 決定性的論證是編譯期的控制流程保證（見上方「主要論證」）：`askpass::run() -> !` 的每個分支都以 `process::exit` 結尾，`sshelter_lib::run()`（唯一呼叫 `tauri::Builder` 的地方）在這個分支上不可能被執行到。實測作為對這個保證之外的殘留問題（打包/啟動方式本身會不會引入額外副作用）的檢查：打包後的 `.app` bundle 內的執行檔,在 `SSHELTER_ASKPASS=1` 時可以乾淨退出、行為與未打包的 debug 執行檔逐位元組一致（正確的 exit code、正確的 stdout/stderr 內容),且四項獨立、非侵入式的訊號（6ms 執行時間、Launch Services 註冊、process 存活時間、無殘留/無當機報告）都與「完全沒有初始化 GUI」一致。唯一未能完成的是「人眼直接觀察畫面上是否有視窗閃現」——這需要互動式 session,本次非互動 agent shell 結構上做不到;嘗試用兩種方式取代人眼觀察（lldb 中斷點、System Events 自動化）皆被 macOS 的權限/授權機制擋下（分別是 `task_for_pid` 除錯授權、Automation 權限對話框），未強行繞過。**在這個限定下,沒有觀察到任何跡象顯示假設 2 不成立**,不需要轉向 Task 6 的 sidecar（`bundle.externalBin`）設計。

（範圍限制：本次僅驗證 macOS bundle。spec 原文同時要求驗證 Linux AppImage,但這次的開發環境只有 macOS——**Linux AppImage 這一半完全沒有測試**,不是「已知沒問題」,也不是「不適用」,單純是這次沒有 Linux 環境可以測試,跟上面 macOS 的結論無關,需要有 Linux 環境時另外補測。）

## 三個假設總覽

| # | 假設 | 結果 |
|---|------|------|
| 1 | `SSH_ASKPASS_REQUIRE=force` 攔截 password 認證提示，零互動 | **成立（YES）**——見 Step 3 |
| 2 | Tauri 打包後的 bundle 自我啟動為 helper 可乾淨退出 | Task 3：**傾向成立（YES）**——決定性論證是編譯期控制流程保證（`askpass::run() -> !` 每個分支都 `exit`，`sshelter_lib::run()` 不可能被執行到），exit code／stdout/stderr 逐位元組正確、執行時間 6ms、Launch Services 零新註冊、無殘留 process/當機報告皆與此一致；嘗試用 lldb 對 `_NSApplicationMain` 設中斷點直接驗證,被 macOS 除錯授權（`task_for_pid`）擋下,「人眼直接觀察無視窗閃現」在非互動 shell 中無法窮盡驗證（見 Task 3 章節） |
| 3 | 未簽章 app 存取自建 keychain 項目的提示行為 | Task 0 **BLOCKED**；Task 1 `round_trip_set_get_delete` 補上第一手證據——`available()` 為 `true`、0.21 秒內完成、無阻塞式對話框（見 Step 5 後續更新；限定於 `cargo test` 執行檔存取自建項目，尚未涵蓋已打包 app 或跨簽章身分存取既有項目） |

**額外發現（非原三假設之一，但影響 Task 2/4 設計）**：`SSH_ASKPASS_REQUIRE=force` 對所有 askpass 風格提示（password **與** host-key 信任確認）都會生效；正式 askpass 替身必須以白名單只回答「password:」形狀的提示，其餘一律拒答，否則會像本次實測一樣對 host-key 提示造成無限迴圈。

## 清理

```bash
docker rm -f sshelter-spike
```

（於下方 git commit 前執行；keychain 未建立任何項目，無需清理；未修改 `~/.ssh/config` 或 `~/.ssh/known_hosts`。）
