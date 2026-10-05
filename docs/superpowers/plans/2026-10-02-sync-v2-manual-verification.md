# Sync v2 — manual two-computer verification (spaces, one sync code)

Run this before Sync v2 leaves beta (spec §11). It covers what the automated tests cannot: two real
computers, the OS keychain, the real relay, and the UI.

## Setup

- **Two computers, A and B**: two OS user accounts, a VM, or two machines. A second checkout with a
  custom config path is NOT a second computer — `~/.ssh/sshelter/`, `sync-state.json`, the device id
  and the keychain entries all follow the OS user. Name them "MacBook-A" and "MacBook-B" under
  Settings → Sync → This computer ("This device" in the 0.16.0 build), so the messages below read
  the same. Each starts with an `~/.ssh/config` that holds at least three plain hosts (none named
  `web`, `db` or `nas`), no `~/.ssh/sshelter*` files and no SSHelter keychain items. Item 9 also
  needs a throwaway sync account on a third computer (a third OS user account will do).
- **Current relay**: `cd relay && npm install && npm run dev` (port 8787). Its console logs every
  request; item 14 reads it. The app accepts only `https://` relay URLs, plus `http://` for
  127.0.0.1, localhost and [::1]: two OS user accounts on one machine both use
  `http://127.0.0.1:8787`, while a second machine or VM needs an HTTPS tunnel to port 8787 (and to
  port 8788 for item 11). On each computer, enter that URL under Settings → Sync → Relay URL and
  Save it before creating or joining.
- **Relay quota**: the relay lets one IP create 20 chains per hour (`relay/src/index.ts`), and with
  `npm run dev` every computer counts against that same limit. A new account uses 2, a new space 1,
  and changing the sync code 1 plus one per space. If a step shows "Paused by the relay's hourly
  limit on new spaces", wait an hour.
- **Old relay** (item 11): `git worktree add ../sshelter-v0.16 v0.16.0`, then
  `cd ../sshelter-v0.16/relay && npm install && npx wrangler dev --port 8788`.
- **v0.16.0 builds** (item 1): install SSHelter 0.16.0 on both computers first. All other items
  use the build under test.
- Keep a terminal open on each computer for `cat ~/.ssh/config`, `ls ~/.ssh/sshelter` and
  `ssh -G <alias> | grep -i hostname`.

## Checklist

1. **Upgrade from v1.** With 0.16.0 on A and B: create a chain on A, join on B, move three hosts
   into sync, and add a synced host whose block contains `Include ~/.ssh/extra.config` (v1 allowed
   it). On A, also save a config file of your own as `~/.ssh/sshelter/mine.config` (`Host mine` with
   a HostName) and add `Include ~/.ssh/sshelter/mine.config` on the line below SSHelter's `Include`
   in `~/.ssh/config`. Install the build under test on A only and start it:
   - Settings → Sync shows "Upgrading sync" for a moment, then the joined pane without asking for
     the words again.
   - The "Sync was upgraded" dialog appears once: the space is named “Synced”, other computers must
     update, the `Include` host stays in `~/.ssh/sshelter-v1-kept.config`, and your own config file
     moved to `~/.ssh/sshelter-local/mine.config`, where ssh keeps reading it. After "Got it" it
     does not come back after a restart.
   - `~/.ssh/sshelter/` holds `synced-<8 hex>.config` with the three hosts; `hosts.config` is gone
     (a backup exists) and so is `mine.config`; the first non-comment line of `~/.ssh/config` is
     `Include ~/.ssh/sshelter/synced-<8 hex>.config`, followed by
     `Include ~/.ssh/sshelter-v1-kept.config`; your own line now reads
     `Include ~/.ssh/sshelter-local/mine.config`, and `ssh -G mine` still resolves its HostName.
   - The sidebar shows a “Synced” group with the cloud icon; double-clicking its header neither
     opens the rename field nor collapses the group.
   - B (still 0.16.0) does not see an edit made on A afterwards. Upgrade B: both end up in the same
     “Synced” space with all hosts, and neither shows "Sync replaced a local change".
2. **Leave, then create.** On A: Advanced → Leave… (without deleting); the dialog says the space
   files move to `~/.ssh/sshelter-local/`. Afterwards `~/.ssh/sshelter-local/` holds
   `synced-<8 hex>.config` (gone from `~/.ssh/sshelter/`), the main config's Include line points
   there at the same position, Settings → Sync shows "Your synced files are now local files" with
   the path (Dismiss removes it), and `ssh -G` still resolves those hosts. Create → the sync-code
   dialog cannot be closed with Esc, a click outside or a close button until "I have saved these
   words" is checked, while the pane behind it already shows the new account. Continue opens "Move
   hosts into a space" with “Personal” selected and every listed host preselected — including the
   hosts of the left account's file, which the sidebar shows as a local file — press Move. The new
   account's Include line sits above the local one. Account → Sync code → Show shows the same 24
   words.
3. **Join and choose spaces.** First, on A create “Work” (Spaces → New space…; with a Chinese,
   Japanese or Korean input method, the Enter that commits the typed name does not create the
   space, the next one does) and add two hosts to it (New host → Target file: its
   `work-<8 hex>.config`, named on its Spaces row); A now has “Personal” and “Work”. On B: Leave
   the upgraded account (set up the optional v1 leftover below now). Joining first with a wrong
   but valid code (`abandon` 23 times, then `art`) says "no sync account matches this sync code"
   and keeps the pasted words; then paste A's new code as a numbered list → Join → "Choose spaces
   for this computer" lists both with "On MacBook-A", both checked. Uncheck Work → Sync 1 space →
   the wizard opens for Personal and says it is waiting for the first sync until A's hosts arrive
   (seconds, no reload). `~/.ssh/sshelter/` has only the Personal file and the Include line lists
   only it; `ssh -G` on B resolves a synced HostName.
   - v1 leftover (optional; set it up before B joins): put a `hosts.config` with one host into
     `~/.ssh/sshelter/` on B (after editing `~/.ssh/config`, Reload from disk). If `~/.ssh/config`
     includes it (by name, or through a hand-written glob such as
     `Include ~/.ssh/sshelter/*.config`), joining moves it to `~/.ssh/sshelter-local/hosts.config`
     (ssh keeps reading it) and shows "Your synced files are now local files" with that path. With
     no Include reading it, joining leaves it where it is without that notice (so
     `~/.ssh/sshelter/` holds it next to the Personal file), and Settings → Sync lists it under
     "Files SSHelter doesn't use".
4. **Turn spaces on and off.** On B turn Work on → its file appears, the Include line lists both
   files, Work's hosts arrive, Devices on A shows "Personal and Work" (or "Work and Personal") for
   MacBook-B. Turn Work off → the confirmation names the file and says the space stays on the other
   computers → the file is gone from B (a backup exists), the Include line drops it, A still has
   Work with all hosts.
5. **Move hosts across spaces.** First turn Work on again on B (item 4 left it off). On A, drag a
   host from the Personal group onto the Work group → on B (with both spaces on) it moves from one
   file to the other, never existing in neither. Drag a host whose block contains `Include` (the
   one from item 1, in `~/.ssh/sshelter-v1-kept.config`) into a space → refused with the reason;
   the same for a local host you add with a shell character in its `HostName` (for example
   `HostName web;id`). Open the wizard: "Can't be synced" lists both hosts with the backend's
   reasons; neither is selectable.
6. **One new space per file.** First, on A save hosts in two files that `~/.ssh/config` includes
   (for example with `Include config.d/*`): `~/.ssh/config.d/homelab.config` and
   `~/.ssh/config.d/office.config`, each with a few hosts and with `Host nas` in both; Reload from
   disk and give both files the sidebar label "Home lab". Then wizard → Move into: One new space per
   file → each group shows "→ new space “…”" ("Home lab" for the first file, "Home lab 2" for the
   second: a name that is taken gets " 2") → Move → the spaces exist, are on for A, and B lists them
   as off. With `Host nas` in both files and both selected, nas is sent once: the copy in the file
   the wizard lists first moves, the other stays local (the wizard then lists it under "Hosts
   defined in more than one file", where "Remove this copy" and "Keep as nas-local" ask first and
   say "Only this computer changes"; Cancel them), and no host fails with "listed in more than one
   group".
7. **The same name in two spaces, and in a space and your own file.** Put `Host web` in both
   Personal and Work (B has both on; New host → Target file: the space's file, named on its Spaces
   row, accepts an alias that already exists):
   - The sidebar marks the copy in the space that comes later in the Include line (Work, here) with
     the amber icon; its tooltip reads "web is also in Personal, which ssh reads first (Personal
     comes first in the Include line). ssh applies every copy and takes each setting from the first
     one that sets it, so a setting only a later copy has still applies, and options that can
     repeat (IdentityFile, LocalForward, RemoteForward, DynamicForward, SendEnv) add up."
   - Click either `web` row: only that row is highlighted, and the pane on the right shows "web is
     defined in more than one place" instead of the editor; it has no fields or buttons. Under "In
     synced spaces, which ssh reads first, in this order:" it lists Personal, then Work, each with
     its file. Personal's copy (the one ssh reads first) says "The sidebar has no action for this
     copy. To rename or remove it, edit this file in a text editor, then choose Reload from disk.";
     Work's says "In the sidebar, open this copy's menu (the ⋯ button or a right-click) and choose
     “Keep this copy as web-local” or “Remove this copy”." On either row the ⋯ menu shows "Move to
     file" and "Remove…" turned off, under "web is defined in Personal and Work, so SSHelter can't
     tell which copy this would change. Select the host to see how to fix that." The row cannot be
     dragged. With two other hosts checked (⌘-click them; the footer says "2 selected"), ⌘-click or
     Shift-click on a `web` row selects it but neither checks it nor clears the others: the footer
     still says "2 selected". A plain click on any row clears them.
   - Work's menu offers "Keep this copy as web-local" and "Remove this copy (the one in Personal
     stays)". Choose "Keep this copy as web-local" → "Keep the copy of web in Work as web-local?"
     says that every computer that syncs Work gets web-local and loses its copy of web, and that
     the copy in Personal is not touched. Confirm → the toast "Renamed the copy of web in Work to
     web-local"; the rename reaches A, and `web` in Personal can be edited again.
   - Then add a local `Host web` to B's `~/.ssh/config` (New host → Target file: `config`): the
     pane for `web` lists Personal under the synced spaces and `config` under "In other files,
     which ssh reads after the synced spaces (their order among themselves isn't shown):" ("not
     synced"), and `config`'s copy now carries the sidebar advice. On the local copy's menu, "Remove
     this copy (the one in Personal stays)" → "Remove the copy of web in config?" says "Only this
     computer changes"; confirm → the toast "Removed the copy of web in config".
   - A copy the sidebar has no action for: with the local `web` gone, add `Host extra web` (the
     name second on its line) to B's `~/.ssh/config` in a text editor and Reload from disk. The
     `web` row opens the same pane, and both copies, Personal's and `config`'s, now say "The
     sidebar has no action for this copy. To rename or remove it, edit this file in a text editor,
     then choose Reload from disk." — the backend lists only a copy whose block starts with the
     name, so no row offers “Keep this copy as web-local”. Delete the `Host extra web` block and
     Reload from disk → `web` opens its editor again.
   - Remove… on a host that is in a space (any host of Personal, say) says "This host is in the
     synced space “Personal”: removing it here removes it on every computer that syncs that space."
     Cancel it.
8. **Approvals.** On B add `ProxyCommand nc %h %p` to a synced host (below, `web` in Personal):
   - A shows "web needs your approval" with Review; Settings → Sync shows "1 host waiting for your
     approval"; A's file is unchanged.
   - Review shows the whole block with the ProxyCommand line highlighted and "Adds ProxyCommand nc
     %h %p"; its Approve and Reject buttons stay greyed for about half a second after the list
     appears. Reject → A keeps its version and nothing is uploaded; edit that host on A → the edit
     reaches B normally and replaces B's block (B's ProxyCommand line is gone).
   - On B add `ProxyCommand nc %h %p` again and change `Host web` to `Host web prod` → A asks again
     ("Applies to: web → web prod", "Adds ProxyCommand nc %h %p"); Approve writes it to A's file.
   - Remove the ProxyCommand on B → A applies that without asking. Add `StrictHostKeyChecking no`
     on B → A asks again ("Adds StrictHostKeyChecking no"). Approve or Reject what is waiting
     before the next bullet.
   - Remove `StrictHostKeyChecking no` on B → A applies that without asking. Add `LocalForward
     5433 db:5432` (only a port) on B → A applies it without asking. Then add `LocalForward
     *:5432 db:5432` → A asks ("Adds LocalForward *:5432 db:5432"; only that line is highlighted).
     Approve or Reject what is waiting before the next bullet.
   - In a text editor on B, put a right-to-left override (U+202E) in the comment of a gated setting
     that is not a command (`ForwardAgent yes #` + U+202E + `x`; ssh does not read a comment, so
     SSHelter lets it sync) → A's review shows it as `⟨U+202E⟩` and the line reads in its real
     order. The same character anywhere in a `ProxyCommand` line (`ProxyCommand nc %h 22 #` + U+202E
     + `x`) is refused, because ssh hands that whole line to a shell: B's space pauses, with "… an
     invisible formatting character … which synced hosts cannot use; move that block to your main
     config" on its row under Settings → Sync → Spaces, the same text after the space's name in
     the Status row ("Personal: …", badge "Error"), and an amber warning marker on the space's
     group header in the sidebar (its tooltip is the error); nothing reaches A. Delete that line
     again; the space resumes at the next round and its error clears everywhere. Approve or Reject
     what is waiting before the next bullet.
   - First turn Work off on A and add three hosts with a `ProxyCommand` to Work on B. Then turn Work
     on on A → one toast, and the review offers "Approve all (3)"; each card starts with "Not in
     Work on this computer yet: Host …". Approve or Reject what is waiting before the next bullet.
   - A synced host with the name of a local host: add a local `Host db` to A's `~/.ssh/config` (New
     host → Target file: `config`), then on B add `Host db` with `ProxyCommand nc %h %p` to Work
     (on for A) → A's review says "Not in Work on this computer yet: Host db" and "Takes over db in
     <path of A's ~/.ssh/config> (synced files are read first)". Approve → A's sidebar marks its
     local `db` as shadowed (item 7).
   - A synced host that another space already has: on B add `Host cache` (a HostName is enough) to
     Personal and wait until it reached A; then add `Host cache` with `ProxyCommand nc %h %p` to
     Work (Personal comes first in the Include line) → A's review says "Not in Work on this
     computer yet: Host cache", "cache is also in <path of Personal's file> (space Personal), which
     is read before Work: its values win wherever it sets one. ProxyCommand from this block still
     applies unless that block sets it too." and "Adds ProxyCommand nc %h %p". Approve or Reject
     what is waiting before the next bullet.
   - Hold a host once more: on B add `ProxyCommand nc %h %p` to `web` and open the review on A. Then
     change that ProxyCommand again on B and let it sync while the review stays open → the review
     keeps showing the version you opened and says "Newer versions arrived while this was open"
     (Approve and Reject go grey for about half a second when that line appears, because it pushes
     the list down). Approve → the newer version is not applied: the notice reads "web in Personal
     changed since you opened this — review it again.", that host's card is marked "Changed since
     you opened this — review it again", and the review now shows the newer version (the buttons go
     grey again while the notice and the list change). Approve or Reject what is waiting before
     item 9.
9. **Change the sync code with an offline second computer.** First, on A add `ProxyCommand nc %h
   %p` to a synced host and let it reach B, but do not review it on B (it waits for approval
   there). B on, then cut B off from the relay (disconnect its network) and edit a synced host on
   B; quit B. (For another OS user on the same machine, which cannot lose the network on its own,
   quit B and edit a synced host in its `~/.ssh/sshelter/` file by hand instead.) On A: Account →
   Sync code → Change… → the confirmation says other computers need the new code and the relay
   must support freezing → Status shows "Changing code" with the step, then the notice "The sync
   code was changed" → Show new sync code → the dialog cannot be closed until "I have saved the new
   sync code" → the notice disappears. Show now returns the new words.
   - Reconnect B and start it: Status reads "Paused", the card says "The sync code was changed on
     MacBook-A…" and keeps B's offline edit pending. The row "1 host waiting for your approval"
     ends with "Enter the new sync code first." and its Review… button is off; Spaces → New
     space… is off with the same words.
   - Before entering the new code on B, open Advanced → Leave… there: it never offers to delete the
     account and says the old sync account stays on the relay for the computers that still use the
     old code. Cancel the dialog.
   - Enter the OLD code → refused ("that is the old sync code…"). Enter the code of an unrelated
     account (a throwaway one created on a third computer) → refused with "none of the spaces this
     computer syncs continue in that sync account; if it is the newest sync code, leave the sync
     account — your synced files stay as local files that ssh keeps reading — and join with it", and
     the words stay.
   - Then enter the new one on B → "Syncing again with the new sync code"; B's spaces and file names
     are unchanged; the offline edit reaches A, and Review… is on again (Reject what is waiting).
     Or, as an alternative to entering it on B: Leave on B and Join with the old code → "this sync
     code was changed on MacBook-A; enter the new sync code", then Join with the new code. A third
     computer joining with the old code shows the same message without touching B.
   - Devices on A: Forget copy says Forget does not lock a computer out and points to changing the
     sync code.
10. **Cancel.** Stop the relay, then start Change… → the row stays at "Sending this computer's
    changes… You can still cancel." Show opens the sync code under "A sync code change is in
    progress. This is still the current sync code, but it stops working once the old data is
    frozen; the new sync code is shown when the change finishes.", and Spaces → New space… is off
    and reads "Wait for the new sync code to be in place, or cancel the change first." Press Cancel
    → the account is back to normal and Show returns the current code. Start the relay again.
    (With the relay running, the steps pass "Freezing the old sync data" within about a second; the
    row then says it can no longer be cancelled and offers no Cancel.)
11. **Relay without freeze or batch pull.** First, on A and B leave the account (keep the new sync
    code from item 9), set Settings → Sync → Relay URL to the old relay, `http://127.0.0.1:8788`,
    and Save, then Create an account on A and Join it on B. On this fresh account: Account → Relay
    reads "Older relay (no version reported)" and "This relay can be updated…", with "How to
    update" opening the README's "Updating your relay" section in the browser (that section exists
    on main only once this branch is merged; before that, check the URL the browser opens). Sync
    code → Change… is disabled with "Your relay can't change the sync code yet — update the relay
    first." Syncing still works between A and B (add a host to Personal on A and watch it reach B;
    the old relay's log shows one `GET …/records` per chain instead of `POST /v1/pull`: the account
    chain every round, each space every third round; an edit from the other computer can take about
    two minutes — press Sync now up to three times to force it). Update that relay in place —
    `git -C ../sshelter-v0.16 checkout --detach feat/sync-v2-spaces`, then restart its
    `npx wrangler dev --port 8788` (same `.wrangler/state`) → Check again → the hint and the block
    disappear, and the existing account keeps syncing.
12. **Rename blocked.** Back on the current relay with the account from items 9–10: on A and B,
    leave the account from item 11, set Settings → Sync → Relay URL to `http://127.0.0.1:8787` and
    Save, then Join with the new sync code you saved in item 9 and keep Work checked. (Joining
    names each file after the space's current id; a computer that stayed in the account through
    item 9 still has files named after the old ids, and an empty file with that old hex would not
    block the rename.) B has Work on, with file `work-<8 hex>.config`. On B create an empty
    `~/.ssh/sshelter/lab-<same 8 hex>.config`; Settings → Sync shows it under "Files SSHelter
    doesn't use". On A rename Work to "Lab" → B shows "The file of “Lab” keeps its old name" (toast
    and Settings → Sync), still uses `work-<8 hex>.config`, and `ssh -G` still works. Dismiss the
    notice → it does not come back on the following syncs while the empty file is still there.
    Delete the empty file on B → the next sync renames B's file and updates the Include line.
13. **Space deleted elsewhere.** On A delete Lab (the former Work; Delete… → "Delete “Lab”
    everywhere?") → B shows "“Lab” was deleted on MacBook-A", its `lab-<8 hex>.config` is gone (a
    backup exists) and the Include line drops it; the notice stays in Settings → Sync until
    dismissed.
14. **Focus and relay usage.** Watch the current relay's log: bringing SSHelter to the front runs
    exactly one sync round (one `POST /v1/pull`), never two; with the window in the background and
    no edits, rounds come about every 5 minutes instead of every 45 seconds (the 45-second cadence
    lasts while the window is in front and for five minutes after it was last brought to the front,
    a host was last saved in SSHelter, or a Sync action last ran). With Settings → Sync open and
    nothing else changing, "last sync … ago" and each computer's "last seen … ago" move on within
    about 30 seconds.
    - If the Status row ever reads "the relay is limiting requests from this network; sync retries
      automatically in a few minutes" or "the relay had trouble answering; sync retries with a
      growing delay", the relay asked for a backoff (90 seconds, doubling up to 15 minutes):
      bringing the window to the front or saving a synced host starts no round until it ends, but
      Sync now starts one at once.
15. **Leave and delete the account.** On B: Leave… offers no relay deletion ("Your other computers
    keep syncing") → B's space files move to `~/.ssh/sshelter-local/` and keep working. (To see the
    warning about unsent changes, stop the relay and edit a synced host on B first: the dialog then
    says "1 change made here and not uploaded yet won't reach your other computers — the files this
    computer keeps still have it." It counts the changes to hosts in B's spaces, which the files
    keep, not the sync account's own records. Start the relay again before you press Leave.) The
    notices B lists after leaving read as before where they are still true: "Your synced files are
    now local files" and, if you did not dismiss it in item 13, "“Lab” was deleted on MacBook-A".
    (A notice that is only about the account that was left reads differently; item 16 checks it,
    on the one computer that can have one.) On A: Forget MacBook-B, then Leave… → "Also delete the
    sync account and every space from the relay" → after leaving, A's files are in
    `~/.ssh/sshelter-local/`, and joining with that code (the new sync code from item 9) fails
    because the account is gone.
16. **A leave that cannot move the files.** First, create an account on A (a created account has
    Personal selected, which the leave needs to fail), then Change… its sync code and let the
    change finish. Only the computer that changed the code gets the notice "The sync code was
    changed", so this is where a notice about the account outlives it: leave it alone, do not press
    "Show new sync code". While joined, edit `~/.ssh/config` in a text editor and do not reload in
    SSHelter; then Leave… → "Could not leave the sync account" with "could not keep this device's
    synced files as local files (…); nothing was changed — try leaving again": still joined, files
    and Include line untouched, nothing new in `~/.ssh/sshelter-local/`, and the notice still has
    its "Show new sync code" button. Reload, Leave again → it works and keeps the outside edit. The
    notice that was never dismissed is still listed under Notices, with its title and now "This was
    about the sync account this computer has since left." and a Dismiss button (no "Show new sync
    code"); Dismiss it.
17. **Leave during a sync code change.** First, create an account on A and join B on the current
    relay, so that A and B sync on it.
    - On A, stop the relay, then Change… → the row stays at "Sending this computer's changes… You can
      still cancel." Leave… → the dialog says leaving cancels the change first. Leave → A leaves.
      Start the relay again: the account is not frozen — B keeps syncing and never shows "Paused".
    - On B, Forget MacBook-A (B is now the last computer), then stall a change past the freeze:
      create spaces until one fails with "the relay is rate-limiting this device; try again later",
      then Change… → the row reads "Copying your spaces… Paused by the relay's hourly limit on new
      spaces; …" and says it can no longer be cancelled. Show opens the sync code under "This is the
      old sync code, and it no longer works: the old sync data is frozen. The new sync code is shown
      when the change finishes." Leave… stays available; its dialog says the change can no longer
      be cancelled and offers no relay deletion although B is the last computer.
      Leave → "Could not leave the sync account" with "a sync code change is in progress; let it
      finish (it resumes on its own) before this computer leaves"; B stays joined.
    - Delete SSHelter's keychain item `sync:mnemonic-next` (service "SSHelter") on B and Leave again →
      "Left the sync account on this computer" with "left the sync account on this computer, but its
      sync code change could not be finished (the new sync code was missing from the keychain), so
      the old sync account can no longer be joined — …"; B shows the not-joined pane with "Your
      synced files are now local files". On A, Join with the old code → "this sync code was changed
      on MacBook-B; enter the new sync code". Restart the relay with an empty `.wrangler/state` (or
      wait an hour) before using it again.
