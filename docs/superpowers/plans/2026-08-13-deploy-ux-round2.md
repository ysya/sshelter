# Deploy UX round 2 — 實作計畫

Spec:`docs/superpowers/specs/2026-08-13-deploy-ux-round2-design.md`。
純前端;每個 task 各自 commit;全程 `pnpm test` + `pnpm build` 綠燈。

### Task 1: `identity-file.ts` 純函式(TDD)
- [ ] `src/lib/identity-file.test.ts`:toTildeSshPath(含 `/.ssh/` 轉換、其餘原樣)、
      identityFileAction(空→write、絕對相等→already、`~/` 字尾配中→already、
      不同把→offer、字尾不可誤配 notwork)
- [ ] `src/lib/identity-file.ts` 實作
- [ ] Commit: `feat(deploy): add identity-file write-back decision helpers`

### Task 2: DeployKeyDialog 寫回 IdentityFile
- [ ] 結果階段依 action 分支:write 自動存 + toast + invalidate keyHygiene;
      already 註記;offer 顯示「改用這把金鑰」按鈕
- [ ] Commit: `feat(deploy): write IdentityFile back after a successful deploy`

### Task 3: 四個入口
- [ ] HostActions 可見按鈕(non-wildcard)
- [ ] CommandPalette:selectedAlias 存在時的「Deploy key to <host>」
- [ ] HostIntelligence key hygiene Section action
- [ ] KeysDialog:per-key「在 app 內部署」+ ui store `deployKeyInitialPub`
      + DeployKeyDialog 預選採用/關閉清空
- [ ] Commit: `feat(deploy): add editor, palette, hygiene and keys-dialog entry points`

### Task 4: IdentityFile 選檔控制
- [ ] FieldControl 特例 → `IdentityFileControl`(金鑰下拉 lazy + plugin-dialog Browse,
      setValue shouldDirty)
- [ ] Commit: `feat(host-editor): pick IdentityFile from detected keys or a file dialog`

### Task 5: 關閉自動修正
- [ ] input/textarea/command 基底加 autoCorrect/autoCapitalize/spellCheck 屬性;
      grep 裸 `<input`/`<textarea`
- [ ] Commit: `fix(ui): disable autocorrect and autocapitalize on all text inputs`

### Task 6: 收尾
- [ ] `pnpm test && pnpm build` 全綠;push main;release-please PR 版本確認
