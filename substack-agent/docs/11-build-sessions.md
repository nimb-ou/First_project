# 11 — Build Sessions

Eighteen scoped Claude Code sessions. Each is one context window's worth of work.

**How to run one:**
1. `cd ~/Desktop/First_project/substack-agent` (later: the `draftsmith/` repo created in Session 1)
2. Start a **fresh** Claude Code session.
3. Paste the session prompt **verbatim**.
4. Check the exit criteria before moving on.
5. `git commit -m "feat(sN): <title>"`

**Why fresh sessions:** each prompt names exactly which docs to read. Reading the whole `docs/` folder every session wastes most of the context budget on material irrelevant to the task at hand.

---

## Session 1 — Monorepo scaffold

```
Read PLAN.md sections 3 and 4 only.

Create the Draftsmith monorepo at ~/Desktop/First_project/draftsmith/.

- pnpm workspaces + Turborepo. Node 22, pnpm 9.
- Workspaces: apps/extension, apps/api, apps/web, packages/shared, packages/prompts, e2e
- Root: tsconfig base with strict:true, noUncheckedIndexedAccess:true, paths for
  @draftsmith/shared and @draftsmith/prompts
- ESLint 9 flat config + Prettier. Add two CUSTOM rules that fail the build:
    1. no-restricted-syntax banning `innerHTML` assignment anywhere
    2. no-restricted-properties banning chrome.storage.sync.set / .get
  Also an import boundary: nothing outside apps/extension/src/substack/** may import
  from apps/extension/src/substack/adapters/**.
- Vitest at the root with workspace projects.
- .github/workflows/ci.yml: install, lint, typecheck, test, build. No E2E yet.
- .gitignore, .env.example, README.md (short — point at docs/).
- Copy the docs/ folder and PLAN.md from ~/Desktop/First_project/substack-agent/ into
  the new repo.
- git init, initial commit.

Stub packages so the workspace type-checks: packages/shared/src/index.ts exporting a
version constant, packages/prompts/src/index.ts likewise.

Do not create the extension, api, or web apps yet — later sessions do that.
Verify with: pnpm install && pnpm lint && pnpm typecheck && pnpm test && pnpm build
```

**Exit:** all five commands pass on a clean clone. Both custom lint rules demonstrably fail on a deliberate violation.

---

## Session 2 — Extension shell

```
Read docs/01-architecture.md in full. Skim PLAN.md section 1.

Build apps/extension: a Manifest V3 extension that opens a side panel and has a
working typed message bus. No agent logic yet.

- Vite + @crxjs/vite-plugin + React 18 + TypeScript + Tailwind + shadcn/ui.
- manifest.config.ts exactly as specified in docs/01 section 1.
- Service worker at src/background/index.ts:
    · MessageRouter with Zod validation on every inbound message
    · Long-lived Port handling for the "sidepanel" channel
    · A RunStore backed by chrome.storage.local implementing the RunState shape from
      docs/01 section 4 (get/put/patch/list/prune). Prune runs older than 30 days and
      cap at 50 on startup.
- Side panel at src/sidepanel/: React app, Zustand store with a chrome.storage.local
  persistence adapter, three routes (Compose / Runs / Settings), dark mode.
- Content script at src/content/index.ts: on load, post a "content.ready" message with
  the current URL. Nothing else yet.
- Toolbar icon click opens the side panel (chrome.sidePanel.setPanelBehavior).
- Placeholder icons (solid colour with a "D") at 16/32/48/128.

Prove the bus works: a "Ping" button in the panel sends run.start with a stub payload,
the worker replies with three fake run.step messages over the Port, and the panel
renders them in a streaming log component.

Write Vitest tests for MessageRouter validation (rejects malformed messages) and
RunStore (round-trip, prune, cap).
```

**Exit:** `pnpm --filter extension build` produces a loadable unpacked extension. Side panel opens, Ping streams three steps. No console errors.

---

## Session 3 — Substack recon (⚠ mostly manual)

```
Read docs/02-substack-integration.md in full.

This session is investigation, not implementation. Its output is ground truth for
Sessions 4 and beyond.

I will do the manual browser work; guide me step by step and record what I report back.

1. Walk me through capturing, from a test Substack publication with DevTools open:
   - GET current user, GET publications, GET archive
   - Opening the editor, typing, and the resulting save request
   - The publish modal's prepublish and publish requests
   For each: method, full path, query params, request headers (EXCLUDING cookies),
   request body, response body.

2. Have me copy one complete real draft_body ProseMirror JSON from a post containing:
   H2, H3, bold, italic, inline code, a link, a bulleted list, a numbered list, a
   blockquote, a code block, and an image. Save it verbatim as
   e2e/fixtures/substack/golden-post.pm.json, and write the equivalent markdown as
   golden-post.md.

3. Have me report the CSRF mechanism: header name, where the token comes from.

4. Have me run these in the editor page console and report the output:
     document.querySelector('.ProseMirror[contenteditable="true"]')
     document.querySelector('.ProseMirror[contenteditable="true"]').pmViewDesc
   plus selectors for the title field, subtitle field, save button, publish button —
   and two ALTERNATIVE selectors for each in case the primary changes.

Then write:
- e2e/fixtures/substack/*.json — one file per captured request/response pair
- docs/_recon-2026-07.md — everything observed, dated, with a warning that it is
  unversioned and will change
- apps/extension/src/substack/types.ts — the SubstackAdapter interface and Zod schemas
  for every response shape you captured
- apps/extension/src/substack/probe.ts — the capability probe from docs/02 section 2.
  It must be strictly read-only. Assert this in a test.

If any endpoint differs from what docs/02 section 3 predicts, UPDATE THAT DOC and say
clearly what changed.
```

**Exit:** fixtures captured, `types.ts` compiles, probe is read-only and tested, `docs/_recon-2026-07.md` written, doc 02 corrected against reality.

---

## Session 4 — Editor bridge

```
Read docs/02-substack-integration.md sections 3-5 and docs/_recon-2026-07.md.

Build the Substack write path.

1. apps/extension/src/substack/md-to-pm.ts
   Markdown → Substack ProseMirror JSON, and the inverse pmToMd.
   Use unified + remark-parse. Allowlist of nodes/marks per docs/02 section 3.
   Unsupported constructs (tables, HTML) degrade to plain paragraphs — never throw.
   Escape any raw HTML as text; it must be impossible to inject nodes via markdown.

2. apps/extension/src/injected/prosemirror-bridge.js
   MAIN-world script exactly as in docs/02 section 4, including the nonce check, the
   pmViewDesc lookup with React-fibre fallback, and setReactValue.
   Operations: probe, readDoc, replaceDoc, insertAtCursor, setTitle.

3. apps/extension/src/content/bridge-client.ts
   Injects the bridge with chrome.scripting.executeScript({world:'MAIN'}), generates
   the nonce, and exposes a promise-based RPC with a 10s timeout per call.

4. apps/extension/src/substack/adapters/v2026-07.ts
   Implements SubstackAdapter using the captured endpoints, with:
   - a token-bucket rate limiter (1 req / 1.5s)
   - exponential backoff with jitter on 429/5xx (2s,4s,8s,16s)
   - the X-Requested-With: Draftsmith header
   - Zod validation of every response; a parse failure emits a drift event

5. apps/extension/src/substack/adapters/dom-fallback.ts — bridge-only, no internal API.
6. apps/extension/src/substack/registry.ts — picks an adapter from probe results, in
   order: v2026-07 → dom-fallback → clipboard mode.

Tests:
- Property-based round-trip for md-to-pm with fast-check, 500 runs (docs/08 section 3)
- Golden fixture equality against golden-post.pm.json
- The 7 table-driven edge cases in docs/08 section 3
- Registry falls back correctly for each degraded Capabilities shape
```

**Exit:** round-trip tests pass; golden fixture matches exactly; manually verified that a generated draft inserts into a real test Substack editor with formatting intact and Cmd-Z undoes it.

---

## Session 5 — Shared contracts

```
Read docs/01-architecture.md section 3, docs/03-agent-and-prompts.md section 3,
docs/05-backend-api-and-data-model.md section 3, docs/06-llm-modes-and-credits.md
sections 4 and 6.

Fill packages/shared with every Zod schema and type used across the extension, API,
and web app. This package is the contract; nothing may duplicate a shape defined here.

src/
  messages.ts      # the full discriminated union from docs/01 section 3
  agent-schemas.ts # RunRequest, RunStep, RunState, Outline, Section, Critique,
                   # SeoOptimisation, VoiceProfile (docs/03 section 4), Angle
  seo.ts           # KeywordCandidate, Cluster, SerpSummary, SeoReport, RUBRIC types
  credits.ts       # PROVIDER_RATES, creditsFor, CREDIT_PRICE_USD, estimate helpers
                   # (docs/06 section 4) — exact implementation given there
  errors.ts        # AppErrorCode union + AppError + helpers (docs/01 section 6)
  api.ts           # request/response schema per endpoint in docs/05 section 3
  models.ts        # ModelId union + the step→model routing table from docs/03 section 1
  index.ts

Rules:
- Zod is the source of truth; export inferred types, never hand-written interfaces.
- Every schema gets .describe() on fields the LLM will see — these become tool-schema
  descriptions and materially affect output quality.
- Add zod-to-json-schema and export toolSchema(name, zodSchema) for forced tool calls.

Tests: creditsFor matches the worked example in docs/06 section 4 to the credit; every
schema parses a valid fixture and rejects a malformed one; the ledger invariant
property test from docs/08 section 3.
```

**Exit:** `pnpm --filter shared test` passes. Credit maths matches the documented table exactly.

---

## Session 6 — LLM layer (BYOK)

```
Read docs/06-llm-modes-and-credits.md sections 3, 5, 6.

Build apps/extension/src/llm/.

1. types.ts — the LLMClient interface from docs/06 section 6.
2. keystore.ts — exactly as specified in docs/06 section 3: non-extractable AES-GCM
   KEK in IndexedDB, AES-GCM encrypted keys in chrome.storage.local, never sync.
3. providers/anthropic.ts — SSE streaming. MUST send
   'anthropic-dangerous-direct-browser-access: true'. Parse content_block_delta,
   message_delta (for usage), tool_use blocks. Support cache_control on the system
   block for prompt caching — this is required, not optional (docs/06 section 5).
4. providers/openai.ts, providers/google.ts, providers/openrouter.ts — same interface.
5. byok-client.ts — implements LLMClient over the providers, reports usd cost.
6. permissions.ts — chrome.permissions.request() for the provider's optional host
   permission, only at the moment a key is saved.
7. validate.ts — 1-token test request; distinguishes 401 (bad key) from network errors.

Also add to the side panel Settings route: provider picker, key input (type=password,
never rendered back), Validate button, connection status, and the honest
"can I use my ChatGPT Plus subscription?" explainer copy from docs/06 section 2.

Tests:
- keystore round-trip; a spy asserts chrome.storage.sync is never touched
- a serialised RunState containing a key-shaped string fails a leak assertion
- SSE parsing against recorded fixtures for each provider, including a mid-stream
  disconnect, a 429 with retry-after, and malformed tool-call JSON
- AbortSignal cancels an in-flight stream within 200ms
```

**Exit:** a real Anthropic key can be saved, validated, and used to stream a completion into the panel. Storage inspection shows no plaintext key. Prompt caching is active (verify `cache_read_input_tokens` > 0 on a second identical call).

---

## Session 7 — Agent core

```
Read docs/03-agent-and-prompts.md sections 1, 2, 6. Re-read docs/01 section 4.

Build apps/extension/src/agent/.

- pipeline.ts — the 10 steps from docs/03 section 1 as an ordered array of Step
  objects: { id, label, gate?, model, maxTokens, execute(state, llm, tools, signal) }
- runner.ts — AgentRunner exactly as in docs/03 section 2: checkpoint after EVERY
  step, resume from state.cursor, honour AbortSignal between and within steps,
  withRetry(3, [1s,3s,8s]) on retryable errors only.
- tools/registry.ts — tool definitions with Zod schemas; only steps 2 (research) and
  8 (seo) get tools. Steps 3-7 explicitly get an empty tool list.
- sanitise.ts — the prompt-injection defences from docs/03 section 6: delimiter
  wrapping with trust labels, control-character stripping, per-source and total token
  caps.
- estimate.ts — pre-flight token/credit estimate per step, so the UI can show the cost
  before anything runs.

For this session, steps 1-9 may use PLACEHOLDER implementations that call the LLM with
a trivial prompt and return correctly-shaped artifacts. Real prompts land in Sessions
8-10. The point of this session is the machinery.

Wire it: run.start from the panel executes the full pipeline against the BYOK client
and streams run.step + run.token over the Port.

Tests:
- resume from every cursor value yields identical final artifacts (deterministic fake LLM)
- forced worker termination mid-run then resume completes correctly
- cancel leaves status 'cancelling' and does not advance cursor
- a non-retryable error stops the run and records error_code
- sanitise strips zero-width chars, RTL overrides, and <|...|> sequences
```

**Exit:** a full 10-step placeholder run completes end to end, survives a forced service-worker kill, and resumes. All runner tests pass.

---

## Session 8 — Voice profiling

```
Read docs/03-agent-and-prompts.md sections 4 and 5.1. Read docs/04 section 6.

Build the voice system.

1. apps/extension/src/agent/voice/ingest.ts
   - Fetch up to 50 archive posts via the Substack adapter (rate-limited, cached 30d,
     keyed by a hash of the post ID list)
   - Strip boilerplate: subscribe buttons, share prompts, footers, paywall markers,
     "Thanks for reading" blocks
   - Convert body HTML/PM JSON to clean markdown

2. apps/extension/src/agent/voice/stats.ts
   Compute DETERMINISTICALLY, in code, never by LLM:
   avg sentence words, sentence-length variance bucket, avg paragraph sentences,
   one-sentence-paragraph rate, contraction rate, avg post words, heading usage rate,
   list usage rate, question rate, first/second-person rate, em-dash rate,
   type-token ratio, and the top 30 words by log-odds against a general English
   baseline (ship a small baseline frequency table).

3. packages/prompts/src/voice-profile.v1.ts — the prompt from docs/03 section 5.1,
   with the computed stats injected and the 12 most recent posts as context.

4. apps/extension/src/agent/voice/build.ts — orchestrates ingest → stats → LLM →
   VoiceProfileSchema validation → cache with source_hash.

5. apps/extension/src/agent/voice/render.ts — serialises a VoiceProfile into the
   drafting system prompt block, including exemplar openings verbatim as few-shot.

6. Cannibalisation check (docs/04 section 6): embed archive posts, cosine-compare a
   proposed topic, warn above 0.82. Use the backend /v1/seo/embed endpoint if
   available; otherwise skip gracefully in this session.

UI: a Voice card in Settings — "Learned from 23 posts · rebuilt 2 days ago", a
Rebuild button, and an expandable view of the profile in plain English.

Tests: stats.ts against three hand-computed fixture posts; boilerplate stripping
against real archive HTML; profile caching invalidates on archive change.
```

**Exit:** a real publication's profile builds in under 30s, and reading it aloud describes that author recognisably.

---

## Session 9 — SEO engine

```
Read docs/04-seo-engine.md in full.

Build the SEO pipeline. The backend does not exist yet — put every vendor call behind
apps/extension/src/seo/client.ts with a MOCK implementation reading from fixtures.
Session 12 swaps in the real backend without touching anything else.

1. src/seo/pipeline.ts — the 7 stages from docs/04 section 3.
2. src/seo/expand.ts — LLM seed expansion (12 queries across 4 intents) + Google
   Suggest expansion, capped at 10 calls, 1 rps.
3. src/seo/score.ts — the deterministic opportunity formula from docs/04 section 3.
4. src/seo/cluster.ts — SERP-overlap clustering (≥4 shared URLs in the top 10).
5. src/seo/authority.ts — authorityCeiling() from docs/04 section 3, plus the UI copy
   explaining it.
6. packages/shared/src/seo/rubric.ts — the full 20-rule RUBRIC from docs/04 section 4.
   Every rule deterministic, every rule with a fix string.
7. src/seo/scorecard.ts — applies the rubric to a draft, returns SeoReport with
   per-rule pass/fail, points, and fixes.
8. packages/prompts/src/seo-optimise.v1.ts — the prompt from docs/04 section 5.

UI: a Scorecard route showing the 0-100 ring, band colour, failed-rule cards with
fixes, an "Apply fix" button per card (targeted single-rule LLM call), and a clearly
separated "Not controllable on Substack" section listing canonical tags, page speed,
schema, and URL structure.

Tests: every rubric rule is deterministic across 1000 runs; score always equals the
sum of awarded points and stays in 0-100; clustering against a fixture SERP set;
authorityCeiling monotonicity.
```

**Exit:** pasting an existing Substack post URL produces a stable score with actionable fixes; "Apply fix" measurably changes the score.

---

## Session 10 — Writing tools

```
Read docs/03-agent-and-prompts.md sections 5.2-5.6 and section 7.

Replace the Session 7 placeholders with real implementations for pipeline steps 3-8.

packages/prompts/src/
  angle-select.v1.ts      # docs/03 section 5.2
  outline-build.v1.ts     # docs/03 section 5.3
  draft-section.v1.ts     # docs/03 section 5.4  ← the important one, copy verbatim
  critique-post.v1.ts     # docs/03 section 5.5
  revise.v1.ts            # applies critique fixes; one pass only; must not rewrite
                          # sections the critique did not flag
  repurpose.v1.ts         # Note / X thread / LinkedIn / 3 email subject lines
                          # (docs/04 section 7)

Each prompt is a versioned template function with an id, a version, a typed input, and
its forced-tool schema. Register them in packages/prompts/src/registry.ts so RunState
can record which prompt version produced each artifact.

Drafting specifics:
- Sections are generated SEQUENTIALLY, not in parallel, and each receives the last 200
  characters of the previous section so transitions work.
- Stream tokens to the UI per section.
- The voice profile and outline go in the system block with cache_control set, so all
  six section calls hit the prompt cache.
- Collect [VERIFY: ...] markers into RunState.artifacts.verifyFlags and surface them
  prominently in the review UI.

Build the eval harness from docs/03 section 7:
  packages/prompts/evals/{cases,judges,run.ts}
  - judges/ai-tells.ts is DETERMINISTIC — the regex list plus the four statistical
    checks in docs/03 section 7. No LLM.
  - judges/voice-match.ts and judges/structure.ts as specified.
  - `pnpm eval --suite draft` runs and writes baselines.json.

Seed at least 6 real cases per suite using publications I will name.
```

**Exit:** a full agent run produces a draft a human would plausibly publish. `pnpm eval` runs and records baselines. AI-tell rate is zero on the draft suite.

---

## Session 11 — Side panel UI

```
Read PLAN.md section 1 for the target layout. Read docs/03 section 1 for the pipeline
the UI represents.

Build the real side panel experience. This is the product's face — spend the session
on it.

Routes:
1. COMPOSE — topic input; goal select (grow subscribers / drive traffic / establish
   authority / convert to paid); length select; voice profile picker; mode select
   (Agent / Co-write / Audit); a collapsible Advanced block (target keyword override,
   quality Balanced/Best, custom instructions). Shows the pre-flight credit estimate
   before the Run button is enabled.

2. RUN — the streaming log. Each step is a row with an icon (pending/running/done/
   failed), a label, elapsed ms, and an expandable detail panel showing that step's
   artifact rendered properly: keywords as a table, outline as a tree, sections as
   prose with live token streaming. Sticky footer: Pause, Cancel, and running credit
   spend.

3. REVIEW — the finished draft in a readable serif column. [VERIFY] flags rendered as
   inline amber chips with a "check this" affordance. Inline SEO score. Actions:
   "Push to editor", "Copy markdown", "Regenerate section", "Run SEO fixes".

4. RUNS — history, filterable, resumable. Clicking a run reopens its Review.

5. SETTINGS — account, plan, credit balance with "≈ N posts left", LLM mode toggle,
   BYOK key management, voice profiles, publication link, data export/delete, About
   with the Substack non-affiliation disclaimer.

Co-write mode: approval gates after Angle and after Outline. The gate renders the
artifact as EDITABLE — the user can rewrite the outline and the edited version is what
flows downstream.

Requirements:
- Full keyboard navigation; visible focus rings; ARIA live region for the run log.
- Every error state renders AppError.message plus AppError.hint with an action button.
- Empty states are instructive, never blank.
- Respects prefers-reduced-motion and prefers-color-scheme.
- No layout shift while streaming — reserve space.

Use shadcn/ui. No new dependencies without saying why.
```

**Exit:** the full flow is usable by someone who has not read the docs. All four Playwright specs 1, 3, 10, 11 could plausibly be written against this UI.

---

## Session 12 — Backend v1

```
Read docs/05-backend-api-and-data-model.md in full.

Build apps/api: Hono on Cloudflare Workers.

1. Supabase project + Drizzle schema exactly as in docs/05 section 2, including all
   indexes, the unique index on credit_ledger.stripe_ref, and every RLS policy in
   section 2. Migrations in apps/api/drizzle/.
2. Auth: Supabase JWT verification middleware; /v1/auth/exchange for the
   launchWebAuthFlow code exchange; /v1/auth/refresh; /v1/me; /v1/me/export;
   DELETE /v1/me (soft delete + 30-day purge queue + Stripe cancel).
3. SEO endpoints: /v1/seo/keywords, /v1/seo/serp, /v1/seo/embed. Real DataForSEO and
   Serper integrations, KV-cached per docs/05 section 2 TTLs, with the daily spend
   circuit breaker.
4. /v1/config — remote config and kill switch, Cache-Control max-age=900.
5. /v1/telemetry/drift — no auth, rate-limited by install id, stores nothing but the
   fields listed in docs/05 section 2.
6. Rate limiting middleware (KV sliding window) per the table in docs/05 section 5.
7. Sentry, structured logging, and a /health endpoint.

Then in the extension: replace the mock seo/client.ts with the real one, and add
src/lib/auth.ts using chrome.identity.launchWebAuthFlow.

Separate staging and production Cloudflare environments with separate Supabase
projects. Document every secret in .env.example (names only, never values).

Tests: integration tests with miniflare — auth middleware rejects bad JWTs, RLS
prevents cross-account reads, cache hits do not call the vendor, the circuit breaker
trips at the cap, rate limits return 429 with the right headers.
```

**Exit:** deployed to staging; the extension signs in and fetches real keyword data. RLS verified by attempting a cross-account read.

---

## Session 13 — Managed credits

```
Read docs/06-llm-modes-and-credits.md sections 4 and 5, plus docs/05 section 4.

Build Managed mode.

Backend:
1. /v1/llm/estimate — per-step token estimates → credits.
2. /v1/llm/reserve — idempotent on runId; 402 when the balance is insufficient;
   writes a credit_ledger row.
3. /v1/llm/chat — the streaming proxy from docs/05 section 4, including the
   stream tee, waitUntil metering from the provider's own usage block, the model
   allowlist, server-side max_tokens clamping, and the input-size guard.
4. /v1/llm/settle — reconciles reservation vs actual, idempotent.
5. A cron (every 15 min) that force-settles runs stuck in 'running' for >30 min.
6. The refund policy table from docs/06 section 4, encoded as a pure function that
   maps (failedStep, status) → refund amount.
7. Nightly reconciliation job: recompute each account's balance from the ledger and
   alert on any drift from accounts.credit_balance.

Extension:
8. managed-client.ts implementing LLMClient over the proxy.
9. Mode switching in Settings, with credits shown for Managed and USD for BYOK.
10. The pre-flight estimate gate: no run starts without the user seeing the cost.
11. Hard per-run credit cap (default 80k), configurable, pauses and asks on breach.

Tests (this is where money bugs live — be thorough):
- reserve → settle with actual < reserved releases exactly the difference
- settle called twice charges once
- client disconnect mid-stream still meters the full usage
- concurrent runs on one account cannot overdraw the balance (test with 10 parallel)
- the ledger invariant holds across 10,000 random operation sequences
- the refund function matches every row of the docs/06 policy table
```

**Exit:** a Managed run completes and debits the correct credits; the ledger balances; forced disconnect still charges correctly.

---

## Session 14 — Stripe billing

```
Read docs/07-business-model-and-pricing.md section 4.

Wire up billing.

1. A setup script (apps/api/scripts/stripe-setup.ts) that idempotently creates the
   three products, six subscription prices, and three credit packs from docs/07
   section 3, with plan and monthly_credits in PRICE METADATA. Runs against test mode
   by default.
2. /v1/billing/checkout — Checkout Session, correct success/cancel URLs, Stripe Tax
   enabled, customer created or reused.
3. /v1/billing/portal — Billing Portal session.
4. /v1/billing/usage — runs, credits, and spend for the dashboard.
5. /v1/stripe/webhook — signature verified, handling every event in the docs/07
   section 4 table. EVERY handler is idempotent via credit_ledger.stripe_ref under
   the unique index; a duplicate insert is caught and ignored.
6. Entitlement sync: plan → feature flags (models available, voice profile count,
   publication count, repurposing on/off) returned by /v1/me.
7. Downgrade behaviour: subscription credits reset, PURCHASED top-up credits are
   preserved. Test this explicitly.
8. past_due handling: banner, 3-day grace, then block new runs.

Extension: upgrade CTAs in the credit-exhausted state and after a free user's run
completes; chrome.tabs.create for Checkout; poll /v1/me every 3s for 60s after
opening Checkout, and refresh on side-panel focus.

Tests: `stripe trigger` each event, replayed THREE TIMES, asserting credits are
granted exactly once. Full lifecycle test: subscribe → use → upgrade → downgrade →
cancel, asserting the balance at each step.
```

**Exit:** end-to-end purchase in Stripe test mode updates the plan and grants credits within 10 seconds. Triple-replayed webhooks grant once.

---

## Session 15 — Web app

```
Read docs/07 sections 3 and 6, docs/09 section 2, docs/10 sections 4 and 5.

Build apps/web with Next.js 15 on Vercel.

Pages:
- /            Landing: the hero promise, a 30-second demo video slot, the three
               differentiators (voice matching, achievable SEO, native editor push),
               social proof slot, pricing summary, install CTA.
- /pricing     The full table from docs/07 section 3, with a credits→posts translator
               and an honest BYOK-vs-Managed comparison.
- /scorecard   THE LEAD MAGNET. Paste any Substack post URL → server-side fetch →
               run the shared rubric → render the score and fixes. No install, no
               signup. Email capture to see the full fix list. Rate-limited by IP.
               This is the highest-ROI page in the plan — build it properly.
- /dashboard   Auth-gated: usage, credit ledger, invoices, plan management,
               data export, account deletion.
- /billing/done  Post-Checkout confirmation.
- /privacy     The full policy from docs/10 section 4.
- /terms       The clauses from docs/10 section 5.
- /security    security.txt, contact, disclosure policy.

Requirements:
- The scorecard imports the rubric from @draftsmith/shared. Do not reimplement it.
- SEO the site properly — you are selling an SEO tool. Metadata, OG images, sitemap,
  robots.txt, JSON-LD SoftwareApplication schema.
- Lighthouse ≥95 on performance and accessibility.
- "Not affiliated with Substack" in the footer.
- No cookie banner unless analytics is added; if added, use a cookieless provider.
```

**Exit:** deployed to Vercel; the scorecard works on a real public Substack URL; privacy and terms are live and linkable from the store listing.

---

## Session 16 — Test suite

```
Read docs/08-testing.md in full.

Bring the test suite to shippable.

1. e2e/fake-substack/ — a Hono app replaying the Session 3 fixtures, with all 8 fault
   modes from docs/08 section 4.
2. e2e/fake-provider/ — deterministic SSE responses including mid-stream disconnect,
   malformed tool JSON, 429 with retry-after, overloaded_error, and max_tokens
   truncation.
3. e2e/fixtures.ts — the Playwright persistent-context extension harness from docs/08
   section 5.
4. All 12 E2E specs from the docs/08 section 5 table. Route https://*.substack.com to
   the fake server with context.route(). Never touch live Substack.
5. Fill unit-test gaps to 70% coverage on src/{agent,substack,llm,lib} and 90% on
   packages/shared.
6. Wire the evals CI job: it runs when packages/prompts changes and enforces the four
   gates in docs/08 section 6.
7. .github/workflows/canary.yml — nightly, against the live test publication:
   read archive, create draft, verify, delete draft. Opens a GitHub issue on failure.
8. Update ci.yml with the E2E job under xvfb and artifact upload.

Then run the full manual QA checklist from docs/08 section 7 and fix everything it
surfaces. Report the results honestly — list what failed, not just what passed.
```

**Exit:** green CI including E2E. Manual QA checklist completed with results recorded in the commit message.

---

## Session 17 — Hardening

```
Read docs/01 sections 5-6, docs/02 section 6, docs/10 sections 6-7.

Production-harden everything.

1. Sentry in the extension, worker, and web app, with the beforeSend scrubber from
   docs/01 section 5. Add a test asserting a payload containing a key-shaped string is
   dropped.
2. The remote kill switch end to end: /v1/config polled every 15 min and on startup;
   substackWriteEnabled=false disables all writes and shows the notice;
   minExtensionVersion below the running version shows a forced-update banner.
   TEST IT by flipping the flag in staging.
3. Drift alerting: a cron that alerts when drift events for one
   (adapterVersion, operation) exceed 20/hour.
4. Structured logging with request ids that correlate extension → worker → Sentry.
5. Graceful degradation review: walk EVERY AppErrorCode and confirm the UI shows a
   specific message, a specific hint, and a working action. Fix any that are generic.
6. Storage hygiene: prune runs >30d, cap at 50, and a manual "clear all local data"
   button in Settings.
7. i18n scaffold: extract all UI strings into _locales/en/messages.json. Do not
   translate yet — just make it possible.
8. Accessibility pass: axe-core in the E2E suite, zero violations.
9. Performance: side panel opens in <300ms; the run log stays smooth with 500 log
   lines; extension bundle under 2MB.
10. Runbooks in docs/_runbooks/: incident.md, rotate-secrets.md, substack-drift.md,
    cws-credentials.md, restore-from-backup.md. Each must be followable at 3am by
    someone who did not write it.

Then run a self-review against docs/10 section 8 (the pre-launch legal checklist) and
report which items are done and which need me to act.
```

**Exit:** kill switch verified in staging; zero axe violations; all five runbooks written; legal checklist status reported.

---

## Session 18 — Store submission

```
Read docs/09-chrome-web-store-launch.md in full.

Prepare and ship the store submission.

1. Final manifest review: every permission still used? Remove anything that is not.
   Bump to version 1.0.0.
2. Icon set: replace the placeholders with the real design at 16/32/48/128. Verify the
   16px reads clearly.
3. Screenshot generation: build an HTML template at 2560x1600 per screenshot, render
   the real UI with realistic seeded data, add the caption bands from docs/09 section
   3, and export five 1280x800 PNGs to store-assets/screenshots/. Also produce the
   440x280 small promo tile.
4. store-assets/listing.md — the name, short description, and full detailed
   description from docs/09 section 2, ready to paste.
5. store-assets/privacy-answers.md — every permission justification and the data-usage
   declaration table from docs/09 section 4, verbatim, ready to paste into the
   dashboard.
6. store-assets/reviewer-notes.md — the notes from docs/09 section 5, with a real test
   account created and pre-loaded with credits.
7. Packaging: `pnpm --filter extension package` producing a zip with manifest.json at
   the ROOT, source maps included, no node_modules, no .env, under 10MB. Verify by
   unzipping to a temp dir and loading it unpacked.
8. .github/workflows/publish.yml from docs/09 section 6, triggered on v* tags.
9. docs/_runbooks/cws-credentials.md — the exact click-by-click for obtaining the
   Chrome Web Store API refresh token.
10. A final pre-submission checklist run (docs/08 section 7), with results recorded.

Then give me a numbered, click-by-click list of exactly what I do in the Chrome Web
Store Developer Dashboard, in order, including which file to upload where and which
text goes in which field. Assume I have never used the dashboard before.
```

**Exit:** a validated zip, all listing assets ready, and a submission walkthrough. Then: submit, and expect 1–3 weeks for first review of an extension with third-party host permissions.

---

## After submission

| When | Do |
|---|---|
| While in review | Set up the Substack publication you will dogfood on. Write three posts with the tool. |
| On approval | Staged rollout 10% → 50% → 100% over 72h, watching Sentry between stages. |
| Day 1–7 | Watch drift events, run success rate, install→signup conversion daily. Respond to every review within 24h. |
| Week 2 | Ship v1.0.1 with whatever the first users hit. Speed of first patch strongly predicts review sentiment. |
| Week 3–4 | Launch the /scorecard lead magnet properly: post it to r/Substack, Indie Hackers, and Substack Notes. |
| Month 2 | v1.1: scheduling queue, content calendar. |
| Month 3 | Evaluate the Ghost or Beehiiv adapter — it de-risks the platform dependency and roughly doubles the addressable market. |
