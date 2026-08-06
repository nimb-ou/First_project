# Draftsmith — Master Build Plan

> **Codename:** Draftsmith (store listing: *"Draftsmith — AI Writing & SEO for Substack"*)
> **What it is:** A Manifest V3 Chrome extension that runs an autonomous writing agent inside Substack — researches a topic, does keyword/SEO work, drafts the post in the author's own voice, optimises it, pushes it into the Substack editor, and schedules or publishes it.
> **Status:** Specification. No code written yet.
> **Owner:** nimitjain.248@gmail.com
> **Date:** 2026-07-31

---

## 0. Read this first — three corrections to the original brief

These change the build. They are not optional details.

### 0.1 "Let users connect their LLM subscription" is not possible

ChatGPT Plus, Claude Pro, Gemini Advanced, and Perplexity Pro are **consumer chat subscriptions with no API surface**. There is no OAuth flow, no token, no supported way for a third-party app to spend a user's Plus/Pro subscription. Products that appear to do this either (a) drive the chat web UI via browser automation — which violates every provider's ToS and gets extensions pulled from the Chrome Web Store, or (b) actually mean *API keys*, which are a separate paid product billed per token.

**What we build instead — two real modes:**

| Mode | What the user connects | Who pays the model | Our revenue |
|---|---|---|---|
| **BYOK** | Their own **API key** (Anthropic / OpenAI / Google / OpenRouter) | The user, directly to the provider | Flat software subscription |
| **Managed** | Nothing — just signs in | We do, via our provider account | Credits sold at a markup |

Both ship. BYOK is the wedge for power users (cheap, private, no trust barrier); Managed is where the margin is. Full design in [`docs/06-llm-modes-and-credits.md`](docs/06-llm-modes-and-credits.md).

### 0.2 Substack has no public write API

There is no documented, supported API for creating or publishing posts. There **is** an undocumented internal JSON API the Substack web app itself calls (`/api/v1/drafts`, `/api/v1/drafts/:id/publish`, etc.). We can call it from a content script running on the user's own Substack tab, using their own session cookies — it is the user's own account acting on the user's own behalf.

Consequences we design around:
- **It will break.** Endpoints and payload shapes change without notice. Everything Substack-facing goes behind a **versioned adapter** with a **ProseMirror DOM fallback** and a **capability probe** that runs at startup. See [`docs/02-substack-integration.md`](docs/02-substack-integration.md).
- **Human-in-the-loop by default.** The agent writes drafts. Publishing requires an explicit click. Auto-publish is opt-in, per-publication, off by default. This is both a product decision and a ToS-risk decision.
- **No credentials ever leave the browser.** We never ask for a Substack password, never proxy Substack traffic through our backend, never store session cookies. See [`docs/10-legal-risk-compliance.md`](docs/10-legal-risk-compliance.md).

### 0.3 Chrome Web Store does not process payments

Google shut down Chrome Web Store Payments in 2021. All billing runs through **Stripe**, on our own web app, linked from the extension. This is explicitly permitted. See [`docs/09-chrome-web-store-launch.md`](docs/09-chrome-web-store-launch.md).

---

## 1. The product in one screen

```
┌─ Chrome side panel (opens on any substack.com page) ─────────────┐
│                                                                   │
│  ● Draftsmith            [ Credits: 24,180 ]  [ ⚙ ]              │
│  ─────────────────────────────────────────────────────────────   │
│                                                                   │
│  What are we writing?                                             │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ why most RAG demos fail in production                    │     │
│  └─────────────────────────────────────────────────────────┘     │
│  Goal: [ Grow subscribers ▾ ]   Length: [ 1400 words ▾ ]         │
│  Voice: [ My last 20 posts ▾ ]  Mode: [ Agent — full run ▾ ]     │
│                                                                   │
│              [  ▶ Run agent  ]                                    │
│  ─────────────────────────────────────────────────────────────   │
│  RUN LOG                                                          │
│  ✓ Learned voice from 20 posts        (2.1s)                      │
│  ✓ Keyword research — 34 candidates   (4.8s)   ▸                  │
│  ✓ Angle chosen: contrarian teardown  (3.2s)   ▸                  │
│  ✓ Outline — 6 sections               (5.0s)   ▸                  │
│  ⟳ Drafting section 3/6 …                                         │
│    SEO audit                                                      │
│    Title + subtitle variants                                      │
│    Push to editor                                                 │
│  ─────────────────────────────────────────────────────────────   │
│  [ Pause ]  [ Edit outline ]           Est. cost: 3,200 credits   │
└───────────────────────────────────────────────────────────────────┘
```

Four surfaces:
1. **Side panel** — the agent workspace (Chrome 114+ `chrome.sidePanel`).
2. **Content script on the Substack editor** — reads and writes the ProseMirror document, fills title/subtitle/SEO fields, drives save.
3. **Service worker** — agent orchestrator, tool router, auth, streaming.
4. **Web app (`app.draftsmith.io`)** — sign-up, Stripe billing, credit top-up, account settings, privacy policy. Deliberately thin.

---

## 2. Feature scope

### v1.0 — ships to the store
- **Voice learning** — ingest 5–50 of the author's past posts from their public archive, build a style profile (sentence rhythm, vocabulary, structure, opening/closing patterns, formatting habits).
- **Agentic drafting** — topic → keyword research → angle selection → outline → section-by-section draft → self-critique → revision.
- **SEO engine** — keyword research, SERP analysis, title/subtitle optimisation (subtitle *is* the meta description on Substack), heading structure, internal linking to the author's own archive, image alt text, slug.
- **Editor bridge** — one-click push of the finished post into the Substack editor with full formatting preserved.
- **SEO scorecard** — live 0–100 score on any open draft with specific, actionable fixes.
- **Repurpose** — turn the post into a Notes teaser, an X thread, and a LinkedIn post.
- **Modes** — `Agent` (full autonomous run), `Co-write` (section at a time, human approves each), `Audit only` (no generation, just scoring).
- **BYOK + Managed credits**, Stripe billing, usage dashboard.

### v1.1–v1.3 — fast-follow
- Scheduling queue and content calendar
- Series/pillar planning with internal-link graph
- A/B title testing against open-rate history
- Multi-publication support
- Team seats
- Ghostwriting for someone else's voice (uploaded samples)

### Explicitly out of scope for v1
- Auto-publish without human approval (opt-in only, v1.2, behind a warning)
- Image generation (licensing/cost complexity)
- Comment auto-replies (spam risk, store-policy risk)
- Any non-Substack platform

---

## 3. Tech stack — decided

| Layer | Choice | Why |
|---|---|---|
| Extension | **Manifest V3**, TypeScript, React 18, Vite + `@crxjs/vite-plugin` | Only supported manifest; CRXJS gives HMR on content scripts |
| UI | Tailwind CSS + shadcn/ui (Radix) | Fast, accessible, no runtime CSS-in-JS in a CSP-restricted context |
| Ext. state | Zustand + `chrome.storage.local` persistence adapter | Small, no boilerplate, survives service-worker death |
| Editor bridge | Direct ProseMirror transactions via a `MAIN`-world injected script | The only reliable way to write into Substack's editor |
| Backend | **Hono on Cloudflare Workers** | Edge latency, streaming-native, cheap at low volume, no cold-start tax |
| DB | **Supabase Postgres** + Drizzle ORM | Managed Postgres, RLS, good local dev story |
| Auth | Supabase Auth (Google + email OTP) via `chrome.identity.launchWebAuthFlow` | Standard, no password handling |
| Payments | **Stripe** — Checkout, Billing Portal, webhooks | CWS payments no longer exist |
| SEO data | **DataForSEO** (primary) + Google Suggest (cheap expansion) + **Serper.dev** (SERP) | Real volume/difficulty data at usable unit cost |
| Web app | Next.js 15 on Vercel | Marketing site + billing + legal pages |
| Tests | Vitest (unit), Playwright with persistent context (E2E), MSW (network mocks) | Playwright is the only credible way to E2E an MV3 extension |
| CI/CD | GitHub Actions → build, test, zip, upload via Chrome Web Store API | Reproducible submissions |
| Errors | Sentry (extension + worker), with PII scrubbing | Required to survive Substack DOM drift |

---

## 4. Repository layout

```
draftsmith/
├─ apps/
│  ├─ extension/              # MV3 extension
│  │  ├─ src/
│  │  │  ├─ background/       # service worker: agent loop, router, auth
│  │  │  ├─ sidepanel/        # React UI
│  │  │  ├─ content/          # substack content script (ISOLATED world)
│  │  │  ├─ injected/         # MAIN-world ProseMirror bridge
│  │  │  ├─ agent/            # planner, tools, prompts, schemas
│  │  │  ├─ substack/         # versioned adapters + capability probe
│  │  │  ├─ llm/              # provider clients (BYOK) + managed proxy client
│  │  │  ├─ lib/              # storage, crypto, messaging, telemetry
│  │  │  └─ types/
│  │  ├─ public/              # icons, _locales
│  │  ├─ manifest.config.ts
│  │  └─ vite.config.ts
│  ├─ api/                    # Hono on Cloudflare Workers
│  │  ├─ src/routes/          # llm, seo, credits, stripe, account
│  │  ├─ src/services/        # metering, provider fanout, rate limits
│  │  └─ wrangler.toml
│  └─ web/                    # Next.js: marketing, billing, legal
├─ packages/
│  ├─ shared/                 # zod schemas, types, credit math — used by all three
│  └─ prompts/                # versioned prompt templates + evals
├─ e2e/                       # Playwright specs + Substack DOM fixtures
├─ docs/                      # this specification
└─ .github/workflows/
```

Monorepo: **pnpm workspaces + Turborepo**. `packages/shared` is the contract between extension and API — every request/response shape is a Zod schema defined once there.

---

## 5. Document index

| Doc | Covers |
|---|---|
| [`docs/01-architecture.md`](docs/01-architecture.md) | Manifest, permissions, message bus, service-worker lifecycle, security model |
| [`docs/02-substack-integration.md`](docs/02-substack-integration.md) | Internal API map, ProseMirror bridge, adapter/probe pattern, breakage handling |
| [`docs/03-agent-and-prompts.md`](docs/03-agent-and-prompts.md) | Agent loop, tool definitions, every system prompt, voice profiling, eval harness |
| [`docs/04-seo-engine.md`](docs/04-seo-engine.md) | What SEO actually means on Substack, keyword pipeline, scoring rubric, data vendors |
| [`docs/05-backend-api-and-data-model.md`](docs/05-backend-api-and-data-model.md) | Full DB schema, every endpoint, RLS policies, rate limiting |
| [`docs/06-llm-modes-and-credits.md`](docs/06-llm-modes-and-credits.md) | BYOK key handling, managed proxy, credit metering, reserve/reconcile |
| [`docs/07-business-model-and-pricing.md`](docs/07-business-model-and-pricing.md) | Unit economics with real token math, tiers, Stripe objects, growth plan |
| [`docs/08-testing.md`](docs/08-testing.md) | Unit/integration/E2E strategy, DOM fixtures, prompt evals, pre-submit checklist |
| [`docs/09-chrome-web-store-launch.md`](docs/09-chrome-web-store-launch.md) | Listing copy, assets, permission justifications, review survival, CI publishing |
| [`docs/10-legal-risk-compliance.md`](docs/10-legal-risk-compliance.md) | Substack ToS, trademark, GDPR, privacy policy, kill switches |
| [`docs/11-build-sessions.md`](docs/11-build-sessions.md) | **18 scoped Claude Code sessions with copy-paste prompts and exit criteria** |

---

## 6. Build order

Eighteen sessions, each scoped to one Claude Code context. Detail and copy-paste prompts in [`docs/11-build-sessions.md`](docs/11-build-sessions.md).

| # | Session | Ships |
|---|---|---|
| 1 | Monorepo scaffold | pnpm + Turbo + TS config + lint + CI skeleton |
| 2 | Extension shell | MV3 manifest, side panel, service worker, message bus |
| 3 | Substack recon | Capability probe + adapter interface + DOM fixtures captured |
| 4 | Editor bridge | Read/write ProseMirror, fill metadata, save draft |
| 5 | Shared contracts | Zod schemas for every message, tool, and API shape |
| 6 | LLM layer — BYOK | Provider clients, streaming, key encryption, key test flow |
| 7 | Agent core | Planner/executor loop, tool registry, cancellation, run log |
| 8 | Voice profiling | Archive ingest, style extraction, profile storage |
| 9 | SEO engine | Keyword pipeline, SERP analysis, scoring rubric |
| 10 | Writing tools | Outline, section draft, critique, revise, title/subtitle |
| 11 | Side panel UI | Full run experience, streaming log, diff review, approval gates |
| 12 | Backend v1 | Hono worker, Supabase schema, auth, SEO proxy |
| 13 | Managed credits | LLM proxy, metering, reserve/reconcile, balance UI |
| 14 | Stripe billing | Products, Checkout, Portal, webhooks, entitlement sync |
| 15 | Web app | Marketing, pricing, dashboard, privacy policy, terms |
| 16 | Test suite | Vitest + Playwright E2E + prompt evals + CI gates |
| 17 | Hardening | Sentry, kill switch, rate limits, adapter drift alerts, i18n scaffold |
| 18 | Store submission | Assets, listing, justifications, CI publish, staged rollout |

**Suggested pace:** sessions 1–5 in week 1, 6–11 in weeks 2–3, 12–15 in week 4, 16–18 in week 5. Realistic solo timeline to a store-approved v1.0: **5–7 weeks**.

---

## 7. Costs before first revenue

| Item | Cost |
|---|---|
| Chrome Web Store developer account | **$5** one-time |
| Domain (`draftsmith.io`) | ~$35/yr |
| Cloudflare Workers | $0 (free tier) → $5/mo |
| Supabase | $0 → $25/mo at scale |
| Vercel | $0 → $20/mo |
| Stripe | 2.9% + $0.30 per charge |
| DataForSEO | Pay-as-you-go, ~$0.02–0.05 per keyword research run |
| Sentry | $0 (developer tier) |
| LLM spend for managed users | Variable — fully covered by credit markup, see doc 07 |
| **Fixed monthly at launch** | **≈ $10–15** |

---

## 8. Success criteria for v1.0

- Agent completes a full run (topic → editor-ready draft) in **under 3 minutes** for a 1,500-word post.
- Voice match is good enough that the author edits **less than 20%** of the text before publishing.
- Editor bridge succeeds on **≥99%** of attempts, with the DOM fallback covering internal-API failures.
- Chrome Web Store approval on **first submission**.
- Managed-mode **gross margin ≥ 70%** at the published credit price.
- 40+ installs and 5 paying customers within 30 days of launch.

---

## 9. How to use this plan with Claude Code

1. `cd ~/Desktop/First_project/substack-agent`
2. Open [`docs/11-build-sessions.md`](docs/11-build-sessions.md).
3. Start a fresh Claude Code session per numbered session. Paste that session's prompt verbatim.
4. Each session prompt names exactly which docs to read — do not tell Claude to read the whole `docs/` folder, it wastes the context budget.
5. At the end of each session, verify against that session's **Exit criteria** before moving on.
6. Commit at every session boundary with `feat(sN): <session title>`.
