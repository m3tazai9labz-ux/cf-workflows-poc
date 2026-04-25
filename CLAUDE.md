# cf-workflows-poc

Cloudflare Workers + Workflows tire-kick. Scaffolded 2026-04-25 via `npm create cloudflare@latest -- --type=hello-world-workflows --lang=ts`.

## Stack

- Cloudflare Workers + Workflows (durable execution)
- TypeScript, compatibility date `2026-04-25`, `nodejs_compat` flag
- Wrangler (bundled via `npm install`)

## Run / deploy

```bash
npm run dev      # local dev (wrangler dev)
npm run deploy   # production deploy (wrangler deploy) — requires `wrangler login` first
```

## Workflow binding

`wrangler.jsonc` declares one workflow:
- Binding: `MY_WORKFLOW`
- Class: `MyWorkflow` (in `src/index.ts`)

To trigger from the Worker fetch handler, the template uses `env.MY_WORKFLOW.create({...})`.

## Status

- No git init (intentional — `--no-git`).
- Not deployed.
- Not wired into any M3ta-OS hub (Hermes, Qu3bii dispatch, daemons) yet — this is exploratory.

## Coexisting agent files

- `AGENTS.md` ships with the C3 template. Don't overwrite it.
