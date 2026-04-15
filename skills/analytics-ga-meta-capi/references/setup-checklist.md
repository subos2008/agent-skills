# Setup checklist: provisioning GA4, Meta Pixel, and CAPI

The skill generates code and edits env files. **It does not create external accounts or provision credentials.** This checklist is printed at Step 7 — the user works through it to get real measurement IDs, pixel IDs, and access tokens, then fills in the env files.

## Checklist

### 1. Create a Google Analytics 4 property

- [ ] Go to https://analytics.google.com
- [ ] Admin → Create property → name it after the target app
- [ ] Create a **Web stream** pointing at the SPA's domain (e.g. `https://onboarding.example.com`)
- [ ] Copy the **Measurement ID** (format: `G-XXXXXXXXXX`)
- [ ] Paste into:
  - `.env.staging` as `VITE_GA_MEASUREMENT_ID=G-XXXXXXXXXX`
  - `.env.production` as `VITE_GA_MEASUREMENT_ID=G-YYYYYYYYYY` *(different property if you want staging/prod separation)*

**Decision: one property for both environments or two?**

- **Two properties (recommended)**: staging events don't contaminate production reports. Slightly more setup.
- **One property**: simpler, but you'll need to filter staging out in every report. Only pick this if you have ≤50 test sessions per month.

### 2. Create a Meta Pixel

- [ ] Go to https://business.facebook.com → Events Manager
- [ ] Connect Data Sources → Web → Meta Pixel → name it after the target app
- [ ] Copy the **Pixel ID** (numeric, ~15 digits)
- [ ] Paste into:
  - `.env.staging` as `VITE_META_PIXEL_ID=1234567890123456`
  - `.env.production` as `VITE_META_PIXEL_ID=<same or separate>`

**Decision: one pixel or two?**

- **One pixel (recommended)**: Meta's ad optimization works best with the maximum event volume. Use a single pixel and add a `test_event_code` at develop-time for filtering.
- **Two pixels**: only if you have large-scale staging traffic that would pollute ad audiences.

### 3. Generate a Meta Conversions API access token

- [ ] In Events Manager, open the pixel you just created
- [ ] Settings → Conversions API → "Generate access token"
- [ ] Copy the token (long string, starts with `EAA`)
- [ ] Set as Supabase secret (do NOT paste into any env file):

```bash
supabase secrets set META_CAPI_TOKEN=EAAxxxxxxxxxxxxx --project-ref <staging-ref>
supabase secrets set META_PIXEL_ID=1234567890123456 --project-ref <staging-ref>

# Repeat for production:
supabase secrets set META_CAPI_TOKEN=EAAyyyyyyyyyyyyy --project-ref <production-ref>
supabase secrets set META_PIXEL_ID=1234567890123456 --project-ref <production-ref>
```

**Why these go to Supabase secrets, not env files**: the CAPI access token is a write credential to your ad account. It must never appear in a client bundle. The `VITE_` prefix is deliberately absent — the skill's code reads these from `Deno.env` inside the edge function, which only runs server-side.

### 4. Deploy the edge functions with the new secrets

```bash
npx supabase functions deploy create-checkout --project-ref <staging-ref>
npx supabase functions deploy stripe-webhook --project-ref <staging-ref>

# Then production:
npx supabase functions deploy create-checkout --project-ref <production-ref>
npx supabase functions deploy stripe-webhook --project-ref <production-ref>
```

### 5. Verify events are flowing — Meta Events Manager

- [ ] Events Manager → your pixel → **Test Events** tab
- [ ] Copy the test event code (format: `TEST12345`)
- [ ] **Note: to use test codes server-side**, add `test_event_code: 'TEST12345'` to the outer object in `sendMetaCAPIEvent`'s body (alongside `data` and `access_token`). This is a dev-time edit — remove it before deploying to production.
- [ ] Open the target SPA in a browser
- [ ] Trigger `Lead` — submit a contact form (or whichever form the Step 6 scan wired `trackLead` into)
- [ ] Trigger `InitiateCheckout` — click the pay button
- [ ] Trigger `Purchase` — complete a Stripe test checkout using `4242 4242 4242 4242` / `tok_visa`
- [ ] In the Test Events tab, for each event you should see **two entries that collapse into one**: one "Browser" (client Pixel) and one "Server" (CAPI), with a green "Deduplicated" indicator.
- [ ] If they don't dedupe, the `purchaseEventId` plumbing is broken. Verify:
  - `create-checkout` response includes the eventId
  - The success redirect URL has `?eid=...` in the query string
  - `stripe-webhook` reads `session.metadata.purchase_event_id`
  - The browser's `trackPurchase(eventId)` call passes the same ID

### 6. Verify events are flowing — GA4 DebugView

- [ ] GA4 admin → DebugView
- [ ] Open the app with `?debug_mode=1` appended to the URL (or set `window.gtag('set', 'debug_mode', true)` in the console before interacting)
- [ ] Trigger the same events as in step 5
- [ ] DebugView should show them in real time. If not, the `VITE_GA_MEASUREMENT_ID` env var is wrong or not loaded in the bundle.

### 7. Production smoke test

Once staging is verified:

- [ ] Deploy the SPA to production
- [ ] Open the live URL in an incognito window
- [ ] Repeat the browser smoke test (devtools → Network → filter `gtag|facebook.net`)
- [ ] Run a real Stripe transaction with a small amount (or use Stripe's `tok_visa` in live mode if available for your account)
- [ ] Watch Events Manager for the dedup indicator on the production pixel

## Compliance note

**This skill installs analytics without a cookie consent banner**, matching the reference project (dinner-matcher). This is a GDPR/ePrivacy risk if you serve users in the UK or EU.

Before shipping to production for EU/UK traffic, add one of:

1. **GA4 Consent Mode v2** with a compliant banner (e.g. CookieYes, Osano, Cookiebot) → GA starts firing only after the user accepts.
2. **Meta Pixel consent API**: call `fbq('consent', 'grant')` / `fbq('consent', 'revoke')` from your banner's state.
3. **Geo-block analytics** for EU/UK users entirely.

None of these are installed by this skill — the seam for wiring them is the `initAnalytics()` function in `src/lib/initAnalytics.ts` (dynamic pattern) or the raw script tags in `index.html` (static pattern).

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| No events in GA4 DebugView | Measurement ID env var not set at build time | Check `.env.staging`, re-run `npm run build`, confirm `%VITE_GA_MEASUREMENT_ID%` replaced in `dist/index.html` |
| Events appear in browser but not in Events Manager test tab | Wrong pixel ID | Verify `VITE_META_PIXEL_ID` matches the pixel you're looking at in Events Manager |
| CAPI events missing from test tab | `META_CAPI_TOKEN` not set as Supabase secret | `supabase secrets list --project-ref <ref>` to verify, re-deploy function after setting |
| Browser and CAPI events not dedup'd | `purchaseEventId` not passing through | Add `console.log` at each stage (create-checkout, webhook, browser) — whichever stage has a different value is the break |
| `fbq is not defined` in console | Dynamic pattern, Turnstile not verified yet | Expected during dev — the verification effect fires once Turnstile resolves. To force-fire in dev, set `VITE_TURNSTILE_SITE_KEY=` (empty) in `.env.local` |
| `VITE_META_PIXEL_ID` literally in HTML | Vite build ran without the env var set | Missing from `.env.<mode>` file or wrong filename — `.env.staging` not `.env.stg` |
