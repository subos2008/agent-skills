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
