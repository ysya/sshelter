# 更新頻道(Stable / Beta)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 讓 SSHelter 可選更新頻道(Stable 預設 / Beta),Beta 透過自動更新收到 GitHub prerelease;並提供發 beta 的 CI 流程。

**Architecture:**
- Beta 頻道的更新清單放在固定的 GitHub prerelease `updater-beta`(資產 `latest.json`),由 CI 腳本 `scripts/beta-channel.mjs` 維護:只在新版本較新時覆寫。
- App 端 Stable 仍用前端 `@tauri-apps/plugin-updater` 的 `check()`(呼叫與順序不變);Beta 改由 Rust 指令以 `updater_builder().endpoints(...)` 讀 `updater-beta` 的清單。
- 新的「publish beta」workflow 以 `X.Y.Z-N` 版號建立 prerelease、四平台 build,完成後更新 Beta 清單;正式版 workflow 也在 build 後更新 Beta 清單。

**Tech Stack:** Tauri 2(`tauri-plugin-updater` 2.10.1)、Rust、React + TypeScript、Zustand、Vitest、GitHub Actions(`tauri-apps/tauri-action@v0`、`gh` CLI)、Node(僅內建模組)。

**Spec:** `docs/superpowers/specs/2026-10-01-update-channels-design.md`

## Global Constraints

- Stable 頻道的更新呼叫不變:`check()` → `update.downloadAndInstall()` → `relaunch()`,同樣的提示行為(同版本只提示一次、`busy` 防重入、toast id `sshelter-update`)。提示 toast 的程式可與 Beta 共用。
- Beta 清單網址固定為 `https://github.com/ysya/sshelter/releases/download/updater-beta/latest.json`;Beta 清單 release 的 tag 固定為 `updater-beta`(prerelease)。
- Beta 版號格式 `^\d+\.\d+\.\d+-\d+$`(例如 `0.16.1-1`),且必須大於 `.release-please-manifest.json` 的 `"."` 版本。
- 不改 `release-please-config.json`,不使用 release-please 的 `prerelease` 設定。`release.yml` 既有 job 不變,只新增 `beta-manifest` job。
- 不新增任何 npm 或 Rust 相依;Node 腳本只用 `node:` 內建模組與 GitHub runner 內建的 `gh`。
- 不設定 `SSHELTER_RELAY_URL`(0.16.0 不內建 relay);程式不需為此修改。
- UI 文案英文;Rust 註解繁體中文(與周圍一致);TS / YAML / JS 註解英文。
- 每個 task 結束時:`pnpm build && pnpm test` 全綠;動到 Rust 的 task 另需 `cd src-tauri && cargo test` 全綠、`cargo build` 只剩既有的 `set_host_enabled` 警告。
- 只用明確路徑 `git add`;不 push;每個 task 一個 commit(Conventional Commits)。

## Review Focus

1. 匯入的設定檔帶來未知的 `updateChannel`(例如 `"nightly"`)或根本沒有這個欄位 → 一律當 Stable,更新照常運作(Task 4 `normalizeUpdateChannel` 測試)。
2. 較新的 beta 發布後才發的舊版正式版修補(例如 `0.17.0-1` 之後的 `0.16.2`)不得把 Beta 清單蓋回去(Task 1 `shouldReplace` 測試)。
3. 非數字 pre-release 的 beta 版號(`0.16.1-beta.1`)必須在建立 release 前就被拒絕,否則 Windows MSI 會在 build 中途失敗、留下半套 release(Task 1 `validateBetaVersion` 測試)。
4. 不比目前正式版新的 beta 版號(正式版 `0.16.0` 已發時的 `0.16.0-1`)必須被拒絕,否則 Beta 使用者永遠收不到它(Task 1 `validateBetaVersion` 測試)。
5. 把 beta 版號寫進 `Cargo.toml` 時只改 `[package]` 的 `version`,不得動到相依套件表格裡的 `version = "…"`(Task 2 `setCargoPackageVersion` 測試)。

---

### Task 1: Beta 頻道腳本(版本比較、版號驗證、清單更新)

**Files:**
- Create: `scripts/beta-channel.mjs`
- Test: `scripts/beta-channel.test.mjs`

**Interfaces:**
- Produces(Task 2 的 workflow 使用):
  - CLI `node scripts/beta-channel.mjs check-version <version>`:版號不合法或不比正式版新 → 非零結束並印出原因。
  - CLI `node scripts/beta-channel.mjs update-manifest <source-tag>`:必要時建立 `updater-beta`,並在來源版本較新時以 `--clobber` 上傳 `latest.json`。需要環境變數 `GH_TOKEN`(與選用的 `GH_REPO`、`GITHUB_SHA`)。
  - 純函式:`parseVersion(text)`、`compareVersions(a, b)`、`shouldReplace(current, next)`、`validateBetaVersion(version, stableVersion)`。

- [ ] **Step 1: 寫失敗的測試**

建立 `scripts/beta-channel.test.mjs`:

```js
import { describe, expect, it } from "vitest";

import { compareVersions, parseVersion, shouldReplace, validateBetaVersion } from "./beta-channel.mjs";

describe("compareVersions", () => {
  it("orders numeric pre-releases below their release and by number", () => {
    expect(compareVersions("0.16.1-1", "0.16.1")).toBeLessThan(0);
    expect(compareVersions("0.16.1", "0.16.1-1")).toBeGreaterThan(0);
    expect(compareVersions("0.16.1-2", "0.16.1-1")).toBeGreaterThan(0);
    expect(compareVersions("0.16.1-10", "0.16.1-9")).toBeGreaterThan(0);
  });

  it("compares each part as a number and ignores a leading v", () => {
    expect(compareVersions("0.16.10", "0.16.9")).toBeGreaterThan(0);
    expect(compareVersions("1.0.0", "0.99.99")).toBeGreaterThan(0);
    expect(compareVersions("v0.16.0", "0.16.0")).toBe(0);
  });

  it("rejects versions the Windows MSI bundle cannot use", () => {
    expect(() => parseVersion("0.16.1-beta.1")).toThrow(/unsupported version/);
    expect(() => parseVersion("0.16")).toThrow(/unsupported version/);
  });
});

describe("shouldReplace", () => {
  it("seeds an empty channel", () => {
    expect(shouldReplace(null, "0.16.0")).toBe(true);
  });

  it("moves forward to a newer beta or release", () => {
    expect(shouldReplace("0.16.0", "0.16.1-1")).toBe(true);
    expect(shouldReplace("0.16.1-1", "0.16.1")).toBe(true);
  });

  it("never moves the channel backwards or rewrites the same version", () => {
    expect(shouldReplace("0.17.0-1", "0.16.2")).toBe(false);
    expect(shouldReplace("0.16.1", "0.16.1")).toBe(false);
  });
});

describe("validateBetaVersion", () => {
  it("accepts X.Y.Z-N newer than the current release", () => {
    expect(() => validateBetaVersion("0.16.1-1", "0.16.0")).not.toThrow();
  });

  it("rejects a non-numeric pre-release before anything is published", () => {
    expect(() => validateBetaVersion("0.16.1-beta.1", "0.16.0")).toThrow(/unsupported version/);
  });

  it("rejects a version without a pre-release suffix", () => {
    expect(() => validateBetaVersion("0.16.1", "0.16.0")).toThrow(/pre-release suffix/);
  });

  it("rejects a beta that is not newer than the current release", () => {
    expect(() => validateBetaVersion("0.16.0-1", "0.16.0")).toThrow(/must be newer/);
    expect(() => validateBetaVersion("0.15.1-1", "0.16.0")).toThrow(/must be newer/);
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm test -- beta-channel`
Expected: FAIL —— 找不到 `./beta-channel.mjs`。

- [ ] **Step 3: 實作腳本**

建立 `scripts/beta-channel.mjs`:

```js
#!/usr/bin/env node
// Beta update channel helpers for CI (spec: docs/superpowers/specs/2026-10-01-update-channels-design.md).
//
//   node scripts/beta-channel.mjs check-version <version>
//     Fails unless <version> is X.Y.Z-N (numeric pre-release: the Windows MSI rule) and newer
//     than the current release in .release-please-manifest.json.
//
//   node scripts/beta-channel.mjs update-manifest <source-tag>
//     Points the Beta channel (the `latest.json` asset on the `updater-beta` prerelease) at the
//     source release, but only when that release is newer than what the channel already offers,
//     so an older stable patch never moves beta users backwards. Creates `updater-beta` on first
//     use. Needs GH_TOKEN (and GH_REPO outside a checkout).
import { execFileSync } from "node:child_process";
import { mkdtempSync, readFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { pathToFileURL } from "node:url";

export const CHANNEL_TAG = "updater-beta";
const MANIFEST = "latest.json";
const CHANNEL_NOTES =
  "Maintained by CI: holds the Beta update channel's latest.json (Settings → General → Update channel). Not a release.";

const VERSION_RE = /^v?(\d+)\.(\d+)\.(\d+)(?:-(\d+))?$/;

/** Parse `X.Y.Z` or `X.Y.Z-N` (an optional leading `v` is ignored). */
export function parseVersion(text) {
  const match = VERSION_RE.exec(String(text).trim());
  if (!match) throw new Error(`unsupported version "${text}" (expected X.Y.Z or X.Y.Z-N)`);
  const [, major, minor, patch, pre] = match;
  return { major: Number(major), minor: Number(minor), patch: Number(patch), pre: pre === undefined ? null : Number(pre) };
}

/** Sort-style comparison; a release outranks every pre-release of the same X.Y.Z. */
export function compareVersions(a, b) {
  const x = parseVersion(a);
  const y = parseVersion(b);
  for (const key of ["major", "minor", "patch"]) {
    if (x[key] !== y[key]) return x[key] - y[key];
  }
  if (x.pre === y.pre) return 0;
  if (x.pre === null) return 1;
  if (y.pre === null) return -1;
  return x.pre - y.pre;
}

/** Whether the channel should move to `next`; `current` is null when it has no manifest yet. */
export function shouldReplace(current, next) {
  return current === null || compareVersions(next, current) > 0;
}

/** A beta is X.Y.Z-N and newer than the current release (otherwise no one would be offered it). */
export function validateBetaVersion(version, stableVersion) {
  if (parseVersion(version).pre === null) {
    throw new Error(`beta version "${version}" needs a numeric pre-release suffix (X.Y.Z-N, e.g. 0.16.1-1)`);
  }
  if (compareVersions(version, stableVersion) <= 0) {
    throw new Error(`beta version ${version} must be newer than the current release ${stableVersion}`);
  }
}

function gh(args) {
  return execFileSync("gh", args, { encoding: "utf8", stdio: ["ignore", "pipe", "inherit"] });
}

function manifestVersion(file) {
  const version = JSON.parse(readFileSync(file, "utf8")).version;
  parseVersion(version); // refuse to publish a manifest we could not compare later
  return version;
}

function checkVersion(version) {
  const stable = JSON.parse(readFileSync(".release-please-manifest.json", "utf8"))["."];
  validateBetaVersion(version, stable);
  console.log(`beta ${version} is newer than the current release ${stable}`);
}

function updateManifest(sourceTag) {
  const work = mkdtempSync(join(tmpdir(), "beta-channel-"));
  const sourceDir = join(work, "source");
  gh(["release", "download", sourceTag, "--pattern", MANIFEST, "--dir", sourceDir]);
  const next = manifestVersion(join(sourceDir, MANIFEST));

  const tags = JSON.parse(gh(["release", "list", "--limit", "1000", "--json", "tagName"])).map((r) => r.tagName);
  let current = null;
  if (!tags.includes(CHANNEL_TAG)) {
    const target = process.env.GITHUB_SHA ?? "main";
    gh(["release", "create", CHANNEL_TAG, "--prerelease", "--title", "Beta update channel", "--notes", CHANNEL_NOTES, "--target", target]);
  } else {
    const assets = JSON.parse(gh(["release", "view", CHANNEL_TAG, "--json", "assets"])).assets.map((a) => a.name);
    if (assets.includes(MANIFEST)) {
      const currentDir = join(work, "current");
      gh(["release", "download", CHANNEL_TAG, "--pattern", MANIFEST, "--dir", currentDir]);
      current = manifestVersion(join(currentDir, MANIFEST));
    }
  }

  if (!shouldReplace(current, next)) {
    console.log(`${CHANNEL_TAG} stays on ${current} (${sourceTag} carries ${next}, which is not newer)`);
    return;
  }
  gh(["release", "upload", CHANNEL_TAG, join(sourceDir, MANIFEST), "--clobber"]);
  console.log(`${CHANNEL_TAG} now offers ${next}${current ? ` (was ${current})` : ""}`);
}

function main([command, arg]) {
  if (command === "check-version" && arg) return checkVersion(arg);
  if (command === "update-manifest" && arg) return updateManifest(arg);
  throw new Error("usage: beta-channel.mjs check-version <version> | update-manifest <source-tag>");
}

if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
  try {
    main(process.argv.slice(2));
  } catch (error) {
    console.error(`::error::${error.message}`);
    process.exit(1);
  }
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `pnpm test -- beta-channel`
Expected: PASS(10 個測試)。

再確認 CLI 能讀到正式版版號(目前 `.release-please-manifest.json` 為 `0.15.1`):

Run: `node scripts/beta-channel.mjs check-version 0.16.1-1`
Expected: `beta 0.16.1-1 is newer than the current release 0.15.1`

Run: `node scripts/beta-channel.mjs check-version 0.16.1-beta.1`
Expected: `::error::unsupported version "0.16.1-beta.1" (expected X.Y.Z or X.Y.Z-N)`,結束碼 1。

- [ ] **Step 5: 全套並 Commit**

Run: `pnpm build && pnpm test`
Expected: 全綠。

```bash
git add scripts/beta-channel.mjs scripts/beta-channel.test.mjs
git commit -m "feat(ci): beta channel script that validates beta versions and maintains the updater-beta manifest"
```

---

### Task 2: CI —— 正式版更新 Beta 清單、「publish beta」workflow、版號寫入腳本

**Files:**
- Create: `scripts/set-app-version.mjs`
- Test: `scripts/set-app-version.test.mjs`
- Create: `.github/workflows/beta.yml`
- Modify: `.github/workflows/release.yml`(只新增 `beta-manifest` job)
- Modify: `README.md`(`## Development` 段落加「Publishing a beta」)

**Interfaces:**
- Consumes: Task 1 的 `node scripts/beta-channel.mjs check-version <version>`、`update-manifest <source-tag>`。
- Produces: CLI `node scripts/set-app-version.mjs <version>`(把版號寫進 `package.json`、`src-tauri/tauri.conf.json`、`src-tauri/Cargo.toml` 的 `[package]`);純函式 `setCargoPackageVersion(toml, version)`、`setJsonVersion(text, version)`。

- [ ] **Step 1: 寫失敗的測試**

建立 `scripts/set-app-version.test.mjs`:

```js
import { describe, expect, it } from "vitest";

import { setCargoPackageVersion, setJsonVersion } from "./set-app-version.mjs";

const CARGO = `[package]
name = "sshelter"
version = "0.16.0"
edition = "2021"

[dependencies]
serde = { version = "1", features = ["derive"] }

[dependencies.tauri]
version = "2"
`;

describe("setCargoPackageVersion", () => {
  it("changes only the [package] version", () => {
    const out = setCargoPackageVersion(CARGO, "0.16.1-1");
    expect(out).toContain('[package]\nname = "sshelter"\nversion = "0.16.1-1"\n');
    expect(out).toContain('serde = { version = "1", features = ["derive"] }');
    expect(out).toContain('[dependencies.tauri]\nversion = "2"\n');
  });

  it("fails loudly when there is no [package] version", () => {
    expect(() => setCargoPackageVersion('[dependencies]\nversion = "1"\n', "0.16.1-1")).toThrow(/\[package\] version/);
  });
});

describe("setJsonVersion", () => {
  it("replaces the top-level version and keeps the other keys", () => {
    const out = JSON.parse(setJsonVersion('{"name":"sshelter","version":"0.16.0","private":true}', "0.16.1-1"));
    expect(out).toEqual({ name: "sshelter", version: "0.16.1-1", private: true });
  });

  it("fails loudly when there is no top-level version", () => {
    expect(() => setJsonVersion('{"name":"sshelter"}', "0.16.1-1")).toThrow(/version/);
  });
});
```

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm test -- set-app-version`
Expected: FAIL —— 找不到 `./set-app-version.mjs`。

- [ ] **Step 3: 實作版號寫入腳本**

建立 `scripts/set-app-version.mjs`:

```js
#!/usr/bin/env node
// Stamps an app version into the three files that carry it, in the CI workspace only — beta
// builds never commit this back (release-please keeps owning the version on main):
//
//   node scripts/set-app-version.mjs 0.16.1-1
import { readFileSync, writeFileSync } from "node:fs";
import { pathToFileURL } from "node:url";

/** Replace the `version` key of the `[package]` table only; dependency tables keep theirs. */
export function setCargoPackageVersion(toml, version) {
  let table = "";
  let replaced = false;
  const lines = toml.split("\n").map((line) => {
    const header = /^\s*\[([^\]]+)\]\s*$/.exec(line);
    if (header) {
      table = header[1].trim();
      return line;
    }
    if (!replaced && table === "package" && /^\s*version\s*=/.test(line)) {
      replaced = true;
      return line.replace(/=\s*"[^"]*"/, `= "${version}"`);
    }
    return line;
  });
  if (!replaced) throw new Error("Cargo.toml has no [package] version");
  return lines.join("\n");
}

/** Replace the top-level `version` of a JSON document. */
export function setJsonVersion(text, version) {
  const data = JSON.parse(text);
  if (typeof data.version !== "string") throw new Error("the JSON document has no top-level version");
  data.version = version;
  return `${JSON.stringify(data, null, 2)}\n`;
}

function main(version) {
  if (!version) throw new Error("usage: set-app-version.mjs <version>");
  for (const file of ["package.json", "src-tauri/tauri.conf.json"]) {
    writeFileSync(file, setJsonVersion(readFileSync(file, "utf8"), version));
  }
  const cargo = "src-tauri/Cargo.toml";
  writeFileSync(cargo, setCargoPackageVersion(readFileSync(cargo, "utf8"), version));
  console.log(`app version set to ${version}`);
}

if (process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href) {
  try {
    main(process.argv[2]);
  } catch (error) {
    console.error(`::error::${error.message}`);
    process.exit(1);
  }
}
```

- [ ] **Step 4: 執行測試確認通過**

Run: `pnpm test -- set-app-version`
Expected: PASS(4 個測試)。

- [ ] **Step 5: `release.yml` 新增 `beta-manifest` job**

在 `.github/workflows/release.yml` 的 `jobs:` 最後(`build` job 之後)加入,其他 job 不動:

```yaml
  beta-manifest:
    # Offer this release on the Beta update channel too, unless the channel already points at a
    # newer beta (scripts/beta-channel.mjs). Runs after every platform uploaded its artifacts, so
    # the release's latest.json is complete; skipped when any build leg failed.
    needs: [release-please, build]
    if: ${{ needs.release-please.outputs.release_created }}
    runs-on: ubuntu-latest
    permissions:
      contents: write
    # Shared with beta.yml: manifest updates queue instead of racing.
    concurrency:
      group: updater-beta-manifest
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Update the Beta channel manifest
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_REPO: ${{ github.repository }}
          TAG_NAME: ${{ needs.release-please.outputs.tag_name }}
        run: node scripts/beta-channel.mjs update-manifest "$TAG_NAME"
```

- [ ] **Step 6: 新增 `.github/workflows/beta.yml`**

建檔前先讀 `.github/workflows/release.yml` 的 `build` job,確認下面的 `build` job 與它的步驟(Linux 相依、pnpm 版本、Node、Rust、cache、relay URL 檢查、`tauri-action` 的 env)一致;它刻意與正式版的 `build` job 重複,讓正式版流程保持不動。

```yaml
name: publish beta

# Publishes a beta for the Beta update channel (Settings → General → Update channel):
#  1. `prepare` checks the version (X.Y.Z-N, newer than the current release) and creates the
#     GitHub prerelease v<version> on this commit. Prereleases never become "latest", so the
#     Stable channel (releases/latest/download/latest.json) never offers them.
#  2. `build` stamps the version into the CI workspace only (scripts/set-app-version.mjs; nothing
#     is committed), then builds and uploads the installers on every platform. Keep its steps in
#     sync with the `build` job in release.yml.
#  3. `beta-manifest` points the `updater-beta` release's latest.json at this beta.
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Beta version X.Y.Z-N, newer than the current release (e.g. 0.16.1-1)"
        required: true
      notes:
        description: "Release notes (optional)"
        required: false
        default: ""

permissions:
  contents: write

concurrency:
  group: publish-beta
  cancel-in-progress: false

jobs:
  prepare:
    runs-on: ubuntu-latest
    outputs:
      tag_name: ${{ steps.release.outputs.tag_name }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Check the version
        env:
          VERSION: ${{ inputs.version }}
        run: node scripts/beta-channel.mjs check-version "$VERSION"

      - name: Create the prerelease
        id: release
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_REPO: ${{ github.repository }}
          VERSION: ${{ inputs.version }}
          NOTES: ${{ inputs.notes }}
        run: |
          if gh release view "v$VERSION" >/dev/null 2>&1; then
            echo "::error::Release v$VERSION already exists."
            exit 1
          fi
          gh release create "v$VERSION" --prerelease --target "$GITHUB_SHA" \
            --title "v$VERSION (beta)" \
            --notes "${NOTES:-Beta build for the Beta update channel (Settings → General → Update channel).}"
          echo "tag_name=v$VERSION" >> "$GITHUB_OUTPUT"

  build:
    needs: prepare
    permissions:
      contents: write
    strategy:
      fail-fast: false
      matrix:
        include:
          - platform: macos-latest
            args: --target universal-apple-darwin
          - platform: ubuntu-22.04 # Linux x86_64
            args: ""
          - platform: ubuntu-22.04-arm # Linux arm64
            args: ""
          - platform: windows-latest # Windows x86_64
            args: ""
    runs-on: ${{ matrix.platform }}
    steps:
      - uses: actions/checkout@v4

      - name: Install Linux dependencies
        if: startsWith(matrix.platform, 'ubuntu')
        run: |
          sudo apt-get update
          # xdg-utils: the AppImage bundler needs xdg-open; the arm64 runner image lacks it.
          sudo apt-get install -y libwebkit2gtk-4.1-dev libappindicator3-dev librsvg2-dev patchelf xdg-utils

      - name: Setup pnpm
        uses: pnpm/action-setup@v4
        with:
          version: 11.3.0

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: lts/*
          cache: pnpm

      - name: Stamp the beta version (workspace only)
        shell: bash
        env:
          VERSION: ${{ inputs.version }}
        run: node scripts/set-app-version.mjs "$VERSION"

      - name: Install Rust stable
        uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ startsWith(matrix.platform, 'macos') && 'aarch64-apple-darwin,x86_64-apple-darwin' || '' }}

      - name: Rust cache
        uses: Swatinem/rust-cache@v2
        with:
          workspaces: "./src-tauri -> target"

      - name: Install frontend dependencies
        run: pnpm install

      - name: Check the sync relay URL
        shell: bash
        env:
          SSHELTER_RELAY_URL: ${{ vars.SSHELTER_RELAY_URL }}
        run: |
          if [ -z "$SSHELTER_RELAY_URL" ]; then
            echo "::notice::SSHELTER_RELAY_URL is not set: this beta has no built-in sync relay (users enter one in Settings → Sync)."
          elif [[ "$SSHELTER_RELAY_URL" != https://* ]]; then
            echo "::error::SSHELTER_RELAY_URL must start with https:// (got '$SSHELTER_RELAY_URL')."
            exit 1
          fi

      - name: Build and upload installers to the prerelease
        uses: tauri-apps/tauri-action@v0
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          TAURI_SIGNING_PRIVATE_KEY: ${{ secrets.TAURI_SIGNING_PRIVATE_KEY }}
          TAURI_SIGNING_PRIVATE_KEY_PASSWORD: ""
          SSHELTER_RELAY_URL: ${{ vars.SSHELTER_RELAY_URL }}
        with:
          # The prerelease already exists (created by `prepare`); attach assets to it by tag.
          tagName: ${{ needs.prepare.outputs.tag_name }}
          args: ${{ matrix.args }}

  beta-manifest:
    needs: [prepare, build]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    # Shared with release.yml: manifest updates queue instead of racing.
    concurrency:
      group: updater-beta-manifest
      cancel-in-progress: false
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: lts/*

      - name: Update the Beta channel manifest
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_REPO: ${{ github.repository }}
          TAG_NAME: ${{ needs.prepare.outputs.tag_name }}
        run: node scripts/beta-channel.mjs update-manifest "$TAG_NAME"
```

若 Step 6 開頭比對時發現 `release.yml` 的 `build` job 與上面任何一步不同(例如 pnpm 版本、Linux 套件),以 `release.yml` 為準調整 `beta.yml`,並在 commit 說明中列出。

- [ ] **Step 7: README 加發 beta 的說明**

在 `README.md` 的 `## Development` 段落最後加入:

```markdown
### Publishing a beta

Betas reach machines whose **Settings → General → Update channel** is **Beta**. In GitHub Actions run **publish beta** with a version `X.Y.Z-N` (numeric suffix only, e.g. `0.16.1-1`) newer than the current release. The workflow creates the prerelease `vX.Y.Z-N`, builds every platform, then points the `updater-beta` release's `latest.json` at it. Stable releases keep going through release-please and are offered on the Beta channel too.
```

- [ ] **Step 8: 驗證 YAML 並 Commit**

Run: `ruby -ryaml -e 'YAML.load_file(".github/workflows/beta.yml"); YAML.load_file(".github/workflows/release.yml"); puts "workflows parse"'`
Expected: `workflows parse`

Run: `pnpm build && pnpm test`
Expected: 全綠。

```bash
git add scripts/set-app-version.mjs scripts/set-app-version.test.mjs .github/workflows/beta.yml .github/workflows/release.yml README.md
git commit -m "ci: publish betas for the Beta update channel and keep its manifest current"
```

---

### Task 3: Rust —— Beta 頻道的檢查與安裝指令

**Files:**
- Create: `src-tauri/src/updater_channel.rs`
- Modify: `src-tauri/src/lib.rs`(`mod`、桌面區塊的 `.manage(...)`、`generate_handler!`)
- Generated: `src/bindings/UpdateInfo.ts`(`cargo test` 產生,一併 commit)

**Interfaces:**
- Produces(Task 4 使用):
  - `#[tauri::command] async fn updater_check_beta(app) -> Result<Option<UpdateInfo>, AppError>`
  - `#[tauri::command] async fn updater_install_beta(app) -> Result<(), AppError>`
  - TS 綁定 `UpdateInfo = { version: string, body: string | null }`(`src/bindings/UpdateInfo.ts`)。

- [ ] **Step 1: 寫模組與失敗的測試**

建立 `src-tauri/src/updater_channel.rs`(先只放常數、型別、純函式與測試,指令在 Step 3):

```rust
//! Beta 更新頻道(spec:docs/superpowers/specs/2026-10-01-update-channels-design.md §3.2)。
//! Stable 頻道沿用前端 `@tauri-apps/plugin-updater` 的 `check()`(讀 tauri.conf.json 的網址);
//! 這裡只處理 Beta:以 `updater_builder().endpoints(...)` 改讀 `updater-beta` release 的清單。
//! 簽章驗證與「遠端版本較新才更新(不降版)」沿用 plugin 預設。

use std::sync::Mutex;

use serde::Serialize;
use tauri::{AppHandle, Url};

use crate::error::AppError;

/// Beta 頻道的更新清單(由 CI 的 scripts/beta-channel.mjs 維護)。
pub const BETA_ENDPOINT: &str = "https://github.com/ysya/sshelter/releases/download/updater-beta/latest.json";

const NOTHING_PENDING: &str = "no beta update is ready to install; check for updates again";

/// 給前端的更新摘要,對應 plugin JS `Update` 的 version / body。
#[derive(Clone, Debug, Serialize)]
#[cfg_attr(test, derive(ts_rs::TS))]
#[cfg_attr(test, ts(export, export_to = "../../src/bindings/"))]
pub struct UpdateInfo {
    pub version: String,
    pub body: Option<String>,
}

fn beta_endpoint() -> Result<Url, AppError> {
    Url::parse(BETA_ENDPOINT).map_err(|e| AppError::Other(format!("invalid beta update endpoint: {e}")))
}

/// 取出等待安裝的更新:每個檢查到的更新最多安裝一次;沒有(app 重開過、或已裝過)就請使用者重新檢查。
fn take_pending<T>(slot: &Mutex<Option<T>>) -> Result<T, AppError> {
    slot.lock().unwrap().take().ok_or_else(|| AppError::Other(NOTHING_PENDING.to_string()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_beta_endpoint_is_the_updater_beta_release_manifest() {
        let url = beta_endpoint().unwrap();
        assert_eq!(url.scheme(), "https");
        assert_eq!(url.as_str(), "https://github.com/ysya/sshelter/releases/download/updater-beta/latest.json");
    }

    #[test]
    fn an_update_is_installed_at_most_once_and_otherwise_needs_a_new_check() {
        let empty: Mutex<Option<String>> = Mutex::new(None);
        assert_eq!(take_pending(&empty).unwrap_err().to_string(), NOTHING_PENDING);

        let slot = Mutex::new(Some("0.16.1-1".to_string()));
        assert_eq!(take_pending(&slot).unwrap(), "0.16.1-1");
        assert_eq!(take_pending(&slot).unwrap_err().to_string(), NOTHING_PENDING);
    }
}
```

在 `src-tauri/src/lib.rs` 的模組宣告區加入 `mod updater_channel;`(依字母順序放在 `mod tray;` 之後)。

- [ ] **Step 2: 執行測試確認通過(純函式)並產生綁定**

Run: `cd src-tauri && cargo test updater_channel`
Expected: 2 個測試 PASS,且 `src/bindings/UpdateInfo.ts` 產生,內容為 `export type UpdateInfo = { version: string, body: string | null, };`(註解行之外)。此時 `cargo build` 會出現 `beta_endpoint`、`take_pending` 未使用的警告,Step 3 接上指令後消失。

- [ ] **Step 3: 實作指令並註冊**

在 `src-tauri/src/updater_channel.rs` 的 `#[cfg(test)] mod tests` 之前加入:

```rust
/// `updater_check_beta` 找到、等使用者按「Install & restart」的更新(只在桌面平台有 updater)。
#[cfg(desktop)]
#[derive(Default)]
pub struct PendingBetaUpdate(Mutex<Option<tauri_plugin_updater::Update>>);

#[cfg(desktop)]
fn plugin_error(e: tauri_plugin_updater::Error) -> AppError {
    AppError::Other(e.to_string())
}

/// 檢查 Beta 頻道:有較新版本就暫存起來(覆寫先前的暫存)並回傳摘要;沒有就清掉暫存、回傳 None。
#[tauri::command]
pub async fn updater_check_beta(app: AppHandle) -> Result<Option<UpdateInfo>, AppError> {
    #[cfg(desktop)]
    {
        use tauri::Manager;
        use tauri_plugin_updater::UpdaterExt;

        let updater = app
            .updater_builder()
            .endpoints(vec![beta_endpoint()?])
            .map_err(plugin_error)?
            .build()
            .map_err(plugin_error)?;
        let update = updater.check().await.map_err(plugin_error)?;
        let info = update.as_ref().map(|u| UpdateInfo { version: u.version.clone(), body: u.body.clone() });
        *app.state::<PendingBetaUpdate>().0.lock().unwrap() = update;
        Ok(info)
    }
    #[cfg(not(desktop))]
    {
        let _ = app;
        Err(AppError::Other("updates are not supported on this platform".to_string()))
    }
}

/// 下載並安裝 `updater_check_beta` 暫存的更新(簽章由 plugin 驗證);重開 app 由前端 `relaunch()` 負責。
#[tauri::command]
pub async fn updater_install_beta(app: AppHandle) -> Result<(), AppError> {
    #[cfg(desktop)]
    {
        use tauri::Manager;

        let update = take_pending(&app.state::<PendingBetaUpdate>().0)?;
        update.download_and_install(|_, _| {}, || {}).await.map_err(plugin_error)
    }
    #[cfg(not(desktop))]
    {
        let _ = app;
        Err(AppError::Other("updates are not supported on this platform".to_string()))
    }
}
```

在 `src-tauri/src/lib.rs`:

1. `use` 區加入 `use updater_channel::{updater_check_beta, updater_install_beta};`。
2. 桌面 plugin 區塊(`#[cfg(desktop)] { builder = builder .plugin(tauri_plugin_updater::Builder::new().build()) …`)在 updater plugin 那一行之後加上 `.manage(updater_channel::PendingBetaUpdate::default())`。
3. `tauri::generate_handler![ … ]` 加入 `updater_check_beta,` 與 `updater_install_beta,`(放在 `app_set_close_to_tray,` 之後)。

- [ ] **Step 4: 測試與 build**

Run: `cd src-tauri && cargo test`
Expected: 全綠(既有 + 2 個新測試)。

Run: `cd src-tauri && cargo build 2>&1 | grep -E '^warning|^error' | sort | uniq -c`
Expected: 只有既有的 `set_host_enabled` 警告。

- [ ] **Step 5: Commit**

```bash
git add src-tauri/src/updater_channel.rs src-tauri/src/lib.rs src/bindings/UpdateInfo.ts
git commit -m "feat(updater): backend commands to check and install from the Beta update channel"
```

---

### Task 4: 前端 —— 頻道設定、Beta 檢查路徑、Sync 的 Beta 標示

**Files:**
- Modify: `src/lib/settings-logic.ts`(`UpdateChannel`、`normalizeUpdateChannel`)
- Test: `src/lib/settings-logic.test.ts`
- Modify: `src/stores/settings.ts`(`updateChannel`、`setUpdateChannel`)
- Modify: `src/lib/updater.ts`(依頻道選擇檢查與安裝方式)
- Modify: `src/components/SettingsDialog.tsx`(Updates 區塊的頻道選單、Sync 分類的 Beta 標記)
- Modify: `src/components/SyncPane.tsx`(「Sync is in beta」文案)
- Modify: `README.md`(Features 的 Auto-update、`## Sync` 段落)

**Interfaces:**
- Consumes: Task 3 的 `updater_check_beta`、`updater_install_beta`、`src/bindings/UpdateInfo.ts`。
- Produces: `type UpdateChannel = "stable" | "beta"`、`normalizeUpdateChannel(value: unknown): UpdateChannel`;store 欄位 `updateChannel` / `setUpdateChannel(channel)`。

- [ ] **Step 1: 寫失敗的測試**

在 `src/lib/settings-logic.test.ts` 的 import 加入 `normalizeUpdateChannel`,並在檔尾加入:

```ts
describe("normalizeUpdateChannel", () => {
  it("keeps the two known channels", () => {
    expect(normalizeUpdateChannel("stable")).toBe("stable");
    expect(normalizeUpdateChannel("beta")).toBe("beta");
  });

  it("treats anything else — missing, misspelled, from a newer build — as Stable", () => {
    expect(normalizeUpdateChannel(undefined)).toBe("stable");
    expect(normalizeUpdateChannel("nightly")).toBe("stable");
    expect(normalizeUpdateChannel("Beta")).toBe("stable");
    expect(normalizeUpdateChannel(1)).toBe("stable");
  });
});
```

(若該檔尚未 import `describe`/`expect`/`it`,沿用檔案既有的 vitest import。)

- [ ] **Step 2: 執行測試確認失敗**

Run: `pnpm test -- settings-logic`
Expected: FAIL —— `normalizeUpdateChannel` 不存在。

- [ ] **Step 3: 實作純邏輯與 store 欄位**

`src/lib/settings-logic.ts` 檔尾加入:

```ts
/** Update channels (Settings → General → Updates). */
export type UpdateChannel = "stable" | "beta";

/**
 * The channel to check, from a persisted or imported value. Anything other
 * than "beta" — missing, misspelled, written by a newer build — means Stable,
 * the code path every install used before channels existed.
 */
export function normalizeUpdateChannel(value: unknown): UpdateChannel {
  return value === "beta" ? "beta" : "stable";
}
```

`src/stores/settings.ts`:
1. 從 `@/lib/settings-logic` 的 import 加入 `type UpdateChannel`。
2. `SettingsState` 介面在 `setAutoCheckUpdates` 之後加入:

```ts
  /** Update channel: Stable (default) or Beta (prereleases). Read it through `normalizeUpdateChannel`. */
  updateChannel: UpdateChannel;
  setUpdateChannel: (channel: UpdateChannel) => void;
```

3. 預設值在 `setAutoCheckUpdates: …` 之後加入:

```ts
      updateChannel: "stable",
      setUpdateChannel: (updateChannel) => set({ updateChannel }),
```

- [ ] **Step 4: 執行測試確認通過**

Run: `pnpm test -- settings-logic`
Expected: PASS。

- [ ] **Step 5: `src/lib/updater.ts` 依頻道檢查**

整個檔案改為(Stable 的 `check()` → `downloadAndInstall()` → `relaunch()` 呼叫與提示行為不變):

```ts
import { check } from "@tauri-apps/plugin-updater";
import { relaunch } from "@tauri-apps/plugin-process";
import { toast } from "sonner";

import type { UpdateInfo } from "@/bindings/UpdateInfo";
import { tauriInvoke } from "@/lib/ipc";
import { normalizeUpdateChannel } from "@/lib/settings-logic";
import { useSettingsStore } from "@/stores/settings";

/** True while a check or install is already running (avoid double prompts). */
let busy = false;
/** Last version a silent check already prompted for — never re-toast the same one. */
let lastPromptedVersion: string | null = null;
/** A stable toast id so repeated prompts replace rather than stack. */
const UPDATE_TOAST_ID = "sshelter-update";

/** What the prompt needs from either channel. */
interface FoundUpdate {
  version: string;
  body?: string | null;
  install: () => Promise<void>;
}

/** Stable: the plugin's own check against tauri.conf.json's endpoint — the pre-channel path. */
async function checkStable(): Promise<FoundUpdate | null> {
  const update = await check();
  if (!update) return null;
  return { version: update.version, body: update.body, install: () => update.downloadAndInstall() };
}

/** Beta: the backend checks the `updater-beta` manifest and installs what it found. */
async function checkBeta(): Promise<FoundUpdate | null> {
  const info = await tauriInvoke<UpdateInfo | null>("updater_check_beta");
  if (!info) return null;
  return { version: info.version, body: info.body, install: () => tauriInvoke<void>("updater_install_beta") };
}

/**
 * Check the selected channel for a newer signed build: Stable reads GitHub
 * Releases' `latest.json`, Beta reads the `updater-beta` release's. When one is
 * found, prompt via a persistent toast; on confirm, download + install + relaunch.
 *
 * `silent` is for the automatic checks: no "up to date" confirmation, no error
 * toasts (dev builds and offline machines would nag otherwise), and the same
 * version is only prompted ONCE per app run — a manual check always prompts.
 */
export async function checkForUpdates({ silent }: { silent: boolean }): Promise<void> {
  if (busy) return;
  busy = true;
  try {
    const channel = normalizeUpdateChannel(useSettingsStore.getState().updateChannel);
    const update = channel === "beta" ? await checkBeta() : await checkStable();
    if (!update) {
      if (!silent) toast.success("SSHelter is up to date");
      return;
    }

    if (silent && update.version === lastPromptedVersion) return;
    lastPromptedVersion = update.version;

    toast.info(`Update available: v${update.version}`, {
      id: UPDATE_TOAST_ID,
      description: update.body?.split("\n")[0],
      duration: Infinity,
      action: {
        label: "Install & restart",
        onClick: () => {
          void installUpdate(update);
        },
      },
    });
  } catch (e) {
    if (!silent) {
      toast.error("Could not check for updates", { description: String(e) });
    } else {
      console.warn("[updater] silent check failed:", e);
    }
  } finally {
    busy = false;
  }
}

async function installUpdate(update: FoundUpdate) {
  const id = toast.loading(`Downloading v${update.version}…`);
  try {
    await update.install();
    toast.success("Update installed — restarting…", { id });
    await relaunch();
  } catch (e) {
    toast.error("Update failed", { id, description: String(e) });
  }
}
```

- [ ] **Step 6: Settings → General 的頻道選單**

`src/components/SettingsDialog.tsx`:

1. 從 `@/lib/settings-logic` 的 import 加入 `normalizeUpdateChannel` 與 `type UpdateChannel`。
2. `GeneralPane` 內、`const setAutoCheckUpdates = …` 之後加入:

```tsx
  const updateChannel = normalizeUpdateChannel(useSettingsStore((s) => s.updateChannel));
  const setUpdateChannel = useSettingsStore((s) => s.setUpdateChannel);
```

3. `onCheckNow` 定義之後加入:

```tsx
  const onChannelChange = (channel: UpdateChannel) => {
    setUpdateChannel(channel);
    // Switching to Beta checks right away so the user learns whether a beta is available.
    if (channel === "beta") void onCheckNow();
  };
```

4. Updates 區塊中,`id="set-auto-update"` 那個 `SettingsRow` 之後加入:

```tsx
          <SettingsRow
            id="set-update-channel"
            label="Update channel"
            description="Beta gets preview builds earlier and may be less stable. Switching back to Stable keeps your current version until a newer stable release is out."
          >
            <Select value={updateChannel} onValueChange={(value) => onChannelChange(normalizeUpdateChannel(value))}>
              <SelectTrigger id="set-update-channel" className="h-7 w-[10rem] text-sm">
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="stable">Stable</SelectItem>
                <SelectItem value="beta">Beta</SelectItem>
              </SelectContent>
            </Select>
          </SettingsRow>
```

- [ ] **Step 7: Sync 的 Beta 標示**

`src/components/SettingsDialog.tsx`:

1. `CATEGORIES` 的型別加上 `badge?: string;`,`sync` 那一筆改為 `{ id: "sync", label: "Sync", icon: RefreshCw, badge: "Beta" },`。
2. 分類按鈕內 `{c.label}` 之後加入:

```tsx
                    {c.badge && (
                      <Badge variant="outline" className="ml-auto h-4 px-1 text-[10px] font-normal">
                        {c.badge}
                      </Badge>
                    )}
```

`src/components/SyncPane.tsx`:

1. `NotJoinedPane` 的「Sync chain」`Section` 說明改為以 `Sync is in beta. ` 開頭:`"Sync is in beta. Keep hosts in sync across your computers without an account. A 24-word recovery phrase is the only secret; the relay only ever stores encrypted records."`
2. `JoinedPane` 的「Sync chain」`Section` 說明改為以 `Sync is in beta · ` 開頭:`` `Sync is in beta · Chain ${s.chain_short ?? ""} · ${s.hosts_in_sync} hosts in sync · last sync ${lastSync}` ``。

- [ ] **Step 8: README**

`README.md`:

1. Features 的 **Auto-update** 條目句尾加上:` Pick the Stable or Beta update channel in Settings → General.`
2. `## Sync` 段落第一句之前加上:`Sync is in beta. This release has no built-in relay: deploy your own from \`relay/\` and enter its URL in *Settings → Sync* first.`

- [ ] **Step 9: 全套驗證與 Commit**

Run: `pnpm build && pnpm test`
Expected: 全綠(`noUnusedLocals` 下沒有未使用的 import)。

Run: `cd src-tauri && cargo test 2>&1 | grep 'test result'`
Expected: 全綠。

```bash
git add src/lib/settings-logic.ts src/lib/settings-logic.test.ts src/stores/settings.ts src/lib/updater.ts src/components/SettingsDialog.tsx src/components/SyncPane.tsx README.md
git commit -m "feat(updater): Stable/Beta update channel setting and mark sync as beta"
```

- [ ] **Step 10: 發版前的手動檢查(交給使用者)**

`pnpm tauri dev` 後到 Settings → General:
- Update channel 為 Stable 時按「Check now」→「SSHelter is up to date」(dev 版本號等於最新正式版,或顯示可更新到更新的正式版)。
- 切到 Beta → 立即檢查 → 在 `updater-beta` 尚未建立前顯示「Could not check for updates」(預期);0.16.0 發布後應顯示「up to date」。
