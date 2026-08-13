# 新增 config 檔案 — 設計(UX 優先)

日期:2026-08-14。原則:**使用者在哪裡選檔案,那裡就能建新檔**;建完自動
接續原操作,零中斷。Include 的正確性(top-level、Host 區塊之前)由後端保證。

## UX 流程

三個入口,全部是既有「選檔案」介面的最後一項「**New config file…**」:

1. **sidebar 檔案 scope 下拉** → 建完把 scope 切到新檔(使用者立刻看到它,
   空檔顯示 0 hosts)。
2. **AddHostDialog 的目標檔選單** → 建完自動選為目標檔,回到表單繼續填。
3. **Move to file**(列 ⋯/右鍵子選單 與 批次動作列的 Move to)→ 建完直接
   把該台/該批 host 搬進新檔 —— 「整理到新檔案」一步完成。

### 建檔對話框(共用元件 `NewConfigFileDialog`)

- 檔名輸入(單一欄位,無路徑),**即時**顯示計畫結果(輸入時呼叫
  `config_plan_new_file`,無 I/O、純計算):
  - 目的地行:`Will be created at ~/.ssh/config.d/work`
  - Include 說明(二擇一):
    - `Loaded automatically by "Include ~/.ssh/config.d/*" — your main config
      is not touched.`(glob 已涵蓋)
    - `SSHelter will add "Include ~/.ssh/work" near the top of ~/.ssh/config.`
      (需插入;敘述誠實,動作照舊 lossless + 先備份)
  - 驗證錯誤 inline 即時顯示(空名、含 `/` 或空白、與既有檔或既有備份撞名、
    glob 要求特定副檔名時自動補上並顯示)。
- Create 按鈕 → `config_create_file` → toast → 關閉並執行 continuation。

### Continuation 模型

ui store:`newFileIntent: null | { kind: "scope" } | { kind: "addHost" } |
{ kind: "move"; aliases: string[] }`(session-only)。dialog 建檔成功後依
intent 收尾:切 scope / `setAddHostTargetFile(path)` / 逐台 `moveHost`
(批次沿用 round 2 的逐台 + toast 摘要)。

## 後端(單一事實來源:Rust 純函式 + 兩個 commands)

- 純函式(TDD):
  - `plan_new_file(name, main_dir, include_patterns) -> Result<NewFilePlan>`:
    - 檔名 gate:非空、無 `/`、無空白、無前導 `-`、無 `..`。
    - 逐一檢查 main config 的 top-level Include patterns(僅 glob 型、展開後
      位於 main config 目錄底下者優先;`config.d/*` 這種目錄 glob 命中 →
      `CoveredByGlob { dir, pattern }`;`*.conf` 字尾型 glob → 名稱自動補字尾
      後命中)。
    - 無可用 glob → `InsertInclude { path: ~/.ssh/<name>, include_line }`。
  - `include_insert_index(items) -> usize`:最後一個 top-level `Include` 之後;
    沒有 Include 時,置於**第一個 Host/Match 區塊之前**、開頭註解之後
    (Include 出現在 Host 區塊後會被 scope 進該 host —— 這是正確性規則)。
- `NewFilePlan`(Serialize + ts-rs):`{ path, covered_by: string | null,
  include_line: string | null, final_name }`。
- command `config_plan_new_file(name) -> NewFilePlan`(無副作用)。
- command `config_create_file(name) -> String(新檔路徑)`:重算 plan →
  建空檔(0600)→ 需要時把 Include 行插入 main config 並 persist(既有
  backup 機制)→ **重載 doc** → 回傳路徑。撞名(檔案已存在)→ Err。

## 驗收

- Rust:plan/insert-index 純函式全案例(glob 涵蓋、字尾 glob、無 glob、
  非法名、插入位置:有 Include/無 Include/開頭註解)。
- 前端:三入口皆可建檔且 continuation 正確;空檔立即出現在 sidebar 與
  所有檔案選單;`pnpm build`+`pnpm test`+`cargo test` 綠。
- README 功能列表補述。
