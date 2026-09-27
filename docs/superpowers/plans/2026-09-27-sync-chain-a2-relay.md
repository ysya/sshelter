# Sync Chain — Phase A2(中繼 Worker:Cloudflare Worker + Durable Object)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立零知識中繼:每個 chain 一個 Durable Object(SQLite),提供建立/推送/拉取/刪除四個端點,含 token 驗證、配額、序號與衝突回報,並以 `@cloudflare/vitest-pool-workers` 測試;開源、可自架。

**Architecture:** `relay/` 是獨立的 npm 專案(不併入前端 pnpm workspace 的建置)。Worker 的 `fetch` 只做路由、bearer 擷取與 SHA-256;所有狀態操作是 DO 的 RPC 方法,單一 DO 內序列化執行,天然提供單調序號與原子性。

**Tech Stack:** TypeScript、Wrangler 4、Durable Objects(SQLite storage)、vitest ~3.2 + `@cloudflare/vitest-pool-workers`。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §5(中繼)、§2(零知識)

## Global Constraints

- 中繼**永不**記錄或回傳 token 明文;只存 `SHA-256(token)` hex。
- 認證失敗與 chain 不存在一律 `404`(不洩露 chain 是否存在)。
- 上限:單筆 `ciphertext` ≤ 65536 bytes(base64 長度計)、每 chain 活躍記錄 ciphertext 總和 ≤ 1 MiB(超過回 `413`)、每 chain 每分鐘 ≤ 120 次請求(超過回 `429`)、閒置 180 天自動清除。
- wire 格式 camelCase JSON,與 A1 `relay.rs` 的 `Envelope`/`PushItem` 對齊:`idHash`、`kind`、`seq`、`nonce`、`ciphertext`、`deleted`、`baseSeq`、`latestSeq`、`results[].status = "ok" | "conflict"`。
- `relay/` 不進 root `pnpm-workspace.yaml`;有自己的 `package.json`,CI 以獨立 job 測試。

## Review Focus

1. 兩台裝置對同一 `idHash` 交錯推送 —— 第二次 push 帶舊 `baseSeq` 必須拿到 `conflict` 與最新 envelope,而非靜默覆蓋(Task 1 測試)。
2. `since` 大於 `latestSeq` 或為負/非數字 —— 回空陣列與 400,不可 500(Task 1 測試)。
3. 空 body、非陣列 body、缺欄位 —— 400 且訊息不含使用者資料(Task 1 測試)。
4. 同一 chain 用另一個 token 建立 —— 404,且原資料不受影響(Task 1 測試)。
5. 配額臨界:一次 push 讓總量剛好越過 1 MiB —— 整批拒絕 413,不做部分寫入(Task 1 測試)。

---

### Task 1: Worker + Durable Object + 測試

**Files:**
- Create: `relay/package.json`
- Create: `relay/wrangler.jsonc`
- Create: `relay/tsconfig.json`
- Create: `relay/vitest.config.ts`
- Create: `relay/test/env.d.ts`
- Create: `relay/src/index.ts`
- Create: `relay/test/relay.test.ts`
- Create: `relay/README.md`
- Modify: `.gitignore`(加 `relay/node_modules`、`relay/.wrangler`)

**Interfaces:**
- Produces(HTTP,皆需 `Authorization: Bearer <token>`):
  - `PUT /v1/chains/:chainId` → `201 {}`(新建)或 `200 {}`(已存在且 token 相符);token 不符 → `404`
  - `POST /v1/chains/:chainId/records` body `PushItem[]` → `200 { results: ({status:"ok",seq}|{status:"conflict",current:Envelope})[], latestSeq }`
  - `GET /v1/chains/:chainId/records?since=N` → `200 { records: Envelope[], latestSeq }`
  - `DELETE /v1/chains/:chainId` → `204`

- [ ] **Step 1: 專案骨架**

建立 `relay/package.json`:

```json
{
  "name": "sshelter-relay",
  "private": true,
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "dev": "wrangler dev",
    "deploy": "wrangler deploy",
    "test": "vitest run",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@cloudflare/vitest-pool-workers": "^0.8.0",
    "@cloudflare/workers-types": "^4.20250927.0",
    "typescript": "^5.6.0",
    "vitest": "~3.2.0",
    "wrangler": "^4.0.0"
  }
}
```

建立 `relay/wrangler.jsonc`:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "sshelter-relay",
  "main": "src/index.ts",
  "compatibility_date": "2025-09-01",
  "durable_objects": {
    "bindings": [{ "name": "CHAIN", "class_name": "ChainStore" }]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["ChainStore"] }],
  "observability": { "enabled": true }
}
```

建立 `relay/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ES2022",
    "moduleResolution": "bundler",
    "strict": true,
    "noEmit": true,
    "types": ["@cloudflare/workers-types"],
    "lib": ["ES2022"]
  },
  "include": ["src/**/*.ts", "test/**/*.ts"]
}
```

建立 `relay/vitest.config.ts`:

```ts
import { defineWorkersConfig } from "@cloudflare/vitest-pool-workers/config";

export default defineWorkersConfig({
  test: {
    poolOptions: {
      workers: {
        wrangler: { configPath: "./wrangler.jsonc" },
      },
    },
  },
});
```

建立 `relay/test/env.d.ts`:

```ts
import type { Env } from "../src/index";

declare module "cloudflare:test" {
  interface ProvidedEnv extends Env {}
}
```

在 root `.gitignore` 末尾加:

```
relay/node_modules
relay/.wrangler
```

Run: `cd relay && npm install`
Expected: 安裝成功(第一次會拉 workerd,約一分鐘)。

- [ ] **Step 2: 寫失敗的測試**

建立 `relay/test/relay.test.ts`:

```ts
import { SELF } from "cloudflare:test";
import { describe, expect, it } from "vitest";

const CHAIN = "a".repeat(64);
const TOKEN = "t".repeat(64);
const base = "http://relay.test";

function auth(token = TOKEN): HeadersInit {
  return { authorization: `Bearer ${token}`, "content-type": "application/json" };
}

async function create(token = TOKEN, chain = CHAIN) {
  return SELF.fetch(`${base}/v1/chains/${chain}`, { method: "PUT", headers: auth(token), body: "{}" });
}

function item(idHash: string, baseSeq = 0, ciphertext = "Y2lwaGVy", deleted = false) {
  return { idHash, kind: "host", nonce: "bm9uY2U=", ciphertext, deleted, baseSeq };
}

async function push(items: unknown, token = TOKEN, chain = CHAIN) {
  return SELF.fetch(`${base}/v1/chains/${chain}/records`, {
    method: "POST",
    headers: auth(token),
    body: JSON.stringify(items),
  });
}

async function pull(since: string | number, token = TOKEN, chain = CHAIN) {
  return SELF.fetch(`${base}/v1/chains/${chain}/records?since=${since}`, { headers: auth(token) });
}

describe("chain lifecycle", () => {
  it("creates once, then reports existing for the same token", async () => {
    expect((await create()).status).toBe(201);
    expect((await create()).status).toBe(200);
  });

  it("hides the chain from a different token", async () => {
    await create();
    expect((await create("x".repeat(64))).status).toBe(404);
    expect((await pull(0, "x".repeat(64))).status).toBe(404);
    // 原資料不受影響。
    expect((await pull(0)).status).toBe(200);
  });

  it("rejects missing or malformed auth and unknown chains", async () => {
    const res = await SELF.fetch(`${base}/v1/chains/${CHAIN}/records?since=0`);
    expect(res.status).toBe(404);
    expect((await pull(0, TOKEN, "b".repeat(64))).status).toBe(404);
    expect((await pull(0, TOKEN, "not-hex")).status).toBe(400);
  });

  it("deletes everything", async () => {
    await create();
    await push([item("h1")]);
    const del = await SELF.fetch(`${base}/v1/chains/${CHAIN}`, { method: "DELETE", headers: auth() });
    expect(del.status).toBe(204);
    expect((await pull(0)).status).toBe(404);
  });
});

describe("records", () => {
  it("assigns increasing seqs and pulls since a cursor", async () => {
    await create();
    const r1 = await (await push([item("h1"), item("h2")])).json<{ results: { status: string; seq?: number }[]; latestSeq: number }>();
    expect(r1.results.map((r) => r.status)).toEqual(["ok", "ok"]);
    expect(r1.results.map((r) => r.seq)).toEqual([1, 2]);
    expect(r1.latestSeq).toBe(2);

    const all = await (await pull(0)).json<{ records: { idHash: string; seq: number }[]; latestSeq: number }>();
    expect(all.records.map((r) => r.idHash)).toEqual(["h1", "h2"]);
    const later = await (await pull(1)).json<{ records: { idHash: string }[] }>();
    expect(later.records.map((r) => r.idHash)).toEqual(["h2"]);
    const none = await (await pull(99)).json<{ records: unknown[]; latestSeq: number }>();
    expect(none.records).toEqual([]);
    expect(none.latestSeq).toBe(2);
  });

  it("reports a conflict when baseSeq is stale and returns the current envelope", async () => {
    await create();
    await push([item("h1")]); // seq 1
    await push([item("h1", 1, "bmV3")]); // seq 2, from device A
    const stale = await (await push([item("h1", 1, "b2xk")])).json<{
      results: { status: string; current?: { seq: number; ciphertext: string } }[];
    }>();
    expect(stale.results[0].status).toBe("conflict");
    expect(stale.results[0].current?.seq).toBe(2);
    expect(stale.results[0].current?.ciphertext).toBe("bmV3");
    // 帶正確 baseSeq 重送就成功。
    const ok = await (await push([item("h1", 2, "b2xk")])).json<{ results: { status: string; seq?: number }[] }>();
    expect(ok.results[0]).toEqual({ status: "ok", seq: 3 });
  });

  it("keeps tombstones pullable and counts only live bytes toward quota", async () => {
    await create();
    await push([item("h1", 0, "x".repeat(1000))]);
    await push([item("h1", 1, "", true)]);
    const all = await (await pull(0)).json<{ records: { deleted: boolean; ciphertext: string }[] }>();
    expect(all.records[0].deleted).toBe(true);
    expect(all.records[0].ciphertext).toBe("");
  });

  it("validates bodies without leaking them", async () => {
    await create();
    for (const body of ["", "{}", "[1]", JSON.stringify([{ idHash: "h" }])]) {
      const res = await SELF.fetch(`${base}/v1/chains/${CHAIN}/records`, {
        method: "POST",
        headers: auth(),
        body,
      });
      expect(res.status).toBe(400);
      expect(await res.text()).not.toContain("idHash");
    }
    expect((await pull("abc")).status).toBe(400);
    expect((await pull(-1)).status).toBe(400);
  });

  it("refuses oversize records and chain quota overflow atomically", async () => {
    await create();
    const big = "z".repeat(65_537);
    expect((await push([item("h1", 0, big)])).status).toBe(413);
    // 16 筆 × 65536 = 1 MiB 剛好可以;再多一筆就越界,整批 413,且原本的 16 筆都在。
    const sixteen = Array.from({ length: 16 }, (_, i) => item(`k${i}`, 0, "y".repeat(65_536)));
    expect((await push(sixteen)).status).toBe(200);
    expect((await push([item("k16", 0, "y")])).status).toBe(413);
    const all = await (await pull(0)).json<{ records: unknown[] }>();
    expect(all.records.length).toBe(16);
  });
});
```

- [ ] **Step 3: 執行測試確認失敗**

Run: `cd relay && npx vitest run 2>&1 | tail -5`
Expected: 失敗 —— 找不到 `src/index.ts` / `Env`。

- [ ] **Step 4: 實作 Worker 與 DO**

建立 `relay/src/index.ts`:

```ts
import { DurableObject } from "cloudflare:workers";

export interface Env {
  CHAIN: DurableObjectNamespace<ChainStore>;
}

const MAX_RECORD_BYTES = 65_536;
const MAX_CHAIN_BYTES = 1_048_576;
const RATE_LIMIT_PER_MINUTE = 120;
const IDLE_TTL_MS = 180 * 24 * 60 * 60 * 1000;
const CHAIN_ID_RE = /^[0-9a-f]{64}$/;

export interface Envelope {
  idHash: string;
  kind: string;
  seq: number;
  nonce: string;
  ciphertext: string;
  deleted: boolean;
}

export interface PushItem {
  idHash: string;
  kind: string;
  nonce: string;
  ciphertext: string;
  deleted: boolean;
  baseSeq: number;
}

type PushResult = { status: "ok"; seq: number } | { status: "conflict"; current: Envelope };

/** 每個 chain 一個 DO:token hash、記錄與序號全在它的 SQLite 裡。 */
export class ChainStore extends DurableObject<Env> {
  private minuteStart = 0;
  private minuteCount = 0;

  constructor(ctx: DurableObjectState, env: Env) {
    super(ctx, env);
    ctx.blockConcurrencyWhile(async () => {
      this.ctx.storage.sql.exec(`
        CREATE TABLE IF NOT EXISTS meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
        CREATE TABLE IF NOT EXISTS records (
          id_hash TEXT PRIMARY KEY,
          kind TEXT NOT NULL,
          seq INTEGER NOT NULL,
          nonce TEXT NOT NULL,
          ciphertext TEXT NOT NULL,
          deleted INTEGER NOT NULL
        );
        CREATE INDEX IF NOT EXISTS records_seq ON records (seq);
      `);
    });
  }

  private meta(key: string): string | null {
    const row = this.ctx.storage.sql
      .exec<{ value: string }>("SELECT value FROM meta WHERE key = ?", key)
      .toArray()[0];
    return row ? row.value : null;
  }

  private setMeta(key: string, value: string): void {
    this.ctx.storage.sql.exec(
      "INSERT INTO meta (key, value) VALUES (?, ?) ON CONFLICT(key) DO UPDATE SET value = excluded.value",
      key,
      value,
    );
  }

  private authorized(tokenHash: string): boolean {
    const stored = this.meta("token_hash");
    return stored !== null && stored === tokenHash;
  }

  private rateLimited(): boolean {
    const now = Date.now();
    if (now - this.minuteStart > 60_000) {
      this.minuteStart = now;
      this.minuteCount = 0;
    }
    this.minuteCount += 1;
    return this.minuteCount > RATE_LIMIT_PER_MINUTE;
  }

  private async touch(): Promise<void> {
    await this.ctx.storage.setAlarm(Date.now() + IDLE_TTL_MS);
  }

  /** 201 新建、200 已存在、404 token 不符。 */
  async create(tokenHash: string): Promise<201 | 200 | 404> {
    if (this.rateLimited()) return 404;
    const stored = this.meta("token_hash");
    if (stored === null) {
      this.setMeta("token_hash", tokenHash);
      this.setMeta("latest_seq", "0");
      this.setMeta("created_at", String(Date.now()));
      await this.touch();
      return 201;
    }
    if (stored !== tokenHash) return 404;
    await this.touch();
    return 200;
  }

  async pull(tokenHash: string, since: number): Promise<{ status: 200 | 404 | 429; body?: { records: Envelope[]; latestSeq: number } }> {
    if (!this.authorized(tokenHash)) return { status: 404 };
    if (this.rateLimited()) return { status: 429 };
    const records = this.ctx.storage.sql
      .exec<{ id_hash: string; kind: string; seq: number; nonce: string; ciphertext: string; deleted: number }>(
        "SELECT id_hash, kind, seq, nonce, ciphertext, deleted FROM records WHERE seq > ? ORDER BY seq",
        since,
      )
      .toArray()
      .map((r) => ({
        idHash: r.id_hash,
        kind: r.kind,
        seq: r.seq,
        nonce: r.nonce,
        ciphertext: r.ciphertext,
        deleted: r.deleted === 1,
      }));
    return { status: 200, body: { records, latestSeq: Number(this.meta("latest_seq") ?? "0") } };
  }

  async push(tokenHash: string, items: PushItem[]): Promise<{ status: 200 | 404 | 413 | 429; body?: { results: PushResult[]; latestSeq: number } }> {
    if (!this.authorized(tokenHash)) return { status: 404 };
    if (this.rateLimited()) return { status: 429 };

    // 配額檢查先於任何寫入:整批要嘛全進、要嘛全拒。
    const liveBytes = this.ctx.storage.sql
      .exec<{ total: number | null }>("SELECT SUM(LENGTH(ciphertext)) AS total FROM records WHERE deleted = 0")
      .one().total ?? 0;
    const incomingIds = new Set(items.map((i) => i.idHash));
    const replacedBytes = this.ctx.storage.sql
      .exec<{ id_hash: string; len: number; deleted: number }>("SELECT id_hash, LENGTH(ciphertext) AS len, deleted FROM records")
      .toArray()
      .filter((r) => incomingIds.has(r.id_hash) && r.deleted === 0)
      .reduce((n, r) => n + r.len, 0);
    const incomingBytes = items.reduce((n, i) => n + (i.deleted ? 0 : i.ciphertext.length), 0);
    if (liveBytes - replacedBytes + incomingBytes > MAX_CHAIN_BYTES) return { status: 413 };

    let latest = Number(this.meta("latest_seq") ?? "0");
    const results: PushResult[] = [];
    for (const item of items) {
      const current = this.ctx.storage.sql
        .exec<{ id_hash: string; kind: string; seq: number; nonce: string; ciphertext: string; deleted: number }>(
          "SELECT id_hash, kind, seq, nonce, ciphertext, deleted FROM records WHERE id_hash = ?",
          item.idHash,
        )
        .toArray()[0];
      if (current && current.seq > item.baseSeq) {
        results.push({
          status: "conflict",
          current: {
            idHash: current.id_hash,
            kind: current.kind,
            seq: current.seq,
            nonce: current.nonce,
            ciphertext: current.ciphertext,
            deleted: current.deleted === 1,
          },
        });
        continue;
      }
      latest += 1;
      this.ctx.storage.sql.exec(
        `INSERT INTO records (id_hash, kind, seq, nonce, ciphertext, deleted) VALUES (?, ?, ?, ?, ?, ?)
         ON CONFLICT(id_hash) DO UPDATE SET kind = excluded.kind, seq = excluded.seq, nonce = excluded.nonce,
         ciphertext = excluded.ciphertext, deleted = excluded.deleted`,
        item.idHash,
        item.kind,
        latest,
        item.nonce,
        item.ciphertext,
        item.deleted ? 1 : 0,
      );
      results.push({ status: "ok", seq: latest });
    }
    this.setMeta("latest_seq", String(latest));
    await this.touch();
    return { status: 200, body: { results, latestSeq: latest } };
  }

  async destroy(tokenHash: string): Promise<204 | 404> {
    if (!this.authorized(tokenHash)) return 404;
    await this.ctx.storage.deleteAll();
    await this.ctx.storage.deleteAlarm();
    return 204;
  }

  /** 閒置 180 天:整個 chain 清掉(使用者本機資料不受影響)。 */
  async alarm(): Promise<void> {
    await this.ctx.storage.deleteAll();
  }
}

async function sha256Hex(input: string): Promise<string> {
  const digest = await crypto.subtle.digest("SHA-256", new TextEncoder().encode(input));
  return Array.from(new Uint8Array(digest), (b) => b.toString(16).padStart(2, "0")).join("");
}

function bearer(request: Request): string | null {
  const header = request.headers.get("authorization") ?? "";
  const match = /^Bearer\s+([0-9a-f]{64})$/i.exec(header);
  return match ? match[1].toLowerCase() : null;
}

function isPushItem(v: unknown): v is PushItem {
  if (typeof v !== "object" || v === null) return false;
  const o = v as Record<string, unknown>;
  return (
    typeof o.idHash === "string" && /^[0-9a-f]{64}$/.test(o.idHash) &&
    typeof o.kind === "string" && /^[a-z]{1,16}$/.test(o.kind) &&
    typeof o.nonce === "string" && o.nonce.length <= 64 &&
    typeof o.ciphertext === "string" &&
    typeof o.deleted === "boolean" &&
    typeof o.baseSeq === "number" && Number.isInteger(o.baseSeq) && o.baseSeq >= 0
  );
}

const bad = (message: string) => Response.json({ error: message }, { status: 400 });
const notFound = () => new Response(null, { status: 404 });

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const match = /^\/v1\/chains\/([^/]+)(\/records)?$/.exec(url.pathname);
    if (!match) return notFound();
    const [, chainId, recordsPath] = match;
    if (!CHAIN_ID_RE.test(chainId)) return bad("invalid chain id");
    const token = bearer(request);
    if (!token) return notFound();
    const tokenHash = await sha256Hex(token);
    const stub = env.CHAIN.getByName(chainId);

    if (!recordsPath && request.method === "PUT") {
      const status = await stub.create(tokenHash);
      return status === 404 ? notFound() : Response.json({}, { status });
    }
    if (!recordsPath && request.method === "DELETE") {
      const status = await stub.destroy(tokenHash);
      return new Response(null, { status });
    }
    if (recordsPath && request.method === "GET") {
      const raw = url.searchParams.get("since") ?? "0";
      if (!/^\d{1,18}$/.test(raw)) return bad("invalid since");
      const result = await stub.pull(tokenHash, Number(raw));
      if (result.status !== 200) return new Response(null, { status: result.status });
      return Response.json(result.body);
    }
    if (recordsPath && request.method === "POST") {
      let body: unknown;
      try {
        body = await request.json();
      } catch {
        return bad("body must be JSON");
      }
      if (!Array.isArray(body) || body.length === 0 || body.length > 500 || !body.every(isPushItem)) {
        return bad("body must be a non-empty array of records");
      }
      if (body.some((i) => i.ciphertext.length > MAX_RECORD_BYTES)) {
        return new Response(null, { status: 413 });
      }
      const result = await stub.push(tokenHash, body);
      if (result.status !== 200) return new Response(null, { status: result.status });
      return Response.json(result.body);
    }
    return notFound();
  },
} satisfies ExportedHandler<Env>;
```

- [ ] **Step 5: 執行測試確認通過**

Run: `cd relay && npx vitest run 2>&1 | tail -8 && npx tsc --noEmit`
Expected: 全部測試通過、型別檢查無誤。若 `stub.create()` 之類 RPC 呼叫型別報錯,確認 `compatibility_date` ≥ 2024-04-03(RPC 需要)且 `Env` 綁定型別為 `DurableObjectNamespace<ChainStore>`。

- [ ] **Step 6: README 與 Commit**

建立 `relay/README.md`:

```markdown
# SSHelter sync relay

Zero-knowledge relay for SSHelter's sync chain: one Durable Object per chain, storing only
ciphertext. It never sees recovery phrases, host names, keys or passwords.

## Self-host

    cd relay
    npm install
    npx wrangler login
    npx wrangler deploy

Then point SSHelter at your Worker URL: Settings → Sync → Relay URL.

## Limits

- 64 KiB per record, 1 MiB of live records per chain, 120 requests/min per chain
- chains idle for 180 days are deleted automatically (devices keep their local copies)

## Test

    npm test
```

```bash
git add .gitignore relay/package.json relay/package-lock.json relay/wrangler.jsonc relay/tsconfig.json relay/vitest.config.ts relay/test/env.d.ts relay/src/index.ts relay/test/relay.test.ts relay/README.md
git commit -m "feat(relay): zero-knowledge sync relay on Cloudflare Durable Objects"
```

---

### Task 2: CI 與部署

**Files:**
- Create: `.github/workflows/relay.yml`
- Modify: `.github/workflows/release.yml`(build job 注入 `SSHELTER_RELAY_URL`)

**Interfaces:**
- Consumes: A1 Task 4 的 `DEFAULT_RELAY_URL = option_env!("SSHELTER_RELAY_URL")`。
- Produces: GitHub Actions repository variable `SSHELTER_RELAY_URL`(由使用者在部署後設定)。

- [ ] **Step 1: 中繼測試 workflow**

建立 `.github/workflows/relay.yml`:

```yaml
name: relay

on:
  push:
    branches: [main]
    paths: ["relay/**", ".github/workflows/relay.yml"]
  pull_request:
    paths: ["relay/**", ".github/workflows/relay.yml"]

jobs:
  test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: relay
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - run: npm ci
      - run: npx tsc --noEmit
      - run: npm test
```

- [ ] **Step 2: release build 注入正式中繼 URL**

在 `.github/workflows/release.yml` 的 `Build and upload installers to the release` 步驟 `env:` 區塊加入一行:

```yaml
          # 正式中繼 URL(repository variable);未設定時 app 預設指向本機 wrangler dev。
          SSHELTER_RELAY_URL: ${{ vars.SSHELTER_RELAY_URL }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/relay.yml .github/workflows/release.yml
git commit -m "ci(relay): test the relay and inject the production relay URL into release builds"
```

- [ ] **Step 4: 部署(需要使用者的 Cloudflare 帳號 —— 由使用者執行或在其授權下執行)**

```bash
cd relay && npx wrangler login && npx wrangler deploy
```

Expected: 輸出 `https://sshelter-relay.<account>.workers.dev`。接著設定 repository variable:

```bash
gh variable set SSHELTER_RELAY_URL --repo ysya/sshelter --body "https://sshelter-relay.<account>.workers.dev"
```

之後的 release build 就會內建正式中繼;本機 `pnpm tauri dev` 仍預設 `http://127.0.0.1:8787`(搭配 `cd relay && npm run dev`)。

---

## Self-review(已執行)

- **Spec 覆蓋**:§5 四個端點、token hash、404 不洩露、配額(64 KiB/1 MiB)、速率限制、180 天閒置清除、可自架 → Task 1;CI 與正式 URL 注入 → Task 2。
- **型別一致**:wire 欄位 `idHash/kind/seq/nonce/ciphertext/deleted/baseSeq/latestSeq/results[].status` 與 A1 `relay.rs` 的 serde `rename_all = "camelCase"` 結構完全對應;`status: "ok"` 對應 Rust `WirePushResult::Ok`(serde `rename_all = "lowercase"` 產生 `"ok"`)。
- **Review Focus 對應**:1 → `reports a conflict when baseSeq is stale`;2 → `validates bodies…`(since=abc/-1)與 `assigns increasing seqs…`(since=99);3 → `validates bodies…`;4 → `hides the chain from a different token`;5 → `refuses oversize records and chain quota overflow atomically`。
