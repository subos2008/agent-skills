# GitHub Actions CI/CD Template

Complete `.github/workflows/deploy.yml` template for the full pipeline: tests -> staging -> e2e -> production -> smoke.

## File: `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [master]

# Only one deploy at a time. Don't cancel in progress (would leave half-deployed state).
concurrency:
  group: deploy
  cancel-in-progress: false

env:
  AWS_DEFAULT_REGION: {AWS_REGION}

jobs:
  # ==========================================================================
  # 1. Unit tests — must pass before any deploy
  # ==========================================================================
  unit-tests:
    name: Unit tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies (app)
        working-directory: web-apps/app
        run: npm ci

      - name: Run app tests
        working-directory: web-apps/app
        run: npm run test:run

      - uses: denoland/setup-deno@v2
        with:
          deno-version: v2.x

      - name: Run edge function tests
        run: deno test supabase/functions/

  # ==========================================================================
  # 2. Deploy to staging
  # ==========================================================================
  deploy-staging:
    name: Deploy staging
    needs: [unit-tests]
    runs-on: ubuntu-latest
    environment: staging  # GitHub Environment scopes secrets to staging
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
      SUPABASE_PROJECT_REF: ${{ secrets.STAGING_SUPABASE_PROJECT_REF }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Set short SHA
        run: echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> "$GITHUB_ENV"

      # ----- Supabase migrations -----
      - uses: supabase/setup-cli@v1
      - name: Push database migrations
        run: |
          npx supabase link --project-ref "$SUPABASE_PROJECT_REF"
          npx supabase db push

      # ----- Edge functions (run in order: schema migrations -> functions -> SPAs) -----
      - name: Deploy edge functions
        run: |
          for fn in {EDGE_FUNCTION_LIST}; do
            echo "Deploying $fn..."
            npx supabase functions deploy "$fn" --project-ref "$SUPABASE_PROJECT_REF"
          done

      # ----- App SPA -----
      - name: Build and deploy app
        working-directory: web-apps/app
        run: |
          npm ci
          VITE_GIT_HASH="$SHORT_SHA" npm run build -- --mode staging

          aws s3 sync dist/ s3://{PROJECT}-stg-app-spa --delete --cache-control "max-age=300"

          aws s3 cp s3://{PROJECT}-stg-app-spa/index.html s3://{PROJECT}-stg-app-spa/index.html \
            --content-type "text/html" \
            --cache-control "no-cache" \
            --metadata-directive REPLACE

          ID=$(aws cloudfront create-invalidation \
            --distribution-id "${{ vars.STG_APP_DISTRIBUTION_ID }}" \
            --paths "/*" \
            --query 'Invalidation.Id' --output text)
          aws cloudfront wait invalidation-completed \
            --distribution-id "${{ vars.STG_APP_DISTRIBUTION_ID }}" --id "$ID"

      # Repeat the build/deploy block above for each additional SPA.
      # Or factor into a composite action / matrix job for cleaner code.

  # ==========================================================================
  # 3. End-to-end tests against staging
  # ==========================================================================
  e2e-tests:
    name: E2E tests (staging)
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - name: Install Playwright
        run: |
          cd e2e
          npm ci
          npx playwright install --with-deps

      - name: Run E2E tests
        run: cd e2e && npx playwright test --project=staging

      - name: Upload Playwright report
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: e2e/playwright-report
          retention-days: 7

  # ==========================================================================
  # 4. Deploy to production
  # ==========================================================================
  deploy-production:
    name: Deploy production
    needs: [e2e-tests]
    runs-on: ubuntu-latest
    environment: production  # Different GitHub Environment, different secrets
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
      SUPABASE_PROJECT_REF: ${{ secrets.PROD_SUPABASE_PROJECT_REF }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - uses: supabase/setup-cli@v1

      - name: Set short SHA
        run: echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> "$GITHUB_ENV"

      # Production .env files come from secrets — they're never committed.
      - name: Write app .env.production
        run: echo "${{ secrets.PROD_APP_ENV }}" > web-apps/app/.env.production

      - name: Push database migrations
        run: |
          npx supabase link --project-ref "$SUPABASE_PROJECT_REF"
          npx supabase db push

      - name: Deploy edge functions
        run: |
          for fn in {EDGE_FUNCTION_LIST}; do
            npx supabase functions deploy "$fn" --project-ref "$SUPABASE_PROJECT_REF"
          done

      - name: Build and deploy app
        working-directory: web-apps/app
        run: |
          npm ci
          VITE_GIT_HASH="$SHORT_SHA" npm run build -- --mode production

          aws s3 sync dist/ s3://{PROJECT}-app-spa --delete --cache-control "max-age=300"

          aws s3 cp s3://{PROJECT}-app-spa/index.html s3://{PROJECT}-app-spa/index.html \
            --content-type "text/html" \
            --cache-control "no-cache" \
            --metadata-directive REPLACE

          ID=$(aws cloudfront create-invalidation \
            --distribution-id "${{ vars.PROD_APP_DISTRIBUTION_ID }}" \
            --paths "/*" \
            --query 'Invalidation.Id' --output text)
          aws cloudfront wait invalidation-completed \
            --distribution-id "${{ vars.PROD_APP_DISTRIBUTION_ID }}" --id "$ID"

  # ==========================================================================
  # 5. Smoke tests against production
  # ==========================================================================
  smoke-tests:
    name: Smoke tests (production)
    needs: [deploy-production]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }

      - name: Run production smoke tests
        run: |
          cd e2e
          npm ci
          npx playwright install --with-deps
          npx playwright test --project=smoke-production
```

## Required GitHub setup

### Repository secrets (Settings -> Secrets and variables -> Actions -> Secrets)

These should be **repository-level** secrets (not environment-scoped) since they're used by both staging and production:

| Secret | Source |
|--------|--------|
| `AWS_ACCESS_KEY_ID` | From the deploy IAM user Terraform created |
| `AWS_SECRET_ACCESS_KEY` | From the deploy IAM user |
| `SUPABASE_ACCESS_TOKEN` | Supabase dashboard -> Account -> Access Tokens |

### Environment secrets (Settings -> Environments)

Create two environments: `staging` and `production`. These let you scope secrets per environment.

**Staging environment secrets:**
| Secret | Value |
|--------|-------|
| `STAGING_SUPABASE_PROJECT_REF` | The staging Supabase project ref |

**Production environment secrets:**
| Secret | Value |
|--------|-------|
| `PROD_SUPABASE_PROJECT_REF` | The production Supabase project ref |
| `PROD_APP_ENV` | Full contents of `.env.production` for the app SPA |
| `PROD_{OTHER_SPA}_ENV` | One per SPA |

### Environment variables (Settings -> Environments -> Variables)

Per-environment **variables** (not secrets) for non-sensitive things like CloudFront IDs:

**Staging:**
- `STG_APP_DISTRIBUTION_ID` — value from `terraform output stg_app_cloudfront_distribution_id`
- `STG_{OTHER_SPA}_DISTRIBUTION_ID` for each SPA

**Production:**
- `PROD_APP_DISTRIBUTION_ID`
- `PROD_{OTHER_SPA}_DISTRIBUTION_ID`

## Why this structure

**One job per stage**: Each stage (test, deploy staging, e2e, deploy production, smoke) is a separate job with `needs:` dependencies. If e2e fails, production deploy doesn't run — staging stays as the latest known-good state.

**`environment:` per deploy**: GitHub Environments give you per-environment secret scoping plus optional approval gates. You can require manual approval before production deploys by configuring it in the environment settings — no workflow change needed.

**Concurrency group**: Prevents two pipelines deploying at once (race conditions). `cancel-in-progress: false` queues new runs instead of cancelling — important because cancelling a half-finished deploy leaves infrastructure in an inconsistent state.

**Migrations -> functions -> SPAs**: This order matters. Edge functions and SPAs may depend on schema changes, so migrations have to land first. Functions go before SPAs because the new SPA build may rely on a new function being available.

**Distribution IDs as variables, not secrets**: They're not sensitive (CloudFront IDs are visible in DNS anyway) and using `vars` makes them visible in workflow runs for debugging.

**Production env files written from secrets**: Never commit production `.env` files. The CI writes them at deploy time from a multi-line GitHub secret.

## Optional: matrix or composite action for multiple SPAs

If you have many SPAs, the repeated build/deploy blocks get verbose. Two cleaner approaches:

### Option A: Matrix job
```yaml
deploy-staging-spas:
  needs: [deploy-staging-functions]
  strategy:
    matrix:
      spa: [app, admin, dashboard]
  steps:
    - uses: actions/checkout@v4
    # ... build and deploy ${{ matrix.spa }}
```

### Option B: Composite action
Create `.github/actions/deploy-spa/action.yml` with the build/deploy steps as inputs (`spa-name`, `bucket`, `distribution-id`, `mode`). Reference it from the workflow once per SPA.

Start with the simple inline version. Refactor if you find yourself copying blocks more than twice.

## Adding manual approval for production

In **Settings -> Environments -> production**, enable "Required reviewers" and add yourself. Now the production deploy job will pause for your approval before running — useful while building confidence in the pipeline.

No workflow changes needed; it's purely a GitHub Environment setting.
