# 更新頻道(Stable / Beta)與 0.16.0 發版 — 設計

日期:2026-10-01。前置:sync chain Phase A 已併入 `main`(見 `2026-09-27-sync-chain-design.md`)。

## 1. 目標與非目標

**目標**
- 已安裝的 SSHelter 可以選擇更新頻道:**Stable**(預設)或 **Beta**。
- 選 Beta 的機器透過**自動更新**收到 prerelease,不需要手動下載安裝檔。
- 選 Stable 的使用者(預設,也就是所有現有使用者)的更新行為**完全不變**,永遠不會看到 beta。

**為什麼先發正式版**:已安裝的 0.15.1 只認得正式版的更新網址
(`releases/latest/download/latest.json`),而 GitHub 的 `latest` 不含 prerelease。因此下一版
(0.16.0)必須以正式版發出,把「更新頻道」帶到使用者手上;之後的 beta 才能經由自動更新送達。
0.16.0 同時帶出 sync chain(標示為 Beta)。

**非目標**
- 降版:Beta 切回 Stable 時停在目前版本,等更新的正式版出來才更新。
- 以頻道當功能開關:Sync 對所有人可見(只標示 Beta),不依頻道隱藏。
- 自架更新伺服器、或把更新清單交給 Cloudflare Worker。
- 0.16.0 內建 relay:**不設定** `SSHELTER_RELAY_URL`;使用者在 Settings → Sync 自行填入
  relay 網址(sync spec §5 的「兩種模式」)。

## 2. 已定案的決策

| 決策 | 選擇 | 理由 |
|---|---|---|
| Sync 在 0.16.0 的可見度 | 所有人可見,標示 Beta | 使用者決定;不另做功能開關 |
| Beta 清單放哪 | 固定的 GitHub prerelease `updater-beta`,資產 `latest.json` | 不需額外伺服器;`latest` 永遠不會指向它 |
| Stable 頻道的程式路徑 | 沿用既有前端 `check()`,**一行不改** | 新更新程式若有 bug 會把使用者鎖在舊版;只讓 Beta 走新程式,出問題切回 Stable 即可恢復 |
| Beta 頻道的程式路徑 | 新的 Rust 指令(`updater_builder().endpoints(...)`) | 前端 `check()` 無法在執行時換網址;Rust API 可以 |
| Beta 版號 | `X.Y.Z-N`(純數字 pre-release,例如 `0.16.1-1`) | Tauri:MSI 的 pre-release 標記只能是數字;`bundle.targets` 為 `all`(含 MSI) |
| 發 beta 的方式 | 新 workflow「Publish beta」(手動執行) | release-please 的正式版流程完全不動;不使用 release-please 的 `prerelease: true`(它會把所有 0.x 版本都標成 prerelease) |
| 內建 relay | 不內建 | 使用者決定 |

## 3. App 端

### 3.1 設定
- `useSettingsStore` 新增 `updateChannel: "stable" | "beta"`(預設 `"stable"`,與其他偏好一起 persist)
  與 `setUpdateChannel(channel)`。
- Settings → General → **Updates** 區塊,在「Check for updates automatically」之後新增一列
  **Update channel**(Select:Stable / Beta)。說明文字:
  「Beta gets preview builds earlier and may be less stable. Switching back to Stable keeps your
  current version until a newer stable release is out.」
- 切換到 Beta 時立即執行一次手動檢查(`checkForUpdates({ silent: false })`),讓使用者馬上知道
  有沒有 beta 可裝。切回 Stable 不檢查。

### 3.2 檢查與安裝
`checkForUpdates({ silent })`(`src/lib/updater.ts`)在開頭讀取 `updateChannel`:

- **Stable**:現有程式碼路徑原封不動 —— `check()` → toast →「Install & restart」→
  `update.downloadAndInstall()` → `relaunch()`。
- **Beta**:改呼叫 Rust 指令,其餘 UX(同版本只提示一次、`busy` 防重入、toast id、錯誤處理)共用:
  - `updater_check_beta()` → `UpdateInfo | null`(`{ version, body }`);有新版就顯示同一個 toast。
  - 按「Install & restart」→ `updater_install_beta()` → 成功後 `relaunch()`。

Rust 新模組 `src-tauri/src/updater_channel.rs`:
- `const BETA_ENDPOINT: &str = "https://github.com/ysya/sshelter/releases/download/updater-beta/latest.json";`
- `updater_check_beta(app)`(async command):
  `app.updater_builder().endpoints(vec![BETA_ENDPOINT 解析成 Url])?.build()?.check().await?`;
  有更新時把 `Update` 存進 managed state(`Mutex<Option<Update>>`),回傳 `UpdateInfo`(ts-rs 綁定)。
- `updater_install_beta(app)`(async command):取出暫存的 `Update`,
  `download_and_install(|_, _| {}, || {})`;沒有暫存的更新 → 錯誤「no update to install; check again」。
- 版本比較、簽章驗證都沿用 updater plugin 的預設:只有遠端版本**大於**目前版本才算更新(不降版),
  簽章用 `tauri.conf.json` 裡的 pubkey 驗證。
- 錯誤:自動(silent)檢查失敗只 `console.warn`(與現在一致,例如 `updater-beta` 尚未建立時);
  手動檢查失敗顯示「Could not check for updates」toast。

### 3.3 Sync 的 Beta 標示
- Settings 左側分類「Sync」旁加 **Beta** 標記(沿用既有 `Badge`)。
- Sync pane 第一個區塊的說明加一句「Sync is in beta.」。

## 4. CI 端

### 4.1 `updater-beta` release
- tag `updater-beta`,**prerelease**(因此永遠不會成為 `latest`),標題「Beta update channel」,
  說明寫明此 release 由 CI 維護、只用來承載 Beta 頻道的 `latest.json`。
- 唯一的資產是 `latest.json`,永遠指向「目前最新的版本」—— beta 或正式版都算。
- 不存在時由 4.2 的腳本自動建立。

### 4.2 共用腳本 `scripts/update-beta-manifest.mjs`
- 輸入:來源 release 的 tag(例如 `v0.16.0`、`v0.16.1-1`)。
- 流程:用 `gh` 下載來源 release 的 `latest.json` → 下載 `updater-beta` 目前的 `latest.json`
  (不存在視為空)→ 新版本**大於**目前版本(或目前為空)才 `gh release upload updater-beta
  latest.json --clobber`,否則記錄「skip」並成功結束。
- 版本比較 `compareVersions(a, b)`:接受 `X.Y.Z` 與 `X.Y.Z-N`(N 為數字);同一個 `X.Y.Z` 時
  正式版大於任何 pre-release;`-N` 依數字比較;其他格式丟錯(讓 workflow 失敗而不是亂寫清單)。
- 版本比較與「要不要覆寫」的決定寫成可匯入的純函式,附 vitest 單元測試。
- 呼叫這支腳本的 job(4.3、4.4)都設 `permissions: contents: write`,並共用跨 workflow 的
  `concurrency: { group: updater-beta-manifest, cancel-in-progress: false }`:正式版與 beta 的清單
  更新若同時發生,會排隊執行,不會出現「較舊的版本最後寫入」把 Beta 頻道蓋回去。

### 4.3 `release.yml`(正式版)
- 新增 job `beta-manifest`:`needs: [release-please, build]`,只在 `release_created` 時執行,
  以正式版 tag 呼叫 4.2 的腳本。四個平台都 build 完才執行,避免多個平台同時更新 `latest.json`
  時互相覆寫(既有註解記載過這個競態)。
- 其餘步驟不變。

### 4.4 新 workflow `.github/workflows/beta.yml`(「Publish beta」)
- 觸發:`workflow_dispatch`,輸入 `version`(必填)、`notes`(選填)。
- `concurrency`:同一時間只跑一個 beta 發布。
- job `prepare`(ubuntu):
  1. `version` 必須符合 `^\d+\.\d+\.\d+-\d+$`。
  2. `version` 必須大於 `.release-please-manifest.json` 的正式版版本(用 4.2 的比較函式)。
  3. tag `v<version>` 不得已存在。
  4. `gh release create v<version> --prerelease --target <當下 commit> --title "v<version> (beta)"`,
     說明用 `notes` 或預設文字。先建立 release,四個平台再掛上檔案(與正式版相同模式,避免
     多個平台同時建立 release)。
- job `build`(矩陣與 `release.yml` 相同:macOS universal、Linux x64、Linux arm64、Windows x64):
  1. checkout 同一個 commit。
  2. **只在 CI 工作目錄**把 `tauri.conf.json` 的 `version`、`Cargo.toml` 的 `[package] version`、
     `package.json` 的 `version` 改成 `<version>`(不 commit 回 `main`;release-please 的版本管理不受影響)。
  3. 與 `release.yml` 相同的相依安裝、簽章金鑰(`TAURI_SIGNING_PRIVATE_KEY`)、`SSHELTER_RELAY_URL`。
  4. `tauri-action` 以 `tagName: v<version>` 把安裝檔與該 release 的 `latest.json` 掛上去。
- job `beta-manifest`:`needs: build`,以 `v<version>` 呼叫 4.2 的腳本。
- 已知取捨:beta tag 指向的 commit 裡,版本檔仍是正式版的版本號;實際安裝檔與 app 回報的版本
  是 beta 版號(CI 修改)。

## 5. 發版順序

1. 實作 App 端與 CI 端,測試通過後併入 `main`(本機)。發版前在 dev 版實際按「Check now」:
   Stable 回報已是最新;Beta 回報檢查失敗(`updater-beta` 尚未建立,預期如此)。
2. 使用者部署 relay:`cd relay && npx wrangler login && npx wrangler deploy`。**不設定**
   `SSHELTER_RELAY_URL`。
3. push `main` → release-please 開 0.16.0 的 release PR → 使用者 merge。
4. 0.16.0 build 完成 → `beta-manifest` job 建立 `updater-beta`,`latest.json` 指向 0.16.0。
5. 兩台電腦從 0.15.1 **自動**更新到 0.16.0(走既有的舊程式,與平常相同)。
6. 兩台都到 Settings → General 把更新頻道切成 **Beta**;到 Settings → Sync 填入 relay 網址,
   依手動清單測試 Sync。
7. 測試中的修正:在 Actions 執行「Publish beta」(例如 `0.16.1-1`)→ 兩台自動收到更新。
   穩定後由 release-please 照常發 0.16.1 正式版;Beta 頻道的機器也會收到它。

## 6. 風險與對策

| 風險 | 對策 |
|---|---|
| 新的更新程式有 bug,把使用者鎖在舊版 | Stable 路徑完全沿用既有程式;只有 Beta 走新程式,出問題切回 Stable 即可 |
| 多個平台同時更新 `latest.json` 互相覆寫 | `beta-manifest` 在四個平台都完成後才執行 |
| beta 被正式版使用者收到 | beta 是 prerelease,GitHub `latest` 不含 prerelease;Stable 只讀 `latest` |
| 較舊的正式版把 Beta 頻道蓋回去 | 腳本只在版本較新時覆寫;正式版與 beta 的清單 job 共用 concurrency group,排隊執行 |
| Windows MSI 不接受非數字 pre-release | 版本號格式驗證 `X.Y.Z-N` |
| release-please 的 `prerelease: true` 讓所有 0.x 變 prerelease | 不使用該設定;beta 由獨立 workflow 發布 |
| beta 期間切回 Stable | 不降版,停在 beta 版號,直到更新的正式版出來 |

## 7. 測試

- **單元測試**
  - `compareVersions` / 覆寫決定(vitest):`0.16.1-1 < 0.16.1`、`0.16.1-2 > 0.16.1-1`、
    `0.16.10 > 0.16.9`、較舊正式版不覆寫較新的 beta、非法格式丟錯。
  - Rust:`BETA_ENDPOINT` 能解析成合法 `Url`;沒有暫存更新時 `updater_install_beta` 回錯誤
    (以純函式或暫存狀態的單元測試涵蓋)。
  - 前端:`updateChannel` 預設為 `"stable"`。
- **發版前手動**:第 5 節第 1 步的 dev 版檢查。
- **首次 beta**:切到 Beta 的機器提示更新;暫時切回 Stable 的機器不提示。

## 8. 文件

- README:Updates 段落補上更新頻道;說明 Sync 是 beta、需要自行部署 relay(0.16.0 不內建)。
- 新增發 beta 的說明(維護者用):Actions →「Publish beta」→ 輸入 `X.Y.Z-N`。
