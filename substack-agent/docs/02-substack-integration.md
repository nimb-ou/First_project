# 02 — Substack Integration

> This is the highest-risk part of the product. Substack publishes no write API and offers no stability guarantee. Everything here must be built to break gracefully.

## 1. Three ways in, ranked

| Path | Reliability | Fidelity | Use |
|---|---|---|---|
| **A. Internal JSON API** via same-origin fetch from the content script | Medium — undocumented, changes without notice | High | Primary: create draft, save, list archive, publish |
| **B. ProseMirror bridge** (MAIN-world injected script) | Medium-high — ProseMirror API is stable, Substack's mount point is not | High | Primary for editing an *already-open* draft; fallback for A |
| **C. Clipboard handoff** — formatted HTML/markdown to clipboard, user pastes | Very high | Medium (some formatting loss) | Last-resort fallback; always available |

**Every write operation implements A → B → C in that order.** The user should never see a dead end.

---

## 2. Capability probe

Runs once per Substack tab on content-script load, and again before any run. Result is cached for 6 hours in `chrome.storage.local`.

```ts
// apps/extension/src/substack/probe.ts
export type Capabilities = {
  adapterVersion: string        // e.g. '2026.07'
  userId: number | null
  publicationId: number | null
  publicationHost: string | null
  canReadArchive: boolean
  canCreateDraft: boolean
  canEditInPlace: boolean       // ProseMirror EditorView found
  canPublish: boolean
  probedAt: number
  degradations: string[]        // human-readable reasons
}

export async function probe(): Promise<Capabilities> {
  const caps = emptyCaps()

  // 1. Identity — this endpoint has been stable the longest; treat as canary.
  const me = await safeJson('/api/v1/user_profile/self')          // or /api/v1/subscriptions
  caps.userId = me?.id ?? null

  // 2. Which publication is the user an author on?
  const pubs = await safeJson('/api/v1/publications')
  const owned = pubs?.filter?.((p: any) => p.role === 'admin' || p.role === 'contributor')
  caps.publicationId = owned?.[0]?.id ?? null
  caps.publicationHost = owned?.[0]?.subdomain ?? null

  // 3. Archive read — cheap, read-only, no side effects.
  caps.canReadArchive = !!(await safeJson('/api/v1/archive?sort=new&limit=1'))

  // 4. Draft create — DRY RUN ONLY. Never create a real draft during a probe.
  //    We validate shape by checking the OPTIONS/HEAD response and the presence
  //    of the editor route, not by writing.
  caps.canCreateDraft = caps.publicationId !== null && caps.canReadArchive

  // 5. In-place edit — ask the injected bridge whether it found an EditorView.
  caps.canEditInPlace = await askBridge('probe').then(r => r.hasEditorView).catch(() => false)

  caps.canPublish = caps.canCreateDraft
  caps.probedAt = Date.now()
  return caps
}
```

**Rule: the probe never mutates anything.** No test drafts, no throwaway posts. Users will notice and it is the fastest route to a one-star review.

When the probe degrades, the side panel shows a specific, honest banner:
> *"Substack changed something on their end. Draftsmith can still write your post, but it can't insert it automatically right now — use **Copy to clipboard** and paste into your editor. We've been notified and usually ship a fix within a day."*

---

## 3. Internal API map (as observed — verify in Session 3)

⚠️ **These are undocumented and unversioned. Session 3's job is to re-verify every one of them against live Substack and record the actual shapes in `e2e/fixtures/substack/`. Do not trust this table without verification.**

All calls are **same-origin** from the content script, so the user's session cookie is sent automatically. Include `credentials: 'include'` and Substack's CSRF token where present.

| Purpose | Method + path | Notes |
|---|---|---|
| Current user | `GET /api/v1/user_profile/self` | Canary for auth state |
| Publications | `GET /api/v1/publications` | Role tells us author vs reader |
| Archive list | `GET /api/v1/archive?sort=new&limit=N&offset=M` | Public posts; used for voice profiling |
| Post detail | `GET /api/v1/posts/{slug}` | Full body HTML |
| List drafts | `GET /api/v1/drafts?limit=N` | Requires author role |
| Create draft | `POST /api/v1/drafts` | Body: `{ draft_title, draft_subtitle, draft_body, type: 'newsletter', audience }` |
| Update draft | `PUT /api/v1/drafts/{id}` | Same shape; send full body |
| Prepublish check | `POST /api/v1/drafts/{id}/prepublish` | Returns validation warnings |
| Publish | `POST /api/v1/drafts/{id}/publish` | Body includes `send_email`, `share_automatically` |
| Schedule | `PUT /api/v1/drafts/{id}` with `post_date` in the future | Behaviour varies |

### `draft_body` format

Substack stores the body as a **JSON-stringified ProseMirror document**, not HTML:

```json
{
  "type": "doc",
  "content": [
    { "type": "heading", "attrs": { "level": 2 }, "content": [{ "type": "text", "text": "The setup" }] },
    { "type": "paragraph", "content": [{ "type": "text", "text": "Plain text here, and " },
      { "type": "text", "marks": [{ "type": "strong" }], "text": "bold here" }] }
  ]
}
```

So we need a **Markdown → Substack ProseMirror JSON** compiler. This is the single most important piece of Substack-specific code in the repo.

```ts
// apps/extension/src/substack/md-to-pm.ts
// Supported node types (allowlist — anything else degrades to paragraph):
//   doc, paragraph, heading(1-6), blockquote, bulletList, orderedList, listItem,
//   codeBlock, horizontalRule, image, captionedImage, button, footnoteAnchor
// Supported marks:
//   strong, em, code, link(href, target), strike
//
// Build on `unified` + `remark-parse` → mdast → PM JSON. ~250 lines.
// MUST round-trip: pmToMd(mdToPm(x)) === x for the supported subset.
// Fuzz-test this with property-based tests (fast-check) in Session 4.
```

---

## 4. ProseMirror bridge (MAIN world)

```js
// apps/extension/src/injected/prosemirror-bridge.js
// Injected into the MAIN world. Has NO chrome.* access. Speaks only postMessage.
(function () {
  const NONCE = document.currentScript?.dataset.nonce
  if (!NONCE) return

  // --- Finding the EditorView ---------------------------------------------
  // Substack does not expose it globally. Strategy, in order:
  //   1. Walk the DOM for [contenteditable] inside the editor container.
  //   2. Read the ProseMirror instance off the DOM node: node.pmViewDesc.
  //      Every ProseMirror-managed node carries `pmViewDesc`; walk up to the
  //      root desc and read `.view` — this is stable across PM versions.
  //   3. Fall back to React fibre traversal (__reactFiber$*) looking for a
  //      prop or state field that is an EditorView instance.
  function findView() {
    const el = document.querySelector('.ProseMirror[contenteditable="true"]')
    if (!el) return null
    let desc = el.pmViewDesc
    while (desc && !desc.view) desc = desc.parent
    if (desc?.view) return desc.view
    return findViewViaFibre(el)
  }

  // --- Operations ----------------------------------------------------------
  const ops = {
    probe: () => ({ hasEditorView: !!findView() }),

    readDoc: () => {
      const v = findView(); if (!v) throw new Error('no-view')
      return { doc: v.state.doc.toJSON(), text: v.state.doc.textBetween(0, v.state.doc.content.size, '\n') }
    },

    // Replace the whole document. Used for "push draft to editor".
    replaceDoc: ({ pmJson }) => {
      const v = findView(); if (!v) throw new Error('no-view')
      const { state } = v
      const node = state.schema.nodeFromJSON(pmJson)          // throws on invalid schema — good
      const tr = state.tr.replaceWith(0, state.doc.content.size, node.content)
      tr.setMeta('addToHistory', true)                        // user can Cmd-Z our insert
      v.dispatch(tr)
      return { ok: true }
    },

    // Insert at cursor. Used for co-write mode.
    insertAtCursor: ({ pmJson }) => { /* same, but tr.replaceSelectionWith */ },

    setTitle: ({ title, subtitle }) => {
      // Title/subtitle are separate contenteditable/textarea nodes outside the PM doc.
      // Set via native setter + input event so React's onChange fires.
      setReactValue(document.querySelector('[data-testid="post-title"], textarea[placeholder*="Title"]'), title)
      setReactValue(document.querySelector('[data-testid="post-subtitle"], textarea[placeholder*="subtitle" i]'), subtitle)
      return { ok: true }
    },
  }

  window.addEventListener('message', async (e) => {
    if (e.source !== window || e.data?.__ds !== NONCE) return
    const { id, op, args } = e.data
    try {
      window.postMessage({ __ds: NONCE, id, ok: true, result: await ops[op](args ?? {}) }, '*')
    } catch (err) {
      window.postMessage({ __ds: NONCE, id, ok: false, error: String(err?.message ?? err) }, '*')
    }
  })

  window.postMessage({ __ds: NONCE, ready: true }, '*')
})()
```

### `setReactValue` — the React-controlled-input trick

React inputs ignore `el.value = x`. You must use the native setter and dispatch a bubbling `input` event:

```js
function setReactValue(el, value) {
  if (!el) return
  const proto = el instanceof HTMLTextAreaElement
    ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype
  Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, value)
  el.dispatchEvent(new Event('input', { bubbles: true }))
}
```

For `contenteditable` title fields, use `document.execCommand('insertText')` after selecting all, which fires the right events in ProseMirror-adjacent editors.

---

## 5. Adapter pattern

```
apps/extension/src/substack/
├─ types.ts            # SubstackAdapter interface — the contract
├─ probe.ts
├─ md-to-pm.ts         # markdown ⇄ Substack ProseMirror JSON
├─ adapters/
│  ├─ v2026-07.ts      # current
│  ├─ v2026-03.ts      # kept for users on older Substack rollouts
│  └─ dom-fallback.ts  # pure bridge, no internal API
└─ registry.ts         # picks an adapter from probe results
```

```ts
export interface SubstackAdapter {
  readonly version: string
  supports(caps: Capabilities): boolean
  listArchive(limit: number): Promise<ArchivePost[]>
  getPost(slug: string): Promise<FullPost>
  createDraft(input: NewDraft): Promise<{ id: number; url: string }>
  updateDraft(id: number, patch: DraftPatch): Promise<void>
  pushToOpenEditor(patch: DraftPatch): Promise<void>   // uses the bridge
  publish(id: number, opts: PublishOptions): Promise<{ url: string }>
}
```

`registry.ts` picks the highest-version adapter whose `supports(caps)` returns true, falling back to `dom-fallback`, then to clipboard mode. **Never hard-code an endpoint outside an adapter file.** When Substack changes, exactly one new file is written and the registry order updated — a 30-minute fix, not a rewrite.

---

## 6. Drift detection

Ship an early-warning system, because you will not find out from users fast enough.

1. **Client side:** every adapter call is wrapped. On unexpected shape (Zod parse failure) or non-2xx, POST to `/v1/telemetry/drift` with: adapter version, operation, HTTP status, Zod error path — **and nothing else**. No post content, no user identifiers beyond an anonymous install ID.
2. **Server side:** if drift events for one `(adapterVersion, operation)` pair exceed 20 in an hour, fire a Slack/email alert.
3. **Synthetic canary:** a nightly GitHub Action runs a Playwright job against a dedicated test Substack publication, exercising read + create-draft + delete-draft. Fails loudly.
4. **Remote kill switch:** `GET /v1/config` returns `{ substackWriteEnabled: boolean, minExtensionVersion: string, notice?: string }`, cached 15 min. If Substack changes something dangerous, we flip writes off for all users within 15 minutes and show `notice` in the panel.

---

## 7. Rate limiting and politeness

We are a guest on Substack's infrastructure. Non-negotiable client-side limits:

- Max **1 request per 1.5 s** per tab to any `/api/v1/*` endpoint, via a token-bucket queue in the content script.
- Archive ingestion for voice profiling: max **50 posts**, fetched at 1 rps, **cached for 30 days**, re-fetch only on explicit user action.
- Exponential backoff with jitter on 429/5xx: 2s, 4s, 8s, 16s, then give up and surface `LLM_RATE_LIMITED`-style guidance.
- **Never** poll. All Substack reads are user-initiated or run-initiated.
- Set a descriptive `X-Requested-With: Draftsmith` header so Substack can identify and contact us if we ever cause a problem. This is a good-faith signal that materially helps if a dispute arises.

---

## 8. Session 3 deliverable — recon checklist

Before writing any adapter, capture ground truth:

- [ ] Log in to a **test** Substack publication (not the user's main one).
- [ ] Open DevTools → Network, filter XHR, and perform each action manually: load archive, open editor, type, save draft, open publish modal, schedule.
- [ ] For each request, save the full request/response (headers minus cookies, body) to `e2e/fixtures/substack/<operation>.json`.
- [ ] Note the CSRF mechanism (header name, where the token comes from).
- [ ] Save one complete real `draft_body` ProseMirror JSON with headings, bold, italic, link, bullet list, blockquote, code block, and an image — this is the golden fixture for `md-to-pm.ts`.
- [ ] Record `document.querySelector` paths for title, subtitle, editor root, save button, and publish button, with at least two alternative selectors each.
- [ ] Write `docs/_recon-2026-07.md` recording what was actually observed, with dates.
