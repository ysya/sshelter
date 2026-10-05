# Sync v2 — B4(前端:Settings → Sync、spaces、核准、側邊欄、搬移精靈、手動清單)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **狀態:已全部執行**(branch `feat/sync-v2-spaces`,`169141c`…`5b55188`,含合併 main 的 `ba0461b`;結束時 `src/` 29 個測試檔、
> 466 個測試,Rust 749 個)。每個 task 開頭的說明列出它的 commit 與審查後改了什麼;整體審查、最終修正與殘留修正見文末
> 「最終審查與修正(已執行)」。下面的程式碼區塊是執行前的計畫,實際的程式碼以 repo 為準;執行中也為五個元件加了 server render
> 的測試(沒有 DOM、不加相依),不再只測純函式。

**Goal:** 把 v1 的 Sync UI 換成 spec §8 的 v2 UI —— 帳戶、同步碼(查看 / 更換 / 別台更換後輸入新碼)、relay 版本與
更新提示、裝置、spaces(開關、改名、刪除、新增、重建)、等待核准的審核、側邊欄依 space 分組與同名標示、搬移精靈
(目標 space、一個來源檔一個 space、不能搬的主機)、v1 升級說明 —— 並接上 B3 後端的全部 commands 與事件;最後補上
README 與兩台電腦的手動驗證清單(spec §10)。

**Architecture:** 沿用現有前端的分層。`src/lib/sync.ts` 以 TanStack Query 包「非祕密」的 commands(回傳
`SyncOverview` 的動作一律先寫進快取);同步碼相關的呼叫(建立、加入、重新加入、查看)繞過 TanStack Query,字詞只放
在元件 state。事件接線抽成 `src/lib/sync-events.ts` 的 `subscribeSyncEvents`(以假的 Tauri event bus 在 Node 測試)
與 `useSyncEvents`。畫面需要的判斷 —— 狀態列、relay 提示、更換同步碼能不能開始、space 列、名稱規則、核准差異、側邊欄
標示、精靈的分組與命名 —— 全是 `src/lib/sync-*.ts` 的純函式,以 vitest 覆蓋;元件照既有慣例不寫測試,只組裝純函式與
既有的 UI primitives。對話框的開關放在 `useUiStore`(session-only),後端資料只在 TanStack Query。

**Tech Stack:** React 19 + TypeScript 5.8、TanStack Query 5、Zustand 5、既有的 shadcn / Radix 元件(`src/components/ui/*`)、
`sonner`、`lucide-react`、`@tauri-apps/api`(`invoke`、`event.listen`)、`@tauri-apps/plugin-opener`;測試:vitest 4
(Node 環境,沒有 DOM)。不新增任何相依。

**Spec:** `docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md`(§8 前端;§7.2–§7.6 UI 驅動的流程;§9 使用者
看到的錯誤;§10 前端測試與手動清單)。後端介面以 `docs/superpowers/plans/2026-10-02-sync-v2-b3b-engine-wiring.md` 文末
「B4 handoff」為準,並沿用它與 `docs/superpowers/plans/2026-10-02-sync-v2-b3a-engine-core.md` 的「對 spec 的解讀」。

## Global Constraints

- 前置:B2、B3a、B3b 都已執行,含 B3b 的最終修正(repo `d469bc0` 的實際程式碼;B3 計畫與它的「B4 handoff」以 repo
  `2e35212` 為準,含採納的 B3 amendments A1–A3):`src/bindings/` 已有 `SyncOverview`、`ReviewedVersion`
  (`{ alias, digest }`)、`ReviewOutcome` 等型別,`PendingApprovalView` 帶這一版的內容指紋 `digest`,`SyncNotice` 含
  `left_account`,`upgraded` 帶 `moved_files`;`src-tauri/src/sync/approval.rs`
  的 `GATED_KEYWORDS` 有 24 個,`LocalForward` / `DynamicForward` 除了只有埠號的平常寫法也受管制;v1 的 Sync commands
  已移除。本計畫只改前端與文件,不碰 `src-tauri/`(執行時有兩個經同意的例外,`bf37fe6`、`7b5d1c9`:後端多拒絕幾類會在畫面上
  看不出來的字元,見 Task 4 的說明與文末)。
- UI 字串、識別字、錯誤訊息、commit 訊息一律英文。註解沿用每個檔案既有的語言:本計畫碰到的前端檔案註解都是英文
  (`src/stores/ui.ts` 的欄位註解是英文,檔頭那段中文不動),新檔案也用英文。
- 不新增 npm 相依(`package.json`、`pnpm-lock.yaml` 不變)。
- **同步碼(24 詞)只存在元件 state**:不進 TanStack Query 的快取(不以 `useQuery` / `useMutation` 承載字詞)、不進
  zustand store、不進 localStorage、不寫 log、不進 toast(複製到剪貼簿的 toast 只說「已複製」)。`createAccount`、
  `joinAccount`、`rejoinAccount`、`showWords` 是直接呼叫 `tauriInvoke` 的函式,錯誤原樣丟回呼叫端;對話框關閉就丟掉字詞。
- 視窗焦點 → 同步由後端處理(`WindowEvent::Focused`;relay 要求退避時,回到前景的喚醒等退避結束才跑):移除前端
  `focus` → `sync_now` 的 listener,前端任何地方都不因焦點呼叫 `sync_now`(它不受退避限制,會繞過退避,也讓 Cloudflare
  免費方案的 relay 用量加倍);只有使用者按「Sync now」才呼叫。Task 1 的
  `listens to the engine's events and never asks for a sync round itself` 釘住事件這一側。
- 文案必須準確:「Forget」只把電腦從清單移除,**不是撤權**(撤權 = 更換同步碼);關掉(取消勾選)一個 space 只移除
  這台的檔案;刪除 space 會在每一台電腦上移除;更換同步碼需要每一台其他電腦輸入新碼,且 relay 要支援 `freeze` ——
  不支援時「Change…」停用、說明原因並附 README「Updating your relay」的連結;`relay.batch_pull === false` 時顯示
  「This relay can be updated…」與同一個連結(`RELAY_UPDATE_URL`)。
- 刪除帳戶(離開時一併刪除 relay 上的帳戶與所有 space)只在這台是裝置清單上唯一的電腦、同步碼沒有在別台被更換
  (不是 `frozen`)、而且沒有過了凍結的更換同步碼時提供(`deleteAccountNote`)。
- Conventional Commits;每個 task 一個 commit,只 `git add` / `git rm` 該 task 列出的路徑;不得 stage `.superpowers/`、
  任何 `Cargo.lock` 或其他無關檔案。
- 每個 task 結束時 `pnpm exec vitest run --dir src` 與 `pnpm exec tsc --noEmit` 都必須通過才 commit(測試數只算 `src/`:
  單純的 `pnpm test` 也會跑 `scripts/` 與 `.claude/worktrees/` 底下的舊 worktree,總數不同);Task 7 另跑 `pnpm build`
  (`tsc && vite build`)。repo 開著 `noUnusedLocals` / `noUnusedParameters`:不留沒用到的 import 或變數。
- `src/bindings/*.ts` 由 ts-rs 產生,不手改;不再產生的 `SyncStatus.ts`、`SyncDevice.ts` 在 Task 6 最後一個使用者
  消失時刪除。
- 本計畫以 repo `d469bc0` 的後端為準(B3b 已全部執行);B4 的七個 task 都已執行(Task 1 `2def6a2`、`32cae39`,其餘見各 task
  開頭的說明),程式碼以 repo 為準。之後若後端再有小修正(例如錯誤訊息字串),只調整相關的文案或
  測試資料,不改語意;缺 command 或欄位就停下來回報(見「B3 amendments」)。前端在三處依後端的文字分辨情況,都有測試
  直接讀 Rust 原始碼對照:`leaveFailureTitle`(`account.rs`)、`SWAP_PENDING_MESSAGE` 與 `CHANGE_CANNOT_FINISH_MESSAGE`
  (`rotation.rs`),以及搬移精靈把「搬了但標記寫不進去」算成已搬的 `TAG_FAILED_PREFIX`(`migrate.rs`);其他錯誤一律原樣顯示。

## Review Focus

1. **同步碼貼成多行、帶編號、大小寫混雜、全形空白,或打錯 / 貼了舊碼**:前端先整理成單行再送;失敗時保留輸入讓
   使用者修正;字詞不進 toast、快取或 store(Task 1 `sync-code commands` 的
   `send the words and device name as the backend's camelCase arguments`、
   `pass backend errors through untouched and never toast (a toast could carry the words)`;既有的 `cleanWordsInput`
   測試在 Task 6 的檔案裡保留)。
2. **審核對話框開著時又來了較新的版本**:畫面上的版本必須就是送出去的版本 —— 對話框只送它顯示的 `{ alias, digest }`
   (`digest` 是那一版內容的指紋:內容變了就不同,同一版之後再被拉下來則不變),顯示的清單在使用者決定之前不會被換掉
   (只提示「Newer versions arrived」);後端回報 `changed` 時說「changed since you opened this — review it again」並換上
   新版本(Task 1 `review commands` 的
   `send exactly the reviewed versions as { spaceId, approvals: [{ alias, digest }] } and return the outcome`;Task 4
   `deciding on exactly the versions shown` 的三個測試)。
3. **審核對話框把差異講錯,或被字元騙**:`Keyword=value` 寫法、大小寫不同、註解掉的行、同名指令換順序、Host 行
   擴大、新主機、B2 審查後新增的受管制設定(`RemoteCommand`、`StrictHostKeyChecking`、`GatewayPorts` 等)、只有帶綁定
   位址才受管制的 `LocalForward` / `DynamicForward`,以及值的註解部分夾著 bidi 或零寬字元 —— 使用者看到的變更必須與
   後端的簽章一致,畫面也不能被重新排序(Task 4 的 `lineKeyword`、`blockLines`、`approvalChanges`、`revealHidden`
   測試,含 `marks a forward the backend gates (a bind address) but not one with only a port, and labels it`、
   `reveals bidi and zero-width characters in the block and in the changes`,以及 `GATED_KEYWORDS` 對照
   `approval.rs` 的 `is the backend's list (approval::GATED_KEYWORDS)`)。
4. **「一個來源檔建立一個 space」撞名與重複**:兩個檔案都叫 `config`、sidebar alias 等於既有 space 名稱、名稱超過 64 字元
   —— 新名稱不分大小寫地唯一、不超過上限,按下之前就顯示;同一台主機定義在兩個檔案(或以另一個名稱出現)只送一次,
   後端不會回「listed in more than one group」(Task 6 `uniqueSpaceName`、`one space per file` 的測試,含
   `sends each host once: …`)。
5. **離開帳戶之後,這台的檔案去了哪裡;更換同步碼期間的離開**:離開(或放棄卡住的 v1 升級)會把 space 檔搬到
   `~/.ssh/sshelter-local/`,留下 `left_account` 提示 —— 這時已經是未加入的畫面,提示必須在那裡列出檔案、可以清掉,toast
   也要說清楚;v1 升級說明則只出現在對話框。更換同步碼期間「Leave…」不停用,對話框說明離開對這次更換的影響;這台已經
   離開之後才回來的錯誤,toast 標題是「Left the sync account on this computer」而不是「Could not leave」(Task 1
   `have a title and a description for every kind`、
   `toasts notices with a way to Settings → Sync, except the upgrade (it has its own dialog)`、
   `titles every error that came after this computer already left as such (account.rs)`、
   `keeps the failure title for the errors where this computer did not leave (account.rs)`;Task 2 `noticeRows` 的
   `lists them on a computer that left, too, so it can tell where its files went`,以及 `leaveRotationNote`、`deleteAccountNote`)。

## 對 spec 的解讀(實作時的決定)

1. **加入後勾選 space**(spec §8「加入後勾選 space」、B3a 解讀 1「join 不勾選任何 space」):加入成功後打開「Choose
   spaces for this computer」,列出帳戶的每個 space(預設全勾,顯示哪些電腦在同步它),按下後逐一 `sync_select_space`;
   之後同 v1 接著開搬移精靈(目標 = 第一個勾選的 space)。「Not now」不勾選,之後在 Spaces 區塊打開。
2. **「一個來源檔建立一個 space」的名稱**(B3b 解讀 13):sidebar 的檔名 alias → `tagForFile`(移植自後端
   `migrate::tag_for_file`)→ `Space`;與帳戶裡的 space 或同一次的其他新名稱撞名時(不分大小寫)加 ` 2`、` 3`…,總長
   不超過 64 字元。精靈在按下之前就顯示每個檔案的新 space 名稱,後端因此不會因撞名讓整組主機失敗。檔名 alias 含控制字元時
   不拿來當名稱(後端會拒絕整組),改用 `tagForFile`(執行後的修正)。
3. **側邊欄的 space 群組**(spec §8):勾選的 space 檔以 space 名稱為標籤,蓋過使用者的檔名 alias(名稱屬於帳戶);
   標題加雲朵圖示;不能在側邊欄雙擊改名(改名在 Settings → Sync,會改每一台的檔名),右鍵選單改成「New host in this
   space」與「Manage spaces…」。執行後另外:space 有問題(資料不見、檔案被拒而暫停、上傳被拒)時標題有警示記號,tooltip 是
   那個問題(`spaceFileProblems`,與狀態列共用 `spaceProblem`);雙擊標題不會收合;space 名稱是別台取的,標題、選單、記號與
   toast 都經 `revealHidden`;Add host 與編輯器的 Move to file 也以 space 名稱列出 space 檔。
4. **同名主機的標示與保護**(spec §4.3,執行中於 `52186dc` 改寫;§8「列尾標示」):ssh 套用每一個符合的 Host 區塊、每個
   設定取第一個讀到的值 —— 排在前面的區塊設定過的以它為準,只有後面那份設定的(包括 ProxyCommand 等受管制的設定)仍然
   生效,可累加的設定(IdentityFile、LocalForward、RemoteForward、DynamicForward、SendEnv)兩份都用。所有講同名主機的文字都照
   這個說(`SSH_COMBINES`),不說後面那份被忽略(執行前的版本寫「ssh 讀的是哪個 space 的」「勝出的那份」,低估了風險:最終
   審查 Important 1)。以 `sync_duplicate_aliases`(後端列出 ssh 第二個才讀、Host 行以這個名稱開頭、在別的檔案的那幾份)為準,
   在那幾份的列尾放琥珀色圖示(`shadowTooltip`:哪一份先讀、為什麼);該列的選單提供「Keep this copy as <alias>-local」與
   「Remove this copy (the one in <file> stays)」,都走以檔案定位的 `sync_resolve_shadowed`,先確認再做(`ShadowFixDialog`:
   另一個 space 的那份,改動會同步到那個 space 的每一台;自己設定檔的那份只動這台;另一份不動;先寫備份),完成後 toast。
   一個名稱只要有一份在勾選的 space 檔、又不只一份(任何 pattern 都算,同後端以名稱找主機的 `find_host_file_index`),就是
   有歧義的名稱(`ambiguousNames`):列以 (alias, 檔案) 選取;靠名稱找主機的動作(編輯器存檔、Remove…、改名、Move to file、
   拖曳、⌘/Shift 多選與批次動作、部署金鑰寫入 IdentityFile)在那個名稱的每一列都停用並說明;編輯器的位置改顯示唯讀的各份
   清單(`DuplicateCopies`:每一份在哪個檔案;只有勾選的 space 檔依 Include 順序排在前面,其他檔案不宣稱順序;ssh 怎麼合併;
   每一份可以怎麼處理 —— 沒有側邊欄動作的那幾份(`Host a web` 與 `Host web`、同一個檔案裡的兩個區塊)請用文字編輯器改那個
   檔案再 Reload from disk)。同步總覽還沒讀到或讀取失敗時,每個不只一份的名稱都當成有歧義(fail closed)。全部不在 space 檔
   的同名主機照舊。
5. **核准的通知與送出的版本**(spec §7.4「跳出通知」、B3b handoff「打開審核對話框」):`sync://approval` 跳出 15 秒的
   warning toast,按「Review」才打開審核對話框,不自動彈出 modal(使用者可能正在編輯);Settings → Sync 也有「N hosts
   waiting for your approval → Review…」。審核對話框掛在 App 層。對話框把打開時讀到的清單固定在畫面上,核准 / 拒絕只送
   這些版本的 `{ alias, digest }`(「全部」= 每個 space 一次、送出畫面上的全部);清單在背後變了(以 space、alias 與 `digest`
   比對:同一台主機換了內容就是新的 `digest`,同一版之後再被拉下來 `digest` 不變)就只顯示「Newer versions
   arrived while this was open」與「Show them」。每次決定之後重讀 `sync_pending_approvals` 並換上最新清單;
   `ReviewOutcome.changed` 非空時說「<alias> changed since you opened this — review it again」,空的就不多說。
   執行後(Task 4 的兩輪修正與最終修正):換上新清單時,對話框開著時換了內容(`digest` 不同)或新出現的主機在自己的卡片上標
   「Changed since you opened this — review it again」/「New since you opened this」,直到使用者對那一台做了決定
   (`adoptNewest`),通知也列出它們與所在的 space(「web in Personal changed since you opened this — review it again.」);重開時等
   第一次重新讀取完成才定下清單(`isSettled`),不會從舊快取說「Nothing is waiting」;畫面上沒有主機時新來的直接採用並標成新的
   (`adoptAtOnce`);決定進行中不能關,「Close」在 footer 最右邊;清單被換掉、或清單上方的鎖定說明、通知、「Newer versions
   arrived」出現、消失或改變時,決定的按鈕停用 600 ms(`CLICK_GUARD_MS`、`listMoved`),免得游標下換成另一台主機的按鈕;
   帳戶鎖住時不能決定(解讀 11)。
6. **核准的差異怎麼顯示**(spec §8「標出受管制的行與簽章差異」):變更清單一律從後端的兩個簽章算(LCS,依順序比較,
   因為簽章依順序比較):新主機、Host 行改變(`Applies to: a → b`)、新增 / 移除 / 改值、只換順序(`Order changed: …`)。
   完整區塊中標色的行 = 關鍵字永遠受管制,或那一行出現在新簽章裡(不是只有埠號的 `LocalForward` / `DynamicForward` 不在
   `GATED_KEYWORDS`,靠簽章對到);改了的 Host 行也標色 —— 只用於顯示,決定權在後端的解析器。另可展開「This
   computer's current version」。來自別台的文字(區塊、變更、alias、裝置與 space 名稱)一律經 `revealHidden`:bidi
   控制字元、零寬與其他格式字元、tab 以外的控制字元、非 ASCII 空白都顯示成 `⟨U+202E⟩`;區塊與變更另以
   `unicode-bidi: bidi-override` 依邏輯順序由左到右顯示,右至左的文字也無法讓一行看起來和 ssh 讀到的不同。
   執行後(Task 4 的修正、`bf37fe6`、最終修正 Part B):`revealHidden` 一次掃描,另外顯示 Default_Ignorable(包括 U+3164 等
   韓文填充字元)、畫出來是空白的 U+2800、U+1D159、U+13441、U+13442、U+303F,以及沒有字母、數字或記號可以依附的組合記號
   (在空白、`=`、shell 符號、行首或另一個被顯示的字元之後);`hidden-parity.test.ts` 讀 `hosts_file.rs` 的拒絕區段,確認後端
   拒絕的每一個 code point 都會被顯示。變更清單保留空白(`whitespace-pre-wrap`)。不在 space 檔裡的主機寫「Not in <space> on
   this computer yet: Host …」而不是「新主機」(同一個 space 檔裡已有別的區塊帶這個名稱時是「A new block in <space>」);設定檔
   裡已有同名主機時依實際的讀取順序說誰先讀(不是 space 的檔案:「Takes over … (synced files are read first)」;別的 space:
   依 Include 順序),並照解讀 4 的合併方式說明這個區塊的哪些受管制設定在對方沒設定時仍然生效;Host 行有多個名稱(或萬用
   字元)時也標色。
7. **提示的 toast**:`sync://notice` 除了 `upgraded` 都跳 toast(附「Open」到 Settings → Sync);`upgraded` 由 App 層的
   一次性對話框說明(spec §8:space「Synced」可改名、其他電腦也要更新、留在本機的主機(`kept_hosts`)、使用者自己放在
   `~/.ssh/sshelter/` 而被搬到 `~/.ssh/sshelter-local/` 的檔案(`moved_files` 的新路徑,通常沒有);每一點一句),跨重啟保留到
   使用者關掉它:「Got it」或 Esc 先在本機關掉對話框,再送 `sync_dismiss_notice`、不等回應;被拒絕(第二個 SSHelter 行程
   從不存檔)時由錯誤 toast 說明,app 照常可用,下次啟動再顯示。Settings → Sync 列出
   `upgraded` 以外的提示 —— 已加入時在狀態區,未加入時(離開之後的 `left_account`)在「Notices」區塊;
   `new_sync_code` 的按鈕是「Show new sync code」,使用者確認存好新碼之後才清掉那則提示。`left_account` 列出搬到
   `~/.ssh/sshelter-local/` 的完整路徑;建立或加入帳戶時,v1 留下、主 config 的 Include(含 glob)還讀著的
   `~/.ssh/sshelter/hosts.config` 被搬開也是這一種(沒有 Include 讀它就留在原地、不提示,Settings → Sync 把它列在
   「Files SSHelter doesn't use」),所以文案不提「離開」,只說檔案還在 ssh 讀得到、不再同步,要同步就在帳戶裡用「Move hosts into a space」。
   更換同步碼的切換與重新加入時,新帳戶沒有接續的勾選 space 也改成本機檔案、留下這則提示,所以同一段文案在已加入的
   畫面也成立。`upgraded.kept_file` 在重跑時可能是 null(另存檔已沒有要留的主機):有檔案、也有主機時才說哪些主機留在本機。
   執行後:後端離開時不清提示,所以未加入時只有 `left_account` 與 `space_deleted` 照原文顯示;`rename_blocked`、`new_sync_code`、
   `other_rotation` 保留標題、內容改成「This was about the sync account this computer has since left.」,也不提供「Show new sync
   code」。「I have saved the new sync code」清掉的是確認當下那則提示的位置(`newSyncCodeNoticeIndex`)。
8. **搬移精靈列哪些主機**:不在任何勾選的 space 檔、所有 pattern 都具名、不在 `sync_unmovable_hosts`、且和**任何**
   勾選 space 裡的主機沒有共同名稱(v1 規則推廣到多個 space;同名的本機主機交給「同名主機」區塊處理)。全選只在每個
   勾選的 space 都是空的、且都完成第一輪時(例如剛建立帳戶);目標 space 第一輪還沒完成時「Move」停用並說明。執行後另外:
   資料不見(`missing`)的 space 列成「(needs rebuild)」、不能選、也不會預設選到(`isValidTarget`、`canMove`);查詢失敗時說明
   原因;搬移進行中不能關;同名主機區塊的兩個修正先確認(解讀 4)。
9. **狀態列的優先順序**:依序取第一個成立的 —— 先是帳戶層級的狀態:升級中、同步碼已在別台更換(Paused)、正在更換
   同步碼、唯讀、keychain 還沒收下新碼、這一輪的錯誤;再來是勾選的 space 的問題(資料不見,或那個 space 自己的 `last_error`:
   檔案被拒而暫停、寫入失敗、上傳被 relay 拒絕):一個時「<space>: <問題>」,好幾個時「N spaces need attention — see Spaces」
   (badge「Error」;`spaceProblem` 與 Spaces 列、側邊欄標題的警示共用;space 名稱經 `revealHidden`);然後是還沒同步過、有待
   上傳、最新。space 的問題讓那個 space 兩個方向都停止同步,帳戶本身卻正常,不能只出現在它自己那一列(最終審查 Important 2;
   執行前的版本沒有這一層,狀態列會在 space 停住時照樣寫「Synced」)。更換同步碼時的 `last_error` 接在步驟後面(「Copying your spaces — …」):
   後端每一步都自己重試(限流、relay 出錯或拒絕、檔案改不成本機檔案、keychain 上鎖、凍結前新碼不見),所以那是狀態、不是
   要使用者處理的錯誤(tone `busy`;凍結前新碼不見時同一區的「Cancel」照常可用);只有凍結之後新碼不見、這次更換永遠做
   不完(`CHANGE_CANNOT_FINISH_MESSAGE`,文字請使用者離開並建立新的同步帳戶)標成錯誤。切換完成、keychain 卻還沒收下新碼
   (`SWAP_PENDING_MESSAGE`,同步照常)只是說明:badge「Saving code」、tone `busy`,不是「Error」。
10. **同步碼的按鈕**:frozen 時不顯示「Show」與「Change…」(舊碼已沒用),改由狀態區的「Enter the new sync code」處理。
    正在更換時「Show」照常可用:第 1–6 步 `sync_show_words` 回的是舊碼,所以那一列說明它在凍結之後就不能用、新碼在更換
    完成時顯示(`syncCodeNote`);切換之後 keychain 還沒收下新碼時,後端回的是暫存的新碼。「Change…」的停用原因對應後端
    `account_ready` 與 `NO_FREEZE_MESSAGE`;keychain 還沒收下上一次的新碼時按下去,後端的 `SWAP_PENDING_BLOCKS_MESSAGE`
    原樣顯示。`relay === null` 是「還沒問過」,不是「不支援 freeze」:不停用「Change…」,Account 區塊看到時自動呼叫一次
    `sync_check_relay`(`useCheckUnknownRelay`;失敗不跳 toast,「Check again」仍在)。執行後另外:Show 開的對話框也照狀態
    說明(`shownCodeNote`:還能取消時「這仍是現在的同步碼,但凍結之後就不能用」,過了凍結「這是舊的同步碼,已經不能用」);
    更換同步碼的確認改說 SSHelter 開始之前會先確認 relay 支援凍結,出現阻擋原因時自動關閉;`changeCodeBlocker` 與
    `structureLock` 共用 `accountBlock` 的階梯(解讀 11)。
11. **space 的結構性操作與核准**(新增、改名、刪除、開 / 關、重建、搬進,以及核准 / 拒絕)在同一把帳戶鎖下停用並說明理由:
    `accountBlock` 照後端 `account_ready` 的順序判斷 —— 升級中、未加入、同步碼已在別台更換、正在更換同步碼(還能取消時說可以
    先取消)、唯讀 —— 由 `structureLock` 給文字。核准 / 拒絕在這些狀態下後端一樣拒絕,所以審核對話框在清單上方顯示
    「Approving and rejecting are off right now. <理由>」,每張卡片與 footer 的核准 / 拒絕都停用(Close 照常),Settings → Sync 的
    「N hosts waiting for your approval」那一列也寫出理由並停用「Review…」(執行前寫「核准 / 拒絕不受限」是錯的:最終審查
    Minor 1)。`changeCodeBlocker` 用同一個階梯。搬進 space 在後端不受 `account_ready` 限制,UI 仍一起鎖。keychain 上鎖時後端
    也會拒絕決定,但總覽看不出來:按鈕照常,失敗時由 toast 說明(決定不處理)。
12. **同名主機清單的查詢 key** 從 `["sync", "duplicates"]` 改成放在 hosts 的 key 底下:`["config", "hosts",
    "syncDuplicates"]`(執行前寫的 `["config", "syncDuplicates"]` 不夠:app 內的新增、存檔、改名、搬移、刪除只讓 hosts 失效,
    同名主機的標示就停在舊的 —— Task 5 審查的 Important,最終修正 `bae0baf`)。任何 config 的重載或編輯都可能改變誰遮蔽誰,
    它跟著 hosts 與 `["config"]` 失效;`useResolveShadowed` 寫入答回的清單,失效時排除這個 key。`sync_unmovable_hosts` 仍在
    `["config", "syncUnmovable"]`。
13. **README 的 Sync 段落**:v1 的說明(`hosts.config`、recovery phrase、「leave the chain and rotate keys」)在 B4 之後
    就錯了,Task 7 一併改寫(含 spec §7.6 第 6 點「仍在 v1 的電腦更新前看不到升級後的修改」)。
14. **離開的對話框**:說明這台的 space 檔會搬到 `~/.ssh/sshelter-local/`、照常是 ssh 讀得到的本機檔案,之後可以搬進
    另一個帳戶;不是最後一台時不提供刪除,說明「其他電腦繼續同步,要刪帳戶請在最後一台離開」;同步碼已在別台更換
    (`frozen`)時也不提供,說明舊帳戶要留在 relay 上讓還在用舊碼的電腦知道;更換同步碼過了凍結時也不提供(那時離開不是
    被拒絕,就是不碰 relay)(`deleteAccountNote`);最後一台時的勾選框預設不勾。搬不過去時後端什麼都不改並回錯誤;刪掉
    relay 上的帳戶之後檔案卻搬不過去也是錯誤 —— 都由 mutation 的錯誤 toast 原樣顯示,而且離開的 mutation 失敗時也重讀總覽
    (這台可能已經離開了)。更換同步碼期間「Leave…」不停用(總覽看不出暫存的新碼還在不在,由後端決定),對話框以
    `leaveRotationNote` 說明:凍結之前離開會先取消這次更換(之後檔案搬不過去時,錯誤是「could not keep this device's synced
    files as local files (…); the sync code change in progress was cancelled, nothing else was changed — try leaving again」);
    凍結之後後端拒絕「a sync code change is in progress; let it finish (it resumes on its own) before this computer leaves」,
    除非暫存的新碼不見了或不能用:那時照常離開(relay 上什麼都不刪,`deleteRemote` 也一樣)並回「left the sync account on
    this computer, but its sync code change could not be finished (the new sync code was missing from the keychain), so the
    old sync account can no longer be joined — create a new sync account on one computer, and on the other computers leave
    the old account (their synced files stay as local files) and join the new one」(`deleteRemote` 時再加「. The sync
    account was not deleted from the relay」)。這句、`frozen` 時要求刪除帳戶的「left the sync account on this computer,
    but did not delete it from the relay: …」(`LEAVE_REPLACED_MESSAGE`),以及離開之後同步碼或狀態清不掉時 `leave_account`
    自己回的「left the sync account, but the sync code could not be removed from the keychain (…); use "Remove sync code" to
    retry」與「left the sync account, but the sync state could not be saved (…); it will be retried automatically」,都是這台
    已經離開之後才回的錯誤:toast 的標題依開頭的「left the sync account」改成「Left the sync account on this computer」
    (`leaveFailureTitle`),內容原樣,並重讀總覽;沒有離開的錯誤開頭不同(「could not keep…」、「the sync account was
    deleted…」、「a sync code change is in progress…」),標題維持「Could not leave the sync account」。
    改名或刪除 space 在主 config 被 app 外修改時照樣成功(檔案那一半下一輪做完),前端不必特別處理。執行後另外:對話框說明
    還沒上傳的修改(`leaveUnsentNote`:勾選的 space 裡這台還沒上傳的主機修改 ——「N changes made here and not uploaded yet won't
    reach your other computers — the files this computer keeps still have them.」;只有帳戶記錄在等時不說);要不要一併刪除帳戶由
    `leaveRequest` 決定(有測試)。
15. **未加入時的升級外殼**(B3b 解讀 4):升級中 `joined` 是 false,而且被拒絕的方式不同(重新加入回 `UPGRADING_MESSAGE`;
    顯示同步碼、更換同步碼、精靈的組回「join or create a sync account first」),所以先看 `upgrading` 再看 `joined`:
    `statusLine` 第一個就判斷它,`NotJoinedPane` 在提供建立 / 加入之前判斷它,升級中只顯示升級狀態。卡住(有
    `last_error`)時提供「Stop syncing」(= `sync_leave_account`,放棄升級):主 config 的 Include 讀得到的 `~/.ssh/sshelter/`
    檔案(同步的主機就在其中)搬到 `~/.ssh/sshelter-local/`,沒被列出的 `hosts.config` 由後端備份後移除;按鈕旁的說明也
    這樣講。執行後:「Stop syncing」先確認;未加入時的升級狀態也用 `statusLine`,不再另寫一份。
16. **「不能同步」的原因一律用後端的訊息**:B2 Task 5 的安全修正之後,同步的主機除了 `Include`、帶引號的 keyword,
    還拒絕以 `=` 開頭的行、含非 ASCII 空白或控制字元的行,以及 `HostName`、`User`、`HostKeyAlias`、`ProxyJump` 的值
    不是「一個不含 shell 字元的詞」。前端不重複這份清單:精靈的「Can't be synced」與搬移結果、側邊欄拖曳的錯誤直接顯示
    後端的 `error`(精靈與拖曳共用同一套拒絕與文字),v1 升級說明只說哪些主機「could not move into a space」(原因也可能
    是 space0 已在別台刪除,B3b 解讀 2),README 只列主要規則。B3b 之後搬移結果裡可能出現的文字:「'web' is already in
    that space」/「'web' is already in that space — it is another name of the host 'a'」、「moved, but its tag could not
    be saved: <cause>」(那台其實搬了,只列在 `failed`)、「the space '<name>' was created, but the hosts could not be moved
    into it (<cause>) — move them into that space instead of creating it again」、「the relay is rate-limiting new spaces
    from this network, so no more are created now; try again in about an hour」、「the config changed on disk and could not
    be reloaded: <error>」。所以結果區塊的標題是「Problems with N hosts」(不說「not moved」),相同原因合併列出、可以捲動;
    toast 把「搬了但標記寫不進去」的主機算成已搬(`TAG_FAILED_PREFIX`,測試對照 `migrate.rs`),寫「Moved N hosts into …, M hosts
    with a problem」(一台都沒搬時「No host moved …」),有問題就用 warning(`moveSummary`,執行後的修正);沒有任何能搬的主機的組
    不建立 space,toast 因此不數 space。送出前 `newSpaceGroups` 去掉重複:前面的組已經
    送出的名字(alias,或已送出主機的另一個名稱)不再送,後端的「listed in more than one group」不會出現;space 名稱在
    B4 產生(解讀 2)。

## B3 amendments(已併入 B3 計畫)

驗證第一版時讀 B3 參考實作發現三處,已被採納並寫進 B3a / B3b 計畫;本計畫以修正後的後端為準,不再需要任何後端改動:

- **A1** 離開帳戶(與放棄卡住的 v1 升級)把這台的 space 檔搬到 `~/.ssh/sshelter-local/`(同名;撞名加 `-2`、`-3`),主
  config 裡的 token 原地換成一般的 `Include`,留下 `SyncNotice::left_account { kept_files }`(完整路徑);搬不過去時
  什麼都不改,錯誤是「could not keep this device's synced files as local files (…); nothing was changed — try leaving
  again」。B4:`noticeMessage` 的 `left_account` 分支、未加入時的「Notices」區塊、離開與「Stop syncing」的文案、手動
  清單第 2、15、16 項。
- **A2** crypto 的錯誤訊息改稱「sync code」(「a sync code has 24 words (got N)」、「invalid sync code: …」)。B4 沒有
  引用舊字串的文案或測試。
- **A3** 改名被擋的提示每個目標只出現一次(目標換了才再提示)。B4 因此不需要 toast 去重;手動清單第 12 項檢查關掉之後
  不再出現。
- 沒有新的 command、參數、事件或 `SyncOverview` 欄位:B4 用到的每個 command、參數名稱(camelCase)、事件與型別都在
  B3b 的「B4 handoff」與 `src/bindings/` 裡。
- B3b 的最終修正(`b675943`…`d469bc0`)也沒有改任何 command、binding、事件或提示種類,只改了行為與文字;B4 依它調整
  離開(解讀 14)、更換同步碼的狀態與「Show」(解讀 9、10)、提示與重新加入(解讀 7)、精靈(解讀 16)與焦點(Global
  Constraints)。
- 執行 B4 時另外有兩個後端修正,不是 B3 的 amendments,而是 B4 的審查找到的:`bf37fe6`(Task 4)與 `7b5d1c9`(最終修正
  Part B),都只改 `src-tauri/src/sync/hosts_file.rs`,讓後端拒絕審核對話框會標示出來的那幾類字元;沒有新的 command、binding
  或事件。

## 檔案結構

| 檔案 | 動作 | 責任 |
|---|---|---|
| `src/lib/sync.ts` | 改寫(Task 1);修改(Task 2、6) | query keys、`useSyncOverview`、回傳 `SyncOverview` 的 mutations、離開失敗的標題、relay 未知時查一次、核准 / 不能搬 / 同名主機的查詢、搬進 space、同步碼的直接呼叫、relay 的兩個外部連結 |
| `src/lib/sync-events.ts` | 新增(Task 1);修改(Task 4) | 事件 → 快取與 toast(`subscribeSyncEvents` / `useSyncEvents`)、衝突與提示的文案、v1 升級說明 |
| `src/components/SyncUpgradeDialog.tsx` | 新增(Task 1) | App 層的一次性升級說明 |
| `src/App.tsx` | 修改(Task 1、4) | `useSyncEvents`、升級說明與審核對話框;移除焦點 → `sync_now` |
| `src/lib/sync-fixtures.ts` | 新增(Task 2) | 測試用的 `SyncOverview` / space / device 建構器(只有 `*.test.ts` import) |
| `src/lib/sync-overview.ts` | 新增(Task 2) | 狀態列(含更換同步碼的錯誤與 keychain 還沒收下新碼)、relay 版本與更新提示、更換同步碼的阻擋原因、同步碼列的說明、凍結說明、裝置列、是否最後一台、刪除帳戶與更換期間離開的說明、提示列 |
| `src/components/sync-primitives.tsx` | 新增(Task 2) | 狀態色、同步碼字詞格、同步碼對話框(建立 / 更換後必須確認存好) |
| `src/components/SyncPane.tsx` | 改寫(Task 2);修改(Task 3、4、6) | Settings → Sync:未加入(建立、加入、relay)、狀態、提示、輸入新同步碼、更換進度、帳戶、裝置、離開 |
| `src/lib/sync-spaces.ts` | 新增(Task 3) | space 列、結構性操作的阻擋原因、space 名稱規則 |
| `src/components/SyncSpacesSection.tsx` | 新增(Task 3);修改(Task 6) | Spaces 區塊、名稱 / 取消勾選 / 刪除對話框、加入後的「Choose spaces」 |
| `src/lib/sync-approvals.ts` | 新增(Task 4) | 受管制的關鍵字、逐行標示、簽章差異、依 space 分組、核准 toast 文案 |
| `src/components/SyncApprovalDialog.tsx` | 新增(Task 4) | 審核對話框(逐台或全部核准 / 拒絕) |
| `src/stores/ui.ts` | 修改(Task 4、6) | `syncApprovalsOpen`;精靈的開關改成帶目標 space 的 `syncMigration` |
| `src/lib/sync-sidebar.ts` | 新增(Task 5) | space 檔的標籤、被遮蔽的同名主機 |
| `src/components/HostList.tsx` | 修改(Task 5) | space 群組(名稱、圖示、選單)、同名標示與以檔案定位的處理 |
| `src/lib/sync-migration.ts` | 改寫(Task 6) | 精靈分組、`tagForFile`、新 space 命名、目標與預設選取 |
| `src/components/SyncMigrationDialog.tsx` | 改寫(Task 6) | 搬移精靈 |
| `src/bindings/SyncStatus.ts`、`src/bindings/SyncDevice.ts` | 刪除(Task 6) | v1 型別 |
| `README.md` | 修改(Task 7) | Sync 段落改成 v2 |
| `docs/superpowers/plans/2026-10-02-sync-v2-manual-verification.md` | 新增(Task 7) | 兩台電腦的手動驗證清單(spec §10) |

Task 依序執行:Task 2–6 都用 Task 1 的 `sync.ts`;Task 3 的 Spaces 區塊插進 Task 2 的 pane;Task 6 的精靈用 Task 3 的
`MAX_SPACE_NAME` 與 Task 5 的 `spaceFileLabels`。Task 1 在 `sync.ts` 尾端暫留舊 pane 與舊精靈還在用的 v1 helper —— 它們
呼叫的 v1 commands 在 B3b 之後已不存在,這段期間舊 pane 本來就不能用 —— Task 2 與 Task 6 換掉最後的使用者時刪除。

Task 1 已在 repo 執行(`2def6a2`,審查後的修正 `32cae39`;之後 `src/` 的測試是 181 個,見 Task 1 開頭的說明)。Task 2–7 的
程式碼已在 repo `32cae39` 的拷貝上逐 task 套用:每個 task 先確認新測試失敗,再確認 `vitest run --dir src` 與
`tsc --noEmit` 全綠(測試數 181 → 207 → 214 → 235 → 239 → 249 → 249),Task 7 之後 `tsc && vite build` 成功。Task 2–7 的程式碼區塊
是從驗證過的樹產生的,並以「照本計畫逐步套用到乾淨的拷貝、每個 task 結束時與驗證過的樹逐位元組比對」再驗證過一次。
這是執行前的驗證;實際執行、審查與修正後的結果見每個 task 開頭的說明與文末「最終審查與修正(已執行)」。

---

### Task 1: 資料層與事件:v2 的 `sync.ts`、`sync-events.ts`、v1 升級說明

> **已執行**(repo `2def6a2`;審查後的修正 `32cae39`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準 ——
> 修正輪之後 `src/lib/sync.ts`、`src/lib/sync.test.ts`、`src/lib/sync-events.ts`、`src/lib/sync-events.test.ts` 與
> `src/components/SyncUpgradeDialog.tsx` 都和下面的區塊不同,Task 2 起的 edit 區塊以 repo 的內容為上下文。修正輪改了:
> - `leaveFailureTitle` 改依開頭的「left the sync account」判斷:這台已經離開之後才回的錯誤,除了 `LEAVE_REPLACED_MESSAGE` 與
>   新碼不見時的放行,還有 `leave_account` 自己回的「left the sync account, but the sync code could not be removed from the
>   keychain (…); use "Remove sync code" to retry」與「left the sync account, but the sync state could not be saved (…); it will
>   be retried automatically」。`leaving` 的兩個測試讀 `account.rs`:`leave_account` 裡每一句「left the sync account…」都得到
>   「Left the sync account on this computer」,沒有離開的錯誤(`LEAVE_ROTATING_MESSAGE`、`kept_error`、`remote_deleted_error`)
>   維持「Could not leave the sync account」。
> - `SyncUpgradeDialog` 先在本機關掉(元件 state),再送 `sync_dismiss_notice`、不等回應:被拒絕(第二個 SSHelter 行程從不
>   存檔)時由錯誤 toast 說明,app 照常可用,提示留著、下次啟動再顯示。Esc、關閉鈕與「Got it」都一樣。
> - `subscribeSyncEvents`:停止之後才到的事件不再處理(`if (!disposed)`),`listen()` 或 unlisten 被拒絕時不留下未處理的
>   rejection;測試的 `stubEventBus` 多了 `failListen` / `failUnlisten`,`subscribeSyncEvents` 的 describe 末尾多三個測試。
>   `listens to the engine's events and never asks for a sync round itself` 先送出每一種事件,再確認沒有呼叫任何 `sync_` command。
> - `useResolveShadowed` 保留 `setQueryData`,並把同名主機清單的 key 排除在 `["config"]` 的失效之外(不再馬上重抓剛寫進去的清單)。
>
> 執行後 `pnpm exec vitest run --dir src` 是 16 個檔案、181 個測試(`2def6a2` 時 177);下面 Step 裡的數字是原本計畫的。
> 之後最終修正又改了這些檔案:`listNames` 搬到 `format.ts`(`b7a1b99`);同名主機清單的 key 移到 hosts 的 key 底下,
> `useResolveShadowed` 留下答回的清單(`bae0baf`,解讀 12);`useSelectSpace(quiet)`(`2358dd9`);釘住「取消訂閱會釋放每一個
> listener」的測試(`c17a7d1`);提示與衝突 toast 裡別台取的 space 名稱經 `revealHidden`(`3d65c6e`)。

`src/lib/sync.ts` 改成 v2 的 API:總覽查詢、所有回傳 `SyncOverview` 的 mutation、核准 / 拒絕(送出看過的
`{ alias, digest }`、回傳 `ReviewOutcome`)、核准 / 不能搬 / 同名主機的查詢、搬進 space 的 mutation、同步碼的直接呼叫,
README「Updating your relay」的連結,以及離開失敗時的 toast 標題(`leaveFailureTitle`,解讀 14)與 relay 未知時查一次
(`useCheckUnknownRelay`,解讀 10)。事件接線從 `App.tsx` 搬到
`src/lib/sync-events.ts`:`sync://status` 寫進總覽快取、`sync://applied` 讓 config 檢視與核准清單失效、
`sync://conflict` 與 `sync://notice` 跳 toast(`upgraded` 除外,它由 App 層的 `SyncUpgradeDialog` 一次性說明;離開
帳戶留下的 `left_account` 列出搬到 `~/.ssh/sshelter-local/` 的檔案);`App.tsx` 不再監聽視窗焦點(後端處理,而且照 relay
的退避等)。舊 pane 與舊精靈還在
用的 v1 helper 暫留在 `sync.ts` 尾端。

**Files:**
- Modify(整個改寫):`src/lib/sync.ts`、`src/lib/sync.test.ts`
- Create: `src/lib/sync-events.ts`、`src/lib/sync-events.test.ts`、`src/components/SyncUpgradeDialog.tsx`
- Modify: `src/App.tsx`

**Interfaces:**
- Consumes(B3b,`src/bindings/`):`SyncOverview`、`SyncNotice`、`SyncConflict`、`PendingApprovalView`、`ReviewedVersion`、
  `ReviewOutcome`、`MigrationFailure`、`MigrationReport`、`NewSpaceGroup`、`DuplicateAlias`;commands 見 B3b「B4 handoff」
  (參數一律 camelCase)。
- Produces(`src/lib/sync.ts`):
  - keys:`syncOverviewKey = ["sync", "overview"]`、`syncApprovalsKey = ["sync", "approvals"]`、
    `syncDuplicatesKey = ["config", "syncDuplicates"]`、`syncUnmovableKey = ["config", "syncUnmovable"]`
  - `errorMessage(error: unknown): string`;`RELAY_DEPLOY_URL`、`RELAY_UPDATE_URL`、`openRelayDeploy(): Promise<void>`、
    `openRelayUpdateGuide(): Promise<void>`;`refreshSyncViews(queryClient: QueryClient): void`
  - `useSyncOverview(refetchInterval?: number | false)` → `UseQueryResult<SyncOverview>`
  - 回傳 `SyncOverview` 的 mutations(變數形狀):`useLeaveAccount` `{ deleteRemote: boolean }`(失敗時也重讀總覽)、
    `useSetRelayUrl` `{ url }`、
    `useCheckRelay` `void`、`useSetDeviceName` `{ name }`、`useForgetDevice` `{ deviceId }`、`useCreateSpace` `{ name }`、
    `useRenameSpace` `{ spaceId, name }`、`useDeleteSpace` / `useSelectSpace` / `useUnselectSpace` / `useRebuildSpace`
    `{ spaceId }`、`useDismissNotice` `{ index }`、`useChangeSyncCode` / `useCancelSyncCodeChange` `void`;`useSyncNow()`
  - `leaveFailureTitle(message: string): string`(離開失敗的 toast 標題)、`useCheckUnknownRelay(unknown: boolean): void`
    (總覽的 `relay` 是 null 時呼叫一次 `sync_check_relay`)
  - 核准 / 拒絕:`approveVersions(spaceId, approvals: ReviewedVersion[]): Promise<ReviewOutcome>`、
    `rejectVersions(spaceId, approvals)`;`useApproveHosts` / `useRejectHosts` `{ spaceId, approvals: ReviewedVersion[] }` →
    `ReviewOutcome`(成功時把 `outcome.overview` 寫進總覽快取)
  - queries:`usePendingApprovals(enabled)` → `PendingApprovalView[]`、`useUnmovableHosts(enabled)` → `MigrationFailure[]`、
    `useDuplicateAliases(enabled)` → `DuplicateAlias[]`
  - `useMoveHostsToSpace()` `{ aliases, spaceId, tagByFile }` → `MigrationReport`;`useMoveFilesToNewSpaces()`
    `{ groups: NewSpaceGroup[], tagByFile }` → `MigrationReport`;`useResolveShadowed()` `{ alias, file, action: "rename" | "remove" }`
  - 同步碼(不經 TanStack Query):`createAccount(deviceName): Promise<string>`、`joinAccount(words, deviceName): Promise<SyncOverview>`、
    `rejoinAccount(words): Promise<SyncOverview>`、`showWords(): Promise<string>`
  - 暫留(Task 2、6 刪除):`syncStatusKey`、`useSyncStatus`、`useLeaveChain`、`useMigrateHosts`、`createChain`、`joinChain`
- Produces(`src/lib/sync-events.ts`):`interface SyncMessage { title; description }`、`listNames(names): string`、
  `conflictMessage(conflicts): SyncMessage | null`、`upgradeExplanation(notice): string[]`、
  `upgradeNotice(notices): { index; notice } | null`、`noticeMessage(notice): SyncMessage`、`openSyncSettings(): void`、
  `subscribeSyncEvents(queryClient): () => void`、`useSyncEvents(): void`
- Produces:`SyncUpgradeDialog`(`src/components/SyncUpgradeDialog.tsx`,掛在 App)


- [ ] **Step 1: 寫失敗的測試:`src/lib/sync.test.ts`**

沿用既有的 relay 部署連結測試,加上「Updating your relay」連結、同步碼呼叫的參數(camelCase)及「錯誤不 toast」,
核准 / 拒絕送出的形狀(`{ spaceId, approvals: [{ alias, digest }] }`),以及離開失敗的標題(讀 `account.rs` 的文字對照)。

把 `src/lib/sync.test.ts` 整個換成:

```ts
import { readFileSync } from "node:fs";
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { toast } from "sonner";

import {
  RELAY_DEPLOY_URL,
  RELAY_UPDATE_URL,
  approveVersions,
  createAccount,
  joinAccount,
  leaveFailureTitle,
  openRelayDeploy,
  openRelayUpdateGuide,
  rejectVersions,
  rejoinAccount,
  showWords,
} from "./sync";

/** Stub the backend: every plugin command ends in `window.__TAURI_INTERNALS__.invoke`. */
function stubBackend(reply: (cmd: string, args: unknown) => Promise<unknown>): Array<[string, unknown]> {
  const calls: Array<[string, unknown]> = [];
  vi.stubGlobal("window", {
    __TAURI_INTERNALS__: {
      invoke: (cmd: string, args: unknown) => {
        calls.push([cmd, args]);
        return reply(cmd, args);
      },
    },
  });
  return calls;
}

beforeEach(() => {
  // sonner's `toast.dismiss` schedules through requestAnimationFrame, which Node lacks.
  vi.stubGlobal("requestAnimationFrame", (cb: FrameRequestCallback) => {
    cb(0);
    return 0;
  });
});

afterEach(() => {
  // Dismissing by id drops a toast from `getToasts()` at once; a bare `dismiss()` does not.
  for (const t of toast.getToasts()) toast.dismiss(t.id);
  vi.unstubAllGlobals();
});

describe("the relay deploy link", () => {
  it("starts Cloudflare's deploy flow for this repository's relay folder", () => {
    const target = new URL(RELAY_DEPLOY_URL);
    expect(target.origin).toBe("https://deploy.workers.cloudflare.com");
    expect(target.searchParams.get("url")).toBe("https://github.com/ysya/sshelter/tree/main/relay");
  });

  it("is the same link as the README buttons", () => {
    for (const readme of ["README.md", "relay/README.md"]) {
      expect(readFileSync(readme, "utf8"), readme).toContain(`(${RELAY_DEPLOY_URL})`);
    }
  });

  it("opens it in the default browser", async () => {
    const calls = stubBackend(async () => undefined);
    await openRelayDeploy();
    expect(calls).toEqual([["plugin:opener|open_url", { url: RELAY_DEPLOY_URL, with: undefined }]]);
    expect(toast.getToasts()).toEqual([]);
  });

  it("says so when the browser cannot be opened", async () => {
    stubBackend(async () => {
      throw "opener refused";
    });
    await openRelayDeploy();
    expect(toast.getToasts()).toEqual([
      expect.objectContaining({ title: "Could not open your browser", description: "opener refused" }),
    ]);
  });
});

describe("the relay update guide link", () => {
  it("points at the relay README's \"Updating your relay\" section", () => {
    expect(RELAY_UPDATE_URL).toBe("https://github.com/ysya/sshelter/blob/main/relay/README.md#updating-your-relay");
    // GitHub derives the anchor from the heading: keep the heading where the link expects it.
    expect(readFileSync("relay/README.md", "utf8")).toMatch(/^## Updating your relay$/m);
    expect(readFileSync("README.md", "utf8")).toContain("relay/README.md#updating-your-relay");
  });

  it("opens it in the default browser", async () => {
    const calls = stubBackend(async () => undefined);
    await openRelayUpdateGuide();
    expect(calls).toEqual([["plugin:opener|open_url", { url: RELAY_UPDATE_URL, with: undefined }]]);
  });
});

describe("sync-code commands", () => {
  const WORDS = "abandon ".repeat(23) + "art";
  const OVERVIEW = { joined: true };

  it("send the words and device name as the backend's camelCase arguments", async () => {
    const calls = stubBackend(async (cmd) => (cmd === "sync_create_account" || cmd === "sync_show_words" ? WORDS : OVERVIEW));
    expect(await createAccount("MacBook-A")).toBe(WORDS);
    expect(await joinAccount(WORDS, "MacBook-B")).toEqual(OVERVIEW);
    expect(await rejoinAccount(WORDS)).toEqual(OVERVIEW);
    expect(await showWords()).toBe(WORDS);
    expect(calls).toEqual([
      ["sync_create_account", { deviceName: "MacBook-A" }],
      ["sync_join_account", { words: WORDS, deviceName: "MacBook-B" }],
      ["sync_rejoin_account", { words: WORDS }],
      ["sync_show_words", {}],
    ]);
  });

  it("pass backend errors through untouched and never toast (a toast could carry the words)", async () => {
    stubBackend(async () => {
      throw "no sync account matches this sync code";
    });
    await expect(joinAccount(WORDS, "MacBook-B")).rejects.toBe("no sync account matches this sync code");
    await expect(rejoinAccount(WORDS)).rejects.toBe("no sync account matches this sync code");
    expect(toast.getToasts()).toEqual([]);
  });
});

describe("review commands", () => {
  const SPACE = "a".repeat(64);

  it("send exactly the reviewed versions as { spaceId, approvals: [{ alias, digest }] } and return the outcome", async () => {
    const outcome = { applied: 1, changed: ["db"], overview: { joined: true } };
    const calls = stubBackend(async () => outcome);
    const shown = [
      { alias: "web", digest: "d1" },
      { alias: "db", digest: "d2" },
    ];
    expect(await approveVersions(SPACE, shown)).toEqual(outcome);
    expect(await rejectVersions(SPACE, [{ alias: "web", digest: "d1" }])).toEqual(outcome);
    expect(calls).toEqual([
      ["sync_approve", { spaceId: SPACE, approvals: shown }],
      ["sync_reject", { spaceId: SPACE, approvals: [{ alias: "web", digest: "d1" }] }],
    ]);
  });
});

describe("leaving", () => {
  it("titles an error that came after this computer already left as such (account.rs)", () => {
    const rust = readFileSync("src-tauri/src/sync/account.rs", "utf8");
    const replaced = /const LEAVE_REPLACED_MESSAGE: &str =\s*"([^"]*)"/.exec(rust)?.[1] ?? "";
    const abandoned = /fn leave_abandoned_message[\s\S]*?format!\(\s*"([^"]*)\{\}"/.exec(rust)?.[1] ?? "";
    const refused = /const LEAVE_ROTATING_MESSAGE: &str =\s*"([^"]*)"/.exec(rust)?.[1] ?? "";
    for (const text of [replaced, abandoned, refused]) expect(text).not.toBe("");
    expect(leaveFailureTitle(replaced)).toBe("Left the sync account on this computer");
    expect(leaveFailureTitle(abandoned)).toBe("Left the sync account on this computer");
    expect(leaveFailureTitle(`${abandoned}. The sync account was not deleted from the relay`)).toBe("Left the sync account on this computer");
    expect(leaveFailureTitle(refused)).toBe("Could not leave the sync account");
    expect(leaveFailureTitle("could not keep this device's synced files as local files (disk full); nothing was changed — try leaving again")).toBe(
      "Could not leave the sync account",
    );
  });
});
```

- [ ] **Step 2: 寫失敗的測試:`src/lib/sync-events.test.ts`**

事件以假的 Tauri event bus 測:`listen()` 經 `transformCallback` 與 `plugin:event|listen` 註冊 handler,測試照 webview 的方式呼叫它。

新增 `src/lib/sync-events.test.ts`:

```ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { QueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { SyncNotice } from "@/bindings/SyncNotice";
import { syncApprovalsKey, syncOverviewKey } from "@/lib/sync";
import { useUiStore } from "@/stores/ui";
import { conflictMessage, listNames, noticeMessage, subscribeSyncEvents, upgradeExplanation, upgradeNotice } from "./sync-events";

describe("listNames", () => {
  it("reads like a sentence", () => {
    expect(listNames([])).toBe("");
    expect(listNames(["web"])).toBe("web");
    expect(listNames(["web", "db"])).toBe("web and db");
    expect(listNames(["web", "db", "cache"])).toBe("web, db and cache");
  });
});

describe("conflictMessage", () => {
  it("names the hosts and their space", () => {
    expect(conflictMessage([{ space_id: "a", space_name: "Work", aliases: ["web"] }])).toEqual({
      title: "Sync replaced a local change",
      description: "web in Work was edited on another computer more recently.",
    });
  });

  it("lists every space and counts every host", () => {
    expect(
      conflictMessage([
        { space_id: "a", space_name: "Work", aliases: ["web", "db"] },
        { space_id: "b", space_name: "Personal", aliases: ["nas"] },
      ]),
    ).toEqual({
      title: "Sync replaced local changes",
      description: "web and db in Work; nas in Personal were edited on another computer more recently.",
    });
  });

  it("says nothing when no host is named", () => {
    expect(conflictMessage([])).toBeNull();
    expect(conflictMessage([{ space_id: "a", space_name: "Work", aliases: [] }])).toBeNull();
  });
});

describe("notices", () => {
  const upgraded: SyncNotice = { kind: "upgraded", kept_file: null, kept_hosts: [], moved_files: [] };
  const kept: SyncNotice = { kind: "upgraded", kept_file: "/home/f/.ssh/sshelter-v1-kept.config", kept_hosts: ["jump", "lab"], moved_files: [] };

  it("explain the v1 upgrade once, with the hosts that stayed local and the user's own files that moved", () => {
    expect(upgradeExplanation(upgraded)).toEqual([
      "Your synced hosts moved into a space named “Synced”, unless another computer had already renamed or deleted it. Rename it or add more spaces in Settings → Sync; each computer chooses which spaces it syncs.",
      "Update SSHelter on your other computers too. Until they are updated, they don't see changes made here.",
    ]);
    expect(upgradeExplanation(kept)[2]).toBe(
      "jump and lab could not move into a space, so they stay on this computer in /home/f/.ssh/sshelter-v1-kept.config, where ssh keeps reading them.",
    );
    expect(upgradeExplanation({ ...kept, kept_hosts: ["jump"] })[2]).toBe(
      "jump could not move into a space, so it stays on this computer in /home/f/.ssh/sshelter-v1-kept.config, where ssh keeps reading it.",
    );
    const mine = "/home/f/.ssh/sshelter-local/mine.config";
    expect(upgradeExplanation({ ...upgraded, moved_files: [mine] })[2]).toBe(
      "Your own config file in ~/.ssh/sshelter moved to /home/f/.ssh/sshelter-local/mine.config, where ssh keeps reading it.",
    );
    expect(upgradeExplanation({ ...kept, moved_files: [mine, "/home/f/.ssh/sshelter-local/lab.config"] }).slice(2)).toEqual([
      "jump and lab could not move into a space, so they stay on this computer in /home/f/.ssh/sshelter-v1-kept.config, where ssh keeps reading them.",
      "Your own config files in ~/.ssh/sshelter moved to /home/f/.ssh/sshelter-local/mine.config and /home/f/.ssh/sshelter-local/lab.config, where ssh keeps reading them.",
    ]);
  });

  it("finds the upgrade notice and its index", () => {
    const deleted: SyncNotice = { kind: "space_deleted", name: "Work", by_device: "MacBook-A" };
    expect(upgradeNotice([deleted, kept])).toEqual({ index: 1, notice: kept });
    expect(upgradeNotice([deleted])).toBeNull();
  });

  it("have a title and a description for every kind", () => {
    expect(noticeMessage(upgraded).title).toBe("Sync was upgraded");
    expect(noticeMessage({ kind: "space_deleted", name: "Work", by_device: "MacBook-A" })).toEqual({
      title: "“Work” was deleted on MacBook-A",
      description: "Its file was backed up and removed from this computer.",
    });
    expect(noticeMessage({ kind: "rename_blocked", space_id: "a", name: "Work", file_name: "work-3fa2c1d9.config" })).toEqual({
      title: "The file of “Work” keeps its old name",
      description:
        "work-3fa2c1d9.config already exists in ~/.ssh/sshelter, so SSHelter did not overwrite it. Move that file away; SSHelter renames the space's file on the next sync.",
    });
    expect(noticeMessage({ kind: "left_account", kept_files: ["/home/f/.ssh/sshelter-local/personal-3fa2c1d9.config"] })).toEqual({
      title: "Your synced files are now local files",
      description:
        "ssh keeps reading /home/f/.ssh/sshelter-local/personal-3fa2c1d9.config, but it no longer syncs. To sync these hosts again, use “Move hosts into a space” in a sync account.",
    });
    expect(noticeMessage({ kind: "left_account", kept_files: ["/x/a.config", "/x/b-2.config"] }).description).toBe(
      "ssh keeps reading /x/a.config and /x/b-2.config, but they no longer sync. To sync these hosts again, use “Move hosts into a space” in a sync account.",
    );
    expect(noticeMessage({ kind: "new_sync_code" })).toEqual({
      title: "The sync code was changed",
      description: "Show the new sync code, save it, and enter it on each of your other computers.",
    });
    expect(noticeMessage({ kind: "other_rotation", devices: ["MacBook-B"] })).toEqual({
      title: "MacBook-B also changed the sync code",
      description:
        "Use one of the new sync codes on every computer. To use the other one on this computer, leave the sync account and join with it.",
    });
  });
});

/**
 * A fake Tauri event bus: `listen()` registers its handler through `transformCallback` and the
 * `plugin:event|listen` command, so both are captured here and `emit` calls the handler the way
 * the webview would. Any other command is recorded — the events must never start a sync round.
 */
function stubEventBus() {
  const callbacks = new Map<number, (event: unknown) => void>();
  const handlers = new Map<string, (event: unknown) => void>();
  const commands: string[] = [];
  let nextId = 1;
  vi.stubGlobal("window", {
    __TAURI_INTERNALS__: {
      transformCallback: (callback: (event: unknown) => void) => {
        const id = nextId++;
        callbacks.set(id, callback);
        return id;
      },
      invoke: async (cmd: string, args: { event?: string; handler?: number }) => {
        commands.push(cmd);
        if (cmd === "plugin:event|listen" && args.event && args.handler) {
          handlers.set(args.event, callbacks.get(args.handler)!);
          return nextId++;
        }
        return undefined;
      },
    },
    __TAURI_EVENT_PLUGIN_INTERNALS__: { unregisterListener: () => undefined },
  });
  const emit = (event: string, payload: unknown) => {
    const handler = handlers.get(event);
    if (!handler) throw new Error(`nobody listens to ${event}`);
    handler({ event, id: 0, payload });
  };
  return { emit, handlers, commands };
}

describe("subscribeSyncEvents", () => {
  beforeEach(() => {
    vi.stubGlobal("requestAnimationFrame", (cb: FrameRequestCallback) => {
      cb(0);
      return 0;
    });
  });

  afterEach(() => {
    for (const t of toast.getToasts()) toast.dismiss(t.id);
    vi.unstubAllGlobals();
    useUiStore.setState({ settingsOpen: false, settingsCategory: "general" });
  });

  async function subscribed() {
    const bus = stubEventBus();
    const queryClient = new QueryClient();
    const stop = subscribeSyncEvents(queryClient);
    await vi.waitFor(() => expect(bus.handlers.size).toBe(4));
    return { ...bus, queryClient, stop };
  }

  it("listens to the engine's events and never asks for a sync round itself", async () => {
    const { handlers, commands, stop } = await subscribed();
    expect([...handlers.keys()].sort()).toEqual(["sync://applied", "sync://conflict", "sync://notice", "sync://status"]);
    stop();
    expect(commands.filter((c) => c.startsWith("sync_"))).toEqual([]);
  });

  it("puts each status push into the overview cache", async () => {
    const { emit, queryClient } = await subscribed();
    emit("sync://status", { joined: true, device_name: "MacBook-A" });
    expect(queryClient.getQueryData(syncOverviewKey)).toEqual({ joined: true, device_name: "MacBook-A" });
  });

  it("refreshes the config views and the approval list when the engine wrote files", async () => {
    const { emit, queryClient } = await subscribed();
    queryClient.setQueryData(["config", "hosts"], { files: [], hosts: [] });
    queryClient.setQueryData(syncApprovalsKey, []);
    emit("sync://applied", 2);
    expect(queryClient.getQueryState(["config", "hosts"])?.isInvalidated).toBe(true);
    expect(queryClient.getQueryState(syncApprovalsKey)?.isInvalidated).toBe(true);
  });

  it("toasts conflicts with their space", async () => {
    const { emit } = await subscribed();
    emit("sync://conflict", [{ space_id: "a", space_name: "Work", aliases: ["web"] }]);
    expect(toast.getToasts()).toEqual([
      expect.objectContaining({ title: "Sync replaced a local change", description: "web in Work was edited on another computer more recently." }),
    ]);
  });

  it("toasts notices with a way to Settings → Sync, except the upgrade (it has its own dialog)", async () => {
    const { emit } = await subscribed();
    emit("sync://notice", { kind: "upgraded", kept_file: null, kept_hosts: [], moved_files: [] });
    expect(toast.getToasts()).toEqual([]);
    emit("sync://notice", { kind: "space_deleted", name: "Work", by_device: "MacBook-A" });
    const [shown] = toast.getToasts();
    expect(shown).toEqual(expect.objectContaining({ title: "“Work” was deleted on MacBook-A" }));
    const action = "action" in shown ? shown.action : undefined;
    if (!action || typeof action !== "object" || !("onClick" in action)) throw new Error("the notice has no button");
    action.onClick(undefined as never);
    expect(useUiStore.getState()).toEqual(expect.objectContaining({ settingsOpen: true, settingsCategory: "sync" }));
  });
});
```

- [ ] **Step 3: 跑測試確認失敗**

Run: `pnpm test src/lib/sync.test.ts src/lib/sync-events.test.ts`
Expected: FAIL —— `sync-events.test.ts` 找不到 `./sync-events`;`sync.test.ts` 6 failed | 4 passed
(`createAccount is not a function`、`openRelayUpdateGuide is not a function`、`leaveFailureTitle is not a function` 等)。

- [ ] **Step 4: 改寫 `src/lib/sync.ts`**

把 `src/lib/sync.ts` 整個換成:

```ts
import { useEffect, useRef } from "react";
import { useMutation, useQuery, useQueryClient, type QueryClient } from "@tanstack/react-query";
import { openUrl } from "@tauri-apps/plugin-opener";
import { toast } from "sonner";

import type { DuplicateAlias } from "@/bindings/DuplicateAlias";
import type { MigrationFailure } from "@/bindings/MigrationFailure";
import type { MigrationReport } from "@/bindings/MigrationReport";
import type { NewSpaceGroup } from "@/bindings/NewSpaceGroup";
import type { PendingApprovalView } from "@/bindings/PendingApprovalView";
import type { ReviewOutcome } from "@/bindings/ReviewOutcome";
import type { ReviewedVersion } from "@/bindings/ReviewedVersion";
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncStatus } from "@/bindings/SyncStatus";
import { tauriInvoke } from "@/lib/ipc";

export const syncOverviewKey = ["sync", "overview"] as const;
export const syncApprovalsKey = ["sync", "approvals"] as const;
/**
 * Both live under ["config"] on purpose: any config reload or edit can change
 * which copy of an alias ssh uses and which hosts can move into a space, so every
 * config invalidation refreshes them too.
 */
export const syncDuplicatesKey = ["config", "syncDuplicates"] as const;
export const syncUnmovableKey = ["config", "syncUnmovable"] as const;

export function errorMessage(error: unknown): string {
  return error instanceof Error ? error.message : String(error);
}

/** Cloudflare's one-click deploy of this repository's relay/ folder — the README buttons' link. */
export const RELAY_DEPLOY_URL = "https://deploy.workers.cloudflare.com/?url=https://github.com/ysya/sshelter/tree/main/relay";

/** The relay README's "Updating your relay" section — linked wherever the app says the relay can be updated. */
export const RELAY_UPDATE_URL = "https://github.com/ysya/sshelter/blob/main/relay/README.md#updating-your-relay";

async function openInBrowser(url: string): Promise<void> {
  try {
    await openUrl(url);
  } catch (e) {
    toast.error("Could not open your browser", { description: errorMessage(e) });
  }
}

/** Open the relay deploy flow in the default browser. */
export function openRelayDeploy(): Promise<void> {
  return openInBrowser(RELAY_DEPLOY_URL);
}

/** Open "Updating your relay" in the default browser. */
export function openRelayUpdateGuide(): Promise<void> {
  return openInBrowser(RELAY_UPDATE_URL);
}

/** After anything that changes the sync account or its spaces: refetch the overview, every config view and the approvals. */
export function refreshSyncViews(queryClient: QueryClient): void {
  void queryClient.invalidateQueries({ queryKey: syncOverviewKey });
  void queryClient.invalidateQueries({ queryKey: ["config"] });
  void queryClient.invalidateQueries({ queryKey: syncApprovalsKey });
}

/** Everything Settings → Sync shows. `sync://status` pushes keep it fresh; the pane also polls while open. */
export function useSyncOverview(refetchInterval: number | false = false) {
  return useQuery<SyncOverview>({
    queryKey: syncOverviewKey,
    queryFn: () => tauriInvoke<SyncOverview>("sync_overview"),
    refetchInterval,
  });
}

/**
 * Non-secret commands that answer with the new overview: prime the cache, then
 * refetch the config views (space files come and go) and the approval list.
 * `refetchOnError`: the command can do part of its work and still answer with an
 * error, so a failure re-reads everything too. `failure` is the toast's title, or
 * picks one from the error's text.
 */
function useOverviewMutation<TVars>(
  cmd: string,
  failure: string | ((message: string) => string),
  args: (vars: TVars) => Record<string, unknown>,
  refetchOnError = false,
) {
  const queryClient = useQueryClient();
  return useMutation<SyncOverview, unknown, TVars>({
    mutationFn: (vars) => tauriInvoke<SyncOverview>(cmd, args(vars)),
    onSuccess: (overview) => {
      queryClient.setQueryData(syncOverviewKey, overview);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
      void queryClient.invalidateQueries({ queryKey: syncApprovalsKey });
    },
    onError: (error) => {
      if (refetchOnError) refreshSyncViews(queryClient);
      const message = errorMessage(error);
      toast.error(typeof failure === "string" ? failure : failure(message), { description: message });
    },
  });
}

const noArgs = () => ({});

/**
 * Leave on this computer; `deleteRemote` also deletes the account and every space
 * from the relay (offered on the last computer only). Some errors arrive after
 * this computer already left (the relay part, a sync code change that can never
 * finish, the keychain), so a failure re-reads the overview as well.
 */
export function useLeaveAccount() {
  return useOverviewMutation<{ deleteRemote: boolean }>("sync_leave_account", leaveFailureTitle, ({ deleteRemote }) => ({ deleteRemote }), true);
}

/**
 * The title of a failed leave. Two errors come after this computer already left —
 * the sync code was changed on another computer (so the account stays on the
 * relay), or a sync code change can never finish — and both start with "left the
 * sync account on this computer".
 */
export function leaveFailureTitle(message: string): string {
  return message.startsWith("left the sync account on this computer") ? "Left the sync account on this computer" : "Could not leave the sync account";
}

export function useSetRelayUrl() {
  return useOverviewMutation<{ url: string }>("sync_set_relay_url", "Could not update the relay URL", ({ url }) => ({ url }));
}

/** Ask the relay again what it supports (after the user updated it). */
export function useCheckRelay() {
  return useOverviewMutation<void>("sync_check_relay", "Could not reach the relay", noArgs);
}

/**
 * `relay` null in the overview means the relay was not asked yet — the first sync
 * asks, and so does the first one after the v1 upgrade — never "no freeze". Ask it
 * once when a view shows that; a failure stays quiet (the status line says why
 * the relay can't be reached) and "Check again" stays available.
 */
export function useCheckUnknownRelay(unknown: boolean): void {
  const queryClient = useQueryClient();
  const asked = useRef(false);
  useEffect(() => {
    if (!unknown || asked.current) return;
    asked.current = true;
    void tauriInvoke<SyncOverview>("sync_check_relay").then(
      (overview) => queryClient.setQueryData(syncOverviewKey, overview),
      () => undefined,
    );
  }, [unknown, queryClient]);
}

export function useSetDeviceName() {
  return useOverviewMutation<{ name: string }>("sync_set_device_name", "Could not rename this computer", ({ name }) => ({ name }));
}

/** Removes a device from the list only — it is NOT revocation (changing the sync code is; see the pane copy). */
export function useForgetDevice() {
  return useOverviewMutation<{ deviceId: string }>("sync_forget_device", "Could not forget the computer", ({ deviceId }) => ({ deviceId }));
}

export function useCreateSpace() {
  return useOverviewMutation<{ name: string }>("sync_create_space", "Could not create the space", ({ name }) => ({ name }));
}

export function useRenameSpace() {
  return useOverviewMutation<{ spaceId: string; name: string }>("sync_rename_space", "Could not rename the space", ({ spaceId, name }) => ({
    spaceId,
    name,
  }));
}

/** Deletes the space on every computer and on the relay (confirmed by the caller). */
export function useDeleteSpace() {
  return useOverviewMutation<{ spaceId: string }>("sync_delete_space", "Could not delete the space", ({ spaceId }) => ({ spaceId }));
}

export function useSelectSpace() {
  return useOverviewMutation<{ spaceId: string }>("sync_select_space", "Could not sync the space on this computer", ({ spaceId }) => ({
    spaceId,
  }));
}

/** Removes only this computer's file of the space (confirmed by the caller). */
export function useUnselectSpace() {
  return useOverviewMutation<{ spaceId: string }>("sync_unselect_space", "Could not remove the space from this computer", ({ spaceId }) => ({
    spaceId,
  }));
}

/** The space's data vanished from the relay: upload this computer's copy again. */
export function useRebuildSpace() {
  return useOverviewMutation<{ spaceId: string }>("sync_rebuild_space", "Could not rebuild the space", ({ spaceId }) => ({ spaceId }));
}

/**
 * Approve exactly the versions the review showed: each `{ alias, digest }` comes
 * from `PendingApprovalView`. A host whose waiting version changed meanwhile is left
 * alone and comes back in `ReviewOutcome.changed`.
 */
export function approveVersions(spaceId: string, approvals: ReviewedVersion[]): Promise<ReviewOutcome> {
  return tauriInvoke<ReviewOutcome>("sync_approve", { spaceId, approvals });
}

/** Reject exactly the versions the review showed; same rules as `approveVersions`. */
export function rejectVersions(spaceId: string, approvals: ReviewedVersion[]): Promise<ReviewOutcome> {
  return tauriInvoke<ReviewOutcome>("sync_reject", { spaceId, approvals });
}

/** A review decision: prime the overview from the outcome, then refetch the config views and the approval list. */
function useReviewMutation(review: typeof approveVersions, failure: string) {
  const queryClient = useQueryClient();
  return useMutation<ReviewOutcome, unknown, { spaceId: string; approvals: ReviewedVersion[] }>({
    mutationFn: ({ spaceId, approvals }) => review(spaceId, approvals),
    onSuccess: (outcome) => {
      queryClient.setQueryData(syncOverviewKey, outcome.overview);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
      void queryClient.invalidateQueries({ queryKey: syncApprovalsKey });
    },
    onError: (error) => toast.error(failure, { description: errorMessage(error) }),
  });
}

export function useApproveHosts() {
  return useReviewMutation(approveVersions, "Could not apply the approved hosts");
}

export function useRejectHosts() {
  return useReviewMutation(rejectVersions, "Could not reject the hosts");
}

/** Clears `SyncOverview.notices[index]`. */
export function useDismissNotice() {
  return useOverviewMutation<{ index: number }>("sync_dismiss_notice", "Could not dismiss the notice", ({ index }) => ({ index }));
}

export function useChangeSyncCode() {
  return useOverviewMutation<void>("sync_change_sync_code", "Could not change the sync code", noArgs);
}

/** Only while `SyncOverview.rotation.cancellable`. */
export function useCancelSyncCodeChange() {
  return useOverviewMutation<void>("sync_cancel_sync_code_change", "Could not cancel changing the sync code", noArgs);
}

export function useSyncNow() {
  return useMutation<void, unknown, void>({
    mutationFn: () => tauriInvoke<void>("sync_now"),
    onError: (error) => toast.error("Could not start sync", { description: errorMessage(error) }),
  });
}

/** Hosts held back until the user approves their gated settings (spec §7.4). */
export function usePendingApprovals(enabled: boolean) {
  return useQuery<PendingApprovalView[]>({
    queryKey: syncApprovalsKey,
    queryFn: () => tauriInvoke<PendingApprovalView[]>("sync_pending_approvals"),
    enabled,
  });
}

/** Local hosts that can never move into a space, each with the backend's reason (an `Include`, a value ssh would pass to a shell, …). */
export function useUnmovableHosts(enabled: boolean) {
  return useQuery<MigrationFailure[]>({
    queryKey: syncUnmovableKey,
    queryFn: () => tauriInvoke<MigrationFailure[]>("sync_unmovable_hosts"),
    enabled,
  });
}

/** Move hosts into one space this computer syncs; refusals come back per host in the report. */
export function useMoveHostsToSpace() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; spaceId: string; tagByFile: boolean }>({
    mutationFn: ({ aliases, spaceId, tagByFile }) =>
      tauriInvoke<MigrationReport>("sync_move_hosts_to_space", { aliases, spaceId, tagByFile }),
    onSuccess: () => refreshSyncViews(queryClient),
    onError: (error) => {
      // A refused batch changed nothing, but a failed write reloads the config from disk.
      refreshSyncViews(queryClient);
      toast.error("Could not move hosts", { description: errorMessage(error) });
    },
  });
}

/** "One new space per file": create each space, then move its hosts in. */
export function useMoveFilesToNewSpaces() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { groups: NewSpaceGroup[]; tagByFile: boolean }>({
    mutationFn: ({ groups, tagByFile }) => tauriInvoke<MigrationReport>("sync_move_files_to_new_spaces", { groups, tagByFile }),
    onSuccess: () => refreshSyncViews(queryClient),
    onError: (error) => {
      refreshSyncViews(queryClient);
      toast.error("Could not move hosts", { description: errorMessage(error) });
    },
  });
}

/** Aliases defined in more than one file where ssh reads a space's copy first: the other copies. */
export function useDuplicateAliases(enabled: boolean) {
  return useQuery<DuplicateAlias[]>({
    queryKey: syncDuplicatesKey,
    queryFn: () => tauriInvoke<DuplicateAlias[]>("sync_duplicate_aliases"),
    enabled,
  });
}

/** Rename or remove a shadowed copy of an alias, addressed by file path (never the copy ssh uses). */
export function useResolveShadowed() {
  const queryClient = useQueryClient();
  return useMutation<DuplicateAlias[], unknown, { alias: string; file: string; action: "rename" | "remove" }>({
    mutationFn: ({ alias, file, action }) => tauriInvoke<DuplicateAlias[]>("sync_resolve_shadowed", { alias, file, action }),
    onSuccess: (remaining) => {
      queryClient.setQueryData(syncDuplicatesKey, remaining);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => {
      // After a failed write the backend reloads the config from disk (or drops
      // it), and a write conflict may have brought in outside edits: refetch the
      // host views and the shadow list instead of trusting the cache.
      void queryClient.invalidateQueries({ queryKey: ["config"] });
      toast.error("Could not update the host", { description: errorMessage(error) });
    },
  });
}

/*
 * Sync-code calls deliberately bypass TanStack Query: `useMutation` keeps
 * `variables` and `data` in its cache, so the words would linger in memory long
 * after the dialog closed. Callers hold them in component state only, drop them
 * when the dialog closes, and never put them in a toast or a log.
 */

/** Create a sync account with the default space "Personal"; resolves to the 24-word sync code. */
export function createAccount(deviceName: string): Promise<string> {
  return tauriInvoke<string>("sync_create_account", { deviceName });
}

/** Join with a sync code. Selects no space: the user picks them next. */
export function joinAccount(words: string, deviceName: string): Promise<SyncOverview> {
  return tauriInvoke<SyncOverview>("sync_join_account", { words, deviceName });
}

/** After the sync code changed on another computer: continue with the new code, keeping spaces, file names and unsent edits. */
export function rejoinAccount(words: string): Promise<SyncOverview> {
  return tauriInvoke<SyncOverview>("sync_rejoin_account", { words });
}

export function showWords(): Promise<string> {
  return tauriInvoke<string>("sync_show_words");
}

/*
 * v1 — the old Sync pane and migration wizard still use these until they are
 * rewritten for v2; each goes away with its last caller.
 */

export const syncStatusKey = ["sync", "status"] as const;

export function useSyncStatus(refetchInterval: number | false = false) {
  return useQuery<SyncStatus>({
    queryKey: syncStatusKey,
    queryFn: () => tauriInvoke<SyncStatus>("sync_status"),
    refetchInterval,
  });
}

export function useLeaveChain() {
  return useLeaveAccount();
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
    mutationFn: ({ aliases, tagByFile }) => tauriInvoke<MigrationReport>("sync_migrate_hosts", { aliases, tagByFile }),
    onSuccess: () => refreshSyncViews(queryClient),
    onError: (error) => toast.error("Could not move hosts", { description: errorMessage(error) }),
  });
}

export function createChain(deviceName: string): Promise<string> {
  return createAccount(deviceName);
}

export function joinChain(words: string, deviceName: string): Promise<SyncOverview> {
  return joinAccount(words, deviceName);
}
```

- [ ] **Step 5: 實作 `src/lib/sync-events.ts`**

新增 `src/lib/sync-events.ts`:

```ts
import { useEffect } from "react";
import { listen } from "@tauri-apps/api/event";
import { useQueryClient, type QueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { SyncConflict } from "@/bindings/SyncConflict";
import type { SyncNotice } from "@/bindings/SyncNotice";
import type { SyncOverview } from "@/bindings/SyncOverview";
import { syncApprovalsKey, syncOverviewKey } from "@/lib/sync";
import { useUiStore } from "@/stores/ui";

export interface SyncMessage {
  title: string;
  description: string;
}

/** Names joined for a sentence: "a", "a and b", "a, b and c". */
export function listNames(names: readonly string[]): string {
  if (names.length <= 1) return names[0] ?? "";
  return `${names.slice(0, -1).join(", ")} and ${names[names.length - 1]}`;
}

/** `sync://conflict`: this computer's unsent edits lost to newer versions from another computer. */
export function conflictMessage(conflicts: readonly SyncConflict[]): SyncMessage | null {
  const named = conflicts.filter((c) => c.aliases.length > 0);
  if (named.length === 0) return null;
  const count = named.reduce((n, c) => n + c.aliases.length, 0);
  return {
    title: count === 1 ? "Sync replaced a local change" : "Sync replaced local changes",
    description: `${named.map((c) => `${listNames(c.aliases)} in ${c.space_name}`).join("; ")} ${count === 1 ? "was" : "were"} edited on another computer more recently.`,
  };
}

type UpgradedNotice = Extract<SyncNotice, { kind: "upgraded" }>;

/** The one-time explanation after the v1 → v2 upgrade (spec §8), one sentence per point. */
export function upgradeExplanation(notice: UpgradedNotice): string[] {
  const lines = [
    "Your synced hosts moved into a space named “Synced”, unless another computer had already renamed or deleted it. Rename it or add more spaces in Settings → Sync; each computer chooses which spaces it syncs.",
    "Update SSHelter on your other computers too. Until they are updated, they don't see changes made here.",
  ];
  if (notice.kept_file && notice.kept_hosts.length > 0) {
    // Why a host stayed is the backend's business (a setting synced hosts can't
    // have, or a space deleted elsewhere); the wizard shows the reason per host.
    const one = notice.kept_hosts.length === 1;
    lines.push(
      `${listNames(notice.kept_hosts)} could not move into a space, so ${one ? "it stays" : "they stay"} on this computer in ${notice.kept_file}, where ssh keeps reading ${one ? "it" : "them"}.`,
    );
  }
  if (notice.moved_files.length > 0) {
    // v1 owned only hosts.config: the user's own files that the main config included
    // from ~/.ssh/sshelter now sit in ~/.ssh/sshelter-local (these are the new paths).
    const one = notice.moved_files.length === 1;
    lines.push(
      `Your own config ${one ? "file" : "files"} in ~/.ssh/sshelter moved to ${listNames(notice.moved_files)}, where ssh keeps reading ${one ? "it" : "them"}.`,
    );
  }
  return lines;
}

/** The upgrade notice waiting in the overview, with the index `sync_dismiss_notice` needs. */
export function upgradeNotice(notices: readonly SyncNotice[]): { index: number; notice: UpgradedNotice } | null {
  const index = notices.findIndex((n) => n.kind === "upgraded");
  if (index < 0) return null;
  return { index, notice: notices[index] as UpgradedNotice };
}

/** Title and description of a notice, for its toast and its row in Settings → Sync. */
export function noticeMessage(notice: SyncNotice): SyncMessage {
  switch (notice.kind) {
    case "upgraded":
      return { title: "Sync was upgraded", description: upgradeExplanation(notice).join(" ") };
    case "space_deleted":
      return {
        title: `“${notice.name}” was deleted on ${notice.by_device}`,
        description: "Its file was backed up and removed from this computer.",
      };
    case "rename_blocked":
      return {
        title: `The file of “${notice.name}” keeps its old name`,
        description: `${notice.file_name} already exists in ~/.ssh/sshelter, so SSHelter did not overwrite it. Move that file away; SSHelter renames the space's file on the next sync.`,
      };
    case "left_account":
      // Not only after leaving: also after a sync code change or a rejoin (spaces the new
      // account does not continue), and when creating or joining moves aside a v1 leftover
      // (`hosts.config`) that an Include still reads — so the copy never says "left".
      return {
        title: "Your synced files are now local files",
        description: `ssh keeps reading ${listNames(notice.kept_files)}, but ${notice.kept_files.length === 1 ? "it no longer syncs" : "they no longer sync"}. To sync these hosts again, use “Move hosts into a space” in a sync account.`,
      };
    case "new_sync_code":
      return {
        title: "The sync code was changed",
        description: "Show the new sync code, save it, and enter it on each of your other computers.",
      };
    case "other_rotation":
      return {
        title: `${listNames(notice.devices)} also changed the sync code`,
        description:
          "Use one of the new sync codes on every computer. To use the other one on this computer, leave the sync account and join with it.",
      };
  }
}

/** Settings, on the Sync category (toast buttons). */
export function openSyncSettings(): void {
  const ui = useUiStore.getState();
  ui.setSettingsCategory("sync");
  ui.setSettingsOpen(true);
}

/**
 * Sync engine → UI. Status pushes refresh every overview reader without polling;
 * applied remote changes refresh the config views (a newly synced host can shadow
 * a local one) and the approval list; conflicts and notices surface as toasts.
 * The backend starts a round itself when the window regains focus, so nothing
 * here asks for one — a second round would double the relay usage.
 * Returns the unsubscribe function.
 */
export function subscribeSyncEvents(queryClient: QueryClient): () => void {
  let disposed = false;
  const unlisten: Array<() => void> = [];
  function on<T>(event: string, handler: (payload: T) => void): void {
    void listen<T>(event, (e) => handler(e.payload)).then((fn) => (disposed ? fn() : unlisten.push(fn)));
  }

  on<SyncOverview>("sync://status", (overview) => queryClient.setQueryData(syncOverviewKey, overview));
  on<number>("sync://applied", () => {
    void queryClient.invalidateQueries({ queryKey: ["config"] });
    void queryClient.invalidateQueries({ queryKey: syncApprovalsKey });
  });
  on<SyncConflict[]>("sync://conflict", (conflicts) => {
    const message = conflictMessage(conflicts);
    if (message) toast.warning(message.title, { description: message.description });
    void queryClient.invalidateQueries({ queryKey: ["config"] });
  });
  on<SyncNotice>("sync://notice", (notice) => {
    if (notice.kind === "upgraded") return; // SyncUpgradeDialog explains it
    const message = noticeMessage(notice);
    toast.info(message.title, {
      description: message.description,
      action: { label: "Open", onClick: () => openSyncSettings() },
    });
  });

  return () => {
    disposed = true;
    unlisten.forEach((u) => u());
  };
}

export function useSyncEvents(): void {
  const queryClient = useQueryClient();
  useEffect(() => subscribeSyncEvents(queryClient), [queryClient]);
}
```

- [ ] **Step 6: 新增 `src/components/SyncUpgradeDialog.tsx`**

新增 `src/components/SyncUpgradeDialog.tsx`:

```tsx
import { useDismissNotice, useSyncOverview } from "@/lib/sync";
import { upgradeExplanation, upgradeNotice } from "@/lib/sync-events";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/**
 * The one-time explanation after this computer moved from v1 sync to spaces
 * (spec §8). It stays up — across restarts — until the user closes it, which
 * dismisses the backend's `upgraded` notice.
 */
export function SyncUpgradeDialog() {
  const overview = useSyncOverview();
  const dismiss = useDismissNotice();
  const found = overview.data ? upgradeNotice(overview.data.notices) : null;
  if (!found) return null;

  const close = () => {
    if (!dismiss.isPending) dismiss.mutate({ index: found.index });
  };

  return (
    <Dialog open onOpenChange={(open) => !open && close()}>
      <DialogContent className="sm:max-w-md">
        <DialogHeader>
          <DialogTitle>Sync was upgraded</DialogTitle>
          <DialogDescription>Sync now keeps hosts in spaces: groups of hosts that each computer chooses whether to sync.</DialogDescription>
        </DialogHeader>
        <ul className="list-disc space-y-1.5 pl-5 text-sm">
          {upgradeExplanation(found.notice).map((line) => (
            <li key={line}>{line}</li>
          ))}
        </ul>
        <DialogFooter>
          <Button type="button" disabled={dismiss.isPending} onClick={close}>
            Got it
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 7: 修改 `src/App.tsx`:掛上 `useSyncEvents` 與升級說明,移除舊的事件與焦點 listener**

`src/App.tsx`:把

```tsx
import { useEffect, useRef } from "react";
import { listen } from "@tauri-apps/api/event";
import { useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { Bot, RotateCw, Settings, Terminal, ServerCog } from "lucide-react";

import type { SyncStatus } from "@/bindings/SyncStatus";
import { useHostsQuery, usePlatform, useLoadConfig } from "@/lib/queries";
import { useUiStore } from "@/stores/ui";
import { useApplyTheme } from "@/lib/theme";
import { useSyncBackendSettings } from "@/lib/backend-settings";
import { useGlobalHotkey } from "@/lib/global-hotkey";
import { useAppShortcuts } from "@/lib/app-shortcuts";
import { tauriInvoke } from "@/lib/ipc";
import { syncDuplicatesKey, syncStatusKey } from "@/lib/sync";
import { clampSidebarWidth } from "@/lib/sidebar-width";
import { HostList } from "@/components/HostList";
import { HostEditor } from "@/components/HostEditor";
```

換成:

```tsx
import { useEffect, useRef } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";
import { Bot, RotateCw, Settings, Terminal, ServerCog } from "lucide-react";

import { useHostsQuery, usePlatform, useLoadConfig } from "@/lib/queries";
import { useUiStore } from "@/stores/ui";
import { useApplyTheme } from "@/lib/theme";
import { useSyncBackendSettings } from "@/lib/backend-settings";
import { useGlobalHotkey } from "@/lib/global-hotkey";
import { useAppShortcuts } from "@/lib/app-shortcuts";
import { useSyncEvents } from "@/lib/sync-events";
import { clampSidebarWidth } from "@/lib/sidebar-width";
import { HostList } from "@/components/HostList";
import { HostEditor } from "@/components/HostEditor";
```

`src/App.tsx`:把

```tsx
import { DeployKeyDialog } from "@/components/DeployKeyDialog";
import { NewConfigFileDialog } from "@/components/NewConfigFileDialog";
import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";
import { SettingsDialog } from "@/components/SettingsDialog";
import { CommandPalette } from "@/components/CommandPalette";
import { DriftBanner } from "@/components/DriftBanner";
```

換成:

```tsx
import { DeployKeyDialog } from "@/components/DeployKeyDialog";
import { NewConfigFileDialog } from "@/components/NewConfigFileDialog";
import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";
import { SyncUpgradeDialog } from "@/components/SyncUpgradeDialog";
import { SettingsDialog } from "@/components/SettingsDialog";
import { CommandPalette } from "@/components/CommandPalette";
import { DriftBanner } from "@/components/DriftBanner";
```

`src/App.tsx`:把

```tsx
  useGlobalHotkey();
  // In-app ⌘F (focus host search) and ⌘N (new host).
  useAppShortcuts();

  const { data, isLoading, isError, error } = useHostsQuery();
  const platform = usePlatform();
```

換成:

```tsx
  useGlobalHotkey();
  // In-app ⌘F (focus host search) and ⌘N (new host).
  useAppShortcuts();
  // Sync engine → UI (status, applied changes, conflicts, notices). The backend
  // syncs on window focus by itself and holds that while the relay asks it to back off; a
  // `sync_now` here would skip the backoff, so only the "Sync now" button calls it.
  useSyncEvents();

  const { data, isLoading, isError, error } = useHostsQuery();
  const platform = usePlatform();
```

`src/App.tsx`:把

```tsx
      });
    }
  }, [isError, error]);

  // Sync engine → UI: status pushes refresh the Settings pane without polling;
  // applied remote changes refresh the host list and the shadowed-alias list
  // (a newly synced host can shadow a local one); conflicts surface as a toast;
  // regaining focus nudges a sync round.
  useEffect(() => {
    let disposed = false;
    const unlisten: Array<() => void> = [];
    void listen<SyncStatus>("sync://status", (e) => queryClient.setQueryData(syncStatusKey, e.payload)).then(
      (fn) => (disposed ? fn() : unlisten.push(fn)),
    );
    void listen<number>("sync://applied", () => {
      void queryClient.invalidateQueries({ queryKey: ["config"] });
      void queryClient.invalidateQueries({ queryKey: syncDuplicatesKey });
    }).then((fn) => (disposed ? fn() : unlisten.push(fn)));
    void listen<string[]>("sync://conflict", (e) => {
      const aliases = e.payload.join(", ");
      toast.warning("Sync overwrote a local change", {
        description: `${aliases} was edited on another device more recently.`,
      });
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    }).then((fn) => (disposed ? fn() : unlisten.push(fn)));
    const onFocus = () => void tauriInvoke("sync_now");
    window.addEventListener("focus", onFocus);
    return () => {
      disposed = true;
      unlisten.forEach((u) => u());
      window.removeEventListener("focus", onFocus);
    };
  }, [queryClient]);

  const hosts = data?.hosts ?? [];
  // Wildcard-only blocks (`Host *`) are config defaults, not hosts — keep them
```

換成:

```tsx
      });
    }
  }, [isError, error]);

  const hosts = data?.hosts ?? [];
  // Wildcard-only blocks (`Host *`) are config defaults, not hosts — keep them
```

`src/App.tsx`:把

```tsx
        <DeployKeyDialog />
        <NewConfigFileDialog />
        <SyncMigrationDialog />
        <McpApprovalDialog />
        <Toaster />
      </div>
```

換成:

```tsx
        <DeployKeyDialog />
        <NewConfigFileDialog />
        <SyncMigrationDialog />
        <SyncUpgradeDialog />
        <McpApprovalDialog />
        <Toaster />
      </div>
```

- [ ] **Step 8: 跑測試確認通過**

Run: `pnpm test`
Expected: PASS —— `Test Files  16 passed (16)`、`Tests  177 passed (177)`(task 開始前 15 / 159)。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出(舊的 `SyncPane.tsx`、`SyncMigrationDialog.tsx` 仍以暫留的 v1 helper 編譯)。

- [ ] **Step 9: Commit**

```bash
git add src/lib/sync.ts src/lib/sync.test.ts src/lib/sync-events.ts src/lib/sync-events.test.ts
git add src/components/SyncUpgradeDialog.tsx src/App.tsx
git commit -m "feat(sync): read the v2 sync overview and engine events"
```

---

### Task 2: Settings → Sync:狀態、提示、輸入新同步碼、更換同步碼、帳戶、裝置、離開

> **已執行**(repo `d0ea55d`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準。審查通過(與計畫逐字相同,
> 文案與比對的後端文字都對照過);十個計畫本身帶來的 Minor 為了不動 Task 3、4、6 的 anchor,留到最終修正一起改(`dcf8875`,
> T2-a…T2-j):相對時間每 30 秒更新(`useNow`);更換同步碼的確認不再宣稱 relay 一定支援凍結(「SSHelter checks that before it
> starts」),出現阻擋原因時自動關閉;卡住的升級按「Stop syncing」先確認,未加入時的升級狀態也用 `statusLine`;「I have saved the
> new sync code」清掉的是確認當下那則提示(`newSyncCodeNoticeIndex`);Show 的對話框在更換同步碼期間說明這是舊碼、何時失效
> (`shownCodeNote`);`SYNC_CODE_WORDS` / `wordCount` 與 `leaveRequest` 有測試;「Files SSHelter doesn't use」分單複數
> (`strayFilesNote`)。最終修正另外讓狀態列報出 space 的問題(解讀 9)、帳戶鎖住時停用審核(解讀 11)、未加入時把已不成立的
> 提示改成中性文字(解讀 7)、離開的對話框說明還沒上傳的修改(解讀 14),並以 server render 測試 Sync pane(`SyncPane.test.tsx`,
> `2e3786d`)。執行時 `src/` 的測試 181 → 207(同計畫)。

`SyncPane.tsx` 整個改寫成 v2:未加入時建立帳戶(預設 space「Personal」)或以同步碼加入,升級外殼與 relay 設定;已加入時
狀態列(依解讀 9 的優先順序)、被凍結時「Enter the new sync code」、更換同步碼的進度與取消、提示列、目錄裡沒在用的檔案、
帳戶(這台的名稱、同步碼 Show / Change…、relay 版本與更新提示;relay 未知時查一次)、裝置(Forget 不是撤權)、離開(檔案搬到
`~/.ssh/sshelter-local/`;只有最後一台、沒被凍結、也沒有過了凍結的更換同步碼時才可刪除帳戶;更換同步碼期間照常可用,對話框說明離開
對它的影響,解讀 14)。未加入時也列出提示
(離開後的 `left_account`,解讀 7)。
判斷都在 `src/lib/sync-overview.ts`;同步碼的字格與對話框放在 `src/components/sync-primitives.tsx`(Task 3 起其他檔案
也用)。這個 task 之後 `sync.ts` 只剩舊精靈用的 v1 helper。

**Files:**
- Create: `src/lib/sync-fixtures.ts`、`src/lib/sync-overview.test.ts`、`src/lib/sync-overview.ts`、`src/components/sync-primitives.tsx`
- Modify(整個改寫):`src/components/SyncPane.tsx`
- Modify: `src/lib/sync.ts`

**Interfaces:**
- Consumes(Task 1):`useSyncOverview`、`useDismissNotice`、`useLeaveAccount`、`useSetRelayUrl`、`useCheckRelay`、`useCheckUnknownRelay`、`useSetDeviceName`、
  `useForgetDevice`、`useChangeSyncCode`、`useCancelSyncCodeChange`、`useSyncNow`、`createAccount`、`joinAccount`、`rejoinAccount`、
  `showWords`、`refreshSyncViews`、`syncOverviewKey`、`openRelayDeploy`、`openRelayUpdateGuide`、`errorMessage`;
  `listNames`、`noticeMessage`(`sync-events.ts`);`relativeTime`(`src/lib/format.ts`);`cleanWordsInput`(`src/lib/sync-migration.ts`)。
- Produces(`src/lib/sync-fixtures.ts`,只給測試):`NOW`、`MINUTE`、`space(overrides)`、`device(overrides)`、`overview(overrides)`
- Produces(`src/lib/sync-overview.ts`):`type Tone = "ok" | "busy" | "warning" | "error"`、`plural(n, word)`、
  `platformLabel(platform)`、`rotationLabel(step)`、`interface StatusLine { badge; tone; text }`、`statusLine(o, now)`、
  `interface RelayDetails { version; updateHint }`、`relayDetails(relay)`、`changeCodeBlocker(o): { reason; updateRelay } | null`、
  `frozenMessage(frozen)`、`interface DeviceRow`、`deviceRows(o, now)`、`isLastDevice(o)`、`deleteAccountNote(o): string | null`、
  `leaveRotationNote(o): string | null`、`syncCodeNote(o): string`、`SWAP_PENDING_MESSAGE`、`CHANGE_CANNOT_FINISH_MESSAGE`、
  `interface NoticeRow { index; title; description; showsNewCode }`、`noticeRows(o)`
- Produces(`src/components/sync-primitives.tsx`):`TONE_TEXT: Record<Tone, string>`、`WordGrid({ words })`、
  `SyncCodeDialog({ words, mode: "created" | "changed" | "shown", onDone })`
- Produces(`SyncPane.tsx`):`SyncPane`(Task 3 加入 Spaces 區塊與加入後的選擇對話框)


- [ ] **Step 1: 寫失敗的測試:測試資料建構器與 `src/lib/sync-overview.test.ts`**

`sync-fixtures.ts` 給之後每個 sync 測試用:一個已加入、健康、最新的狀態,測試以 overrides 調整。

新增 `src/lib/sync-fixtures.ts`:

```ts
import type { SyncDeviceView } from "@/bindings/SyncDeviceView";
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncSpaceView } from "@/bindings/SyncSpaceView";

/*
 * Test builders for the sync bindings: a joined, healthy, up-to-date state that
 * each test bends with overrides. Only `*.test.ts` files import this module.
 */

export const NOW = Date.UTC(2026, 9, 2, 12, 0, 0);
export const MINUTE = 60_000;

export function space(overrides: Partial<SyncSpaceView> = {}): SyncSpaceView {
  const id = overrides.id ?? "3fa2c1d9".padEnd(64, "0");
  return {
    id,
    name: "Personal",
    selected: true,
    file_name: `personal-${id.slice(0, 8)}.config`,
    file_path: `/home/f/.ssh/sshelter/personal-${id.slice(0, 8)}.config`,
    hosts: 3,
    pending_uploads: 0,
    approvals: 0,
    first_sync_pending: false,
    missing: false,
    last_error: null,
    created_at_ms: NOW - 86_400_000,
    synced_on: ["MacBook-A"],
    ...overrides,
  };
}

export function device(overrides: Partial<SyncDeviceView> = {}): SyncDeviceView {
  return {
    id: "device-a",
    name: "MacBook-A",
    platform: "macos",
    joined_at_ms: NOW - 86_400_000,
    last_seen_ms: NOW - 5 * MINUTE,
    is_this: true,
    spaces: [],
    ...overrides,
  };
}

export function overview(overrides: Partial<SyncOverview> = {}): SyncOverview {
  return {
    joined: true,
    account_short: "8b01e4aa",
    device_id: "device-a",
    device_name: "MacBook-A",
    relay_url: "https://relay.example.com",
    relay: { url: "https://relay.example.com", version: "0.2.0", batch_pull: true, freeze: true },
    last_sync_ms: NOW - 2 * MINUTE,
    last_error: null,
    read_only: false,
    upgrading: false,
    frozen: null,
    rotation: null,
    devices: [device()],
    spaces: [space()],
    pending_uploads: 0,
    approvals_waiting: 0,
    stray_files: [],
    notices: [],
    phrase_cleanup_pending: false,
    ...overrides,
  };
}
```

新增 `src/lib/sync-overview.test.ts`:

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";

import { noticeMessage } from "./sync-events";
import { MINUTE, NOW, device, overview, space } from "./sync-fixtures";
import {
  CHANGE_CANNOT_FINISH_MESSAGE,
  SWAP_PENDING_MESSAGE,
  changeCodeBlocker,
  deleteAccountNote,
  deviceRows,
  frozenMessage,
  isLastDevice,
  leaveRotationNote,
  noticeRows,
  plural,
  relayDetails,
  rotationLabel,
  statusLine,
  syncCodeNote,
} from "./sync-overview";

describe("plural", () => {
  it("counts", () => {
    expect(plural(1, "host")).toBe("1 host");
    expect(plural(0, "host")).toBe("0 hosts");
    expect(plural(2, "change")).toBe("2 changes");
  });
});

describe("statusLine", () => {
  it("is up to date with the time of the last sync", () => {
    expect(statusLine(overview(), NOW)).toEqual({ badge: "Synced", tone: "ok", text: "Up to date · last sync 2m ago" });
  });

  it("counts changes waiting to upload", () => {
    expect(statusLine(overview({ pending_uploads: 3 }), NOW).text).toBe("3 changes waiting to upload · last sync 2m ago");
  });

  it("waits for the first sync", () => {
    expect(statusLine(overview({ last_sync_ms: null }), NOW)).toEqual({ badge: "Waiting", tone: "busy", text: "Waiting for the first sync" });
  });

  it("shows the engine's error", () => {
    expect(statusLine(overview({ last_error: "the relay had trouble answering" }), NOW)).toEqual({
      badge: "Error",
      tone: "error",
      text: "the relay had trouble answering",
    });
  });

  it("puts the states that stop syncing before an error", () => {
    const broken = { last_error: "boom" };
    expect(statusLine(overview({ ...broken, read_only: true }), NOW).badge).toBe("Read-only");
    expect(statusLine(overview({ ...broken, rotation: { step: "copying", cancellable: false, paused_until_ms: null } }), NOW)).toEqual({
      badge: "Changing code",
      tone: "busy",
      text: "Copying your spaces — boom",
    });
    expect(statusLine(overview({ ...broken, frozen: { detected_at_ms: NOW, by_devices: ["MacBook-B"] } }), NOW)).toEqual({
      badge: "Paused",
      tone: "warning",
      text: "The sync code was changed on another computer",
    });
  });

  it("shows a sync code change's error with its step: the backend retries it, unless the change can never finish", () => {
    const copying = { step: "copying" as const, cancellable: false, paused_until_ms: null };
    const limited = "the relay is limiting requests from this network; sync retries automatically in a few minutes";
    expect(statusLine(overview({ rotation: copying }), NOW)).toEqual({ badge: "Changing code", tone: "busy", text: "Copying your spaces" });
    expect(statusLine(overview({ rotation: copying, last_error: limited }), NOW)).toEqual({
      badge: "Changing code",
      tone: "busy",
      text: `Copying your spaces — ${limited}`,
    });
    expect(statusLine(overview({ rotation: copying, last_error: CHANGE_CANNOT_FINISH_MESSAGE }), NOW)).toEqual({
      badge: "Changing code",
      tone: "error",
      text: `Copying your spaces — ${CHANGE_CANNOT_FINISH_MESSAGE}`,
    });
  });

  it("says the new sync code is still being saved to the keychain without calling it an error", () => {
    expect(statusLine(overview({ last_error: SWAP_PENDING_MESSAGE }), NOW)).toEqual({ badge: "Saving code", tone: "busy", text: SWAP_PENDING_MESSAGE });
  });

  it("shows an unfinished v1 upgrade and why it is stuck", () => {
    expect(statusLine(overview({ joined: false, upgrading: true }), NOW)).toEqual({
      badge: "Upgrading",
      tone: "busy",
      text: "Moving this computer to the new sync format…",
    });
    expect(statusLine(overview({ joined: false, upgrading: true, last_error: "keychain locked" }), NOW).tone).toBe("error");
  });
});

describe("the backend texts the pane tells apart", () => {
  it("are rotation.rs's SWAP_PENDING_MESSAGE and NEXT_CODE_GONE_AFTER_FREEZE_MESSAGE", () => {
    const rust = readFileSync("src-tauri/src/sync/rotation.rs", "utf8");
    const quoted = (name: string) => new RegExp(`${name}: &str =\\s*"([^"]*)"`).exec(rust)?.[1];
    expect(quoted("SWAP_PENDING_MESSAGE")).toBe(SWAP_PENDING_MESSAGE);
    expect(quoted("NEXT_CODE_GONE_AFTER_FREEZE_MESSAGE")).toBe(CHANGE_CANNOT_FINISH_MESSAGE);
  });
});

describe("rotationLabel", () => {
  it("names what is happening at each step", () => {
    expect(rotationLabel("prepared")).toBe("Sending this computer's changes");
    expect(rotationLabel("local_changes_sent")).toBe("Freezing the old sync data");
    expect(rotationLabel("freezing")).toBe("Freezing the old sync data");
    expect(rotationLabel("copying")).toBe("Copying your spaces");
    expect(rotationLabel("deleting")).toBe("Removing the old copies");
    expect(rotationLabel("switching")).toBe("Switching to the new sync code");
  });
});

describe("relayDetails", () => {
  it("shows the version of an up-to-date relay without a hint", () => {
    expect(relayDetails(overview().relay)).toEqual({ version: "Relay 0.2.0", updateHint: null });
  });

  it("says an older relay can be updated, and what it is missing", () => {
    expect(relayDetails({ url: "u", version: null, batch_pull: false, freeze: false })).toEqual({
      version: "Older relay (no version reported)",
      updateHint:
        "This relay can be updated: it checks one space at a time, which uses more of its request limit, and it can't change the sync code.",
    });
    expect(relayDetails({ url: "u", version: "0.1.0", batch_pull: false, freeze: true }).updateHint).toBe(
      "This relay can be updated: it checks one space at a time, which uses more of its request limit.",
    );
  });

  it("has nothing to say before the relay was checked", () => {
    expect(relayDetails(null)).toEqual({ version: "Not checked yet", updateHint: null });
  });
});

describe("changeCodeBlocker", () => {
  it("lets a healthy account change its sync code, also before the relay was checked", () => {
    expect(changeCodeBlocker(overview())).toBeNull();
    expect(changeCodeBlocker(overview({ relay: null }))).toBeNull();
  });

  it("needs a relay that can freeze, with a pointer to the update guide", () => {
    expect(changeCodeBlocker(overview({ relay: { url: "u", version: null, batch_pull: true, freeze: false } }))).toEqual({
      reason: "Your relay can't change the sync code yet — update the relay first.",
      updateRelay: true,
    });
  });

  it("explains every other state that blocks it", () => {
    expect(changeCodeBlocker(overview({ frozen: { detected_at_ms: NOW, by_devices: [] } }))?.reason).toBe(
      "The sync code was already changed on another computer — enter the new one first.",
    );
    expect(changeCodeBlocker(overview({ rotation: { step: "prepared", cancellable: true, paused_until_ms: null } }))?.reason).toBe(
      "The sync code is being changed.",
    );
    expect(changeCodeBlocker(overview({ read_only: true }))?.reason).toBe("Update SSHelter first: this sync account uses a newer format.");
    expect(changeCodeBlocker(overview({ joined: false }))?.reason).toBe("Join or create a sync account first.");
  });
});

describe("syncCodeNote", () => {
  it("says what Show gives during a sync code change, and otherwise why Change… is unavailable", () => {
    expect(syncCodeNote(overview())).toBe("Needed to add another computer. Shown only on request.");
    expect(syncCodeNote(overview({ rotation: { step: "copying", cancellable: false, paused_until_ms: null } }))).toBe(
      "While the sync code is being changed, Show gives the old code: it stops working once the old data is frozen, and the new code is shown when the change finishes.",
    );
    expect(syncCodeNote(overview({ read_only: true }))).toBe("Update SSHelter first: this sync account uses a newer format.");
  });
});

describe("frozenMessage", () => {
  it("names who changed the sync code and promises the unsent changes", () => {
    expect(frozenMessage({ detected_at_ms: NOW, by_devices: ["MacBook-B"] })).toBe(
      "The sync code was changed on MacBook-B. Enter the new sync code to keep syncing; changes this computer has not uploaded yet are kept and sent afterwards.",
    );
  });

  it("covers a rejected upload before the marker is known, and two computers at once", () => {
    expect(frozenMessage({ detected_at_ms: NOW, by_devices: [] })).toBe(
      "The relay no longer accepts this computer's changes: the sync code was probably changed on another computer. Enter the new sync code to keep syncing; changes this computer has not uploaded yet are kept and sent afterwards.",
    );
    expect(frozenMessage({ detected_at_ms: NOW, by_devices: ["MacBook-B", "Mac-mini"] })).toBe(
      "MacBook-B and Mac-mini changed the sync code at the same time. Enter either new sync code to keep syncing; changes this computer has not uploaded yet are kept and sent afterwards.",
    );
  });
});

describe("deviceRows", () => {
  const work = space({ id: "b".repeat(64), name: "Work" });
  const o = overview({
    spaces: [space(), work],
    devices: [
      device({ spaces: [space().id] }),
      device({ id: "device-b", name: "MacBook-B", platform: "linux", is_this: false, last_seen_ms: NOW - 3 * 60 * MINUTE, spaces: [space().id, work.id, "gone".repeat(16)] }),
      device({ id: "device-c", name: "Old PC", platform: "windows", is_this: false, last_seen_ms: NOW - 40 * 86_400_000, spaces: [] }),
    ],
  });

  it("names the platform, the last contact and the spaces each computer syncs", () => {
    expect(deviceRows(o, NOW)).toEqual([
      { id: "device-a", name: "MacBook-A (this computer)", isThis: true, detail: "macOS · Personal" },
      { id: "device-b", name: "MacBook-B", isThis: false, detail: "Linux · last seen 3h ago · Personal and Work" },
      { id: "device-c", name: "Old PC", isThis: false, detail: "Windows · last seen 1mo ago · no spaces" },
    ]);
  });

  it("offers deleting the account only on the last listed computer", () => {
    expect(isLastDevice(o)).toBe(false);
    expect(isLastDevice(overview())).toBe(true);
    expect(isLastDevice(overview({ devices: [] }))).toBe(false);
  });
});

describe("deleteAccountNote", () => {
  it("offers deleting the account only on the last listed computer, never after the sync code changed elsewhere or past a change's freeze", () => {
    expect(deleteAccountNote(overview())).toBeNull();
    expect(deleteAccountNote(overview({ devices: [device(), device({ id: "device-b", name: "MacBook-B", is_this: false })] }))).toBe(
      "Your other computers keep syncing. To delete the sync account from the relay, leave on the last computer.",
    );
    expect(deleteAccountNote(overview({ frozen: { detected_at_ms: NOW, by_devices: ["MacBook-B"] } }))).toBe(
      "The sync code was changed on another computer, so leaving removes only this computer: the old sync account stays on the relay for the computers that still use the old code.",
    );
    expect(deleteAccountNote(overview({ rotation: { step: "copying", cancellable: false, paused_until_ms: null } }))).toBe(
      "The sync code change in progress can no longer be cancelled, so leaving now never deletes the sync account from the relay.",
    );
    expect(deleteAccountNote(overview({ rotation: { step: "prepared", cancellable: true, paused_until_ms: null } }))).toBeNull();
  });
});

describe("leaveRotationNote", () => {
  it("says what leaving does to a sync code change: it cancels one that can still be cancelled; past the freeze it is refused unless the change can never finish", () => {
    expect(leaveRotationNote(overview())).toBeNull();
    expect(leaveRotationNote(overview({ rotation: { step: "local_changes_sent", cancellable: true, paused_until_ms: null } }))).toBe(
      "A sync code change is in progress. Leaving cancels it first: nothing on the relay is frozen yet.",
    );
    expect(leaveRotationNote(overview({ rotation: { step: "copying", cancellable: false, paused_until_ms: null } }))).toBe(
      "A sync code change is in progress and can no longer be cancelled, so SSHelter lets this computer leave only if the change can never finish (its new sync code is gone from the keychain). Otherwise, let it finish first.",
    );
  });
});

describe("noticeRows", () => {
  it("lists the notices with their index, leaving the upgrade to its dialog", () => {
    const o = overview({
      notices: [
        { kind: "upgraded", kept_file: null, kept_hosts: [], moved_files: [] },
        { kind: "new_sync_code" },
        { kind: "space_deleted", name: "Work", by_device: "MacBook-B" },
      ],
    });
    expect(noticeRows(o)).toEqual([
      {
        index: 1,
        title: "The sync code was changed",
        description: "Show the new sync code, save it, and enter it on each of your other computers.",
        showsNewCode: true,
      },
      { index: 2, title: "“Work” was deleted on MacBook-B", description: "Its file was backed up and removed from this computer.", showsNewCode: false },
    ]);
  });

  it("lists them on a computer that left, too, so it can tell where its files went", () => {
    const left = { kind: "left_account" as const, kept_files: ["/home/f/.ssh/sshelter-local/personal-3fa2c1d9.config"] };
    expect(noticeRows(overview({ joined: false, devices: [], spaces: [], notices: [left] }))).toEqual([
      { index: 0, title: "Your synced files are now local files", description: noticeMessage(left).description, showsNewCode: false },
    ]);
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `pnpm exec vitest run --dir src src/lib/sync-overview.test.ts`
Expected: FAIL —— `Cannot find module './sync-overview'`。

- [ ] **Step 3: 實作 `src/lib/sync-overview.ts`**

新增 `src/lib/sync-overview.ts`:

```ts
import type { RotationStep } from "@/bindings/RotationStep";
import type { SyncFrozenView } from "@/bindings/SyncFrozenView";
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncRelayView } from "@/bindings/SyncRelayView";
import { relativeTime } from "@/lib/format";
import { listNames, noticeMessage } from "@/lib/sync-events";

/** How a status reads: ok = all good, busy = working on it, warning = needs the user, error = failed. */
export type Tone = "ok" | "busy" | "warning" | "error";

export function plural(n: number, word: string): string {
  return `${n} ${word}${n === 1 ? "" : "s"}`;
}

const PLATFORMS: Record<string, string> = { macos: "macOS", linux: "Linux", windows: "Windows" };

export function platformLabel(platform: string): string {
  return PLATFORMS[platform] ?? platform;
}

const ROTATION_LABELS: Record<RotationStep, string> = {
  prepared: "Sending this computer's changes",
  local_changes_sent: "Freezing the old sync data",
  freezing: "Freezing the old sync data",
  copying: "Copying your spaces",
  deleting: "Removing the old copies",
  switching: "Switching to the new sync code",
};

/** What changing the sync code is doing now (spec §7.5 steps 2–7). */
export function rotationLabel(step: RotationStep): string {
  return ROTATION_LABELS[step];
}

export interface StatusLine {
  badge: string;
  tone: Tone;
  text: string;
}

/**
 * Backend texts the pane tells apart (`rotation::SWAP_PENDING_MESSAGE` and
 * `rotation::NEXT_CODE_GONE_AFTER_FREEZE_MESSAGE`; a test reads them from the Rust
 * source). Every other error is shown as it comes.
 */
export const SWAP_PENDING_MESSAGE =
  "the keychain did not accept the new sync code yet; SSHelter keeps the new code and retries, and syncing continues meanwhile";
export const CHANGE_CANNOT_FINISH_MESSAGE =
  "the new sync code is missing from the keychain, so this sync code change can never finish and the old sync account can no longer be joined — leave the sync account on this computer, then create a new sync account on one computer; the other computers leave the old account and join the new one";

/** The Sync pane's status row. States that stop syncing come first; then errors; then progress. */
export function statusLine(o: SyncOverview, now: number): StatusLine {
  if (o.upgrading) {
    return { badge: "Upgrading", tone: o.last_error ? "error" : "busy", text: o.last_error ?? "Moving this computer to the new sync format…" };
  }
  if (o.frozen) return { badge: "Paused", tone: "warning", text: "The sync code was changed on another computer" };
  if (o.rotation) {
    // The backend retries every step by itself (the relay's limits, a locked keychain…),
    // so its error is status, not a task — except a change that can never finish: that
    // text tells the user to leave and start a new sync account.
    const step = rotationLabel(o.rotation.step);
    if (!o.last_error) return { badge: "Changing code", tone: "busy", text: step };
    return { badge: "Changing code", tone: o.last_error === CHANGE_CANNOT_FINISH_MESSAGE ? "error" : "busy", text: `${step} — ${o.last_error}` };
  }
  if (o.read_only) {
    return { badge: "Read-only", tone: "warning", text: "This sync account uses a newer format — update SSHelter to keep syncing" };
  }
  // The new sync code is in use; only saving it to the keychain is still being retried.
  if (o.last_error === SWAP_PENDING_MESSAGE) return { badge: "Saving code", tone: "busy", text: o.last_error };
  if (o.last_error) return { badge: "Error", tone: "error", text: o.last_error };
  if (o.last_sync_ms === null) return { badge: "Waiting", tone: "busy", text: "Waiting for the first sync" };
  const last = `last sync ${relativeTime(o.last_sync_ms, now)}`;
  if (o.pending_uploads > 0) return { badge: "Synced", tone: "ok", text: `${plural(o.pending_uploads, "change")} waiting to upload · ${last}` };
  return { badge: "Synced", tone: "ok", text: `Up to date · ${last}` };
}

export interface RelayDetails {
  version: string;
  /** Set when the relay lacks a feature this app can use: show it with the "Updating your relay" link. */
  updateHint: string | null;
}

/** The relay's version and what an update would bring (spec §6.4: no `pull-batch` or no `freeze`). */
export function relayDetails(relay: SyncRelayView | null): RelayDetails {
  if (!relay) return { version: "Not checked yet", updateHint: null };
  const missing: string[] = [];
  if (!relay.batch_pull) missing.push("it checks one space at a time, which uses more of its request limit");
  if (!relay.freeze) missing.push("it can't change the sync code");
  return {
    version: relay.version ? `Relay ${relay.version}` : "Older relay (no version reported)",
    updateHint: missing.length > 0 ? `This relay can be updated: ${missing.join(", and ")}.` : null,
  };
}

/**
 * Why "Change sync code" is unavailable, or null when it can start. Before the
 * relay was checked the backend checks it itself, so an unknown relay does not block.
 */
export function changeCodeBlocker(o: SyncOverview): { reason: string; updateRelay: boolean } | null {
  const block = (reason: string, updateRelay = false) => ({ reason, updateRelay });
  if (!o.joined) return block("Join or create a sync account first.");
  if (o.frozen) return block("The sync code was already changed on another computer — enter the new one first.");
  if (o.rotation) return block("The sync code is being changed.");
  if (o.read_only) return block("Update SSHelter first: this sync account uses a newer format.");
  if (o.relay && !o.relay.freeze) return block("Your relay can't change the sync code yet — update the relay first.", true);
  return null;
}

/**
 * What the "Sync code" row says (the frozen state has its own row). During a change
 * `sync_show_words` still gives the old code: say when it stops working.
 */
export function syncCodeNote(o: SyncOverview): string {
  if (o.rotation) {
    return "While the sync code is being changed, Show gives the old code: it stops working once the old data is frozen, and the new code is shown when the change finishes.";
  }
  return changeCodeBlocker(o)?.reason ?? "Needed to add another computer. Shown only on request.";
}

/** The frozen banner (spec §7.5 "other computers"): who changed the sync code, and that nothing unsent is lost. */
export function frozenMessage(frozen: SyncFrozenView): string {
  const kept = "Enter the new sync code to keep syncing; changes this computer has not uploaded yet are kept and sent afterwards.";
  const by = frozen.by_devices;
  if (by.length === 0) {
    return `The relay no longer accepts this computer's changes: the sync code was probably changed on another computer. ${kept}`;
  }
  if (by.length === 1) return `The sync code was changed on ${by[0]}. ${kept}`;
  return `${listNames(by)} changed the sync code at the same time. ${kept.replace("the new sync code", "either new sync code")}`;
}

export interface DeviceRow {
  id: string;
  name: string;
  isThis: boolean;
  detail: string;
}

/** The Devices list: platform, last contact (other computers only) and the spaces each one syncs. */
export function deviceRows(o: SyncOverview, now: number): DeviceRow[] {
  const names = new Map(o.spaces.map((s) => [s.id, s.name]));
  return o.devices.map((d) => {
    const spaces = d.spaces.flatMap((id) => names.get(id) ?? []);
    const parts = [platformLabel(d.platform)];
    if (!d.is_this) parts.push(`last seen ${relativeTime(d.last_seen_ms, now)}`);
    parts.push(spaces.length > 0 ? listNames(spaces) : "no spaces");
    return { id: d.id, name: d.is_this ? `${d.name} (this computer)` : d.name, isThis: d.is_this, detail: parts.join(" · ") };
  });
}

/** Leaving may also delete the account from the relay only when no other computer is listed (spec §7.3). */
export function isLastDevice(o: SyncOverview): boolean {
  return o.devices.length > 0 && o.devices.every((d) => d.is_this);
}

/**
 * Why leaving does not offer to delete the sync account from the relay, or null
 * when it does: only the last listed computer may delete it (spec §7.3), and never
 * after the sync code changed elsewhere — the old account then tells the computers
 * still on the old code about the change (the backend refuses that delete too) —
 * nor while a sync code change is past the freeze (leaving is refused then, or goes
 * through without touching the relay).
 */
export function deleteAccountNote(o: SyncOverview): string | null {
  if (o.frozen) {
    return "The sync code was changed on another computer, so leaving removes only this computer: the old sync account stays on the relay for the computers that still use the old code.";
  }
  if (o.rotation && !o.rotation.cancellable) {
    return "The sync code change in progress can no longer be cancelled, so leaving now never deletes the sync account from the relay.";
  }
  if (!isLastDevice(o)) return "Your other computers keep syncing. To delete the sync account from the relay, leave on the last computer.";
  return null;
}

/**
 * What leaving does to a sync code change in progress, or null when none runs
 * (spec §7.5). Leave stays available: the overview can't tell whether the staged
 * new code still exists, so the backend decides — before the freeze it cancels the
 * change; after it, it refuses (its text is shown as it comes) unless the change
 * can never finish, and then this computer leaves.
 */
export function leaveRotationNote(o: SyncOverview): string | null {
  if (!o.rotation) return null;
  return o.rotation.cancellable
    ? "A sync code change is in progress. Leaving cancels it first: nothing on the relay is frozen yet."
    : "A sync code change is in progress and can no longer be cancelled, so SSHelter lets this computer leave only if the change can never finish (its new sync code is gone from the keychain). Otherwise, let it finish first.";
}

export interface NoticeRow {
  index: number;
  title: string;
  description: string;
  /** The `new_sync_code` notice: its button shows the new code, and saving it dismisses the notice. */
  showsNewCode: boolean;
}

/** Notices for the pane, with the index `sync_dismiss_notice` needs. The upgrade has its own dialog. */
export function noticeRows(o: SyncOverview): NoticeRow[] {
  return o.notices.flatMap((notice, index) =>
    notice.kind === "upgraded" ? [] : [{ index, ...noticeMessage(notice), showsNewCode: notice.kind === "new_sync_code" }],
  );
}
```

- [ ] **Step 4: 新增 `src/components/sync-primitives.tsx`**

新增 `src/components/sync-primitives.tsx`:

```tsx
import { useState } from "react";
import { Copy } from "lucide-react";
import { toast } from "sonner";

import { errorMessage } from "@/lib/sync";
import type { Tone } from "@/lib/sync-overview";
import { copyText } from "@/lib/clipboard";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/** Text color for a status tone (Settings → Sync rows, space rows). */
export const TONE_TEXT: Record<Tone, string> = {
  ok: "text-muted-foreground",
  busy: "text-muted-foreground",
  warning: "text-amber-700 dark:text-amber-400",
  error: "text-destructive",
};

/** The 24 words, numbered, with a copy button. The words never go into a toast. */
export function WordGrid({ words }: { words: string }) {
  const copy = async () => {
    try {
      await copyText(words);
      toast.success("Sync code copied — clear your clipboard when done");
    } catch (error) {
      toast.error("Clipboard unavailable", { description: errorMessage(error) });
    }
  };
  return (
    <div className="space-y-2">
      <ol className="grid grid-cols-3 gap-x-4 gap-y-1 rounded-md border bg-muted/40 p-3 font-mono text-sm select-text">
        {words.split(" ").map((w, i) => (
          <li key={`${i}-${w}`} className="flex gap-2">
            <span className="w-5 text-right text-muted-foreground tabular-nums">{i + 1}</span>
            {w}
          </li>
        ))}
      </ol>
      <Button type="button" variant="outline" size="sm" className="h-7" onClick={() => void copy()}>
        <Copy className="size-3.5" /> Copy
      </Button>
    </div>
  );
}

const COPY = {
  created: {
    title: "Your sync code",
    description:
      "Enter these 24 words on every other computer you want to sync. Anyone with them can read and change your synced hosts, so keep them in a password manager. Any computer in this sync account can show them again under Settings → Sync.",
    confirm: "I have saved these words somewhere safe",
  },
  changed: {
    title: "Your new sync code",
    description:
      "The old sync code no longer works. Enter these 24 words on each of your other computers (Settings → Sync → Enter the new sync code), and replace the old code in your password manager.",
    confirm: "I have saved the new sync code",
  },
  shown: {
    title: "Sync code",
    description: "Enter these words on another computer under Settings → Sync → Join with a sync code.",
    confirm: null,
  },
} as const;

/**
 * The sync code in a dialog. `created` and `changed` cannot be dismissed until the
 * user confirms they saved the words; `shown` closes freely. Owners keep `words`
 * in component state and set it back to null in `onDone`.
 */
export function SyncCodeDialog({ words, mode, onDone }: { words: string | null; mode: keyof typeof COPY; onDone: () => void }) {
  const [saved, setSaved] = useState(false);
  const copy = COPY[mode];
  const mustConfirm = copy.confirm !== null;
  const finish = () => {
    setSaved(false);
    onDone();
  };
  return (
    <Dialog
      open={words !== null}
      onOpenChange={(open) => {
        if (!open && (!mustConfirm || saved)) finish();
      }}
    >
      <DialogContent
        className="sm:max-w-lg"
        showCloseButton={!mustConfirm}
        onEscapeKeyDown={(e) => {
          if (mustConfirm && !saved) e.preventDefault();
        }}
        onPointerDownOutside={(e) => {
          if (mustConfirm && !saved) e.preventDefault();
        }}
      >
        <DialogHeader>
          <DialogTitle>{copy.title}</DialogTitle>
          <DialogDescription>{copy.description}</DialogDescription>
        </DialogHeader>
        <WordGrid words={words ?? ""} />
        {copy.confirm !== null && (
          <>
            <label className="flex items-center gap-2 text-sm">
              <Checkbox checked={saved} onCheckedChange={(v) => setSaved(v === true)} />
              {copy.confirm}
            </label>
            <DialogFooter>
              <Button type="button" disabled={!saved} onClick={finish}>
                Continue
              </Button>
            </DialogFooter>
          </>
        )}
      </DialogContent>
    </Dialog>
  );
}
```

- [ ] **Step 5: 改寫 `src/components/SyncPane.tsx`**

同步碼對話框屬於 `SyncPane`(在已加入 / 未加入的分支之上):建立帳戶會立刻讓 `joined` 變 true,放在 `NotJoinedPane` 裡的對話框會在使用者確認之前被卸載。

把 `src/components/SyncPane.tsx` 整個換成:

```tsx
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { ExternalLink, Eye, KeyRound, Loader2, RefreshCw, UserMinus } from "lucide-react";
import { toast } from "sonner";

import type { SyncFrozenView } from "@/bindings/SyncFrozenView";
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncRotationView } from "@/bindings/SyncRotationView";
import { cleanWordsInput } from "@/lib/sync-migration";
import {
  createAccount,
  errorMessage,
  joinAccount,
  openRelayDeploy,
  openRelayUpdateGuide,
  refreshSyncViews,
  rejoinAccount,
  showWords,
  syncOverviewKey,
  useCancelSyncCodeChange,
  useChangeSyncCode,
  useCheckRelay,
  useCheckUnknownRelay,
  useDismissNotice,
  useForgetDevice,
  useLeaveAccount,
  useSetDeviceName,
  useSetRelayUrl,
  useSyncNow,
  useSyncOverview,
} from "@/lib/sync";
import {
  changeCodeBlocker,
  deleteAccountNote,
  deviceRows,
  frozenMessage,
  leaveRotationNote,
  noticeRows,
  relayDetails,
  rotationLabel,
  statusLine,
  syncCodeNote,
} from "@/lib/sync-overview";
import { useUiStore } from "@/stores/ui";
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { SyncCodeDialog, TONE_TEXT } from "@/components/sync-primitives";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

/** Words in a pasted sync code after tidying (the Join / Enter buttons need 24). */
function wordCount(raw: string): number {
  const cleaned = cleanWordsInput(raw);
  return cleaned === "" ? 0 : cleaned.split(" ").length;
}

/**
 * Settings → Sync. The sync-code dialogs live HERE, above the joined / not-joined
 * split: creating an account flips `joined` immediately, and a dialog owned by
 * NotJoinedPane would unmount before the user confirmed the words. The words only
 * ever live in component state — never in a query cache, a store, a log or a toast.
 */
export function SyncPane() {
  const overview = useSyncOverview(5_000);
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);
  const dismiss = useDismissNotice();
  const [createdWords, setCreatedWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // shown on request
  const [newCode, setNewCode] = useState<{ words: string; noticeIndex: number } | null>(null); // after changing it

  if (!overview.data) {
    // A failed overview query never turns into data: show why instead of loading forever.
    return overview.isError ? (
      <p className="px-3 py-3 text-sm text-destructive">Could not load sync status: {errorMessage(overview.error)}</p>
    ) : (
      <p className="px-3 py-3 text-sm text-muted-foreground">Loading sync status…</p>
    );
  }

  return (
    <>
      {/* While upgrading, `joined` is false: NotJoinedPane checks `upgrading` before it offers create or join. */}
      {overview.data.joined ? (
        <JoinedPane
          overview={overview.data}
          onShowWords={setShownWords}
          onShowNewCode={(words, noticeIndex) => setNewCode({ words, noticeIndex })}
        />
      ) : (
        <NotJoinedPane overview={overview.data} onCreated={setCreatedWords} />
      )}

      <SyncCodeDialog
        mode="created"
        words={createdWords}
        onDone={() => {
          setCreatedWords(null);
          setMigrationOpen(true);
        }}
      />
      <SyncCodeDialog mode="shown" words={shownWords} onDone={() => setShownWords(null)} />
      <SyncCodeDialog
        mode="changed"
        words={newCode?.words ?? null}
        onDone={() => {
          // Saved: the "new sync code" notice has done its job.
          if (newCode) dismiss.mutate({ index: newCode.noticeIndex });
          setNewCode(null);
        }}
      />
    </>
  );
}

/** The relay must be reachable BEFORE create/join; joined computers cannot switch relays. */
function RelayUrlRow({ current }: { current: string }) {
  const setRelayUrl = useSetRelayUrl();
  const [draft, setDraft] = useState(current);
  return (
    <SettingsRow id="sync-relay" label="Relay URL" description="https:// only (plain http is allowed for localhost). Self-host from the repository's relay/ folder.">
      <div className="flex items-center gap-1.5">
        <Input id="sync-relay" value={draft} onChange={(e) => setDraft(e.target.value)} placeholder="https://relay.example.com" className="h-7 w-64 font-mono text-xs" />
        <Button type="button" variant="secondary" size="sm" className="h-7" disabled={draft.trim() === current || setRelayUrl.isPending} onClick={() => setRelayUrl.mutate({ url: draft })}>
          Save
        </Button>
      </div>
    </SettingsRow>
  );
}

/** Create or join. Errors keep the form as it was so the user can fix a typo. */
function NotJoinedPane({ overview: o, onCreated }: { overview: SyncOverview; onCreated: (words: string) => void }) {
  const queryClient = useQueryClient();
  const [deviceName, setDeviceName] = useState(o.device_name);
  const [words, setWords] = useState("");
  const [busy, setBusy] = useState<"create" | "join" | null>(null);
  const leave = useLeaveAccount();
  const dismiss = useDismissNotice();
  // After leaving, `left_account` says where this computer's files went.
  const notices = noticeRows(o);

  const onCreate = async () => {
    setBusy("create");
    try {
      onCreated(await createAccount(deviceName));
      refreshSyncViews(queryClient);
    } catch (error) {
      toast.error("Could not create the sync account", { description: errorMessage(error) });
    } finally {
      setBusy(null);
    }
  };

  const onJoin = async () => {
    setBusy("join");
    try {
      const joined = await joinAccount(cleanWordsInput(words), deviceName);
      setWords("");
      queryClient.setQueryData(syncOverviewKey, joined);
      refreshSyncViews(queryClient);
      toast.success("Joined the sync account");
    } catch (error) {
      // Keep the pasted words so a typo can be fixed.
      toast.error("Could not join the sync account", { description: errorMessage(error) });
    } finally {
      setBusy(null);
    }
  };

  // Release builds made without a built-in relay start with an empty relay URL:
  // the user must enter one before creating or joining (the backend refuses too).
  const relayMissing = o.relay_url.trim() === "";
  const blocked = busy !== null || deviceName.trim() === "" || relayMissing;

  return (
    <>
      {/* Errors while not joined — a state file set aside at startup, an I/O error that
          needs a restart, sync running in another SSHelter process — show here. */}
      {o.last_error && !o.upgrading && (
        <Section title="Sync error">
          <SettingsGroup>
            <SettingsRow label="Status" description={o.last_error}>
              <Badge variant="destructive">Error</Badge>
            </SettingsRow>
          </SettingsGroup>
        </Section>
      )}

      {o.phrase_cleanup_pending && (
        <Section title="Cleanup needed" description="You left the sync account, but the sync code is still in the keychain.">
          <SettingsGroup>
            <SettingsRow label="Sync code" description="Retry removing it from the OS keychain.">
              <Button type="button" variant="outline" size="sm" className="h-7" disabled={leave.isPending} onClick={() => leave.mutate({ deleteRemote: false })}>
                Remove sync code
              </Button>
            </SettingsRow>
          </SettingsGroup>
        </Section>
      )}

      {notices.length > 0 && (
        <Section title="Notices">
          <SettingsGroup>
            {notices.map((n) => (
              <SettingsRow key={`${n.index}-${n.title}`} label={n.title} description={n.description}>
                <Button type="button" variant="ghost" size="sm" className="h-7 text-muted-foreground" disabled={dismiss.isPending} onClick={() => dismiss.mutate({ index: n.index })}>
                  Dismiss
                </Button>
              </SettingsRow>
            ))}
          </SettingsGroup>
        </Section>
      )}

      {o.upgrading ? (
        <Section title="Upgrading sync" description="This computer synced with an earlier SSHelter. Its hosts move into a space named “Synced”; nothing needs to be entered again.">
          <SettingsGroup>
            <SettingsRow label="Status" description={o.last_error ?? "Moving this computer to the new sync format…"}>
              {o.last_error ? <Badge variant="destructive">Error</Badge> : <Loader2 className="size-4 animate-spin text-muted-foreground" />}
            </SettingsRow>
            {o.last_error && (
              <SettingsRow label="Stop syncing" description="Gives up the upgrade on this computer. The files in ~/.ssh/sshelter that your SSH config includes — its synced hosts among them — move to ~/.ssh/sshelter-local/, where ssh keeps reading them.">
                <Button type="button" variant="outline" size="sm" className="h-7" disabled={leave.isPending} onClick={() => leave.mutate({ deleteRemote: false })}>
                  Stop syncing
                </Button>
              </SettingsRow>
            )}
          </SettingsGroup>
        </Section>
      ) : (
        <>
          {relayMissing && (
            <Section
              title="Relay"
              description="This build has no built-in relay. Deploy your own, enter its URL, then create or join a sync account. Use the same relay URL on every computer."
            >
              <SettingsGroup>
                <SettingsRow
                  label="Deploy a relay"
                  description="Opens Cloudflare in your browser: it copies the relay into a new repository on your GitHub or GitLab account and deploys it (the free plan is enough). Paste the workers.dev URL it gives you below."
                >
                  <Button type="button" variant="outline" size="sm" className="h-7" onClick={() => void openRelayDeploy()}>
                    <ExternalLink className="size-3.5" /> Deploy to Cloudflare
                  </Button>
                </SettingsRow>
                <RelayUrlRow current={o.relay_url} />
              </SettingsGroup>
            </Section>
          )}

          <Section
            title="Sync account"
            description="Sync is in beta. Keep hosts in sync across your computers without signing up anywhere: a 24-word sync code is the only secret, and the relay only ever stores encrypted data."
          >
            <SettingsGroup>
              <SettingsRow id="sync-device-name" label="This computer" description="Shown to your other computers.">
                <Input id="sync-device-name" value={deviceName} onChange={(e) => setDeviceName(e.target.value)} className="h-7 w-48 text-sm" />
              </SettingsRow>
              <SettingsRow
                label="Create a sync account"
                description={relayMissing ? "Enter a relay URL above first." : "Starts with one space, “Personal”, and shows the sync code to enter on your other computers."}
              >
                <Button type="button" size="sm" className="h-7" disabled={blocked} onClick={() => void onCreate()}>
                  {busy === "create" && <Loader2 className="size-3.5 animate-spin" />} Create
                </Button>
              </SettingsRow>
            </SettingsGroup>
          </Section>

          <Section title="Join with a sync code" description="Paste the 24 words from a computer that already syncs. Nothing is sent until you press Join.">
            <div className="space-y-2">
              <Textarea
                value={words}
                onChange={(e) => setWords(e.target.value)}
                placeholder="abandon ability able …"
                rows={3}
                className="font-mono text-sm"
                aria-label="Sync code"
                autoCorrect="off"
                autoCapitalize="off"
                spellCheck={false}
              />
              <Button type="button" size="sm" className="h-7" disabled={blocked || wordCount(words) !== 24} onClick={() => void onJoin()}>
                {busy === "join" && <Loader2 className="size-3.5 animate-spin" />} Join
              </Button>
            </div>
          </Section>

          {!relayMissing && (
            <Section title="Advanced" description="Change this before creating or joining if you self-host the relay or the default one is unreachable.">
              <SettingsGroup>
                <RelayUrlRow current={o.relay_url} />
              </SettingsGroup>
            </Section>
          )}
        </>
      )}
    </>
  );
}

/** Frozen (spec §7.5): the sync code changed elsewhere. The new words stay in this component's state only. */
function RejoinRow({ frozen }: { frozen: SyncFrozenView }) {
  const queryClient = useQueryClient();
  const [words, setWords] = useState("");
  const [busy, setBusy] = useState(false);

  const onRejoin = async () => {
    setBusy(true);
    try {
      queryClient.setQueryData(syncOverviewKey, await rejoinAccount(cleanWordsInput(words)));
      setWords("");
      refreshSyncViews(queryClient);
      // Spaces the new account does not continue became local files: a `left_account` notice lists them.
      toast.success("Syncing again with the new sync code");
    } catch (error) {
      // Every refusal changes nothing (the old code, another account's code, files that can't
      // be kept local…): show it as it comes and keep the words for another try.
      toast.error("Could not use the new sync code", { description: errorMessage(error) });
    } finally {
      setBusy(false);
    }
  };

  return (
    <div className="space-y-2 px-3 py-2">
      <p className="text-sm font-medium">Enter the new sync code</p>
      <p className="text-xs text-muted-foreground">{frozenMessage(frozen)}</p>
      <Textarea
        value={words}
        onChange={(e) => setWords(e.target.value)}
        placeholder="abandon ability able …"
        rows={3}
        className="font-mono text-sm"
        aria-label="New sync code"
        autoCorrect="off"
        autoCapitalize="off"
        spellCheck={false}
      />
      <Button type="button" size="sm" className="h-7" disabled={busy || wordCount(words) !== 24} onClick={() => void onRejoin()}>
        {busy && <Loader2 className="size-3.5 animate-spin" />} Use the new sync code
      </Button>
    </div>
  );
}

/** Changing the sync code is running: where it is, a relay pause, and Cancel until the old data is frozen. Its errors show in the status row. */
function RotationRow({ rotation }: { rotation: SyncRotationView }) {
  const cancel = useCancelSyncCodeChange();
  const paused = rotation.paused_until_ms
    ? ` Paused by the relay's hourly limit on new spaces; continues at ${new Date(rotation.paused_until_ms).toLocaleTimeString([], { hour: "2-digit", minute: "2-digit" })}.`
    : "";
  return (
    <SettingsRow
      label="Changing the sync code"
      description={`${rotationLabel(rotation.step)}…${paused} ${rotation.cancellable ? "You can still cancel." : "It can no longer be cancelled: your other computers are already stopped."}`}
    >
      {rotation.cancellable ? (
        <Button type="button" variant="outline" size="sm" className="h-7" disabled={cancel.isPending} onClick={() => cancel.mutate()}>
          Cancel
        </Button>
      ) : (
        <Loader2 className="size-4 animate-spin text-muted-foreground" />
      )}
    </SettingsRow>
  );
}

function JoinedPane({
  overview: o,
  onShowWords,
  onShowNewCode,
}: {
  overview: SyncOverview;
  onShowWords: (words: string) => void;
  onShowNewCode: (words: string, noticeIndex: number) => void;
}) {
  const syncNow = useSyncNow();
  const dismiss = useDismissNotice();
  const status = statusLine(o, Date.now());
  const [reading, setReading] = useState<number | "code" | null>(null);

  /** Reads the sync code from the keychain into the caller's dialog state. */
  const readWords = async (key: number | "code", show: (words: string) => void) => {
    setReading(key);
    try {
      show(await showWords());
    } catch (error) {
      toast.error("Could not read the sync code", { description: errorMessage(error) });
    } finally {
      setReading(null);
    }
  };

  return (
    <>
      <Section title="Sync" description={`Sync is in beta · account ${o.account_short ?? ""}`}>
        <SettingsGroup>
          <SettingsRow label="Status" description={status.text}>
            <div className="flex items-center gap-1.5">
              <Badge variant={status.tone === "error" ? "destructive" : status.tone === "ok" ? "secondary" : "outline"} className={cn(status.tone === "warning" && TONE_TEXT.warning)}>
                {status.badge}
              </Badge>
              <Button type="button" variant="ghost" size="icon" className="size-7" aria-label="Sync now" disabled={syncNow.isPending} onClick={() => syncNow.mutate()}>
                <RefreshCw className="size-3.5" />
              </Button>
            </div>
          </SettingsRow>
          {o.frozen && <RejoinRow frozen={o.frozen} />}
          {o.rotation && <RotationRow rotation={o.rotation} />}
          {noticeRows(o).map((n) => (
            <SettingsRow key={`${n.index}-${n.title}`} label={n.title} description={n.description}>
              <div className="flex items-center gap-1.5">
                {n.showsNewCode && (
                  <Button type="button" size="sm" className="h-7" disabled={reading !== null} onClick={() => void readWords(n.index, (words) => onShowNewCode(words, n.index))}>
                    <KeyRound className="size-3.5" /> Show new sync code
                  </Button>
                )}
                {!n.showsNewCode && (
                  <Button type="button" variant="ghost" size="sm" className="h-7 text-muted-foreground" disabled={dismiss.isPending} onClick={() => dismiss.mutate({ index: n.index })}>
                    Dismiss
                  </Button>
                )}
              </div>
            </SettingsRow>
          ))}
          {o.stray_files.length > 0 && (
            <SettingsRow
              label="Files SSHelter doesn't use"
              description={`ssh doesn't read ${o.stray_files.join(", ")} in ~/.ssh/sshelter: they are not in SSHelter's Include line. SSHelter leaves them alone — copy any host you still need into your SSH config before deleting them.`}
            >
              <span />
            </SettingsRow>
          )}
        </SettingsGroup>
      </Section>

      <AccountSection overview={o} reading={reading === "code"} onShowCode={() => void readWords("code", onShowWords)} />
      <DevicesSection overview={o} />
      <LeaveSection overview={o} />
    </>
  );
}

function AccountSection({ overview: o, reading, onShowCode }: { overview: SyncOverview; reading: boolean; onShowCode: () => void }) {
  const setDeviceName = useSetDeviceName();
  const checkRelay = useCheckRelay();
  const changeCode = useChangeSyncCode();
  const [nameDraft, setNameDraft] = useState(o.device_name);
  const [confirmChange, setConfirmChange] = useState(false);
  const relay = relayDetails(o.relay);
  const blocker = changeCodeBlocker(o);
  // Null = the relay was not asked yet, never "no freeze": ask it once.
  useCheckUnknownRelay(o.relay === null);

  return (
    <Section title="Account">
      <SettingsGroup>
        <SettingsRow id="sync-name" label="This computer">
          <div className="flex items-center gap-1.5">
            <Input id="sync-name" value={nameDraft} onChange={(e) => setNameDraft(e.target.value)} className="h-7 w-40 text-sm" />
            <Button
              type="button"
              variant="secondary"
              size="sm"
              className="h-7"
              disabled={nameDraft.trim() === "" || nameDraft === o.device_name || setDeviceName.isPending}
              onClick={() => setDeviceName.mutate({ name: nameDraft })}
            >
              Rename
            </Button>
          </div>
        </SettingsRow>
        {o.frozen ? (
          <SettingsRow label="Sync code" description="The sync code was changed on another computer: enter the new one above.">
            <span />
          </SettingsRow>
        ) : (
          <SettingsRow label="Sync code" description={syncCodeNote(o)}>
            <div className="flex items-center gap-1.5">
              {blocker?.updateRelay && (
                <Button type="button" variant="ghost" size="sm" className="h-7" onClick={() => void openRelayUpdateGuide()}>
                  <ExternalLink className="size-3.5" /> How to update
                </Button>
              )}
              {/* During a change this is still the old code; the row says when it stops working. */}
              <Button type="button" variant="outline" size="sm" className="h-7" disabled={reading} onClick={onShowCode}>
                <Eye className="size-3.5" /> Show
              </Button>
              <Button type="button" variant="outline" size="sm" className="h-7" disabled={blocker !== null || changeCode.isPending} onClick={() => setConfirmChange(true)}>
                Change…
              </Button>
            </div>
          </SettingsRow>
        )}
        {/* Read-only while joined: the cursors and every record's seq belong to this relay (spec §7.3). */}
        <SettingsRow label="Relay" description={`${o.relay_url} · ${relay.version}${relay.updateHint ? ` · ${relay.updateHint}` : ""}`}>
          <div className="flex items-center gap-1.5">
            {relay.updateHint && (
              <Button type="button" variant="ghost" size="sm" className="h-7" onClick={() => void openRelayUpdateGuide()}>
                <ExternalLink className="size-3.5" /> How to update
              </Button>
            )}
            <Button type="button" variant="outline" size="sm" className="h-7" disabled={checkRelay.isPending} onClick={() => checkRelay.mutate()}>
              {checkRelay.isPending && <Loader2 className="size-3.5 animate-spin" />} Check again
            </Button>
          </div>
        </SettingsRow>
      </SettingsGroup>

      <AlertDialog
        open={confirmChange}
        onOpenChange={(open) => {
          if (!changeCode.isPending) setConfirmChange(open);
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Change the sync code?</AlertDialogTitle>
            <AlertDialogDescription>
              Use this when a computer with the sync code was lost or stolen. SSHelter creates a new sync code, freezes the old data on the relay so
              nothing more can be written to it, and copies every space to new locations. A computer that only has the old code keeps what it had but
              can't read or write anything new.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <ul className="list-disc space-y-1 pl-5 text-sm text-muted-foreground">
            <li>Every other computer stops syncing until you enter the new sync code on it. Changes it hasn't uploaded yet are kept and sent afterwards.</li>
            <li>Your relay must support freezing data — this one does, or SSHelter would not offer it.</li>
            <li>You can cancel only until the old data is frozen.</li>
          </ul>
          <AlertDialogFooter>
            <AlertDialogCancel disabled={changeCode.isPending}>Cancel</AlertDialogCancel>
            <AlertDialogAction
              disabled={changeCode.isPending}
              onClick={(e) => {
                // Keep the dialog open until the request settles: Radix's Close
                // (which Action composes) skips its auto-close when the click
                // handler calls preventDefault first.
                e.preventDefault();
                changeCode.mutate(undefined, { onSuccess: () => setConfirmChange(false) });
              }}
            >
              {changeCode.isPending && <Loader2 className="size-3.5 animate-spin" />} Change sync code
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </Section>
  );
}

function DevicesSection({ overview: o }: { overview: SyncOverview }) {
  const forget = useForgetDevice();
  return (
    <Section
      title="Devices"
      description="Every computer in this sync account. Forget only removes a computer from this list — it does not lock it out: a computer that still has the sync code keeps syncing. To lock out a lost computer, change the sync code."
    >
      <SettingsGroup>
        {deviceRows(o, Date.now()).map((d) => (
          <SettingsRow key={d.id} label={d.name} description={d.detail}>
            {!d.isThis && (
              <Button type="button" variant="ghost" size="sm" className="h-7 text-muted-foreground" aria-label={`Forget ${d.name}`} disabled={forget.isPending} onClick={() => forget.mutate({ deviceId: d.id })}>
                <UserMinus className="size-3.5" /> Forget
              </Button>
            )}
          </SettingsRow>
        ))}
      </SettingsGroup>
    </Section>
  );
}

function LeaveSection({ overview: o }: { overview: SyncOverview }) {
  const leave = useLeaveAccount();
  const [open, setOpen] = useState(false);
  const [deleteRemote, setDeleteRemote] = useState(false);
  // null = this computer may also delete the account from the relay.
  const deleteNote = deleteAccountNote(o);
  // Leave stays available during a sync code change; the dialog says what leaving does to it.
  const rotationNote = leaveRotationNote(o);

  return (
    <Section title="Advanced" description="The relay only stores encrypted records.">
      <SettingsGroup>
        <SettingsRow label="Leave sync account" description="This computer stops syncing; its space files become local files that ssh keeps reading.">
          <Button
            type="button"
            variant="outline"
            size="sm"
            className="h-7 text-destructive hover:text-destructive"
            onClick={() => {
              setDeleteRemote(false);
              setOpen(true);
            }}
          >
            Leave…
          </Button>
        </SettingsRow>
      </SettingsGroup>

      <AlertDialog
        open={open}
        onOpenChange={(next) => {
          if (!leave.isPending) setOpen(next);
        }}
      >
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Leave the sync account?</AlertDialogTitle>
            <AlertDialogDescription>
              This computer stops syncing. Its space files move to ~/.ssh/sshelter-local/ and keep working as local files: ssh still reads
              them, and you can later move their hosts into another sync account. Joining again needs the sync code.
            </AlertDialogDescription>
          </AlertDialogHeader>
          {rotationNote && <p className={cn("text-sm", TONE_TEXT.warning)}>{rotationNote}</p>}
          {deleteNote === null ? (
            <label className="flex items-start gap-2 text-sm">
              <Checkbox className="mt-0.5" checked={deleteRemote} onCheckedChange={(v) => setDeleteRemote(v === true)} />
              <span>
                Also delete the sync account and every space from the relay. No other computer is listed; without this, the relay deletes them after
                180 days unused.
              </span>
            </label>
          ) : (
            <p className="text-sm text-muted-foreground">{deleteNote}</p>
          )}
          <AlertDialogFooter>
            <AlertDialogCancel disabled={leave.isPending}>Cancel</AlertDialogCancel>
            <AlertDialogAction
              disabled={leave.isPending}
              onClick={(e) => {
                // Keep the dialog open until the request settles (see the change-code dialog).
                e.preventDefault();
                leave.mutate(
                  { deleteRemote: deleteNote === null && deleteRemote },
                  {
                    onSuccess: () => {
                      setOpen(false);
                      toast.success("Left the sync account");
                    },
                  },
                );
              }}
            >
              {leave.isPending && <Loader2 className="size-3.5 animate-spin" />} Leave
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </Section>
  );
}
```

- [ ] **Step 6: 修改 `src/lib/sync.ts`:刪掉舊 pane 用的 v1 helper**

`src/lib/sync.ts`:把

```ts
}

/*
 * v1 — the old Sync pane and migration wizard still use these until they are
 * rewritten for v2; each goes away with its last caller.
 */

export const syncStatusKey = ["sync", "status"] as const;
```

換成:

```ts
}

/*
 * v1 — the old migration wizard still uses these until it is rewritten for v2;
 * they go away with it.
 */

export const syncStatusKey = ["sync", "status"] as const;
```

`src/lib/sync.ts`:把

```ts
  });
}

export function useLeaveChain() {
  return useLeaveAccount();
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
```

換成:

```ts
  });
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
```

`src/lib/sync.ts`:把

```ts
    onError: (error) => toast.error("Could not move hosts", { description: errorMessage(error) }),
  });
}

export function createChain(deviceName: string): Promise<string> {
  return createAccount(deviceName);
}

export function joinChain(words: string, deviceName: string): Promise<SyncOverview> {
  return joinAccount(words, deviceName);
}
```

換成:

```ts
    onError: (error) => toast.error("Could not move hosts", { description: errorMessage(error) }),
  });
}
```

- [ ] **Step 7: 跑測試確認通過**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  17 passed (17)`、`Tests  207 passed (207)`(task 開始前 16 / 181)。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出。

- [ ] **Step 8: Commit**

```bash
git add src/lib/sync-fixtures.ts src/lib/sync-overview.ts src/lib/sync-overview.test.ts
git add src/components/sync-primitives.tsx src/components/SyncPane.tsx src/lib/sync.ts
git commit -m "feat(sync): show the v2 account, devices and sync code in Settings → Sync"
```

---

### Task 3: Spaces:清單、開關、改名、刪除、新增、重建;加入後選擇 space

> **已執行**(repo `8b4d92f`;審查後的修正 `1b2c0a0`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準。
> 修正輪:「Choose spaces」在逐一開啟 space 的期間不能被關掉(Esc、點外面都不行,關閉鈕先藏起來),`onClose` 每次打開只呼叫
> 一次 —— 否則中途關掉之後還會再呼叫一次,Task 6 接上精靈之後會跳出不該出現的精靈。審查的其他 Minor 在最終修正一起改
> (`2358dd9`,T3-a…T3-h):確認與命名對話框淡出時保留原本那一列(不再出現「Stop syncing ""…」),命名對話框在建立 / 改名進行中
> 不能關;Rebuild 的轉圈只在那一列;「Choose spaces」遵守 `structureLock`,失敗時以名稱彙總成一則 toast;更換同步碼的鎖定說明
> 提到可以先取消;資料不見的 space 關掉時不再保證之後可以再打開;兩個確認共用一個對話框;space 名稱輸入框在 IME 組字的 Enter
> 不送出(`isImeKey`);測試釘住 64 個 astral 字元可以、65 個不行,以及 `MAX_SPACE_NAME` 等於 `spaces.rs` 的值。最終修正另外讓
> space 的問題由 `spaceProblem` 統一判斷(解讀 9),別台取的 space 名稱在列、確認與 toast 裡經 `revealHidden`(`3d65c6e`)。執行時
> `src/` 的測試 207 → 214(同計畫)。

Spaces 區塊(spec §8):每個 space 的名稱、檔名、主機數、哪些電腦在同步、狀態(chain 不見了 → 「Rebuild from this
computer」/「Delete space…」,spec §9);開關只影響這台(關掉前確認,說明檔案會備份並移除、其他電腦不受影響);改名;
刪除(確認「everywhere」);「New space…」。名稱規則與後端 `clean_space_name` 相同,對話框先說哪裡不對。加入帳戶之後打開
「Choose spaces for this computer」(解讀 1)。

**Files:**
- Create: `src/lib/sync-spaces.test.ts`、`src/lib/sync-spaces.ts`、`src/components/SyncSpacesSection.tsx`
- Modify: `src/components/SyncPane.tsx`

**Interfaces:**
- Consumes(Task 1、2):`useCreateSpace`、`useRenameSpace`、`useDeleteSpace`、`useSelectSpace`、`useUnselectSpace`、`useRebuildSpace`;
  `listNames`;`plural`、`Tone`;`TONE_TEXT`;測試用 `overview`、`space`、`NOW`。
- Produces(`src/lib/sync-spaces.ts`):`MAX_SPACE_NAME = 64`、`interface SpaceRow { id; name; selected; fileName; detail;
  syncedOn; status; missing; pendingUploads }`、`spaceRows(o)`、`structureLock(o): string | null`、
  `spaceNameError(name, spaces, exceptId?): string | null`
- Produces(`SyncSpacesSection.tsx`):`SpacesSection({ overview })`、`ChooseSpacesDialog({ open, overview, onClose(selected: string[]) })`


- [ ] **Step 1: 寫失敗的測試:`src/lib/sync-spaces.test.ts`**

新增 `src/lib/sync-spaces.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { NOW, overview, space } from "./sync-fixtures";
import { MAX_SPACE_NAME, spaceNameError, spaceRows, structureLock } from "./sync-spaces";

const WORK_ID = "b".repeat(64);

describe("spaceRows", () => {
  it("shows a synced space's file, host count and the computers that sync it", () => {
    expect(spaceRows(overview({ spaces: [space({ synced_on: ["MacBook-A", "Mac-mini"] })] }))).toEqual([
      {
        id: space().id,
        name: "Personal",
        selected: true,
        fileName: "personal-3fa2c1d9.config",
        detail: "personal-3fa2c1d9.config · 3 hosts",
        syncedOn: "On MacBook-A and Mac-mini",
        status: null,
        missing: false,
        pendingUploads: 0,
      },
    ]);
  });

  it("knows nothing about the hosts of a space this computer does not sync", () => {
    const [row] = spaceRows(
      overview({ spaces: [space({ id: WORK_ID, name: "Work", selected: false, file_name: null, file_path: null, hosts: null, synced_on: [] })] }),
    );
    expect(row).toEqual(
      expect.objectContaining({ selected: false, fileName: null, detail: "Not on this computer", syncedOn: "Not synced on any computer" }),
    );
  });

  it("puts the most urgent state of a space first", () => {
    const status = (overrides: Parameters<typeof space>[0]) => spaceRows(overview({ spaces: [space(overrides)] }))[0].status;
    expect(status({ missing: true, last_error: "this space's data is missing on the relay", approvals: 2 })).toEqual({
      tone: "error",
      text: "Its data is missing on the relay. Rebuild it from this computer, or delete the space.",
    });
    expect(status({ last_error: "duplicate Host web", first_sync_pending: true })).toEqual({ tone: "error", text: "duplicate Host web" });
    expect(status({ first_sync_pending: true, approvals: 1 })).toEqual({ tone: "busy", text: "Syncing for the first time…" });
    expect(status({ approvals: 2, pending_uploads: 1 })).toEqual({ tone: "warning", text: "2 hosts waiting for your approval" });
    expect(status({ pending_uploads: 1 })).toEqual({ tone: "ok", text: "1 change waiting to upload" });
  });
});

describe("structureLock", () => {
  it("lets a healthy account change its spaces", () => {
    expect(structureLock(overview())).toBeNull();
  });

  it("says why the spaces cannot change right now", () => {
    expect(structureLock(overview({ frozen: { detected_at_ms: NOW, by_devices: [] } }))).toBe("Enter the new sync code first.");
    expect(structureLock(overview({ rotation: { step: "copying", cancellable: false, paused_until_ms: null } }))).toBe(
      "Spaces can change again once the new sync code is in place.",
    );
    expect(structureLock(overview({ read_only: true }))).toBe("Update SSHelter to change spaces: this sync account uses a newer format.");
    expect(structureLock(overview({ joined: false }))).toBe("Join or create a sync account first.");
  });
});

describe("spaceNameError", () => {
  const spaces = [space(), space({ id: WORK_ID, name: "Work" })];

  it("accepts a new name, and a space's own name when renaming it", () => {
    expect(spaceNameError("Homelab", spaces)).toBeNull();
    expect(spaceNameError("  work  ", spaces, WORK_ID)).toBeNull();
  });

  it("applies the backend's rules: not empty, no control characters, at most 64 characters, unique ignoring case", () => {
    expect(spaceNameError("   ", spaces)).toBe("Enter a name.");
    expect(spaceNameError("Lab\u0007", spaces)).toBe("A name can't contain control characters.");
    expect(spaceNameError("x".repeat(MAX_SPACE_NAME), spaces)).toBeNull();
    expect(spaceNameError("x".repeat(MAX_SPACE_NAME + 1), spaces)).toBe("Use at most 64 characters.");
    expect(spaceNameError("é".repeat(MAX_SPACE_NAME), spaces)).toBeNull(); // characters, not bytes
    expect(spaceNameError(" WORK ", spaces)).toBe("A space named “WORK” already exists.");
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `pnpm exec vitest run --dir src src/lib/sync-spaces.test.ts`
Expected: FAIL —— `Cannot find module './sync-spaces'`。

- [ ] **Step 3: 實作 `src/lib/sync-spaces.ts`**

新增 `src/lib/sync-spaces.ts`:

```ts
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncSpaceView } from "@/bindings/SyncSpaceView";
import { listNames } from "@/lib/sync-events";
import { plural, type Tone } from "@/lib/sync-overview";

/** The backend's limit (`spaces::clean_space_name`), in characters. */
export const MAX_SPACE_NAME = 64;

export interface SpaceRow {
  id: string;
  name: string;
  selected: boolean;
  fileName: string | null;
  /** `work-3fa2c1d9.config · 3 hosts`, or "Not on this computer". */
  detail: string;
  syncedOn: string;
  status: { tone: Tone; text: string } | null;
  /** The relay lost the space's data: offer "Rebuild" and "Delete" (spec §9). */
  missing: boolean;
  pendingUploads: number;
}

function spaceStatus(s: SyncSpaceView): SpaceRow["status"] {
  if (s.missing) return { tone: "error", text: "Its data is missing on the relay. Rebuild it from this computer, or delete the space." };
  if (s.last_error) return { tone: "error", text: s.last_error };
  if (s.first_sync_pending) return { tone: "busy", text: "Syncing for the first time…" };
  if (s.approvals > 0) return { tone: "warning", text: `${plural(s.approvals, "host")} waiting for your approval` };
  if (s.pending_uploads > 0) return { tone: "ok", text: `${plural(s.pending_uploads, "change")} waiting to upload` };
  return null;
}

/** The Spaces list (spec §8), in the backend's order (the order of the Include line). */
export function spaceRows(o: SyncOverview): SpaceRow[] {
  return o.spaces.map((s) => ({
    id: s.id,
    name: s.name,
    selected: s.selected,
    fileName: s.file_name,
    detail: s.selected && s.file_name ? [s.file_name, s.hosts === null ? null : plural(s.hosts, "host")].filter(Boolean).join(" · ") : "Not on this computer",
    syncedOn: s.synced_on.length > 0 ? `On ${listNames(s.synced_on)}` : "Not synced on any computer",
    status: spaceStatus(s),
    missing: s.missing,
    pendingUploads: s.pending_uploads,
  }));
}

/**
 * Why spaces cannot be created, renamed, deleted, turned on or off right now —
 * the same states the backend refuses (`account::account_ready`) — or null.
 */
export function structureLock(o: SyncOverview): string | null {
  if (!o.joined) return "Join or create a sync account first.";
  if (o.frozen) return "Enter the new sync code first.";
  if (o.rotation) return "Spaces can change again once the new sync code is in place.";
  if (o.read_only) return "Update SSHelter to change spaces: this sync account uses a newer format.";
  return null;
}

/**
 * The backend's rules for a space name (`spaces::clean_space_name` and the
 * case-insensitive uniqueness check), so the dialog can say what is wrong before
 * sending. `exceptId` is the space being renamed.
 */
export function spaceNameError(name: string, spaces: readonly SyncSpaceView[], exceptId?: string): string | null {
  const trimmed = name.trim();
  if (trimmed === "") return "Enter a name.";
  if (/\p{Cc}/u.test(trimmed)) return "A name can't contain control characters.";
  if ([...trimmed].length > MAX_SPACE_NAME) return `Use at most ${MAX_SPACE_NAME} characters.`;
  const lower = trimmed.toLowerCase();
  if (spaces.some((s) => s.id !== exceptId && s.name.toLowerCase() === lower)) return `A space named “${trimmed}” already exists.`;
  return null;
}
```

- [ ] **Step 4: 新增 `src/components/SyncSpacesSection.tsx`**

新增 `src/components/SyncSpacesSection.tsx`:

```tsx
import { useState } from "react";
import { Loader2, MoreHorizontal, Pencil, Plus, Trash2 } from "lucide-react";
import { toast } from "sonner";

import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncSpaceView } from "@/bindings/SyncSpaceView";
import { useCreateSpace, useDeleteSpace, useRebuildSpace, useRenameSpace, useSelectSpace, useUnselectSpace } from "@/lib/sync";
import { listNames } from "@/lib/sync-events";
import { plural } from "@/lib/sync-overview";
import { spaceNameError, spaceRows, structureLock, type SpaceRow } from "@/lib/sync-spaces";
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { TONE_TEXT } from "@/components/sync-primitives";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Input } from "@/components/ui/input";
import { Switch } from "@/components/ui/switch";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuSeparator, DropdownMenuTrigger } from "@/components/ui/dropdown-menu";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

type Naming = { mode: "create" } | { mode: "rename"; row: SpaceRow };

/**
 * Settings → Sync → Spaces (spec §8): each space with its file, host count and
 * the computers that sync it; a switch turns it on or off for THIS computer;
 * rename, delete (everywhere) and "New space". Unselecting and deleting are
 * confirmed first, with copy that says exactly which computers lose what.
 */
export function SpacesSection({ overview: o }: { overview: SyncOverview }) {
  const select = useSelectSpace();
  const rebuild = useRebuildSpace();
  const [naming, setNaming] = useState<Naming | null>(null);
  const [unselecting, setUnselecting] = useState<SpaceRow | null>(null);
  const [deleting, setDeleting] = useState<SpaceRow | null>(null);
  const lock = structureLock(o);
  const rows = spaceRows(o);

  return (
    <Section
      title="Spaces"
      description="Each space is a group of hosts with its own file in ~/.ssh/sshelter. Turn a space on to sync it on this computer; turning it off removes only this computer's file."
    >
      <SettingsGroup>
        {rows.length === 0 && (
          <p className="px-3 py-3 text-sm text-muted-foreground">This sync account has no spaces yet. Create one to start syncing hosts.</p>
        )}
        {rows.map((row) => (
          <div key={row.id} className="flex items-start justify-between gap-4 px-3 py-2">
            <div className="min-w-0 space-y-0.5 select-none">
              <p className="truncate text-sm">{row.name}</p>
              <p className="truncate text-xs text-muted-foreground" title={`${row.detail} · ${row.syncedOn}`}>
                {row.detail} · {row.syncedOn}
              </p>
              {row.status && <p className={cn("text-xs", TONE_TEXT[row.status.tone])}>{row.status.text}</p>}
              {row.missing && (
                <div className="flex gap-1.5 pt-1">
                  <Button type="button" variant="outline" size="sm" className="h-7" disabled={lock !== null || rebuild.isPending} onClick={() => rebuild.mutate({ spaceId: row.id })}>
                    {rebuild.isPending && <Loader2 className="size-3.5 animate-spin" />} Rebuild from this computer
                  </Button>
                  <Button type="button" variant="outline" size="sm" className="h-7 text-destructive hover:text-destructive" disabled={lock !== null} onClick={() => setDeleting(row)}>
                    Delete space…
                  </Button>
                </div>
              )}
            </div>
            <div className="flex shrink-0 items-center gap-1 pt-0.5">
              <Switch
                checked={row.selected}
                disabled={lock !== null || select.isPending}
                aria-label={`Sync ${row.name} on this computer`}
                onCheckedChange={(on) => (on ? select.mutate({ spaceId: row.id }) : setUnselecting(row))}
              />
              <DropdownMenu>
                <DropdownMenuTrigger asChild>
                  <Button type="button" variant="ghost" size="icon" className="size-7 text-muted-foreground" aria-label={`Actions for ${row.name}`} disabled={lock !== null}>
                    <MoreHorizontal className="size-3.5" />
                  </Button>
                </DropdownMenuTrigger>
                <DropdownMenuContent align="end">
                  <DropdownMenuItem onSelect={() => setNaming({ mode: "rename", row })}>
                    <Pencil className="size-3.5" /> Rename…
                  </DropdownMenuItem>
                  <DropdownMenuSeparator />
                  <DropdownMenuItem variant="destructive" onSelect={() => setDeleting(row)}>
                    <Trash2 className="size-3.5" /> Delete…
                  </DropdownMenuItem>
                </DropdownMenuContent>
              </DropdownMenu>
            </div>
          </div>
        ))}
        <SettingsRow label="New space" description={lock ?? "Starts empty and syncs on this computer."}>
          <Button type="button" variant="outline" size="sm" className="h-7" disabled={lock !== null} onClick={() => setNaming({ mode: "create" })}>
            <Plus className="size-3.5" /> New space…
          </Button>
        </SettingsRow>
      </SettingsGroup>

      <Dialog
        open={naming !== null}
        onOpenChange={(open) => {
          if (!open) setNaming(null);
        }}
      >
        <DialogContent className="sm:max-w-sm">
          {naming && <SpaceNameForm key={naming.mode === "rename" ? naming.row.id : "new"} naming={naming} spaces={o.spaces} onClose={() => setNaming(null)} />}
        </DialogContent>
      </Dialog>
      <UnselectDialog row={unselecting} onClose={() => setUnselecting(null)} />
      <DeleteDialog row={deleting} onClose={() => setDeleting(null)} />
    </Section>
  );
}

function SpaceNameForm({ naming, spaces, onClose }: { naming: Naming; spaces: SyncSpaceView[]; onClose: () => void }) {
  const create = useCreateSpace();
  const rename = useRenameSpace();
  const current = naming.mode === "rename" ? naming.row.name : "";
  const [name, setName] = useState(current);
  const error = spaceNameError(name, spaces, naming.mode === "rename" ? naming.row.id : undefined);
  const pending = create.isPending || rename.isPending;
  const disabled = error !== null || name.trim() === current || pending;

  const submit = () => {
    if (disabled) return;
    const trimmed = name.trim();
    if (naming.mode === "create") {
      create.mutate(
        { name: trimmed },
        {
          onSuccess: () => {
            toast.success(`Created “${trimmed}”`);
            onClose();
          },
        },
      );
    } else {
      rename.mutate(
        { spaceId: naming.row.id, name: trimmed },
        {
          onSuccess: () => {
            toast.success(`Renamed to “${trimmed}”`);
            onClose();
          },
        },
      );
    }
  };

  return (
    <>
      <DialogHeader>
        <DialogTitle>{naming.mode === "create" ? "New space" : `Rename “${current}”`}</DialogTitle>
        <DialogDescription>
          {naming.mode === "create"
            ? "The space starts empty and syncs on this computer. Its file in ~/.ssh/sshelter is named after it."
            : "Every computer that syncs this space renames its file to match."}
        </DialogDescription>
      </DialogHeader>
      <Input
        autoFocus
        value={name}
        onChange={(e) => setName(e.target.value)}
        onKeyDown={(e) => {
          if (e.key === "Enter") submit();
        }}
        aria-label="Space name"
        placeholder="Work"
        className="h-8"
      />
      {name.trim() !== "" && error && <p className="text-xs text-destructive">{error}</p>}
      <DialogFooter>
        <Button type="button" variant="outline" onClick={onClose}>
          Cancel
        </Button>
        <Button type="button" disabled={disabled} onClick={submit}>
          {pending && <Loader2 className="size-4 animate-spin" />} {naming.mode === "create" ? "Create" : "Rename"}
        </Button>
      </DialogFooter>
    </>
  );
}

/** Turning a space off removes only this computer's file (backed up first); the space stays everywhere else. */
function UnselectDialog({ row, onClose }: { row: SpaceRow | null; onClose: () => void }) {
  const unselect = useUnselectSpace();
  return (
    <AlertDialog
      open={row !== null}
      onOpenChange={(open) => {
        if (!open && !unselect.isPending) onClose();
      }}
    >
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Stop syncing “{row?.name}” on this computer?</AlertDialogTitle>
          <AlertDialogDescription>
            <span className="font-mono">{row?.fileName}</span> is backed up and removed from this computer, and ssh stops reading it. The space stays in your sync
            account and on your other computers; turn it back on any time.
            {row && row.pendingUploads > 0 && ` ${plural(row.pendingUploads, "change")} made here and not uploaded yet won't reach them — the backup keeps them.`}
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={unselect.isPending}>Cancel</AlertDialogCancel>
          <AlertDialogAction
            disabled={unselect.isPending}
            onClick={(e) => {
              // Keep the dialog open until the request settles.
              e.preventDefault();
              if (row) unselect.mutate({ spaceId: row.id }, { onSuccess: onClose });
            }}
          >
            {unselect.isPending && <Loader2 className="size-3.5 animate-spin" />} Remove from this computer
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}

/** Deleting removes the space on every computer and on the relay. */
function DeleteDialog({ row, onClose }: { row: SpaceRow | null; onClose: () => void }) {
  const del = useDeleteSpace();
  return (
    <AlertDialog
      open={row !== null}
      onOpenChange={(open) => {
        if (!open && !del.isPending) onClose();
      }}
    >
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>Delete “{row?.name}” everywhere?</AlertDialogTitle>
          <AlertDialogDescription>
            The space and its hosts are removed from every computer that syncs it — each one backs up its file first — and from the relay. This can't be
            undone. To stop syncing it only on this computer, turn it off instead.
          </AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel disabled={del.isPending}>Cancel</AlertDialogCancel>
          <AlertDialogAction
            variant="destructive"
            disabled={del.isPending}
            onClick={(e) => {
              e.preventDefault();
              if (row) {
                const name = row.name;
                del.mutate(
                  { spaceId: row.id },
                  {
                    onSuccess: () => {
                      toast.success(`Deleted “${name}”`);
                      onClose();
                    },
                  },
                );
              }
            }}
          >
            {del.isPending && <Loader2 className="size-3.5 animate-spin" />} Delete space
          </AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}

/**
 * Right after joining (which selects no space): pick the spaces to sync here.
 * Every space starts checked; each one chosen gets its file and a first sync.
 * `onClose` receives the ids that were turned on.
 */
export function ChooseSpacesDialog({ open, overview, onClose }: { open: boolean; overview: SyncOverview | undefined; onClose: (selected: string[]) => void }) {
  return (
    <Dialog
      open={open}
      onOpenChange={(next) => {
        if (!next) onClose([]);
      }}
    >
      <DialogContent className="sm:max-w-md">{open && overview && <ChooseSpacesForm overview={overview} onClose={onClose} />}</DialogContent>
    </Dialog>
  );
}

function ChooseSpacesForm({ overview, onClose }: { overview: SyncOverview; onClose: (selected: string[]) => void }) {
  const select = useSelectSpace();
  // Fixed when the dialog opens: each space drops out of `overview` as soon as it is turned on.
  const [candidates] = useState(() => overview.spaces.filter((s) => !s.selected));
  const [chosen, setChosen] = useState(() => new Set(candidates.map((s) => s.id)));
  const [busy, setBusy] = useState(false);

  const toggle = (id: string, on: boolean) =>
    setChosen((prev) => {
      const next = new Set(prev);
      if (on) next.add(id);
      else next.delete(id);
      return next;
    });

  const run = async () => {
    setBusy(true);
    const done: string[] = [];
    for (const s of candidates.filter((c) => chosen.has(c.id))) {
      try {
        await select.mutateAsync({ spaceId: s.id });
        done.push(s.id);
      } catch {
        // The mutation already showed why; keep going with the others.
      }
    }
    setBusy(false);
    onClose(done);
  };

  return (
    <>
      <DialogHeader>
        <DialogTitle>Choose spaces for this computer</DialogTitle>
        <DialogDescription>
          Each space you choose gets its own file in ~/.ssh/sshelter and syncs from now on. You can change this any time under Settings → Sync → Spaces.
        </DialogDescription>
      </DialogHeader>
      {candidates.length === 0 ? (
        <p className="text-sm text-muted-foreground">This sync account has no spaces yet. Create one under Spaces.</p>
      ) : (
        <div className="max-h-[40vh] space-y-2 overflow-y-auto pr-1">
          {candidates.map((s) => (
            <label key={s.id} className="flex items-start gap-2 text-sm">
              <Checkbox className="mt-0.5" checked={chosen.has(s.id)} disabled={busy} onCheckedChange={(v) => toggle(s.id, v === true)} />
              <span className="min-w-0">
                <span className="block truncate">{s.name}</span>
                <span className="block truncate text-xs text-muted-foreground">
                  {s.synced_on.length > 0 ? `On ${listNames(s.synced_on)}` : "Not synced on any computer"}
                </span>
              </span>
            </label>
          ))}
        </div>
      )}
      <DialogFooter>
        <Button type="button" variant="outline" disabled={busy} onClick={() => onClose([])}>
          Not now
        </Button>
        {candidates.length > 0 && (
          <Button type="button" disabled={busy || chosen.size === 0} onClick={() => void run()}>
            {busy && <Loader2 className="size-4 animate-spin" />} Sync {plural(chosen.size, "space")}
          </Button>
        )}
      </DialogFooter>
    </>
  );
}
```

- [ ] **Step 5: 修改 `src/components/SyncPane.tsx`:Spaces 區塊與加入後的選擇對話框**

`src/components/SyncPane.tsx`:把

```tsx
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { SyncCodeDialog, TONE_TEXT } from "@/components/sync-primitives";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
```

換成:

```tsx
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { SyncCodeDialog, TONE_TEXT } from "@/components/sync-primitives";
import { ChooseSpacesDialog, SpacesSection } from "@/components/SyncSpacesSection";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
```

`src/components/SyncPane.tsx`:把

```tsx
  const [createdWords, setCreatedWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // shown on request
  const [newCode, setNewCode] = useState<{ words: string; noticeIndex: number } | null>(null); // after changing it

  if (!overview.data) {
    // A failed overview query never turns into data: show why instead of loading forever.
```

換成:

```tsx
  const [createdWords, setCreatedWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // shown on request
  const [newCode, setNewCode] = useState<{ words: string; noticeIndex: number } | null>(null); // after changing it
  const [choosingSpaces, setChoosingSpaces] = useState(false); // right after joining, which selects no space

  if (!overview.data) {
    // A failed overview query never turns into data: show why instead of loading forever.
```

`src/components/SyncPane.tsx`:把

```tsx
          onShowNewCode={(words, noticeIndex) => setNewCode({ words, noticeIndex })}
        />
      ) : (
        <NotJoinedPane overview={overview.data} onCreated={setCreatedWords} />
      )}

      <SyncCodeDialog
        mode="created"
```

換成:

```tsx
          onShowNewCode={(words, noticeIndex) => setNewCode({ words, noticeIndex })}
        />
      ) : (
        <NotJoinedPane overview={overview.data} onCreated={setCreatedWords} onJoined={() => setChoosingSpaces(true)} />
      )}

      <ChooseSpacesDialog open={choosingSpaces} overview={overview.data} onClose={() => setChoosingSpaces(false)} />

      <SyncCodeDialog
        mode="created"
```

`src/components/SyncPane.tsx`:把

```tsx
}

/** Create or join. Errors keep the form as it was so the user can fix a typo. */
function NotJoinedPane({ overview: o, onCreated }: { overview: SyncOverview; onCreated: (words: string) => void }) {
  const queryClient = useQueryClient();
  const [deviceName, setDeviceName] = useState(o.device_name);
  const [words, setWords] = useState("");
```

換成:

```tsx
}

/** Create or join. Errors keep the form as it was so the user can fix a typo. */
function NotJoinedPane({
  overview: o,
  onCreated,
  onJoined,
}: {
  overview: SyncOverview;
  onCreated: (words: string) => void;
  onJoined: () => void;
}) {
  const queryClient = useQueryClient();
  const [deviceName, setDeviceName] = useState(o.device_name);
  const [words, setWords] = useState("");
```

`src/components/SyncPane.tsx`:把

```tsx
      queryClient.setQueryData(syncOverviewKey, joined);
      refreshSyncViews(queryClient);
      toast.success("Joined the sync account");
    } catch (error) {
      // Keep the pasted words so a typo can be fixed.
      toast.error("Could not join the sync account", { description: errorMessage(error) });
```

換成:

```tsx
      queryClient.setQueryData(syncOverviewKey, joined);
      refreshSyncViews(queryClient);
      toast.success("Joined the sync account");
      onJoined();
    } catch (error) {
      // Keep the pasted words so a typo can be fixed.
      toast.error("Could not join the sync account", { description: errorMessage(error) });
```

`src/components/SyncPane.tsx`:把

```tsx
        </SettingsGroup>
      </Section>

      <AccountSection overview={o} reading={reading === "code"} onShowCode={() => void readWords("code", onShowWords)} />
      <DevicesSection overview={o} />
      <LeaveSection overview={o} />
```

換成:

```tsx
        </SettingsGroup>
      </Section>

      <SpacesSection overview={o} />
      <AccountSection overview={o} reading={reading === "code"} onShowCode={() => void readWords("code", onShowWords)} />
      <DevicesSection overview={o} />
      <LeaveSection overview={o} />
```

- [ ] **Step 6: 跑測試確認通過**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  18 passed (18)`、`Tests  214 passed (214)`。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出。

- [ ] **Step 7: Commit**

```bash
git add src/lib/sync-spaces.ts src/lib/sync-spaces.test.ts src/components/SyncSpacesSection.tsx src/components/SyncPane.tsx
git commit -m "feat(sync): list, create, rename, delete and choose spaces"
```

---

### Task 4: 等待核准:審核對話框與 `sync://approval`

> **已執行**(repo `8fd2583`;兩輪審查後的修正 `bb2b763`、`510fb99`;後端另外的修正 `bf37fe6`)。下面保留原本的步驟作為紀錄,
> 不要再執行;實際的程式碼以 repo 為準 —— 這個對話框是安全邊界,兩輪修正改了很多,`src/lib/sync-approvals.ts`、
> `src/components/SyncApprovalDialog.tsx` 都和下面的區塊不同,另外多了 `src/components/SyncApprovalDialog.test.tsx`(server render)。
> - 第一輪(`bb2b763`):換上新清單時,對話框開著時換了內容或新出現的主機在自己的卡片上標出來,直到使用者對它做了決定,通知
>   也列出它們與所在的 space(`adoptNewest`);`revealHidden` 也顯示畫不出東西的字元(Default_Ignorable,例如 U+3164,與
>   U+2800);不在 space 檔裡的主機寫「Not in <space> on this computer yet」而不是「New host」,設定檔裡已有同名主機時說核准會
>   影響哪一份,多個名稱的 Host 行也標色;重開時等重新讀取完成才定下清單(`isSettled`);載入失敗時藏起「Approve all」/「Reject
>   all」;轉圈在正在跑的按鈕上,決定進行中不能關;forward 的值照 Rust 的方式修剪空白;通知與 toast 的 alias 經 `revealHidden`;
>   「Approve all」只回報做成的;變更清單保留空白;測試同時釘住 `GATED_SPELLINGS`。
> - 第二輪(`510fb99`):沒有依附對象的組合記號一次掃描就顯示出來;U+1D159、U+13441、U+13442 也顯示;同名主機依實際的讀取
>   順序說誰先讀(`takeoverRank`;自己 space 檔裡另一個區塊帶這個名稱時是「A new block in <space>」),也檢查擴大的範圍新加的
>   名稱;畫面上沒有主機時新來的直接顯示並標成新的(`adoptAtOnce`);「Close」移到 footer 最右邊(`ReviewFooter`);`plural`
>   搬到 `format.ts`(避免 import cycle)。
> - 後端(`bf37fe6`,經同意的例外;spec §7.4 於 `e704189` 補上):`hosts_file.rs` 在 OpenSSH 註解以外拒絕 Default_Ignorable 與
>   畫出來是空白的 U+2800、U+1D159、U+13441、U+13442、U+303F,並在整行交給 shell 的五個 keyword 裡拒絕詞首的組合記號或
>   非 ASCII 符號(`Forbidden::StrayCharacter`)。Rust 740 → 746。
> - 最終修正:takeover 的說明改成 ssh 實際的合併方式(解讀 4、6);帳戶鎖住時停用決定(解讀 11);清單或清單上方的內容移動
>   之後 600 ms 的點擊保護(解讀 5);「settings that run programs…」那句只剩一個常數(`GATED_SETTINGS`),行的切法與 keyword
>   的 pattern 各只有一份;後端拒絕與 `revealHidden` 的對照測試(Part B,見文末)。
>
> 執行時 `src/` 的測試 214 → 235(同計畫)→ 268(第一輪)→ 285(第二輪)。

審核對話框(spec §7.4、§8):依 space 分組列出等待核准的主機,每台顯示來源裝置與時間、核准會造成的變更(解讀 6)、
完整區塊(受管制的行標色)、可展開的目前版本;逐台或全部核准 / 拒絕 —— 只送畫面上那幾版的 `{ alias, digest }`,版本在
對話框開著時變了就說明並換上新版本(解讀 5)。`sync://approval` 讓核准清單失效並跳出帶
「Review」的 toast(解讀 5);Settings → Sync 的狀態區加上「N hosts waiting for your approval」。受管制關鍵字的清單以
測試對照 `src-tauri/src/sync/approval.rs` 的 `GATED_KEYWORDS`(repo `c82d359` 時 24 個,依後端順序、以文件上的拼法
標示);不是只有埠號的 `LocalForward` / `DynamicForward` 也進簽章,也有標籤。對話框以 `revealHidden` 顯示別台來的文字、
依邏輯順序排版(解讀 6)。

**Files:**
- Create: `src/lib/sync-approvals.test.ts`、`src/lib/sync-approvals.ts`、`src/components/SyncApprovalDialog.tsx`
- Modify: `src/lib/sync-events.test.ts`、`src/lib/sync-events.ts`、`src/stores/ui.ts`、`src/components/SyncPane.tsx`、`src/App.tsx`

**Interfaces:**
- Consumes(Task 1、2):`usePendingApprovals`、`useApproveHosts`、`useRejectHosts`(`{ spaceId, approvals }` →
  `ReviewOutcome`)、`errorMessage`、`syncApprovalsKey`;`SyncMessage`;`plural`;`relativeTime`;測試用 `NOW`、`overview`;
  bindings `PendingApprovalView`、`ReviewedVersion`、`ReviewOutcome`、`ApprovalNotice`、`GatedDirective`。
- Produces(`src/lib/sync-approvals.ts`):`GATED_KEYWORDS`(常見拼法)、`keywordLabel(keyword)`(含 `LocalForward`、
  `DynamicForward`)、`revealHidden(text): string`、`displayLines(text): string[]`、`lineKeyword(line): string | null`、
  `interface BlockLine { text; gated; scope }`、`blockLines(view)`、`type ApprovalChange`(`new_host` / `scope` / `added` /
  `removed` / `changed` / `moved`)、`approvalChanges(view)`、`changeText(change)`、`interface ApprovalGroup { spaceId;
  spaceName; views }`、`approvalGroups(views)`、`reviewedVersions(views): ReviewedVersion[]`、`sameVersions(a, b): boolean`、
  `combineOutcomes(outcomes): { applied; changed }`、`changedNotice(changed): string | null`、
  `approvalMessage(notices): SyncMessage | null`
- Produces(`src/stores/ui.ts`):`syncApprovalsOpen: boolean`、`setSyncApprovalsOpen(open: boolean)`
- Produces:`SyncApprovalDialog`(掛在 App)


- [ ] **Step 1: 寫失敗的測試:`src/lib/sync-approvals.test.ts`**

第一個測試讀 `src-tauri/src/sync/approval.rs`,確認前端的關鍵字清單與後端的 `GATED_KEYWORDS` 一致(同 `sync.test.ts` 讀 README 的作法)。

新增 `src/lib/sync-approvals.test.ts`:

```ts
import { readFileSync } from "node:fs";
import { describe, expect, it } from "vitest";

import type { PendingApprovalView } from "@/bindings/PendingApprovalView";
import { NOW, overview } from "./sync-fixtures";
import {
  GATED_KEYWORDS,
  approvalChanges,
  approvalGroups,
  approvalMessage,
  blockLines,
  changeText,
  changedNotice,
  combineOutcomes,
  displayLines,
  lineKeyword,
  revealHidden,
  reviewedVersions,
  sameVersions,
} from "./sync-approvals";

function pending(overrides: Partial<PendingApprovalView> = {}): PendingApprovalView {
  return {
    space_id: "a".repeat(64),
    space_name: "Work",
    alias: "web",
    digest: "d1",
    text: "Host web\n  HostName 10.0.0.1\n  ProxyCommand nc %h 22\n",
    current_text: null,
    applied: { host: "", gated: [] },
    incoming: { host: "web", gated: [{ keyword: "proxycommand", value: "nc %h 22" }] },
    from_device: "MacBook-B",
    updated_at_ms: NOW,
    ...overrides,
  };
}

describe("GATED_KEYWORDS", () => {
  it("is the backend's list (approval::GATED_KEYWORDS)", () => {
    const rust = readFileSync("src-tauri/src/sync/approval.rs", "utf8");
    const list = /pub const GATED_KEYWORDS: \[&str; \d+\] = \[([^\]]*)\]/.exec(rust)?.[1] ?? "";
    const backend = [...list.matchAll(/"([a-z0-9]+)"/g)].map((m) => m[1]);
    expect(backend.length).toBeGreaterThan(0);
    expect(GATED_KEYWORDS.map((k) => k.toLowerCase())).toEqual(backend);
  });
});

describe("revealHidden", () => {
  it("shows the characters that could make a line read differently from what ssh runs", () => {
    expect(revealHidden("nc %h 22 #\u202Eevil")).toBe("nc %h 22 #⟨U+202E⟩evil"); // right-to-left override
    expect(revealHidden("a\u200Bb\u2066c\u2069")).toBe("a⟨U+200B⟩b⟨U+2066⟩c⟨U+2069⟩"); // zero-width space, isolates
    expect(revealHidden("x\u00A0y\uFEFFz\u0007")).toBe("x⟨U+00A0⟩y⟨U+FEFF⟩z⟨U+0007⟩");
  });

  it("leaves ordinary text alone, tabs and non-Latin letters included", () => {
    expect(revealHidden("\tProxyCommand nc café 日本 %h")).toBe("\tProxyCommand nc café 日本 %h");
  });

  it("splits a block into display lines, CRLF included", () => {
    expect(displayLines("Host web\r\n  User\u200B root\r\n")).toEqual(["Host web", "  User⟨U+200B⟩ root"]);
  });
});

describe("lineKeyword", () => {
  it("reads the keyword in either spelling, ignoring case", () => {
    expect(lineKeyword("  ProxyCommand nc %h 22")).toBe("proxycommand");
    expect(lineKeyword("\tforwardagent=yes")).toBe("forwardagent");
    expect(lineKeyword("FORWARDX11 yes")).toBe("forwardx11");
    expect(lineKeyword("Host web prod")).toBe("host");
  });

  it("has none for comments, disabled lines and blanks", () => {
    expect(lineKeyword("  # ProxyCommand nc %h 22")).toBeNull();
    expect(lineKeyword("")).toBeNull();
    expect(lineKeyword("   ")).toBeNull();
  });
});

describe("blockLines", () => {
  it("marks the gated settings of the incoming block", () => {
    expect(blockLines(pending())).toEqual([
      { text: "Host web", gated: false, scope: false },
      { text: "  HostName 10.0.0.1", gated: false, scope: false },
      { text: "  ProxyCommand nc %h 22", gated: true, scope: false },
    ]);
  });

  it("marks the Host line when an existing host now applies to more names", () => {
    const view = pending({
      text: "Host web prod\n  ForwardAgent=yes\n",
      current_text: "Host web\n  ForwardAgent yes\n",
      applied: { host: "web", gated: [{ keyword: "forwardagent", value: "yes" }] },
      incoming: { host: "web prod", gated: [{ keyword: "forwardagent", value: "yes" }] },
    });
    expect(blockLines(view)).toEqual([
      { text: "Host web prod", gated: false, scope: true },
      { text: "  ForwardAgent=yes", gated: true, scope: false },
    ]);
  });
});

describe("forwards and hidden characters", () => {
  it("marks a forward the backend gates (a bind address) but not one with only a port, and labels it", () => {
    const view = pending({
      text: "Host web\n  LocalForward 8080 db:80\n  LocalForward *:5432 db:5432\n  DynamicForward 0.0.0.0:1080\n",
      incoming: {
        host: "web",
        gated: [
          { keyword: "localforward", value: "*:5432 db:5432" },
          { keyword: "dynamicforward", value: "0.0.0.0:1080" },
        ],
      },
    });
    expect(blockLines(view).map((line) => line.gated)).toEqual([false, false, true, true]);
    expect(approvalChanges(view).map(changeText)).toEqual([
      "New host on this computer: Host web",
      "Adds LocalForward *:5432 db:5432",
      "Adds DynamicForward 0.0.0.0:1080",
    ]);
  });

  it("reveals bidi and zero-width characters in the block and in the changes", () => {
    const view = pending({
      text: "Host web\r\n  ProxyCommand nc %h 22 #\u202E evil\r\n",
      incoming: { host: "web", gated: [{ keyword: "proxycommand", value: "nc %h 22 #\u202E evil" }] },
    });
    expect(blockLines(view)).toEqual([
      { text: "Host web", gated: false, scope: false },
      { text: "  ProxyCommand nc %h 22 #⟨U+202E⟩ evil", gated: true, scope: false },
    ]);
    expect(approvalChanges(view).map(changeText)).toContain("Adds ProxyCommand nc %h 22 #⟨U+202E⟩ evil");
  });
});

describe("approvalChanges", () => {
  it("lists a new host and each gated setting it brings", () => {
    const changes = approvalChanges(pending());
    expect(changes).toEqual([
      { kind: "new_host", host: "web" },
      { kind: "added", keyword: "proxycommand", value: "nc %h 22" },
    ]);
    expect(changes.map(changeText)).toEqual(["New host on this computer: Host web", "Adds ProxyCommand nc %h 22"]);
  });

  it("covers the settings added in review: commands on the server and host-key checks", () => {
    const view = pending({
      text: "Host web\n  StrictHostKeyChecking no\n  RemoteCommand tmux attach\n",
      incoming: {
        host: "web",
        gated: [
          { keyword: "stricthostkeychecking", value: "no" },
          { keyword: "remotecommand", value: "tmux attach" },
        ],
      },
    });
    expect(blockLines(view).map((line) => line.gated)).toEqual([false, true, true]);
    expect(approvalChanges(view).map(changeText)).toEqual([
      "New host on this computer: Host web",
      "Adds StrictHostKeyChecking no",
      "Adds RemoteCommand tmux attach",
    ]);
  });

  it("shows a changed value, a removal and a wider Host line, and leaves untouched settings out", () => {
    const view = pending({
      current_text: "Host web\n  LocalCommand a\n  ProxyCommand nc %h 22\n  ForwardAgent yes\n",
      applied: {
        host: "web",
        gated: [
          { keyword: "localcommand", value: "a" },
          { keyword: "proxycommand", value: "nc %h 22" },
          { keyword: "forwardagent", value: "yes" },
        ],
      },
      incoming: {
        host: "web prod",
        gated: [
          { keyword: "localcommand", value: "b" },
          { keyword: "forwardagent", value: "yes" },
        ],
      },
    });
    expect(approvalChanges(view).map(changeText)).toEqual([
      "Applies to: web → web prod",
      "LocalCommand: a → b",
      "Removes ProxyCommand nc %h 22",
    ]);
  });

  it("calls a reordered setting a reorder (the order decides which value ssh uses)", () => {
    const view = pending({
      current_text: "Host web\n  ProxyCommand x\n  LocalCommand y\n",
      applied: { host: "web", gated: [{ keyword: "proxycommand", value: "x" }, { keyword: "localcommand", value: "y" }] },
      incoming: { host: "web", gated: [{ keyword: "localcommand", value: "y" }, { keyword: "proxycommand", value: "x" }] },
    });
    expect(approvalChanges(view)).toEqual([{ kind: "moved", keyword: "localcommand", value: "y" }]);
    expect(approvalChanges(view).map(changeText)).toEqual(["Order changed: LocalCommand y"]);
  });

  it("adds a second setting of the same kind next to an existing one", () => {
    const view = pending({
      current_text: "Host web\n  RemoteForward 9000 localhost:9000\n",
      applied: { host: "web", gated: [{ keyword: "remoteforward", value: "9000 localhost:9000" }] },
      incoming: {
        host: "web",
        gated: [
          { keyword: "remoteforward", value: "9000 localhost:9000" },
          { keyword: "remoteforward", value: "5432 db:5432" },
        ],
      },
    });
    expect(approvalChanges(view).map(changeText)).toEqual(["Adds RemoteForward 5432 db:5432"]);
  });
});

describe("approvalGroups", () => {
  it("groups by space in the order the backend listed them", () => {
    const personal = { space_id: "p".repeat(64), space_name: "Personal" };
    const groups = approvalGroups([pending(), pending({ ...personal, alias: "nas" }), pending({ alias: "db" })]);
    expect(groups.map((g) => [g.spaceName, g.views.map((v) => v.alias)])).toEqual([
      ["Work", ["web", "db"]],
      ["Personal", ["nas"]],
    ]);
    expect(groups[0].spaceId).toBe("a".repeat(64));
  });
});

describe("deciding on exactly the versions shown", () => {
  it("sends back the alias and digest of each shown version", () => {
    expect(reviewedVersions([pending(), pending({ alias: "db", digest: "d2" })])).toEqual([
      { alias: "web", digest: "d1" },
      { alias: "db", digest: "d2" },
    ]);
  });

  it("notices when the waiting list moved on: new content for a host, a new host, one gone", () => {
    const shown = [pending(), pending({ alias: "db", digest: "d2" })];
    expect(sameVersions(shown, [shown[1], shown[0]])).toBe(true);
    // Pulled again later, the same version keeps its digest; a renamed space does not make it new either.
    expect(sameVersions(shown, shown.map((v) => ({ ...v, space_name: "Office" })))).toBe(true);
    expect(sameVersions(shown, [pending({ digest: "d3" }), shown[1]])).toBe(false);
    expect(sameVersions(shown, [shown[0]])).toBe(false);
    expect(sameVersions(shown, [...shown, pending({ alias: "nas", digest: "d4" })])).toBe(false);
    expect(sameVersions(shown, [shown[0], pending({ space_id: "b".repeat(64), alias: "db", digest: "d2" })])).toBe(false);
  });

  it("says which hosts changed while the review was open, and nothing when none did", () => {
    const outcome = (applied: number, changed: string[]) => ({ applied, changed, overview: overview() });
    expect(combineOutcomes([outcome(2, []), outcome(0, ["db"]), outcome(1, ["nas"])])).toEqual({ applied: 3, changed: ["db", "nas"] });
    expect(changedNotice([])).toBeNull();
    expect(changedNotice(["web"])).toBe("web changed since you opened this — review it again.");
    expect(changedNotice(["web", "db"])).toBe("web, db changed since you opened this — review them again.");
  });
});

describe("approvalMessage", () => {
  it("names a single host, counts several, and stays quiet for none", () => {
    expect(approvalMessage([{ space_id: "a", space_name: "Work", aliases: ["web"] }])?.title).toBe("web needs your approval");
    expect(
      approvalMessage([
        { space_id: "a", space_name: "Work", aliases: ["web", "db"] },
        { space_id: "b", space_name: "Personal", aliases: ["nas"] },
      ]),
    ).toEqual({
      title: "3 synced hosts need your approval",
      description:
        "They came from another computer with settings that run programs, share your credentials, environment or network, or relax host-key checks. Nothing changes until you approve them.",
    });
    expect(approvalMessage([])).toBeNull();
  });
});
```

- [ ] **Step 2: 寫失敗的測試:`src/lib/sync-events.test.ts` 加上 `sync://approval`**

`src/lib/sync-events.test.ts`:把

```ts
  afterEach(() => {
    for (const t of toast.getToasts()) toast.dismiss(t.id);
    vi.unstubAllGlobals();
    useUiStore.setState({ settingsOpen: false, settingsCategory: "general" });
  });

  async function subscribed() {
    const bus = stubEventBus();
    const queryClient = new QueryClient();
    const stop = subscribeSyncEvents(queryClient);
    await vi.waitFor(() => expect(bus.handlers.size).toBe(4));
    return { ...bus, queryClient, stop };
  }

  it("listens to the engine's events and never asks for a sync round itself", async () => {
    const { handlers, commands, emit, stop } = await subscribed();
    expect([...handlers.keys()].sort()).toEqual(["sync://applied", "sync://conflict", "sync://notice", "sync://status"]);
    // Every handler runs once: only the "Sync now" button may start a round.
    emit("sync://status", { joined: true, device_name: "MacBook-A" });
    emit("sync://applied", 1);
    emit("sync://conflict", [{ space_id: "a", space_name: "Work", aliases: ["web"] }]);
    emit("sync://notice", { kind: "space_deleted", name: "Work", by_device: "MacBook-A" });
    stop();
    expect(commands.filter((c) => c.startsWith("sync_"))).toEqual([]);
  });
```

換成:

```ts
  afterEach(() => {
    for (const t of toast.getToasts()) toast.dismiss(t.id);
    vi.unstubAllGlobals();
    useUiStore.setState({ settingsOpen: false, settingsCategory: "general", syncApprovalsOpen: false });
  });

  async function subscribed() {
    const bus = stubEventBus();
    const queryClient = new QueryClient();
    const stop = subscribeSyncEvents(queryClient);
    await vi.waitFor(() => expect(bus.handlers.size).toBe(5));
    return { ...bus, queryClient, stop };
  }

  it("listens to the engine's events and never asks for a sync round itself", async () => {
    const { handlers, commands, emit, stop } = await subscribed();
    expect([...handlers.keys()].sort()).toEqual(["sync://applied", "sync://approval", "sync://conflict", "sync://notice", "sync://status"]);
    // Every handler runs once: only the "Sync now" button may start a round.
    emit("sync://status", { joined: true, device_name: "MacBook-A" });
    emit("sync://applied", 1);
    emit("sync://approval", [{ space_id: "a", space_name: "Work", aliases: ["web"] }]);
    emit("sync://conflict", [{ space_id: "a", space_name: "Work", aliases: ["web"] }]);
    emit("sync://notice", { kind: "space_deleted", name: "Work", by_device: "MacBook-A" });
    stop();
    expect(commands.filter((c) => c.startsWith("sync_"))).toEqual([]);
  });
```

`src/lib/sync-events.test.ts`:把

```ts
    ]);
  });

  it("toasts notices with a way to Settings → Sync, except the upgrade (it has its own dialog)", async () => {
    const { emit } = await subscribed();
    emit("sync://notice", { kind: "upgraded", kept_file: null, kept_hosts: [], moved_files: [] });
```

換成:

```ts
    ]);
  });

  it("announces hosts waiting for approval with a button that opens the review", async () => {
    const { emit, queryClient } = await subscribed();
    queryClient.setQueryData(syncApprovalsKey, []);
    emit("sync://approval", [{ space_id: "a", space_name: "Work", aliases: ["web", "db"] }]);
    expect(queryClient.getQueryState(syncApprovalsKey)?.isInvalidated).toBe(true);
    const [shown] = toast.getToasts();
    expect(shown).toEqual(expect.objectContaining({ title: "2 synced hosts need your approval" }));
    const action = "action" in shown ? shown.action : undefined;
    if (!action || typeof action !== "object" || !("onClick" in action)) throw new Error("the toast has no button");
    action.onClick(undefined as never);
    expect(useUiStore.getState().syncApprovalsOpen).toBe(true);
  });

  it("toasts notices with a way to Settings → Sync, except the upgrade (it has its own dialog)", async () => {
    const { emit } = await subscribed();
    emit("sync://notice", { kind: "upgraded", kept_file: null, kept_hosts: [], moved_files: [] });
```

- [ ] **Step 3: 跑測試確認失敗**

Run: `pnpm exec vitest run --dir src src/lib/sync-approvals.test.ts src/lib/sync-events.test.ts`
Expected: FAIL —— `Cannot find module './sync-approvals'`;`sync-events.test.ts` 的 `subscribeSyncEvents` 測試等不到第 5 個
listener(`expected 4 to be 5`)。

- [ ] **Step 4: 實作 `src/lib/sync-approvals.ts`**

新增 `src/lib/sync-approvals.ts`:

```ts
import type { ApprovalNotice } from "@/bindings/ApprovalNotice";
import type { GatedDirective } from "@/bindings/GatedDirective";
import type { PendingApprovalView } from "@/bindings/PendingApprovalView";
import type { ReviewOutcome } from "@/bindings/ReviewOutcome";
import type { ReviewedVersion } from "@/bindings/ReviewedVersion";
import type { SyncMessage } from "@/lib/sync-events";

/**
 * Settings a synced host may only bring in after this computer approves them
 * (spec §7.4 and the B2 review): they run local programs, share this computer's
 * credentials, environment or network with the other side, run commands on the
 * server, turn off the host-key checks that keep a redirect safe, or open
 * forwarded ports to the local network. The backend's `approval::GATED_KEYWORDS`
 * decides; this copy (documented spelling, same order) only labels and
 * highlights. A test keeps them equal.
 */
export const GATED_KEYWORDS = [
  "ProxyCommand",
  "LocalCommand",
  "PermitLocalCommand",
  "KnownHostsCommand",
  "PKCS11Provider",
  "SecurityKeyProvider",
  "ForwardAgent",
  "ForwardX11",
  "ForwardX11Trusted",
  "RemoteForward",
  "SmartcardDevice",
  "XAuthLocation",
  "RemoteCommand",
  "StrictHostKeyChecking",
  "NoHostAuthenticationForProxyCommand",
  "VerifyHostKeyDNS",
  "UserKnownHostsFile",
  "GlobalKnownHostsFile",
  "SendEnv",
  "GSSAPIDelegateCredentials",
  "IdentityAgent",
  "PermitRemoteOpen",
  "NoHostAuthenticationForLocalhost",
  "GatewayPorts",
] as const;

/**
 * Gated unless written in the plain port-only form (`approval::gated_forward`):
 * a signature can hold them although they are not in `GATED_KEYWORDS`.
 */
const SOMETIMES_GATED = ["LocalForward", "DynamicForward"] as const;

const ALWAYS_GATED = new Set<string>(GATED_KEYWORDS.map((k) => k.toLowerCase()));
const LABELS = new Map<string, string>([...GATED_KEYWORDS, ...SOMETIMES_GATED].map((k) => [k.toLowerCase(), k]));

/** A lowercase keyword in its usual spelling (`proxycommand` → `ProxyCommand`). */
export function keywordLabel(keyword: string): string {
  return LABELS.get(keyword) ?? keyword;
}

/**
 * Text as ssh reads it, shown so the screen cannot lie about it: bidi controls
 * (which reorder text on screen), zero-width and other format characters,
 * controls other than tab, and non-ASCII spaces become a visible `⟨U+202E⟩`.
 * The backend refuses most of them in synced hosts, but a gated value can still
 * carry some in the part OpenSSH treats as a comment.
 */
export function revealHidden(text: string): string {
  return text.replace(/[\p{Cf}\p{Cc}\p{Zs}\p{Zl}\p{Zp}]/gu, (ch) =>
    ch === "\t" || ch === " " ? ch : `⟨U+${ch.codePointAt(0)!.toString(16).toUpperCase().padStart(4, "0")}⟩`,
  );
}

/** A block's lines for display: one trailing CR per line (CRLF files) dropped, hidden characters revealed. */
export function displayLines(text: string): string[] {
  return text
    .replace(/\r?\n$/, "")
    .split("\n")
    .map((line) => revealHidden(line.replace(/\r$/, "")));
}

/** The keyword of a config line, lowercased (`Keyword value` or `Keyword=value`); null for blanks and comments. */
export function lineKeyword(line: string): string | null {
  const match = /^\s*([A-Za-z][A-Za-z0-9]*)\s*(?:=|\s|$)/.exec(line);
  return match ? match[1].toLowerCase() : null;
}

export interface BlockLine {
  text: string;
  /** A gated setting: highlighted in the review. */
  gated: boolean;
  /** The Host line of a host this computer has, now applying to different names. */
  scope: boolean;
}

/** The part of a directive line after its keyword (and `=`), trimmed: what a signature holds as `value`. */
function restOfLine(line: string): string {
  return line.replace(/^\s*[A-Za-z][A-Za-z0-9]*\s*=?/, "").trim();
}

/**
 * The incoming block, line by line, for the review dialog (display only — the
 * backend's parser decides). A line is marked when its keyword is always gated,
 * or when the incoming signature holds it (a forward with a bind address).
 */
export function blockLines(view: PendingApprovalView): BlockLine[] {
  const scopeChanged = view.current_text !== null && view.applied.host !== view.incoming.host;
  return view.text
    .replace(/\r?\n$/, "")
    .split("\n")
    .map((raw) => {
      const line = raw.replace(/\r$/, "");
      const keyword = lineKeyword(line);
      const signed = keyword !== null && view.incoming.gated.some((g) => g.keyword === keyword && g.value === restOfLine(line));
      return {
        text: revealHidden(line),
        gated: keyword !== null && (ALWAYS_GATED.has(keyword) || signed),
        scope: scopeChanged && keyword === "host",
      };
    });
}

export type ApprovalChange =
  | { kind: "new_host"; host: string }
  | { kind: "scope"; from: string; to: string }
  | { kind: "added"; keyword: string; value: string }
  | { kind: "removed"; keyword: string; value: string }
  | { kind: "changed"; keyword: string; from: string; to: string }
  | { kind: "moved"; keyword: string; value: string };

const sameDirective = (a: GatedDirective, b: GatedDirective) => a.keyword === b.keyword && a.value === b.value;

/**
 * How the gated settings differ, in order (the approval signature compares them
 * in order: the first value of a keyword is the one ssh uses). An LCS diff keeps
 * the untouched ones out; within each run of edits a removal and an addition of
 * the same keyword read as one change, and the same setting removed in one place
 * and added in another reads as a reorder.
 */
function gatedChanges(applied: GatedDirective[], incoming: GatedDirective[]): ApprovalChange[] {
  const n = applied.length;
  const m = incoming.length;
  const lcs = Array.from({ length: n + 1 }, () => new Array<number>(m + 1).fill(0));
  for (let i = n - 1; i >= 0; i--) {
    for (let j = m - 1; j >= 0; j--) {
      lcs[i][j] = sameDirective(applied[i], incoming[j]) ? lcs[i + 1][j + 1] + 1 : Math.max(lcs[i + 1][j], lcs[i][j + 1]);
    }
  }

  const out: ApprovalChange[] = [];
  let removed: GatedDirective[] = [];
  let added: GatedDirective[] = [];
  const flush = () => {
    for (const add of added) {
      const k = removed.findIndex((r) => r.keyword === add.keyword);
      if (k >= 0) {
        out.push({ kind: "changed", keyword: add.keyword, from: removed[k].value, to: add.value });
        removed.splice(k, 1);
      } else {
        out.push({ kind: "added", ...add });
      }
    }
    for (const r of removed) out.push({ kind: "removed", ...r });
    removed = [];
    added = [];
  };

  let i = 0;
  let j = 0;
  while (i < n || j < m) {
    if (i < n && j < m && sameDirective(applied[i], incoming[j])) {
      flush();
      i += 1;
      j += 1;
    } else if (j < m && (i === n || lcs[i][j + 1] >= lcs[i + 1][j])) {
      added.push(incoming[j]);
      j += 1;
    } else {
      removed.push(applied[i]);
      i += 1;
    }
  }
  flush();

  // The same setting removed in one run and added in another moved.
  const result: ApprovalChange[] = [];
  const merged = new Set<number>();
  out.forEach((change, index) => {
    if (merged.has(index)) return;
    if (change.kind === "added" || change.kind === "removed") {
      const twin = out.findIndex(
        (other, k) =>
          k > index &&
          !merged.has(k) &&
          (other.kind === "added" || other.kind === "removed") &&
          other.kind !== change.kind &&
          other.keyword === change.keyword &&
          other.value === change.value,
      );
      if (twin >= 0) {
        merged.add(twin);
        result.push({ kind: "moved", keyword: change.keyword, value: change.value });
        return;
      }
    }
    result.push(change);
  });
  return result;
}

/** What approving would change on this computer (spec §7.4 signature differences). */
export function approvalChanges(view: PendingApprovalView): ApprovalChange[] {
  const head: ApprovalChange[] =
    view.current_text === null
      ? [{ kind: "new_host", host: view.incoming.host }]
      : view.applied.host !== view.incoming.host
        ? [{ kind: "scope", from: view.applied.host, to: view.incoming.host }]
        : [];
  return [...head, ...gatedChanges(view.applied.gated, view.incoming.gated)];
}

/** One change for the review, with hidden characters revealed (`revealHidden`). */
export function changeText(change: ApprovalChange): string {
  const v = revealHidden;
  switch (change.kind) {
    case "new_host":
      return `New host on this computer: Host ${v(change.host)}`;
    case "scope":
      return `Applies to: ${v(change.from)} → ${v(change.to)}`;
    case "added":
      return `Adds ${keywordLabel(change.keyword)} ${v(change.value)}`;
    case "removed":
      return `Removes ${keywordLabel(change.keyword)} ${v(change.value)}`;
    case "changed":
      return `${keywordLabel(change.keyword)}: ${v(change.from)} → ${v(change.to)}`;
    case "moved":
      return `Order changed: ${keywordLabel(change.keyword)} ${v(change.value)}`;
  }
}

export interface ApprovalGroup {
  spaceId: string;
  spaceName: string;
  views: PendingApprovalView[];
}

/** Pending hosts per space, in the order the backend listed them (`sync_approve` takes one space at a time). */
export function approvalGroups(views: readonly PendingApprovalView[]): ApprovalGroup[] {
  const groups = new Map<string, ApprovalGroup>();
  for (const view of views) {
    const group = groups.get(view.space_id);
    if (group) group.views.push(view);
    else groups.set(view.space_id, { spaceId: view.space_id, spaceName: view.space_name, views: [view] });
  }
  return [...groups.values()];
}

/** What a decision sends back for the versions the review showed (`sync_approve` / `sync_reject`). */
export function reviewedVersions(views: readonly PendingApprovalView[]): ReviewedVersion[] {
  return views.map((v) => ({ alias: v.alias, digest: v.digest }));
}

/**
 * Whether two lists hold the same versions, in any order. A version is its space,
 * alias and content digest: new content for a host comes with a new digest, while
 * the same version pulled again later keeps its digest.
 */
export function sameVersions(a: readonly PendingApprovalView[], b: readonly PendingApprovalView[]): boolean {
  const key = (v: PendingApprovalView) => JSON.stringify([v.space_id, v.alias, v.digest]);
  const shown = new Set(a.map(key));
  return a.length === b.length && shown.size === a.length && b.every((v) => shown.has(key(v)));
}

/** The outcomes of one decision (one call per space) as one. */
export function combineOutcomes(outcomes: readonly ReviewOutcome[]): { applied: number; changed: string[] } {
  return { applied: outcomes.reduce((n, o) => n + o.applied, 0), changed: outcomes.flatMap((o) => o.changed) };
}

/** What the review says when hosts changed while it was open (their newer version waits); null when none did. */
export function changedNotice(changed: readonly string[]): string | null {
  if (changed.length === 0) return null;
  return `${changed.join(", ")} changed since you opened this — review ${changed.length === 1 ? "it" : "them"} again.`;
}

/** `sync://approval`: hosts newly held back this round. */
export function approvalMessage(notices: readonly ApprovalNotice[]): SyncMessage | null {
  const aliases = notices.flatMap((n) => n.aliases);
  if (aliases.length === 0) return null;
  return {
    title: aliases.length === 1 ? `${aliases[0]} needs your approval` : `${aliases.length} synced hosts need your approval`,
    description:
      "They came from another computer with settings that run programs, share your credentials, environment or network, or relax host-key checks. Nothing changes until you approve them.",
  };
}
```

- [ ] **Step 5: 修改 `src/stores/ui.ts`:審核對話框的開關**

`src/stores/ui.ts`:把

```ts
  /** Whether the "Move hosts into sync" wizard is open. Session-only. */
  syncMigrationOpen: boolean;
  setSyncMigrationOpen: (open: boolean) => void;
}

/**
```

換成:

```ts
  /** Whether the "Move hosts into sync" wizard is open. Session-only. */
  syncMigrationOpen: boolean;
  setSyncMigrationOpen: (open: boolean) => void;
  /** Whether the review of synced hosts waiting for approval is open (approval toast, Settings → Sync). Session-only. */
  syncApprovalsOpen: boolean;
  setSyncApprovalsOpen: (open: boolean) => void;
}

/**
```

`src/stores/ui.ts`:把

```ts
      setPaletteOpen: (paletteOpen) => set({ paletteOpen }),
      syncMigrationOpen: false,
      setSyncMigrationOpen: (syncMigrationOpen) => set({ syncMigrationOpen }),
    }),
    {
      name: UI_STORAGE_KEY,
```

換成:

```ts
      setPaletteOpen: (paletteOpen) => set({ paletteOpen }),
      syncMigrationOpen: false,
      setSyncMigrationOpen: (syncMigrationOpen) => set({ syncMigrationOpen }),
      syncApprovalsOpen: false,
      setSyncApprovalsOpen: (syncApprovalsOpen) => set({ syncApprovalsOpen }),
    }),
    {
      name: UI_STORAGE_KEY,
```

- [ ] **Step 6: 修改 `src/lib/sync-events.ts`:`sync://approval`**

`src/lib/sync-events.ts`:把

```ts
import { useQueryClient, type QueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { SyncConflict } from "@/bindings/SyncConflict";
import type { SyncNotice } from "@/bindings/SyncNotice";
import type { SyncOverview } from "@/bindings/SyncOverview";
import { syncApprovalsKey, syncOverviewKey } from "@/lib/sync";
import { useUiStore } from "@/stores/ui";

export interface SyncMessage {
```

換成:

```ts
import { useQueryClient, type QueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { ApprovalNotice } from "@/bindings/ApprovalNotice";
import type { SyncConflict } from "@/bindings/SyncConflict";
import type { SyncNotice } from "@/bindings/SyncNotice";
import type { SyncOverview } from "@/bindings/SyncOverview";
import { syncApprovalsKey, syncOverviewKey } from "@/lib/sync";
import { approvalMessage } from "@/lib/sync-approvals";
import { useUiStore } from "@/stores/ui";

export interface SyncMessage {
```

`src/lib/sync-events.ts`:把

```ts
/**
 * Sync engine → UI. Status pushes refresh every overview reader without polling;
 * applied remote changes refresh the config views (a newly synced host can shadow
 * a local one) and the approval list; conflicts and notices surface as toasts.
 * The backend starts a round itself when the window regains focus, so nothing
 * here asks for one — a second round would double the relay usage.
 * Returns the unsubscribe function.
```

換成:

```ts
/**
 * Sync engine → UI. Status pushes refresh every overview reader without polling;
 * applied remote changes refresh the config views (a newly synced host can shadow
 * a local one) and the approval list; conflicts, hosts held for approval and
 * notices surface as toasts.
 * The backend starts a round itself when the window regains focus, so nothing
 * here asks for one — a second round would double the relay usage.
 * Returns the unsubscribe function.
```

`src/lib/sync-events.ts`:把

```ts
    if (message) toast.warning(message.title, { description: message.description });
    void queryClient.invalidateQueries({ queryKey: ["config"] });
  });
  on<SyncNotice>("sync://notice", (notice) => {
    if (notice.kind === "upgraded") return; // SyncUpgradeDialog explains it
    const message = noticeMessage(notice);
```

換成:

```ts
    if (message) toast.warning(message.title, { description: message.description });
    void queryClient.invalidateQueries({ queryKey: ["config"] });
  });
  on<ApprovalNotice[]>("sync://approval", (notices) => {
    void queryClient.invalidateQueries({ queryKey: syncApprovalsKey });
    const message = approvalMessage(notices);
    if (!message) return;
    toast.warning(message.title, {
      description: message.description,
      duration: 15_000,
      action: { label: "Review", onClick: () => useUiStore.getState().setSyncApprovalsOpen(true) },
    });
  });
  on<SyncNotice>("sync://notice", (notice) => {
    if (notice.kind === "upgraded") return; // SyncUpgradeDialog explains it
    const message = noticeMessage(notice);
```

- [ ] **Step 7: 新增 `src/components/SyncApprovalDialog.tsx`**

新增 `src/components/SyncApprovalDialog.tsx`:

```tsx
import { useEffect, useState } from "react";
import { Loader2, ShieldAlert } from "lucide-react";
import { toast } from "sonner";

import type { PendingApprovalView } from "@/bindings/PendingApprovalView";
import type { ReviewOutcome } from "@/bindings/ReviewOutcome";
import { errorMessage, useApproveHosts, usePendingApprovals, useRejectHosts } from "@/lib/sync";
import {
  approvalChanges,
  approvalGroups,
  blockLines,
  changeText,
  changedNotice,
  combineOutcomes,
  displayLines,
  revealHidden,
  reviewedVersions,
  sameVersions,
} from "@/lib/sync-approvals";
import { relativeTime } from "@/lib/format";
import { plural } from "@/lib/sync-overview";
import { useUiStore } from "@/stores/ui";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/**
 * Shown in logical order, left to right, whatever the characters are: right-to-left
 * text cannot make a line look different from what ssh reads. Text from other
 * computers also goes through `revealHidden`, which makes bidi and zero-width
 * characters visible.
 */
const LOGICAL_ORDER = "[direction:ltr] [unicode-bidi:bidi-override]";

/**
 * Review of synced hosts held back because they bring settings that run
 * programs, share credentials or relax host-key checks (spec §7.4,
 * `GATED_KEYWORDS`). Each host shows the full incoming
 * block with the gated lines marked and what approving changes; approve or reject
 * one host, or everything at once — always exactly the versions on screen.
 * Opened from the approval toast and from Settings → Sync.
 */
export function SyncApprovalDialog() {
  const open = useUiStore((s) => s.syncApprovalsOpen);
  const setOpen = useUiStore((s) => s.setSyncApprovalsOpen);
  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogContent className="sm:max-w-2xl">{open && <ApprovalReview onClose={() => setOpen(false)} />}</DialogContent>
    </Dialog>
  );
}

function ApprovalReview({ onClose }: { onClose: () => void }) {
  const pending = usePendingApprovals(true);
  const approve = useApproveHosts();
  const reject = useRejectHosts();
  // The versions on screen. They stay put until the user decides or asks for the
  // newest ones, so a version that arrives meanwhile never takes the place of the
  // one being approved; the backend checks each digest too and reports what changed.
  const [shown, setShown] = useState<PendingApprovalView[] | null>(null);
  const [notice, setNotice] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);

  useEffect(() => {
    if (shown === null && pending.data) setShown(pending.data);
  }, [shown, pending.data]);

  const views = shown ?? [];
  const groups = approvalGroups(views);
  const newer = shown !== null && pending.data !== undefined && !sameVersions(shown, pending.data);

  /**
   * One call per space with the versions on screen, then the newest list. Hosts
   * that changed while the review was open are left alone: say so, show their
   * newer version. Stops at the first failure (the mutation toasts it).
   */
  const decide = async (decision: "approve" | "reject", batches: { spaceId: string; views: PendingApprovalView[] }[], all: boolean) => {
    setBusy(true);
    setNotice(null);
    const outcomes: ReviewOutcome[] = [];
    try {
      for (const b of batches) {
        const args = { spaceId: b.spaceId, approvals: reviewedVersions(b.views) };
        outcomes.push(await (decision === "approve" ? approve.mutateAsync(args) : reject.mutateAsync(args)));
      }
    } catch {
      // Already shown by the mutation; what went through stays done.
    }
    const fresh = await pending.refetch();
    if (fresh.data) setShown(fresh.data);
    const { applied, changed } = combineOutcomes(outcomes);
    setNotice(changedNotice(changed));
    if (all && applied > 0 && changed.length === 0) {
      toast.success(decision === "approve" ? `Applied ${plural(applied, "host")}` : `Kept this computer's version of ${plural(applied, "host")}`);
    }
    setBusy(false);
  };

  return (
    <>
      <DialogHeader>
        <DialogTitle>Review synced hosts</DialogTitle>
        <DialogDescription>
          These hosts came from your other computers with settings that run programs (on this computer or on the server), share your credentials,
          environment or network, or relax host-key checks. Nothing is applied until you approve it. Rejecting keeps this computer's version and
          sends nothing back.
        </DialogDescription>
      </DialogHeader>

      {notice && (
        <p className="rounded-md border border-amber-500/40 bg-amber-500/10 p-2 text-xs text-amber-800 dark:text-amber-300">{notice}</p>
      )}
      {newer && !busy && (
        <div className="flex items-center justify-between gap-2 rounded-md border p-2 text-xs">
          <span>Newer versions arrived while this was open.</span>
          <Button
            type="button"
            variant="outline"
            size="sm"
            className="h-7"
            onClick={() => {
              setNotice(null);
              setShown(pending.data ?? []);
            }}
          >
            Show them
          </Button>
        </div>
      )}

      {pending.isError ? (
        <p className="text-sm text-destructive">Could not load the hosts: {errorMessage(pending.error)}</p>
      ) : shown === null ? (
        <p className="text-sm text-muted-foreground">Loading…</p>
      ) : views.length === 0 ? (
        <p className="text-sm text-muted-foreground">Nothing is waiting for your approval.</p>
      ) : (
        <div className="max-h-[55vh] space-y-4 overflow-y-auto pr-1">
          {groups.map((g) => (
            <section key={g.spaceId} className="space-y-2">
              <p className="text-xs font-semibold tracking-wide text-muted-foreground uppercase">{revealHidden(g.spaceName)}</p>
              {g.views.map((view) => (
                <PendingHost
                  key={view.alias}
                  view={view}
                  busy={busy}
                  onApprove={() => void decide("approve", [{ spaceId: view.space_id, views: [view] }], false)}
                  onReject={() => void decide("reject", [{ spaceId: view.space_id, views: [view] }], false)}
                />
              ))}
            </section>
          ))}
        </div>
      )}

      <DialogFooter>
        <Button type="button" variant="outline" onClick={onClose}>
          Close
        </Button>
        {views.length > 1 && (
          <>
            <Button type="button" variant="outline" disabled={busy} onClick={() => void decide("reject", groups, true)}>
              Reject all
            </Button>
            <Button type="button" disabled={busy} onClick={() => void decide("approve", groups, true)}>
              {busy && <Loader2 className="size-4 animate-spin" />} Approve all ({views.length})
            </Button>
          </>
        )}
      </DialogFooter>
    </>
  );
}

function PendingHost({ view, busy, onApprove, onReject }: { view: PendingApprovalView; busy: boolean; onApprove: () => void; onReject: () => void }) {
  return (
    <div className="space-y-2 rounded-md border p-3">
      <div className="flex items-start justify-between gap-3">
        <div className="min-w-0">
          <p className="flex items-center gap-1.5 font-mono text-sm">
            <ShieldAlert className="size-3.5 shrink-0 text-amber-600 dark:text-amber-400" aria-hidden />
            <span className={LOGICAL_ORDER}>{revealHidden(view.alias)}</span>
          </p>
          <p className={cn("text-xs text-muted-foreground", LOGICAL_ORDER)}>
            From {revealHidden(view.from_device)} · {relativeTime(view.updated_at_ms)}
          </p>
        </div>
        <div className="flex shrink-0 gap-1.5">
          <Button type="button" variant="outline" size="sm" className="h-7" disabled={busy} onClick={onReject}>
            Reject
          </Button>
          <Button type="button" size="sm" className="h-7" disabled={busy} onClick={onApprove}>
            Approve
          </Button>
        </div>
      </div>
      <ul className="list-disc space-y-0.5 pl-5 text-xs">
        {approvalChanges(view).map((change, i) => (
          <li key={i} className={cn("break-all font-mono", LOGICAL_ORDER)}>
            {changeText(change)}
          </li>
        ))}
      </ul>
      <pre className="overflow-x-auto rounded-md bg-muted/40 p-2 font-mono text-xs leading-5" aria-label={`Incoming block for ${view.alias}`}>
        {blockLines(view).map((line, i) => (
          <div key={i} className={cn(LOGICAL_ORDER, (line.gated || line.scope) && "-mx-1 rounded-sm bg-amber-500/15 px-1 text-amber-800 dark:text-amber-300")}>
            {line.text || " "}
          </div>
        ))}
      </pre>
      {view.current_text !== null && (
        <details className="text-xs">
          <summary className="cursor-default text-muted-foreground select-none">This computer's current version</summary>
          <pre className="mt-1 overflow-x-auto rounded-md bg-muted/40 p-2 font-mono leading-5">
            {displayLines(view.current_text).map((line, i) => (
              <div key={i} className={LOGICAL_ORDER}>
                {line || " "}
              </div>
            ))}
          </pre>
        </details>
      )}
    </div>
  );
}
```

- [ ] **Step 8: 修改 `src/App.tsx` 與 `src/components/SyncPane.tsx`**

`src/App.tsx`:把

```tsx
import { NewConfigFileDialog } from "@/components/NewConfigFileDialog";
import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";
import { SyncUpgradeDialog } from "@/components/SyncUpgradeDialog";
import { SettingsDialog } from "@/components/SettingsDialog";
import { CommandPalette } from "@/components/CommandPalette";
import { DriftBanner } from "@/components/DriftBanner";
```

換成:

```tsx
import { NewConfigFileDialog } from "@/components/NewConfigFileDialog";
import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";
import { SyncUpgradeDialog } from "@/components/SyncUpgradeDialog";
import { SyncApprovalDialog } from "@/components/SyncApprovalDialog";
import { SettingsDialog } from "@/components/SettingsDialog";
import { CommandPalette } from "@/components/CommandPalette";
import { DriftBanner } from "@/components/DriftBanner";
```

`src/App.tsx`:把

```tsx
  useGlobalHotkey();
  // In-app ⌘F (focus host search) and ⌘N (new host).
  useAppShortcuts();
  // Sync engine → UI (status, applied changes, conflicts, notices). The backend
  // syncs on window focus by itself and holds that while the relay asks it to back off; a
  // `sync_now` here would skip the backoff, so only the "Sync now" button calls it.
  useSyncEvents();
```

換成:

```tsx
  useGlobalHotkey();
  // In-app ⌘F (focus host search) and ⌘N (new host).
  useAppShortcuts();
  // Sync engine → UI (status, applied changes, conflicts, approvals, notices). The backend
  // syncs on window focus by itself and holds that while the relay asks it to back off; a
  // `sync_now` here would skip the backoff, so only the "Sync now" button calls it.
  useSyncEvents();
```

`src/App.tsx`:把

```tsx
        <NewConfigFileDialog />
        <SyncMigrationDialog />
        <SyncUpgradeDialog />
        <McpApprovalDialog />
        <Toaster />
      </div>
```

換成:

```tsx
        <NewConfigFileDialog />
        <SyncMigrationDialog />
        <SyncUpgradeDialog />
        <SyncApprovalDialog />
        <McpApprovalDialog />
        <Toaster />
      </div>
```

`src/components/SyncPane.tsx`:把

```tsx
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { ExternalLink, Eye, KeyRound, Loader2, RefreshCw, UserMinus } from "lucide-react";
import { toast } from "sonner";

import type { SyncFrozenView } from "@/bindings/SyncFrozenView";
```

換成:

```tsx
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { ExternalLink, Eye, KeyRound, Loader2, RefreshCw, ShieldAlert, UserMinus } from "lucide-react";
import { toast } from "sonner";

import type { SyncFrozenView } from "@/bindings/SyncFrozenView";
```

`src/components/SyncPane.tsx`:把

```tsx
  frozenMessage,
  leaveRotationNote,
  noticeRows,
  relayDetails,
  rotationLabel,
  statusLine,
```

換成:

```tsx
  frozenMessage,
  leaveRotationNote,
  noticeRows,
  plural,
  relayDetails,
  rotationLabel,
  statusLine,
```

`src/components/SyncPane.tsx`:把

```tsx
}) {
  const syncNow = useSyncNow();
  const dismiss = useDismissNotice();
  const status = statusLine(o, Date.now());
  const [reading, setReading] = useState<number | "code" | null>(null);

```

換成:

```tsx
}) {
  const syncNow = useSyncNow();
  const dismiss = useDismissNotice();
  const openApprovals = useUiStore((s) => s.setSyncApprovalsOpen);
  const status = statusLine(o, Date.now());
  const [reading, setReading] = useState<number | "code" | null>(null);

```

`src/components/SyncPane.tsx`:把

```tsx
          </SettingsRow>
          {o.frozen && <RejoinRow frozen={o.frozen} />}
          {o.rotation && <RotationRow rotation={o.rotation} />}
          {noticeRows(o).map((n) => (
            <SettingsRow key={`${n.index}-${n.title}`} label={n.title} description={n.description}>
              <div className="flex items-center gap-1.5">
```

換成:

```tsx
          </SettingsRow>
          {o.frozen && <RejoinRow frozen={o.frozen} />}
          {o.rotation && <RotationRow rotation={o.rotation} />}
          {o.approvals_waiting > 0 && (
            <SettingsRow
              label={`${plural(o.approvals_waiting, "host")} waiting for your approval`}
              description="They bring settings that run programs, share your credentials, environment or network, or relax host-key checks. Nothing changes until you approve them."
            >
              <Button type="button" size="sm" className="h-7" onClick={() => openApprovals(true)}>
                <ShieldAlert className="size-3.5" /> Review…
              </Button>
            </SettingsRow>
          )}
          {noticeRows(o).map((n) => (
            <SettingsRow key={`${n.index}-${n.title}`} label={n.title} description={n.description}>
              <div className="flex items-center gap-1.5">
```

- [ ] **Step 9: 跑測試確認通過**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  19 passed (19)`、`Tests  235 passed (235)`。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出。

- [ ] **Step 10: Commit**

```bash
git add src/lib/sync-approvals.ts src/lib/sync-approvals.test.ts src/lib/sync-events.ts src/lib/sync-events.test.ts
git add src/stores/ui.ts src/components/SyncApprovalDialog.tsx src/components/SyncPane.tsx src/App.tsx
git commit -m "feat(sync): review synced hosts that wait for approval"
```

---

### Task 5: 側邊欄:每個 space 一組、被遮蔽的同名主機

> **已執行**(repo `1b618cd`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準 —— `HostList.tsx` 與
> `src/lib/sync-sidebar.ts` 在最終修正之後和下面的區塊差很多,另外多了 `src/lib/sync-labels.ts`、`src/components/DuplicateCopies.tsx`、
> `src/components/ShadowFixDialog.tsx` 與 `HostList.test.tsx`(server render)。審查(與計畫逐字相同,遮蔽的判斷與後端的 Rust
> 測試結果一致)的 Important 不在這個 task 的檔案:app 內新增、存檔、刪除、改名、搬移之後,同名主機的標示不會更新(只少標、
> 不誤標);它和其他 Minor 一起在最終修正改(`bae0baf`:同名主機清單的 key 移到 hosts 的 key 底下,解讀 12;雙擊 space 標題
> 不再收合;「Manage spaces…」用 `openSyncSettings`;Add host 與編輯器的 Move to file 也以 space 名稱列出 space 檔;測試分得出
> Include 順序與載入順序)。最終修正另外:琥珀色記號的 tooltip 與「Remove this copy (the one in <file> stays)」照 ssh 實際的合併
> 方式說(`f55b0eb`);space 有問題時群組標題有警示記號(`b3ae59f`);有歧義的名稱以 (alias, 檔案) 選取,靠名稱找主機的動作都
> 停用,編輯器的位置改成唯讀的各份清單(`c8cc4a8`);兩個修正先確認(`3345e24`);殘留修正讓各份清單只指向真的有的動作、
> 只在知道的地方宣稱讀取順序、同步總覽未知時一律當成有歧義、⌘/Shift 點擊不清掉已勾選的列(`1d6994a`、`1c2fbb4`)。詳見
> 解讀 3、4。執行時 `src/` 的測試 285 → 289(+4,同計畫)。

側邊欄本來就依檔案分組,勾選的 space 各是一個檔案,所以每個 space 自然是一組;這個 task 讓它們以 space 名稱為標籤、
加雲朵圖示、改右鍵選單、停用側邊欄改名(解讀 3),並在 ssh 不會讀的那份同名主機列尾標示、提供以檔案定位的處理
(解讀 4)。「Move to file」與檔案篩選也因此顯示 space 名稱;拖進 space 群組沿用 `config_move_host`(後端套搬進 space
的規則,spec §7.2)。

**Files:**
- Create: `src/lib/sync-sidebar.test.ts`、`src/lib/sync-sidebar.ts`
- Modify: `src/components/HostList.tsx`

**Interfaces:**
- Consumes(Task 1):`useSyncOverview`、`useDuplicateAliases`、`useResolveShadowed`;bindings `SyncSpaceView`、`DuplicateAlias`、`HostSummary`。
- Produces(`src/lib/sync-sidebar.ts`):`spaceFileLabels(spaces): Map<string, string>`(檔案路徑 → space 名稱,依 Include 順序)、
  `shadowKey(file, alias): string`、`shadowedCopies(duplicates, hosts, spaceFiles): Map<string, string>`(shadowKey → 勝出的 space 檔)


- [ ] **Step 1: 寫失敗的測試:`src/lib/sync-sidebar.test.ts`**

新增 `src/lib/sync-sidebar.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import type { HostSummary } from "@/bindings/HostSummary";
import { space } from "./sync-fixtures";
import { shadowKey, shadowedCopies, spaceFileLabels } from "./sync-sidebar";

const MAIN = "/home/f/.ssh/config";
const PERSONAL = "/home/f/.ssh/sshelter/personal-3fa2c1d9.config";
const WORK = "/home/f/.ssh/sshelter/work-8b01e4aa.config";

function host(alias: string, file: string, patterns: string[] = [alias]): HostSummary {
  return { alias, patterns, source_file: file, tags: [], hostname: null, user: null };
}

describe("spaceFileLabels", () => {
  it("labels each synced space's file with the space name, in Include order", () => {
    const labels = spaceFileLabels([
      space({ file_path: PERSONAL }),
      space({ id: "c".repeat(64), name: "Old", selected: false, file_name: null, file_path: null, hosts: null }),
      space({ id: "b".repeat(64), name: "Work", file_path: WORK }),
    ]);
    expect([...labels.entries()]).toEqual([
      [PERSONAL, "Personal"],
      [WORK, "Work"],
    ]);
  });
});

describe("shadowedCopies", () => {
  const hosts = [host("web", PERSONAL), host("web", WORK), host("web", MAIN), host("db", MAIN)];

  it("maps each shadowed copy to the space file ssh reads instead (the first in Include order)", () => {
    const copies = shadowedCopies(
      [
        { alias: "web", local_file: MAIN },
        { alias: "web", local_file: WORK },
      ],
      hosts,
      [PERSONAL, WORK],
    );
    expect(copies).toEqual(
      new Map([
        [shadowKey(MAIN, "web"), PERSONAL],
        [shadowKey(WORK, "web"), PERSONAL],
      ]),
    );
  });

  it("skips a copy whose winner is not in the loaded hosts yet", () => {
    expect(shadowedCopies([{ alias: "db", local_file: MAIN }], hosts, [PERSONAL, WORK]).size).toBe(0);
  });

  it("keys by file and alias, so the same alias in two files never collides", () => {
    expect(shadowKey(MAIN, "web")).not.toBe(shadowKey(WORK, "web"));
    expect(shadowKey("/a b", "c")).not.toBe(shadowKey("/a", "b c"));
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `pnpm exec vitest run --dir src src/lib/sync-sidebar.test.ts`
Expected: FAIL —— `Cannot find module './sync-sidebar'`。

- [ ] **Step 3: 實作 `src/lib/sync-sidebar.ts`**

新增 `src/lib/sync-sidebar.ts`:

```ts
import type { DuplicateAlias } from "@/bindings/DuplicateAlias";
import type { HostSummary } from "@/bindings/HostSummary";
import type { SyncSpaceView } from "@/bindings/SyncSpaceView";

/**
 * The files of the spaces this computer syncs, each labeled with its space's
 * name (spec §8: the sidebar shows one group per synced space). Insertion order
 * is the backend's, which is the order of the Include line.
 */
export function spaceFileLabels(spaces: readonly SyncSpaceView[]): Map<string, string> {
  return new Map(spaces.flatMap((s) => (s.selected && s.file_path ? [[s.file_path, s.name] as [string, string]] : [])));
}

/** Identifies one copy of an alias: the same alias can sit in several files. */
export function shadowKey(file: string, alias: string): string {
  return JSON.stringify([file, alias]);
}

/**
 * The copies ssh never reads (spec §4.3: ssh uses the copy in the space file
 * that comes first in the Include line), keyed by `shadowKey`, each mapped to
 * the space file whose copy wins — the row marker names it. `spaceFiles` is in
 * Include order (`spaceFileLabels(...).keys()`).
 */
export function shadowedCopies(
  duplicates: readonly DuplicateAlias[],
  hosts: readonly HostSummary[],
  spaceFiles: readonly string[],
): Map<string, string> {
  const out = new Map<string, string>();
  for (const d of duplicates) {
    const winner = spaceFiles.find((file) => file !== d.local_file && hosts.some((h) => h.source_file === file && h.alias === d.alias));
    if (winner) out.set(shadowKey(d.local_file, d.alias), winner);
  }
  return out;
}
```

- [ ] **Step 4: 修改 `src/components/HostList.tsx`**

`src/components/HostList.tsx`:把

```tsx
  X,
  FilePlus2,
  Copy,
} from "lucide-react";

import type { HostSummary } from "@/bindings/HostSummary";
```

換成:

```tsx
  X,
  FilePlus2,
  Copy,
  Cloud,
  TriangleAlert,
  Settings2,
} from "lucide-react";

import type { HostSummary } from "@/bindings/HostSummary";
```

`src/components/HostList.tsx`:把

```tsx
import { toast } from "sonner";
import { buildNewOrder } from "@/lib/reorder";
import { SEARCH_INPUT_ID } from "@/lib/app-shortcuts";

/** Sentinel Select value for the "All files" scope (Radix items can't be empty). */
const ALL_FILES = "__all__";
```

換成:

```tsx
import { toast } from "sonner";
import { buildNewOrder } from "@/lib/reorder";
import { SEARCH_INPUT_ID } from "@/lib/app-shortcuts";
import { useDuplicateAliases, useResolveShadowed, useSyncOverview } from "@/lib/sync";
import { shadowKey, shadowedCopies, spaceFileLabels } from "@/lib/sync-sidebar";

/** Sentinel Select value for the "All files" scope (Radix items can't be empty). */
const ALL_FILES = "__all__";
```

`src/components/HostList.tsx`:把

```tsx
  onMoveToNew?: () => void;
  /** Opens the shared remove-confirmation dialog (owned by HostList). */
  onRemove?: () => void;
  /** Row can be drag-reordered (within its source file). Off while searching. */
  draggable?: boolean;
  /** True while THIS row is the drag source — rendered semi-transparent. */
```

換成:

```tsx
  onMoveToNew?: () => void;
  /** Opens the shared remove-confirmation dialog (owned by HostList). */
  onRemove?: () => void;
  /**
   * This copy is shadowed: ssh reads the same alias from the space file labeled
   * `winner` (spec §4.3). Marked at the row's end, with file-addressed fixes.
   */
  shadow?: { winner: string; onKeepAsLocal: () => void; onRemoveCopy: () => void };
  /** Row can be drag-reordered (within its source file). Off while searching. */
  draggable?: boolean;
  /** True while THIS row is the drag source — rendered semi-transparent. */
```

`src/components/HostList.tsx`:把

```tsx
  onMoveTo,
  onMoveToNew,
  onRemove,
  draggable,
  dragging,
  indicator,
```

換成:

```tsx
  onMoveTo,
  onMoveToNew,
  onRemove,
  shadow,
  draggable,
  dragging,
  indicator,
```

`src/components/HostList.tsx`:把

```tsx
            {secondary}
          </span>
        )}
      </button>
      {/*
       * Connect affordance — overlays the row's right edge, hidden until the
```

換成:

```tsx
            {secondary}
          </span>
        )}
        {shadow && (
          <span
            className="shrink-0 text-amber-600 dark:text-amber-400"
            title={`ssh uses ${host.alias} from ${shadow.winner}, which comes first in the Include line`}
          >
            <TriangleAlert className="size-3" aria-label={`Shadowed by ${shadow.winner}`} />
          </span>
        )}
      </button>
      {/*
       * Connect affordance — overlays the row's right edge, hidden until the
```

`src/components/HostList.tsx`:把

```tsx
                  )}
                </DropdownMenuSubContent>
              </DropdownMenuSub>
            )}
            {onRemove && (
              <>
```

換成:

```tsx
                  )}
                </DropdownMenuSubContent>
              </DropdownMenuSub>
            )}
            {shadow && (
              <>
                <DropdownMenuSeparator />
                <DropdownMenuItem onSelect={shadow.onKeepAsLocal}>
                  <TriangleAlert className="size-3.5" />
                  Keep this copy as {host.alias}-local
                </DropdownMenuItem>
                <DropdownMenuItem onSelect={shadow.onRemoveCopy}>
                  <Trash2 className="size-3.5" />
                  Remove this copy (ssh uses {shadow.winner})
                </DropdownMenuItem>
              </>
            )}
            {onRemove && (
              <>
```

`src/components/HostList.tsx`:把

```tsx
            </ContextMenuSubContent>
          </ContextMenuSub>
        )}
        {onRemove && (
          <>
            <ContextMenuSeparator />
```

換成:

```tsx
            </ContextMenuSubContent>
          </ContextMenuSub>
        )}
        {shadow && (
          <>
            <ContextMenuSeparator />
            <ContextMenuItem onSelect={shadow.onKeepAsLocal}>
              <TriangleAlert className="size-3.5" />
              Keep this copy as {host.alias}-local
            </ContextMenuItem>
            <ContextMenuItem onSelect={shadow.onRemoveCopy}>
              <Trash2 className="size-3.5" />
              Remove this copy (ssh uses {shadow.winner})
            </ContextMenuItem>
          </>
        )}
        {onRemove && (
          <>
            <ContextMenuSeparator />
```

`src/components/HostList.tsx`:把

```tsx
  const setAddHostTargetFile = useUiStore((s) => s.setAddHostTargetFile);
  const setDeployKeyAlias = useUiStore((s) => s.setDeployKeyAlias);
  const setNewFileIntent = useUiStore((s) => s.setNewFileIntent);
  const terminalId = useSettingsStore((s) => s.terminalId);
  const hostTerminals = useSettingsStore((s) => s.hostTerminals);
  const newTabConnect = useSettingsStore((s) => s.newTabConnect);
```

換成:

```tsx
  const setAddHostTargetFile = useUiStore((s) => s.setAddHostTargetFile);
  const setDeployKeyAlias = useUiStore((s) => s.setDeployKeyAlias);
  const setNewFileIntent = useUiStore((s) => s.setNewFileIntent);
  const setSettingsOpen = useUiStore((s) => s.setSettingsOpen);
  const setSettingsCategory = useUiStore((s) => s.setSettingsCategory);
  const terminalId = useSettingsStore((s) => s.terminalId);
  const hostTerminals = useSettingsStore((s) => s.hostTerminals);
  const newTabConnect = useSettingsStore((s) => s.newTabConnect);
```

`src/components/HostList.tsx`:把

```tsx
  const moveHost = useMoveHost();
  const removeHost = useRemoveHost();
  const setTags = useSetTags();
  // Row-menu remove confirmation — one dialog shared by every row.
  const [removeTarget, setRemoveTarget] = useState<string | null>(null);

```

換成:

```tsx
  const moveHost = useMoveHost();
  const removeHost = useRemoveHost();
  const setTags = useSetTags();
  const resolveShadowed = useResolveShadowed();
  // Row-menu remove confirmation — one dialog shared by every row.
  const [removeTarget, setRemoveTarget] = useState<string | null>(null);

```

`src/components/HostList.tsx`:把

```tsx
  // Select even for files that currently have zero hosts.
  const { data } = useHostsQuery();
  const files = useMemo(() => data?.files ?? [], [data]);
  // Auto heuristic over the FULL file set (the "clear back to this" baseline)…
  const autoLabels = useMemo(() => shortLabels(files), [files]);
  // …overlaid with the user's per-file display aliases (an override wins).
  const labels = useMemo(() => labelsFor(files, fileAliases), [files, fileAliases]);
  // "Move to file" targets per SOURCE file (a host's own file never appears).
  // Keyed by source_file, not section — tag-mode sections aren't files.
  const moveTargetsByFile = useMemo(
```

換成:

```tsx
  // Select even for files that currently have zero hosts.
  const { data } = useHostsQuery();
  const files = useMemo(() => data?.files ?? [], [data]);
  // Synced spaces: each selected space's file is labeled with the space name
  // (spec §8), in Include order. The name comes from the sync account, so it wins
  // over a local display alias and is renamed in Settings → Sync, not inline.
  const overview = useSyncOverview();
  const spaceLabels = useMemo(() => spaceFileLabels(overview.data?.spaces ?? []), [overview.data]);
  // Copies of an alias that a space file shadows (ssh reads the space's copy).
  const duplicates = useDuplicateAliases(spaceLabels.size > 0);
  const shadows = useMemo(
    () => shadowedCopies(duplicates.data ?? [], hosts, [...spaceLabels.keys()]),
    [duplicates.data, hosts, spaceLabels],
  );
  // Auto heuristic over the FULL file set (the "clear back to this" baseline)…
  const autoLabels = useMemo(() => shortLabels(files), [files]);
  // …overlaid with the user's per-file display aliases (an override wins), and
  // the space names over both.
  const labels = useMemo(
    () => labelsFor(files, { ...fileAliases, ...Object.fromEntries(spaceLabels) }),
    [files, fileAliases, spaceLabels],
  );

  /** The shadow marker and fixes for one row, when ssh reads another copy of its alias. */
  const shadowFor = (host: HostSummary) => {
    const winner = shadows.get(shadowKey(host.source_file, host.alias));
    if (!winner) return undefined;
    const fix = (action: "rename" | "remove") =>
      resolveShadowed.mutate({ alias: host.alias, file: host.source_file, action });
    return {
      winner: labels.get(winner) ?? basename(winner),
      onKeepAsLocal: () => fix("rename"),
      onRemoveCopy: () => fix("remove"),
    };
  };
  // "Move to file" targets per SOURCE file (a host's own file never appears).
  // Keyed by source_file, not section — tag-mode sections aren't files.
  const moveTargetsByFile = useMemo(
```

`src/components/HostList.tsx`:把

```tsx
                !searchActive &&
                collapsedGroups.includes(section.file);
              const alias = fileAliases[section.file];
              const isEditing = editingFile === section.file;
              return (
                <div
```

換成:

```tsx
                !searchActive &&
                collapsedGroups.includes(section.file);
              const alias = fileAliases[section.file];
              // A synced space's group: named by the account, renamed in Settings → Sync.
              const spaceName = section.kind === "file" ? spaceLabels.get(section.file) : undefined;
              const isEditing = editingFile === section.file;
              return (
                <div
```

`src/components/HostList.tsx`:把

```tsx
                              onDoubleClick={(e) => {
                                e.preventDefault();
                                e.stopPropagation();
                                beginEdit(section.file);
                              }}
                              className="flex w-full items-center justify-between rounded-sm px-2 py-1.5 select-none hover:bg-muted/50 focus-visible:ring-2 focus-visible:ring-ring/50 focus-visible:outline-none cursor-default"
                              title={
                                alias
                                  ? `${alias} — ${section.file} (double-click to rename)`
                                  : `${section.file} (double-click to rename)`
                              }
                              aria-expanded={!isCollapsed}
                            >
```

換成:

```tsx
                              onDoubleClick={(e) => {
                                e.preventDefault();
                                e.stopPropagation();
                                if (!spaceName) beginEdit(section.file);
                              }}
                              className="flex w-full items-center justify-between rounded-sm px-2 py-1.5 select-none hover:bg-muted/50 focus-visible:ring-2 focus-visible:ring-ring/50 focus-visible:outline-none cursor-default"
                              title={
                                spaceName
                                  ? `Synced space “${spaceName}” — ${section.file} (rename it in Settings → Sync)`
                                  : alias
                                    ? `${alias} — ${section.file} (double-click to rename)`
                                    : `${section.file} (double-click to rename)`
                              }
                              aria-expanded={!isCollapsed}
                            >
```

`src/components/HostList.tsx`:把

```tsx
                                <span className="truncate text-[0.6875rem] font-semibold tracking-[0.08em] text-muted-foreground uppercase">
                                  {section.name}
                                </span>
                              </span>
                              <span className="font-mono text-[0.6875rem] text-muted-foreground/70 tabular-nums">
                                {section.hosts.length}
```

換成:

```tsx
                                <span className="truncate text-[0.6875rem] font-semibold tracking-[0.08em] text-muted-foreground uppercase">
                                  {section.name}
                                </span>
                                {spaceName && (
                                  <Cloud className="size-3 shrink-0 text-muted-foreground/70" aria-label="Synced space" />
                                )}
                              </span>
                              <span className="font-mono text-[0.6875rem] text-muted-foreground/70 tabular-nums">
                                {section.hosts.length}
```

`src/components/HostList.tsx`:把

```tsx
                            }}
                          >
                            <Plus />
                            New host in this file
                          </ContextMenuItem>
                          <ContextMenuSeparator />
                          <ContextMenuItem onSelect={() => setViewFile(section.file)}>
                            <FileText />
                            View file
                          </ContextMenuItem>
                          <ContextMenuItem onSelect={() => beginEdit(section.file)}>
                            <Pencil />
                            Rename label
                          </ContextMenuItem>
                        </ContextMenuContent>
                      </ContextMenu>
                    ))}
```

換成:

```tsx
                            }}
                          >
                            <Plus />
                            {spaceName ? "New host in this space" : "New host in this file"}
                          </ContextMenuItem>
                          <ContextMenuSeparator />
                          <ContextMenuItem onSelect={() => setViewFile(section.file)}>
                            <FileText />
                            View file
                          </ContextMenuItem>
                          {spaceName ? (
                            <ContextMenuItem
                              onSelect={() => {
                                setSettingsCategory("sync");
                                setSettingsOpen(true);
                              }}
                            >
                              <Settings2 />
                              Manage spaces…
                            </ContextMenuItem>
                          ) : (
                            <ContextMenuItem onSelect={() => beginEdit(section.file)}>
                              <Pencil />
                              Rename label
                            </ContextMenuItem>
                          )}
                        </ContextMenuContent>
                      </ContextMenu>
                    ))}
```

`src/components/HostList.tsx`:把

```tsx
                                setNewFileIntent({ kind: "move", aliases: [host.alias] })
                              }
                              onRemove={() => setRemoveTarget(host.alias)}
                              showTags={showHostTags && groupMode === "file"}
                              // Draggable when reordering OR a cross-file move
                              // is possible (single-host files can drag out).
```

換成:

```tsx
                                setNewFileIntent({ kind: "move", aliases: [host.alias] })
                              }
                              onRemove={() => setRemoveTarget(host.alias)}
                              shadow={shadowFor(host)}
                              showTags={showHostTags && groupMode === "file"}
                              // Draggable when reordering OR a cross-file move
                              // is possible (single-host files can drag out).
```

- [ ] **Step 5: 跑測試確認通過**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  20 passed (20)`、`Tests  239 passed (239)`。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出。

- [ ] **Step 6: Commit**

```bash
git add src/lib/sync-sidebar.ts src/lib/sync-sidebar.test.ts src/components/HostList.tsx
git commit -m "feat(sidebar): group hosts by synced space and mark shadowed copies"
```

---

### Task 6: 搬移精靈:目標 space、一個來源檔一個 space、不能搬的主機;移除 v1 型別

> **已執行**(repo `8b01d97`)。下面保留原本的步驟作為紀錄,不要再執行;實際的程式碼以 repo 為準。審查通過(新名稱以 30,000
> 個隨機案例、去重以 60,000 個隨機設定檔對照後端,沒有不一致);Minor 在最終修正一起改(`b24a690`,T6-a):查詢失敗時說明原因,
> 不會一直「Loading hosts…」;資料不見的 space 列成「(needs rebuild)」、不能選也不會預設選到(`isValidTarget`、`canMove`);
> 搬移進行中不能關;toast 把「搬了但標記寫不進去」算成已搬(`TAG_FAILED_PREFIX`,測試對照 `migrate.rs`),有任何問題就用
> warning,一台都沒搬時寫「No host moved …」(`moveSummary`);被拒絕與失敗的清單可以捲動,相同原因合併列出(`groupByReason`);
> 檔名 alias 含控制字元時不拿來當 space 名稱;補上審查找到的存活突變的測試。精靈裡的「Remove this copy」/「Keep this copy as
> …-local」也先確認(`ShadowFixDialog`),space 名稱經 `revealHidden`;另有 `SyncMigrationDialog.test.tsx`(server render)。執行時
> `src/` 的測試 289 → 299(+10,同計畫)。

精靈改成 spec §7.2 / §8 的版本:「Move into」選擇這台勾選的 space 或「One new space per file」(每組標頭顯示新 space 的
名稱,解讀 2)、列出不能搬的主機與原因、v1 的同名主機區塊推廣到所有 space;送出前去掉跨組重複的主機,結果區塊照
後端的文字列出問題(解讀 16)。ui store 的開關改成帶目標 space 的
`syncMigration`;Spaces 區塊加上「Move hosts here…」與「Move hosts into a space」,建立帳戶與加入後選完 space 都接著打開
精靈(解讀 1)。最後的 v1 helper 與 `SyncStatus.ts`、`SyncDevice.ts` 一起刪除。

**Files:**
- Modify(整個改寫):`src/lib/sync-migration.test.ts`、`src/lib/sync-migration.ts`、`src/components/SyncMigrationDialog.tsx`
- Modify: `src/stores/ui.ts`、`src/components/SyncPane.tsx`、`src/components/SyncSpacesSection.tsx`、`src/lib/sync.ts`
- Delete: `src/bindings/SyncStatus.ts`、`src/bindings/SyncDevice.ts`

**Interfaces:**
- Consumes(Task 1、3、5):`useSyncOverview`、`useMoveHostsToSpace`、`useMoveFilesToNewSpaces`、`useDuplicateAliases`、
  `useUnmovableHosts`、`useResolveShadowed`;`MAX_SPACE_NAME`;`spaceFileLabels`;`plural`;`labelsFor`、`basename`;測試用 `space`。
- Produces(`src/lib/sync-migration.ts`):`isSyncableHost`、`cleanWordsInput`、`keepVisible`(不變);`interface MigrationGroup
  { file; hosts }`、`groupHostsForMigration(hosts, spaceFiles: readonly string[], unmovable: ReadonlySet<string>)`;
  `PER_FILE = "__per_file__"`、`tagForFile(path)`、`uniqueSpaceName(base, taken)`、`plannedSpaceNames(groups, fileAliases,
  existing): Map<string, string>`、`newSpaceGroups(groups, selected, names): NewSpaceGroup[]`、
  `migrationTarget(spaces, requested): string`、`preselectAll(spaces): boolean`
- Produces(`src/stores/ui.ts`):`syncMigration: { spaceId: string | null } | null`、`setSyncMigration(value)`(取代
  `syncMigrationOpen` / `setSyncMigrationOpen`)


- [ ] **Step 1: 寫失敗的測試:改寫 `src/lib/sync-migration.test.ts`**

把 `src/lib/sync-migration.test.ts` 整個換成:

```ts
import { describe, expect, it } from "vitest";
import type { HostSummary } from "@/bindings/HostSummary";
import { space } from "./sync-fixtures";
import {
  PER_FILE,
  cleanWordsInput,
  groupHostsForMigration,
  isSyncableHost,
  keepVisible,
  migrationTarget,
  newSpaceGroups,
  plannedSpaceNames,
  preselectAll,
  tagForFile,
  uniqueSpaceName,
} from "./sync-migration";

function host(alias: string, file: string, patterns: string[] = [alias]): HostSummary {
  return { alias, patterns, source_file: file, tags: [], hostname: null, user: null };
}

const MAIN = "/home/f/.ssh/config";
const LAB = "/home/f/.ssh/config.d/homelab.config";
const PERSONAL = "/home/f/.ssh/sshelter/personal-3fa2c1d9.config";
const WORK = "/home/f/.ssh/sshelter/work-8b01e4aa.config";
const NONE = new Set<string>();

describe("isSyncableHost", () => {
  it("requires every pattern to be a plain name (same rule as the backend)", () => {
    expect(isSyncableHost(host("web", "/f"))).toBe(true);
    expect(isSyncableHost(host("web", "/f", ["web", "web.example.com"]))).toBe(true);
    expect(isSyncableHost(host("*", "/f", ["*"]))).toBe(false);
    expect(isSyncableHost(host("web", "/f", ["web", "*.internal"]))).toBe(false);
    expect(isSyncableHost(host("web", "/f", ["web", "!prod"]))).toBe(false);
    expect(isSyncableHost(host("web?", "/f", ["web?"]))).toBe(false);
    expect(isSyncableHost(host("w", "/f", []))).toBe(false);
  });
});

describe("cleanWordsInput", () => {
  it("joins lines, strips numbering and punctuation, lowercases", () => {
    const raw = "1. Abandon\n2) abandon,\n3 - ABANDON\n\n  about ";
    expect(cleanWordsInput(raw)).toBe("abandon abandon abandon about");
  });

  it("collapses whitespace including full-width spaces", () => {
    expect(cleanWordsInput("a　b   c")).toBe("a b c");
  });
});

describe("groupHostsForMigration", () => {
  it("groups real hosts by source file and skips wildcards and hosts already in a space", () => {
    const hosts = [
      host("web", MAIN),
      host("*", MAIN, ["*"]),
      host("mixed", MAIN, ["mixed", "*.internal"]),
      host("db", LAB),
      host("synced", PERSONAL),
      host("other", WORK),
    ];
    const groups = groupHostsForMigration(hosts, [PERSONAL, WORK], NONE);
    expect(groups.map((g) => g.file)).toEqual([MAIN, LAB]);
    expect(groups[0].hosts.map((h) => h.alias)).toEqual(["web"]);
    expect(groups[1].hosts.map((h) => h.alias)).toEqual(["db"]);
  });

  it("returns no groups when nothing is left to move", () => {
    expect(groupHostsForMigration([host("synced", PERSONAL)], [PERSONAL], NONE)).toEqual([]);
  });

  it("leaves out local hosts that share any name with a host in any space", () => {
    const hosts = [
      host("web", PERSONAL),
      host("app-1", WORK, ["app-1", "app"]),
      host("web", MAIN), // same alias as a synced host
      host("web-prod", MAIN, ["web-prod", "web"]), // a later pattern is synced
      host("app", LAB), // matches a synced host's later pattern, in another space
      host("db", MAIN),
      host("app-2", LAB), // similar, but no shared name
    ];
    const groups = groupHostsForMigration(hosts, [PERSONAL, WORK], NONE);
    expect(groups.map((g) => [g.file, g.hosts.map((h) => h.alias)])).toEqual([
      [MAIN, ["db"]],
      [LAB, ["app-2"]],
    ]);
  });

  it("leaves out hosts the backend can never move (it lists them with the reason)", () => {
    const groups = groupHostsForMigration([host("jump", MAIN), host("db", MAIN)], [PERSONAL], new Set(["jump"]));
    expect(groups[0].hosts.map((h) => h.alias)).toEqual(["db"]);
  });
});

describe("keepVisible", () => {
  it("drops selected hosts the wizard no longer lists", () => {
    const before = groupHostsForMigration([host("web", MAIN), host("db", MAIN)], [PERSONAL], NONE);
    const selected = new Set(["web", "db"]);
    expect(keepVisible(selected, before)).toBe(selected);
    // The first sync of the space brings a synced `web`: the local `web` leaves the list and the selection.
    const after = groupHostsForMigration([host("web", PERSONAL), host("web", MAIN), host("db", MAIN)], [PERSONAL], NONE);
    expect([...keepVisible(selected, after)]).toEqual(["db"]);
  });

  it("keeps the same set when every selected host is still listed", () => {
    const groups = groupHostsForMigration([host("a", "/f"), host("b", "/f"), host("c", "/g")], [PERSONAL], NONE);
    const selected = new Set(["a", "c"]);
    expect(keepVisible(selected, groups)).toBe(selected);
    const none = new Set<string>();
    expect(keepVisible(none, groups)).toBe(none);
  });

  it("drops everything when nothing is listed", () => {
    expect(keepVisible(new Set(["a"]), []).size).toBe(0);
  });
});

describe("tagForFile", () => {
  it("is the backend's tag_for_file: the file name without .config/.conf, lowercase, [a-z0-9_-] only", () => {
    expect(tagForFile(LAB)).toBe("homelab");
    expect(tagForFile("/home/f/.ssh/config.d/web.conf")).toBe("web");
    expect(tagForFile(MAIN)).toBe("config");
    expect(tagForFile("/x/My Servers (old).config")).toBe("my-servers-old");
    expect(tagForFile("/x/Lab.CONFIG")).toBe("lab-config"); // the suffix check is case-sensitive, like the backend
    expect(tagForFile("C:\\Users\\f\\.ssh\\work_vm.conf")).toBe("work_vm");
  });
});

describe("uniqueSpaceName", () => {
  it("adds a number when the name is taken, ignoring case", () => {
    expect(uniqueSpaceName("homelab", ["Personal"])).toBe("homelab");
    expect(uniqueSpaceName("homelab", ["HomeLab"])).toBe("homelab 2");
    expect(uniqueSpaceName("homelab", ["homelab", "homelab 2"])).toBe("homelab 3");
  });

  it("stays within 64 characters", () => {
    const name = uniqueSpaceName("x".repeat(64), ["x".repeat(64)]);
    expect(name).toBe(`${"x".repeat(62)} 2`);
  });
});

describe("one space per file", () => {
  const groups = groupHostsForMigration([host("web", MAIN), host("db", LAB), host("nas", "/home/f/.orbstack/ssh/config")], [PERSONAL], NONE);

  it("names each new space after its file: the sidebar alias first, else the file's tag, never a taken name", () => {
    const names = plannedSpaceNames(groups, { [LAB]: "Home lab" }, ["Personal", "config"]);
    expect([...names.entries()]).toEqual([
      [MAIN, "config 2"],
      [LAB, "Home lab"],
      ["/home/f/.orbstack/ssh/config", "config 3"],
    ]);
  });

  it("sends only the files with selected hosts", () => {
    const names = plannedSpaceNames(groups, {}, []);
    expect(newSpaceGroups(groups, new Set(["db", "nas"]), names)).toEqual([
      { name: "homelab", aliases: ["db"] },
      { name: "config 2", aliases: ["nas"] },
    ]);
  });

  it("sends each host once: a name an earlier file already sends, as an alias or as another name of its host, stays out", () => {
    const twice = groupHostsForMigration(
      [host("web", MAIN), host("db", MAIN, ["db", "database"]), host("web", LAB), host("database", LAB), host("nas", LAB)],
      [PERSONAL],
      NONE,
    );
    expect(newSpaceGroups(twice, new Set(["web", "db", "database", "nas"]), plannedSpaceNames(twice, {}, []))).toEqual([
      { name: "config", aliases: ["web", "db"] },
      { name: "homelab", aliases: ["nas"] },
    ]);
  });
});

describe("migrationTarget", () => {
  const personal = space();
  const work = space({ id: "b".repeat(64), name: "Work", first_sync_pending: true });
  const off = space({ id: "c".repeat(64), name: "Old", selected: false, file_name: null, file_path: null, hosts: null });

  it("keeps the space the wizard was opened for while this computer syncs it", () => {
    expect(migrationTarget([personal, work], work.id)).toBe(work.id);
  });

  it("otherwise picks the first synced space that finished its first sync", () => {
    expect(migrationTarget([work, personal], null)).toBe(personal.id);
    expect(migrationTarget([work], off.id)).toBe(work.id);
  });

  it("falls back to one new space per file when this computer syncs no space", () => {
    expect(migrationTarget([off], null)).toBe(PER_FILE);
    expect(migrationTarget([], null)).toBe(PER_FILE);
  });
});

describe("preselectAll", () => {
  it("selects every host only while all synced spaces are empty and done with their first sync", () => {
    expect(preselectAll([space({ hosts: 0 })])).toBe(true); // right after "Create"
    expect(preselectAll([space({ hosts: 0 }), space({ id: "b".repeat(64), hosts: 2 })])).toBe(false);
    expect(preselectAll([space({ hosts: 0, first_sync_pending: true })])).toBe(false);
    expect(preselectAll([])).toBe(false);
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `pnpm exec vitest run --dir src src/lib/sync-migration.test.ts`
Expected: FAIL —— 15 failed | 5 passed(`tagForFile is not a function`、`migrationTarget is not a function` 等;舊的
`groupHostsForMigration` 不認得 space 檔清單與不能搬的主機)。

- [ ] **Step 3: 改寫 `src/lib/sync-migration.ts`**

把 `src/lib/sync-migration.ts` 整個換成:

```ts
import type { HostSummary } from "@/bindings/HostSummary";
import type { NewSpaceGroup } from "@/bindings/NewSpaceGroup";
import type { SyncSpaceView } from "@/bindings/SyncSpaceView";
import { MAX_SPACE_NAME } from "@/lib/sync-spaces";
import { basename } from "@/lib/utils";

/**
 * Same rule as the backend's `hosts_file::is_syncable_block`: every pattern must be
 * a plain name. `Host web *.internal` or `Host web !prod` is a wildcard rule, not a
 * host — the sidebar's `isWildcardOnly` (all patterns wildcard, `!` ignored) is the
 * wrong predicate for sync.
 */
export function isSyncableHost(h: HostSummary): boolean {
  return h.patterns.length > 0 && h.patterns.every((p) => p !== "" && !/[*?!]/.test(p));
}

/**
 * Tidy a pasted sync code before the backend validates it: one line,
 * single spaces, lowercase, no list numbering or stray punctuation.
 */
export function cleanWordsInput(raw: string): string {
  return raw
    .split(/\r?\n/)
    .map((line) => line.replace(/^\s*\d+\s*[.)\-:]?\s*/, ""))
    .join(" ")
    .toLowerCase()
    .replace(/[^a-z\s　]/g, " ")
    .replace(/[\s　]+/g, " ")
    .trim();
}

export interface MigrationGroup {
  file: string;
  hosts: HostSummary[];
}

/**
 * Hosts that can still move into a space, grouped by their current file
 * (`spaceFiles` = the files of the spaces this computer syncs). A local host that
 * shares any name with a host in any space is left out: in the same space the
 * backend refuses it, in another space ssh would only ever use one of the two —
 * and right after joining these are exactly the local copies of hosts the account
 * already has (the duplicates list offers to rename or remove them). Hosts the
 * backend can never move (`unmovable`, from `sync_unmovable_hosts`: an `Include`,
 * a value ssh would pass to a shell, …) are listed separately, each with its reason.
 */
export function groupHostsForMigration(
  hosts: readonly HostSummary[],
  spaceFiles: readonly string[],
  unmovable: ReadonlySet<string>,
): MigrationGroup[] {
  const inSpace = new Set(spaceFiles);
  const syncedNames = new Set(hosts.filter((h) => inSpace.has(h.source_file)).flatMap((h) => h.patterns));
  const byFile = new Map<string, HostSummary[]>();
  for (const h of hosts) {
    if (inSpace.has(h.source_file) || !isSyncableHost(h) || unmovable.has(h.alias)) continue;
    if (h.patterns.some((p) => syncedNames.has(p))) continue;
    const bucket = byFile.get(h.source_file);
    if (bucket) bucket.push(h);
    else byFile.set(h.source_file, [h]);
  }
  return [...byFile.entries()].map(([file, hosts]) => ({ file, hosts }));
}

/**
 * The selection limited to hosts the wizard still lists. A selected host can
 * drop out of the list — e.g. a same-name local host once the first sync brings
 * in its synced twin — and must then neither count toward "Move N hosts" nor be
 * submitted. Returns `selected` itself when nothing was dropped, so a state
 * update with the result is a no-op.
 */
export function keepVisible(selected: Set<string>, groups: { hosts: { alias: string }[] }[]): Set<string> {
  const visible = new Set(groups.flatMap((g) => g.hosts.map((h) => h.alias)));
  const kept = new Set([...selected].filter((alias) => visible.has(alias)));
  return kept.size === selected.size ? selected : kept;
}

/** The wizard's target value for "one new space per source file". */
export const PER_FILE = "__per_file__";

/**
 * The backend's `migrate::tag_for_file`: the file name without `.config` / `.conf`
 * (case-sensitive), lowercased, every run of characters outside `[a-z0-9_-]`
 * turned into one `-`, with `-` trimmed from both ends.
 */
export function tagForFile(path: string): string {
  const name = basename(path);
  const stem = (name.endsWith(".config") ? name.slice(0, -7) : name.endsWith(".conf") ? name.slice(0, -5) : name).toLowerCase();
  let out = "";
  for (const ch of stem) {
    if (/^[a-z0-9_-]$/.test(ch)) out += ch;
    else if (!out.endsWith("-")) out += "-";
  }
  return out.replace(/^-+|-+$/g, "");
}

function truncate(name: string, max: number): string {
  return [...name].slice(0, max).join("").trimEnd();
}

/** `base`, or `base 2`, `base 3`… — the first one no name in `taken` matches, ignoring case (like the backend). */
export function uniqueSpaceName(base: string, taken: Iterable<string>): string {
  const used = new Set([...taken].map((n) => n.toLowerCase()));
  const first = truncate(base, MAX_SPACE_NAME);
  if (!used.has(first.toLowerCase())) return first;
  for (let n = 2; ; n += 1) {
    const suffix = ` ${n}`;
    const name = truncate(base, MAX_SPACE_NAME - suffix.length) + suffix;
    if (!used.has(name.toLowerCase())) return name;
  }
}

/**
 * "One new space per source file" (spec §7.2): each file's space is named after
 * the file — the sidebar label the user gave it, else its tag (`tagForFile`) —
 * and made unique against the account's spaces and the other new ones, so no
 * group fails on a taken name. Computed for every listed file so the names stay
 * put while the user changes the selection.
 */
export function plannedSpaceNames(
  groups: readonly MigrationGroup[],
  fileAliases: Record<string, string>,
  existing: readonly string[],
): Map<string, string> {
  const taken = [...existing];
  const names = new Map<string, string>();
  for (const g of groups) {
    const base = fileAliases[g.file]?.trim() || tagForFile(g.file) || "Space";
    const name = uniqueSpaceName(base, taken);
    taken.push(name);
    names.set(g.file, name);
  }
  return names;
}

/**
 * What `sync_move_files_to_new_spaces` gets: one group per file that has selected
 * hosts, each host once. The backend moves a name only for the first group that
 * lists it — as an alias, or as another name of a host that group moves
 * (`Host db database`) — and fails it as "listed in more than one group" after
 * that, so a host defined in two files goes with the file listed first.
 */
export function newSpaceGroups(groups: readonly MigrationGroup[], selected: ReadonlySet<string>, names: ReadonlyMap<string, string>): NewSpaceGroup[] {
  const sent = new Set<string>();
  return groups.flatMap((g) => {
    const aliases: string[] = [];
    for (const h of g.hosts) {
      if (!selected.has(h.alias) || sent.has(h.alias)) continue;
      aliases.push(h.alias);
      for (const name of [h.alias, ...h.patterns]) sent.add(name);
    }
    const name = names.get(g.file);
    return aliases.length > 0 && name ? [{ name, aliases }] : [];
  });
}

/**
 * Where the wizard moves hosts: the space it was opened for, while this computer
 * syncs it; else the first synced space past its first sync (else any synced
 * one); else one new space per file.
 */
export function migrationTarget(spaces: readonly SyncSpaceView[], requested: string | null): string {
  const synced = spaces.filter((s) => s.selected);
  if (requested && synced.some((s) => s.id === requested)) return requested;
  return (synced.find((s) => !s.first_sync_pending && !s.missing) ?? synced[0])?.id ?? PER_FILE;
}

/**
 * Preselect every listed host only for a fresh account — every synced space empty
 * and past its first sync (right after "Create") — otherwise the user picks.
 */
export function preselectAll(spaces: readonly SyncSpaceView[]): boolean {
  const synced = spaces.filter((s) => s.selected);
  return synced.length > 0 && synced.every((s) => !s.first_sync_pending && s.hosts === 0);
}
```

- [ ] **Step 4: 修改 `src/stores/ui.ts`:精靈帶目標 space**

`src/stores/ui.ts`:把

```ts
  /** Whether the ⌘K command palette is open (also driven by the global quick-connect hotkey). */
  paletteOpen: boolean;
  setPaletteOpen: (open: boolean) => void;
  /** Whether the "Move hosts into sync" wizard is open. Session-only. */
  syncMigrationOpen: boolean;
  setSyncMigrationOpen: (open: boolean) => void;
  /** Whether the review of synced hosts waiting for approval is open (approval toast, Settings → Sync). Session-only. */
  syncApprovalsOpen: boolean;
  setSyncApprovalsOpen: (open: boolean) => void;
```

換成:

```ts
  /** Whether the ⌘K command palette is open (also driven by the global quick-connect hotkey). */
  paletteOpen: boolean;
  setPaletteOpen: (open: boolean) => void;
  /**
   * The "Move hosts into a space" wizard: open while non-null; `spaceId` is the
   * space it should move hosts into (null = let the wizard pick). Session-only.
   */
  syncMigration: { spaceId: string | null } | null;
  setSyncMigration: (value: { spaceId: string | null } | null) => void;
  /** Whether the review of synced hosts waiting for approval is open (approval toast, Settings → Sync). Session-only. */
  syncApprovalsOpen: boolean;
  setSyncApprovalsOpen: (open: boolean) => void;
```

`src/stores/ui.ts`:把

```ts
      setSettingsCategory: (settingsCategory) => set({ settingsCategory }),
      paletteOpen: false,
      setPaletteOpen: (paletteOpen) => set({ paletteOpen }),
      syncMigrationOpen: false,
      setSyncMigrationOpen: (syncMigrationOpen) => set({ syncMigrationOpen }),
      syncApprovalsOpen: false,
      setSyncApprovalsOpen: (syncApprovalsOpen) => set({ syncApprovalsOpen }),
    }),
```

換成:

```ts
      setSettingsCategory: (settingsCategory) => set({ settingsCategory }),
      paletteOpen: false,
      setPaletteOpen: (paletteOpen) => set({ paletteOpen }),
      syncMigration: null,
      setSyncMigration: (syncMigration) => set({ syncMigration }),
      syncApprovalsOpen: false,
      setSyncApprovalsOpen: (syncApprovalsOpen) => set({ syncApprovalsOpen }),
    }),
```

- [ ] **Step 5: 改寫 `src/components/SyncMigrationDialog.tsx`**

把 `src/components/SyncMigrationDialog.tsx` 整個換成:

```tsx
import { useEffect, useMemo, useRef, useState } from "react";
import { Loader2 } from "lucide-react";
import { toast } from "sonner";

import type { MigrationFailure } from "@/bindings/MigrationFailure";
import type { MigrationReport } from "@/bindings/MigrationReport";
import { useHostsQuery } from "@/lib/queries";
import { labelsFor } from "@/lib/host-display";
import {
  PER_FILE,
  groupHostsForMigration,
  keepVisible,
  migrationTarget,
  newSpaceGroups,
  plannedSpaceNames,
  preselectAll,
} from "@/lib/sync-migration";
import { spaceFileLabels } from "@/lib/sync-sidebar";
import { plural } from "@/lib/sync-overview";
import {
  useDuplicateAliases,
  useMoveFilesToNewSpaces,
  useMoveHostsToSpace,
  useResolveShadowed,
  useSyncOverview,
  useUnmovableHosts,
} from "@/lib/sync";
import { useSettingsStore } from "@/stores/settings";
import { useUiStore } from "@/stores/ui";
import { basename } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";
import { Select, SelectContent, SelectItem, SelectSeparator, SelectTrigger, SelectValue } from "@/components/ui/select";

/**
 * "Move hosts into a space" (spec §7.2, §8): pick existing hosts (grouped by
 * file) and move them into one space this computer syncs — or into one new space
 * per source file — optionally tagging those from included files with their old
 * file's name (never the main config's "config"). Lists the hosts that can never
 * be synced and why, and resolves aliases a space now shadows — addressed by
 * file, never by first match, so the copy ssh uses is never touched.
 */
export function SyncMigrationDialog() {
  const target = useUiStore((s) => s.syncMigration);
  const setTarget = useUiStore((s) => s.setSyncMigration);
  return (
    <Dialog
      open={target !== null}
      onOpenChange={(open) => {
        if (!open) setTarget(null);
      }}
    >
      <DialogContent className="sm:max-w-lg">
        {target && <MigrationFlow requested={target.spaceId} onClose={() => setTarget(null)} />}
      </DialogContent>
    </Dialog>
  );
}

function MigrationFlow({ requested, onClose }: { requested: string | null; onClose: () => void }) {
  const overview = useSyncOverview();
  const hostsQuery = useHostsQuery();
  const fileAliases = useSettingsStore((s) => s.fileAliases);
  const moveToSpace = useMoveHostsToSpace();
  const moveToNewSpaces = useMoveFilesToNewSpaces();
  const duplicates = useDuplicateAliases(true);
  const unmovable = useUnmovableHosts(true);
  const resolve = useResolveShadowed();
  const [tagByFile, setTagByFile] = useState(true);
  const [selected, setSelected] = useState<Set<string>>(new Set());
  // null = follow `migrationTarget` (the requested space, else a sensible default).
  const [chosenTarget, setChosenTarget] = useState<string | null>(null);
  // The previous Move's failures (if any), kept on screen until the next Move
  // replaces them or the dialog closes (this component unmounts then).
  const [failedMoves, setFailedMoves] = useState<MigrationFailure[]>([]);
  // Guards the default-selection effect below so it only ever settles once per
  // dialog opening (this component unmounts when the dialog closes).
  const didDefaultSelect = useRef(false);

  const spaces = useMemo(() => overview.data?.spaces ?? [], [overview.data]);
  const synced = spaces.filter((s) => s.selected);
  const spaceFiles = useMemo(() => [...spaceFileLabels(spaces).keys()], [spaces]);
  // The user's pick while it is still valid (a space can be turned off meanwhile).
  const pickValid = chosenTarget === PER_FILE || synced.some((s) => s.id === chosenTarget);
  const target = chosenTarget !== null && pickValid ? chosenTarget : migrationTarget(spaces, requested);
  const targetSpace = synced.find((s) => s.id === target) ?? null;
  const unmovableAliases = useMemo(() => new Set((unmovable.data ?? []).map((f) => f.alias)), [unmovable.data]);
  const loaded = Boolean(overview.data && hostsQuery.data && unmovable.data);
  // Only group once everything is loaded: before the overview arrives, no file
  // counts as a space file, and synced hosts would be listed (and preselected)
  // as if they were local.
  const groups = useMemo(
    () => (overview.data && hostsQuery.data && unmovable.data ? groupHostsForMigration(hostsQuery.data.hosts, spaceFiles, unmovableAliases) : []),
    [overview.data, hostsQuery.data, unmovable.data, spaceFiles, unmovableAliases],
  );
  const files = useMemo(() => hostsQuery.data?.files ?? [], [hostsQuery.data]);
  const labels = useMemo(() => labelsFor(files, { ...fileAliases, ...Object.fromEntries(spaceFileLabels(spaces)) }), [files, fileAliases, spaces]);
  const newNames = useMemo(() => plannedSpaceNames(groups, fileAliases, spaces.map((s) => s.name)), [groups, fileAliases, spaces]);
  // The target space was just turned on and its first (baseline) sync has not
  // finished: hosts moved now would race it, and the backend refuses them anyway.
  const waitingForFirstSync = targetSpace?.first_sync_pending === true;

  // The default selection is decided once per dialog opening, and only after no
  // synced space is waiting for its first sync. Everything is preselected only
  // for a fresh account (all synced spaces empty, e.g. right after Create);
  // otherwise the user picks. A manual toggle settles it too.
  useEffect(() => {
    if (didDefaultSelect.current || !overview.data || groups.length === 0) return;
    if (spaces.some((s) => s.selected && s.first_sync_pending)) return;
    didDefaultSelect.current = true;
    if (preselectAll(spaces)) setSelected(new Set(groups.flatMap((g) => g.hosts.map((h) => h.alias))));
  }, [groups, overview.data, spaces]);

  // Whenever the list changes, drop selected hosts it no longer shows — e.g. a
  // same-name local host once a first sync brings in its synced twin. The effect
  // runs after render, so the count and the submitted aliases also go through
  // `keepVisible` for the render in between.
  useEffect(() => {
    setSelected((prev) => keepVisible(prev, groups));
  }, [groups]);
  const visibleSelected = useMemo(() => keepVisible(selected, groups), [selected, groups]);

  const toggle = (alias: string, on: boolean) => {
    didDefaultSelect.current = true;
    setSelected((prev) => {
      const next = new Set(prev);
      if (on) next.add(alias);
      else next.delete(alias);
      return next;
    });
  };

  const done = (report: MigrationReport, into: string) => {
    // `failed` also lists a host that moved but whose tag could not be saved.
    const failed = report.failed.length;
    toast.success(`Moved ${plural(report.moved.length, "host")} ${into}${failed ? `, ${plural(failed, "host")} with a problem` : ""}`);
    setSelected(new Set());
    // A failed write stops the batch, so entries after it are "not attempted" —
    // surfaced below, not just as a count.
    setFailedMoves(report.failed);
  };

  const run = () => {
    const aliases = [...visibleSelected];
    if (target === PER_FILE) {
      const newGroups = newSpaceGroups(groups, visibleSelected, newNames);
      // A group none of whose hosts can move creates no space, so the toast does not count spaces.
      moveToNewSpaces.mutate({ groups: newGroups, tagByFile }, { onSuccess: (report) => done(report, "into new spaces") });
    } else if (targetSpace) {
      moveToSpace.mutate({ aliases, spaceId: targetSpace.id, tagByFile }, { onSuccess: (report) => done(report, `into ${targetSpace.name}`) });
    }
  };

  const pending = moveToSpace.isPending || moveToNewSpaces.isPending;
  const dups = duplicates.data ?? [];
  const cannotMove = unmovable.data ?? [];

  return (
    <>
      <DialogHeader>
        <DialogTitle>Move hosts into a space</DialogTitle>
        <DialogDescription>
          Selected hosts move into the space's file in ~/.ssh/sshelter (a backup is written first) and appear on every computer that syncs that space.
          Wildcard blocks stay where they are. Space files are read before your main config, so moved hosts' own options now take precedence over wildcard
          blocks (like <span className="font-mono">Host *</span>) earlier in that file.
        </DialogDescription>
      </DialogHeader>

      <div className="flex items-center gap-2 text-sm">
        <span className="shrink-0 text-muted-foreground">Move into</span>
        <Select value={target} onValueChange={setChosenTarget}>
          <SelectTrigger className="h-7 min-w-0 flex-1 text-sm" aria-label="Target space">
            <SelectValue />
          </SelectTrigger>
          <SelectContent>
            {synced.map((s) => (
              <SelectItem key={s.id} value={s.id}>
                {s.name}
                {s.first_sync_pending ? " (first sync…)" : ""}
              </SelectItem>
            ))}
            {synced.length > 0 && <SelectSeparator />}
            <SelectItem value={PER_FILE}>One new space per file</SelectItem>
          </SelectContent>
        </Select>
      </div>

      {!loaded ? (
        <p className="text-sm text-muted-foreground">Loading hosts…</p>
      ) : groups.length === 0 ? (
        <p className="text-sm text-muted-foreground">Every host that can be synced is already in a space.</p>
      ) : (
        <div className="max-h-[40vh] space-y-3 overflow-y-auto pr-1">
          {groups.map((g) => (
            <div key={g.file} className="space-y-1">
              <p className="text-xs font-semibold tracking-wide text-muted-foreground uppercase">
                {labels.get(g.file) ?? basename(g.file)}
                {target === PER_FILE && <span className="font-normal normal-case tracking-normal"> → new space “{newNames.get(g.file)}”</span>}
              </p>
              {g.hosts.map((h) => (
                <label key={h.alias} className="flex items-center gap-2 text-sm">
                  <Checkbox checked={selected.has(h.alias)} onCheckedChange={(v) => toggle(h.alias, v === true)} />
                  <span className="font-mono">{h.alias}</span>
                  {h.hostname && (
                    <span className="truncate text-xs text-muted-foreground">
                      {h.user ? `${h.user}@` : ""}
                      {h.hostname}
                    </span>
                  )}
                </label>
              ))}
            </div>
          ))}
        </div>
      )}

      {groups.length > 0 && (
        <label className="flex items-center gap-2 text-sm">
          <Checkbox checked={tagByFile} onCheckedChange={(v) => setTagByFile(v === true)} />
          Tag hosts from included files with their file name (keeps your grouping in tag view)
        </label>
      )}

      {groups.length > 0 && waitingForFirstSync && targetSpace && (
        <p className="text-sm text-muted-foreground">Waiting for the first sync of {targetSpace.name} to finish…</p>
      )}

      {cannotMove.length > 0 && (
        <div className="space-y-1.5 rounded-md border p-3 text-xs">
          <p className="font-medium">Can't be synced</p>
          {cannotMove.map((f) => (
            <div key={f.alias} className="space-y-0.5">
              <span className="font-mono">{f.alias}</span>
              <p className="text-muted-foreground">{f.error}</p>
            </div>
          ))}
        </div>
      )}

      {failedMoves.length > 0 && (
        <div className="space-y-1.5 rounded-md border border-destructive/40 bg-destructive/10 p-3 text-xs">
          <p className="font-medium text-destructive">Problems with {plural(failedMoves.length, "host")}</p>
          {failedMoves.map((f) => (
            <div key={f.alias} className="space-y-0.5">
              <span className="font-mono">{f.alias}</span>
              <p className="text-muted-foreground">{f.error}</p>
            </div>
          ))}
        </div>
      )}

      {dups.length > 0 && (
        <div className="space-y-1.5 rounded-md border border-amber-500/40 bg-amber-500/10 p-3 text-xs">
          <p className="font-medium text-amber-700 dark:text-amber-400">Hosts defined in more than one file</p>
          <p className="text-muted-foreground">
            ssh uses the copy in the space listed first, but still takes any option that copy does not set from these, and IdentityFile entries add up. Keep
            this copy under a new name, or remove it and use only the one ssh reads:
          </p>
          {dups.map((d) => (
            <div key={`${d.alias}-${d.local_file}`} className="flex items-center justify-between gap-2">
              <span className="font-mono">
                {d.alias} <span className="text-muted-foreground">in {labels.get(d.local_file) ?? basename(d.local_file)}</span>
              </span>
              <div className="flex gap-1">
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs" disabled={resolve.isPending} onClick={() => resolve.mutate({ alias: d.alias, file: d.local_file, action: "rename" })}>
                  Keep as {d.alias}-local
                </Button>
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs text-destructive" disabled={resolve.isPending} onClick={() => resolve.mutate({ alias: d.alias, file: d.local_file, action: "remove" })}>
                  Remove this copy
                </Button>
              </div>
            </div>
          ))}
        </div>
      )}

      <DialogFooter>
        <Button type="button" variant="outline" onClick={onClose}>
          Close
        </Button>
        {groups.length > 0 && (
          <Button type="button" disabled={visibleSelected.size === 0 || pending || waitingForFirstSync || (target !== PER_FILE && !targetSpace)} onClick={run}>
            {pending && <Loader2 className="size-4 animate-spin" />} Move {plural(visibleSelected.size, "host")}
          </Button>
        )}
      </DialogFooter>
    </>
  );
}
```

- [ ] **Step 6: 修改 `src/components/SyncPane.tsx` 與 `src/components/SyncSpacesSection.tsx`:打開精靈的入口**

`src/components/SyncPane.tsx`:把

```tsx
 */
export function SyncPane() {
  const overview = useSyncOverview(5_000);
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);
  const dismiss = useDismissNotice();
  const [createdWords, setCreatedWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // shown on request
```

換成:

```tsx
 */
export function SyncPane() {
  const overview = useSyncOverview(5_000);
  const openMigration = useUiStore((s) => s.setSyncMigration);
  const dismiss = useDismissNotice();
  const [createdWords, setCreatedWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // shown on request
```

`src/components/SyncPane.tsx`:把

```tsx
        <NotJoinedPane overview={overview.data} onCreated={setCreatedWords} onJoined={() => setChoosingSpaces(true)} />
      )}

      <ChooseSpacesDialog open={choosingSpaces} overview={overview.data} onClose={() => setChoosingSpaces(false)} />

      <SyncCodeDialog
        mode="created"
        words={createdWords}
        onDone={() => {
          setCreatedWords(null);
          setMigrationOpen(true);
        }}
      />
      <SyncCodeDialog mode="shown" words={shownWords} onDone={() => setShownWords(null)} />
```

換成:

```tsx
        <NotJoinedPane overview={overview.data} onCreated={setCreatedWords} onJoined={() => setChoosingSpaces(true)} />
      )}

      <ChooseSpacesDialog
        open={choosingSpaces}
        overview={overview.data}
        onClose={(selected) => {
          setChoosingSpaces(false);
          // As after creating: offer to move this computer's hosts into a space next.
          if (selected.length > 0) openMigration({ spaceId: selected[0] });
        }}
      />

      <SyncCodeDialog
        mode="created"
        words={createdWords}
        onDone={() => {
          setCreatedWords(null);
          openMigration({ spaceId: null });
        }}
      />
      <SyncCodeDialog mode="shown" words={shownWords} onDone={() => setShownWords(null)} />
```

`src/components/SyncSpacesSection.tsx`:把

```tsx
import { useState } from "react";
import { Loader2, MoreHorizontal, Pencil, Plus, Trash2 } from "lucide-react";
import { toast } from "sonner";

import type { SyncOverview } from "@/bindings/SyncOverview";
```

換成:

```tsx
import { useState } from "react";
import { FolderInput, Loader2, MoreHorizontal, Pencil, Plus, Trash2 } from "lucide-react";
import { toast } from "sonner";

import type { SyncOverview } from "@/bindings/SyncOverview";
```

`src/components/SyncSpacesSection.tsx`:把

```tsx
import { listNames } from "@/lib/sync-events";
import { plural } from "@/lib/sync-overview";
import { spaceNameError, spaceRows, structureLock, type SpaceRow } from "@/lib/sync-spaces";
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { TONE_TEXT } from "@/components/sync-primitives";
```

換成:

```tsx
import { listNames } from "@/lib/sync-events";
import { plural } from "@/lib/sync-overview";
import { spaceNameError, spaceRows, structureLock, type SpaceRow } from "@/lib/sync-spaces";
import { useUiStore } from "@/stores/ui";
import { cn } from "@/lib/utils";
import { Section, SettingsGroup, SettingsRow } from "@/components/settings-primitives";
import { TONE_TEXT } from "@/components/sync-primitives";
```

`src/components/SyncSpacesSection.tsx`:把

```tsx
export function SpacesSection({ overview: o }: { overview: SyncOverview }) {
  const select = useSelectSpace();
  const rebuild = useRebuildSpace();
  const [naming, setNaming] = useState<Naming | null>(null);
  const [unselecting, setUnselecting] = useState<SpaceRow | null>(null);
  const [deleting, setDeleting] = useState<SpaceRow | null>(null);
```

換成:

```tsx
export function SpacesSection({ overview: o }: { overview: SyncOverview }) {
  const select = useSelectSpace();
  const rebuild = useRebuildSpace();
  const openMigration = useUiStore((s) => s.setSyncMigration);
  const [naming, setNaming] = useState<Naming | null>(null);
  const [unselecting, setUnselecting] = useState<SpaceRow | null>(null);
  const [deleting, setDeleting] = useState<SpaceRow | null>(null);
```

`src/components/SyncSpacesSection.tsx`:把

```tsx
                  <DropdownMenuItem onSelect={() => setNaming({ mode: "rename", row })}>
                    <Pencil className="size-3.5" /> Rename…
                  </DropdownMenuItem>
                  <DropdownMenuSeparator />
                  <DropdownMenuItem variant="destructive" onSelect={() => setDeleting(row)}>
                    <Trash2 className="size-3.5" /> Delete…
```

換成:

```tsx
                  <DropdownMenuItem onSelect={() => setNaming({ mode: "rename", row })}>
                    <Pencil className="size-3.5" /> Rename…
                  </DropdownMenuItem>
                  {row.selected && !row.missing && (
                    <DropdownMenuItem onSelect={() => openMigration({ spaceId: row.id })}>
                      <FolderInput className="size-3.5" /> Move hosts here…
                    </DropdownMenuItem>
                  )}
                  <DropdownMenuSeparator />
                  <DropdownMenuItem variant="destructive" onSelect={() => setDeleting(row)}>
                    <Trash2 className="size-3.5" /> Delete…
```

`src/components/SyncSpacesSection.tsx`:把

```tsx
        <SettingsRow label="New space" description={lock ?? "Starts empty and syncs on this computer."}>
          <Button type="button" variant="outline" size="sm" className="h-7" disabled={lock !== null} onClick={() => setNaming({ mode: "create" })}>
            <Plus className="size-3.5" /> New space…
          </Button>
        </SettingsRow>
      </SettingsGroup>
```

換成:

```tsx
        <SettingsRow label="New space" description={lock ?? "Starts empty and syncs on this computer."}>
          <Button type="button" variant="outline" size="sm" className="h-7" disabled={lock !== null} onClick={() => setNaming({ mode: "create" })}>
            <Plus className="size-3.5" /> New space…
          </Button>
        </SettingsRow>
        <SettingsRow label="Move hosts into a space" description="Choose which of this computer's hosts should follow you to your other computers.">
          <Button type="button" variant="outline" size="sm" className="h-7" disabled={lock !== null} onClick={() => openMigration({ spaceId: null })}>
            Choose hosts…
          </Button>
        </SettingsRow>
      </SettingsGroup>
```

- [ ] **Step 7: 移除 v1:`src/lib/sync.ts` 的暫留 helper 與兩個 binding**

`src/lib/sync.ts`:把

```ts
import type { ReviewOutcome } from "@/bindings/ReviewOutcome";
import type { ReviewedVersion } from "@/bindings/ReviewedVersion";
import type { SyncOverview } from "@/bindings/SyncOverview";
import type { SyncStatus } from "@/bindings/SyncStatus";
import { tauriInvoke } from "@/lib/ipc";

export const syncOverviewKey = ["sync", "overview"] as const;
```

換成:

```ts
import type { ReviewOutcome } from "@/bindings/ReviewOutcome";
import type { ReviewedVersion } from "@/bindings/ReviewedVersion";
import type { SyncOverview } from "@/bindings/SyncOverview";
import { tauriInvoke } from "@/lib/ipc";

export const syncOverviewKey = ["sync", "overview"] as const;
```

`src/lib/sync.ts`:把

```ts
export function showWords(): Promise<string> {
  return tauriInvoke<string>("sync_show_words");
}

/*
 * v1 — the old migration wizard still uses these until it is rewritten for v2;
 * they go away with it.
 */

export const syncStatusKey = ["sync", "status"] as const;

export function useSyncStatus(refetchInterval: number | false = false) {
  return useQuery<SyncStatus>({
    queryKey: syncStatusKey,
    queryFn: () => tauriInvoke<SyncStatus>("sync_status"),
    refetchInterval,
  });
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
    mutationFn: ({ aliases, tagByFile }) => tauriInvoke<MigrationReport>("sync_migrate_hosts", { aliases, tagByFile }),
    onSuccess: () => refreshSyncViews(queryClient),
    onError: (error) => toast.error("Could not move hosts", { description: errorMessage(error) }),
  });
}
```

換成:

```ts
export function showWords(): Promise<string> {
  return tauriInvoke<string>("sync_show_words");
}
```

刪除不再產生的兩個 binding,並確認沒有任何地方還用到 v1 的名稱:

```bash
git rm src/bindings/SyncStatus.ts src/bindings/SyncDevice.ts
grep -rnE "SyncStatus|SyncDevice\b|useSyncStatus|useMigrateHosts|syncMigrationOpen" src
```

Expected:`grep` 沒有輸出。

- [ ] **Step 8: 跑測試確認通過**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  20 passed (20)`、`Tests  249 passed (249)`(`sync-migration.test.ts` 由 10 個變成 20 個)。

Run: `pnpm exec tsc --noEmit`
Expected: 沒有輸出。

- [ ] **Step 9: Commit**

```bash
git add src/lib/sync-migration.ts src/lib/sync-migration.test.ts src/components/SyncMigrationDialog.tsx
git add src/stores/ui.ts src/components/SyncPane.tsx src/components/SyncSpacesSection.tsx src/lib/sync.ts
git commit -m "feat(sync): move hosts into a space or one new space per file"
```

(Step 7 的 `git rm` 已經把兩個刪除放進 index。)

---

### Task 7: 文件:README 的 Sync 段落與兩台電腦的手動清單

> **已執行**(repo `6548997`;審查後的修正 `723b99d`、`e793ae8`、`e3ea14e`)。下面保留原本的步驟作為紀錄,不要再執行;README
> 與手動清單以 repo 為準。執行時先照實際的程式行為改了八處(例如第 8 項的 RTL 例子改用 `ForwardAgent`,因為 ProxyCommand 那一
> 行現在整行會被拒絕;第 14 項加上退避)。審查找到四個會讓測試者誤判失敗的步驟(第 8 項「保留 ProxyCommand」那一步、讓 B 的
> space 停住的 RTL 例子、第 10 項的 Cancel 只有不到一秒的時間 —— 改成先停掉 relay、第 14 項的輪詢節奏),第一輪修正;第二輪調整
> 第 12、9 項與 README;第三輪從頭到尾走一遍,讓每一項都從前一項留下的狀態開始。最終修正之後清單與 README 再跟著改過
> (`e363a73`、`8a9cd02`、`5b55188`):第 8 項 LocalForward 的順序、第 7 項照新的側邊欄重寫,並新增檢查 —— space 的錯誤出現在
> 狀態列與群組標題、別的 space 已有的主機、暫停時審核被鎖、更換期間的 Show、離開時的未上傳修改與殘留提示、相對時間、
> IME Enter;README 照 ssh 實際的方式說明同名主機。`src/` 的測試數不變(299)。

README 的 Sync 段落改寫成 v2(解讀 13,含離開後檔案在 `~/.ssh/sshelter-local/`)。手動清單(spec §10「手動」)放在
v1 清單旁邊,涵蓋:從 v1 升級(含使用者自己放在 `~/.ssh/sshelter/` 的檔案被搬到 `~/.ssh/sshelter-local/`)、離開後再建立
(檔案搬到 `~/.ssh/sshelter-local/`)、加入與選擇 space、開關 space、跨 space
搬移、一個來源檔一個 space、跨 space 同名、危險設定的核准、更換同步碼(含一台離線的電腦)與取消、沒有 freeze / 批次
查詢的 relay、改名被擋(關掉之後不再出現)、在別台刪除的 space、焦點與 relay 用量、離開與刪除帳戶、搬不過去的離開、
更換同步碼期間的離開(凍結前離開等於取消;凍結後被拒絕;新碼不見時照常離開,並指向新的同步帳戶);
核准那一項另外檢查對話框開著時才到的新版本不會被套用,更換同步碼那一項檢查被凍結的電腦離開時不提供刪除帳戶、
輸入別的帳戶的碼會被拒絕,一個來源檔一個 space 那一項檢查兩個檔案都有的主機只送一次,加入那一項檢查 v1 留下的
`hosts.config` 只在主 config 的 Include(含 glob)還讀它時被搬到 `~/.ssh/sshelter-local/`、沒有 Include 讀它就留在原地。
`sync.test.ts` 檢查的兩個 README 連結(部署按鈕、`relay/README.md#updating-your-relay`)都留在原處。

**Files:**
- Modify: `README.md`
- Create: `docs/superpowers/plans/2026-10-02-sync-v2-manual-verification.md`

**Interfaces:**
- Consumes:Task 1–6 的 UI 文案(清單逐字引用按鈕與訊息)。
- Produces:無程式介面。


- [ ] **Step 1: 修改 `README.md`**

`README.md`:把

```markdown

## Sync

Sync is in beta. Open **Settings → Sync**. *Create* shows a 24-word recovery phrase — store it in a password manager; it is the only secret and anyone holding it can read your synced hosts. On another computer choose *Join* and paste the words. SSHelter needs an existing SSH config: on a new machine create an empty `~/.ssh/config` first.

Synced hosts live in `~/.ssh/sshelter/hosts.config`, which SSHelter `Include`s at the top of your main config, so plain `ssh` keeps working and the file survives uninstalling SSHelter. Use *Choose hosts…* to move existing hosts in (hosts from included files can be tagged with their file name). Hosts in other files stay local to that computer. Edits to `hosts.config` made outside SSHelter sync like any other edit, including deletions; if the file disappears or is emptied, SSHelter restores it from the chain.

*Forget* (in the Devices list) only removes a device from that list; a device that still has the phrase keeps syncing. If a device is lost, leave the chain, start a new one on the devices you keep, and rotate the keys it could see.

The relay stores only ciphertext and is open source (`relay/`). Builds made without a built-in relay ask for one first, so deploy your own to Cloudflare. The free Workers plan covers a few computers syncing a handful of spaces; Cloudflare's free daily limits (100,000 Durable Object requests and 100,000 rows written) are the ceiling:

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ysya/sshelter/tree/main/relay)

The button copies `relay/` into a new repository on your GitHub or GitLab account and deploys it to your Cloudflare account. When it finishes, enter the Worker's `https://…workers.dev` URL in *Settings → Sync → Relay URL* on every computer in the chain, then create or join. To deploy from a checkout instead: `cd relay && npm install && npx wrangler login && npx wrangler deploy`. To run it on your own server, use Docker Compose ([relay/README.md](relay/README.md#self-host-with-docker-compose)). To update a relay you deployed, see [Updating your relay](relay/README.md#updating-your-relay). Relay URLs must use `https://`, except `localhost` for development.

## Development

```

換成:

```markdown

## Sync

Sync is in beta. Open **Settings → Sync**. *Create* shows a 24-word sync code — store it in a password manager; it is the only secret, and anyone holding it can read and change your synced hosts. Any computer that already syncs can show it again. On another computer choose *Join with a sync code*, paste the words, and pick the spaces to sync there. SSHelter needs an existing SSH config: on a new machine create an empty `~/.ssh/config` first.

Synced hosts live in **spaces** — for example *Personal* and *Work* — and each computer chooses which spaces it syncs. Every synced space is one file in `~/.ssh/sshelter/`, and SSHelter keeps one `Include` line at the top of your main config that lists exactly those files, so plain `ssh` keeps working and the files survive uninstalling SSHelter. If an alias is in two spaces, ssh uses the space listed first and the sidebar marks the other copy. Use *Move hosts into a space* to move existing hosts in, or create one space per file (hosts from included files can be tagged with their file name). Hosts in other files stay local to that computer. Edits to a space's file made outside SSHelter sync like any other edit, including deletions; if the file disappears or is emptied, SSHelter restores it from the relay. Turning a space off removes only that computer's file; deleting a space removes it from every computer. Leaving the sync account moves that computer's space files to `~/.ssh/sshelter-local/`, where ssh keeps reading them as ordinary local files.

A synced host that brings `ProxyCommand`, `RemoteCommand`, `ForwardAgent`, `StrictHostKeyChecking` or another setting that runs programs, shares your credentials, environment or network, or relaxes host-key checks waits until you approve it on each computer. Synced hosts can't use `Include`, and their `HostName`, `User`, `HostKeyAlias` and `ProxyJump` must each be one word without characters a shell would interpret; SSHelter lists any host it can't sync, with the reason.

*Forget* (in the Devices list) only removes a computer from that list; a computer that still has the sync code keeps syncing. If a computer is lost, use *Change sync code*: the old code stops working, and each of your other computers asks for the new one (changes they had not uploaded yet are kept). Changing the sync code needs a relay that can freeze data; SSHelter tells you when yours needs an update first.

Computers that synced with SSHelter 0.16 upgrade by themselves: their synced hosts move into a space named *Synced*. Update SSHelter on all of them — a computer still on 0.16 does not see changes made after the upgrade.

The relay stores only ciphertext and is open source (`relay/`). Builds made without a built-in relay ask for one first, so deploy your own to Cloudflare. The free Workers plan covers a few computers syncing a handful of spaces; Cloudflare's free daily limits (100,000 Durable Object requests and 100,000 rows written) are the ceiling:

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/ysya/sshelter/tree/main/relay)

The button copies `relay/` into a new repository on your GitHub or GitLab account and deploys it to your Cloudflare account. When it finishes, enter the Worker's `https://…workers.dev` URL in *Settings → Sync → Relay URL* on every computer you sync, then create or join. To deploy from a checkout instead: `cd relay && npm install && npx wrangler login && npx wrangler deploy`. To run it on your own server, use Docker Compose ([relay/README.md](relay/README.md#self-host-with-docker-compose)). To update a relay you deployed, see [Updating your relay](relay/README.md#updating-your-relay). Relay URLs must use `https://`, except `localhost` for development.

## Development

```

- [ ] **Step 2: 新增手動驗證清單**

新增 `docs/superpowers/plans/2026-10-02-sync-v2-manual-verification.md`:

```markdown
# Sync v2 — manual two-computer verification (spaces, one sync code)

Run this before Sync v2 leaves beta (spec §11). It covers what the automated tests cannot: two real
computers, the OS keychain, the real relay, and the UI.

## Setup

- **Two computers, A and B**: two OS user accounts, a VM, or two machines. A second checkout with a
  custom config path is NOT a second computer — `~/.ssh/sshelter/`, `sync-state.json`, the device id
  and the keychain entries all follow the OS user. Name them "MacBook-A" and "MacBook-B" under
  Settings → Sync → This computer, so the messages below read the same.
- **Current relay**: `cd relay && npm install && npm run dev` (port 8787), reachable from both.
  Its console logs every request; items 13 and 14 read it.
- **Old relay** (item 11): `git worktree add ../sshelter-v0.16 v0.16.0`, then
  `cd ../sshelter-v0.16/relay && npm install && npx wrangler dev --port 8788`.
- **v0.16.0 builds** (item 1): install SSHelter 0.16.0 on both computers first. All other items
  use the build under test.
- Keep a terminal open on each computer for `cat ~/.ssh/config`, `ls ~/.ssh/sshelter` and
  `ssh -G <alias> | grep -i hostname`.

## Checklist

1. **Upgrade from v1.** With 0.16.0 on A and B: create a chain on A, join on B, move three hosts
   into sync, and add a synced host whose block contains `Include ~/.ssh/extra.config` (v1 allowed
   it). On A, also save a config file of your own as `~/.ssh/sshelter/mine.config` (`Host mine` with
   a HostName) and add `Include ~/.ssh/sshelter/mine.config` on the line below SSHelter's `Include`
   in `~/.ssh/config`. Install the build under test on A only and start it:
   - Settings → Sync shows "Upgrading sync" for a moment, then the joined pane without asking for
     the words again.
   - The "Sync was upgraded" dialog appears once: the space is named “Synced”, other computers must
     update, the `Include` host stays in `~/.ssh/sshelter-v1-kept.config`, and your own config file
     moved to `~/.ssh/sshelter-local/mine.config`, where ssh keeps reading it. After "Got it" it
     does not come back after a restart.
   - `~/.ssh/sshelter/` holds `synced-<8 hex>.config` with the three hosts; `hosts.config` is gone
     (a backup exists) and so is `mine.config`; the first non-comment line of `~/.ssh/config` is
     `Include ~/.ssh/sshelter/synced-<8 hex>.config`, followed by
     `Include ~/.ssh/sshelter-v1-kept.config`; your own line now reads
     `Include ~/.ssh/sshelter-local/mine.config`, and `ssh -G mine` still resolves its HostName.
   - The sidebar shows a “Synced” group with the cloud icon; double-clicking its header does not
     open the rename field.
   - B (still 0.16.0) does not see an edit made on A afterwards. Upgrade B: both end up in the same
     “Synced” space with all hosts, and neither shows "Sync replaced a local change".
2. **Leave, then create.** On A: Advanced → Leave… (without deleting); the dialog says the space
   files move to `~/.ssh/sshelter-local/`. Afterwards `~/.ssh/sshelter-local/` holds
   `synced-<8 hex>.config` (gone from `~/.ssh/sshelter/`), the main config's Include line points
   there at the same position, Settings → Sync shows "Your synced files are now local files" with
   the path (Dismiss removes it), and `ssh -G` still resolves those hosts. Create → the sync-code
   dialog cannot be closed with Esc, a click outside or a close button until "I have saved these
   words" is checked, while the pane behind it already shows the new account. Continue opens "Move
   hosts into a space" with “Personal” selected and every listed host preselected — including the
   hosts of the left account's file, which the sidebar shows as a local file. The new account's
   Include line sits above the local one. Account → Sync code → Show shows the same 24 words.
3. **Join and choose spaces.** A has “Personal” and “Work” (Spaces → New space…). On B: Leave the
   upgraded account, then paste A's new code as a numbered list → Join → "Choose spaces for this computer" lists both with "On
   MacBook-A", both checked. Uncheck Work → Sync 1 space → the wizard opens for Personal and says it
   is waiting for the first sync until A's hosts arrive (seconds, no reload). `~/.ssh/sshelter/`
   has only the Personal file and the Include line lists only it; `ssh -G` on B resolves a synced
   HostName. Joining with a wrong but valid code says "no sync account matches this sync code" and
   keeps the pasted words. If B still has `~/.ssh/sshelter/hosts.config` from a v1 chain it left
   long ago and `~/.ssh/config` still includes it (by name, or through a hand-written glob such as
   `Include ~/.ssh/sshelter/*.config`), joining moves it to `~/.ssh/sshelter-local/hosts.config`
   (ssh keeps reading it) and shows "Your synced files are now local files" with that path. With no
   Include reading it, joining leaves it where it is without that notice, and Settings → Sync lists
   it under "Files SSHelter doesn't use".
4. **Turn spaces on and off.** On B turn Work on → its file appears, the Include line lists both
   files, Work's hosts arrive, Devices on A shows "Personal and Work" for MacBook-B. Turn Work off →
   the confirmation names the file and says the space stays on the other computers → the file is
   gone from B (a backup exists), the Include line drops it, A still has Work with all hosts.
5. **Move hosts across spaces.** On A, drag a host from the Personal group onto the Work group →
   on B (with both spaces on) it moves from one file to the other, never existing in neither. Drag a
   host whose block contains `Include` into a space → refused with the reason; the same for a host
   whose `HostName` contains a shell character (for example `HostName web;id`). Open the wizard:
   "Can't be synced" lists both hosts with the backend's reasons; neither is selectable.
6. **One new space per file.** On A, with hosts in `~/.ssh/config.d/homelab.config` (give that
   file the sidebar label "Home lab"): wizard → Move into: One new space per file → each group shows
   "→ new space “…”" ("Home lab" for that file; a name that is taken gets " 2") → Move → the spaces
   exist, are on for A, and B lists them as off. With `Host web` in two of those files and both
   selected, web is sent once: the copy in the file the wizard lists first moves, the other stays
   local (the wizard then lists it under "Hosts defined in more than one file"), and no host fails
   with "listed in more than one group".
7. **Same alias in two spaces.** Put `Host web` in both Personal and Work (B has both on) → the
   sidebar marks the copy in the space that comes later in the Include line with the amber icon
   ("ssh uses web from Personal…"); its row menu offers "Keep this copy as web-local" (renames only
   that copy, and the rename syncs) and "Remove this copy".
8. **Approvals.** On B add `ProxyCommand nc %h %p` to a synced host:
   - A shows "web needs your approval" with Review; Settings → Sync shows "1 host waiting for your
     approval"; A's file is unchanged.
   - Review shows the whole block with the ProxyCommand line highlighted and "Adds ProxyCommand nc
     %h %p". Reject → A keeps its version and nothing is uploaded; edit that host on A → the edit
     reaches B normally.
   - On B change `Host web` to `Host web prod` (keeping the ProxyCommand) → A asks again with
     "Applies to: web → web prod"; Approve writes it to A's file.
   - Remove the ProxyCommand on B → A applies that without asking. Add `StrictHostKeyChecking no`
     on B → A asks again ("Adds StrictHostKeyChecking no").
   - Add `LocalForward *:5432 db:5432` on B → A asks ("Adds LocalForward *:5432 db:5432", the line
     highlighted); `LocalForward 5433 db:5432` (only a port) applies without asking.
   - In a text editor on B, put a right-to-left override (U+202E) after a `#` in a gated value
     (`ProxyCommand nc %h 22 #` + U+202E + `x`) → A's review shows it as `⟨U+202E⟩` and the line reads
     in its real order.
   - With three gated hosts in Work and Work off on A: turn Work on → one toast, and the review
     offers "Approve all (3)".
   - Open the review on A for a held host, then change that host's ProxyCommand again on B and let it
     sync while the review stays open → the review keeps showing the version you opened and says
     "Newer versions arrived while this was open". Approve → the newer version is not applied: "web
     changed since you opened this — review it again", and the review now shows the newer version.
9. **Change the sync code with an offline second computer.** B on, then cut B off from the relay
   (disconnect its network) and edit a synced host on B; quit B. On A: Account → Sync code →
   Change… → the confirmation says other computers need the new code and the relay must support
   freezing → Status shows "Changing code" with the step, then the notice "The sync code was
   changed" → Show new sync code → the dialog cannot be closed until "I have saved the new sync
   code" → the notice disappears. Show now returns the new words.
   - Reconnect B and start it: Status reads "Paused", the card says "The sync code was changed on
     MacBook-A…" and keeps B's offline edit pending. Enter the OLD code → refused ("that is the old
     sync code…"). Enter the code of an unrelated account (a throwaway one created on a third
     computer) → refused with "none of the spaces this computer syncs continue in that sync account;
     if it is the newest sync code, leave the sync account — your synced files stay as local files
     that ssh keeps reading — and join with it", and the words stay. Enter the new one → "Syncing
     again with the new sync code"; B's spaces and file names are unchanged; the offline edit
     reaches A.
   - On a third computer (or after Leave on B), Join with the old code → "this sync code was
     changed on MacBook-A; enter the new sync code".
   - Before entering the new code on a frozen computer, open Advanced → Leave… there: it never offers
     to delete the account and says the old sync account stays on the relay for the computers that
     still use the old code.
   - Devices on A: Forget copy says Forget does not lock a computer out and points to changing the
     sync code.
10. **Cancel.** Start Change… again and press Cancel while the row still offers it → the account
    is back to normal and Show returns the current code. If the steps run past "Freezing the old
    sync data" first, the row says it can no longer be cancelled and offers no Cancel.
11. **Relay without freeze or batch pull.** On a fresh account against the old relay
    (`http://127.0.0.1:8788`, set while not joined): Account → Relay reads "Older relay (no version
    reported)" and "This relay can be updated…", with "How to update" opening the README's
    "Updating your relay" section in the browser. Sync code → Change… is disabled with "Your relay
    can't change the sync code yet — update the relay first." Syncing still works between A and B
    (one request per space in the relay log). Update that relay in place —
    `git -C ../sshelter-v0.16 checkout --detach feat/sync-v2-spaces`, then restart its
    `npx wrangler dev --port 8788` (same `.wrangler/state`) → Check again → the hint and the block
    disappear, and the existing account keeps syncing.
12. **Rename blocked.** B has Work on, with file `work-<8 hex>.config`. On B create an empty
    `~/.ssh/sshelter/lab-<same 8 hex>.config`; Settings → Sync shows it under "Files SSHelter
    doesn't use". On A rename Work to "Lab" → B shows "The file of “Lab” keeps its old name" (toast
    and Settings → Sync), still uses `work-<8 hex>.config`, and `ssh -G` still works. Dismiss the
    notice → it does not come back on the following syncs while the empty file is still there.
    Delete the empty file on B → the next sync renames B's file and updates the Include line.
13. **Space deleted elsewhere.** On A delete Work (Delete… → "Delete “Work” everywhere?") → B shows
    "“Work” was deleted on MacBook-A", its Work file is gone (a backup exists) and the Include line
    drops it; the notice stays in Settings → Sync until dismissed.
14. **Focus and relay usage.** Watch the current relay's log: bringing SSHelter to the front runs
    exactly one sync round (one `POST /v1/pull`), never two; with the window in the background and
    no edits, rounds come about every 5 minutes instead of every 45 seconds.
15. **Leave and delete the account.** On B: Leave… offers no relay deletion ("Your other computers
    keep syncing") → B's space files move to `~/.ssh/sshelter-local/` and keep working. On A: Forget
    MacBook-B, then Leave… → "Also delete the sync account and every space from the relay" → after
    leaving, A's files are in `~/.ssh/sshelter-local/`, and joining with that code fails because the
    account is gone.
16. **A leave that cannot move the files.** While joined, edit `~/.ssh/config` in a text editor and
    do not reload in SSHelter; then Leave… → "Could not leave the sync account" with "could not keep
    this device's synced files as local files (…); nothing was changed — try leaving again": still
    joined, files and Include line untouched, nothing new in `~/.ssh/sshelter-local/`. Reload, Leave
    again → it works and keeps the outside edit.
17. **Leave during a sync code change.** A and B syncing on the current relay.
    - On A, stop the relay, then Change… → the row stays at "Sending this computer's changes… You can
      still cancel." Leave… → the dialog says leaving cancels the change first. Leave → A leaves.
      Start the relay again: the account is not frozen — B keeps syncing and never shows "Paused".
    - On B, Forget MacBook-A (B is now the last computer), then stall a change past the freeze:
      create spaces until one fails with "the relay is rate-limiting this device; try again later",
      then Change… → the row reads "Copying your spaces… Paused by the relay's hourly limit on new
      spaces; …" and says it can no longer be cancelled. Leave… stays available; its dialog says the
      change can no longer be cancelled and offers no relay deletion although B is the last computer.
      Leave → "Could not leave the sync account" with "a sync code change is in progress; let it
      finish (it resumes on its own) before this computer leaves"; B stays joined.
    - Delete SSHelter's keychain item `sync:mnemonic-next` (service "SSHelter") on B and Leave again →
      "Left the sync account on this computer" with "left the sync account on this computer, but its
      sync code change could not be finished (the new sync code was missing from the keychain), so
      the old sync account can no longer be joined — …"; B shows the not-joined pane with "Your
      synced files are now local files". On A, Join with the old code → "this sync code was changed
      on MacBook-B; enter the new sync code". Restart the relay with an empty `.wrangler/state` (or
      wait an hour) before using it again.
```

- [ ] **Step 3: 跑測試與建置**

Run: `pnpm exec vitest run --dir src`
Expected: PASS —— `Test Files  20 passed (20)`、`Tests  249 passed (249)`(README 的兩個連結測試仍通過)。

Run: `pnpm build`
Expected: `tsc` 沒有輸出,`vite build` 以 `✓ built in …` 結束(既有的「Some chunks are larger than 500 kB」警告不變)。

- [ ] **Step 4: Commit**

```bash
git add README.md docs/superpowers/plans/2026-10-02-sync-v2-manual-verification.md
git commit -m "docs: describe Sync v2 and add the two-computer checklist"
```

---

## 最終審查與修正(已執行)

七個 task 完成之後的整體審查(`.superpowers/sdd/2026-10-02-sync-v2-b4-frontend/final-review.md`,範圍 `169141c..e3ea14e`)結論是
「修正後可以合併」:沒有 Critical,五個 Important、六個 Minor。spec §4.3 先改寫成 ssh 實際合併同名 Host 區塊的方式(`52186dc`);
修正分兩次依序派出(後端 Part B 先,前端 Part A 後,避免同時 commit)。報告:`final-fix-report.md`。

- **Important 與處理**
  1. 同名主機的說明低估了風險:ssh 套用每一個符合的 Host 區塊,後面那份才設定的受管制設定(例如 ProxyCommand)照樣生效,
     可累加的設定兩份都用。審核對話框的 takeover 說明、側邊欄的 tooltip 與移除選項、精靈與 README 都改成這個說法,並列出會生效
     的受管制設定(`f55b0eb`;以 OpenSSH 10.3p1 的 `ssh -G` 驗證;解讀 4、6)。
  2. space 暫停、資料不見或上傳被拒時只有 Spaces 那一列看得到(比 v1 退步):狀態列與側邊欄的群組標題都會報出來(`b3ae59f`,
     解讀 9)。
  3. 手動清單第 8 項有一步會誤判失敗:照審查給的文字改(`e363a73`)。
  4. 同一個名稱在 space 檔與主 config 各有一份時,編輯、刪除、搬移、拖曳都作用在主 config 那一份,即使從同步的那一列操作:
     有歧義的名稱改以 (alias, 檔案) 選取,靠名稱找主機的動作都停用並說明,編輯器的位置改成唯讀的各份清單(`c8cc4a8`,解讀 4)。
     以 (alias, 檔案) 定位的後端編輯留給 SP2。
  5. 側邊欄的「Remove this copy」按一下就執行,而另一個 space 的那份刪掉會同步到那個 space 的每一台:兩個修正都先確認、說明
     會影響誰,完成後 toast;一般的「Remove…」也說明 space 裡的主機會在每一台被刪除(`3345e24`)。
- **Minor**:核准 / 拒絕在帳戶鎖住時停用並說明(`489262f`,解讀 11);離開之後只保留仍然成立的提示、其他改成中性文字,離開的
  對話框說明還沒上傳的修改(`eaad54a`,解讀 7、14);一般「Remove…」的同步說明(同 Important 5);後端接受部分 C1 控制字元與
  格式字元由 Part B 處理;app 內的 relay 連結指向 main(main 的 relay 還是 v2 以前的版本)記為發行前的注意事項 —— 這個 branch
  合併進 main 之前不發行。
- **Part B,後端拒絕與對話框顯示對齊**(`7b5d1c9`、`ee28dfc`):`hosts_file.rs` 在 OpenSSH 註解以外拒絕 tab 以外的每一個控制
  字元(C0、DEL、C1)與每一個 Cf 格式字元(21 個區段、170 個 code point,Unicode 16 與 17 相同),註解與 CRLF 照常;
  `src/lib/hidden-parity.test.ts` 讀 `hosts_file.rs` 的五類拒絕,確認 `revealHidden` 會顯示其中每一個 code point(約 4,433 個),
  `revealHidden` 也顯示 U+303F。Rust 746 → 749;`src/` 22 個檔案、305 個測試。
- **Part A,前端**(`b7a1b99`…`bea36e3`,17 個 commit):上面的 Important 1、2、4、5 與 Minor,加上各 task 留到這裡的項目
  (T1-a、T2-a…T2-j、T3-a…T3-h、T4-a、T4-b、T5-a、T5-c、T6-a,內容見各 task 開頭的說明),以及 HostList、Sync pane 的 server
  render 測試(`2e3786d`)。`src/` 28 個檔案、412 個測試。
- **合併 main**(`ba0461b`):帶進 main 上的 IME 修正(`aa41d31`:共用的 `isImeKey`,既有的 Enter handler 在 IME 組字時不送出,
  與一個掃描所有 Enter handler 的測試)。`src/` 29 個檔案、413 個測試。
- **殘留修正**(整體複審 `final-rereview.md` 的八個 Minor 與一個範圍外的項目,`1d6994a`…`5b55188`):各份清單只指向真的有的
  動作(後端只列 Host 行以這個名稱開頭、在別的檔案的那份;其他的請用文字編輯器改那個檔案,再 Reload from disk),也只在知道的
  地方說讀取順序,同步總覽未知時一律當成有歧義(`1d6994a`);離開的未上傳數只算勾選的 space 裡的主機修改(`db1a260`);
  ⌘/Shift 點擊有歧義的列不清掉已勾選的列(`1c2fbb4`);審核對話框上方任何內容出現、消失或改變時都啟動點擊保護(`80c6a26`);
  別台取的 space 名稱在每個顯示的地方經 `revealHidden`(`3d65c6e`);過時的註解、手動清單與 README(`056dd6e`、`8a9cd02`、
  `841ba8d`、`5b55188`)。

結束時(HEAD `5b55188`):`./node_modules/.bin/vitest run --dir src` 29 個檔案、466 個測試全過,`tsc --noEmit` 與 `vite build`
成功,Rust 749 個測試全過(略過兩個 keychain 測試)。執行中多出的檔案:`src/components/DuplicateCopies.tsx`、`ShadowFixDialog.tsx`、
`src/lib/sync-labels.ts`、`use-last-non-null.ts`、`use-now.ts`、`ime.ts`(分支與 main 各自加入,合併時合在一起);測試
`SyncApprovalDialog.test.tsx`、`SyncMigrationDialog.test.tsx`、`SyncPane.test.tsx`、`HostList.test.tsx`、`DuplicateCopies.test.tsx`、
`src/lib/hidden-parity.test.ts`、`ime.test.ts`、`src/stores/ui.test.ts`,以及來自 main 的 `enter-handlers.test.ts`。決定不做的:
hook 層級的測試(沒有 DOM 測試環境,也不加相依);萬用字元涵蓋的接管(`Host prod*` 與 `prod1`)、私用區 / 未指派 / 外觀相似的
字元;空的 space 在側邊欄沒有群組標題;以任何 pattern 解析 alias(同 v1 與拖曳);keychain 上鎖時後端會拒絕決定,但總覽看不
出來。

---

## Self-review(已執行)

- **Spec 覆蓋**:§8 Settings → Sync 的帳戶(同步碼 Show / Change、relay URL 與版本、「relay 可以更新」、裝置清單含
  勾選的 space 與 Forget)→ Task 2;Spaces(名稱、檔名、開關、主機數、改名、刪除、New space)→ Task 3;等待核准
  (清單與審核對話框、逐台或全部)→ Task 4;更換同步碼(確認 → 進度 → 顯示新碼)→ Task 2;未加入(加入或建立、加入後
  勾選 space)→ Task 2、3;側邊欄(每個 space 一組、已同步標記、跨 space 同名列尾標示)→ Task 5;搬移精靈(目標
  space、一個來源檔一個 space、不能搬的區塊與原因)→ Task 6;v1 升級的一次性說明 → Task 1。§7.2 的跨 space 搬移走
  側邊欄拖曳(Task 5,後端規則)與精靈(Task 6);§7.3 建立 / 加入 / 離開 / 刪除帳戶 / relay URL → Task 2;§7.4 核准
  → Task 4;§7.5 更換、取消、其他電腦重新輸入、兩台同時更換的提示 → Task 2(`frozenMessage`、`noticeRows`、
  `other_rotation`;更換期間的離開 `leaveRotationNote`、`deleteAccountNote`,這台已經離開之後的錯誤標題 `leaveFailureTitle`
  在 Task 1);§7.3 離開後的檔案(B3 A1 的 `left_account`)→ Task 1(文案)與 Task 2(未加入時的提示、離開的
  文案);§7.6 → Task 1(說明)與 Task 2(升級外殼)。§9:沒有批次端點 / 沒有 freeze → Task 2
  `relayDetails`、`changeCodeBlocker`;`429`、批次 `5xx`、帳戶 chain `404`、v1 升級失敗、keychain 讀取失敗、狀態寫不進
  磁碟 → 後端的 `last_error` 顯示在狀態列(Task 2);push `409 frozen` → 輸入新同步碼(Task 2);space chain `404`(帳戶
  仍有它)→ 重建 / 刪除(Task 3),(已 tombstone)→ `space_deleted` 提示(Task 1、2);單一 space 失敗 → space 列的錯誤
  (Task 3);等待核准中又有新版本 → 清單隨事件重新查詢(Task 4);改名的目標檔已存在 → `rename_blocked` 提示(Task 1、2);
  目錄裡不在 Include 上的檔案 → 「Files SSHelter doesn't use」(Task 2)。§10 前端測試:space 清單與勾選(Task 3)、
  核准對話框(Task 4)、側邊欄分組與同名標示(Task 5)、搬移精靈(Task 6)、更換同步碼的流程狀態(Task 2
  `statusLine` / `rotationLabel` / `changeCodeBlocker` / `frozenMessage` / `noticeRows`);手動清單 → Task 7。
- **使用者指定的限制**:同步碼只在元件 state(Task 1 的直接呼叫、Task 2 的對話框);移除焦點 → `sync_now`(Task 1;
  spec §6.4 的退避也因此不被繞過);
  Forget / 關掉 space / 刪除 space / 更換同步碼 / 批次查詢提示的文案(Task 2、3);刪除兩個 binding(Task 6);手動清單
  (Task 7);不新增相依。
- **Placeholder 掃描**:每個程式步驟都是完整檔案或精確的「把 … 換成 …」;沒有 TBD、TODO 或「同 Task N」。
- **型別一致**:`SyncMessage`(Task 1)→ `approvalMessage`(Task 4);`Tone`、`plural`(Task 2)→ Task 3、4、6;
  `TONE_TEXT`(Task 2)→ Task 3;`MAX_SPACE_NAME`(Task 3)→ Task 6;`spaceFileLabels`(Task 5)→ Task 6;
  `syncApprovalsOpen`(Task 4)、`syncMigration`(Task 6)在 store、元件與事件裡同名;command 名稱與參數照 B4 handoff。
- **Review Focus**:五項各有測試,寫在對應 task 裡(見上方清單的測試名稱)。
- **驗證**:Task 1 已在 repo 執行(`2def6a2`、`32cae39`;之後 `src/` 的測試 181 個)。Task 2–7 依序套用在 repo `32cae39` 的
  拷貝上;每個 task 先確認新測試失敗(Task 2、3、5:缺模組;Task 4:缺模組 + 6 個事件測試;Task 6:15 failed),再確認
  `vitest run --dir src` 與 `tsc --noEmit` 全綠(181 → 207 → 214 → 235 → 239 → 249 → 249);Task 7 之後 `tsc && vite build`
  成功。另把 Task 2–7 的程式碼區塊照步驟套用到 `32cae39` 的乾淨拷貝,每個 task 結束時與驗證過的樹逐位元組相同。
  沒有在真的 Tauri 視窗裡操作(元件沒有自動化測試,
  畫面行為由手動清單涵蓋)。這是執行前的檢查;執行後的結果見「最終審查與修正(已執行)」。
