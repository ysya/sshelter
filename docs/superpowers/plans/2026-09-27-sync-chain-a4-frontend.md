# Sync Chain — Phase A4(前端:Sync pane、配對/上手、遷入 wizard、事件、文件)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓使用者從 Settings → Sync 建立/加入 chain、看到裝置與同步狀態、把既有主機遷入同步檔、處理被遮蔽的同名主機;並把後端事件接進 UI。

**Architecture:** 沿用 `src/lib/mcp.ts` + `McpPane` 的模式:TanStack Query 包 Tauri commands、Settings 分頁輪詢狀態;新增 `SyncPane.tsx`(獨立檔案,避免 `SettingsDialog.tsx` 再長)與 `SyncMigrationDialog.tsx`;`App.tsx` 監聽 `sync://status`/`sync://conflict` 事件與視窗焦點。純邏輯(遷入分組、助記詞輸入整理)抽成 `src/lib/sync-migration.ts` 以 vitest 覆蓋。

**Tech Stack:** React + TypeScript、TanStack Query、Zustand、shadcn/ui 既有元件(Dialog、AlertDialog、Checkbox、Textarea、Switch、Badge、Button、Input)、`@tauri-apps/api/event`。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §7(前端)、§10(migration)

## Global Constraints

- UI 文案英文;既有元件與樣式慣例(`Section`/`SettingsGroup`/`SettingsRow`);不新增 npm 相依。
- 助記詞只在使用者明確按下按鈕時顯示,顯示區塊不可被 clipboard 以外的方式自動複製;不進 toast、不進 console。
- `pnpm build`(tsc + vite)與 `pnpm test` 全綠;每 task 一個 commit。
- 所有 Tauri 呼叫走 `tauriInvoke`;型別以 `src/bindings/*.ts` 為準(A3 生成)。

## Review Focus

1. 使用者把 24 詞貼成多行、含逗號或編號(`1. abandon`)—— 前端先整理成單行空白分隔再送後端(Task 1 `cleanWordsInput` 測試)。
2. 遷入清單必須排除 wildcard 區塊與已在同步檔的主機(Task 1 `groupHostsForMigration` 測試)。
3. 建立 chain 後使用者關掉助記詞視窗前未確認 —— 必須有「I have saved these words」勾選才可關閉(Task 2)。
4. 加入 chain 時 relay 404(助記詞錯)—— 顯示可讀錯誤、表單保留輸入(Task 2)。
5. 離開 chain 時「Delete from relay」預設不勾、且說明會影響其他裝置(Task 2)。

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
  - `useSyncStatus(refetchInterval)`、`useCreateChain()`、`useJoinChain()`、`useLeaveChain()`、`useSyncNow()`、`useSetRelayUrl()`、`useSetDeviceName()`、`useRemoveDevice()`、`useShowWords()`、`useMigrateHosts()`、`useDuplicateAliases(enabled)`
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
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { toast } from "sonner";

import type { DuplicateAlias } from "@/bindings/DuplicateAlias";
import type { MigrationReport } from "@/bindings/MigrationReport";
import type { SyncStatus } from "@/bindings/SyncStatus";
import { tauriInvoke } from "@/lib/ipc";

export const syncStatusKey = ["sync", "status"] as const;
export const syncDuplicatesKey = ["sync", "duplicates"] as const;

function errorMessage(error: unknown): string {
  return error instanceof Error ? error.message : String(error);
}

export function useSyncStatus(refetchInterval: number | false = false) {
  return useQuery<SyncStatus>({
    queryKey: syncStatusKey,
    queryFn: () => tauriInvoke<SyncStatus>("sync_status"),
    refetchInterval,
  });
}

function useStatusMutation<TVars>(cmd: string, failure: string, map: (v: TVars) => Record<string, unknown>) {
  const queryClient = useQueryClient();
  return useMutation<SyncStatus, unknown, TVars>({
    mutationFn: (vars) => tauriInvoke<SyncStatus>(cmd, map(vars)),
    onSuccess: (status) => {
      queryClient.setQueryData(syncStatusKey, status);
      queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => toast.error(failure, { description: errorMessage(error) }),
  });
}

/** Returns the 24 recovery words; the caller shows them exactly once. */
export function useCreateChain() {
  const queryClient = useQueryClient();
  return useMutation<string, unknown, { deviceName: string }>({
    mutationFn: ({ deviceName }) => tauriInvoke<string>("sync_create_chain", { deviceName }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: syncStatusKey });
      queryClient.invalidateQueries({ queryKey: ["config"] });
    },
    onError: (error) => toast.error("Could not create sync chain", { description: errorMessage(error) }),
  });
}

export function useJoinChain() {
  return useStatusMutation<{ words: string; deviceName: string }>(
    "sync_join_chain",
    "Could not join sync chain",
    ({ words, deviceName }) => ({ words, deviceName }),
  );
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

export function useRemoveDevice() {
  return useStatusMutation<{ deviceId: string }>("sync_remove_device", "Could not remove device", ({ deviceId }) => ({ deviceId }));
}

export function useSyncNow() {
  return useMutation<void, unknown, void>({
    mutationFn: () => tauriInvoke<void>("sync_now"),
    onError: (error) => toast.error("Could not start sync", { description: errorMessage(error) }),
  });
}

/** Deliberately a mutation: the phrase is read only on an explicit click, never cached. */
export function useShowWords() {
  return useMutation<string, unknown, void>({
    mutationFn: () => tauriInvoke<string>("sync_show_words"),
    onError: (error) => toast.error("Could not read recovery phrase", { description: errorMessage(error) }),
  });
}

export function useMigrateHosts() {
  const queryClient = useQueryClient();
  return useMutation<MigrationReport, unknown, { aliases: string[]; tagByFile: boolean }>({
    mutationFn: ({ aliases, tagByFile }) => tauriInvoke<MigrationReport>("sync_migrate_hosts", { aliases, tagByFile }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["config"] });
      queryClient.invalidateQueries({ queryKey: syncStatusKey });
      queryClient.invalidateQueries({ queryKey: syncDuplicatesKey });
    },
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
```

- [ ] **Step 5: ui store 與 App 事件**

`src/stores/ui.ts`:`SettingsCategory` 聯集加 `| "sync"`;`UiState` 加:

```ts
  /** Whether the "Move hosts into sync" wizard is open. Session-only. */
  syncMigrationOpen: boolean;
  setSyncMigrationOpen: (open: boolean) => void;
```

實作加 `syncMigrationOpen: false, setSyncMigrationOpen: (syncMigrationOpen) => set({ syncMigrationOpen }),`。

`src/App.tsx`:在既有 `useEffect` 區加入(import `listen` from `@tauri-apps/api/event`、`useQueryClient`、`toast`、`syncStatusKey`、`tauriInvoke`):

```tsx
  // Sync engine → UI: status pushes refresh the Settings pane without polling;
  // conflicts surface as a toast; regaining focus nudges a sync round.
  const queryClient = useQueryClient();
  useEffect(() => {
    const unlisten: Array<() => void> = [];
    void listen<SyncStatus>("sync://status", (e) => queryClient.setQueryData(syncStatusKey, e.payload)).then((u) => unlisten.push(u));
    void listen<string[]>("sync://conflict", (e) => {
      const aliases = e.payload.join(", ");
      toast.warning("Sync overwrote a local change", {
        description: `${aliases} was edited on another device more recently.`,
      });
      queryClient.invalidateQueries({ queryKey: ["config"] });
    }).then((u) => unlisten.push(u));
    const onFocus = () => void tauriInvoke("sync_now");
    window.addEventListener("focus", onFocus);
    return () => {
      unlisten.forEach((u) => u());
      window.removeEventListener("focus", onFocus);
    };
  }, [queryClient]);
```

(`import type { SyncStatus } from "@/bindings/SyncStatus";`)

- [ ] **Step 6: 型別檢查、測試、Commit**

Run: `pnpm build && pnpm test`
Expected: 全綠。

```bash
git add src/lib/sync.ts src/lib/sync-migration.ts src/lib/sync-migration.test.ts src/stores/ui.ts src/App.tsx
git commit -m "feat(sync): frontend hooks, event wiring and migration helpers"
```

---

### Task 2: Settings → Sync pane

**Files:**
- Create: `src/components/SyncPane.tsx`
- Modify: `src/components/SettingsDialog.tsx`(CATEGORIES 加 `{ id: "sync", label: "Sync", icon: RefreshCw }`;`{category === "sync" && <SyncPane />}`;import)

**Interfaces:**
- Consumes: Task 1 hooks、`Section`/`SettingsGroup`(`@/components/settings-primitives`)、`SettingsRow`(SettingsDialog 內部元件 —— 匯出它:在 `SettingsDialog.tsx` 的 `function SettingsRow` 前加 `export`)、`useSettingsStore().setFileAlias`、`useUiStore().setSyncMigrationOpen`
- Produces: `<SyncPane />`

- [ ] **Step 1: 實作**

建立 `src/components/SyncPane.tsx`:

```tsx
import { useState } from "react";
import { Copy, Eye, Loader2, RefreshCw, Trash2 } from "lucide-react";
import { toast } from "sonner";

import { cleanWordsInput } from "@/lib/sync-migration";
import {
  useCreateChain,
  useJoinChain,
  useLeaveChain,
  useRemoveDevice,
  useSetDeviceName,
  useSetRelayUrl,
  useShowWords,
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
import { Label } from "@/components/ui/label";
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

export function SyncPane() {
  const status = useSyncStatus(5_000);
  if (status.isLoading || !status.data) {
    return <p className="px-3 py-3 text-sm text-muted-foreground">Loading sync status…</p>;
  }
  return status.data.joined ? <JoinedPane /> : <NotJoinedPane defaultDeviceName={status.data.device_name} />;
}

/** Create or join. The 24 words are shown exactly once, behind an explicit confirmation. */
function NotJoinedPane({ defaultDeviceName }: { defaultDeviceName: string }) {
  const [deviceName, setDeviceName] = useState(defaultDeviceName);
  const [words, setWords] = useState("");
  const [shownWords, setShownWords] = useState<string | null>(null);
  const [saved, setSaved] = useState(false);
  const create = useCreateChain();
  const join = useJoinChain();
  const setFileAlias = useSettingsStore((s) => s.setFileAlias);
  const fileAliases = useSettingsStore((s) => s.fileAliases);
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);
  const status = useSyncStatus();

  const labelManagedFile = () => {
    const file = status.data?.managed_file;
    if (file && !fileAliases[file]) setFileAlias(file, "Synced");
  };

  const onCreate = () =>
    create.mutate(
      { deviceName },
      {
        onSuccess: (phrase) => {
          setShownWords(phrase);
          labelManagedFile();
        },
      },
    );

  const onJoin = () =>
    join.mutate(
      { words: cleanWordsInput(words), deviceName },
      {
        onSuccess: () => {
          labelManagedFile();
          toast.success("Joined the sync chain");
          setMigrationOpen(true);
        },
      },
    );

  return (
    <>
      <Section
        title="Sync chain"
        description="Keep hosts in sync across your computers without an account. A 24-word recovery phrase is the only secret; the relay only ever stores encrypted records."
      >
        <SettingsGroup>
          <SettingsRow id="sync-device-name" label="This device" description="Shown to your other devices.">
            <Input id="sync-device-name" value={deviceName} onChange={(e) => setDeviceName(e.target.value)} className="h-7 w-48 text-sm" />
          </SettingsRow>
          <SettingsRow label="Start a new chain" description="Creates the recovery phrase you will enter on other devices.">
            <Button type="button" size="sm" className="h-7" disabled={create.isPending || deviceName.trim() === ""} onClick={onCreate}>
              {create.isPending && <Loader2 className="size-3.5 animate-spin" />} Create
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
          />
          <Button type="button" size="sm" className="h-7" disabled={join.isPending || cleanWordsInput(words).split(" ").length !== 24} onClick={onJoin}>
            {join.isPending && <Loader2 className="size-3.5 animate-spin" />} Join
          </Button>
        </div>
      </Section>

      <Dialog open={shownWords !== null} onOpenChange={(open) => { if (!open && saved) { setShownWords(null); setSaved(false); setMigrationOpen(true); } }}>
        <DialogContent className="sm:max-w-lg" onEscapeKeyDown={(e) => { if (!saved) e.preventDefault(); }} onPointerDownOutside={(e) => { if (!saved) e.preventDefault(); }}>
          <DialogHeader>
            <DialogTitle>Your recovery phrase</DialogTitle>
            <DialogDescription>
              Enter these 24 words on every other device. Anyone with them can read your synced hosts, so keep them in a password manager — SSHelter can show them again from this device only.
            </DialogDescription>
          </DialogHeader>
          <WordGrid words={shownWords ?? ""} />
          <label className="flex items-center gap-2 text-sm">
            <Checkbox checked={saved} onCheckedChange={(v) => setSaved(v === true)} />
            I have saved these words somewhere safe
          </label>
          <DialogFooter>
            <Button type="button" disabled={!saved} onClick={() => { setShownWords(null); setSaved(false); setMigrationOpen(true); }}>
              Continue
            </Button>
          </DialogFooter>
        </DialogContent>
      </Dialog>
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
      toast.error("Clipboard unavailable", { description: String(error) });
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
      <Button type="button" variant="outline" size="sm" className="h-7" onClick={copy}>
        <Copy className="size-3.5" /> Copy
      </Button>
    </div>
  );
}

function JoinedPane() {
  const status = useSyncStatus(5_000);
  const s = status.data!;
  const syncNow = useSyncNow();
  const showWords = useShowWords();
  const leave = useLeaveChain();
  const removeDevice = useRemoveDevice();
  const setDeviceName = useSetDeviceName();
  const setRelayUrl = useSetRelayUrl();
  const setMigrationOpen = useUiStore((s) => s.setSyncMigrationOpen);
  const [words, setWords] = useState<string | null>(null);
  const [leaveOpen, setLeaveOpen] = useState(false);
  const [deleteRemote, setDeleteRemote] = useState(false);
  const [nameDraft, setNameDraft] = useState(s.device_name);
  const [relayDraft, setRelayDraft] = useState(s.relay_url);

  const lastSync = s.last_sync_ms ? new Date(s.last_sync_ms).toLocaleString() : "never";

  return (
    <>
      <Section title="Sync chain" description={`Chain ${s.chain_short ?? ""} · ${s.hosts_in_sync} hosts in sync · last sync ${lastSync}`}>
        <SettingsGroup>
          <SettingsRow label="Status" description={s.last_error ?? (s.pending > 0 ? `${s.pending} change${s.pending === 1 ? "" : "s"} waiting to upload` : "Up to date")}>
            <div className="flex items-center gap-1.5">
              <Badge variant={s.last_error ? "destructive" : s.read_only ? "outline" : "secondary"}>
                {s.last_error ? "Error" : s.read_only ? "Read-only" : "Synced"}
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
            <Button type="button" variant="outline" size="sm" className="h-7" disabled={showWords.isPending} onClick={() => showWords.mutate(undefined, { onSuccess: setWords })}>
              <Eye className="size-3.5" /> Show
            </Button>
          </SettingsRow>
        </SettingsGroup>
      </Section>

      <Section title="Devices" description="Every device that joined this chain. Removing one stops it from receiving updates until it rejoins.">
        <SettingsGroup>
          {s.devices.map((d) => (
            <SettingsRow key={d.id} label={d.name + (d.is_this ? " (this device)" : "")} description={`${d.platform} · last seen ${new Date(d.last_seen_ms).toLocaleString()}`}>
              {!d.is_this && (
                <Button type="button" variant="ghost" size="icon" className="size-7 text-muted-foreground" aria-label={`Remove ${d.name}`} disabled={removeDevice.isPending} onClick={() => removeDevice.mutate({ deviceId: d.id })}>
                  <Trash2 className="size-3.5" />
                </Button>
              )}
            </SettingsRow>
          ))}
        </SettingsGroup>
      </Section>

      <Section title="Advanced" description="The relay only stores encrypted records; self-host it from the repository's relay/ folder if you prefer.">
        <SettingsGroup>
          <SettingsRow id="sync-relay" label="Relay URL">
            <div className="flex items-center gap-1.5">
              <Input id="sync-relay" value={relayDraft} onChange={(e) => setRelayDraft(e.target.value)} className="h-7 w-64 font-mono text-xs" />
              <Button type="button" variant="secondary" size="sm" className="h-7" disabled={relayDraft === s.relay_url || setRelayUrl.isPending} onClick={() => setRelayUrl.mutate({ url: relayDraft })}>
                Save
              </Button>
            </div>
          </SettingsRow>
          <SettingsRow label="Leave chain" description="This device keeps every file it has; it just stops syncing.">
            <Button type="button" variant="outline" size="sm" className="h-7 text-destructive hover:text-destructive" onClick={() => setLeaveOpen(true)}>
              Leave…
            </Button>
          </SettingsRow>
        </SettingsGroup>
      </Section>

      <Dialog open={words !== null} onOpenChange={(open) => { if (!open) setWords(null); }}>
        <DialogContent className="sm:max-w-lg">
          <DialogHeader>
            <DialogTitle>Recovery phrase</DialogTitle>
            <DialogDescription>Enter these words on the new device under Settings → Sync → Join.</DialogDescription>
          </DialogHeader>
          <WordGrid words={words ?? ""} />
        </DialogContent>
      </Dialog>

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

- [ ] **Step 2: 型別檢查與手動驗證**

Run: `pnpm build && pnpm test`
Expected: 全綠。

Run: `cd relay && npm run dev`(另一終端)+ `pnpm tauri dev`:Settings → Sync → 輸入裝置名 → Create → 24 詞視窗未勾選不可關閉 → 勾選 Continue → 狀態列顯示 Synced、Devices 含本機;Show 再次顯示同一組詞;把 relay 關掉再按 Sync now → 狀態變 Error 且 app 其他功能正常;Leave(不勾刪除)→ 回到未加入畫面,`~/.ssh/sshelter/hosts.config` 仍在。

- [ ] **Step 3: Commit**

```bash
git add src/components/SyncPane.tsx src/components/SettingsDialog.tsx
git commit -m "feat(sync): settings pane to create, join, inspect and leave a sync chain"
```

---

### Task 3: 遷入 wizard 與同名主機處理

**Files:**
- Create: `src/components/SyncMigrationDialog.tsx`
- Modify: `src/App.tsx`(掛載 `<SyncMigrationDialog />`)

**Interfaces:**
- Consumes: `groupHostsForMigration`、`useMigrateHosts`、`useDuplicateAliases`、`useSyncStatus`、`useHostsQuery`、`useRenameHost`(`{ alias, patterns }`)、`useRemoveHost`(`{ alias }`)、`labelsFor`(`@/lib/host-display`)、ui store `syncMigrationOpen`
- Produces: `<SyncMigrationDialog />`

- [ ] **Step 1: 實作**

建立 `src/components/SyncMigrationDialog.tsx`:

```tsx
import { useEffect, useMemo, useState } from "react";
import { Loader2 } from "lucide-react";
import { toast } from "sonner";

import { useHostsQuery, useRemoveHost, useRenameHost } from "@/lib/queries";
import { labelsFor } from "@/lib/host-display";
import { groupHostsForMigration } from "@/lib/sync-migration";
import { useDuplicateAliases, useMigrateHosts, useSyncStatus } from "@/lib/sync";
import { useSettingsStore } from "@/stores/settings";
import { useUiStore } from "@/stores/ui";
import { basename } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { Checkbox } from "@/components/ui/checkbox";
import { Dialog, DialogContent, DialogDescription, DialogFooter, DialogHeader, DialogTitle } from "@/components/ui/dialog";

/**
 * "Move hosts into sync": pick existing hosts (grouped by file) to move into the
 * synced file, optionally tagging them with their old file's name; then resolve
 * aliases that the synced file now shadows.
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
  const rename = useRenameHost();
  const remove = useRemoveHost();
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
          <p className="text-muted-foreground">The synced file is included first, so these local blocks are ignored by ssh:</p>
          {dups.map((d) => (
            <div key={`${d.alias}-${d.local_file}`} className="flex items-center justify-between gap-2">
              <span className="font-mono">{d.alias} <span className="text-muted-foreground">in {basename(d.local_file)}</span></span>
              <div className="flex gap-1">
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs" disabled={rename.isPending} onClick={() => rename.mutate({ alias: d.alias, patterns: [`${d.alias}-local`] }, { onSuccess: () => duplicates.refetch() })}>
                  Keep as {d.alias}-local
                </Button>
                <Button type="button" variant="outline" size="sm" className="h-6 px-2 text-xs text-destructive" disabled={remove.isPending} onClick={() => remove.mutate({ alias: d.alias }, { onSuccess: () => duplicates.refetch() })}>
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

> `useRenameHost` 的變數形狀以 `src/lib/queries.ts` 現況為準(`{ alias, patterns }`);`useRemoveHost` 的 `remove.mutate({ alias })` 會刪掉**本地**那份重複區塊 —— 同步檔的那份不受影響,因為 `config_remove_host` 依 alias 找到的是第一個定義… **注意**:`find_host_file_index` 會先命中同步檔(Include 在前)。因此「Remove local」必須刪指定檔案的區塊:若 `config_remove_host` 無法指定檔案,改為只提供「Keep as `<alias>-local`」一個動作(rename 同樣可能命中同步檔 —— 需驗證 `config_rename_host` 是否按第一個命中)。實作前先讀 `commands.rs` 的 `find_host_file_index`;若確認會命中同步檔,把兩顆按鈕改成一顆「Open in editor」(`setSelectedAlias(alias)`),讓使用者在編輯器裡處理,並在此註明限制。

`src/App.tsx`:`import { SyncMigrationDialog } from "@/components/SyncMigrationDialog";` 並在 `<NewConfigFileDialog />` 後掛 `<SyncMigrationDialog />`。

- [ ] **Step 2: 型別檢查、測試、手動驗證**

Run: `pnpm build && pnpm test`
Expected: 全綠。

手動:在已加入的裝置開 Settings → Sync → Choose hosts… → 清單依檔案分組、預設全選、wildcard 不出現 → Move → toast 顯示數量;sidebar 依 tag 模式看到原檔名 tag;第二台加入後,若本地有同名主機,對話框出現 amber 區塊。

- [ ] **Step 3: Commit**

```bash
git add src/components/SyncMigrationDialog.tsx src/App.tsx
git commit -m "feat(sync): migration wizard for moving hosts into the synced file"
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

Synced hosts live in `~/.ssh/sshelter/hosts.config`, which SSHelter `Include`s from your main config, so plain `ssh` keeps working and the file survives uninstalling SSHelter. Use *Choose hosts…* to move existing hosts in (optionally tagged with their old file name). Hosts in other files stay local to that computer.

The relay stores only ciphertext and can be self-hosted from `relay/` (`npx wrangler deploy`); point *Settings → Sync → Relay URL* at yours.
```

- [ ] **Step 2: 驗證清單文件**

建立 `docs/superpowers/plans/2026-09-27-sync-chain-manual-verification.md`,內容為下列清單(執行時逐項填結果):

```markdown
# Sync chain — manual end-to-end verification (Phase A)

Setup: relay `cd relay && npm run dev`; device A = `pnpm tauri dev`; device B = second checkout
with `SSHELTER_CONFIG`-style custom config path (Settings → Files → custom config path) pointing
at a temp dir seeded with `Host b-only`.

1. A: Create chain → words dialog blocks close until confirmed → Devices shows A.
2. A: Choose hosts… → move two hosts with "tag by file" → hosts.config contains both, tags added,
   main config has `Include ~/.ssh/sshelter/hosts.config` above the first Host.
3. B: Join with the words (paste with numbering) → migration dialog opens → hosts from A appear in
   B's sidebar under "Synced" within 45 s; `ssh -G <alias>` on B resolves the synced HostName.
4. B: edit a synced host → A shows the change within 45 s (no manual reload).
5. A and B offline (stop relay): edit the same host on both; A edits first, B second; start relay →
   B's text wins on both; A shows the "Sync overwrote a local change" toast.
6. Hand-edit hosts.config on A with a text editor → A uploads the change (pending → 0) without
   needing an app reload.
7. A: Remove device B → B's next sync still works (Phase A does not revoke) but B disappears from
   A's device list; B: Leave chain → B keeps hosts.config; Join again with words → back in sync.
8. Wrong phrase on Join → readable error, form keeps the input. Relay stopped → Status "Error",
   editing/connecting still works.
9. Windows build: repeat 3–4 on the Windows device (paths under `C:\Users\…\.ssh\sshelter\`).
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

- **Spec 覆蓋**:§7 Settings Sync pane(未加入/建立後/已加入、Show pairing code、裝置清單、relay URL、Leave)→ Task 2;§7 新裝置上手(Join → 遷入對話框)與 §10 wizard 主機/tag 部分、同名主機處理 → Task 3;事件與焦點觸發 → Task 1;「Synced」預設顯示名 → Task 2 `labelManagedFile`;§7 Keys dialog 開關與金鑰警示、`Sync passwords` 開關屬 Phase B,刻意不在此。
- **型別一致**:hooks 的 command 名稱與 A3 註冊清單一致(`sync_status`…`sync_duplicate_aliases`);`SyncStatus` 欄位(`managed_file`、`hosts_in_sync`、`devices[].is_this`)與 A3 `status_from` 一致。
- **Review Focus 對應**:1 → Task 1 `cleanWordsInput` 測試;2 → Task 1 `groupHostsForMigration` 測試;3 → Task 2 的 `saved` 勾選守衛(escape/outside 阻擋);4 → Task 2 手動驗證第 8 項(後端 404 → toast,表單狀態保留);5 → Task 2 Leave 對話框預設 `deleteRemote=false` 與說明文字。
- **已知風險**(Task 3 註記):「Remove local / Keep as -local」依賴 `config_remove_host`/`config_rename_host` 能定位到**非同步檔**的那份區塊;若既有實作以第一個命中為準,退化為「Open in editor」。實作者必須先讀 `find_host_file_index`。
