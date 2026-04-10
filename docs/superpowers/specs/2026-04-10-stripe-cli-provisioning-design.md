# Stripe CLI Provisioning — Design Spec

## Problem

The `stripe-checkout-supabase` skill's setup step (Step 6) requires manual clickops in the Stripe dashboard to create products, prices, and webhook endpoints, then copy-pasting IDs and secrets into code and Supabase. This is error-prone, tedious, and can lead to drift between test and live mode configurations.

## Solution

Split the skill into two explicit phases and replace dashboard clickops with an interactive CLI-driven flow that Claude executes (with user consent per batch of commands).

## Phase Structure

### Phase 1 — Code Generation (existing Steps 1–5, minor changes)

Generates all code artifacts: migration, shared helpers, edge functions, frontend hooks, config.

**Key change:** The `PRICES` map in `create-checkout/index.ts` uses sentinel placeholders instead of blanks:

```typescript
const PRICES = {
  test: {
    "1m": "price_TODO_test_1m",
    "3m": "price_TODO_test_3m",
  },
  live: {
    "1m": "price_TODO_live_1m",
    "3m": "price_TODO_live_3m",
  },
} as const;
```

The `price_TODO_` prefix fails visibly at Stripe's API if Phase 2 hasn't been run, rather than silently misconfiguring.

Phase 1 produces deployable code that compiles and deploys, but cannot process payments until Phase 2 wires in real credentials.

**Step 6 in SKILL.md** changes from "read the setup checklist and do clickops" to "run Phase 2 (environment provisioning) to create Stripe objects and wire credentials. Phase 2 can be run now or later."

**Step 7 (deploy and verify)** stays as-is.

Everything else in Phase 1 is unchanged — the skill remains fully self-contained with no dependency on official Stripe skills.

### Phase 2 — Environment Provisioning (replaces setup-checklist.md)

Interactive, CLI-driven, re-entrant. Can run immediately after Phase 1 or independently later (e.g., adding a new environment, rotating secrets).

For each batch of commands, Claude presents them and asks: "Want me to run these, or would you prefer to run them yourself?"

#### Step 2.1 — Collect credentials

Claude asks for:

- Stripe secret key for test mode (`sk_test_...`)
- Stripe secret key for live mode (`sk_live_...`)
- Supabase project refs and URLs for each environment (staging, production)
- Subscriber app URL and onboarding app URL per environment

Secrets are never written to files or logged. They are used only in CLI commands and passed to `supabase secrets set`.

#### Step 2.2 — Create products and prices

Claude reads the `PRICES` map from the generated `create-checkout/index.ts` to determine plan slugs, amounts, currencies, and intervals. On a fresh run (placeholders present), it creates all products and prices. On a re-run (real IDs already present), it asks the user whether to create new products/prices or keep existing ones — this supports adding a new environment without recreating Stripe objects.

Creates identical products and prices in both test and live modes in a single pass to avoid drift:

```bash
# Test mode
stripe products create --name "Monthly Membership" --api-key sk_test_...
stripe prices create --product prod_xxx --unit-amount 999 --currency usd \
  --recurring[interval]=month --api-key sk_test_...

# Live mode
stripe products create --name "Monthly Membership" --api-key sk_live_...
stripe prices create --product prod_yyy --unit-amount 999 --currency usd \
  --recurring[interval]=month --api-key sk_live_...
```

After execution, captures the returned price IDs and replaces `price_TODO_*` placeholders in `create-checkout/index.ts` with real IDs.

#### Step 2.3 — Create webhook endpoints

For each Supabase environment (staging + production), creates a webhook endpoint pointing at the correct Supabase project:

```bash
stripe webhook_endpoints create \
  --url https://<project-ref>.supabase.co/functions/v1/stripe-webhook \
  --enabled-events checkout.session.completed,customer.subscription.deleted \
  --api-key <sk_test_ or sk_live_ depending on environment>
```

Captures the webhook signing secret (`whsec_...`) from the response for use in Step 2.4.

Test-mode webhook endpoint points at the staging Supabase project. Live-mode webhook endpoint points at the production Supabase project.

#### Step 2.4 — Set Supabase secrets

For each environment:

```bash
npx supabase secrets set --project-ref <ref> \
  STRIPE_SECRET_KEY=<sk_test_ or sk_live_> \
  STRIPE_WEBHOOK_SECRET=<whsec_ from step 2.3> \
  SUBSCRIBER_APP_URL=<subscriber URL for this env> \
  ONBOARDING_APP_URL=<onboarding URL for this env>
```

#### Step 2.5 — Optional RAK hardening

Documents as an optional post-setup step. RAK creation is dashboard-only (no API or CLI support), so this cannot be automated.

Provides the specific permissions the edge functions need:

- `checkout_sessions: write`
- `customers: read`
- `webhook_endpoints: write`
- `billing_portal_sessions: write`

Instructs the user to create a RAK in the Stripe dashboard with these permissions, then replace `STRIPE_SECRET_KEY` in Supabase secrets with the RAK.

#### Step 2.6 — Local development (reference only)

Not automated. Kept as documentation for developers who want to test webhooks locally:

- `stripe listen --forward-to localhost:54321/functions/v1/stripe-webhook`
- Test card numbers (`4242 4242 4242 4242`, etc.)
- Note that `stripe listen` generates its own `whsec_...` that differs from the remote webhook secret

## File Changes

| File | Change | Details |
|------|--------|---------|
| `SKILL.md` | Edit | Rewrite Step 6 to reference Phase 2. Describe the two-phase model. |
| `references/setup-checklist.md` | Rewrite | Replace dashboard clickops with CLI-driven Phase 2 flow (steps 2.1–2.6) |
| `references/edge-functions.md` | Minor edit | Update `PRICES` map template to use `price_TODO_*` sentinel placeholders |
| `references/database.md` | No change | |
| `references/frontend-integration.md` | No change | |
| `references/shared-helpers.md` | No change | |

No new files. No deleted files.

## Design Decisions

**Why sentinel placeholders instead of empty strings?** An empty string or `"PASTE_HERE"` might be overlooked. `price_TODO_test_1m` will produce a clear Stripe API error ("No such price: price_TODO_test_1m") that immediately tells you Phase 2 hasn't been run.

**Why create test and live together?** Drift between test and live configurations is a real source of bugs. Creating both in one pass ensures the product structure, price amounts, and webhook event subscriptions match exactly.

**Why ask before running each batch?** Some users want full automation, others want to review CLI commands before execution. The "want me to run these?" pattern respects both preferences. Secrets are involved, so the user should always have the option to inspect what's being executed.

**Why keep the skill self-contained?** The official `stripe-best-practices` skill covers general Stripe API guidance, but this skill should work independently — it may be installed in projects that don't have the official skills.

**Why not automate RAK creation?** Stripe's API does not support programmatic RAK creation. It's dashboard-only. Documenting the specific permissions needed is the best we can do.

**Why not automate local dev setup?** `stripe listen` is a long-running foreground process used during active development, not a one-time provisioning step. It doesn't fit the "run once, configure environment" model of Phase 2.
