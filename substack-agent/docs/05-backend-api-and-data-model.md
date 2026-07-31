# 05 — Backend API & Data Model

## 1. Why there is a backend at all

BYOK users could in principle run entirely client-side. We still need a backend for:

1. **Managed LLM mode** — our provider keys must never be in the extension.
2. **SEO vendor keys** + shared response cache (cuts vendor cost ~80%).
3. **Stripe** — subscriptions, credits, webhooks, entitlements.
4. **Remote config / kill switch** — turn off Substack writes within 15 minutes when they change something.
5. **Drift telemetry** — early warning on adapter breakage.

BYOK users touch endpoints 2–5 only. Their post content never reaches our servers, and we say so prominently.

**Stack:** Hono on Cloudflare Workers · Supabase Postgres (Drizzle) · Cloudflare KV (cache, rate limits) · Stripe.

---

## 2. Data model

```sql
-- ============ identity ============
create table users (
  id            uuid primary key default gen_random_uuid(),
  email         text unique not null,
  created_at    timestamptz not null default now(),
  deleted_at    timestamptz,                       -- soft delete, 30-day purge job
  locale        text not null default 'en-US',
  install_id    text                                -- links pre-signup anonymous usage
);

create table accounts (
  id                     uuid primary key default gen_random_uuid(),
  owner_user_id          uuid not null references users(id) on delete cascade,
  plan                   text not null default 'free',    -- free|byok|pro|studio
  plan_status            text not null default 'active',  -- active|past_due|canceled|trialing
  stripe_customer_id     text unique,
  stripe_subscription_id text unique,
  current_period_end     timestamptz,
  credit_balance         bigint not null default 0,       -- credits, integer, never float
  monthly_credit_grant   bigint not null default 0,
  last_grant_at          timestamptz,
  created_at             timestamptz not null default now()
);
create index on accounts (stripe_customer_id);

create table account_members (
  account_id uuid references accounts(id) on delete cascade,
  user_id    uuid references users(id) on delete cascade,
  role       text not null default 'member',       -- owner|admin|member
  primary key (account_id, user_id)
);

-- ============ substack linkage (metadata only — never credentials) ============
create table publications (
  id              uuid primary key default gen_random_uuid(),
  account_id      uuid not null references accounts(id) on delete cascade,
  substack_pub_id bigint,
  subdomain       text,
  custom_domain   text,
  display_name    text,
  post_count      int default 0,
  first_post_at   timestamptz,
  authority_ceiling int,                           -- computed, see doc 04 §3
  created_at      timestamptz not null default now(),
  unique (account_id, substack_pub_id)
);

-- Voice profiles sync only if the user opts in. Default is local-only.
create table voice_profiles (
  id             uuid primary key default gen_random_uuid(),
  publication_id uuid not null references publications(id) on delete cascade,
  version        int not null default 1,
  profile        jsonb not null,
  source_hash    text not null,                    -- hash of source post ids
  built_at       timestamptz not null default now()
);

-- ============ usage & billing ============
create table runs (
  id                uuid primary key,              -- client-generated, idempotency key
  account_id        uuid not null references accounts(id) on delete cascade,
  publication_id    uuid references publications(id) on delete set null,
  mode              text not null,                 -- agent|cowrite|audit
  llm_mode          text not null,                 -- byok|managed
  status            text not null,                 -- running|done|error|canceled
  topic_hash        text,                          -- sha256(topic) — NOT the topic itself
  input_tokens      bigint default 0,
  output_tokens     bigint default 0,
  credits_reserved  bigint default 0,
  credits_charged   bigint default 0,
  provider_cost_usd numeric(10,6) default 0,       -- our true cost, for margin analysis
  error_code        text,
  started_at        timestamptz not null default now(),
  finished_at       timestamptz
);
create index on runs (account_id, started_at desc);

-- Append-only ledger. Balance is derivable; `accounts.credit_balance` is a
-- materialised cache reconciled nightly. Never update a ledger row.
create table credit_ledger (
  id          bigserial primary key,
  account_id  uuid not null references accounts(id) on delete cascade,
  delta       bigint not null,                     -- + grant/purchase/refund, - spend
  reason      text not null,                       -- monthly_grant|purchase|run_spend|refund|adjustment|signup_bonus
  run_id      uuid references runs(id),
  stripe_ref  text,
  balance_after bigint not null,
  created_at  timestamptz not null default now()
);
create index on credit_ledger (account_id, created_at desc);
create unique index on credit_ledger (stripe_ref) where stripe_ref is not null;  -- webhook idempotency

-- ============ ops ============
create table drift_events (
  id               bigserial primary key,
  install_id       text,                           -- anonymous
  adapter_version  text not null,
  operation        text not null,
  http_status      int,
  error_path       text,
  extension_version text,
  created_at       timestamptz not null default now()
);
create index on drift_events (adapter_version, operation, created_at desc);

create table seo_cache (
  cache_key   text primary key,
  payload     jsonb not null,
  vendor      text not null,
  cost_usd    numeric(10,6) default 0,
  expires_at  timestamptz not null
);
```

### Data minimisation — deliberate omissions

We do **not** store: post content, drafts, topics in plaintext, Substack cookies or tokens, BYOK API keys, or IP addresses beyond Cloudflare's own transient logs. `runs.topic_hash` exists only for dedupe and abuse detection. This is what makes the privacy policy short and the Chrome Web Store review easy.

### RLS

Supabase RLS on every user-facing table:

```sql
alter table accounts enable row level security;
create policy account_read on accounts for select
  using (id in (select account_id from account_members where user_id = auth.uid()));
create policy account_write on accounts for update
  using (id in (select account_id from account_members where user_id = auth.uid() and role in ('owner','admin')));
```

The Worker uses the **service role key** for metering and webhooks (bypasses RLS) and the **user's JWT** for everything else. Never ship the service role key anywhere near the client.

---

## 3. API surface

Base: `https://api.draftsmith.io`. All responses `application/json`. Auth: `Authorization: Bearer <supabase_jwt>` except where noted.

### Auth
| Method | Path | Notes |
|---|---|---|
| `POST` | `/v1/auth/exchange` | Exchange the `launchWebAuthFlow` code for a session. No auth. |
| `POST` | `/v1/auth/refresh` | Refresh token rotation |
| `GET` | `/v1/me` | User + account + plan + balance + entitlements. The extension's boot call. |
| `DELETE` | `/v1/me` | GDPR delete. Soft-deletes, queues 30-day purge, cancels Stripe sub. |
| `GET` | `/v1/me/export` | GDPR export — JSON of everything we hold. |

### LLM proxy (Managed mode only)
| Method | Path | Notes |
|---|---|---|
| `POST` | `/v1/llm/estimate` | `{ steps[] }` → `{ credits, breakdown[] }`. Pre-flight, no charge. |
| `POST` | `/v1/llm/reserve` | `{ runId, credits }` → holds credits. Idempotent on `runId`. |
| `POST` | `/v1/llm/chat` | **SSE stream.** `{ runId, model, messages, tools?, max_tokens }` |
| `POST` | `/v1/llm/settle` | `{ runId }` → reconciles reservation vs actual, releases the remainder |

### SEO
| Method | Path | Notes |
|---|---|---|
| `POST` | `/v1/seo/keywords` | `{ seeds[], locale }` → enriched candidates. Cached 7d. |
| `POST` | `/v1/seo/serp` | `{ query, locale }` → top 10 + PAA + related. Cached 24h. |
| `POST` | `/v1/seo/embed` | `{ texts[] }` → vectors, for cannibalisation checks |

Both `seo/*` endpoints are available to **BYOK users too** — that is what their subscription buys.

### Billing
| Method | Path | Notes |
|---|---|---|
| `POST` | `/v1/billing/checkout` | → Stripe Checkout URL. Opened in a new tab. |
| `POST` | `/v1/billing/portal` | → Stripe Billing Portal URL |
| `GET` | `/v1/billing/usage?from=&to=` | Runs, credits, spend for the dashboard |
| `POST` | `/v1/stripe/webhook` | **No auth** — verified by Stripe signature |

### Ops
| Method | Path | Notes |
|---|---|---|
| `GET` | `/v1/config` | Remote config + kill switch. No auth. `Cache-Control: max-age=900`. |
| `POST` | `/v1/telemetry/drift` | No auth. Heavily rate-limited by install ID. |

---

## 4. Streaming proxy

```ts
// apps/api/src/routes/llm.ts
app.post('/v1/llm/chat', requireAuth, async (c) => {
  const body = ChatRequest.parse(await c.req.json())
  const account = c.get('account')

  if (account.credit_balance <= 0) return c.json(err('CREDITS_EXHAUSTED'), 402)
  if (!(await rateLimit(c.env.KV, `llm:${account.id}`, 60, 60))) return c.json(err('RATE_LIMITED'), 429)
  if (!ALLOWED_MODELS.has(body.model)) return c.json(err('BAD_MODEL'), 400)
  if (estimateTokens(body.messages) > MAX_INPUT_TOKENS) return c.json(err('LLM_CONTEXT_TOO_LARGE'), 400)

  const upstream = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'x-api-key': c.env.ANTHROPIC_API_KEY,
      'anthropic-version': '2023-06-01',
      'content-type': 'application/json',
    },
    body: JSON.stringify({ ...body, stream: true }),
  })
  if (!upstream.ok) return c.json(err(mapProviderError(upstream.status)), upstream.status)

  // Tee the stream: one copy to the client, one to the meter.
  const [toClient, toMeter] = upstream.body!.tee()
  c.executionCtx.waitUntil(meterFromStream(c.env, body.runId, account.id, body.model, toMeter))

  return new Response(toClient, {
    headers: { 'content-type': 'text/event-stream', 'cache-control': 'no-cache', 'x-run-id': body.runId },
  })
})
```

`meterFromStream` reads the SSE `message_delta` / `message_stop` events, extracts the authoritative `usage` block from the provider, converts to credits (doc 06 §4), and writes a `credit_ledger` row. Metering runs in `waitUntil` so a client disconnect does not skip the charge — **a disconnected client still consumed provider tokens and must still be billed.**

---

## 5. Rate limits

Per account, sliding window in KV:

| Endpoint | Limit |
|---|---|
| `/v1/llm/chat` | 60/min, 600/hour |
| `/v1/seo/keywords` | 20/hour (cache misses only) |
| `/v1/seo/serp` | 60/hour (cache misses only) |
| `/v1/telemetry/drift` | 10/hour per install ID |
| Everything else | 120/min |

Plus a **global vendor spend circuit breaker**: if today's DataForSEO spend exceeds `SEO_DAILY_CAP_USD`, serve cache-only and degrade to Google Suggest. Alert on trip.

---

## 6. Secrets

Cloudflare Worker secrets (`wrangler secret put`), never in `wrangler.toml`:

```
ANTHROPIC_API_KEY          OPENAI_API_KEY (fallback)
SUPABASE_URL               SUPABASE_SERVICE_ROLE_KEY   SUPABASE_JWT_SECRET
STRIPE_SECRET_KEY          STRIPE_WEBHOOK_SECRET
DATAFORSEO_LOGIN           DATAFORSEO_PASSWORD         SERPER_API_KEY
SENTRY_DSN                 ALERT_WEBHOOK_URL
```

Rotation runbook in `docs/_runbooks/rotate-secrets.md` (write in Session 17). Staging and production are separate Cloudflare environments with separate Supabase projects and Stripe test/live modes — never share.
