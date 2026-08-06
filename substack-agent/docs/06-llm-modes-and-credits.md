# 06 — LLM Modes & Credit System

## 1. The two modes

```
┌─────────────────────── BYOK ────────────────────────┐   ┌────────────── MANAGED ──────────────┐
│                                                      │   │                                      │
│  Side panel ──► Service worker ──► api.anthropic.com │   │  Side panel ──► Service worker       │
│                       ▲                              │   │                     │                │
│                  user's key                          │   │                     ▼                │
│              (encrypted, local only)                 │   │        api.draftsmith.io/v1/llm      │
│                                                      │   │                     │                │
│  Post content NEVER touches our servers.             │   │                     ▼                │
│  We charge for the software.                         │   │             api.anthropic.com        │
│                                                      │   │        (our key, metered, marked up) │
└──────────────────────────────────────────────────────┘   └──────────────────────────────────────┘
```

**Why ship both:**
- BYOK removes the biggest objection from technical users ("I'm not sending my unpublished writing to your server") and has ~95% gross margin with zero variable cost.
- Managed removes the biggest objection from everyone else ("what's an API key?") and is where volume revenue comes from.
- Together they cover the whole market with one codebase, because the only difference is which `LLMClient` implementation is instantiated.

**Onboarding default is Managed** with free starter credits. BYOK is offered as "Advanced" in settings. Do not make new users choose a provider on first run — that is where funnels die.

---

## 2. What to tell users about "connect your ChatGPT/Claude subscription"

Users will ask. The settings screen answers it directly, because being straight about this builds more trust than dodging it:

> **Can I use my ChatGPT Plus or Claude Pro subscription?**
> No — and neither can any other extension, despite what some claim. Those are chat
> subscriptions with no API. The only way a third-party app can use them is by
> automating the chat website, which breaks the provider's terms and gets extensions
> removed from the Chrome Web Store. We won't do that.
>
> What you *can* connect is an **API key**, which is a separate pay-per-use product
> from the same companies. It's usually cheaper than you'd expect — a typical
> 1,500-word post costs about **$0.10–0.25** in API credits.
>
> Or just use Managed mode and we handle it.

---

## 3. BYOK implementation

### Supported providers

| Provider | Endpoint | Browser CORS | Default model |
|---|---|---|---|
| Anthropic | `https://api.anthropic.com/v1/messages` | Requires `anthropic-dangerous-direct-browser-access: true` | `claude-sonnet-5` |
| OpenAI | `https://api.openai.com/v1/chat/completions` | Allowed | `gpt-4.1` |
| Google | `https://generativelanguage.googleapis.com/v1beta/...` | Allowed | `gemini-2.5-pro` |
| OpenRouter | `https://openrouter.ai/api/v1/chat/completions` | Allowed | user-selected |

Anthropic is the recommended default — prose quality on step 5 is the product.

### Key storage

```ts
// apps/extension/src/lib/keystore.ts
//
// Threat model: another extension cannot read our storage. Malicious page JS cannot
// either. The real risks are (a) us accidentally logging it, (b) us accidentally
// syncing it, (c) it landing in a Sentry payload. Encryption at rest mitigates casual
// disk inspection and makes accidental leakage inert.

const WRAP_KEY_ID = 'ds.kek.v1'

async function getKek(): Promise<CryptoKey> {
  // Non-extractable AES-GCM key, generated once per install, held in IndexedDB.
  // Non-extractable means even our own code cannot serialise it out.
  const existing = await idbGet(WRAP_KEY_ID)
  if (existing) return existing
  const key = await crypto.subtle.generateKey({ name: 'AES-GCM', length: 256 }, false, ['encrypt', 'decrypt'])
  await idbPut(WRAP_KEY_ID, key)
  return key
}

export async function saveKey(provider: Provider, plaintext: string) {
  const kek = await getKek()
  const iv = crypto.getRandomValues(new Uint8Array(12))
  const ct = await crypto.subtle.encrypt({ name: 'AES-GCM', iv }, kek, new TextEncoder().encode(plaintext))
  await chrome.storage.local.set({                       // local, NEVER sync
    [`key.${provider}`]: { iv: [...iv], ct: [...new Uint8Array(ct)], savedAt: Date.now() },
  })
}

export async function getKey(provider: Provider): Promise<string | null> { /* inverse */ }
```

Enforcement rules, checked in CI:
- ESLint rule banning `chrome.storage.sync.set` anywhere in the repo.
- Sentry `beforeSend` drops any event whose serialised body matches `/sk-[A-Za-z0-9_-]{20,}|sk-ant-[A-Za-z0-9_-]{20,}/`.
- A unit test asserts `JSON.stringify(runState)` never contains a key-shaped string.
- The key is read at call time and never placed in Zustand, React state, or `RunState`.

### Key validation flow

On paste: request the optional host permission → send a 1-token request to the provider → on success show `✓ Connected · claude-sonnet-5 · $0.00 spent` → on 401 show `That key was rejected. Check you copied the whole thing.` Never store an unvalidated key.

### Cost display in BYOK

Show the user their *actual provider cost* per run — not credits. `~$0.18 this run · ~$4.20 this month (estimated)`. It builds trust and reinforces that BYOK is cheap, which is the whole pitch.

---

## 4. Credit system (Managed mode)

### Definition

> **1 credit = 1 token of `claude-sonnet-5` output, at our list price.**

Anchoring credits to a single reference unit keeps the maths explainable. Everything else converts via a multiplier table.

```ts
// packages/shared/src/credits.ts

// Provider list prices, USD per million tokens. UPDATE WHEN PRICING CHANGES —
// this table is the single source of truth for margin.
export const PROVIDER_RATES = {
  'claude-opus-5':            { in:  5.00, out: 25.00 },
  'claude-sonnet-5':          { in:  3.00, out: 15.00 },
  'claude-haiku-4-5-20251001':{ in:  0.80, out:  4.00 },
} as const

export const REFERENCE = PROVIDER_RATES['claude-sonnet-5'].out   // 15.00 / Mtok
export const MARKUP = 2.6                                        // see doc 07 §2

// Credits charged for one call.
export function creditsFor(model: keyof typeof PROVIDER_RATES, inTok: number, outTok: number): number {
  const r = PROVIDER_RATES[model]
  const costUsd = (inTok * r.in + outTok * r.out) / 1_000_000
  const credits = (costUsd / (REFERENCE / 1_000_000)) * MARKUP
  return Math.ceil(credits)                                       // always round up, never down
}

// What we sell credits for.
export const CREDIT_PRICE_USD = (REFERENCE / 1_000_000) * MARKUP  // ≈ $0.000039 per credit
```

Worked example — one 1,500-word post in Agent mode on Balanced quality:

| Step | Model | In | Out | Our cost | Credits charged |
|---|---|---|---|---|---|
| Voice (cached after first) | sonnet | 0 | 0 | $0.0000 | 0 |
| Research | haiku | 6,000 | 1,200 | $0.0096 | 640 |
| Angle | sonnet | 4,000 | 900 | $0.0255 | 1,700 |
| Outline | sonnet | 5,000 | 1,600 | $0.0390 | 2,600 |
| Draft (6 × section) | sonnet | 30,000 | 9,000 | $0.2250 | 15,000 |
| Critique | sonnet | 8,000 | 1,500 | $0.0465 | 3,100 |
| Revise | sonnet | 12,000 | 4,000 | $0.0960 | 6,400 |
| SEO pass | haiku | 10,000 | 2,500 | $0.0180 | 1,200 |
| **Total** | | **75,000** | **20,700** | **$0.460** | **≈ 30,640** |

At `CREDIT_PRICE_USD ≈ $0.000039`, 30,640 credits = **$1.20 revenue** against **$0.46 cost** → **62% gross margin before Stripe fees**, ~59% after. The first Voice run adds ~$0.09 one-off.

On "Best" quality (Opus for drafting/revision), cost roughly doubles to ~$0.95 and credits to ~63,000 (**$2.46**) — margin holds because the markup is multiplicative.

**Display credits in thousands.** "30.6k credits" reads better than a raw number, and a plain-English translator sits next to the balance: *"≈ 9 posts left."*

### Reserve → stream → settle

Credits must be charged even if the client vanishes mid-stream, and must not be over-charged if a run is cancelled.

```
1. RESERVE   POST /v1/llm/reserve { runId, credits: estimate * 1.3 }
             → ledger row (-reserved, reason='run_reserve'), idempotent on runId
             → 402 if balance insufficient; UI offers top-up before any work starts

2. STREAM    each /v1/llm/chat call meters actual usage from the provider's own
             usage block (never estimate from text length) and accumulates on the
             run record. Metering runs in waitUntil — client disconnect does not
             skip it.

3. SETTLE    POST /v1/llm/settle { runId }
             → charged = sum(actual); ledger row (+reserved - charged, reason='run_settle')
             → run.credits_charged = charged, run.provider_cost_usd = actual USD

4. SWEEP     cron every 15 min: any run in 'running' with started_at < 30 min ago is
             force-settled at actual usage and marked 'error'. No reservation leaks.
```

### Refund policy — encoded, not ad hoc

| Situation | Action |
|---|---|
| Run fails before step 5 (Draft) | Full auto-refund |
| Run fails during/after step 5 | Charge actual usage only; unused reservation released |
| User cancels mid-run | Charge actual usage to the cancellation point |
| Provider 5xx | Full auto-refund of that call |
| User dislikes the output | Manual, one goodwill refund per account per month via support |

Auto-refunds write a `credit_ledger` row with `reason='refund'` and surface in the usage dashboard. Users see every credit movement — opacity here generates support tickets and chargebacks.

---

## 5. Cost controls

**Client side:**
- Pre-flight estimate shown before every run: *"This will use about 31k credits (≈ 9% of your monthly grant)."* Nothing starts without the user seeing the number.
- Hard cap per run, configurable, default 80k credits. Exceeding it pauses and asks.
- Voice profiles cached until the archive hash changes — the biggest single input cost, paid once.
- Aggressive prompt caching: the voice profile and outline are stable across all six section calls. Use Anthropic prompt caching with `cache_control` on the system block — this cuts step-5 input cost by ~85% and is the single highest-leverage optimisation in the product. **Do this in Session 6, not later.**

**Server side:**
- Per-account daily credit ceiling (10× monthly grant / 30) to contain runaway loops.
- Model allowlist — the client cannot request an arbitrary model string.
- `max_tokens` clamped server-side per step type.
- Anomaly alert if one account exceeds 3× its trailing 7-day median in a day.

With prompt caching enabled, the worked example above drops to roughly **$0.28 cost / 30.6k credits** → **~77% margin**. Budget with caching off, ship with it on.

---

## 6. `LLMClient` interface

One interface, two implementations. Everything above the interface is mode-agnostic.

```ts
// apps/extension/src/llm/types.ts
export interface LLMClient {
  readonly mode: 'byok' | 'managed'
  stream(req: {
    step: StepId
    model: ModelId
    system: string
    messages: Message[]
    tools?: ToolDef[]
    toolChoice?: { name: string }
    maxTokens: number
    cacheSystem?: boolean
    signal: AbortSignal
  }): AsyncIterable<
    | { type: 'text'; delta: string }
    | { type: 'tool_use'; name: string; input: unknown }
    | { type: 'usage'; inputTokens: number; outputTokens: number; cacheReadTokens?: number }
    | { type: 'error'; error: AppError }
  >
  estimate(steps: StepId[]): Promise<{ credits: number; usd?: number }>
}
```

`ByokClient` talks to the provider directly and reports `usd`. `ManagedClient` talks to our proxy and reports `credits`. The agent runner never knows which it has.
