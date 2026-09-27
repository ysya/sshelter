# Sync Chain — Phase A2(中繼 Worker:Cloudflare Worker + Durable Object)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 建立零知識中繼:每個 chain 一個 Durable Object(SQLite),提供建立/推送/拉取/刪除四個端點,含 token 驗證、嚴格 base64 驗證、配額(含 tombstone、判定接受後才計)、序號與衝突回報、每 chain 速率限制、每 IP 建鏈與總請求限制、讀不存在的 chain 不配置儲存,並以 Workers Vitest 整合(`@cloudflare/vitest-plugin`)在 workerd 裡測試;開源、可自架。

**Architecture:** `relay/` 是獨立的 npm 專案(不併入前端 pnpm workspace 的建置)。Worker 的 `fetch` 只做路由、bearer 擷取、SHA-256、byte 上限的串流讀 body、格式驗證與每 IP 限制;所有 chain 狀態操作是 `ChainStore` DO 的 RPC 方法,單一 DO 內序列化執行,天然提供單調序號與原子性。`ChainStore` **只在 `PUT` 建立時建 schema**,其他 RPC 先查 `sqlite_master`,沒有 `meta` 表就 404 —— 讀不存在的 chain 不留下任何持久化資料。每 IP 限制由 `IpLimiter` DO(以 IP 為名)計數兩個桶(建鏈 20/小時、所有請求 1200/小時),閒置一小時後自我清除。

**Tech Stack:** TypeScript、Wrangler 4、Durable Objects(SQLite storage,RPC)、vitest 4.1 + `@cloudflare/vitest-plugin`(取代已進入遷移期的 `@cloudflare/vitest-pool-workers`)、`wrangler types` 產生執行期型別(不再依賴 `@cloudflare/workers-types`)。

**Spec:** `docs/superpowers/specs/2026-09-27-sync-chain-design.md` §5(中繼)、§2(零知識、HTTPS)

## Global Constraints

- 中繼**永不**記錄或回傳 token 明文;只存 `SHA-256(token)` hex。
- 認證失敗與 chain 不存在一律 `404`(不洩露 chain 是否存在);**讀取不存在的 chain 不得建立 schema 或寫入任何儲存**。
- `nonce`/`ciphertext` 必須是嚴格的標準 base64(ASCII、非空、長度為 4 的倍數、`={0,2}` padding),否則 `400`;配額以字元數計(ASCII 才能讓 JS `.length` 與 SQLite `LENGTH()` 一致)。
- 上限(spec §5):單筆 `ciphertext` ≤ 65536 字元;每 chain 儲存總量 = 所有列(**含 tombstone**)的 `LENGTH(ciphertext) + LENGTH(nonce)` 總和 ≤ 1 MiB、記錄數 ≤ 4096(超過 → 整批 `413`,不部分寫入);request body ≤ 1 MiB(串流讀取、超過即取消 → `413`);每次 push 1–200 筆;每 chain 每分鐘 ≤ 120 次請求(所有端點都計,含建鏈那一次;已授權但超限一律 `429`);每 IP 每小時 ≤ 20 次建鏈請求(`PUT` 嘗試,不論結果)、≤ 1200 次任何請求(`429`);閒置 180 天自動清除,成功授權的 pull 也刷新期限(每 instance 6 小時最多一次)。
- 配額以「判定接受之後」的用量計算:conflict 的項目不佔配額;被取代的舊列先扣掉再加新列。
- `DELETE` 之後同一個 DO instance 必須能再次建立(`deleteAll()` 連 table 一起清,`PUT` 重建 schema)。
- wire 格式 camelCase JSON,與 A1 `record.rs`/`relay.rs` 對齊:`idHash`、`kind`、`seq`、`nonce`、`ciphertext`、`deleted`、`baseSeq`、`latestSeq`、`results[].status = "ok" | "conflict"`。
- `relay/` 不進 root `pnpm-workspace.yaml`;有自己的 `package.json`,CI 以獨立 job 測試。
- 測試用的 token / idHash / nonce / ciphertext fixture 必須本身就通過 Worker 的驗證(64 hex、嚴格 base64)。

## Review Focus

1. 兩台裝置對同一 `idHash` 交錯推送 —— 第二次 push 帶舊 `baseSeq` 必須拿到 `conflict` 與最新 envelope,而非靜默覆蓋(Task 1 測試)。
2. `since` 大於 `latestSeq` 或為負/非數字 —— 回空陣列與 400,不可 500(Task 1 測試)。
3. 空 body、非陣列 body、缺欄位、非 base64、含 NUL/非 ASCII、同一批重複 `idHash` —— 400 且訊息不含使用者資料(Task 1 測試)。
4. 同一 chain 用另一個 token 建立 —— 404,且原資料不受影響(Task 1 測試)。
5. 配額臨界:一次 push 讓總量越過 1 MiB —— 整批拒絕 413、不做部分寫入;conflict 的項目不佔配額;tombstone 佔配額;替換既有記錄先扣舊列(Task 1 測試)。
6. `DELETE` 之後同一個 DO instance 再 `PUT` —— 必須是 201 且可正常 push/pull,而不是 missing-table 500(Task 1 測試)。
7. GET/POST/DELETE 一個從未建立的 chain —— 404,而且 `ChainStore.exists()` 之後仍為 false(沒有建 schema)(Task 1 測試)。
8. 同一 IP 一小時內第 21 次建鏈、第 1201 次請求 —— 429(Task 1 `ip-limit.test.ts`)。
9. body 超過 1 MiB(有或沒有 `Content-Length`)—— 413,且不嘗試 JSON 解析(Task 1 測試)。

---

### Task 1: Worker + Durable Objects + 測試

**Files:**
- Create: `relay/package.json`
- Create: `relay/wrangler.jsonc`
- Create: `relay/tsconfig.json`
- Create: `relay/test/tsconfig.json`
- Create: `relay/vitest.config.ts`
- Create: `relay/test/env.d.ts`
- Create: `relay/src/index.ts`
- Create: `relay/worker-configuration.d.ts`(由 `npx wrangler types` 產生並 commit)
- Create: `relay/test/relay.test.ts`
- Create: `relay/test/ip-limit.test.ts`
- Create: `relay/README.md`
- Modify: `.gitignore`(加 `relay/node_modules`、`relay/.wrangler`)

**Interfaces:**
- Produces(HTTP,皆需 `Authorization: Bearer <64 hex>`):
  - `PUT /v1/chains/:chainId` → `201 {}`(新建)/ `200 {}`(已存在且 token 相符)/ `404`(token 不符)/ `429`(此 IP 一小時內已送 20 次建鏈請求或 1200 次任何請求;或此 chain 一分鐘內已 120 次)
  - `POST /v1/chains/:chainId/records` body `PushItem[]`(1–200 筆)→ `200 { results: ({status:"ok",seq}|{status:"conflict",current:Envelope})[], latestSeq }`;`400` 格式錯;`413` 單筆/配額/body 過大;`429` 速率
  - `GET /v1/chains/:chainId/records?since=N` → `200 { records: Envelope[], latestSeq }`
  - `DELETE /v1/chains/:chainId` → `204`
- Produces(TypeScript):`export class ChainStore`(RPC:`exists`、`create`、`pull`、`push`、`destroy`)、`export class IpLimiter`(RPC:`allow(bucket)`)、`export interface Envelope`、`export interface PushItem`;綁定名 `CHAIN`、`IP_LIMIT`。

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
    "types": "wrangler types",
    "test": "vitest run",
    "typecheck": "tsc --noEmit -p tsconfig.json && tsc --noEmit -p test/tsconfig.json"
  },
  "devDependencies": {
    "@cloudflare/vitest-plugin": "^1.2.8",
    "typescript": "^5.9.3",
    "vitest": "^4.1.11",
    "wrangler": "^4.141.0"
  }
}
```

(`@cloudflare/vitest-plugin` 的 peer dependency 是 `vitest ^4.1.0`;**不要**裝 vitest 5。不裝 `@cloudflare/workers-types`:執行期型別由 `wrangler types` 產生。)

建立 `relay/wrangler.jsonc`:

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "sshelter-relay",
  "main": "src/index.ts",
  // ≥ 2026-02-24:deleteAll() 連 alarm 一起刪,destroy/alarm 不用再各自 deleteAlarm()。
  "compatibility_date": "2026-08-20",
  "durable_objects": {
    "bindings": [
      { "name": "CHAIN", "class_name": "ChainStore" },
      { "name": "IP_LIMIT", "class_name": "IpLimiter" }
    ]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["ChainStore", "IpLimiter"] }],
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
    "types": [],
    "lib": ["ES2022"]
  },
  "include": ["src/**/*.ts", "worker-configuration.d.ts"]
}
```

建立 `relay/test/tsconfig.json`(官方作法:測試用 `@cloudflare/vitest-plugin/types` 取得 `cloudflare:test` / `cloudflare:workers` 的型別):

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "moduleResolution": "bundler",
    "types": ["@cloudflare/vitest-plugin/types"]
  },
  "include": ["./**/*.ts", "../worker-configuration.d.ts"]
}
```

建立 `relay/vitest.config.ts`:

```ts
import { cloudflareTest } from "@cloudflare/vitest-plugin";
import { defineConfig } from "vitest/config";

export default defineConfig({
  plugins: [cloudflareTest({ wrangler: { configPath: "./wrangler.jsonc" } })],
});
```

建立 `relay/test/env.d.ts`(`Env` 是 `wrangler types` 產生的全域介面):

```ts
declare module "cloudflare:workers" {
  interface ProvidedEnv extends Env {}
}
```

在 root `.gitignore` 末尾加:

```
relay/node_modules
relay/.wrangler
```

Run: `cd relay && npm install`
Expected: 安裝成功(第一次會拉 workerd,約一分鐘)。`npm ls vitest` 顯示 4.1.x。

- [ ] **Step 2: 寫失敗的測試**

建立 `relay/test/relay.test.ts`(每個測試用自己的 chain id,不依賴測試間的儲存隔離語意;所有 fixture 都通過 Worker 的驗證):

```ts
import { env, exports } from "cloudflare:workers";
import { beforeEach, describe, expect, it } from "vitest";

const TOKEN = "b".repeat(64);
const OTHER_TOKEN = "c".repeat(64);
const base = "https://relay.test";
const SMALL = "YQ=="; // base64("a"),4 字元
const NONCE = "bm9uY2U="; // base64("nonce"),8 字元

/** 64-hex chain id,每個測試一條新 chain。 */
function newChainId(): string {
  const bytes = crypto.getRandomValues(new Uint8Array(32));
  return Array.from(bytes, (b) => b.toString(16).padStart(2, "0")).join("");
}

/** 64-hex idHash fixture:必須通過 Worker 自己的格式驗證。 */
function hid(n: number): string {
  return n.toString(16).padStart(64, "0");
}

function auth(token = TOKEN): HeadersInit {
  return { authorization: `Bearer ${token}`, "content-type": "application/json", "cf-connecting-ip": "203.0.113.7" };
}

function relay(path: string, init?: RequestInit) {
  return exports.default.fetch(new Request(`${base}${path}`, init));
}

function item(idHash: string, baseSeq = 0, ciphertext = "Y2lwaGVy", deleted = false) {
  return { idHash, kind: "host", nonce: NONCE, ciphertext, deleted, baseSeq };
}

let chain = "";
beforeEach(() => {
  chain = newChainId();
});

const create = (token = TOKEN, id = chain) => relay(`/v1/chains/${id}`, { method: "PUT", headers: auth(token), body: "{}" });
const push = (items: unknown, token = TOKEN, id = chain) =>
  relay(`/v1/chains/${id}/records`, { method: "POST", headers: auth(token), body: JSON.stringify(items) });
const pull = (since: string | number, token = TOKEN, id = chain) => relay(`/v1/chains/${id}/records?since=${since}`, { headers: auth(token) });
const destroy = (token = TOKEN, id = chain) => relay(`/v1/chains/${id}`, { method: "DELETE", headers: auth(token) });

type PushBody = { results: { status: string; seq?: number; current?: { seq: number; ciphertext: string; deleted: boolean } }[]; latestSeq: number };
type PullBody = { records: { idHash: string; seq: number; ciphertext: string; deleted: boolean }[]; latestSeq: number };

describe("chain lifecycle", () => {
  it("creates once, then reports existing for the same token", async () => {
    expect((await create()).status).toBe(201);
    expect((await create()).status).toBe(200);
  });

  it("hides the chain from a different token and leaves its data alone", async () => {
    await create();
    await push([item(hid(1))]);
    expect((await create(OTHER_TOKEN)).status).toBe(404);
    expect((await pull(0, OTHER_TOKEN)).status).toBe(404);
    expect((await push([item(hid(2))], OTHER_TOKEN)).status).toBe(404);
    expect((await destroy(OTHER_TOKEN)).status).toBe(404);
    const mine = (await (await pull(0)).json()) as PullBody;
    expect(mine.records.map((r) => r.idHash)).toEqual([hid(1)]);
  });

  it("rejects missing or malformed auth, unknown chains and bad ids", async () => {
    expect((await relay(`/v1/chains/${chain}/records?since=0`)).status).toBe(404);
    expect((await relay(`/v1/chains/${chain}/records?since=0`, { headers: { authorization: "Bearer not-hex" } })).status).toBe(404);
    expect((await pull(0, TOKEN, newChainId())).status).toBe(404);
    expect((await pull(0, TOKEN, "not-hex")).status).toBe(400);
    expect((await relay("/v2/nope", { headers: auth() })).status).toBe(404);
  });

  it("does not allocate storage for chains that were never created", async () => {
    const id = newChainId();
    expect((await pull(0, TOKEN, id)).status).toBe(404);
    expect((await push([item(hid(1))], TOKEN, id)).status).toBe(404);
    expect((await destroy(TOKEN, id)).status).toBe(404);
    // 讀過三次,DO 仍然沒有 schema(沒寫任何東西 → 關閉後就不存在)。
    expect(await env.CHAIN.getByName(id).exists()).toBe(false);
    expect((await create(TOKEN, id)).status).toBe(201);
    expect(await env.CHAIN.getByName(id).exists()).toBe(true);
  });

  it("deletes everything and lets the same instance be created again", async () => {
    await create();
    await push([item(hid(1))]);
    expect((await destroy()).status).toBe(204);
    // 同一個 DO instance(同名)之後的請求不可變成 missing-table 500。
    expect((await pull(0)).status).toBe(404);
    expect(await env.CHAIN.getByName(chain).exists()).toBe(false);
    expect((await create()).status).toBe(201);
    const r = (await (await push([item(hid(2))])).json()) as PushBody;
    expect(r.results[0]).toEqual({ status: "ok", seq: 1 });
    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records.map((x) => x.idHash)).toEqual([hid(2)]);
  });
});

describe("records", () => {
  it("assigns increasing seqs and pulls since a cursor", async () => {
    await create();
    const r1 = (await (await push([item(hid(1)), item(hid(2))])).json()) as PushBody;
    expect(r1.results.map((r) => r.status)).toEqual(["ok", "ok"]);
    expect(r1.results.map((r) => r.seq)).toEqual([1, 2]);
    expect(r1.latestSeq).toBe(2);

    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records.map((r) => r.idHash)).toEqual([hid(1), hid(2)]);
    const later = (await (await pull(1)).json()) as PullBody;
    expect(later.records.map((r) => r.idHash)).toEqual([hid(2)]);
    const none = (await (await pull(99)).json()) as PullBody;
    expect(none.records).toEqual([]);
    expect(none.latestSeq).toBe(2);
  });

  it("reports a conflict when baseSeq is stale and returns the current envelope", async () => {
    await create();
    await push([item(hid(1))]); // seq 1
    await push([item(hid(1), 1, "bmV3")]); // seq 2, from device A
    const stale = (await (await push([item(hid(1), 1, "b2xk")])).json()) as PushBody;
    expect(stale.results[0].status).toBe("conflict");
    expect(stale.results[0].current?.seq).toBe(2);
    expect(stale.results[0].current?.ciphertext).toBe("bmV3");
    expect(stale.latestSeq).toBe(2);
    // 帶正確 baseSeq 重送就成功。
    const ok = (await (await push([item(hid(1), 2, "b2xk")])).json()) as PushBody;
    expect(ok.results[0]).toEqual({ status: "ok", seq: 3 });
  });

  it("keeps tombstones pullable and counts them toward the quota", async () => {
    await create();
    await push([item(hid(1), 0, "x".repeat(1000))]);
    await push([item(hid(1), 1, "dG9tYg==", true)]);
    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records).toHaveLength(1);
    expect(all.records[0].deleted).toBe(true);
    expect(all.records[0].ciphertext).toBe("dG9tYg==");
    // 既有的 tombstone 佔 8 + 8 = 16 bytes;15 筆 64 KiB 把用量吃到 16 + 983,160 = 983,176;
    // 再來一筆 64 KiB 的 tombstone → 1,048,720 > 1 MiB,一樣越界。
    const big = "y".repeat(65_536);
    const fifteen = Array.from({ length: 15 }, (_, i) => item(hid(100 + i), 0, big));
    expect((await push(fifteen)).status).toBe(200);
    expect((await push([item(hid(200), 0, big, true)])).status).toBe(413);
  });

  it("validates bodies without leaking them", async () => {
    await create();
    const bad = [
      "",
      "{}",
      "[]",
      "[1]",
      JSON.stringify([{ idHash: "h" }]),
      JSON.stringify([item("nothex")]),
      JSON.stringify([item(hid(1), 0, "abc")]), // 長度不是 4 的倍數
      JSON.stringify([item(hid(1), 0, "AA\u0000A")]), // NUL:SQLite LENGTH() 會在這裡停
      JSON.stringify([item(hid(1), 0, "Y2lw中")]), // 非 ASCII
      JSON.stringify([item(hid(1), 0, "")]), // 空密文
      JSON.stringify([{ ...item(hid(1)), nonce: "not base64!" }]),
    ];
    for (const body of bad) {
      const res = await relay(`/v1/chains/${chain}/records`, { method: "POST", headers: auth(), body });
      expect(res.status, body).toBe(400);
      expect(await res.text()).not.toContain('idHash":');
    }
    // 同一批重複 idHash:拒絕(否則第二筆的 baseSeq 判定會依賴第一筆的寫入順序)。
    expect((await push([item(hid(1)), item(hid(1))])).status).toBe(400);
    // 超過 200 筆。
    expect((await push(Array.from({ length: 201 }, (_, i) => item(hid(i))))).status).toBe(400);
    expect((await pull("abc")).status).toBe(400);
    expect((await pull(-1)).status).toBe(400);
  });

  it("refuses oversize records, oversize bodies and chain quota overflow atomically", async () => {
    await create();
    expect((await push([item(hid(1), 0, "z".repeat(65_540))])).status).toBe(413);
    // body 超過 1 MiB:串流讀到上限就取消,不解析。分別測有宣告長度(string body)與沒有(stream body)。
    const oversized = "[" + "x".repeat(1_048_600);
    expect((await relay(`/v1/chains/${chain}/records`, { method: "POST", headers: auth(), body: oversized })).status).toBe(413);
    const stream = new Blob([oversized]).stream();
    expect((await relay(`/v1/chains/${chain}/records`, { method: "POST", headers: auth(), body: stream, duplex: "half" } as RequestInit)).status).toBe(413);
    // 15 × (65536 + 8) = 983,160 可以;第 16 筆會到 1,048,704 > 1 MiB → 整批 413,原本 15 筆都在。
    const big = "y".repeat(65_536);
    const fifteen = Array.from({ length: 15 }, (_, i) => item(hid(i), 0, big));
    expect((await push(fifteen)).status).toBe(200);
    expect((await push([item(hid(15), 0, big), item(hid(16), 0, SMALL)])).status).toBe(413);
    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records).toHaveLength(15);
  });

  it("charges replacements net of the old row and never charges conflicting items", async () => {
    await create();
    const big = "y".repeat(65_536);
    const fifteen = Array.from({ length: 15 }, (_, i) => item(hid(i), 0, big));
    await push(fifteen); // seq 1..15,用量 983,160
    // 替換 hid(0)(baseSeq 正確):先扣舊列再加新列,總量不變 → 接受。
    expect((await push([item(hid(0), 1, big)])).status).toBe(200);
    // hid(1) 帶過期 baseSeq → conflict(不寫入、不佔配額);hid(99) 很小 → 接受。
    const r = (await (await push([item(hid(1), 0, big), item(hid(99), 0, SMALL)])).json()) as PushBody;
    expect(r.results.map((x) => x.status)).toEqual(["conflict", "ok"]);
    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records).toHaveLength(16);
  });

  it("caps the number of records per chain", async () => {
    await create();
    for (let batch = 0; batch < 20; batch++) {
      const items = Array.from({ length: 200 }, (_, i) => item(hid(batch * 200 + i), 0, SMALL));
      expect((await push(items)).status).toBe(200);
    }
    // 4000 筆之後再 100 筆 → 4100 > 4096:整批 413。
    expect((await push(Array.from({ length: 100 }, (_, i) => item(hid(5000 + i), 0, SMALL)))).status).toBe(413);
  });

  it("rate-limits a chain to 120 requests per minute on every endpoint", async () => {
    await create(); // 第 1 次(建鏈那一次也計數)
    for (let i = 0; i < 119; i++) {
      expect((await pull(0)).status).toBe(200);
    }
    // 第 121 次起,已授權的任何端點一律 429(不是 404、不是靜默成功)。
    expect((await pull(0)).status).toBe(429);
    expect((await create()).status).toBe(429);
    expect((await push([item(hid(1))])).status).toBe(429);
    expect((await destroy()).status).toBe(429);
  });
});
```

建立 `relay/test/ip-limit.test.ts`(獨立檔案:每 IP 計數是跨測試累積的,放在自己的儲存隔離範圍裡;總請求上限直接打 DO 的 RPC,不用真的送 1200 個 HTTP 請求):

```ts
import { env, exports } from "cloudflare:workers";
import { expect, it } from "vitest";

const TOKEN = "d".repeat(64);
const IP = "198.51.100.9";

function newChainId(): string {
  const bytes = crypto.getRandomValues(new Uint8Array(32));
  return Array.from(bytes, (b) => b.toString(16).padStart(2, "0")).join("");
}

function create(id: string) {
  return exports.default.fetch(
    new Request(`https://relay.test/v1/chains/${id}`, {
      method: "PUT",
      headers: { authorization: `Bearer ${TOKEN}`, "content-type": "application/json", "cf-connecting-ip": IP },
      body: "{}",
    }),
  );
}

// 建鏈桶計的是 PUT 嘗試(不論 201/200/404),不是成功建立的 chain 數(spec §5)。
it("allows 20 chain-creation requests per IP per hour, then answers 429", async () => {
  const first = newChainId();
  expect((await create(first)).status).toBe(201);
  for (let i = 1; i < 20; i++) {
    expect((await create(newChainId())).status).toBe(201);
  }
  expect((await create(newChainId())).status).toBe(429);
  // 已建立的 chain 不受建鏈限制影響(GET 只計入總請求桶)。
  const pull = await exports.default.fetch(
    new Request(`https://relay.test/v1/chains/${first}/records?since=0`, {
      headers: { authorization: `Bearer ${TOKEN}`, "cf-connecting-ip": IP },
    }),
  );
  expect(pull.status).toBe(200);
});

it("allows 1200 requests per IP per hour on the request bucket", async () => {
  const limiter = env.IP_LIMIT.getByName("ip:unit-test");
  for (let i = 0; i < 1200; i++) {
    expect(await limiter.allow("request")).toBe(true);
  }
  expect(await limiter.allow("request")).toBe(false);
  // 桶彼此獨立:建鏈桶還沒用過。
  expect(await limiter.allow("create")).toBe(true);
});
```

- [ ] **Step 3: 執行測試確認失敗**

Run: `cd relay && npx vitest run 2>&1 | tail -5`
Expected: 失敗 —— 找不到 `src/index.ts`(Worker 的 `main`)。

- [ ] **Step 4: 實作 Worker 與 DO**

建立 `relay/src/index.ts`:

```ts
import { DurableObject } from "cloudflare:workers";

const MAX_RECORD_BYTES = 65_536; // 單筆 ciphertext 字元數(嚴格 base64 → 字元 = byte)
const MAX_CHAIN_BYTES = 1_048_576; // 所有列(含 tombstone)的 ciphertext + nonce 長度總和
const MAX_CHAIN_RECORDS = 4096;
const MAX_BODY_BYTES = 1_048_576;
const MAX_ITEMS_PER_PUSH = 200;
const RATE_LIMIT_PER_MINUTE = 120;
const IDLE_TTL_MS = 180 * 24 * 60 * 60 * 1000;
const TOUCH_INTERVAL_MS = 6 * 60 * 60 * 1000;
const IP_WINDOW_MS = 60 * 60 * 1000;
const HEX64 = /^[0-9a-f]{64}$/;
const BASE64 = /^[A-Za-z0-9+/]*={0,2}$/;

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

interface Row {
  id_hash: string;
  kind: string;
  seq: number;
  nonce: string;
  ciphertext: string;
  deleted: number;
}

const SCHEMA = `
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
`;

const ROW_COLUMNS = "id_hash, kind, seq, nonce, ciphertext, deleted";

function toEnvelope(r: Row): Envelope {
  return { idHash: r.id_hash, kind: r.kind, seq: r.seq, nonce: r.nonce, ciphertext: r.ciphertext, deleted: r.deleted === 1 };
}

/** 每個 chain 一個 DO:token hash、記錄與序號全在它的 SQLite 裡。 */
export class ChainStore extends DurableObject<Env> {
  private minuteStart = 0;
  private minuteCount = 0;
  private lastTouch = 0;

  /**
   * chain 是否已建立 = `meta` 表存在。只讀 `sqlite_master`,不寫任何東西:讀一個從未建立的 chain
   * 不能留下持久化的空資料庫(沒寫過儲存的 DO 在關閉後就不存在)。schema 只在 `create()` 建。
   */
  async exists(): Promise<boolean> {
    return this.hasSchema();
  }

  private hasSchema(): boolean {
    return this.ctx.storage.sql.exec("SELECT name FROM sqlite_master WHERE type = 'table' AND name = 'meta'").toArray().length > 0;
  }

  private meta(key: string): string | null {
    const row = this.ctx.storage.sql.exec<{ value: string }>("SELECT value FROM meta WHERE key = ?", key).toArray()[0];
    return row ? row.value : null;
  }

  private setMeta(key: string, value: string): void {
    this.ctx.storage.sql.exec(
      "INSERT INTO meta (key, value) VALUES (?, ?) ON CONFLICT(key) DO UPDATE SET value = excluded.value",
      key,
      value,
    );
  }

  /** 已建立且 token 相符。不存在與不符都回 false(呼叫端一律 404)。 */
  private authorized(tokenHash: string): boolean {
    if (!this.hasSchema()) return false;
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

  private row(idHash: string): Row | undefined {
    return this.ctx.storage.sql.exec<Row>(`SELECT ${ROW_COLUMNS} FROM records WHERE id_hash = ?`, idHash).toArray()[0];
  }

  /** 目前用量:所有列(含 tombstone)的 ciphertext + nonce 長度總和與筆數。 */
  private usage(): { bytes: number; count: number } {
    const r = this.ctx.storage.sql
      .exec<{ bytes: number; count: number }>(
        "SELECT COALESCE(SUM(LENGTH(ciphertext) + LENGTH(nonce)), 0) AS bytes, COUNT(*) AS count FROM records",
      )
      .one();
    return { bytes: Number(r.bytes), count: Number(r.count) };
  }

  /** 刷新閒置期限。寫入一律刷新;讀取節流(同一 instance 每 6 小時一次),唯讀裝置的 chain 才不會被清掉。 */
  private async touch(force: boolean): Promise<void> {
    const now = Date.now();
    if (!force && now - this.lastTouch < TOUCH_INTERVAL_MS) return;
    this.lastTouch = now;
    await this.ctx.storage.setAlarm(now + IDLE_TTL_MS);
  }

  /** 201 新建、200 已存在、404 token 不符、429 超過每 chain 速率。建鏈的每 IP 限制在 Worker 層。 */
  async create(tokenHash: string): Promise<201 | 200 | 404 | 429> {
    if (!this.hasSchema()) {
      this.ctx.storage.sql.exec(SCHEMA);
      this.setMeta("token_hash", tokenHash);
      this.setMeta("latest_seq", "0");
      this.setMeta("created_at", String(Date.now()));
      this.rateLimited(); // 建鏈那一次也算進每 chain 的請求數(第 1 次,不可能超限)
      await this.touch(true);
      return 201;
    }
    if (this.meta("token_hash") !== tokenHash) return 404;
    if (this.rateLimited()) return 429;
    await this.touch(true);
    return 200;
  }

  async pull(tokenHash: string, since: number): Promise<{ status: 200 | 404 | 429; body?: { records: Envelope[]; latestSeq: number } }> {
    if (!this.authorized(tokenHash)) return { status: 404 };
    if (this.rateLimited()) return { status: 429 };
    const records = this.ctx.storage.sql
      .exec<Row>(`SELECT ${ROW_COLUMNS} FROM records WHERE seq > ? ORDER BY seq`, since)
      .toArray()
      .map(toEnvelope);
    await this.touch(false);
    return { status: 200, body: { records, latestSeq: Number(this.meta("latest_seq") ?? "0") } };
  }

  async push(tokenHash: string, items: PushItem[]): Promise<{ status: 200 | 404 | 413 | 429; body?: { results: PushResult[]; latestSeq: number } }> {
    if (!this.authorized(tokenHash)) return { status: 404 };
    if (this.rateLimited()) return { status: 429 };

    // 1. 先判定每筆是接受還是 conflict —— 不寫入。
    let latest = Number(this.meta("latest_seq") ?? "0");
    const existing = new Map<string, Row>();
    for (const item of items) {
      const current = this.row(item.idHash);
      if (current) existing.set(item.idHash, current);
    }
    const plan: { item: PushItem; result: PushResult }[] = [];
    for (const item of items) {
      const current = existing.get(item.idHash);
      if (current && current.seq > item.baseSeq) {
        plan.push({ item, result: { status: "conflict", current: toEnvelope(current) } });
      } else {
        latest += 1;
        plan.push({ item, result: { status: "ok", seq: latest } });
      }
    }

    // 2. 以「接受之後」的用量檢查配額:conflict 不算;被取代的舊列先扣再加;tombstone 一樣算。
    //    輸入已驗證為嚴格 base64(ASCII),所以 JS `.length` 與 SQLite `LENGTH()` 一致。
    const usage = this.usage();
    let bytes = usage.bytes;
    let count = usage.count;
    for (const { item, result } of plan) {
      if (result.status !== "ok") continue;
      const previous = existing.get(item.idHash);
      if (previous) bytes -= previous.ciphertext.length + previous.nonce.length;
      else count += 1;
      bytes += item.ciphertext.length + item.nonce.length;
    }
    if (bytes > MAX_CHAIN_BYTES || count > MAX_CHAIN_RECORDS) return { status: 413 };

    // 3. 寫入(單一 DO、無 await 穿插 → 整批原子)。
    for (const { item, result } of plan) {
      if (result.status !== "ok") continue;
      this.ctx.storage.sql.exec(
        `INSERT INTO records (${ROW_COLUMNS}) VALUES (?, ?, ?, ?, ?, ?)
         ON CONFLICT(id_hash) DO UPDATE SET kind = excluded.kind, seq = excluded.seq, nonce = excluded.nonce,
         ciphertext = excluded.ciphertext, deleted = excluded.deleted`,
        item.idHash,
        item.kind,
        result.seq,
        item.nonce,
        item.ciphertext,
        item.deleted ? 1 : 0,
      );
    }
    this.setMeta("latest_seq", String(latest));
    await this.touch(true);
    return { status: 200, body: { results: plan.map((p) => p.result), latestSeq: latest } };
  }

  async destroy(tokenHash: string): Promise<204 | 404 | 429> {
    if (!this.authorized(tokenHash)) return 404;
    if (this.rateLimited()) return 429;
    // compatibility_date ≥ 2026-02-24:deleteAll() 連 alarm 一起刪;schema 也沒了,下次 PUT 重建。
    await this.ctx.storage.deleteAll();
    this.lastTouch = 0;
    return 204;
  }

  /** 閒置 180 天:整個 chain 清掉(使用者本機資料不受影響)。儲存清空後 DO 會在關閉時消失。 */
  async alarm(): Promise<void> {
    await this.ctx.storage.deleteAll();
    this.lastTouch = 0;
  }
}

type Bucket = "create" | "request";
const IP_LIMITS: Record<Bucket, number> = { create: 20, request: 1200 };
type Windows = Record<Bucket, { start: number; count: number }>;

/** 每個來源 IP 一個小 DO:一小時視窗內分桶計數(建鏈、所有請求);閒置一小時由 alarm 清空自己。 */
export class IpLimiter extends DurableObject<Env> {
  async allow(bucket: Bucket): Promise<boolean> {
    const now = Date.now();
    const windows: Windows = (await this.ctx.storage.get<Windows>("windows")) ?? {
      create: { start: now, count: 0 },
      request: { start: now, count: 0 },
    };
    const w = windows[bucket];
    if (now - w.start >= IP_WINDOW_MS) {
      w.start = now;
      w.count = 0;
    }
    w.count += 1;
    await this.ctx.storage.put("windows", windows);
    await this.ctx.storage.setAlarm(now + IP_WINDOW_MS);
    return w.count <= IP_LIMITS[bucket];
  }

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

/** 嚴格的標準 base64:非空、ASCII 字母表、長度為 4 的倍數、padding ≤ 2。 */
function isStrictBase64(s: string): boolean {
  return s.length > 0 && s.length % 4 === 0 && BASE64.test(s);
}

function isPushItem(v: unknown): v is PushItem {
  if (typeof v !== "object" || v === null) return false;
  const o = v as Record<string, unknown>;
  return (
    typeof o.idHash === "string" && HEX64.test(o.idHash) &&
    typeof o.kind === "string" && /^[a-z]{1,16}$/.test(o.kind) &&
    typeof o.nonce === "string" && o.nonce.length <= 64 && isStrictBase64(o.nonce) &&
    typeof o.ciphertext === "string" && isStrictBase64(o.ciphertext) &&
    typeof o.deleted === "boolean" &&
    typeof o.baseSeq === "number" && Number.isInteger(o.baseSeq) && o.baseSeq >= 0
  );
}

/** 以 byte 上限讀 body:超過就取消串流回 null,不把整個 body 讀進記憶體再檢查。 */
async function readBodyCapped(request: Request, maxBytes: number): Promise<string | null> {
  const reader = request.body?.getReader();
  if (!reader) return "";
  const chunks: Uint8Array[] = [];
  let total = 0;
  for (;;) {
    const { done, value } = await reader.read();
    if (done) break;
    total += value.byteLength;
    if (total > maxBytes) {
      await reader.cancel();
      return null;
    }
    chunks.push(value);
  }
  const bytes = new Uint8Array(total);
  let offset = 0;
  for (const chunk of chunks) {
    bytes.set(chunk, offset);
    offset += chunk.byteLength;
  }
  return new TextDecoder().decode(bytes);
}

const bad = (message: string) => Response.json({ error: message }, { status: 400 });
const notFound = () => new Response(null, { status: 404 });
const status = (code: number) => new Response(null, { status: code });

export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const url = new URL(request.url);
    const match = /^\/v1\/chains\/([^/]+)(\/records)?$/.exec(url.pathname);
    if (!match) return notFound();
    const [, chainId, recordsPath] = match;
    if (!HEX64.test(chainId)) return bad("invalid chain id");
    const token = bearer(request);
    if (!token) return notFound();

    // 每 IP 限制(Cloudflare 在邊緣覆寫 CF-Connecting-IP,客戶端偽造不了;本機測試自己帶 header)。
    // 任何請求都會啟動一個 chain DO(即使不存在),所以總請求也要限;建鏈另外計一桶。
    const ip = request.headers.get("cf-connecting-ip") ?? "unknown";
    const limiter = env.IP_LIMIT.getByName(`ip:${ip}`);
    if (!(await limiter.allow("request"))) return status(429);
    const isCreate = !recordsPath && request.method === "PUT";
    if (isCreate && !(await limiter.allow("create"))) return status(429);

    const tokenHash = await sha256Hex(token);
    const stub = env.CHAIN.getByName(chainId);

    if (isCreate) {
      const code = await stub.create(tokenHash);
      if (code === 404) return notFound();
      if (code === 429) return status(429);
      return Response.json({}, { status: code });
    }
    if (!recordsPath && request.method === "DELETE") {
      return status(await stub.destroy(tokenHash));
    }
    if (recordsPath && request.method === "GET") {
      const raw = url.searchParams.get("since") ?? "0";
      if (!/^\d{1,18}$/.test(raw)) return bad("invalid since");
      const result = await stub.pull(tokenHash, Number(raw));
      if (result.status !== 200) return status(result.status);
      return Response.json(result.body);
    }
    if (recordsPath && request.method === "POST") {
      // 宣告長度只用來提前拒絕;真正的上限由串流讀取保證(沒有 Content-Length 的請求也一樣)。
      const declared = Number(request.headers.get("content-length") ?? "0");
      if (declared > MAX_BODY_BYTES) return status(413);
      const text = await readBodyCapped(request, MAX_BODY_BYTES);
      if (text === null) return status(413);
      let body: unknown;
      try {
        body = JSON.parse(text);
      } catch {
        return bad("body must be JSON");
      }
      if (!Array.isArray(body) || body.length === 0 || body.length > MAX_ITEMS_PER_PUSH || !body.every(isPushItem)) {
        return bad("body must be a non-empty array of records");
      }
      if (new Set(body.map((i) => i.idHash)).size !== body.length) return bad("duplicate idHash in one push");
      if (body.some((i) => i.ciphertext.length > MAX_RECORD_BYTES)) return status(413);
      const result = await stub.push(tokenHash, body);
      if (result.status !== 200) return status(result.status);
      return Response.json(result.body);
    }
    return notFound();
  },
} satisfies ExportedHandler<Env>;
```

接著產生執行期型別與 `Env`(全域介面,含 `CHAIN`/`IP_LIMIT` 綁定):

Run: `cd relay && npx wrangler types`
Expected: 產生 `relay/worker-configuration.d.ts`(要 commit);內含 `interface Env { CHAIN: DurableObjectNamespace<...ChainStore>; IP_LIMIT: DurableObjectNamespace<...IpLimiter>; }` 與該 compatibility date 的 runtime 型別。

- [ ] **Step 5: 執行測試與型別檢查確認通過**

Run: `cd relay && npx vitest run 2>&1 | tail -8 && npm run typecheck`
Expected: 兩個測試檔全部通過(`relay.test.ts` 13 個、`ip-limit.test.ts` 2 個)、兩份 tsconfig 型別檢查無誤。若 `stub.create()` 之類 RPC 呼叫型別報錯,確認 `worker-configuration.d.ts` 已重新產生且 `Env` 綁定型別是 `DurableObjectNamespace<ChainStore>`。若 `exports.default.fetch` 或 `env.CHAIN` 型別報錯,確認 `test/tsconfig.json` 的 `types` 是 `@cloudflare/vitest-plugin/types` 且 `env.d.ts` 的 `ProvidedEnv extends Env`。若串流 body 的測試在 workerd 裡因 `duplex` 選項報錯,改用 `new Request(url, { method: "POST", headers, body: new Blob([oversized]) })`(Blob body 同樣沒有 `Content-Length` 可提前拒絕的保證,仍走串流上限)。

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

Then point SSHelter at your Worker URL: Settings → Sync → Relay URL. The app only accepts
`https://` relays (plain `http://` is allowed for `localhost` / `127.0.0.1` during development).

## Limits

- 64 KiB per record, 1 MiB and 4096 records per chain (tombstones count), 1 MiB per request
- 120 requests/min per chain (any endpoint); per IP: 20 chain-creation requests/hour, 1200 requests/hour
- chains idle for 180 days are deleted automatically (reads count as activity; devices keep their local copies)
- reading a chain that was never created returns 404 and allocates nothing

## Test

    npm test          # runs inside workerd via @cloudflare/vitest-plugin
    npm run typecheck
```

```bash
git add .gitignore relay/package.json relay/package-lock.json relay/wrangler.jsonc relay/tsconfig.json relay/test/tsconfig.json relay/vitest.config.ts relay/test/env.d.ts relay/worker-configuration.d.ts relay/src/index.ts relay/test/relay.test.ts relay/test/ip-limit.test.ts relay/README.md
git commit -m "feat(relay): zero-knowledge sync relay on Cloudflare Durable Objects"
```

---

### Task 2: CI 與部署

**Files:**
- Create: `.github/workflows/relay.yml`
- Modify: `.github/workflows/release.yml`(build job 注入 `SSHELTER_RELAY_URL`)

**Interfaces:**
- Consumes: A1 Task 4 的 `DEFAULT_RELAY_URL = option_env!("SSHELTER_RELAY_URL")`;A1 Task 5 的 client 只接受 `https://`(非 loopback)。
- Produces: GitHub Actions repository variable `SSHELTER_RELAY_URL`(由使用者在部署後設定;必須是 `https://` URL)。

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
      - run: npm run typecheck
      - run: npm test
```

- [ ] **Step 2: release build 注入正式中繼 URL**

在 `.github/workflows/release.yml` 的 `Build and upload installers to the release` 步驟 `env:` 區塊加入一行:

```yaml
          # 正式中繼 URL(repository variable,必須是 https://);未設定時 app 預設指向本機 wrangler dev。
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

Expected: 輸出 `https://sshelter-relay.<account>.workers.dev`。接著設定 repository variable(**必須是 https://**,app 端會拒絕非 loopback 的 http):

```bash
gh variable set SSHELTER_RELAY_URL --repo ysya/sshelter --body "https://sshelter-relay.<account>.workers.dev"
```

之後的 release build 就會內建正式中繼;本機 `pnpm tauri dev` 仍預設 `http://127.0.0.1:8787`(搭配 `cd relay && npm run dev`)。

---

## Self-review(已執行)

- **Spec 覆蓋**:§5 四個端點、PUT 只給 Create、token hash、404 不洩露且不配置儲存、嚴格 base64、配額(64 KiB / 1 MiB 含 tombstone / 4096 筆 / body 1 MiB 串流上限 / 200 筆)、每 chain 速率、每 IP 建鏈與總請求限制、180 天閒置清除且 pull 刷新、delete 後可重建、可自架 → Task 1;CI 與正式 URL 注入 → Task 2。
- **型別一致**:wire 欄位 `idHash/kind/seq/nonce/ciphertext/deleted/baseSeq/latestSeq/results[].status` 與 A1 `record.rs::Envelope`、`relay.rs::PushItem` 的 serde `rename_all = "camelCase"` 結構完全對應;`status: "ok"` 對應 Rust `WirePushResult::Ok`(serde `rename_all = "lowercase"` 產生 `"ok"`);A1 client 送的 nonce/ciphertext 是 `base64::STANDARD`(含 padding)→ 通過這裡的嚴格驗證;A1 client `push` 送的是裸陣列 body,與這裡的 `Array.isArray(body)` 一致。
- **測試數字**:15 × (65,536 + 8) = 983,160 ≤ 1,048,576;16 × 65,544 = 1,048,704 > 1,048,576;`SMALL`(4)+ nonce(8)= 12;4000 × 12 = 48,000;`"z".repeat(65_540)` 是 4 的倍數(通過 base64 驗證)且 > 65,536 → 413。
- **Review Focus 對應**:1 → `reports a conflict when baseSeq is stale`;2 → `validates bodies…`(since=abc/-1)與 `assigns increasing seqs…`(since=99);3 → `validates bodies…`(含非 base64、NUL、非 ASCII、重複 idHash、201 筆);4 → `hides the chain from a different token…`;5 → `refuses oversize records…`、`charges replacements net of the old row…`、`keeps tombstones…`;6 → `deletes everything and lets the same instance be created again`;7 → `does not allocate storage for chains that were never created`;8 → `ip-limit.test.ts`;9 → `refuses oversize records, oversize bodies…`(string 與 stream body)。
- **Codex review(2026-09-27,兩輪)已納入**:配額在判定接受之後計算、tombstone 與 nonce 計入、記錄數上限、body 大小串流上限、每 IP 建鏈與總請求限制(finding 4、H1、M6);`deleteAll()` 後 schema 重建與讀不存在的 chain 不建 schema(18、H1);嚴格 base64 讓 JS/SQLite 長度一致(H2);pull 刷新 TTL(M7);fixture 改成合法 hex/base64(19);測試型別改用官方 `@cloudflare/vitest-plugin/types` 與 `wrangler types`(20);測試數量 13 + 2(L1)。
- **Codex 第三輪已納入**:建鏈那一次也計入每 chain 速率、已授權超限一律 429(含 PUT/DELETE)(R3-M2);建鏈桶明確定義為「PUT 嘗試次數」並同步 spec/README/測試用語(R3-M3);tombstone 配額測試註解補上既有 16 bytes(R3-L2);`duplex: "half"` 在 workerd 會被忽略,串流 body 測試不依賴它(只是為了 Node 型別/相容而保留)。
- **未採用 Workers Rate Limiting binding 的原因**:它的 `period` 只能是 10 或 60 秒、per-colo 且 best-effort,官方文件與 Miniflare README 都沒有說明本機/測試支援;DO 計數器可精確表達「每小時 N 次」且能在 vitest 裡驗證。
