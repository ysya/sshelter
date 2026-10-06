# SP3 金鑰插槽:手動驗證清單(Mac + Windows)

前置:兩台都在 Beta 頻道、已加入同一個同步帳戶、都勾選 Personal。Mac 有 `~/.ssh/id_mac`、`~/.ssh/id_mac2`,Windows 有
`~/.ssh/id_win`,這些公鑰都已加到測試伺服器。Mac 更新之前,Personal 裡已經有一台主機 `web`(`IdentityFile ~/.ssh/id_mac`),
Mac 的 `~/.ssh/config` 裡(不在 Personal 裡)另有一台本機主機也用 `~/.ssh/id_mac`(給 5 用)。先只把 Mac 更新到含 SP3 的 beta,
Windows 暫時留在 0.17.0-4。

1. **升級後的詢問**(Mac):Personal 已有用 `~/.ssh/id_mac` 的主機 `web`。重開 app → 跳出「Keys used by synced hosts」,
   列出 `web uses id_mac.` 與 `web: IdentityFile ~/.ssh/id_mac → ~/.ssh/sshelter/keys/id_mac-…`。按「Later」→
   Settings → Sync 出現「1 key used by synced hosts isn't set up」。再重開 app,不再自動跳出。
2. **Sync key**(Mac):Settings 那一列按「Set up…」→「Sync key」→ toast「id_mac syncs to your other computers」。`web` 那一行
   變成 `IdentityFile ~/.ssh/sshelter/keys/id_mac-xxxxxxxx`,該路徑是指向 `~/.ssh/id_mac` 的 symlink;`ssh web` 連得上。
3. **舊版電腦**(Windows 仍是 0.17.0-4):同步之後 `web` 在 Windows 連不上,lint 顯示 `IdentityFile not found`(預期;所以
   release notes 要提醒每台都更新)。把 Windows 更新到含 SP3 的 beta → 下一輪自動落地:
   `%USERPROFILE%\.ssh\sshelter\keys\id_mac-xxxxxxxx` 存在,`icacls` 只列出自己的帳戶;`ssh web` 連得上。
4. **Keep on this computer**(Mac):在 Personal 新增主機 `api`,在編輯器加上 `IdentityFile ~/.ssh/id_mac2` 並儲存 → 立刻跳出
   對話框 →「Keep on this computer」。Windows 同步之後跳出「Keys for this computer」;側邊欄的 `api` 有鑰匙標記。按「Pick…」→
   選 `id_win` → toast「id_mac2 uses id_win on this computer」,標記消失,`ssh api` 連得上;該插槽是 hard link(或複製)。
   全部挑完之後,Settings → Sync 沒有殘留的提示。
5. **搬進 space**(Mac):把一台用 `~/.ssh/id_mac` 的本機主機用「Move to file」搬進 Personal → 不再詢問(這把金鑰已經有插槽),
   主機改指到同一個插槽。
6. **本機挑的優先**:Mac 在 Keys 對話框對 4 的插槽按「Sync this key」→ 先跳出確認「Sync id_mac2 to your other computers?」,
   說明這把金鑰有沒有 passphrase(沒有時是「No passphrase — your sync code and every joined computer can use this key once it syncs.」);
   按「Cancel」什麼都不變,再按一次、按「Sync key」→ Windows 的插槽不被換掉,Keys 對話框顯示
   「A synced key is available」→「Use the synced key」→ 插槽換成同步的金鑰;`id_win` 沒被動到。
7. **Stop syncing**(Mac):對 2 的插槽按「Stop syncing」→ toast「id_mac no longer syncs; computers that have it keep their copy」;
   Windows 的副本仍在、`ssh web` 照樣連得上。
8. **刪除副本**:Mac 在編輯器把用 2 的插槽的兩台主機(`web` 與 5 搬進來的那台)的 `IdentityFile` 那一行刪掉(或停用成註解)並儲存。
   不要改回 `~/.ssh/id_mac`:那樣「Keys used by synced hosts」會自動沿用同一個插槽,又把那一行改寫回插槽路徑。→ Windows 同步之後,
   Keys 對話框顯示「Not in use」→「Delete copy」→ 確認對話框說「The copy of id_mac on this computer is deleted. Other computers aren't
   affected.」→「Delete copy」→ 檔案刪除。
9. **金鑰換了**(只看狀態;伺服器換上新公鑰之前 `api` 連不上):Mac 用 `ssh-keygen -f ~/.ssh/id_mac2` 重新產生 6 已改成同步的
   那把金鑰,切回 app → Keys 對話框顯示
   「This computer's key changed — your other computers still have the previous one」→「Sync the new key」→ 確認對話框說明的是
   新產生的那把有沒有 passphrase →「Sync key」→ Windows 顯示
   「A synced key is available」→「Use the synced key」→ 舊的副本留成 `<file>.previous-xxxxxxxx`。
10. **擋路的檔案**:Mac 先用 `ssh-keygen` 產生 `~/.ssh/id_mac3`,在 Personal 新增主機 `blocked`,在編輯器加上
    `IdentityFile ~/.ssh/id_mac3` 並儲存 → 對話框 →「Keep on this computer」,建立一個新插槽。Windows 同步之後跳出
    「Keys for this computer」,按「Done」略過(不要挑金鑰:挑了,插槽路徑上就是那把金鑰的連結,不會被擋);Windows 在那個插槽路徑
    (`blocked` 的 `IdentityFile`,`%USERPROFILE%\.ssh\sshelter\keys\id_mac3-xxxxxxxx`)放一個自己的檔案;Mac 按
    「Sync this key」、在確認對話框按「Sync key」→ Windows 的 Keys 對話框顯示「A file SSHelter didn't create is in the way: … Move it, then sync again.」,
    檔案沒被改。移走之後下一輪落地。
11. **更換同步碼**(Mac):確認對話框多一條「Keys you synced stay on every computer that has them. …」;完成畫面顯示「If a computer
    was lost, also replace these synced keys on your servers: …」。Windows 以新同步碼重新加入後,插槽與副本都在。
12. **離開帳戶**(Windows):先把 9 重新產生的 `~/.ssh/id_mac2.pub` 加到測試伺服器,確認 Windows 上 `ssh api` 連得上(用的是 9 之後
    同步來的新副本)。再 Leave:Leave 對話框有「Keys in ~/.ssh/sshelter/keys stay on this computer.」;離開後 `ssh api` 照樣連得上,
    `%USERPROFILE%\.ssh\sshelter\keys\` 裡的插槽檔都還在。
13. **Windows 的路徑**:Windows 上對一台主機做 Deploy key(寫入 IdentityFile)→ 寫進去的是 `~/.ssh/...`,不是 `C:\Users\...`。
14. **lint**:手動把主機指到一個帳戶裡沒有的插槽(例如 `IdentityFile ~/.ssh/sshelter/keys/nothing-00000000`)→ lint 顯示
    「IdentityFile not found: … (a key slot your sync account doesn't have — set the key up on the computer that has it)」。帳戶裡有、
    這台卻還沒有金鑰的插槽顯示「IdentityFile not found: … (a synced key slot — pick a key for it in Keys)」(16 的 Windows 在挑金鑰之前,
    `lab` 就是這樣)。
15. **PEM 金鑰**:用一把舊式 PEM 金鑰(`ssh-keygen -m PEM`)的主機 → 對話框的「Sync key」不能按,說明
    「This key isn't in the OpenSSH format, … Convert it with ssh-keygen -p -f <file>, or keep it on this computer.」;
    「Keep on this computer」照常可用。
16. **從 Settings 開對話框**:16 到 18 兩台都要在帳戶裡(Windows 在 12 離開了:用 Mac 的 Account → Sync code → Show 看到的
    同步碼重新加入)。重新加入之後、勾選 Personal 之前(先不勾,按一次 Sync now):12 搬到 `sshelter-local\` 的主機還用著那些插槽,
    `ssh api` 照樣連得上;Keys 對話框裡 `id_mac2`、`id_mac3` 的插槽是「Ready」並列出 `api`、`blocked`(不是「Not in use」,沒有
    「Delete copy」)。然後勾選 Personal,等第一輪同步完成:`web`、`api`、`blocked` 在 Personal 與 `sshelter-local\` 的舊檔裡各有一份
    (預期:同名的主機有兩份時 SSHelter 不改寫它們,編輯器會說明;舊檔留著,19 會用到)。Mac 先用 `ssh-keygen` 產生 `~/.ssh/id_mac4` 與 `~/.ssh/id_mac5`,
    在 Personal 新增主機 `lab`,在編輯器加上 `IdentityFile ~/.ssh/id_mac4` 並儲存 → 跳出對話框 →「Keep on this computer」。
    再新增主機 `lab2`,加上 `IdentityFile ~/.ssh/id_mac5` 並儲存,用頁尾的「Close」關掉對話框 → Settings → Sync 出現
    「{N} keys used by synced hosts aren't set up」(N 是還沒設定的金鑰數,15 沒處理的 PEM 金鑰也算在內;只有 `id_mac5` 時是
    「1 key used by synced hosts isn't set up」)→ 按「Set up…」→「Keys used by synced hosts」對話框開在 Settings 的上面
    (頁尾是「Later」),可以操作:在 `id_mac5` 那一列按「Sync key」→ toast「id_mac5 syncs to your other computers」,
    `lab2` 的 `IdentityFile` 變成 `~/.ssh/sshelter/keys/id_mac5-xxxxxxxx`,那一列消失(沒有別的列時對話框自己關掉),
    回到還開著的 Settings → Sync,列上的數字少 1(變成 0 就整列消失)。
    Windows 同步之後跳出「Keys for this computer」,按「Done」略過 → Settings → Sync 出現
    「{N} key slots need a key on this computer」(N 是這台還需要挑金鑰的插槽數;只有 `id_mac4` 時是
    「1 key slot needs a key on this computer」)→ 按「Pick…」→ Keys 對話框開在 Settings 的上面,對 `id_mac4` 按
    「Pick a key on this computer…」→ 選 `id_win` → toast「id_mac4 uses id_win on this computer」;關掉 Keys 對話框後
    回到還開著的 Settings → Sync,列上的數字少 1(變成 0 就整列消失)。
17. **很多插槽時的版面**(Mac 與 Windows 各做一次):帳戶裡至少有 4 個插槽(2、4、10、16 建立的 `id_mac`、`id_mac2`、`id_mac3`、
    `id_mac4`、`id_mac5` 共 5 個,15 可能再多一個)。開 Keys 對話框,視窗在預設大小(1100×720)與拖到最小(800×560)各看一次:
    對話框不比視窗高,上下緣都在視窗裡,內容太多時對話框自己可以捲動(「New key」區塊捲得到);「Keys used by synced hosts」的
    清單在自己裡面捲動,捲到最後一列,每一列的名稱、狀態、檔案路徑與所有按鈕都完整顯示、沒有被切掉(按鈕多時換行)。
18. **錯過更換的電腦不能停止同步**(兩台都在帳戶裡,`id_mac5` 是同步的插槽):Mac 做 Account → Sync code → Change…,等更換做完、
    存好新同步碼(做法同 11)。Windows 還沒輸入新同步碼時(Sync now 之後 Status 是「Paused」),Keys 對話框對 `id_mac5` 按
    「Stop syncing」→ toast「Could not change how the key is shared」,說明
    「the sync code was changed on another device; enter the new sync code first」,插槽仍是「Synced to your computers」
    (對 `id_mac4` 按「Sync this key」、在確認對話框按「Sync key」也一樣被擋,說明相同)。Windows 在「Enter the new sync code」那一列輸入新同步碼
    (「Use the new sync code」,toast「Syncing again with the new sync code」)後再按「Stop syncing」→ toast
    「id_mac5 no longer syncs; computers that have it keep their copy」;Mac 同步之後 `id_mac5` 顯示
    「Each computer uses its own key」,Windows 的副本仍在。
19. **搬出 space 的主機**(兩台都在帳戶裡):Mac 用「Move to file」把 `api` 搬到 `~/.ssh/config`(Personal 裡還有別的主機)→
    同步之後 Mac 上 `ssh api` 照樣連得上,`~/.ssh/sshelter/keys/id_mac2-xxxxxxxx` 還在;Keys 對話框裡 `id_mac2` 是「Ready」並列出
    `api`(不是「Not used on this computer」),lint 沒有「IdentityFile not found」。Windows 同步之後 Personal 裡沒有 `api` 了,但 16
    留在 `sshelter-local\` 舊檔裡的 `api` 還用著同一個插槽:Windows 的 Keys 對話框裡 `id_mac2` 仍是「Ready」(不是「Not in use」,
    沒有「Delete copy」),`ssh api` 照樣連得上。
20. **之前的帳戶留下的插槽**(兩台都在帳戶裡;`web` 在 8 之後沒有 `IdentityFile`,在 Windows 上又有兩份,所以這裡用新的主機):Mac 先用
    `ssh-keygen` 產生 `~/.ssh/id_mac6`(公鑰加到測試伺服器),在 Personal 新增主機 `web6`,在編輯器加上 `IdentityFile ~/.ssh/id_mac6`
    並儲存 → 對話框 →「Sync key」;Windows 同步之後落地它的副本,兩台的 `ssh web6` 都連得上。兩台都 Leave(`web6` 都搬到
    sshelter-local,照樣連得上)。Mac 在「Create a sync account」按「Create」建立新帳戶;Windows 用 Mac 這時顯示的新同步碼加入、勾選
    Personal,等第一輪同步完成。Mac 用「Move hosts into a space」把 `web6` 搬進新帳戶的 Personal → 跳出「Keys used by synced hosts」,
    `id_mac6` 那一列是「web6 uses id_mac6.」,問題下面是「From your previous sync account. Its hosts keep using
    ~/.ssh/sshelter/keys/id_mac6-xxxxxxxx.」,沒有「Rename」,也沒有要改寫的行 → 按「Sync key」→ toast「id_mac6 syncs to your other
    computers」;`web6` 的 `IdentityFile` 一個字都沒變,Keys 對話框裡 `id_mac6` 是「Ready」、「Synced to your computers」。Windows 還留著
    舊帳戶同步來的副本:Mac 按「Sync key」之前,Windows 的 Keys 對話框已經列著 `id_mac6`(「Ready」,但沒有任何動作,它還不在新帳戶裡),
    也不會問要不要同步它(它的 `web6` 在 Personal 與 `sshelter-local\` 各有一份,同名主機有兩份時 SSHelter 不改寫);同步之後 `id_mac6`
    是新帳戶的同步插槽,Windows 直接用那份副本(「Ready」,不另外落地),`ssh web6` 連得上,lint 沒有「IdentityFile not found」。
