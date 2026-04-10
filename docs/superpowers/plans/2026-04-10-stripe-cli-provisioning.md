# Stripe CLI Provisioning Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace manual Stripe dashboard setup in the `stripe-checkout-supabase` skill with an interactive CLI-driven provisioning flow, split into two independent phases (code generation + environment provisioning).

**Architecture:** Phase 1 (code generation) stays mostly unchanged but uses `price_TODO_*` sentinel placeholders. Phase 2 (environment provisioning) is a new CLI-driven flow in `references/setup-checklist.md` that creates Stripe objects, wires IDs into code, and sets Supabase secrets — all via Stripe CLI and Supabase CLI, with user consent before each batch.

**Tech Stack:** Stripe CLI, Supabase CLI, Markdown (skill authoring)

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `skills/stripe-checkout-supabase/SKILL.md` | Edit | Rewrite Step 6 to describe Phase 2 model, update Step 7 preamble |
| `skills/stripe-checkout-supabase/references/setup-checklist.md` | Rewrite | Replace dashboard clickops with CLI-driven Phase 2 flow (steps 2.1–2.6) |
| `skills/stripe-checkout-supabase/references/edge-functions.md` | Edit | Update PRICES map placeholder values to `price_TODO_*` sentinel format; update Stripe API version to `2026-03-25` (dahlia) |

---

### Task 1: Update PRICES map placeholders in edge-functions.md

**Files:**
- Modify: `skills/stripe-checkout-supabase/references/edge-functions.md:34-45`

- [ ] **Step 1: Update the PRICES map template**

Replace the current placeholder values with `price_TODO_*` sentinels and update the comment to explain Phase 2 wiring:

```typescript
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
```

- [ ] **Step 2: Commit**

```bash
git add skills/stripe-checkout-supabase/references/edge-functions.md
git commit -m "Update PRICES map to use price_TODO_* sentinel placeholders"
```

---

### Task 2: Rewrite setup-checklist.md as CLI-driven Phase 2

**Files:**
- Rewrite: `skills/stripe-checkout-supabase/references/setup-checklist.md`

- [ ] **Step 1: Rewrite setup-checklist.md**

Replace the entire file with the CLI-driven Phase 2 flow. The new content:

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add skills/stripe-checkout-supabase/references/setup-checklist.md
git commit -m "Rewrite setup-checklist as CLI-driven Phase 2 provisioning flow"
```

---

### Task 3: Update SKILL.md with two-phase model

**Files:**
- Modify: `skills/stripe-checkout-supabase/SKILL.md:141-149` (Step 6)
- Modify: `skills/stripe-checkout-supabase/SKILL.md:35` (architecture diagram note)

- [ ] **Step 1: Update Step 6 in SKILL.md**

Replace the current Step 6 content (lines 141–149):

```markdown
## Step 6: Stripe dashboard setup

Read `references/setup-checklist.md` for the clickthrough. Summary:

1. Create product(s) in Stripe dashboard — one product per subscription offering. For plan variants (1m/3m/6m), attach multiple recurring prices to the same product.
2. Copy each price ID into the `PRICES` map in `create-checkout/index.ts`. Use separate maps for test and live.
3. Create a webhook endpoint pointing at `https://<project>.supabase.co/functions/v1/stripe-webhook`. Subscribe to `checkout.session.completed` and `customer.subscription.deleted`. Copy the signing secret.
4. Set edge function secrets in the Supabase dashboard: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `SUBSCRIBER_APP_URL`, `ONBOARDING_APP_URL`. (Staging and production projects get their own secrets.)
5. For local testing: `stripe listen --forward-to localhost:54321/functions/v1/stripe-webhook`. Use test mode card `4242 4242 4242 4242`.
```

With:

```markdown
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
```

- [ ] **Step 2: Update the architecture overview note**

Replace the line at the end of the architecture diagram section (line 35):

```markdown
Three edge functions, one migration, a couple of React hooks, and some dashboard clicks. The full code is in `references/` — this file is the guide.
```

With:

```markdown
Three edge functions, one migration, and a couple of React hooks. The full code is in `references/` — this file is the guide. Setup has two phases: Phase 1 generates the code (Steps 1–5), Phase 2 provisions Stripe and Supabase environments via CLI (Step 6).
```

- [ ] **Step 3: Update Step 7 preamble**

The current Step 7 ("Deploy and verify") references the smoke test that's now part of Phase 2's setup-checklist.md. Add a one-line note at the top of Step 7 (after the `## Step 7: Deploy and verify` heading):

```markdown
After Phase 2 provisioning is complete, deploy the edge functions and run the smoke tests from `references/setup-checklist.md`.
```

- [ ] **Step 4: Commit**

```bash
git add skills/stripe-checkout-supabase/SKILL.md
git commit -m "Update SKILL.md with two-phase model and CLI provisioning"
```

---

### Task 4: Update Stripe API version in edge function templates

**Files:**
- Modify: `skills/stripe-checkout-supabase/references/edge-functions.md`

The edge function templates use `apiVersion: '2023-08-16'` which is outdated. The latest Stripe API version is `2026-03-25` (dahlia).

- [ ] **Step 1: Update apiVersion in all three edge function templates**

In `create-checkout/index.ts` template, replace:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2023-08-16',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

With:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

In `stripe-webhook/index.ts` template, replace:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2023-08-16',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

With:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

In `create-portal-session/index.ts` template, replace:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2023-08-16',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

With:

```typescript
      const stripe = new Stripe(stripeSecretKey, {
        apiVersion: '2026-03-25',
        httpClient: Stripe.createFetchHttpClient(),
      })
```

- [ ] **Step 2: Commit**

```bash
git add skills/stripe-checkout-supabase/references/edge-functions.md
git commit -m "Update Stripe API version to 2026-03-25 (dahlia) in edge function templates"
```
