# Sync v2 — B1(relay:批次查詢、凍結、版本資訊、更新已部署的 relay)Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** relay 新增 `POST /v1/pull`(一次查多條 chain)、`POST /v1/chains/{id}/freeze`(凍結,供更換同步碼)與
`GET /v1/info`(版本與功能),並提供「Update relay」workflow 讓用部署按鈕建立的 relay 能更新。既有端點與行為完全不變。

**Architecture:** 全部在 `relay/src/index.ts`:Worker router 加兩條新路徑與 `/freeze` 後綴;`ChainStore` 加 `freeze()` 與
「凍結後 push 一律 409」;`IpLimiter` 加 `pull` 桶並讓 `allow()` 接受成本參數(相容舊的儲存格式)。批次查詢依序呼叫每條
chain 的既有 `pull()`,以序列化後的 JSON 長度計算回應預算。更新 workflow 放在 `relay/.github/`(部署按鈕複製 `relay/` 時
一起帶到使用者 repo;在本 repo 不會被執行),邏輯放在無相依套件的 Node 腳本,以 `node --test` 測試;workflow 只開 PR。

**Tech Stack:** Cloudflare Workers + Durable Objects(SQLite)、TypeScript、`@cloudflare/vitest-plugin`、Node 22 內建模組
(`node:test`、`node:fs`)、GitHub Actions、`gh` CLI。

**Spec:** `docs/superpowers/specs/2026-10-02-sync-v2-spaces-design.md` §6(relay)、§12(審查紀錄 #1、#10、#11、#12)

> **執行後註記(2026-10-02)**:本計畫已以 Subagent-driven 執行(29bf6c1..8057054,加上最終審查的修正)。下列內容
> 已被取代,以 spec §6 與 §13 為準:
> - Task 1 的 `export const RELAY_VERSION` → 模組內私有常數(workerd 不接受主模組的非 handler export)。
> - Task 1 的兩次 `limiter.allow()` → 每個請求一次 `allowMany()`;讀取刷新閒置期限改以 alarm 時間節流。
> - Task 2 與 Review Focus 3「刪除後重建是全新、未凍結的 chain」→ 凍結的 chain 刪除時只清記錄、維持凍結
>   (`PUT` 回 `200`、push 回 `409`)。
> - Task 3 的 workflow:不再覆寫 workflow 檔(只比較並提示);`.github/` 只管理 `scripts/` 的兩個檔案;拆成兩個 job
>   (測試沒有寫入權限、開 PR 的 job 不執行上游程式碼);不降版;同一個上游 commit 已有任何狀態的 PR 就不再開;
>   relay 的 vitest 排除 `.github/**`。

## Global Constraints

- relay 程式碼的註解用繁體中文(沿用 `relay/src/index.ts`);識別字、錯誤訊息、JSON 欄位用英文。`relay/selfhost/` 與
  `relay/.github/` 的工具腳本沿用英文註解。
- 既有端點(`PUT/DELETE /v1/chains/{id}`、`GET/POST /v1/chains/{id}/records`)與既有測試**不得修改行為**;既有測試必須原樣通過。
- relay 不新增 npm 相依套件;更新腳本只用 Node 內建模組(Node ≥ 20)。
- GitHub Actions 的 `run:` 內不得出現 `${{ }}`;輸入一律經 `env:` 傳入。
- Conventional Commits;只 stage 明確列出的路徑;不得 stage `.superpowers/`。
- 驗證指令:`cd relay && npm run typecheck && npm test`;`node --test relay/.github/scripts/update-relay.test.mjs`;
  `actionlint .github/workflows/relay.yml relay/.github/workflows/update-relay.yml`(本機有安裝時)。
- 數值(照抄 spec):批次每次 1–64 項、body 上限 64 KiB、回應預算 2 MiB(2,097,152)、`pull` 桶每 IP 每小時 12000、
  `RELAY_VERSION = "0.2.0"`、features `["pull-batch", "freeze"]`。

## Review Focus

1. 現有 relay 上已存在的 `IpLimiter` 儲存沒有 `pull` 欄位 —— 升級後第一次批次查詢不能壞掉,要從 0 開始計(Task 1 測試
   `starts the pull bucket for limiters stored by an older relay`)。
2. client 送大寫 hex 的 chain 或權杖 —— 批次端點只接受小寫,必須回 `400` 而不是 `not_found`,錯誤才看得出來(Task 1 測試
   `rejects malformed batches`)。
3. 凍結的 chain 被刪除後又以同一組權杖建立 —— 是一條全新的、未凍結的 chain(Task 2 測試
   `keeps a frozen chain readable and deletable; a re-created chain starts unfrozen`)。
4. 使用者 repo 的 `wrangler.jsonc` 已改過 Worker 名稱,`.github/` 裡還有自己的 workflow —— 更新後名稱保留、自己的檔案
   不被刪(Task 3 測試 `keeps the Worker name` 與 `never touches the owner's other GitHub files`)。
5. GitHub Actions 沒有被允許開 PR —— workflow 失敗時要印出可手動開 PR 的網址,而不是只有權限錯誤(Task 3 workflow 步驟)。

---

### Task 1: 批次查詢 `POST /v1/pull`、`pull` 桶、`GET /v1/info`

**Files:**
- Modify: `relay/src/index.ts`(router、新的 `batchPull` / `info` 處理、`IpLimiter`)
- Modify: `relay/package.json`(`version` → `0.2.0`)
- Modify: `relay/test/tsconfig.json`(`resolveJsonModule`)
- Create: `relay/test/batch.test.ts`
- Modify: `relay/test/ip-limit.test.ts`
- Modify: `relay/selfhost/smoke-test.mjs`(write 階段加批次查詢與 `/v1/info`)
- Modify: `relay/README.md`(API 與限制)

**Interfaces:**
- Produces(B2 的 relay client 依此實作):
  - `POST /v1/pull`,無 `Authorization`;body `Array<{ chain: string; token: string; since: number }>`(1–64 項;
    `chain`、`token` 為 64 字元小寫 hex;`since` 為非負 safe integer;`chain` 不重複)。成功 `200 { results: PullEntry[] }`,
    順序同請求,`PullEntry` = `{ chain, status: "ok", records: Envelope[], latestSeq }` |
    `{ chain, status: "not_found" | "rate_limited" | "deferred" }`。格式錯誤 `400 { error }`;body 過大 `413`;
    `request` 桶或 `pull` 桶不足 `429`。
  - `GET /v1/info` → `200 { relay: "sshelter-relay", version: "0.2.0", features: ["pull-batch", "freeze"] }`;其他方法 `404`。
  - `export const RELAY_VERSION = "0.2.0"`(`relay/src/index.ts`)。
  - `IpLimiter.allow(bucket: "create" | "request" | "pull", cost = 1): Promise<boolean>`。

- [ ] **Step 1: 寫失敗的測試**

`relay/test/tsconfig.json` 的 `compilerOptions` 加上 `"resolveJsonModule": true`:

```json
{
  "extends": "../tsconfig.json",
  "compilerOptions": {
    "moduleResolution": "bundler",
    "types": ["@cloudflare/vitest-plugin/types"],
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["./**/*.ts", "../worker-configuration.d.ts"]
}
```

建立 `relay/test/batch.test.ts`:

```ts
import { env, exports } from "cloudflare:workers";
import { describe, expect, it } from "vitest";

import pkg from "../package.json";
import { RELAY_VERSION } from "../src/index";

const TOKEN = "b".repeat(64);
const OTHER_TOKEN = "c".repeat(64);
const NONCE = "bm9uY2U="; // base64("nonce")

function newChainId(): string {
  const bytes = crypto.getRandomValues(new Uint8Array(32));
  return Array.from(bytes, (b) => b.toString(16).padStart(2, "0")).join("");
}

function hid(n: number): string {
  return n.toString(16).padStart(64, "0");
}

function request(path: string, init: RequestInit = {}, ip = "203.0.113.20") {
  const headers = new Headers(init.headers);
  headers.set("cf-connecting-ip", ip);
  return exports.default.fetch(new Request(`https://relay.test${path}`, { ...init, headers }));
}

const create = (id: string, token = TOKEN) =>
  request(`/v1/chains/${id}`, { method: "PUT", headers: { authorization: `Bearer ${token}` }, body: "{}" });
const push = (id: string, items: unknown[], token = TOKEN) =>
  request(`/v1/chains/${id}/records`, {
    method: "POST",
    headers: { authorization: `Bearer ${token}`, "content-type": "application/json" },
    body: JSON.stringify(items),
  });
const pull = (id: string, since = 0, token = TOKEN) =>
  request(`/v1/chains/${id}/records?since=${since}`, { headers: { authorization: `Bearer ${token}` } });
const batch = (items: unknown, ip?: string) =>
  request("/v1/pull", { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(items) }, ip);

function record(n: number, ciphertext = "Y2lwaGVy") {
  return { idHash: hid(n), kind: "host", nonce: NONCE, ciphertext, deleted: false, baseSeq: 0 };
}

type Entry = { chain: string; status: string; records?: Array<{ idHash: string; seq: number }>; latestSeq?: number };
type BatchBody = { results: Entry[] };

describe("POST /v1/pull", () => {
  it("returns every chain's records in request order", async () => {
    const a = newChainId();
    const b = newChainId();
    await create(a);
    await create(b);
    await push(a, [record(1)]);
    await push(b, [record(2), record(3)]);
    const res = await batch([
      { chain: b, token: TOKEN, since: 0 },
      { chain: a, token: TOKEN, since: 0 },
    ]);
    expect(res.status).toBe(200);
    const body = (await res.json()) as BatchBody;
    expect(body.results.map((r) => [r.chain, r.status, r.latestSeq])).toEqual([
      [b, "ok", 2],
      [a, "ok", 1],
    ]);
    expect(body.results[0].records?.map((r) => r.idHash)).toEqual([hid(2), hid(3)]);
    expect(body.results[1].records?.map((r) => r.idHash)).toEqual([hid(1)]);
  });

  it("pulls each chain since its own cursor", async () => {
    const a = newChainId();
    await create(a);
    await push(a, [record(1), record(2)]);
    const body = (await (await batch([{ chain: a, token: TOKEN, since: 1 }])).json()) as BatchBody;
    expect(body.results[0].records?.map((r) => r.seq)).toEqual([2]);
    expect(body.results[0].latestSeq).toBe(2);
  });

  it("answers not_found for unknown chains and wrong tokens without allocating storage", async () => {
    const known = newChainId();
    const unknown = newChainId();
    await create(known);
    const body = (await (
      await batch([
        { chain: known, token: OTHER_TOKEN, since: 0 },
        { chain: unknown, token: TOKEN, since: 0 },
      ])
    ).json()) as BatchBody;
    expect(body.results).toEqual([
      { chain: known, status: "not_found" },
      { chain: unknown, status: "not_found" },
    ]);
    expect(await env.CHAIN.getByName(unknown).exists()).toBe(false);
  });

  it("reports a chain's own rate limit per item and still serves the others", async () => {
    const busy = newChainId();
    const calm = newChainId();
    await create(busy); // 第 1 次
    await create(calm);
    for (let i = 0; i < 119; i++) {
      expect((await pull(busy)).status).toBe(200); // 第 2..120 次
    }
    const body = (await (
      await batch([
        { chain: busy, token: TOKEN, since: 0 },
        { chain: calm, token: TOKEN, since: 0 },
      ])
    ).json()) as BatchBody;
    expect(body.results.map((r) => r.status)).toEqual(["rate_limited", "ok"]);
  });

  it("rejects malformed batches", async () => {
    const a = newChainId();
    const ok = { chain: a, token: TOKEN, since: 0 };
    expect((await batch({ chain: a })).status).toBe(400);
    expect((await batch([])).status).toBe(400);
    expect((await batch(Array.from({ length: 65 }, () => ({ ...ok, chain: newChainId() })))).status).toBe(400);
    expect((await batch([{ ...ok, chain: a.toUpperCase() }])).status).toBe(400);
    expect((await batch([{ ...ok, token: TOKEN.toUpperCase() }])).status).toBe(400);
    expect((await batch([{ ...ok, since: -1 }])).status).toBe(400);
    expect((await batch([{ ...ok, since: 1.5 }])).status).toBe(400);
    expect((await batch([ok, ok])).status).toBe(400);
    expect((await request("/v1/pull", { method: "POST", body: "not json" })).status).toBe(400);
    expect((await request("/v1/pull", { method: "POST", body: "[" + " ".repeat(70_000) + "]" })).status).toBe(413);
    expect((await request("/v1/pull")).status).toBe(404);
  });

  it("defers the remaining items once the response passes the 2 MiB budget", async () => {
    // 每條 chain 15 筆、每筆 65,532 字元密文:約 0.98 MB,低於每 chain 1 MiB 的額度。
    const big = "A".repeat(65_532);
    const chains = [newChainId(), newChainId(), newChainId()];
    for (const id of chains) {
      await create(id);
      expect((await push(id, Array.from({ length: 15 }, (_, i) => record(i + 1, big)))).status).toBe(200);
    }
    const small = newChainId();
    await create(small);
    await push(small, [record(1)]);
    const body = (await (
      await batch([...chains, small].map((chain) => ({ chain, token: TOKEN, since: 0 })))
    ).json()) as BatchBody;
    // 前兩條後累計約 1.97 MB(< 2 MiB)所以第三條照做;做完超過預算,第四條延後。
    expect(body.results.map((r) => r.status)).toEqual(["ok", "ok", "ok", "deferred"]);
    expect(body.results[3]).toEqual({ chain: small, status: "deferred" });
  });

  it("charges the pull bucket per item and refuses the whole batch when it runs out", async () => {
    const ip = "198.51.100.77";
    const a = newChainId();
    const b = newChainId();
    await create(a);
    await create(b);
    const limiter = env.IP_LIMIT.getByName(`ip:${ip}`);
    expect(await limiter.allow("pull", 11_999)).toBe(true);
    const items = [
      { chain: a, token: TOKEN, since: 0 },
      { chain: b, token: TOKEN, since: 0 },
    ];
    expect((await batch(items, ip)).status).toBe(429);
    expect((await batch(items, "198.51.100.78")).status).toBe(200);
  });
});

describe("GET /v1/info", () => {
  it("reports the relay version and its features", async () => {
    const res = await request("/v1/info");
    expect(res.status).toBe(200);
    expect(await res.json()).toEqual({ relay: "sshelter-relay", version: RELAY_VERSION, features: ["pull-batch", "freeze"] });
    expect(RELAY_VERSION).toBe(pkg.version);
    expect((await request("/v1/info", { method: "POST" })).status).toBe(404);
  });
});
```

在 `relay/test/ip-limit.test.ts` 的 import 改成同時引入 `runInDurableObject`,並在檔尾加兩個測試:

```ts
import { env, exports } from "cloudflare:workers";
import { runInDurableObject } from "cloudflare:test";
import { expect, it } from "vitest";
```

```ts
it("allows 12000 pulled chains per IP per hour on the pull bucket, charged by cost", async () => {
  const limiter = env.IP_LIMIT.getByName("ip:unit-test-pull");
  expect(await limiter.allow("pull", 11_990)).toBe(true);
  expect(await limiter.allow("pull", 10)).toBe(true);
  expect(await limiter.allow("pull")).toBe(false);
  // 其他桶不受影響。
  expect(await limiter.allow("request")).toBe(true);
});

it("starts the pull bucket for limiters stored by an older relay", async () => {
  const limiter = env.IP_LIMIT.getByName("ip:unit-test-legacy");
  await runInDurableObject(limiter, async (_instance, state) => {
    const now = Date.now();
    // 0.1.0 的儲存格式:只有 create 與 request 兩個桶。
    await state.storage.put("windows", { create: { start: now, count: 3 }, request: { start: now, count: 5 } });
  });
  expect(await limiter.allow("pull", 5)).toBe(true);
  expect(await limiter.allow("create")).toBe(true);
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd relay && npm test`
Expected: FAIL —— `batch.test.ts` 的 `/v1/pull` 回 `404`、`RELAY_VERSION` 不存在;`ip-limit.test.ts` 的 `allow("pull", …)`
因為沒有 `pull` 桶而失敗。

- [ ] **Step 3: 實作**

`relay/package.json`:`"version": "0.1.0"` → `"version": "0.2.0"`。

`relay/src/index.ts` —— 在檔案開頭的常數區(`MAX_ITEMS_PER_PUSH` 之後)加入:

```ts
const MAX_PULL_ITEMS = 64;
const MAX_PULL_BODY_BYTES = 65_536;
const PULL_RESPONSE_BUDGET = 2 * 1024 * 1024; // 已放入回應的 JSON 字元數(全為 ASCII,字元 = byte)

/** relay 版本;與 package.json 的 version 相同(測試檢查),app 以 `GET /v1/info` 讀取。 */
export const RELAY_VERSION = "0.2.0";
const FEATURES = ["pull-batch", "freeze"] as const;
```

在 `PushItem` 介面之後加入:

```ts
export interface PullItem {
  chain: string;
  token: string;
  since: number;
}

type PullEntry =
  | { chain: string; status: "ok"; records: Envelope[]; latestSeq: number }
  | { chain: string; status: "not_found" | "rate_limited" | "deferred" };
```

把 `IpLimiter` 整個換成(相容沒有 `pull` 欄位的舊儲存):

```ts
type Bucket = "create" | "request" | "pull";
const IP_LIMITS: Record<Bucket, number> = { create: 20, request: 1200, pull: 12000 };
type Windows = Record<Bucket, { start: number; count: number }>;

/** 每個來源 IP 一個小 DO:一小時視窗內分桶計數(建鏈、所有請求、批次查詢的 chain 數);閒置一小時由 alarm 清空自己。 */
export class IpLimiter extends DurableObject<Env> {
  /** `cost` 一次計入多筆(批次查詢每項計 1);超過上限也照計,與單筆的語意相同。 */
  async allow(bucket: Bucket, cost = 1): Promise<boolean> {
    const now = Date.now();
    // 0.1.0 存的 windows 沒有 pull 欄位:缺的桶從 0 開始。
    const stored = await this.ctx.storage.get<Partial<Windows>>("windows");
    const windows: Windows = {
      create: stored?.create ?? { start: now, count: 0 },
      request: stored?.request ?? { start: now, count: 0 },
      pull: stored?.pull ?? { start: now, count: 0 },
    };
    const w = windows[bucket];
    if (now - w.start >= IP_WINDOW_MS) {
      w.start = now;
      w.count = 0;
    }
    w.count += cost;
    await this.ctx.storage.put("windows", windows);
    await this.ctx.storage.setAlarm(now + IP_WINDOW_MS);
    return w.count <= IP_LIMITS[bucket];
  }

  async alarm(): Promise<void> {
    await this.ctx.storage.deleteAll();
  }
}
```

在 `isPushItem` 之後加入:

```ts
function isPullItem(v: unknown): v is PullItem {
  if (typeof v !== "object" || v === null) return false;
  const o = v as Record<string, unknown>;
  return (
    typeof o.chain === "string" && HEX64.test(o.chain) &&
    typeof o.token === "string" && HEX64.test(o.token) &&
    typeof o.since === "number" && Number.isSafeInteger(o.since) && o.since >= 0
  );
}
```

在 `status()` 輔助函式之後、`export default` 之前加入:

```ts
/** 每 IP 的限流 DO(Cloudflare 在邊緣覆寫 CF-Connecting-IP;自架時由 Caddy 覆寫)。 */
function limiterFor(request: Request, env: Env) {
  const ip = request.headers.get("cf-connecting-ip") ?? "unknown";
  return env.IP_LIMIT.getByName(`ip:${ip}`);
}

/** `GET /v1/info`:版本與功能,app 據此決定要不要用批次查詢與凍結。 */
async function info(request: Request, env: Env): Promise<Response> {
  if (!(await limiterFor(request, env).allow("request"))) return status(429);
  return Response.json({ relay: "sshelter-relay", version: RELAY_VERSION, features: FEATURES });
}

/**
 * `POST /v1/pull`:一次查多條 chain。每項各自走該 chain 的 `pull()`(各自驗證權杖、計入每分鐘限制、刷新閒置期限)。
 * 依序處理;累計的回應 JSON 超過預算後,剩下的項目回 deferred(不執行),第一項一律執行。
 */
async function batchPull(request: Request, env: Env): Promise<Response> {
  const limiter = limiterFor(request, env);
  if (!(await limiter.allow("request"))) return status(429);
  const declared = Number(request.headers.get("content-length") ?? "0");
  if (declared > MAX_PULL_BODY_BYTES) return status(413);
  const text = await readBodyCapped(request, MAX_PULL_BODY_BYTES);
  if (text === null) return status(413);
  let body: unknown;
  try {
    body = JSON.parse(text);
  } catch {
    return bad("body must be JSON");
  }
  if (!Array.isArray(body) || body.length === 0 || body.length > MAX_PULL_ITEMS || !body.every(isPullItem)) {
    return bad("body must be a non-empty array of at most 64 pull items");
  }
  if (new Set(body.map((i) => i.chain)).size !== body.length) return bad("duplicate chain in one pull");
  if (!(await limiter.allow("pull", body.length))) return status(429);

  const results: PullEntry[] = [];
  let used = 0;
  for (const item of body) {
    if (results.length > 0 && used > PULL_RESPONSE_BUDGET) {
      results.push({ chain: item.chain, status: "deferred" });
      continue;
    }
    const result = await env.CHAIN.getByName(item.chain).pull(await sha256Hex(item.token), item.since);
    const entry: PullEntry =
      result.status === 200 && result.body
        ? { chain: item.chain, status: "ok", records: result.body.records, latestSeq: result.body.latestSeq }
        : { chain: item.chain, status: result.status === 429 ? "rate_limited" : "not_found" };
    used += JSON.stringify(entry).length;
    results.push(entry);
  }
  return Response.json({ results });
}
```

`export default` 的 `fetch` 開頭(`const url = new URL(request.url);` 之後、chain 路徑比對之前)加入:

```ts
    if (url.pathname === "/v1/info") return request.method === "GET" ? info(request, env) : notFound();
    if (url.pathname === "/v1/pull") return request.method === "POST" ? batchPull(request, env) : notFound();
```

並把既有的 IP 限流兩行改用 `limiterFor`:

```ts
    const limiter = limiterFor(request, env);
    if (!(await limiter.allow("request"))) return status(429);
```

(刪掉原本的 `const ip = …` 與 `const limiter = env.IP_LIMIT.getByName(...)` 兩行;保留其上方說明 CF-Connecting-IP 的註解,
移到 `limiterFor` 上。)

`relay/selfhost/smoke-test.mjs` —— 在 `call()` 之後加入:

```js
async function callBatch(items) {
  const res = await fetch(new URL("/v1/pull", base), {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(items),
    redirect: "manual",
  });
  return { status: res.status, json: await res.json() };
}
```

在 `write` 階段、`writeFileSync(stateFile, …)` 之前加入:

```js
  const info = await fetch(new URL("/v1/info", base)).then(async (r) => ({ status: r.status, json: await r.json() }));
  check("the relay reports its features", [info.status, info.json.features], [200, ["pull-batch", "freeze"]]);
  const unknown = hex(32);
  check("a batch pull answers each chain on its own", await callBatch([
    { chain, token, since: 0 },
    { chain: unknown, token: hex(32), since: 0 },
  ]), {
    status: 200,
    json: { results: [{ chain, status: "ok", records: [{ ...record, seq: 1 }], latestSeq: 1 }, { chain: unknown, status: "not_found" }] },
  });
```

並把檔首的用法說明第一行改成:

```js
//   node smoke-test.mjs <base-url> <state-file> write  — create a chain, push, hit a conflict, batch-pull; save its credentials
```

`relay/README.md` 的 `## Limits` 清單第二行改成:

```markdown
- 120 requests/min per chain (any endpoint); per IP: 20 chain-creation requests/hour, 1200 requests/hour, 12000 chains pulled/hour through `POST /v1/pull`
```

並在 `## Limits` 之前加一節:

```markdown
## API

Besides the per-chain endpoints (`PUT`/`DELETE /v1/chains/{id}`, `GET`/`POST /v1/chains/{id}/records`):

- `POST /v1/pull` — pull up to 64 chains in one request: `[{ "chain", "token", "since" }]`. Each item is checked like
  `GET /v1/chains/{id}/records` and answers `ok`, `not_found`, `rate_limited` or `deferred` (the response passed its
  2 MiB budget; ask again).
- `POST /v1/chains/{id}/freeze` — stop all further writes to a chain (used when the sync code changes).
- `GET /v1/info` — the relay version and features, so the app knows whether this relay needs an update.
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd relay && npm run typecheck && npm test`
Expected: PASS(既有測試與新測試全綠)。

- [ ] **Step 5: Commit**

```bash
git add relay/src/index.ts relay/package.json relay/test/tsconfig.json relay/test/batch.test.ts relay/test/ip-limit.test.ts relay/selfhost/smoke-test.mjs relay/README.md
git commit -m "feat(relay): pull many chains in one request and report the relay version"
```

---

### Task 2: 凍結 `POST /v1/chains/{id}/freeze`

**Files:**
- Modify: `relay/src/index.ts`(`ChainStore.freeze`、push 的 409、router 的 `/freeze`)
- Modify: `relay/test/relay.test.ts`
- Modify: `relay/selfhost/smoke-test.mjs`(write 階段加凍結檢查)

**Interfaces:**
- Consumes: Task 1 的 `limiterFor()`、router 結構。
- Produces(B2/B3 依此實作):`POST /v1/chains/{id}/freeze`,`Authorization: Bearer <token>`;`204` 凍結(冪等)、
  `404` 不存在或權杖不符、`429` 超過每 chain 速率。凍結後 `POST /v1/chains/{id}/records` 一律
  `409 { "status": "frozen" }` 且不寫入;`GET` 記錄、批次查詢、`DELETE` 照常;`PUT` 回 `200` 且維持凍結。
  `ChainStore.freeze(tokenHash): Promise<204 | 404 | 429>`;`ChainStore.push` 的回傳狀態多了 `409`。

- [ ] **Step 1: 寫失敗的測試**

在 `relay/test/relay.test.ts` 的 `const destroy = …` 之後加入:

```ts
const freeze = (token = TOKEN, id = chain) => relay(`/v1/chains/${id}/freeze`, { method: "POST", headers: auth(token) });
```

並在檔尾加入:

```ts
describe("freeze", () => {
  it("refuses every push after a freeze and writes nothing", async () => {
    await create();
    await push([item(hid(1))]);
    expect((await freeze()).status).toBe(204);
    const refused = await push([item(hid(2))]);
    expect(refused.status).toBe(409);
    expect(await refused.json()).toEqual({ status: "frozen" });
    // 已存在的記錄被「更新」也一樣被擋。
    expect((await push([item(hid(1), 1, "bmV3")])).status).toBe(409);
    const all = (await (await pull(0)).json()) as PullBody;
    expect(all.records.map((r) => r.idHash)).toEqual([hid(1)]);
    expect(all.latestSeq).toBe(1);
  });

  it("keeps a frozen chain readable and deletable; a re-created chain starts unfrozen", async () => {
    await create();
    await push([item(hid(1))]);
    await freeze();
    expect((await pull(0)).status).toBe(200);
    expect((await create()).status).toBe(200);
    expect((await push([item(hid(2))])).status).toBe(409);
    expect((await destroy()).status).toBe(204);
    expect((await create()).status).toBe(201);
    expect((await push([item(hid(3))])).status).toBe(200);
  });

  it("is idempotent and needs the chain's own token", async () => {
    await create();
    expect((await freeze(OTHER_TOKEN)).status).toBe(404);
    expect((await push([item(hid(1))])).status).toBe(200);
    expect((await freeze()).status).toBe(204);
    expect((await freeze()).status).toBe(204);
    const id = newChainId();
    expect((await freeze(TOKEN, id)).status).toBe(404);
    expect(await env.CHAIN.getByName(id).exists()).toBe(false);
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `cd relay && npm test -- relay.test.ts`
Expected: FAIL —— `/freeze` 回 `404`(router 不認得),凍結後的 push 回 `200`。

- [ ] **Step 3: 實作**

`ChainStore` 加入(放在 `authorized()` 之後):

```ts
  /** 凍結 = 更換同步碼時的寫入截止點;存在 meta,不可解除(DELETE 會連 chain 一起清掉)。 */
  private frozen(): boolean {
    return this.meta("frozen") === "1";
  }

  /** 204 凍結(冪等)、404 不存在或權杖不符、429 超過每 chain 速率。凍結後 push 一律 409,讀取與刪除照常。 */
  async freeze(tokenHash: string): Promise<204 | 404 | 429> {
    if (!this.authorized(tokenHash)) return 404;
    if (this.rateLimited()) return 429;
    this.setMeta("frozen", "1");
    return 204;
  }
```

`ChainStore.push` 的簽章與開頭改成:

```ts
  async push(tokenHash: string, items: PushItem[]): Promise<{ status: 200 | 404 | 409 | 413 | 429; body?: { results: PushResult[]; latestSeq: number } }> {
    if (!this.authorized(tokenHash)) return { status: 404 };
    if (this.rateLimited()) return { status: 429 };
    if (this.frozen()) return { status: 409 };
```

router:把 chain 路徑的比對與分支改成(其餘不變):

```ts
    const match = /^\/v1\/chains\/([^/]+)(\/records|\/freeze)?$/.exec(url.pathname);
    if (!match) return notFound();
    const [, chainId, suffix] = match;
    const recordsPath = suffix === "/records";
    const freezePath = suffix === "/freeze";
```

`isCreate` 改成 `const isCreate = suffix === undefined && request.method === "PUT";`;`DELETE` 分支的條件改成
`suffix === undefined && request.method === "DELETE"`;在 `DELETE` 分支之後加入:

```ts
    if (freezePath && request.method === "POST") {
      return status(await stub.freeze(tokenHash));
    }
```

push 分支裡 `const result = await stub.push(tokenHash, body);` 之後加入:

```ts
      if (result.status === 409) return Response.json({ status: "frozen" }, { status: 409 });
```

`relay/selfhost/smoke-test.mjs` 的 `write` 階段、`writeFileSync(stateFile, …)` 之前加入:

```js
  const frozenChain = hex(32);
  const frozenToken = hex(32);
  check("create a chain to freeze", (await call("PUT", `/v1/chains/${frozenChain}`, frozenToken)).status, 201);
  check("freeze it", (await call("POST", `/v1/chains/${frozenChain}/freeze`, frozenToken)).status, 204);
  check("a frozen chain refuses pushes", await call("POST", `/v1/chains/${frozenChain}/records`, frozenToken, {
    body: [{ ...record, baseSeq: 0 }],
  }), { status: 409, json: { status: "frozen" } });
  check("a frozen chain still reads", (await call("GET", `/v1/chains/${frozenChain}/records?since=0`, frozenToken)).status, 200);
  check("a frozen chain can be deleted", (await call("DELETE", `/v1/chains/${frozenChain}`, frozenToken)).status, 204);
```

- [ ] **Step 4: 跑測試確認通過**

Run: `cd relay && npm run typecheck && npm test`
Expected: PASS(含 Task 1 與既有測試)。

本機有 Docker 時再跑一次 self-host 冒煙測試(CI 也會跑):

```bash
cd relay/selfhost
docker build -q -f Dockerfile -t sshelter-relay-b1 .. && docker run -d --name relay-b1 -p 127.0.0.1:8787:8080 -v relay-b1:/data sshelter-relay-b1
for _ in $(seq 1 60); do curl -s -o /dev/null http://127.0.0.1:8787/ && break; sleep 1; done
node smoke-test.mjs http://127.0.0.1:8787 "$TMPDIR/b1-state.json" write && docker restart relay-b1 >/dev/null
for _ in $(seq 1 60); do curl -s -o /dev/null http://127.0.0.1:8787/ && break; sleep 1; done
node smoke-test.mjs http://127.0.0.1:8787 "$TMPDIR/b1-state.json" read
docker rm -f relay-b1 && docker volume rm relay-b1 && docker image rm sshelter-relay-b1
```

Expected: 每一行都是 `✓`。

- [ ] **Step 5: Commit**

```bash
git add relay/src/index.ts relay/test/relay.test.ts relay/selfhost/smoke-test.mjs
git commit -m "feat(relay): freeze a chain so no further writes land on it"
```

---

### Task 3: 「Update relay」workflow 與文件

**Files:**
- Create: `relay/.github/workflows/update-relay.yml`
- Create: `relay/.github/scripts/update-relay.mjs`
- Create: `relay/.github/scripts/update-relay.test.mjs`
- Create: `relay/.gitignore`
- Modify: `.github/workflows/relay.yml`(新增 job 跑腳本測試)
- Modify: `relay/README.md`(「Updating your relay」)
- Modify: `README.md`(Sync 一節連到更新說明)

**Interfaces:**
- Consumes: 無(與 Task 1、2 無程式相依;只需要它們存在於上游)。
- Produces:`node .github/scripts/update-relay.mjs apply <upstream-relay-dir> <repo-dir>` 印出摘要 JSON
  `{ workerName, removed: string[], durableObjectsChanged: boolean, newMigrationTags: string[] }`;
  `node .github/scripts/update-relay.mjs pr-body <summary.json> <upstream> <ref> <sha>` 印出 PR 內文。
  匯出的純函式:`stripJsonComments(text)`、`readWorkerName(text)`、`withWorkerName(text, name)`、
  `planSync(upstreamFiles, repoFiles)`、`describeWranglerChanges(oldText, newText)`、`prBody(summary, upstream, ref, sha)`、
  `listFiles(dir)`、`applyUpdate(upstreamDir, repoDir)`。

- [ ] **Step 1: 寫失敗的測試**

建立 `relay/.github/scripts/update-relay.test.mjs`:

```js
import { deepStrictEqual, strictEqual, throws } from "node:assert/strict";
import { existsSync, mkdirSync, mkdtempSync, readFileSync, writeFileSync } from "node:fs";
import { tmpdir } from "node:os";
import { dirname, join } from "node:path";
import { describe, it } from "node:test";

import {
  applyUpdate,
  describeWranglerChanges,
  listFiles,
  planSync,
  prBody,
  readWorkerName,
  withWorkerName,
} from "./update-relay.mjs";

const UPSTREAM_WRANGLER = `{
  "$schema": "node_modules/wrangler/config-schema.json",
  // a comment with "name": "decoy" inside, before the real name
  "name": "sshelter-relay",
  "main": "src/index.ts",
  "compatibility_date": "2026-08-20",
  "durable_objects": {
    "bindings": [
      { "name": "CHAIN", "class_name": "ChainStore" },
      { "name": "IP_LIMIT", "class_name": "IpLimiter" }
    ]
  },
  "migrations": [{ "tag": "v1", "new_sqlite_classes": ["ChainStore", "IpLimiter"] }]
}
`;

function tree(files) {
  const root = mkdtempSync(join(tmpdir(), "update-relay-"));
  for (const [path, content] of Object.entries(files)) {
    mkdirSync(dirname(join(root, path)), { recursive: true });
    writeFileSync(join(root, path), content);
  }
  return root;
}

describe("the Worker name", () => {
  it("reads the top-level name, not a binding's or a comment's", () => {
    strictEqual(readWorkerName(UPSTREAM_WRANGLER), "sshelter-relay");
  });

  it("keeps the Worker name and changes nothing else", () => {
    const renamed = withWorkerName(UPSTREAM_WRANGLER, "my-relay");
    strictEqual(readWorkerName(renamed), "my-relay");
    strictEqual(renamed.replace('"name": "my-relay"', '"name": "sshelter-relay"'), UPSTREAM_WRANGLER);
  });

  it("refuses to guess when the first name key is not the Worker name", () => {
    const noTopLevel = UPSTREAM_WRANGLER.replace('  "name": "sshelter-relay",\n', "");
    throws(() => withWorkerName(noTopLevel, "my-relay"), /no top-level Worker name/);
    // 頂層 name 移到最後:第一個不在註解裡的 "name" 是 binding 的,替換後驗證會失敗。
    const nameLast = noTopLevel.replace(/\n}\n$/, ',\n  "name": "sshelter-relay"\n}\n');
    strictEqual(readWorkerName(nameLast), "sshelter-relay");
    throws(() => withWorkerName(nameLast, "my-relay"), /not the Worker name/);
  });
});

describe("planSync", () => {
  it("copies upstream files and removes files upstream dropped", () => {
    const plan = planSync(["src/index.ts", "package.json"], ["src/index.ts", "src/old.ts", "package.json"]);
    deepStrictEqual(plan, { copy: ["src/index.ts", "package.json"], remove: ["src/old.ts"] });
  });

  it("never touches the owner's other GitHub files but keeps its own up to date", () => {
    const plan = planSync(
      [".github/workflows/update-relay.yml", ".github/scripts/update-relay.mjs", "src/index.ts"],
      [".github/workflows/update-relay.yml", ".github/workflows/mine.yml", ".github/scripts/update-relay.mjs", "src/index.ts"],
    );
    deepStrictEqual(plan.copy, [".github/workflows/update-relay.yml", ".github/scripts/update-relay.mjs", "src/index.ts"]);
    deepStrictEqual(plan.remove, []);
  });
});

describe("describeWranglerChanges", () => {
  it("reports new migration tags and binding changes", () => {
    const next = UPSTREAM_WRANGLER.replace(
      '[{ "tag": "v1", "new_sqlite_classes": ["ChainStore", "IpLimiter"] }]',
      '[{ "tag": "v1", "new_sqlite_classes": ["ChainStore", "IpLimiter"] }, { "tag": "v2", "new_sqlite_classes": ["Extra"] }]',
    ).replace('{ "name": "IP_LIMIT", "class_name": "IpLimiter" }', '{ "name": "IP_LIMIT", "class_name": "IpLimiter" }, { "name": "EXTRA", "class_name": "Extra" }');
    deepStrictEqual(describeWranglerChanges(UPSTREAM_WRANGLER, next), { durableObjectsChanged: true, newMigrationTags: ["v2"] });
    deepStrictEqual(describeWranglerChanges(UPSTREAM_WRANGLER, UPSTREAM_WRANGLER), { durableObjectsChanged: false, newMigrationTags: [] });
  });
});

describe("prBody", () => {
  it("names the source, the kept Worker name, removed files and migrations", () => {
    const body = prBody(
      { workerName: "my-relay", removed: ["src/old.ts"], durableObjectsChanged: false, newMigrationTags: ["v2"] },
      "ysya/sshelter",
      "v0.18.0",
      "a".repeat(40),
    );
    for (const part of ["v0.18.0", "ysya/sshelter", "a".repeat(40), "`my-relay`", "`src/old.ts`", "v2"]) {
      strictEqual(body.includes(part), true, part);
    }
  });
});

describe("applyUpdate", () => {
  it("syncs the files, keeps the Worker name and the owner's files, and reports what changed", () => {
    const upstream = tree({
      "wrangler.jsonc": UPSTREAM_WRANGLER,
      "src/index.ts": "new",
      ".github/workflows/update-relay.yml": "workflow v2",
      ".gitignore": "node_modules/\n",
    });
    const repo = tree({
      "wrangler.jsonc": withWorkerName(UPSTREAM_WRANGLER, "my-relay"),
      "src/index.ts": "old",
      "src/removed.ts": "gone upstream",
      ".github/workflows/update-relay.yml": "workflow v1",
      ".github/workflows/mine.yml": "mine",
      "node_modules/x/index.js": "dependency",
    });
    const summary = applyUpdate(upstream, repo);
    deepStrictEqual(summary, { workerName: "my-relay", removed: ["src/removed.ts"], durableObjectsChanged: false, newMigrationTags: [] });
    strictEqual(readFileSync(join(repo, "src/index.ts"), "utf8"), "new");
    strictEqual(existsSync(join(repo, "src/removed.ts")), false);
    strictEqual(readFileSync(join(repo, ".github/workflows/update-relay.yml"), "utf8"), "workflow v2");
    strictEqual(readFileSync(join(repo, ".github/workflows/mine.yml"), "utf8"), "mine");
    strictEqual(existsSync(join(repo, "node_modules/x/index.js")), true);
    strictEqual(readWorkerName(readFileSync(join(repo, "wrangler.jsonc"), "utf8")), "my-relay");
    deepStrictEqual(listFiles(repo).includes(".gitignore"), true);
  });
});
```

- [ ] **Step 2: 跑測試確認失敗**

Run: `node --test relay/.github/scripts/update-relay.test.mjs`
Expected: FAIL —— `Cannot find module './update-relay.mjs'`。

- [ ] **Step 3: 實作腳本**

建立 `relay/.github/scripts/update-relay.mjs`:

```js
// Updates a relay deployed with the "Deploy to Cloudflare" button to an upstream SSHelter release.
// Used by .github/workflows/update-relay.yml. Node >= 20, no dependencies.
//   node update-relay.mjs apply <upstream-relay-dir> <repo-dir>          -> prints a JSON summary
//   node update-relay.mjs pr-body <summary.json> <upstream> <ref> <sha>  -> prints the pull request body
import { copyFileSync, mkdirSync, readFileSync, readdirSync, rmSync, statSync, writeFileSync } from "node:fs";
import { dirname, join, relative, sep } from "node:path";
import { pathToFileURL } from "node:url";

// The only files under .github/ that belong to the relay; everything else there is the owner's.
const MANAGED_GITHUB_FILES = new Set([
  ".github/workflows/update-relay.yml",
  ".github/scripts/update-relay.mjs",
  ".github/scripts/update-relay.test.mjs",
]);
const SKIPPED_DIRS = new Set([".git", "node_modules", ".wrangler"]);

/** Drop `//` and `/* *\/` comments outside strings, so JSON.parse can read a .jsonc file. */
export function stripJsonComments(text) {
  let out = "";
  for (let i = 0; i < text.length; i++) {
    const c = text[i];
    if (c === '"') {
      const start = i;
      for (i++; i < text.length && text[i] !== '"'; i++) if (text[i] === "\\") i++;
      out += text.slice(start, i + 1);
    } else if (c === "/" && text[i + 1] === "/") {
      while (i < text.length && text[i] !== "\n") i++;
      out += "\n";
    } else if (c === "/" && text[i + 1] === "*") {
      i = text.indexOf("*/", i + 2) + 1;
      if (i === 0) throw new Error("unterminated block comment in wrangler.jsonc");
    } else {
      out += c;
    }
  }
  return out;
}

const parseJsonc = (text) => JSON.parse(stripJsonComments(text));

export function readWorkerName(wranglerText) {
  const name = parseJsonc(wranglerText).name;
  if (typeof name !== "string" || name === "") throw new Error("wrangler.jsonc has no Worker name");
  return name;
}

/** Put `name` back as the Worker name. Refuses rather than guess when the first "name" key is not the top-level one. */
export function withWorkerName(wranglerText, name) {
  const before = parseJsonc(wranglerText);
  if (typeof before.name !== "string") throw new Error("wrangler.jsonc has no top-level Worker name");
  // The first `"name": "..."` outside a comment. A match inside a comment disappears when the text up to its end
  // is stripped, so the stripped prefix no longer ends with it.
  const pattern = /"name"\s*:\s*"[^"]*"/g;
  let index = -1;
  let length = 0;
  for (let m = pattern.exec(wranglerText); m; m = pattern.exec(wranglerText)) {
    if (stripJsonComments(wranglerText.slice(0, m.index + m[0].length)).endsWith(m[0])) {
      index = m.index;
      length = m[0].length;
      break;
    }
  }
  if (index < 0) throw new Error("wrangler.jsonc has no top-level Worker name");
  const next = `${wranglerText.slice(0, index)}"name": ${JSON.stringify(name)}${wranglerText.slice(index + length)}`;
  const after = parseJsonc(next);
  if (after.name !== name || JSON.stringify({ ...after, name: before.name }) !== JSON.stringify(before)) {
    throw new Error("wrangler.jsonc: the first \"name\" key is not the Worker name; keep your Worker name by hand");
  }
  return next;
}

export function planSync(upstreamFiles, repoFiles) {
  const managed = (path) => !path.startsWith(".github/") || MANAGED_GITHUB_FILES.has(path);
  const upstream = new Set(upstreamFiles);
  return {
    copy: upstreamFiles.filter(managed),
    remove: repoFiles.filter((path) => managed(path) && !upstream.has(path)),
  };
}

export function describeWranglerChanges(oldText, newText) {
  const before = parseJsonc(oldText);
  const after = parseJsonc(newText);
  const tags = (config) => (config.migrations ?? []).map((m) => m.tag);
  const known = new Set(tags(before));
  return {
    durableObjectsChanged: JSON.stringify(before.durable_objects ?? null) !== JSON.stringify(after.durable_objects ?? null),
    newMigrationTags: tags(after).filter((tag) => !known.has(tag)),
  };
}

export function prBody(summary, upstream, ref, sha) {
  const removed = summary.removed.length ? summary.removed.map((f) => `\`${f}\``).join(", ") : "none";
  return [
    `Updates this relay to **${ref}** of [${upstream}](https://github.com/${upstream}) (commit \`${sha}\`).`,
    "",
    `Merging deploys it through Cloudflare Workers Builds. The Worker name \`${summary.workerName}\` is kept, so the relay URL and its stored data stay the same.`,
    "",
    `- Files removed: ${removed}`,
    `- Durable Object bindings: ${summary.durableObjectsChanged ? "**changed** — review `wrangler.jsonc` before merging" : "unchanged"}`,
    `- Migrations: ${summary.newMigrationTags.length ? `**new tags ${summary.newMigrationTags.join(", ")}** — they run on deploy and cannot be undone` : "none new"}`,
  ].join("\n");
}

/** Every file under `dir` as a POSIX path relative to it, skipping .git, node_modules and .wrangler. */
export function listFiles(dir) {
  const out = [];
  const walk = (current) => {
    for (const entry of readdirSync(current, { withFileTypes: true })) {
      if (entry.isDirectory()) {
        if (!SKIPPED_DIRS.has(entry.name)) walk(join(current, entry.name));
      } else if (entry.isFile()) {
        out.push(relative(dir, join(current, entry.name)).split(sep).join("/"));
      }
    }
  };
  walk(dir);
  return out.sort();
}

export function applyUpdate(upstreamDir, repoDir) {
  const oldWrangler = readFileSync(join(repoDir, "wrangler.jsonc"), "utf8");
  const workerName = readWorkerName(oldWrangler);
  const plan = planSync(listFiles(upstreamDir), listFiles(repoDir));
  for (const path of plan.copy) {
    mkdirSync(dirname(join(repoDir, path)), { recursive: true });
    copyFileSync(join(upstreamDir, path), join(repoDir, path));
  }
  for (const path of plan.remove) rmSync(join(repoDir, path));
  const newWrangler = withWorkerName(readFileSync(join(repoDir, "wrangler.jsonc"), "utf8"), workerName);
  writeFileSync(join(repoDir, "wrangler.jsonc"), newWrangler);
  return { workerName, removed: plan.remove, ...describeWranglerChanges(oldWrangler, newWrangler) };
}

const invokedDirectly = process.argv[1] && import.meta.url === pathToFileURL(process.argv[1]).href;
if (invokedDirectly) {
  const [command, ...args] = process.argv.slice(2);
  if (command === "apply" && args.length === 2 && statSync(args[0]).isDirectory()) {
    console.log(JSON.stringify(applyUpdate(args[0], args[1])));
  } else if (command === "pr-body" && args.length === 4) {
    console.log(prBody(JSON.parse(readFileSync(args[0], "utf8")), args[1], args[2], args[3]));
  } else {
    console.error("usage: update-relay.mjs apply <upstream-relay-dir> <repo-dir> | pr-body <summary.json> <upstream> <ref> <sha>");
    process.exit(2);
  }
}
```

建立 `relay/.gitignore`:

```gitignore
node_modules/
.wrangler/
dist/
```

建立 `relay/.github/workflows/update-relay.yml`:

```yaml
name: Update relay

# Brings this relay up to date with an SSHelter release. It opens a pull request; merging it
# deploys through Cloudflare Workers Builds with the same Worker name, URL and stored data.
# Weekly runs only happen when the repository variable AUTO_UPDATE is "true".
on:
  workflow_dispatch:
    inputs:
      ref:
        description: "Upstream tag, branch or commit (empty: the latest release)"
        required: false
        default: ""
      upstream:
        description: "Upstream repository"
        required: false
        default: "ysya/sshelter"
  schedule:
    - cron: "17 6 * * 1"

permissions:
  contents: write
  pull-requests: write

concurrency:
  group: update-relay
  cancel-in-progress: false

jobs:
  update:
    if: github.event_name == 'workflow_dispatch' || vars.AUTO_UPDATE == 'true'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - name: Resolve the upstream commit
        id: source
        env:
          GH_TOKEN: ${{ github.token }}
          UPSTREAM: ${{ inputs.upstream || 'ysya/sshelter' }}
          REF: ${{ inputs.ref }}
        run: |
          set -euo pipefail
          if [ -z "$REF" ]; then REF=$(gh api "repos/$UPSTREAM/releases/latest" --jq .tag_name); fi
          SHA=$(gh api "repos/$UPSTREAM/commits/$REF" --jq .sha)
          {
            echo "upstream=$UPSTREAM"
            echo "ref=$REF"
            echo "sha=$SHA"
          } >> "$GITHUB_OUTPUT"
      - name: Download the upstream relay
        env:
          UPSTREAM: ${{ steps.source.outputs.upstream }}
          SHA: ${{ steps.source.outputs.sha }}
        run: |
          set -euo pipefail
          mkdir -p "$RUNNER_TEMP/upstream"
          curl -fsSL "https://codeload.github.com/$UPSTREAM/tar.gz/$SHA" | tar -xz -C "$RUNNER_TEMP/upstream" --strip-components=1
          test -f "$RUNNER_TEMP/upstream/relay/wrangler.jsonc"
      - name: Apply it
        run: node .github/scripts/update-relay.mjs apply "$RUNNER_TEMP/upstream/relay" . > "$RUNNER_TEMP/summary.json"
      - name: Test it
        run: |
          npm ci
          npm test
      - name: Open a pull request
        env:
          GH_TOKEN: ${{ github.token }}
          UPSTREAM: ${{ steps.source.outputs.upstream }}
          REF: ${{ steps.source.outputs.ref }}
          SHA: ${{ steps.source.outputs.sha }}
          REPO: ${{ github.repository }}
          SERVER: ${{ github.server_url }}
        run: |
          set -euo pipefail
          if [ -z "$(git status --porcelain)" ]; then
            echo "This relay already matches $REF."
            exit 0
          fi
          BRANCH="update-relay/${SHA:0:12}"
          git switch -c "$BRANCH"
          git add -A
          git -c user.name="github-actions[bot]" -c user.email="41898282+github-actions[bot]@users.noreply.github.com" \
            commit -m "chore: update relay to $REF"
          git push --force origin "$BRANCH"
          node "$RUNNER_TEMP/upstream/relay/.github/scripts/update-relay.mjs" pr-body "$RUNNER_TEMP/summary.json" "$UPSTREAM" "$REF" "$SHA" > "$RUNNER_TEMP/body.md"
          if ! gh pr create --head "$BRANCH" --title "Update relay to $REF" --body-file "$RUNNER_TEMP/body.md"; then
            echo "::error::GitHub Actions may not open pull requests in this repository. Open one by hand: $SERVER/$REPO/compare/$BRANCH?expand=1 (or allow it under Settings → Actions → General)."
            exit 1
          fi
```

`.github/workflows/relay.yml` 在 `selfhost` job 之前加入:

```yaml
  # The "Update relay" helper that ships inside relay/.github for deployed copies.
  update-script:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: lts/*
      - run: node --test relay/.github/scripts/update-relay.test.mjs
```

- [ ] **Step 4: 跑測試確認通過**

Run: `node --test relay/.github/scripts/update-relay.test.mjs`
Expected: PASS(全部測試)。

Run: `actionlint .github/workflows/relay.yml relay/.github/workflows/update-relay.yml`(本機有 actionlint 時)
Expected: 沒有輸出。

- [ ] **Step 5: 文件**

`relay/README.md` 在 `## Self-host with Docker Compose` 之後加入:

```markdown
## Updating your relay

SSHelter tells you in *Settings → Sync* when your relay is missing something the app can use.

- **Deployed with the button (GitHub):** in your relay repository, open *Actions → Update relay → Run workflow*. It
  opens a pull request that brings the relay up to the latest SSHelter release (or the tag you enter), runs its tests,
  and lists removed files and any Durable Object or migration change. Merge it and Cloudflare deploys it with the same
  Worker name, so the URL and the stored chains stay the same. Set the repository variable `AUTO_UPDATE` to `true`
  for a weekly check. If the run says Actions may not open pull requests, use the link it prints or allow it under
  *Settings → Actions → General*.
- **Deployed before this workflow existed:** copy `.github/workflows/update-relay.yml` and `.github/scripts/update-relay.mjs`
  from this folder into your relay repository, or deploy from a checkout of this repository:
  `cd relay && npm install && npx wrangler login && npx wrangler deploy` (the Worker name is the same, so the URL and the
  data stay).
- **Deployed with wrangler:** `git pull`, then `npx wrangler deploy` again.
- **Docker Compose:** `git pull && docker compose up -d --build` in `relay/selfhost`.
- **GitLab:** download the `relay/` folder of the release you want, replace your repository's files with it (keep the
  `name` in `wrangler.jsonc`), and push.
```

`README.md` 的 Sync 一節,`To run it on your own server, use Docker Compose (...)` 這句之後加上:
`To update a relay you deployed, see [Updating your relay](relay/README.md#updating-your-relay).`

- [ ] **Step 6: Commit**

```bash
git add relay/.github/workflows/update-relay.yml relay/.github/scripts/update-relay.mjs relay/.github/scripts/update-relay.test.mjs relay/.gitignore .github/workflows/relay.yml relay/README.md README.md
git commit -m "feat(relay): update a deployed relay through a pull request"
```

---

## 手動驗證(合併後,由使用者執行)

1. 用部署按鈕在測試用的 Cloudflare / GitHub 帳號部署一份新 relay,確認新 repo 裡有 `.github/workflows/update-relay.yml`
   與 `.github/scripts/update-relay.mjs`。若沒有 → 依 spec §6.5,README 的「Deployed before this workflow existed」作法
   就是唯一路徑(另開 issue 更新文件措辭)。
2. 在既有的 relay(`https://<your-relay>.workers.dev`)用 `cd relay && npx wrangler deploy` 更新,之後
   `curl -s https://<your-relay>.workers.dev/v1/info` 應回 `{"relay":"sshelter-relay","version":"0.2.0",...}`。
