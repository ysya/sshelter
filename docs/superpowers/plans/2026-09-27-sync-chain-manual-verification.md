# Sync chain — manual end-to-end verification (Phase A)

Setup: relay `cd relay && npm run dev` reachable from both devices. Device A and device B must be
**two OS user accounts, a VM, or two physical machines** — a second checkout with a custom config
path is NOT a second device: `~/.ssh/sshelter/`, `sync-state.json`, the device id and the keychain
entry all follow the OS user, so two processes in one account would fight over the same sync assets.

1. A: Settings → Sync → Relay URL `http://sync.example.com` is refused (https message); the local
   `http://127.0.0.1:8787` is accepted. Create chain → the words dialog is still open while Status
   already says Synced and Devices lists A → cannot close until confirmed → Continue opens the
   migration wizard. While joined, the Relay URL row is read-only ("Leave the chain to switch relays").
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
   Forget is not revocation, the copy says so). B: Leave chain → B keeps hosts.config. A: delete one
   synced host and add another. B: Join again with the words → the deleted host disappears from B's
   hosts.config (a backup exists), the new one appears, and a host that only B had is uploaded on
   the following round (baseline round, then normal round).
8. Wrong phrase on Join (valid words, wrong chain) → "no sync chain matches" error, form keeps the
   input, and the relay's `.wrangler/state` gains no new chain. Relay stopped → Status "Error",
   editing/connecting still works.
9. B: add a local `Host <synced alias>` to the main config → wizard shows the amber "shadow" block →
   "Keep as -local" renames only the main-config copy; "Remove local" removes only that copy.
10. Windows build: repeat 3–4 on the Windows device (paths under `C:\Users\…\.ssh\sshelter\`).
11. Save-time planning: with the relay running, move a host into sync on A → Settings → Sync shows
    "1 change waiting to upload" right at save and "Up to date" within a few seconds.
12. Synced-file rules: add `Host *.internal` to A's hosts.config in a text editor → Status shows
    "…uses wildcard patterns; move that block to your main config"; no synced host is deleted or
    uploaded on either device; after moving the block to the main config, syncing resumes.
13. Lifecycle: press Join twice in quick succession (or call `sync_join_chain` twice from devtools)
    → one succeeds, the other says "already in a sync chain". Leave, Join again, quit and restart
    the app → no transient "no config loaded" error appears, and Show returns the same words.
14. Engine reload: hand-edit a synced host's HostName in A's hosts.config → A's sidebar and editor
    show the new value without a manual reload (and the edit uploads, as in item 6).
15. Unreadable state: quit A and replace `sync-state.json` (in the local data directory:
    `~/Library/Application Support/org.homelab.sshelter/` on macOS, `~/.local/share/org.homelab.sshelter/`
    on Linux, `%LOCALAPPDATA%\org.homelab.sshelter\` on Windows) with `{` → A starts normally,
    Settings → Sync shows an error naming `sync-state.unreadable-<ms>.json`, and that file holds the
    old content.
16. Vanished synced file: while joined, quit A, delete `~/.ssh/sshelter/hosts.config`, start A → the
    synced hosts come back in hosts.config (restored from the chain) and B still has all of them
    (nothing was deleted on B).
