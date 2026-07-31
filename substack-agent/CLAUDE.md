# Draftsmith — Claude Code Guidelines

## What this repo is
Specification for **Draftsmith**, a Manifest V3 Chrome extension: an AI agent that
researches, writes in the author's voice, SEO-optimises, and inserts posts into the
Substack editor. Two LLM modes (BYOK + Managed credits), Stripe billing, Chrome Web
Store distribution.

Right now this directory contains **only documentation**. The code repo is created in
Session 1 at `~/Desktop/First_project/draftsmith/`.

## Start here
- [`PLAN.md`](PLAN.md) — master plan. Read section 0 first; it contains three
  corrections to the original brief that change the build.
- [`docs/11-build-sessions.md`](docs/11-build-sessions.md) — 18 copy-paste session
  prompts. This is the entry point for actually building.

## Reading discipline
Each session prompt names exactly which docs to read. **Do not read the whole `docs/`
folder** — it is ~30k tokens and most of it is irrelevant to any given session.

## Non-negotiable rules while building

1. **No Substack-specific code outside `apps/extension/src/substack/`.** Enforced by an
   ESLint import-boundary rule. This is what makes a Ghost/Beehiiv port a two-week job
   instead of a rewrite.
2. **No `innerHTML`, no `eval`, no `new Function`, no remotely-loaded code.** MV3 bans
   it and the store enforces it. LLM output is rendered as text, never executed.
3. **API keys never touch `chrome.storage.sync`, Sentry, logs, or `RunState`.**
   Encrypted at rest, read at call time only.
4. **Every step of an agent run is checkpointed** to `chrome.storage.local`. The
   service worker will die mid-run; the run must survive it.
5. **Zod-validate at every boundary** — including messages from the injected script,
   which shares a world with page JS.
6. **Credits round up, never down.** Metering reads the provider's own usage block,
   never an estimate. Every ledger write is idempotent.
7. **Human approves before anything is published.** Non-negotiable, product and legal.
8. **Never invent facts in generated prose.** Unverified claims get a `[VERIFY]` marker
   surfaced to the author.

## Commands (once the repo exists)
```bash
pnpm install
pnpm dev                    # extension HMR + api local + web
pnpm --filter extension build
pnpm test                   # vitest
pnpm test:e2e               # playwright, needs a build first
pnpm eval --suite draft     # prompt evals, costs money
pnpm --filter extension package
```

## Commit convention
`feat(sN): <session title>` at each session boundary. One session, one commit.
