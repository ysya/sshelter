# 右鍵選單：在對應的 config 檔案新增 host

- 日期：2026-06-19
- 狀態：設計待審
- 範圍：純前端（Rust 後端不需改動）

## 背景與目標

目前新增 host 只有一個入口：工具列的 `+` 按鈕（`AddHostDialog`），開啟後需要**手動從下拉選單選擇目標檔案**。當使用者在「All files」分組檢視下，想把新 host 直接加進某個特定的 config 檔案時，得先開 dialog、再從清單裡找出那個檔案，多了一道手續。

本功能在左側清單的**檔案分組標題（group header）上提供右鍵選單**，讓使用者直接「在此檔案新增 host」，並順手把兩個目前藏在 hover 按鈕／雙擊的既有功能也搬進右鍵選單，提升可發現性。

### 使用者已確認的決策

1. 右鍵要新增的是**一筆完整的 host 主機區塊**（等同現有的「New host」），不是在現有 host 裡加 option 行。
2. 右鍵觸發位置是**檔案分組標題**。
3. 選單包含三項：**在此檔案新增 host**、**檢視原始檔案**、**重新命名顯示名稱**。

## 非目標（YAGNI）

- 不在 host 列上加右鍵選單（host 層級的 rename/move/duplicate/remove 不在本次範圍）。
- 不在現有 host 裡新增 option 行。
- 不改動任何 Rust 後端指令（`config_add_host` 等皆已存在）。
- context-menu 元件只實作本功能用得到的子元件，不整套搬 shadcn（checkbox/radio/submenu 用到再加）。

## 整體方法：重用現有的 `AddHostDialog`

右鍵點「在此檔案新增 host」時，**不另做新流程**，而是開啟現有的 `AddHostDialog` 並自動帶入該檔案為目標檔案。`AddHostDialog` 已具備完整的表單、驗證、提交與 toast，唯一缺口是「目標檔案無法從外部預先指定」。

傳遞目標檔案的方式比較過兩種：

| 方案 | 說明 | 取捨 |
|------|------|------|
| **(採用) ui store 加 `addHostTargetFile`** | 右鍵時設好它再開 dialog，dialog 讀取它預選檔案 | 乾淨，與現有 `addHostOpen` 的「store 驅動開啟」機制一致 |
| setFileScope 切到該檔案再開 dialog | 透過既有 fileScope seeding 路徑 | 有副作用——改變使用者的清單顯示範圍，不採用 |

單一檔案 scope（平面清單、無標題）下沒有 header 可右鍵，但該情境工具列 `+` 已會自動帶入 fileScope，兩者互補無缺口。

## 具體變更

### 1. 新增 `src/components/ui/context-menu.tsx`

照 `src/components/ui/dropdown-menu.tsx` 的 pattern 與樣式包 radix `ContextMenu`（`import { ContextMenu as ContextMenuPrimitive } from "radix-ui"`，已確認 radix-ui 1.5.0 有 export）。

僅實作本功能用得到的子元件：`ContextMenu`(Root)、`ContextMenuTrigger`、`ContextMenuContent`(含 Portal)、`ContextMenuItem`(支援 `inset` / `variant`)、`ContextMenuSeparator`、`ContextMenuLabel`。

樣式直接沿用 dropdown-menu 的 class，差別在於 CSS 變數前綴改為 context-menu 對應的版本（`--radix-context-menu-content-available-height`、`--radix-context-menu-content-transform-origin`）。ContextMenu 在游標位置開啟，不需要 dropdown 的 `align` / `sideOffset`。

### 2. `src/stores/ui.ts`

新增 session-only 狀態（不持久化，與 `addHostOpen` 同層）：

```ts
/** 右鍵「在此檔案新增 host」預先指定的目標檔案（null = 未指定，沿用 fileScope）。 */
addHostTargetFile: string | null;
setAddHostTargetFile: (file: string | null) => void;
```

### 3. `src/lib/add-host-target.ts`（新增純函式）+ 測試

把「dialog 開啟時要預選哪個檔案」的決策抽成可測純函式：

```ts
/**
 * dialog 開啟時的初始目標檔案：右鍵指定的檔案優先，其次沿用 fileScope，
 * 兩者都不是已載入檔案時回空字串（維持「請選擇」）。
 */
export function initialAddHostTarget(
  requested: string | null, // addHostTargetFile（右鍵指定）
  scope: string | null,     // fileScope
  files: string[],
): string {
  if (requested && files.includes(requested)) return requested;
  if (scope && files.includes(scope)) return scope;
  return "";
}
```

`src/lib/add-host-target.test.ts` 涵蓋：requested 有效、requested 無效時 fall back scope、兩者皆無效回空、requested 優先於 scope、requested 不在 files 清單時忽略。

### 4. `src/components/AddHostDialog.tsx`

- seeding useEffect 改用 `initialAddHostTarget(addHostTargetFile, fileScope, files)` 取代目前只看 fileScope 的邏輯。
- `resetForm()` 內加上 `setAddHostTargetFile(null)`，使提交成功（onSuccess → resetForm）與取消（onOpenChange(false) → resetForm）兩條關閉路徑都會清掉預選檔案，避免殘留影響下一次「+」開啟。

> 註：`AddHostDialog` 有 icon 與 labeled 兩個 instance；只有 store 驅動的 icon instance 會被右鍵觸發。labeled instance 僅出現在零 host 的空狀態，該情境沒有 group header 可右鍵，故 `addHostTargetFile` 不會被設定，不受影響。

### 5. `src/components/HostList.tsx`

在 group header 的**非編輯狀態**分支外層包 `ContextMenu` + `ContextMenuTrigger asChild`（包住現有的 `group/header` sticky wrapper `<div>`），選單內容：

```
[+]  New host in this file      → setAddHostTargetFile(file); setAddHostOpen(true)
────────────────────────────
[▤]  View file                  → setViewFile(file)
[✎]  Rename label               → beginEdit(file)
```

文案沿用專案既有英文 UI 風格（與 "New host"、"View file" 一致）。編輯狀態（header 變 input）分支不包 ContextMenu——該狀態下 input 自有原生右鍵選單。

## 資料流

```
右鍵 group header
  → ContextMenu 開啟（游標位置）
  → 點「New host in this file」
      → setAddHostTargetFile(section.file)
      → setAddHostOpen(true)
  → AddHostDialog(icon) 因 addHostOpen=true 開啟
      → seeding useEffect: initialAddHostTarget(addHostTargetFile, …) 預選該檔案
  → 使用者填 alias 等欄位 → Create host
      → useAddHost → config_add_host(targetFile, …)
      → onSuccess: 關閉 + resetForm（清 addHostTargetFile）+ 選取新 host + toast
```

## 邊界情況與風險

- **單一檔案 scope**：無 header 可右鍵；由工具列 `+` 的既有 fileScope seeding 覆蓋。
- **既有 header 互動**（單擊摺疊 / 雙擊重新命名 / hover 檢視按鈕）：右鍵是獨立的 `onContextMenu` 事件，不衝突。
- **「Rename label」與焦點**：從 context menu 觸發 `beginEdit` 會把 header 切成 autoFocus input；Radix 選單關閉時會把焦點還給 trigger，可能與 input autoFocus 競爭。實作時若觀察到搶焦點，於 `ContextMenuContent` 的 `onCloseAutoFocus` 呼叫 `event.preventDefault()` 解決。
- **`ContextMenuTrigger asChild` 與 sticky**：Trigger 只附加事件 handler，不影響現有 `sticky top-0 z-10` 定位；實機驗證一次即可確認。

## 驗證計畫

- `npm test`（vitest）：新增的 `add-host-target.test.ts` 通過，既有測試不破。
- `npm run build`（tsc + vite）：型別與建置無誤。
- 實機 `npm run tauri dev` 手動驗證：
  1. All files 檢視下，右鍵某檔案標題 → 「New host in this file」→ dialog 開啟且目標檔案已預選為該檔 → 新增成功落在正確檔案。
  2. 右鍵 →「View file」開啟原始檔案檢視。
  3. 右鍵 →「Rename label」進入行內重新命名、autoFocus 正常。
  4. 連續操作：右鍵 A 檔新增取消後，再按工具列 `+`，目標檔案不應殘留為 A（addHostTargetFile 已清）。
