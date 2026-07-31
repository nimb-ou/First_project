# 09 — Chrome Web Store: Submission, Review, Launch

## 1. Account setup

1. Go to the [Chrome Web Store Developer Dashboard](https://chrome.google.com/webstore/devconsole) and pay the **$5 one-time** registration fee.
2. **Register as a group/organisation publisher**, not a personal account, if you ever intend to sell the business — transferring a personal-account extension is painful.
3. **Verify a publisher email and a website domain.** Verified publishers get a badge and, anecdotally, smoother reviews. Domain verification requires a DNS TXT record on `draftsmith.io`.
4. Enable 2FA. An extension account takeover is catastrophic — the attacker can push malicious code to every install.

---

## 2. Listing

### Name (45 char limit)
```
Draftsmith — AI Writing & SEO for Substack
```
**Trademark note:** "Substack" is Substack Inc.'s trademark. Using it **descriptively** ("for Substack") is generally defensible nominative fair use; using it as your own brand ("Substack AI Writer") is not. Rules: never lead with their mark, never use their logo or wordmark styling, never use their orange, and put a disclaimer in the description. Read Substack's brand guidelines before submitting; if they publish a restriction on third-party naming, follow it and rename.

### Short description (132 char limit)
```
An AI agent that researches, writes in your voice, and SEO-optimises your Substack posts — then drops them straight in your editor.
```

### Detailed description

Store search indexes the **first ~200 characters** most heavily. Front-load keywords, then write for humans.

```
Draftsmith is an AI writing and SEO agent for Substack writers. Give it a topic and it
researches keywords, learns your voice from your past posts, writes a complete draft,
optimises it for search, and inserts it into your Substack editor — in about two minutes.

Not another chatbot. An agent that does the whole job.

━━ HOW IT WORKS ━━
1. Open any Substack page and click Draftsmith in the side panel.
2. Type a topic. Pick a goal and a length.
3. Watch it work: keyword research → angle → outline → draft → self-critique → SEO pass.
4. Review, edit, and push it into your editor with one click.

━━ IT WRITES LIKE YOU ━━
Draftsmith reads up to 50 of your published posts and builds a style profile: sentence
rhythm, vocabulary, how you open, how you close, how formal you are, the words you'd
never use. Most users edit less than 20% of what it produces. It is not trying to sound
like ChatGPT — it is trying to sound like you.

━━ REAL SEO, HONESTLY SCOPED ━━
• Keyword research with real volume and difficulty data
• Filtered to keywords your publication can realistically rank for — not vanity terms
• SERP analysis: what's ranking and what it's missing
• Title and subtitle optimisation (your subtitle is your meta description)
• Heading structure, internal linking to your archive, image alt text, slug
• A 0–100 scorecard with one-click fixes on any draft
• Cannibalisation warnings when you're about to compete with your own post
We optimise every lever Substack gives you, and we tell you plainly where Substack
itself is the limit.

━━ ALSO INCLUDED ━━
• Co-write mode — approve each step instead of a full auto run
• Audit-only mode — score an existing draft, generate nothing
• Repurposing — turn each post into a Substack Note, an X thread, and a LinkedIn post
• Separate email subject line suggestions (they should not match your SEO title)

━━ BRING YOUR OWN KEY, OR DON'T ━━
Use your own Anthropic, OpenAI, Google, or OpenRouter API key and your writing never
touches our servers. Or use Managed mode and we handle everything. Your choice, clearly
explained, switchable any time.

━━ PRICING ━━
Free: 1 post per month. BYOK: $9/mo. Pro: $29/mo (11 posts). Studio: $79/mo (38 posts).
No credit card for the free tier.

━━ PRIVACY ━━
We never ask for your Substack password. We never store your Substack session. In BYOK
mode your drafts never leave your browser. Full policy: https://draftsmith.io/privacy

━━ NOT AFFILIATED ━━
Draftsmith is an independent tool and is not affiliated with, endorsed by, or sponsored
by Substack Inc. "Substack" is a trademark of Substack Inc., used here descriptively.

Questions: support@draftsmith.io
```

### Category & metadata
- **Category:** Productivity (Workflow & Planning)
- **Language:** English (add locales later via `_locales/`)
- Website, support email, and privacy policy URL — all three required, all must be live and reachable before submission.

---

## 3. Assets

| Asset | Spec | Notes |
|---|---|---|
| Icon | 128×128 PNG, no alpha at edges | Also ship 16/32/48. Must read at 16px — test it. |
| Screenshots | **1280×800** PNG, 1–5 (ship all 5) | The single biggest conversion lever on the listing |
| Small promo tile | 440×280 PNG | Required for featuring |
| Marquee promo tile | 1400×560 PNG | Optional; only used if Google features you |

### The five screenshots

Each gets a bold caption baked into the image (top third, large type — most people never read the description).

1. **"Topic in. Finished draft out."** — split screen: the composer with a topic typed, the Substack editor with the finished post.
2. **"It writes in your voice."** — the voice profile card next to two paragraphs, one labelled "your past post", one "Draftsmith", visually indistinguishable.
3. **"SEO that's actually achievable."** — the keyword table with the authority-ceiling filter visible and a "too hard for you right now" row greyed out.
4. **"Score any draft, fix it in one click."** — the scorecard at 62/100 with three fix cards.
5. **"Your key or ours."** — the mode toggle with the privacy note.

Build these in Figma at 2560×1600 and export at 50%. Do not use raw unannotated screenshots — they convert poorly.

---

## 4. Privacy disclosures — the part that gets extensions rejected

The **Privacy practices** tab requires a justification for every permission and a data-usage declaration. Vague answers trigger manual review; inaccurate ones get you removed.

### Single purpose statement
```
Draftsmith helps Substack writers create and optimise newsletter posts by generating
drafts in the author's own writing style and inserting them into the Substack editor.
```
One purpose, one sentence. Chrome's single-purpose policy is enforced. Do not list features here.

### Permission justifications — use these verbatim

| Permission | Justification |
|---|---|
| `storage` | Stores the user's settings, writing-style profile, and in-progress drafts locally so a generation can resume if the browser closes it. No data is stored remotely in bring-your-own-key mode. |
| `sidePanel` | The extension's entire user interface is a side panel, so the user can see their draft and the Substack editor at the same time. |
| `scripting` | Injects a small script into the user's own Substack editor page to insert generated text into the editor, which cannot be done from an isolated content script. |
| `tabs` | Locates and focuses the user's already-open Substack editor tab so the generated draft can be inserted into the correct tab. We read only the URL, to identify Substack tabs. |
| `identity` | Signs the user into their Draftsmith account using a standard OAuth flow. |
| `alarms` | Refreshes the user's session token and resumes generations that were interrupted when the browser suspended the extension. |
| Host: `substack.com`, `*.substack.com` | The extension reads the user's own published posts to learn their writing style, and writes generated drafts into their own Substack editor. It operates only on the user's own account, using their existing session, and only when the user starts a generation. |
| Host: `api.draftsmith.io` | The extension's own backend, used for account, billing, and SEO keyword data. |
| Optional hosts: AI provider APIs | Requested only if the user chooses to connect their own AI provider API key, so their requests go directly to that provider instead of through our servers. |
| Remote code | **No.** All code is bundled in the package. The extension fetches text data only, never executable code. |

### Data usage declaration

Declare **truthfully**:

| Data type | Collected? | Detail |
|---|---|---|
| Personally identifiable information | **Yes** — email | For account and billing only |
| Authentication information | **No** | We never collect Substack credentials. BYOK keys are stored locally, encrypted, and never transmitted to us. |
| Website content | **Yes, conditionally** | In Managed mode, post text is transmitted to our servers to generate the draft. It is not stored after generation. In BYOK mode it is never transmitted to us. Declare this and explain it. |
| Personal communications | No |
| Location / health / financial / web history / user activity | No |

Then tick the three certifications: data is not sold, not used for unrelated purposes, and not used for creditworthiness/lending.

**"Website content: Yes" will not get you rejected. Getting caught understating it will.** Reviewers compare declarations against actual network behaviour.

---

## 5. Surviving review

Timeline: usually **1–3 business days**; extensions with host permissions on a third-party site can take **1–3 weeks** on first submission. Plan for two weeks.

### Rejection causes, in order of likelihood for this product

1. **Permission not justified** → every permission has a specific, functional justification above. Never say "for functionality".
2. **Single purpose violation** → do not mention future features (multi-platform, teams) in the listing. Ship narrow.
3. **Undisclosed data collection** → declare Managed-mode content transmission.
4. **Privacy policy inadequate** → it must specifically name what the *extension* collects, not be a generic website policy. See doc 10 §4.
5. **Obfuscated/minified-beyond-recognition code** → minification is allowed, obfuscation is not. Ship **source maps** in the uploaded zip; it visibly speeds review.
6. **Misleading metadata** → no keyword stuffing, no competitor names, no "#1" claims.
7. **Trademark** → the disclaimer and descriptive-use rules in §2.

### If rejected
The email cites a specific policy section. Fix exactly that, add a note in the "Reviewer notes" field describing what changed, resubmit. Do not resubmit unchanged and do not argue. Two or three rounds is normal.

### Reviewer notes field — always fill it in
```
Test account (please use, no Substack account needed for most flows):
  email: reviewer@draftsmith.io   password: <in the private field>
  This account is pre-loaded with credits.

To test the core flow:
  1. Sign in with the account above.
  2. Open https://substack.com/home — the side panel opens from the toolbar icon.
  3. Enter any topic and click "Run agent".
  4. The generated draft appears in the panel; "Push to editor" inserts it into an
     open Substack editor.

Notes on permissions:
  - AI provider host permissions are OPTIONAL and are only requested if a user
    chooses to connect their own API key. The default flow never requests them.
  - We do not collect Substack credentials or session cookies at any point.
  - No remote code is loaded. Source maps are included in this package.

Demo video: https://draftsmith.io/review-demo (2 min, unlisted)
```

---

## 6. Packaging & CI publishing

### Build
```bash
pnpm --filter extension build      # vite build → apps/extension/dist
pnpm --filter extension package    # zips dist → dist-zip/draftsmith-<version>.zip
```

Zip requirements: `manifest.json` at the **root** of the zip (not inside a folder), no `node_modules`, no `.env`, source maps included, under 10 MB.

### Automated publish
```yaml
# .github/workflows/publish.yml
on:
  push:
    tags: ['v*']
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm test && pnpm build && pnpm --filter extension package
      - name: Upload & publish
        uses: mnao305/chrome-extension-upload@v5
        with:
          file-path: apps/extension/dist-zip/draftsmith-${{ github.ref_name }}.zip
          extension-id: ${{ secrets.CHROME_EXTENSION_ID }}
          client-id: ${{ secrets.CHROME_CLIENT_ID }}
          client-secret: ${{ secrets.CHROME_CLIENT_SECRET }}
          refresh-token: ${{ secrets.CHROME_REFRESH_TOKEN }}
          publish: true
```

Getting the API credentials: Google Cloud Console → enable the **Chrome Web Store API** → create an OAuth client (Desktop) → run the one-time consent flow to obtain a refresh token. Document the exact steps in `docs/_runbooks/cws-credentials.md` during Session 18 — this is fiddly and you will forget it.

### Release process
1. `pnpm changeset` → version bump + changelog
2. Merge to `main`
3. `git tag v1.0.1 && git push --tags`
4. CI builds, tests, uploads, publishes
5. **Staged rollout: 10% → 50% → 100% over 72 hours** (dashboard setting). Watch Sentry between stages.

---

## 7. Post-launch

**Watch daily for week 1:** Sentry error rate, drift events, install→signup conversion, run success rate, credit-spend anomalies.

**Reviews.** Respond to every one, publicly, within 24 hours — Google surfaces developer responses and it materially affects install rate. For 1-star reviews caused by Substack drift: fix, ship, then reply with "fixed in v1.0.x, sorry about that." Many users update their rating.

**Updates.** Ship a small improvement every 2 weeks for the first three months. Update recency is an input to store ranking, and it signals the extension is alive.

**Never** buy installs or reviews. It is detectable, and removal is permanent.
