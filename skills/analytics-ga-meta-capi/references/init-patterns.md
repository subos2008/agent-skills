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
