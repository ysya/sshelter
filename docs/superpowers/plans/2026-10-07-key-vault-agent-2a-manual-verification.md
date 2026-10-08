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
7. Unsaved host edits survive the Keychain: in Hosts, change a field of a host without saving (the save bar shows "Unsaved changes"),
   switch to the Keychain and back to Hosts. The change and the save bar are still there, and Save saves it.

## Move

8. Before Move, on A: in the host editor, set the IdentityFile of a second synced host in `db`'s space (`db2`) to `db`'s key file (the
   own-key slot's detail shows that file under "On this computer") and Save. SSHelter points `db2` at the slot by itself, without
   asking. The slot keeps "File for now", the banner still says "2 keys can move into SSHelter", and `ssh db` still connects without an
   approval window.
9. The banner says "2 keys can move into SSHelter". Move: both slots lose "File for now"; the slot directory keeps only the `.pub`
   files; the original key file of the own-key slot is still in `~/.ssh` and now appears under "Other key files".
10. `~/.ssh/config` starts with the `Include ~/.ssh/sshelter/agent/config` line; `ssh web` shows the approval window, and works after Allow.
11. Make one move fail: on a computer with a file-for-now slot, rename the key file a link points to, then Move. The other keys move; that
    one keeps "File for now", the toast names it and says why, and its detail shows the same reason.
12. A key SSHelter's agent can never hold (a security key, a DSA key, a legacy PEM key) stays a file. On the computer from item 11,
    while its slot still waits, point a synced host that doesn't use item 11's slot at a legacy PEM key
    (`ssh-keygen -t rsa -b 2048 -m PEM -f ~/.ssh/id_pem`, empty passphrase, its `.pub` on the test server) and answer "Sync this key to
    your other computers?" with Keep on this computer. That slot has no "File for now", the banner still counts only item 11's slot,
    and its detail has no "Move into SSHelter" button. Put the renamed file back and press Move: item 11's slot moves; the PEM slot is
    left alone with no message about it, its link stays in the slot directory, and `ssh` to its host still uses the file, without an
    approval window.
13. The launch hint appears once a key is in SSHelter while neither Launch at login nor Keep running in menu bar is on. "Turn on launch
    at login" turns on three settings: Launch at login, Show menu bar icon and Keep running in menu bar when window closes (Settings →
    General shows all three on), and the hint goes. On another computer, Dismiss: the hint stays gone after a restart.
14. Launch hint with the menu bar icon off: on a computer where you didn't press Dismiss, turn Show menu bar icon, Keep running in menu bar
    when window closes and Launch at login off in Settings → General, then switch to Hosts and back to the Keychain: the hint is there.
    "Turn on launch at login" turns the icon on too. Close the window: SSHelter keeps running in the menu bar (Windows: the tray), and the
    icon's Open SSHelter brings the window back.

## Keys always in SSHelter

15. On A, create a new synced slot for a host (`api`). On B, after a sync, the slot arrives in SSHelter directly (no "File for now",
    only the `.pub` in the slot directory), and `ssh api` works on B after the approval.
16. B's detail of that slot lists A under "Other computers" as "in SSHelter" once A moved its key; a computer still on 0.17.0-6, or one
    that still holds the synced copy as a file for now, shows as "a synced file".
17. Lock the login keychain (macOS: Keychain Access → Lock; or take SSHelter's `vault:key:…` entry's access away), then let a new synced
    key arrive: it lands as "File for now" and `ssh` works with the file; unlock and Move: it goes into SSHelter.
18. Pick a key on this computer… / Change… on a slot in SSHelter: the picked key is copied into SSHelter (its file stays where it
    is), hosts use it after the approval, and the key it replaced is kept (plan 2b lists it).
19. Use the synced key on a slot whose own key is in SSHelter: the synced key replaces it and the old one is kept.
20. "Keep a file" is offered nowhere.
21. Quit SSHelter (Quit in the menu bar or tray icon's menu; closing the window keeps it running when Keep running in menu bar is on):
    `ssh web` can't use its key (OpenSSH says `no such identity`). Start SSHelter again: `ssh web` shows the approval window and works
    after Allow.
22. Remove the `Include ~/.ssh/sshelter/agent/config` line from `~/.ssh/config` in another editor, then Reload from disk (the toolbar
    button). The Keychain shows "Hosts that use keys in SSHelter can't reach its agent." with Fix, `ssh web` can't use its key, and
    SSHelter doesn't put the line back by itself (a sync or a host save leaves it out). Fix puts the line back as the first line of
    `~/.ssh/config`, the message goes, and `ssh web` shows the approval window again.
23. In the host editor, point a host in `~/.ssh/config` (not in a synced space) at a key in SSHelter: put the slot path `web` uses
    (`~/.ssh/sshelter/keys/<name>-<id>`) in its IdentityFile and Save. `~/.ssh/sshelter/agent/config` lists that host at once, with no
    wait for a sync, and `ssh <host>` shows the approval window.

## Export

24. Export private key… on a key without a passphrase, adding one (typed twice): the save dialog suggests the key's name; the file is
    `-rw-------` (Windows: only you in its Security tab); `ssh-keygen -y -f <file>` asks for the passphrase and prints the public key.
25. Export without a passphrase: `ssh-keygen -y -f <file>` prints the public key without asking. Hosts still use SSHelter's key.
26. Export into `~/.ssh/sshelter/keys/`: refused with "Choose a folder outside ~/.ssh/sshelter: SSHelter manages that folder.";
    nothing is written. Cancel the save dialog: nothing is written, the export dialog stays open.
27. Export private key… on a key without a passphrase: type a passphrase and its repeat, press Export… and cancel the system save dialog.
    The export dialog stays open with what you typed, and nothing is written. Clear the first field: the repeat field goes, Export… is
    enabled, and exporting now writes a file without a passphrase (`ssh-keygen -y -f <file>` prints the public key without asking).
28. On a Mac, Export private key… to `~/Desktop` and to `~/Documents`: it works (macOS may ask once whether SSHelter can access that
    folder), the file is `-rw-------`, and no `.tmp…` file is left next to it (`ls -a`).
29. Export to host… → In the app, on a host that uses another key: the deploy form says "<host> will use <key> instead of <old key>";
    after Deploy the host block has one `IdentityFile` line, the slot path, and `Host *` is unchanged. `ssh <host>` works after the
    approval.
30. Export to host… → In the app, on the Mac and on Windows: pick a host. The host picker closes and the deploy dialog opens at once and
    can be used right away: type the password and press Deploy. Nothing is stuck unclickable: after you close the deploy dialog, the
    Keychain responds to clicks.
31. Export to host… → In the app with a key in SSHelter, on a host in `~/.ssh/config` (not in a synced space): right after Deploy,
    `~/.ssh/sshelter/agent/config` lists the host and `ssh <host>` shows the approval window, with no wait for a sync.
32. A key file whose name has a space (`ssh-keygen -t ed25519 -f "$HOME/.ssh/id test"`): Export to host… → In the app on a host. The
    host's line is written in double quotes (`IdentityFile "~/.ssh/id test"`), `ssh <host>` logs in with that key, and Config lint shows
    no "IdentityFile not found" for it.
33. A failed write offers a retry: Export to host… → In the app on a host in `~/.ssh/config`. With SSHelter's window still in view (a
    window that was minimized or hidden reads the config again when it comes back, and then the write works), change `~/.ssh/config`
    outside SSHelter (`echo '# test' >> ~/.ssh/config` in a terminal), then Deploy. The key is deployed, but writing the host's
    IdentityFile fails with "Failed to save host" (the file changed on disk since it was loaded), and the result screen offers "Use this
    key — IdentityFile …". Press it: it says "IdentityFile … written to the host config.", the host has the one `IdentityFile` line, and
    the line you added is still in `~/.ssh/config`.
34. Export to host… on a key file in ~/.ssh: "In Terminal (ssh-copy-id)" is offered on the Mac and not on Windows; it runs ssh-copy-id
    and changes no settings. A key in SSHelter offers only In the app.
35. Export to host on a host in a synced space with a key file: the SP3 "sync this key?" question follows.

## Windows

36. With a key in SSHelter and Git for Windows using its own ssh, the Git hint shows the `git config --global core.sshCommand
    C:/Windows/System32/OpenSSH/ssh.exe` command; Copy, run it, return to the Keychain: the hint is gone; `git fetch` over ssh shows the
    approval window and works.
37. The Git hint follows git's setting. Unset it again (`git config --global --unset core.sshCommand`) and switch to Hosts and back to the
    Keychain: the hint shows the command. Run the command and switch again: the hint is gone. With git not installed (or not on the PATH
    SSHelter starts with) there is no hint.
38. Export private key… to a FAT or exFAT USB stick fails with an error, and no file is left on the stick (such a drive can't make a file
    readable only by you).

## Pointers

39. Settings → Sync → "Pick…" closes Settings and shows, in the Keychain, the first key that needs one.
40. "Keys for this computer" says "…or do it later in Keychain"; its Pick… still picks in place.
41. The sidebar's missing-key marker says "pick one in Keychain"; pressing a host in a key's detail switches to Hosts with that host
    selected.
42. With the Keychain showing, select a host three ways, switching back to the Keychain before each: in the command palette (⌘K on the
    Mac, Ctrl+K on Windows) press ⌘↵ (Windows: Ctrl+Enter) on a host, which edits it, while plain Enter connects; add a host with New
    host; click a host's issue in Config lint (a host with `IdentityFile ~/.ssh/missing` has one). Each time the sidebar switches to
    Hosts and shows that host.

## Without a sync account

43. Do these last: they leave the sync account. On a computer whose keys are in SSHelter, leave the sync account (Settings → Sync →
    Leave…). The Keychain still lists those keys, each with "Not in your sync account". `ssh` to a host in `~/.ssh/sshelter-local/` that
    uses one of them works after the approval, and Export private key… still works. Stop that host using the key (host editor): the
    key's detail says "Not in use", and Delete copy removes the key from SSHelter.
44. On the computer from item 12, once it has left the account as in item 43: stop the PEM key's host using its slot. The slot's detail
    says "Not in use"; Delete copy removes the link and its `.pub` from `~/.ssh/sshelter/keys/`, the row goes, and `~/.ssh/id_pem` stays.
45. On Windows without a sync account (after item 43), start SSHelter a second time while it runs. In the second window, Delete copy on a
    key no host uses (or Move, or Move into SSHelter, when offered) refuses with "Sync is running in another SSHelter process — quit it to
    use sync here" and changes nothing: the first window still shows the key as it was.
    Note: in release and beta builds from the single-instance fix (`fix/mcp-single-instance`) on, a second launch hands over to the
    running SSHelter: check that instead (no second process or window; the running window comes to the front). As written, this item
    applies only to builds before that fix and to debug builds (`tauri dev`), which don't register single instance.
