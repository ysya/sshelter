# SP3 金鑰插槽:手動驗證清單(Mac + Windows)

前置:兩台都在 Beta 頻道、已加入同一個同步帳戶、都勾選 Personal。Mac 有 `~/.ssh/id_mac`、`~/.ssh/id_mac2`,Windows 有
`~/.ssh/id_win`,這些公鑰都已加到測試伺服器。先只把 Mac 更新到含 SP3 的 beta,Windows 暫時留在 0.17.0-4。

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
6. **本機挑的優先**:Mac 在 Keys 對話框對 4 的插槽按「Sync this key」→ Windows 的插槽不被換掉,Keys 對話框顯示
   「A synced key is available」→「Use the synced key」→ 插槽換成同步的金鑰;`id_win` 沒被動到。
7. **Stop syncing**(Mac):對 2 的插槽按「Stop syncing」→ toast「id_mac no longer syncs; computers that have it keep their copy」;
   Windows 的副本仍在、`ssh web` 照樣連得上。
8. **刪除副本**:Mac 把用 2 的插槽的主機(`web` 與 5 搬進來的那台)都改成不用插槽 → Windows 的 Keys 對話框顯示「Not in use」→
   「Delete copy」→ 確認 → 檔案刪除。
9. **金鑰換了**(只看狀態;伺服器換上新公鑰之前 `api` 連不上):Mac 用 `ssh-keygen -f ~/.ssh/id_mac2` 重新產生 6 已改成同步的
   那把金鑰,切回 app → Keys 對話框顯示
   「This computer's key changed — your other computers still have the previous one」→「Sync the new key」→ Windows 顯示
   「A synced key is available」→「Use the synced key」→ 舊的副本留成 `<file>.previous-xxxxxxxx`。
10. **擋路的檔案**:Mac 以「Keep on this computer」建立一個新插槽;Windows 在那個插槽路徑(主機的 `IdentityFile`)放一個自己的
    檔案;Mac 改成「Sync this key」→ Windows 的 Keys 對話框顯示「A file SSHelter didn't create is in the way: … Move it, then
    sync again.」,檔案沒被改。移走之後下一輪落地。
11. **更換同步碼**(Mac):確認對話框多一條「Keys you synced stay on every computer that has them. …」;完成畫面顯示「If a computer
    was lost, also replace these synced keys on your servers: …」。Windows 以新同步碼重新加入後,插槽與副本都在。
12. **離開帳戶**(Windows):Leave 對話框有「Keys in ~/.ssh/sshelter/keys stay on this computer.」;離開後 `ssh web` 照常。
13. **Windows 的路徑**:Windows 上對一台主機做 Deploy key(寫入 IdentityFile)→ 寫進去的是 `~/.ssh/...`,不是 `C:\Users\...`。
14. **lint**:手動把主機指到一個不存在的插槽 → lint 顯示「IdentityFile not found: … (a synced key slot — pick a key for it in Keys)」。
15. **PEM 金鑰**:用一把舊式 PEM 金鑰(`ssh-keygen -m PEM`)的主機 → 對話框的「Sync key」不能按,說明
    「This key isn't in the OpenSSH format, … Convert it with ssh-keygen -p -f <file>, or keep it on this computer.」;
    「Keep on this computer」照常可用。
16. **從 Settings 開對話框**:16 到 18 兩台都要在帳戶裡(Windows 在 12 離開了:用 Mac 的 Account → Sync code → Show 看到的
    同步碼重新加入、勾選 Personal,等第一輪同步完成)。Mac 先用 `ssh-keygen` 產生 `~/.ssh/id_mac3` 與 `~/.ssh/id_mac4`,
    在 Personal 新增主機 `lab`,在編輯器加上 `IdentityFile ~/.ssh/id_mac3` 並儲存 → 跳出對話框 →「Keep on this computer」。
    再新增主機 `lab2`,`IdentityFile ~/.ssh/id_mac4`,儲存後用頁尾的「Close」關掉對話框 → Settings → Sync 出現
    「1 key used by synced hosts isn't set up」→ 按「Set up…」→「Keys used by synced hosts」對話框開在 Settings 的上面
    (頁尾是「Later」),可以操作:按「Sync key」→ toast「id_mac4 syncs to your other computers」,`lab2` 的 `IdentityFile`
    變成 `~/.ssh/sshelter/keys/id_mac4-xxxxxxxx`,對話框自己關掉,回到還開著的 Settings → Sync,那一列消失。
    Windows 同步之後跳出「Keys for this computer」,按「Done」略過 → Settings → Sync 出現
    「1 key slot needs a key on this computer」→ 按「Pick…」→ Keys 對話框開在 Settings 的上面,對 `id_mac3` 按
    「Pick a key on this computer…」→ 選 `id_win` → toast「id_mac3 uses id_win on this computer」;關掉 Keys 對話框後
    回到還開著的 Settings → Sync,那一列消失。
17. **很多插槽時的版面**(Mac 與 Windows 各做一次):帳戶裡有 4 個插槽(2、4、16 建立的 `id_mac`、`id_mac2`、`id_mac3`、
    `id_mac4`;少了就再建)。開 Keys 對話框,視窗在預設大小(1100×720)與拖到最小(800×560)各看一次:對話框不比視窗高,
    上下緣都在視窗裡,內容太多時對話框自己可以捲動(「New key」區塊捲得到);「Keys used by synced hosts」的清單在自己裡面
    捲動,捲到最後一列,每一列的名稱、狀態、檔案路徑與所有按鈕都完整顯示、沒有被切掉(按鈕多時換行)。
18. **更換同步碼期間的拒絕**(兩台都在帳戶裡,`id_mac4` 是同步的插槽):Mac 停掉 relay,Account → Sync code → Change… →
    狀態列停在「Sending this computer's changes… You can still cancel.」。Keys 對話框對 `id_mac4` 按「Stop syncing」→
    toast「Could not change how the key is shared」,說明「finish or cancel changing the sync code first」,插槽仍是
    「Synced to your computers」(對 `id_mac3` 按「Sync this key」,或在對話框用「Sync key」建立新插槽 → 同樣被擋,說明相同;
    建立時的 toast 是「Could not set up the key」)。啟動 relay,讓更換做完、存好新同步碼。Windows 在輸入新同步碼之前
    (Sync now 之後 Status 是「Paused」)對 `id_mac4` 按「Stop syncing」→ 同一個 toast,說明
    「the sync code was changed on another device; enter the new sync code first」,插槽沒變。Windows 輸入新同步碼
    (「Syncing again with the new sync code」)後再按「Stop syncing」→ toast「id_mac4 no longer syncs; computers that have it
    keep their copy」;Mac 同步之後 `id_mac4` 顯示「Each computer uses its own key」,Windows 的副本仍在。
