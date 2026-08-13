# Sidebar round 1 —「找得快、看得懂」設計

日期:2026-08-13。依據:三份業界研究(Termius/Tabby、Royal TSX/SecureCRT/MobaXterm、
Core Shell/SSH Config Editor/VS Code/JetBrains/Warp/iTerm2)+ 使用者核可的兩輪路線。
本 spec 只涵蓋第一輪四項;第二輪(hover ⋯、多選批次、跨檔案拖曳)另立 spec。

範圍:純前端。`HostSummary` 已帶 `tags`,免動後端。

## 1. 搜尋前綴語法(`src/lib/host-filter.ts`,TDD)

把 `HostList.tsx` 內的 `matches()` 抽成純函式模組並升級:

- `parseQuery(q)`:以空白切 token;`#x` → tag 條件、`@x` → user 條件,其餘為自由
  文字。裸前綴(單獨 `#` 或 `@`)視為未完成輸入,忽略該 token。
  `:port` 不做 —— `HostSummary` 沒有 port 欄位,等 summary 加欄位後再升級,
  不用 hostname 子字串魚目混珠。
- `hostMatches(host, parsed)`:所有條件 **AND**;`#x` 對 tags、`@x` 對 user,均為
  不分大小寫子字串比對。自由文字的 hay 擴充為
  alias + patterns + tags + source_file + **hostname + user**(現況漏了後兩者)。
- 搜尋框 placeholder 改為 `Search…  #tag @user`。
- Termius/Royal 慣例「搜尋時強制展開群組、空群組消失」既有行為保留。

## 2. 分組模式:依檔案 / 依 tag

- ui store 加 `groupMode: "file" | "tag"`(隨 `sshelter-ui` 持久化,與 fileScope 同級)。
- 切換控制:搜尋框下那排(file-scope Select 旁)加兩顆 size-6 icon toggle
  (FileText=依檔案、Tags=依 tag),active 者高亮。
- tag 模式的 sections:
  - host 依其每個 tag 各出現一次(多 tag = 多處出現,Gmail label 模型);
    無 tag 的歸尾端「Untagged」群組。
  - 群組排序:tag 字母序,Untagged 恆末。
  - 摺疊狀態沿用 `collapsedGroups`,tag 群組的 key 用 `tag:<name>` 前綴避免與
    檔案路徑相撞。
  - wildcard Defaults 區塊只在檔案模式顯示(它是 config 結構,不是 host)。
  - 拖曳排序停用(與搜尋中相同理由:過濾後順序無意義)。
  - 群組標頭:tag 名 + 數量;無右鍵選單、無雙擊改名(round 1 不做 tag 改名)。
  - fileScope 仍先套用(scope 到單檔後再依 tag 分組)。
- 檔案模式行為完全不變。

## 3. 列上 tag chips

- settings store 加 `showHostTags: boolean`(預設 true)+ Settings 開關
  (Appearance 區,label "Show tags in host list")。
- 檔案模式且 `showHostTags` 時:alias 之後、次要資訊之前,顯示至多 2 個
  tag chip(`bg-muted` 圓角、`text-[0.625rem]` mono、truncate),超過 2 個以
  `+n` 表示;整列 title tooltip 帶完整 tag 列表。
- tag 模式下不顯示 chips(群組即 tag,重複資訊)。
- 佈局約束:chips 位於 flex 中段、`shrink-0`,alias 與次要資訊照舊 truncate,
  不得把列高撐破 28px。

## 4. ⌘K 最近連線優先

- settings store 加 `recentConnections: Record<string, number>`(alias → epoch ms)
  與 `recordConnection(alias)`(寫入當下時間,僅保留最近 20 筆)。
- `useConnect` 成功時呼叫 `recordConnection`(queries.ts 內以
  `useSettingsStore.getState()` 呼叫,不引入 hook 依賴)。
- CommandPalette:輸入框改受控;query 為空時,清單頂部插入「Recent」群組 ——
  依時間戳新→舊取前 5、且 alias 仍存在於目前 hosts;query 非空時不渲染
  (避免與 Hosts 群組重複命中)。項目行為與 Hosts 群組相同(Enter 連線、
  ⌘Enter 編輯),value 加 `recent:` 前綴保持唯一。
- host 改名/刪除造成的殘留 alias:渲染時過濾不存在者即可,不主動清理。

## 驗收

- vitest:host-filter 全組合(前綴、AND、大小寫、裸前綴忽略、hostname/user 自由文字)。
- 手動:切分組模式、摺疊 tag 群組後重啟保留、chips 顯示與設定開關、
  ⌘K Recent 出現且搜尋時消失。
- 既有測試不退;`pnpm build` 綠。
