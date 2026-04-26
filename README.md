# cf-workflows-poc

Cloudflare Workers + Workflows tire-kick. Generated from the official `hello-world-workflows` template, kept around as a reference for durable-execution patterns.

## What this demonstrates

Cloudflare Workflows give you durable, restart-safe execution for multi-step jobs. The single `MyWorkflow` class in [`src/index.ts`](src/index.ts) exercises every primitive worth knowing:

| Primitive | Purpose |
|---|---|
| `step.do(name, fn)` | Run a step durably — output is checkpointed |
| `step.do(name, { retries, timeout }, fn)` | Add retry/backoff and timeout policy |
| `step.waitForEvent(name, { type, timeout })` | Pause until an HTTP event arrives (human approval, webhook) |
| `step.sleep(name, duration)` | Durable sleep that survives Worker restarts |

The `fetch` handler exposes:

- `GET /` — spawn a new Workflow instance, return its id and status
- `GET /?instanceId=<id>` — poll status of an existing instance

## Stack

- Cloudflare Workers + Workflows
- TypeScript
- Wrangler 4
- Compatibility date `2026-04-25`, `nodejs_compat` flag

## Run locally

```bash
npm install
npm run dev
# Worker is at http://localhost:8787
curl http://localhost:8787                       # spawn an instance
curl "http://localhost:8787/?instanceId=<id>"    # poll status
```

## Deploy

```bash
wrangler login    # one-time
npm run deploy    # publishes via `wrangler deploy`
```

## Trigger the wait-for-approval step

The Workflow pauses on `step.waitForEvent("request-approval", ...)`. Resume it by POSTing to:

```
/accounts/{account_id}/workflows/{workflow_name}/instances/{instance_id}/events/request-approval
```

Cloudflare API docs: <https://developers.cloudflare.com/workflows/build/events-and-parameters/>

## Project conventions

- `CLAUDE.md` — context for Claude Code sessions
- `AGENTS.md` — context for the C3-generated agent toolchain (do not overwrite)
- `.dev.vars` and `.env*` are gitignored; use `wrangler secret put` for production secrets

## License

UNLICENSED — internal scaffold, not for redistribution.
