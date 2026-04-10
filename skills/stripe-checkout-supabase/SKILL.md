---
name: stripe-checkout-supabase
description: Install Stripe subscription checkout into a React/Vite SPA + Supabase project. Creates three Deno edge functions (create-checkout, stripe-webhook, create-portal-session), a database migration for subscription state, frontend hooks for onboarding and subscriber apps, and the Stripe dashboard setup checklist. Use when the user wants to add paid subscriptions, Stripe checkout, a billing portal, or recurring payments to a Supabase-backed SPA — or mentions adding Stripe to an onboarding flow, wiring a webhook, gating users on subscription status, or "making money" from their app. Also use when scaffolding a new project that follows the dinner-matcher / comejoinus architecture.
---

# Stripe Checkout on Supabase

Installs production-grade Stripe subscription checkout into a React/Vite SPA backed by Supabase (Postgres + Auth + Deno edge functions). Mirrors the architecture used in dinner-matcher / comejoinus.app.

## Architecture

```
Onboarding SPA                    Subscriber SPA
    |                                  |
    | invoke('create-checkout')        | invoke('create-portal-session')
    v                                  v
+--------------------+           +-----------------------+
| create-checkout    |           | create-portal-session |
| (Deno edge fn)     |           | (Deno edge fn)        |
+--------------------+           +-----------------------+
    |                                  |
    v                                  v
Stripe Checkout session            Stripe Billing Portal
    |
    | checkout.session.completed
    v
+--------------------+
| stripe-webhook     |-----> users.subscription_status = 'active'
| (Deno edge fn)     |       users.stripe_customer_id = cus_xxx
+--------------------+       link email to auth user
                             send welcome email (optional)
                             fire Meta CAPI Purchase (optional)
```

Three edge functions, one migration, and a couple of React hooks. The full code is in `references/` — this file is the guide. Setup has two phases: Phase 1 generates the code (Steps 1–5), Phase 2 provisions Stripe and Supabase environments via CLI (Step 6).

## What gets created

| Component | File | Purpose |
|-----------|------|---------|
| Checkout session | `supabase/functions/create-checkout/index.ts` | Creates a Stripe Checkout subscription session |
| Webhook handler | `supabase/functions/stripe-webhook/index.ts` | Activates user on `checkout.session.completed`, cancels on `customer.subscription.deleted` |
| Billing portal | `supabase/functions/create-portal-session/index.ts` | Opens Stripe billing portal for existing customers |
| DB migration | `supabase/migrations/NNNNN_stripe_subscriptions.sql` | Adds `subscription_status` enum + `stripe_customer_id` to `users` |
| CORS helper | `supabase/functions/_shared/cors.ts` | Origin-allowlist CORS (installed only if missing) |
| Config toml | `supabase/config.toml` (edits) | `verify_jwt = false` for the new functions |
| Onboarding hook | `web-apps/{onboarding}/src/hooks/useCheckout.ts` | React hook that calls `create-checkout` and redirects |
| Subscriber hook | `web-apps/{subscribers}/src/hooks/useStripeActions.ts` | React hook for manage-billing + resubscribe |

## Architectural decisions worth understanding

Before generating code, know why the pieces look the way they do — you will need this to adapt the templates to the target project.

**Manual JWT verification with `jose`.** Supabase migrated to ES256 signing keys. The edge function gateway's built-in `verify_jwt` does not support ES256, so all functions set `verify_jwt = false` in `config.toml` and verify JWTs themselves via the project JWKS endpoint. `create-checkout` and `create-portal-session` require a valid JWT; `stripe-webhook` verifies Stripe's signature instead.

**Anonymous Supabase auth pre-checkout.** The onboarding flow calls `supabase.auth.signInAnonymously()` early (typically after a Turnstile captcha) so that by the time the user hits checkout there is already a `userId`. This ID is passed to Stripe as `client_reference_id`, which the webhook uses to link the Stripe customer to the Supabase user. If the target project requires a real login before checkout, that still works — the JWT is the JWT.

**Price IDs hardcoded, not env vars.** Stripe price IDs are not secrets. Embedding them in code (in a `{test: {...}, live: {...}}` map, keyed by plan slug) keeps the price catalog in version control and removes an env-var coordination burden across environments. The runtime picks test vs live based on the `sk_live_` / `sk_test_` prefix of `STRIPE_SECRET_KEY`.

**Metadata as a correlation channel.** Stripe metadata is the only way to pass context from `create-checkout` through Stripe's async pipeline to the webhook. Use it for: the pre-allocated `purchase_event_id` (for Meta CAPI deduplication), the originating trace ID (so Honeycomb spans correlate across functions), the `fbc` Facebook click ID, and the client IP + user agent (captured before Stripe's server-side call so CAPI events have real browser data).

**Duplicate account pre-check.** Before creating a Stripe session, `create-checkout` looks up the submitted email and phone against existing `users` rows (other than the current anonymous user) and returns `409 account_exists` if either matches. This blocks a subscribed user from accidentally paying twice with a fresh anonymous session.

**Webhook idempotency is lightweight.** The webhook does a single `users` UPDATE on `checkout.session.completed`; if Stripe retries, the UPDATE is a no-op. If you add side effects (welcome emails, analytics), guard them with "only if this is the first time we saw subscription_status flip to active".

**CORS is origin-allowlisted, not wildcard.** The `_shared/cors.ts` helper accepts an explicit list of allowed origins and only returns an `Access-Control-Allow-Origin` header when the request origin matches. This is stricter than `*` and still works with Supabase's `invoke()`.

## Step 1: Gather requirements

Before generating anything, ask:

1. **Project paths** — where do the edge functions live? (`supabase/functions/`) Where do the SPAs live? (`web-apps/onboarding/`, `web-apps/subscribers/`, or different)
2. **Plans and prices** — what subscription plans are there? e.g., `1m`, `3m`, `6m` durations, or `basic`/`pro` tiers. What are the prices? (Currency, amount.) The skill generates a `{test: {...}, live: {...}}` map; price IDs go in once Stripe dashboard is set up.
3. **Existing users table** — does `users` already have `subscription_status` and `stripe_customer_id`? If yes, skip the migration. If no, generate it. Is the PK `auth.users(id)` like dinner-matcher, or a separate `user_id` FK column?
4. **App URLs** — what are the onboarding + subscriber origins for CORS? (e.g., `https://onboarding.myapp.com`, `https://subscribers.myapp.com` plus staging variants)
5. **Success / cancel URLs** — where should Stripe redirect after success (typically subscriber app with `?welcome=true`) and after cancel (typically back to onboarding)
6. **Duplicate check** — should the skill pre-check email+phone duplicates before creating a session? (Default: yes, for paid-once subscription products)
7. **Auth flow** — does onboarding use anonymous auth before checkout, or does the user log in first? Either works; just affects how the caller obtains the JWT.
8. **Meta CAPI / analytics** — does the project already have `_shared/meta-capi.ts`? If yes, wire the CAPI calls in. If no, leave the calls out but keep the metadata fields on the Stripe session so they can be added later without another migration.
9. **Welcome email** — does the project have a notifications helper like dinner-matcher's `createNotification`? If yes, wire it in. If no, leave a `// TODO: send welcome email here` hook.
10. **Sentry + tracing** — does the project have `_shared/sentry.ts` and `_shared/tracing.ts`? If yes, import them. If no, use the minimal stubs from `references/shared-helpers.md`.

## Step 2: Database migration

Read `references/database.md` for the migration template. It adds (guarded with `IF NOT EXISTS`):

- `subscription_status` enum (`active`, `cancelled`, `paused`, `incomplete`)
- `users.subscription_status` column defaulting to `incomplete`
- `users.stripe_customer_id text`
- Index on `subscription_status`
- RLS policies — subscriber-only read on their own row, service-role write

If the target project already has these columns (check `supabase/migrations/*.sql` first), skip the migration and just verify the shape matches.

## Step 3: Shared helpers

Read `references/shared-helpers.md`. Check which of these exist in `supabase/functions/_shared/`:

- `cors.ts` — origin-allowlist CORS helper
- `tracing.ts` — OpenTelemetry wrapper (optional, can stub)
- `sentry.ts` — Sentry init (optional, can stub)
- `meta-capi.ts` — Meta Conversions API (optional, only if project uses Meta ads)

Create minimal versions for any that are missing. The edge functions import from these, so they must exist — but a no-op stub is fine for tracing/sentry/meta-capi if the project does not use them yet.

## Step 4: Edge functions

Read `references/edge-functions.md` for the complete source of all three functions. The key things to customize per project:

- **`PRICES` map** in `create-checkout/index.ts` — the plan slugs and Stripe price IDs
- **Allowed origins** in the `corsHeaders` call — onboarding domain + staging
- **Success / cancel URLs** in the Stripe session create call
- **Webhook side effects** — in `stripe-webhook/index.ts`, what happens after `subscription_status` is set to `active` (welcome email, CAPI events, analytics)
- **Subscriber app origins** in `create-portal-session/index.ts`

Then add to `supabase/config.toml`:

```toml
[functions.create-checkout]
verify_jwt = false

[functions.stripe-webhook]
verify_jwt = false

[functions.create-portal-session]
verify_jwt = false
```

All three disable gateway JWT verification — checkout and portal do it manually with `jose`, webhook verifies Stripe's signature instead.

## Step 5: Frontend integration

Read `references/frontend-integration.md` for the React hooks. Two pieces:

**Onboarding (`useCheckout.ts`)** — called from the final step of the onboarding flow. Signs the user in anonymously if not already signed in, then invokes `create-checkout` with email + phone + selected plan, and redirects the browser to the returned Stripe Checkout URL.

**Subscribers (`useStripeActions.ts`)** — provides `manageSubscription(customerId)` which opens the Stripe billing portal, and `resubscribe(userId, email)` which kicks off a new checkout for a cancelled user.

Both hooks use `supabase.functions.invoke()` which automatically attaches the user's JWT.

## Step 6: Environment provisioning (Phase 2)

Read `references/setup-checklist.md` for the full interactive flow. This step uses the Stripe CLI and Supabase CLI to create Stripe objects and wire credentials — no dashboard clickops needed.

**Phase 2 is independent of Phase 1** — it can run immediately after code generation, or later when the user has their Stripe keys ready. It can also be re-run to add new environments or rotate secrets.

Summary of what Phase 2 does:

1. Collects Stripe secret keys (test + live) and Supabase project refs for each environment
2. Creates products and prices in both test and live modes via `stripe products create` / `stripe prices create`
3. Replaces `price_TODO_*` sentinels in `create-checkout/index.ts` with real price IDs
4. Creates webhook endpoints via `stripe webhook_endpoints create` for each environment
5. Sets Supabase edge function secrets via `npx supabase secrets set`
6. Optionally guides RAK hardening (manual dashboard step — Stripe doesn't support programmatic RAK creation)

For each batch of CLI commands, ask the user: "Want me to run these, or would you prefer to run them yourself?"

## Step 7: Deploy and verify

After Phase 2 provisioning is complete, deploy the edge functions and run the smoke tests from `references/setup-checklist.md`.

```bash
# Deploy migration
npx supabase db push --project-ref $REF

# Deploy edge functions
npx supabase functions deploy create-checkout --project-ref $REF
npx supabase functions deploy stripe-webhook --project-ref $REF
npx supabase functions deploy create-portal-session --project-ref $REF
```

Verify end-to-end:

1. Open onboarding SPA, complete flow, hit checkout → lands on Stripe hosted page
2. Pay with `4242 4242 4242 4242` → redirects back to subscriber app
3. Check `users` row: `subscription_status = 'active'`, `stripe_customer_id` populated
4. Open subscriber app → "Manage billing" button should work → billing portal loads
5. Cancel in portal → webhook fires → `subscription_status = 'cancelled'` in DB

## Adapting for different architectures

**Single price, no plans.** Replace the `PRICES` map with a single `STRIPE_PRICE_ID` env var; drop the `plan` parameter from the request body.

**One-time payment, not subscription.** Change `mode: 'subscription'` to `mode: 'payment'` in the Stripe session call. The `customer.subscription.deleted` webhook branch becomes unnecessary; use `payment_intent.succeeded` or keep `checkout.session.completed` as the activation trigger.

**No anonymous auth.** The onboarding hook calls `signInAnonymously()` before invoking `create-checkout`. If the target project requires real login first, drop that call — the user's real JWT is already attached by `supabase.functions.invoke()`.

**Gateway verify_jwt instead of manual.** If the project is still on the legacy HS256 Supabase signing keys, you can set `verify_jwt = true` in `config.toml` for `create-checkout` and `create-portal-session` and delete the `jose` verification blocks. The webhook must stay at `false` (it uses Stripe's signature, not a Supabase JWT).
