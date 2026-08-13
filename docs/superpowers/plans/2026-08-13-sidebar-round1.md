# Sidebar round 1 — 實作計畫

Spec:`docs/superpowers/specs/2026-08-13-sidebar-round1-design.md`。
純前端;每 task 各自 commit;全程 `pnpm test`+`pnpm build` 綠。

### Task 1: `host-filter.ts`(TDD)
- [ ] `src/lib/host-filter.test.ts`:自由文字(含 hostname/user)、`#tag`、`@user`、
      AND 組合、大小寫、裸 `#`/`@` 忽略、空字串全通過
- [ ] `src/lib/host-filter.ts` 實作 parseQuery + hostMatches;HostList 改用之並刪內部 matches
- [ ] placeholder 改 `Search…  #tag @user`
- [ ] Commit: `feat(host-list): add #tag @user search prefixes`

### Task 2: 分組模式(依檔案/依 tag)
- [ ] ui store `groupMode`(persist partialize 加入)
- [ ] header 兩顆 icon toggle(FileText/Tags)
- [ ] tag 模式 sections(多 tag 多次出現、Untagged 恆末、`tag:` collapse key、
      無 Defaults、停用拖曳、標頭無選單)
- [ ] Commit: `feat(host-list): group hosts by file or by tag`

### Task 3: tag chips
- [ ] settings store `showHostTags`(預設 true)+ SettingsDialog 開關
- [ ] HostRow 檔案模式顯示 ≤2 chips + `+n`,tooltip 全列;tag 模式抑制
- [ ] Commit: `feat(host-list): show tag chips on host rows`

### Task 4: ⌘K Recent
- [ ] settings store `recentConnections` + `recordConnection`(cap 20)
- [ ] `useConnect` onSuccess 記錄
- [ ] CommandPalette 受控輸入;query 空時頂部 Recent 群組(前 5、`recent:` value)
- [ ] Commit: `feat(palette): surface recent connections first`

### Task 5: 收尾
- [ ] 全套測試 + build;更新 README 功能列表(分組/搜尋語法/Recent)
- [ ] push main;確認 release-please PR 內容
