# Stripe dashboard + environment setup

Everything above this step is code generation. This step is clickops in the Stripe and Supabase dashboards, plus setting secrets. Do it once per environment (staging + production).

## Stripe dashboard

Stripe has a test mode toggle in the top nav. Do the full setup in test mode first, then repeat in live mode for production — the price IDs are separate between modes, which is why the `PRICES` map in `create-checkout/index.ts` has `test` and `live` branches.

### 1. Create the product and prices

1. Go to **Products → Add product**
2. Fill in name (e.g., "Monthly Membership"), description, image
3. Under **Pricing**:
   - Pricing model: **Standard pricing** (or **Package pricing** if you want volume discounts)
   - Billing period: **Recurring**, **Monthly** (or whatever)
   - Price: the amount
4. **Save product**
5. Copy the price ID (`price_...`) and paste it into `PRICES.test.1m` (or the matching plan slug)
6. To add more plans for the same product, go back to the product and click **Add another price** — one product, many recurring prices is the right shape for 1m/3m/6m durations or monthly/annual pairs. Copy each new price ID into the map.

Do this in **test mode** to populate `PRICES.test.*`, then **toggle to live mode and repeat** to populate `PRICES.live.*`. The mode toggle is a hard partition — test and live prices never mix.

### 2. Create the webhook endpoint

1. Go to **Developers → Webhooks → Add endpoint**
2. **Endpoint URL**: `https://<supabase-project-ref>.supabase.co/functions/v1/stripe-webhook`
3. **Events to send**: click **Select events** and check:
   - `checkout.session.completed`
   - `customer.subscription.deleted`
   - (Optional: `customer.subscription.updated` if you want to track pause / resume)
4. **Add endpoint**
5. On the endpoint detail page, click **Reveal signing secret** under the Signing secret section. Copy the value (`whsec_...`) — you need it in the next step as `STRIPE_WEBHOOK_SECRET`.

Create a separate webhook endpoint in test mode (pointing at staging Supabase) and in live mode (pointing at production Supabase). Each has its own signing secret.

### 3. (Optional) Configure the billing portal

Default billing portal settings work fine. If you want to customize what subscribers can do:

1. **Settings → Billing → Customer portal**
2. Toggle features: update card, cancel subscription, view invoices, switch plans
3. Set a **Return URL** — the portal uses this as a fallback, but the edge function also sets `return_url` per session so this mainly matters for direct portal links

## Supabase secrets

Edge functions read secrets from environment variables set via the Supabase dashboard. Set these for **each environment** (staging and production projects have separate secret stores).

### Required

| Secret | Where to get it | Scope |
|--------|-----------------|-------|
| `STRIPE_SECRET_KEY` | Stripe → Developers → API keys → Secret key. Use `sk_test_...` in staging, `sk_live_...` in production. | create-checkout, stripe-webhook, create-portal-session |
| `STRIPE_WEBHOOK_SECRET` | Stripe → Developers → Webhooks → endpoint → Signing secret (`whsec_...`) | stripe-webhook |
| `SUBSCRIBER_APP_URL` | Your subscriber SPA URL, e.g., `https://subscribers.myapp.com` | all three |
| `ONBOARDING_APP_URL` | Your onboarding SPA URL, e.g., `https://onboarding.myapp.com` | create-checkout |
| `SUPABASE_URL` | Automatically set by Supabase | all three |
| `SUPABASE_SERVICE_ROLE_KEY` | Automatically set by Supabase | create-checkout, stripe-webhook |

### Optional

| Secret | Purpose |
|--------|---------|
| `SB_JWT_ISSUER` | Override the JWT issuer if it differs from `${SUPABASE_URL}/auth/v1` |
| `SENTRY_DSN_CREATE_CHECKOUT` | Sentry DSN for create-checkout (per function) |
| `SENTRY_DSN_STRIPE_WEBHOOK` | Sentry DSN for stripe-webhook |
| `SENTRY_DSN_PORTAL` | Sentry DSN for create-portal-session |
| `META_CAPI_ACCESS_TOKEN` | Meta Conversions API access token (if using Meta ads) |
| `META_CAPI_PIXEL_ID` | Meta Pixel ID (if using Meta ads) |

### Setting secrets

Via the dashboard: **Project Settings → Edge Functions → Environment variables**.

Via the CLI:

```bash
npx supabase secrets set --project-ref $REF \
  STRIPE_SECRET_KEY=sk_test_xxx \
  STRIPE_WEBHOOK_SECRET=whsec_xxx \
  SUBSCRIBER_APP_URL=https://subscribers.stg.myapp.com \
  ONBOARDING_APP_URL=https://onboarding.stg.myapp.com
```

Never commit `sk_live_*` or `whsec_*` values to git. Store them in a password manager and paste into each environment directly.

## Local development

For testing the webhook locally (Deno `supabase functions serve` + Stripe CLI):

```bash
# Terminal 1 — run edge functions locally
npx supabase functions serve --env-file supabase/.env.local --no-verify-jwt

# Terminal 2 — forward Stripe webhook events to localhost
stripe listen --forward-to localhost:54321/functions/v1/stripe-webhook

# stripe listen prints a whsec_... — put that in supabase/.env.local as
# STRIPE_WEBHOOK_SECRET so the local webhook verifies signatures correctly.
# This is different from the dashboard webhook secret.
```

Test card numbers:
- `4242 4242 4242 4242` — success
- `4000 0000 0000 9995` — declined for insufficient funds
- `4000 0025 0000 3155` — requires 3DS authentication
- Any future expiry, any CVC, any postal code

## Smoke test checklist

After deploying to staging:

1. **Checkout happy path**
   - Open onboarding SPA
   - Complete the flow up to the plan selection step
   - Click pay → lands on Stripe hosted page
   - Enter `4242 4242 4242 4242`, any future expiry, any CVC → success
   - Redirects to subscriber app with `?welcome=true`
2. **Webhook fires**
   - In Stripe dashboard → Developers → Webhooks → endpoint → click the latest event
   - Status should be `200 OK`
   - Check Supabase function logs for `create-checkout` and `stripe-webhook` — no errors
3. **DB state**
   - Query `users` row in Supabase: `subscription_status = 'active'`, `stripe_customer_id` starts with `cus_`
   - Auth user should now have `email_confirmed_at` set (Stripe trusted the email)
4. **Billing portal**
   - Log into subscriber app
   - Click "Manage billing" → opens Stripe portal
   - Click cancel subscription → confirms cancel
5. **Cancel webhook fires**
   - Back in subscriber app, refetch user → `subscription_status = 'cancelled'`
6. **Duplicate blocked**
   - Open a fresh onboarding session (incognito)
   - Use the same email as the user above
   - Expect `account_exists` error on checkout, not a new Stripe session

Only after all six pass should you promote the changes to production.
