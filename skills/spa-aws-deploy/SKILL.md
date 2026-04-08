---
name: spa-aws-deploy
description: Set up AWS deployment infrastructure for React/Vite SPAs with Supabase backend. Creates Terraform modules (S3, CloudFront, Route 53, ACM), deploy scripts (per-app + smart parallel deploy), and GitHub Actions CI/CD pipeline. Use when the user wants to deploy a new SPA project to AWS, set up hosting infrastructure for a web app, create a CI/CD pipeline, scaffold S3+CloudFront hosting, or mentions deploying a Vite/React app with Supabase. Also use when adding a new SPA to an existing project that follows this pattern.
---

# SPA + Supabase AWS Deploy

Sets up production-grade AWS hosting for React/Vite SPAs backed by Supabase. Everything is infrastructure-as-code with automated CI/CD.

## Architecture

```
User --> Route 53 --> CloudFront (HTTPS, SPA routing) --> S3 (private, OAC)
                                                           |
                                                     React SPA loads
                                                           |
                                                     Supabase (Auth, DB, Edge Functions)
```

Each SPA gets its own S3 bucket + CloudFront distribution + DNS record. Environments (staging, production) are isolated via a shared Terraform module instantiated twice with different variables.

## What gets created

| Component | What | Why |
|-----------|------|-----|
| **Terraform modules** | `spa/`, `environment/`, `homepage/` | S3, CloudFront, Route 53, ACM certs, deploy IAM user |
| **Deploy scripts** | Per-app `deploy.sh` + `deploy-smart.sh` | Build, S3 sync, CloudFront invalidation, change detection |
| **GitHub Actions** | `.github/workflows/deploy.yml` | tests -> staging -> e2e -> production -> smoke |
| **Edge function deploys** | In CI + deploy scripts | `supabase functions deploy` integrated into pipeline |

## Step 1: Gather requirements

Before generating anything, ask:

1. **Project name** -- used for S3 bucket prefixes, Terraform naming (e.g., `myapp`)
2. **Domain** -- e.g., `myapp.com`
3. **SPAs** -- names and subdomains:
   - Is there a homepage at the apex domain? (`myapp.com`)
   - Other SPAs? e.g., `app` -> `app.myapp.com`, `admin` -> `admin.myapp.com`
4. **Environments** -- staging + production, or just one?
   - If staging: subdomain convention? Default: `stg.myapp.com`, `app.stg.myapp.com`
5. **AWS region** -- default `eu-west-2`
6. **Supabase project refs** -- one per environment
7. **Edge functions** -- names of functions to deploy (e.g., `stripe-webhook`, `create-checkout`)
8. **Existing infra?** -- Route 53 hosted zone? Terraform state bucket? AWS account/profile?

## Step 2: Terraform infrastructure

Read `references/terraform.md` for complete module templates. Generate the following structure:

```
terraform/
  main.tf              # Environments, homepage, deploy IAM user, state config
  providers.tf         # AWS primary region + us-east-1 alias (for ACM)
  variables.tf         # Root input variables
  outputs.tf           # CF distribution IDs + S3 bucket names (consumed by deploy scripts)
  terraform.tfvars     # (gitignored) Actual secret values
  modules/
    environment/       # Per-env: ACM cert + DNS validation + SPA loop
    spa/               # Reusable: one S3 bucket + CloudFront + Route 53 record
    homepage/          # Special: apex domain + www redirect (if needed)
```

### Key design decisions to understand

**Multi-provider for ACM**: CloudFront needs certificates in `us-east-1` regardless of where other resources live. The environment module receives both providers.

**SPA routing**: CloudFront custom error responses return `/index.html` with HTTP 200 for 404/403 errors. This is what makes client-side routing work -- without it, deep links like `/settings/profile` would 404.

**S3 security**: Buckets block all public access. CloudFront authenticates via Origin Access Control (OAC), and the bucket policy only allows requests from that specific CloudFront distribution ARN.

**Deploy IAM user**: A dedicated user with minimal permissions (S3 put/delete, CloudFront invalidation). CI/CD and local deploys both use this user. Never use root/admin creds for deploys.

**Remote state**: S3 backend with encryption. The state bucket itself is created manually (one-time bootstrap).

**Environment isolation**: Same Terraform module instantiated twice with different variables -- different bucket prefixes, domain suffixes, Supabase credentials. Staging can suppress things like email sends.

## Step 3: Deploy scripts

Read `references/deploy-scripts.md` for templates. Create:

### Per-SPA deploy script (`web-apps/{name}/deploy.sh`)

Each SPA gets an identical deploy script that:
1. Reads bucket name + CloudFront distribution ID from `terraform output`
2. Builds with `npm run build -- --mode $ENV` (Vite loads `.env.$ENV`)
3. Syncs to S3 with `--delete` and 5-minute cache
4. Overrides `index.html` to `no-cache` (HTML is always fresh, assets are cached)
5. Creates CloudFront invalidation and waits for completion

This `terraform output` approach means deploy scripts never hardcode AWS resource IDs -- Terraform is the single source of truth.

### Smart deploy (`deploy-smart.sh`)

An orchestrator that only redeploys what changed:
1. Curls each deployed app's `<meta name="version">` tag
2. Compares to current `git rev-parse --short HEAD`
3. Skips apps already at HEAD
4. Deploys changed apps + edge functions in parallel
5. Reports what was deployed/skipped and total time

### Version tracking

Inject git hash into builds: `VITE_GIT_HASH=$(git rev-parse --short HEAD)`. Render as `<meta name="version" content="...">` in index.html. This enables smart deploy's change detection.

## Step 4: Environment configuration

### Vite environment files pattern

```
web-apps/{name}/
  .env.staging       # Checked into git (staging Supabase URL, anon key, etc.)
  .env.production    # NOT in git (written by CI from GitHub secrets)
```

**Staging** is safe to commit -- it contains only staging credentials. Add a `VITE_ENV_HIGHLIGHT` color (e.g., `#f59e0b` amber) so the app can show a visual indicator that you're on staging.

**Production** env files are stored as GitHub Environment secrets and written to disk at CI deploy time.

Build selects environment automatically:
```bash
npm run build -- --mode staging     # loads .env.staging
npm run build -- --mode production  # loads .env.production
```

## Step 5: GitHub Actions CI/CD

Read `references/github-actions.md` for the workflow template. The pipeline:

```
push to master
       |
  unit-tests ----> deploy-staging ----> e2e-tests ----> deploy-production ----> smoke-tests
```

### Key patterns

**Concurrency**: One deployment at a time, queued not cancelled:
```yaml
concurrency: { group: deploy, cancel-in-progress: false }
```

**Environment separation**: GitHub Environments scope secrets per env. Staging secrets differ from production.

**Deploy order**: Migrations -> edge functions -> SPAs (functions may depend on new schema).

**Production env files**: Written from secrets at deploy time, not committed:
```yaml
- run: echo "${{ secrets.PROD_APP_ENV }}" > web-apps/app/.env.production
```

**CloudFront invalidation**: Each SPA deploy creates an invalidation and waits for it to complete before the job finishes.

## Step 6: Supabase edge functions

Edge functions deploy via `npx supabase functions deploy {name} --project-ref $REF`.

Include in:
- CI/CD workflow (after migrations, before SPAs)
- `deploy-smart.sh` (parallel with SPA deploys)
- Individual commands documented in project README

Migrations run first: `npx supabase link --project-ref $REF && npx supabase db push`

## Step 7: Initial setup checklist

After generating all files, walk the user through:

1. Create S3 bucket for Terraform state (manual, one-time bootstrap)
2. `terraform init` to initialize
3. Create or import Route 53 hosted zone, update domain registrar nameservers
4. `terraform plan` -- review, then `terraform apply`
5. Create IAM access keys for the deploy user Terraform created
6. Set up GitHub secrets:
   - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (from deploy user)
   - `SUPABASE_ACCESS_TOKEN` (from Supabase dashboard)
   - Per-environment: `PROD_{APP}_ENV` with full .env.production contents
7. Create GitHub Environments: `staging`, `production`
8. Create `.env.staging` files for each SPA (Supabase URL + anon key)
9. Add `<meta name="version">` to each SPA's index.html
10. First deploy: `./deploy-all.sh staging`
11. Verify: HTTPS works, SPA routing works (deep links), correct Supabase project

## Adapting for existing projects

If the user already has a project with some infrastructure:
- Check what exists (hosted zone, S3 buckets, etc.) and import into Terraform state
- Generate only the missing pieces
- Adapt deploy scripts to match existing naming conventions
