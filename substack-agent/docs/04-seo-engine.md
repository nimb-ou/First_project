# 04 — SEO Engine

## 1. What SEO actually means on Substack

Be honest with users about this, because competitors overpromise and it is a trust advantage.

**What you control on a Substack post:**

| Element | Control | SEO weight |
|---|---|---|
| Post title | Full | Becomes `<title>` and `<h1>` — highest |
| Subtitle | Full | Becomes the **meta description** and `og:description` — high for CTR |
| Slug (URL) | Editable before publish, then permanent | Medium |
| Body headings (H2/H3) | Full | Medium — structure and featured-snippet eligibility |
| Body copy | Full | High |
| Image alt text | Full | Low-medium; also accessibility |
| Internal links to your own archive | Full | Medium — this is underused and cheap |
| External outbound links | Full | Low |
| Social preview image | Full | CTR on shares, not ranking |
| Publication description / about | Full | Site-level relevance |
| Custom domain | Paid Substack feature | Meaningful — consolidates authority |

**What you do NOT control:**
- Canonical tags, robots directives, sitemap structure, `hreflang`
- Schema.org / structured data beyond what Substack emits
- Page speed, Core Web Vitals, JS payload
- URL structure beyond the slug
- Whether `*.substack.com` subdomain authority accrues to you (it largely does not — this is the single strongest argument for a custom domain)

**So the engine's honest positioning:** *"We optimise every lever Substack gives you, and we tell you when the remaining bottleneck is Substack itself."* The scorecard explicitly has a "Not controllable on Substack" section so users understand where the ceiling is.

---

## 2. Data sources

| Need | Vendor | Cost | Notes |
|---|---|---|---|
| Keyword volume, difficulty, CPC | **DataForSEO** — Labs `keyword_ideas` + `keyword_overview` | ~$0.01–0.05 per call | Best price/quality. Pay-as-you-go, no minimum. Primary. |
| Live SERP (top 10 + PAA + related) | **Serper.dev** or DataForSEO SERP | ~$0.001–0.003/query | Serper is cheaper and faster for plain Google SERP |
| Query expansion (free) | **Google Suggest** `https://suggestqueries.google.com/complete/search?client=firefox&q=` | Free | Rate-limit hard (max 10/run, 1 rps). Nice-to-have, not load-bearing. |
| Author's own archive | Substack internal API | Free | Internal-link graph, cannibalisation check |

**All vendor calls go through our backend.** The extension never holds a DataForSEO key and never calls Google directly. Three reasons: key security, response caching (a keyword's data is shared across all users — cache 7 days, which cuts vendor cost by ~80%), and it keeps the extension's `host_permissions` minimal for store review.

**Caching:** Cloudflare KV, keyed `seo:kw:{locale}:{sha256(keyword)}`, TTL 7 days. SERP results TTL 24 hours. Budget guard: hard monthly spend cap per vendor with a circuit breaker that degrades to Suggest-only.

---

## 3. Keyword pipeline

```
topic string
   │
   ├─► 1. SEED EXPANSION
   │      LLM (haiku) generates 12 seed queries a real person would type,
   │      across intents: informational / comparison / how-to / problem-aware.
   │      + Google Suggest expansion of the top 3 seeds.
   │
   ├─► 2. ENRICH
   │      DataForSEO keyword_ideas on the seeds → up to 200 candidates with
   │      volume, KD, CPC, trend. Filtered to the user's target locale.
   │
   ├─► 3. FILTER
   │      Drop: volume < 30, KD > (user's authority ceiling), obvious
   │      transactional/branded intent, near-duplicates (>0.9 cosine on
   │      normalised tokens).
   │
   ├─► 4. SCORE  ← deterministic, in code
   │      opportunity = log10(volume+1) × intentFit × (1 - KD/100)^1.5 × freshness
   │
   ├─► 5. CLUSTER
   │      Group by SERP overlap: two keywords are one cluster if their top-10
   │      results share ≥4 URLs. This is the only reliable clustering signal.
   │      One cluster = one post. Prevents cannibalisation.
   │
   ├─► 6. SERP ANALYSIS  (top cluster only — expensive)
   │      Fetch top 10 for the head term. Extract: title patterns, content
   │      type (listicle/guide/opinion/news), word counts, PAA questions,
   │      publication dates, domain authority proxy.
   │
   └─► 7. GAP FINDING
          LLM (haiku) reads the SERP summary + the author's voice profile and
          answers: what can THIS author offer that none of these 10 can?
          Output feeds the angle-selection step.
```

### Authority ceiling

A brand-new Substack cannot rank for KD 70 terms. Estimate the user's realistic ceiling once, at onboarding:

```ts
function authorityCeiling(pub: PublicationStats): number {
  // Rough, honest heuristic. Refined later with real GSC data if the user connects it.
  const ageMonths   = monthsSince(pub.firstPostAt)
  const postCount   = pub.publishedPostCount
  const customDomain= pub.hasCustomDomain ? 12 : 0
  const base = Math.min(35, 8 + Math.log2(postCount + 1) * 4 + Math.min(ageMonths, 36) * 0.4)
  return Math.round(base + customDomain)   // typically 15-50
}
```

Show it plainly: *"Your realistic difficulty ceiling right now is ~28. We're filtering to keywords you can actually win, not the ones with the biggest numbers."* This one behaviour will differentiate you from every generic AI SEO tool.

---

## 4. Scorecard

0–100, computed **deterministically in code** — never ask an LLM for a number. The LLM writes the *fixes*, the code computes the *score*. This makes the score stable and explainable.

```ts
// packages/shared/src/seo/score.ts
export const RUBRIC = [
  // --- Title (25) ---
  { id: 'title.length',      max: 6,  test: t => t.length >= 30 && t.length <= 60,
    fix: 'Aim for 30–60 characters so Google does not truncate it.' },
  { id: 'title.keyword',     max: 8,  test: (t, kw) => includesLoose(t, kw),
    fix: 'Include your target keyword, ideally in the first half.' },
  { id: 'title.frontLoaded', max: 5,  test: (t, kw) => positionOf(t, kw) <= t.length * 0.5 },
  { id: 'title.curiosity',   max: 6,  test: t => hasNumber(t) || hasPowerWord(t) || isQuestion(t) },

  // --- Subtitle = meta description (20) ---
  { id: 'sub.present',       max: 5,  test: s => s.trim().length > 0 },
  { id: 'sub.length',        max: 8,  test: s => s.length >= 70 && s.length <= 155,
    fix: 'Substack uses your subtitle as the meta description. 70–155 characters.' },
  { id: 'sub.keyword',       max: 4,  test: (s, kw) => includesLoose(s, kw) },
  { id: 'sub.notDuplicate',  max: 3,  test: (s, t) => similarity(s, t) < 0.7 },

  // --- Structure (20) ---
  { id: 'struct.h2Count',    max: 6,  test: d => countH2(d) >= 3 },
  { id: 'struct.h2Keyword',  max: 4,  test: (d, kw) => headings(d).some(h => includesLoose(h, kw)) },
  { id: 'struct.noH1',       max: 3,  test: d => countH1(d) === 0,
    fix: 'Substack renders the post title as the H1. Do not add another.' },
  { id: 'struct.paraLength', max: 4,  test: d => avgParagraphWords(d) <= 90 },
  { id: 'struct.scannable',  max: 3,  test: d => hasList(d) || hasBlockquote(d) },

  // --- Content (20) ---
  { id: 'content.length',    max: 6,  test: (d, _, serp) => words(d) >= serp.medianWords * 0.8 },
  { id: 'content.density',   max: 5,  test: (d, kw) => inRange(kwDensity(d, kw), 0.004, 0.018),
    fix: 'Keyword appears too often or too rarely — target roughly 0.5–1.5%.' },
  { id: 'content.semantic',  max: 5,  test: (d, _, serp) => coverage(d, serp.entities) >= 0.6,
    fix: 'Top-ranking pages all cover these related concepts and yours does not: {missing}' },
  { id: 'content.paaAnswered', max: 4, test: (d, _, serp) => answeredPAA(d, serp.paa) >= 2,
    fix: 'Answering "People also ask" questions directly wins featured snippets.' },

  // --- Links & media (15) ---
  { id: 'links.internal',    max: 6,  test: d => internalLinks(d) >= 2 && internalLinks(d) <= 6,
    fix: 'Link 2–6 of your own past posts. Substack authority is built by internal linking.' },
  { id: 'links.external',    max: 3,  test: d => externalLinks(d) >= 1 },
  { id: 'links.anchorText',  max: 3,  test: d => links(d).every(l => !GENERIC.has(l.text.toLowerCase())),
    fix: 'Replace "click here" / "read more" with descriptive anchor text.' },
  { id: 'media.alt',         max: 3,  test: d => images(d).every(i => i.alt?.length > 8) },
]
```

Bands: **0–49 Needs work** (red) · **50–74 Decent** (amber) · **75–89 Strong** (green) · **90–100 Excellent**.

Each failed rule renders as a card with the rule name, the fix, and — where applicable — a **one-click "Apply fix"** that runs a tiny targeted LLM call and patches just that element. This is the feature users will screenshot.

---

## 5. SEO optimisation prompt — `seo.optimise.v1`

```
You are optimising a finished newsletter post for search, without damaging it as a
piece of writing. The writing wins every tradeoff.

You will be given the draft, the target keyword cluster, what is currently ranking,
and a list of failed rubric rules. Fix ONLY the failed rules.

Deliverables:

1. TITLES — 5 candidates. Each must be a title this specific writer would actually
   publish (see voice profile), 30-60 chars, target keyword in the first half of at
   least 3 of them. Vary the mechanism: number, question, contrarian claim, outcome
   promise, curiosity gap. For each, state in 6 words why it might beat the current
   top result.

2. SUBTITLE — 3 candidates, 70-155 chars. This becomes the meta description, so it
   must (a) make sense as a subtitle above the post and (b) work as a search snippet
   that earns the click. It must not restate the title.

3. SLUG — lowercase, hyphenated, 3-6 words, keyword-leading, no stop words.

4. HEADING REWRITES — only for headings that fail a rule. Keep the author's heading
   style ({{subheadingStyle}}). Do not keyword-stuff; at most one heading carries the
   target keyword.

5. INTERNAL LINKS — from the author's archive below, propose 2-4 links. For each give:
   the exact sentence in the draft to attach it to, the anchor text (descriptive, 2-5
   words, not the post title verbatim), and the slug. Only propose links that a reader
   would genuinely want to follow. A forced internal link is worse than none.

6. SEMANTIC GAPS — concepts the top-10 pages cover that this draft does not. For each,
   say where it would go and in one sentence what to say. Do NOT write the copy;
   flag it for the author. Never invent facts to fill a gap.

7. ALT TEXT — for each image, descriptive alt text under 125 characters. Describe the
   image, do not stuff keywords.

Hard rules:
- Never rewrite body prose. You are editing metadata and structure only.
- Never add a keyword to a sentence where it does not belong.
- If a rubric rule cannot be fixed without hurting the writing, say so explicitly in
  `declined` with the reason. Declining is correct behaviour.

Voice profile: {{voiceProfile}}
Draft: {{draft}}
Keyword cluster: {{cluster}}
SERP summary: {{serpSummary}}
Author archive index: {{archiveIndex}}
Failed rules: {{failedRules}}

Return via `emit_seo_optimisation`.
```

---

## 6. Cannibalisation check

Cheap, high-value, almost nobody does it. Before drafting:

1. Load the author's archive index (title + slug + subtitle + first 200 words), cached.
2. Embed each post once (`text-embedding-3-small` via backend, or a local MiniLM in a WASM worker for BYOK privacy) and store vectors in `chrome.storage.local`.
3. Cosine-compare the proposed target keyword + angle against existing posts.
4. If similarity > 0.82, warn: *"You already wrote about this in **{title}** ({date}). Publishing a near-duplicate splits your ranking. Options: update that post instead / write a distinct angle / make this part 2 and link them."*

---

## 7. Repurposing (v1 feature, cheap to build)

After delivery, one call each (haiku):
- **Substack Note** — 1–2 sentence teaser + link. Notes drive more subscriber growth than SEO does for most publications; do not treat this as an afterthought.
- **X thread** — 5–8 posts, hook-first, last post links.
- **LinkedIn post** — 150–250 words, no links in body (LinkedIn suppresses them), link in first comment.
- **Email subject line variants** — 3, optimised for open rate, distinct from the SEO title. Substack lets you set the email subject separately from the post title; most authors do not know this.

That last point is worth surfacing as a tip in the UI — SEO title and email subject line have opposite optimisation targets, and separating them is free upside.
