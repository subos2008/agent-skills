# Analytics (GA4 + Meta Pixel + Meta CAPI) Skill — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a reusable Claude Code skill (`analytics-ga-meta-capi`) that installs dinner-matcher's proven GA4 + Meta Pixel + Meta CAPI analytics pattern into any React/Vite + Supabase project.

**Architecture:** A single skill in `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/` consisting of a `SKILL.md` orchestration guide plus six reference files containing extracted code templates and instructions. The skill is a companion to `stripe-checkout-supabase` — it detects whether Stripe edge functions exist and patches them with CAPI wiring via anchored insertions.

**Tech Stack:** Markdown-only skill (no code execution). Templates target React 19 + TypeScript + Vite + Tailwind + Supabase (Deno edge functions) + OpenTelemetry / Honeycomb.

**Spec:** `docs/superpowers/specs/2026-04-11-analytics-ga-meta-capi-skill-design.md`

---

## Reality-vs-spec reconciliation (read before starting)

The design spec was written at a sketch level. While researching for this plan I read dinner-matcher's actual source files and found three naming discrepancies. **Use the real names throughout** — they are the source of truth.

| Spec name (incorrect) | Real name (use this) | Location |
|---|---|---|
| `sendCapiEvent()` | `sendMetaCAPIEvent()` | `supabase/functions/_shared/meta-capi.ts` |
| `META_CAPI_ACCESS_TOKEN` | `META_CAPI_TOKEN` | Deno env var read inside `sendMetaCAPIEvent` |
| (unspecified) | `META_PIXEL_ID` | Also read inside `sendMetaCAPIEvent` for the Graph API URL |

Additionally, the real Meta Graph API version in dinner-matcher is `v21.0` (not `v18.0` or other).

---

## File structure overview

Files created at `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/`:

```
skills/analytics-ga-meta-capi/
├── SKILL.md                          Orchestration guide — Task 8
├── TESTING.md                        Pre-publish validation checklist — Task 9
└── references/
    ├── client-wrappers.md            analytics.ts + meta-pixel.ts templates — Task 2
    ├── init-patterns.md              static (index.html) + dynamic (initAnalytics.ts + App.tsx) — Task 3
    ├── server-capi.md                _shared/meta-capi.ts template (portable, no OTEL) — Task 4
    ├── stripe-integration.md         Anchored patches for create-checkout + stripe-webhook — Task 5
    ├── event-scan-patterns.md        Grep patterns + suggested event names for Step 6 scan — Task 6
    └── setup-checklist.md            User-facing GA/Meta/CAPI provisioning steps — Task 7
```

Also modified:

```
agent-skills/
├── README.md                         Add analytics-ga-meta-capi to skills table — Task 10
├── .claude-plugin/
│   ├── plugin.json                   Bump 0.3.1 → 0.4.0, update description — Task 11
│   └── marketplace.json              Bump 0.3.1 → 0.4.0, update description — Task 11
```

Each reference file has **one responsibility** — split by layer (client / init / server / Stripe / scan / setup) so the skill can load just the files it needs at each step. SKILL.md remains the only file loaded unconditionally at skill invocation.

---

## Scope check

The spec describes one cohesive skill. No decomposition required. The plan produces a single installable output: a new skill directory under `~/Dropbox/agent-skills/skills/`.

---

## Working directory

All paths in this plan are relative to `~/Dropbox/agent-skills` unless prefixed with `/Users/ryan/dinner-matcher/` (source of extracted templates).

---

## Task 1: Scaffold skill directory

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/`
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/`

- [ ] **Step 1: Create directories**

```bash
mkdir -p ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references
```

- [ ] **Step 2: Verify directories exist**

```bash
ls -d ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references
```

Expected output: the path printed unchanged.

- [ ] **Step 3: Do NOT commit yet**

The directory is empty — git has nothing to track. The first commit lands in Task 2 when `client-wrappers.md` is added.

---

## Task 2: Write `references/client-wrappers.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/client-wrappers.md`
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/onboarding/src/lib/analytics.ts`
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/onboarding/src/lib/meta-pixel.ts`
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/subscribers/src/lib/meta-pixel.ts` (for `trackPurchase`)

**Goal of this reference file**: provide the two client-side wrapper modules that get copied verbatim (modulo comments) into any target SPA's `src/lib/` directory. These are framework-free primitives: `analytics.ts` wraps `window.gtag`, `meta-pixel.ts` wraps `window.fbq` plus fbclid/fbc handling.

- [ ] **Step 1: Re-read the three source files to confirm the current content matches the snapshot below**

```bash
# From /Users/ryan/dinner-matcher:
cat web-apps/onboarding/src/lib/analytics.ts
cat web-apps/onboarding/src/lib/meta-pixel.ts
cat web-apps/subscribers/src/lib/meta-pixel.ts
```

If any file has changed since this plan was written, use the new content instead. The plan-time snapshots are below as a fallback.

- [ ] **Step 2: Write `references/client-wrappers.md`**

Content:

````markdown
# Client wrappers: `analytics.ts` + `meta-pixel.ts`

These two modules live at `src/lib/analytics.ts` and `src/lib/meta-pixel.ts` in the target SPA. They are thin, framework-free primitives — `analytics.ts` wraps `window.gtag`, `meta-pixel.ts` wraps `window.fbq` and adds `fbclid` capture for ad attribution.

Both are copy-paste ready. Replace **every existing** `src/lib/analytics.ts` or `src/lib/meta-pixel.ts` in the target SPA with these versions — do not merge or diff with older code, this skill is the source of truth for the pattern.

## `src/lib/analytics.ts`

```typescript
declare global {
  interface Window {
    gtag?: (...args: unknown[]) => void
  }
}

export function trackEvent(name: string, params?: Record<string, string | number>) {
  if (typeof window.gtag === 'function') {
    window.gtag('event', name, params)
  }
}
```

**Design notes:**

- Only one public function: `trackEvent(name, params)`. Domain-specific helpers (e.g. `trackStepCompleted`) are NOT included — the skill ships the primitive, and Step 6 of the installer (domain event scan) wires in application-specific events.
- The `typeof window.gtag === 'function'` guard is essential. GA4 may not be loaded yet (dynamic pattern: after Turnstile verification) or may be blocked by a privacy extension.
- `params` values are restricted to `string | number` because GA4 rejects nested objects in custom parameters.

## `src/lib/meta-pixel.ts`

```typescript
declare global {
  interface Window {
    fbq?: (...args: unknown[]) => void
  }
}

function track(event: string, params?: Record<string, string | number>, eventId?: string) {
  if (typeof window.fbq === 'function') {
    const options = eventId ? { eventID: eventId } : undefined
    if (params && options) {
      window.fbq('track', event, params, options)
    } else if (params) {
      window.fbq('track', event, params)
    } else if (options) {
      window.fbq('track', event, {}, options)
    } else {
      window.fbq('track', event)
    }
  }
}

// --- Starter event set ---

export function trackLead() {
  track('Lead')
}

export function trackInitiateCheckout(eventId?: string) {
  track('InitiateCheckout', undefined, eventId)
}

export function trackPurchase(eventId?: string) {
  track('Purchase', undefined, eventId)
}

// Generic wrapper for custom events not in the starter set
export function trackPixelEvent(name: string, params?: Record<string, string | number>, eventId?: string) {
  track(name, params, eventId)
}

// --- fbclid / fbc helpers ---

const FBCLID_KEY = 'cju_fbclid'
const FBCLID_TS_KEY = 'cju_fbclid_ts'

/**
 * Capture fbclid from URL query params (Meta appends this to ad click URLs).
 * Call once on app mount. Stores in sessionStorage for the checkout flow.
 */
export function captureFbclid() {
  const params = new URLSearchParams(window.location.search)
  const fbclid = params.get('fbclid')
  if (fbclid) {
    sessionStorage.setItem(FBCLID_KEY, fbclid)
    sessionStorage.setItem(FBCLID_TS_KEY, Date.now().toString())
  }
}

/**
 * Construct the fbc parameter from a stored fbclid.
 * Format: fb.1.{timestamp_ms}.{fbclid}
 * Returns null if no fbclid was captured.
 */
export function getFbc(): string | null {
  const fbclid = sessionStorage.getItem(FBCLID_KEY)
  const ts = sessionStorage.getItem(FBCLID_TS_KEY)
  if (!fbclid || !ts) return null
  return `fb.1.${ts}.${fbclid}`
}
```

**Design notes:**

- The four-arm conditional inside `track()` is deliberate and reflects `fbq`'s positional argument quirk: passing `undefined` for `params` changes how `fbq` interprets the fourth argument. Don't refactor to `fbq('track', event, params || {}, options)` — it breaks the dedup.
- `trackLead`, `trackInitiateCheckout`, and `trackPurchase` are the three **Meta standard events** every Stripe-SaaS install needs for ad-campaign optimization. They're pre-wired for the dedup flow: `InitiateCheckout` and `Purchase` accept an `eventId` that matches the server-side CAPI event ID.
- `trackPixelEvent` is the escape hatch for custom events. Domain events proposed by the Step 6 scan use this wrapper unless they map to a standard Meta event.
- The `cju_` sessionStorage key prefix is dinner-matcher's convention ("come join us"). Change to a target-app prefix if you want; it's isolated to this file.
- `fbc` format is `fb.{subdomain_index}.{timestamp_ms}.{fbclid}`. Subdomain index `1` means "from the root domain" which is what Meta expects for server-sent events.

## When to load which

- **Always load both files**, regardless of whether the target SPA uses the static or dynamic install pattern. The file contents are identical for both.
- Import sites:
  - `analytics.ts`: imported by any file calling `trackEvent(...)`
  - `meta-pixel.ts`: imported by the App root (for `captureFbclid`), by checkout flows (for `trackInitiateCheckout`), and by any purchase-confirmation pages (for `trackPurchase`)

## Verification after generation

```bash
# In the target SPA directory:
npx tsc --noEmit  # must pass — these files are type-complete
```

If `tsc` errors on `sessionStorage` or `crypto.subtle`, the target's `tsconfig.json` is missing `"lib": ["dom"]`. Don't modify these wrapper files — fix the tsconfig.
````

- [ ] **Step 3: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/client-wrappers.md
grep -c "captureFbclid" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/client-wrappers.md
grep -c "trackPurchase" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/client-wrappers.md
```

Expected: file is ~140 lines, `captureFbclid` appears ≥2 times, `trackPurchase` appears ≥2 times.

- [ ] **Step 4: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/client-wrappers.md && \
  git commit -m "analytics-ga-meta-capi: add client wrappers reference"
```

---

## Task 3: Write `references/init-patterns.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md`
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/homepage/index.html` (static pattern)
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/onboarding/src/lib/initAnalytics.ts` (dynamic pattern)
- Source (read-only): `/Users/ryan/dinner-matcher/web-apps/onboarding/src/App.tsx` lines 1-150 (wiring example)

**Goal of this reference file**: the two installation patterns. The skill's Step 2 picks one based on user choice. This file contains both side-by-side so a single reference loads everything needed for either path.

- [ ] **Step 1: Write `references/init-patterns.md`**

Content:

````markdown
# Installation patterns: static vs dynamic

The skill supports two installation patterns. Pick **one per target SPA**:

| Pattern | When to use | Trade-off |
|---|---|---|
| **Static** | Marketing/landing SPAs where every visit should be tracked from the first byte. No bot filtering in place. | Fires on page parse — bots inflate audiences. No consent-gate seam. |
| **Dynamic** | Authenticated SPAs with bot filtering (Turnstile captcha, login gate). | One paint frame delay before the pixel loads. Cleaner for consent gates later. |

The two patterns are **mutually exclusive** within a single SPA. Do not generate both — pick one at Step 1 of the installer.

---

## Pattern A: Static (script tags in `index.html`)

Insert these two `<script>` blocks inside the `<head>` of `index.html`, after the viewport meta tag and before any other scripts.

**Insert point:** right after `<meta name="viewport"...>`, before any `<link rel="stylesheet">` or font-preload tags. Putting analytics early in `<head>` maximises the chance the pixel fires on the first paint.

### Snippet to inject

```html
<!-- Meta Pixel -->
<script>
  !function(f,b,e,v,n,t,s)
  {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
  n.callMethod.apply(n,arguments):n.queue.push(arguments)};
  if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
  n.queue=[];t=b.createElement(e);t.async=!0;
  t.src=v;s=b.getElementsByTagName(e)[0];
  s.parentNode.insertBefore(t,s)}(window, document,'script',
  'https://connect.facebook.net/en_US/fbevents.js');
  fbq('init', '%VITE_META_PIXEL_ID%');
  fbq('track', 'PageView');
</script>
<!-- Google Analytics (GA4) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=%VITE_GA_MEASUREMENT_ID%"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', '%VITE_GA_MEASUREMENT_ID%');
</script>
```

### How the placeholders work

Vite replaces `%VITE_FOO%` at build time with the value of `FOO` from `.env.<mode>`. The `VITE_` prefix is Vite's safety-list — it's the only env vars that get exposed to the client bundle. `%VITE_META_PIXEL_ID%` in `index.html` → the real pixel ID in `dist/index.html`.

**Do not use `import.meta.env.VITE_META_PIXEL_ID` here** — that syntax only works inside JavaScript modules, not in `index.html`. The `%...%` form is specifically for HTML entry files.

### Fallback if env vars are missing

If `VITE_META_PIXEL_ID` or `VITE_GA_MEASUREMENT_ID` is unset when Vite builds, the placeholder stays literal in the HTML (e.g. `fbq('init', '%VITE_META_PIXEL_ID%')`) and Meta's `fbq` will log a console error but won't break the page. This is a soft failure — the setup checklist warns the user to double-check env values before deploying.

### File anchor for the inserter

When editing `index.html`, the inserter finds `<meta name="viewport"` and inserts the snippet **after** that line. If the viewport meta tag is not present, insert immediately after `<head>`. Do not insert at any other position — analytics must be early in `<head>` to fire reliably on `PageView`.

### Complete example `index.html` structure

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <!-- ^ insert analytics snippet here, immediately after viewport -->
    <title>App Title</title>
    <!-- rest of head -->
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

---

## Pattern B: Dynamic (`initAnalytics.ts` + `App.tsx` edit)

Creates one new file and edits one existing file.

### `src/lib/initAnalytics.ts` (new file)

```typescript
const GA_ID = import.meta.env.VITE_GA_MEASUREMENT_ID as string | undefined
const PIXEL_ID = import.meta.env.VITE_META_PIXEL_ID as string | undefined

let initialized = false

export function initAnalytics() {
  if (initialized) return
  initialized = true

  // GA4
  if (GA_ID) {
    const script = document.createElement('script')
    script.async = true
    script.src = `https://www.googletagmanager.com/gtag/js?id=${GA_ID}`
    document.head.appendChild(script)

    window.dataLayer = window.dataLayer || []
    window.gtag = function () {
      // eslint-disable-next-line prefer-rest-params
      window.dataLayer!.push(arguments)
    }
    window.gtag('js', new Date())
    window.gtag('config', GA_ID)
  }

  // Meta Pixel
  if (PIXEL_ID) {
    const n = (window.fbq = function () {
      // eslint-disable-next-line prefer-rest-params
      n.callMethod ? n.callMethod.apply(n, arguments) : n.queue.push(arguments)
    } as unknown as typeof window.fbq & { callMethod?: Function; queue: unknown[]; loaded: boolean; version: string; push: Function })
    if (!window._fbq) window._fbq = n as never
    ;(n as any).push = n
    ;(n as any).loaded = true
    ;(n as any).version = '2.0'
    ;(n as any).queue = []

    const script = document.createElement('script')
    script.async = true
    script.src = 'https://connect.facebook.net/en_US/fbevents.js'
    document.head.appendChild(script)

    window.fbq!('init', PIXEL_ID)
    window.fbq!('track', 'PageView')
  }
}

declare global {
  interface Window {
    dataLayer?: unknown[]
    _fbq?: unknown
  }
}
```

**Design notes:**

- `let initialized = false` + `if (initialized) return` is the idempotency guard. Multiple calls are safe — only the first runs.
- `if (GA_ID)` and `if (PIXEL_ID)` guards mean the function no-ops cleanly if env vars are missing. The Step 5 env-var edits add both keys but the target developer can leave one blank for dev.
- The `(window.fbq = function () { ... } as unknown as ...)` cast is ugly but necessary. `fbq`'s inline snippet does property assignment on itself (`n.callMethod`, `n.queue`, etc.) which TypeScript can't express via the clean `declare global` form. Leave it as-is.
- The Pixel snippet intentionally fires `fbq('track', 'PageView')` at init time. That's the base PageView event Meta uses for retargeting audiences.

### `src/App.tsx` (edit — wire the init call)

Add two imports at the top:

```typescript
import { captureFbclid } from './lib/meta-pixel'
import { initAnalytics } from './lib/initAnalytics'
```

Add this state ref near existing refs in the root component:

```typescript
const analyticsInitRef = useRef(false)
```

Add a `useEffect` that calls `captureFbclid` on mount:

```typescript
useEffect(() => { captureFbclid() }, [])
```

Add a `useEffect` that calls `initAnalytics` when the app's verification gate passes:

```typescript
// Init analytics on first Turnstile verification, or immediately if no site key (local dev)
useEffect(() => {
  if (analyticsInitRef.current) return
  if (turnstile.isVerified || !import.meta.env.VITE_TURNSTILE_SITE_KEY) {
    analyticsInitRef.current = true
    initAnalytics()
  }
}, [turnstile.isVerified])
```

### If the target SPA does not use Turnstile

Replace the verification check with an immediate fire:

```typescript
useEffect(() => {
  if (analyticsInitRef.current) return
  analyticsInitRef.current = true
  initAnalytics()
}, [])
```

This still keeps the idempotency guard while firing on mount. Use this variant when the target SPA has no bot-gate.

### If the target SPA uses a different bot gate (e.g. hCaptcha, reCAPTCHA)

Swap `turnstile.isVerified` for the equivalent state variable from the target's captcha hook. The pattern is the same: wait for `isVerified`, then fire `initAnalytics()` once.

### File anchor for the inserter

The inserter looks for existing `useEffect` calls in `App.tsx` and inserts the new `useEffect`s **after the last existing one** and **before the JSX return statement**. If no existing `useEffect`s are present, insert immediately after the last `useRef`/`useState` declaration. Never insert inside an existing hook body.

---

## Which pattern does a given SPA need?

A Step 1 heuristic the installer can use:

| Signal in target SPA | Pattern to recommend |
|---|---|
| `index.html` has existing inline `<script>` blocks | Static (pattern is already HTML-first) |
| `package.json` has `@marsidev/react-turnstile` or similar | Dynamic (wait for bot gate) |
| `src/main.tsx` imports from `react-router-dom` and has auth routes | Dynamic (dashboard-style SPA) |
| Neither | Ask the user |

The question in Step 1 is: *"This SPA has/doesn't have a bot-gate (Turnstile, etc.). Do you want analytics to fire on first byte (static) or after verification (dynamic)?"*
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md
grep -c "initAnalytics" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md
grep -c "Pattern A: Static" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md
grep -c "Pattern B: Dynamic" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md
```

Expected: file is ~170 lines, `initAnalytics` appears ≥5 times, both pattern headers appear exactly once.

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/init-patterns.md && \
  git commit -m "analytics-ga-meta-capi: add init patterns reference"
```

---

## Task 4: Write `references/server-capi.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md`
- Source (read-only): `/Users/ryan/dinner-matcher/supabase/functions/_shared/meta-capi.ts`

**Goal of this reference file**: the portable `_shared/meta-capi.ts` helper. Dinner-matcher's version imports OpenTelemetry (`@opentelemetry/api`) for tracing — the skill ships a **portable version with the OTEL import removed** so it works in projects that don't have observability set up. The reference file explains how to add tracing back.

- [ ] **Step 1: Write `references/server-capi.md`**

Content:

````markdown
# Server CAPI helper: `_shared/meta-capi.ts`

The Meta Conversions API (CAPI) helper. One file, two public functions:

- `sendMetaCAPIEvent(event)` — POST a single event to Meta's Graph API
- `buildUserData(params)` — SHA256-hash PII fields into the shape Meta expects

Plus one private helper: `sha256(value)`.

## File: `supabase/functions/_shared/meta-capi.ts`

```typescript
/**
 * Meta Conversions API (CAPI) helper for Supabase Edge Functions.
 *
 * Sends server-side events to Meta for ad attribution and optimisation.
 * Used alongside the browser pixel — deduplication via shared event_id.
 */

const META_GRAPH_API_VERSION = 'v21.0'

interface UserData {
  em?: string[]       // SHA256 hashed email
  ph?: string[]       // SHA256 hashed phone
  client_ip_address?: string  // unhashed
  client_user_agent?: string  // unhashed
  fbc?: string        // click ID (fb.1.{ts}.{fbclid})
  external_id?: string[]  // SHA256 hashed user ID
  ct?: string[]       // SHA256 hashed city
  country?: string[]  // SHA256 hashed country code
}

interface CustomData {
  currency?: string
  value?: number
}

interface CAPIEvent {
  event_name: string
  event_id: string
  event_time: number
  event_source_url: string
  action_source: 'website'
  user_data: UserData
  custom_data?: CustomData
}

/**
 * Send a single event to Meta Conversions API.
 * Logs errors but does not throw — CAPI failures should not break checkout/webhook flows.
 */
export async function sendMetaCAPIEvent(event: CAPIEvent): Promise<void> {
  try {
    const pixelId = Deno.env.get('META_PIXEL_ID')
    const accessToken = Deno.env.get('META_CAPI_TOKEN')

    if (!pixelId || !accessToken) {
      console.warn('META_PIXEL_ID or META_CAPI_TOKEN not set — skipping CAPI event')
      return
    }

    const url = `https://graph.facebook.com/${META_GRAPH_API_VERSION}/${pixelId}/events`

    const response = await fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        data: [event],
        access_token: accessToken,
      }),
    })

    if (!response.ok) {
      const body = await response.text()
      console.error(`Meta CAPI error (${response.status}):`, body)
    }
  } catch (err) {
    // Preserve non-throwing behaviour — CAPI failures must not break checkout flows
    console.error('Meta CAPI request failed:', err)
  }
}

/**
 * SHA256 hash a value for Meta CAPI user data parameters.
 * Meta requires lowercase, trimmed, SHA256-hashed values for PII fields.
 */
export async function sha256(value: string): Promise<string> {
  const data = new TextEncoder().encode(value.trim().toLowerCase())
  const hashBuffer = await crypto.subtle.digest('SHA-256', data)
  const hashArray = Array.from(new Uint8Array(hashBuffer))
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('')
}

/**
 * Build the user_data object for a CAPI event.
 * Hashes PII fields, passes IP/user agent through unhashed.
 */
export async function buildUserData(params: {
  email?: string | null
  phone?: string | null
  userId?: string | null
  city?: string | null
  country?: string | null
  clientIp?: string | null
  clientUserAgent?: string | null
  fbc?: string | null
}): Promise<UserData> {
  const userData: UserData = {}

  if (params.email) {
    userData.em = [await sha256(params.email)]
  }
  if (params.phone) {
    userData.ph = [await sha256(params.phone)]
  }
  if (params.userId) {
    userData.external_id = [await sha256(params.userId)]
  }
  if (params.city) {
    userData.ct = [await sha256(params.city)]
  }
  // Default country is 'gb'. Override via params.country if the target app serves other regions.
  userData.country = [await sha256(params.country || 'gb')]

  if (params.clientIp) {
    userData.client_ip_address = params.clientIp
  }
  if (params.clientUserAgent) {
    userData.client_user_agent = params.clientUserAgent
  }
  if (params.fbc) {
    userData.fbc = params.fbc
  }

  return userData
}
```

## Differences from dinner-matcher's version

Dinner-matcher's `meta-capi.ts` wraps the `fetch` call in an OpenTelemetry span for observability. **This reference file strips the OTEL import** so the helper is portable to projects without observability set up.

### How to add tracing back (optional)

If the target project already has `@opentelemetry/api` imported and a tracer initialised, replace the body of `sendMetaCAPIEvent` with:

```typescript
import { trace, SpanStatusCode } from 'npm:@opentelemetry/api@1'

export async function sendMetaCAPIEvent(event: CAPIEvent): Promise<void> {
  return trace.getTracer('shared').startActiveSpan('meta-capi.send-event', async (span) => {
    span.setAttribute('event_name', event.event_name)
    span.setAttribute('event_id', event.event_id)
    try {
      // ... body as shown above, plus span.setAttribute('response_status', response.status) after fetch
    } catch (err) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: err instanceof Error ? err.message : String(err) })
      span.recordException(err instanceof Error ? err : new Error(String(err)))
      console.error('Meta CAPI request failed:', err)
    } finally {
      span.end()
    }
  })
}
```

**Do not add the tracing wrapper if the target project does not already have OTEL set up.** The skill ships the portable version; tracing is a project-level choice, not an analytics concern.

## Key design decisions

**Fail-open, not fail-closed.** A Meta Graph API outage must not break the checkout flow. Every error path returns from `sendMetaCAPIEvent` without throwing. The caller (`create-checkout`, `stripe-webhook`) treats the call as fire-and-forget-with-logging.

**Await before responding, despite fail-open.** Even though CAPI errors don't throw, the call must be `await`ed before the Deno edge function responds. Deno terminates the function after the response is returned — a pending `fetch()` would be killed mid-flight. Do not use `.then()` or `void sendMetaCAPIEvent(...)` — always `await`.

**Country defaults to `gb`.** Dinner-matcher is UK-only. The reference-file version accepts `params.country` as an override for apps serving other regions. When in doubt, pass `'gb'`, `'us'`, or whatever lowercase ISO-3166-1 alpha-2 code matches the user's actual country. Meta requires the country field present — omitting it hurts event match quality.

**PII hashing uses lowercase + trim.** Meta's dedup engine hashes the same way server-side to match events. Feed them raw inputs and let `sha256()` normalise.

**Env vars are read per-call, not module-level.** `Deno.env.get('META_PIXEL_ID')` inside the function body (not at the top of the file) because Deno edge functions can have env vars updated between invocations — reading at module load time would cache stale values.

## Verification after generation

```bash
# In the target project's supabase/functions directory:
deno check _shared/meta-capi.ts
```

Expected: no errors. If `Deno.env` type is unresolved, the target project is missing the Deno type lib in its `deno.json` (`"compilerOptions": { "lib": ["deno.ns"] }`).
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md
grep -c "sendMetaCAPIEvent" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md
grep -c "META_CAPI_TOKEN" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md
grep -c "v21.0" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md
```

Expected: file is ~200 lines, `sendMetaCAPIEvent` appears ≥3 times, `META_CAPI_TOKEN` appears ≥2 times, `v21.0` appears ≥1 time.

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/server-capi.md && \
  git commit -m "analytics-ga-meta-capi: add server CAPI helper reference"
```

---

## Task 5: Write `references/stripe-integration.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md`
- Source (read-only): `/Users/ryan/dinner-matcher/supabase/functions/create-checkout/index.ts`
- Source (read-only): `/Users/ryan/dinner-matcher/supabase/functions/stripe-webhook/index.ts`

**Goal of this reference file**: the anchored patches that wire CAPI calls into existing `create-checkout` and `stripe-webhook` functions. This is the highest-risk step of the installer — it edits existing code. The reference file specifies:

1. Exact sentinel anchors (strings to find in the target file)
2. Exact snippets to insert before / after each anchor
3. Complete before/after examples for copy-verification
4. What to do when anchors don't match (Scenario C from the spec)

- [ ] **Step 1: Write `references/stripe-integration.md`**

Content:

````markdown
# Stripe integration: patching `create-checkout` and `stripe-webhook`

This reference file contains the anchored patches that wire Meta CAPI events into the project's Stripe edge functions. Only run this step if **both** files exist:

- `supabase/functions/create-checkout/index.ts`
- `supabase/functions/stripe-webhook/index.ts`

If either is missing, skip this step and print: *"No Stripe functions detected. Install `stripe-checkout-supabase` first, then re-run this skill."*

## Overview of what gets wired

```
create-checkout:
  1. Import sendMetaCAPIEvent + buildUserData
  2. Extract fbc from request body (client passes via getFbc())
  3. Extract clientIp from x-forwarded-for header
  4. Extract clientUserAgent from user-agent header
  5. Generate purchaseEventId = crypto.randomUUID()
  6. Add purchase_event_id + fbc + client_ip + client_user_agent to Stripe metadata
  7. Append &eid=${purchaseEventId} to success_url
  8. After Stripe session created, await sendMetaCAPIEvent({event_name: 'InitiateCheckout', ...})

stripe-webhook:
  1. Import sendMetaCAPIEvent + buildUserData
  2. Inside case 'checkout.session.completed', after user update:
  3. Read purchase_event_id + fbc + client_ip + client_user_agent from session.metadata
  4. Look up user's email, phone, city from DB
  5. Build userData via buildUserData
  6. await Promise.all([
       sendMetaCAPIEvent({event_name: 'Purchase', event_id: purchaseEventId, ...}),
       sendMetaCAPIEvent({event_name: 'CompleteRegistration', event_id: purchaseEventId + '-reg', ...}),
     ])
```

The same `purchaseEventId` flows through:

```
create-checkout generates it
  → Stripe session metadata.purchase_event_id
  → success_url ?eid=<id> (returned to browser)
  → stripe-webhook reads metadata.purchase_event_id
  → CAPI Purchase event.event_id
  → Browser Pixel Purchase event.eventID (matches via eid query param)
  → Meta dedup collapses the pair
```

---

## Patch 1: `create-checkout/index.ts`

### Anchors (sentinel strings the inserter looks for)

| ID | Anchor | Insert where |
|---|---|---|
| A1 | `import Stripe from` (first import block) | Add `import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'` after the last import |
| A2 | `const { email, phone, plan, cancelUrl` | Extend destructure to include `eventId` and `fbc` |
| A3 | `stripe.checkout.sessions.create({` | Before this call, insert clientIp/clientUserAgent extraction and `purchaseEventId` generation |
| A4 | `metadata: {` (inside the `sessions.create` call) | Add `purchase_event_id`, `fbc`, `client_ip`, `client_user_agent` entries |
| A5 | `success_url:` (inside the `sessions.create` call) | Append `&eid=${purchaseEventId}` to the URL string |
| A6 | `return new Response(` (final successful response) | Before this return, insert the `await sendMetaCAPIEvent({event_name: 'InitiateCheckout', ...})` call |

If any anchor is not found, **abort the patch only** (not the whole skill), print the paste-me snippets below, and mark Step 4 as completed-with-manual-action.

### Patch snippets

**A1 — Imports (add after last existing import):**

```typescript
import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'
```

**A2 — Extend request body destructure:**

Change:
```typescript
const { email, phone, plan, cancelUrl } = await req.json()
```
to:
```typescript
const { email, phone, plan, cancelUrl, eventId, fbc } = await req.json()
```

If the existing line has other fields, preserve them — only *add* `eventId` and `fbc`.

**A3 — Pre-session-create metadata extraction (insert immediately before `stripe.checkout.sessions.create({`):**

```typescript
// Extract client info for CAPI (used both in InitiateCheckout and passed through Stripe for Purchase)
const clientIp = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() || null
const clientUserAgent = req.headers.get('user-agent')

// Generate a purchase event ID for deduplication through Stripe → webhook → browser
const purchaseEventId = crypto.randomUUID()
```

**A4 — Metadata field additions (inside the `metadata: { ... }` object):**

Add these fields (merge with any existing metadata fields):

```typescript
metadata: {
  purchase_event_id: purchaseEventId,
  ...(fbc && { fbc }),
  ...(clientIp && { client_ip: clientIp }),
  ...(clientUserAgent && { client_user_agent: clientUserAgent }),
  // ... preserve any existing fields ...
},
```

**A5 — `success_url` update:**

Change:
```typescript
success_url: `${subscriberAppUrl}/?welcome=true&email=${encodeURIComponent(email)}`,
```
to:
```typescript
success_url: `${subscriberAppUrl}/?welcome=true&email=${encodeURIComponent(email)}&eid=${purchaseEventId}`,
```

If the existing `success_url` is different, just append `&eid=${purchaseEventId}` to whatever query string the target uses.

**A6 — CAPI InitiateCheckout event (insert before the final successful `return new Response(`):**

```typescript
// Send InitiateCheckout CAPI event — must await before returning (Deno terminates after response)
await sendMetaCAPIEvent({
  event_name: 'InitiateCheckout',
  event_id: eventId || crypto.randomUUID(),
  event_time: Math.floor(Date.now() / 1000),
  event_source_url: `${onboardingAppUrl}/`,
  action_source: 'website',
  user_data: await buildUserData({
    email,
    phone,
    userId,
    clientIp,
    clientUserAgent,
    fbc,
  }),
})
```

Note: `userId` in the `user_data` call assumes the target function has already verified a JWT and captured the user ID. If the target's auth flow is different, swap `userId` for whatever the target uses (or omit it — it's optional in `buildUserData`). If the target has a `city` lookup, add `city: cityName` to the call.

If the target has `onboardingAppUrl` under a different name (e.g. `APP_URL`), swap it in the `event_source_url` string.

---

## Patch 2: `stripe-webhook/index.ts`

### Anchors

| ID | Anchor | Insert where |
|---|---|---|
| B1 | `import Stripe from` (first import block) | Add `import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'` after the last import |
| B2 | `case 'checkout.session.completed': {` | After the user-activation update (`users.subscription_status = 'active'`) and before the `break` |
| B3 | (inside B2's block, after the update) | Insert the full CAPI Purchase + CompleteRegistration block |

### Patch snippets

**B1 — Imports (add after last existing import):**

```typescript
import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'
```

**B2 + B3 — CAPI events block (insert after user update, before `break`):**

```typescript
// Send CAPI events (non-blocking — don't delay webhook response)
const purchaseEventId = session.metadata?.purchase_event_id
const fbc = session.metadata?.fbc || null
const clientIp = session.metadata?.client_ip || null
const clientUserAgent = session.metadata?.client_user_agent || null

// Look up user's name, phone and city for CAPI
const { data: userRow } = await supabase
  .from('users')
  .select('first_name, phone, city_id, city:city_id(name)')
  .eq('id', userId)
  .single()

const phone = userRow?.phone || null
const cityName = userRow?.city ? (userRow.city as { name: string }).name : null
const subscriberAppUrl = Deno.env.get('SUBSCRIBER_APP_URL')
const eventSourceUrl = subscriberAppUrl ? `${subscriberAppUrl}/` : ''

// Get actual amount from Stripe (amount_total is in pence/cents)
const amountTotal = session.amount_total
const purchaseValue = amountTotal ? amountTotal / 100 : undefined
const purchaseCurrency = session.currency?.toUpperCase() || 'GBP'

const userData = await buildUserData({
  email: customerEmail,
  phone,
  userId,
  city: cityName,
  clientIp,             // passed through Stripe metadata from create-checkout
  clientUserAgent,
  fbc,
})

// Await CAPI events — Deno terminates after response, so fire-and-forget would be killed
await Promise.all([
  sendMetaCAPIEvent({
    event_name: 'Purchase',
    event_id: purchaseEventId || crypto.randomUUID(),
    event_time: Math.floor(Date.now() / 1000),
    event_source_url: eventSourceUrl,
    action_source: 'website',
    user_data: userData,
    ...(purchaseValue != null && {
      custom_data: {
        currency: purchaseCurrency,
        value: purchaseValue,
      },
    }),
  }),
  sendMetaCAPIEvent({
    event_name: 'CompleteRegistration',
    event_id: purchaseEventId ? `${purchaseEventId}-reg` : crypto.randomUUID(),
    event_time: Math.floor(Date.now() / 1000),
    event_source_url: eventSourceUrl,
    action_source: 'website',
    user_data: userData,
    ...(purchaseValue != null && {
      custom_data: {
        currency: purchaseCurrency,
        value: purchaseValue,
      },
    }),
  }),
])
```

Note: this snippet assumes the target's user table has `first_name`, `phone`, `city_id`, and a `city` relation. If the target schema is different, adapt the `select` and the `cityName` / `phone` extraction. **Do not block on this adaptation during installation** — if anchors match but the user table doesn't, patch anyway and flag the schema mismatch in the final summary for the user to manually align.

---

## Anchor matching rules

**Exact-match sentinels.** The inserter greps for the anchor as a literal string. Minor whitespace differences are OK; the inserter should tolerate leading whitespace and trailing spaces but not rename the identifiers.

**First-match wins.** Each anchor appears at most once in the file. If an anchor appears multiple times (e.g. multiple `metadata: {` blocks), the inserter matches the first occurrence inside the `stripe.checkout.sessions.create({` call — not a different call site.

**Order of patches.** Apply in order: A1 → A2 → A3 → A4 → A5 → A6 for create-checkout, then B1 → B2+B3 for stripe-webhook. Each patch re-reads the file before its edit so line numbers stay correct after previous inserts.

**Abort cleanly on mismatch.** If any anchor doesn't match, stop patching *that file* (continue to the next file), print the manual-paste snippets for the missed anchors, and mark the step completed-with-manual-action. Do not write partial patches.

**Duplicate-patch detection.** Before patching, grep the target file for `sendMetaCAPIEvent`. If it's already present, the file has been patched already — skip with a "already patched" message. Do not re-patch.

## Verification after patching

```bash
# In the target project:
deno check supabase/functions/create-checkout/index.ts
deno check supabase/functions/stripe-webhook/index.ts
```

Both must return without errors. If either fails, the patch broke the file — revert and re-run with manual paste.

Also grep for the expected new content:

```bash
grep -c "sendMetaCAPIEvent" supabase/functions/create-checkout/index.ts  # expect ≥1
grep -c "purchaseEventId" supabase/functions/create-checkout/index.ts     # expect ≥2 (generation + usage)
grep -c "sendMetaCAPIEvent" supabase/functions/stripe-webhook/index.ts    # expect ≥2 (Purchase + CompleteRegistration)
grep -c "purchase_event_id" supabase/functions/stripe-webhook/index.ts    # expect ≥1
```
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md
grep -c "Anchor" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md
grep -c "sendMetaCAPIEvent" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md
grep -c "purchase_event_id" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md
```

Expected: file is ~280 lines, `Anchor` appears ≥10 times, `sendMetaCAPIEvent` appears ≥6 times.

- [ ] **Step 3: Verify anchors actually match dinner-matcher's current files**

```bash
cd /Users/ryan/dinner-matcher
# Each of these greps must return ≥1 match — otherwise the anchor is already stale
grep -c "import Stripe from" supabase/functions/create-checkout/index.ts
grep -c "const { email, phone, plan, cancelUrl" supabase/functions/create-checkout/index.ts
grep -c "stripe.checkout.sessions.create({" supabase/functions/create-checkout/index.ts
grep -c "case 'checkout.session.completed'" supabase/functions/stripe-webhook/index.ts
```

Expected: all four return ≥1. If any return 0, dinner-matcher has diverged from this plan and the anchors in `stripe-integration.md` need re-derivation before committing.

- [ ] **Step 4: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/stripe-integration.md && \
  git commit -m "analytics-ga-meta-capi: add Stripe integration patches reference"
```

---

## Task 6: Write `references/event-scan-patterns.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/event-scan-patterns.md`

**Goal of this reference file**: the grep pattern catalog that powers Step 6 (domain event scan) of the installer. This is new content — not extracted from dinner-matcher — that codifies a set of heuristics for finding event hook points in a React/TypeScript codebase.

- [ ] **Step 1: Write `references/event-scan-patterns.md`**

Content:

````markdown
# Domain event scan: grep patterns for Step 6

Step 6 of the installer scans the target SPA for likely event hook points and proposes tracking calls. This file is the pattern catalog.

## How the scan runs

1. Scope: grep **only inside the currently-targeted SPA's `src/` directory**. Do not scan other SPAs, node_modules, or tests.
2. For each pattern below, run the grep and collect `file:line` matches.
3. Deduplicate: one suggestion per unique `file:line`. If multiple patterns match the same location, pick the highest-priority pattern.
4. Rank by confidence (P0 patterns first, then P1, then P2).
5. Present interactively, one at a time, with y/n/rename.

## Pattern catalog

### P0 — Form submissions (highest confidence)

| Grep | Signal | Suggested event |
|---|---|---|
| `onSubmit=\{` on JSX `<form>` tags | User-initiated form submission | `trackLead()` + `trackEvent('form_submitted')` |
| `handleSubmit\s*=` or `const handleSubmit` near `await fetch\|await supabase` | Form handler with network call | `trackEvent('<form_name>_submitted')` |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "onSubmit=\{" src/
```

**When to use `trackLead()` vs `trackEvent('form_submitted')`**: `Lead` is a Meta standard event reserved for contact/signup forms where the user intent is "I'm interested." For every form, propose both if the form is in a top-of-funnel path (welcome screen, contact page); propose only `trackEvent` for utility forms (profile edit, settings).

**Auto-skip**: forms inside `/admin/` or `/settings/` paths — these are administration, not conversion.

### P0 — Checkout-related calls

| Grep | Signal | Action |
|---|---|---|
| `supabase.functions.invoke\(['"]create-checkout` | Stripe checkout initiation | **Already handled by Step 4 Stripe patches** — do not propose |
| `window.location.href.*stripe.*checkout.sessions` | Direct Stripe redirect | Propose `trackInitiateCheckout(eventId)` before the redirect |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "create-checkout|stripe.*checkout.sessions" src/
```

Skip matches inside `src/lib/` (library code, not user interactions).

### P1 — CTA button clicks

| Grep | Signal | Suggested event |
|---|---|---|
| `onClick=\{[^}]*\}` on `<button>` with text matching `/sign ?up\|get started\|start\|join\|subscribe\|continue/i` | CTA click | `trackEvent('cta_click', { cta: '<button_text>' })` |

**Ripgrep commands** (two-step: find buttons first, then filter by surrounding text):
```bash
# Step 1: find all button onClick handlers
rg -n --type=tsx "onClick=\{[^}]*\}" src/ | rg "button"

# Step 2: for each match, read 3 lines around it and check for CTA text
# (the installer does this in Claude rather than shell)
```

**Renaming**: ask the user for a more descriptive name than `cta_click` if the button's role is specific ("book_dinner", "upgrade_plan"). Default to `cta_click` if unclear.

### P1 — Authentication events

| Grep | Signal | Suggested event |
|---|---|---|
| `supabase.auth.signInWithOtp` | Magic-link login started | `trackEvent('login_started')` before, `trackEvent('login_completed')` after |
| `supabase.auth.signUp\(` | Password-based signup | `trackLead()` + `trackEvent('signup_completed')` |
| `supabase.auth.signInAnonymously` | Anonymous session creation | Usually too-early-in-funnel — skip unless user asks |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "supabase\.auth\.(signIn|signUp|signOut)" src/
```

### P1 — Funnel step navigation

| Grep | Signal | Suggested event |
|---|---|---|
| `navigate\(['"]/step/` or `useNavigate` call inside an onClick | Multi-step funnel navigation | `trackEvent('step_completed', { step: '<step_id>' })` |
| `setStep\s*\(` or `onNext\s*\(` | Local step state advance | Same as above |

**Ripgrep command:**
```bash
rg -n --type=tsx "navigate\(['\"]/step/|setStep\(|onNext\(" src/
```

### P2 — Engagement events (lower confidence, offer cautiously)

| Grep | Signal | Suggested event |
|---|---|---|
| `setXEnabled\s*\(\s*true` | Toggle flipped on | `trackEvent('feature_enabled', { feature: '<name>' })` |
| `\.update\(\s*\{[^}]*opted_in:\s*true` | Database opt-in write | `trackEvent('opted_in')` |

**Ripgrep command:**
```bash
rg -n --type=tsx --type=ts "opted_in|opt_in" src/
```

These P2 patterns often produce false positives — skip by default unless the user confirms.

### Negative patterns (auto-skip, never propose)

- `trackEvent\|trackLead\|trackPixelEvent` — the wrapper is already present at this site, don't double-track
- Paths containing `/test/`, `/tests/`, `__tests__`, `.test.`, `.spec.` — test files, not production
- Paths containing `/admin/` — administration, not conversion
- Files starting with `use` and returning hooks (`useFoo.ts`) — utility hooks, tracking belongs at the call site

## Suggestion presentation format

After running all patterns, present results one at a time:

```
Found <N> candidate event hook points across <M> files.
For each, I'll show the code and propose an event name.
You say y/n/rename. Nothing is edited until you confirm.

[1/<N>] <relative/path/to/file.tsx>:<line>
        <code context, 2-3 lines>
        Suggested: <tracking code to insert>
        y / n / rename: _
```

### On `y` — wire the event

Anchored insertion rules:

- **Inside an `onSubmit` handler**: insert the track call as the **first statement** of the handler, before any validation or async work. If the handler is an inline arrow function (`onSubmit={(e) => { ... }}`), extract it to a named handler first. If the handler is already named, add the call at the top.
- **Inside an `onClick` handler**: same — first statement, even before `e.preventDefault()`.
- **Inside an auth call chain**: `trackEvent('login_started')` before the `await signInWithOtp(...)`, `trackEvent('login_completed')` after it resolves successfully.
- **Inside a navigation call**: `trackEvent('step_completed', { step })` on the line above the `navigate(...)` call.

**Always add the import** at the top of the file if not already present:

```typescript
import { trackEvent } from './lib/analytics'
// and/or:
import { trackLead, trackInitiateCheckout, trackPixelEvent } from './lib/meta-pixel'
```

Path depth depends on the file location — use `./`, `../`, or `../../` as needed.

### On `rename` — customise the event name

Prompt: `"Event name (default: <suggested>):"`. User types a new name (or hits enter to accept). Then wire it with the new name. Use GA4 conventions: `snake_case`, verb-noun, no spaces.

### On `n` — skip

No edit. Move to the next suggestion. The `n` option is the safe default for unsure matches.

### Final summary

At the end:

```
Wired <K> events across <M> files. Skipped <L>. Renamed <R>.
```

## Event naming conventions

| Standard Meta events (use capitalised) | Custom events (use snake_case) |
|---|---|
| `Lead` | `form_submitted` |
| `InitiateCheckout` | `cta_click` |
| `Purchase` | `step_completed` |
| `CompleteRegistration` | `login_started` |
| `ViewContent` | `login_completed` |
| `AddToCart` | `feature_enabled` |

Standard Meta events go through **both** `trackPixelEvent` (with capitalised name) and `trackEvent` (with snake_case equivalent) so both backends see the conversion. Custom events go through `trackEvent` only unless the user explicitly asks for pixel coverage.
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/event-scan-patterns.md
grep -c "Ripgrep command" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/event-scan-patterns.md
grep -c "P0\|P1\|P2" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/event-scan-patterns.md
```

Expected: file is ~200 lines, `Ripgrep command` appears ≥5 times, priority markers appear ≥6 times.

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/event-scan-patterns.md && \
  git commit -m "analytics-ga-meta-capi: add event scan patterns reference"
```

---

## Task 7: Write `references/setup-checklist.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/setup-checklist.md`

**Goal of this reference file**: the user-facing provisioning checklist the skill prints at Step 7. These are out-of-band actions the user takes in external systems (Google Analytics, Meta Business Manager, Supabase secrets CLI).

- [ ] **Step 1: Write `references/setup-checklist.md`**

Content:

````markdown
# Setup checklist: provisioning GA4, Meta Pixel, and CAPI

The skill generates code and edits env files. **It does not create external accounts or provision credentials.** This checklist is printed at Step 7 — the user works through it to get real measurement IDs, pixel IDs, and access tokens, then fills in the env files.

## Checklist

### 1. Create a Google Analytics 4 property

- [ ] Go to https://analytics.google.com
- [ ] Admin → Create property → name it after the target app
- [ ] Create a **Web stream** pointing at the SPA's domain (e.g. `https://onboarding.example.com`)
- [ ] Copy the **Measurement ID** (format: `G-XXXXXXXXXX`)
- [ ] Paste into:
  - `.env.staging` as `VITE_GA_MEASUREMENT_ID=G-XXXXXXXXXX`
  - `.env.production` as `VITE_GA_MEASUREMENT_ID=G-YYYYYYYYYY` *(different property if you want staging/prod separation)*

**Decision: one property for both environments or two?**

- **Two properties (recommended)**: staging events don't contaminate production reports. Slightly more setup.
- **One property**: simpler, but you'll need to filter staging out in every report. Only pick this if you have ≤50 test sessions per month.

### 2. Create a Meta Pixel

- [ ] Go to https://business.facebook.com → Events Manager
- [ ] Connect Data Sources → Web → Meta Pixel → name it after the target app
- [ ] Copy the **Pixel ID** (numeric, ~15 digits)
- [ ] Paste into:
  - `.env.staging` as `VITE_META_PIXEL_ID=1234567890123456`
  - `.env.production` as `VITE_META_PIXEL_ID=<same or separate>`

**Decision: one pixel or two?**

- **One pixel (recommended)**: Meta's ad optimization works best with the maximum event volume. Use a single pixel and add a `test_event_code` at develop-time for filtering.
- **Two pixels**: only if you have large-scale staging traffic that would pollute ad audiences.

### 3. Generate a Meta Conversions API access token

- [ ] In Events Manager, open the pixel you just created
- [ ] Settings → Conversions API → "Generate access token"
- [ ] Copy the token (long string, starts with `EAA`)
- [ ] Set as Supabase secret (do NOT paste into any env file):

```bash
supabase secrets set META_CAPI_TOKEN=EAAxxxxxxxxxxxxx --project-ref <staging-ref>
supabase secrets set META_PIXEL_ID=1234567890123456 --project-ref <staging-ref>

# Repeat for production:
supabase secrets set META_CAPI_TOKEN=EAAyyyyyyyyyyyyy --project-ref <production-ref>
supabase secrets set META_PIXEL_ID=1234567890123456 --project-ref <production-ref>
```

**Why these go to Supabase secrets, not env files**: the CAPI access token is a write credential to your ad account. It must never appear in a client bundle. The `VITE_` prefix is deliberately absent — the skill's code reads these from `Deno.env` inside the edge function, which only runs server-side.

### 4. Deploy the edge functions with the new secrets

```bash
npx supabase functions deploy create-checkout --project-ref <staging-ref>
npx supabase functions deploy stripe-webhook --project-ref <staging-ref>

# Then production:
npx supabase functions deploy create-checkout --project-ref <production-ref>
npx supabase functions deploy stripe-webhook --project-ref <production-ref>
```

### 5. Verify events are flowing — Meta Events Manager

- [ ] Events Manager → your pixel → **Test Events** tab
- [ ] Copy the test event code (format: `TEST12345`)
- [ ] **Note: to use test codes server-side**, add `test_event_code: 'TEST12345'` to the outer object in `sendMetaCAPIEvent`'s body (alongside `data` and `access_token`). This is a dev-time edit — remove it before deploying to production.
- [ ] Open the target SPA in a browser
- [ ] Trigger `Lead` — submit a contact form (or whichever form the Step 6 scan wired `trackLead` into)
- [ ] Trigger `InitiateCheckout` — click the pay button
- [ ] Trigger `Purchase` — complete a Stripe test checkout using `4242 4242 4242 4242` / `tok_visa`
- [ ] In the Test Events tab, for each event you should see **two entries that collapse into one**: one "Browser" (client Pixel) and one "Server" (CAPI), with a green "Deduplicated" indicator.
- [ ] If they don't dedupe, the `purchaseEventId` plumbing is broken. Verify:
  - `create-checkout` response includes the eventId
  - The success redirect URL has `?eid=...` in the query string
  - `stripe-webhook` reads `session.metadata.purchase_event_id`
  - The browser's `trackPurchase(eventId)` call passes the same ID

### 6. Verify events are flowing — GA4 DebugView

- [ ] GA4 admin → DebugView
- [ ] Open the app with `?debug_mode=1` appended to the URL (or set `window.gtag('set', 'debug_mode', true)` in the console before interacting)
- [ ] Trigger the same events as in step 5
- [ ] DebugView should show them in real time. If not, the `VITE_GA_MEASUREMENT_ID` env var is wrong or not loaded in the bundle.

### 7. Production smoke test

Once staging is verified:

- [ ] Deploy the SPA to production
- [ ] Open the live URL in an incognito window
- [ ] Repeat the browser smoke test (devtools → Network → filter `gtag|facebook.net`)
- [ ] Run a real Stripe transaction with a small amount (or use Stripe's `tok_visa` in live mode if available for your account)
- [ ] Watch Events Manager for the dedup indicator on the production pixel

## Compliance note

**This skill installs analytics without a cookie consent banner**, matching the reference project (dinner-matcher). This is a GDPR/ePrivacy risk if you serve users in the UK or EU.

Before shipping to production for EU/UK traffic, add one of:

1. **GA4 Consent Mode v2** with a compliant banner (e.g. CookieYes, Osano, Cookiebot) → GA starts firing only after the user accepts.
2. **Meta Pixel consent API**: call `fbq('consent', 'grant')` / `fbq('consent', 'revoke')` from your banner's state.
3. **Geo-block analytics** for EU/UK users entirely.

None of these are installed by this skill — the seam for wiring them is the `initAnalytics()` function in `src/lib/initAnalytics.ts` (dynamic pattern) or the raw script tags in `index.html` (static pattern).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No events in GA4 DebugView | Measurement ID env var not set at build time | Check `.env.staging`, re-run `npm run build`, confirm `%VITE_GA_MEASUREMENT_ID%` replaced in `dist/index.html` |
| Events appear in browser but not in Events Manager test tab | Wrong pixel ID | Verify `VITE_META_PIXEL_ID` matches the pixel you're looking at in Events Manager |
| CAPI events missing from test tab | `META_CAPI_TOKEN` not set as Supabase secret | `supabase secrets list --project-ref <ref>` to verify, re-deploy function after setting |
| Browser and CAPI events not dedup'd | `purchaseEventId` not passing through | Add `console.log` at each stage (create-checkout, webhook, browser) — whichever stage has a different value is the break |
| `fbq is not defined` in console | Dynamic pattern, Turnstile not verified yet | Expected during dev — the verification effect fires once Turnstile resolves. To force-fire in dev, set `VITE_TURNSTILE_SITE_KEY=` (empty) in `.env.local` |
| `VITE_META_PIXEL_ID` literally in HTML | Vite build ran without the env var set | Missing from `.env.<mode>` file or wrong filename — `.env.staging` not `.env.stg` |
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/setup-checklist.md
grep -c "^- \[ \]" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/setup-checklist.md
grep -c "Troubleshooting" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/setup-checklist.md
```

Expected: file is ~160 lines, checkbox items appear ≥15 times, Troubleshooting section present.

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/references/setup-checklist.md && \
  git commit -m "analytics-ga-meta-capi: add setup checklist reference"
```

---

## Task 8: Write `SKILL.md` (orchestration guide)

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md`

**Goal**: The main entry point. Contains frontmatter, architecture diagram, the TaskCreate instruction, Steps 0–7 with STOP markers, and pointers to each reference file. This is the only file loaded by default when the skill is invoked.

- [ ] **Step 1: Write `SKILL.md`**

Content:

````markdown
---
name: analytics-ga-meta-capi
description: Install Google Analytics 4, Meta Pixel, and server-side Meta Conversions API (CAPI) into a React/Vite SPA + Supabase project. Generates client wrappers, a portable CAPI helper, anchored patches for existing Stripe edge functions (create-checkout + stripe-webhook), and an interactive scan that walks the target SPA and proposes domain event hook points. Supports both static (index.html script tags) and dynamic (initAnalytics.ts after Turnstile verification) installation patterns. Preserves the client+server dedup invariant via a shared purchaseEventId and fbclid → fbc attribution plumbing. Safe to run before or after stripe-checkout-supabase — detects Stripe functions and patches cleanly or skips with instructions. Use when the user wants to add GA4 / Meta Pixel / ad conversion tracking to a subscription SaaS built on Supabase + Stripe, mentions Meta CAPI, Conversions API, pixel deduplication, fbclid, or "making my ad dashboard work".
---

# Analytics: GA4 + Meta Pixel + Meta CAPI

Installs dinner-matcher's proven analytics pattern into a React/Vite SPA backed by Supabase. Three layers: client-side GA4 + Pixel wrappers, server-side Meta Conversions API helper, and anchored patches into Stripe edge functions that wire CAPI events onto checkout success.

## Architecture

```
Browser                                    Supabase Edge Functions
-------                                    -----------------------
index.html OR initAnalytics.ts
  |
  +-- gtag.js (GA4)              ----->    N/A (client only)
  +-- fbevents.js (Meta Pixel)   ----->    N/A (client only)
  |
src/lib/analytics.ts                       (GA4 event wrapper)
src/lib/meta-pixel.ts                      (Pixel wrapper + fbclid capture + fbc reconstruct)
  |
  | trackLead() on form submit        -->  client Pixel `Lead` event
  | trackInitiateCheckout() on pay    -->  client Pixel `InitiateCheckout`
  |                                           |
  |                                           v
  |                                        _shared/meta-capi.ts   (server CAPI helper)
  |                                           |
  |                                      create-checkout (patched)
  |                                         - generates purchaseEventId (UUID)
  |                                         - stores in Stripe metadata
  |                                         - sends InitiateCheckout to CAPI
  |                                           |
  |                                           v
  |                                      stripe-webhook (patched)
  |                                         - reads purchaseEventId from metadata
  |                                         - sends Purchase to CAPI with same eventId
  |
  | <-- success redirect with ?eid=<purchaseEventId>
  v
trackPurchase(eventId) on confirmation --> client Pixel `Purchase` (deduped by eventId)
```

**Key invariant**: the same `purchaseEventId` flows from `create-checkout` → Stripe metadata → `stripe-webhook` → redirect URL → browser Pixel `Purchase` event. Meta's dedup engine collapses the client Pixel fire and the server CAPI fire into one conversion.

## What gets installed

| Layer | File | Purpose |
|---|---|---|
| Client | `web-apps/<app>/src/lib/analytics.ts` | GA4 `trackEvent` wrapper |
| Client | `web-apps/<app>/src/lib/meta-pixel.ts` | Pixel wrapper + `trackLead/InitiateCheckout/Purchase` + `captureFbclid/getFbc` |
| Client | `web-apps/<app>/src/lib/initAnalytics.ts` | *(Dynamic pattern only)* — loads gtag/fbevents scripts after a verification gate |
| Client | `web-apps/<app>/index.html` | *(Static pattern only)* — inline `<script>` tags with `%VITE_*%` placeholders |
| Client | `web-apps/<app>/src/App.tsx` | *(Dynamic pattern only)* — `useEffect` calls for `captureFbclid()` and `initAnalytics()` |
| Client | `.env.staging` / `.env.production` | `VITE_GA_MEASUREMENT_ID`, `VITE_META_PIXEL_ID` |
| Server | `supabase/functions/_shared/meta-capi.ts` | `sendMetaCAPIEvent`, `buildUserData`, `sha256` |
| Server | `supabase/functions/create-checkout/index.ts` | *(Patch)* `InitiateCheckout` CAPI call + metadata plumbing |
| Server | `supabase/functions/stripe-webhook/index.ts` | *(Patch)* `Purchase` + `CompleteRegistration` CAPI calls |

## EXECUTION RULES — READ BEFORE STARTING

**Mechanism 1 — TaskCreate at entry.** Run Step 0 first (discovery probe is read-only, produces the SPA list and Stripe-function presence needed by Step 1). Then create a TaskCreate entry for each of Steps 1–7. Mark each `in_progress` when you start it and `completed` when done. Do not mark complete without verification.

**Mechanism 2 — Hard STOP checkpoints.** Three explicit stops:

1. **After Step 2, before Step 3** (server-side work begins): show what was written, ask *"proceed with server CAPI wiring?"* User says yes/no. Client-only installs exit cleanly here.
2. **Before Step 4** (Stripe function patches): show each planned edit as a diff preview, ask for explicit confirmation. If any sentinel anchor is missing, abort the **patch only** (not the whole skill), fall through to Scenario C: print paste-me snippets, mark Step 4 as completed-with-manual-action, proceed to Step 5.
3. **Before Step 6** (domain event scan): *"I'm about to grep the app for event hook points. This will read ~N files. Continue?"*

**Mechanism 3 — Evidence-based step completion.** Do not mark a step `completed` without grepping / reading the result to prove it landed:

| Step | Verification |
|---|---|
| 2 | Read each generated file, confirm `export function trackEvent` etc. present |
| 3 | Read `_shared/meta-capi.ts`, grep for `sendMetaCAPIEvent` export |
| 4 | Grep patched files for `sendMetaCAPIEvent('InitiateCheckout')` / `'Purchase'` |
| 5 | Grep env files for `VITE_GA_MEASUREMENT_ID=` and `VITE_META_PIXEL_ID=` |
| 6 | For each `y`-confirmed event, grep the target file for the wrapped call |

If verification fails, leave the task `in_progress`, print the mismatch, and ask whether to retry or abort.

## Step 0 — Discovery probe (read-only)

Detect the repo layout. Run these checks:

```bash
# SPA discovery
ls -d web-apps/*/ 2>/dev/null
# Expect: list of SPA directories. If the repo doesn't use web-apps/*, ask the user for custom paths.

# Stripe function detection
test -f supabase/functions/create-checkout/index.ts && echo "create-checkout: present" || echo "create-checkout: missing"
test -f supabase/functions/stripe-webhook/index.ts && echo "stripe-webhook: present" || echo "stripe-webhook: missing"
test -f supabase/functions/_shared/meta-capi.ts && echo "meta-capi.ts: present" || echo "meta-capi.ts: missing"
```

Print a summary:

```
Discovery:
  SPAs found: homepage, onboarding, subscribers, admin
  create-checkout: present
  stripe-webhook: present
  meta-capi.ts: missing (will generate)
```

## Step 1 — Gather requirements

Ask the user, one question at a time:

1. **Which SPA?** — present the list from Step 0. User picks one, or provides a custom path.
2. **Which install pattern?** — `static` (script tags in `index.html`, fires on first byte) or `dynamic` (`initAnalytics.ts`, fires after verification hook). Recommend based on heuristics from `references/init-patterns.md` (Turnstile / auth-gate detection).
3. **Do you already have GA4 measurement ID and Meta Pixel ID?** — if yes, capture them now. If no, the setup checklist walks through creation.
4. **Install server-side CAPI now?** — if `create-checkout` + `stripe-webhook` both present in Step 0: yes/no (default yes). If either missing: skip, offer "client-only now, re-run later".

Create the Task entries for Steps 1–7 now.

## Step 2 — Generate client-side files

Load: `references/client-wrappers.md` + `references/init-patterns.md`

- Write `web-apps/<app>/src/lib/analytics.ts` (from `client-wrappers.md`)
- Write `web-apps/<app>/src/lib/meta-pixel.ts` (from `client-wrappers.md`)
- If **dynamic pattern**: write `web-apps/<app>/src/lib/initAnalytics.ts` (from `init-patterns.md` Pattern B), then edit `src/App.tsx` to add the `captureFbclid()` + `initAnalytics()` `useEffect` hooks.
- If **static pattern**: edit `web-apps/<app>/index.html` to inject the GA4 + Pixel `<script>` snippets from `init-patterns.md` Pattern A.

**Verification**: read each new/edited file, confirm the expected exports / snippets are present.

**STOP CHECKPOINT 1** — Show the user what was generated. Ask: *"Proceed with server-side CAPI wiring?"* If no, exit cleanly (Step 5 env vars still runs for client-side; then skip to Step 6). If yes, continue.

## Step 3 — Generate `_shared/meta-capi.ts` (if missing)

Load: `references/server-capi.md`

If `supabase/functions/_shared/meta-capi.ts` exists from a previous install, skip this step. Otherwise write the file from `server-capi.md`.

**Verification**: grep the new file for `export async function sendMetaCAPIEvent`, `export async function buildUserData`, `META_CAPI_TOKEN`.

## Step 4 — Patch Stripe edge functions

Load: `references/stripe-integration.md`

**STOP CHECKPOINT 2** — Before touching the Stripe files, show the planned edits as diff previews. Get explicit user confirmation.

Apply patches in order: A1 → A2 → A3 → A4 → A5 → A6 on `create-checkout`, then B1 → B2+B3 on `stripe-webhook`. Each patch re-reads the file before its edit.

**If any anchor doesn't match** — abort the patch for *that file only* (continue the other file), print paste-me snippets, mark this step as "completed-with-manual-action" in the task.

**Verification**: grep each patched file for `sendMetaCAPIEvent` and `purchase_event_id`. Run `deno check` on each patched file.

## Step 5 — Env vars and secrets

- Add to `web-apps/<app>/.env.staging`:
  ```
  VITE_GA_MEASUREMENT_ID=G-XXXXXXXXXX
  VITE_META_PIXEL_ID=1234567890123456
  ```
- Add to `web-apps/<app>/.env.production` with different values (or the same, user's choice).
- Add to `web-apps/<app>/.env.local` as **commented-out** lines (`# VITE_GA_MEASUREMENT_ID=G-...`) so the target developer knows they exist.

- Print (**do not run**) the Supabase secrets commands for CAPI:
  ```bash
  supabase secrets set META_CAPI_TOKEN=<token> --project-ref <ref>
  supabase secrets set META_PIXEL_ID=<id> --project-ref <ref>
  ```

**Verification**: grep env files for `VITE_GA_MEASUREMENT_ID=` and `VITE_META_PIXEL_ID=`.

## Step 6 — Domain event scan (interactive)

Load: `references/event-scan-patterns.md`

**STOP CHECKPOINT 3** — *"I'm about to grep `web-apps/<app>/src/` for event hook points. Continue?"*

1. Run each grep pattern from `event-scan-patterns.md`
2. Collect `file:line` matches, deduplicate, rank by priority (P0 → P1 → P2)
3. Present one suggestion at a time in the format from the reference file
4. For each `y`, make an anchored insertion + add imports
5. For each `rename`, prompt for a new event name
6. For each `n`, skip
7. Final summary: *"Wired K events across M files. Skipped L. Renamed R."*

**Verification**: for each `y`-confirmed event, grep the target file for the new wrapper call.

## Step 7 — Setup checklist (print, do not execute)

Load: `references/setup-checklist.md`

Print the full checklist contents. Do not execute any of its steps — the user works through them manually (Google Analytics UI, Meta Business Manager, `supabase secrets set`).

Include the compliance note about missing cookie consent as the closing line.

Mark Task 7 completed.

## Reference files

| File | Loaded by |
|---|---|
| [`references/client-wrappers.md`](references/client-wrappers.md) | Step 2 |
| [`references/init-patterns.md`](references/init-patterns.md) | Step 2 |
| [`references/server-capi.md`](references/server-capi.md) | Step 3 |
| [`references/stripe-integration.md`](references/stripe-integration.md) | Step 4 |
| [`references/event-scan-patterns.md`](references/event-scan-patterns.md) | Step 6 |
| [`references/setup-checklist.md`](references/setup-checklist.md) | Step 7 |

## Forward compatibility

If `stripe-checkout-supabase` changes the structure of `create-checkout/index.ts` in a future version, this skill's anchors may break. When that happens, update `references/stripe-integration.md` with new anchors. The skill still aborts cleanly on old installs — no silent corruption.
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md
grep -c "^## Step" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md
grep -c "STOP CHECKPOINT" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md
grep -c "references/" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md
```

Expected: file is ~200 lines, 8 `## Step` headers (0 through 7), `STOP CHECKPOINT` appears ≥3 times, `references/` appears ≥12 times (6 files × 2 mentions each).

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/SKILL.md && \
  git commit -m "analytics-ga-meta-capi: add orchestration SKILL.md"
```

---

## Task 9: Write `TESTING.md`

**Files:**
- Create: `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/TESTING.md`

**Goal**: Pre-publish validation checklist — the manual QA steps for the skill author (not the skill user) to run before shipping a new version.

- [ ] **Step 1: Write `TESTING.md`**

Content:

````markdown
# Testing checklist — analytics-ga-meta-capi

Manual pre-publish validation. Run these before bumping the plugin version. Not automated — markdown-only skills can't have unit tests.

## 1. Smoke test on dinner-matcher admin SPA

`web-apps/admin/` currently has no analytics. It's the closest thing to a "fresh" target in the reference repo.

- [ ] Open a Claude Code session in `/Users/ryan/dinner-matcher`
- [ ] Invoke the skill: `/plugin invoke analytics-ga-meta-capi` (or the equivalent)
- [ ] Pick SPA: `admin`
- [ ] Pick pattern: `dynamic` (admin is auth-gated)
- [ ] Walk through Steps 0–7
- [ ] After completion:
  - [ ] `admin/src/lib/analytics.ts` exists and matches `references/client-wrappers.md`
  - [ ] `admin/src/lib/meta-pixel.ts` exists and matches
  - [ ] `admin/src/lib/initAnalytics.ts` exists and matches
  - [ ] `admin/src/App.tsx` has the two new `useEffect` hooks
  - [ ] `admin/.env.staging` has `VITE_GA_MEASUREMENT_ID=` and `VITE_META_PIXEL_ID=`
- [ ] Build the admin SPA: `cd web-apps/admin && npm run build`
- [ ] Expected: `tsc --noEmit` passes, Vite bundle succeeds
- [ ] Revert the changes: `cd /Users/ryan/dinner-matcher && git restore web-apps/admin/`

## 2. Anchor verification on dinner-matcher's real Stripe files

This is not an install — just a grep check to confirm the anchors in `stripe-integration.md` still match reality.

- [ ] For each anchor in `references/stripe-integration.md`, run `grep -c` in the corresponding dinner-matcher file:

```bash
cd /Users/ryan/dinner-matcher

# create-checkout anchors
grep -c "import Stripe from" supabase/functions/create-checkout/index.ts
grep -c "const { email, phone, plan, cancelUrl" supabase/functions/create-checkout/index.ts
grep -c "stripe.checkout.sessions.create({" supabase/functions/create-checkout/index.ts
grep -c "metadata: {" supabase/functions/create-checkout/index.ts
grep -c "success_url:" supabase/functions/create-checkout/index.ts

# stripe-webhook anchors
grep -c "import Stripe from" supabase/functions/stripe-webhook/index.ts
grep -c "case 'checkout.session.completed'" supabase/functions/stripe-webhook/index.ts
```

- [ ] Every grep returns ≥1. If any returns 0, the anchor is stale — update `references/stripe-integration.md` before publishing.

## 3. Scenario C rehearsal (anchor mismatch)

- [ ] In a scratch worktree, mangle an anchor in `create-checkout/index.ts`:
  ```bash
  cd /tmp && git clone /Users/ryan/dinner-matcher dinner-matcher-test
  cd dinner-matcher-test
  sed -i.bak 's/stripe.checkout.sessions.create/stripe.checkout.Sessions.create/' supabase/functions/create-checkout/index.ts
  ```
- [ ] Run the skill against this worktree, pick a SPA, reach Step 4
- [ ] Expected: the A3 anchor fails to match. Skill prints the paste-me snippet for A3 and continues to A4 (or exits the patch cleanly)
- [ ] No partial patch: `create-checkout/index.ts` should be unchanged from the mangled state
- [ ] Clean up: `rm -rf /tmp/dinner-matcher-test`

## 4. Scenario A rehearsal (no Stripe functions)

- [ ] In a scratch worktree, delete the Stripe functions:
  ```bash
  cd /tmp && git clone /Users/ryan/dinner-matcher dinner-matcher-test
  cd dinner-matcher-test
  rm -rf supabase/functions/create-checkout supabase/functions/stripe-webhook supabase/functions/_shared/meta-capi.ts
  ```
- [ ] Run the skill against this worktree, pick `web-apps/onboarding`, pick dynamic pattern
- [ ] Expected: Step 0 reports Stripe functions missing. Step 4 is skipped with the "install stripe-checkout-supabase first, then re-run" message.
- [ ] Steps 2, 3, 5 still run. Step 6 still runs. Step 7 still prints.
- [ ] `onboarding/src/lib/` has the client wrappers. `_shared/meta-capi.ts` is generated.
- [ ] Clean up: `rm -rf /tmp/dinner-matcher-test`

## 5. Step 6 domain event scan sanity check

- [ ] Run the skill on dinner-matcher's onboarding SPA up to Step 6
- [ ] Confirm the scan finds at least:
  - The contact form (`onSubmit` in the contact step)
  - The pay button (already handled by Stripe patches — should be skipped)
  - The welcome-step CTA click
- [ ] Reject each suggestion with `n` — confirm no edits are made on skip
- [ ] Revert any env var / wrapper file changes from earlier steps

## 6. Plugin metadata check

- [ ] `.claude-plugin/plugin.json` version matches `.claude-plugin/marketplace.json` version
- [ ] README.md lists the new skill in the skills table
- [ ] `skills/analytics-ga-meta-capi/SKILL.md` has `name:` and `description:` in frontmatter
- [ ] `/plugin marketplace update subos2008-skills` (from any Claude Code session) discovers the new version

## Sign-off

All boxes checked → tag a release → push.
````

- [ ] **Step 2: Verify the file was written correctly**

```bash
wc -l ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/TESTING.md
grep -c "^- \[ \]" ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/TESTING.md
```

Expected: file is ~100 lines, checkbox items appear ≥20 times.

- [ ] **Step 3: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add skills/analytics-ga-meta-capi/TESTING.md && \
  git commit -m "analytics-ga-meta-capi: add TESTING checklist"
```

---

## Task 10: Update root `README.md` to list the new skill

**Files:**
- Modify: `~/Dropbox/agent-skills/README.md`

- [ ] **Step 1: Read the current README.md**

```bash
cat ~/Dropbox/agent-skills/README.md
```

Find the "Skills included" section. It currently has two rows (spa-aws-deploy, stripe-checkout-supabase).

- [ ] **Step 2: Add the new skill row**

Use Edit to insert a new row in the table, after the `stripe-checkout-supabase` row and before the closing of the table.

Old:
```markdown
| `stripe-checkout-supabase` | Install Stripe subscription checkout into a React/Vite SPA + Supabase project: three Deno edge functions (create-checkout, stripe-webhook, create-portal-session), DB migration, frontend hooks, and Stripe dashboard setup checklist. |
```

New:
```markdown
| `stripe-checkout-supabase` | Install Stripe subscription checkout into a React/Vite SPA + Supabase project: three Deno edge functions (create-checkout, stripe-webhook, create-portal-session), DB migration, frontend hooks, and Stripe dashboard setup checklist. |
| `analytics-ga-meta-capi` | Install Google Analytics 4 + Meta Pixel + server-side Meta Conversions API into a React/Vite SPA + Supabase project. Client wrappers, portable CAPI helper, anchored patches for existing Stripe edge functions, interactive domain-event scan, and setup checklist. Companion to `stripe-checkout-supabase`. |
```

- [ ] **Step 3: Verify**

```bash
grep -c "analytics-ga-meta-capi" ~/Dropbox/agent-skills/README.md
```

Expected: ≥1.

- [ ] **Step 4: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add README.md && \
  git commit -m "README: list analytics-ga-meta-capi skill"
```

---

## Task 11: Bump plugin version to `0.4.0`

**Files:**
- Modify: `~/Dropbox/agent-skills/.claude-plugin/plugin.json`
- Modify: `~/Dropbox/agent-skills/.claude-plugin/marketplace.json`

A new skill is a minor version bump (`0.3.1` → `0.4.0`) per semver — adds functionality, no breaking changes.

- [ ] **Step 1: Edit `.claude-plugin/plugin.json`**

Current content:
```json
{
  "name": "subos2008-skills",
  "description": "Personal collection of Claude Code skills. Includes: spa-aws-deploy (AWS hosting for React/Vite SPAs with Supabase), stripe-checkout-supabase (Stripe subscription checkout on Supabase edge functions).",
  "version": "0.3.1",
  "author": {
    "name": "subos2008"
  }
}
```

Replace with:
```json
{
  "name": "subos2008-skills",
  "description": "Personal collection of Claude Code skills. Includes: spa-aws-deploy (AWS hosting for React/Vite SPAs with Supabase), stripe-checkout-supabase (Stripe subscription checkout on Supabase edge functions), analytics-ga-meta-capi (GA4 + Meta Pixel + Meta CAPI for React/Vite SPAs).",
  "version": "0.4.0",
  "author": {
    "name": "subos2008"
  }
}
```

- [ ] **Step 2: Edit `.claude-plugin/marketplace.json`**

Current content:
```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "subos2008-skills",
  "description": "subos2008's personal Claude Code skills collection",
  "owner": {
    "name": "subos2008"
  },
  "plugins": [
    {
      "name": "subos2008-skills",
      "source": "./",
      "description": "Personal Claude Code skills: spa-aws-deploy, stripe-checkout-supabase, and more.",
      "version": "0.3.1",
      "category": "development"
    }
  ]
}
```

Replace with:
```json
{
  "$schema": "https://anthropic.com/claude-code/marketplace.schema.json",
  "name": "subos2008-skills",
  "description": "subos2008's personal Claude Code skills collection",
  "owner": {
    "name": "subos2008"
  },
  "plugins": [
    {
      "name": "subos2008-skills",
      "source": "./",
      "description": "Personal Claude Code skills: spa-aws-deploy, stripe-checkout-supabase, analytics-ga-meta-capi.",
      "version": "0.4.0",
      "category": "development"
    }
  ]
}
```

- [ ] **Step 3: Verify both files show 0.4.0**

```bash
grep -c '"version": "0.4.0"' ~/Dropbox/agent-skills/.claude-plugin/plugin.json
grep -c '"version": "0.4.0"' ~/Dropbox/agent-skills/.claude-plugin/marketplace.json
```

Expected: both return `1`.

- [ ] **Step 4: Commit**

```bash
cd ~/Dropbox/agent-skills && \
  git add .claude-plugin/plugin.json .claude-plugin/marketplace.json && \
  git commit -m "Bump version to 0.4.0 for analytics-ga-meta-capi skill"
```

---

## Task 12: Final repo check + self-review

**No new files — verification only.**

- [ ] **Step 1: Verify the full skill layout**

```bash
find ~/Dropbox/agent-skills/skills/analytics-ga-meta-capi -type f | sort
```

Expected output:
```
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/SKILL.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/TESTING.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/client-wrappers.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/event-scan-patterns.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/init-patterns.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/server-capi.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/setup-checklist.md
/Users/ryan/Dropbox/agent-skills/skills/analytics-ga-meta-capi/references/stripe-integration.md
```

If any file is missing, go back to the corresponding task.

- [ ] **Step 2: Verify all commits landed**

```bash
cd ~/Dropbox/agent-skills && git log --oneline -15
```

Expected: the last 11 commits are the Tasks 2–11 commits from this plan (in order), plus the spec commit (`0b756c3`). No unintended commits.

- [ ] **Step 3: Verify working tree is clean**

```bash
cd ~/Dropbox/agent-skills && git status
```

Expected: `nothing to commit, working tree clean`. If any files are untracked/modified, investigate before considering the plan complete.

- [ ] **Step 4: Run the TESTING.md checklist Section 2 (anchor verification)**

```bash
cd /Users/ryan/dinner-matcher
grep -c "import Stripe from" supabase/functions/create-checkout/index.ts
grep -c "const { email, phone, plan, cancelUrl" supabase/functions/create-checkout/index.ts
grep -c "stripe.checkout.sessions.create({" supabase/functions/create-checkout/index.ts
grep -c "case 'checkout.session.completed'" supabase/functions/stripe-webhook/index.ts
```

Expected: each returns ≥1. If any returns 0, the anchors in `stripe-integration.md` are already stale — that's a bug to fix before releasing.

- [ ] **Step 5: Do NOT push**

Per user instructions in `CLAUDE.md` / `MEMORY.md`: pushing to master triggers CI/CD and is the user's decision. Stop here and tell the user the skill is ready to push when they're satisfied with it.

---

## Self-Review Checklist (run after completing the plan above)

**1. Spec coverage** — every scope decision from the spec maps to a task:

- [x] Full stack (client + server + fbclid) → Tasks 2, 3, 4, 5
- [x] Two installation patterns → Task 3 (`init-patterns.md`) + Task 8 (Step 1 in SKILL.md)
- [x] No cookie consent banner → Task 7 (compliance note in `setup-checklist.md`)
- [x] Generic wrappers + starter set → Task 2 (`client-wrappers.md`)
- [x] Single monolithic skill, per-SPA install → Task 8 (Step 1 in SKILL.md)
- [x] Soft-dependency detection → Task 8 (Step 0) + Task 5 (abort-cleanly in stripe-integration.md)
- [x] Three install scenarios (A/B/C) → Task 5 + Task 9 (TESTING sections 3+4)
- [x] TaskCreate + STOP checkpoints + evidence-based completion → Task 8 (Execution rules section)
- [x] 6 reference files → Tasks 2–7
- [x] Step 6 domain event scan → Task 6 (`event-scan-patterns.md`) + Task 8 (Step 6 orchestration)
- [x] Skill-level and installed-output verification → Task 9 (`TESTING.md`)
- [x] Forward-compat note for Stripe schema drift → Task 8 (last section of SKILL.md)

**2. Placeholder scan** — searched for `TODO`, `TBD`, `FIXME`, `XXX`, `...`. One legitimate use in `supabase secrets set META_CAPI_TOKEN=<token>` (user-fills value). No actual placeholders.

**3. Type consistency** — function names, env var names, and file paths are consistent across all tasks. Reality-vs-spec reconciliation section at the top pins `sendMetaCAPIEvent`, `META_CAPI_TOKEN`, `META_PIXEL_ID` as the authoritative names used throughout.

**4. Task ordering** — references are written before SKILL.md (which references them), plugin metadata bumped after all files are in place, verification runs last. No forward dependencies violated.

---

## Execution handoff

Plan complete and saved to `~/Dropbox/agent-skills/docs/superpowers/plans/2026-04-11-analytics-ga-meta-capi-skill.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Best for this plan because tasks are largely independent (reference files) and a fresh subagent per task keeps context pristine.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints for review. Better if you want to see every edit as it happens.

Which approach?
