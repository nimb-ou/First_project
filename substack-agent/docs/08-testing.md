# 08 — Testing Strategy

## 1. What can actually break

Ordered by likelihood × damage. The test suite budget follows this order, not code coverage.

1. **Substack changes their DOM or internal API** — near-certain, repeatedly. → Fixture tests + nightly canary.
2. **Markdown → ProseMirror JSON produces a malformed doc** and the editor rejects or mangles it. → Property-based round-trip tests.
3. **Prompt regression** — a prompt edit makes the prose worse in a way no unit test sees. → Eval harness with a CI gate.
4. **Credit metering is wrong** — under-charging loses money silently, over-charging causes chargebacks. → Exhaustive unit tests + ledger invariant checks.
5. **Service worker dies mid-run and the run is lost.** → Resume tests with forced worker termination.
6. **A BYOK key leaks into storage, logs, or Sentry.** → Dedicated leak tests + lint rules.
7. **Stripe webhook double-processing grants double credits.** → Idempotency tests with replayed events.

---

## 2. Layers

```
             ┌──────────────────────────────────────┐
  ~12 specs  │  E2E — Playwright + real extension   │  slow, high confidence
             ├──────────────────────────────────────┤
  ~40 specs  │  Integration — worker + msw + fake   │
             │  Substack server + fake provider     │
             ├──────────────────────────────────────┤
  ~200 specs │  Unit — Vitest                       │  fast, run on save
             ├──────────────────────────────────────┤
  ~30 cases  │  Evals — prompt quality (nightly +   │  LLM-in-the-loop
             │  on prompt change)                   │
             └──────────────────────────────────────┘
```

---

## 3. Unit tests (Vitest)

Priority modules, with the specific properties to assert:

### `md-to-pm.ts` — the highest-value unit tests in the repo
```ts
import fc from 'fast-check'

test('round-trips every supported markdown construct', () => {
  fc.assert(fc.property(arbitraryMarkdown(), (md) => {
    const pm = mdToPm(md)
    expect(() => SubstackSchema.nodeFromJSON(pm)).not.toThrow()   // schema-valid
    expect(normalise(pmToMd(pm))).toEqual(normalise(md))          // lossless
  }), { numRuns: 500 })
})

test('golden fixture matches real Substack output byte-for-byte', () => {
  const md = readFixture('golden-post.md')
  expect(mdToPm(md)).toEqual(readFixture('golden-post.pm.json'))  // captured from live Substack
})

test.each([
  ['nested lists', '- a\n  - b\n    - c'],
  ['link inside bold', '**see [this](https://x.com)**'],
  ['code fence with language', '```ts\nconst x = 1\n```'],
  ['blockquote with list', '> - a\n> - b'],
  ['image with alt', '![a cat sitting](https://x.com/c.png)'],
  ['unsupported: table', '| a | b |\n|---|---|\n| 1 | 2 |'],   // must degrade, not throw
  ['html injection', '<script>alert(1)</script>'],             // must be escaped as text
])('handles %s', (_, md) => { expect(() => mdToPm(md)).not.toThrow() })
```

### `credits.ts`
- `creditsFor` matches hand-computed values for each model at boundary token counts.
- Always rounds up; never returns 0 for non-zero usage; never negative.
- Ledger invariant: `sum(delta) === balance_after` of the latest row, for 10,000 random operation sequences.
- Reserve → settle with actual < reserved releases exactly the difference.
- Reserve → settle called twice charges once (idempotent).

### `keystore.ts`
- Encrypt/decrypt round-trip.
- `chrome.storage.sync` is never called — assert on a spy over the whole module graph.
- A serialised `RunState` containing a key-shaped string fails a leak assertion.
- Sentry `beforeSend` strips `sk-ant-...` and `sk-...` from arbitrarily nested payloads.

### `score.ts` (SEO rubric)
- Every rule is deterministic: same input → same score, 1,000 runs.
- Score is always 0–100 and equals the sum of awarded points.
- Each rule has a `fix` string and a test that the fix text is non-empty.

### `runner.ts`
- Resume from every possible `cursor` value produces the same final artifacts as an uninterrupted run (with a deterministic fake LLM).
- `AbortSignal` mid-step leaves status `cancelling` and does not advance `cursor`.
- A step throwing a non-retryable error stops the run and records `error_code`.

---

## 4. Integration tests

**Fake Substack server** (`e2e/fake-substack/`) — a Hono app replaying the captured fixtures, plus fault-injection modes:

```ts
export const FAULTS = [
  'ok',
  'drift_field_renamed',      // draft_body → body
  'drift_endpoint_404',       // /api/v1/drafts moved
  'drift_shape_changed',      // draft_body is now HTML not PM JSON
  'rate_limited_429',
  'auth_expired_401',
  'slow_8s',
  'partial_5xx',              // fails on the 3rd call only
] as const
```

Assert for every fault: the adapter registry falls back correctly, the user sees the right `AppError` with a useful `hint`, a drift event is emitted, and **no data is lost** — the draft is always recoverable via clipboard.

**Fake LLM provider** — deterministic SSE responses, including: valid stream, mid-stream disconnect, malformed tool-call JSON, `429` with `retry-after`, `overloaded_error`, and a response that exceeds `max_tokens`. Assert the runner handles each without losing checkpointed work.

**Stripe** — use the Stripe CLI (`stripe trigger`) against a local worker. Replay each webhook **three times** and assert credits are granted exactly once.

---

## 5. E2E (Playwright)

MV3 extensions require a persistent context and headed Chromium:

```ts
// e2e/fixtures.ts
import { test as base, chromium, type BrowserContext } from '@playwright/test'
import path from 'node:path'

const EXT = path.resolve(__dirname, '../apps/extension/dist')

export const test = base.extend<{ context: BrowserContext; extensionId: string }>({
  context: async ({}, use) => {
    const ctx = await chromium.launchPersistentContext('', {
      channel: 'chromium',
      args: [`--disable-extensions-except=${EXT}`, `--load-extension=${EXT}`],
    })
    await use(ctx)
    await ctx.close()
  },
  extensionId: async ({ context }, use) => {
    let [sw] = context.serviceWorkers()
    if (!sw) sw = await context.waitForEvent('serviceworker')
    await use(sw.url().split('/')[2])
  },
})
```

### The 12 E2E specs

| # | Spec | Asserts |
|---|---|---|
| 1 | Fresh install → onboarding → sign in (mock OAuth) | Side panel opens, `/v1/me` fetched, plan badge shows Free |
| 2 | BYOK key paste → optional permission granted → validated | Key encrypted in storage, plaintext absent, badge shows Connected |
| 3 | Full agent run on the fake Substack editor | Draft appears in the ProseMirror doc with correct headings/bold/links |
| 4 | Run with side panel closed mid-run, then reopened | Run resumes from checkpoint; no duplicate steps in the log |
| 5 | Service worker force-terminated mid-run | `RunState` survives; resume completes |
| 6 | Cancel mid-run | Stops within 2s; credits settled at actual usage |
| 7 | Insufficient credits | Run blocked *before* any LLM call; top-up CTA shown |
| 8 | Adapter drift (fake server in `drift_endpoint_404`) | Clipboard fallback offered; correct message; drift event posted |
| 9 | SEO scorecard on an existing draft | Score is deterministic; each failed rule has a working "Apply fix" |
| 10 | Co-write mode approval gates | Run pauses at outline; edited outline is used downstream |
| 11 | Repurpose outputs | Note / X thread / LinkedIn generated and copyable |
| 12 | Sign out | All local state cleared, including keys and voice profiles |

**Never run E2E against real Substack.** Route `https://*.substack.com` to the fake server with `context.route()`. The only thing that touches live Substack is the nightly canary, against a dedicated throwaway publication.

---

## 6. Prompt evals

Run: on every change under `packages/prompts/`, and nightly.

```
pnpm eval --suite voice     # 8 publications × voice-match judge
pnpm eval --suite outline   # 10 topics × structure + rubric checks
pnpm eval --suite draft     # 12 sections × voice-match + AI-tells + word budget
```

**CI gate:**
- Mean voice-match score may not drop more than **0.2** vs the recorded baseline.
- AI-tell rate (deterministic detector) may not increase **at all**.
- Word-budget adherence must stay **≥90%** within ±15%.
- Zero fabricated-fact flags on the fact-risk suite (a fixed set of prompts designed to bait invented statistics).

Baselines live in `packages/prompts/evals/baselines.json`, updated deliberately by a human with a commit message explaining why.

---

## 7. Manual QA — pre-submission checklist

Run this before **every** Chrome Web Store submission. No exceptions.

**Install & permissions**
- [ ] Fresh Chrome profile, load unpacked, no console errors
- [ ] Permission prompt text is what you expect users to see
- [ ] Extension works with **no** optional permissions granted (Managed mode)
- [ ] Uninstall leaves no orphaned storage

**Core flow**
- [ ] Full agent run on a real test Substack: topic → editor, under 3 minutes
- [ ] Formatting survives: H2, H3, bold, italic, link, bullet list, numbered list, blockquote, code block
- [ ] Title and subtitle populate correctly
- [ ] Cmd-Z undoes the insert cleanly
- [ ] Voice profile from 20 real posts is recognisably the author

**Resilience**
- [ ] Offline mid-run → clear error, work not lost
- [ ] Airplane mode at start → useful message, no crash
- [ ] Two Substack tabs open → correct tab targeted
- [ ] Non-Substack tab → panel explains what to do
- [ ] Substack logged out → prompts sign-in, no crash

**Billing**
- [ ] Checkout in Stripe test mode → plan updates within 10s
- [ ] Billing portal opens and cancels correctly
- [ ] Credits decrement visibly; the ledger matches the balance
- [ ] Zero-credit state blocks runs gracefully

**Security**
- [ ] `chrome.storage.local` contains no plaintext API key (inspect manually)
- [ ] `chrome.storage.sync` is empty
- [ ] Sentry test event contains no content and no key
- [ ] Injected script rejects `postMessage` without the nonce (test from the page console)

**Store readiness**
- [ ] Version bumped in `package.json` and reflected in the built manifest
- [ ] All screenshots current (retake if any UI changed)
- [ ] Privacy policy URL live and matching actual behaviour
- [ ] Permission justifications match the manifest exactly

---

## 8. CI

```yaml
# .github/workflows/ci.yml
on: [push, pull_request]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint          # eslint + the no-innerHTML / no-storage.sync rules
      - run: pnpm typecheck
      - run: pnpm test          # vitest, coverage gate 70% on src/{agent,substack,llm,lib}
      - run: pnpm build
      - run: pnpm exec playwright install --with-deps chromium
      - run: xvfb-run pnpm test:e2e
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: playwright-report, path: playwright-report/ }

  evals:
    if: contains(github.event.head_commit.modified, 'packages/prompts')
    runs-on: ubuntu-latest
    steps: [ ... pnpm eval --ci ]   # uses a repo secret ANTHROPIC_API_KEY
```

Nightly canary is a separate scheduled workflow (`canary.yml`) that runs against the live test publication and opens a GitHub issue on failure.
