# Own SSH client — phase 0 spike — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Find out, with running code, the facts phase 1 of SSHelter's own SSH client depends on: can russh 0.64.1 live next to the app's `ssh-key` 0.6.7, can the key vault sign for it, does it do exec, shell with a PTY, host key checks, keepalive and a two-hop jump against a real sshd and on Windows. Also move the agent's local transport into the `ipc/` module the design needs (the one piece of this plan that is real product code).

**Architecture:** A throwaway crate `spike/russh/` (its own `Cargo.lock`, the app library as an optional path dependency for the vault, `russh = "=0.64.1"`) holds a scratch-sshd harness and the tests; it ships nothing and phase 1 rewrites `ssh/russh_engine.rs` from scratch. Task 5 moves `agent/server.rs`, `agent/pipe_windows.rs` and `agent/peer.rs` into `src-tauri/src/ipc/` with no change in behavior. Task 6 turns the test log into the spike report.

**Tech Stack:** Rust 2021 (rustc 1.98 here; russh needs 1.89), russh 0.64.1 (tokio, aws-lc-rs, `ssh-key =0.7.0-rc.11`), the app's `ssh-key` 0.6.7 vault, OpenSSH `sshd`/`ssh`/`ssh-keygen` as scratch servers, GitHub Actions `windows-latest`, a few small Python 3 scripts (a lock-file diff, two exact-match edit scripts for Task 5, a report filler).

**Spec:** `docs/superpowers/specs/2026-10-10-own-ssh-client-design.md` — §4 (process model, `ipc/`), §6 (engine, russh facts, §6.3 limits), §7 (IPC/CLI), §12 (tests; read it as updated by commit bfd5b35), §13 (phase 0), §15 (verification log; as updated by 8b78077), §16 (open items). Output: `docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md`.

## Global Constraints

- **Where.** Repository `/Users/ysya/project/sideproj/sshelter`, branch `next/own-ssh` (planned at HEAD 8b78077, from main b6cd95f).
- **Toolchain.** Every cargo command is prefixed `PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH` (Homebrew's rustc is broken here; that toolchain is rustc/cargo 1.98.0; russh 0.64.1 needs 1.89 or newer).
- **The app's Rust tests.** From `src-tauri/`, exactly one `--`, all filters after it, then the two skips: `PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- ipc:: agent:: --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain`.
- **The spike's commands** run in `spike/russh/` (`cargo test --offline …`). The one `cargo fetch` in Task 1 needs the network and is the only local step that does; everything after it is `--offline`.
- **No pnpm, no app.** Never run `pnpm`/`npm`, never launch the app, never push. Task 4 and Task 6 say where a push would be needed; that is the user's decision.
- **Isolation.** Tests never touch the real `~/.ssh`, the Keychain, the agent or the app data directory, and never contact a real host. The scratch sshd lives in a temp directory on 127.0.0.1 and logs in only the user running the tests; the system `ssh` the harness runs gets `-F /dev/null`.
- **Spawning.** The spike crate has no `clippy.toml`, so a direct `std::process::Command::spawn`/`output` is fine there. Code in `src-tauri/` keeps going through `crate::process`.
- **Commits.** Conventional commits in English, no attribution lines, files staged by path (never `git add -A`). `spike/` is committed on the branch (it documents the experiment). App files change only in Task 2 (one line) and Task 5.
- **Comments.** `agent/`, `sync/`, `vault/`, `ipc/`: Traditional Chinese. `mcp.rs`, `config/`, the spike crate and the workflows: English.
- **Nothing weakened.** No change to `vault/`. Task 2 changes `mod vault;` to `pub mod vault;` in `src-tauri/src/lib.rs` — one line, checked: no other change and no new warning — and that line is the only app change outside Task 5. `src-tauri/Cargo.toml` and `src-tauri/Cargo.lock` stay byte-identical: run `git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock` at the end of every task; it must print nothing.
- **Facts checked by the controller on 2026-10-10 (they replace the old wording of spec §12).**
  1. The repo has no scratch-sshd harness. `src-tauri/src/agent/openssh_tests.rs` says in its header that it needs no sshd (it runs `ssh-add` and `ssh-keygen` against SSHelter's own agent). Task 2 builds the harness from scratch.
  2. A scratch sshd works as a normal user on this Mac (OpenSSH 10.3p1): a temp dir with `hostkey` (ssh-keygen ed25519) and `authorized_keys` (0600); config lines `Port <free port>`, `ListenAddress 127.0.0.1`, `HostKey <absolute>`, `PidFile none`, `UsePAM no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `PubkeyAuthentication yes`, `AuthorizedKeysFile <absolute>`, `StrictModes no`, `LogLevel ERROR`; `sshd -t -f` passes and `sshd -D -e -f <config>` is usable after about a second; public-key login works and the remote exit code and output come back. `Port 0` is rejected ("Badly formatted port number"): pick a free port first by binding a TCP socket to port 0. sshd's stderr prints one harmless line `BSM audit: … setaudit_addr failed: Operation not permitted`. Password and keyboard-interactive login cannot be verified on a PAM-less scratch sshd: those are "verify on a real host".
  3. The lib crate is named `sshelter_lib` (its crate-type includes `rlib`, so a path dependency works). `src-tauri/src/lib.rs:19` has `mod vault;` (private) and `vault/mod.rs` has `pub mod material;`, so `vault::material` is not reachable from another crate without that one-line visibility change. A path dependency on the library pulls Tauri into the spike build: a long first build.
  4. The app's `Cargo.lock` already contains tokio 1.52.3, aws-lc-rs 1.18.1, aws-lc-sys 0.45.0, ring 0.17.14 and rustls 0.23.40, so russh's defaults add no new TLS/crypto-backend stack. (They do add a second RustCrypto generation for SSH keys — see Task 1; the coexistence question is `ssh-key` 0.6.7 next to `ssh-key` 0.7.0-rc.)
- **Spike rule.** These tests state what the design expects of russh and of the vault. If one fails because russh (or sshd) behaves differently, that is the result, not a defect of the plan: keep the failing output; change the assertion to what actually happens, keeping the original expectation in a one-line comment `// the design expected: …`; print the observed value with `fact(...)`; re-run until green; the row goes into the report's list of assumptions that turned out wrong. A name that does not compile is fixed against the compiler and listed in the report. Stop and report to the controller instead of working around it only when Task 1's `cargo fetch` cannot resolve, or when sshd refuses the vault's signature for every key type in Task 2.
- **Facts for the report.** Tests print measurements as `FACT <key> = <value>` lines (`russh_spike::fact`); Task 6 collects them. Run test binaries with `-- --nocapture` when you want to see them.

## Review Focus

The input classes and failure modes the spec implies that no task's happy path exercises, most likely first. Each has a test in the task named after it.

1. **A legacy server that offers only `ssh-rsa` (SHA-1)** for its host key and accepts only SHA-1 RSA user signatures (old routers, NAS firmware): the user would expect the connection to work with russh's default algorithm lists, with the vault signing with flag 0. Pinned in Task 2: `an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1`.
2. **A server whose host key algorithm russh does not offer** (nothing in common): the user would expect a message that names both lists, not a hang and not the bare "unknown key" a refused host key gives. Pinned in Task 3c: `no_common_host_key_algorithm_names_both_lists`.
3. **A command that ends without an exit status** (killed by a signal; the connection cut mid-command): the user would expect "killed by KILL" or "connection lost", never exit code 0 and never a wait that does not end. Pinned in Task 3a: `a_command_killed_by_a_signal_reports_the_signal_and_no_exit_status`; in Task 3d: `a_connection_cut_during_an_exec_ends_it_without_an_exit_status`.
4. **Resizing the terminal before the shell is ready** (the `window-change` goes out between `pty-req` and `shell`): the user would expect the last size to win and the shell to be unharmed. Pinned in Task 3b: `a_window_change_sent_before_the_shell_is_ready_is_not_lost` (and an observation of the case where there is no PTY yet).
5. **Very long output** (`cat` of a big log through `ssh_exec`): the user would expect all bytes to arrive, or the reader to stop at a cap and close the channel, and the connection to stay usable either way. Pinned in Task 3a: `thirty_two_mebibytes_of_output_arrive_complete` and `a_reader_that_stops_at_a_cap_and_closes_the_channel_leaves_the_connection_usable`.

## File Structure

- `spike/russh/Cargo.toml`, `Cargo.lock` — the throwaway crate and its own lock file (Task 1).
- `spike/russh/src/lib.rs` — `fact`, `within`, the module list.
- `spike/russh/src/harness.rs` — tool lookup, `ssh-keygen` keys, the scratch `sshd`, the system `ssh` client (Task 2).
- `spike/russh/src/client.rs` — the russh `Handler`, `connect`, login and exec helpers (Task 2).
- `spike/russh/src/fixture.rs` — a scratch server plus connect-and-log-in steps shared by the tests (Task 2).
- `spike/russh/src/signer.rs` — russh's `Signer` backed by the vault (Task 2).
- `spike/russh/src/shell.rs`, `auth.rs`, `proxy.rs` — PTY shell helpers, password/keyboard-interactive helpers, a misbehaving TCP link (Task 3).
- `spike/russh/tests/*.rs` — one file per topic: `coexistence`, `harness_smoke`, `vault_signer`, `exec`, `shell`, `auth_methods`, `host_key`, `keepalive`, `jump`, `channels`, `windows_exec`.
- `.github/workflows/spike-windows.yml` — the Windows job (Task 4).
- `src-tauri/src/lib.rs` — `pub mod vault;` (Task 2), `mod ipc;` (Task 5).
- `src-tauri/src/ipc/{mod,server,pipe_windows,peer}.rs` — the moved transport (Task 5); `src-tauri/src/agent/{mod,broker,oneshot,openssh_tests,session}.rs` — callers; `.github/workflows/test-windows.yml` — the test filter.
- `docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md` — the report (Task 6).

---

### Task 1: The two `ssh-key` versions live in one build

**Files:**
- Create: `spike/russh/Cargo.toml`
- Create: `spike/russh/src/lib.rs`
- Create: `spike/russh/tests/coexistence.rs`
- Create (written by `cargo fetch`): `spike/russh/Cargo.lock`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: the package `russh-spike` (library `russh_spike`) with `pub fn fact(key: &str, value: impl std::fmt::Display)` and `pub async fn within<T>(seconds: u64, what: &str, future: impl std::future::Future<Output = T>) -> T`; the cargo feature `app-vault` (on by default) that links `sshelter_lib`; a committed `Cargo.lock` that every later task builds with `--offline`.

What shapes this task:
- In `Cargo.toml` the app library is `sshelter = { path = "../../src-tauri" }` (the dependency key is the *package* name); in code it is `sshelter_lib::…` (the *library* name).
- Linking the library means compiling Tauri and everything it pulls in (about 700 crates): a long first build, budget ten minutes. `tauri::generate_context!` reads `../dist` at compile time; it exists in this checkout, and on a fresh one `mkdir -p dist && echo '<!doctype html>' > dist/index.html` is enough (as in `test-windows.yml`).
- The spike's lock file starts as a copy of the app's, so every crate the app already uses keeps its version and the delta in step 6 shows what russh adds.
- russh's defaults (`aws-lc-rs`, `flate2`, `rsa`) reuse the TLS crypto the app already builds. They do not reuse RustCrypto: russh's own `Cargo.toml` (docs.rs) lists a second, pre-release generation — `ssh-key =0.7.0-rc.11`, `rsa =0.10.0-rc.18`, `p256`/`p384`/`p521` 0.14, `ed25519-dalek` 3, `curve25519-dalek` 5, `ecdsa` 0.17, `elliptic-curve` 0.14, `crypto-bigint` 0.7, `ml-kem` 0.3, … — next to the app's `ssh-key` 0.6.7, `rsa` 0.9 and `ssh-encoding` 0.2. This task measures that.

- [ ] **Step 1: Write the manifest, `lib.rs` and the failing test**

`spike/russh/Cargo.toml`:

```toml
[package]
name = "russh-spike"
version = "0.0.0"
edition = "2021"
publish = false
description = "Phase 0 spike for SSHelter's own SSH client. Throwaway, not product code."

# Its own workspace root: the crate belongs to no workspace, and its Cargo.lock is not src-tauri's.
[workspace]

[features]
default = ["app-vault"]
# The app library (sshelter_lib: Tauri and everything it links). Task 2 signs through its vault::material.
# The Windows job builds with --no-default-features (see .github/workflows/spike-windows.yml).
app-vault = ["dep:sshelter"]

[dependencies]
russh = "=0.64.1"
tokio = { version = "1.52", features = ["rt-multi-thread", "macros", "net", "io-util", "time", "sync"] }
tempfile = "3"
thiserror = "2"
sshelter = { path = "../../src-tauri", optional = true }

[dev-dependencies]
# The app's ssh-key (the vault's version) under another name, so a test can hold it next to the ssh-key russh pins.
ssh-key-app = { package = "ssh-key", version = "0.6.7" }
```

`spike/russh/src/lib.rs`:

```rust
//! Phase 0 spike for SSHelter's own SSH client (docs/superpowers/plans/2026-10-10-own-ssh-phase0-spike.md).
//!
//! THROWAWAY, NOT PRODUCT CODE. It answers questions about russh 0.64.1 and about the app's key vault. Phase 1 writes
//! `ssh/russh_engine.rs` behind the `SshEngine` trait from scratch; nothing here is copied into src-tauri. The crate has no
//! clippy.toml, so a direct `Command::spawn` is fine here (in src-tauri every spawn goes through `crate::process`).

/// One measured fact. Task 6 greps these lines out of `cargo test -- --nocapture`: `FACT <key> = <value>`.
pub fn fact(key: &str, value: impl std::fmt::Display) {
    eprintln!("FACT {key} = {value}");
}

/// Await `future` for at most `seconds`; panic naming `what` when it takes longer (a hung test must fail, not hang CI).
pub async fn within<T>(seconds: u64, what: &str, future: impl std::future::Future<Output = T>) -> T {
    match tokio::time::timeout(std::time::Duration::from_secs(seconds), future).await {
        Ok(value) => value,
        Err(_) => panic!("timed out after {seconds} s: {what}"),
    }
}
```

`spike/russh/tests/coexistence.rs`:

```rust
//! Task 1: the app's `ssh-key` 0.6.7 and the `ssh-key` 0.7.0-rc that russh pins live in one build, and agree about a public key.

/// `test_keys::PLAIN_PUBLIC` and `PLAIN_FINGERPRINT` from src-tauri/src/sync/slot_rules.rs (the app's own tests pin them with ssh-key 0.6.7).
const ED25519_LINE: &str = "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIF5M9xdffT0p33BD1LiLFTiEvrjv4IZMFADC81ex4ndf";
const ED25519_FINGERPRINT: &str = "SHA256:9Q3QMhBJBcoUNE88XYEQbCPlcFByPPyVPJ6enJtQ+ew";
/// `test_keys::ECDSA_PUBLIC` and `ECDSA_FINGERPRINT`.
const ECDSA_LINE: &str = "ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEjFNKVD5g/ngM+x8oESowshddffHjvh+l4MYyyXDZMfnXpzylT6xjkTiMop0/8K1KN1+LyseBdXlLj6j5m8mvU=";
const ECDSA_FINGERPRINT: &str = "SHA256:vUthAmDZoxYXCTAPEZUn5qtWSMHWQCEcUfpnyM05mMs";

#[test]
fn both_ssh_key_versions_parse_the_same_public_keys_and_agree_on_the_fingerprints() {
    for (line, expected) in [(ED25519_LINE, ED25519_FINGERPRINT), (ECDSA_LINE, ECDSA_FINGERPRINT)] {
        let app = ssh_key_app::PublicKey::from_openssh(line).expect("ssh-key 0.6.7 parses the line");
        let russh = russh::keys::PublicKey::from_openssh(line).expect("ssh-key 0.7.0-rc parses the line");
        assert_eq!(app.fingerprint(ssh_key_app::HashAlg::Sha256).to_string(), expected);
        assert_eq!(russh.fingerprint(russh::keys::HashAlg::Sha256).to_string(), expected);
    }
}

/// The committed Cargo.lock holds both generations side by side: that is the whole coexistence question.
#[test]
fn the_lock_file_holds_ssh_key_0_6_7_and_a_0_7_release_candidate() {
    let lock = include_str!("../Cargo.lock");
    let versions: Vec<&str> = lock
        .split("[[package]]")
        .filter(|package| package.lines().any(|line| line == "name = \"ssh-key\""))
        .filter_map(|package| package.lines().find_map(|line| line.strip_prefix("version = \"")?.strip_suffix('"')))
        .collect();
    assert!(versions.contains(&"0.6.7"), "ssh-key 0.6.7 (the vault's) is missing: {versions:?}");
    assert!(versions.iter().any(|version| version.starts_with("0.7.0-rc.")), "russh's ssh-key 0.7.0-rc is missing: {versions:?}");
}

/// With the app library linked in (the default feature), its public API is callable next to russh. `starts_hidden` is public in src-tauri/src/lib.rs.
#[cfg(feature = "app-vault")]
#[test]
fn the_app_library_is_linked_into_the_same_binary() {
    assert!(sshelter_lib::starts_hidden(Some(std::ffi::OsStr::new("1"))));
}
```

- [ ] **Step 2: Start from the app's lock file**

```bash
cd /Users/ysya/project/sideproj/sshelter
cp src-tauri/Cargo.lock spike/russh/Cargo.lock
```

- [ ] **Step 3: Run it offline to see it fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-run
```

Expected: FAIL before compiling anything: `error: no matching package named `russh` found`, `location searched: crates.io index`, and the note that offline mode "can sometimes cause surprising resolution failures". (The cargo registry on this machine holds every crate of the app's lock, 695 of 695, but not russh.)

- [ ] **Step 4: Fetch — the only network step**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo fetch
```

Expected: it downloads `russh v0.64.1` and its new dependencies (only the new ones: the app's 695 are cached) and rewrites `Cargo.lock`. If it fails with a resolution error (a `links` conflict, an incompatible requirement), copy the whole error into the session scratchpad: that error is the answer to this task, and a NO-GO unless one retry resolves it. Retry once with the other crypto backend, `russh = { version = "=0.64.1", default-features = false, features = ["ring", "flate2", "rsa"] }`, and record which of the two resolved. Stop and report to the controller either way.

- [ ] **Step 5: Build and run the test (green)**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo build --offline
cargo test --offline --test coexistence
```

Expected: `cargo build` succeeds with both `ssh-key` generations in the tree (the first build compiles the whole app library and russh: several minutes), then `test result: ok. 3 passed; 0 failed`. The three tests: the two `ssh-key` generations parse the same two public keys and print the fingerprints the app's own tests pin; the committed lock holds `ssh-key` 0.6.7 and a `0.7.0-rc.*`; the app library is linked into the test binary. If the build stops on `generate_context!`, create the `dist` placeholder above.

- [ ] **Step 6: Record what the dependency graph looks like**

Save the lock delta script to the session scratchpad as `lock_delta.py` (it only reads two files):

```python
"""Task 1: what the spike's Cargo.lock adds to (or changes in) the app's. Usage: python3 -I lock_delta.py APP_LOCK SPIKE_LOCK
Prints FACT lines (for the report) and the lists behind them."""
import collections
import re
import sys


def load(path):
    packages = collections.defaultdict(set)
    for name, version in re.findall(r'\[\[package\]\]\nname = "([^"]+)"\nversion = "([^"]+)"', open(path).read()):
        packages[name].add(version)
    return packages


app, spike = load(sys.argv[1]), load(sys.argv[2])
new_crates = sorted(f"{name} {version}" for name in spike if name not in app for version in spike[name])
second_copies = sorted(f"{name}: app has {sorted(app[name])}, spike adds {sorted(spike[name] - app[name])}" for name in spike if name in app and spike[name] - app[name] and not (app[name] - spike[name]))
replaced = sorted(f"{name}: app has {sorted(app[name])}, spike has {sorted(spike[name])}" for name in spike if name in app and spike[name] - app[name] and app[name] - spike[name])
dropped = sorted(f"{name} {version}" for name in app if name not in spike for version in app[name])
prerelease = sorted(f"{name} {version}" for name in spike for version in spike[name] if re.search(r"-(rc|alpha|beta|pre)", version))

print(f"FACT lock.new_crates = {len(new_crates)}")
print(f"FACT lock.second_versions_of_crates_the_app_has = {len(second_copies)}")
print(f"FACT lock.app_versions_replaced = {len(replaced)}")
print(f"FACT lock.dropped_from_the_app_lock = {len(dropped)}")
print(f"FACT lock.prerelease_crates = {len(prerelease)}")
for title, items in [("New crates", new_crates), ("A second version of a crate the app already has", second_copies),
                     ("App versions REPLACED by another version (this would change src-tauri/Cargo.lock)", replaced),
                     ("In the app lock only (dev-dependencies are not resolved through a path dependency)", dropped),
                     ("Pre-release crates in the spike lock", prerelease)]:
    print(f"\n## {title} ({len(items)})")
    print("\n".join(items) if items else "(none)")
```

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo tree --offline -i ssh-key@0.6.7 | tee target/cargo-tree-ssh-key.txt && cargo tree --offline -i ssh-key@0.7.0-rc.11 | tee -a target/cargo-tree-ssh-key.txt
cargo tree --offline -i aws-lc-rs | tee target/cargo-tree-aws-lc-rs.txt
python3 -I "$SCRATCH/lock_delta.py" ../../src-tauri/Cargo.lock Cargo.lock | tee target/lock-delta.txt
```

(`$SCRATCH` is the session scratchpad directory.) Expected shape of the first command, as far as it can be known before russh is fetched: two roots, one per generation, the app's reached through `sshelter`, the russh one through `russh v0.64.1`, and the spike's own dev-dependency listed apart (the order of the blocks may differ):

```
ssh-key v0.6.7
└── sshelter v0.16.0 (…/src-tauri)
    └── russh-spike v0.0.0 (…/spike/russh)
[dev-dependencies]
└── russh-spike v0.0.0 (…/spike/russh)
ssh-key v0.7.0-rc.11
└── russh v0.64.1
    └── russh-spike v0.0.0 (…/spike/russh)
```

`aws-lc-rs` must show one version, 1.18.1, shared by the app (through rustls) and russh. In the delta, `FACT lock.app_versions_replaced` must be 0: anything else means adding russh to the app would change versions the app already ships; list those crates in the report. `FACT lock.new_crates` is the size of what phase 1 adds to `src-tauri/Cargo.lock`. The crates "in the app lock only" are the app's dev-dependencies (`ts-rs`, `mockito`, …), which a path dependency does not resolve.

- [ ] **Step 7: Check the app's files are untouched, then commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock
git add spike/russh/Cargo.toml spike/russh/Cargo.lock spike/russh/src/lib.rs spike/russh/tests/coexistence.rs
git diff --cached --stat
git commit -m "chore(spike): russh 0.64.1 and the app's ssh-key 0.6.7 build together"
```

Expected: the first command prints nothing; the staged files are exactly those four.

---

### Task 2: The vault signs for russh against a scratch sshd

**Files:**
- Modify: `src-tauri/src/lib.rs:19` (`mod vault;` → `pub mod vault;`) — the only app change outside Task 5
- Create: `spike/russh/src/harness.rs`, `spike/russh/src/client.rs`, `spike/russh/src/fixture.rs`, `spike/russh/src/signer.rs`
- Modify: `spike/russh/src/lib.rs` (append the module lines)
- Test: `spike/russh/tests/harness_smoke.rs`, `spike/russh/tests/vault_signer.rs`

**Interfaces:**
- Consumes: Task 1's crate and lock; `russh_spike::{fact, within}`.
- Produces for every later task:
  - `harness`: `Tools { sshd, ssh, ssh_keygen: PathBuf }`; `tools() -> Option<Tools>` (`None` and a "skipped" line when OpenSSH is missing, a failure when `SSHELTER_REQUIRE_OPENSSH=1`); `KeyKind::{Ed25519, EcdsaP256, Rsa3072}`; `TestKey { kind, private_path: PathBuf, private_text: String, public_line: String }`; `generate_key(&Tools, &Path, name: &str, KeyKind) -> TestKey`; `SshdOptions { host_key: KeyKind, extra_config: Vec<String>, authorized_keys: Vec<String> }` (with `Default`); `Sshd { port: u16, user: String, host_public_line: String }` with `Sshd::start(&Tools, SshdOptions) -> Sshd` and `log(&self) -> String`; `system_ssh(&Tools, &Sshd, identity: &Path, remote_command: &str) -> std::process::Output`.
  - `client`: `HostKeyVerdict::{AcceptAny, RejectAll, Pinned(String), Ask(UnboundedSender<HostKeyQuestion>)}`; `HostKeyQuestion { fingerprint, algorithm, reply: oneshot::Sender<bool> }`; `Observed { host_keys, disconnect }` with `wait_for_disconnect(&self, seconds) -> String`; `Connection { handle: Handle<SpikeHandler>, observed: Observed }`; `connect(port, Config, HostKeyVerdict) -> Result<Connection, russh::Error>`; `connect_over(stream, Config, HostKeyVerdict)`; `login_with_key_file(&mut Handle, user, &Path) -> Result<AuthResult, russh::Error>`; `ExecOutput { stdout, stderr, exit_status: Option<u32>, exit_signal: Option<String>, refused }` with `text()`; `exec(&Handle, command) -> Result<ExecOutput, russh::Error>`; `drain(&mut Channel<Msg>) -> ExecOutput`.
  - `fixture::Server { sshd, key }` with `Server::start(&Tools, extra_config: &[&str])`, `start_with(&Tools, host_key, user_key, extra_config)`, `host_fingerprint() -> String`, `login(&self, &mut Connection)`, `connect_and_login(&self, port, Config, HostKeyVerdict) -> Connection`, `session() -> Connection`.
  - `signer::VaultSigner` (`new(Material)`, public `calls`, `last_flags`, `last_algorithm`) and `signer::SignerError`.

What shapes this task:
- **There is no harness to reuse.** The spec's first wording ("reuse `openssh_tests.rs`'s way of starting sshd") was wrong: that file never starts a server. What the new harness takes from it: the tool lookup (including Win32-OpenSSH in `System32`), the `SSHELTER_REQUIRE_OPENSSH=1` skip-or-fail rule, and "never the real `~/.ssh`". The sshd config lines are the ones the controller verified (see Global Constraints, fact 2). Config values for one keyword: sshd keeps the first, so a test's extra lines go before the defaults.
- **The `Signer` contract**, read from russh 0.64.1's source on docs.rs (`auth.rs`, `client/encrypted.rs`, `keys/agent/client.rs`):
  - The trait (without the `async-trait` feature) is `pub trait Signer: Sized { type Error: From<SendError>; fn auth_sign(&mut self, key: &AgentIdentity, hash_alg: Option<HashAlg>, to_sign: Vec<u8>) -> impl Future<Output = Result<Vec<u8>, Self::Error>> + Send; }`, and `Handle::authenticate_publickey_with(user, PublicKey, hash_alg, &mut S) -> Result<AuthResult, S::Error>` calls it.
  - `to_sign` is `string(session id)` followed by `byte 50, string user, string "ssh-connection", string "publickey", byte 1, string algorithm, string key blob`: the bytes RFC 4252 §7 says to sign, with no signature yet.
  - russh expects `to_sign` back unchanged with `u32 length || signature blob` appended (the blob is `string algorithm || string signature`, one SSH string). It then cuts the session id off the front and sends the rest; a returned buffer of the same length as `to_sign` is taken as "nothing signed" and nothing is sent. This is what russh's own `AgentClient::sign_request` returns.
  - `Material::sign(data, flags)` returns exactly that blob (`signature_blob` in `vault/material.rs`), so the bridge is `to_sign ++ u32(len(blob)) ++ blob`.
  - The RSA hash arrives as `hash_alg`: `Some(Sha512)` is flag 4 (`SSH_AGENT_RSA_SHA2_512`), `Some(Sha256)` is 2, `None` is 0 (`ssh-rsa`, SHA-1). russh passes `hash_alg` for every key type, and the agent protocol only reads the flag for RSA, so the signer looks at the key first.
  - Only bytes cross between the two `ssh-key` generations (`Vec<u8>` in and out; the public key travels as an OpenSSH text line), so no type of one appears in the other's API.
- `best_supported_rsa_hash()` returns `Some(Some(hash))` when the server lists `rsa-sha2-*`, `Some(None)` when it lists only `ssh-rsa`, and `None` when it sent no `server-sig-algs`; the tests flatten it to `Option<HashAlg>`.

- [ ] **Step 1: Write the harness smoke test (it proves the harness, with the system `ssh`, before russh is involved)**

`spike/russh/tests/harness_smoke.rs`:

```rust
//! Task 2, step 1: the scratch sshd works with the system ssh client, so a later failure is russh's or the vault's, not the harness's.

use russh_spike::harness::{generate_key, system_ssh, tools, KeyKind, Sshd, SshdOptions};

#[test]
fn the_system_ssh_client_logs_in_to_the_scratch_sshd_and_gets_the_exit_code() {
    let Some(tools) = tools() else { return };
    let keys = tempfile::tempdir().unwrap();
    let key = generate_key(&tools, keys.path(), "user", KeyKind::Ed25519);
    let sshd = Sshd::start(&tools, SshdOptions { authorized_keys: vec![key.public_line.clone()], ..SshdOptions::default() });

    let out = system_ssh(&tools, &sshd, &key.private_path, "echo harness-ok; exit 5");
    assert_eq!(
        String::from_utf8_lossy(&out.stdout),
        "harness-ok\n",
        "ssh stderr: {}; sshd log: {}",
        String::from_utf8_lossy(&out.stderr),
        sshd.log()
    );
    assert_eq!(out.status.code(), Some(5));
}

/// The negative control: a key that is not in authorized_keys is refused, so the test above proves the server really checks keys.
#[test]
fn a_key_that_is_not_in_authorized_keys_is_refused() {
    let Some(tools) = tools() else { return };
    let keys = tempfile::tempdir().unwrap();
    let allowed = generate_key(&tools, keys.path(), "allowed", KeyKind::Ed25519);
    let stranger = generate_key(&tools, keys.path(), "stranger", KeyKind::Ed25519);
    let sshd = Sshd::start(&tools, SshdOptions { authorized_keys: vec![allowed.public_line.clone()], ..SshdOptions::default() });

    let out = system_ssh(&tools, &sshd, &stranger.private_path, "echo must-not-run");
    assert_eq!(out.status.code(), Some(255), "ssh exits 255 when it cannot log in");
    assert!(out.stdout.is_empty());
}
```

- [ ] **Step 2: Run it to see it fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test harness_smoke
```

Expected: FAIL to compile: `error[E0432]: unresolved import `russh_spike::harness`` … `could not find `harness` in `russh_spike``. (`--no-default-features` leaves the app library out: it is not needed until step 6.)

- [ ] **Step 3: Write the harness**

`spike/russh/src/harness.rs`:

```rust
//! A scratch OpenSSH server for the spike's tests.
//!
//! src-tauri has no sshd harness (`agent/openssh_tests.rs` only runs ssh-keygen and ssh-add against SSHelter's own agent), so this
//! builds one: a temp directory, a free localhost port, a throwaway host key and authorized_keys, and `sshd -D -e -f <config>` run
//! as the current user (no root, no PAM). Nothing here reads or writes the real ~/.ssh: the system `ssh` client used by
//! `system_ssh` gets `-F /dev/null`.
//!
//! Checked by hand on macOS (OpenSSH 10.3p1) before this plan was written: `Port 0` is rejected ("Badly formatted port number"), so
//! the port is picked first; sshd prints one harmless line `BSM audit: ... setaudit_addr failed: Operation not permitted`;
//! password and keyboard-interactive login cannot work without PAM, so this sshd offers public keys only.

use std::io::Read;
use std::net::{Ipv4Addr, SocketAddr, TcpListener, TcpStream};
use std::path::{Path, PathBuf};
use std::process::{Child, Command, Stdio};
use std::time::{Duration, Instant};

/// The OpenSSH programs the tests need.
pub struct Tools {
    pub sshd: PathBuf,
    pub ssh: PathBuf,
    pub ssh_keygen: PathBuf,
}

/// `Some` when sshd, ssh and ssh-keygen are all found. When they are not: `SSHELTER_REQUIRE_OPENSSH=1` (CI) makes that a failure, as in
/// `agent/openssh_tests.rs`; anything else prints "skipped" and the caller returns early.
pub fn tools() -> Option<Tools> {
    let found = find_sshd().and_then(|sshd| Some(Tools { sshd, ssh: find_in_path("ssh")?, ssh_keygen: find_in_path("ssh-keygen")? }));
    if found.is_none() {
        let required = std::env::var("SSHELTER_REQUIRE_OPENSSH").is_ok_and(|v| v == "1");
        assert!(!required, "sshd, ssh and ssh-keygen are required (SSHELTER_REQUIRE_OPENSSH=1)");
        eprintln!("skipped: sshd, ssh or ssh-keygen is not found (or this is not a Unix machine)");
    }
    found
}

/// sshd re-executes itself and insists on an absolute path, and a normal user rarely has it on PATH (Linux keeps it in /usr/sbin).
/// No scratch sshd on Windows: Win32-OpenSSH's sshd must run as a service, which `spike-windows.yml` sets up instead.
fn find_sshd() -> Option<PathBuf> {
    if !cfg!(unix) {
        return None;
    }
    ["/usr/sbin/sshd", "/usr/local/sbin/sshd", "/opt/homebrew/sbin/sshd", "/usr/bin/sshd"]
        .iter()
        .map(PathBuf::from)
        .find(|path| path.is_file())
        .or_else(|| find_in_path("sshd"))
}

fn find_in_path(name: &str) -> Option<PathBuf> {
    let file = if cfg!(windows) { format!("{name}.exe") } else { name.to_string() };
    #[cfg(windows)]
    {
        // The Win32-OpenSSH in System32 first: Git for Windows puts MSYS2 copies of ssh and ssh-keygen on PATH (see agent/openssh_tests.rs).
        if let Some(root) = std::env::var_os("SystemRoot") {
            let path = Path::new(&root).join("System32").join("OpenSSH").join(&file);
            if path.is_file() {
                return Some(path);
            }
        }
    }
    std::env::split_paths(&std::env::var_os("PATH")?).map(|dir| dir.join(&file)).find(|path| path.is_file())
}

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum KeyKind {
    Ed25519,
    EcdsaP256,
    Rsa3072,
}

impl KeyKind {
    fn keygen_args(self) -> &'static [&'static str] {
        match self {
            KeyKind::Ed25519 => &["-t", "ed25519"],
            KeyKind::EcdsaP256 => &["-t", "ecdsa", "-b", "256"],
            KeyKind::Rsa3072 => &["-t", "rsa", "-b", "3072"],
        }
    }
}

/// A key pair made by ssh-keygen, in the OpenSSH private key format the vault reads.
pub struct TestKey {
    pub kind: KeyKind,
    pub private_path: PathBuf,
    pub private_text: String,
    /// `<type> <base64> spike`: the authorized_keys form.
    pub public_line: String,
}

pub fn generate_key(tools: &Tools, dir: &Path, name: &str, kind: KeyKind) -> TestKey {
    let private_path = dir.join(name);
    let out = Command::new(&tools.ssh_keygen)
        .args(["-q", "-N", "", "-C", "spike"])
        .args(kind.keygen_args())
        .arg("-f")
        .arg(&private_path)
        .output()
        .expect("run ssh-keygen");
    assert!(out.status.success(), "ssh-keygen failed: {}", String::from_utf8_lossy(&out.stderr));
    let mut public_path = private_path.clone().into_os_string();
    public_path.push(".pub");
    TestKey {
        kind,
        private_text: std::fs::read_to_string(&private_path).expect("read the private key"),
        public_line: std::fs::read_to_string(&public_path).expect("read the public key").trim().to_string(),
        private_path,
    }
}

pub struct SshdOptions {
    /// The host key's type.
    pub host_key: KeyKind,
    /// Lines for sshd_config. sshd keeps the FIRST value of a keyword, so these come before the defaults below and may override them.
    pub extra_config: Vec<String>,
    /// Public key lines for authorized_keys.
    pub authorized_keys: Vec<String>,
}

impl Default for SshdOptions {
    fn default() -> Self {
        SshdOptions { host_key: KeyKind::Ed25519, extra_config: Vec::new(), authorized_keys: Vec::new() }
    }
}

/// A running scratch sshd. Dropping it kills the listener (connections already open end when their client goes away).
pub struct Sshd {
    pub port: u16,
    /// The account sshd authenticates: the user running the tests (without root, sshd can only log in that user).
    pub user: String,
    /// The host key in authorized_keys form.
    pub host_public_line: String,
    dir: tempfile::TempDir,
    child: Child,
}

impl Sshd {
    pub fn start(tools: &Tools, options: SshdOptions) -> Sshd {
        let dir = tempfile::Builder::new().prefix("spike-sshd-").tempdir().expect("temp dir");
        let host_key = generate_key(tools, dir.path(), "hostkey", options.host_key);
        let authorized = dir.path().join("authorized_keys");
        let lines: String = options.authorized_keys.iter().map(|line| format!("{line}\n")).collect();
        std::fs::write(&authorized, lines).expect("write authorized_keys");
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            std::fs::set_permissions(&authorized, std::fs::Permissions::from_mode(0o600)).expect("chmod authorized_keys");
        }
        let config_path = dir.path().join("sshd_config");
        let log_path = dir.path().join("sshd.log");
        for attempt in 1..=3 {
            let port = free_port();
            std::fs::write(&config_path, config_text(port, &host_key.private_path, &authorized, &options.extra_config)).expect("write sshd_config");
            // `-t` checks the config and the keys without starting anything, so a typo fails here, in sshd's own words.
            let check = Command::new(&tools.sshd).arg("-t").arg("-f").arg(&config_path).output().expect("run sshd -t");
            assert!(
                check.status.success(),
                "sshd -t rejected the config:\n{}\n--- config:\n{}",
                String::from_utf8_lossy(&check.stderr),
                std::fs::read_to_string(&config_path).unwrap_or_default()
            );
            let log = std::fs::File::create(&log_path).expect("create sshd.log");
            let mut child = Command::new(&tools.sshd)
                .args(["-D", "-e", "-f"])
                .arg(&config_path)
                .stdin(Stdio::null())
                .stdout(Stdio::null())
                .stderr(Stdio::from(log))
                .spawn()
                .expect("start sshd");
            if wait_for_banner(&mut child, port) {
                return Sshd { port, user: current_user(), host_public_line: host_key.public_line, dir, child };
            }
            let _ = child.kill();
            let _ = child.wait();
            eprintln!("sshd did not come up on port {port} (attempt {attempt}); its log:\n{}", std::fs::read_to_string(&log_path).unwrap_or_default());
        }
        panic!("sshd did not start in 3 attempts");
    }

    /// What sshd wrote to stderr (LogLevel ERROR: a few lines at most).
    pub fn log(&self) -> String {
        std::fs::read_to_string(self.dir.path().join("sshd.log")).unwrap_or_default()
    }
}

impl Drop for Sshd {
    fn drop(&mut self) {
        let _ = self.child.kill();
        let _ = self.child.wait();
    }
}

fn config_text(port: u16, host_key: &Path, authorized_keys: &Path, extra: &[String]) -> String {
    let mut lines: Vec<String> = extra.to_vec();
    lines.extend([
        format!("Port {port}"),
        "ListenAddress 127.0.0.1".to_string(),
        format!("HostKey \"{}\"", host_key.display()),
        "PidFile none".to_string(),
        "UsePAM no".to_string(),
        "PasswordAuthentication no".to_string(),
        "KbdInteractiveAuthentication no".to_string(),
        "PubkeyAuthentication yes".to_string(),
        format!("AuthorizedKeysFile \"{}\"", authorized_keys.display()),
        "StrictModes no".to_string(),
        "LogLevel ERROR".to_string(),
    ]);
    lines.join("\n") + "\n"
}

fn free_port() -> u16 {
    TcpListener::bind((Ipv4Addr::LOCALHOST, 0)).expect("bind a free port").local_addr().expect("local address").port()
}

/// Waits (up to 10 s) for an SSH banner on `port` from a sshd that is still running; false when the child exits first or nothing greets.
fn wait_for_banner(child: &mut Child, port: u16) -> bool {
    let deadline = Instant::now() + Duration::from_secs(10);
    while Instant::now() < deadline {
        if child.try_wait().ok().flatten().is_some() {
            return false;
        }
        if let Ok(mut stream) = TcpStream::connect_timeout(&SocketAddr::from((Ipv4Addr::LOCALHOST, port)), Duration::from_millis(500)) {
            let _ = stream.set_read_timeout(Some(Duration::from_secs(2)));
            let mut banner = [0u8; 8];
            if stream.read_exact(&mut banner).is_ok() && &banner == b"SSH-2.0-" {
                // Somebody else may own the port (the pick can race with another test): ours must still be alive a moment later.
                std::thread::sleep(Duration::from_millis(200));
                return child.try_wait().ok().flatten().is_none();
            }
        }
        std::thread::sleep(Duration::from_millis(100));
    }
    false
}

fn current_user() -> String {
    let out = Command::new("id").arg("-un").output().expect("run id -un");
    assert!(out.status.success(), "id -un failed");
    String::from_utf8_lossy(&out.stdout).trim().to_string()
}

/// Runs the system `ssh` client against `sshd`. `-F /dev/null` keeps it from reading the real ~/.ssh/config; the agent and known_hosts are off.
pub fn system_ssh(tools: &Tools, sshd: &Sshd, identity: &Path, remote_command: &str) -> std::process::Output {
    Command::new(&tools.ssh)
        .args(["-F", "/dev/null", "-p"])
        .arg(sshd.port.to_string())
        .arg("-i")
        .arg(identity)
        .args([
            "-o",
            "IdentitiesOnly=yes",
            "-o",
            "IdentityAgent=none",
            "-o",
            "BatchMode=yes",
            "-o",
            "StrictHostKeyChecking=no",
            "-o",
            "UserKnownHostsFile=/dev/null",
            "-o",
            "LogLevel=ERROR",
        ])
        .arg(format!("{}@127.0.0.1", sshd.user))
        .arg(remote_command)
        .output()
        .expect("run ssh")
}
```

Append to `spike/russh/src/lib.rs`:

```rust

pub mod harness;
```

- [ ] **Step 4: Run it (green)**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test harness_smoke
```

Expected: `test result: ok. 2 passed` in about a second; no `sshd` left running (`pgrep -fl spike-sshd-` prints nothing). If `sshd -t rejected the config` fires, its message contains sshd's words and the generated config: fix the config line it names. If the tools are missing the tests print `skipped:` and pass; on this Mac they are there.

- [ ] **Step 5: Commit the harness**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/src/harness.rs spike/russh/src/lib.rs spike/russh/tests/harness_smoke.rs
git commit -m "chore(spike): a scratch sshd harness for the spike's tests"
```

- [ ] **Step 6: Write the vault signer tests**

`spike/russh/tests/vault_signer.rs`:

```rust
//! Task 2: russh's `Signer`, backed by the app's vault (`Material::sign`), logs in to a scratch sshd with every key type the vault signs.
#![cfg(feature = "app-vault")]

use russh::client::{AuthResult, Config};
use russh::keys::HashAlg;
use russh_spike::client::{connect, exec, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::{generate_key, tools, KeyKind, Tools};
use russh_spike::signer::VaultSigner;
use russh_spike::{fact, within};
use sshelter_lib::vault::material::open;

struct Login {
    result: AuthResult,
    signer: VaultSigner,
    /// What `best_supported_rsa_hash` said, flattened: `None` means plain ssh-rsa.
    hash_alg: Option<HashAlg>,
    /// What `echo` printed over the session that the vault's signature opened (empty when the login failed).
    echoed: String,
}

/// Logs in to `server` presenting `presented_public_line` and signing with the vault entry made from `signing_key_text`, then runs `echo`.
/// The two come from one key pair, except in the negative control at the end of this file.
async fn log_in_through_the_vault(server: &Server, presented_public_line: &str, signing_key_text: &str) -> Login {
    let material = open(signing_key_text, None).expect("the vault opens the OpenSSH key ssh-keygen wrote");
    let mut signer = VaultSigner::new(material);
    let public = russh::keys::PublicKey::from_openssh(presented_public_line).expect("russh parses the public key line");

    let mut connection = within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::AcceptAny)).await.expect("connect");
    let hash_alg = within(10, "server-sig-algs", connection.handle.best_supported_rsa_hash()).await.expect("best_supported_rsa_hash").flatten();
    let result = within(30, "authenticate_publickey_with", connection.handle.authenticate_publickey_with(server.sshd.user.as_str(), public, hash_alg, &mut signer))
        .await
        .expect("authenticate_publickey_with");
    let echoed = if result.success() {
        within(20, "exec after the login", exec(&connection.handle, "echo vault-ok")).await.expect("exec").text()
    } else {
        String::new()
    };
    Login { result, signer, hash_alg, echoed }
}

/// A scratch sshd with a `host_key` host key (and `server_config` in front of its defaults) that accepts a fresh `kind` user key, and the vault login to it.
async fn login_with(tools: &Tools, kind: KeyKind, host_key: KeyKind, server_config: &[&str]) -> (Login, Server) {
    let server = Server::start_with(tools, host_key, kind, server_config);
    let login = log_in_through_the_vault(&server, &server.key.public_line, &server.key.private_text).await;
    (login, server)
}

#[tokio::test(flavor = "multi_thread")]
async fn an_ed25519_key_in_the_vault_logs_in() {
    let Some(tools) = tools() else { return };
    let (login, server) = login_with(&tools, KeyKind::Ed25519, KeyKind::Ed25519, &[]).await;
    assert!(login.result.success(), "refused; sshd log: {}", server.sshd.log());
    assert_eq!(login.signer.last_algorithm.as_deref(), Some("ssh-ed25519"));
    assert_eq!((login.signer.calls, login.signer.last_flags), (1, 0));
    assert_eq!(login.echoed, "vault-ok\n");
}

#[tokio::test(flavor = "multi_thread")]
async fn an_ecdsa_p256_key_in_the_vault_logs_in() {
    let Some(tools) = tools() else { return };
    let (login, server) = login_with(&tools, KeyKind::EcdsaP256, KeyKind::Ed25519, &[]).await;
    assert!(login.result.success(), "refused; sshd log: {}", server.sshd.log());
    assert_eq!(login.signer.last_algorithm.as_deref(), Some("ecdsa-sha2-nistp256"));
    assert_eq!((login.signer.calls, login.signer.last_flags), (1, 0));
    assert_eq!(login.echoed, "vault-ok\n");
}

#[tokio::test(flavor = "multi_thread")]
async fn an_rsa_3072_key_in_the_vault_logs_in_with_a_sha2_signature() {
    let Some(tools) = tools() else { return };
    let (login, server) = login_with(&tools, KeyKind::Rsa3072, KeyKind::Ed25519, &[]).await;
    assert!(login.result.success(), "refused; sshd log: {}", server.sshd.log());
    let algorithm = login.signer.last_algorithm.clone().unwrap_or_default();
    fact("signer.rsa.hash_alg_offered_by_russh", format!("{:?}", login.hash_alg));
    fact("signer.rsa.algorithm_on_the_wire", &algorithm);
    assert!(algorithm == "rsa-sha2-512" || algorithm == "rsa-sha2-256", "a modern server must get a SHA-2 RSA signature, got {algorithm}");
    assert!(login.hash_alg.is_some());
    assert_eq!(login.echoed, "vault-ok\n");
}

/// Review Focus 1. A legacy server (old router, NAS firmware) offers only ssh-rsa for the host key and accepts only ssh-rsa (SHA-1) user signatures.
/// russh must still talk to it with its default algorithm lists, and the vault must sign with flag 0. Skipped where the OpenSSH build
/// refuses SHA-1 signatures altogether (set SPIKE_SKIP_SHA1=1, e.g. on RHEL/Fedora crypto policies).
#[tokio::test(flavor = "multi_thread")]
async fn an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1() {
    if std::env::var_os("SPIKE_SKIP_SHA1").is_some() {
        eprintln!("skipped: SPIKE_SKIP_SHA1 is set");
        return;
    }
    let Some(tools) = tools() else { return };
    let (login, server) =
        login_with(&tools, KeyKind::Rsa3072, KeyKind::Rsa3072, &["HostKeyAlgorithms ssh-rsa", "PubkeyAcceptedAlgorithms ssh-rsa"]).await;
    fact("signer.sha1_only.hash_alg_offered_by_russh", format!("{:?}", login.hash_alg));
    fact("signer.sha1_only.algorithm_on_the_wire", login.signer.last_algorithm.clone().unwrap_or_default());
    assert!(login.result.success(), "refused; sshd log: {}", server.sshd.log());
    assert_eq!(login.hash_alg, None, "no rsa-sha2 variant on offer, so russh must ask for ssh-rsa");
    assert_eq!(login.signer.last_flags, 0);
    assert_eq!(login.signer.last_algorithm.as_deref(), Some("ssh-rsa"));
    assert_eq!(login.echoed, "vault-ok\n");
}

/// The negative control: sshd really verifies what the vault signs. The server knows key A; russh presents A's public key but the vault holds B.
#[tokio::test(flavor = "multi_thread")]
async fn a_signature_from_another_key_is_refused() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let other_dir = tempfile::tempdir().unwrap();
    let other = generate_key(&tools, other_dir.path(), "other", KeyKind::Ed25519);

    let login = log_in_through_the_vault(&server, &server.key.public_line, &other.private_text).await;
    assert!(!login.result.success(), "sshd accepted a signature made by a different key");
    assert_eq!(login.signer.calls, 1, "russh asked the vault once and the server said no");
    assert!(login.echoed.is_empty());
}
```

The fifth test is the negative control: sshd really verifies what the vault signs, so the other four are not passing for the wrong reason.

- [ ] **Step 7: Run them to see them fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --test vault_signer
```

Expected: FAIL to compile with `error[E0603]: module `vault` is private` (that is `mod vault;` in `src-tauri/src/lib.rs`) and `error[E0432]: unresolved imports `russh_spike::client`, `russh_spike::fixture`, `russh_spike::signer``.

- [ ] **Step 8: The one-line app change**

In `src-tauri/src/lib.rs`, line 19:

```rust
pub mod vault;
```

(it was `mod vault;`). Check the app still compiles with the one warning it already had, and nothing else:

```bash
cd /Users/ysya/project/sideproj/sshelter/src-tauri
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo check --offline --lib 2>&1 | grep -E "^(warning|error)"
```

Expected exactly: `warning: function `set_host_enabled` is never used` and `warning: `sshelter` (lib) generated 1 warning` (both are there before the change too). Any `private_interfaces` or other new warning means the line is not "just one line": stop and report.

- [ ] **Step 9: Write the russh side**

`spike/russh/src/client.rs`:

```rust
//! The spike's russh client side: a `Handler` that asks the test what to do with the host key, `connect`, login and exec helpers.
//!
//! Written against the docs.rs pages of russh 0.64.1 (read on 2026-10-10). Where a name here differs from the compiler's, the compiler wins.

use std::net::Ipv4Addr;
use std::path::Path;
use std::sync::{Arc, Mutex};
use std::time::Duration;

use russh::client::{self, AuthResult, Config, DisconnectReason, Handle, Handler, Msg};
use russh::keys::{HashAlg, PrivateKeyWithHashAlg, PublicKeyOrCertificate};
use russh::{Channel, ChannelMsg};
use tokio::sync::{mpsc, oneshot};

/// A host key the server presented and the test must judge. Answer through `reply` (the stand-in for the confirmation window).
pub struct HostKeyQuestion {
    pub fingerprint: String,
    pub algorithm: String,
    pub reply: oneshot::Sender<bool>,
}

#[derive(Clone)]
pub enum HostKeyVerdict {
    AcceptAny,
    RejectAll,
    /// Accept only this "SHA256:..." fingerprint.
    Pinned(String),
    /// Ask the test; the handshake waits for the answer.
    Ask(mpsc::UnboundedSender<HostKeyQuestion>),
}

/// What the handler saw, shared with the test.
#[derive(Clone, Default)]
pub struct Observed {
    /// "SHA256:..." of each host key the server presented, in order.
    pub host_keys: Arc<Mutex<Vec<String>>>,
    /// The `Debug` text of the reason, once russh has called `Handler::disconnected`.
    pub disconnect: Arc<Mutex<Option<String>>>,
}

impl Observed {
    /// Waits until russh has called `Handler::disconnected` and returns the reason.
    pub async fn wait_for_disconnect(&self, seconds: u64) -> String {
        crate::within(seconds, "russh to report the disconnect", async {
            loop {
                let reason = self.disconnect.lock().unwrap().clone();
                if let Some(reason) = reason {
                    return reason;
                }
                tokio::time::sleep(Duration::from_millis(50)).await;
            }
        })
        .await
    }
}

pub struct SpikeHandler {
    verdict: HostKeyVerdict,
    observed: Observed,
}

impl SpikeHandler {
    pub fn new(verdict: HostKeyVerdict) -> (SpikeHandler, Observed) {
        let observed = Observed::default();
        (SpikeHandler { verdict, observed: observed.clone() }, observed)
    }
}

impl Handler for SpikeHandler {
    type Error = russh::Error;

    async fn check_server_key(&mut self, server_public_key: &PublicKeyOrCertificate) -> Result<bool, Self::Error> {
        let key = server_public_key.public_key();
        let fingerprint = key.fingerprint(HashAlg::Sha256).to_string();
        self.observed.host_keys.lock().unwrap().push(fingerprint.clone());
        Ok(match &self.verdict {
            HostKeyVerdict::AcceptAny => true,
            HostKeyVerdict::RejectAll => false,
            HostKeyVerdict::Pinned(pinned) => *pinned == fingerprint,
            HostKeyVerdict::Ask(questions) => {
                let (reply, answer) = oneshot::channel();
                let question = HostKeyQuestion { fingerprint, algorithm: key.algorithm().to_string(), reply };
                questions.send(question).is_ok() && answer.await.unwrap_or(false)
            }
        })
    }

    async fn disconnected(&mut self, reason: DisconnectReason<Self::Error>) -> Result<(), Self::Error> {
        *self.observed.disconnect.lock().unwrap() = Some(format!("{reason:?}"));
        match reason {
            DisconnectReason::ReceivedDisconnect(_) => Ok(()),
            DisconnectReason::Error(error) => Err(error),
        }
    }
}

pub struct Connection {
    pub handle: Handle<SpikeHandler>,
    pub observed: Observed,
}

/// TCP to 127.0.0.1:`port`, then the SSH handshake. `Err` carries russh's own error (a rejected host key is `Error::UnknownKey`).
pub async fn connect(port: u16, config: Config, verdict: HostKeyVerdict) -> Result<Connection, russh::Error> {
    let (handler, observed) = SpikeHandler::new(verdict);
    let handle = client::connect(Arc::new(config), (Ipv4Addr::LOCALHOST, port), handler).await?;
    Ok(Connection { handle, observed })
}

/// The same handshake over a stream somebody else carries (a direct-tcpip channel in the jump tests).
pub async fn connect_over<R>(stream: R, config: Config, verdict: HostKeyVerdict) -> Result<Connection, russh::Error>
where
    R: tokio::io::AsyncRead + tokio::io::AsyncWrite + Unpin + Send + 'static,
{
    let (handler, observed) = SpikeHandler::new(verdict);
    let handle = client::connect_stream(Arc::new(config), stream, handler).await?;
    Ok(Connection { handle, observed })
}

/// Public-key login with a key russh loads itself (the vault's `Signer` is the other path: signer.rs).
pub async fn login_with_key_file(handle: &mut Handle<SpikeHandler>, user: &str, key_path: &Path) -> Result<AuthResult, russh::Error> {
    let key = russh::keys::load_secret_key(key_path, None).map_err(russh::Error::Keys)?;
    // `Some(Some(hash))`: the server lists rsa-sha2-*; `Some(None)`: it lists ssh-rsa only; `None`: it said nothing (no server-sig-algs).
    let hash_alg = handle.best_supported_rsa_hash().await?.flatten();
    handle.authenticate_publickey(user, PrivateKeyWithHashAlg::new(Arc::new(key), hash_alg)).await
}

/// What one exec channel delivered.
#[derive(Debug, Default)]
pub struct ExecOutput {
    pub stdout: Vec<u8>,
    pub stderr: Vec<u8>,
    pub exit_status: Option<u32>,
    /// The signal name when the command was killed by one (`Debug` of russh's `Sig`, e.g. "KILL").
    pub exit_signal: Option<String>,
    /// The server answered a channel request with Failure.
    pub refused: bool,
}

impl ExecOutput {
    pub fn text(&self) -> String {
        String::from_utf8_lossy(&self.stdout).into_owned()
    }
}

/// Opens a session channel, runs `command` (no PTY) and reads the channel until russh reports it closed.
pub async fn exec(handle: &Handle<SpikeHandler>, command: &str) -> Result<ExecOutput, russh::Error> {
    let mut channel = handle.channel_open_session().await?;
    channel.exec(true, command).await?;
    Ok(drain(&mut channel).await)
}

/// Reads `channel` until `wait()` says it is closed. Every channel needs a reader like this one: russh stalls the whole connection
/// behind a channel nobody reads (the spec's §6.3; `tests/channels.rs` measures it).
pub async fn drain(channel: &mut Channel<Msg>) -> ExecOutput {
    let mut out = ExecOutput::default();
    while let Some(message) = channel.wait().await {
        match message {
            ChannelMsg::Data { data } => out.stdout.extend_from_slice(&data),
            ChannelMsg::ExtendedData { data, ext: 1 } => out.stderr.extend_from_slice(&data),
            ChannelMsg::ExitStatus { exit_status } => out.exit_status = Some(exit_status),
            ChannelMsg::ExitSignal { signal_name, .. } => out.exit_signal = Some(format!("{signal_name:?}")),
            ChannelMsg::Failure => out.refused = true,
            _ => {}
        }
    }
    out
}
```

`spike/russh/src/fixture.rs`:

```rust
//! Test setup shared by the integration tests: a scratch sshd that accepts one fresh Ed25519 user key, and the connect-and-log-in steps.

use russh::client::Config;
use russh::keys::HashAlg;

use crate::client::{connect, login_with_key_file, Connection, HostKeyVerdict};
use crate::harness::{generate_key, KeyKind, Sshd, SshdOptions, TestKey, Tools};
use crate::within;

pub struct Server {
    pub sshd: Sshd,
    /// The user key sshd accepts.
    pub key: TestKey,
    _keys: tempfile::TempDir,
}

impl Server {
    /// A scratch sshd with an Ed25519 host key, accepting one fresh Ed25519 user key. `extra_config` goes in front of the defaults.
    pub fn start(tools: &Tools, extra_config: &[&str]) -> Server {
        Server::start_with(tools, KeyKind::Ed25519, KeyKind::Ed25519, extra_config)
    }

    pub fn start_with(tools: &Tools, host_key: KeyKind, user_key: KeyKind, extra_config: &[&str]) -> Server {
        let keys = tempfile::tempdir().expect("temp dir for the user key");
        let key = generate_key(tools, keys.path(), "user", user_key);
        let sshd = Sshd::start(
            tools,
            SshdOptions {
                host_key,
                extra_config: extra_config.iter().map(|line| line.to_string()).collect(),
                authorized_keys: vec![key.public_line.clone()],
            },
        );
        Server { sshd, key, _keys: keys }
    }

    /// "SHA256:..." of this server's host key, computed with the ssh-key that russh pins.
    pub fn host_fingerprint(&self) -> String {
        russh::keys::PublicKey::from_openssh(&self.sshd.host_public_line)
            .expect("russh parses the host key line")
            .fingerprint(HashAlg::Sha256)
            .to_string()
    }

    /// Log `connection` in with the user key; panics (with sshd's log) when the server refuses.
    pub async fn login(&self, connection: &mut Connection) {
        let result = within(30, "publickey login", login_with_key_file(&mut connection.handle, &self.sshd.user, &self.key.private_path))
            .await
            .expect("the login call");
        assert!(result.success(), "sshd refused the user key; its log: {}", self.sshd.log());
    }

    /// Connect to `port` (this server's own, or a proxy's in front of it) and log in.
    pub async fn connect_and_login(&self, port: u16, config: Config, verdict: HostKeyVerdict) -> Connection {
        let mut connection = within(30, "connect", connect(port, config, verdict)).await.expect("connect");
        self.login(&mut connection).await;
        connection
    }

    /// The common case: default config, host key accepted, logged in.
    pub async fn session(&self) -> Connection {
        self.connect_and_login(self.sshd.port, Config::default(), HostKeyVerdict::AcceptAny).await
    }
}
```

`spike/russh/src/signer.rs`:

```rust
//! russh's `Signer` backed by the app's key vault (`sshelter_lib::vault::material::Material::sign`).
//!
//! The contract, read from russh 0.64.1 (`auth.rs`, `client/encrypted.rs`, `keys/agent/client.rs` on docs.rs):
//! - `to_sign` is `string(session id) || SSH_MSG_USERAUTH_REQUEST ...` with the public key, but no signature yet: exactly the bytes to sign.
//! - the returned buffer must be `to_sign` unchanged, followed by the signature as one SSH string (`u32 length || signature blob`);
//!   russh then cuts the session id off the front and sends the rest. This is what russh's own `AgentClient::sign_request` returns.
//! - `Material::sign` returns the signature blob (`string algorithm || string signature`) the SSH agent protocol uses, so the length prefix is all that is missing.
//! - for RSA the algorithm comes from `hash_alg`: `Some(Sha512)` is flag 4 and `Some(Sha256)` is flag 2 (the agent protocol's flags,
//!   `SSH_AGENT_RSA_SHA2_512` / `_256`), `None` is flag 0, which is `ssh-rsa` (SHA-1). Other key types ignore the flag.

use std::sync::Arc;

use russh::keys::agent::AgentIdentity;
use russh::keys::HashAlg;
use sshelter_lib::vault::material::{Material, SSH_AGENT_RSA_SHA2_256, SSH_AGENT_RSA_SHA2_512};

#[derive(Debug, thiserror::Error)]
pub enum SignerError {
    #[error(transparent)]
    Send(#[from] russh::SendError),
    #[error("the vault could not sign: {0}")]
    Sign(String),
}

pub struct VaultSigner {
    material: Arc<Material>,
    /// How many times russh asked for a signature.
    pub calls: usize,
    /// The flag handed to `Material::sign` the last time.
    pub last_flags: u32,
    /// The algorithm name inside the last signature blob ("ssh-ed25519", "rsa-sha2-512", ...).
    pub last_algorithm: Option<String>,
}

impl VaultSigner {
    pub fn new(material: Material) -> VaultSigner {
        VaultSigner { material: Arc::new(material), calls: 0, last_flags: 0, last_algorithm: None }
    }
}

impl russh::Signer for VaultSigner {
    type Error = SignerError;

    async fn auth_sign(&mut self, key: &AgentIdentity, hash_alg: Option<HashAlg>, to_sign: Vec<u8>) -> Result<Vec<u8>, Self::Error> {
        // The agent protocol's flag only means something for RSA keys, but russh hands over its `hash_alg` whatever the key type is.
        let flags = if key.public_key().algorithm().is_rsa() {
            match hash_alg {
                Some(HashAlg::Sha512) => SSH_AGENT_RSA_SHA2_512,
                Some(HashAlg::Sha256) => SSH_AGENT_RSA_SHA2_256,
                _ => 0,
            }
        } else {
            0
        };
        // The design says the vault is called through spawn_blocking from the connection runtime (RSA signing takes milliseconds).
        let material = Arc::clone(&self.material);
        let (to_sign, blob) = tokio::task::spawn_blocking(move || {
            let blob = material.sign(&to_sign, flags).map_err(|error| error.to_string());
            (to_sign, blob)
        })
        .await
        .map_err(|error| SignerError::Sign(error.to_string()))?;
        let blob = blob.map_err(SignerError::Sign)?;

        self.calls += 1;
        self.last_flags = flags;
        self.last_algorithm = algorithm_of(&blob);

        let mut signed = to_sign;
        signed.extend_from_slice(&(blob.len() as u32).to_be_bytes());
        signed.extend_from_slice(&blob);
        Ok(signed)
    }
}

/// The first SSH string of a signature blob: the algorithm name.
fn algorithm_of(blob: &[u8]) -> Option<String> {
    let length = u32::from_be_bytes(blob.get(..4)?.try_into().ok()?) as usize;
    String::from_utf8(blob.get(4..4 + length)?.to_vec()).ok()
}

#[cfg(test)]
mod tests {
    use super::algorithm_of;

    #[test]
    fn the_algorithm_is_the_first_ssh_string_of_the_blob() {
        let mut blob = vec![0, 0, 0, 11];
        blob.extend_from_slice(b"ssh-ed25519");
        blob.extend_from_slice(&[0, 0, 0, 2, 9, 9]);
        assert_eq!(algorithm_of(&blob).as_deref(), Some("ssh-ed25519"));
        assert_eq!(algorithm_of(&[0, 0, 0, 5, b'a']), None, "a length that runs past the end");
        assert_eq!(algorithm_of(&[0, 0]), None, "no room for a length");
    }
}
```

Append to `spike/russh/src/lib.rs`:

```rust
pub mod client;
pub mod fixture;
#[cfg(feature = "app-vault")]
pub mod signer;
```

- [ ] **Step 10: Make it compile against the real russh**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --test vault_signer --no-run
```

This code was written from russh 0.64.1's pages on docs.rs and type-checked against a stub built from those signatures, never against the real crate. If the compiler disagrees, the compiler wins; look here first:

| Symptom | Likely cause and fix |
|---|---|
| `method `check_server_key` has an incompatible type for trait` (or the same for `disconnected`, `auth_sign`) | The `async fn` future is not `Send`, or the trait spells the return type differently. Write the method as `fn …(&mut self, …) -> impl std::future::Future<Output = …> + Send { async move { … } }` and move what it needs into the block. |
| `non-exhaustive patterns` on `HashAlg`, `ChannelMsg` or `AuthResult` | Add a `_ =>` arm (the code has one for the first two); for `AuthResult` use `if let` / `let … else`. |
| `` `russh::client::Msg` / `russh::keys::…` not found `` | Use the path the compiler suggests and note it for the report (docs.rs lists `Msg` among `client`'s enums; `Disconnect`, `Preferred`, `MethodKind`, `SendError`, `Channel`, `ChannelMsg`, `Signer` at the crate root; `PublicKey`, `HashAlg`, `Algorithm`, `PrivateKeyWithHashAlg`, `PublicKeyOrCertificate`, `load_secret_key` and `agent::AgentIdentity` under `russh::keys`). |
| `Algorithm::is_rsa` moves a value | `algorithm()` returns an owned `Algorithm` and `is_rsa` takes `self`; if it returns a reference in this version, use `matches!(…, Algorithm::Rsa { .. })`. |
| a `Debug`/`Display` bound is missing in a `{…:?}` | Format with the other one, or drop the field from the message. |

- [ ] **Step 11: Run the vault signer tests (green)**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --test vault_signer -- --nocapture
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib
```

Expected: `test result: ok. 5 passed` for the first (Ed25519, ECDSA P-256, RSA 3072, the SHA-1-only server, the wrong key refused), with lines like `FACT signer.rsa.algorithm_on_the_wire = rsa-sha2-512` and `FACT signer.sha1_only.algorithm_on_the_wire = ssh-rsa`; and the library's unit test `signer::tests::the_algorithm_is_the_first_ssh_string_of_the_blob` passes. A refusal shows sshd's log in the assertion message. Apply the Spike rule to anything that differs (for example a server that sends no `server-sig-algs` changes which test branch `hash_alg` takes, and a Linux whose OpenSSL disables SHA-1 fails the fourth test: set `SPIKE_SKIP_SHA1=1` there and say so in the report).

- [ ] **Step 12: Commit (two commits: the app line alone, then the spike)**

```bash
cd /Users/ysya/project/sideproj/sshelter
git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock
git add src-tauri/src/lib.rs
git commit -m "chore(vault): make the vault module public for the phase 0 spike"
git add spike/russh/src/client.rs spike/russh/src/fixture.rs spike/russh/src/signer.rs spike/russh/src/lib.rs spike/russh/tests/vault_signer.rs
git commit -m "test(spike): russh's Signer backed by the vault logs in with ed25519, ecdsa and rsa"
```

Expected: the `git diff --stat` prints nothing; the first commit changes one line in one file.

---

### Task 3: Exec, shell, login methods, host keys, keepalive and a jump against a scratch sshd

**Files:**
- Create: `spike/russh/src/shell.rs`, `spike/russh/src/auth.rs`, `spike/russh/src/proxy.rs`
- Modify: `spike/russh/src/lib.rs` (append the module lines, one group at a time)
- Test: `spike/russh/tests/exec.rs`, `shell.rs`, `auth_methods.rs`, `host_key.rs`, `keepalive.rs`, `jump.rs`, `channels.rs`

**Interfaces:**
- Consumes: everything Task 2 produces (`harness`, `client`, `fixture::Server`). None of these tests needs the vault, so they run with `--no-default-features` (no Tauri in the build).
- Produces: `shell::{open_shell(&Handle<SpikeHandler>, term: &str, cols: u32, rows: u32) -> Result<Channel<Msg>, russh::Error>, type_line(&Channel<Msg>, &str), read_until(&mut Channel<Msg>, needle) -> Result<String, String>, read_until_found(&mut Channel<Msg>, impl Fn(&str) -> Option<T>) -> Result<T, String>, stty_size(&mut Channel<Msg>) -> Result<String, String>}`; `auth::{login_password(&mut Handle, user, password) -> Result<AuthResult, russh::Error>, login_keyboard_interactive(&mut Handle, user, answer) -> Result<bool, russh::Error>}`; `proxy::{Proxy::start(target_port) -> Proxy, Proxy::set(Mode), Proxy { port }, Mode::{Forward, Blackhole, Cut}}`.

How the groups work. These are characterization tests: they pin what russh and sshd do. Where a group adds a helper (3b, 3c, 3d) the failing step is the missing helper (a compile error); where it adds none (3a, 3e) there is no meaningful red, and a failure when you first run them is a finding (Spike rule). Every group ends green and committed. The per-group commit keeps a failing group from hiding a good one.

#### 3a: exec — exit code, both streams, a signal, long output, a cap, a timeout

- [ ] **Step 1: Write the tests**

`spike/russh/tests/exec.rs`:

```rust
//! Task 3a: exec channels against a scratch sshd: exit code, both streams, a command killed by a signal, a lot of output, a capped reader, a timeout.

use std::time::{Duration, Instant};

use russh::ChannelMsg;
use russh_spike::client::{drain, exec};
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::{fact, within};

#[tokio::test(flavor = "multi_thread")]
async fn exec_returns_stdout_stderr_and_the_exit_code() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let out = within(20, "exec", exec(&connection.handle, "printf out; printf err >&2; exit 7")).await.unwrap();
    assert_eq!(out.text(), "out");
    assert_eq!(String::from_utf8_lossy(&out.stderr), "err");
    assert_eq!(out.exit_status, Some(7));
    assert_eq!(out.exit_signal, None);
    assert!(!out.refused);
}

/// Review Focus 3, first half. `exec` replaces the login shell, so the process sshd waits for is the one that kills itself: sshd
/// sends `exit-signal` (checked with `ssh -vv`: "rtype exit-signal"), never `exit-status`. The caller must not read that as exit code 0.
#[tokio::test(flavor = "multi_thread")]
async fn a_command_killed_by_a_signal_reports_the_signal_and_no_exit_status() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let out = within(20, "exec", exec(&connection.handle, "exec sh -c 'kill -9 $$'")).await.unwrap();
    fact("exec.killed_by_signal.exit_status", format!("{:?}", out.exit_status));
    fact("exec.killed_by_signal.exit_signal", format!("{:?}", out.exit_signal));
    assert_eq!(out.exit_status, None);
    assert_eq!(out.exit_signal.as_deref(), Some("KILL"));
}

/// Review Focus 5, first half. 32 MiB is sixteen times russh's initial window (2 MiB), so this only completes if window adjustments keep up.
#[tokio::test(flavor = "multi_thread")]
async fn thirty_two_mebibytes_of_output_arrive_complete() {
    const SIZE: usize = 32 * 1024 * 1024;
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let started = Instant::now();
    let out = within(120, "32 MiB of output", exec(&connection.handle, &format!("head -c {SIZE} /dev/zero"))).await.unwrap();
    let seconds = started.elapsed().as_secs_f64();
    fact("exec.long_output.bytes", out.stdout.len());
    fact("exec.long_output.seconds", format!("{seconds:.2}"));
    fact("exec.long_output.mib_per_second", format!("{:.1}", SIZE as f64 / 1_048_576.0 / seconds));
    assert_eq!(out.stdout.len(), SIZE);
    assert_eq!(out.exit_status, Some(0));
}

/// Review Focus 5, second half. A command that never stops printing (`yes`) must not wedge the connection: the reader stops at a cap,
/// closes the channel, and the next exec on the same connection still works. This is how `ssh_exec` will limit output.
#[tokio::test(flavor = "multi_thread")]
async fn a_reader_that_stops_at_a_cap_and_closes_the_channel_leaves_the_connection_usable() {
    const CAP: usize = 1024 * 1024;
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let mut channel = connection.handle.channel_open_session().await.unwrap();
    channel.exec(true, "yes spike").await.unwrap();
    let mut read = 0usize;
    within(30, "reading up to the cap", async {
        while let Some(message) = channel.wait().await {
            if let ChannelMsg::Data { data } = message {
                read += data.len();
                if read >= CAP {
                    break;
                }
            }
        }
    })
    .await;
    assert!(read >= CAP, "the channel ended after {read} bytes");
    channel.close().await.unwrap();

    let started = Instant::now();
    let out = within(20, "exec after the capped channel", exec(&connection.handle, "echo alive")).await.unwrap();
    fact("exec.capped_reader.next_exec_ms", started.elapsed().as_millis());
    assert_eq!(out.text(), "alive\n");
}

/// russh has no per-command timeout; the engine's `exec(command, timeout)` has to be built from `tokio::time::timeout` and `Channel::close`.
#[tokio::test(flavor = "multi_thread")]
async fn a_command_timeout_is_ours_to_enforce_and_the_connection_survives_it() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let mut channel = connection.handle.channel_open_session().await.unwrap();
    channel.exec(true, "sleep 30").await.unwrap();
    let started = Instant::now();
    let waited = tokio::time::timeout(Duration::from_secs(1), drain(&mut channel)).await;
    assert!(waited.is_err(), "sleep 30 cannot have finished");
    fact("exec.timeout.waited_ms", started.elapsed().as_millis());
    channel.close().await.unwrap();

    let out = within(20, "exec after the timed-out channel", exec(&connection.handle, "echo alive")).await.unwrap();
    assert_eq!(out.text(), "alive\n");
}
```

- [ ] **Step 2: Run them**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test exec -- --nocapture
```

Expected: `test result: ok. 5 passed`, with `FACT exec.killed_by_signal.exit_signal = Some("KILL")`, `FACT exec.long_output.mib_per_second = …` (tens to hundreds on loopback), `FACT exec.capped_reader.next_exec_ms = …` and `FACT exec.timeout.waited_ms = …`. The "killed by a signal" test relies on sshd sending `exit-signal` (confirmed with `ssh -vv`: `rtype exit-signal`) and on `format!("{:?}", Sig::KILL)` being `KILL`. If the 32 MiB test or the capped reader hang, `within` fails them with the name of what timed out: that is Review Focus 5 failing, and the most valuable thing this run can find.

- [ ] **Step 3: Commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/tests/exec.rs
git commit -m "test(spike): exec against a scratch sshd — exit codes, signals, long output, a cap"
```

#### 3b: shell with a PTY — prompt, command, exit status, window size

- [ ] **Step 1: Write the tests**

`spike/russh/tests/shell.rs`:

```rust
//! Task 3b: an interactive shell with a PTY: a prompt, a command, the exit status, the window size.
//!
//! The scratch sshd forces `/bin/sh` (`ForceCommand`), so the prompt and the startup files are the same on every machine instead of
//! the developer's zsh configuration. The markers are chosen so the PTY's echo of what was typed can never be mistaken for the answer.

use std::time::Instant;

use russh_spike::client::drain;
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::shell::{open_shell, read_until, stty_size, type_line};
use russh_spike::{fact, within};

const SHELL_SERVER: &[&str] = &["ForceCommand /bin/sh"];

#[tokio::test(flavor = "multi_thread")]
async fn a_pty_shell_prints_a_prompt_runs_a_command_and_reports_its_exit_status() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, SHELL_SERVER);
    let connection = server.session().await;

    let started = Instant::now();
    let mut channel = within(20, "open the shell", open_shell(&connection.handle, "xterm-256color", 80, 24)).await.unwrap();
    // A POSIX sh prompt ends in "$ " (dash: "$ ", macOS bash as sh: "sh-3.2$ ").
    within(20, "the prompt", read_until(&mut channel, "$ ")).await.expect("a prompt");
    fact("shell.first_prompt_ms", started.elapsed().as_millis());

    // "$((6*7))" is typed, "42" is printed: the echo of the typed line does not contain SPIKE_42.
    type_line(&channel, "echo SPIKE_$((6*7))").await.unwrap();
    within(10, "the command's output", read_until(&mut channel, "SPIKE_42")).await.expect("the output");

    type_line(&channel, "exit 3").await.unwrap();
    let rest = within(10, "the shell's exit", drain(&mut channel)).await;
    assert_eq!(rest.exit_status, Some(3));
}

#[tokio::test(flavor = "multi_thread")]
async fn the_size_given_with_the_pty_request_is_the_initial_size() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, SHELL_SERVER);
    let connection = server.session().await;

    let mut channel = open_shell(&connection.handle, "xterm-256color", 97, 31).await.unwrap();
    within(20, "the prompt", read_until(&mut channel, "$ ")).await.unwrap();
    assert_eq!(within(10, "stty size", stty_size(&mut channel)).await.unwrap(), "31 97");
}

#[tokio::test(flavor = "multi_thread")]
async fn a_window_change_while_the_shell_runs_is_applied() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, SHELL_SERVER);
    let connection = server.session().await;

    let mut channel = open_shell(&connection.handle, "xterm-256color", 80, 24).await.unwrap();
    within(20, "the prompt", read_until(&mut channel, "$ ")).await.unwrap();
    channel.window_change(100, 30, 0, 0).await.unwrap();
    assert_eq!(within(10, "stty size", stty_size(&mut channel)).await.unwrap(), "30 100");
}

/// Review Focus 4. The user resizes the terminal the moment `sshelter connect` starts: the `window-change` goes out after `pty-req` but
/// before the shell is ready (nothing waits for the replies). It must not be lost, and must not break the shell.
#[tokio::test(flavor = "multi_thread")]
async fn a_window_change_sent_before_the_shell_is_ready_is_not_lost() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, SHELL_SERVER);
    let connection = server.session().await;

    let mut channel = connection.handle.channel_open_session().await.unwrap();
    channel.request_pty(true, "xterm-256color", 80, 24, 0, 0, &[]).await.unwrap();
    channel.window_change(120, 40, 0, 0).await.unwrap();
    channel.request_shell(true).await.unwrap();
    within(20, "the prompt", read_until(&mut channel, "$ ")).await.unwrap();
    assert_eq!(within(10, "stty size", stty_size(&mut channel)).await.unwrap(), "40 120");
}

/// A fact, not a requirement: what sshd does with a `window-change` that arrives before there is a PTY at all. The broker has to keep the
/// latest size and replay it once the shell is up if sshd drops this one.
#[tokio::test(flavor = "multi_thread")]
#[ignore = "an observation for the report: run with --ignored --nocapture"]
async fn observe_a_window_change_sent_before_the_pty_request() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, SHELL_SERVER);
    let connection = server.session().await;

    let mut channel = connection.handle.channel_open_session().await.unwrap();
    channel.window_change(120, 40, 0, 0).await.unwrap();
    channel.request_pty(true, "xterm-256color", 80, 24, 0, 0, &[]).await.unwrap();
    channel.request_shell(true).await.unwrap();
    within(20, "the prompt", read_until(&mut channel, "$ ")).await.unwrap();
    let size = within(10, "stty size", stty_size(&mut channel)).await.unwrap();
    fact("shell.window_change_before_pty_request.final_size", &size);
    fact("shell.window_change_before_pty_request.was_kept", size == "40 120");
}
```

- [ ] **Step 2: Run them to see them fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test shell
```

Expected: FAIL to compile: `error[E0432]: unresolved import `russh_spike::shell``.

- [ ] **Step 3: Write the shell helpers**

`spike/russh/src/shell.rs`:

```rust
//! Interactive-shell helpers: open a PTY shell, type into it, read until something shows up.

use russh::client::Msg;
use russh::{Channel, ChannelMsg};

use crate::client::SpikeHandler;

/// Session channel, `pty-req` (no terminal modes), then `shell`. Nothing waits for the replies.
pub async fn open_shell(handle: &russh::client::Handle<SpikeHandler>, term: &str, cols: u32, rows: u32) -> Result<Channel<Msg>, russh::Error> {
    let channel = handle.channel_open_session().await?;
    channel.request_pty(true, term, cols, rows, 0, 0, &[]).await?;
    channel.request_shell(true).await?;
    Ok(channel)
}

/// Types `line` and presses Enter.
pub async fn type_line(channel: &Channel<Msg>, line: &str) -> Result<(), russh::Error> {
    let bytes = format!("{line}\n");
    channel.data(bytes.as_bytes()).await
}

/// Reads (stdout and stderr arrive merged on a PTY) until `find` returns something for everything read so far.
/// `Err` carries what was read when the channel ended first.
pub async fn read_until_found<T>(channel: &mut Channel<Msg>, find: impl Fn(&str) -> Option<T>) -> Result<T, String> {
    let mut seen = String::new();
    while let Some(message) = channel.wait().await {
        if let ChannelMsg::Data { data } | ChannelMsg::ExtendedData { data, .. } = message {
            seen.push_str(&String::from_utf8_lossy(&data));
            if let Some(found) = find(&seen) {
                return Ok(found);
            }
        }
    }
    Err(format!("the channel closed first; read so far: {seen:?}"))
}

pub async fn read_until(channel: &mut Channel<Msg>, needle: &str) -> Result<String, String> {
    read_until_found(channel, |seen| seen.contains(needle).then(|| seen.to_string())).await
}

/// Asks the shell for the terminal size and returns "<rows> <cols>". The command echoed back by the PTY reads `SIZE[%s]`, the answer
/// `SIZE[40 120]`: only the answer has two numbers between the brackets.
pub async fn stty_size(channel: &mut Channel<Msg>) -> Result<String, String> {
    type_line(channel, "printf 'SIZE[%s]\\n' \"$(stty size)\"").await.map_err(|error| error.to_string())?;
    read_until_found(channel, size_in).await
}

fn size_in(output: &str) -> Option<String> {
    output.split("SIZE[").skip(1).find_map(|rest| {
        let inside = rest.split(']').next()?;
        let numbers: Vec<&str> = inside.split(' ').collect();
        let digits = |text: &&str| !text.is_empty() && text.bytes().all(|byte| byte.is_ascii_digit());
        (numbers.len() == 2 && numbers.iter().all(digits)).then(|| inside.to_string())
    })
}

#[cfg(test)]
mod tests {
    use super::size_in;

    #[test]
    fn only_the_answer_counts_not_the_echoed_command() {
        assert_eq!(size_in("printf 'SIZE[%s]\\n' \"$(stty size)\"\r\nSIZE[40 120]\r\n"), Some("40 120".to_string()));
        assert_eq!(size_in("printf 'SIZE[%s]\\n' \"$(stty size)\"\r\n"), None);
        assert_eq!(size_in("SIZE[0 0]"), Some("0 0".to_string()));
        assert_eq!(size_in("SIZE[40]"), None);
        assert_eq!(size_in("SIZE[40 12x]"), None);
    }
}
```

Append to `spike/russh/src/lib.rs`:

```rust
pub mod shell;
```

- [ ] **Step 4: Run them (green), then the observation**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo test --offline --no-default-features --lib shell
cargo test --offline --no-default-features --test shell -- --nocapture
cargo test --offline --no-default-features --test shell -- --ignored --nocapture
```

Expected: the unit test `shell::tests::only_the_answer_counts_not_the_echoed_command` passes; the shell tests: `4 passed; 0 failed; 1 ignored` with `FACT shell.first_prompt_ms = …`; the ignored one, run alone, prints `FACT shell.window_change_before_pty_request.final_size = …` and `…was_kept = true|false`. The scratch sshd forces `/bin/sh`, so the prompt ends in `$ ` on macOS (`sh-3.2$ `) and Linux (`$ `); a machine where the test user is root would print `# ` — run as a normal user. Checked with the system `ssh -tt` against this sshd configuration: the shell prints `SIZE[…]`, `SPIKE_42` and exits with the status given to `exit`.

- [ ] **Step 5: Commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/src/shell.rs spike/russh/src/lib.rs spike/russh/tests/shell.rs
git commit -m "test(spike): a PTY shell against a scratch sshd — prompt, exit status, window size"
```

#### 3c: login methods and the host key callback

- [ ] **Step 1: Write the tests**

`spike/russh/tests/auth_methods.rs`:

```rust
//! Task 3c, part 1: what a key-only server says to the other login methods. The success paths of password and keyboard-interactive need a
//! real host (a scratch sshd has no PAM): "verify on a real host" in the report. What runs here is the call sequence up to the server's
//! refusal, and the list of methods the refusal carries (the `tried` list of `ConnectError::AuthFailed`).

use russh::client::{AuthResult, Config, KeyboardInteractiveAuthResponse};
use russh::MethodKind;
use russh_spike::auth::{login_keyboard_interactive, login_password};
use russh_spike::client::{connect, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::{fact, within};

async fn unauthenticated(server: &Server) -> russh_spike::client::Connection {
    within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::AcceptAny)).await.unwrap()
}

#[tokio::test(flavor = "multi_thread")]
async fn authenticate_none_lists_the_methods_the_server_offers() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let mut connection = unauthenticated(&server).await;

    let result = connection.handle.authenticate_none(server.sshd.user.as_str()).await.unwrap();
    let AuthResult::Failure { remaining_methods, partial_success } = result else { panic!("a key-only server accepted `none`") };
    fact("auth.none.remaining_methods", format!("{remaining_methods:?}"));
    assert!(!partial_success);
    assert!(remaining_methods.contains(&MethodKind::PublicKey));
    assert!(!remaining_methods.contains(&MethodKind::Password));
    assert!(!remaining_methods.contains(&MethodKind::KeyboardInteractive));
}

#[tokio::test(flavor = "multi_thread")]
async fn a_password_is_refused_by_a_key_only_server_and_the_refusal_lists_what_remains() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let mut connection = unauthenticated(&server).await;

    let result = login_password(&mut connection.handle, &server.sshd.user, "not-the-password").await.unwrap();
    let AuthResult::Failure { remaining_methods, .. } = result else { panic!("the password was accepted") };
    fact("auth.password.remaining_methods", format!("{remaining_methods:?}"));
    assert!(remaining_methods.contains(&MethodKind::PublicKey));
}

#[tokio::test(flavor = "multi_thread")]
async fn keyboard_interactive_is_refused_by_a_key_only_server() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let mut connection = unauthenticated(&server).await;

    let response = connection.handle.authenticate_keyboard_interactive_start(server.sshd.user.as_str(), None::<String>).await.unwrap();
    assert!(matches!(response, KeyboardInteractiveAuthResponse::Failure { .. }), "got {response:?}");

    // The helper that will drive the prompts on a real host agrees: refused, and it never asked a question.
    let mut connection = unauthenticated(&server).await;
    let mut asked = 0;
    let accepted = login_keyboard_interactive(&mut connection.handle, &server.sshd.user, |_, _, _| {
        asked += 1;
        Vec::new()
    })
    .await
    .unwrap();
    assert!(!accepted);
    assert_eq!(asked, 0);
}
```

`spike/russh/tests/host_key.rs`:

```rust
//! Task 3c, part 2: the host key callback. `check_server_key` is the only place the engine learns the server's key; it can say yes, no,
//! or take its time (the confirmation window).

use std::borrow::Cow;
use std::time::{Duration, Instant};

use russh::client::Config;
use russh::keys::{Algorithm, HashAlg};
use russh::Preferred;
use russh_spike::client::{connect, exec, HostKeyQuestion, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::{tools, KeyKind};
use russh_spike::{fact, within};
use tokio::sync::mpsc;

#[tokio::test(flavor = "multi_thread")]
async fn accepting_the_key_connects_and_the_handler_saw_the_servers_fingerprint() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::AcceptAny)).await.unwrap();
    assert_eq!(*connection.observed.host_keys.lock().unwrap(), vec![server.host_fingerprint()]);
}

#[tokio::test(flavor = "multi_thread")]
async fn rejecting_the_key_fails_the_connect_with_unknown_key() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let error = within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::RejectAll)).await.err().expect("a rejected host key must fail the connect");
    fact("host_key.reject.error", format!("{error:?}"));
    assert!(matches!(error, russh::Error::UnknownKey), "got {error:?}");
}

/// Both "a key we have never seen" and "a key that changed" reach the engine's policy as the same callback, and a no from either is the
/// same error: the engine must remember WHY it said no (`HostKeyMismatch` vs the user declining) because russh does not tell.
#[tokio::test(flavor = "multi_thread")]
async fn a_pinned_fingerprint_that_matches_connects_and_one_that_differs_is_refused() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);

    let pinned = HostKeyVerdict::Pinned(server.host_fingerprint());
    within(30, "connect with the right pin", connect(server.sshd.port, Config::default(), pinned)).await.unwrap();

    let wrong = HostKeyVerdict::Pinned("SHA256:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA".to_string());
    let error = within(30, "connect with the wrong pin", connect(server.sshd.port, Config::default(), wrong)).await.err().expect("a changed key must fail");
    assert!(matches!(error, russh::Error::UnknownKey), "got {error:?}");
}

/// The broker asks in the app window: the handshake waits for the answer and goes on afterwards.
#[tokio::test(flavor = "multi_thread")]
async fn the_answer_can_arrive_seconds_later_and_the_connection_goes_on() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let (questions, mut inbox) = mpsc::unbounded_channel::<HostKeyQuestion>();
    let fingerprint = server.host_fingerprint();
    let asker = tokio::spawn(async move {
        let question = inbox.recv().await.expect("the handler asks");
        assert_eq!(question.fingerprint, fingerprint);
        tokio::time::sleep(Duration::from_secs(2)).await;
        let _ = question.reply.send(true);
        question.algorithm
    });

    let started = Instant::now();
    let mut connection = within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::Ask(questions))).await.unwrap();
    assert!(started.elapsed() >= Duration::from_secs(2), "the handshake did not wait for the answer");
    fact("host_key.ask.algorithm", asker.await.unwrap());
    server.login(&mut connection).await;
    assert_eq!(within(20, "exec", exec(&connection.handle, "echo asked")).await.unwrap().text(), "asked\n");
}

/// A fact, not a requirement: sshd drops a connection that has not logged in within `LoginGraceTime` (default 120 s), and the time the
/// user needs to read the host key dialog counts. With 3 s of grace and an answer after 6 s, this shows how the connect fails.
#[tokio::test(flavor = "multi_thread")]
#[ignore = "an observation for the report: run with --ignored --nocapture"]
async fn observe_an_answer_slower_than_the_servers_login_grace_time() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &["LoginGraceTime 3"]);
    let (questions, mut inbox) = mpsc::unbounded_channel::<HostKeyQuestion>();
    tokio::spawn(async move {
        let question = inbox.recv().await.expect("the handler asks");
        tokio::time::sleep(Duration::from_secs(6)).await;
        let _ = question.reply.send(true);
    });
    let started = Instant::now();
    let outcome = within(30, "connect", connect(server.sshd.port, Config::default(), HostKeyVerdict::Ask(questions))).await;
    fact("host_key.slow_answer.outcome", match &outcome {
        Ok(_) => "connected".to_string(),
        Err(error) => format!("failed: {error:?}"),
    });
    fact("host_key.slow_answer.after_ms", started.elapsed().as_millis());
}

/// Review Focus 2. A server whose host key algorithm russh does not offer: the error names both lists, which is what the user message needs.
/// (The client is limited to rsa-sha2-256 host keys; the server only has ssh-ed25519. The same error is what a legacy server with only
/// ssh-dss or only a cipher russh dropped would produce.)
#[tokio::test(flavor = "multi_thread")]
async fn no_common_host_key_algorithm_names_both_lists() {
    const ONLY_RSA_SHA256: &[Algorithm] = &[Algorithm::Rsa { hash: Some(HashAlg::Sha256) }];
    let Some(tools) = tools() else { return };
    let server = Server::start_with(&tools, KeyKind::Ed25519, KeyKind::Ed25519, &["HostKeyAlgorithms ssh-ed25519"]);
    let config = Config { preferred: Preferred { key: Cow::Borrowed(ONLY_RSA_SHA256), ..Preferred::DEFAULT }, ..Config::default() };

    let error = within(30, "connect", connect(server.sshd.port, config, HostKeyVerdict::AcceptAny)).await.err().expect("no common algorithm must fail");
    fact("host_key.no_common_algorithm.error", format!("{error:?}"));
    match error {
        russh::Error::NoCommonAlgo { kind, ours, theirs } => {
            fact("host_key.no_common_algorithm.kind", format!("{kind:?}"));
            assert!(theirs.iter().any(|name| name == "ssh-ed25519"), "the server's list: {theirs:?}");
            assert!(ours.iter().any(|name| name == "rsa-sha2-256"), "our list: {ours:?}");
        }
        other => panic!("expected NoCommonAlgo, got {other:?}"),
    }
}
```

Password and keyboard-interactive *success* cannot be tested here: a scratch sshd has no PAM (fact 2). What runs is the call sequence up to the server's refusal and the list of methods the refusal carries; the success paths are "verify on a real host" in the report.

- [ ] **Step 2: Run them to see them fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test auth_methods
```

Expected: FAIL to compile: `error[E0432]: unresolved import `russh_spike::auth``. (`host_key.rs` needs nothing new.)

- [ ] **Step 3: Write the login helpers**

`spike/russh/src/auth.rs`:

```rust
//! Password and keyboard-interactive login helpers.
//!
//! A scratch sshd cannot say yes to either (no PAM, so no password check for the user running the tests): only the refusal paths run
//! in `tests/auth_methods.rs`. The success paths of these helpers must be verified on a real host; the spike report says so.

use russh::client::{AuthResult, Handle, KeyboardInteractiveAuthResponse, Prompt};

use crate::client::SpikeHandler;

pub async fn login_password(handle: &mut Handle<SpikeHandler>, user: &str, password: &str) -> Result<AuthResult, russh::Error> {
    handle.authenticate_password(user, password).await
}

/// Keyboard-interactive: `answer(name, instructions, prompts)` returns one string per prompt, for as many rounds as the server asks.
/// `Ok(true)` when the server accepted, `Ok(false)` when it refused.
pub async fn login_keyboard_interactive(
    handle: &mut Handle<SpikeHandler>,
    user: &str,
    mut answer: impl FnMut(&str, &str, &[Prompt]) -> Vec<String>,
) -> Result<bool, russh::Error> {
    let mut response = handle.authenticate_keyboard_interactive_start(user, None::<String>).await?;
    loop {
        match response {
            KeyboardInteractiveAuthResponse::Success => return Ok(true),
            KeyboardInteractiveAuthResponse::Failure { .. } => return Ok(false),
            KeyboardInteractiveAuthResponse::InfoRequest { name, instructions, prompts } => {
                let answers = answer(&name, &instructions, &prompts);
                response = handle.authenticate_keyboard_interactive_respond(answers).await?;
            }
        }
    }
}
```

Append to `spike/russh/src/lib.rs`:

```rust
pub mod auth;
```

- [ ] **Step 4: Run them (green), then the observation**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo test --offline --no-default-features --test auth_methods --test host_key -- --nocapture
cargo test --offline --no-default-features --test host_key -- --ignored --nocapture
```

Expected: `auth_methods`: `3 passed` with `FACT auth.none.remaining_methods = MethodSet([PublicKey])` (the exact `Debug` text may differ); `host_key`: `5 passed; 1 ignored`, with `FACT host_key.reject.error = UnknownKey` (if the variant differs, that is the finding: update the assertions and the report) and `FACT host_key.no_common_algorithm.error = NoCommonAlgo { kind: …, ours: […], theirs: ["ssh-ed25519"] }`. The ignored test (an answer that arrives after sshd's `LoginGraceTime`, set to 3 s, with the answer at 6 s) prints `FACT host_key.slow_answer.outcome = …`: how the connect fails when the user takes longer than the server allows to read the host key dialog.

- [ ] **Step 5: Commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/src/auth.rs spike/russh/src/lib.rs spike/russh/tests/auth_methods.rs spike/russh/tests/host_key.rs
git commit -m "test(spike): login methods and host key decisions against a scratch sshd"
```

#### 3d: a link that misbehaves — keepalive, a cut connection, a stalled handshake

- [ ] **Step 1: Write the tests**

`spike/russh/tests/keepalive.rs`:

```rust
//! Task 3d: a link that misbehaves (the proxy in src/proxy.rs): keepalive, a connection dropped mid-command, a handshake that never answers.

use std::time::{Duration, Instant};

use russh::client::Config;
use russh_spike::client::{connect, drain, exec, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::proxy::{Mode, Proxy};
use russh_spike::{fact, within};

fn keepalive_config(interval_secs: u64, max: usize) -> Config {
    Config { keepalive_interval: Some(Duration::from_secs(interval_secs)), keepalive_max: max, ..Config::default() }
}

/// russh's `keepalive_interval` is "nothing received for this long", and `keepalive_max` is how many unanswered probes it tolerates. The spec's
/// `keepalive_secs` / `keepalive_max_missed` map onto them; this measures how long a silent link takes to be noticed.
#[tokio::test(flavor = "multi_thread")]
async fn keepalive_closes_a_silent_link_and_reports_keepalive_timeout() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let proxy = Proxy::start(server.sshd.port).await;
    let connection = server.connect_and_login(proxy.port, keepalive_config(1, 2), HostKeyVerdict::AcceptAny).await;

    proxy.set(Mode::Blackhole);
    let silenced = Instant::now();
    let reason = connection.observed.wait_for_disconnect(30).await;
    let noticed_after = silenced.elapsed();
    fact("keepalive.interval_1s_max_2.noticed_after_ms", noticed_after.as_millis());
    fact("keepalive.interval_1s_max_2.reason", &reason);
    assert!(reason.contains("KeepaliveTimeout"), "the reason was {reason}");
    assert!(noticed_after >= Duration::from_secs(2), "noticed after only {noticed_after:?}");
    assert!(noticed_after <= Duration::from_secs(10), "noticed after {noticed_after:?}");

    // After that the handle is dead: an exec fails promptly instead of hanging.
    let result = within(10, "exec on the dead connection", exec(&connection.handle, "echo never")).await;
    assert!(result.is_err(), "exec on a connection that russh closed returned {result:?}");
}

#[tokio::test(flavor = "multi_thread")]
async fn a_link_that_answers_keeps_the_session_alive_across_several_intervals() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let proxy = Proxy::start(server.sshd.port).await;
    let connection = server.connect_and_login(proxy.port, keepalive_config(1, 2), HostKeyVerdict::AcceptAny).await;

    tokio::time::sleep(Duration::from_secs(6)).await;
    assert!(connection.observed.disconnect.lock().unwrap().is_none(), "the session was closed while the peer was answering");
    assert_eq!(within(20, "exec", exec(&connection.handle, "echo still-here")).await.unwrap().text(), "still-here\n");
}

/// Review Focus 3, second half. The link dies while a command runs: the exec must END, with no exit status, instead of waiting for ever
/// or inventing an exit code.
#[tokio::test(flavor = "multi_thread")]
async fn a_connection_cut_during_an_exec_ends_it_without_an_exit_status() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let proxy = Proxy::start(server.sshd.port).await;
    let connection = server.connect_and_login(proxy.port, Config::default(), HostKeyVerdict::AcceptAny).await;

    let mut channel = connection.handle.channel_open_session().await.unwrap();
    channel.exec(true, "sleep 30").await.unwrap();
    tokio::time::sleep(Duration::from_millis(500)).await;

    proxy.set(Mode::Cut);
    let cut = Instant::now();
    let out = within(10, "the exec to end after the cut", drain(&mut channel)).await;
    fact("cut_during_exec.ended_after_ms", cut.elapsed().as_millis());
    assert_eq!(out.exit_status, None);
    let reason = connection.observed.wait_for_disconnect(10).await;
    fact("cut_during_exec.reason", &reason);
}

/// russh's `Config` has no connect or handshake timeout, and `client::connect` waits for the server's banner for as long as it takes.
/// The engine must wrap the whole connect in `tokio::time::timeout`.
#[tokio::test(flavor = "multi_thread")]
async fn a_handshake_that_never_gets_an_answer_needs_our_own_timeout() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let proxy = Proxy::start(server.sshd.port).await;
    proxy.set(Mode::Blackhole);

    let started = Instant::now();
    let outcome = tokio::time::timeout(Duration::from_secs(2), connect(proxy.port, Config::default(), HostKeyVerdict::AcceptAny)).await;
    fact("handshake_stall.waited_ms", started.elapsed().as_millis());
    assert!(outcome.is_err(), "connect finished against a link that delivers nothing");
}
```

- [ ] **Step 2: Run them to see them fail**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test keepalive
```

Expected: FAIL to compile: `error[E0432]: unresolved import `russh_spike::proxy``.

- [ ] **Step 3: Write the proxy (the file ends with its own unit tests, which need no russh)**

`spike/russh/src/proxy.rs`:

```rust
//! A TCP proxy in front of a scratch sshd that can go silent (bytes held, sockets open: a dead Wi-Fi link) or cut the connections.
//! Keepalive, a connection dropped mid-command and a handshake that never answers all need a link that misbehaves on demand.

use std::net::Ipv4Addr;

use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::tcp::{OwnedReadHalf, OwnedWriteHalf};
use tokio::net::{TcpListener, TcpStream};
use tokio::sync::watch;
use tokio::task::JoinHandle;

#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Mode {
    /// Bytes flow both ways.
    Forward,
    /// Bytes are read and held, nothing is delivered, the sockets stay open.
    Blackhole,
    /// Every connection is closed now, and new ones are closed on arrival.
    Cut,
}

pub struct Proxy {
    /// Where clients connect (127.0.0.1).
    pub port: u16,
    mode: watch::Sender<Mode>,
    accept_loop: JoinHandle<()>,
}

impl Proxy {
    pub async fn start(target_port: u16) -> Proxy {
        let listener = TcpListener::bind((Ipv4Addr::LOCALHOST, 0)).await.expect("bind the proxy");
        let port = listener.local_addr().expect("proxy address").port();
        let (mode, watcher) = watch::channel(Mode::Forward);
        let accept_loop = tokio::spawn(async move {
            while let Ok((client, _)) = listener.accept().await {
                let watcher = watcher.clone();
                tokio::spawn(async move {
                    if *watcher.borrow() == Mode::Cut {
                        return;
                    }
                    let Ok(server) = TcpStream::connect((Ipv4Addr::LOCALHOST, target_port)).await else { return };
                    let (client_read, client_write) = client.into_split();
                    let (server_read, server_write) = server.into_split();
                    let up = tokio::spawn(pipe(client_read, server_write, watcher.clone()));
                    let down = tokio::spawn(pipe(server_read, client_write, watcher));
                    let _ = tokio::join!(up, down);
                });
            }
        });
        Proxy { port, mode, accept_loop }
    }

    pub fn set(&self, mode: Mode) {
        self.mode.send_replace(mode);
    }
}

impl Drop for Proxy {
    fn drop(&mut self) {
        self.accept_loop.abort();
    }
}

async fn pipe(mut from: OwnedReadHalf, mut to: OwnedWriteHalf, mut mode: watch::Receiver<Mode>) {
    let mut buffer = vec![0u8; 16 * 1024];
    loop {
        let read = tokio::select! {
            read = from.read(&mut buffer) => read,
            _ = mode.wait_for(|mode| *mode == Mode::Cut) => break,
        };
        let count = match read {
            Ok(0) | Err(_) => break,
            Ok(count) => count,
        };
        // Black hole: hold what was read until the link comes back (or is cut).
        loop {
            let current = *mode.borrow();
            match current {
                Mode::Forward => break,
                Mode::Cut => return,
                Mode::Blackhole => {}
            }
            if mode.changed().await.is_err() {
                return;
            }
        }
        if to.write_all(&buffer[..count]).await.is_err() {
            break;
        }
    }
    let _ = to.shutdown().await;
}

#[cfg(test)]
mod tests {
    use std::time::Duration;

    use tokio::io::{AsyncReadExt, AsyncWriteExt};
    use tokio::net::{TcpListener, TcpStream};

    use super::{Mode, Proxy};

    /// A server that sends back every byte it receives. Returns its port.
    async fn echo_server() -> u16 {
        let listener = TcpListener::bind(("127.0.0.1", 0)).await.unwrap();
        let port = listener.local_addr().unwrap().port();
        tokio::spawn(async move {
            while let Ok((mut socket, _)) = listener.accept().await {
                tokio::spawn(async move {
                    let mut buffer = [0u8; 1024];
                    while let Ok(count) = socket.read(&mut buffer).await {
                        if count == 0 || socket.write_all(&buffer[..count]).await.is_err() {
                            break;
                        }
                    }
                });
            }
        });
        port
    }

    async fn round_trip(client: &mut TcpStream, bytes: &[u8]) -> Vec<u8> {
        client.write_all(bytes).await.unwrap();
        let mut reply = vec![0u8; bytes.len()];
        client.read_exact(&mut reply).await.unwrap();
        reply
    }

    #[tokio::test]
    async fn forward_passes_bytes_both_ways() {
        let proxy = Proxy::start(echo_server().await).await;
        let mut client = TcpStream::connect(("127.0.0.1", proxy.port)).await.unwrap();
        assert_eq!(round_trip(&mut client, b"hello").await, b"hello");
    }

    #[tokio::test]
    async fn blackhole_holds_the_bytes_until_forward_releases_them() {
        let proxy = Proxy::start(echo_server().await).await;
        let mut client = TcpStream::connect(("127.0.0.1", proxy.port)).await.unwrap();
        assert_eq!(round_trip(&mut client, b"warm").await, b"warm");

        proxy.set(Mode::Blackhole);
        client.write_all(b"held").await.unwrap();
        let mut reply = [0u8; 4];
        let silent = tokio::time::timeout(Duration::from_millis(500), client.read_exact(&mut reply)).await;
        assert!(silent.is_err(), "nothing may arrive through a black hole");

        proxy.set(Mode::Forward);
        tokio::time::timeout(Duration::from_secs(3), client.read_exact(&mut reply)).await.expect("released in time").unwrap();
        assert_eq!(&reply, b"held");
    }

    #[tokio::test]
    async fn cut_closes_open_connections_and_new_ones() {
        let proxy = Proxy::start(echo_server().await).await;
        let mut client = TcpStream::connect(("127.0.0.1", proxy.port)).await.unwrap();
        assert_eq!(round_trip(&mut client, b"warm").await, b"warm");

        proxy.set(Mode::Cut);
        let mut byte = [0u8; 1];
        let read = tokio::time::timeout(Duration::from_secs(3), client.read(&mut byte)).await.expect("closed in time");
        assert!(matches!(read, Ok(0) | Err(_)), "an open connection must end after a cut, got {read:?}");

        let mut late = TcpStream::connect(("127.0.0.1", proxy.port)).await.unwrap();
        let read = tokio::time::timeout(Duration::from_secs(3), late.read(&mut byte)).await.expect("closed in time");
        assert!(matches!(read, Ok(0) | Err(_)), "a new connection must be closed on arrival, got {read:?}");
    }
}
```

Append to `spike/russh/src/lib.rs`:

```rust
pub mod proxy;
```

- [ ] **Step 4: Run the proxy's tests, then the keepalive tests (green)**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo test --offline --no-default-features --lib proxy
cargo test --offline --no-default-features --test keepalive -- --nocapture
```

Expected: `3 passed` (forward, black hole, cut — these use no russh and were run for real when this plan was written) and then `4 passed` with `FACT keepalive.interval_1s_max_2.noticed_after_ms = …` (between 2000 and 10000; the design's `keepalive_secs`/`keepalive_max_missed` map onto russh's `keepalive_interval` ("nothing received for this long") and `keepalive_max`), `FACT keepalive.interval_1s_max_2.reason = Error(KeepaliveTimeout)`, `FACT cut_during_exec.ended_after_ms = …`, `FACT handshake_stall.waited_ms` (about 2000).

- [ ] **Step 5: Commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/src/proxy.rs spike/russh/src/lib.rs spike/russh/tests/keepalive.rs
git commit -m "test(spike): keepalive, a cut connection and a stalled handshake through a pausable proxy"
```

#### 3e: a two-hop jump and "every channel has a reader"

- [ ] **Step 1: Write the tests**

`spike/russh/tests/jump.rs`:

```rust
//! Task 3e, part 1: a two-hop jump the way the engine will build it: sshd A -> direct-tcpip -> `into_stream` -> `connect_stream` -> sshd B.
//! (OpenSSH's own `ssh -J` refuses to jump through the host it is going to, so there are two scratch servers.)

use russh::client::Config;
use russh::Disconnect;
use russh_spike::client::{connect_over, exec, Connection, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::{fact, within};

/// Opens a direct-tcpip channel from `first` to `second`'s port and runs a whole SSH handshake over it.
async fn handshake_through(first: &Connection, second: &Server, verdict: HostKeyVerdict) -> Result<Connection, russh::Error> {
    let channel = first.handle.channel_open_direct_tcpip("127.0.0.1", u32::from(second.sshd.port), "127.0.0.1", 0).await?;
    connect_over(channel.into_stream(), Config::default(), verdict).await
}

#[tokio::test(flavor = "multi_thread")]
async fn a_two_hop_jump_runs_a_command_on_the_second_host() {
    let Some(tools) = tools() else { return };
    let (a, b) = (Server::start(&tools, &[]), Server::start(&tools, &[]));
    let first = a.session().await;

    let mut second = within(30, "the second handshake", handshake_through(&first, &b, HostKeyVerdict::AcceptAny)).await.expect("jump");
    b.login(&mut second).await;
    let out = within(20, "exec on B", exec(&second.handle, "echo hop-$((20+22))")).await.unwrap();
    assert_eq!(out.text(), "hop-42\n");

    // The second handshake really was with B: each hop checks its own host key.
    assert_eq!(*second.observed.host_keys.lock().unwrap(), vec![b.host_fingerprint()]);
    assert_ne!(b.host_fingerprint(), a.host_fingerprint());
    assert_eq!(*first.observed.host_keys.lock().unwrap(), vec![a.host_fingerprint()]);
}

#[tokio::test(flavor = "multi_thread")]
async fn the_second_hop_checks_its_own_host_key_and_the_first_hop_survives_a_refusal() {
    let Some(tools) = tools() else { return };
    let (a, b) = (Server::start(&tools, &[]), Server::start(&tools, &[]));
    let first = a.session().await;

    let error = within(30, "the second handshake", handshake_through(&first, &b, HostKeyVerdict::RejectAll)).await.err().expect("refused");
    assert!(matches!(error, russh::Error::UnknownKey), "got {error:?}");
    assert_eq!(within(20, "exec on A", exec(&first.handle, "echo a-still-works")).await.unwrap().text(), "a-still-works\n");
}

/// The jump target is a port nothing listens on: sshd A's refusal comes back as a channel-open failure, quickly, and A stays usable.
#[tokio::test(flavor = "multi_thread")]
async fn a_jump_to_a_closed_port_fails_cleanly() {
    let Some(tools) = tools() else { return };
    let (a, b) = (Server::start(&tools, &[]), Server::start(&tools, &[]));
    let first = a.session().await;
    let closed_port = b.sshd.port;
    drop(b);
    tokio::time::sleep(std::time::Duration::from_millis(300)).await;

    let started = std::time::Instant::now();
    let result = first.handle.channel_open_direct_tcpip("127.0.0.1", u32::from(closed_port), "127.0.0.1", 0).await;
    let error = result.err().expect("nothing listens there");
    fact("jump.closed_port.error", format!("{error:?}"));
    fact("jump.closed_port.after_ms", started.elapsed().as_millis());
    assert!(matches!(error, russh::Error::ChannelOpenFailure(_)), "got {error:?}");
    assert_eq!(within(20, "exec on A", exec(&first.handle, "echo a-still-works")).await.unwrap().text(), "a-still-works\n");
}

/// Hop lifetimes: when the outer connection goes away, the inner one (carried inside it) must end too, or the broker leaks sessions.
#[tokio::test(flavor = "multi_thread")]
async fn closing_the_first_hop_ends_the_second() {
    let Some(tools) = tools() else { return };
    let (a, b) = (Server::start(&tools, &[]), Server::start(&tools, &[]));
    let first = a.session().await;
    let mut second = within(30, "the second handshake", handshake_through(&first, &b, HostKeyVerdict::AcceptAny)).await.expect("jump");
    b.login(&mut second).await;

    first.handle.disconnect(Disconnect::ByApplication, "", "en").await.unwrap();
    let reason = second.observed.wait_for_disconnect(10).await;
    fact("jump.close_first_hop.second_hop_reason", &reason);
}
```

`spike/russh/tests/channels.rs`:

```rust
//! Task 3e, part 2: "every channel has a reader". The spec (§6.3) says russh stalls the whole connection behind a channel nobody reads, and that
//! the maintainer declined to change it. The first test is the discipline that works; the second measures what really happens without it.

use std::time::Duration;

use russh::ChannelMsg;
use russh_spike::client::exec;
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::{fact, within};

/// A second channel works while the first one prints without end, as long as something reads the first.
#[tokio::test(flavor = "multi_thread")]
async fn a_second_channel_works_while_the_first_is_drained_in_the_background() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let mut busy = connection.handle.channel_open_session().await.unwrap();
    busy.exec(true, "yes spike").await.unwrap();
    let reader = tokio::spawn(async move {
        let mut bytes = 0usize;
        while let Some(message) = busy.wait().await {
            if let ChannelMsg::Data { data } = message {
                bytes += data.len();
            }
        }
        bytes
    });

    let out = within(20, "exec next to a busy channel", exec(&connection.handle, "echo alive")).await.unwrap();
    assert_eq!(out.text(), "alive\n");
    reader.abort();
}

/// A fact, not a requirement: the same, but nobody reads the busy channel. Does the neighbour still get its answer within 8 seconds,
/// and what happens to the busy one once somebody starts reading?
#[tokio::test(flavor = "multi_thread")]
#[ignore = "an observation for the report: run with --ignored --nocapture"]
async fn observe_an_unread_channel_next_to_a_working_one() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    let connection = server.session().await;

    let mut unread = connection.handle.channel_open_session().await.unwrap();
    unread.exec(true, "yes spike").await.unwrap();
    // Give the server time to fill russh's window and queue for the unread channel.
    tokio::time::sleep(Duration::from_secs(2)).await;

    let neighbour = tokio::time::timeout(Duration::from_secs(8), exec(&connection.handle, "echo alive")).await;
    fact("channels.unread_neighbour.answered_within_8s", neighbour.is_ok());

    let mut resumed = 0usize;
    let _ = tokio::time::timeout(Duration::from_secs(3), async {
        while let Some(message) = unread.wait().await {
            if let ChannelMsg::Data { data } = message {
                resumed += data.len();
            }
        }
    })
    .await;
    fact("channels.unread_neighbour.bytes_read_once_a_reader_started", resumed);
}
```

The jump uses two scratch servers because OpenSSH's own `ssh -J` refuses to jump through the host it is going to ("jumphost loop"). Each hop checks its own host key, and the second handshake runs over `Channel::into_stream()` handed to `client::connect_stream`, the way spec §6.2 step 2 describes.

- [ ] **Step 2: Run them**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo test --offline --no-default-features --test jump --test channels -- --nocapture
cargo test --offline --no-default-features --test channels -- --ignored --nocapture
```

Expected: `jump`: `4 passed` (with `FACT jump.closed_port.error = ChannelOpenFailure(…)` and `FACT jump.close_first_hop.second_hop_reason = …`); `channels`: `1 passed; 1 ignored`; the ignored run prints `FACT channels.unread_neighbour.answered_within_8s = true|false`, the fact behind spec §6.3's "a channel nobody reads stalls the whole connection" (PR 730), and how many bytes arrive once a reader starts. No helper is new here, so no red; apply the Spike rule to anything that differs, in particular to `closing_the_first_hop_ends_the_second` (if the inner session survives the outer one, the broker has to close hops inside-out; record that).

- [ ] **Step 3: Run the whole suite once and commit**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline
cd /Users/ysya/project/sideproj/sshelter
git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock
git add spike/russh/tests/jump.rs spike/russh/tests/channels.rs
git commit -m "test(spike): a two-hop jump and the every-channel-has-a-reader discipline"
```

Expected: every test binary ends `ok`; `windows_exec` does not exist yet (Task 4); the `git diff --stat` prints nothing.

---

### Task 4: russh connects to Windows OpenSSH and runs a command

**Files:**
- Create: `spike/russh/tests/windows_exec.rs`
- Create: `.github/workflows/spike-windows.yml`

**Interfaces:**
- Consumes: `client::{connect, exec, login_with_key_file, HostKeyVerdict}`, `fixture::Server`, `harness::tools`, `fact`, `within`.
- Produces: a GitHub Actions job `spike windows / russh` that installs and starts the runner's OpenSSH server, authorizes a throwaway key and runs the test; the facts `windows.host_key` and `windows.echo_stdout`, which Task 6 copies from the job log into the report.

What shapes this task:
- The job builds the spike with `--no-default-features`, without the app library. `src-tauri/build.rs` embeds the Common Controls manifest only into artifacts of the `src-tauri` package; a test executable of another crate that linked the app library may not start on Windows (`STATUS_ENTRYPOINT_NOT_FOUND`, 0xc0000139), which would hide the answer this job exists for. The vault signs the same bytes on every platform, and `test-windows.yml` already runs `vault::` on Windows. So the Windows test authenticates with russh's own key loading, not the vault.
- Win32-OpenSSH's `sshd` must run as a service (it creates user tokens), so the Windows job cannot use the scratch sshd of Task 2; it sets up the runner's own server and the test reads where it is from `SPIKE_SSH_PORT`, `SPIKE_SSH_USER`, `SPIKE_SSH_KEY`. The same test body also runs against a scratch sshd on macOS/Linux, which is how this task is checked locally.
- Running the job needs the branch pushed. The push is the controller's or the user's decision, not this plan's. `workflow_dispatch` alone only appears in the Actions tab once the file is on the default branch, so the workflow also has a `push` trigger for `next/own-ssh`, limited to `spike/**` and its own file.
- Things that could not be checked offline, marked `VERIFY` in the file: that `Add-WindowsCapability` offers the server on `windows-latest` (the Chocolatey fallback is in the same step), the `icacls` lines, that the service creates its host keys on first start, and that the default shell is `cmd.exe` (the test commands work in `cmd.exe` and PowerShell alike). `actionlint` was run on the file.

- [ ] **Step 1: Write the test**

`spike/russh/tests/windows_exec.rs`:

```rust
//! Task 4: connect, log in and run a command, against the OpenSSH server described by the environment. `spike-windows.yml` sets
//! SPIKE_SSH_PORT, SPIKE_SSH_USER and SPIKE_SSH_KEY (a passphrase-less OpenSSH private key) for the Windows runner's own sshd. Nothing in the
//! flow is Windows-specific; the job is what makes it so. Without those variables the first test prints "skipped" and passes, and the second
//! test runs the same flow against a scratch sshd, so the flow is exercised on every machine that has OpenSSH.

use std::path::Path;

use russh::client::Config;
use russh_spike::client::{connect, exec, login_with_key_file, HostKeyVerdict};
use russh_spike::fixture::Server;
use russh_spike::harness::tools;
use russh_spike::{fact, within};

async fn connect_log_in_and_exec(port: u16, user: &str, key: &Path) {
    let mut connection = within(60, "connect", connect(port, Config::default(), HostKeyVerdict::AcceptAny)).await.expect("connect");
    fact("windows.host_key", connection.observed.host_keys.lock().unwrap().join(","));
    let result = within(60, "publickey login", login_with_key_file(&mut connection.handle, user, key)).await.expect("login call");
    assert!(result.success(), "the server refused the key");

    let hello = within(60, "echo", exec(&connection.handle, "echo spike-windows-ok")).await.expect("exec");
    fact("windows.echo_stdout", format!("{:?}", hello.text()));
    assert!(hello.text().contains("spike-windows-ok"), "stdout was {:?}", hello.text());
    assert_eq!(hello.exit_status, Some(0));

    let exit = within(60, "exit 3", exec(&connection.handle, "exit 3")).await.expect("exec");
    assert_eq!(exit.exit_status, Some(3), "the remote exit code must come back through the server too");
}

#[tokio::test(flavor = "multi_thread")]
async fn connect_log_in_and_exec_against_the_server_in_the_environment() {
    let (Ok(port), Ok(user), Ok(key)) = (std::env::var("SPIKE_SSH_PORT"), std::env::var("SPIKE_SSH_USER"), std::env::var("SPIKE_SSH_KEY")) else {
        eprintln!("skipped: SPIKE_SSH_PORT, SPIKE_SSH_USER and SPIKE_SSH_KEY are not all set");
        return;
    };
    connect_log_in_and_exec(port.parse().expect("SPIKE_SSH_PORT is a port number"), &user, Path::new(&key)).await;
}

/// The rehearsal: the same flow against a scratch sshd (skipped where there is no sshd, as on the Windows runner).
#[tokio::test(flavor = "multi_thread")]
async fn the_same_flow_against_a_scratch_sshd() {
    let Some(tools) = tools() else { return };
    let server = Server::start(&tools, &[]);
    connect_log_in_and_exec(server.sshd.port, &server.sshd.user, &server.key.private_path).await;
}
```

- [ ] **Step 2: Run it locally (the environment test skips, the rehearsal runs)**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --no-default-features --test windows_exec -- --nocapture
```

Expected: `test result: ok. 2 passed`; the log has `skipped: SPIKE_SSH_PORT, SPIKE_SSH_USER and SPIKE_SSH_KEY are not all set` and `FACT windows.echo_stdout = "spike-windows-ok\n"` from the rehearsal. If the rehearsal fails here, the flow itself is wrong, and no Windows run can say anything useful yet.

- [ ] **Step 3: Write the workflow**

`.github/workflows/spike-windows.yml`:

```yaml
# Phase 0 spike (docs/superpowers/plans/2026-10-10-own-ssh-phase0-spike.md, Task 4): does russh connect to Windows OpenSSH and run a command?
# It starts the runner's own OpenSSH server and runs spike/russh's `windows_exec` test against it.
#
# - The spike is built with --no-default-features, without the app library: src-tauri/build.rs embeds the Common Controls manifest only into
#   the artifacts of the src-tauri package, so a test executable of another crate that linked the app library may not start on Windows
#   (0xc0000139). The vault signs the same bytes on every platform, and test-windows.yml already runs vault:: on Windows.
# - `workflow_dispatch` only shows up in the Actions tab once this file is on the default branch, so the push trigger below is what lets the
#   spike branch run it before anything is merged.
name: spike windows

on:
  workflow_dispatch:
  push:
    branches: [next/own-ssh]
    paths: ["spike/**", ".github/workflows/spike-windows.yml"]

permissions:
  contents: read

jobs:
  russh:
    runs-on: windows-latest
    timeout-minutes: 45
    defaults:
      run:
        working-directory: spike/russh
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
      - uses: Swatinem/rust-cache@v2
        with:
          workspaces: "./spike/russh -> target"

      # VERIFY on the runner. The image ships the OpenSSH client; the server is a Feature on Demand. If that capability cannot be added,
      # the Chocolatey package (Win32-OpenSSH) is the fallback. The service creates its host keys the first time it starts.
      - name: Install and start the OpenSSH server
        shell: pwsh
        run: |
          $ErrorActionPreference = 'Stop'
          $capability = Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
          $capability | Format-Table Name, State
          if ($capability.State -ne 'Installed') {
            try {
              Add-WindowsCapability -Online -Name $capability.Name | Out-Null
            } catch {
              Write-Warning "Feature on Demand failed ($_); trying Chocolatey"
              choco install openssh -y --no-progress --params '"/SSHServerFeature"'
            }
          }
          Get-Service sshd | Format-Table Name, Status, StartType
          Set-Service -Name sshd -StartupType Manual
          Start-Service sshd
          Get-Service sshd | Format-Table Name, Status, StartType

      # The runner user is an Administrator, so Win32-OpenSSH reads C:\ProgramData\ssh\administrators_authorized_keys for it (the
      # "Match Group administrators" block of the default sshd_config) and refuses the file unless only SYSTEM and Administrators can write it.
      # The Windows client refuses a private key other users can read. VERIFY the icacls lines on the runner.
      - name: Authorize a throwaway key for the runner user
        shell: pwsh
        run: |
          $ErrorActionPreference = 'Stop'
          $openssh = Join-Path $env:SystemRoot 'System32\OpenSSH'
          $dir = Join-Path $env:RUNNER_TEMP 'spike-ssh'
          New-Item -ItemType Directory -Force -Path $dir | Out-Null
          $key = Join-Path $dir 'id_ed25519'
          & "$openssh\ssh-keygen.exe" -q -t ed25519 -N '' -C spike -f $key
          if ($LASTEXITCODE -ne 0) { throw 'ssh-keygen failed' }
          icacls.exe $key /inheritance:r /grant "$($env:USERNAME):F" | Out-Null
          $authorized = Join-Path $env:ProgramData 'ssh\administrators_authorized_keys'
          Get-Content "$key.pub" | Set-Content -Path $authorized -Encoding ascii
          icacls.exe $authorized /inheritance:r /grant 'Administrators:F' /grant 'SYSTEM:F' | Out-Null
          Restart-Service sshd
          Add-Content -Path $env:GITHUB_ENV -Encoding utf8 -Value 'SPIKE_SSH_PORT=22'
          Add-Content -Path $env:GITHUB_ENV -Encoding utf8 -Value "SPIKE_SSH_USER=$env:USERNAME"
          Add-Content -Path $env:GITHUB_ENV -Encoding utf8 -Value "SPIKE_SSH_KEY=$key"

      # Separates a server that is set up wrong from a russh that cannot talk to it.
      - name: The OpenSSH client logs in
        shell: pwsh
        run: |
          $openssh = Join-Path $env:SystemRoot 'System32\OpenSSH'
          & "$openssh\ssh.exe" -V
          & "$openssh\ssh.exe" -F NUL -p $env:SPIKE_SSH_PORT -i $env:SPIKE_SSH_KEY -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL "$env:SPIKE_SSH_USER@127.0.0.1" 'echo client-ok'
          if ($LASTEXITCODE -ne 0) { throw "the OpenSSH client could not log in (exit $LASTEXITCODE): the server setup is wrong, not russh" }

      - name: Connect and exec with russh
        env:
          RUST_BACKTRACE: "1"
        run: cargo test --locked --no-default-features --test windows_exec -- --nocapture

      - name: What the OpenSSH server logged
        if: failure()
        shell: pwsh
        run: Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 40 -ErrorAction SilentlyContinue | Format-List TimeCreated, Message
```

- [ ] **Step 4: Check the workflow file**

```bash
cd /Users/ysya/project/sideproj/sshelter
actionlint .github/workflows/spike-windows.yml && echo clean
```

Expected: `clean`. (`actionlint` is installed here; without it, `ruby -ryaml -e 'YAML.load_file(".github/workflows/spike-windows.yml")'` at least proves the YAML loads.) The PowerShell is not checked by `actionlint`; the `VERIFY` comments are the list of what the first real run decides.

- [ ] **Step 5: Commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
git add spike/russh/tests/windows_exec.rs .github/workflows/spike-windows.yml
git commit -m "ci(spike): run russh against the Windows runner's OpenSSH server"
```

- [ ] **Step 6: The Windows run — needs a push, so ask first**

Tell the controller: "The branch `next/own-ssh` must be pushed for `spike windows` to run (the push trigger fires on it; or `gh workflow run spike-windows.yml --ref next/own-ssh` once the file is on a branch GitHub can see). Pushing is your decision; nothing in this plan pushes." If the controller pushes, read the run:

```bash
RUN_ID=$(gh run list --workflow spike-windows.yml --branch next/own-ssh --limit 1 --json databaseId --jq '.[0].databaseId')
gh run view "$RUN_ID" --json conclusion,url
gh run view "$RUN_ID" --log | grep -E "FACT |client-ok|test result|error"
```

Expected on a good run: `client-ok` from the OpenSSH client step, `test result: ok. 2 passed` (the environment test ran; the rehearsal printed nothing and returned) and the two `FACT windows.*` lines. A failure in the "OpenSSH client logs in" step is the server setup (fix the PowerShell); a failure in the russh step with the client step green is the result this task exists for: copy the error text. Fixes to the PowerShell are fixes to this task: amend with a new commit and ask for another run. Task 6 needs the run's URL, conclusion and `FACT` lines, or the statement that it has not run.

---

### Task 5: The agent's local transport moves to `ipc/`

**Files:**
- Create: `src-tauri/src/ipc/mod.rs`
- Move: `src-tauri/src/agent/server.rs` → `src-tauri/src/ipc/server.rs`; `src-tauri/src/agent/pipe_windows.rs` → `src-tauri/src/ipc/pipe_windows.rs`; `src-tauri/src/agent/peer.rs` → `src-tauri/src/ipc/peer.rs` (`git mv`, so the history follows)
- Modify: `src-tauri/src/lib.rs` (`mod ipc;`), `src-tauri/src/agent/mod.rs`, `agent/broker.rs`, `agent/oneshot.rs`, `agent/openssh_tests.rs`, `agent/session.rs`, `src-tauri/src/ipc/server.rs`, `src-tauri/src/ipc/pipe_windows.rs`, `.github/workflows/test-windows.yml`
- Test: the moved tests, plus one new guard test in `ipc/mod.rs`

**Interfaces:**
- Consumes: nothing from Tasks 1–4 (this task is independent of the spike crate; it only shares the branch).
- Produces: `crate::ipc::{peer, server}` and, on Windows, `crate::ipc::pipe_windows`, with exactly the items they had: `server::{MAX_CONNECTIONS, Stream, Handler, Started, listen_unix, take_lock, dispatch, check_socket_path, peer, IDLE_TIMEOUT}`, `pipe_windows::{pipe_name, listen, accept, one_shot}`, `peer::{ProcInfo, Program, identify, process_chain, executable_base}`. New test helpers `ipc::server::testing::{PING, PONG, echoing, ping}` and `agent::session::testing::NoKeys`. After this task nothing under `src-tauri/src/ipc/` mentions `crate::agent`.

What shapes this task:
- **No behavior change.** `agent` now imports from `ipc`; the pipe name (`sshelter-agent-<hash>`), thread names and error texts stay as they were. The agent keeps working exactly as before; this only proves the split (spec §4, §13 phase 0).
- **The tests lose the agent protocol.** `server.rs`'s tests used the agent protocol as a handy request/response, and its `testing::{NoKeys, serving}` came from `agent::session`. The design says `ipc/` has no agent protocol and `agent/` is deleted in 1.0, so `ipc`'s tests use a one-byte ping/pong handler. Nothing is lost: the agent protocol over `listen_unix` stays covered by `agent::openssh_tests` (real `ssh-add` and `ssh-keygen`) and by `agent::oneshot`'s tests; `NoKeys` (the agent's "no keys" authority, which `oneshot` needs) moves to `agent::session::testing`. A guard test keeps `ipc/` from reaching back into `agent`.
- **Already run, in a scratch copy of `src-tauri` outside the repo, when this plan was written:** the steps below, in this order, with these results: `agent::` lists 167 tests before; after step 3 the guard fails and 36 tests pass (27 `peer` + 9 `server`); after step 4 `ipc:: agent::` runs 168 tests, all green (the 167 plus the guard), and the whole lib suite `1417 passed; 0 failed; 2 filtered out`; the Windows branch type-checks for `x86_64-pc-windows-msvc`, and a deliberate error under `cfg(windows)` is reported. The Windows-only code (`pipe_windows.rs` and the Windows parts of `oneshot.rs`) can only be run by `test-windows.yml`; a failure in its next run is a defect of this task.
- **Left for phase 1, on purpose:** `ipc/server.rs` still calls `sync::slot_files::ensure_keys_dir` and `ipc/pipe_windows.rs` `sync::slot_files_windows::current_user_token` (the file layer the design deletes, so `ipc` will need its own owner-only directory and token helpers); the pipe name, thread names and the "agent's socket path is too long" message still say "agent".
- Running the tests rewrites `src/bindings/*.ts` (the `ts-rs` export tests); the content is identical, so `git status` must show nothing under `src/bindings/`.

Pick the session scratchpad directory as `$SCRATCH`; the files saved there below are tools, not part of the repo.

- [ ] **Step 1: Record which tests exist before**

```bash
cd /Users/ysya/project/sideproj/sshelter/src-tauri
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- agent:: --list 2>/dev/null > "$SCRATCH/ipc-before.txt"
tail -1 "$SCRATCH/ipc-before.txt"
```

Expected: `167 tests, 0 benchmarks` at HEAD 8b78077 (27 under `agent::peer`, 9 under `agent::server`; if other commits changed the number, the file is still the baseline).

- [ ] **Step 2: Write the guard test and the module — fails to compile**

Create `src-tauri/src/ipc/mod.rs`:

```rust
//! 本機 IPC 的傳輸層(own-ssh spec §4):Unix socket 或 Windows named pipe 的伺服器(`server`、`pipe_windows`),以及連上來的程式是誰(`peer`)。
//! 這一層不認得任何協定:同一使用者的連線來了就交給呼叫端給的 `Handler`。第 0 期從 `agent/` 搬過來,`agent/` 暫時是唯一的使用者(1.0 會刪掉它),
//! 所以這裡的程式(連測試也是)不能引用 `agent`:下面的測試守著。
pub mod peer;
#[cfg(windows)]
pub mod pipe_windows;
pub mod server;

#[cfg(test)]
mod tests {
    /// 之後留下來的是 `ipc/`,不是 `agent/`:`ipc/` 的程式不能依賴 `agent`。要找的字串在執行時才組起來,免得這個檔案自己出現它。
    #[test]
    fn the_ipc_module_does_not_reach_into_the_agent() {
        let needle = ["crate", "agent"].join("::");
        for (file, source) in [
            ("mod.rs", include_str!("mod.rs")),
            ("peer.rs", include_str!("peer.rs")),
            ("pipe_windows.rs", include_str!("pipe_windows.rs")),
            ("server.rs", include_str!("server.rs")),
        ] {
            assert!(!source.contains(&needle), "ipc/{file} mentions {needle}");
        }
    }
}
```

In `src-tauri/src/lib.rs`, add `mod ipc;` between `mod fsutil;` and `mod keys;`:

```rust
mod fsutil;
mod ipc;
mod keys;
```

Run:

```bash
cd /Users/ysya/project/sideproj/sshelter/src-tauri
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- ipc::
```

Expected: FAIL to compile: `error[E0583]: file not found for module `peer``, the same for `server`, and `error: couldn't read `src/ipc/peer.rs`` (and `server.rs`, `pipe_windows.rs`) from the guard's `include_str!`.

- [ ] **Step 3: Move the files and update the agent side — the guard fails**

```bash
cd /Users/ysya/project/sideproj/sshelter
git mv src-tauri/src/agent/server.rs src-tauri/src/ipc/server.rs
git mv src-tauri/src/agent/pipe_windows.rs src-tauri/src/ipc/pipe_windows.rs
git mv src-tauri/src/agent/peer.rs src-tauri/src/ipc/peer.rs
```

Save this as `$SCRATCH/agent_side.py` and run it from the repository root (every edit must match exactly once, or it stops and names the edit):

```python
"""Task 5, step 4: the agent side of the move. Run from the repo root: python3 -I agent_side.py src-tauri
Each edit must match exactly once; the script stops at the first one that does not."""
import pathlib
import sys

root = pathlib.Path(sys.argv[1])


def edit(relative, old, new):
    path = root / relative
    text = path.read_text()
    found = text.count(old)
    assert found == 1, f"{relative}: expected 1 match, found {found} for:\n{old}"
    path.write_text(text.replace(old, new))


# agent/mod.rs: the three modules are gone from here; the agent imports them from ipc/.
edit("src/agent/mod.rs",
     "pub mod oneshot;\n#[cfg(test)]\nmod openssh_tests;\npub mod peer;\n#[cfg(windows)]\npub mod pipe_windows;\npub mod prompt;\npub mod protocol;\npub mod server;\npub mod session;",
     "pub mod oneshot;\n#[cfg(test)]\nmod openssh_tests;\npub mod prompt;\npub mod protocol;\npub mod session;")
edit("src/agent/mod.rs",
     "use crate::error::AppError;\nuse crate::state::AppState;",
     "use crate::error::AppError;\n#[cfg(windows)]\nuse crate::ipc::pipe_windows;\nuse crate::ipc::{peer, server};\nuse crate::state::AppState;")

# agent/broker.rs
edit("src/agent/broker.rs", "use crate::agent::peer::Program;\n", "")
edit("src/agent/broker.rs",
     "use crate::error::AppError;\nuse crate::sync::env::Keychain;",
     "use crate::error::AppError;\nuse crate::ipc::peer::Program;\nuse crate::sync::env::Keychain;")

# agent/oneshot.rs
edit("src/agent/oneshot.rs",
     "use crate::agent::server::Stream;\nuse crate::agent::{agent_dir, home_dir, peer, session, AppAgentHost};\nuse crate::error::AppError;\nuse crate::state::AppState;",
     "use crate::agent::{agent_dir, home_dir, session, AppAgentHost};\nuse crate::error::AppError;\nuse crate::ipc::peer;\nuse crate::ipc::server::Stream;\nuse crate::state::AppState;")
edit("src/agent/oneshot.rs",
     "if let Ok(pid) = crate::agent::server::peer(&stream) {\n                    let _ = stream.set_read_timeout(Some(crate::agent::server::IDLE_TIMEOUT));",
     "if let Ok(pid) = crate::ipc::server::peer(&stream) {\n                    let _ = stream.set_read_timeout(Some(crate::ipc::server::IDLE_TIMEOUT));")
edit("src/agent/oneshot.rs", "if crate::agent::server::peer(&stream).is_ok() {", "if crate::ipc::server::peer(&stream).is_ok() {")
edit("src/agent/oneshot.rs", "crate::agent::server::check_socket_path(&path)?;", "crate::ipc::server::check_socket_path(&path)?;")
edit("src/agent/oneshot.rs", "let pipe = crate::agent::pipe_windows::one_shot(&name)?;", "let pipe = crate::ipc::pipe_windows::one_shot(&name)?;")
edit("src/agent/oneshot.rs", "let accepted = crate::agent::pipe_windows::accept(&pipe);", "let accepted = crate::ipc::pipe_windows::accept(&pipe);")
edit("src/agent/oneshot.rs", "    use crate::agent::server::testing::NoKeys;\n", "    use crate::agent::session::testing::NoKeys;\n")

# agent/openssh_tests.rs
edit("src/agent/openssh_tests.rs",
     "use crate::agent::server::{Handler, Stream};\nuse crate::agent::{oneshot, peer, session};\nuse crate::error::AppError;",
     "use crate::agent::{oneshot, session};\nuse crate::error::AppError;\nuse crate::ipc::peer;\nuse crate::ipc::server::{Handler, Stream};")
edit("src/agent/openssh_tests.rs", "crate::agent::server::listen_unix(&agent_dir, handler).unwrap();", "crate::ipc::server::listen_unix(&agent_dir, handler).unwrap();")
edit("src/agent/openssh_tests.rs", "crate::agent::pipe_windows::listen(&agent_dir, &name, handler).unwrap();", "crate::ipc::pipe_windows::listen(&agent_dir, &name, handler).unwrap();")

# agent/session.rs: the "no keys" authority the agent's own tests use now lives with the agent (it was in server.rs's test helpers).
edit("src/agent/session.rs",
     "\n#[cfg(test)]\nmod tests {\n    use super::*;\n    use crate::agent::protocol::*;",
     "\n/// 不提供任何金鑰的 authority:只想測連線與端點、不在乎簽章的測試用(`oneshot`)。\n#[cfg(test)]\npub(crate) mod testing {\n    use ssh_key::public::KeyData;\n\n    use super::{SignAuthority, SignRequest};\n\n    pub struct NoKeys;\n\n    impl SignAuthority for NoKeys {\n        fn identities(&self) -> Vec<(KeyData, String)> {\n            Vec::new()\n        }\n        fn sign(&self, _request: &SignRequest) -> Option<Vec<u8>> {\n            None\n        }\n    }\n}\n\n#[cfg(test)]\nmod tests {\n    use super::*;\n    use crate::agent::protocol::*;")
print("agent side done")
```

```bash
cd /Users/ysya/project/sideproj/sshelter
python3 -I "$SCRATCH/agent_side.py" src-tauri
cd src-tauri
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- ipc::
```

Expected: it compiles; `test result: FAILED. 36 passed; 1 failed`, the failure being `ipc::tests::the_ipc_module_does_not_reach_into_the_agent` with `ipc/pipe_windows.rs mentions crate::agent` (the moved files still import the agent; the 27 `peer` and 9 `server` tests pass).

- [ ] **Step 4: Cut the moved files loose from the agent — green**

Save this as `$SCRATCH/moved_files.py` and run it the same way. `peer.rs` needs nothing; `pipe_windows.rs` and `server.rs` lose their imports of the agent and their tests use the ping/pong handler:

```python
"""Task 5, step 6: edit the three files after the move. Run from the repo root: python3 -I moved_files.py src-tauri
peer.rs needs nothing. pipe_windows.rs and server.rs lose their imports of the agent: their tests use a protocol-free handler now."""
import pathlib
import sys

root = pathlib.Path(sys.argv[1])


def edit(relative, old, new):
    path = root / relative
    text = path.read_text()
    found = text.count(old)
    assert found == 1, f"{relative}: expected 1 match, found {found} for:\n{old}"
    path.write_text(text.replace(old, new))


# ---- ipc/pipe_windows.rs (Windows only; the Windows job and the scratch type-check in step 8 cover it)
edit("src/ipc/pipe_windows.rs",
     "use crate::agent::server::{dispatch, take_lock, Handler, Started};\nuse crate::error::AppError;\nuse crate::sync::slot_files_windows::current_user_token;",
     "use crate::error::AppError;\nuse crate::ipc::server::{dispatch, take_lock, Handler, Started};\nuse crate::sync::slot_files_windows::current_user_token;")
edit("src/ipc/pipe_windows.rs",
     "    use super::*;\n    use crate::agent::protocol::{read_frame, write_frame, SSH_AGENTC_REQUEST_IDENTITIES, SSH_AGENT_IDENTITIES_ANSWER};\n    use crate::agent::server::testing::serving;\n    use std::sync::mpsc;",
     "    use super::*;\n    use crate::ipc::server::testing::{echoing, ping, PONG};\n    use std::sync::mpsc;")
edit("src/ipc/pipe_windows.rs",
     "assert_eq!(listen(&agent, &name, serving(tx.clone())).unwrap(), Started::Running);",
     "assert_eq!(listen(&agent, &name, echoing(tx.clone())).unwrap(), Started::Running);")
edit("src/ipc/pipe_windows.rs",
     "        write_frame(&mut client, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();\n        assert_eq!(read_frame(&mut client).unwrap().unwrap()[0], SSH_AGENT_IDENTITIES_ANSWER);",
     "        assert_eq!(ping(&mut client), PONG);")
edit("src/ipc/pipe_windows.rs",
     "assert_eq!(listen(&agent, &name, serving(tx)).unwrap(), Started::OtherInstance);",
     "assert_eq!(listen(&agent, &name, echoing(tx)).unwrap(), Started::OtherInstance);")
edit("src/ipc/pipe_windows.rs",
     "let err = listen(&dir.path().join(\"agent\"), &name, serving(tx)).unwrap_err().to_string();",
     "let err = listen(&dir.path().join(\"agent\"), &name, echoing(tx)).unwrap_err().to_string();")

# ---- ipc/server.rs: the test helpers no longer speak the agent protocol
edit("src/ipc/server.rs", '''/// 兩個平台的端點測試共用:不提供任何金鑰的 authority,以及把對方 PID 送出來再服務連線的 handler。
#[cfg(test)]
pub(crate) mod testing {
    use std::sync::mpsc::Sender;
    use std::sync::{Arc, Mutex};

    use ssh_key::public::KeyData;

    use super::Handler;
    use crate::agent::session::{serve, SignAuthority, SignRequest};

    pub struct NoKeys;

    impl SignAuthority for NoKeys {
        fn identities(&self) -> Vec<(KeyData, String)> {
            Vec::new()
        }
        fn sign(&self, _request: &SignRequest) -> Option<Vec<u8>> {
            None
        }
    }

    pub fn serving(pids: Sender<Option<u32>>) -> Handler {
        let pids = Mutex::new(pids);
        Arc::new(move |mut stream, pid| {
            let _ = pids.lock().unwrap().send(pid);
            let _ = serve(&mut stream, &NoKeys);
        })
    }
}
''', '''/// 兩個平台的端點測試共用:不認得任何協定的 handler。端點測試只看連線、權限與名額,不看協定的內容,所以對方連上來送一個位元組(`PING`),
/// handler 先把對方的 PID 送出來,再回一個位元組(`PONG`)。
#[cfg(test)]
pub(crate) mod testing {
    use std::io::{Read, Write};
    use std::sync::mpsc::Sender;
    use std::sync::{Arc, Mutex};

    use super::Handler;

    pub const PING: u8 = 0x70;
    pub const PONG: u8 = 0x71;

    pub fn echoing(pids: Sender<Option<u32>>) -> Handler {
        let pids = Mutex::new(pids);
        Arc::new(move |mut stream, pid| {
            let _ = pids.lock().unwrap().send(pid);
            let mut byte = [0u8; 1];
            if stream.read_exact(&mut byte).is_ok() && byte[0] == PING {
                let _ = stream.write_all(&[PONG]);
            }
        })
    }

    /// 在 `stream` 上送出 `PING`,回傳讀到的那個位元組(對方沒回應就 panic)。
    pub fn ping(stream: &mut (impl Read + Write)) -> u8 {
        stream.write_all(&[PING]).unwrap();
        let mut byte = [0u8; 1];
        stream.read_exact(&mut byte).unwrap();
        byte[0]
    }
}
''')
edit("src/ipc/server.rs", '''mod tests {
    use super::testing::serving;
    use super::*;
    use crate::agent::protocol::{read_frame, write_frame, SSH_AGENTC_REQUEST_IDENTITIES, SSH_AGENT_IDENTITIES_ANSWER};
    use std::os::unix::fs::PermissionsExt;''', '''mod tests {
    use super::testing::{echoing, ping, PONG};
    use super::*;
    use std::os::unix::fs::PermissionsExt;''')
edit("src/ipc/server.rs", '''    fn identities(sock: &Path) -> u8 {
        let mut stream = UnixStream::connect(sock).unwrap();
        write_frame(&mut stream, &[SSH_AGENTC_REQUEST_IDENTITIES]).unwrap();
        read_frame(&mut stream).unwrap().unwrap()[0]
    }
''', '''    /// 連上 `sock`,送 `PING`,回傳讀到的位元組(handler 在運作就是 `PONG`)。
    fn pong(sock: &Path) -> u8 {
        ping(&mut UnixStream::connect(sock).unwrap())
    }
''')
edit("src/ipc/server.rs",
     "assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::Running);\n        assert_eq!(identities(&agent.join(\"sock\")), SSH_AGENT_IDENTITIES_ANSWER);\n        assert_eq!(rx.recv_timeout",
     "assert_eq!(listen_unix(&agent, echoing(tx)).unwrap(), Started::Running);\n        assert_eq!(pong(&agent.join(\"sock\")), PONG);\n        assert_eq!(rx.recv_timeout")
edit("src/ipc/server.rs",
     "assert_eq!(listen_unix(&agent, serving(tx.clone())).unwrap(), Started::Running);\n        assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::OtherInstance);\n        assert_eq!(identities(&agent.join(\"sock\")), SSH_AGENT_IDENTITIES_ANSWER, \"the first one still answers\");",
     "assert_eq!(listen_unix(&agent, echoing(tx.clone())).unwrap(), Started::Running);\n        assert_eq!(listen_unix(&agent, echoing(tx)).unwrap(), Started::OtherInstance);\n        assert_eq!(pong(&agent.join(\"sock\")), PONG, \"the first one still answers\");")
edit("src/ipc/server.rs",
     "assert_eq!(listen_unix(&agent, serving(tx)).unwrap(), Started::Running);\n        assert_eq!(identities(&agent.join(\"sock\")), SSH_AGENT_IDENTITIES_ANSWER);\n    }",
     "assert_eq!(listen_unix(&agent, echoing(tx)).unwrap(), Started::Running);\n        assert_eq!(pong(&agent.join(\"sock\")), PONG);\n    }")
edit("src/ipc/server.rs",
     "let err = listen_unix(&agent, serving(tx)).unwrap_err().to_string();",
     "let err = listen_unix(&agent, echoing(tx)).unwrap_err().to_string();")
print("moved files done")
```

```bash
cd /Users/ysya/project/sideproj/sshelter
python3 -I "$SCRATCH/moved_files.py" src-tauri
cd src-tauri
PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH cargo test --offline --lib -- ipc:: agent:: --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain
```

Expected: `test result: ok. 168 passed; 0 failed; 0 ignored; 0 measured; 1251 filtered out`, including the five `agent::openssh_tests` (three of them run the real `ssh-add`/`ssh-keygen` against the agent through the moved `listen_unix`), and the one pre-existing warning (`function `set_host_enabled` is never used`) and no other.

- [ ] **Step 5: No test was lost; the whole suite is green**

```bash
cd /Users/ysya/project/sideproj/sshelter/src-tauri
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo test --offline --lib -- ipc:: agent:: --list 2>/dev/null > "$SCRATCH/ipc-after.txt"
norm() { grep ': test$' "$1" | sed -E 's/: test$//; s/^(agent|ipc):://' | sort; }
echo "lost:"; comm -23 <(norm "$SCRATCH/ipc-before.txt") <(norm "$SCRATCH/ipc-after.txt")
echo "new:"; comm -13 <(norm "$SCRATCH/ipc-before.txt") <(norm "$SCRATCH/ipc-after.txt")
cargo test --offline --lib -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain
```

Expected: `lost:` lists nothing; `new:` lists exactly `tests::the_ipc_module_does_not_reach_into_the_agent`; the suite ends `test result: ok. 1417 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out` (about 100 seconds; the two filtered-out are the skips).

- [ ] **Step 6: Type-check the Windows branch from macOS**

The whole app cannot be checked for Windows from here (`aws-lc-sys` needs a C toolchain for MSVC), so, as SP3 and the vault plan did, check the real files in a scratch crate that stubs what they borrow. Save these two files in `$SCRATCH/winck/` (`Cargo.toml`, and `src/lib.rs.in`; `@REPO@` is replaced with the repository root):

`$SCRATCH/winck/Cargo.toml`:

```toml
[package]
name = "winck"
version = "0.0.0"
edition = "2021"
publish = false

[workspace]

[dependencies]
sha2 = { version = "0.10", features = ["oid"] }
tempfile = "3"

[target.'cfg(windows)'.dependencies]
windows-sys = { version = "0.61", features = ["Win32_Foundation", "Win32_Security", "Win32_Security_Authorization", "Win32_Storage_FileSystem", "Win32_System_Diagnostics_ToolHelp", "Win32_System_IO", "Win32_System_Pipes", "Win32_System_SystemServices", "Win32_System_Threading"] }
```

`$SCRATCH/winck/src/lib.rs.in`:

```rust
//! Scratch crate: type-checks the real src-tauri/src/ipc/ files (and slot_files_windows.rs) for x86_64-pc-windows-msvc from macOS.
//! The crate-internal items they use are stubbed; @REPO@ is replaced with the repository root by sed.
#![allow(dead_code)]

pub mod error {
    #[derive(Debug)]
    pub enum AppError {
        Io(std::io::Error),
        Other(String),
    }
    impl From<std::io::Error> for AppError {
        fn from(error: std::io::Error) -> Self {
            AppError::Io(error)
        }
    }
    impl std::fmt::Display for AppError {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            match self {
                AppError::Io(error) => write!(f, "io error: {error}"),
                AppError::Other(message) => write!(f, "{message}"),
            }
        }
    }
}

pub mod process {
    pub fn without_spawns<T>(f: impl FnOnce() -> T) -> T {
        f()
    }
    pub fn spawn(command: &mut std::process::Command) -> std::io::Result<std::process::Child> {
        command.spawn()
    }
}

pub mod sync {
    pub mod slot_files {
        pub fn ensure_keys_dir(dir: &std::path::Path) -> Result<(), crate::error::AppError> {
            std::fs::create_dir_all(dir)?;
            Ok(())
        }
    }
    #[cfg(windows)]
    #[path = "@REPO@/src-tauri/src/sync/slot_files_windows.rs"]
    pub mod slot_files_windows;
}

pub mod ipc {
    #[path = "@REPO@/src-tauri/src/ipc/peer.rs"]
    pub mod peer;
    #[cfg(windows)]
    #[path = "@REPO@/src-tauri/src/ipc/pipe_windows.rs"]
    pub mod pipe_windows;
    #[path = "@REPO@/src-tauri/src/ipc/server.rs"]
    pub mod server;
}
```

```bash
cd "$SCRATCH/winck"
cp /Users/ysya/project/sideproj/sshelter/src-tauri/Cargo.lock Cargo.lock
sed "s#@REPO@#/Users/ysya/project/sideproj/sshelter#g" src/lib.rs.in > src/lib.rs
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
cargo check --offline --tests --target x86_64-pc-windows-msvc
```

Expected: `Finished`. Prove the check is real once: keep a copy of the file, append `fn deliberate_error() -> u32 { "not a number" }` to `src-tauri/src/ipc/pipe_windows.rs`, re-run (expect `error[E0308]: mismatched types`), then put the copy back (`cp "$SCRATCH/pw.keep" src-tauri/src/ipc/pipe_windows.rs`) and re-run to `Finished`. If the Windows target's standard library is missing, say so in the report instead of skipping this.

- [ ] **Step 7: Keep the Windows job running the moved tests**

`.github/workflows/test-windows.yml` selects tests by module path, and `agent::` no longer covers the named pipe and the program identification. In that file, replace

```yaml
      # slot_files_windows), the key vault (vault::) and the SSH agent (agent:: covers the named pipe, program identification and
      # the real ssh-add / ssh-keygen against the agent). The rest of the suite runs on macOS; parts of it use the system keychain.
```

with

```yaml
      # slot_files_windows), the key vault (vault::), the local IPC transport (ipc:: covers the named pipe and program identification)
      # and the SSH agent (agent:: covers the real ssh-add / ssh-keygen against the agent). The rest of the suite runs on macOS; parts of
      # it use the system keychain.
```

and replace

```yaml
        # Quoted: an unquoted `vault:: agent::` is read as a YAML mapping and the workflow would not load.
        run: "cargo test --lib -- sync::slot_rules sync::slot_files vault:: agent::"
```

with

```yaml
        # Quoted: an unquoted `vault:: ipc:: agent::` is read as a YAML mapping and the workflow would not load.
        run: "cargo test --lib -- sync::slot_rules sync::slot_files vault:: ipc:: agent::"
```

Check: `actionlint .github/workflows/test-windows.yml && echo clean` prints `clean`.

- [ ] **Step 8: Check what changed, then commit**

```bash
cd /Users/ysya/project/sideproj/sshelter
grep -rnE "agent::(server|peer|pipe_windows)" src-tauri/src || echo "no old paths left"
git status --short
git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock src/bindings
```

Expected: `no old paths left`; `git status --short` shows the three renames (`agent/peer.rs`, `agent/pipe_windows.rs`, `agent/server.rs` → `ipc/…`), the new `ipc/mod.rs`, and modifications to `lib.rs`, `agent/{broker,mod,oneshot,openssh_tests,session}.rs`, `ipc/{pipe_windows,server}.rs` and `.github/workflows/test-windows.yml` — nothing else; the last command prints nothing.

```bash
git add src-tauri/src/ipc src-tauri/src/agent/mod.rs src-tauri/src/agent/broker.rs src-tauri/src/agent/oneshot.rs src-tauri/src/agent/openssh_tests.rs src-tauri/src/agent/session.rs src-tauri/src/lib.rs .github/workflows/test-windows.yml
git commit -m "refactor(ipc): move the agent's local transport into ipc/"
```

---

### Task 6: The spike report

**Files:**
- Create: `docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md`

**Interfaces:**
- Consumes: the test output of Tasks 1–3 (`FACT` lines), `spike/russh/target/cargo-tree-*.txt` and `lock-delta.txt` from Task 1, Task 5's results, and the `spike windows` run of Task 4 (or the fact that it has not run).
- Produces: the report: a table of each question with its result and evidence, the list of design assumptions that turned out wrong with the proposed spec change, what was not verified, and a go/no-go line. It says in its first lines that the spike crate is throwaway and not product code, and that phase 1 re-implements `ssh/russh_engine.rs` behind the `SshEngine` trait.

The report is written in Traditional Chinese like the other documents in `docs/superpowers/specs/`. Its numbers are not typed by hand: a script fills the template from the test log (`PASS`/`FAIL` per test, `FACT` values, files), so the evidence in the table is the evidence of one run. What stays manual is judgement: which wrong-assumption rows survive, and the Windows section.

- [ ] **Step 1: One full run, kept as a log**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
mkdir -p target
cargo test --offline -- --include-ignored --nocapture 2>&1 | tee target/spike-run.log | grep -E "^(test result|error)|FAILED|panicked"
```

Expected: only `test result: ok.` lines (one per test binary: the library, then `auth_methods`, `channels`, `coexistence`, `exec`, `harness_smoke`, `host_key`, `jump`, `keepalive`, `shell`, `vault_signer`, `windows_exec`). `--include-ignored` also runs the three `observe_…` tests, whose job is to print facts. A `FAILED` here is a test nobody resolved under the Spike rule: go back to the task that owns it.

- [ ] **Step 2: Collect the facts that are not test output**

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
export PATH=$HOME/.rustup/toolchains/stable-aarch64-apple-darwin/bin:$PATH
# Keep Task 1's saved files; regenerate only if one is missing. `-i ssh-key` without a version exits 101 (two versions in the lock).
[ -s target/cargo-tree-ssh-key.txt ] || { cargo tree --offline -i ssh-key@0.6.7 && cargo tree --offline -i ssh-key@0.7.0-rc.11; } > target/cargo-tree-ssh-key.txt
[ -s target/cargo-tree-aws-lc-rs.txt ] || cargo tree --offline -i aws-lc-rs > target/cargo-tree-aws-lc-rs.txt
python3 -I "$SCRATCH/lock_delta.py" ../../src-tauri/Cargo.lock Cargo.lock > target/lock-delta.txt
grep '^FACT ' target/lock-delta.txt >> target/spike-run.log
cd ../../src-tauri
{
  echo "FACT ipc.tests_listed = $(cargo test --offline --lib -- ipc:: agent:: --list 2>/dev/null | tail -1)"
  echo "FACT ipc.lib_suite = $(cargo test --offline --lib -- --skip secrets::tests::round_trip_set_get_delete --skip askpass::tests::env_secret_takes_priority_over_keychain 2>&1 | grep '^test result')"
  echo "FACT app.cargo_files_changed = $(git diff --stat "$(git merge-base main HEAD)" HEAD -- Cargo.toml Cargo.lock | wc -l | tr -d ' ')"
} >> ../spike/russh/target/spike-run.log
tail -3 ../spike/russh/target/spike-run.log
```

Expected: `ipc.tests_listed = 168 tests, 0 benchmarks`; `ipc.lib_suite = test result: ok. 1417 passed; 0 failed; …` (this step re-runs the whole lib suite, about 100 seconds: a last regression check of everything the plan changed in the app); `app.cargo_files_changed = 0` (anything else means `src-tauri/Cargo.toml` or `Cargo.lock` changed on this branch: find out why before going on).

- [ ] **Step 3: Save the template and the filler script**

Save as `$SCRATCH/fill_report.py`:

````python
"""Task 6: fill the spike report from the spike's test log.
Usage: python3 -I fill_report.py LOG TEMPLATE OUT WINDOWS      (WINDOWS is pass, fail or notrun: the result of the `spike windows` job)
Directives in the template:
  {{test:NAME}}      PASS, FAIL, MISSING, or "not run (ignored)" for the test function NAME (the last component of its path)
  {{all:A,B,C}}      PASS when every test passed, else FAIL: followed by the names that did not
  {{fact:KEY}}       the value of the log line `FACT KEY = value`, or (not printed)
  {{file:PATH}}      the file's contents in a fenced block (PATH relative to the current directory)
  {{conditions}}     the go/no-go conditions as a table; {{verdict}} the line that follows from them
  {{tests}} {{facts}} every test with its outcome; every FACT line
  {{today}}          today's date
It stops at a directive it does not know, so a typo cannot reach the report."""
import datetime
import re
import sys

log_path, template_path, out_path, windows = sys.argv[1:5]
assert windows in ("pass", "fail", "notrun"), windows
log = open(log_path).read()

outcomes = {}
for name, outcome in re.findall(r"^test (\S+) \.\.\. (ok|FAILED|ignored)", log, re.M):
    outcomes.setdefault(name.split("::")[-1], []).append(outcome)
facts = dict(re.findall(r"^FACT (\S+) = (.*)$", log, re.M))

CONDITIONS = [
    ("兩代 `ssh-key` 並存、編譯、互相認得公鑰",
     ["both_ssh_key_versions_parse_the_same_public_keys_and_agree_on_the_fingerprints",
      "the_lock_file_holds_ssh_key_0_6_7_and_a_0_7_release_candidate",
      "the_app_library_is_linked_into_the_same_binary"]),
    ("保管庫簽的簽章被 sshd 接受(Ed25519、ECDSA P-256、RSA 3072),錯的金鑰被拒",
     ["an_ed25519_key_in_the_vault_logs_in", "an_ecdsa_p256_key_in_the_vault_logs_in",
      "an_rsa_3072_key_in_the_vault_logs_in_with_a_sha2_signature", "a_signature_from_another_key_is_refused"]),
    ("exec、PTY shell、主機金鑰、keepalive、兩層跳板的基本行為",
     ["exec_returns_stdout_stderr_and_the_exit_code", "a_pty_shell_prints_a_prompt_runs_a_command_and_reports_its_exit_status",
      "accepting_the_key_connects_and_the_handler_saw_the_servers_fingerprint", "rejecting_the_key_fails_the_connect_with_unknown_key",
      "keepalive_closes_a_silent_link_and_reports_keepalive_timeout", "a_two_hop_jump_runs_a_command_on_the_second_host"]),
]


def test(name):
    results = outcomes.get(name)
    if not results:
        return "MISSING"
    if len(results) > 1:
        return "AMBIGUOUS (the name occurs in more than one test binary)"
    return {"ok": "PASS", "FAILED": "FAIL", "ignored": "not run (ignored)"}[results[0]]


def all_of(names):
    bad = [name for name in names if test(name) != "PASS"]
    return "PASS" if not bad else "FAIL: " + ", ".join(f"`{name}` ({test(name)})" for name in bad)


def windows_text():
    return {"pass": "PASS(`spike windows` 通過)", "fail": "FAIL(`spike windows` 失敗,見「Windows」一節)", "notrun": "未跑(要先 push 分支,見「Windows」一節)"}[windows]


def conditions():
    rows = ["| 條件 | 狀態 |", "|---|---|"] + [f"| {label} | {all_of(names)} |" for label, names in CONDITIONS]
    return "\n".join(rows + [f"| Windows 實機連線與 exec | {windows_text()} |"])


def verdict():
    failed = [label for label, names in CONDITIONS if all_of(names) != "PASS"]
    if failed or windows == "fail":
        return "**判定:不可行(no-go)** —— 沒過的條件:" + ";".join(failed + (["Windows 實機連線與 exec"] if windows == "fail" else [])) + "。原因與可能的出路寫在下面的表裡。"
    if windows == "notrun":
        return "**判定:待定** —— 其餘條件都過了,Windows 還沒跑(要先 push 分支);結果出來之前不能開第 1 期。"
    return "**判定:可行(go)** —— 條件都過了。規格先照下面「規格裡不成立的假設」的表修改,再開第 1 期。"


def fill(match):
    kind, _, argument = match.group(1).partition(":")
    if kind == "test":
        return test(argument)
    if kind == "all":
        return all_of(argument.split(","))
    if kind == "fact":
        return facts.get(argument, "(not printed)")
    if kind == "file":
        return "\n```\n" + open(argument).read().rstrip() + "\n```\n"
    if kind == "conditions":
        return conditions()
    if kind == "verdict":
        return verdict()
    if kind == "tests":
        return "\n".join(["| test | outcome |", "|---|---|"] + [f"| `{name}` | {test(name)} |" for name in sorted(outcomes)])
    if kind == "facts":
        return "```\n" + "\n".join(f"{key} = {value}" for key, value in sorted(facts.items())) + "\n```"
    if kind == "today":
        return datetime.date.today().isoformat()
    sys.exit(f"unknown directive: {match.group(0)}")


text = re.sub(r"\{\{([^}]+)\}\}", fill, open(template_path).read())
open(out_path, "w").write(text)
print(f"wrote {out_path}: {len(outcomes)} tests, {len(facts)} facts")
````

Save as `$SCRATCH/report.tmpl.md` (the outer fence is four backticks because the template itself contains fences):

````markdown
# 自有 SSH 用戶端:第 0 期試驗報告

- 日期:{{today}}
- 對應:`docs/superpowers/specs/2026-10-10-own-ssh-client-design.md` 第 13 節第 0 期、第 16 節;計畫 `docs/superpowers/plans/2026-10-10-own-ssh-phase0-spike.md`
- 環境:macOS(OpenSSH 10.3p1、Apple silicon)、russh `=0.64.1`、app 的 `ssh-key` 0.6.7;Windows 是 GitHub Actions 的 `windows-latest`。
- **`spike/russh/` 是拋棄式的試驗程式,不是產品程式碼。** 第 1 期要在 `ssh/russh_engine.rs` 裡、`SshEngine` 介面後面重寫一遍;這裡的檔案一個也不會複製進 `src-tauri/`。唯一進產品的是第 21 項的 `ipc/` 搬移(行為不變)。
- 證據來源:在 `spike/russh/` 執行 `cargo test --offline -- --include-ignored --nocapture` 的輸出;下表的通過/失敗與 `FACT` 值都來自那一次。測試名稱可以直接在 `spike/russh/tests/` 與 `spike/russh/src/` 找到。

## 結論

{{conditions}}

{{verdict}}

## 每個問題的結果

| # | 問題 | 結果 | 證據 |
|---|---|---|---|
| 1 | app 的 `ssh-key` 0.6.7 與 russh 釘的 `ssh-key` 0.7.0-rc 能在同一個 build 並存嗎? | {{all:both_ssh_key_versions_parse_the_same_public_keys_and_agree_on_the_fingerprints,the_lock_file_holds_ssh_key_0_6_7_and_a_0_7_release_candidate,the_app_library_is_linked_into_the_same_binary}} | 附錄的 `cargo tree -i ssh-key`;`coexistence` 的三個測試 |
| 2 | 把 russh 加進 app,`src-tauri/Cargo.lock` 會多什麼? | 新增 92 個套件(34 個新 crate 加 58 個既有 crate 的第二版本;預覽版 3 個:pkcs1 0.8.0-rc.4、rsa 0.10.0-rc.18、ssh-key 0.7.0-rc.11;`zeroize` 1.8.2 → 1.9.1 是唯一被換掉的版本。Task 1 實測;`lock_delta.py` 的原始 `new_crates`/`prerelease_crates` 是 35/5,含試驗套件本身與 wasi build-metadata 的誤判,不要直接引用);app 既有的套件被換版本:{{fact:lock.app_versions_replaced}} 個;已有的套件多出第二個版本:{{fact:lock.second_versions_of_crates_the_app_has}} 個 | 附錄的 lock delta;`aws-lc-rs` 只有一個版本(附錄) |
| 3 | 保管庫的 `Material::sign` 能接成 russh 的 `Signer` 嗎(Ed25519、ECDSA P-256、RSA 3072)? | {{all:an_ed25519_key_in_the_vault_logs_in,an_ecdsa_p256_key_in_the_vault_logs_in,an_rsa_3072_key_in_the_vault_logs_in_with_a_sha2_signature}};RSA 在線上用的演算法:{{fact:signer.rsa.algorithm_on_the_wire}}(russh 給的 hash_alg:{{fact:signer.rsa.hash_alg_offered_by_russh}}) | `vault_signer`;簽章格式見計畫 Task 2 |
| 4 | sshd 真的在驗保管庫簽的東西嗎(負向對照) | {{test:a_signature_from_another_key_is_refused}} | `vault_signer` |
| 5 | 只講 `ssh-rsa`(SHA-1)的舊伺服器(Review Focus 1) | {{test:an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1}};russh 給的 hash_alg:{{fact:signer.sha1_only.hash_alg_offered_by_russh}};線上演算法:{{fact:signer.sha1_only.algorithm_on_the_wire}} | `vault_signer` |
| 6 | exec:結束碼、stdout、stderr | {{test:exec_returns_stdout_stderr_and_the_exit_code}} | `exec` |
| 7 | exec:被訊號殺掉(Review Focus 3) | {{test:a_command_killed_by_a_signal_reports_the_signal_and_no_exit_status}};exit_status = {{fact:exec.killed_by_signal.exit_status}},exit_signal = {{fact:exec.killed_by_signal.exit_signal}} | `exec` |
| 8 | exec:很長的輸出(Review Focus 5) | {{test:thirty_two_mebibytes_of_output_arrive_complete}}({{fact:exec.long_output.mib_per_second}} MiB/s);讀到上限就關 channel,連線仍可用:{{test:a_reader_that_stops_at_a_cap_and_closes_the_channel_leaves_the_connection_usable}}(下一個 exec {{fact:exec.capped_reader.next_exec_ms}} ms) | `exec` |
| 9 | exec 的逾時 | {{test:a_command_timeout_is_ours_to_enforce_and_the_connection_survives_it}}:russh 沒有指令逾時,引擎自己用 `tokio::time::timeout` 加 `Channel::close`,連線之後仍可用 | `exec` |
| 10 | shell:PTY、提示、指令、結束碼 | {{test:a_pty_shell_prints_a_prompt_runs_a_command_and_reports_its_exit_status}};第一個提示在 {{fact:shell.first_prompt_ms}} ms 內出現 | `shell` |
| 11 | shell:視窗大小(開 PTY 時給的、執行中改的、shell 就緒前改的:Review Focus 4) | {{all:the_size_given_with_the_pty_request_is_the_initial_size,a_window_change_while_the_shell_runs_is_applied,a_window_change_sent_before_the_shell_is_ready_is_not_lost}};連 PTY 都還沒有就送 window-change:最後大小 {{fact:shell.window_change_before_pty_request.final_size}},保留了嗎:{{fact:shell.window_change_before_pty_request.was_kept}} | `shell` |
| 12 | 密碼與 keyboard-interactive | 拒絕的路徑:{{all:authenticate_none_lists_the_methods_the_server_offers,a_password_is_refused_by_a_key_only_server_and_the_refusal_lists_what_remains,keyboard_interactive_is_refused_by_a_key_only_server}};`none` 認證列出的方法:{{fact:auth.none.remaining_methods}}。**成功的路徑沒有驗證:scratch sshd 沒有 PAM,要在真實主機上測** | `auth_methods` |
| 13 | 主機金鑰:接受、拒絕、釘住、比對 | {{all:accepting_the_key_connects_and_the_handler_saw_the_servers_fingerprint,rejecting_the_key_fails_the_connect_with_unknown_key,a_pinned_fingerprint_that_matches_connects_and_one_that_differs_is_refused}};被拒時 russh 回的錯誤:{{fact:host_key.reject.error}} | `host_key` |
| 14 | 主機金鑰確認要等人:等得了嗎 | {{all:the_answer_can_arrive_seconds_later_and_the_connection_goes_on,a_connect_that_returned_ok_on_a_dead_session_is_found_out_by_the_first_call}};答案比伺服器的 `LoginGraceTime` 慢:connect 回 {{fact:host_key.slow_answer.outcome}},緊接著的第一次呼叫 {{fact:host_key.slow_answer.first_call_after_connect}},200 ms 後 {{fact:host_key.slow_answer.call_200ms_later}},登入 {{fact:host_key.slow_answer.login_after}},handler 記到的斷線 {{fact:host_key.slow_answer.disconnect_seen}}(對照組、預設寬限時間:第一次呼叫 {{fact:host_key.slow_answer_control_default_grace.first_call_after_connect}},登入 {{fact:host_key.slow_answer_control_default_grace.login_after}});等答案時連線被切斷,第一次呼叫的面貌(51 回合):{{fact:host_key.cut_during_prompt.first_call.total}},之後 `is_closed()`:{{fact:host_key.cut_during_prompt.is_closed_after_the_first_call}},`Handle` future:{{fact:host_key.cut_during_prompt.handle_future}};sshd 真正掉線的時間(`LoginGraceTime 1`,秒):{{fact:host_key.grace_enforcement.login_grace_1.seconds_until_dropped}} | `host_key` |
| 15 | 沒有共同的主機金鑰演算法(Review Focus 2) | {{test:no_common_host_key_algorithm_names_both_lists}};錯誤:{{fact:host_key.no_common_algorithm.error}} | `host_key` |
| 16 | keepalive | {{test:keepalive_closes_a_silent_link_and_reports_keepalive_timeout}}:interval 1 秒、max 2,{{fact:keepalive.interval_1s_max_2.noticed_after_ms}} ms 後察覺,原因 {{fact:keepalive.interval_1s_max_2.reason}};有回應的連線不會被關:{{test:a_link_that_answers_keeps_the_session_alive_across_several_intervals}} | `keepalive` |
| 17 | 連線在指令執行中被切斷(Review Focus 3) | {{test:a_connection_cut_during_an_exec_ends_it_without_an_exit_status}}:{{fact:cut_during_exec.ended_after_ms}} ms 內結束,原因 {{fact:cut_during_exec.reason}} | `keepalive` |
| 18 | 握手一直沒有回應 | {{test:a_handshake_that_never_gets_an_answer_needs_our_own_timeout}}:russh 的 `Config` 沒有連線或握手逾時,`connect` 會一直等 | `keepalive` |
| 19 | 兩層跳板(direct-tcpip → `into_stream` → `connect_stream`) | {{all:a_two_hop_jump_runs_a_command_on_the_second_host,the_second_hop_checks_its_own_host_key_and_the_first_hop_survives_a_refusal,a_jump_to_a_closed_port_fails_cleanly,closing_the_first_hop_ends_the_second}};連到沒人聽的埠:{{fact:jump.closed_port.error}};關掉第一跳後,第二跳的 `Handle` future:{{fact:jump.close_first_hop.second_hop_end}}、`is_closed()`:{{fact:jump.close_first_hop.second_hop_is_closed}}、handler 的 `disconnected`:{{fact:jump.close_first_hop.second_hop_reason}}(russh 不會替內層跳板呼叫它;原因要看外層自己的紀錄:{{fact:jump.close_first_hop.first_hop_reason}}) | `jump` |
| 20 | 每個 channel 都有讀取工作 | {{test:a_second_channel_works_while_the_first_is_drained_in_the_background}};沒人讀的 channel 旁邊的 exec 在 8 秒內回應了嗎:{{fact:channels.unread_neighbour.answered_within_8s}}(之後開始讀,收到 {{fact:channels.unread_neighbour.bytes_read_once_a_reader_started}} bytes) | `channels` |
| 21 | `ipc/` 搬移(不改行為) | `ipc::` 與 `agent::` 共列 {{fact:ipc.tests_listed}};整個 lib 測試:{{fact:ipc.lib_suite}};Windows 分支型別檢查通過;`test-windows.yml` 的過濾加了 `ipc::` | 計畫 Task 5;`git log` 裡的 `refactor(ipc)` |
| 22 | 為了試驗動到 app 的哪裡 | 只有 `src-tauri/src/lib.rs` 的 `mod vault;` 改成 `pub mod vault;`(一行,沒有新的警告),以及第 21 項;`src-tauri/Cargo.toml` 與 `Cargo.lock` 對 main 的差異:{{fact:app.cargo_files_changed}} 行 | `git diff` |

## 規格裡不成立的假設

每一列都要用上面的證據核對一次:證據支持規格原本的說法,就把那一列刪掉(並在下一節寫一行「核對過、沒問題」);不支持就留著。標「文件」的列是讀 russh 0.64.1 在 docs.rs 的原始碼得到的,測試已經在 Task 2 到 Task 3 驗證它沒有被推翻。

| 規格位置 | 原本的說法 | 實測 | 建議的規格修改 |
|---|---|---|---|
| §12(bfd5b35 之前的版本) | 整合測試「沿用 `agent/openssh_tests.rs` 起 sshd 的模式」 | 那個檔案從來沒起過 sshd(它只用 `ssh-keygen`、`ssh-add`)。試驗自己建了一套:臨時目錄、空閒埠、測試用主機金鑰與 authorized_keys、一般使用者執行 `sshd -D -e -f`(`spike/russh/src/harness.rs`,`harness_smoke` 兩個測試證明它能用)。 | 規格已在 bfd5b35 改正。第 1 期把這套搬進 app 的測試支援程式碼。 |
| §15、§6.3(文件) | `Signer::auth_sign` 收待簽緩衝區、回傳附加 SSH 編碼簽章;`authenticate_publickey_with(user, PublicKey, hash_alg, &mut impl Signer)` | 大致對,細節有三處:`auth_sign(&mut self, key: &AgentIdentity, hash_alg: Option<HashAlg>, to_sign: Vec<u8>) -> Vec<u8>`(key 是 `AgentIdentity`,緩衝區是 `Vec<u8>`);`to_sign` 開頭是 `string(session id)`;回傳必須是 `to_sign` 原樣加上 `u32 長度 ‖ 簽章 blob`,`Material::sign` 回的正是那個 blob,RSA 用 `hash_alg` 選雜湊(`Some(Sha512)` = flag 4、`Some(Sha256)` = 2、`None` = 0 即 `ssh-rsa`)。三種金鑰的結果見第 3 項。 | §6.1 的 `AuthSource::sign` 一節寫上這個合約。 |
| §15(8b78077 加的一行) | 「russh 的預設不會帶進新的加密堆疊」 | TLS 後端確實沒有新增(`aws-lc-rs` 與 app 共用同一個版本);但 russh 帶進第二代 RustCrypto 預覽版(`ssh-key =0.7.0-rc.11`、`rsa =0.10.0-rc.18`、`p256`/`p384`/`p521` 0.14、`ed25519-dalek` 3、`curve25519-dalek` 5、`ecdsa` 0.17、`elliptic-curve` 0.14、`crypto-bigint` 0.7、`ml-kem` 0.3 等),`Cargo.lock` 新增 {{fact:lock.new_crates}} 個套件,其中 {{fact:lock.prerelease_crates}} 個是預覽版。 | 改成「不新增 TLS/加密後端,但新增 SSH 用的第二代 RustCrypto 預覽版;`cargo audit` 與升版要看兩代」。 |
| §6.3 | 「演算法用 russh 預設…第一版不提供舊演算法」 | russh 預設的主機金鑰清單最後一項是 `ssh-rsa`(SHA-1,`Preferred::DEFAULT.key` 的 `Rsa { hash: None }`)。只講 `ssh-rsa` 的舊伺服器連得上:{{test:an_rsa_key_in_the_vault_logs_in_to_a_server_that_only_speaks_ssh_rsa_sha1}}。其他舊東西(CBC、SHA-1 的 MAC、`diffie-hellman-group1`/`14-sha1`)確實沒有;沒有共同演算法時的錯誤是 {{fact:host_key.no_common_algorithm.error}}。 | 寫明「預設含 `ssh-rsa`(SHA-1)的主機金鑰簽章」,並決定要不要在 `Config::preferred` 把它拿掉;沒有共同演算法的錯誤訊息用 `NoCommonAlgo` 的兩個清單組出來。 |
| §6.2 第 5 步 | 每 `keepalive_secs` 一次,連續 `keepalive_max_missed` 次沒回應就斷線 | russh 的 `keepalive_interval` 是「這麼久沒有收到任何東西才送一次」,`keepalive_max` 是未回應的探測數;interval 1 秒、max 2 的連線在 {{fact:keepalive.interval_1s_max_2.noticed_after_ms}} ms 後才被判定斷線,原因是 `KeepaliveTimeout`。 | 設定名對上 russh 的語意,並把察覺斷線所需的時間寫給使用者看。 |
| §6.2 第 2 步 | 「連第一跳(TCP + 連線逾時)」 | russh 的 `Config` 沒有連線或握手逾時;握手沒有回應時 `connect` 一直等(第 18 項)。 | 引擎用 `tokio::time::timeout` 包住整個 `connect`(含握手),時間取 host 記錄的 `connect_timeout_secs`。 |
| §6.1 `Session::exec(command, timeout)` | `exec` 帶逾時 | russh 沒有指令逾時(第 9 項)。 | 在 §6.1 註明:逾時由引擎用 `tokio::time::timeout` 加 `Channel::close` 做。 |
| §6.2 第 3 步 | 主機金鑰:`Pinned` 繼續、`Unknown` 問人、`Mismatch` 警告 | `check_server_key` 只能回 true/false;「沒看過」和「不符」被拒時是同一個錯誤({{fact:host_key.reject.error}}),russh 不會告訴引擎為什麼。使用者看確認視窗的時間受伺服器 `LoginGraceTime`(預設 120 秒)限制,而且等答案期間沒有人在讀 socket(russh 在 handler 回傳前不 poll;client/mod.rs:2018):答案太慢時 connect 仍回 {{fact:host_key.slow_answer.outcome}},失敗到之後的呼叫才浮現(第一次呼叫 {{fact:host_key.slow_answer.first_call_after_connect}},200 ms 後 {{fact:host_key.slow_answer.call_200ms_later}}),面貌依呼叫而異(空的 `AuthResult::Failure`、`RecvError`、`Inconsistent`、`SendError`;`best_supported_rsa_hash` 甚至可能正常回答,所以 `login_with_key_file` 的第一步不能當存活探針),而且第一次呼叫得到正常的拒絕也不代表連線還活著(8 次裡 3 次)。 | 引擎自己記下拒絕的原因(決定回 `HostKeyMismatch` 還是使用者取消);連線是否還活著看 `is_closed()`、handler 記下的 `disconnected` 或 `Handle` future 是否已結束(伺服器正式送 SSH_MSG_DISCONNECT 時該 future 是 `Ok(())`,client/mod.rs:1314-1319,所以判準是「結束了」不是「Err」),不靠錯誤的 variant 判斷(否則斷線會被誤報成認證失敗);使用者答完、釘住之後若連線已死就重連,第二次不會再問(伺服器的寬限時間引擎無從得知,所以不是「限制確認視窗的時間」)。 |
| §7.1 `exec_result{exit_code, …}` | 結果有結束碼 | 指令被訊號殺掉時沒有結束碼(exit_status = {{fact:exec.killed_by_signal.exit_status}},exit_signal = {{fact:exec.killed_by_signal.exit_signal}});連線被切斷時也沒有(第 17 項)。 | `exit_code` 改成可空,加 `signal` 欄位;MCP 的 `ssh_exec` 要回「被 KILL 終止」或「連線中斷」,不能回 0。 |
| §7.3 視窗大小 | `SIGWINCH` → `window_change` 直接送 | PTY 建好之後送的有效(第 11 項);連 PTY 都還沒有時送的,最後大小是 {{fact:shell.window_change_before_pty_request.final_size}}(保留了嗎:{{fact:shell.window_change_before_pty_request.was_kept}})。 | broker 記住最後一次的大小,shell 就緒後再送一次。 |
| §6.3 | 「每個 channel 都有專屬的讀取工作(沒人讀的 channel 會卡住整條連線)」 | 沒人讀的 channel 旁邊的 exec 在 8 秒內回應了嗎:{{fact:channels.unread_neighbour.answered_within_8s}}。 | 照結果決定:`false` 就維持現在的紀律並在 §6.3 寫上實測;`true` 就把「會卡住」改成「在某個輸出量以上才會卡住」並找出那個量。 |

## 核對過、沒問題

(上表刪掉的列,各寫一行:規格哪一處、哪個測試證明它是對的。沒有刪就寫「無」。)

## 用到的 russh 0.64.1 介面

下面這些名稱都是從 docs.rs 讀來的,Task 2 到 Task 3 逐一編譯過。與 docs.rs 不同、編譯時改過的名稱:(寫在這一行;一個都沒改就寫「無」)

- 簽章:`russh::Signer::auth_sign(&mut self, &AgentIdentity, Option<HashAlg>, Vec<u8>) -> Vec<u8>`、`russh::keys::agent::AgentIdentity::public_key()`、`Handle::authenticate_publickey_with(user, PublicKey, Option<HashAlg>, &mut S)`、`Handle::best_supported_rsa_hash()`
- 連線:`client::{connect, connect_stream, Config { keepalive_interval, keepalive_max, preferred }, Handler::{check_server_key(&PublicKeyOrCertificate), disconnected(DisconnectReason)}}`、`russh::keys::{PublicKey, HashAlg, Algorithm, PrivateKeyWithHashAlg, PublicKeyOrCertificate, load_secret_key}`、`Preferred::DEFAULT`
- 認證:`Handle::{authenticate_publickey, authenticate_password, authenticate_none, authenticate_keyboard_interactive_start, authenticate_keyboard_interactive_respond}`、`client::{AuthResult, KeyboardInteractiveAuthResponse, Prompt}`、`MethodKind`
- channel:`Handle::{channel_open_session, channel_open_direct_tcpip, disconnect, is_closed}`、`Channel::{exec, request_pty, request_shell, window_change, data, wait, close, into_stream}`、`ChannelMsg::{Data, ExtendedData, ExitStatus, ExitSignal, Failure}`、`Disconnect::ByApplication`
- 錯誤:`russh::Error::{UnknownKey, KeepaliveTimeout, NoCommonAlgo, ChannelOpenFailure, Keys}`、`SendError`

## 沒驗證到的(要在真實環境補)

- 密碼與 keyboard-interactive 的**成功**路徑:scratch sshd 沒有 PAM。`auth::login_password` 與 `auth::login_keyboard_interactive` 只跑到伺服器拒絕。第 1 期的整合測試要在有 PAM 的主機或容器上補。
- Windows 上的 PTY/shell(ConPTY)與視窗大小;`sshelter connect` 在 Windows Terminal 與傳統 console 的行為(第 3 期)。
- Linux:這份試驗只在 macOS 跑過;RHEL/Fedora 的 crypto policy 會讓第 5 項失敗(用 `SPIKE_SKIP_SHA1=1` 略過)。
- 長時間連線(數小時)、rekey(`Limits` 預設 1 GiB 或 3600 秒)、大量並行 channel。
- `cargo audit` 對 russh 兩代 RustCrypto 的結果。

## 附錄

### `cargo tree -i ssh-key`
{{file:target/cargo-tree-ssh-key.txt}}

### `cargo tree -i aws-lc-rs`
{{file:target/cargo-tree-aws-lc-rs.txt}}

### Cargo.lock 差異(試驗的 lock 對 app 的 lock)
{{file:target/lock-delta.txt}}

### 所有測試
{{tests}}

### 所有 FACT
{{facts}}
````

- [ ] **Step 4: Generate the report**

The last argument is the result of the `spike windows` job from Task 4 step 6: `pass` (green), `fail` (red) or `notrun` (nobody pushed the branch yet). It decides the verdict line; the script never writes "go" while Windows is `notrun`.

```bash
cd /Users/ysya/project/sideproj/sshelter/spike/russh
python3 -I "$SCRATCH/fill_report.py" target/spike-run.log "$SCRATCH/report.tmpl.md" ../../docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md notrun
```

(Replace `notrun` with `pass` or `fail` when there is a run. Running this step again overwrites the report, so do step 5 again after it.) Expected: `wrote …: 46 tests, N facts`. Then `grep -n "not printed\|MISSING\|AMBIGUOUS\|FAIL" docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md` from the repository root should print nothing; a `(not printed)` is a fact whose test did not run or did not reach its `fact(...)` line, and a `MISSING` is a test that did not run: fix the cause and redo steps 1–4.

- [ ] **Step 5: The Windows section**

If the job has not run, append this text exactly:

```bash
cd /Users/ysya/project/sideproj/sshelter
cat >> docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md <<'TEXT'

## Windows

`spike windows` 還沒跑:要先把分支 `next/own-ssh` push 上去(push 由使用者決定)。結果出來後,把這一節換成下面的內容,再用 `pass` 或 `fail` 重新產生報告。
TEXT
```

If it has run, append the run and the lines that matter:

```bash
cd /Users/ysya/project/sideproj/sshelter
RUN_ID=$(gh run list --workflow spike-windows.yml --branch next/own-ssh --limit 1 --json databaseId --jq '.[0].databaseId')
{
  printf '\n## Windows\n\n`spike windows` 的執行:%s;結論:%s。\n\n```\n' \
    "$(gh run view "$RUN_ID" --json url --jq .url)" "$(gh run view "$RUN_ID" --json conclusion --jq .conclusion)"
  gh run view "$RUN_ID" --log | grep -E 'FACT windows\.|client-ok|test result' | cut -f3- | sed -E 's/^[0-9T:.Z-]+ //'
  printf '```\n'
} >> docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md
```

(`gh run view --log` prints `job<TAB>step<TAB>timestamp text` lines; adjust the `cut`/`sed` if the format differs.) If the run failed, add under it, in your own words and with the log's error text, what failed: the server setup (the OpenSSH client step) or russh (the test step).

- [ ] **Step 6: Judge the rows, then commit**

Open the report and do the manual part once:
1. In "規格裡不成立的假設", check every row against the evidence above it. A row whose evidence supports the spec's original statement is deleted, and a one-line entry goes under "核對過、沒問題" (which spec point, which test proves it). A row the evidence supports stays. Replace the instruction line under "核對過、沒問題" with those entries, or with `無` if nothing was deleted.
2. Read the verdict line. If it says no-go or pending, the reasons are the failed conditions and the Windows section; do not edit the line.
3. Under "用到的 russh 0.64.1 介面", replace the parenthesised line with the names the compiler made you change (from Task 2 step 10 and the other groups), or with `無`.
4. The first lines must still say that `spike/russh/` is throwaway and not product code.

```bash
cd /Users/ysya/project/sideproj/sshelter
grep -n "(上表刪掉的列\|(寫在這一行" docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md || echo "no instruction lines left"
git status --short
git diff --stat -- src-tauri/Cargo.toml src-tauri/Cargo.lock
git add docs/superpowers/specs/2026-10-10-own-ssh-spike-report.md
git commit -m "docs: the phase 0 spike report for the own SSH client"
git log --oneline next/own-ssh -12
```

Expected: `no instruction lines left`; `git status --short` lists nothing but the report (everything else was committed task by task; `spike/russh/target/` is ignored); the `git diff --stat` prints nothing; the log shows the Task 1–5 commits and this one, and nothing was pushed. Hand the report's verdict, its wrong-assumption rows and the "沒驗證到的" list back to the controller.

---

## Plan notes

What was run while this plan was written (so the executor knows what is already proven), all outside the repository, in copies under the session scratchpad:
- **Task 5, end to end**, in a copy of `src-tauri`: the guard fails with `E0583` first, then by assertion with 36 tests passing, then everything passes (168 in `ipc:: agent::`, 1417 in the whole lib suite, 0 failed); the Windows branch type-checks for `x86_64-pc-windows-msvc` and a deliberate error is reported; the edit scripts were replayed step by step on a pristine copy and produced identical files.
- **The one-line `pub mod vault;`**: `cargo check --lib` prints the same single pre-existing warning (`set_host_enabled`) before and after.
- **The harness**: `harness_smoke` (the scratch sshd, the system `ssh`, the negative control) passes on this Mac with OpenSSH 10.3p1, and no `sshd` is left behind; `sshd -t -f` accepts every config variant the tests use (`ForceCommand /bin/sh`, `HostKeyAlgorithms ssh-rsa` with an RSA host key, `PubkeyAcceptedAlgorithms ssh-rsa`, `AuthenticationMethods publickey`, `LoginGraceTime`). With the system `ssh`: the forced `/bin/sh` shell prints `sh-3.2$ `, answers the markers and returns the exit status; `exec sh -c 'kill -9 $$'` makes sshd send `exit-signal`; stdout, stderr and exit codes arrive; `ssh -J` through the same host is refused by the client ("jumphost loop"), hence two servers.
- **The library's five unit tests** (the proxy's three against real tokio, the shell marker parser, the signer's blob parser) and `harness_smoke` pass in the scratch copy; the signer's `Material::sign` call, `spawn_blocking` and the `SSH_AGENT_RSA_SHA2_*` constants compiled against the real app library. Everything that calls russh only compiled, against the stub below.
- **The whole spike crate type-checks** for macOS and (without the app library) for `x86_64-pc-windows-msvc` against a *stub* of russh whose signatures were copied from docs.rs/russh/0.64.1. The stub proves the spike code is consistent with those signatures, not that the signatures are right.
- `actionlint` is clean on `spike-windows.yml` and on the edited `test-windows.yml`; the lock-delta script and the report filler ran on test inputs; the `E0432`/`E0583` red messages quoted above are the compiler's own.

What could not be verified offline (russh is not in the registry, and the plan forbids fetching it before Task 1):
- **Everything russh does at run time.** Which error a refused host key produces (`Error::UnknownKey` is expected), what `best_supported_rsa_hash` returns for a legacy server, whether russh accepts an `ssh-rsa` host key signature with its defaults, what keepalive does and how fast, whether a window change before the PTY exists is kept, whether the inner session ends with the outer one, whether an unread channel stalls its neighbours, the `Sig::KILL` debug text. The tests assert the design's expectation, and the Spike rule says what to do when they differ.
- **Names to confirm with the compiler after `cargo fetch`** (all read from docs.rs for 0.64.1): `russh::Signer::auth_sign(&mut self, &AgentIdentity, Option<HashAlg>, Vec<u8>)` and `russh::keys::agent::AgentIdentity::public_key()`; `Handler::check_server_key(&mut self, &PublicKeyOrCertificate)` and `Handler::disconnected(DisconnectReason<Self::Error>)` with `DisconnectReason::{ReceivedDisconnect, Error}`; `Handle::{authenticate_publickey_with, authenticate_publickey, authenticate_password, authenticate_none, authenticate_keyboard_interactive_start/respond, best_supported_rsa_hash, channel_open_session, channel_open_direct_tcpip, disconnect, is_closed}`; `Channel::{exec, request_pty, request_shell, window_change, data, wait, close, into_stream}`; `ChannelMsg::{Data, ExtendedData, ExitStatus, ExitSignal, Failure}`; `client::{Config.keepalive_interval/keepalive_max/preferred, connect, connect_stream, AuthResult, KeyboardInteractiveAuthResponse, Prompt, Msg}`; `Error::{UnknownKey, KeepaliveTimeout, NoCommonAlgo, ChannelOpenFailure, Keys}`; `MethodKind`, `Preferred::DEFAULT`, `Disconnect::ByApplication`, `SendError`, `keys::{PublicKey, HashAlg, Algorithm::is_rsa, PrivateKeyWithHashAlg::new, PublicKeyOrCertificate::public_key, load_secret_key}`. Task 2 step 10 lists the likeliest mismatches and their fixes.
- **Windows**: whether the Feature on Demand installs on `windows-latest`, the `icacls` lines, whether the service generates its host keys on first start, and whether the spike's test executable starts at all are decided by the first run; that is why the job has a Chocolatey fallback and `VERIFY` comments, and why it builds without the app library.
- **Linux**: untested; RHEL/Fedora crypto policies reject SHA-1 signatures (`SPIKE_SKIP_SHA1=1`).

Outside phase 0 on purpose: SFTP, port forwarding, OpenSSH certificates, agent forwarding (the spec's non-goals), the `AttachConsole`/VT-mode console work of `sshelter connect` on Windows (phase 3), and the IPC protocol itself (phase 1).

Where this plan departs from the brief, and why:
- The scratch-sshd harness is built in Task 2, not Task 3: Task 2 is the first test that needs a server, and Task 3 only extends what it needs from it.
- The vault is reached through a path dependency on the app library plus the one-line `pub mod vault;`, as the controller preferred. The fallback, if that ever proves too heavy, is to compile `vault/material.rs` unchanged into a small helper crate with `#[path]` and a stub `AppError`; it was not needed, and the spike would then not prove the whole app's dependency graph resolves with russh, which is the more valuable fact.
- The app library is an optional dependency (`app-vault`, on by default) so the Windows job and the session tests build without Tauri.
- The Windows workflow also triggers on `push` to `next/own-ssh`, because `workflow_dispatch` alone needs the file on the default branch.
- `ipc/`'s tests use a ping/pong handler instead of the agent protocol, `NoKeys` moved to `agent::session::testing`, and `test-windows.yml` gained `ipc::` in its filter; none of that changes behavior, and without the last one the Windows named-pipe tests would silently stop running.
