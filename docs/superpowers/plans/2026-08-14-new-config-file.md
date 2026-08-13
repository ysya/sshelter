# 新增 config 檔案 — 實作計畫

Spec:`docs/superpowers/specs/2026-08-14-new-config-file-design.md`。
每 task 各自 commit;全程三套測試綠。

### Task 1: Rust 純函式(TDD)+ 兩個 commands
- [ ] `config/newfile.rs`:`plan_new_file` + `include_insert_index` 失敗測試 → 實作
- [ ] `NewFilePlan`(ts-rs)、`config_plan_new_file`、`config_create_file`;lib.rs 註冊
- [ ] `cargo test` 綠、bindings 生成
- [ ] Commit: `feat(config): plan and create included config files`

### Task 2: NewConfigFileDialog + ui store intent
- [ ] ui store `newFileIntent` + setter;queries.ts `usePlanNewFile`/`useCreateConfigFile`
- [ ] `NewConfigFileDialog`(即時 plan 預覽、inline 驗證、Create → continuation)
      掛上 App
- [ ] Commit: `feat(config): new-config-file dialog with live include preview`

### Task 3: 三入口接線
- [ ] scope Select 尾項「New config file…」(intent: scope)
- [ ] AddHostDialog 目標檔選單尾項(intent: addHost;建完 setAddHostTargetFile)
- [ ] Move to file 子選單與批次 Move to 尾項(intent: move { aliases })
- [ ] Commit: `feat(host-list): create a new config file from every file picker`

### Task 4: 收尾
- [ ] README;全套測試;push;release PR 確認
