# Sidebar round 2 — 實作計畫

Spec:`docs/superpowers/specs/2026-08-13-sidebar-round2-design.md`。
純前端;每 task 各自 commit;全程 `pnpm test`+`pnpm build` 綠。

### Task 1: 列選單擴充(⋯ + 右鍵等價)
- [ ] HostRow 動作模型改為傳入 host 級 handlers(connect/deploy/moveTo/remove)
      + `otherFiles` 標籤資料;helper 同時產 ContextMenu 與 DropdownMenu 兩套項目
- [ ] hover 疊層加 ⋯(right-7),`pr-8` → `pr-14`;Remove 走 AlertDialog 確認
- [ ] Commit: `feat(host-list): hover actions menu with move and remove`

### Task 2: `selection-range.ts`(TDD)+ 多選批次
- [ ] `rangeBetween` 純函式 + 測試(正反向、anchor 缺席、同一列、單元素)
- [ ] HostList `checked` Set;⌘/Ctrl 切換、Shift 區間、一般點擊/Esc 清空;
      勾選視覺(bg-primary/8 + 左緣 accent)
- [ ] 底部 sticky 動作列:Move to ▾ / Add tag / Remove… / ✕;逐台 mutateAsync +
      toast 摘要;完成清空
- [ ] Commit: `feat(host-list): multi-select with batch move, tag and remove`

### Task 3: 跨檔案拖曳搬移
- [ ] 檔案模式下跨群組 dragOver → 目標群組高亮(標頭+容器),drop →
      `moveHost`;同檔案排序不變;tag 模式/搜尋中仍停用
- [ ] Commit: `feat(host-list): drag a host onto another file group to move it`

### Task 4: 收尾
- [ ] README 功能列表補批次管理與拖曳搬移;全套測試;push;release PR 確認
