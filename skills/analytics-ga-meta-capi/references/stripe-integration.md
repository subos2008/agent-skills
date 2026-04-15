# Stripe integration: patching `create-checkout` and `stripe-webhook`

This reference file contains the anchored patches that wire Meta CAPI events into the project's Stripe edge functions. Only run this step if **both** files exist:

- `supabase/functions/create-checkout/index.ts`
- `supabase/functions/stripe-webhook/index.ts`

If either is missing, skip this step and print: *"No Stripe functions detected. Install `stripe-checkout-supabase` first, then re-run this skill."*

## Overview of what gets wired

```
create-checkout:
  1. Import sendMetaCAPIEvent + buildUserData
  2. Extract fbc from request body (client passes via getFbc())
  3. Extract clientIp from x-forwarded-for header
  4. Extract clientUserAgent from user-agent header
  5. Generate purchaseEventId = crypto.randomUUID()
  6. Add purchase_event_id + fbc + client_ip + client_user_agent to Stripe metadata
  7. Append &eid=${purchaseEventId} to success_url
  8. After Stripe session created, await sendMetaCAPIEvent({event_name: 'InitiateCheckout', ...})

stripe-webhook:
  1. Import sendMetaCAPIEvent + buildUserData
  2. Inside case 'checkout.session.completed', after user update:
  3. Read purchase_event_id + fbc + client_ip + client_user_agent from session.metadata
  4. Look up user's email, phone, city from DB
  5. Build userData via buildUserData
  6. await Promise.all([
       sendMetaCAPIEvent({event_name: 'Purchase', event_id: purchaseEventId, ...}),
       sendMetaCAPIEvent({event_name: 'CompleteRegistration', event_id: purchaseEventId + '-reg', ...}),
     ])
```

The same `purchaseEventId` flows through:

```
create-checkout generates it
  → Stripe session metadata.purchase_event_id
  → success_url ?eid=<id> (returned to browser)
  → stripe-webhook reads metadata.purchase_event_id
  → CAPI Purchase event.event_id
  → Browser Pixel Purchase event.eventID (matches via eid query param)
  → Meta dedup collapses the pair
```

---

## Patch 1: `create-checkout/index.ts`

### Anchors (sentinel strings the inserter looks for)

| ID | Anchor | Insert where |
|---|---|---|
| A1 | `import Stripe from` (first import block) | Add `import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'` after the last import |
| A2 | `const { email, phone, plan, cancelUrl` | Extend destructure to include `eventId` and `fbc` |
| A3 | `stripe.checkout.sessions.create({` | Before this call, insert clientIp/clientUserAgent extraction and `purchaseEventId` generation |
| A4 | `metadata: {` (inside the `sessions.create` call) | Add `purchase_event_id`, `fbc`, `client_ip`, `client_user_agent` entries |
| A5 | `success_url:` (inside the `sessions.create` call) | Append `&eid=${purchaseEventId}` to the URL string |
| A6 | `return new Response(` (final successful response) | Before this return, insert the `await sendMetaCAPIEvent({event_name: 'InitiateCheckout', ...})` call |

If any anchor is not found, **abort the patch only** (not the whole skill), print the paste-me snippets below, and mark Step 4 as completed-with-manual-action.

### Patch snippets

**A1 — Imports (add after last existing import):**

```typescript
import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'
```

**A2 — Extend request body destructure:**

Change:
```typescript
const { email, phone, plan, cancelUrl } = await req.json()
```
to:
```typescript
const { email, phone, plan, cancelUrl, eventId, fbc } = await req.json()
```

If the existing line has other fields, preserve them — only *add* `eventId` and `fbc`.

**A3 — Pre-session-create metadata extraction (insert immediately before `stripe.checkout.sessions.create({`):**

```typescript
// Extract client info for CAPI (used both in InitiateCheckout and passed through Stripe for Purchase)
const clientIp = req.headers.get('x-forwarded-for')?.split(',')[0]?.trim() || null
const clientUserAgent = req.headers.get('user-agent')

// Generate a purchase event ID for deduplication through Stripe → webhook → browser
const purchaseEventId = crypto.randomUUID()
```

**A4 — Metadata field additions (inside the `metadata: { ... }` object):**

Add these fields (merge with any existing metadata fields):

```typescript
metadata: {
  purchase_event_id: purchaseEventId,
  ...(fbc && { fbc }),
  ...(clientIp && { client_ip: clientIp }),
  ...(clientUserAgent && { client_user_agent: clientUserAgent }),
  // ... preserve any existing fields ...
},
```

**A5 — `success_url` update:**

Change:
```typescript
success_url: `${subscriberAppUrl}/?welcome=true&email=${encodeURIComponent(email)}`,
```
to:
```typescript
success_url: `${subscriberAppUrl}/?welcome=true&email=${encodeURIComponent(email)}&eid=${purchaseEventId}`,
```

If the existing `success_url` is different, just append `&eid=${purchaseEventId}` to whatever query string the target uses.

**A6 — CAPI InitiateCheckout event (insert before the final successful `return new Response(`):**

```typescript
// Send InitiateCheckout CAPI event — must await before returning (Deno terminates after response)
await sendMetaCAPIEvent({
  event_name: 'InitiateCheckout',
  event_id: eventId || crypto.randomUUID(),
  event_time: Math.floor(Date.now() / 1000),
  event_source_url: `${onboardingAppUrl}/`,
  action_source: 'website',
  user_data: await buildUserData({
    email,
    phone,
    userId,
    clientIp,
    clientUserAgent,
    fbc,
  }),
})
```

Note: `userId` in the `user_data` call assumes the target function has already verified a JWT and captured the user ID. If the target's auth flow is different, swap `userId` for whatever the target uses (or omit it — it's optional in `buildUserData`). If the target has a `city` lookup, add `city: cityName` to the call.

If the target has `onboardingAppUrl` under a different name (e.g. `APP_URL`), swap it in the `event_source_url` string.

---

## Patch 2: `stripe-webhook/index.ts`

### Anchors

| ID | Anchor | Insert where |
|---|---|---|
| B1 | `import Stripe from` (first import block) | Add `import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'` after the last import |
| B2 | `case 'checkout.session.completed': {` | After the user-activation update (`users.subscription_status = 'active'`) and before the `break` |
| B3 | (inside B2's block, after the update) | Insert the full CAPI Purchase + CompleteRegistration block |

### Patch snippets

**B1 — Imports (add after last existing import):**

```typescript
import { sendMetaCAPIEvent, buildUserData } from '../_shared/meta-capi.ts'
```

**B2 + B3 — CAPI events block (insert after user update, before `break`):**

```typescript
// Send CAPI events (non-blocking — don't delay webhook response)
const purchaseEventId = session.metadata?.purchase_event_id
const fbc = session.metadata?.fbc || null
const clientIp = session.metadata?.client_ip || null
const clientUserAgent = session.metadata?.client_user_agent || null

// Look up user's name, phone and city for CAPI
const { data: userRow } = await supabase
  .from('users')
  .select('first_name, phone, city_id, city:city_id(name)')
  .eq('id', userId)
  .single()

const phone = userRow?.phone || null
const cityName = userRow?.city ? (userRow.city as { name: string }).name : null
const subscriberAppUrl = Deno.env.get('SUBSCRIBER_APP_URL')
const eventSourceUrl = subscriberAppUrl ? `${subscriberAppUrl}/` : ''

// Get actual amount from Stripe (amount_total is in pence/cents)
const amountTotal = session.amount_total
const purchaseValue = amountTotal ? amountTotal / 100 : undefined
const purchaseCurrency = session.currency?.toUpperCase() || 'GBP'

const userData = await buildUserData({
  email: customerEmail,
  phone,
  userId,
  city: cityName,
  clientIp,             // passed through Stripe metadata from create-checkout
  clientUserAgent,
  fbc,
})

// Await CAPI events — Deno terminates after response, so fire-and-forget would be killed
await Promise.all([
  sendMetaCAPIEvent({
    event_name: 'Purchase',
    event_id: purchaseEventId || crypto.randomUUID(),
    event_time: Math.floor(Date.now() / 1000),
    event_source_url: eventSourceUrl,
    action_source: 'website',
    user_data: userData,
    ...(purchaseValue != null && {
      custom_data: {
        currency: purchaseCurrency,
        value: purchaseValue,
      },
    }),
  }),
  sendMetaCAPIEvent({
    event_name: 'CompleteRegistration',
    event_id: purchaseEventId ? `${purchaseEventId}-reg` : crypto.randomUUID(),
    event_time: Math.floor(Date.now() / 1000),
    event_source_url: eventSourceUrl,
    action_source: 'website',
    user_data: userData,
    ...(purchaseValue != null && {
      custom_data: {
        currency: purchaseCurrency,
        value: purchaseValue,
      },
    }),
  }),
])
```

Note: this snippet assumes the target's user table has `first_name`, `phone`, `city_id`, and a `city` relation. If the target schema is different, adapt the `select` and the `cityName` / `phone` extraction. **Do not block on this adaptation during installation** — if anchors match but the user table doesn't, patch anyway and flag the schema mismatch in the final summary for the user to manually align.

---

## Anchor matching rules

**Exact-match sentinels.** The inserter greps for the anchor as a literal string. Minor whitespace differences are OK; the inserter should tolerate leading whitespace and trailing spaces but not rename the identifiers.

**First-match wins.** Each anchor appears at most once in the file. If an anchor appears multiple times (e.g. multiple `metadata: {` blocks), the inserter matches the first occurrence inside the `stripe.checkout.sessions.create({` call — not a different call site.

**Order of patches.** Apply in order: A1 → A2 → A3 → A4 → A5 → A6 for create-checkout, then B1 → B2+B3 for stripe-webhook. Each patch re-reads the file before its edit so line numbers stay correct after previous inserts.

**Abort cleanly on mismatch.** If any anchor doesn't match, stop patching *that file* (continue to the next file), print the manual-paste snippets for the missed anchors, and mark the step completed-with-manual-action. Do not write partial patches.

**Duplicate-patch detection.** Before patching, grep the target file for `sendMetaCAPIEvent`. If it's already present, the file has been patched already — skip with a "already patched" message. Do not re-patch.

## Verification after patching

```bash
# In the target project:
deno check supabase/functions/create-checkout/index.ts
deno check supabase/functions/stripe-webhook/index.ts
```

Both must return without errors. If either fails, the patch broke the file — revert and re-run with manual paste.

Also grep for the expected new content:

```bash
grep -c "sendMetaCAPIEvent" supabase/functions/create-checkout/index.ts  # expect ≥1
grep -c "purchaseEventId" supabase/functions/create-checkout/index.ts     # expect ≥2 (generation + usage)
grep -c "sendMetaCAPIEvent" supabase/functions/stripe-webhook/index.ts    # expect ≥2 (Purchase + CompleteRegistration)
grep -c "purchase_event_id" supabase/functions/stripe-webhook/index.ts    # expect ≥1
```
