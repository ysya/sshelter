# SSHelter — Phase 1b: Config 編輯器 UI 實作計畫

> REQUIRED SUB-SKILL: superpowers:subagent-driven-development. Frontend is verified by `pnpm build` (tsc + vite) + code review (no headless GUI run); the user runs `pnpm tauri dev` for the visual check.

**Goal:** 一個可用的 master-detail 圖形化 SSH config 編輯器，接上 Phase 1a 的 `config_*` 命令：列出/搜尋/分組 host、編輯 host 欄位並無損存檔、新增/刪除 host、設定 group/tags、外部變更時提示重載。

**Architecture:** React + shadcn/ui + Tailwind v4。TanStack Query 包每個 `config_*` invoke（reads = useQuery，writes = useMutation + invalidate）。Zustand 放 UI 狀態（選取的 alias、搜尋字串）。Host 編輯器用 react-hook-form + zod，把 `HostDetail.options`（keyword/value/enabled 清單）對映成「first-class 欄位 + Advanced raw 清單」，存檔時 diff 出 `HostFieldChange[]`。連線（開終端機）屬 Phase 2，不在此。

**Tech Stack:** 既有 React 19 + TanStack Query v5 + Zustand v5 + shadcn/ui + Tailwind v4。新增 shadcn 元件（sidebar/input/select/switch/tabs/dialog/sonner/scroll-area/badge/label/button…）、react-hook-form + zod + @hookform/resolvers。型別用 `src/bindings/*.ts`（ts-rs 產生：HostSummary/HostDetail/HostOption/HostFieldChange/LoadResult/DriftInfo）。

> 對應 spec §9（前端）與 §13 Phase 1 的 UI 部分。後端命令見 Phase 1a plan。

## first-class 欄位分類（編輯器分頁）
- **Connection**：HostName, User, Port
- **Authentication**：IdentityFile, IdentitiesOnly, AddKeysToAgent, UseKeychain(僅 macOS 顯示), ForwardAgent
- **Forwarding**：ProxyJump, LocalForward, RemoteForward, DynamicForward
- **Reliability**：ServerAliveInterval, ServerAliveCountMax, ConnectTimeout, Compression, RequestTTY, StrictHostKeyChecking
- **Advanced (raw)**：其餘所有 options，以 keyword/value 清單呈現（可增刪）。

## 存檔 diff 規則（最關鍵正確性）
編輯器載入 `HostDetail` 時，記住原始 options（keyword→value）。存檔時對映目前表單：
- 欄位有值且與原始不同 → `HostFieldChange{keyword, value, remove:false}`
- 欄位清空但原本有值 → `{keyword, value:"", remove:true}`
- 未變動的欄位 → 不送（確保後端不 touch、維持 byte-identical）
送出 `config_save_host(alias, changes)`，成功後 invalidate host detail + summaries。對應後端「只改變更行」不變量。

## File Structure（本階段建立）
| 檔案 | 職責 |
|---|---|
| `src/lib/queries.ts` | TanStack Query hooks：`useLoadConfig`、`useHosts`、`useHostDetail(alias)`、`useSaveHost`、`useAddHost`、`useRemoveHost`、`useSetGroup`、`useSetTags`、`useSetOptionEnabled`、`useDrift`；queryKeys 常數 |
| `src/stores/ui.ts`（擴充） | selectedAlias、search、（沿用 theme） |
| `src/lib/hostFields.ts` | first-class 欄位 schema（分類、label、type）+ option↔form 對映 + diff 計算 |
| `src/components/HostList.tsx` | 左側清單：搜尋、依 group 分組、選取、來源檔 badge |
| `src/components/HostEditor.tsx` | 右側 rhf+zod 表單（分頁 + Advanced raw）、dirty 追蹤、存檔 |
| `src/components/AddHostDialog.tsx` | 新增 host（選目標檔 + alias + 基本欄位） |
| `src/components/DriftBanner.tsx` | 偵測外部變更時提示重載 |
| `src/App.tsx`（改寫） | 載入流程 + master-detail 版面 + toaster |

## Tasks（subagent-driven；build-verify + code review）
### Task UI-1 — 資料層 + 欄位對映 + App 殼層 + Host 清單
- `queries.ts`（所有 `config_*` 的 hooks，型別取自 `src/bindings`）、`hostFields.ts`（分類 schema + option↔form 對映 + `computeChanges(original, form)` 回傳 `HostFieldChange[]`，含單元測試 via vitest 或純函式可測）、`stores/ui.ts` 擴充、`App.tsx`（啟動呼叫 `config_load`、master-detail 版面 + sonner Toaster）、`HostList.tsx`（搜尋/分組/選取/來源檔 badge）。
- 驗收：`pnpm build` 綠；`computeChanges` 純函式若加 vitest 則測試綠（否則至少型別正確）。
### Task UI-2 — Host 編輯器 + 新增/刪除 + group/tags + drift banner
- `HostEditor.tsx`（rhf+zod、分頁 first-class 欄位 + Advanced raw 清單、UseKeychain 僅 macOS、dirty 追蹤、存檔走 `computeChanges`→`useSaveHost`）、`AddHostDialog.tsx`、`DriftBanner.tsx`（`useDrift` 輪詢/視窗 focus 時檢查，變更則提示重載）、刪除 host、group/tags 編輯。
- 驗收：`pnpm build` 綠；存檔 diff 邏輯正確（review 重點）。

## Self-Review
- spec §9 覆蓋：Query/Zustand 分層、rhf+zod 表單、master-detail、欄位分組、drift 提示、extraOptions 保留 ✅
- 連線/金鑰/agent 不在此（後續 phase）。
- 型別一律取自 `src/bindings/*.ts`，避免與 Rust drift。
