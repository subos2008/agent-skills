# Analytics (GA4 + Meta Pixel + Meta CAPI) Skill — Design Spec

## Problem

The dinner-matcher repo has a proven pattern for installing Google Analytics 4 (GA4) and Meta Pixel into React/Vite SPAs, plus a server-side Meta Conversions API (CAPI) integration that deduplicates against the client-side Pixel via a shared `purchaseEventId`. It also captures `fbclid` from ad-referral URLs, stores it in `sessionStorage`, reconstructs the `fbc` cookie format, and passes it through Stripe Checkout metadata so both browser and server events attribute back to the originating Meta ad.

The pattern is copy-pasted across SPAs (`onboarding`, `subscribers`, `homepage`). Each copy is slightly drifted. There is no single source of truth, no installer, and no documented integration contract with the project's Stripe edge functions.

## Solution

A new Claude Code skill — `analytics-ga-meta-capi` — installed into `~/Dropbox/agent-skills/skills/analytics-ga-meta-capi/`, that packages this pattern as a reusable installer for React/Vite + Supabase projects.

The skill is a companion to the existing `stripe-checkout-supabase` skill. It detects whether the Stripe edge functions exist and patches them with CAPI wiring if so. It is safe to run before or after `stripe-checkout-supabase`.

## Scope decisions (confirmed with user during brainstorming)

1. **Full stack**: client-side trackers (GA4 + Pixel) + server-side Meta CAPI + fbclid attribution plumbing. Not client-only.
2. **Two installation patterns**: skill asks at install time whether the target SPA uses the static pattern (script tags in `index.html`, fires on first byte — good for marketing/landing pages) or the dynamic pattern (`initAnalytics.ts`, fires after a gating hook like Turnstile verification — good for bot-filtered authenticated apps).
3. **No cookie consent banner**: matches the dinner-matcher repo exactly. A compliance note in the generated README flags the gap for EU/UK shipping.
4. **Generic event wrappers + starter set**: `trackEvent`, `trackPixelEvent` as generic primitives, plus pre-wired `trackLead()`, `trackInitiateCheckout()`, `trackPurchase()` that are deduped with server CAPI via `purchaseEventId`. No dinner-matcher-specific event catalog.
5. **Approach: single monolithic skill, per-SPA install**. Mirrors the structure of `stripe-checkout-supabase` exactly. User runs the skill once per SPA they want analytics on.

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

**Key invariant preserved by the skill**: the same `purchaseEventId` flows from `create-checkout` → Stripe metadata → `stripe-webhook` → redirect URL → browser Pixel `Purchase` event. Meta's dedup engine collapses the client Pixel fire and the server CAPI fire into one conversion.

**fbclid attribution**: `captureFbclid()` runs on app mount, reads `?fbclid=` from URL, stores in `sessionStorage` as `cju_fbclid` + `cju_fbclid_ts`, and `getFbc()` reconstructs the `fb.1.{ts}.{fbclid}` format. Both the client Pixel calls and the server CAPI calls pass this `fbc` value.

## File layout

Files created or edited in the target repo. `<app>` is the SPA selected at install time.

```
web-apps/<app>/
├── src/lib/
│   ├── analytics.ts                  [NEW]   GA4 wrapper: trackEvent(name, params), trackPageView(path)
│   ├── meta-pixel.ts                 [NEW]   Pixel wrapper: trackPixelEvent, trackLead,
│   │                                         trackInitiateCheckout, trackPurchase,
│   │                                         captureFbclid, getFbc
│   └── initAnalytics.ts              [NEW — dynamic pattern only]
│                                             Dynamic loader: appendChild of gtag.js + fbevents.js,
│                                             guards on env vars present, idempotent via module-level flag
├── src/App.tsx                       [EDIT — dynamic pattern only]
│                                             Add useEffect that calls initAnalytics() + captureFbclid()
├── index.html                        [EDIT — static pattern only]
│                                             Inject GA4 gtag + Meta Pixel snippets with
│                                             %VITE_GA_MEASUREMENT_ID% / %VITE_META_PIXEL_ID% placeholders
├── .env.staging                      [EDIT]  Add VITE_GA_MEASUREMENT_ID, VITE_META_PIXEL_ID
├── .env.production                   [EDIT]  Same (with different values)
└── .env.local                        [EDIT — if exists] Same, commented out by default

supabase/functions/
├── _shared/
│   └── meta-capi.ts                  [NEW if missing]
│                                             sendCapiEvent(eventName, eventId, userData, customData)
│                                             SHA256 hashing (email/phone/userId/city/country),
│                                             Graph API POST, error-tolerant (logs and moves on)
├── create-checkout/index.ts          [PATCH — only if file exists]
│                                             - Generate purchaseEventId (crypto.randomUUID())
│                                             - Add to Stripe session metadata
│                                             - Capture client IP + User-Agent from request headers
│                                             - Capture fbc from request body
│                                             - Send InitiateCheckout CAPI event
│                                             - Return purchaseEventId to client in response
├── stripe-webhook/index.ts           [PATCH — only if file exists]
│                                             - Read purchaseEventId from session.metadata
│                                             - Read fbc/IP/UA from session.metadata
│                                             - Send Purchase CAPI event (deduped by eventId)
│                                             - Optionally send CompleteRegistration CAPI event
└── .env                              [EDIT]  META_CAPI_ACCESS_TOKEN, META_PIXEL_ID
                                              (server secrets, set via `supabase secrets set`)
```

### Out of scope (user handles externally)

- GA4 property creation (setup checklist walks through Google Analytics UI)
- Meta Pixel creation (setup checklist walks through Meta Business Manager)
- Meta CAPI access token generation (setup checklist walks through)
- Cookie consent banner (explicitly omitted, matching repo behavior — compliance note in README)
- Event catalog beyond `Lead` / `InitiateCheckout` / `Purchase` (Step 6 scan proposes some, user wires in the rest)

### Patch strategy for Stripe edge functions

The skill uses **anchored insertions**, not regex replacement. It looks for specific sentinel patterns (`stripe.checkout.sessions.create({`, `metadata: {`, the webhook's `case 'checkout.session.completed':` block) and inserts new lines before/after them. If a sentinel isn't found (user has hand-rolled a different Stripe setup), the skill stops and prints the exact snippet to paste manually rather than corrupting the file.

## Skill flow

When invoked, the skill runs these steps in order. Each step is described in `SKILL.md`; full code templates live in `references/*.md`.

### Step 0 — Discovery probe (read-only)

Detect repo layout: find `web-apps/*` SPAs, find `supabase/functions/`, check for `_shared/meta-capi.ts`, check for `create-checkout`, `stripe-webhook`. Print a short summary.

### Step 1 — Gather requirements (questions, one at a time)

- Which SPA? Presents the list discovered in Step 0 (e.g., on dinner-matcher: `homepage`, `onboarding`, `subscribers`, `admin`). User picks one, or provides a custom path if the repo doesn't follow the `web-apps/*` convention.
- Which install pattern? `static` or `dynamic` — recommend based on whether the selected SPA has Turnstile / auth gating
- Do you already have GA4 measurement ID and Meta Pixel ID, or should the setup checklist walk you through creating them?
- Install server-side CAPI now? (Yes if `create-checkout` + `stripe-webhook` exist, otherwise offer "client-only now, re-run later")

### Step 2 — Generate client-side files

Write `src/lib/analytics.ts`, `src/lib/meta-pixel.ts`, and (dynamic pattern only) `src/lib/initAnalytics.ts`.

- **Static pattern**: insert GA4 + Pixel `<script>` snippets into `index.html` with `%VITE_*%` placeholders.
- **Dynamic pattern**: insert `initAnalytics()` + `captureFbclid()` `useEffect` into `App.tsx` (anchored at the existing Turnstile verification effect if present, otherwise at the end of the component).

### Step 3 — Generate `_shared/meta-capi.ts` (if missing)

Write the CAPI helper with SHA256 user-data hashing, Graph API POST, fail-open error handling.

### Step 4 — Patch Stripe edge functions (only if they exist)

- `create-checkout/index.ts`: insert `purchaseEventId` generation, metadata fields, IP/UA/fbc capture from request, `InitiateCheckout` CAPI call, response field.
- `stripe-webhook/index.ts`: insert `Purchase` CAPI call (reads metadata, dedups by eventId), optional `CompleteRegistration` call.
- Anchored inserts with sentinel matching — abort cleanly if file structure doesn't match and print manual-paste snippets.

### Step 5 — Env vars and secrets

- Add `VITE_GA_MEASUREMENT_ID` + `VITE_META_PIXEL_ID` to `.env.staging`, `.env.production`, and `.env.local` (commented).
- Print `supabase secrets set` commands for `META_CAPI_ACCESS_TOKEN` + `META_PIXEL_ID` (do not run — user executes).

### Step 6 — Domain event scan (interactive)

Grep the target SPA for likely event hook points using a curated pattern list:

| Pattern | Signal | Suggested event |
|---|---|---|
| `onSubmit=` on `<form>` | form submission | `trackLead()` + `trackEvent('form_submitted')` |
| `onClick=` with text matching `/sign ?up\|get started\|start\|join\|subscribe/i` | CTA click | `trackEvent('cta_click')` |
| `useNavigate()` / `navigate(` inside button handlers | funnel navigation | `trackEvent('step_completed')` |
| `supabase.auth.signIn` / `signUp` / `signInWithOtp` | auth event | `trackEvent('login_started')` / `login_completed` |
| `supabase.functions.invoke('create-checkout'` | already handled by plumbing | skip |
| Welcome/success pages (JSX text matching "welcome", "thank you", "success") | purchase confirmation | already wired if CAPI installed |
| Toggle/opt-in patterns (`setXEnabled(true)`, `update({ opted_in: true })`) | engagement | `trackEvent('engagement')` |

Rank suggestions by confidence, deduplicate to one per `file:line`. Present an interactive list:

```
Found 7 candidate event hook points. For each, I'll show the code and propose an event name.
You say y/n/rename. Nothing is edited until you confirm.

[1/7] web-apps/onboarding/src/steps/ContactStep.tsx:42
      <form onSubmit={handleContactSubmit}>
      Suggested: trackLead() + trackEvent('contact_submitted')
      y / n / rename: _
```

For each `y`, make an anchored insertion into the handler. For `rename`, prompt for a new event name and use that. For `n`, skip. At the end, print a summary: "Wired 4 events across 3 files. Skipped 3."

**Design notes for Step 6:**

- Runs **last** because plumbing comes before application-specific instrumentation. If the scan fails or finds nothing, the rest of the install is already done.
- **Additive only** — never removes or modifies existing tracking code, only wraps handlers.
- `n` is the safe default; the skill errs toward not wiring events it's unsure about.
- Event names default to **GA4 style** (`snake_case`, verb-noun). The scan also calls `trackPixelEvent` with the same name so both backends see it, unless the suggestion maps to a standard Meta event (`Lead`, `InitiateCheckout`, `Purchase`) in which case it uses Meta's capitalization.
- Only runs against the **currently targeted SPA**, not the whole repo.

### Step 7 — Setup checklist (printed, not executed)

- Create GA4 property → paste measurement ID into env files
- Create Meta Pixel → paste pixel ID
- Generate Meta CAPI access token → `supabase secrets set META_CAPI_ACCESS_TOKEN=...`
- Verify in Meta Events Manager → confirm client Pixel and CAPI events match on `eventId`
- Compliance note: *"This skill matches dinner-matcher's setup, which has no cookie consent banner. Add one before shipping to EU/UK users."*

## Execution tracking

A 7-step skill that edits existing code needs structure. Three mechanisms, combined, keep execution on rails:

### Mechanism 1 — TaskCreate at entry (primary)

First instruction in `SKILL.md`: *"Run Step 0 first (discovery probe is read-only, produces the SPA list and Stripe-function presence needed by Step 1). Then create a TaskCreate entry for each of Steps 1–7. Mark each `in_progress` when you start it and `completed` when done. Do not mark complete without verification."*

This is the harness's native mechanism, visible to the user in the CLI, and survives within a session.

### Mechanism 2 — Hard checkpoints at risky transitions (secondary)

Explicit STOP markers in `SKILL.md` at three high-cost-of-failure points:

1. **After Step 2 (client files generated) — before Step 3 (server-side work begins)**: stop, show what was written, ask "proceed with server CAPI wiring?" User says yes/no. Client-only installs exit cleanly here.
2. **Before Step 4 (Stripe function patches)**: stop, show each planned edit as a diff preview, ask for explicit confirmation. Highest-risk step — edits existing code. If any sentinel anchor is missing, abort the **patch only** (not the whole skill) and fall through to Scenario C behavior: print paste-me snippets, mark Step 4 as completed-with-manual-action, proceed to Step 5.
3. **Before Step 6 (domain event scan wiring)**: meta-checkpoint before the scan starts: "I'm about to grep the app for event hook points. This will read ~N files. Continue?"

### Mechanism 3 — Verification before task completion (tertiary)

Step completion is evidence-based, not declarative. Each step has a verification the skill must perform before marking its task `completed`:

| Step | Verification |
|---|---|
| 2 (client files) | Read back each generated file, confirm `export function trackEvent` etc. present |
| 3 (CAPI helper) | Read `_shared/meta-capi.ts`, grep for `sendCapiEvent` export |
| 4 (Stripe patches) | Grep patched files for inserted anchor content (e.g. `sendCapiEvent('InitiateCheckout'`) |
| 5 (env vars) | Grep env files for the new `VITE_GA_MEASUREMENT_ID` / `VITE_META_PIXEL_ID` lines |
| 6 (domain events) | For each `y`-confirmed event, grep the target file for the wrapped call |

If verification fails, the task stays `in_progress`, the skill prints the mismatch, and asks whether to retry or abort.

**Rejected mechanism: state file.** A `.claude/analytics-skill-state.json` was considered for cross-session resume and idempotency but rejected. The skill runs once per SPA in normal use — resume-across-sessions is a theoretical benefit unlikely to fire. Idempotency is better served by Step 0 probing the filesystem directly. State files create a second source of truth that drifts from reality.

## Reference file breakdown

`SKILL.md` is the orchestration guide; code templates live in `references/*.md` so they load only when needed.

```
skills/analytics-ga-meta-capi/
├── SKILL.md                          Orchestration guide (~200 lines)
│                                     - Frontmatter (name, description)
│                                     - Architecture diagram
│                                     - TaskCreate instruction
│                                     - Steps 0-7 with inline STOP markers
│                                     - Pointers to references/
│
└── references/
    ├── client-wrappers.md            analytics.ts, meta-pixel.ts templates
    │                                 (~250 lines — full file contents)
    │
    ├── init-patterns.md              Both installation patterns:
    │                                 - Static: index.html snippets
    │                                 - Dynamic: initAnalytics.ts + App.tsx edit
    │                                 (~150 lines)
    │
    ├── server-capi.md                _shared/meta-capi.ts full template
    │                                 + SHA256 hashing explanation
    │                                 + Graph API endpoint + payload shape
    │                                 + Fail-open error handling rationale
    │                                 (~250 lines)
    │
    ├── stripe-integration.md         Exact patches for create-checkout and
    │                                 stripe-webhook:
    │                                 - Sentinel anchors
    │                                 - Full before/after snippets
    │                                 - purchaseEventId flow diagram
    │                                 - fbc/IP/UA metadata plumbing
    │                                 (~350 lines)
    │
    ├── event-scan-patterns.md        Grep patterns for Step 6:
    │                                 - onSubmit forms, CTA onClick
    │                                 - Auth calls, navigation handlers
    │                                 - Pattern → suggested event name mapping
    │                                 - Naming conventions (GA4 snake_case vs Meta standard)
    │                                 (~150 lines)
    │
    └── setup-checklist.md            User-facing setup tasks:
                                      - Create GA4 property (steps)
                                      - Create Meta Pixel (steps)
                                      - Generate Meta CAPI access token (steps)
                                      - supabase secrets set commands
                                      - Verification in Meta Events Manager
                                      - Compliance note re: cookie consent gap
                                      (~120 lines)
```

Total: SKILL.md + 6 reference files, ~1300–1500 lines. Same ballpark as `stripe-checkout-supabase` (1275 lines).

**Rationale for the split:**

- `SKILL.md` loads only flow + architecture — cheap to read at skill invocation, doesn't pollute context with code until needed.
- One reference file per "layer" — client wrappers, init patterns, server CAPI, Stripe integration, scan patterns, setup. Each file is what Claude needs for a specific step. Step 2 loads `client-wrappers.md` + `init-patterns.md`. Step 4 loads `stripe-integration.md`. Step 6 loads `event-scan-patterns.md`. No step needs all of them at once.
- **Why `stripe-integration.md` is separate from `server-capi.md`**: the CAPI helper is standalone and reusable (just a function). The Stripe patches are the wiring between that function and existing edge functions — fragile because they touch existing code. Keeping them in a dedicated file makes "anchors broke" easy to debug.
- **Why `event-scan-patterns.md` exists**: the scan patterns are a list that might get tuned over time (new frameworks, new conventions). Keeping them in a reference file means tuning doesn't touch the main guide.

## Soft-dependency contract with `stripe-checkout-supabase`

`stripe-checkout-supabase` creates `create-checkout` and `stripe-webhook`. `analytics-ga-meta-capi` patches them with CAPI calls. The first skill knows nothing about the second. The second skill knows the first exists and detects whether its output is present.

### Detection, not coupling

At Step 0 the skill runs three existence checks, nothing more:

```
supabase/functions/_shared/meta-capi.ts       -> present? skip generation, else create
supabase/functions/create-checkout/index.ts   -> present? enable Step 4 patch, else skip
supabase/functions/stripe-webhook/index.ts    -> present? enable Step 4 patch, else skip
```

No version checks, no AST parsing, no "is this dinner-matcher's stripe-checkout-supabase or someone else's?" Just: *does the file exist?* If yes, the sentinel-anchored patch is attempted. If anchors don't match, the patch aborts cleanly and prints a manual snippet.

### Install scenarios

| Scenario | `create-checkout` exists? | Skill behavior |
|---|---|---|
| **A. Fresh project, no Stripe yet** | No | Install client side + `_shared/meta-capi.ts`. Print: *"No Stripe functions detected. Install `stripe-checkout-supabase` next, then re-run this skill to wire CAPI events."* Exit cleanly at Step 3. |
| **B. Stripe already installed by `stripe-checkout-supabase`** | Yes, anchors match | Full install: client + CAPI helper + Stripe patches. |
| **C. Stripe installed but hand-rolled (anchors don't match)** | Yes, anchors don't match | Install client + CAPI helper. Print diff snippets for Stripe files and ask user to paste manually. Mark Step 4 as completed-with-manual-action rather than failed. |

The skill is safe to run on any project, in any order relative to `stripe-checkout-supabase`. The cost of getting the order wrong is a re-run, not a broken install.

### Forward compatibility

Note in `SKILL.md`: *"If `stripe-checkout-supabase` changes the structure of `create-checkout/index.ts` in a future version, this skill's anchors may break. When that happens, update `references/stripe-integration.md` with new anchors. The skill still aborts cleanly on old installs."*

### Deliberate non-feature

The skill does **not** modify `stripe-checkout-supabase`'s own reference files to add CAPI. The two skills stay independently installable — `stripe-checkout-supabase` users who don't want Meta tracking should never see CAPI code in their Stripe files, so CAPI lives only in a second, opt-in skill invocation.

## Verification

### Skill-level validation (before publishing)

Before the skill is considered ready to ship in `~/Dropbox/agent-skills`:

1. **Smoke test on a fresh SPA** — create a throwaway Vite React + Supabase project, run the skill end-to-end, verify Steps 0–7 all complete and the generated code type-checks (`tsc --noEmit`) and lints.
2. **Smoke test on dinner-matcher itself** — run the skill in a worktree against `web-apps/admin` (which currently has no analytics) and verify the output matches the existing `onboarding` / `subscribers` pattern.
3. **Stripe patch anchor verification** — point the skill at dinner-matcher's real `create-checkout/index.ts` and `stripe-webhook/index.ts`, confirm sentinel anchors match. If anchors don't survive on the reference repo, they won't survive anywhere.
4. **Scenario A and C rehearsal** — temporarily move `create-checkout` aside and re-run to verify the skill exits cleanly with the "install Stripe first" message. Mangle an anchor to verify the manual-paste fallback fires.

Documented as a short `TESTING.md` at the skill root. Manual checks, not automated tests.

### Installed-output verification (what the skill tells the user to do)

After the skill finishes, `setup-checklist.md` and the skill's final printed message tell the user to run this verification sequence:

**1. Build + typecheck**
```
cd web-apps/<app>
npm run build
```
If `tsc` fails, the generator is buggy — report it.

**2. Local smoke test**
- `npm run dev`, open the app.
- Devtools Network tab, filter to `gtag` and `facebook.net`. Confirm both scripts load (dynamic pattern: confirm they load *after* the gating condition).
- Console: `window.gtag` and `window.fbq` should both be functions.
- Visit `?fbclid=test123`, confirm `sessionStorage.getItem('cju_fbclid')` returns `test123`.

**3. Meta Events Manager test-events tab**
- In Meta Events Manager → Test Events, enter the test code, open the app.
- Trigger `Lead` (submit a form), `InitiateCheckout` (click pay), `Purchase` (complete test Stripe session using `tok_visa`).
- For each event, Events Manager should show **two rows that collapse into one** — one labeled "Browser" (client Pixel) and one labeled "Server" (CAPI), with a green "Deduplicated" badge. If they don't dedupe, the `eventId` plumbing is broken — check that `purchaseEventId` flows through Stripe metadata correctly.

**4. GA4 DebugView**
- In GA4 admin → DebugView, with the app in debug mode (`?debug_mode=1` or `window.gtag('set', 'debug_mode', true)`).
- Trigger events, confirm they appear in the real-time stream.

**5. Production smoke test**
- After deploying the SPA, repeat the browser smoke test on the live URL.
- Re-run a small test purchase on the real Stripe environment and watch Events Manager confirm dedup on the prod Pixel.

### What the skill does NOT verify automatically

- None of the above is automated — all require a browser, a real Meta account, and a real Stripe transaction. The skill prints the checklist.
- The skill does not validate that the GA4 property ID and Pixel ID are real at install time. If the user pastes `G-WRONGID` into env files, no events flow and the only signal is "nothing shows up in GA4." The setup checklist calls this out as a failure mode to check for.

## Summary

A single monolithic skill, `analytics-ga-meta-capi`, packages dinner-matcher's proven GA4 + Meta Pixel + Meta CAPI pattern as a reusable per-SPA installer. It mirrors `stripe-checkout-supabase`'s structure (SKILL.md + 6 reference files), supports both static and dynamic installation patterns, handles both fresh and Stripe-already-installed projects via detection-based soft dependency, and includes an interactive domain-event scan that walks the target SPA's code and proposes tracking hook points. Execution is kept on rails by TaskCreate tracking + three hard checkpoints + evidence-based step completion. Cookie consent banner is deliberately out of scope (matching the repo), and the gap is called out in the generated README.

The terminal state of this spec is handoff to the `writing-plans` skill to produce an implementation plan.
