# Key vault and agent, plan 1 — manual verification

**Note:** plan 2a replaced the controls that items 2, 3, 8, 21, 29, 30 and 31 name (the Keys dialog, Only in SSHelter, Keep a file)
with the Keychain's *Move*, *Pick a key on this computer…* and *Export private key…*; check a 2a build with `2026-10-07-key-vault-agent-2a-manual-verification.md`.

Run on a Mac and on a Windows computer (the user's Windows has OpenSSH 9.5), with a beta build of this branch. Use a test
server or a throwaway host entry; never a production key you can't replace. Record each item as pass / fail with a note.

## Setup

1. Two computers in one sync account, a synced key slot used by a host (`web`) that both computers can reach.
2. In Keys → Keys used by synced hosts, the slot shows Ready on both.

## Moving a key into SSHelter

3. Click Only in SSHelter on one computer. The toast says the key is now only in SSHelter and, the first time, points at Launch at login and Keep running in menu bar when window closes.
4. The slot directory `~/.ssh/sshelter/keys/` keeps only `<file>.pub` for that slot; the private key file is gone.
5. `~/.ssh/config` starts with `Include ~/.ssh/sshelter/agent/config`; nothing else in the file changed (compare with the backup).
6. `~/.ssh/sshelter/agent/config` starts with `# Managed by SSHelter. Changes here are overwritten.` and lists `Host web` with
   `IdentityAgent` (`~/.ssh/sshelter/agent/sock` on the Mac, `//./pipe/sshelter-agent-<hex>` on Windows) and `IdentitiesOnly yes`.
7. The host list in SSHelter does not show `web` twice.
8. Only in SSHelter on a slot whose key is a security key (`sk-…`) or a DSA key is refused with "SSHelter's agent can't use this kind of key…"; the slot keeps its file and `ssh web` still works.

## Approvals from a terminal

9. In Terminal (Windows Terminal on Windows), `ssh web`: the approval window appears on top, also while SSHelter is hidden in the tray.
   Title "Allow Terminal to use <key>?" (Windows: "Allow WindowsTerminal to use <key>?"), the chain line (for example
   "Terminal → login → zsh → ssh"), `<user>@web` (the known_hosts name), the key fingerprint, "Remember for 4 hours" checked.
10. Allow: the session opens. `exit`, `ssh web` again: no window.
11. From another terminal app (iTerm2, VS Code's terminal): the window asks again (another program).
12. Deny: ssh does not log in with this key. No answer for 60 seconds: same, and the window closes.
13. Claude Code (or another AI tool) runs `ssh web`: the window names the tool's app first in the title and the chain; remembering
    it does not let Terminal skip the window.
14. `git fetch` in a repository whose remote host uses this key, with several remotes at once (`git fetch --all`): one window, and every
    fetch succeeds after one Allow.
15. `ssh -A web`, then on `web` run `ssh other-host-using-the-same-key`: refused without a window (forwarded request).
16. `ssh-keygen -Y sign -f ~/.ssh/sshelter/keys/<file>.pub -n test somefile`: the window says "an unknown host" and offers no
    Remember.
17. Two different programs ask at once (for example `ssh web` in Terminal and in iTerm2 together). Double-click Allow on the first window: the
    second request's Allow can't be pressed for a moment, so the double click answers only the first.

## Passphrases

18. Repeat 3–7, 9 and 10 with a key that has a passphrase: the first window has a Passphrase field. A wrong passphrase shows
    "That passphrase didn't work." and asks again; three wrong ones refuse.
19. Without "Remember on this computer": a second program within 4 hours gets the window without the passphrase field.
20. With "Remember on this computer": after quitting and reopening SSHelter, the window has no passphrase field.
21. With a passphrase key opened in this session but not remembered on this computer, do Keep a file, replace the slot's key with another key, then
    Only in SSHelter again. Run `ssh web` right away: the new key works after its own approval, and the old opened key isn't used.

## Connect from SSHelter

22. Connect on `web` (main window and tray quick connect): the terminal runs
    `ssh -o IdentityAgent=… -o ForwardAgent=no web` and logs in without an approval window.
23. With a passphrase key that is not remembered: "Unlock <key> to connect to <user>@web" with Cancel and Unlock.
24. Connect on a host whose key is a normal file: unchanged (password auto-fill still works where it did).
25. Connect on a host you haven't connected to before (or remove its line from `known_hosts` first), leave ssh's fingerprint
    question open for more than a minute, then answer yes: ssh can't use the key ("communication with agent failed") and
    SSHelter shows "Connect to web again"; Connect again logs in.
26. With `ControlMaster auto` and `ControlPersist` set for `web`: a second Connect while the first session is open logs in through
    the master, and no "Connect again" message appears, then or 10 minutes later.

## When SSHelter isn't there, or things break

27. Quit SSHelter, `ssh web` from Terminal: ssh cannot use the key (it says so); reopen SSHelter and it works again.
28. Start a second SSHelter (`--mcp-host` while the app runs): the first keeps answering; no error in the second.
    Note: from the single-instance fix (`fix/mcp-single-instance`) on, a second launch hands over to the running SSHelter: check that
    instead (no second process, no window, and the first keeps answering). As written, this item applies only to builds before that fix.
29. Remove the Include line from `~/.ssh/config` by hand: Keys shows "Hosts that use keys in SSHelter can't reach its agent." with
    Fix. Fix puts the line back first; a sync round does not add it back on its own before you press Fix.
30. Keep a file: the confirm says any program can use the file without asking; afterwards the private key is back in the slot, the
    host drops out of `agent/config`, and `ssh web` works without a window.
31. Lose the vault: quit SSHelter, rename `vault.json` in SSHelter's data folder (next to `sync-state.json`), start SSHelter. After a sync
    round the synced key is back in the vault (Only in SSHelter still works); a Connect on `web` before that round finishes opens
    no terminal and says the key isn't in SSHelter on this computer. Repeat with a key that is not synced: the slot asks for
    a key again.

## Windows only

32. The pipe `\\.\pipe\sshelter-agent-<hex>` exists only while SSHelter runs; `ssh web` from PowerShell gets the window
    ("WindowsTerminal → pwsh → ssh").
