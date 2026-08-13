# Sidebar round 2 —「管得動」設計

日期:2026-08-13。第一輪(v0.9.0)已出貨;本輪做使用者核可路線的後三項:
hover ⋯ 操作鈕、多選批次、跨檔案拖曳搬移。純前端,復用既有 mutations
(`useMoveHost`/`useRemoveHost`/`useSetTags`),後端零改動。

## 5. 列尾 hover ⋯ 按鈕

右鍵選單的發現性問題:看不見的功能等於不存在。解法是把同一組動作
同時放在 hover 浮現的 ⋯ 按鈕裡(現代清單 UI 慣例;經典管理器沒有 hover
按鈕,但我們的 Play 鈕已定調 hover 模式)。

- HostRow 的 hover 疊層從一顆(▶ right-1)變兩顆:**⋯ 在 right-7、▶ 在
  right-1**;hover 預留空間 `pr-8` 改 `pr-14`。
- ⋯ 開 DropdownMenu(Radix ContextMenu 無法由按鈕觸發,兩個 primitive
  各渲染一份同內容選單),右鍵選單同步擴充,兩邊等價:
  - Connect
  - Deploy key…
  - Move to file ▸(其他已載入檔案的子選單;僅一個檔案時整項隱藏)
  - ───
  - Remove…(AlertDialog 確認後 `useRemoveHost`;確認文案含 alias)
- defaults(wildcard)列維持無選單、無 ⋯。
- 選單項目以一個小 helper 產生兩套(ContextMenu*/DropdownMenu*),避免
  行為漂移。

## 6. 多選批次

- 選取模型:HostList 本地 state `checked: Set<string>`(alias);與編輯器的
  `selectedAlias` 完全獨立。
  - **⌘/Ctrl+click** 列 → 切換該列勾選(不改變編輯器選取)。
  - **Shift+click** → 從「最後一次點擊的列」到本列的**可見順序**區間全選
    (跨群組亦可)。區間計算抽純函式 `selection-range.ts`(TDD):
    `rangeBetween(visible: string[], anchor: string | null, target: string): string[]`
    —— anchor 不在 visible 時退化為只選 target。
  - 一般 click 維持原行為(選給編輯器)且**清空**勾選;Esc 清空勾選。
  - tag 模式下同一 host 可能出現兩列 —— 勾選以 alias 為準,兩列同步顯示。
- 視覺:勾選列加 `bg-primary/8` + 左緣 2px accent;數量顯示在底部列。
- 底部動作列(勾選 >0 時浮現,sidebar 底部 sticky):
  `N selected · [Move to ▾] [Add tag] [Remove…] [✕]`
  - **Move to ▾**:DropdownMenu 列其他檔案 → 對每個 alias 依序
    `moveHost.mutateAsync`(已在目標檔者跳過),完成 toast 摘要;逐台寫入
    (各自 backup)是既有後端語意,spec 接受此成本。
  - **Add tag**:Popover/inline input 收 tag 名 → 對每台
    `setTags(alias, dedupe([...現有, tag]))`(現有 tags 來自 HostSummary)。
  - **Remove…**:AlertDialog 確認(列出數量與前幾個 alias)→ 逐台
    `removeHost.mutateAsync`。
  - 動作完成後清空勾選。
- 搜尋中允許勾選(過濾不影響已勾選集合;動作以集合為準)。

## 7. 拖曳跨檔案搬移(檔案模式限定)

- 現況:跨群組拖曳顯示 no-drop。改為:拖到**另一個檔案群組**的列或標頭上
  → 該群組整體高亮(標頭 `bg-primary/10` + 群組容器 ring-1 ring-primary/40,
  不顯示插入線 —— 位置語意是「搬到該檔案末尾」,與編輯器 Move to file 相同,
  由 `config_move_host` 決定)。
- Drop → `moveHost.mutate({ alias, targetFile })`;成功 toast
  `Moved <alias> → <file label>`。
- 同檔案內拖曳 = 原有排序行為,完全不變。
- tag 模式與搜尋中維持整體停用拖曳(round 1 既有規則)。
- 多選拖曳(拖整批)不在本輪 —— 底部動作列已涵蓋批次搬移。

## 驗收

- vitest:`selection-range` 全案例(正反向、anchor 缺席、anchor==target、
  單元素);既有 115 測試不退。
- 手動:⋯ 選單四動作、右鍵等價、多選三批次動作與 Esc/一般點擊清空、
  跨檔案拖曳高亮與搬移、tag 模式下拖曳仍停用。
- `pnpm build` + `cargo test` 綠;README 功能列表補批次管理。
