# 右鍵選單：在對應 config 檔案新增 host — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在左側清單的檔案分組標題上提供右鍵選單，讓使用者直接在該檔案新增 host（並順手檢視原始檔案／重新命名顯示名稱）。

**Architecture:** 純前端。右鍵選單透過新的 `context-menu.tsx`（radix `ContextMenu`）呈現；「New host in this file」把目標檔案寫進 ui store 的 `addHostTargetFile` 再開啟現有的 `AddHostDialog`，dialog 用一個可測純函式 `initialAddHostTarget` 決定預選檔案。Rust 後端完全不動。

**Tech Stack:** React 19、Tauri 2、zustand、radix-ui 1.5.0、TanStack Query、vitest（純函式測試）。

## Global Constraints

- 對使用者以繁體中文溝通；程式碼、識別字、commit 訊息一律英文。
- Conventional Commits 格式；每個 commit 結尾加 `Claude-Session: https://claude.ai/code/session_016Tf82JX4ihBoEC7ZvbMpWE`。
- 純前端：不修改 `src-tauri/` 任何檔案。
- radix 元件一律 `import { X as XPrimitive } from "radix-ui"`（整合套件，非 `@radix-ui/react-*`）。
- 無元件測試框架；只有純函式有 vitest 測試（node 環境）。UI 接線靠 `tsc` + 手動驗證。
- UI 文案用英文，與既有風格一致（"New host"、"View file"）。
- spec：`docs/superpowers/specs/2026-06-19-context-menu-add-host-design.md`。

---

### Task 1: `initialAddHostTarget` 目標檔案解析純函式

**Files:**
- Create: `src/lib/add-host-target.ts`
- Test: `src/lib/add-host-target.test.ts`

**Interfaces:**
- Produces: `initialAddHostTarget(requested: string | null, scope: string | null, files: string[]): string`

- [ ] **Step 1: 寫失敗測試**

`src/lib/add-host-target.test.ts`：

```ts
import { describe, expect, it } from "vitest";
import { initialAddHostTarget } from "@/lib/add-host-target";

const FILES = ["/a/config", "/a/config.d/work", "/a/config.d/home"];

describe("initialAddHostTarget", () => {
  it("uses the requested file when it is a loaded file", () => {
    expect(initialAddHostTarget("/a/config.d/work", null, FILES)).toBe("/a/config.d/work");
  });

  it("prefers the requested file over the current scope", () => {
    expect(initialAddHostTarget("/a/config.d/work", "/a/config.d/home", FILES)).toBe(
      "/a/config.d/work",
    );
  });

  it("falls back to scope when nothing is requested", () => {
    expect(initialAddHostTarget(null, "/a/config.d/home", FILES)).toBe("/a/config.d/home");
  });

  it("ignores a requested file that is not loaded", () => {
    expect(initialAddHostTarget("/a/ghost", "/a/config.d/home", FILES)).toBe(
      "/a/config.d/home",
    );
  });

  it("returns empty string when neither requested nor scope is a loaded file", () => {
    expect(initialAddHostTarget("/a/ghost", "/a/gone", FILES)).toBe("");
    expect(initialAddHostTarget(null, null, FILES)).toBe("");
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `npm test -- add-host-target`
Expected: FAIL（`initialAddHostTarget` is not defined / 模組不存在）

- [ ] **Step 3: 寫最小實作**

`src/lib/add-host-target.ts`：

```ts
/**
 * The target file an Add-host dialog should preselect when it opens.
 *
 * A `requested` file (set by the file-header right-click "New host in this
 * file") wins; otherwise the current sidebar `scope` (fileScope) is used. Only
 * a path that is actually a loaded file counts — anything else yields "" so the
 * picker stays on its "select a file" placeholder.
 */
export function initialAddHostTarget(
  requested: string | null,
  scope: string | null,
  files: string[],
): string {
  if (requested && files.includes(requested)) return requested;
  if (scope && files.includes(scope)) return scope;
  return "";
}
```

- [ ] **Step 4: 跑測試確認通過**

Run: `npm test -- add-host-target`
Expected: PASS（5 個測試全綠）

- [ ] **Step 5: Commit**

```bash
git add src/lib/add-host-target.ts src/lib/add-host-target.test.ts
git commit -m "feat(host-list): add target-file resolver for right-click add-host"
```

---

### Task 2: ui store 追蹤右鍵指定的目標檔案

**Files:**
- Modify: `src/stores/ui.ts`

**Interfaces:**
- Consumes: 無
- Produces: store 欄位 `addHostTargetFile: string | null`、`setAddHostTargetFile: (file: string | null) => void`

- [ ] **Step 1: 在 `UiState` interface 加欄位**

在 `addHostOpen` / `setAddHostOpen` 宣告（`src/stores/ui.ts:23-25`）之後插入：

```ts
  /**
   * The file the right-click "New host in this file" wants the Add-host dialog
   * to preselect (null = none; the dialog falls back to fileScope). Session-only.
   */
  addHostTargetFile: string | null;
  setAddHostTargetFile: (file: string | null) => void;
```

- [ ] **Step 2: 在 store 實作加初始值與 setter**

在 `addHostOpen: false,` / `setAddHostOpen: ...`（`src/stores/ui.ts:58-59`）之後插入：

```ts
      addHostTargetFile: null,
      setAddHostTargetFile: (addHostTargetFile) => set({ addHostTargetFile }),
```

> `partialize`（`src/stores/ui.ts:68`）維持不變——此欄位是 session-only，不應持久化。

- [ ] **Step 3: 型別檢查**

Run: `npx tsc --noEmit`
Expected: 無錯誤。

- [ ] **Step 4: Commit**

```bash
git add src/stores/ui.ts
git commit -m "feat(host-list): track right-click add-host target file in ui store"
```

---

### Task 3: `context-menu.tsx` UI primitive

**Files:**
- Create: `src/components/ui/context-menu.tsx`

**Interfaces:**
- Consumes: `radix-ui` `ContextMenu`、`@/lib/utils` `cn`
- Produces: `ContextMenu`、`ContextMenuTrigger`、`ContextMenuContent`、`ContextMenuItem`、`ContextMenuLabel`、`ContextMenuSeparator`

- [ ] **Step 1: 建立元件**（照 `dropdown-menu.tsx` 的 pattern，CSS 變數改 context-menu 版）

`src/components/ui/context-menu.tsx`：

```tsx
"use client"

import * as React from "react"
import { ContextMenu as ContextMenuPrimitive } from "radix-ui"

import { cn } from "@/lib/utils"

function ContextMenu({
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Root>) {
  return <ContextMenuPrimitive.Root data-slot="context-menu" {...props} />
}

function ContextMenuTrigger({
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Trigger>) {
  return (
    <ContextMenuPrimitive.Trigger data-slot="context-menu-trigger" {...props} />
  )
}

function ContextMenuContent({
  className,
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Content>) {
  return (
    <ContextMenuPrimitive.Portal>
      <ContextMenuPrimitive.Content
        data-slot="context-menu-content"
        className={cn(
          "z-50 max-h-(--radix-context-menu-content-available-height) min-w-40 origin-(--radix-context-menu-content-transform-origin) overflow-x-hidden overflow-y-auto rounded-lg bg-popover p-1 text-popover-foreground shadow-md ring-1 ring-foreground/10 duration-100 data-[side=bottom]:slide-in-from-top-2 data-[side=left]:slide-in-from-right-2 data-[side=right]:slide-in-from-left-2 data-[side=top]:slide-in-from-bottom-2 data-[state=closed]:overflow-hidden data-open:animate-in data-open:fade-in-0 data-open:zoom-in-95 data-closed:animate-out data-closed:fade-out-0 data-closed:zoom-out-95",
          className,
        )}
        {...props}
      />
    </ContextMenuPrimitive.Portal>
  )
}

function ContextMenuItem({
  className,
  inset,
  variant = "default",
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Item> & {
  inset?: boolean
  variant?: "default" | "destructive"
}) {
  return (
    <ContextMenuPrimitive.Item
      data-slot="context-menu-item"
      data-inset={inset}
      data-variant={variant}
      className={cn(
        "group/context-menu-item relative flex cursor-default items-center gap-1.5 rounded-md px-1.5 py-1 text-sm outline-hidden select-none focus:bg-accent focus:text-accent-foreground not-data-[variant=destructive]:focus:**:text-accent-foreground data-inset:pl-7 data-[variant=destructive]:text-destructive data-[variant=destructive]:focus:bg-destructive/10 data-[variant=destructive]:focus:text-destructive dark:data-[variant=destructive]:focus:bg-destructive/20 data-disabled:pointer-events-none data-disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:shrink-0 [&_svg:not([class*='size-'])]:size-4 data-[variant=destructive]:*:[svg]:text-destructive",
        className,
      )}
      {...props}
    />
  )
}

function ContextMenuLabel({
  className,
  inset,
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Label> & {
  inset?: boolean
}) {
  return (
    <ContextMenuPrimitive.Label
      data-slot="context-menu-label"
      data-inset={inset}
      className={cn(
        "px-1.5 py-1 text-xs font-medium text-muted-foreground data-inset:pl-7",
        className,
      )}
      {...props}
    />
  )
}

function ContextMenuSeparator({
  className,
  ...props
}: React.ComponentProps<typeof ContextMenuPrimitive.Separator>) {
  return (
    <ContextMenuPrimitive.Separator
      data-slot="context-menu-separator"
      className={cn("-mx-1 my-1 h-px bg-border", className)}
      {...props}
    />
  )
}

export {
  ContextMenu,
  ContextMenuTrigger,
  ContextMenuContent,
  ContextMenuItem,
  ContextMenuLabel,
  ContextMenuSeparator,
}
```

- [ ] **Step 2: 型別檢查**

Run: `npx tsc --noEmit`
Expected: 無錯誤。

- [ ] **Step 3: Commit**

```bash
git add src/components/ui/context-menu.tsx
git commit -m "feat(ui): add context-menu primitive"
```

---

### Task 4: `AddHostDialog` 用 `initialAddHostTarget` 預選並在關閉時清理

**Files:**
- Modify: `src/components/AddHostDialog.tsx`

**Interfaces:**
- Consumes: Task 1 `initialAddHostTarget`、Task 2 `addHostTargetFile` / `setAddHostTargetFile`

- [ ] **Step 1: 匯入純函式**

在既有 import 區（`src/components/AddHostDialog.tsx` 頂部）加：

```ts
import { initialAddHostTarget } from "@/lib/add-host-target";
```

- [ ] **Step 2: 取得新的 store 欄位**

`fileScope` 取用處（`src/components/AddHostDialog.tsx:68`）附近，加上：

```ts
  const addHostTargetFile = useUiStore((s) => s.addHostTargetFile);
  const setAddHostTargetFile = useUiStore((s) => s.setAddHostTargetFile);
```

- [ ] **Step 3: seeding useEffect 改用純函式**

把現有的 seeding effect（`src/components/AddHostDialog.tsx:69-73`）：

```ts
  useEffect(() => {
    if (open && targetFile === "" && fileScope && files.includes(fileScope)) {
      setTargetFile(fileScope);
    }
  }, [open, targetFile, fileScope, files]);
```

改為：

```ts
  useEffect(() => {
    if (open && targetFile === "") {
      const seeded = initialAddHostTarget(addHostTargetFile, fileScope, files);
      if (seeded) setTargetFile(seeded);
    }
  }, [open, targetFile, addHostTargetFile, fileScope, files]);
```

- [ ] **Step 4: `resetForm` 清掉右鍵目標**

把 `resetForm`（`src/components/AddHostDialog.tsx:75-81`）末尾加一行，使提交成功與取消兩條關閉路徑都會清除：

```ts
  function resetForm() {
    setTargetFile("");
    setAlias("");
    setHostName("");
    setUser("");
    setPort("");
    setAddHostTargetFile(null);
  }
```

- [ ] **Step 5: 型別檢查 + 既有測試**

Run: `npx tsc --noEmit && npm test`
Expected: tsc 無錯誤；既有測試全綠（含 Task 1 的新測試）。

- [ ] **Step 6: Commit**

```bash
git add src/components/AddHostDialog.tsx
git commit -m "feat(host-list): seed AddHostDialog target from right-click file"
```

---

### Task 5: `HostList` 檔案分組標題右鍵選單

**Files:**
- Modify: `src/components/HostList.tsx`

**Interfaces:**
- Consumes: Task 2 `setAddHostOpen` / `setAddHostTargetFile`、Task 3 context-menu 元件

- [ ] **Step 1: 匯入 context-menu 元件與 Pencil icon**

在 `lucide-react` import（`src/components/HostList.tsx:2-11`）加入 `Pencil`：

```ts
import {
  Search,
  ServerOff,
  Server,
  Globe,
  ChevronRight,
  FileText,
  Play,
  SlidersHorizontal,
  Pencil,
} from "lucide-react";
```

在 `FileViewDialog` import（`src/components/HostList.tsx:29`）之後加：

```ts
import {
  ContextMenu,
  ContextMenuContent,
  ContextMenuItem,
  ContextMenuSeparator,
  ContextMenuTrigger,
} from "@/components/ui/context-menu";
```

並在頂部加 `Plus`（選單「New host」用）：lucide 的 `Plus` 尚未匯入，於上面的 lucide import 區一併加入 `Plus`。

- [ ] **Step 2: 取得 store setters**

在 `HostList` 元件的 `useUiStore` 取用區（`src/components/HostList.tsx:240-247`）加入：

```ts
  const setAddHostOpen = useUiStore((s) => s.setAddHostOpen);
  const setAddHostTargetFile = useUiStore((s) => s.setAddHostTargetFile);
```

- [ ] **Step 3: 用 ContextMenu 包住非編輯狀態的 header**

把非編輯分支的外層 wrapper（`src/components/HostList.tsx:576` 起的 `<div className="group/header sidebar-sticky-header sticky top-0 z-10 rounded-sm">` 整塊，到對應 `</div>`）用 `ContextMenu` + `ContextMenuTrigger asChild` 包起來，並在其後加 `ContextMenuContent`：

```tsx
                      <ContextMenu>
                        <ContextMenuTrigger asChild>
                          <div className="group/header sidebar-sticky-header sticky top-0 z-10 rounded-sm">
                            {/* …既有的 header <button> 與 View-file <Button> 原封不動… */}
                          </div>
                        </ContextMenuTrigger>
                        <ContextMenuContent>
                          <ContextMenuItem
                            onSelect={() => {
                              setAddHostTargetFile(section.file);
                              setAddHostOpen(true);
                            }}
                          >
                            <Plus />
                            New host in this file
                          </ContextMenuItem>
                          <ContextMenuSeparator />
                          <ContextMenuItem onSelect={() => setViewFile(section.file)}>
                            <FileText />
                            View file
                          </ContextMenuItem>
                          <ContextMenuItem onSelect={() => beginEdit(section.file)}>
                            <Pencil />
                            Rename label
                          </ContextMenuItem>
                        </ContextMenuContent>
                      </ContextMenu>
```

> 只包「非編輯狀態」分支；編輯狀態（header 變 `Input`）維持不變，`Input` 自有原生右鍵選單。

- [ ] **Step 4: 型別檢查**

Run: `npx tsc --noEmit`
Expected: 無錯誤。

- [ ] **Step 5: Commit**

```bash
git add src/components/HostList.tsx
git commit -m "feat(host-list): right-click file headers to add a host, view, or rename"
```

---

### Task 6: 完整驗證

**Files:** 無（驗證關卡）

- [ ] **Step 1: 完整測試 + 建置**

Run: `npm test && npm run build`
Expected: 所有 vitest 測試通過；`tsc && vite build` 成功無錯。

- [ ] **Step 2: 實機手動驗證**

Run: `npm run tauri dev`
逐項確認：
1. **All files** 檢視下右鍵某檔案標題 → 「New host in this file」→ dialog 開啟且「Target file」已預選為該檔 → 填 alias 後 Create → 新 host 落在正確檔案。
2. 右鍵 →「View file」→ 開啟該檔的原始檢視 dialog。
3. 右鍵 →「Rename label」→ header 進入行內編輯且 autoFocus 正常（若觀察到焦點被選單搶走，於 `ContextMenuContent` 加 `onCloseAutoFocus={(e) => e.preventDefault()}`，回 Task 5 重新驗證）。
4. 右鍵 A 檔「New host」後**取消** → 再按工具列 `+` → 「Target file」不應殘留為 A（應為空或目前 fileScope）。
5. 既有互動回歸：單擊標題仍可摺疊、雙擊仍可重新命名、hover 仍顯示 View-file 按鈕。

- [ ] **Step 3: 無新增 commit**（驗證關卡；若 Step 2.3 需修正則該修正自行 commit）

---

## Self-Review

**Spec coverage:**
- 右鍵選單機制 → Task 3（元件）+ Task 5（接線）。✓
- 在對應檔案新增 host → Task 1（解析）+ Task 2（store）+ Task 4（dialog 預選）+ Task 5（選單項）。✓
- View file / Rename label 兩項 → Task 5。✓
- 關閉時清理 `addHostTargetFile` → Task 4 Step 4。✓
- 單一檔案 scope 由既有 `+` 覆蓋 → 無需新任務（沿用既有 fileScope seeding，Task 4 保留 fallback）。✓
- Rust 後端不動 → 全計畫無 `src-tauri/` 變更。✓

**Placeholder scan:** 無 TBD/TODO；每個程式碼步驟均含完整內容。✓

**Type consistency:** `initialAddHostTarget(requested, scope, files)` 簽章在 Task 1 定義、Task 4 消費一致；`addHostTargetFile` / `setAddHostTargetFile` 在 Task 2 定義、Task 4/5 消費一致；context-menu export 名稱在 Task 3 定義、Task 5 匯入一致。✓
