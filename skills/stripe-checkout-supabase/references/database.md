# Database migration

The webhook needs two fields on the `users` table: the subscription state, and the Stripe customer ID. The application uses `subscription_status` for access control (RLS policies check it, the frontend gates features on it), and `stripe_customer_id` for opening the billing portal.

## If you are creating the `users` table for the first time

Embed the columns directly in the initial schema. This is how dinner-matcher does it (see `supabase/migrations/00001_initial_schema.sql`):

```sql
CREATE TYPE subscription_status AS ENUM ('active', 'cancelled', 'paused', 'incomplete');

CREATE TABLE users (
  id                    uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  email                 text,
  phone                 text,
  subscription_status   subscription_status NOT NULL DEFAULT 'incomplete',
  stripe_customer_id    text,
  -- ... your other columns
  created_at            timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_subscription_status ON users(subscription_status);
```

The `id` column references `auth.users(id)` so the row is automatically deleted when the Supabase Auth user is deleted. `DEFAULT 'incomplete'` means a row created during onboarding (before payment) correctly reflects that the user has not paid yet.

## If you are adding Stripe to an existing project

Create a new migration file named like `NNNNN_stripe_subscriptions.sql` (use the next available number in `supabase/migrations/`):

```sql
-- Add subscription status + Stripe customer ID to existing users table.
-- Safe to re-run — all guards use IF NOT EXISTS.

DO $$
BEGIN
  IF NOT EXISTS (SELECT 1 FROM pg_type WHERE typname = 'subscription_status') THEN
    CREATE TYPE subscription_status AS ENUM ('active', 'cancelled', 'paused', 'incomplete');
  END IF;
END $$;

ALTER TABLE users
  ADD COLUMN IF NOT EXISTS subscription_status subscription_status NOT NULL DEFAULT 'incomplete',
  ADD COLUMN IF NOT EXISTS stripe_customer_id text;

CREATE INDEX IF NOT EXISTS idx_users_subscription_status ON users(subscription_status);
```

The `DO $$ ... END $$` block is needed because `CREATE TYPE ... IF NOT EXISTS` does not exist in Postgres — you have to check the catalog by hand.

## RLS policies

The webhook writes to `users` with the service role, which bypasses RLS, so no policy is needed for the write path. For reads, users should be able to see their own `subscription_status` and `stripe_customer_id` so the frontend can gate features and open the billing portal:

```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_select_own"
  ON users FOR SELECT
  USING (auth.uid() = id);
```

If you already have a broader "own row" select policy on `users`, do nothing — it already covers these new columns.

## Gating access by subscription

Typical pattern: feature tables reference `users` and have an RLS policy that only allows access if the owner has an active subscription. For example, if your app has a `pool` table (user opt-ins for events):

```sql
CREATE POLICY "pool_select_active_subscribers"
  ON pool FOR SELECT
  USING (
    user_id = auth.uid()
    AND EXISTS (
      SELECT 1 FROM users
      WHERE users.id = auth.uid()
      AND subscription_status = 'active'
    )
  );
```

This makes subscription status a hard gate at the database level — a cancelled user cannot see their own pool rows even if the frontend forgets to check. That is the right default for a paid product: the source of truth is the database, and every layer above it is a convenience.
