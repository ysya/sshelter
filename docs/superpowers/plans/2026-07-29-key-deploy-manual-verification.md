# In-app key deploy — end-to-end verification record

Date: 2026-08-13
Verifier: automated session (Claude Code), production execution path replayed
outside the GUI; GUI click-through items are listed at the bottom for a final
human pass.

## Environment

- macOS, OpenSSH_10.3p1 (>= 8.5 gate satisfied)
- Disposable sshd: `linuxserver/openssh-server` (OpenSSH 10.3 inside), user
  `spike`, password `hunter2`, password auth enabled
- **Deviation from the plan:** host port **2299** instead of 2222 — the
  machine's `vpn-gateway` container already binds 127.0.0.1:2222 (and 2223).
  The container was bound to `127.0.0.1:2299` only, so the temporary
  `~/.ssh/config` block used `HostName 127.0.0.1` instead of `localhost`
  (`localhost` resolves to `::1` first, which the 127.0.0.1-only mapping does
  not serve — `ssh-keyscan localhost` came back empty until the switch).
- Deployed key: throwaway ed25519 pair generated in the session scratchpad,
  never the user's real keys.
- Password path exercised: the `SSHELTER_ASKPASS_SECRET` env fallback. The
  keychain path (`SSHELTER_ASKPASS_ACCOUNT`) is covered by `secrets.rs`
  round-trip unit tests and the Task 0 spike; the helper binary is the same
  executable either way.

The deploy invocations below replayed `run_ssh_deploy` exactly: the argv from
`build_deploy_argv` (`-T`, `StrictHostKeyChecking=yes`, `BatchMode=no`,
`NumberOfPasswordPrompts=1`, `KbdInteractiveAuthentication=no`,
`ConnectTimeout=10`), `SSH_ASKPASS` pointed at the debug `sshelter` binary,
`SSH_ASKPASS_REQUIRE=force`, `REMOTE_SCRIPT` as the remote command, public key
via stdin.

## Results

0. **Askpass helper whitelist (re-check of Task 2/3 against today's binary)** ✅
   - `spike@127.0.0.1's password: ` → prints the password, exit 0, no GUI window.
   - Host-key style prompt → `[sshelter-askpass] refused`, exit 1, no output.

1. **Fresh host, correct password** ✅
   - Host key trusted via `ssh-keyscan` → append to `known_hosts`
     (the `deploy_trust_host_key` write path; its New/Mismatch gating is
     unit-tested).
   - Deploy printed `SSHELTER_ADDED`, exit 0.
   - `ssh -i <throwaway> -o IdentitiesOnly=yes -o BatchMode=yes` logged in
     **without a password** (`LOGIN-OK-NO-PASSWORD`).

2. **Redeploy same key** ✅ — printed `SSHELTER_EXISTS`, exit 0 (maps to
   "Key was already there — nothing added").

3. **Wrong password** ✅ — exit 255, stderr `Permission denied
   (publickey,password,keyboard-interactive)` (classified WrongPassword).
   The container's sshd does not surface `Failed password` lines through
   `docker logs`, so single-attempt was proven differently: a counting
   `SSH_ASKPASS` wrapper recorded **exactly one** helper invocation —
   one invocation = one password sent = one attempt
   (`NumberOfPasswordPrompts=1` holds).

4. **Remember password → app restart → Saved/Show** ⚠️ GUI —
   `remember`-only-on-success promotion and tmp-item cleanup are unit-tested
   (`deploy.rs`), keychain round-trip is unit-tested (`secrets.rs`); the
   HostEditor Saved badge → Show → same password loop needs the human pass.

5. **No remember → nothing persisted** ✅ (mechanical half) —
   after all env-fallback deploys,
   `security find-generic-password -s SSHelter -a "deploy-tmp:sshelter-verify"`
   → *could not be found* (nothing was ever written). The in-app unchecked-box
   path relies on the same unit-tested branch; GUI pass re-confirms.

6. **Password never on a command line** ✅ — 60 tight `ps -axo command`
   samples during a live deploy captured the full `ssh -T … sshelter-verify
   <script>` argv; `hunter2` appeared in **zero** samples. The password moves
   through the keychain or helper stdout only.

7. **Tampered host key → hard abort** ✅ — with a known_hosts carrying a wrong
   ed25519 key for `[127.0.0.1]:2299`, deploy exited 255 with `REMOTE HOST
   IDENTIFICATION HAS CHANGED` + `Host key verification failed` (classified
   HostKeyFailed). ssh aborts before authentication, so the password is never
   sent. Note: performed against a scratch `UserKnownHostsFile` — behaviour is
   identical and the user's real `known_hosts` stays untouched; the
   precheck-side Mismatch classification is unit-tested
   (`compare_host_keys`), and the Mismatch result screen renders **no**
   continue button (`DeployKeyDialog` ResultStage).

8. **Host down → unreachable** ✅ — deploy against a closed port exited 255
   with `Connection refused` (classified Unreachable → "Could not reach the
   host").

## Cleanup

- `sshelter-verify` container stopped and removed.
- Temporary `Host sshelter-verify` block removed; `~/.ssh/config` verified
  **byte-identical** to the pre-verification backup.
- `[127.0.0.1]:2299` removed from `known_hosts` via `ssh-keygen -R`.
- Keychain: no `SSHelter` items were created (env-fallback path only).

## Remaining GUI click-through (short human pass)

Using any password host (or the docker recipe above): right-click a host →
*Deploy key…* → confirm the fingerprint sheet appears for a new host, deploy
with **Remember** checked → quit and reopen the app → HostEditor → Password
shows *Saved*, *Show* returns the password, *Delete* clears it. Each backing
layer of these steps is verified above or in unit tests; this pass checks the
wiring of eyes-and-clicks only.
