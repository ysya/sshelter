# Sync Chain — Phase A4(前端:Sync pane、配對/上手、遷入 wizard、事件、文件)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓使用者從 Settings → Sync 建立/加入 chain(加入前就能改 relay URL)、確認並保存助記詞、看到裝置與同步狀態、把既有主機遷入同步檔、以檔案定位處理被遮蔽的同名主機;並把後端事件(狀態、衝突、已套用)接進 UI。

**Architecture:** 沿用 `src/lib/mcp.ts` + `McpPane` 的模式:TanStack Query 包「非祕密」的 Tauri commands、Settings 分頁輪詢狀態;**助記詞相關呼叫(建立回傳、Show、Join 輸入)不經 TanStack Query 的 cache**(`useMutation` 會把 `variables`/`data` 留在 cache 裡),改為直接 `tauriInvoke` + 元件 local state。助記詞確認畫面的 state 放在 `SyncPane`(父層),不隨 `joined` 切換卸載。新增 `SyncPane.tsx`、`SyncMigrationDialog.tsx`;`App.tsx` 監聽 `sync://status`/`sync://conflict`/`sync://applied` 與視窗焦點。純邏輯抽成 `src/lib/sync-migration.ts` 以 vitest 覆蓋。

**Tech Stack:** React + TypeScript、TanStack Query、Zustand、shadcn/ui 既有元件(Dialog、AlertDialog、Checkbox、Textarea、Badge、Button、Input)、`@tauri-apps/api/event`。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §7(前端)、§10(migration、shadowed alias)、§2(Forget device 不是撤權)

## Global Constraints

- UI 文案英文;既有元件與樣式慣例(`Section`/`SettingsGroup`/`SettingsRow`);不新增 npm 相依。
- 助記詞只在使用者明確按下按鈕時顯示,顯示區塊不可被 clipboard 以外的方式自動複製;不進 toast、不進 console、**不進 TanStack Query cache**(不用 `useMutation`/`useQuery` 承載 words)。
- 助記詞確認對話框的 state 屬於 `SyncPane`,建立成功後 `joined` 變 true 時它必須仍然掛著,直到使用者勾「I have saved these words」。
- Forget device 的文案不得暗示撤權(spec §2):明講「a device that still has the recovery phrase keeps syncing」。
- `App.tsx` 已有 `const queryClient = useQueryClient();`(line ~59),**重用它**,不要再宣告一次;repo 開著 `noUnusedLocals`,不留未使用的 import。
- `pnpm build`(tsc + vite)與 `pnpm test` 全綠;每 task 一個 commit。
- 所有 Tauri 呼叫走 `tauriInvoke`;型別以 `src/bindings/*.ts` 為準(A3 生成:`SyncStatus` 含 `phrase_cleanup_pending`、`DuplicateAlias`、`MigrationReport`)。

## Review Focus

1. 使用者把 24 詞貼成多行、含逗號或編號(`1. abandon`)—— 前端先整理成單行空白分隔再送後端(Task 1 `cleanWordsInput` 測試)。
2. 遷入清單必須排除 wildcard 區塊與已在同步檔的主機(Task 1 `groupHostsForMigration` 測試)。
3. 建立 chain 後狀態立刻變 joined —— 助記詞對話框必須還在,且未勾選前不可關閉(Task 2:state 在 `SyncPane`;手動驗證)。
4. 加入 chain 時 relay 404(助記詞錯)—— 顯示可讀錯誤、表單保留輸入(Task 2 手動驗證)。
5. 離開 chain 時「Delete from relay」預設不勾、且說明會影響其他裝置(Task 2)。
6. 預設中繼不可達或要用自架 —— 未加入畫面就能改 relay URL(Task 2)。
7. 另一台裝置改了主機 —— 45 秒內本機 sidebar 更新、不需 reload(Task 1 `sync://applied` → invalidate `["config"]`;手動驗證)。

---

### Task 1: hooks、純邏輯與事件接線

**Files:**
- Create: `src/lib/sync.ts`
- Create: `src/lib/sync-migration.ts`
- Create: `src/lib/sync-migration.test.ts`
- Modify: `src/stores/ui.ts`(`SettingsCategory` 加 `"sync"`;新增 `syncMigrationOpen`)
- Modify: `src/App.tsx`(事件監聽、焦點觸發)

**Interfaces:**
- Consumes: `src/bindings/{SyncStatus,SyncDevice,MigrationReport,DuplicateAlias}.ts`、commands(A3)
- Produces:
  - hooks(非祕密):`useSyncStatus(refetchInterval)`、`useLeaveChain()`、`useSyncNow()`、`useSetRelayUrl()`、`useSetDeviceName()`、`useForgetDevice()`、`useMigrateHosts()`、`useDuplicateAliases(enabled)`、`useResolveShadowed()`
  - 祕密呼叫(不進 cache):`createChain(deviceName): Promise<string>`、`joinChain(words, deviceName): Promise<SyncStatus>`、`showWords(): Promise<string>`
  - `refreshSyncViews(queryClient)`、`errorMessage(error)`
  - `cleanWordsInput(raw: string): string`
  - `groupHostsForMigration(hosts: HostSummary[], managedFile: string): { file: string; hosts: HostSummary[] }[]`
  - ui store:`syncMigrationOpen: boolean; setSyncMigrationOpen(open)`

- [ ] **Step 1: 寫失敗的測試(純邏輯)**

建立 `src/lib/sync-migration.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { HostSummary } from "@/bindings/HostSummary";
import { cleanWordsInput, groupHostsForMigration } from "./sync-migration";

function host(alias: string, file: string, patterns: string[] = [alias]): HostSummary {
  return { alias, patterns, source_file: file, tags: [], hostname: null, user: null };
}

describe("cleanWordsInput", () => {
  it("joins lines, strips numbering and punctuation, lowercases", () => {
    const raw = "1. Abandon\n2) abandon,\n3 - ABANDON\n\n  about ";
    expect(cleanWordsInput(raw)).toBe("abandon abandon abandon about");
  });

  it("collapses whitespace including full-width spaces", () => {
    expect(cleanWordsInput("a　b   c")).toBe("a b c");
  });
});

describe("groupHostsForMigration", () => {
  const managed = "/home/f/.ssh/sshelter/hosts.config";

  it("groups real hosts by source file and skips wildcards and already-synced hosts", () => {
    const hosts = [
      host("web", "/home/f/.ssh/config"),
      host("*", "/home/f/.ssh/config", ["*"]),
      host("db", "/home/f/.ssh/config.d/homelab.config"),
      host("synced", managed),
    ];
    const groups = groupHostsForMigration(hosts, managed);
    expect(groups.map((g) => g.file)).toEqual(["/home/f/.ssh/config", "/home/f/.ssh/config.d/homelab.config"]);
    expect(groups[0].hosts.map((h) => h.alias)).toEqual(["web"]);
    expect(groups[1].hosts.map((h) => h.alias)).toEqual(["db"]);
  });

  it("returns no groups when nothing is left to migrate", () => {
    expect(groupHostsForMigration([host("synced", managed)], managed)).toEqual([]);
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm test -- sync-migration`
Expected: FAIL —— 找不到模組。

- [ ] **Step 3: 實作純邏輯**

建立 `src/lib/sync-migration.ts`:

```ts
import type { HostSummary } from "@/bindings/HostSummary";
import { isWildcardOnly } from "@/lib/host-display";

/**
 * Tidy a pasted recovery phrase before the backend validates it: one line,
 * single spaces, lowercase, no list numbering or stray punctuation.
 */
export function cleanWordsInput(raw: string): string {
  return raw
    .split(/\r?\n/)
    .map((line) => line.replace(/^\s*\d+\s*[.)\-:]?\s*/, ""))
    .join(" ")
    .toLowerCase()
    .replace(/[^a-z\s\u3000]/g, " ")
    .replace(/[\s\u3000]+/g, " ")
    .trim();
}

/** Hosts that can still be moved into the synced file, grouped by their current file. */
export function groupHostsForMigration(
  hosts: HostSummary[],
  managedFile: string,
): { file: string; hosts: HostSummary[] }[] {
  const byFile = new Map<string, HostSummary[]>();
  for (const h of hosts) {
    if (h.source_file === managedFile || isWildcardOnly(h)) continue;
    const bucket = byFile.get(h.source_file);
    if (bucket) bucket.push(h);
    else byFile.set(h.source_file, [h]);
  }
  return [...byFile.entries()].map(([file, hosts]) => ({ file, hosts }));
}
```

Run: `pnpm test -- sync-migration`
Expected: 4 passed。

- [ ] **Step 4: hooks**

建立 `src/lib/sync.ts`:

```ts
import { useMutation, useQuery, useQueryClient, type QueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { DuplicateAlias } from "@/bindings/DuplicateAlias";
import type { MigrationReport } from "@/bindings/MigrationReport";
import type { SyncStatus } from "@/bindings/SyncStatus";
import { tauriInvoke } from "@/lib/ipc";

export const syncStatusKey = ["sync", "status"] as const;
export const syncDuplicatesKey = ["sync", "duplicates"] as const;

export function errorMessage(error: unknown): string {
  return error instanceof Error ? error.message : String(error);
}

/** After anything that changes chain membership: refetch status, hosts and duplicates. */
export function refreshSyncViews(queryClient: QueryClient): void {
  void queryClient.invalidateQueries({ queryKey: syncStatusKey });
  void queryClient.invalidateQueries({ queryKey: ["config"] });
  void queryClient.invalidateQueries({ queryKey: syncDuplicatesKey });
}

export function useSyncStatus(refetchInterval: number | false = false) {
  return useQuery<SyncStatus>({
    queryKey: syncStatusKey,
    queryFn: () => tauriInvoke<SyncStatus>("sync_status"),
    refetchInterval,
  });
}

/** Non-secret mutations that return the new status: prime the cache, refresh config views. */
function useStatusMutation<TVars>(cmd: string, failure: string, map: (v: TVars) => Record<string, unknown>) {
  const queryClient = useQueryClient();
  return useMutation<SyncStatus, unknown, TVars>({
    mutationFn: (vars) => tauriInvoke<SyncStatus>(cmd, map(vars)),
    onSuccess: (status) => {
      queryClient.setQueryData(syncStatusKey, status);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => toast.error(failure, { description: errorMessage(error) }),
  });
}

export function useLeaveChain() {
  return useStatusMutation<{ deleteRemote: boolean }>("sync_leave_chain", "Could not leave sync chain", ({ deleteRemote }) => ({ deleteRemote }));
}

export function useSetRelayUrl() {
  return useStatusMutation<{ url: string }>("sync_set_relay_url", "Could not update relay URL", ({ url }) => ({ url }));
}

export function useSetDeviceName() {
  return useStatusMutation<{ name: string }>("sync_set_device_name", "Could not rename this device", ({ name }) => ({ name }));
}

/** Removes a device from the list only — it is NOT revocation (see the pane copy). */
export function useForgetDevice() {
  return useStatusMutation<{ deviceId: string }>("sync_forget_device", "Could not forget device", ({ deviceId }) => ({ deviceId }));
}

export function useSyncNow() {
  return useMutation<void, unknown, void>({
    mutationFn: () => tauriInvoke<void>("sync_now"),
    onError: (error) => toast.error("Could not start sync", { description: errorMessage(error) }),
  });
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
    mutationFn: ({ aliases, tagByFile }) => tauriInvoke<MigrationReport>("sync_migrate_hosts", { aliases, tagByFile }),
    onSuccess: () => refreshSyncViews(queryClient),
    onError: (error) => toast.error("Could not move hosts", { description: errorMessage(error) }),
  });
}

export function useDuplicateAliases(enabled: boolean) {
  return useQuery<DuplicateAlias[]>({
    queryKey: syncDuplicatesKey,
    queryFn: () => tauriInvoke<DuplicateAlias[]>("sync_duplicate_aliases"),
    enabled,
  });
}

/** Rename or remove the LOCAL copy of a shadowed alias, addressed by file path (never the synced copy). */
export function useResolveShadowed() {
  const queryClient = useQueryClient();
  return useMutation<DuplicateAlias[], unknown, { alias: string; file: string; action: "rename" | "remove" }>({
    mutationFn: ({ alias, file, action }) => tauriInvoke<DuplicateAlias[]>("sync_resolve_shadowed", { alias, file, action }),
    onSuccess: (remaining) => {
      queryClient.setQueryData(syncDuplicatesKey, remaining);
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => toast.error("Could not update the local host", { description: errorMessage(error) }),
  });
}

/*
 * Recovery-phrase calls deliberately bypass TanStack Query: `useMutation` keeps
 * `variables` and `data` in its cache, so the words would linger in memory long
 * after the dialog closed. Callers hold the result in component state only and
 * drop it when the dialog closes.
 */

export function createChain(deviceName: string): Promise<string> {
  return tauriInvoke<string>("sync_create_chain", { deviceName });
}

export function joinChain(words: string, deviceName: string): Promise<SyncStatus> {
  return tauriInvoke<SyncStatus>("sync_join_chain", { words, deviceName });
}

export function showWords(): Promise<string> {
  return tauriInvoke<string>("sync_show_words");
}
```

- [ ] **Step 5: ui store 與 App 事件**

`src/stores/ui.ts`:`SettingsCategory` 聯集加 `| "sync"`;`UiState` 加:

```ts
  /** Whether the "Move hosts into sync" wizard is open. Session-only. */
  syncMigrationOpen: boolean;
  setSyncMigrationOpen: (open: boolean) => void;
```

實作加 `syncMigrationOpen: false, setSyncMigrationOpen: (syncMigrationOpen) => set({ syncMigrationOpen }),`。

`src/App.tsx`:**重用既有的 `const queryClient = useQueryClient();`**(已在 `App()` 內宣告,不要再宣告一次),在既有 `useEffect` 區加入(新增 import:`listen` from `@tauri-apps/api/event`、`syncStatusKey` from `@/lib/sync`、`tauriInvoke` from `@/lib/ipc`、`import type { SyncStatus } from "@/bindings/SyncStatus"`;`toast` 與 `useQueryClient` 已 import):

```tsx
  // Sync engine → UI: status pushes refresh the Settings pane without polling;
  // applied remote changes refresh the host list; conflicts surface as a toast;
  // regaining focus nudges a sync round.
  useEffect(() => {
    const unlisten: Array<() => void> = [];
    void listen<SyncStatus>("sync://status", (e) => queryClient.setQueryData(syncStatusKey, e.payload)).then((u) => unlisten.push(u));
    void listen<number>("sync://applied", () => {
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    }).then((u) => unlisten.push(u));
    void listen<string[]>("sync://conflict", (e) => {
      const aliases = e.payload.join(", ");
      toast.warning("Sync overwrote a local change", {
        description: `${aliases} was edited on another device more recently.`,
      });
      void queryClient.invalidateQueries({ queryKey: ["config"] });
    }).then((u) => unlisten.push(u));
    const onFocus = () => void tauriInvoke("sync_now");
    window.addEventListener("focus", onFocus);
    return () => {
      unlisten.forEach((u) => u());
      window.removeEventListener("focus", onFocus);
    };
  }, [queryClient]);
```

- [ ] **Step 6: 型別檢查、測試、Commit**

Run: `pnpm build && pnpm test`
Expected: 全綠(特別注意 `noUnusedLocals`:沒有未使用的 import)。

```bash
git add src/lib/sync.ts src/lib/sync-migration.ts src/lib/sync-migration.test.ts src/stores/ui.ts src/App.tsx
git commit -m "feat(sync): frontend hooks, event wiring and migration helpers"
```

---

### Task 2: Settings → Sync pane

**Files:**
- Create: `src/components/SyncPane.tsx`
- Modify: `src/components/SettingsDialog.tsx`(CATEGORIES 加 `{ id: "sync", label: "Sync", icon: RefreshCw }`;`{category === "sync" && <SyncPane />}`;import;`function SettingsRow` 前加 `export`)

**Interfaces:**
- Consumes: Task 1 hooks 與祕密呼叫、`Section`/`SettingsGroup`(`@/components/settings-primitives`)、`SettingsRow`(SettingsDialog 匯出)、`useSettingsStore().setFileAlias`/`fileAliases`、`useUiStore().setSyncMigrationOpen`、`copyText`(`@/lib/clipboard`)
- Produces: `<SyncPane />`

- [ ] **Step 1: 實作**

建立 `src/components/SyncPane.tsx`:

```tsx
import { useState } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { Copy, Eye, Loader2, RefreshCw, UserMinus } from "lucide-react";
import { toast } from "sonner";

import type { SyncStatus } from "@/bindings/SyncStatus";
import { cleanWordsInput } from "@/lib/sync-migration";
import {
  createChain,
  errorMessage,
  joinChain,
  refreshSyncViews,
  showWords,
  useForgetDevice,
  useLeaveChain,
  useSetDeviceName,
  useSetRelayUrl,
  useSyncNow,
  useSyncStatus,
} from "@/lib/sync";
import { copyText } from "@/lib/clipboard";
import { useSettingsStore } from "@/stores/settings";
import { useUiStore } from "@/stores/ui";
import { Section, SettingsGroup } from "@/components/settings-primitives";
import { SettingsRow } from "@/components/SettingsDialog";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/**
 * Settings → Sync. The recovery-phrase dialogs live HERE, above the joined /
 * not-joined split: creating a chain flips `joined` immediately, and a dialog
 * owned by NotJoinedPane would unmount before the user confirmed the words.
 * The words only ever live in component state — never in a query cache.
 */
export function SyncPane() {
  const status = useSyncStatus(5_000);
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);
  const [freshWords, setFreshWords] = useState<string | null>(null); // just created; must be confirmed
  const [shownWords, setShownWords] = useState<string | null>(null); // re-shown on request
  const [saved, setSaved] = useState(false);

  if (status.isLoading || !status.data) {
    return <p className="px-3 py-3 text-sm text-muted-foreground">Loading sync status…</p>;
  }

  const finishOnboarding = () => {
    setFreshWords(null);
    setSaved(false);
    setMigrationOpen(true);
  };

  return (
    <>
      {status.data.joined ? (
        <JoinedPane status={status.data} onShowWords={setShownWords} />
      ) : (
        <NotJoinedPane status={status.data} onCreated={setFreshWords} />
      )}

      <Dialog open={freshWords !== null} onOpenChange={(open) => { if (!open && saved) finishOnboarding(); }}>
        <DialogContent className="sm:max-w-lg" onEscapeKeyDown={(e) => { if (!saved) e.preventDefault(); }} onPointerDownOutside={(e) => { if (!saved) e.preventDefault(); }}>
          <DialogHeader>
            <DialogTitle>Your recovery phrase</DialogTitle>
            <DialogDescription>
              Enter these 24 words on every other device. Anyone with them can read your synced hosts, so keep them in a password manager — SSHelter can show them again from a device that is already in the chain.
            </DialogDescription>
          </DialogHeader>
          <WordGrid words={freshWords ?? ""} />
          <label className="flex items-center gap-2 text-sm">
            <Checkbox checked={saved} onCheckedChange={(v) => setSaved(v === true)} />
            I have saved these words somewhere safe
          </label>
          <DialogFooter>
            <Button type="button" disabled={!saved} onClick={finishOnboarding}>
              Continue
            </Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>

      <Dialog open={shownWords !== null} onOpenChange={(open) => { if (!open) setShownWords(null); }}>
        <DialogContent className="sm:max-w-lg">
          <DialogHeader>
            <DialogTitle>Recovery phrase</DialogTitle>
            <DialogDescription>Enter these words on the new device under Settings → Sync → Join.</DialogDescription>
          </DialogHeader>
          <WordGrid words={shownWords ?? ""} />
        </DialogContent>
      </Dialog>
    </>
  );
}

/** Advanced row shared by both panes: the relay must be reachable BEFORE create/join. */
function RelayUrlRow({ current }: { current: string }) {
  const setRelayUrl = useSetRelayUrl();
  const [draft, setDraft] = useState(current);
  return (
    <SettingsRow id="sync-relay" label="Relay URL" description="https:// only (plain http is allowed for localhost). Self-host from the repository's relay/ folder.">
      <div className="flex items-center gap-1.5">
        <Input id="sync-relay" value={draft} onChange={(e) => setDraft(e.target.value)} className="h-7 w-64 font-mono text-xs" />
        <Button type="button" variant="secondary" size="sm" className="h-7" disabled={draft.trim() === current || setRelayUrl.isPending} onClick={() => setRelayUrl.mutate({ url: draft })}>
          Save
        </Button>
      </div>
    </SettingsRow>
  );
}

/** Create or join. Errors keep the form as it was so the user can fix a typo. */
function NotJoinedPane({ status, onCreated }: { status: SyncStatus; onCreated: (words: string) => void }) {
  const queryClient = useQueryClient();
  const [deviceName, setDeviceName] = useState(status.device_name);
  const [words, setWords] = useState("");
  const [busy, setBusy] = useState<"create" | "join" | null>(null);
  const leave = useLeaveChain();
  const setFileAlias = useSettingsStore((s) => s.setFileAlias);
  const fileAliases = useSettingsStore((s) => s.fileAliases);
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);

  const labelManagedFile = () => {
    if (status.managed_file && !fileAliases[status.managed_file]) setFileAlias(status.managed_file, "Synced");
  };

  const onCreate = async () => {
    setBusy("create");
    try {
      const phrase = await createChain(deviceName);
      labelManagedFile();
      onCreated(phrase);
      refreshSyncViews(queryClient);
    } catch (error) {
      toast.error("Could not create sync chain", { description: errorMessage(error) });
    } finally {
      setBusy(null);
    }
  };

  const onJoin = async () => {
    setBusy("join");
    try {
      await joinChain(cleanWordsInput(words), deviceName);
      labelManagedFile();
      setWords("");
      toast.success("Joined the sync chain");
      refreshSyncViews(queryClient);
      setMigrationOpen(true);
    } catch (error) {
      // Keep the pasted words so a typo can be fixed.
      toast.error("Could not join sync chain", { description: errorMessage(error) });
    } finally {
      setBusy(null);
    }
  };

  const wordCount = cleanWordsInput(words) === "" ? 0 : cleanWordsInput(words).split(" ").length;

  return (
    <>
      {status.phrase_cleanup_pending && (
        <Section title="Cleanup needed" description="You left the chain, but the recovery phrase is still in the keychain.">
          <SettingsGroup>
            <SettingsRow label="Recovery phrase" description="Retry removing it from the OS keychain.">
              <Button type="button" variant="outline" size="sm" className="h-7" disabled={leave.isPending} onClick={() => leave.mutate({ deleteRemote: false })}>
                Remove phrase
              </Button>
            </SettingsRow>
          </SettingsGroup>
        </Section>
      )}

      <Section
        title="Sync chain"
        description="Keep hosts in sync across your computers without an account. A 24-word recovery phrase is the only secret; the relay only ever stores encrypted records."
      >
        <SettingsGroup>
          <SettingsRow id="sync-device-name" label="This device" description="Shown to your other devices.">
            <Input id="sync-device-name" value={deviceName} onChange={(e) => setDeviceName(e.target.value)} className="h-7 w-48 text-sm" />
          </SettingsRow>
          <SettingsRow label="Start a new chain" description="Creates the recovery phrase you will enter on other devices.">
            <Button type="button" size="sm" className="h-7" disabled={busy !== null || deviceName.trim() === ""} onClick={() => void onCreate()}>
              {busy === "create" && <Loader2 className="size-3.5 animate-spin" />} Create
            </Button>
          </SettingsRow>
        </SettingsGroup>
      </Section>

      <Section title="Join an existing chain" description="Paste the 24 words from another device. Nothing is sent until you press Join.">
        <div className="space-y-2">
          <Textarea
            value={words}
            onChange={(e) => setWords(e.target.value)}
            placeholder="abandon ability able …"
            rows={3}
            className="font-mono text-sm"
            aria-label="Recovery phrase"
            autoCorrect="off"
            autoCapitalize="off"
            spellCheck={false}
          />
          <Button type="button" size="sm" className="h-7" disabled={busy !== null || wordCount !== 24 || deviceName.trim() === ""} onClick={() => void onJoin()}>
            {busy === "join" && <Loader2 className="size-3.5 animate-spin" />} Join
          </Button>
        </div>
      </Section>

      <Section title="Advanced" description="Change this before creating or joining if you self-host the relay or the default one is unreachable.">
        <SettingsGroup>
          <RelayUrlRow current={status.relay_url} />
        </SettingsGroup>
      </Section>
    </>
  );
}

function WordGrid({ words }: { words: string }) {
  const list = words.split(" ");
  const copy = async () => {
    try {
      await copyText(words);
      toast.success("Recovery phrase copied — clear your clipboard when done");
    } catch (error) {
      toast.error("Clipboard unavailable", { description: errorMessage(error) });
    }
  };
  return (
    <div className="space-y-2">
      <ol className="grid grid-cols-3 gap-x-4 gap-y-1 rounded-md border bg-muted/40 p-3 font-mono text-sm select-text">
        {list.map((w, i) => (
          <li key={`${i}-${w}`} className="flex gap-2">
            <span className="w-5 text-right text-muted-foreground tabular-nums">{i + 1}</span>
            {w}
          </li>
        ))}
      </ol>
      <Button type="button" variant="outline" size="sm" className="h-7" onClick={() => void copy()}>
        <Copy className="size-3.5" /> Copy
      </Button>
    </div>
  );
}

function JoinedPane({ status: s, onShowWords }: { status: SyncStatus; onShowWords: (words: string) => void }) {
  const syncNow = useSyncNow();
  const leave = useLeaveChain();
  const forget = useForgetDevice();
  const setDeviceName = useSetDeviceName();
  const setMigrationOpen = useUiStore((st) => st.setSyncMigrationOpen);
  const [revealing, setRevealing] = useState(false);
  const [leaveOpen, setLeaveOpen] = useState(false);
  const [deleteRemote, setDeleteRemote] = useState(false);
  const [nameDraft, setNameDraft] = useState(s.device_name);

  const lastSync = s.last_sync_ms ? new Date(s.last_sync_ms).toLocaleString() : "never";

  const reveal = async () => {
    setRevealing(true);
    try {
      onShowWords(await showWords());
    } catch (error) {
      toast.error("Could not read recovery phrase", { description: errorMessage(error) });
    } finally {
      setRevealing(false);
    }
  };

  return (
    <>
      <Section title="Sync chain" description={`Chain ${s.chain_short ?? ""} · ${s.hosts_in_sync} hosts in sync · last sync ${lastSync}`}>
        <SettingsGroup>
          <SettingsRow label="Status" description={s.last_error ?? (s.pending > 0 ? `${s.pending} change${s.pending === 1 ? "" : "s"} waiting to upload` : "Up to date")}>
            <div className="flex items-center gap-1.5">
              <Badge variant={s.last_error ? "destructive" : s.read_only ? "outline" : "secondary"}>
                {s.read_only ? "Read-only" : s.last_error ? "Error" : "Synced"}
              </Badge>
              <Button type="button" variant="ghost" size="icon" className="size-7" aria-label="Sync now" disabled={syncNow.isPending} onClick={() => syncNow.mutate()}>
                <RefreshCw className="size-3.5" />
              </Button>
            </div>
          </SettingsRow>
          <SettingsRow id="sync-name" label="This device">
            <div className="flex items-center gap-1.5">
              <Input id="sync-name" value={nameDraft} onChange={(e) => setNameDraft(e.target.value)} className="h-7 w-40 text-sm" />
              <Button type="button" variant="secondary" size="sm" className="h-7" disabled={nameDraft.trim() === "" || nameDraft === s.device_name || setDeviceName.isPending} onClick={() => setDeviceName.mutate({ name: nameDraft })}>
                Rename
              </Button>
            </div>
          </SettingsRow>
          <SettingsRow label="Move hosts into sync" description="Choose which existing hosts should follow you to other devices.">
            <Button type="button" variant="outline" size="sm" className="h-7" onClick={() => setMigrationOpen(true)}>
              Choose hosts…
            </Button>
          </SettingsRow>
          <SettingsRow label="Recovery phrase" description="Needed to add another device. Shown only on request.">
            <Button type="button" variant="outline" size="sm" className="h-7" disabled={revealing} onClick={() => void reveal()}>
              <Eye className="size-3.5" /> Show
            </Button>
          </SettingsRow>
        </SettingsGroup>
      </Section>

      <Section
        title="Devices"
        description="Every device that joined this chain. Forget only removes a device from this list — a device that still has the recovery phrase keeps syncing. If a device was lost, leave this chain, start a new one on the devices you keep, and rotate the keys it could see."
      >
        <SettingsGroup>
          {s.devices.map((d) => (
            <SettingsRow key={d.id} label={d.name + (d.is_this ? " (this device)" : "")} description={`${d.platform} · last seen ${new Date(d.last_seen_ms).toLocaleString()}`}>
              {!d.is_this && (
                <Button type="button" variant="ghost" size="sm" className="h-7 text-muted-foreground" aria-label={`Forget ${d.name}`} disabled={forget.isPending} onClick={() => forget.mutate({ deviceId: d.id })}>
                  <UserMinus className="size-3.5" /> Forget
                </Button>
              )}
            </SettingsRow>
          ))}
        </SettingsGroup>
      </Section>

      <Section title="Advanced" description="The relay only stores encrypted records.">
        <SettingsGroup>
          <RelayUrlRow current={s.relay_url} />
          <SettingsRow label="Leave chain" description="This device keeps every file it has; it just stops syncing.">
            <Button type="button" variant="outline" size="sm" className="h-7 text-destructive hover:text-destructive" onClick={() => setLeaveOpen(true)}>
              Leave…
            </Button>
          </SettingsRow>
        </SettingsGroup>
      </Section>

      <AlertDialog open={leaveOpen} onOpenChange={setLeaveOpen}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>Leave the sync chain?</AlertDialogTitle>
            <AlertDialogDescription>
              Your hosts stay in <span className="font-mono">{s.managed_file}</span> and keep working. Rejoining later needs the recovery phrase.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <label className="flex items-center gap-2 text-sm">
            <Checkbox checked={deleteRemote} onCheckedChange={(v) => setDeleteRemote(v === true)} />
            Also delete the chain from the relay (other devices will stop syncing)
          </label>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction onClick={() => leave.mutate({ deleteRemote }, { onSuccess: () => { setLeaveOpen(false); toast.success("Left the sync chain"); } })}>
              Leave
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </>
  );
}
```

`SettingsDialog.tsx`:`import { RefreshCw } from "lucide-react"`(併入既有 lucide import)、`import { SyncPane } from "@/components/SyncPane"`;CATEGORIES 在 `ai` 之後加 `{ id: "sync", label: "Sync", icon: RefreshCw }`;分頁渲染加 `{category === "sync" && <SyncPane />}`;`function SettingsRow` 改為 `export function SettingsRow`。

> `lucide-react` 是否有 `UserMinus`:先 `grep -c "UserMinus" node_modules/lucide-react/dist/lucide-react.d.ts`;沒有就改用 `X`。不要留下任何未使用的 import(`noUnusedLocals`)。

- [ ] **Step 2: 型別檢查與手動驗證**

Run: `pnpm build && pnpm test`
Expected: 全綠。

Run: `cd relay && npm run dev`(另一終端)+ `pnpm tauri dev`:Settings → Sync → 先把 Relay URL 改成 `http://sync.example.com` → 儲存被拒(https 訊息);改回 `http://127.0.0.1:8787` → OK。輸入裝置名 → Create → **狀態列已是 Synced、Devices 已含本機,但 24 詞視窗仍在**;未勾選不可關閉(Esc/點外面無效)→ 勾選 Continue → 遷入 wizard 開啟。Show 再次顯示同一組詞。用亂序/錯字的 24 詞 Join(先 Leave)→ toast 可讀錯誤、textarea 內容保留。把 relay 關掉再按 Sync now → 狀態變 Error 且 app 其他功能正常;Leave(不勾刪除)→ 回到未加入畫面,`~/.ssh/sshelter/hosts.config` 仍在。

- [ ] **Step 3: Commit**

```bash
git add src/components/SyncPane.tsx src/components/SettingsDialog.tsx
git commit -m "feat(sync): settings pane to create, join, inspect and leave a sync chain"
```

---

### Task 3: 遷入 wizard 與同名主機處理(檔案定位)

**Files:**
- Create: `src/components/SyncMigrationDialog.tsx`
- Modify: `src/App.tsx`(掛載 `<SyncMigrationDialog />`)

**Interfaces:**
- Consumes: `groupHostsForMigration`、`useMigrateHosts`、`useDuplicateAliases`、`useResolveShadowed`、`useSyncStatus`、`useHostsQuery`、`labelsFor`(`@/lib/host-display`)、`basename`(`@/lib/utils`)、ui store `syncMigrationOpen`
- Produces: `<SyncMigrationDialog />`

- [ ] **Step 1: 實作**

建立 `src/components/SyncMigrationDialog.tsx`:

```tsx
import { useEffect, useMemo, useState } from "react";
import { Loader2 } from "lucide-react";
import { toast } from "sonner";

import { useHostsQuery } from "@/lib/queries";
import { labelsFor } from "@/lib/host-display";
import { groupHostsForMigration } from "@/lib/sync-migration";
import { useDuplicateAliases, useMigrateHosts, useResolveShadowed, useSyncStatus } from "@/lib/sync";
import { useSettingsStore } from "@/stores/settings";
import { useUiStore } from "@/stores/ui";
import { basename } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/**
 * "Move hosts into sync": pick existing hosts (grouped by file) to move into the
 * synced file, optionally tagging them with their old file's name; then resolve
 * aliases that the synced file now shadows — addressed by file, never by
 * first-match, so the synced copy is never touched.
 */
export function SyncMigrationDialog() {
  const open = useUiStore((s) => s.syncMigrationOpen);
  const setOpen = useUiStore((s) => s.setSyncMigrationOpen);
  return (
    <Dialog open={open} onOpenChange={setOpen}>
      <DialogContent className="sm:max-w-lg">{open && <MigrationFlow onClose={() => setOpen(false)} />}</DialogContent>
    </Dialog>
  );
}

function MigrationFlow({ onClose }: { onClose: () => void }) {
  const status = useSyncStatus();
  const hostsQuery = useHostsQuery();
  const fileAliases = useSettingsStore((s) => s.fileAliases);
  const migrate = useMigrateHosts();
  const duplicates = useDuplicateAliases(true);
  const resolve = useResolveShadowed();
  const [tagByFile, setTagByFile] = useState(true);
  const [selected, setSelected] = useState<Set<string>>(new Set());

  const managed = status.data?.managed_file ?? "";
  const files = useMemo(() => hostsQuery.data?.files ?? [], [hostsQuery.data]);
  const labels = useMemo(() => labelsFor(files, fileAliases), [files, fileAliases]);
  const groups = useMemo(() => groupHostsForMigration(hostsQuery.data?.hosts ?? [], managed), [hostsQuery.data, managed]);

  // Default to everything selected the first time the list is known.
  useEffect(() => {
    if (selected.size === 0 && groups.length > 0) {
      setSelected(new Set(groups.flatMap((g) => g.hosts.map((h) => h.alias))));
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [groups.length]);

  const toggle = (alias: string, on: boolean) =>
    setSelected((prev) => {
      const next = new Set(prev);
      if (on) next.add(alias);
      else next.delete(alias);
      return next;
    });

  const run = () =>
    migrate.mutate(
      { aliases: [...selected], tagByFile },
      {
        onSuccess: (report) => {
          const failed = report.failed.length;
          toast.success(`Moved ${report.moved.length} host${report.moved.length === 1 ? "" : "s"} into sync${failed ? `, ${failed} failed` : ""}`);
          setSelected(new Set());
        },
      },
    );

  const dups = duplicates.data ?? [];

  return (
    <>
      <DialogHeader>
        <DialogTitle>Move hosts into sync</DialogTitle>
        <DialogDescription>
          Selected hosts move into <span className="font-mono">{basename(managed)}</span> (a backup is written first) and appear on every device in the chain. Wildcard blocks stay where they are.
        </DialogDescription>
      </DialogHeader>

      {groups.length === 0 ? (
        <p className="text-sm text-muted-foreground">Every host is already in the synced file.</p>
      ) : (
        <div className="max-h-[40vh] space-y-3 overflow-y-auto pr-1">
          {groups.map((g) => (
            <div key={g.file} className="space-y-1">
              <p className="text-xs font-semibold tracking-wide text-muted-foreground uppercase">{labels.get(g.file) ?? basename(g.file)}</p>
              {g.hosts.map((h) => (
                <label key={h.alias} className="flex items-center gap-2 text-sm">
                  <Checkbox checked={selected.has(h.alias)} onCheckedChange={(v) => toggle(h.alias, v === true)} />
                  <span className="font-mono">{h.alias}</span>
                  {h.hostname && <span className="truncate text-xs text-muted-foreground">{h.user ? `${h.user}@` : ""}{h.hostname}</span>}
                </label>
              ))}
            </div>
          ))}
        </div>
      )}

      {groups.length > 0 && (
        <label className="flex items-center gap-2 text-sm">
          <Checkbox checked={tagByFile} onCheckedChange={(v) => setTagByFile(v === true)} />
          Tag each host with its current file name (keeps your grouping in tag view)
        </label>
      )}

      {dups.length > 0 && (
        <div className="space-y-1.5 rounded-md border border-amber-500/40 bg-amber-500/10 p-3 text-xs">
          <p className="font-medium text-amber-700 dark:text-amber-400">Synced hosts shadow local definitions</p>
          <p className="text-muted-foreground">The synced file is included first, so ssh ignores these local blocks. Keep the local one under a new name, or remove it and use the synced version:</p>
          {dups.map((d) => (
            <div key={`${d.alias}-${d.local_file}`} className="flex items-center justify-between gap-2">
              <span className="font-mono">{d.alias} <span className="text-muted-foreground">in {basename(d.local_file)}</span></span>
              <div className="flex gap-1">
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs" disabled={resolve.isPending} onClick={() => resolve.mutate({ alias: d.alias, file: d.local_file, action: "rename" })}>
                  Keep as {d.alias}-local
                </Button>
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs text-destructive" disabled={resolve.isPending} onClick={() => resolve.mutate({ alias: d.alias, file: d.local_file, action: "remove" })}>
                  Remove local
                </Button>
              </div>
            </div>
          ))}
        </div>
      )}

      <DialogFooter>
        <Button type="button" variant="outline" onClick={onClose}>Close</Button>
        {groups.length > 0 && (
          <Button type="button" disabled={selected.size === 0 || migrate.isPending} onClick={run}>
            {migrate.isPending && <Loader2 className="size-4 animate-spin" />} Move {selected.size} host{selected.size === 1 ? "" : "s"}
          </Button>
        )}
      </DialogFooter>
    </>
  );
}
```

`src/App.tsx`:`import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";` 並在 `<NewConfigFileDialog />` 後掛 `<SyncMigrationDialog />`。

- [ ] **Step 2: 型別檢查、測試、手動驗證**

Run: `pnpm build && pnpm test`
Expected: 全綠。

手動:在已加入的裝置開 Settings → Sync → Choose hosts… → 清單依檔案分組、預設全選、wildcard 不出現 → Move → toast 顯示數量;sidebar 依 tag 模式看到原檔名 tag;在主 config 手動加一個與同步主機同名的 `Host`,重開對話框 → amber 區塊出現 → 「Keep as -local」後主 config 那份改名、`hosts.config` 那份原封不動(`cat` 兩個檔案確認);「Remove local」只刪主 config 那份。

- [ ] **Step 3: Commit**

```bash
git add src/components/SyncMigrationDialog.tsx src/App.tsx
git commit -m "feat(sync): migration wizard with file-addressed shadowed alias handling"
```

---

### Task 4: 文件與端到端驗證清單

**Files:**
- Modify: `README.md`(Features 加 Sync 條目;新增 `## Sync` 段落)
- Create: `docs/superpowers/plans/2026-09-27-sync-chain-manual-verification.md`

- [ ] **Step 1: README**

Features 清單(在 AI Access 條目之後)加:

```markdown
- **Sync (no account)** — create a sync chain on one computer, enter its 24-word recovery phrase on the others, and your synced hosts follow you. Records are end-to-end encrypted before they reach the relay (which is open source and self-hostable); private keys and passwords are opt-in and never leave a device unencrypted.
```

新增段落(在 `## AI Access (MCP)` 之後):

```markdown
## Sync

Open **Settings → Sync**. *Create* shows a 24-word recovery phrase — store it in a password manager; it is the only secret and anyone holding it can read your synced hosts. On another computer choose *Join* and paste the words.

Synced hosts live in `~/.ssh/sshelter/hosts.config`, which SSHelter `Include`s at the top of your main config, so plain `ssh` keeps working and the file survives uninstalling SSHelter. Use *Choose hosts…* to move existing hosts in (optionally tagged with their old file name). Hosts in other files stay local to that computer.

*Forget device* only removes a device from the list; a device that still has the phrase keeps syncing. If a device is lost, leave the chain, start a new one on the devices you keep, and rotate the keys it could see.

The relay stores only ciphertext and can be self-hosted from `relay/` (`npx wrangler deploy`); point *Settings → Sync → Relay URL* at yours (`https://` required, except `localhost` for development).
```

- [ ] **Step 2: 驗證清單文件**

建立 `docs/superpowers/plans/2026-09-27-sync-chain-manual-verification.md`,內容為下列清單(執行時逐項填結果):

```markdown
# Sync chain — manual end-to-end verification (Phase A)

Setup: relay `cd relay && npm run dev` reachable from both devices. Device A and device B must be
**two OS user accounts, a VM, or two physical machines** — a second checkout with a custom config
path is NOT a second device: `~/.ssh/sshelter/`, `sync-state.json`, the device id and the keychain
entry all follow the OS user, so two processes in one account would fight over the same sync assets.

1. A: Settings → Sync → Relay URL `http://sync.example.com` is refused (https message); the local
   `http://127.0.0.1:8787` is accepted. Create chain → the words dialog is still open while Status
   already says Synced and Devices lists A → cannot close until confirmed → Continue opens the
   migration wizard.
2. A: Choose hosts… → move two hosts with "tag by file" → hosts.config contains both, tags added,
   main config's first non-comment line is `Include ~/.ssh/sshelter/hosts.config` (above any
   existing Include).
3. B: Join with the words (paste with numbering) → migration dialog opens → hosts from A appear in
   B's sidebar under "Synced" within 45 s **without pressing reload**; `ssh -G <alias>` on B
   resolves the synced HostName.
4. B: edit a synced host → A's sidebar/editor shows the change within 45 s (no manual reload).
5. A and B offline (stop relay): edit the same host on both; A edits first, B second; start relay →
   B's text wins on both; A shows the "Sync overwrote a local change" toast.
6. Hand-edit hosts.config on A with a text editor → A uploads the change (pending → 0) without
   needing an app reload. While the relay is stopped, save an edit in A's UI, then start the relay:
   the edit is still there afterwards (the round that raced it was discarded and re-run).
7. A: Forget device B → B disappears from A's device list; B's next sync still works (by design —
   Forget is not revocation, the copy says so). B: Leave chain → B keeps hosts.config; Join again
   with the words → back in sync.
8. Wrong phrase on Join (valid words, wrong chain) → "no sync chain matches" error, form keeps the
   input, and the relay's `.wrangler/state` gains no new chain. Relay stopped → Status "Error",
   editing/connecting still works.
9. B: add a local `Host <synced alias>` to the main config → wizard shows the amber "shadow" block →
   "Keep as -local" renames only the main-config copy; "Remove local" removes only that copy.
10. Windows build: repeat 3–4 on the Windows device (paths under `C:\Users\…\.ssh\sshelter\`).
```

- [ ] **Step 3: 全套與 Commit**

Run: `pnpm build && pnpm test && (cd src-tauri && cargo test 2>&1 | grep 'test result')`
Expected: 全綠。

```bash
git add README.md docs/superpowers/plans/2026-09-27-sync-chain-manual-verification.md
git commit -m "docs(sync): describe sync chain setup and record the manual verification checklist"
```

---

## Self-review(已執行)

- **Spec 覆蓋**:§7 Settings Sync pane(未加入含 relay URL/建立後的確認畫面在父層/已加入、Show pairing code、裝置清單 Forget、relay URL、Leave、phrase cleanup 重試)→ Task 2;§7 新裝置上手(Join → 遷入對話框)與 §10 wizard 主機/tag 部分、檔案定位的同名主機處理 → Task 3;事件(status/applied/conflict)與焦點觸發 → Task 1;「Synced」預設顯示名 → Task 2 `labelManagedFile`;§2 Forget 不是撤權的文案 → Task 2 + README;§7 Keys dialog 開關與金鑰警示、`Sync passwords` 開關屬 Phase B,刻意不在此。
- **型別一致**:hooks 的 command 名稱與 A3 註冊清單一致(`sync_status`、`sync_create_chain`、`sync_join_chain`、`sync_show_words`、`sync_leave_chain`、`sync_now`、`sync_set_relay_url`、`sync_set_device_name`、`sync_forget_device`、`sync_migrate_hosts`、`sync_duplicate_aliases`、`sync_resolve_shadowed`);`SyncStatus` 欄位(`managed_file`、`hosts_in_sync`、`devices[].is_this`、`phrase_cleanup_pending`、`read_only`)與 A3 `status_from` 一致;`sync_resolve_shadowed` 的 `action` 字串與 A3 `ShadowedAction` 的 serde 小寫一致;`sync://applied` payload 為數字。
- **Review Focus 對應**:1 → Task 1 `cleanWordsInput` 測試;2 → Task 1 `groupHostsForMigration` 測試;3 → Task 2 的 `freshWords`/`saved` 在 `SyncPane`(escape/outside 阻擋)+ 手動驗證第 1 項;4 → Task 2 `onJoin` catch 保留 `words` + 手動第 8 項;5 → Task 2 Leave 對話框預設 `deleteRemote=false` 與說明文字;6 → Task 2 `RelayUrlRow` 在 `NotJoinedPane`;7 → Task 1 `sync://applied` listener + 手動第 3/4 項。
- **Codex review(2026-09-27)已納入**:確認畫面被卸載(11)、shadowed 操作以第一個命中定位(12 → `useResolveShadowed` + A3 `sync_resolve_shadowed`)、一般遠端更新不刷新 cache(25 → `sync://applied`)、relay URL 只能在 joined 改(26)、useMutation 快取助記詞(27 → 直接 invoke + local state)、雙 checkout 不是兩台裝置(28)、重複宣告 `queryClient` 與未使用 `Label` import(29)、Forget device 文案(1)。
