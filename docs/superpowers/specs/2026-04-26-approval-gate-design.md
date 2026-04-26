# Approval Gate — Design Spec

**Date:** 2026-04-26
**Project:** cf-workflows-poc → repurposed as `approval-gate`
**Status:** Approved (design phase). Implementation plan to follow via writing-plans skill.
**Author:** Coach Meta + Claude

## Summary

Replace the placeholder `MyWorkflow` in `cf-workflows-poc` with `ApprovalGate`: a Cloudflare Workflow that exposes "approval-gate-as-a-service" for any M3ta-OS caller (Hermes, Qu3bii dispatch, NemoClaw, brand pipelines). The gate accepts a request to perform a T2/T3 action, prompts Coach Meta on Apple-native channels (iMessage primary, Apple Mail fallback, Todoist last-resort), waits durably for a decision, persists the audit trail in Supabase, and notifies the caller.

`step.waitForEvent` (Cloudflare Workflows' native primitive) is the foundation. The Workflow's durability — surviving Worker restarts, multi-day timeouts, retries — is exactly the property M3ta-OS lacks today for HITL governance.

## Problem statement

M3ta-OS defines T0–T4 governance tiers but has no durable, reusable approval primitive. Today every pipeline that needs a human decision (Distribution Engine HITL, Riverland outreach send, Stripe link gen, T3 external sends) reinvents the prompt-and-wait logic locally with ad-hoc retries, no audit trail, and no shared semantics. This spec consolidates that into one substrate so:

- Adding HITL to a new pipeline is a `POST /request` away
- All approval decisions land in one Supabase audit table for compliance and analytics
- Channel logic (iMessage / Mail / Todoist fallbacks) is owned in one place
- The `T3 external send strict-confirmation rule` becomes machine-enforceable, not memo-enforced

## Non-goals (out of scope for this spec)

- Multi-tenant org / RBAC roles — single-user system (Coach Meta)
- Web dashboard / UI — Apple-native channels only in v1
- Cutting over existing Distribution Engine, Riverland, or other pipelines' bespoke approval logic — they keep working; per-pipeline migration is a follow-up
- Replacing `~/.m3ta-os/whatsapp/send.py`, `apple-bridge`, or other dispatchers — they're consumed, not replaced
- Multi-decider workflows (e.g., "needs 2 of 3 approvers") — single-decider only

## Architecture

```
caller (Hermes / Qu3bii / NemoClaw / brand pipeline)
   │ POST /request {action, payload, tier, timeout, channel?, webhook?}
   │ Authorization: Bearer <APPROVAL_GATE_TOKEN>
   ▼
[Cloudflare Worker — fetch handler]
   │ validate auth + payload
   │ create Workflow instance with payload
   │ return { id, status_url }
   ▼
[ApprovalGate Workflow — src/approval-gate.ts]
   ├ step.do "persist-pending"     → Supabase INSERT approval_requests (status=pending)
   ├ step.do "send-prompt"         → call apple-bridge HTTP shim (iMessage → Mail → Todoist)
   ├ step.waitForEvent "decision"  → durable wait, timeout from request
   ├ step.do "persist-final"       → Supabase UPDATE approval_requests (status=...)
   └ step.do "notify-caller"       → POST webhook (if provided), else no-op
                              ▲
[user replies "approve <id>" or "deny <id> reason: ..." in iMessage]
                              │
                              ▼
[apple-bridge daemon — ~/.m3ta-os/approval-gate-resolver/]
   parses inbound iMessage, signs HMAC, POSTs:
   POST /resolve/:id {decision, reason?, hmac}
                              ▲
                              │
[Cloudflare Worker — fetch handler]
   verify HMAC against APPROVAL_RESOLVE_SECRET
   look up workflow instance by request id
   call instance.sendEvent("decision", { ... })
```

**Why two trust domains:**
`POST /request` and `POST /resolve/:id` use different secrets because the threat models differ. A leaked Bearer token allows submitting fake approval requests (annoying but bounded — Coach Meta sees the prompt). A leaked resolve secret allows arbitrary auto-approval (catastrophic). Splitting them lets the resolver run on a less-privileged surface.

## Components

| Component | Location | Responsibility |
|---|---|---|
| `ApprovalGate` workflow | `src/approval-gate.ts` | Durable orchestration: persist, prompt, wait, finalize, notify |
| Worker fetch handler | `src/index.ts` | Routes `POST /request`, `GET /status/:id`, `POST /resolve/:id` |
| Channel adapter shim | `~/.m3ta-os/approval-gate-shim/send.sh` | HTTP-callable wrapper around iMessage / Mail / Todoist via apple-bridge |
| Resolver daemon | `~/.m3ta-os/approval-gate-resolver/` | Watches iMessage for "approve <id>" / "deny <id>" replies, POSTs to `/resolve/:id` with HMAC |
| Hermes skill | `~/.hermes/skills/m3ta-os/approval-gate.md` | Operational skill: callers can request/check approvals via natural language |
| Caller reference | `docs/superpowers/specs/approval-gate-caller-guide.md` | Written separately during impl: copy-pasteable patterns for Qu3bii / NemoClaw / brand pipelines |

The shim is a **local HTTP service** the Worker calls outbound, because Cloudflare Workers cannot directly invoke macOS osascript. Two options for that callback:
1. **Cloudflare Tunnel** from the Mac to the Worker (preferred — already familiar territory; `~/.cloudflared/`)
2. **Polling**: Worker enqueues prompts to a Supabase row; a Mac daemon polls and dispatches

v1 picks **option 1 (Cloudflare Tunnel)**. Lower latency, simpler reasoning. Polling is the fallback if the tunnel proves flaky.

## Data model — Supabase

Project: `m3ta-platform` (ref `ojuxzyaydrzgkszqarcr`, us-west-2). New table:

```sql
create table approval_requests (
  id            uuid primary key default gen_random_uuid(),
  requester     text not null,                  -- e.g. "hermes:telegram", "qu3bii", "edge-hydration:riverland"
  action        text not null,                  -- short verb summary, e.g. "send-outreach-email"
  payload       jsonb not null,                 -- arbitrary caller payload, shown to approver
  tier          text not null check (tier in ('T2','T3','T4')),
  status        text not null default 'pending'
                check (status in ('pending','approved','denied','timeout','error')),
  channel       text,                           -- which channel resolved it: 'imessage'|'mail'|'todoist'|'api'
  decided_by    text,                           -- 'coach-meta' for now; future-proof for multi-decider
  decided_at    timestamptz,
  reason        text,                           -- optional free-text reason from denier
  webhook       text,                           -- optional caller webhook
  workflow_id   text not null,                  -- Cloudflare Workflow instance id
  timeout_at    timestamptz not null,
  created_at    timestamptz not null default now()
);

create index approval_requests_status_idx on approval_requests (status, timeout_at);
create index approval_requests_requester_idx on approval_requests (requester, created_at desc);
```

RLS: enabled, service-role only (no anon access — this is internal infra).

## API contract

### `POST /request`

```jsonc
// Headers: Authorization: Bearer <APPROVAL_GATE_TOKEN>
// Body:
{
  "requester": "hermes:slack",                // required, short identifier
  "action": "send-outreach-email",            // required, short verb
  "payload": {                                 // required, shown to approver verbatim
    "to": "mike@riverland.com",
    "subject": "Edge Hydration sponsorship follow-up",
    "preview": "Hi Mike — circling back on..."
  },
  "tier": "T3",                                // required: T2 | T3 | T4
  "timeout_seconds": 3600,                     // optional, default 3600 (1h)
  "channel": "imessage",                       // optional REQUESTED channel; if omitted runs default cascade. Note: the Supabase row's `channel` column records the channel that ACTUALLY resolved the request, which may differ if the requested channel failed and a fallback succeeded.
  "webhook": "https://hermes.../approval-cb"   // optional caller webhook (POSTed final state)
}

// Response 202 Accepted:
{
  "id": "uuid",
  "status": "pending",
  "status_url": "https://approval-gate.../status/<id>",
  "timeout_at": "2026-04-26T18:30:00Z"
}
```

### `GET /status/:id`

```jsonc
// Headers: Authorization: Bearer <APPROVAL_GATE_TOKEN>
// Response 200:
{
  "id": "uuid",
  "status": "pending|approved|denied|timeout|error",
  "decided_by": "coach-meta",
  "decided_at": "2026-04-26T17:55:00Z",
  "reason": "looks good",
  "channel": "imessage"
}
```

### `POST /resolve/:id` (called by resolver daemon, not external callers)

```jsonc
// Body:
{
  "decision": "approved" | "denied",
  "reason": "optional free-text",
  "hmac": "<hex-encoded HMAC-SHA256 of `${id}:${decision}:${reason||''}` using APPROVAL_RESOLVE_SECRET>"
}

// Response 200:
{ "id": "uuid", "status": "approved" | "denied" }

// Response 401 if HMAC fails
// Response 409 if already resolved
```

## Channel adapter

Default cascade when `channel` is omitted in the request:

1. **iMessage** (primary) — fast, low friction, native Apple ecosystem.
2. **Apple Mail** — fallback if iMessage delivery times out (15s send window).
3. **Todoist** — last resort. Creates a task with label `@approval-pending`, body containing the action summary + `id`. User must respond out-of-band via API call to `/resolve/:id` (no auto-resolution from Todoist completion in v1).

iMessage prompt template (one message, ≤ 320 chars):
```
🔐 Approval needed (T3)
Action: send-outreach-email
To: mike@riverland.com
Subject: Edge Hydration sponsorship follow-up
Reply: approve abc123  OR  deny abc123 reason: ...
Times out in 60m.
```

The `id` is shortened to first 6 chars in prompts for usability; resolver daemon expands by querying Supabase. Collision risk over a 1h window is negligible (birthday-paradox calc: <0.001% with <100 concurrent pending requests).

## Failure modes & timeouts

| Failure | Behavior |
|---|---|
| iMessage send fails | Cascade to Mail (per channel adapter logic) |
| All channels fail | Workflow logs error, persists `status=error`, calls webhook with error |
| Decision timeout | `status=timeout`. Caller defines fail-open / fail-closed via webhook handler. Default semantics: caller treats `timeout` as deny. |
| Worker crash mid-flight | Workflow state is durable. On restart, picks up at last completed step. |
| Supabase down | Worker KV write-through cache; `step.do "persist-pending"` retries with backoff (max 5 attempts). If all fail, Workflow proceeds with prompt and logs reconciliation event for next run. |
| HMAC fails on resolve | 401, no state change. Logged. |
| Duplicate resolve | 409, no state change. Logged. |

## Caller integration patterns

### Hermes (operational skill)

`~/.hermes/skills/m3ta-os/approval-gate.md` exposes verbs to Telegram/Slack/Discord users:
- "request approval for <action>" — opens a dialog, gathers payload, submits
- "check approval <id>" — polls `/status/:id`
- "list pending approvals" — Supabase query

### Qu3bii dispatch

Qu3bii's T2/T3 router gets a new step: before executing any T2+ action, `POST /request` to the gate. The Workflow waits; Qu3bii caller polls `/status` or registers a webhook. Existing Qu3bii dispatch logic is untouched — this is added as a pre-execution hook for tiered actions.

### NemoClaw

NemoClaw's persistent loops that perform external sends (Apollo enrichment outreach, Stripe link gen) wrap their existing send call in a request → wait → execute pattern. Reference snippet shipped in `approval-gate-caller-guide.md` (written during impl).

### Brand pipelines (Distribution Engine, Riverland, etc.)

Existing pipelines keep their own approval logic in v1. New campaigns adopt the gate. Cutover of existing pipelines = follow-up specs, not this one.

## Auth model summary

| Endpoint | Caller | Secret |
|---|---|---|
| `POST /request` | Any M3ta-OS service | `APPROVAL_GATE_TOKEN` (Bearer) |
| `GET /status/:id` | Any M3ta-OS service | `APPROVAL_GATE_TOKEN` (Bearer) |
| `POST /resolve/:id` | Resolver daemon only | `APPROVAL_RESOLVE_SECRET` (HMAC over body fields) |

Both secrets stored as Cloudflare Worker secrets via `wrangler secret put`. Local copies in `~/.m3ta-os/.env` (chmod 600) for the resolver daemon.

## Configuration — env vars and Worker secrets

| Var | Where | Purpose |
|---|---|---|
| `APPROVAL_GATE_TOKEN` | Worker secret + caller-side `~/.m3ta-os/.env` | Bearer auth for `POST /request`, `GET /status/:id` |
| `APPROVAL_RESOLVE_SECRET` | Worker secret + resolver daemon `~/.m3ta-os/.env` | HMAC key for `POST /resolve/:id` |
| `SUPABASE_URL` | Worker secret | Supabase project URL (`https://ojuxzyaydrzgkszqarcr.supabase.co`) |
| `SUPABASE_SERVICE_KEY` | Worker secret | Service-role key for `approval_requests` writes |
| `APPLE_BRIDGE_SHIM_URL` | Worker secret | Cloudflare Tunnel URL pointing at the Mac-side shim |
| `APPLE_BRIDGE_SHIM_TOKEN` | Worker secret + Mac shim env | Bearer auth between Worker and shim |

## Open questions (raised here, deferred to plan / impl)

1. **iMessage parser robustness** — natural-language deny reasons could break the regex. Plan to use a forgiving parser: `^(approve|deny)\s+([a-z0-9]{6})(?:\s+reason:\s*(.+))?$` case-insensitive, plus a fallback "didn't understand" reply.
2. **Webhook retry policy for caller notify** — exponential backoff, 3 attempts, then dead-letter to a Supabase column? Defer to impl; sane default = 3 retries with 30s/60s/120s backoff, then `webhook_status='dead_letter'`.
3. **Cloudflare Tunnel vs polling for the channel adapter callback** — v1 picks tunnel. If reliability is an issue in the first 30 days of use, fall back to polling.
4. **Audit-trail retention** — Supabase row never deleted. Index on `created_at` keeps queries fast even at 100k+ rows.

## Success criteria

- A caller can `POST /request` and receive a decision via webhook within `timeout_seconds` for at least 95% of requests over a rolling 7-day window.
- Coach Meta can approve/deny from any iPhone or Mac via iMessage with no other tools open.
- Audit trail is queryable via Supabase MCP.
- Hermes "request approval for X" works end-to-end from Telegram/Slack.
- Existing M3ta-OS pipelines are NOT broken (out-of-scope = no cutover).

## Next step

Per the brainstorming skill, the next step is to invoke the **writing-plans** skill to produce a detailed implementation plan from this spec.
