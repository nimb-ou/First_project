# 01 — Architecture

## 1. Manifest V3

`apps/extension/manifest.config.ts` (CRXJS builds `manifest.json` from this):

```ts
import { defineManifest } from '@crxjs/vite-plugin'
import pkg from './package.json'

export default defineManifest({
  manifest_version: 3,
  name: 'Draftsmith — AI Writing & SEO for Substack',
  short_name: 'Draftsmith',
  version: pkg.version,
  description:
    'An AI writing agent for Substack: research, draft in your voice, optimise for SEO, and push straight into your editor.',
  minimum_chrome_version: '116',

  icons: { 16: 'icons/16.png', 32: 'icons/32.png', 48: 'icons/48.png', 128: 'icons/128.png' },

  action: { default_title: 'Draftsmith' },          // no popup — click opens the side panel
  side_panel: { default_path: 'sidepanel.html' },

  background: { service_worker: 'src/background/index.ts', type: 'module' },

  content_scripts: [
    {
      matches: ['https://substack.com/*', 'https://*.substack.com/*'],
      js: ['src/content/index.ts'],
      run_at: 'document_idle',
      all_frames: false,
    },
  ],

  web_accessible_resources: [
    {
      resources: ['src/injected/prosemirror-bridge.js'],
      matches: ['https://substack.com/*', 'https://*.substack.com/*'],
    },
  ],

  permissions: [
    'storage',        // settings, voice profiles, cached runs
    'sidePanel',      // the main UI surface
    'scripting',      // inject the MAIN-world bridge
    'tabs',           // find/focus the user's Substack editor tab
    'identity',       // OAuth sign-in via launchWebAuthFlow
    'alarms',         // scheduled runs, token refresh
  ],

  host_permissions: [
    'https://substack.com/*',
    'https://*.substack.com/*',
    'https://api.draftsmith.io/*',
  ],

  optional_host_permissions: [
    'https://api.anthropic.com/*',
    'https://api.openai.com/*',
    'https://generativelanguage.googleapis.com/*',
    'https://openrouter.ai/*',
  ],

  content_security_policy: {
    extension_pages:
      "script-src 'self'; object-src 'self'; connect-src 'self' https://api.draftsmith.io https://*.supabase.co https://api.anthropic.com https://api.openai.com https://generativelanguage.googleapis.com https://openrouter.ai;",
  },
})
```

### Permission discipline — this decides whether you pass review

Every permission above must be individually justifiable in the store listing. Rules:

- **No `<all_urls>`. No `webRequest`. No `cookies`.** Any of these triggers deep manual review and, for `cookies` on a third-party site, near-certain rejection.
- **BYOK provider hosts are `optional_host_permissions`.** Requested at runtime with `chrome.permissions.request()` only when the user actually pastes a key for that provider. A user in Managed mode never grants them. This is the single biggest review-risk reduction in the whole design.
- **`tabs` not `activeTab`** — we need to locate a Substack editor tab that may not be the active one. Justify precisely: *"to find and focus the user's open Substack editor tab so the generated draft can be inserted."*
- **No `host_permissions` for Google/Bing.** All SEO data comes through our own API. The extension never talks to a search engine directly.

---

## 2. Component map

```
┌──────────────────────────────────────────────────────────────────────┐
│                          SIDE PANEL (React)                          │
│  Run composer · streaming run log · diff review · settings · billing │
└───────────────────────────┬──────────────────────────────────────────┘
                            │ chrome.runtime port "sidepanel"
                            ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    SERVICE WORKER (orchestrator)                     │
│                                                                      │
│  MessageRouter ── AgentRunner ── ToolRegistry ── LLMClient           │
│        │              │              │              │                │
│        │              │              │              ├── BYOK direct  │
│        │              │              │              └── Managed proxy│
│        │              │              └── substack.* / seo.* / write.*│
│        │              └── run state persisted to chrome.storage      │
│        └── AuthManager (Supabase session, refresh via alarms)        │
└──────────┬─────────────────────────────────┬─────────────────────────┘
           │ tabs.sendMessage                │ fetch
           ▼                                 ▼
┌────────────────────────────┐   ┌──────────────────────────────────────┐
│  CONTENT SCRIPT (ISOLATED) │   │   api.draftsmith.io (CF Worker)      │
│  substack.com/*            │   │   /v1/llm/*  /v1/seo/*  /v1/credits  │
│  · capability probe        │   │   /v1/stripe/webhook                 │
│  · internal-API calls      │   └──────────────────────────────────────┘
│    (same-origin, user's    │
│     own session cookies)   │
│  · window.postMessage ──┐  │
└─────────────────────────┼──┘
                          ▼
        ┌──────────────────────────────────────┐
        │  INJECTED SCRIPT (MAIN world)        │
        │  Direct ProseMirror EditorView access│
        │  dispatch(tr) · read doc · set title │
        └──────────────────────────────────────┘
```

### Why the two-script split

Substack's editor is ProseMirror. To insert rich content reliably you must dispatch a real ProseMirror `Transaction` — synthesising keyboard/paste events is flaky and loses formatting. `EditorView` lives on the page's own JS heap, which an ISOLATED-world content script cannot reach. So:

- **Injected script (MAIN world)** — same JS context as Substack's app. Finds the `EditorView`, dispatches transactions. Communicates only via `window.postMessage` with a nonce.
- **Content script (ISOLATED)** — has `chrome.*` APIs. Bridges between the service worker and the injected script. Also makes the same-origin `fetch` calls to Substack's internal API (same origin ⇒ session cookies ride along automatically, no `cookies` permission needed).

---

## 3. Message bus

One typed channel. Every message is a Zod-validated discriminated union in `packages/shared/src/messages.ts`.

```ts
// packages/shared/src/messages.ts
import { z } from 'zod'

export const Envelope = z.object({
  id: z.string().uuid(),
  ts: z.number().int(),
  source: z.enum(['sidepanel', 'background', 'content', 'injected']),
})

export const Msg = z.discriminatedUnion('type', [
  // sidepanel → background
  z.object({ ...Envelope.shape, type: z.literal('run.start'),  payload: RunRequest }),
  z.object({ ...Envelope.shape, type: z.literal('run.cancel'), payload: z.object({ runId: z.string() }) }),
  z.object({ ...Envelope.shape, type: z.literal('run.approve'),payload: z.object({ runId: z.string(), step: z.string(), edited: z.string().optional() }) }),

  // background → sidepanel (streamed over a long-lived Port)
  z.object({ ...Envelope.shape, type: z.literal('run.step'),   payload: RunStep }),
  z.object({ ...Envelope.shape, type: z.literal('run.token'),  payload: z.object({ runId: z.string(), delta: z.string() }) }),
  z.object({ ...Envelope.shape, type: z.literal('run.done'),   payload: RunResult }),
  z.object({ ...Envelope.shape, type: z.literal('run.error'),  payload: AppError }),

  // background → content
  z.object({ ...Envelope.shape, type: z.literal('sb.probe') }),
  z.object({ ...Envelope.shape, type: z.literal('sb.readDraft') }),
  z.object({ ...Envelope.shape, type: z.literal('sb.writeDraft'), payload: DraftPatch }),
  z.object({ ...Envelope.shape, type: z.literal('sb.listArchive'), payload: z.object({ limit: z.number().max(50) }) }),
  z.object({ ...Envelope.shape, type: z.literal('sb.createDraft'), payload: NewDraft }),
])
export type Msg = z.infer<typeof Msg>
```

Rules:
- **Nothing is trusted across a boundary.** Parse with Zod on receipt at every hop, including messages from the injected script — that script shares a world with page JS, so treat its output as untrusted input.
- **Streaming uses `chrome.runtime.connect` Ports**, not `sendMessage`. `sendMessage` round-trips are unsuitable for token streams.
- **`window.postMessage` carries a per-session nonce** generated by the content script and handed to the injected script at injection time. Reject any message without it — otherwise any page script can drive your bridge.

---

## 4. Service-worker lifetime — the constraint that shapes everything

MV3 service workers are killed after ~30 seconds idle. An agent run takes 60–180 seconds. Design:

1. **All run state lives in `chrome.storage.local`, never in worker memory.** A `RunState` record is written after every step transition.
2. **An open Port keeps the worker alive** while the side panel is open. Chrome resets the idle timer on port activity, so an active token stream sustains the worker.
3. **If the side panel is closed mid-run**, the worker may die. On revival (`chrome.runtime.onStartup` / next message), `AgentRunner.resume(runId)` reads `RunState` and continues from the last completed step. Every step is therefore **idempotent and checkpointed**.
4. **`chrome.alarms` (min 30s) as a heartbeat** for long backend waits — never `setTimeout` for anything over 20 seconds.
5. **Cancellation** flips `RunState.status = 'cancelling'`; the runner checks it between steps and aborts in-flight `fetch` via `AbortController`.

```ts
// apps/extension/src/background/run-state.ts
export type RunState = {
  runId: string
  status: 'planning' | 'running' | 'awaiting_approval' | 'cancelling' | 'done' | 'error'
  request: RunRequest
  completedSteps: RunStep[]        // checkpointed outputs, replayable
  cursor: number                   // index of next step to execute
  artifacts: { outline?: Outline; sections?: Section[]; seo?: SeoReport; draft?: DraftPatch }
  usage: { inputTokens: number; outputTokens: number; creditsReserved: number }
  createdAt: number
  updatedAt: number
}
```

Storage budget: `chrome.storage.local` gives ~10 MB (unlimited with the `unlimitedStorage` permission — **do not request it**, it looks greedy in review). Prune runs older than 30 days on startup; cap stored runs at 50.

---

## 5. Security model

| Asset | Where it lives | Protection |
|---|---|---|
| BYOK API key | `chrome.storage.local` **only** — never `chrome.storage.sync` | AES-GCM encrypted with a non-extractable key from `crypto.subtle`, wrapped per-install. Never logged, never sent to our backend, redacted in Sentry. |
| Supabase session | `chrome.storage.local` | Short-lived access token + refresh; refreshed on alarm |
| Substack session | Substack's own cookies, untouched | We never read, store, or transmit them |
| Post content | Memory + `chrome.storage.local` during a run | In Managed mode the post text necessarily transits our proxy — disclosed in the privacy policy. In BYOK mode it never touches our servers. |
| Voice profile | `chrome.storage.local`, optional encrypted sync to backend | Opt-in |

Hard rules for implementation:
- **No `eval`, no `new Function`, no remotely-fetched JS.** MV3 forbids it and the store enforces it. LLM output is rendered as *text/markdown*, never executed. If the agent ever produces code, it is displayed, never run.
- **Sanitise all model output before it reaches the DOM.** Markdown → ProseMirror nodes via an allowlist schema. No `innerHTML` anywhere in the codebase — add an ESLint rule to enforce it.
- **Redact before telemetry.** Sentry `beforeSend` strips `apiKey`, `token`, `content`, `draft`, and anything matching `/sk-[A-Za-z0-9-_]{20,}/`.

---

## 6. Error taxonomy

`packages/shared/src/errors.ts` — one shape, so the UI can always say something useful:

```ts
export type AppErrorCode =
  | 'SUBSTACK_ADAPTER_STALE'   // probe failed; internal API changed
  | 'SUBSTACK_NO_EDITOR_TAB'   // no editor open
  | 'SUBSTACK_WRITE_FAILED'    // bridge could not apply the patch
  | 'LLM_KEY_INVALID'
  | 'LLM_RATE_LIMITED'
  | 'LLM_CONTEXT_TOO_LARGE'
  | 'CREDITS_EXHAUSTED'
  | 'NETWORK'
  | 'AUTH_EXPIRED'
  | 'INTERNAL'

export type AppError = {
  code: AppErrorCode
  message: string        // user-facing, plain English, no stack traces
  retryable: boolean
  hint?: string          // "Top up credits" / "Open your Substack editor"
  runId?: string
}
```

`SUBSTACK_ADAPTER_STALE` is the one that matters most. When it fires: the extension degrades to **clipboard mode** (copy the formatted post, user pastes), tells the user plainly what happened, and reports an anonymised drift event to `/v1/telemetry/drift` so we learn about breakage within minutes of it starting.
