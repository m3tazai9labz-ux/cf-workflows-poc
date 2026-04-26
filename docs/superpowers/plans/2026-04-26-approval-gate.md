# Approval Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Repurpose `cf-workflows-poc` from the hello-world template into `ApprovalGate` — a durable, reusable HITL approval-gate-as-a-service for any M3ta-OS caller (Hermes, Qu3bii dispatch, NemoClaw, brand pipelines), backed by Cloudflare Workflows + Supabase + Apple-native channels.

**Architecture:** Cloudflare Worker exposes `POST /request`, `GET /status/:id`, `POST /resolve/:id`. The `ApprovalGate` Workflow persists state to Supabase, calls a Mac-side shim (over Cloudflare Tunnel) to dispatch iMessage/Mail/Todoist prompts, then `waitForEvent` blocks durably until a resolver daemon parses the user's iMessage reply and POSTs back with HMAC-signed payload. Audit trail in `approval_requests` table.

**Tech Stack:** Cloudflare Workers + Workflows · TypeScript · Wrangler 4 · Vitest with `@cloudflare/vitest-pool-workers` · Supabase (Postgres) · Node.js 22 (Mac shim) · Python 3.12 (Mac resolver) · launchd · Cloudflare Tunnel

**Spec:** `docs/superpowers/specs/2026-04-26-approval-gate-design.md`

---

## File structure

**Worker (in this repo, `~/CloudDocs/03_Projects/cf-workflows-poc/`):**

| File | Action | Responsibility |
|---|---|---|
| `wrangler.jsonc` | modify | Workflow class becomes `ApprovalGate`; Worker name unchanged |
| `package.json` | modify | Add deps: `zod`, `@supabase/supabase-js`, `vitest`, `@cloudflare/vitest-pool-workers` |
| `vitest.config.ts` | create | Vitest config with Cloudflare pool |
| `migrations/001_approval_requests.sql` | create | Supabase table + indexes |
| `src/types.ts` | create | Shared TS types: `ApprovalRequest`, `Decision`, `Tier` |
| `src/auth.ts` | create | `verifyBearer`, `verifyHmac`, `signHmac` |
| `src/supabase.ts` | create | Thin wrapper: `insertPending`, `updateFinal`, `getById` |
| `src/approval-gate.ts` | create | `ApprovalGate` Workflow class with 4 durable steps |
| `src/index.ts` | replace | Worker fetch handler with 3 routes |
| `tests/auth.test.ts` | create | Bearer + HMAC unit tests |
| `tests/index.test.ts` | create | Route integration tests |
| `tests/approval-gate.test.ts` | create | Workflow step unit tests with mocked `WorkflowStep` |

**Mac-side (in this repo, `mac-side/`):**

| File | Action | Responsibility |
|---|---|---|
| `mac-side/shim/package.json` | create | Node package, hono server, deps |
| `mac-side/shim/server.ts` | create | Hono server: `POST /send` cascades through channels |
| `mac-side/shim/channels/imessage.ts` | create | iMessage send via osascript |
| `mac-side/shim/channels/mail.ts` | create | Apple Mail send via osascript |
| `mac-side/shim/channels/todoist.ts` | create | Todoist task creation via REST |
| `mac-side/shim/channels/cascade.ts` | create | Try each channel in order, return first success |
| `mac-side/shim/.env.example` | create | Env template (no real secrets) |
| `mac-side/shim/com.m3taos.approval-gate-shim.plist` | create | launchd autostart |
| `mac-side/shim/README.md` | create | Setup runbook |
| `mac-side/resolver/resolver.py` | create | Polls chat.db, parses replies, signs HMAC, POSTs to Worker |
| `mac-side/resolver/requirements.txt` | create | Python deps |
| `mac-side/resolver/.env.example` | create | Env template |
| `mac-side/resolver/com.m3taos.approval-gate-resolver.plist` | create | launchd autostart |
| `mac-side/resolver/README.md` | create | Setup runbook |

**Outside this repo (touched by tasks but lives elsewhere):**

| Path | Action |
|---|---|
| `~/.hermes/skills/m3ta-os/approval-gate.md` | replace stub with operational skill |
| `~/.m3ta-os/.env` | append `APPROVAL_GATE_TOKEN`, `APPROVAL_RESOLVE_SECRET`, `APPLE_BRIDGE_SHIM_TOKEN` |
| `~/.cloudflared/config.yml` | add tunnel ingress for shim |
| Cloudflare Tunnel | new tunnel `approval-gate-shim` |
| Cloudflare Worker secrets | 6 secrets via `wrangler secret put` |
| Supabase `m3ta-platform` | apply migration |

---

## Task 1: Project prep — deps, vitest, wrangler config

**Files:**
- Modify: `package.json`
- Modify: `wrangler.jsonc`
- Create: `vitest.config.ts`
- Create: `src/types.ts`

- [ ] **Step 1: Update `wrangler.jsonc` to point to the new Workflow class**

Edit `wrangler.jsonc` so the workflows entry has `class_name: "ApprovalGate"` (binding `MY_WORKFLOW` stays — the deployed Worker URL doesn't change; the class rename will be applied on next deploy):

```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "cf-workflows-poc",
  "main": "src/index.ts",
  "compatibility_date": "2026-04-25",
  "observability": { "enabled": true },
  "workflows": [
    {
      "name": "workflow-cf-workflows-poc",
      "binding": "MY_WORKFLOW",
      "class_name": "ApprovalGate"
    }
  ],
  "upload_source_maps": true,
  "compatibility_flags": ["nodejs_compat"]
}
```

- [ ] **Step 2: Add dependencies**

Run:

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
npm install --save zod @supabase/supabase-js
npm install --save-dev vitest@^2 @cloudflare/vitest-pool-workers
```

Expected: `package.json` updated, `node_modules/` populated, lockfile updated. No errors.

- [ ] **Step 3: Update `package.json` scripts**

Add a `test` script. After install, `package.json` should have:

```jsonc
{
  "name": "cf-workflows-poc",
  "version": "0.0.0",
  "private": true,
  "scripts": {
    "deploy": "wrangler deploy",
    "start": "wrangler dev",
    "dev": "wrangler dev",
    "test": "vitest run",
    "test:watch": "vitest",
    "cf-typegen": "wrangler types"
  }
}
```

- [ ] **Step 4: Create `vitest.config.ts`**

```typescript
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

- [ ] **Step 5: Create `src/types.ts`**

```typescript
export type Tier = "T2" | "T3" | "T4";

export type Decision = "approved" | "denied" | "timeout" | "error";

export type Status = "pending" | Decision;

export type Channel = "imessage" | "mail" | "todoist";

export interface ApprovalRequestInput {
  requester: string;
  action: string;
  payload: Record<string, unknown>;
  tier: Tier;
  timeout_seconds?: number;
  channel?: Channel;
  webhook?: string;
}

export interface ApprovalRequestRow {
  id: string;
  requester: string;
  action: string;
  payload: Record<string, unknown>;
  tier: Tier;
  status: Status;
  channel: Channel | null;
  decided_by: string | null;
  decided_at: string | null;
  reason: string | null;
  webhook: string | null;
  workflow_id: string;
  timeout_at: string;
  created_at: string;
}

export interface ResolveBody {
  decision: "approved" | "denied";
  reason?: string;
  hmac: string;
}
```

- [ ] **Step 6: Regenerate worker types**

```bash
npm run cf-typegen
```

Expected: `worker-configuration.d.ts` regenerated. The `Env` interface should now include `MY_WORKFLOW: Workflow`.

- [ ] **Step 7: Compile check**

```bash
npx tsc --noEmit
```

Expected: no type errors. (`src/index.ts` still references the old `MyWorkflow` class — we'll fix that in Task 5.)

If errors mention `MyWorkflow`, that's expected and will be resolved in Task 5. If errors mention anything else, fix before continuing.

- [ ] **Step 8: Commit**

```bash
git add wrangler.jsonc package.json package-lock.json vitest.config.ts src/types.ts worker-configuration.d.ts
git commit -m "chore: prep approval-gate — deps, vitest, types, wrangler class rename"
```

---

## Task 2: Supabase migration — `approval_requests` table

**Files:**
- Create: `migrations/001_approval_requests.sql`

- [ ] **Step 1: Write the migration**

Create `migrations/001_approval_requests.sql`:

```sql
-- 001_approval_requests.sql
-- ApprovalGate audit table — see docs/superpowers/specs/2026-04-26-approval-gate-design.md

create table if not exists approval_requests (
  id            uuid primary key default gen_random_uuid(),
  requester     text not null,
  action        text not null,
  payload       jsonb not null,
  tier          text not null check (tier in ('T2','T3','T4')),
  status        text not null default 'pending'
                check (status in ('pending','approved','denied','timeout','error')),
  channel       text check (channel in ('imessage','mail','todoist')),
  decided_by    text,
  decided_at    timestamptz,
  reason        text,
  webhook       text,
  workflow_id   text not null,
  timeout_at    timestamptz not null,
  created_at    timestamptz not null default now()
);

create index if not exists approval_requests_status_idx
  on approval_requests (status, timeout_at);

create index if not exists approval_requests_requester_idx
  on approval_requests (requester, created_at desc);

alter table approval_requests enable row level security;

-- Service-role only — internal infra, no anon access
create policy approval_requests_service_only
  on approval_requests
  for all
  using (auth.role() = 'service_role');
```

- [ ] **Step 2: Apply migration via Supabase MCP**

In a Claude Code session, invoke `mcp__plugin_supabase_supabase__apply_migration` with:
- `project_id`: `ojuxzyaydrzgkszqarcr`
- `name`: `001_approval_requests`
- `query`: contents of the SQL file above

Expected: success response with no errors. If running outside Claude Code, paste the SQL into the Supabase SQL editor for project `m3ta-platform` and run it.

- [ ] **Step 3: Verify the table**

Invoke `mcp__plugin_supabase_supabase__list_tables` with `project_id: ojuxzyaydrzgkszqarcr`, `schemas: ["public"]`.

Expected: response includes `approval_requests` with the columns/indexes/RLS policy listed above.

- [ ] **Step 4: Commit**

```bash
git add migrations/001_approval_requests.sql
git commit -m "feat(db): add approval_requests audit table"
```

---

## Task 3: Auth utilities — Bearer + HMAC (TDD)

**Files:**
- Create: `src/auth.ts`
- Create: `tests/auth.test.ts`

- [ ] **Step 1: Write the failing tests**

Create `tests/auth.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { verifyBearer, verifyHmac, signHmac } from "../src/auth";

describe("verifyBearer", () => {
  it("accepts a valid Bearer header", () => {
    expect(verifyBearer("Bearer s3cret", "s3cret")).toBe(true);
  });
  it("rejects a wrong secret", () => {
    expect(verifyBearer("Bearer wrong", "s3cret")).toBe(false);
  });
  it("rejects missing header", () => {
    expect(verifyBearer(null, "s3cret")).toBe(false);
  });
  it("rejects malformed header", () => {
    expect(verifyBearer("Token s3cret", "s3cret")).toBe(false);
  });
});

describe("signHmac / verifyHmac roundtrip", () => {
  const secret = "test-secret-32-bytes-of-entropy-go";

  it("signs and verifies the same payload", async () => {
    const sig = await signHmac("abc123:approved:looks good", secret);
    const ok = await verifyHmac("abc123:approved:looks good", sig, secret);
    expect(ok).toBe(true);
  });

  it("rejects modified payload", async () => {
    const sig = await signHmac("abc123:approved:", secret);
    const ok = await verifyHmac("abc123:denied:", sig, secret);
    expect(ok).toBe(false);
  });

  it("rejects modified signature", async () => {
    const sig = await signHmac("abc123:approved:", secret);
    const tampered = sig.slice(0, -2) + "00";
    const ok = await verifyHmac("abc123:approved:", tampered, secret);
    expect(ok).toBe(false);
  });

  it("rejects different secret", async () => {
    const sig = await signHmac("abc123:approved:", secret);
    const ok = await verifyHmac("abc123:approved:", sig, "different-secret");
    expect(ok).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm test -- auth
```

Expected: all 8 tests fail with "Cannot find module '../src/auth'".

- [ ] **Step 3: Implement `src/auth.ts`**

```typescript
// Constant-time comparison via Web Crypto. Workers runtime supports
// `crypto.subtle` and `crypto.timingSafeEqual` is not available; we
// implement byte-equal compare manually after decoding.

export function verifyBearer(
  authHeader: string | null,
  expected: string,
): boolean {
  if (!authHeader) return false;
  const [scheme, token] = authHeader.split(" ", 2);
  if (scheme !== "Bearer" || !token) return false;
  return constantTimeEqual(token, expected);
}

export async function signHmac(payload: string, secret: string): Promise<string> {
  const key = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(secret),
    { name: "HMAC", hash: "SHA-256" },
    false,
    ["sign"],
  );
  const sig = await crypto.subtle.sign("HMAC", key, new TextEncoder().encode(payload));
  return bufferToHex(sig);
}

export async function verifyHmac(
  payload: string,
  hexSignature: string,
  secret: string,
): Promise<boolean> {
  if (!/^[0-9a-f]+$/i.test(hexSignature)) return false;
  const expected = await signHmac(payload, secret);
  return constantTimeEqual(hexSignature.toLowerCase(), expected.toLowerCase());
}

function constantTimeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) {
    diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return diff === 0;
}

function bufferToHex(buf: ArrayBuffer): string {
  return [...new Uint8Array(buf)]
    .map((b) => b.toString(16).padStart(2, "0"))
    .join("");
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm test -- auth
```

Expected: all 8 tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/auth.ts tests/auth.test.ts
git commit -m "feat(auth): Bearer + HMAC verification with constant-time compare"
```

---

## Task 4: Supabase client wrapper

**Files:**
- Create: `src/supabase.ts`

- [ ] **Step 1: Write the wrapper**

```typescript
import { createClient, type SupabaseClient } from "@supabase/supabase-js";
import type { ApprovalRequestRow, Channel, Decision, Status, Tier } from "./types";

export function getClient(env: { SUPABASE_URL: string; SUPABASE_SERVICE_KEY: string }): SupabaseClient {
  return createClient(env.SUPABASE_URL, env.SUPABASE_SERVICE_KEY, {
    auth: { persistSession: false },
  });
}

export async function insertPending(
  client: SupabaseClient,
  row: {
    requester: string;
    action: string;
    payload: Record<string, unknown>;
    tier: Tier;
    timeout_at: string;
    workflow_id: string;
    webhook: string | null;
  },
): Promise<ApprovalRequestRow> {
  const { data, error } = await client
    .from("approval_requests")
    .insert({ ...row, status: "pending" as Status })
    .select()
    .single();
  if (error) throw new Error(`insertPending: ${error.message}`);
  return data as ApprovalRequestRow;
}

export async function updateFinal(
  client: SupabaseClient,
  id: string,
  patch: {
    status: Decision;
    channel: Channel | null;
    decided_by: string | null;
    decided_at: string | null;
    reason: string | null;
  },
): Promise<void> {
  const { error } = await client
    .from("approval_requests")
    .update(patch)
    .eq("id", id);
  if (error) throw new Error(`updateFinal: ${error.message}`);
}

export async function getById(
  client: SupabaseClient,
  id: string,
): Promise<ApprovalRequestRow | null> {
  const { data, error } = await client
    .from("approval_requests")
    .select("*")
    .eq("id", id)
    .maybeSingle();
  if (error) throw new Error(`getById: ${error.message}`);
  return data as ApprovalRequestRow | null;
}
```

- [ ] **Step 2: Compile check**

```bash
npx tsc --noEmit
```

Expected: no errors in `src/supabase.ts`. (Errors elsewhere are still expected from old template code that Task 5 replaces.)

- [ ] **Step 3: Commit**

```bash
git add src/supabase.ts
git commit -m "feat(db): Supabase wrapper for approval_requests"
```

---

## Task 5: ApprovalGate workflow class

**Files:**
- Replace: `src/approval-gate.ts`
- Modify: `src/index.ts` (will be fully replaced in Task 6; for now just remove the old `MyWorkflow` class to unblock compile)

- [ ] **Step 1: Delete the old `MyWorkflow` from `src/index.ts`**

Strip the file down to a placeholder default export so the project compiles while we build out `ApprovalGate`:

```typescript
// Placeholder — full Worker fetch handler implemented in Task 6.
// MyWorkflow has been replaced by ApprovalGate (see src/approval-gate.ts).
import { ApprovalGate } from "./approval-gate";
export { ApprovalGate };

export default {
  async fetch(_req: Request, _env: Env): Promise<Response> {
    return new Response("not implemented", { status: 501 });
  },
};
```

- [ ] **Step 2: Create `src/approval-gate.ts` with the full workflow**

```typescript
import {
  WorkflowEntrypoint,
  WorkflowEvent,
  WorkflowStep,
} from "cloudflare:workers";
import { getClient, insertPending, updateFinal } from "./supabase";
import type { ApprovalRequestInput, Channel, Decision, Tier } from "./types";

export interface ApprovalGateParams {
  requester: string;
  action: string;
  payload: Record<string, unknown>;
  tier: Tier;
  timeout_seconds: number;
  channel?: Channel;
  webhook: string | null;
  // Pre-allocated row id from /request handler so the response can return it
  // before the Workflow has run. The handler inserts the row in Supabase too,
  // but we re-insert defensively here to keep the Workflow self-contained when
  // someone replays from outside the normal entrypoint.
  request_id: string;
}

export interface DecisionEvent {
  decision: "approved" | "denied";
  reason?: string;
  // Note: we do NOT carry a `channel` field on the decision event. The Workflow
  // already knows which channel the prompt was sent on (`sent` local). v1
  // semantics: `approval_requests.channel` = "channel the prompt landed on",
  // not "channel the user replied via". The resolver only listens to iMessage,
  // so resolution-channel is always implicit; if we ever support multi-channel
  // resolution we'll add it here.
}

const CHANNEL_CASCADE: Channel[] = ["imessage", "mail", "todoist"];

export class ApprovalGate extends WorkflowEntrypoint<Env, ApprovalGateParams> {
  async run(event: WorkflowEvent<ApprovalGateParams>, step: WorkflowStep) {
    const params = event.payload;
    const id = params.request_id;
    const sb = getClient(this.env);

    // Step 1 — persist pending row (idempotent: row was inserted by /request,
    // but we upsert here for safety so partial failures still self-heal).
    await step.do(
      "persist-pending",
      { retries: { limit: 5, delay: "2 second", backoff: "exponential" } },
      async () => {
        await insertPending(sb, {
          requester: params.requester,
          action: params.action,
          payload: params.payload,
          tier: params.tier,
          timeout_at: new Date(Date.now() + params.timeout_seconds * 1000).toISOString(),
          workflow_id: event.instanceId,
          webhook: params.webhook,
        }).catch((e: Error) => {
          // If row already exists (insert from /request handler), that's fine.
          if (!e.message.includes("duplicate") && !e.message.includes("unique")) throw e;
        });
      },
    );

    // Step 2 — send prompt via Mac shim, walking cascade until one succeeds.
    const channelsToTry = params.channel
      ? [params.channel]
      : CHANNEL_CASCADE;

    const sent = await step.do(
      "send-prompt",
      { retries: { limit: 3, delay: "5 second", backoff: "exponential" }, timeout: "30 seconds" },
      async () => {
        for (const ch of channelsToTry) {
          try {
            const resp = await fetch(`${this.env.APPLE_BRIDGE_SHIM_URL}/send`, {
              method: "POST",
              headers: {
                "Authorization": `Bearer ${this.env.APPLE_BRIDGE_SHIM_TOKEN}`,
                "Content-Type": "application/json",
              },
              body: JSON.stringify({
                channel: ch,
                id_short: id.slice(0, 6),
                tier: params.tier,
                action: params.action,
                payload_summary: summarize(params.payload),
                timeout_minutes: Math.round(params.timeout_seconds / 60),
              }),
            });
            if (resp.ok) return ch;
          } catch (_) {
            // try next channel
          }
        }
        throw new Error("all channels failed");
      },
    );

    // Step 3 — durable wait for decision event.
    let decisionEvent: DecisionEvent | null = null;
    let timedOut = false;
    try {
      const ev = await step.waitForEvent<DecisionEvent>("await-decision", {
        type: "decision",
        timeout: `${params.timeout_seconds} seconds`,
      });
      decisionEvent = ev.payload;
    } catch (_) {
      timedOut = true;
    }

    // Step 4 — persist final row.
    const finalStatus: Decision = timedOut
      ? "timeout"
      : decisionEvent!.decision;

    await step.do(
      "persist-final",
      { retries: { limit: 5, delay: "2 second", backoff: "exponential" } },
      async () => {
        await updateFinal(sb, id, {
          status: finalStatus,
          channel: sent, // channel the prompt was actually sent on
          decided_by: timedOut ? null : "coach-meta",
          decided_at: timedOut ? null : new Date().toISOString(),
          reason: timedOut ? null : (decisionEvent!.reason ?? null),
        });
      },
    );

    // Step 5 — optional caller webhook.
    if (params.webhook) {
      await step.do(
        "notify-caller",
        { retries: { limit: 3, delay: "30 second", backoff: "exponential" }, timeout: "10 seconds" },
        async () => {
          await fetch(params.webhook!, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              id,
              status: finalStatus,
              channel: sent,
              reason: timedOut ? null : decisionEvent!.reason,
            }),
          });
        },
      );
    }
  }
}

function summarize(payload: Record<string, unknown>): string {
  // Compact summary for inclusion in iMessage / mail body.
  // Trims nested objects to one level of stringified preview.
  const lines: string[] = [];
  for (const [k, v] of Object.entries(payload)) {
    if (typeof v === "string") {
      lines.push(`${k}: ${v.length > 80 ? v.slice(0, 77) + "…" : v}`);
    } else {
      lines.push(`${k}: ${JSON.stringify(v).slice(0, 80)}`);
    }
  }
  return lines.join("\n");
}
```

- [ ] **Step 3: Compile check**

```bash
npx tsc --noEmit
```

Expected: no errors. If `Env` is missing fields, run `npm run cf-typegen` and re-check; type generation pulls from `wrangler.jsonc`. If `Env` doesn't have `APPLE_BRIDGE_SHIM_URL`, `APPLE_BRIDGE_SHIM_TOKEN`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` — those are secrets, so they'll appear in `Env` only after `wrangler secret put` (Task 9). For now, add them to `Env` manually in `worker-configuration.d.ts` if needed, or use `(this.env as unknown as { ... })` in the code above.

To unblock typecheck without deploying secrets, add a temporary `env.d.ts` next to `src/`:

```typescript
// src/env.d.ts — augments the Env interface with secret bindings until they're set.
interface Env {
  MY_WORKFLOW: Workflow;
  APPLE_BRIDGE_SHIM_URL: string;
  APPLE_BRIDGE_SHIM_TOKEN: string;
  SUPABASE_URL: string;
  SUPABASE_SERVICE_KEY: string;
  APPROVAL_GATE_TOKEN: string;
  APPROVAL_RESOLVE_SECRET: string;
}
```

- [ ] **Step 4: Commit**

```bash
git add src/approval-gate.ts src/index.ts src/env.d.ts
git commit -m "feat(workflow): ApprovalGate with persist/send/wait/finalize/notify steps"
```

---

## Task 6: Worker fetch handler — `POST /request`

**Files:**
- Replace: `src/index.ts`
- Create: `tests/index.test.ts`

- [ ] **Step 1: Write the failing test**

Create `tests/index.test.ts`:

```typescript
import { describe, it, expect, beforeAll } from "vitest";
import { SELF, env } from "cloudflare:test";

describe("POST /request", () => {
  beforeAll(() => {
    // vitest-pool-workers exposes env from wrangler config; secrets must be
    // provided via .dev.vars during test. See wrangler docs.
  });

  it("rejects missing Authorization header with 401", async () => {
    const resp = await SELF.fetch("https://example.com/request", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({}),
    });
    expect(resp.status).toBe(401);
  });

  it("rejects malformed body with 400", async () => {
    const resp = await SELF.fetch("https://example.com/request", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: "not json",
    });
    expect(resp.status).toBe(400);
  });

  it("creates a workflow instance and returns 202 with id", async () => {
    const resp = await SELF.fetch("https://example.com/request", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        requester: "test:vitest",
        action: "test-action",
        payload: { foo: "bar" },
        tier: "T2",
        timeout_seconds: 60,
      }),
    });
    expect(resp.status).toBe(202);
    const body = await resp.json<{ id: string; status: string }>();
    expect(body.id).toMatch(/^[0-9a-f-]{36}$/);
    expect(body.status).toBe("pending");
  });
});
```

- [ ] **Step 2: Add `.dev.vars` for tests**

```bash
cat > .dev.vars <<'EOF'
APPROVAL_GATE_TOKEN=test-bearer-token-not-real
APPROVAL_RESOLVE_SECRET=test-resolve-secret-not-real-32b
SUPABASE_URL=https://ojuxzyaydrzgkszqarcr.supabase.co
SUPABASE_SERVICE_KEY=test-service-key-replaced-in-prod
APPLE_BRIDGE_SHIM_URL=http://localhost:9999
APPLE_BRIDGE_SHIM_TOKEN=test-shim-token-not-real
EOF
```

`.dev.vars` is in `.gitignore` already (Task 0 setup). Confirm with `git check-ignore .dev.vars` — should print `.dev.vars`.

- [ ] **Step 3: Run tests to verify they fail**

```bash
npm test -- index
```

Expected: `creates a workflow instance` test fails because `POST /request` is not yet implemented (returns 501).

- [ ] **Step 4: Implement `POST /request` route in `src/index.ts`**

Replace `src/index.ts` with:

```typescript
import { z } from "zod";
import { ApprovalGate } from "./approval-gate";
import { verifyBearer, verifyHmac } from "./auth";
import { getClient, getById, insertPending, updateFinal } from "./supabase";
import type { ApprovalRequestInput, Channel, ResolveBody, Tier } from "./types";

export { ApprovalGate };

const RequestSchema = z.object({
  requester: z.string().min(1).max(64),
  action: z.string().min(1).max(64),
  payload: z.record(z.unknown()),
  tier: z.enum(["T2", "T3", "T4"]),
  timeout_seconds: z.number().int().positive().max(86400).optional(),
  channel: z.enum(["imessage", "mail", "todoist"]).optional(),
  webhook: z.string().url().optional(),
});

export default {
  async fetch(req: Request, env: Env): Promise<Response> {
    const url = new URL(req.url);

    // POST /request
    if (req.method === "POST" && url.pathname === "/request") {
      if (!verifyBearer(req.headers.get("Authorization"), env.APPROVAL_GATE_TOKEN)) {
        return json({ error: "unauthorized" }, 401);
      }

      let body: unknown;
      try {
        body = await req.json();
      } catch {
        return json({ error: "invalid json" }, 400);
      }
      const parsed = RequestSchema.safeParse(body);
      if (!parsed.success) {
        return json({ error: "validation_failed", issues: parsed.error.issues }, 400);
      }

      const input = parsed.data;
      const timeout = input.timeout_seconds ?? 3600;
      const id = crypto.randomUUID();
      const timeoutAt = new Date(Date.now() + timeout * 1000).toISOString();

      // Pre-insert the pending row so /status/:id works immediately.
      const sb = getClient(env);
      try {
        await insertPending(sb, {
          requester: input.requester,
          action: input.action,
          payload: input.payload,
          tier: input.tier,
          timeout_at: timeoutAt,
          workflow_id: id,
          webhook: input.webhook ?? null,
        });
      } catch (e) {
        return json({ error: "db_error", detail: (e as Error).message }, 500);
      }

      // Spawn the workflow with the row id as instance id (so /resolve/:id can
      // call MY_WORKFLOW.get(id).sendEvent("decision", ...)).
      await env.MY_WORKFLOW.create({
        id,
        params: {
          requester: input.requester,
          action: input.action,
          payload: input.payload,
          tier: input.tier,
          timeout_seconds: timeout,
          channel: input.channel,
          webhook: input.webhook ?? null,
          request_id: id,
        },
      });

      return json(
        {
          id,
          status: "pending",
          status_url: `${url.origin}/status/${id}`,
          timeout_at: timeoutAt,
        },
        202,
      );
    }

    return json({ error: "not_found" }, 404);
  },
};

function json(obj: unknown, status: number = 200): Response {
  return new Response(JSON.stringify(obj), {
    status,
    headers: { "Content-Type": "application/json" },
  });
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
npm test -- index
```

Expected: 401, 400, and 202 tests all pass. The 202 path will hit the real Supabase from inside the worker pool — that's fine for now since the test uses a temporary requester and the row is harmless. If Supabase calls hang the test, mock the `getClient` import via vitest's `vi.mock("../src/supabase")` and assert the mock got called.

- [ ] **Step 6: Commit**

```bash
git add src/index.ts tests/index.test.ts
git commit -m "feat(api): POST /request route — validate, persist, spawn workflow"
```

---

## Task 7: Worker fetch handler — `GET /status/:id`

**Files:**
- Modify: `src/index.ts`
- Modify: `tests/index.test.ts`

- [ ] **Step 1: Add the failing tests**

Append to `tests/index.test.ts`:

```typescript
describe("GET /status/:id", () => {
  it("rejects missing Authorization with 401", async () => {
    const resp = await SELF.fetch("https://example.com/status/00000000-0000-0000-0000-000000000000");
    expect(resp.status).toBe(401);
  });

  it("returns 404 for unknown id", async () => {
    const resp = await SELF.fetch(
      "https://example.com/status/00000000-0000-0000-0000-000000000000",
      { headers: { "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}` } },
    );
    expect(resp.status).toBe(404);
  });

  it("returns 200 with status row for a freshly created request", async () => {
    const create = await SELF.fetch("https://example.com/request", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        requester: "test:vitest",
        action: "status-test",
        payload: {},
        tier: "T2",
        timeout_seconds: 60,
      }),
    });
    const { id } = await create.json<{ id: string }>();

    const resp = await SELF.fetch(`https://example.com/status/${id}`, {
      headers: { "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}` },
    });
    expect(resp.status).toBe(200);
    const body = await resp.json<{ id: string; status: string }>();
    expect(body.id).toBe(id);
    expect(body.status).toBe("pending");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm test -- index
```

Expected: the three new tests fail (route not implemented → 404 from default handler).

- [ ] **Step 3: Implement the route**

In `src/index.ts`, before the final `return json({ error: "not_found" }, 404);`, add:

```typescript
    // GET /status/:id
    const statusMatch = url.pathname.match(/^\/status\/([0-9a-f-]{36})$/);
    if (req.method === "GET" && statusMatch) {
      if (!verifyBearer(req.headers.get("Authorization"), env.APPROVAL_GATE_TOKEN)) {
        return json({ error: "unauthorized" }, 401);
      }
      const id = statusMatch[1];
      const sb = getClient(env);
      const row = await getById(sb, id);
      if (!row) return json({ error: "not_found" }, 404);
      return json({
        id: row.id,
        status: row.status,
        decided_by: row.decided_by,
        decided_at: row.decided_at,
        reason: row.reason,
        channel: row.channel,
      });
    }
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm test -- index
```

Expected: all `GET /status/:id` tests pass.

- [ ] **Step 5: Commit**

```bash
git add src/index.ts tests/index.test.ts
git commit -m "feat(api): GET /status/:id route"
```

---

## Task 8: Worker fetch handler — `POST /resolve/:id`

**Files:**
- Modify: `src/index.ts`
- Modify: `tests/index.test.ts`

- [ ] **Step 1: Add the failing tests**

Append to `tests/index.test.ts`:

```typescript
import { signHmac } from "../src/auth";

describe("POST /resolve/:id", () => {
  async function createRequest(): Promise<string> {
    const create = await SELF.fetch("https://example.com/request", {
      method: "POST",
      headers: {
        "Authorization": `Bearer ${env.APPROVAL_GATE_TOKEN}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        requester: "test:vitest",
        action: "resolve-test",
        payload: {},
        tier: "T2",
        timeout_seconds: 60,
      }),
    });
    return (await create.json<{ id: string }>()).id;
  }

  it("rejects bad HMAC with 401", async () => {
    const id = await createRequest();
    const resp = await SELF.fetch(`https://example.com/resolve/${id}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ decision: "approved", hmac: "deadbeef" }),
    });
    expect(resp.status).toBe(401);
  });

  it("accepts valid HMAC and returns 200", async () => {
    const id = await createRequest();
    const payload = `${id}:approved:`;
    const hmac = await signHmac(payload, env.APPROVAL_RESOLVE_SECRET);
    const resp = await SELF.fetch(`https://example.com/resolve/${id}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ decision: "approved", hmac }),
    });
    expect(resp.status).toBe(200);
  });

  it("rejects unknown id with 404", async () => {
    const fakeId = "00000000-0000-0000-0000-000000000000";
    const payload = `${fakeId}:approved:`;
    const hmac = await signHmac(payload, env.APPROVAL_RESOLVE_SECRET);
    const resp = await SELF.fetch(`https://example.com/resolve/${fakeId}`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ decision: "approved", hmac }),
    });
    expect(resp.status).toBe(404);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm test -- index
```

Expected: 3 new tests fail.

- [ ] **Step 3: Implement the route**

Add to `src/index.ts` (in the same chain, before the final 404):

```typescript
    // POST /resolve/:id
    const resolveMatch = url.pathname.match(/^\/resolve\/([0-9a-f-]{36})$/);
    if (req.method === "POST" && resolveMatch) {
      const id = resolveMatch[1];
      let body: ResolveBody;
      try {
        body = (await req.json()) as ResolveBody;
      } catch {
        return json({ error: "invalid json" }, 400);
      }
      if (!body.decision || !body.hmac) {
        return json({ error: "validation_failed" }, 400);
      }
      if (body.decision !== "approved" && body.decision !== "denied") {
        return json({ error: "validation_failed" }, 400);
      }
      const signedPayload = `${id}:${body.decision}:${body.reason ?? ""}`;
      const hmacOk = await verifyHmac(signedPayload, body.hmac, env.APPROVAL_RESOLVE_SECRET);
      if (!hmacOk) return json({ error: "unauthorized" }, 401);

      const sb = getClient(env);
      const row = await getById(sb, id);
      if (!row) return json({ error: "not_found" }, 404);
      if (row.status !== "pending") return json({ error: "already_resolved", status: row.status }, 409);

      // Send decision event to workflow instance.
      const instance = await env.MY_WORKFLOW.get(id);
      await instance.sendEvent({
        type: "decision",
        payload: {
          decision: body.decision,
          reason: body.reason,
        },
      });

      return json({ id, status: body.decision });
    }
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npm test -- index
```

Expected: all resolve tests pass. The "valid HMAC" test will succeed even though the underlying workflow may not have time to update Supabase before the test ends — the route returns 200 as soon as the event is dispatched.

- [ ] **Step 5: Commit**

```bash
git add src/index.ts tests/index.test.ts
git commit -m "feat(api): POST /resolve/:id route with HMAC verify"
```

---

## Task 9: Cloudflare Worker secrets + deploy

**Files:**
- (none — this is a deploy + secret-management task)

- [ ] **Step 1: Generate strong secrets locally**

```bash
APPROVAL_GATE_TOKEN=$(openssl rand -hex 32)
APPROVAL_RESOLVE_SECRET=$(openssl rand -hex 32)
APPLE_BRIDGE_SHIM_TOKEN=$(openssl rand -hex 32)
echo "APPROVAL_GATE_TOKEN=$APPROVAL_GATE_TOKEN"
echo "APPROVAL_RESOLVE_SECRET=$APPROVAL_RESOLVE_SECRET"
echo "APPLE_BRIDGE_SHIM_TOKEN=$APPLE_BRIDGE_SHIM_TOKEN"
```

- [ ] **Step 2: Append the secrets to `~/.m3ta-os/.env`**

```bash
{
  echo ""
  echo "# Approval Gate (cf-workflows-poc) — added $(date +%Y-%m-%d)"
  echo "APPROVAL_GATE_TOKEN=$APPROVAL_GATE_TOKEN"
  echo "APPROVAL_RESOLVE_SECRET=$APPROVAL_RESOLVE_SECRET"
  echo "APPLE_BRIDGE_SHIM_TOKEN=$APPLE_BRIDGE_SHIM_TOKEN"
} >> ~/.m3ta-os/.env
chmod 600 ~/.m3ta-os/.env
```

- [ ] **Step 3: Set Cloudflare Worker secrets**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
echo "$APPROVAL_GATE_TOKEN" | wrangler secret put APPROVAL_GATE_TOKEN
echo "$APPROVAL_RESOLVE_SECRET" | wrangler secret put APPROVAL_RESOLVE_SECRET
echo "$APPLE_BRIDGE_SHIM_TOKEN" | wrangler secret put APPLE_BRIDGE_SHIM_TOKEN
echo "https://ojuxzyaydrzgkszqarcr.supabase.co" | wrangler secret put SUPABASE_URL
# Get the Supabase service-role key from project settings; do NOT echo it to history
read -s -p "SUPABASE_SERVICE_KEY: " SUPABASE_SERVICE_KEY
echo "$SUPABASE_SERVICE_KEY" | wrangler secret put SUPABASE_SERVICE_KEY
unset SUPABASE_SERVICE_KEY
# APPLE_BRIDGE_SHIM_URL is set later (Task 12) once the tunnel is up
```

Expected: each `wrangler secret put` returns `Success!`.

- [ ] **Step 4: Verify secrets are set**

```bash
wrangler secret list
```

Expected: 5 secrets listed (`APPROVAL_GATE_TOKEN`, `APPROVAL_RESOLVE_SECRET`, `APPLE_BRIDGE_SHIM_TOKEN`, `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`). `APPLE_BRIDGE_SHIM_URL` will be added in Task 12.

- [ ] **Step 5: Deploy**

```bash
npm run deploy
```

Expected: deploy succeeds; URL printed is `https://cf-workflows-poc.coachmeta-158.workers.dev`. Note: until Task 12 sets `APPLE_BRIDGE_SHIM_URL`, `step.do "send-prompt"` will fail at runtime — that's fine, no real callers are hitting the gate yet.

- [ ] **Step 6: Smoke-test the deployed `/request` endpoint**

```bash
curl -i -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "smoke:curl",
    "action": "smoke-test",
    "payload": {"foo":"bar"},
    "tier": "T2",
    "timeout_seconds": 60
  }'
```

Expected: HTTP 202 with a JSON body containing `id`, `status: "pending"`, `status_url`, `timeout_at`.

- [ ] **Step 7: Commit deploy notes**

No code changes from this task. Skip the commit if there are no uncommitted files. Otherwise:

```bash
git status
```

If anything's tracked-and-modified (e.g., a generated `worker-configuration.d.ts`), commit it:

```bash
git add -A
git commit -m "chore(deploy): set Worker secrets and ship approval-gate v1"
```

---

## Task 10: Mac shim — scaffold + iMessage channel

**Files:**
- Create: `mac-side/shim/package.json`
- Create: `mac-side/shim/server.ts`
- Create: `mac-side/shim/channels/imessage.ts`
- Create: `mac-side/shim/.env.example`

- [ ] **Step 1: Scaffold the Node project**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
mkdir -p mac-side/shim/channels
cd mac-side/shim
npm init -y
npm install hono @hono/node-server zod
npm install --save-dev typescript @types/node tsx
npx tsc --init --target ES2022 --module ESNext --moduleResolution bundler --strict --esModuleInterop --rootDir . --outDir dist
```

- [ ] **Step 2: Update `mac-side/shim/package.json` scripts**

```jsonc
{
  "name": "approval-gate-shim",
  "type": "module",
  "scripts": {
    "dev": "tsx watch server.ts",
    "start": "tsx server.ts"
  }
}
```

- [ ] **Step 3: Create `mac-side/shim/.env.example`**

```bash
cat > .env.example <<'EOF'
APPLE_BRIDGE_SHIM_TOKEN=set-via-~/.m3ta-os/.env
TODOIST_API_TOKEN=set-via-~/.m3ta-os/.env
PORT=9999
EOF
```

- [ ] **Step 4: Create `mac-side/shim/channels/imessage.ts`**

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const exec = promisify(execFile);

export async function sendIMessage(args: {
  recipient: string;
  body: string;
}): Promise<void> {
  // AppleScript escapes — use heredoc-style indirection via osascript -e.
  const script = `
    tell application "Messages"
      set targetService to first service whose service type = iMessage
      set targetBuddy to buddy "${args.recipient}" of targetService
      send "${args.body.replace(/["\\]/g, "\\$&")}" to targetBuddy
    end tell
  `;
  await exec("osascript", ["-e", script]);
}
```

- [ ] **Step 5: Create `mac-side/shim/server.ts` with `POST /send` for the iMessage channel only (cascade comes in Task 11)**

```typescript
import { Hono } from "hono";
import { serve } from "@hono/node-server";
import { z } from "zod";
import { sendIMessage } from "./channels/imessage.js";

const PORT = Number(process.env.PORT ?? 9999);
const TOKEN = process.env.APPLE_BRIDGE_SHIM_TOKEN;
if (!TOKEN) {
  console.error("APPLE_BRIDGE_SHIM_TOKEN required");
  process.exit(1);
}
const RECIPIENT = process.env.APPROVAL_RECIPIENT ?? "coachmeta@eagleeyevisionlabz.com";

const SendSchema = z.object({
  channel: z.enum(["imessage", "mail", "todoist"]),
  id_short: z.string().regex(/^[0-9a-f]{6}$/),
  tier: z.enum(["T2", "T3", "T4"]),
  action: z.string().min(1).max(64),
  payload_summary: z.string().max(2000),
  timeout_minutes: z.number().int().positive(),
});

const app = new Hono();

app.post("/send", async (c) => {
  const auth = c.req.header("Authorization");
  if (auth !== `Bearer ${TOKEN}`) return c.json({ error: "unauthorized" }, 401);

  const parsed = SendSchema.safeParse(await c.req.json().catch(() => null));
  if (!parsed.success) return c.json({ error: "validation_failed" }, 400);

  const { channel, id_short, tier, action, payload_summary, timeout_minutes } = parsed.data;
  const body = [
    `🔐 Approval needed (${tier})`,
    `Action: ${action}`,
    payload_summary,
    `Reply: approve ${id_short}  OR  deny ${id_short} reason: ...`,
    `Times out in ${timeout_minutes}m.`,
  ].join("\n");

  if (channel === "imessage") {
    try {
      await sendIMessage({ recipient: RECIPIENT, body });
      return c.json({ ok: true, channel: "imessage" });
    } catch (e) {
      return c.json({ ok: false, error: (e as Error).message }, 502);
    }
  }
  return c.json({ ok: false, error: `channel ${channel} not yet implemented` }, 501);
});

app.get("/health", (c) => c.json({ ok: true }));

serve({ fetch: app.fetch, port: PORT });
console.log(`shim listening on ${PORT}`);
```

- [ ] **Step 6: Smoke test the shim locally**

In one terminal:

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc/mac-side/shim
APPLE_BRIDGE_SHIM_TOKEN=test-local APPROVAL_RECIPIENT="$YOUR_OWN_IMESSAGE_HANDLE" npm run dev
```

In another terminal:

```bash
curl -X POST http://localhost:9999/send \
  -H "Authorization: Bearer test-local" \
  -H "Content-Type: application/json" \
  -d '{
    "channel":"imessage",
    "id_short":"abc123",
    "tier":"T2",
    "action":"smoke-test",
    "payload_summary":"foo: bar",
    "timeout_minutes":5
  }'
```

Expected: HTTP 200 `{"ok":true,"channel":"imessage"}` and an iMessage arrives on your own number/email. If you're not on iMessage, the AppleScript will fail — that's a real signal, not a test bug.

- [ ] **Step 7: Commit**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
git add mac-side/shim
git commit -m "feat(shim): scaffold Mac-side shim with iMessage channel"
```

---

## Task 11: Mac shim — Mail + Todoist + cascade

**Files:**
- Create: `mac-side/shim/channels/mail.ts`
- Create: `mac-side/shim/channels/todoist.ts`
- Create: `mac-side/shim/channels/cascade.ts`
- Modify: `mac-side/shim/server.ts`

- [ ] **Step 1: Implement `channels/mail.ts`**

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const exec = promisify(execFile);

export async function sendMail(args: {
  to: string;
  subject: string;
  body: string;
}): Promise<void> {
  const escape = (s: string) => s.replace(/["\\]/g, "\\$&");
  const script = `
    tell application "Mail"
      set newMsg to make new outgoing message with properties {subject:"${escape(args.subject)}", content:"${escape(args.body)}", visible:false}
      tell newMsg
        make new to recipient at end of to recipients with properties {address:"${escape(args.to)}"}
        send
      end tell
    end tell
  `;
  await exec("osascript", ["-e", script]);
}
```

- [ ] **Step 2: Implement `channels/todoist.ts`**

```typescript
export async function sendTodoist(args: {
  token: string;
  content: string;
  description: string;
  labels?: string[];
}): Promise<void> {
  const resp = await fetch("https://api.todoist.com/api/v1/tasks", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${args.token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      content: args.content,
      description: args.description,
      labels: args.labels ?? ["approval-pending"],
    }),
  });
  if (!resp.ok) {
    throw new Error(`todoist ${resp.status}: ${await resp.text()}`);
  }
}
```

- [ ] **Step 3: Implement `channels/cascade.ts`**

```typescript
import { sendIMessage } from "./imessage.js";
import { sendMail } from "./mail.js";
import { sendTodoist } from "./todoist.js";

export type Channel = "imessage" | "mail" | "todoist";

export interface CascadeArgs {
  channels: Channel[];
  imessageRecipient: string;
  mailTo: string;
  todoistToken: string | undefined;
  body: string;
  subject: string;
  todoistContent: string;
}

export async function cascade(args: CascadeArgs): Promise<{ channel: Channel } | { error: string }> {
  for (const ch of args.channels) {
    try {
      if (ch === "imessage") {
        await sendIMessage({ recipient: args.imessageRecipient, body: args.body });
      } else if (ch === "mail") {
        await sendMail({ to: args.mailTo, subject: args.subject, body: args.body });
      } else if (ch === "todoist") {
        if (!args.todoistToken) throw new Error("todoist token missing");
        await sendTodoist({
          token: args.todoistToken,
          content: args.todoistContent,
          description: args.body,
        });
      }
      return { channel: ch };
    } catch (_) {
      // try next
    }
  }
  return { error: "all channels failed" };
}
```

- [ ] **Step 4: Update `server.ts` to use the cascade**

Replace the `app.post("/send", ...)` handler:

```typescript
import { cascade, type Channel } from "./channels/cascade.js";

const RECIPIENT = process.env.APPROVAL_RECIPIENT ?? "coachmeta@eagleeyevisionlabz.com";
const MAIL_TO = process.env.APPROVAL_MAIL_TO ?? "coachmeta@eagleeyevisionlabz.com";
const TODOIST_TOKEN = process.env.TODOIST_API_TOKEN;

const DEFAULT_CASCADE: Channel[] = ["imessage", "mail", "todoist"];

app.post("/send", async (c) => {
  const auth = c.req.header("Authorization");
  if (auth !== `Bearer ${TOKEN}`) return c.json({ error: "unauthorized" }, 401);

  const parsed = SendSchema.safeParse(await c.req.json().catch(() => null));
  if (!parsed.success) return c.json({ error: "validation_failed" }, 400);

  const { channel, id_short, tier, action, payload_summary, timeout_minutes } = parsed.data;
  const body = [
    `🔐 Approval needed (${tier})`,
    `Action: ${action}`,
    payload_summary,
    `Reply: approve ${id_short}  OR  deny ${id_short} reason: ...`,
    `Times out in ${timeout_minutes}m.`,
  ].join("\n");
  const subject = `Approval needed (${tier}): ${action} [${id_short}]`;
  const todoistContent = `Approve ${id_short}: ${action} (${tier})`;

  const channels: Channel[] = channel ? [channel] : DEFAULT_CASCADE;
  const result = await cascade({
    channels,
    imessageRecipient: RECIPIENT,
    mailTo: MAIL_TO,
    todoistToken: TODOIST_TOKEN,
    body,
    subject,
    todoistContent,
  });

  if ("error" in result) return c.json({ ok: false, error: result.error }, 502);
  return c.json({ ok: true, channel: result.channel });
});
```

- [ ] **Step 5: Smoke test the cascade**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc/mac-side/shim
APPLE_BRIDGE_SHIM_TOKEN=test-local TODOIST_API_TOKEN="$TODOIST_API_TOKEN" npm run dev
```

```bash
# Force fallback by passing a channel that fails — easiest is an explicit channel
curl -X POST http://localhost:9999/send \
  -H "Authorization: Bearer test-local" -H "Content-Type: application/json" \
  -d '{"channel":"todoist","id_short":"abc123","tier":"T2","action":"cascade-test","payload_summary":"x: y","timeout_minutes":5}'
```

Expected: HTTP 200 `{"ok":true,"channel":"todoist"}` and a new task in Todoist tagged `@approval-pending`.

- [ ] **Step 6: Commit**

```bash
git add mac-side/shim
git commit -m "feat(shim): mail + todoist channels with cascade"
```

---

## Task 12: Cloudflare Tunnel + set `APPLE_BRIDGE_SHIM_URL`

**Files:**
- Modify: `~/.cloudflared/config.yml` (or new tunnel config)
- Create: `mac-side/shim/com.m3taos.approval-gate-shim.plist`

- [ ] **Step 1: Create or reuse a Cloudflare Tunnel**

```bash
cloudflared tunnel login          # opens a browser, one-time
cloudflared tunnel create approval-gate-shim
```

Expected: prints a tunnel ID and writes credentials to `~/.cloudflared/<tunnel-id>.json`.

- [ ] **Step 2: Add ingress rule**

Append to `~/.cloudflared/config.yml` (create if missing):

```yaml
tunnel: <tunnel-id>
credentials-file: /Users/c0achm3ta/.cloudflared/<tunnel-id>.json

ingress:
  - hostname: approval-shim.eagleeyevisionlabz.com
    service: http://localhost:9999
  - service: http_status:404
```

(Use any apex you control on Cloudflare. If you'd rather use a workers.dev subdomain, skip the public hostname and rely on the tunnel's internal `*.cfargotunnel.com` URL.)

- [ ] **Step 3: Route DNS**

```bash
cloudflared tunnel route dns approval-gate-shim approval-shim.eagleeyevisionlabz.com
```

Expected: success message.

- [ ] **Step 4: Run the tunnel**

```bash
cloudflared tunnel run approval-gate-shim &
```

- [ ] **Step 5: Verify the tunnel**

```bash
curl https://approval-shim.eagleeyevisionlabz.com/health
```

Expected: `{"ok":true}`.

- [ ] **Step 6: Set the Worker secret**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
echo "https://approval-shim.eagleeyevisionlabz.com" | wrangler secret put APPLE_BRIDGE_SHIM_URL
npm run deploy
```

- [ ] **Step 7: Create `com.m3taos.approval-gate-shim.plist` for autostart**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.m3taos.approval-gate-shim</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/c0achm3ta/.local/bin/tsx</string>
    <string>/Users/c0achm3ta/CloudDocs/03_Projects/cf-workflows-poc/mac-side/shim/server.ts</string>
  </array>
  <key>EnvironmentVariables</key>
  <dict>
    <key>APPLE_BRIDGE_SHIM_TOKEN</key>
    <string>__SET_FROM_LAUNCHCTL_SETENV__</string>
  </dict>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/Users/c0achm3ta/.m3ta-os/logs/approval-gate-shim.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/c0achm3ta/.m3ta-os/logs/approval-gate-shim.err.log</string>
</dict>
</plist>
```

- [ ] **Step 8: Install and load the plist**

```bash
mkdir -p ~/.m3ta-os/logs
cp ~/CloudDocs/03_Projects/cf-workflows-poc/mac-side/shim/com.m3taos.approval-gate-shim.plist ~/Library/LaunchAgents/
# Inject env vars (don't put secrets in the plist)
launchctl setenv APPLE_BRIDGE_SHIM_TOKEN "$APPLE_BRIDGE_SHIM_TOKEN"
launchctl setenv TODOIST_API_TOKEN "$TODOIST_API_TOKEN"
launchctl load ~/Library/LaunchAgents/com.m3taos.approval-gate-shim.plist
launchctl list | grep approval-gate-shim
```

Expected: the launchctl list shows the label with PID.

- [ ] **Step 9: Smoke test end-to-end via the deployed Worker**

```bash
curl -i -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"requester":"smoke:e2e-shim","action":"e2e-shim","payload":{"x":1},"tier":"T2","timeout_seconds":120}'
```

Expected: HTTP 202; an iMessage arrives on your phone within ~5 seconds.

- [ ] **Step 10: Commit**

```bash
git add mac-side/shim
git commit -m "feat(shim): add launchd plist and tunnel config notes"
```

---

## Task 13: Mac resolver daemon (Python)

**Files:**
- Create: `mac-side/resolver/resolver.py`
- Create: `mac-side/resolver/requirements.txt`
- Create: `mac-side/resolver/.env.example`
- Create: `mac-side/resolver/README.md`

- [ ] **Step 1: Create `requirements.txt`**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
mkdir -p mac-side/resolver
cat > mac-side/resolver/requirements.txt <<'EOF'
httpx>=0.27
python-dotenv>=1.0
EOF
```

- [ ] **Step 2: Create `.env.example`**

```bash
cat > mac-side/resolver/.env.example <<'EOF'
APPROVAL_GATE_BASE_URL=https://cf-workflows-poc.coachmeta-158.workers.dev
APPROVAL_GATE_TOKEN=set-via-~/.m3ta-os/.env
APPROVAL_RESOLVE_SECRET=set-via-~/.m3ta-os/.env
SUPABASE_URL=https://ojuxzyaydrzgkszqarcr.supabase.co
SUPABASE_ANON_KEY=set-via-~/.m3ta-os/.env
POLL_INTERVAL_SECONDS=2
EOF
```

- [ ] **Step 3: Write `resolver.py`**

```python
"""
Approval Gate resolver daemon.

Polls the macOS Messages chat.db every POLL_INTERVAL_SECONDS for new inbound
messages matching `approve <id6>` / `deny <id6> reason: ...`. Looks up the
full request id (from the 6-char prefix) via the approval-gate /status
introspection endpoint, signs an HMAC, POSTs to /resolve/:id.

Run via launchd (com.m3taos.approval-gate-resolver.plist).
"""
from __future__ import annotations

import hashlib
import hmac
import os
import re
import sqlite3
import time
from dataclasses import dataclass
from pathlib import Path

import httpx
from dotenv import load_dotenv

load_dotenv(Path.home() / ".m3ta-os" / ".env")
load_dotenv()  # plus any local .env

BASE_URL = os.environ["APPROVAL_GATE_BASE_URL"]
GATE_TOKEN = os.environ["APPROVAL_GATE_TOKEN"]
RESOLVE_SECRET = os.environ["APPROVAL_RESOLVE_SECRET"].encode()
SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_ANON_KEY = os.environ.get("SUPABASE_ANON_KEY", "")
POLL = float(os.environ.get("POLL_INTERVAL_SECONDS", "2"))

CHAT_DB = Path.home() / "Library" / "Messages" / "chat.db"
STATE_FILE = Path.home() / ".m3ta-os" / "approval-gate-resolver.state"
STATE_FILE.parent.mkdir(parents=True, exist_ok=True)

PATTERN = re.compile(
    r"^\s*(approve|deny)\s+([0-9a-f]{6})(?:\s+reason:\s*(.+))?\s*$",
    re.IGNORECASE,
)


@dataclass
class InboundMsg:
    rowid: int
    text: str
    handle: str
    date_utc: int  # apple-epoch nanoseconds


def get_last_seen_rowid() -> int:
    if STATE_FILE.exists():
        return int(STATE_FILE.read_text().strip() or "0")
    return 0


def write_last_seen_rowid(rowid: int) -> None:
    STATE_FILE.write_text(str(rowid))


def fetch_new_messages(last_rowid: int) -> list[InboundMsg]:
    # chat.db is read-only via SQLite. Use uri readonly mode for safety.
    uri = f"file:{CHAT_DB}?mode=ro"
    con = sqlite3.connect(uri, uri=True)
    try:
        cur = con.execute(
            """
            select m.rowid, m.text, h.id, m.date
            from message m
            join handle h on m.handle_id = h.rowid
            where m.is_from_me = 0
              and m.text is not null
              and m.rowid > ?
            order by m.rowid asc
            limit 100
            """,
            (last_rowid,),
        )
        return [InboundMsg(rowid=r[0], text=r[1], handle=r[2], date_utc=r[3]) for r in cur.fetchall()]
    finally:
        con.close()


def resolve_full_id(short_id: str) -> str | None:
    """Look up the full UUID for a 6-char prefix among PENDING approval requests."""
    # Use Supabase REST with anon key; RLS allows service-role only, so this
    # actually fails — we use the gate's own /status endpoint instead. Implement
    # a thin /by-prefix/:short_id endpoint? For v1, simpler: query Supabase via
    # service-role. Keep the service role local to this daemon.
    service_key = os.environ.get("SUPABASE_SERVICE_KEY")
    if not service_key:
        # fallback: query the gate's pending-requests endpoint (not yet built)
        return None
    resp = httpx.get(
        f"{SUPABASE_URL}/rest/v1/approval_requests",
        params={"select": "id", "status": "eq.pending", "id": f"like.{short_id}*"},
        headers={
            "Authorization": f"Bearer {service_key}",
            "apikey": service_key,
        },
        timeout=5,
    )
    rows = resp.json()
    if not rows:
        return None
    if len(rows) > 1:
        # collision — caller will need to be more specific; reply over iMessage
        return None
    return rows[0]["id"]


def post_resolve(full_id: str, decision: str, reason: str | None) -> int:
    payload = f"{full_id}:{decision}:{reason or ''}"
    sig = hmac.new(RESOLVE_SECRET, payload.encode(), hashlib.sha256).hexdigest()
    body = {"decision": decision, "hmac": sig}
    if reason:
        body["reason"] = reason
    resp = httpx.post(
        f"{BASE_URL}/resolve/{full_id}",
        json=body,
        timeout=10,
    )
    return resp.status_code


def loop_once() -> None:
    last = get_last_seen_rowid()
    msgs = fetch_new_messages(last)
    for m in msgs:
        match = PATTERN.match(m.text or "")
        if not match:
            write_last_seen_rowid(m.rowid)
            continue
        decision = match.group(1).lower()
        short = match.group(2).lower()
        reason = match.group(3)
        full = resolve_full_id(short)
        if not full:
            print(f"[resolver] no pending match for short id {short!r}, skipping")
            write_last_seen_rowid(m.rowid)
            continue
        status = post_resolve(full, decision, reason)
        print(f"[resolver] {decision} {full} -> {status}")
        write_last_seen_rowid(m.rowid)


def main() -> None:
    print(f"[resolver] starting; chat.db={CHAT_DB}; poll={POLL}s")
    while True:
        try:
            loop_once()
        except Exception as e:  # noqa: BLE001 — daemon must not die on a bad row
            print(f"[resolver] error: {e!r}")
        time.sleep(POLL)


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Append `SUPABASE_SERVICE_KEY` to `~/.m3ta-os/.env`**

The resolver needs the service-role key to look up full UUIDs from 6-char prefixes. (Storing it locally + securing chmod 600 is the v1 trade-off; an alternative is a `/by-prefix/:short` Worker endpoint — captured as Open Question 5 in the spec.)

```bash
read -s -p "SUPABASE_SERVICE_KEY (same one set in Worker secrets): " key
echo "SUPABASE_SERVICE_KEY=$key" >> ~/.m3ta-os/.env
chmod 600 ~/.m3ta-os/.env
```

- [ ] **Step 5: Write `mac-side/resolver/README.md`**

```markdown
# Approval Gate Resolver

Polls macOS Messages chat.db for inbound `approve <id6>` / `deny <id6> reason: ...` replies, signs HMAC, POSTs to `/resolve/:id`.

## Setup

1. Grant Terminal (or whatever runs `python`) **Full Disk Access** in System Settings → Privacy & Security → Full Disk Access. Without it, chat.db is unreadable.
2. `python3 -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
3. Copy `.env.example` → `.env`; values come from `~/.m3ta-os/.env`.
4. Run: `python resolver.py`. Should print a startup line and poll silently.

## launchd autostart

`com.m3taos.approval-gate-resolver.plist` ships next door (Task 14). Once installed it tracks state in `~/.m3ta-os/approval-gate-resolver.state` and logs to `~/.m3ta-os/logs/approval-gate-resolver.log`.

## Reset

To re-process old messages (debug only):

```
rm ~/.m3ta-os/approval-gate-resolver.state
```

## Limitations (v1)

- Single decider (Coach Meta). No multi-handle support.
- Short id collision returns "no match" — sender must wait for user to ping back. Plan to surface a "didn't understand" reply in v1.1.
- Polling chat.db at 2s is fine on M5; bump to 5s on lower-power hardware.
```

- [ ] **Step 6: Manual smoke test**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc/mac-side/resolver
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python resolver.py &
RESOLVER_PID=$!
```

In a separate terminal, fire a request and reply via iMessage:

```bash
curl -i -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"requester":"smoke:e2e","action":"e2e-test","payload":{"hello":"world"},"tier":"T2","timeout_seconds":300}'
```

Reply to the iMessage prompt with `approve <id6>` (use the first 6 chars of the returned `id`).

Expected: resolver log prints `[resolver] approved <full-uuid> -> 200`. Querying `/status/<id>` returns `status: approved`.

```bash
kill $RESOLVER_PID
```

- [ ] **Step 7: Commit**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
git add mac-side/resolver
git commit -m "feat(resolver): chat.db poller, HMAC sign, POST /resolve/:id"
```

---

## Task 14: Resolver launchd autostart

**Files:**
- Create: `mac-side/resolver/com.m3taos.approval-gate-resolver.plist`

- [ ] **Step 1: Write the plist**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.m3taos.approval-gate-resolver</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/c0achm3ta/CloudDocs/03_Projects/cf-workflows-poc/mac-side/resolver/.venv/bin/python</string>
    <string>/Users/c0achm3ta/CloudDocs/03_Projects/cf-workflows-poc/mac-side/resolver/resolver.py</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardOutPath</key>
  <string>/Users/c0achm3ta/.m3ta-os/logs/approval-gate-resolver.log</string>
  <key>StandardErrorPath</key>
  <string>/Users/c0achm3ta/.m3ta-os/logs/approval-gate-resolver.err.log</string>
</dict>
</plist>
```

- [ ] **Step 2: Install and load**

```bash
cp ~/CloudDocs/03_Projects/cf-workflows-poc/mac-side/resolver/com.m3taos.approval-gate-resolver.plist ~/Library/LaunchAgents/
launchctl load ~/Library/LaunchAgents/com.m3taos.approval-gate-resolver.plist
launchctl list | grep approval-gate-resolver
```

Expected: label appears in launchctl list with a PID.

- [ ] **Step 3: Verify Full Disk Access**

System Settings → Privacy & Security → Full Disk Access → ensure the Python binary at `mac-side/resolver/.venv/bin/python` is present and toggled ON. Without it the daemon will start but every poll will fail with `unable to open database file`.

```bash
tail -n 20 ~/.m3ta-os/logs/approval-gate-resolver.log ~/.m3ta-os/logs/approval-gate-resolver.err.log
```

Expected: log shows the startup line, no `unable to open database file`.

- [ ] **Step 4: Commit**

```bash
git add mac-side/resolver/com.m3taos.approval-gate-resolver.plist
git commit -m "feat(resolver): launchd autostart plist"
```

---

## Task 15: Operational Hermes skill

**Files:**
- Replace: `~/.hermes/skills/m3ta-os/approval-gate.md`

- [ ] **Step 1: Replace the stub with an operational skill**

Write to `~/.hermes/skills/m3ta-os/approval-gate.md`:

```markdown
---
name: approval-gate
description: Submit, check, or list HITL approval requests. Use when the user says "request approval for X", "ask Coach Meta to approve Y", "check approval <id>", or "what's pending approval".
---

# Approval Gate (operational)

Reusable HITL approval substrate. Backed by the `cf-workflows-poc` Cloudflare Worker + Workflow.

## Endpoints

| Verb | Endpoint | Auth |
|---|---|---|
| Submit | `POST https://cf-workflows-poc.coachmeta-158.workers.dev/request` | `Authorization: Bearer $APPROVAL_GATE_TOKEN` |
| Check | `GET https://cf-workflows-poc.coachmeta-158.workers.dev/status/:id` | `Authorization: Bearer $APPROVAL_GATE_TOKEN` |

`APPROVAL_GATE_TOKEN` lives in `~/.m3ta-os/.env`.

## When invoked from Telegram/Slack/Discord/etc.

### "request approval for ..."

1. Extract: action verb, payload (key:value lines or JSON), tier (default T2), timeout (default 1h)
2. POST `/request` with `requester=hermes:<channel>`
3. Reply with the short id and the full status URL: "Submitted: status `pending` (id: abc123). Check at <status_url>."

### "check approval <id>"

1. Accept full UUID or 6-char short id (look up via Supabase if short)
2. GET `/status/:id`
3. Reply with status, decided_by, decided_at, reason

### "list pending approvals"

1. Supabase MCP: `select id, requester, action, tier, timeout_at from approval_requests where status='pending' order by created_at desc limit 20`
2. Reply with a numbered list

## Non-goals

- This skill is the *requester* side. Coach Meta resolves via iMessage replies (handled by the resolver daemon, not Hermes).
- Hermes does NOT auto-approve anything.

## Spec

See `~/CloudDocs/03_Projects/cf-workflows-poc/docs/superpowers/specs/2026-04-26-approval-gate-design.md`.
```

- [ ] **Step 2: Verify Hermes picks up the skill**

```bash
ls -la ~/.hermes/skills/m3ta-os/approval-gate.md
```

Hermes auto-loads skills from this dir on next agent spawn. No commit needed in this repo (it's outside the repo).

- [ ] **Step 3: Smoke-test from Hermes**

In a Telegram/Slack chat with the relevant Hermes-bridged bot, send:

```
request approval for sending Riverland follow-up email to mike@riverland.com — tier T3
```

Expected: Hermes responds with a status link and an iMessage prompt arrives within 5 seconds.

---

## Task 16: Caller integration guide

**Files:**
- Create: `docs/superpowers/specs/approval-gate-caller-guide.md`

- [ ] **Step 1: Write the guide**

```markdown
# Approval Gate — Caller integration guide

Copy-pasteable patterns for callers (Qu3bii dispatch, NemoClaw, brand pipelines, ad-hoc scripts).

## Endpoint

`POST https://cf-workflows-poc.coachmeta-158.workers.dev/request`

Headers: `Authorization: Bearer $APPROVAL_GATE_TOKEN` (`~/.m3ta-os/.env`)

## Request payload

| Field | Type | Required | Notes |
|---|---|---|---|
| `requester` | string | yes | Short identifier: `hermes:slack`, `qu3bii`, `edge-hydration:riverland` |
| `action` | string | yes | Verb summary: `send-outreach-email`, `create-stripe-link` |
| `payload` | object | yes | Free-form, shown to approver verbatim — keep keys short |
| `tier` | enum | yes | `T2` `T3` `T4` |
| `timeout_seconds` | int | no | Default 3600 (1h) |
| `channel` | enum | no | `imessage` `mail` `todoist`. Omit for full cascade. |
| `webhook` | url | no | POSTed final state when decided |

## Response

```json
{
  "id": "uuid",
  "status": "pending",
  "status_url": "https://.../status/<id>",
  "timeout_at": "ISO-8601"
}
```

## Patterns

### Synchronous wait (Python)

```python
import os, httpx, time
TOKEN = os.environ["APPROVAL_GATE_TOKEN"]
BASE = "https://cf-workflows-poc.coachmeta-158.workers.dev"

def request_approval(action: str, payload: dict, tier: str, timeout_s: int = 3600) -> dict:
    h = {"Authorization": f"Bearer {TOKEN}"}
    r = httpx.post(f"{BASE}/request", json={
        "requester": "my-script",
        "action": action,
        "payload": payload,
        "tier": tier,
        "timeout_seconds": timeout_s,
    }, headers=h, timeout=10)
    r.raise_for_status()
    return r.json()

def wait_for_decision(id: str, poll_s: int = 5) -> dict:
    h = {"Authorization": f"Bearer {TOKEN}"}
    while True:
        r = httpx.get(f"{BASE}/status/{id}", headers=h, timeout=10)
        body = r.json()
        if body["status"] != "pending":
            return body
        time.sleep(poll_s)
```

### Webhook (Hermes / Qu3bii pattern)

Provide a `webhook` URL when calling `/request`. The Workflow POSTs final state:

```json
{
  "id": "uuid",
  "status": "approved" | "denied" | "timeout",
  "channel": "imessage" | "mail" | "todoist" | null,
  "reason": "free text or null"
}
```

Webhook receivers should:
- Verify the call comes from the approval-gate (currently no signature; future: HMAC)
- Treat `timeout` as deny by default
- Be idempotent (the gate retries up to 3 times on non-2xx)

### Qu3bii dispatch hook

In Qu3bii's T2/T3 router, before executing a tiered action:

```python
# pseudocode
if action.tier in ("T2", "T3", "T4"):
    decision = call_approval_gate(action)
    if decision["status"] != "approved":
        log(f"action {action.id} not approved: {decision['status']}")
        return
execute(action)
```

### NemoClaw external send

Wrap external sends (Apollo enrichment, outreach) in request → wait → execute. Keep loops idempotent so a denial leaves no partial state.

## Common payload shapes

### Outreach email send

```json
{
  "to": "mike@riverland.com",
  "subject": "Edge Hydration sponsorship follow-up",
  "preview": "Hi Mike — circling back on..."
}
```

### Stripe link generation

```json
{
  "amount_cents": 9700,
  "product": "FNPC starter pack",
  "customer": "Maria Lopez",
  "purpose": "client onboarding"
}
```

### Roku partnership decision

```json
{
  "partner": "RokuChannel",
  "deal_value": "$45k ARR",
  "term_months": 12,
  "blockers": ["legal-review-pending"]
}
```

## Failure modes (caller-side)

| If | Then |
|---|---|
| `/request` returns 401 | Token is wrong — refresh from `~/.m3ta-os/.env` |
| `/request` returns 400 | Validation failed; check `issues` array in response body |
| `/request` returns 500 | Supabase down; retry with backoff |
| `/status` says `timeout` | Gate decided no answer = no action; caller decides fail-open vs fail-closed |
| `/status` hangs at `pending` past timeout | Workflow may be stuck; check Cloudflare dashboard logs |
```

- [ ] **Step 2: Commit**

```bash
git add docs/superpowers/specs/approval-gate-caller-guide.md
git commit -m "docs: caller integration guide for approval gate"
```

---

## Task 17: End-to-end smoke test

**Files:**
- (none — test only; outputs to console)

- [ ] **Step 1: Verify all daemons are running**

```bash
launchctl list | grep approval-gate
```

Expected: both `com.m3taos.approval-gate-shim` and `com.m3taos.approval-gate-resolver` listed with PIDs.

```bash
ps aux | grep cloudflared | grep -v grep
```

Expected: a cloudflared process for the `approval-gate-shim` tunnel.

- [ ] **Step 2: Submit a real request**

```bash
source ~/.m3ta-os/.env
RESPONSE=$(curl -s -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "smoke:final",
    "action": "final-e2e-test",
    "payload": {"summary":"end-to-end test of approval gate v1"},
    "tier": "T2",
    "timeout_seconds": 600
  }')
echo "$RESPONSE" | jq
ID=$(echo "$RESPONSE" | jq -r .id)
echo "ID: $ID"
```

Expected: `id` echoed, iMessage prompt arrives within 5s.

- [ ] **Step 3: Approve via iMessage**

Reply to the iMessage prompt: `approve <first-6-chars-of-id>`.

- [ ] **Step 4: Verify decision propagated**

```bash
sleep 5
curl -s "https://cf-workflows-poc.coachmeta-158.workers.dev/status/$ID" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" | jq
```

Expected: `status: "approved"`, `decided_by: "coach-meta"`, `channel: "imessage"`, `decided_at` populated.

- [ ] **Step 5: Verify Supabase audit row**

Via Supabase MCP `mcp__plugin_supabase_supabase__execute_sql`:

```sql
select id, requester, action, status, channel, decided_at, reason
from approval_requests
where id = '<ID>';
```

Expected: one row matching the decision.

- [ ] **Step 6: Test timeout path**

```bash
RESPONSE=$(curl -s -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "smoke:timeout",
    "action": "timeout-test",
    "payload": {},
    "tier": "T2",
    "timeout_seconds": 60
  }')
ID=$(echo "$RESPONSE" | jq -r .id)
echo "ID: $ID — wait 70s then check"
sleep 70
curl -s "https://cf-workflows-poc.coachmeta-158.workers.dev/status/$ID" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" | jq
```

Expected: `status: "timeout"`. Do NOT reply to the iMessage during the wait.

- [ ] **Step 7: Test deny path**

```bash
RESPONSE=$(curl -s -X POST "https://cf-workflows-poc.coachmeta-158.workers.dev/request" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "requester": "smoke:deny",
    "action": "deny-test",
    "payload": {},
    "tier": "T2",
    "timeout_seconds": 600
  }')
ID=$(echo "$RESPONSE" | jq -r .id)
echo "Reply: deny ${ID:0:6} reason: testing the deny path"
```

Reply via iMessage with `deny <id6> reason: testing the deny path`.

```bash
sleep 5
curl -s "https://cf-workflows-poc.coachmeta-158.workers.dev/status/$ID" \
  -H "Authorization: Bearer $APPROVAL_GATE_TOKEN" | jq
```

Expected: `status: "denied"`, `reason: "testing the deny path"`.

- [ ] **Step 8: Final commit**

If anything was modified during smoke testing (env files, tweaks), commit it. Otherwise:

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
git status
```

If clean — done. If not — review and commit:

```bash
git add -A
git commit -m "chore: smoke test fixes"
git push origin main
```

---

## Task 18: Memory + close-out

**Files:**
- Create: `~/.claude/projects/-Users-c0achm3ta/memory/project_approval_gate.md`
- Modify: `~/.claude/projects/-Users-c0achm3ta/memory/MEMORY.md`

- [ ] **Step 1: Add a memory file for the project**

Write `~/.claude/projects/-Users-c0achm3ta/memory/project_approval_gate.md`:

```markdown
---
name: Approval Gate
description: Cloudflare Workflow-based HITL approval gate. Reusable substrate for Hermes / Qu3bii / NemoClaw / brand pipelines. Replaces ad-hoc approval logic across M3ta-OS.
type: project
---

The `cf-workflows-poc` repo (project ID `cf-workflows-poc`, Worker URL https://cf-workflows-poc.coachmeta-158.workers.dev) hosts the Approval Gate. Spec at `docs/superpowers/specs/2026-04-26-approval-gate-design.md`. Caller guide at `docs/superpowers/specs/approval-gate-caller-guide.md`.

**Why:** M3ta-OS had T0–T4 governance tiers but no durable approval primitive. Every pipeline reinvented prompt-and-wait with no audit trail.

**How to apply:**
- For new T2/T3 actions: `POST /request` instead of bespoke approval logic
- For existing pipelines (Distribution Engine, Riverland): keep their flows in v1; per-pipeline cutover is a follow-up
- For natural-language requests via Telegram/Slack: use the Hermes `approval-gate` skill at `~/.hermes/skills/m3ta-os/approval-gate.md`
- Audit trail: Supabase `approval_requests` table in project `m3ta-platform`

**Daemons (autostart):** `com.m3taos.approval-gate-shim` (Node, port 9999) + `com.m3taos.approval-gate-resolver` (Python, polls chat.db). Both behind Cloudflare Tunnel `approval-gate-shim`.
```

- [ ] **Step 2: Add to memory index**

Append one line to `~/.claude/projects/-Users-c0achm3ta/memory/MEMORY.md`:

```markdown
- [Approval Gate](project_approval_gate.md) — Cloudflare Workflow HITL approval-gate-as-a-service for Hermes/Qu3bii/NemoClaw/brands; spec + caller guide in cf-workflows-poc repo
```

- [ ] **Step 3: Push the final commit**

```bash
cd ~/CloudDocs/03_Projects/cf-workflows-poc
git push origin main
```

Expected: clean push.

---

## Self-review notes for the implementing engineer

If you find any of the following while implementing, fix inline and continue — do NOT escalate:
- A test in Task 6/7/8 hangs on Supabase calls: mock `getClient` with vitest's `vi.mock("../src/supabase")`
- Cloudflare Workflow class rename refuses to apply: delete the deployed Workflow first via `wrangler workflows delete workflow-cf-workflows-poc` and redeploy
- `tsx` not found in launchd plist: `npm i -g tsx` or use the project-local path `node_modules/.bin/tsx`
- chat.db permission error: Full Disk Access for the Python binary is non-negotiable on macOS 26+
- Cloudflare Tunnel hostname collision: pick a different subdomain in `config.yml`

Escalate if any of:
- Cloudflare Workflows API rejects `instance.sendEvent` for a class-renamed Workflow (this is a Cloudflare bug, not your bug)
- Supabase RLS prevents the resolver daemon from looking up by short id even with service-role key (wrong key was set)
- Both iMessage and Mail osascript invocations fail in the shim — likely a TCC permission, not code

## Done criteria

- [ ] All 18 tasks complete
- [ ] All vitest tests passing (`npm test`)
- [ ] Two launchd daemons running and surviving a logout/login cycle
- [ ] Cloudflare Tunnel up
- [ ] End-to-end approve, deny, and timeout paths verified
- [ ] Supabase audit rows present for all three smoke-test cases
- [ ] Hermes skill replaces the stub and works from Telegram/Slack
- [ ] Caller guide committed
- [ ] Memory updated
- [ ] All commits pushed to `main`
