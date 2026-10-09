# Key vault plan 2b-1 — follow-ups for plan 2b-2 and later

Plan 2b-1 (`2026-10-08-key-vault-agent-2b1-new-keys.md`) is on branch `feat/keychain-2b`, rebased onto main 1612872.

This list keeps what the task reviews and the final whole-branch review parked, so nothing is lost when the execution ledger goes.
The final review triaged every item here as "can wait": none loses a key or uploads one. Each line says where it belongs.

Spec: `docs/superpowers/specs/2026-10-07-key-vault-agent-design.md`.
Manual checklist: `2026-10-08-key-vault-agent-2b1-manual-verification.md`.

## Should land in 2b-2

- **One lock for slot files.**
  - Some commands write slot files or vault entries before their `mutate`: `set_delivery`, `delete_copy`, New key's vault put, and the Reuse landings. They can race the lock-free maintenance passes, which are a joined round's step 6b and the local pass at the end of every sync attempt.
  - The effects are recoverable: a stray link reported as "in the way", or an orphan `.pub`.
  - A slot-files mutex, shared by the commands' file phase and the passes, removes the whole class.
  - 2b-1 widened the exposure, because the local pass now also runs without an account.
- **Write back only local-only entries.**
  - With an account, the local pass writes back the whole slot map.
  - That is safe today only because every `key_slots` writer outside `syncing` bumps the generation.
- **Ask instead of an automatic Reuse for vault candidates.**
  - The Sync key dialog sends Reuse without asking.
  - Sometimes the suggested account slot is held on this computer as a file: a `SyncedCopy` leftover, the §11 fallback, or a Linked "File for now". A Reuse then moves the hosts from the approval-gated vault key onto that file.
  - The non-vault Reuse landing still writes a plaintext `SyncedCopy`. Spec §4.3 says synced keys land in the vault, so check this landing against it.
- **Say before a Move that it will keep the file.**
  - `KeyFilePreview` can't carry a reason that blocks the move, so the Move option's text can promise a removal that won't happen.
  - The toast explains afterwards. A dry-run reason in the preview would let the form say it up front.
- **Other directives that name the key.**
  - Move reads only `IdentityFile`. A `ProxyCommand ssh -i ~/.ssh/id_work …` still uses the file Move deleted, and so can `LocalCommand`, `KnownHostsCommand` or `RemoteCommand`. The key can be recovered through Export.
  - Consider "an option SSHelter doesn't switch names it" as a reason to keep the file.
- **Can't-resolve matcher and symlinks.**
  - Suppose `~/.ssh` is a symlinked directory and the key is picked by its resolved path. Then `IdentityFile ~/.ssh/%h` isn't matched.
  - `//` and `/./` aren't collapsed either.
  - Fix: canonicalize the longest existing literal directory prefix.
- **Unanswered Sync key questions.**
  - After Close or Later, only the "not set up" row in Settings → Sync shows a kept slot.
  - Meanwhile another computer's copy of the host names a slot it doesn't have.
  - Consider a sidebar mark.
- **Orphans.** These are both covered by 2b-2's orphan listing (spec §14):
  - a crash between New key's vault put and its commit leaves a vault entry and `.pub` with no record;
  - so does a `land_synced_key` failure after the entry was restored.
- **Lost error on an unused vault key without an account.**
  - `maintain_local` never clears the lost error on a non-local-only vault key that no host uses, for example a previous account's key.
  - After the entry comes back, the Sync key dialog doesn't offer it.
- **Delete key… predicate.**
  - The UI offers Delete key… when no host alias uses the key.
  - `delete_copy` also counts top-level and `Match` IdentityFile uses, so the delete is refused safely.
- **Two vault slots with one key** give two rows in the Sync key dialog. This only happens with a previous account's keys.
- **Release notes.**
  - A Move makes the vault the only copy, so tell users to export a backup.
  - 0.17.0-7 ignores `local_only`. After a downgrade, a key only on this computer comes back as a previous-account key, and its slot id (never the key) can appear in the device record.

## Small fixes

- `in_the_way_message` ends "Move it, then sync again." That is the wrong next step for New key, and for a restore that a file in the way refuses.
- Some docs still say "new slot" where a restore returns the existing id: `ImportedKey` in `local_keys.rs`, `ImportResult` in `dto.rs`, and `submitImport` in `NewKey.tsx`.
- `mark_lost_if_gone` reads `vault.json` once per slot. Read the ids once per pass.
- A member who forges an account slot whose file name equals a local-only slot's lands their `.pub` over it, because `contested` counts only live slots.
- Show agent problems in the Move form or its confirm: SSHelter's agent not running, or the Include line missing.
- **Generate:**
  - Check `local_snapshot` before a slow RSA generation, not only the name.
  - The bad-name test doesn't pin the order of checks.
  - The two RSA bit-size tests could be one table.
- **New key pane:**
  - No unit tests for the pane's state logic: a failed preview becoming `problem`, the name following the preview and kind, and clearing the passphrase also clearing the repeat.
  - The `KeyDetail` import → `onSelect` wiring has no test.
  - A rejected `pick()` is unhandled.
  - `use-file-drop`'s handler doesn't check `gone`.
  - Focus lands on the body after Cancel in the confirms.
- **Add a host for this key:**
  - The File select switches from uncontrolled to controlled; use `value={state.file}`.
  - The dialog empties while it fades out.
  - The File select starts empty even when there is only one config file.
  - Enter doesn't submit.
- **Host editor:**
  - The `~/.ssh` menu label `{k.name}` doesn't go through `revealHidden`.
  - The §11 note shows for a lost vault key; use `s.in_vault && hasKeyHere(s)`.
  - `aria-label="Pick a detected key"` doesn't match the title `Pick a key`.
  - A typed IdentityFile with a space is written as typed; quoting only covers picks.
- **Sync status:**
  - On a double failure, `last_error` and `sync_once`'s return value can disagree.
  - Without an account, a refused local-pass commit leaves `last_error` set.
  - Nothing tests that the pass runs before `syncing` is cleared.
- **Checklist item 42:** say to skip "Move hosts into a space" after "Sync N spaces".
- **Tests:**
  - Task 1's tests sit at the end of the module.
  - Nothing tests end to end that a real `Conflict` shows up inside "couldn't be switched".
  - Deleting a lost key with an account has no test of its own.
