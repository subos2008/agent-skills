---
name: analytics-ga-meta-capi
description: Install Google Analytics 4, Meta Pixel, and server-side Meta Conversions API (CAPI) into a React/Vite SPA + Supabase project. Generates client wrappers, a portable CAPI helper, anchored patches for existing Stripe edge functions (create-checkout + stripe-webhook), and an interactive scan that walks the target SPA and proposes domain event hook points. Supports both static (index.html script tags) and dynamic (initAnalytics.ts after Turnstile verification) installation patterns. Preserves the client+server dedup invariant via a shared purchaseEventId and fbclid → fbc attribution plumbing. Safe to run before or after stripe-checkout-supabase — detects Stripe functions and patches cleanly or skips with instructions. Use when the user wants to add GA4 / Meta Pixel / ad conversion tracking to a subscription SaaS built on Supabase + Stripe, mentions Meta CAPI, Conversions API, pixel deduplication, fbclid, or "making my ad dashboard work".
---

# Analytics: GA4 + Meta Pixel + Meta CAPI

Installs dinner-matcher's proven analytics pattern into a React/Vite SPA backed by Supabase. Three layers: client-side GA4 + Pixel wrappers, server-side Meta Conversions API helper, and anchored patches into Stripe edge functions that wire CAPI events onto checkout success.

## Architecture

```
Browser                                    Supabase Edge Functions
-------                                    -----------------------
index.html OR initAnalytics.ts
  |
  +-- gtag.js (GA4)              ----->    N/A (client only)
  +-- fbevents.js (Meta Pixel)   ----->    N/A (client only)
  |
src/lib/analytics.ts                       (GA4 event wrapper)
src/lib/meta-pixel.ts                      (Pixel wrapper + fbclid capture + fbc reconstruct)
  |
  | trackLead() on form submit        -->  client Pixel `Lead` event
  | trackInitiateCheckout() on pay    -->  client Pixel `InitiateCheckout`
  |                                           |
  |                                           v
  |                                        _shared/meta-capi.ts   (server CAPI helper)
  |                                           |
  |                                      create-checkout (patched)
  |                                         - generates purchaseEventId (UUID)
  |                                         - stores in Stripe metadata
  |                                         - sends InitiateCheckout to CAPI
  |                                           |
  |                                           v
  |                                      stripe-webhook (patched)
  |                                         - reads purchaseEventId from metadata
  |                                         - sends Purchase to CAPI with same eventId
  |
  | <-- success redirect with ?eid=<purchaseEventId>
  v
trackPurchase(eventId) on confirmation --> client Pixel `Purchase` (deduped by eventId)
```

**Key invariant**: the same `purchaseEventId` flows from `create-checkout` → Stripe metadata → `stripe-webhook` → redirect URL → browser Pixel `Purchase` event. Meta's dedup engine collapses the client Pixel fire and the server CAPI fire into one conversion.

## What gets installed

| Layer | File | Purpose |
|---|---|---|
| Client | `web-apps/<app>/src/lib/analytics.ts` | GA4 `trackEvent` wrapper |
| Client | `web-apps/<app>/src/lib/meta-pixel.ts` | Pixel wrapper + `trackLead/InitiateCheckout/Purchase` + `captureFbclid/getFbc` |
| Client | `web-apps/<app>/src/lib/initAnalytics.ts` | *(Dynamic pattern only)* — loads gtag/fbevents scripts after a verification gate |
| Client | `web-apps/<app>/index.html` | *(Static pattern only)* — inline `<script>` tags with `%VITE_*%` placeholders |
| Client | `web-apps/<app>/src/App.tsx` | *(Dynamic pattern only)* — `useEffect` calls for `captureFbclid()` and `initAnalytics()` |
| Client | `.env.staging` / `.env.production` | `VITE_GA_MEASUREMENT_ID`, `VITE_META_PIXEL_ID` |
| Server | `supabase/functions/_shared/meta-capi.ts` | `sendMetaCAPIEvent`, `buildUserData`, `sha256` |
| Server | `supabase/functions/create-checkout/index.ts` | *(Patch)* `InitiateCheckout` CAPI call + metadata plumbing |
| Server | `supabase/functions/stripe-webhook/index.ts` | *(Patch)* `Purchase` + `CompleteRegistration` CAPI calls |

## EXECUTION RULES — READ BEFORE STARTING

**Mechanism 1 — TaskCreate at entry.** Run Step 0 first (discovery probe is read-only, produces the SPA list and Stripe-function presence needed by Step 1). Then create a TaskCreate entry for each of Steps 1–7. Mark each `in_progress` when you start it and `completed` when done. Do not mark complete without verification.

**Mechanism 2 — Hard STOP checkpoints.** Three explicit stops:

1. **After Step 2, before Step 3** (server-side work begins): show what was written, ask *"proceed with server CAPI wiring?"* User says yes/no. Client-only installs exit cleanly here.
2. **Before Step 4** (Stripe function patches): show each planned edit as a diff preview, ask for explicit confirmation. If any sentinel anchor is missing, abort the **patch only** (not the whole skill), fall through to Scenario C: print paste-me snippets, mark Step 4 as completed-with-manual-action, proceed to Step 5.
3. **Before Step 6** (domain event scan): *"I'm about to grep the app for event hook points. This will read ~N files. Continue?"*

**Mechanism 3 — Evidence-based step completion.** Do not mark a step `completed` without grepping / reading the result to prove it landed:

| Step | Verification |
|---|---|
| 2 | Read each generated file, confirm `export function trackEvent` etc. present |
| 3 | Read `_shared/meta-capi.ts`, grep for `sendMetaCAPIEvent` export |
| 4 | Grep patched files for `sendMetaCAPIEvent('InitiateCheckout')` / `'Purchase'` |
| 5 | Grep env files for `VITE_GA_MEASUREMENT_ID=` and `VITE_META_PIXEL_ID=` |
| 6 | For each `y`-confirmed event, grep the target file for the wrapped call |

If verification fails, leave the task `in_progress`, print the mismatch, and ask whether to retry or abort.

## Step 0 — Discovery probe (read-only)

Detect the repo layout. Run these checks:

```bash
# SPA discovery
ls -d web-apps/*/ 2>/dev/null
# Expect: list of SPA directories. If the repo doesn't use web-apps/*, ask the user for custom paths.

# Stripe function detection
test -f supabase/functions/create-checkout/index.ts && echo "create-checkout: present" || echo "create-checkout: missing"
test -f supabase/functions/stripe-webhook/index.ts && echo "stripe-webhook: present" || echo "stripe-webhook: missing"
test -f supabase/functions/_shared/meta-capi.ts && echo "meta-capi.ts: present" || echo "meta-capi.ts: missing"
```

Print a summary:

```
Discovery:
  SPAs found: homepage, onboarding, subscribers, admin
  create-checkout: present
  stripe-webhook: present
  meta-capi.ts: missing (will generate)
```

## Step 1 — Gather requirements

Ask the user, one question at a time:

1. **Which SPA?** — present the list from Step 0. User picks one, or provides a custom path.
2. **Which install pattern?** — `static` (script tags in `index.html`, fires on first byte) or `dynamic` (`initAnalytics.ts`, fires after verification hook). Recommend based on heuristics from `references/init-patterns.md` (Turnstile / auth-gate detection).
3. **Do you already have GA4 measurement ID and Meta Pixel ID?** — if yes, capture them now. If no, the setup checklist walks through creation.
4. **Install server-side CAPI now?** — if `create-checkout` + `stripe-webhook` both present in Step 0: yes/no (default yes). If either missing: skip, offer "client-only now, re-run later".

Create the Task entries for Steps 1–7 now.

## Step 2 — Generate client-side files

Load: `references/client-wrappers.md` + `references/init-patterns.md`

- Write `web-apps/<app>/src/lib/analytics.ts` (from `client-wrappers.md`)
- Write `web-apps/<app>/src/lib/meta-pixel.ts` (from `client-wrappers.md`)
- If **dynamic pattern**: write `web-apps/<app>/src/lib/initAnalytics.ts` (from `init-patterns.md` Pattern B), then edit `src/App.tsx` to add the `captureFbclid()` + `initAnalytics()` `useEffect` hooks.
- If **static pattern**: edit `web-apps/<app>/index.html` to inject the GA4 + Pixel `<script>` snippets from `init-patterns.md` Pattern A.

**Verification**: read each new/edited file, confirm the expected exports / snippets are present.

**STOP CHECKPOINT 1** — Show the user what was generated. Ask: *"Proceed with server-side CAPI wiring?"* If no, exit cleanly (Step 5 env vars still runs for client-side; then skip to Step 6). If yes, continue.

## Step 3 — Generate `_shared/meta-capi.ts` (if missing)

Load: `references/server-capi.md`

If `supabase/functions/_shared/meta-capi.ts` exists from a previous install, skip this step. Otherwise write the file from `server-capi.md`.

**Verification**: grep the new file for `export async function sendMetaCAPIEvent`, `export async function buildUserData`, `META_CAPI_TOKEN`.

## Step 4 — Patch Stripe edge functions

Load: `references/stripe-integration.md`

**STOP CHECKPOINT 2** — Before touching the Stripe files, show the planned edits as diff previews. Get explicit user confirmation.

Apply patches in order: A1 → A2 → A3 → A4 → A5 → A6 on `create-checkout`, then B1 → B2+B3 on `stripe-webhook`. Each patch re-reads the file before its edit.

**If any anchor doesn't match** — abort the patch for *that file only* (continue the other file), print paste-me snippets, mark this step as "completed-with-manual-action" in the task.

**Verification**: grep each patched file for `sendMetaCAPIEvent` and `purchase_event_id`. Run `deno check` on each patched file.

## Step 5 — Env vars and secrets

- Add to `web-apps/<app>/.env.staging`:
  ```
  VITE_GA_MEASUREMENT_ID=G-XXXXXXXXXX
  VITE_META_PIXEL_ID=1234567890123456
  ```
- Add to `web-apps/<app>/.env.production` with different values (or the same, user's choice).
- Add to `web-apps/<app>/.env.local` as **commented-out** lines (`# VITE_GA_MEASUREMENT_ID=G-...`) so the target developer knows they exist.

- Print (**do not run**) the Supabase secrets commands for CAPI:
  ```bash
  supabase secrets set META_CAPI_TOKEN=<token> --project-ref <ref>
  supabase secrets set META_PIXEL_ID=<id> --project-ref <ref>
  ```

**Verification**: grep env files for `VITE_GA_MEASUREMENT_ID=` and `VITE_META_PIXEL_ID=`.

## Step 6 — Domain event scan (interactive)

Load: `references/event-scan-patterns.md`

**STOP CHECKPOINT 3** — *"I'm about to grep `web-apps/<app>/src/` for event hook points. Continue?"*

1. Run each grep pattern from `event-scan-patterns.md`
2. Collect `file:line` matches, deduplicate, rank by priority (P0 → P1 → P2)
3. Present one suggestion at a time in the format from the reference file
4. For each `y`, make an anchored insertion + add imports
5. For each `rename`, prompt for a new event name
6. For each `n`, skip
7. Final summary: *"Wired K events across M files. Skipped L. Renamed R."*

**Verification**: for each `y`-confirmed event, grep the target file for the new wrapper call.

## Step 7 — Setup checklist (print, do not execute)

Load: `references/setup-checklist.md`

Print the full checklist contents. Do not execute any of its steps — the user works through them manually (Google Analytics UI, Meta Business Manager, `supabase secrets set`).

Include the compliance note about missing cookie consent as the closing line.

Mark Task 7 completed.

## Reference files

| File | Loaded by |
|---|---|
| [`references/client-wrappers.md`](references/client-wrappers.md) | Step 2 |
| [`references/init-patterns.md`](references/init-patterns.md) | Step 2 |
| [`references/server-capi.md`](references/server-capi.md) | Step 3 |
| [`references/stripe-integration.md`](references/stripe-integration.md) | Step 4 |
| [`references/event-scan-patterns.md`](references/event-scan-patterns.md) | Step 6 |
| [`references/setup-checklist.md`](references/setup-checklist.md) | Step 7 |

## Forward compatibility

If `stripe-checkout-supabase` changes the structure of `create-checkout/index.ts` in a future version, this skill's anchors may break. When that happens, update `references/stripe-integration.md` with new anchors. The skill still aborts cleanly on old installs — no silent corruption.
