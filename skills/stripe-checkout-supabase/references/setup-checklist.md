# Phase 2: Environment provisioning

Phase 2 creates Stripe objects (products, prices, webhook endpoints), wires their IDs into the generated code, and sets Supabase edge function secrets. It runs after Phase 1 (code generation) and can be re-run independently to add environments or rotate secrets.

For each batch of commands below, present them to the user and ask: "Want me to run these, or would you prefer to run them yourself?"

Never write secrets to files or log them.

## Step 2.1 — Identify environments

Ask the user what environments they have. Common setups:

- **Single environment** — just production (Stripe live mode, one Supabase project)
- **Two environments** — staging (Stripe test mode) + production (Stripe live mode)
- **Three environments** — dev + staging + production

For each environment, collect:

1. **Environment name** (e.g., "production", "staging")
2. **Stripe mode** — test or live. Typically: staging → test, production → live
3. **Stripe secret key** for that mode (`sk_test_...` or `sk_live_...`) — from Stripe Dashboard → Developers → API keys
4. **Supabase project ref** (e.g., `abcdefghij`)
5. **Subscriber app URL** (e.g., `https://subscribers.myapp.com`)
6. **Onboarding app URL** (e.g., `https://onboarding.myapp.com`)

Then run Steps 2.2–2.5 **for each environment in sequence**, completing one fully before starting the next.

## Step 2.2 — Create products and prices

Read the `PRICES` map from the generated `create-checkout/index.ts` to determine plan slugs. On a fresh run (sentinels present), create all products and prices. On a re-run (real IDs already present), ask the user whether to create new products/prices or keep existing ones.

**If this is the first environment being provisioned**, create the product and prices:

```bash
stripe products create \
  --name "PRODUCT_NAME" \
  --api-key SK_FOR_THIS_ENV

# One command per plan slug:
stripe prices create \
  --product prod_RETURNED_ID \
  --unit-amount AMOUNT_IN_CENTS \
  --currency CURRENCY \
  -d "recurring[interval]=INTERVAL" \
  --api-key SK_FOR_THIS_ENV
```

Capture the returned price IDs and update the matching branch (`test` or `live`) of the `PRICES` map in `create-checkout/index.ts`, replacing the `price_TODO_*` sentinels.

**If this is a subsequent environment**, mirror the product/price structure from the previous environment. Use the same product name, amounts, currency, and intervals — just a different `--api-key`. This keeps test and live configurations in sync.

## Step 2.3 — Create webhook endpoint

Create a webhook endpoint pointing at this environment's Supabase project:

```bash
stripe webhook_endpoints create \
  --url "https://PROJECT_REF.supabase.co/functions/v1/stripe-webhook" \
  --enabled-events checkout.session.completed,customer.subscription.deleted \
  --api-key SK_FOR_THIS_ENV
```

Capture the webhook signing secret (`whsec_...`) from the response — it's needed in Step 2.4.

## Step 2.4 — Set Supabase secrets

Set the edge function secrets for this environment:

```bash
npx supabase secrets set --project-ref PROJECT_REF \
  STRIPE_SECRET_KEY=SK_FOR_THIS_ENV \
  STRIPE_WEBHOOK_SECRET=WHSEC_FROM_STEP_2_3 \
  SUBSCRIBER_APP_URL=SUBSCRIBER_URL_FOR_THIS_ENV \
  ONBOARDING_APP_URL=ONBOARDING_URL_FOR_THIS_ENV
```

Never commit `sk_live_*` or `whsec_*` values to git.

### Optional secrets

The edge functions also read these optional environment variables. Set them if the project uses these features:

| Secret | Purpose |
|--------|---------|
| `SB_JWT_ISSUER` | Override the JWT issuer if it differs from `${SUPABASE_URL}/auth/v1` |
| `SENTRY_DSN_CREATE_CHECKOUT` | Sentry DSN for create-checkout (per function) |
| `SENTRY_DSN_STRIPE_WEBHOOK` | Sentry DSN for stripe-webhook |
| `SENTRY_DSN_PORTAL` | Sentry DSN for create-portal-session |
| `META_CAPI_ACCESS_TOKEN` | Meta Conversions API access token (if using Meta ads) |
| `META_CAPI_PIXEL_ID` | Meta Pixel ID (if using Meta ads) |

## Step 2.5 — Smoke test this environment

Deploy the edge functions to this environment (see Step 7 in the main skill), then verify:

1. **Checkout happy path** — open onboarding SPA, complete flow, pay with `4242 4242 4242 4242` (test mode) or a real card (live mode) → redirects to subscriber app
2. **Webhook fires** — check Stripe dashboard (Developers → Webhooks → endpoint) for `200 OK`; check Supabase function logs for errors
3. **DB state** — `users` row has `subscription_status = 'active'`, `stripe_customer_id` starts with `cus_`
4. **Billing portal** — log into subscriber app, click "Manage billing" → portal loads
5. **Cancel webhook** — cancel in portal → `subscription_status = 'cancelled'` in DB
6. **Duplicate blocked** — fresh incognito session, same email → `account_exists` error

Only after all six pass should you proceed to the next environment (or to production use if this is the only/last environment).

**Then go back to Step 2.2 for the next environment**, if there is one.

---

## Optional: Restricted API key hardening

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

## Local development (reference only)

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
