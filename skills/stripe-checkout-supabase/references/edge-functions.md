# Edge function source

Three Deno functions, all deployed as Supabase edge functions. Copy these into `supabase/functions/{name}/index.ts` and adjust the highlighted spots per project.

All three import from `_shared/` — make sure those helpers exist first (see `shared-helpers.md`).

## `create-checkout/index.ts`

Creates a Stripe Checkout session for a subscription. Verifies the caller's Supabase JWT manually (since gateway `verify_jwt` is disabled), looks up duplicate accounts, creates the session with the caller's `userId` as `client_reference_id` so the webhook can link it back, and returns the hosted checkout URL.

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import Stripe from 'https://esm.sh/stripe@22.0.1?target=deno'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.99.1'
import * as jose from 'jsr:@panva/jose@6'
import { initTracing, traced, withSpan, currentTraceId } from '../_shared/tracing.ts'
import { initSentry, Sentry } from '../_shared/sentry.ts'
import { corsHeaders } from '../_shared/cors.ts'
// Optional — remove if project does not use Meta CAPI:
// import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'

const tracer = initTracing('create-checkout')
initSentry(Deno.env.get('SENTRY_DSN_CREATE_CHECKOUT') ?? '')

const SUPABASE_JWT_ISSUER = Deno.env.get('SB_JWT_ISSUER') ??
  Deno.env.get('SUPABASE_URL')! + '/auth/v1'
const JWKS = jose.createRemoteJWKSet(
  new URL(Deno.env.get('SUPABASE_URL')! + '/auth/v1/.well-known/jwks.json'),
)

// Stripe Price IDs — NOT secrets. Hardcoded here so the price catalog lives
// in version control. Phase 2 (environment provisioning) replaces these
// sentinels with real price IDs from Stripe. If you see a Stripe API error
// like "No such price: price_TODO_test_1m", it means Phase 2 hasn't been run.
// Key structure: PRICES[env][plan_slug] = 'price_xxx'
const PRICES: Record<string, Record<string, string>> = {
  test: {
    '1m': 'price_TODO_test_1m',
    '3m': 'price_TODO_test_3m',
    '6m': 'price_TODO_test_6m',
  },
  live: {
    '1m': 'price_TODO_live_1m',
    '3m': 'price_TODO_live_3m',
    '6m': 'price_TODO_live_6m',
  },
}

serve(async (req) => {
  const cors = corsHeaders(req, {
    // CUSTOMIZE: replace with the onboarding origins for this project
    origins: [
      'https://onboarding.myapp.com',
      'https://onboarding.stg.myapp.com',
      'http://localhost:5173',
    ],
  })

  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: cors })
  }

  return traced(tracer, 'create-checkout', req, async (rootSpan) => {
    try {
      const stripeSecretKey = Deno.env.get('STRIPE_SECRET_KEY')
      const subscriberAppUrl = Deno.env.get('SUBSCRIBER_APP_URL')
      const onboardingAppUrl = Deno.env.get('ONBOARDING_APP_URL')

      if (!stripeSecretKey || !subscriberAppUrl || !onboardingAppUrl) {
        throw new Error('Missing required configuration')
      }

      // Verify JWT manually (gateway verify_jwt disabled for ES256 compat).
      // The JWT proves the caller went through the real onboarding flow
      // (including any captcha on the anonymous sign-in).
      const userId = await withSpan(tracer, 'verify-jwt', async (span) => {
        const authHeader = req.headers.get('Authorization')
        if (!authHeader?.startsWith('Bearer ')) {
          span.setAttribute('auth_method', 'none')
          return null
        }
        const token = authHeader.replace('Bearer ', '')
        try {
          const { payload } = await jose.jwtVerify(token, JWKS, {
            issuer: SUPABASE_JWT_ISSUER,
          })
          span.setAttribute('auth_method', 'jwt')
          return payload.sub ?? null
        } catch {
          span.setAttribute('auth_method', 'invalid_jwt')
          return null
        }
      })

      if (!userId) {
        return new Response(
          JSON.stringify({ error: 'Valid authorization required' }),
          { status: 401, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      const stripeEnv = stripeSecretKey.startsWith('sk_live_') ? 'live' : 'test'
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25.dahlia',
        httpClient: Stripe.createFetchHttpClient(),
      })

      const { email, phone, plan, cancelUrl, eventId, fbc } = await req.json()

      if (!email) {
        return new Response(
          JSON.stringify({ error: 'email is required' }),
          { status: 400, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      // CUSTOMIZE: default plan slug
      const selectedPlan = plan || '3m'

      const stripePriceId = PRICES[stripeEnv]?.[selectedPlan]
      if (!stripePriceId) {
        return new Response(
          JSON.stringify({ error: `Invalid plan: ${selectedPlan}` }),
          { status: 400, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      // Duplicate account check — block a new anonymous session from paying
      // when an existing account already owns this email or phone.
      const supabaseUrl = Deno.env.get('SUPABASE_URL')!
      const supabaseServiceRoleKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
      const supabase = createClient(supabaseUrl, supabaseServiceRoleKey)

      const { data: emailMatch } = await supabase
        .from('users')
        .select('id')
        .neq('id', userId)
        .eq('email', email)
        .limit(1)

      let phoneMatch: unknown[] | null = null
      if (phone) {
        const { data } = await supabase
          .from('users')
          .select('id')
          .neq('id', userId)
          .eq('phone', phone)
          .limit(1)
        phoneMatch = data
      }

      if ((emailMatch && emailMatch.length > 0) || (phoneMatch && phoneMatch.length > 0)) {
        return new Response(
          JSON.stringify({ error: 'account_exists' }),
          { status: 409, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      // Capture client info before we hand off to Stripe. The webhook runs
      // server-to-server and has no browser context, so anything we want in
      // CAPI Purchase events must be stashed in Stripe metadata now.
      const clientIp = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() || null
      const clientUserAgent = req.headers.get('user-agent')

      // Pre-allocate the Purchase event_id so the webhook and the browser
      // can both fire the Meta Purchase event with the same ID (dedup).
      const purchaseEventId = crypto.randomUUID()

      const session = await stripe.checkout.sessions.create({
        mode: 'subscription',
        customer_email: email,
        client_reference_id: userId,
        metadata: {
          purchase_event_id: purchaseEventId,
          ...(currentTraceId() && { checkout_trace_id: currentTraceId()! }),
          ...(fbc && { fbc }),
          ...(clientIp && { client_ip: clientIp }),
          ...(clientUserAgent && { client_user_agent: clientUserAgent }),
        },
        line_items: [{ price: stripePriceId, quantity: 1 }],
        success_url: `${subscriberAppUrl}/?welcome=true&email=${encodeURIComponent(email)}&eid=${purchaseEventId}`,
        cancel_url: cancelUrl || `${onboardingAppUrl}/?checkout=cancelled`,
      })

      // Optional — fire Meta CAPI InitiateCheckout. Remove if not using Meta.
      // await sendMetaCAPIEvent({
      //   event_name: 'InitiateCheckout',
      //   event_id: eventId || crypto.randomUUID(),
      //   event_time: Math.floor(Date.now() / 1000),
      //   event_source_url: `${onboardingAppUrl}/`,
      //   action_source: 'website',
      //   user_data: await buildUserData({
      //     email, phone, userId, clientIp, clientUserAgent, fbc,
      //   }),
      // })

      return new Response(
        JSON.stringify({ url: session.url }),
        { headers: { ...cors, 'Content-Type': 'application/json' } },
      )
    } catch (err) {
      Sentry.captureException(err)
      await Sentry.flush(2000)
      return new Response(
        JSON.stringify({ error: 'Internal server error' }),
        { status: 500, headers: { ...cors, 'Content-Type': 'application/json' } },
      )
    }
  })
})
```

## `stripe-webhook/index.ts`

Receives webhook events from Stripe. Two event types matter: `checkout.session.completed` (activate the user) and `customer.subscription.deleted` (mark them cancelled). The webhook verifies Stripe's signature instead of a Supabase JWT.

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import Stripe from 'https://esm.sh/stripe@22.0.1?target=deno'
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2.99.1'
import { initTracing, traced, withSpan } from '../_shared/tracing.ts'
import { initSentry, Sentry } from '../_shared/sentry.ts'

const tracer = initTracing('stripe-webhook')
initSentry(Deno.env.get('SENTRY_DSN_STRIPE_WEBHOOK') ?? '')

serve(async (req) => {
  return traced(tracer, 'stripe-webhook', req, async (rootSpan) => {
    try {
      const stripeSecretKey = Deno.env.get('STRIPE_SECRET_KEY')
      const webhookSecret = Deno.env.get('STRIPE_WEBHOOK_SECRET')
      const supabaseUrl = Deno.env.get('SUPABASE_URL')
      const supabaseServiceRoleKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')

      if (!stripeSecretKey || !webhookSecret || !supabaseUrl || !supabaseServiceRoleKey) {
        throw new Error('Missing required environment variables')
      }

      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25.dahlia',
        httpClient: Stripe.createFetchHttpClient(),
      })
      const supabase = createClient(supabaseUrl, supabaseServiceRoleKey)

      const body = await req.text()
      const signature = req.headers.get('stripe-signature')
      if (!signature) {
        return new Response('Missing stripe-signature header', { status: 400 })
      }

      // Verify Stripe signature. Must use constructEventAsync on Deno
      // (sync version uses Node crypto internals that are not available).
      const event = await withSpan(tracer, 'verify-stripe-signature', async (span) => {
        const e = await stripe.webhooks.constructEventAsync(body, signature, webhookSecret)
        span.setAttribute('event_type', e.type)
        return e
      })
      rootSpan.setAttribute('event_type', event.type)

      switch (event.type) {
        case 'checkout.session.completed': {
          await withSpan(tracer, 'handle-checkout-completed', async (span) => {
            const session = event.data.object as Stripe.Checkout.Session
            const userId = session.client_reference_id
            const stripeCustomerId = session.customer as string
            const customerEmail = session.customer_email

            span.setAttribute('stripe_customer_id', stripeCustomerId ?? 'unknown')
            span.setAttribute('user_id', userId ?? 'unknown')

            if (!userId) {
              console.error('No client_reference_id in checkout session')
              return
            }

            // Primary side effect — activate the user. This is the single
            // source of truth for "has paid"; everything else flows from it.
            const { error: updateError } = await supabase
              .from('users')
              .update({
                subscription_status: 'active',
                stripe_customer_id: stripeCustomerId,
              })
              .eq('id', userId)

            if (updateError) {
              console.error('Error updating user:', updateError)
              return
            }

            // Link the submitted email to the auth.users row. This converts
            // an anonymous Supabase user into one that can log in via magic
            // link. email_confirm: true skips the confirmation round-trip
            // (we trust the email because Stripe just charged a card at it).
            if (customerEmail) {
              const { error: authError } = await supabase.auth.admin.updateUserById(userId, {
                email: customerEmail,
                email_confirm: true,
              })
              if (authError) {
                console.error('Error linking email to auth user:', authError)
              }
            }

            // Correlate this webhook span with the originating create-checkout
            // span so Honeycomb can show the full request chain.
            const purchaseEventId = session.metadata?.purchase_event_id
            const checkoutTraceId = session.metadata?.checkout_trace_id
            if (checkoutTraceId) {
              rootSpan.setAttribute('payment.checkout_trace_id', checkoutTraceId)
              span.setAttribute('payment.checkout_trace_id', checkoutTraceId)
            }
            if (purchaseEventId) {
              rootSpan.setAttribute('payment.correlation_id', purchaseEventId)
              span.setAttribute('payment.correlation_id', purchaseEventId)
            }

            // TODO: project-specific side effects. Typical options:
            //
            //   1. Send welcome email (e.g., via Resend / Supabase functions).
            //   2. Fire Meta CAPI Purchase + CompleteRegistration events,
            //      using purchaseEventId as the event_id for dedup with the
            //      browser-side pixel event on the success URL.
            //   3. Enqueue onboarding tasks (add to pool, send SMS, etc.).
            //
            // Use session.metadata.client_ip / client_user_agent / fbc if
            // wiring CAPI — they were captured in create-checkout because
            // the webhook itself has no browser context.
          })
          break
        }

        case 'customer.subscription.deleted': {
          await withSpan(tracer, 'handle-subscription-deleted', async (span) => {
            const subscription = event.data.object as Stripe.Subscription
            const stripeCustomerId = subscription.customer as string
            span.setAttribute('stripe_customer_id', stripeCustomerId)

            const { error: cancelError } = await supabase
              .from('users')
              .update({ subscription_status: 'cancelled' })
              .eq('stripe_customer_id', stripeCustomerId)

            if (cancelError) {
              console.error('Error cancelling subscription:', cancelError)
            }
          })
          break
        }
      }

      return new Response(JSON.stringify({ received: true }), {
        headers: { 'Content-Type': 'application/json' },
      })
    } catch (err) {
      Sentry.captureException(err)
      await Sentry.flush(2000)
      const message = err instanceof Error ? err.message : 'Webhook error'
      console.error('Webhook error:', message)
      // Return 400 (not 500) so Stripe does not retry on malformed signatures.
      return new Response(JSON.stringify({ error: message }), {
        status: 400,
        headers: { 'Content-Type': 'application/json' },
      })
    }
  })
})
```

## `create-portal-session/index.ts`

Opens a Stripe billing portal session for an existing customer. This is how subscribers update their card, view invoices, and cancel.

```typescript
import { serve } from 'https://deno.land/std@0.177.0/http/server.ts'
import Stripe from 'https://esm.sh/stripe@22.0.1?target=deno'
import * as jose from 'jsr:@panva/jose@6'
import { initTracing, traced, withSpan } from '../_shared/tracing.ts'
import { initSentry, Sentry } from '../_shared/sentry.ts'
import { corsHeaders } from '../_shared/cors.ts'

const tracer = initTracing('create-portal-session')
initSentry(Deno.env.get('SENTRY_DSN_PORTAL') ?? '')

const SUPABASE_JWT_ISSUER = Deno.env.get('SB_JWT_ISSUER') ??
  Deno.env.get('SUPABASE_URL')! + '/auth/v1'
const JWKS = jose.createRemoteJWKSet(
  new URL(Deno.env.get('SUPABASE_URL')! + '/auth/v1/.well-known/jwks.json'),
)

serve(async (req) => {
  const cors = corsHeaders(req, {
    // CUSTOMIZE: replace with the subscriber origins for this project
    origins: [
      'https://subscribers.myapp.com',
      'https://subscribers.stg.myapp.com',
      'http://localhost:5174',
    ],
  })

  if (req.method === 'OPTIONS') {
    return new Response('ok', { headers: cors })
  }

  return traced(tracer, 'create-portal-session', req, async (rootSpan) => {
    try {
      const authResult = await withSpan(tracer, 'verify-jwt', async (span) => {
        const authHeader = req.headers.get('Authorization')
        if (!authHeader?.startsWith('Bearer ')) {
          return { error: 'Missing authorization header', status: 401 }
        }
        const token = authHeader.replace('Bearer ', '')
        try {
          await jose.jwtVerify(token, JWKS, { issuer: SUPABASE_JWT_ISSUER })
          return null
        } catch {
          return { error: 'Invalid token', status: 401 }
        }
      })

      if (authResult) {
        return new Response(
          JSON.stringify({ error: authResult.error }),
          { status: authResult.status, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      const stripeSecretKey = Deno.env.get('STRIPE_SECRET_KEY')
      const subscriberAppUrl = Deno.env.get('SUBSCRIBER_APP_URL')
      if (!stripeSecretKey || !subscriberAppUrl) {
        throw new Error('Missing required configuration')
      }

      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25.dahlia',
        httpClient: Stripe.createFetchHttpClient(),
      })

      const { customerId } = await req.json()
      if (!customerId) {
        return new Response(
          JSON.stringify({ error: 'customerId is required' }),
          { status: 400, headers: { ...cors, 'Content-Type': 'application/json' } },
        )
      }

      const session = await stripe.billingPortal.sessions.create({
        customer: customerId,
        return_url: subscriberAppUrl,
      })

      return new Response(
        JSON.stringify({ url: session.url }),
        { headers: { ...cors, 'Content-Type': 'application/json' } },
      )
    } catch (err) {
      Sentry.captureException(err)
      await Sentry.flush(2000)
      const message = err instanceof Error ? err.message : 'Internal server error'
      return new Response(
        JSON.stringify({ error: message }),
        { status: 500, headers: { ...cors, 'Content-Type': 'application/json' } },
      )
    }
  })
})
```

## `supabase/config.toml` additions

All three functions must disable gateway JWT verification. The checkout and portal functions verify the caller's JWT themselves with `jose`; the webhook verifies Stripe's signature instead.

```toml
[functions.create-checkout]
verify_jwt = false

[functions.stripe-webhook]
verify_jwt = false

[functions.create-portal-session]
verify_jwt = false
```

If the project is still on legacy HS256 Supabase signing keys, `create-checkout` and `create-portal-session` can set `verify_jwt = true` instead and drop the `jose` blocks. The webhook must stay `false` regardless — it authenticates via Stripe's signature, not the Supabase JWT.
