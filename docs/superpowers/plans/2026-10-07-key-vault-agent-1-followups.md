# Key vault and agent, plan 1 — follow-ups for plans 2 and 3

Plan 1 (`2026-10-07-key-vault-agent-1-core.md`) is on main at 655a83a. This list keeps what its task reviews and its final
whole-branch review parked, so nothing is lost when the execution ledger goes. Each line says where it belongs.
Spec: `docs/superpowers/specs/2026-10-07-key-vault-agent-design.md`.

## Must hold in plan 2 (the vault can become a key's only copy)

- **Never overwrite `vault:key`.** If the keychain wrongly reports "no entry", plan 1 sets the vault file aside as
  `vault.keyless-<ms>.json`, and the next `put` stores a new key under the same account. The set-aside file can then never be
  opened. That is harmless in plan 1, where every vault key has another copy, but not once the vault holds the only copy. Use
  generation-suffixed accounts so a new key never replaces an old one. **Done in 2a:** a new vault key gets an account of its own,
  `vault:key:<16 hex>`, recorded in the file header (`key_account`, format version 2); plan 1's files keep `vault:key` (version 1).
- **Show what plan 1 hides.** The Keychain page lists orphan vault entries (no slot points to them) and the set-aside files
  `vault.keyless-*` / `vault.unreadable-*`.
- **Remember the delivery choice per computer and slot.** Today, if the vault is lost while another computer changes the slot's
  key, the new key lands as a plain file without the Keep-a-file confirm. Remembering "this computer wants this slot in the
  vault" (with `keyprefs`) prevents that.

## Plan 2 (Keychain page, key picker, Export, import, migration, keyprefs)

- **Moving a key in.**
  - Move into SSHelter or Keep the file too: an own-mode slot's original key file stays in `~/.ssh` after a move into the vault.
  - On Windows' copy fallback, SSHelter's copy is renamed to `.previous-*` instead of removed when the original still exists.
- **Recognizing vault keys.**
  - `existing_slot_for` doesn't recognize an own key that is held in the vault (import). **Done in 2a:** setup reuses a vault
    slot's key (`existing_slot_for` matches a slot in the vault by its fingerprint).
  - `device_slot` labels every vault key "synced copy" in the other-computers list. **Done in 2a:** the device record carries
    `in_vault`.
  - `delete_copy`'s wording for a vault key.
- **Leftover files.** A private key file left by the user at a vault slot's path can only be removed by hand. The UI could say so.
- **When this computer can't use the key.**
  - A synced key that hasn't reached this computer yet (no source here) isn't refused at Connect; ssh shows its own
    "no such identity". The host page should explain this, along with spec §11's "This host's key is in SSHelter; open
    SSHelter to connect".
  - `stored_ids` lists the ids of a keyless vault, so Connect's check passes and signing fails later. The host page should
    surface it.
- **Slot-path spellings.** Absolute or quoted slot paths, the config loader's exact-token skip for `agent/config`, and lint's
  `.pub` exception for any slot path.
- **Agent settings UI.** `Vault::set_settings` is `#[cfg(test)]` today. Remove the attribute when the settings screen needs it.
- **Hiding "Only in SSHelter" for keys the agent can't use.** Do this once slot views carry the key type: sk-*, DSA, an
  unsupported cipher, or a key ssh-key can't parse. Own-mode payloads carry no key type today; the backend already refuses
  these keys with a message. **Done in 2a:** the control is gone; keys SSHelter's agent can't hold aren't moved into the vault
  (setup, Move) and aren't marked File for now.

## Plan 3 (MCP `run` removal, docs, Windows, Touch ID / Windows Hello, lock detection, hidden launch)

- **Windows has only been type-checked.** The first real run of the vault and agent tests is the Windows CI job after main is
  pushed.
- **Approval window.**
  - A queued request's 60 s timeout starts when it is queued, not when it is shown (spec §5.3 should say which).
  - The window is always light, and the native title bar text is static.
  - The hidden-launch exit guard is still missing.
- **Screen lock.** `clear()` can race a request that is already past its prompt. Add a generation counter together with lock
  detection.
- **Windows.**
  - Agent pipe connections have no idle timeout. Unix has one.
  - `vault.json`'s DACL is applied after the rename, so SYSTEM and Administrators can briefly read the ciphertext.
- **Lifecycle.** An instance that lost the agent lock (`OtherInstance`) never retries. After the other instance exits there is
  no agent, and no problem is shown.
- **Notifications.**
  - A tray quick connect with the main window hidden shows no "Connect again" toast.
  - The tray's "key isn't in SSHelter" refusal is only logged.
- **Program identification.**
  - The `INLINE_FLAGS` heuristic has limits: `perl -pe`, a glued `-e'…'`, and `perl -p`/`-c` merge scripts.
  - The Linux `stat` parse is untested.
  - iTerm2's identity includes its version, so it asks once more after an update.

## Accepted as is (no plan unless it bites)

- **Include wiring.**
  - Deleting the whole `~/.ssh/sshelter/agent/` directory makes the next round add the Include back.
  - If a non-default root is loaded first, a later default root's missing Include reads as a removal (Fix recovers).
  - Fix refuses on a non-default root.
- **`ssh -G`.**
  - `effective_config` has no timeout.
  - Hosts with a saved password run `ssh -G` twice.
  - The header-only rewrite of `agent/config` still runs the endpoint lookup.
- **Connect channel.**
  - The 60 s window counts awake time (a Mac's sleep pauses it).
  - The `late` notice isn't tied to the connecting program's name.
  - On Windows, an `Err` after the lifetime ended logs "connect channel stopped".
- **Memory and races.**
  - Zeroizing starts at the approval answer; serde scratch copies of key text are not zeroized (spec §3 accepts same-user
    memory reads).
  - `set_delivery` racing a round's step 6b can leave a file "in the way" (SP3-wide).
  - `hosts_file`'s `join(" ")` on shared Include lines (pre-existing).
- **Protocol.**
  - A connection bound for forwarding can still list keys (spec §5.2 refuses signing only).
  - The `vault::material` test matrix is narrow (P-384/521, RSA-4096, other ciphers).
