# Phase 2: Environment provisioning

Phase 2 creates Stripe objects (products, prices, webhook endpoints), wires their IDs into the generated code, and sets Supabase edge function secrets. It runs after Phase 1 (code generation) and can be re-run independently to add environments or rotate secrets.

For each batch of commands below, present them to the user and ask: "Want me to run these, or would you prefer to run them yourself?"

Never write secrets to files or log them.

## Step 2.1 — Collect credentials

Ask the user for:

1. **Stripe test secret key** (`sk_test_...`) — from Stripe Dashboard → Developers → API keys (test mode)
2. **Stripe live secret key** (`sk_live_...`) — same page, live mode toggle
3. **Environments** — for each environment (typically staging + production):
   - Supabase project ref (e.g., `abcdefghij`)
   - Subscriber app URL (e.g., `https://subscribers.stg.myapp.com`)
   - Onboarding app URL (e.g., `https://onboarding.stg.myapp.com`)
4. **Environment-to-Stripe-mode mapping** — typically:
   - Staging → test mode (`sk_test_`)
   - Production → live mode (`sk_live_`)

## Step 2.2 — Create products and prices

Read the `PRICES` map from the generated `create-checkout/index.ts` to determine plan slugs. On a fresh run (sentinels present), create all products and prices. On a re-run (real IDs already present), ask the user whether to create new products/prices or keep existing ones.

Create identical products and prices in both test and live modes in a single pass:

```bash
# Test mode — create product
stripe products create \
  --name "PRODUCT_NAME" \
  --api-key sk_test_...

# Test mode — create prices (one per plan slug)
stripe prices create \
  --product prod_RETURNED_TEST_ID \
  --unit-amount AMOUNT_IN_CENTS \
  --currency CURRENCY \
  -d "recurring[interval]=INTERVAL" \
  --api-key sk_test_...

# Live mode — create identical product
stripe products create \
  --name "PRODUCT_NAME" \
  --api-key sk_live_...

# Live mode — create identical prices
stripe prices create \
  --product prod_RETURNED_LIVE_ID \
  --unit-amount AMOUNT_IN_CENTS \
  --currency CURRENCY \
  -d "recurring[interval]=INTERVAL" \
  --api-key sk_live_...
```

After execution, capture the returned price IDs and replace the `price_TODO_*` sentinels in `create-checkout/index.ts` with the real IDs.

## Step 2.3 — Create webhook endpoints

For each environment, create a webhook endpoint pointing at the Supabase project's edge function URL. Use the Stripe secret key that matches the environment's mode (test for staging, live for production):

```bash
stripe webhook_endpoints create \
  --url "https://PROJECT_REF.supabase.co/functions/v1/stripe-webhook" \
  --enabled-events checkout.session.completed,customer.subscription.deleted \
  --api-key SK_FOR_THIS_MODE
```

Capture the webhook signing secret (`whsec_...`) from the response — it's needed in Step 2.4.

Repeat for each environment. Staging gets a test-mode endpoint, production gets a live-mode endpoint.

## Step 2.4 — Set Supabase secrets

For each environment, set the edge function secrets:

```bash
npx supabase secrets set --project-ref PROJECT_REF \
  STRIPE_SECRET_KEY=SK_FOR_THIS_ENV \
  STRIPE_WEBHOOK_SECRET=WHSEC_FROM_STEP_2_3 \
  SUBSCRIBER_APP_URL=SUBSCRIBER_URL_FOR_THIS_ENV \
  ONBOARDING_APP_URL=ONBOARDING_URL_FOR_THIS_ENV
```

Never commit `sk_live_*` or `whsec_*` values to git.

## Step 2.5 — Optional: Restricted API key hardening

Stripe does not support programmatic creation of Restricted API Keys (RAKs) — this is a manual dashboard step.

If the user wants tighter security, guide them to create a RAK in the Stripe Dashboard (Developers → API Keys → Create restricted key) with these permissions:

| Resource | Permission |
|----------|-----------|
| Checkout Sessions | Write |
| Customers | Read |
| Webhook Endpoints | Write |
| Billing Portal Sessions | Write |

After creating the RAK (`rk_test_...` / `rk_live_...`), replace `STRIPE_SECRET_KEY` in Supabase secrets:

```bash
npx supabase secrets set --project-ref PROJECT_REF \
  STRIPE_SECRET_KEY=rk_live_NEW_RAK_VALUE
```

This is optional. The full secret key works correctly and is only exposed server-side in Supabase edge functions. RAKs reduce blast radius if a key is ever compromised.

## Step 2.6 — Local development (reference only)

This section is not automated. Use it when actively developing and testing webhooks locally.

```bash
# Terminal 1 — run edge functions locally
npx supabase functions serve --env-file supabase/.env.local --no-verify-jwt

# Terminal 2 — forward Stripe webhook events to localhost
stripe listen --forward-to localhost:54321/functions/v1/stripe-webhook
```

`stripe listen` prints a `whsec_...` — put that in `supabase/.env.local` as `STRIPE_WEBHOOK_SECRET`. This is different from the remote webhook signing secret.

Test card numbers:
- `4242 4242 4242 4242` — success
- `4000 0000 0000 9995` — declined (insufficient funds)
- `4000 0025 0000 3155` — requires 3DS authentication
- Any future expiry, any CVC, any postal code

## Smoke test checklist

After provisioning an environment, deploy (Step 7 in main skill) and verify:

1. **Checkout happy path** — open onboarding SPA, complete flow, pay with `4242...` → redirects to subscriber app
2. **Webhook fires** — check Stripe dashboard (Developers → Webhooks → endpoint) for `200 OK`; check Supabase function logs for errors
3. **DB state** — `users` row has `subscription_status = 'active'`, `stripe_customer_id` starts with `cus_`
4. **Billing portal** — log into subscriber app, click "Manage billing" → portal loads
5. **Cancel webhook** — cancel in portal → `subscription_status = 'cancelled'` in DB
6. **Duplicate blocked** — fresh incognito session, same email → `account_exists` error

Only after all six pass should you promote to the next environment.
