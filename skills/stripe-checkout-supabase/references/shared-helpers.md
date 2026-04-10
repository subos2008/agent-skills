# Shared helpers

The edge functions import from `supabase/functions/_shared/`. If the target project does not already have these files, create minimal versions so the generated code compiles and runs. You can upgrade them later without touching the Stripe functions.

## `_shared/cors.ts` (required)

Origin-allowlist CORS helper. Small, copy as-is:

```typescript
interface CorsOptions {
  origins: string[]
  methods?: string
  headers?: string
}

const BASE_HEADERS = 'authorization, x-client-info, apikey, content-type'

export function corsHeaders(req: Request, options: CorsOptions): Record<string, string> {
  const origin = req.headers.get('Origin')
  const headers: Record<string, string> = {
    'Access-Control-Allow-Headers': options.headers ?? BASE_HEADERS,
    'Access-Control-Allow-Methods': options.methods ?? 'POST, OPTIONS',
  }
  if (origin && options.origins.includes(origin)) {
    headers['Access-Control-Allow-Origin'] = origin
    headers['Vary'] = 'Origin'
  }
  return headers
}
```

## `_shared/tracing.ts` (required — stub is fine)

The Stripe functions use `initTracing`, `traced`, `withSpan`, and `currentTraceId`. If the project does not have real OpenTelemetry wired up yet, a no-op stub works:

```typescript
// Minimal stub — replace with real OTel when tracing is set up.
// See the dinner-matcher _shared/tracing.ts for the real version, which
// has to work around Deno 2.1 OTel issues (custom SpanProcessor, manual
// OTLP serialization). Do not copy the default OTel Deno examples — they
// silently fail to flush on Supabase edge functions.

interface StubSpan {
  setAttribute(key: string, value: unknown): void
}

const stubSpan: StubSpan = {
  setAttribute: () => {},
}

interface StubTracer {
  _name: string
}

export function initTracing(serviceName: string): StubTracer {
  return { _name: serviceName }
}

export async function traced<T>(
  _tracer: StubTracer,
  _name: string,
  _req: Request,
  fn: (span: StubSpan) => Promise<T>,
): Promise<T> {
  return await fn(stubSpan)
}

export async function withSpan<T>(
  _tracer: StubTracer,
  _name: string,
  fn: (span: StubSpan) => Promise<T>,
): Promise<T> {
  return await fn(stubSpan)
}

export function currentTraceId(): string | null {
  return null
}
```

When the project wires up real Honeycomb tracing, replace this file — the function-level imports do not change. See `execution/025-observability.md` in dinner-matcher for the real implementation and the list of Deno 2.1 OTel gotchas.

## `_shared/sentry.ts` (required — stub is fine)

Same story — a stub lets the functions compile if the project is not on Sentry yet:

```typescript
// Minimal stub — replace with real @sentry/deno init when Sentry is set up.

interface StubSentry {
  captureException(err: unknown): void
  flush(timeoutMs: number): Promise<boolean>
}

export const Sentry: StubSentry = {
  captureException: (err) => {
    console.error('[sentry stub]', err)
  },
  flush: async () => true,
}

export function initSentry(_dsn: string): void {
  // no-op
}
```

When wiring real Sentry, the typical pattern is:

```typescript
import * as Sentry from 'https://deno.land/x/sentry/index.mjs'

export { Sentry }

export function initSentry(dsn: string) {
  if (!dsn) return
  Sentry.init({
    dsn,
    tracesSampleRate: 0,
    integrations: [],
  })
}
```

Per-function DSNs are stored in env vars (e.g., `SENTRY_DSN_CREATE_CHECKOUT`). Sentry flush must be awaited before returning from the handler because Deno kills the isolate immediately after the response — fire-and-forget loses the event.

## `_shared/meta-capi.ts` (optional)

Only needed if the project uses Meta Conversions API. If it does not, delete the `sendMetaCAPIEvent` / `buildUserData` imports and call sites from `create-checkout/index.ts` and `stripe-webhook/index.ts`. The rest of the code works unchanged.

If the project does use Meta CAPI, see dinner-matcher's `supabase/functions/_shared/meta-capi.ts` for the full implementation — it handles SHA-256 hashing of user data, the `fbc` / `fbp` click ID plumbing, and posting to Meta's Conversions API endpoint. That file is mostly project-agnostic and can be copied directly.

## `_shared/create-notification.ts` (optional)

Dinner-matcher's webhook calls `createNotification(...)` to enqueue a welcome email via a notifications table + SQS pipeline. If the target project has a different email system (direct Resend call, Postmark, etc.), inline that instead:

```typescript
// Inside handle-checkout-completed, after the users UPDATE:
await fetch('https://api.resend.com/emails', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${Deno.env.get('RESEND_API_KEY')}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    from: 'welcome@myapp.com',
    to: customerEmail,
    subject: 'Welcome!',
    html: `<p>Hi ${firstName}, welcome aboard.</p>`,
  }),
})
```

Keep it awaited — fire-and-forget dies with the isolate. If the email provider is slow (>1s), push the send into a queue (SQS, pg_cron, etc.) so the webhook stays fast enough that Stripe does not retry.
