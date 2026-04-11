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
