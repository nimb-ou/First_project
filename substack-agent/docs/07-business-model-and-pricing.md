# 07 — Business Model & Pricing

## 1. Who pays and why

| Segment | Size signal | Pain | Willingness to pay |
|---|---|---|---|
| **Serious solo newsletter writers** (weekly, 500–10k subs, monetising or trying to) | The core. Substack has ~50k+ publications with paid subscriptions. | Publishing cadence is the whole game and they miss weeks. Discovery beyond Substack's own network is a black box. | **High** — they already pay for Grammarly, Canva, an email tool. $20–50/mo is normal. |
| Newsletter operators running 2–5 publications | Small but rich | Pure throughput | Very high — $99+/mo |
| Agencies / ghostwriters | Small | Voice-matching a client is the hard part | Very high, but wants team seats — v1.2 |
| Hobbyist writers | Huge | "I have nothing to write about" | Low — free tier, converts rarely. Serve them cheaply. |

**Beachhead: the serious solo writer.** Every v1 decision serves them.

---

## 2. Choosing the markup

The user's instinct — *"charge a bit higher than the actual credit cost"* — is right in direction, wrong in magnitude. A thin markup on tokens loses money once you account for everything a token call actually costs you:

| Layer | Real cost per $1.00 of provider spend |
|---|---|
| Provider tokens | $1.00 |
| Stripe (2.9% + $0.30, small tickets) | ~$0.12 |
| Failed runs, retries, refunds | ~$0.08 |
| SEO vendor calls | ~$0.10 |
| Infra (Workers, Supabase, Vercel, Sentry) | ~$0.06 |
| Support time | ~$0.15 |
| **Break-even multiplier** | **≈ 1.5×** |

So a 1.2× markup is a **loss-making business** at any volume. A 2.6× markup gives ~62% gross margin uncached, ~77% with prompt caching on — which is a normal, defensible SaaS margin and leaves room for a free tier, refunds, and a real support answer when something breaks.

**Decision: `MARKUP = 2.6`.**

Two important framing points:
1. **Never show users the raw provider cost next to the credit price.** Credits are an abstraction with a legitimate purpose: they smooth across models, absorb provider price changes without repricing, and bundle the SEO and infrastructure costs a token count does not reflect. This is standard and honest. What is *not* acceptable is claiming credits are "at cost" — so don't claim it.
2. **BYOK users pay a flat fee, not a markup.** They cover their own tokens. That subscription is nearly pure margin and is a genuinely better deal for heavy users — say so, and let them choose. Offering the cheaper option openly is worth more in trust than the margin you give up.

---

## 3. Tiers

| | **Free** | **BYOK** | **Pro** | **Studio** |
|---|---|---|---|---|
| Price | $0 | **$9/mo** | **$29/mo** | **$79/mo** |
| Annual (2 months free) | — | $90/yr | $290/yr | $790/yr |
| LLM | Managed | Your own key | Managed | Managed |
| Credits/mo | 40k (**≈ 1 post**) | Unlimited* | 350k (**≈ 11 posts**) | 1.2M (**≈ 38 posts**) |
| Draft model | Sonnet | Your choice | Sonnet + Opus | Sonnet + Opus |
| Voice profiles | 1 | 3 | 3 | Unlimited |
| Publications | 1 | 1 | 1 | 5 |
| SEO research | 5 runs/mo | Unlimited | Unlimited | Unlimited |
| SEO scorecard | ✓ | ✓ | ✓ | ✓ |
| Editor push | ✓ | ✓ | ✓ | ✓ |
| Repurposing | — | ✓ | ✓ | ✓ |
| Scheduling (v1.1) | — | ✓ | ✓ | ✓ |
| Team seats (v1.2) | — | — | — | 3 |
| Support | Community | Email | Email, 24h | Priority, 4h |

\* "Unlimited" on BYOK = fair use, 100 runs/month, then a soft rate limit. Our cost per BYOK run is only the SEO vendor calls (~$0.03), so this is safe.

**Credit top-ups** (any paid plan, credits never expire while subscribed):
- 250k — $12 (≈ 8 posts)
- 800k — $32 (≈ 25 posts)
- 2.5M — $89 (≈ 80 posts)

Top-ups price slightly below the equivalent subscription rate so upgrading always beats repeat top-ups. That is the intended nudge.

### Why these numbers

- **$29 Pro** sits where writers already pay (Grammarly Premium $30, Ahrefs Lite $129, Jasper $49). If one post per week takes two hours and this saves half of it, the ROI argument writes itself.
- **11 posts/month on Pro** covers weekly publishing with slack for redrafts. Under-provisioning credits is the fastest way to make a subscription feel like a metered punishment — users then ration usage, get less value, and churn.
- **$9 BYOK** is deliberately cheap. It converts the technical sceptics who would otherwise never sign up, and its margin is ~95%.
- **Free = 1 post** is enough to experience the full pipeline once, which is the actual conversion event. A free tier that only shows a teaser converts worse than one that delivers a complete result.

### Unit economics per tier (monthly, at typical usage)

| Tier | Revenue | LLM cost | SEO cost | Stripe | Infra | **Gross margin** |
|---|---|---|---|---|---|---|
| Free | $0 | $0.28 | $0.05 | $0 | $0.02 | **−$0.35** (CAC) |
| BYOK | $9.00 | $0 | $0.60 | $0.56 | $0.10 | **$7.74 (86%)** |
| Pro (uses 80%) | $29.00 | $2.50 | $0.90 | $1.14 | $0.20 | **$24.26 (84%)** |
| Pro (uses 100%) | $29.00 | $3.13 | $1.10 | $1.14 | $0.20 | **$23.43 (81%)** |
| Studio (uses 80%) | $79.00 | $8.60 | $2.40 | $2.59 | $0.40 | **$65.01 (82%)** |

(With prompt caching on. Free-tier cost is the customer-acquisition line — at a 4% free→paid conversion it costs ~$8.75 to acquire a Pro customer through the free tier, which is excellent.)

**Break-even: ~4 Pro customers** covers all fixed costs. **50 Pro customers ≈ $1,200 MRR at ~83% margin.**

---

## 4. Stripe setup

### Objects

```
Products                          Prices
─────────────────────────────────────────────────────────────
Draftsmith BYOK                   price_byok_monthly      $9/mo
                                  price_byok_yearly       $90/yr
Draftsmith Pro                    price_pro_monthly       $29/mo
                                  price_pro_yearly        $290/yr
Draftsmith Studio                 price_studio_monthly    $79/mo
                                  price_studio_yearly     $790/yr
Credit Pack 250k                  price_pack_250k         $12  (one-time)
Credit Pack 800k                  price_pack_800k         $32  (one-time)
Credit Pack 2.5M                  price_pack_2500k        $89  (one-time)
```

Put `plan` and `monthly_credits` in **price metadata**, so adding a tier never requires a code deploy.

### Flow

The extension cannot host Stripe Checkout (CSP + review risk). Instead:

1. Side panel → `POST /v1/billing/checkout { priceId }` → returns a Checkout URL.
2. `chrome.tabs.create({ url })` — Checkout opens in a normal tab.
3. `success_url` = `https://app.draftsmith.io/billing/done?session_id={CHECKOUT_SESSION_ID}` — a plain page saying "Done, go back to the extension."
4. The extension polls `GET /v1/me` every 3s for 60s after opening Checkout, and updates the plan badge when it flips. Also refresh on side-panel focus.

### Webhooks — `POST /v1/stripe/webhook`

| Event | Action |
|---|---|
| `checkout.session.completed` | Subscription → set plan, grant credits. One-time → add pack credits. |
| `customer.subscription.updated` | Sync plan/status/period end. On upgrade, prorate the credit grant. |
| `customer.subscription.deleted` | Downgrade to free at period end. **Keep purchased top-up credits** — they were paid for separately. |
| `invoice.paid` | Monthly credit grant. Idempotent on `invoice.id`. |
| `invoice.payment_failed` | `past_due`; in-app banner; block new runs after a 3-day grace period. |
| `charge.dispute.created` | Freeze the account, alert, manual review |

**Idempotency is non-negotiable.** Stripe retries. Every handler writes `credit_ledger.stripe_ref = event.id` under a unique index; a duplicate insert is caught and ignored. Verify signatures with `STRIPE_WEBHOOK_SECRET` on every request — an unverified webhook endpoint is a free-credits API.

### Tax

Enable **Stripe Tax**. Digital services to EU/UK consumers trigger VAT obligations from the first sale. Set product `tax_code` to `txcd_10000000` (SaaS). Register for EU OSS once EU revenue is material. Do not defer this — retroactive VAT is expensive.

---

## 5. Free → paid mechanics

The free tier gives one full post per month. The conversion moment is when the run *finishes*, not when it's blocked. Sequence:

1. Free user completes a run → sees a complete, good draft in their editor.
2. Panel then shows: *"That used your free post for July. Pro gives you 11 a month plus Opus drafting — $29."* with a one-click upgrade.
3. If they don't convert, a monthly email when credits reset: *"Your free post is back."* Re-engagement beats nagging.

Anti-abuse: free tier is keyed to a verified email and an install ID; one free grant per email; disposable-domain blocklist; if `runs` for an account exceed 3× the grant via multi-account signals, require a card on file for continued free use.

---

## 6. Go-to-market

**Distribution is the hard part, not the build.** The extension is 6 weeks; distribution is forever.

1. **Chrome Web Store SEO** — the store has its own search. Target "substack", "newsletter writing", "AI writing", "SEO writer" in the title, short description, and first 200 characters of the long description. The single most-neglected acquisition channel for extensions.
2. **Dogfood publicly** — run a Substack about newsletter growth, written with the tool, and say so. Every post is both a lead magnet and a proof of quality. This is the highest-ROI channel and it costs nothing but time.
3. **Substack Notes** — where Substack writers actually gather. Be useful there for months before selling anything.
4. **r/Substack, r/newsletters, Indie Hackers** — participate; don't drop links.
5. **Free SEO scorecard as a lead magnet** — a public web page where anyone pastes a Substack URL and gets a score. No install required. Best top-of-funnel asset in the plan; build it in Session 15.
6. **Lifetime deal on AppSumo** — consider only after the product is stable. It brings cash and reviews but a permanently expensive support cohort. Not before month 6.

**Targets:** 100 installs and 5 paying by day 30 · 500 installs and 30 paying (~$800 MRR) by day 90 · 2,000 installs and 120 paying (~$3,200 MRR) by month 6.

---

## 7. What kills this business

Name the risks now, so they get designed for rather than discovered.

| Risk | Severity | Mitigation |
|---|---|---|
| **Substack ships its own AI writer** | Existential | Likely at some point. Defence: be better at *voice matching and SEO* specifically, and be cross-platform-ready (Ghost, Beehiiv adapters are a 2-week port each because everything is behind the adapter interface). Do not build anything Substack-specific outside `src/substack/`. |
| **Substack blocks the integration** | High | Human-in-the-loop only, polite rate limits, identifying header, no credential handling, instant kill switch. Clipboard mode means the product still works if the integration dies. |
| **Substack DOM/API churn** | Medium, constant | Adapter pattern, nightly canary, drift telemetry, 24h fix SLA. Budget ~2 hours/week forever. |
| **Provider price increases** | Medium | Multiplicative markup absorbs it automatically; model routing table is one file. |
| **Google devalues AI content** | Medium | Voice matching + first-hand angle selection is exactly the defence. Never position as "mass-produce content". |
| **Chrome Web Store removal** | High | Minimal permissions, no remote code, accurate disclosures, no automation of third-party sites without user action. See doc 09. |
| **Commodity competition** | Medium | The moat is voice profiling quality and the Substack-native editor bridge, not the LLM. Invest the eval budget there. |
