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

### 結論（Verdict）

**假設 3：無法驗證（BLOCKED），非「成立」也非「不成立」。** 需要人工在真正互動式終端機（非本 agent 的背景 shell）重跑 Step 5 的三行 `security` 指令並親眼觀察是否跳出鑰匙圈存取對話框，才能得出結論。

---

## 三個假設總覽

| # | 假設 | 結果 |
|---|------|------|
| 1 | `SSH_ASKPASS_REQUIRE=force` 攔截 password 認證提示，零互動 | **成立（YES）**——見 Step 3 |
| 2 | Tauri 打包後的 bundle 自我啟動為 helper 可乾淨退出 | 本 task 不驗證，留給 Task 3 |
| 3 | 未簽章 app 存取自建 keychain 項目的提示行為 | **無法驗證（BLOCKED）**——見 Step 5，需人工於互動式終端機重跑 |

**額外發現（非原三假設之一，但影響 Task 2/4 設計）**：`SSH_ASKPASS_REQUIRE=force` 對所有 askpass 風格提示（password **與** host-key 信任確認）都會生效；正式 askpass 替身必須以白名單只回答「password:」形狀的提示，其餘一律拒答，否則會像本次實測一樣對 host-key 提示造成無限迴圈。

## 清理

```bash
docker rm -f sshelter-spike
```

（於下方 git commit 前執行；keychain 未建立任何項目，無需清理；未修改 `~/.ssh/config` 或 `~/.ssh/known_hosts`。）
