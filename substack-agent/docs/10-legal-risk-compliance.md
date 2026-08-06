# 10 — Legal, Risk & Compliance

> Not legal advice. This is an engineering-grade risk register with the design decisions that follow from it. Have a lawyer review the Terms and Privacy Policy before taking money — budget $500–1,500 for a SaaS terms review.

## 1. The Substack question

**The risk:** Substack's Terms of Use, like most platforms', restrict automated access, scraping, and use of undocumented APIs. We build on an undocumented internal API and inject scripts into their pages.

**Why this is a defensible position, not a reckless one:**

| Factor | Our design |
|---|---|
| Whose account? | The user's own. We never act on data the user cannot already access. |
| Whose credentials? | The user's own browser session. We never request, store, transmit, or handle a Substack password or cookie. |
| Who initiates? | Always the user, always by an explicit click. No background polling, no scheduled scraping. |
| What is published? | Nothing, without an explicit human approval click. Auto-publish is off and opt-in. |
| Load imposed? | ≤1 req/1.5s per tab, archive capped at 50 posts, 30-day cache, exponential backoff. |
| Identifiable? | `X-Requested-With: Draftsmith` on every request, so Substack can find and contact us. |
| Reversible? | Remote kill switch disables all writes within 15 minutes. |

This is materially the same posture as a password manager, an accessibility tool, or Grammarly — user-directed automation of the user's own account. It is not scraping, not resale of platform data, not credential harvesting.

**It is still a real risk.** Substack could send a cease-and-desist, change their API to break us, or add anti-automation measures. Mitigations, in order:

1. **Clipboard mode always works.** If the integration is killed entirely, the product still generates the post and the user pastes it. That is a degraded product, not a dead one — this single decision de-risks the whole business.
2. **The adapter interface is platform-agnostic.** Ghost, Beehiiv, and WordPress all have *official, documented* APIs. Porting is roughly two weeks each. Keep every Substack-specific line inside `src/substack/`; enforce with an ESLint import boundary rule.
3. **Reach out proactively.** Once you have ~200 users, email Substack partnerships describing what you built and how you limit load. Many platforms are fine with well-behaved integrations and some will formalise access. Being the developer who introduced themselves beats being the one discovered in the logs.
4. **Respond immediately to any contact.** If Substack objects, comply first and negotiate second. Have the kill switch tested.

**Do not:** create accounts programmatically, bypass rate limits, publish without human approval, scrape other people's publications at scale, resell Substack data, or use the Substack logo/wordmark.

---

## 2. Trademark

- **"Substack"** is Substack Inc.'s trademark. Descriptive use ("for Substack", "works with Substack") is nominative fair use when: the product cannot reasonably be described without it, you use no more of the mark than necessary, and you do not imply endorsement.
- **Do:** put the disclaimer in the store listing, the website footer, and the extension's About panel. Use your own name and visual identity everywhere.
- **Do not:** use "Substack" as the first word of your product name, in your domain, in your logo, in Substack's typeface or brand orange, or in a way that implies partnership.
- Check Substack's published brand guidelines before submission and follow whatever they say over the general rule above.
- Clear "Draftsmith" against USPTO/EUIPO before spending on branding. Consider filing a word mark in class 9/42 (~$250–350 US) once you have revenue.

---

## 3. AI-specific obligations

**Provider terms.** Anthropic, OpenAI, and Google all permit building products on their APIs, and all require:
- Not presenting model output as human-authored where that would mislead → your users know; disclose in Terms that output is AI-generated.
- Usage policy enforcement → you must not knowingly enable prohibited uses. Add a Terms clause and a lightweight abuse check.
- Attribution where required → check each provider's current branding requirements for "powered by" disclosure.

**Output ownership.** Under current US law, purely AI-generated text is not copyrightable; human-edited work generally is. Say this plainly in the Terms rather than letting users assume: *"You own your posts. Note that purely machine-generated text may not be eligible for copyright protection in some jurisdictions; your edits and creative direction are yours."*

**No warranty on facts.** The agent can be wrong. The `[VERIFY]` mechanism (doc 03 §5.4) exists so unverified claims are surfaced, not buried. Terms must disclaim accuracy, and the UI must show `[VERIFY]` flags prominently — this is both an ethical obligation and your best defamation defence.

**EU AI Act.** A writing assistant is minimal-risk. The relevant obligation is **transparency**: users must know they are interacting with AI, and AI-generated content should be disclosed where it could mislead. Both are satisfied by the product being openly an AI writer. No conformity assessment required.

---

## 4. Privacy policy

Must be a **live, dedicated page** at `https://draftsmith.io/privacy`, written about the extension specifically. Required sections:

```
1. WHO WE ARE — legal entity, address, contact email, EU/UK representative if applicable

2. WHAT WE COLLECT
   Account:      email, plan, billing state (via Stripe)
   Usage:        run counts, token counts, model used, timestamps, error codes
   Content:      MANAGED MODE ONLY — your topic and draft text are sent to our servers
                 and forwarded to our AI provider to generate your post. We do not
                 store the text after the generation completes. We do not train on it.
                 BYOK MODE — your content never reaches our servers.
   Local only:   your API key (encrypted in your browser), your writing style profile,
                 your in-progress drafts.

3. WHAT WE NEVER COLLECT
   Your Substack password. Your Substack session cookies. Your subscriber list. Your
   browsing history. Any data from websites other than your own Substack pages.

4. THIRD PARTIES — Anthropic (AI, zero-retention API tier), Stripe (payments),
   Supabase (database), Cloudflare (hosting), Sentry (errors, PII-scrubbed),
   DataForSEO/Serper (keyword data — we send keywords, never your content).
   Link each one's privacy policy.

5. RETENTION — usage records 24 months; content not retained; account data until
   deletion + 30 days; backups purged within 90 days.

6. YOUR RIGHTS (GDPR/CCPA) — access, export, correct, delete, object, portability.
   How to exercise: in-app buttons plus privacy@draftsmith.io. 30-day response.

7. LEGAL BASIS (GDPR) — contract performance for the service; legitimate interest
   for security and error monitoring; consent for optional analytics.

8. CHILDREN — 16+ only.

9. CHANGES — 30 days' notice by email for material changes.
```

**Ship the in-app "Export my data" and "Delete my account" buttons in v1** (endpoints already specified in doc 05 §3). Retrofitting GDPR compliance is far more expensive than building it in, and reviewers check that the policy's promises are actually implemented.

**Zero-retention:** request the zero-data-retention tier from Anthropic for the Managed proxy. It makes section 2 above simply true and is a genuine selling point.

---

## 5. Terms of Service — clauses that matter

Beyond boilerplate:

- **Acceptable use** — no spam, no bulk content farms, no impersonation, no illegal content, no using the tool to generate content violating Substack's or the AI provider's policies. Reserve the right to suspend.
- **AI output disclaimer** — no warranty of accuracy, originality, or non-infringement; the user is responsible for reviewing before publishing. Load-bearing.
- **Third-party platform disclaimer** — not affiliated with Substack; Substack may change or block the integration; no refund obligation arises from a third-party platform change (but offer them anyway — goodwill is cheaper than chargebacks).
- **Credits** — credits are a prepaid service unit, not currency; not redeemable for cash; expire 12 months after purchase (state it, then be generous in practice); subscription credits reset monthly and do not roll over.
- **Refunds** — 14-day money-back on first subscription payment (also satisfies EU consumer withdrawal rights for digital services); pro-rated on annual within 30 days. State it clearly; honouring it costs less than disputes.
- **Limitation of liability** — capped at fees paid in the preceding 12 months.
- **Governing law** — your jurisdiction. If you incorporate, incorporate before launch, not after.

---

## 6. Security obligations

| Requirement | Implementation |
|---|---|
| Encryption in transit | TLS everywhere; HSTS on all domains |
| Encryption at rest | Supabase default; BYOK keys AES-GCM in the browser |
| Secret management | Cloudflare secrets; never in git; quarterly rotation; documented runbook |
| Access control | Supabase RLS on all user tables; service-role key server-side only |
| Dependency hygiene | Dependabot + `pnpm audit` in CI; block on high severity |
| Incident response | `docs/_runbooks/incident.md`: detect → contain → assess → notify within 72h (GDPR) → post-mortem |
| Extension account security | 2FA mandatory; publish credentials in GitHub secrets only; no shared logins |

**Highest-severity scenario: the extension developer account is compromised** and a malicious update ships to every user. Defences: 2FA, a separate publish-only Google account, a hardware key, and CI-only publishing (no manual uploads) so every shipped artifact has a commit hash behind it.

---

## 7. Risk register

| # | Risk | L | I | Mitigation | Owner action |
|---|---|---|---|---|---|
| 1 | Substack ToS action | Med | High | §1 posture, clipboard fallback, kill switch, proactive outreach | Test kill switch monthly |
| 2 | Substack DOM/API drift | **High** | Med | Adapters, nightly canary, drift telemetry, 24h SLA | Reserve 2h/week |
| 3 | CWS rejection | Med | Med | Doc 09 §4–5 | Budget 2 weeks for review |
| 4 | CWS removal post-launch | Low | **Critical** | Minimal permissions, accurate disclosures, CI-only publishing | Keep a web-app fallback path |
| 5 | Provider price rise | Med | Low | Multiplicative markup, one routing table | Review pricing quarterly |
| 6 | Credit metering bug | Med | High | Ledger invariants, exhaustive tests, nightly reconciliation | Alert on balance drift |
| 7 | BYOK key leak | Low | **Critical** | Encryption, lint rules, leak tests, Sentry scrubbing | Manual storage inspection each release |
| 8 | Prompt injection via SERP | Med | Med | Delimiters, no tools in generation steps, sanitisation | Add injection cases to evals |
| 9 | Defamation/false facts in output | Low | High | `[VERIFY]` flags, fact-risk critique, Terms disclaimer | Never remove the flags |
| 10 | Substack ships a competitor | Med | High | Voice + SEO depth, platform-agnostic adapters | Keep the port path warm |
| 11 | Chargebacks | Med | Low | Clear pricing, visible ledger, generous refunds, Stripe Radar | Monitor dispute rate <0.5% |
| 12 | Single-founder bus factor | High | High | Everything in git, runbooks written, no undocumented infra | Write runbooks in Session 17 |

---

## 8. Pre-launch legal checklist

- [ ] Entity formed (LLC or equivalent) — before taking payment
- [ ] Business bank account + Stripe account in the entity's name
- [ ] Terms of Service live, reviewed by a lawyer
- [ ] Privacy Policy live, matches actual data flows exactly
- [ ] Cookie/consent banner on the web app if using any analytics
- [ ] Stripe Tax enabled; VAT/GST registration path understood
- [ ] "Not affiliated with Substack" disclaimer in: store listing, website footer, extension About panel
- [ ] AI provider terms reviewed; zero-retention tier requested
- [ ] `security.txt` and `security@draftsmith.io` live
- [ ] Data export and delete endpoints implemented and manually tested
- [ ] Kill switch tested end-to-end at least once
- [ ] Incident runbook written and readable at 3am
