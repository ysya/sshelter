# Deploy UX round 2 — 設計

日期:2026-08-13。來源:使用者對 v0.7.0 in-app 部署的四項回饋。已與使用者確認的決策:
寫入行為=「沒有才自動寫入」、入口=四處全加、選檔=「金鑰下拉 + Browse…」。

範圍:**純前端**(TypeScript/React),後端 Rust 零改動。

## A. 部署成功後寫回 IdentityFile

問題:部署成功後 host 的 config 沒有 IdentityFile,非預設命名的金鑰下次連線仍用不到。

- 新純函式模組 `src/lib/identity-file.ts`(vitest 覆蓋):
  - `toTildeSshPath(abs)`:路徑含 `/.ssh/` 時轉成 `~/.ssh/<其後>`,否則原樣回傳。
  - `identityFileAction(existing: string[], deployedPrivateAbs: string) → "write" | "already" | "offer"`:
    `existing` 為空 → `write`;任一項指向部署那把(絕對路徑相等,或 `~/` 項的字尾
    比對 —— 與 `deploy-key-select` 同規則)→ `already`;否則 → `offer`。
- `DeployKeyDialog`:結果為 `added` / `alreadyPresent` 時,以 `useKeyHygiene` 的
  `identity_files` 判斷:
  - `write` → 自動 `config_save_host(alias, [{IdentityFile, ~/.ssh/<key>}])`,toast
    告知,結果畫面註明「已寫入」,並 invalidate `keyHygiene(alias)`。
  - `already` → 結果畫面註明「config 已指向這把金鑰」。
  - `offer` → 不自動改;結果畫面顯示「改用這把金鑰」按鈕,按下才寫
    (`set_host_field` 語意:改第一個 IdentityFile 指令,其餘行不動)。

## B. 部署入口 ×4

1. **HostEditor 標頭**:`HostActions` 加可見的 Deploy key 按鈕(Upload 圖示 +
   tooltip),wildcard host 不顯示。
2. **⌘K palette**:有 `selectedAlias` 時,Actions 群組出現「Deploy key to <host>」。
3. **Key hygiene 區塊**:`HostIntelligence` 金鑰健檢 Section 的 action 槽加
   「Deploy key…」小按鈕。
4. **Keys dialog**:每把有 `.pub` 的金鑰加「在 app 內部署」按鈕,沿用現有的
   選 host 面板;選定後關閉 Keys dialog、開啟部署對話框並**預選該把金鑰**。
   ui store 新增 `deployKeyInitialPub: string | null`;`DeployKeyDialog` 預選時
   優先採用它,關閉對話框時一併清空。現有 ✈️ 終端機版保留不動。

## C. IdentityFile 欄位選檔

`HostEditor` 的 `FieldControl` 對 `IdentityFile` 改渲染複合控制:原本的文字輸入
(react-hook-form register 不變)右側加:

- **金鑰下拉**(DropdownMenu):列 `useKeys` 偵測到的 `~/.ssh` 私鑰(開啟選單才
  抓,lazy),選了填 `~/.ssh/<name>`。
- **Browse…**:`@tauri-apps/plugin-dialog` 的 `open()`(依賴與 `dialog:default`
  capability 都已就緒,首次在前端使用),選到的路徑經 `toTildeSshPath` 正規化。

兩者都以 `setValue(..., { shouldDirty: true })` 寫回表單,存檔流程不變。

## D. 全域關閉自動修正

`ui/input.tsx`、`ui/textarea.tsx`、`ui/command.tsx` 基底元件加
`autoCorrect="off" autoCapitalize="off" spellCheck={false}`(置於 `{...props}`
之前,個別使用處仍可覆寫),並掃除裸 `<input>`/`<textarea>` 漏網。

## 測試與驗收

- vitest:`identity-file.test.ts`(action 判斷 + tilde 轉換);既有 94 測試不退。
- `pnpm build` 過;手動 GUI:四個入口都能開部署對話框、部署後 config 寫入符合
  三分支行為、IdentityFile 下拉/Browse 可用、輸入框不再自動修正字首。
