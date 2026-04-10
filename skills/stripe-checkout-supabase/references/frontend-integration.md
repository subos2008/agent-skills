# Frontend integration

Two hooks — one for the onboarding SPA, one for the subscriber SPA. Both assume the project has `supabase` exported from `src/lib/supabase.ts` (the usual `createClient(url, anonKey)` pattern).

## Onboarding: `useCheckout.ts`

Put this in `web-apps/{onboarding}/src/hooks/useCheckout.ts`. Call `startCheckout()` from the final onboarding step (after all the questionnaire data has been saved to `users` / `question_answers`). It signs the user in anonymously if they are not already signed in, invokes `create-checkout`, and redirects the browser to the returned Stripe URL.

```typescript
import { useState, useCallback } from 'react'
import { supabase } from '../lib/supabase'

interface CheckoutParams {
  email: string
  phone?: string
  plan: string        // e.g., '1m' | '3m' | '6m'
  eventId?: string    // for Meta Pixel dedup, optional
  fbc?: string | null // Facebook click ID, optional
}

export function useCheckout() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const startCheckout = useCallback(async (params: CheckoutParams): Promise<boolean> => {
    setError(null)
    setLoading(true)

    try {
      // Ensure we have a JWT to send. If the project uses anonymous auth,
      // sign in now if there is no session yet. (No-op if already signed in.)
      const { data: sessionData } = await supabase.auth.getSession()
      if (!sessionData.session) {
        const { error: anonError } = await supabase.auth.signInAnonymously()
        if (anonError) throw anonError
      }

      const { data, error: fnError } = await supabase.functions.invoke(
        'create-checkout',
        { body: params },
      )

      if (fnError) {
        // The function returns 409 with { error: 'account_exists' } if the
        // submitted email / phone is already tied to another user.
        const errorBody = data?.error ?? fnError.message ?? ''
        if (errorBody.includes('account_exists')) {
          setError('account_exists')
          return false
        }
        throw new Error(errorBody)
      }

      if (data?.url) {
        window.location.href = data.url
        return true
      }

      throw new Error('No checkout URL returned')
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong')
      return false
    } finally {
      setLoading(false)
    }
  }, [])

  const clearError = useCallback(() => setError(null), [])

  return { startCheckout, loading, error, clearError }
}
```

### How it fits into the onboarding flow

Typical wiring in the final onboarding step:

```typescript
const { startCheckout, loading, error } = useCheckout()

const onPayClick = async () => {
  // 1. Save all onboarding data first (update users + question_answers rows)
  const saved = await saveOnboardingData(formData)
  if (!saved) return

  // 2. Kick off checkout
  await startCheckout({
    email: formData.email,
    phone: formData.phone,
    plan: formData.selectedPlan,
  })
}
```

Handle `error === 'account_exists'` in the UI with a message like "You already have an account — log in instead" and a link to the subscriber app.

## Subscribers: `useStripeActions.ts`

Put this in `web-apps/{subscribers}/src/hooks/useStripeActions.ts`. Provides two actions for logged-in subscribers:

- `manageSubscription(customerId)` — opens the Stripe billing portal so the user can update card, view invoices, cancel
- `resubscribe(userId, email)` — kicks off a new checkout session for a user whose subscription was previously cancelled

```typescript
import { useState } from 'react'
import { supabase } from '../lib/supabase'

export function useStripeActions() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const manageSubscription = async (stripeCustomerId: string) => {
    setLoading(true)
    setError(null)
    try {
      const { data, error: fnError } = await supabase.functions.invoke(
        'create-portal-session',
        { body: { customerId: stripeCustomerId } },
      )
      if (fnError) {
        const context = data?.error ?? fnError.message
        throw new Error(context)
      }
      if (data?.url) {
        window.location.href = data.url
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong')
    } finally {
      setLoading(false)
    }
  }

  const resubscribe = async (userId: string, email: string | null) => {
    setLoading(true)
    setError(null)
    try {
      const { data, error: fnError } = await supabase.functions.invoke(
        'create-checkout',
        {
          body: {
            userId,
            email,
            cancelUrl: window.location.origin,
          },
        },
      )
      if (fnError) {
        const context = data?.error ?? fnError.message
        throw new Error(context)
      }
      if (data?.url) {
        window.location.href = data.url
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Something went wrong')
    } finally {
      setLoading(false)
    }
  }

  const clearError = () => setError(null)

  return { manageSubscription, resubscribe, loading, error, clearError }
}
```

### Typical wiring in a subscriber Account screen

```tsx
function AccountTab({ user }: { user: UserRow }) {
  const { manageSubscription, resubscribe, loading } = useStripeActions()

  if (user.subscription_status === 'active') {
    return (
      <button
        onClick={() => manageSubscription(user.stripe_customer_id!)}
        disabled={loading}
      >
        Manage billing
      </button>
    )
  }

  return (
    <button
      onClick={() => resubscribe(user.id, user.email)}
      disabled={loading}
    >
      Resubscribe
    </button>
  )
}
```

## Detecting success on return

After a successful checkout, Stripe redirects to:

```
{SUBSCRIBER_APP_URL}/?welcome=true&email={email}&eid={purchaseEventId}
```

In the subscriber SPA root, check for `?welcome=true` on mount to trigger a welcome UI (confetti, tour, whatever). The `eid` query param is the `purchase_event_id` from the Stripe session metadata — use it as the Meta Pixel `eventID` if you want to dedupe the browser Purchase pixel with the server-side CAPI Purchase event.

```typescript
useEffect(() => {
  const params = new URLSearchParams(window.location.search)
  if (params.get('welcome') === 'true') {
    const purchaseEventId = params.get('eid') ?? undefined
    // fire browser Purchase pixel with eventID: purchaseEventId
    // show welcome screen
  }
}, [])
```

Note that the user may land here before the webhook has finished running (Stripe's redirect and webhook are independent). If the UI gates on `subscription_status === 'active'`, either poll briefly on the welcome screen until it flips, or trust the `?welcome=true` flag as "payment succeeded" and refetch after a short delay.
