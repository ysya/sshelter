# Key vault plan 2a (Keychain, keys always in SSHelter) — manual verification

Run on a Mac and on a Windows computer with a beta build of this branch. Start from computers on 0.17.0-6 that sync a key slot, so the
update path is tested too. Use a test server or a throwaway host entry, never a production key you can't replace. Record each item
as pass / fail with a note.

## Setup

1. Two computers (A and B) in one sync account. A has a synced slot used by `web`, and an own-key slot used by `db`; both were set up
   on 0.17.0-6, so they are files (links or synced copies).
2. Update both to this build. Nothing changes by itself: `ssh web` and `ssh db` work as before, and `~/.ssh/config` is unchanged.

## The Keychain

3. The toolbar's key button switches the sidebar to Keychain, and pressing it again switches back. The `Hosts | Keychain` switch at the
   top of the sidebar does the same. Quit and reopen SSHelter: the sidebar shows the view it had.
4. "In SSHelter" lists both slots with "Synced" / "Own key on each computer" and "File for now". "Other key files in ~/.ssh" is closed;
   opened, it lists the key files no slot links to, with "in ssh-agent" on the ones loaded into ssh-agent.
5. Search finds a key by name, type and fingerprint, and shows matching key files even while that section is closed.
6. The old SSH keys dialog and its "agent: N keys" line are gone.

## Move

7. The banner says "2 keys can move into SSHelter". Move: both slots lose "File for now"; the slot directory keeps only the `.pub`
   files; the original key file of the own-key slot is still in `~/.ssh` and now appears under "Other key files".
8. `~/.ssh/config` starts with the `Include ~/.ssh/sshelter/agent/config` line; `ssh web` shows the approval window, and works after Allow.
9. Make one move fail: on a computer with a file-for-now slot, rename the key file a link points to, then Move. The other keys move; that
   one keeps "File for now", the toast names it and says why, and its detail shows the same reason.
10. A key SSHelter's agent can never hold (a security key, a DSA key, a legacy PEM key) stays a file. On the computer from item 9, while its
    slot still waits, point a synced host at a legacy PEM key (`ssh-keygen -t rsa -b 2048 -m PEM -f ~/.ssh/id_pem`, empty passphrase, its
    `.pub` on the test server) and answer "Sync this key to your other computers?" with Keep on this computer. That slot has no "File for
    now", the banner still counts only item 9's slot, and its detail has no "Move into SSHelter" button. Put the renamed file back and press
    Move: item 9's slot moves; the PEM slot is left alone with no message about it, its link stays in the slot directory, and `ssh` to its
    host still uses the file, without an approval window.
11. The launch hint appears once a key is in SSHelter while neither Launch at login nor Keep running in menu bar is on. "Turn on launch
    at login" turns on three settings: Launch at login, Show menu bar icon and Keep running in menu bar when window closes (Settings →
    General shows all three on), and the hint goes. On another computer, Dismiss: the hint stays gone after a restart.
12. Launch hint with the menu bar icon off: on a computer where you didn't press Dismiss, turn Show menu bar icon, Keep running in menu bar
    when window closes and Launch at login off in Settings → General, then switch to Hosts and back to the Keychain: the hint is there.
    "Turn on launch at login" turns the icon on too. Close the window: SSHelter keeps running in the menu bar (Windows: the tray), and the
    icon's Open SSHelter brings the window back.

## Keys always in SSHelter

13. On A, create a new synced slot for a host (`api`). On B, after a sync, the slot arrives in SSHelter directly (no "File for now",
    only the `.pub` in the slot directory), and `ssh api` works on B after the approval.
14. B's detail of that slot lists A under "Other computers" as "in SSHelter" once A moved its key; a computer still on 0.17.0-6, or one
    that still holds the synced copy as a file for now, shows as "a synced file".
15. Lock the login keychain (macOS: Keychain Access → Lock; or take SSHelter's `vault:key:…` entry's access away), then let a new synced
    key arrive: it lands as "File for now" and `ssh` works with the file; unlock and Move: it goes into SSHelter.
16. Pick a key on this computer… / Change… on a slot in SSHelter: the picked key is copied into SSHelter (its file stays where it
    is), hosts use it after the approval, and the key it replaced is kept (plan 2b lists it).
17. Use the synced key on a slot whose own key is in SSHelter: the synced key replaces it and the old one is kept.
18. "Keep a file" is offered nowhere.

## Export

19. Export private key… on a key without a passphrase, adding one (typed twice): the save dialog suggests the key's name; the file is
    `-rw-------` (Windows: only you in its Security tab); `ssh-keygen -y -f <file>` asks for the passphrase and prints the public key.
20. Export without a passphrase: `ssh-keygen -y -f <file>` prints the public key without asking. Hosts still use SSHelter's key.
21. Export into `~/.ssh/sshelter/keys/`: refused with "Choose a folder outside ~/.ssh/sshelter: SSHelter manages that folder.";
    nothing is written. Cancel the save dialog: nothing is written, the export dialog stays open.
22. Export private key… on a key without a passphrase: type a passphrase and its repeat, press Export… and cancel the system save dialog.
    The export dialog stays open with what you typed, and nothing is written. Clear the first field: the repeat field goes, Export… is
    enabled, and exporting now writes a file without a passphrase (`ssh-keygen -y -f <file>` prints the public key without asking).
23. On a Mac, Export private key… to `~/Desktop` and to `~/Documents`: it works (macOS may ask once whether SSHelter can access that
    folder), the file is `-rw-------`, and no `.tmp…` file is left next to it (`ls -a`).
24. Export to host… → In the app, on a host that uses another key: the deploy form says "<host> will use <key> instead of <old key>";
    after Deploy the host block has one `IdentityFile` line, the slot path, and `Host *` is unchanged. `ssh <host>` works after the
    approval.
25. Export to host… → In the app, on the Mac and on Windows: pick a host. The host picker closes and the deploy dialog opens at once and
    can be used right away: type the password and press Deploy. Nothing is stuck unclickable: after you close the deploy dialog, the
    Keychain responds to clicks.
26. Export to host… on a key file in ~/.ssh: "In Terminal (ssh-copy-id)" is offered on the Mac and not on Windows; it runs ssh-copy-id
    and changes no settings. A key in SSHelter offers only In the app.
27. Export to host on a host in a synced space with a key file: the SP3 "sync this key?" question follows.

## Windows

28. With a key in SSHelter and Git for Windows using its own ssh, the Git hint shows the `git config --global core.sshCommand
    C:/Windows/System32/OpenSSH/ssh.exe` command; Copy, run it, return to the Keychain: the hint is gone; `git fetch` over ssh shows the
    approval window and works.
29. The Git hint follows git's setting. Unset it again (`git config --global --unset core.sshCommand`) and switch to Hosts and back to the
    Keychain: the hint shows the command. Run the command and switch again: the hint is gone. With git not installed (or not on the PATH
    SSHelter starts with) there is no hint.
30. Export private key… to a FAT or exFAT USB stick fails with an error, and no file is left on the stick (such a drive can't make a file
    readable only by you).

## Pointers

31. Settings → Sync → "Pick…" closes Settings and shows, in the Keychain, the first key that needs one.
32. "Keys for this computer" says "…or do it later in Keychain"; its Pick… still picks in place.
33. The sidebar's missing-key marker says "pick one in Keychain"; pressing a host in a key's detail switches to Hosts with that host
    selected.
34. With the Keychain showing, select a host three ways: in the command palette (⌘K on the Mac, Ctrl+K on Windows) press ⌘↵ (Windows:
    Ctrl+Enter) on a host, which edits it, while plain Enter connects; add a host with New host; click a host's issue in Config lint (a
    host with `IdentityFile ~/.ssh/missing` has one). Each time the sidebar switches to Hosts and shows that host.
