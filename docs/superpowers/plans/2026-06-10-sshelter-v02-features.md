# SSHelter — v0.2 功能批次實作計畫（市場研究導出）

> REQUIRED SUB-SKILL: superpowers:subagent-driven-development。依據 2026-06-10 市場研究（競品掃描 + 社群痛點挖掘）排定的五項功能。終端機實際開啟、tray 選單、Tailscale 整合需真機驗證；其餘全部 headless 可測。

**Goal:** 把 SSHelter 從「偶爾開的設定編輯器」升級成「每日使用的 SSH 入口」：快速連線（palette + 選單列）、金鑰衛生助手、config 智慧套件（ssh -G 生效預覽／lint／ProxyJump 鏈）、主機探索（known_hosts + Tailscale）、備份歷史與還原。

**研究結論對應：**
| 功能 | 證據 |
|---|---|
| 快速連線 palette + 選單列 | 社群第一名許願（hejki 選單列、Raycast SSH 擴充存在即證明）；把 app 變 daily driver |
| 金鑰衛生助手 | 「Too many authentication failures」= 最高頻實際痛點；GUI 圈無人做 |
| config 智慧套件 | shadow/first-match-wins 是知名 footgun；`ssh -G` 是官方除錯法但無 GUI 呈現；建立在現成無損 parser 上的獨家差異化 |
| 主機探索 | XPipe 最被稱讚的就是 discovery；解新機 onboarding |
| 備份歷史/還原 | 無損 diff + 本地版本化 = 對 Termius 雲端帳號反彈的正面回應 |

---

## 模組與命令契約

### D1 — `connect` 模組 + tray 選單列（Rust）
- `src-tauri/src/connect.rs`：
  - alias 驗證：`^[A-Za-z0-9._@%-]+$` **且**必須存在於已載入 doc 的 patterns。
  - `TerminalKind { MacTerminal, ITerm2, LinuxCustom(String), … }`；`detect_terminals() -> Vec<TerminalInfo{id,label}>`（macOS：Terminal.app 恆在、iTerm2 偵測 /Applications；Linux：$TERMINAL + 探測表 ptyxis/gnome-terminal/konsole/kitty/alacritty/wezterm/foot/xterm）。
  - `build_launch(terminal, alias) -> LaunchSpec{program, args}`（**純函式、可單元測試**：macOS 產 osascript AppleScript、引號跳脫；Linux 產 per-emulator argv；絕不走 sh -c）。
  - `launch(spec)`：spawn、繼承環境。
- 命令：`connect_launch(alias, terminal_override: Option<String>)`、`connect_list_terminals()`。
- Tray（`src-tauri/src/tray.rs`）：`tauri` 加 `tray-icon` feature；選單 = 最多 25 台主機（依清單序）+ 分隔線 + Open SSHelter + Quit；`config_load` 成功後重建選單（命令注入 `AppHandle`）；menu event `connect:<alias>` → `connect::launch`。
- 測試：alias 驗證、AppleScript 跳脫、各終端機 argv 表、tray 選單建構函式（純函式部分）。實際 spawn/tray 顯示＝真機驗收。

### D2 — config 智慧套件 + 金鑰衛生（Rust）
- `src-tauri/src/config/intel.rs`：
  - `effective_config(alias, config_path: Option<&Path>) -> Vec<(String,String)>`：跑 `ssh -G [-F path] alias`（alias 先驗證），解析 `key value` 行（重複 key 如 identityfile 保留多值）。**測試用 `-F` 指向 temp config，headless 完全可測**（`ssh -G` 不連線）。
  - `lint(doc) -> Vec<LintIssue{severity: error|warning|info, file, alias: Option, keyword: Option, message}>`，規則：
    1. 區塊內重複指令（多值 key 白名單豁免：identityfile/localforward/remoteforward/dynamicforward/certificatefile/sendenv/setenv）→ warning「後面那行無效」。
    2. 跨區塊/檔重複 alias（後者會被 shadow）→ warning。
    3. IdentityFile 路徑（~ 展開後）不存在 → error。
    4. `StrictHostKeyChecking no` → warning（安全）。
    5. ProxyJump 值不含 `@`/`:`/`.` 且不匹配任何 alias → warning「引用未定義主機」。
  - `jump_chain(doc, alias) -> Vec<ChainNode{name, defined: bool}>`：解析（含 effective 的）ProxyJump、逗號鏈、遞迴一層 hop 自己的 ProxyJump，cycle guard、深度上限 5。
  - `key_hygiene(doc, alias) -> KeyHygiene{ identity_files: Vec<{path, exists}>, identities_only: bool, explicit: bool /* 任一匹配區塊有設 IdentityFile，非 ssh 預設 id_* 清單 */ }`（resolution 以 `ssh -G` 為準）。
- 命令：`config_effective(alias)`、`config_lint()`、`config_jump_chain(alias)`、`config_key_hygiene(alias)`。DTO 走 ts-rs。
- 測試：lint 每條規則正反例（fixtures）、ssh -G 解析（-F temp config）、chain（含 cycle）、hygiene（explicit vs default）。

### D3 — 探索 + 備份（Rust）
- `src-tauri/src/discover.rs`：
  - `parse_known_hosts(text) -> Vec<KnownHostEntry{host, port}>`（跳過 hashed `|1|`、@marker、註解；`[host]:port` 形式）。
  - `parse_tailscale_status(json) -> Vec<TailscalePeer{host_name, dns_name, online}>`（純函式吃 JSON 字串，測試餵 fixture）。
  - `discover(doc) -> Vec<Suggestion{name, host_name, port, source: "known_hosts"|"tailscale", online: Option<bool>}>`：過濾已在 config（alias 或 HostName 比對）；tailscale binary 找不到就跳過。
- `config/commands.rs` 增：`discover_hosts()`、`config_list_backups() -> Vec<BackupInfo{path, file, timestamp_ms}>`（掃管理中檔案旁的 `<name>.<millis>.bak`）、`config_restore_backup(backup_path)`（**安全驗證**：canonicalize 後必須是受管檔案同目錄、檔名符 `.bak` 模式；還原前先備份現行；atomic_write；之後前端 reload）。
- 測試：known_hosts/tailscale 解析、discover 過濾、backups 列表/還原（tempdir）、restore 的路徑驗證拒絕任意路徑。

### D4 — 前端：palette + 連線（React）
- shadcn `command`（cmdk）。`⌘K` 開啟：主機清單（fuzzy）——**Enter＝連線**（toast）、**⌘Enter＝在編輯器開啟**；動作群組：New host、Toggle theme、Copy ssh command。
- 編輯器標頭 + 清單列 hover 加「連線」按鈕（terminal icon）。
- 終端機選擇：設定下拉（`connect_list_terminals`），localStorage 持久化，作為 `terminal_override` 傳入。
- `useConnect()`、`useTerminals()` hooks。

### D5 — 前端：智慧套件 + 探索 + 備份 UI（React）
- 編輯器：Authentication 區塊內嵌**金鑰衛生卡**（resolved key + exists/explicit/IdentitiesOnly 徽章 + 一鍵「Set IdentitiesOnly yes」走既有 save）；**Effective config** 區段（ssh -G 結果，與 host 區塊值差異標示可後補）；Forwarding 區塊上方 **ProxyJump 鏈**（laptop → hop → target，未定義 hop 標紅）。
- 側欄列 + 編輯器標頭：lint 徽章（error/warning 計數）；點開 issues 清單（popover 或編輯器頂部摺疊區）。
- 工具列：**Discover** 按鈕（dialog：建議清單 + 來源徽章 + 選目標檔一鍵加入）；**History** 按鈕（dialog：備份列表 + Restore 確認）。

### 驗收
- 全部：`cargo test` 綠（新測試 ≥ 25）、`pnpm build` 綠、vitest 綠、兩段式 review（D1/D2/D3 安全關鍵）。
- headless 截圖：palette 開啟、編輯器含 hygiene/effective/chain/lint、discover dialog。
- 真機驗收（使用者）：實際開終端機連線、tray 選單、Tailscale 探索。
