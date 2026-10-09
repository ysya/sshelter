# Key vault plan 2b-1 (bring keys into SSHelter) — manual verification

Run on a Mac and on a Windows computer with a beta build of this branch. Start from two computers in one sync account that already run
the 2a beta (0.17.0-7), so the update path is tested too. Use a test server and throwaway keys and host entries, never a key you can't
replace. A key you want to log in with needs its `.pub` in the test server's `authorized_keys`. Record each item as pass / fail with a
note.

Commands are for a Mac shell. On Windows use PowerShell with the same `ssh-keygen` options (when it asks for a passphrase, press Enter
for none), and write `$HOME\.ssh\<name>` where a command has `~/.ssh/<name>`. "A" is the computer you work on, "B" the other one in the
account. A step marked Mac or Windows is for that computer only. The Windows pass is item 43.

## Setup

1. Two computers (A and B) in one sync account, both on this build, Personal on for both, and the test server reachable from both. On
   A and B back up `~/.ssh/config` (`cp ~/.ssh/config ~/.ssh/config.before-2b1`) and make a scratch folder outside `~/.ssh` for keys you
   paste or choose (`mkdir ~/sshelter-test`; Windows: `mkdir $HOME\sshelter-test`). Many items add a host by hand to `~/.ssh/config`, in
   this shape (HostName and User are your test server's):
   ```
   Host <alias>
     HostName <test server>
     User <user>
     IdentityFile <key file>
   ```
   Press Reload from disk (the toolbar button) after every hand edit, and remove the host blocks you made by hand when their item is
   done. A host you add in Personal also reaches B and would clutter its checks, so remove it when its item is done too, unless a later
   item names it.
2. Update both computers to this build. Nothing changes by itself: the Keychain lists the keys it had with the same badges, and `ssh`
   to the hosts that use them works as before. Open the Keychain with the toolbar's key button or the `Hosts | Keychain` switch at the
   top of the sidebar: "In SSHelter" has two buttons beside its title, "New key" and "Generate key".

## New key: paste

3. **Paste a key with Windows line endings.** Make a key and a copy of it with CRLF line endings and blank lines around it, and put
   that copy on the clipboard. Mac: `ssh-keygen -t ed25519 -f ~/sshelter-test/paste_key` (no passphrase), then
   `(printf '\r\n\r\n'; awk 'BEGIN{ORS="\r\n"} 1' ~/sshelter-test/paste_key; printf '\r\n\r\n') | pbcopy`. Windows:
   `ssh-keygen -t ed25519 -f $HOME\sshelter-test\paste_key`, then
   ``(Get-Content $HOME\sshelter-test\paste_key) -join "`r`n" | Set-Content -NoNewline $HOME\sshelter-test\paste_key_crlf.txt``, open
   that file in Notepad, add blank lines above and below the key, select all and copy. In the Keychain press "New key": the pane is
   titled "New key" and says "SSHelter keeps the key in its vault. Programs ask before they use it." Leave "Paste" selected, click the
   "Private key" box (its placeholder is `-----BEGIN OPENSSH PRIVATE KEY-----`), paste, type the name `paste_test` and press "Add to
   SSHelter".
   - It is added at once, with no confirm: the toast says "paste_test is in SSHelter" and the new key is selected.
   - Its detail has the badge "This computer only" and `ssh-ed25519` beside it, and under the title "Only this computer has this key.
     Export a copy to keep a backup." "Created" is today's date (YYYY-MM-DD) and "Passphrase" is "No". "Fingerprint" equals what
     `ssh-keygen -l -f ~/sshelter-test/paste_key` prints. "On this computer" says "Ready" and "In SSHelter — programs ask before they
     use it". "Hosts" says "No hosts use it."
   - `~/.ssh/sshelter/keys/` (Windows: `%USERPROFILE%\.ssh\sshelter\keys\`) holds only `paste_test-<8 hex>.pub`, no file without `.pub`.
4. **Text SSHelter refuses, a bad name, and a key with a passphrase.**
   - Paste `hello` and press "Add to SSHelter": the toast "Could not add the key" says "This isn't an OpenSSH private key SSHelter can
     read." Your text stays in the box and nothing is added: the list and `~/.ssh/sshelter/keys/` are unchanged.
   - Paste a legacy PEM key (`ssh-keygen -t rsa -b 2048 -m PEM -f ~/sshelter-test/pem_key`, no passphrase; paste `pem_key`): "Could not
     add the key" says "This key isn't in the OpenSSH format. Convert it with ssh-keygen -p -f <file>, then try again."
   - Paste about 20,000 characters of any text (Mac: `head -c 20000 /dev/zero | tr '\0' a | pbcopy`; Windows: `("a" * 20000) |
     Set-Clipboard`): "Could not add the key" says "This key is larger than 16 KiB, which SSHelter's vault doesn't take."
   - Type `my key` (or `x.pub`) as the name: red text "Use letters, digits, '.', '_' or '-'; start with a letter or digit; don't end
     with .pub." and "Add to SSHelter" is off.
   - Make a key with a passphrase (`ssh-keygen -t ed25519 -f ~/sshelter-test/pass_key`, passphrase `test-pass`) and paste it as
     `pass_key`: it is added, and its "Passphrase" says "Yes". Item 25 logs in with it.
5. **The same key twice.** Paste the same key again under another name (`paste_again`): "Could not add the key" says "This key is
   already in SSHelter as paste_test." Nothing is added.

## New key: from a file

6. **Choose a file, then drop one.** Make two keys, `ssh-keygen -t ed25519 -f ~/sshelter-test/file_key` and `drop_key` (no
   passphrase). Press "New key", then "From a file" and "Choose a file…". Pick `file_key` in the system dialog (Mac: press ⌘⇧G and type
   the folder; Windows: type the path into the File name box).
   - Before anything is added the form shows the file's path, then `ssh-ed25519 · SHA256:…` (the fingerprint `ssh-keygen -l -f`
     prints), the Name `file_key` (from the file name), and the choice "Move into SSHelter" / "Keep the file too" with "Move into
     SSHelter" pressed. Until you choose a file the form says "No file chosen. You can also drop a key file here."
   - Drag `drop_key` from Finder (Explorer) onto the SSHelter window: the form switches to it, with its path, type and fingerprint, and
     the Name `drop_key`. Type another Name (`custom`) and drop `file_key`: the Name stays `custom`.
   - Start on "Paste" with some text in the box and drop a file: the form switches to "From a file" with that file. Dropping two files
     at once does nothing, and neither does dropping while another key's detail or the Generate key form is showing.
   - "Cancel": the main pane says "No key selected" and nothing was added.
7. **Keep the file too.** Make `ssh-keygen -t ed25519 -f ~/.ssh/keep_key`, add a host block `keephost` that names it, and Reload from
   disk. New key, From a file, `~/.ssh/keep_key`, press "Keep the file too": the form says "The file stays. Any program can use it
   without asking." and no longer lists `keephost`. Press "Add to SSHelter": it is added at once, with no confirm. The toast says
   "keep_key is in SSHelter" with "The file stays: any program can use it without asking." `~/.ssh/keep_key` is still there and
   `keephost` still names `~/.ssh/keep_key`. `ssh keephost` logs in without an approval window.
8. **Move into SSHelter.** Make `ssh-keygen -t ed25519 -f ~/.ssh/move_key` (`.pub` on the test server), add a host block `movehost`
   that names `~/.ssh/move_key`, and Reload from disk. New key, From a file, `~/.ssh/move_key`: with "Move into SSHelter" pressed the
   form says "SSHelter keeps the only copy on this computer and removes the file. Export private key… gets a file back." and
   "movehost will use the key in SSHelter." Press "Add to SSHelter" and confirm (item 9).
   - The toast says "move_key is in SSHelter" with "Removed" and the file's full path as its second line (for example "Removed
     /Users/<you>/.ssh/move_key.").
   - `~/.ssh/move_key` is gone and `~/.ssh/move_key.pub` is still there. `movehost` now has
     `IdentityFile ~/.ssh/sshelter/keys/move_key-<8 hex>`.
   - The new key is selected: "This computer only", "Created" is today, and "Hosts" lists `movehost`. `~/.ssh/sshelter/agent/config`
     lists `Host movehost`, and `~/.ssh/config` starts with `Include ~/.ssh/sshelter/agent/config`.
   - `ssh movehost` shows the approval window and logs in after Allow.
9. **The Move confirm.** Use a key file as in item 8 (a new throwaway key if item 8 used its key up) and press "Add to SSHelter" with
   "Move into SSHelter" pressed.
   - A confirm opens: "Move move_key into SSHelter?" and "SSHelter keeps the only copy of this key on this computer and removes
     /Users/<you>/.ssh/move_key. Export private key… gets a file back." with "Cancel" and "Move into SSHelter". While it is open
     nothing is added: the list and `~/.ssh/sshelter/keys/` are unchanged.
   - "Cancel", and Escape, close it and leave the form as it was (file, name, "Move into SSHelter" pressed), with nothing added. Press
     "Add to SSHelter" again and the confirm opens again.
   - With "Keep the file too" pressed, and for a pasted key (item 3), there is no confirm: the key is added at once.
   - While the confirm is open, drag another key file onto the window: it is ignored. The confirm still names the first file, and after
     "Cancel" the form still shows the first file.
   - A long path wraps. Make one throwaway key in nested folders so that its full path is over 100 characters (for example
     `~/sshelter-test/a-folder-with-a-long-name/another-folder-with-a-long-name/yet-another-long-folder/long_key`), and another with a
     70-character file name without hyphens or spaces. For each, the confirm's text wraps inside the dialog; nothing is cut off or runs
     past its edge.
10. **Move and a `Host *` block.** Make `ssh-keygen -t ed25519 -f ~/.ssh/star_key` and add this to the end of `~/.ssh/config` (it makes
    every host use the key, so remove it afterwards), then Reload from disk:
    ```
    Host *
      IdentityFile ~/.ssh/star_key
    ```
    If `~/.ssh/config` already has a `Host *` block, add the IdentityFile line to it instead of making a second block: two `Host *`
    blocks count as "more than one copy" and the Move would keep the file (item 16).
    New key, From a file, `~/.ssh/star_key`: the form says "* will use the key in SSHelter." Move it. The `Host *` block now reads
    `IdentityFile ~/.ssh/sshelter/keys/star_key-<8 hex>` and `~/.ssh/star_key` is gone. `~/.ssh/sshelter/agent/config` lists
    `Host *`, so every `ssh` goes through SSHelter's agent until you undo this. Remove the line (or the block), Reload from disk, and
    Delete key… the key (item 36).
11. **Move with a default identity file.** Pick a key that ssh tries by itself: `~/.ssh/id_ed25519`, or a throwaway
    `~/.ssh/id_ecdsa` (`ssh-keygen -t ecdsa -f ~/.ssh/id_ecdsa`; ssh offers it to every host, so remove it afterwards). If you use your
    real default key, press "Cancel" after reading the form and don't add it. With "Move into SSHelter" pressed the form says "ssh also
    tries ~/.ssh/id_ed25519 for hosts that don't name a key. After the move, those hosts won't find it." (with the key's own name).
    With "Keep the file too" pressed, and for a key with another name such as `move_key`, that sentence is not shown.
12. **Import from ~/.ssh.** Make `ssh-keygen -t ed25519 -f ~/.ssh/import_key`. Open "Other key files in ~/.ssh" in the list and select
    `import_key`: its detail has "Copy public key", "Export to host…", "Import into SSHelter…" and "Add a host for this key…". Press
    "Import into SSHelter…": the main pane shows New key with "From a file" pressed, the path of `import_key`, its type and
    fingerprint, and the Name `import_key`. "Cancel" returns to "No key selected". Select `import_key` and press "Import into
    SSHelter…" again, choose "Keep the file too" and "Add to SSHelter": it is added, and `import_key` is still listed under "Other key
    files in ~/.ssh".
13. **Files that can't be added.** In New key, From a file:
    - Choose `~/sshelter-test/paste_key.pub`, or a text file: under the file line it says "This file isn't a private key." and "Add to
      SSHelter" is off.
    - Choose a `.pub` file in `~/.ssh/sshelter/keys/`: "SSHelter already manages this file."
    - Choose `~/sshelter-test/pem_key` (item 4): "This key isn't in the OpenSSH format. Convert it with ssh-keygen -p -f <file>, then try
      again."
14. **A key file an SP3 slot links to.** Make `ssh-keygen -t ed25519 -f ~/.ssh/linked_key`. In Personal add a host `linkedhost` that
    names it (New host with Target file "Personal", then set IdentityFile in the editor with the key button and Save). When "Keys used
    by synced hosts" opens, press "Keep on this computer". The Keychain lists `linked_key` with "File for now" and the banner "1 key
    can move into SSHelter"; don't press Move. New key, From a file, `~/.ssh/linked_key`: under the file line it says "This key is
    already in SSHelter as linked_key." and "Add to SSHelter" is off. Paste the file's text instead: "Could not add the key" says
    "This key is already in SSHelter as linked_key." Afterwards remove `linkedhost` (or B will ask for a key for it) and use "Delete
    copy" on the `linked_key` row.
15. **The config changed on disk.** Make `ssh-keygen -t ed25519 -f ~/.ssh/drift_key`, add a host block `drifthost` that names it, and
    Reload from disk. New key, From a file, `~/.ssh/drift_key`. Keep SSHelter's window in view beside your editor or terminal (a window
    that was minimized or hidden reads the config again when it comes back, and then there is nothing to refuse). In the other editor
    add a second host block `ghost` to `~/.ssh/config` that also names `~/.ssh/drift_key`, and do not reload in SSHelter.
    - Press "Add to SSHelter" and confirm the Move. Before anything is added the toast "Could not add the key" says "Your SSH config
      changed on disk since SSHelter loaded it. Reload it, then try again." The key is not in the list, `~/.ssh/drift_key` is still
      there, and the form still holds the file and name.
    - Press Reload on the "Changed on disk" banner ("Config reloaded from disk"), then "Add to SSHelter" and the confirm again: the Move
      works, and `drifthost` and `ghost` both have the new slot path.

## When Move keeps the file

In each item: make a fresh throwaway key, set up the config as described, Reload from disk, then New key, From a file, the key, "Add
to SSHelter", and confirm the Move. The key is still added: the toast says "<name> is in SSHelter", the new key is selected with "This
computer only", and the toast's second line is the reason below. `~/.ssh/<name>` is still there and no host's IdentityFile line is
rewritten (compare the IdentityFile lines in `~/.ssh/config` and in the files in `~/.ssh/sshelter/` before and after; Mac:
`grep -rn IdentityFile ~/.ssh/config ~/.ssh/sshelter/`). The reason shows only in the toast, which fades after a few seconds: watch for
it and repeat the item if you miss it. Remove what you added and Delete key… the key before the next item.

16. **A host with several copies.** `ssh-keygen -t ed25519 -f ~/.ssh/dup_key`, and two host blocks named `dup` in `~/.ssh/config`, both
    with `IdentityFile ~/.ssh/dup_key`. The form lists `dup` once. Reason: "The file stays: dup have more than one copy, so SSHelter
    didn't change them."
17. **A host in a space that hasn't finished its first sync.** Needs a space this computer has off: if there is none, make one on the
    other computer (Settings → Sync → Spaces → New space…, named "Work") and leave it off here. Turn the network off, then turn "Work"
    on in Settings → Sync → Spaces: its row has not finished syncing (it reads "Syncing for the first time…", or shows a relay error
    while you are offline). Make `ssh-keygen -t ed25519 -f ~/.ssh/early_key` and add a host `early` to Work's file (New host, Target
    file "Work") that names it. New key, From a file, `~/.ssh/early_key`, Move. Reason: "The file stays: early are in a space that
    hasn't finished its first sync." If the first sync finishes before you get there, skip the item and say so. Turn the network back
    on afterwards.
18. **An IdentityFile outside a Host block.** Make two keys, `top_key` and `match_key`, and do each in its own run.
    - Put `IdentityFile ~/.ssh/top_key` at the very top of `~/.ssh/config`, above any Host line. Move `top_key`.
    - Put this in `~/.ssh/config`:
      ```
      Match host <test server>
        IdentityFile ~/.ssh/match_key
      ```
      Move `match_key`.

    Both: the form lists no host, and the reason is "The file stays: an IdentityFile outside a Host block names it, and SSHelter only
    switches hosts."
19. **An IdentityFile SSHelter can't resolve.** Make a key whose file is named after a host: `ssh-keygen -t ed25519 -f ~/.ssh/github.com`.
    Put this in `~/.ssh/config`:
    ```
    Host *
      IdentityFile ~/.ssh/%h
    ```
    Move `~/.ssh/github.com`. The form lists no host. Reason: "The file stays: an IdentityFile SSHelter can't resolve (a token, a
    variable or a relative path) may name it." `~/.ssh/github.com` is still there and the `Host *` line still reads `IdentityFile ~/.ssh/%h`
    (ssh would still find the file for `github.com`). Remove the block.
20. **A key file that is a symbolic link.** Mac: `ssh-keygen -t ed25519 -f ~/.ssh/real_key && ln -s ~/.ssh/real_key ~/.ssh/link_key`.
    Move `~/.ssh/link_key`. Reason: "The file stays: it's a link to another file, so SSHelter didn't remove either." Both `link_key`
    and `real_key` are still there. (Windows only if Developer Mode or an elevated PowerShell lets you make a link.)

Two more reasons are not checked by hand because they need a failing disk write: "The file stays: {hosts} couldn't be switched
({error})." and "The file stays: it couldn't be removed ({error})." Two others are not reachable by hand: "The file stays: SSHelter
couldn't check which hosts use it (no config loaded)." and "The file stays: your SSH config changed on disk since SSHelter loaded it."
(a change in the moment between the check in item 15 and the write).

## Generate key

21. **Generate each kind.** Press "Generate key" (beside "In SSHelter"). The form says "A new key made in SSHelter. It exists only in
    SSHelter until you export it.", offers "Ed25519" (pressed), "RSA 3072", "RSA 4096" and "ECDSA P-256", suggests the Name
    `id_ed25519`, and has no Comment field: only "Name", "Passphrase (optional)" and, once you type a passphrase, "Repeat the
    passphrase".
    - "RSA 3072" and "RSA 4096" change the Name to `id_rsa` and show "An RSA key takes a few seconds to make."; "ECDSA P-256" suggests
      `id_ecdsa`. Once you type your own Name, changing the kind keeps it.
    - Generate each kind under its own name (`gen_ed`, `gen_rsa3072`, `gen_rsa4096`, `gen_ecdsa`). Each time the toast says
      "Generated <name>" with the key's fingerprint (`SHA256:…`) as its second line, and the new key opens with "This computer only",
      its type (`ssh-ed25519`, `ssh-rsa`, `ssh-rsa`, `ecdsa-sha2-nistp256`), "Created" today and "Passphrase" "No". Only the `.pub` is in
      `~/.ssh/sshelter/keys/`.
    - On "RSA 4096" the button reads "Generating…" and every control in the form is off, and the window keeps responding: type into
      the sidebar's "Search keys…" box and move and resize the window while it works. It finishes within seconds.
    - Export private key… (then "Export…") for the two RSA keys: `ssh-keygen -l -f <file>` prints 3072 and 4096 (RSA).
    - "Generate a key file…" is gone: it isn't beside "Other key files in ~/.ssh" and nothing in the Keychain opens it.
22. **A passphrase, and the name as the comment.** Generate key, Ed25519, Name `gen_pass`, type a passphrase and a different repeat:
    "Generate" stays off until both match. Generate it. "Passphrase" is "Yes". Export private key…: "Export gen_pass?" says "It stays
    protected by its passphrase."; "Export…" and save as `~/sshelter-test/gen_pass_export`. Then run
    `ssh-keygen -y -f ~/sshelter-test/gen_pass_export`: it asks for the passphrase and prints `ssh-ed25519 AAAA… gen_pass`, so the
    key's name is its comment. Export `gen_ed` (item 21) without adding a passphrase and run `ssh-keygen -y -f` on that file: it
    prints `ssh-ed25519 AAAA… gen_ed` without asking.
23. **A New key form started during a slow generate.** Generate key, "RSA 4096", Name `gen_slow`, press "Generate". At once press "New
    key" and paste some text into the "Private key" box without adding it. When the generate finishes, the toast "Generated gen_slow"
    shows and `gen_slow` is in the list, but the New key form still holds your text: the pane does not jump to the new key. (If it
    finishes before you can press "New key", repeat the item.)

## Add a host for this key

24. **Add a host on a generated key.** Select `gen_ed` and press "Add a host for this key…".
    - The dialog is titled "Add a host for gen_ed" and says "The new host uses this key: IdentityFile
      ~/.ssh/sshelter/keys/gen_ed-<8 hex>". Its fields are "Host" (placeholder `github.com`), "HostName" (placeholder `optional`),
      "User" (placeholder `git`) and "File".
    - "Add host" stays off until Host has an alias and a File is chosen. "File" shows "Select a config file" (unless the sidebar is
      scoped to a file) and lists every config file SSHelter loaded: the main config, files it includes and your synced spaces by
      name. `a b` in Host shows "A host alias can't contain spaces."; an alias that exists already (any host in your config, say `testbox`)
      shows "testbox already exists."; only spaces shows "Enter a host alias."
    - Host `github.com`, User `git`, File = the main config, "Add host": the toast says "Added github.com", the dialog closes, and
      `github.com` is under the key's "Hosts" (pressing it opens Hosts with `github.com` selected). `~/.ssh/config` has `Host github.com`
      with `User git` and `IdentityFile ~/.ssh/sshelter/keys/gen_ed-<8 hex>`, and `~/.ssh/sshelter/agent/config` lists
      `Host github.com` at once.
    - Press "Copy public key" and add it to GitHub as a throwaway key (a deploy key on a scratch repository, or a throwaway account).
      `ssh -T git@github.com` shows the approval window and, after Allow, authenticates.
25. **A key with a passphrase logs in through the approval window.** Select `pass_key` (item 4), "Add a host for this key…": Host
    `passhost`, HostName = your test server, User = your user, File = the main config, "Add host". Put `pass_key.pub` on the test
    server. `ssh passhost`: the approval window has a "Passphrase" field. A wrong passphrase shows "That passphrase didn't work." and
    asks again; the right one logs in.
26. **A key file whose path has a space.** `ssh-keygen -t ed25519 -f "$HOME/.ssh/id test"` (Windows: `"$HOME\.ssh\id test"`), no
    passphrase, `.pub` on the test server. Open "Other key files in ~/.ssh", select `id test`, "Add a host for this key…": the dialog
    says "The new host uses this key: IdentityFile ~/.ssh/id test". Host `spacehost`, HostName = your test server, User = your user,
    File = the main config, "Add host". `~/.ssh/config` has `IdentityFile "~/.ssh/id test"` in double quotes. `ssh -G spacehost` prints
    an `identityfile` line and no error, Config lint (the toolbar's shield button) shows no "IdentityFile not found" for `spacehost`,
    and `ssh spacehost` logs in with that key.
27. **A host in a synced space asks the Sync key dialog.** Generate key, Ed25519, Name `ask_key`. "Add a host for this key…": Host
    `askhost`, File = "Personal", "Add host".
    - The toast says "Added askhost" and "Keys used by synced hosts" opens: "askhost uses ask_key.", "Sync this key to your other
      computers?", "This key is only on this computer, in SSHelter. Its hosts keep using ~/.ssh/sshelter/keys/ask_key-<8 hex>." with no
      "Rename" and no rewritten lines, and the buttons "Keep on this computer" and "Sync key". Press "Close".
    - Best effort, once more with the alias `askhost2`: the write takes a few milliseconds, so you can rarely hit it by hand. Press "Add
      host" and, as fast as you can, Escape. If you manage it, the dialog stays open until the host is written and then closes with the
      toast and the Sync key dialog above; if Escape ever closes it without them, that is a fail.

## The host editor

28. **Three ways to pick a key.** In Hosts select a host that is in `~/.ssh/config` (not in a synced space). Its IdentityFile row has
    the field, a key button (tooltip "Pick a key") and a folder button (tooltip "Browse…").
    - The key button's menu lists "Keys in SSHelter" (this computer's keys by name, `paste_test` among them) above "Keys in ~/.ssh"
      (the key files, `import_key` among them). Pick `paste_test` under "Keys in SSHelter": the field becomes
      `~/.ssh/sshelter/keys/paste_test-<8 hex>`, the row's note reads "paste_test is in SSHelter: ssh can use it only while SSHelter is
      running.", and "Unsaved changes" shows. Save: `~/.ssh/sshelter/agent/config` lists the host at once and `ssh <host>` shows the
      approval window.
    - Pick `import_key` under "Keys in ~/.ssh": the field becomes `~/.ssh/import_key` and the note goes.
    - The folder button opens the system dialog "Choose an identity file". Pick `~/sshelter-test/file_key`: the field holds its full
      path, as picked (it is outside `~/.ssh`).
29. **A picked path with a space is written quoted.** Make a folder with a space and a key in it:
    `mkdir "$HOME/sshelter-test/my keys" && ssh-keygen -t ed25519 -f "$HOME/sshelter-test/my keys/id_space"`. In the host editor press the
    folder button and pick `id_space`: the field shows the path in double quotes (`"/Users/<you>/sshelter-test/my keys/id_space"`).
    Save, then `ssh -G <host>` prints an `identityfile` line and no error. Then pick `id test` (item 26) under "Keys in ~/.ssh": the
    field is `"~/.ssh/id test"`.
30. **The note in a narrow window.** On a host whose IdentityFile names a key in SSHelter (item 28), make the editor as narrow as it
    gets: the window at its minimum width and the sidebar dragged to its widest. The note wraps onto several lines under the
    "IdentityFile" label; the field and its two buttons stay visible on the right; nothing is cut off and the editor does not scroll
    sideways. Raise Settings → Appearance → Text size and look again.

## Keys used by synced hosts, on two computers

Setup for 31 to 33, on A: two throwaway keys, `ssh-keygen -t ed25519 -f ~/.ssh/kkeep` and `~/.ssh/ksync`, both `.pub` on the test server.
In Personal add the host `hostkeep` (New host, Target file "Personal"), set its IdentityFile to `~/.ssh/kkeep` with the key button's
"Keys in ~/.ssh" and Save. When "Keys used by synced hosts" opens, press "Close": leave the question unanswered. B has Personal on and
synced.

31. **Before A answers, B shows the host and a lint message.** On A: New key, From a file, `~/.ssh/kkeep`, Move, confirm. "Keys used by
    synced hosts" opens: "hostkeep uses kkeep." and "Sync this key to your other computers?". Leave it open. Sync B (Sync now in
    Settings → Sync, or wait a minute). B lists `hostkeep`, and Config lint lists "IdentityFile not found:
    ~/.ssh/sshelter/keys/kkeep-<8 hex> (a key slot your sync account doesn't have — set the key up on the computer that has it)" for
    it (the `<8 hex>` is the one in A's `~/.ssh/sshelter/keys/kkeep-<8 hex>.pub`). B shows nothing else: no `kkeep` in its Keychain and
    no notice in Settings → Sync.
32. **"Keep on this computer".** On A press "Keep on this computer" (toast "kkeep stays on this computer"). A's `kkeep` now has the badge
    "Own key on each computer" and no longer the note "Only this computer has this key. Export a copy to keep a backup." After B syncs,
    B's Keychain lists `kkeep` with "Own key on each computer" and "Needs a key", the "Keys for this computer" dialog opens, and Config
    lint's message for `hostkeep` now ends "(a synced key slot — pick a key for it in Keychain)". No key reached B: it has no `.pub`
    and no private key file for `kkeep` in `~/.ssh/sshelter/keys/`.
33. **"Sync key".** On A make `ksync` local-only and use it from Personal: New key, From a file, `~/.ssh/ksync`, "Keep the file too"
    (add it), then "Add a host for this key…": Host `hostsync`, File = "Personal", "Add host". In "Keys used by synced hosts" press "Sync
    key" (toast "ksync syncs to your other computers"). After B syncs, B's Keychain lists `ksync` with "Synced" and "Ready", "In
    SSHelter — programs ask before they use it", and `~/.ssh/sshelter/keys/` on B has `ksync-<8 hex>.pub` and no private key file.
    `ssh hostsync` on B shows the approval window and logs in. After A syncs again, A's `ksync` lists B's name under "Other computers"
    with "in SSHelter".
34. **A reused account key lands in the vault.** Needs the same key on A as a "This computer only" key and in the account as a synced
    key that A's spaces don't use yet. On A: make `ssh-keygen -t ed25519 -f ~/sshelter-test/twin_key` and add it with New key, Paste, as
    `twin`. On B: copy `twin_key` and `twin_key.pub` into B's `~/.ssh`, create a space "Reuse" (Settings → Sync → Spaces → New space…;
    leave it off on A), add a host `twinhost` to it (New host, Target file "Reuse") that names `~/.ssh/twin_key`, Save, and in "Keys
    used by synced hosts" press "Sync key". After A syncs, on A select `twin`, "Add a host for this key…": Host `reuse-host`, File =
    "Personal", "Add host". No dialog asks (the account already has this key).
    - `reuse-host` now names the account's slot (`IdentityFile ~/.ssh/sshelter/keys/twin_key-<8 hex>`, not a `twin-…` one).
    - The Keychain lists `twin_key` as "Synced" with "In SSHelter — programs ask before they use it", and `twin` still as "This
      computer only".
    - `~/.ssh/sshelter/keys/` has `.pub` files for both and no private key file. `vault.json` (the data folder, item 36) holds the
      account key: `grep -c <8 hex of twin_key> vault.json` prints a number above 0.
35. **A locked keychain is not a lost key (Mac).** Generate key `lock_key`, "Add a host for this key…": Host `lockhost`, File =
    "Personal", "Add host". "Keys used by synced hosts" opens: leave it open. Lock the login keychain (Keychain Access, right-click
    "login", Lock Keychain "login"). Press "Keep on this computer" (or "Sync key"); if macOS asks for the keychain password, press
    Cancel. The toast "Could not set up the key" carries the keychain's own error (it starts with "keychain error:"), not "This key
    is no longer in SSHelter's vault. If you exported a copy, add it again with New key." The dialog stays open and nothing changes:
    `lock_key` is still "This computer only" and `lockhost` is unchanged. Unlock the keychain and press the same button again: it
    works (toast "lock_key stays on this computer").

## Delete key

36. **Delete key… on a key no host uses.** Add a throwaway key with New key, Paste, as `del_key`. Note the 8 characters in its
    `~/.ssh/sshelter/keys/del_key-<8 hex>.pub`. SSHelter's data folder (next to `sync-state.json`) is
    `~/Library/Application Support/org.homelab.sshelter/` on the Mac and `%LOCALAPPDATA%\org.homelab.sshelter\` on Windows, and holds
    `vault.json`. Mac: `grep -c <8 hex> "$HOME/Library/Application Support/org.homelab.sshelter/vault.json"` prints a number above 0
    (Windows: `Select-String <8 hex> $env:LOCALAPPDATA\org.homelab.sshelter\vault.json` finds a line).
    - Press "Delete key…" at the bottom of the detail. The confirm says "Delete del_key?" and "SSHelter removes this key from this
      computer. It's the only copy unless you exported one." with "Cancel" and "Delete". "Cancel" changes nothing.
    - "Delete": the toast says "Deleted del_key", the row leaves the list, and the main pane says "No key selected" and "Choose a key
      from the list." The `.pub` is gone from `~/.ssh/sshelter/keys/`, and the `grep` above prints 0 (Windows: finds nothing).
37. **Not offered while a host uses the key.** Select `gen_ed` (item 24): "Hosts" lists `github.com` and there is no "Delete key…"
    button. Change `github.com`'s IdentityFile in the host editor, or remove the host, and come back: "Delete key…" is there.

## Layout

38. **The "In SSHelter" header in a narrow sidebar.** In the Keychain drag the sidebar's edge as far left as it goes. The title "In
    SSHelter" and the buttons "New key" and "Generate key" wrap onto two lines instead of overflowing; both buttons stay fully visible
    and work, and nothing scrolls sideways. Repeat at the largest Settings → Appearance → Text size, and double-click the edge to reset
    the width. Choosing a file with "Choose a file…" and dropping one both start the New key form with the file: item 6. On a computer
    with no key in SSHelter at all (for example a Windows user that never added one) the list says "No keys in SSHelter yet" and "Add
    one with New key or Generate key. Keys used by synced hosts appear here too."

## A file that can never move

39. **It stays a file.** `ssh-keygen -t rsa -b 2048 -m PEM -f ~/.ssh/id_pem` (no passphrase). In Personal add a host `pemhost` that
    names `~/.ssh/id_pem` (key button, "Keys in ~/.ssh") and Save; in "Keys used by synced hosts" press "Keep on this computer". The
    `id_pem` row has no "File for now" badge and the banner's count does not include it. Its detail's "On this computer" reads "It
    stays a file: This key isn't in the OpenSSH format. Convert it with ssh-keygen -p -f <file>, then try again." and there is no "Move
    into SSHelter" button. With a FIDO security key (`ssh-keygen -t ed25519-sk`) it reads "It stays a file: SSHelter's agent can't use
    this kind of key (for example a security key or a DSA key), so it stays as a file." Afterwards remove `pemhost` and use "Delete
    copy" on the `id_pem` row.

## Keys only on this computer, with and without an account

40. **A key lost from the vault stays listed.** Make a key `lost_key` (Generate key, Ed25519) and use it from a host: "Add a host for
    this key…" `losthost`, File = the main config. Export it first (Export private key…, `~/sshelter-test/lost_key_export`). Quit
    SSHelter, rename `vault.json` in the data folder (item 36) to `vault.json.aside`, start SSHelter, and wait for a sync attempt
    (switch to another app and back). `lost_key` is still in the list with the badges "This computer only" and "Error", and its
    detail's "On this computer" reads "This key is no longer in SSHelter's vault. If you exported a copy, add it again with New key."
    (every key that is only on this computer shows the same while the vault is set aside; synced keys come back from the account).
    "Hosts" lists `losthost`, so there is no "Delete key…". Change `losthost`'s IdentityFile in the host editor: "Delete key…"
    appears; press it and "Delete": the toast says "Deleted lost_key" and the row is gone. New key, From a file, `lost_key_export`
    ("Keep the file too") adds it again. To get the other keys back, quit SSHelter, delete the new `vault.json` (if there is one), and
    rename `vault.json.aside` back.
41. **Without a sync account.** Do this last: it leaves the sync account. Settings → Sync → Leave…, then Leave. The Keychain still
    lists the keys that are "This computer only" and their detail still works.
    - Delete a key's `.pub` in Finder or Explorer (`~/.ssh/sshelter/keys/<name>-<8 hex>.pub`). Switch to another app and back to
      SSHelter, or wait up to 5 minutes: the `.pub` is back.
    - Repeat item 40 with a new key now that there is no account: the key stays listed with the same error, and Delete key… removes
      it once no host uses it. Put the vault back as item 40 says afterwards: item 42 needs `ksync`.

## A key from a previous sync account

42. **A previous account's key in the vault.** After item 41 this computer has left the account, and `ksync` (item 33) is still in
    SSHelter. Settings → Sync → "Create a sync account" → Create (confirm the sync code it shows). In Personal add a host `prevhost`
    (New host, Target file "Personal") and set its IdentityFile with the key button, "Keys in SSHelter", `ksync`; Save. "Keys used by
    synced hosts" opens: "prevhost uses ksync.", "Sync this key to your other computers?", "From your previous sync account, in SSHelter
    on this computer. Its hosts keep using ~/.ssh/sshelter/keys/ksync-<8 hex>." with no "Rename" and no rewritten lines. Press "Sync
    key": the toast says "ksync syncs to your other computers", `prevhost` still names `~/.ssh/sshelter/keys/ksync-<8 hex>` (it is not
    rewritten), and `ksync` is "Synced" in the new account.

## Windows

43. **Windows.** Run items 3, 6, 8, 9, 11, 21 and 24 on the Windows computer too (PowerShell for the commands; "Terminal" there is
    Windows Terminal), and check:
    - The Windows paths read right: the confirm in item 9 says "…and removes C:\Users\<you>\.ssh\move_key." and the toast in item 8
      "Removed C:\Users\<you>\.ssh\move_key."; the default-file sentence in item 11 still says `~/.ssh/id_ed25519`. The host lines use
      `~/.ssh/sshelter/keys/<name>-<8 hex>` as on the Mac. The long path in item 9 (with backslashes) wraps too.
    - `~/.ssh/sshelter/agent/config` names the pipe: `IdentityAgent //./pipe/sshelter-agent-<hex>`. In item 8 `ssh movehost` and in
      item 24 `ssh -T git@github.com`, run from PowerShell, show the approval window and log in.
    - A Browse pick (the folder button in the host editor) of a key outside `~/.ssh` whose path has a space is written in double
      quotes: `mkdir "$HOME\My Keys"` and `ssh-keygen -t ed25519 -f "$HOME\My Keys\id_test"`, pick it: the field holds
      `"C:\Users\<you>\My Keys\id_test"`; Save, and `ssh -G <host>` prints an `identityfile` line and no error.
    - Item 20 (the symbolic link) only if Developer Mode or an elevated PowerShell lets you make one.

When you are done: remove the host blocks you added by hand, restore `~/.ssh/config` from `~/.ssh/config.before-2b1` if it is simpler
(press Fix in the Keychain if the `Include ~/.ssh/sshelter/agent/config` line is missing afterwards), Delete key… the test keys, and
delete `~/sshelter-test` and the `~/.ssh/*_key` files you made.
