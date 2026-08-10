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
| **Terraform roots** | `iam/`, `clusters/` | Split by who applies them — see "Three identities" below |
| **Terraform modules** | `spa/`, `environment/`, `homepage/` | S3, CloudFront, Route 53, ACM certs |
| **Deploy scripts** | Per-app `deploy.sh` + `deploy-smart.sh` | Build, S3 sync, CloudFront invalidation, change detection |
| **GitHub Actions** | `.github/workflows/deploy.yml` | tests -> staging -> e2e -> production -> smoke |
| **Edge function deploys** | In CI + deploy scripts | `supabase functions deploy` integrated into pipeline |

## Read these before generating anything

**This page is an index. It is deliberately not enough to build from.** The skill
is in `references/`:

- `references/terraform.md` — the module and root templates, in full
- `references/deploy-scripts.md` — deploy scripts, and the credential model
- `references/github-actions.md` — the CI/CD pipeline

Read all three before writing any file.

The most common failure by far is an agent that skims this page, believes it has
the skill, and reconstructs from the summary something subtly different — most
expensively, running deploys under an admin role, which the credential model
below forbids in as many words. Working implementations of this pattern are
usually on the same disk; one read of an existing `terraform/` or `deploy.sh`
settles more than several rounds of reasoning.

## Step 1: Gather requirements

Before generating anything, ask:

1. **Project name** -- used for S3 bucket prefixes, Terraform naming (e.g., `myapp`)
2. **Domain** -- e.g., `myapp.com`. **"None yet" is a supported answer** — see "Phase 1: no domain yet"
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

**Templates and the root layout: `references/terraform.md`.** Generate from
there, not from this page.

### Key design decisions to understand

**Three identities, and never fewer.** Deploys run as a **deploy user** whose
policy covers uploading build output and creating invalidations, and nothing
else — it deliberately cannot create or delete buckets and distributions, so it
cannot apply Terraform. Infrastructure is applied by a **Terraform user** with a
scoped long-lived key and **no IAM permissions, so it cannot widen itself**. Only
the bootstrap root that creates those two users runs as an **admin** identity,
and it is applied rarely and by hand.

That split is why there are two Terraform roots: `iam/` (admin-applied, creates
the users and holds their keys in its own state) and `clusters/` (Terraform-user
applied, everything else). Never use admin credentials for a routine deploy, and
do not collapse the Terraform user into the admin one to save a step — the whole
point of the middle tier is that the credential you use every week cannot grant
itself anything.

**Multi-provider for ACM**: CloudFront needs certificates in `us-east-1` regardless of where other resources live. The environment module receives both providers.

**SPA routing**: CloudFront custom error responses return `/index.html` with HTTP 200 for 404/403 errors. This is what makes client-side routing work -- without it, deep links like `/settings/profile` would 404.

**S3 security**: Buckets block all public access. CloudFront authenticates via Origin Access Control (OAC), and the bucket policy only allows requests from that specific CloudFront distribution ARN.

**Remote state**: S3 backend with encryption. The state bucket itself is created manually (one-time bootstrap).

**Environment isolation**: Same Terraform module instantiated twice with different variables -- different bucket prefixes, domain suffixes, Supabase credentials. Staging can suppress things like email sends.

**Software is not infrastructure**: a build version must never reach Terraform state. Where a project also runs Lambda, `references/terraform.md` has the pattern that keeps them separable.

## Phase 1: no domain yet

Every CloudFront distribution gets a free `*.cloudfront.net` hostname with
working HTTPS, so a site can be live and inspectable before a domain exists.
This is the normal starting state for a new project, not an edge case — an
acquirer, a reviewer or a founder checking staging does not need it on its final
hostname, and waiting on a brand decision to get anything deployed is a bad
trade.

To do it:

- Skip the `environment` and `homepage` modules. The `environment` module's body
  is an ACM certificate plus DNS validation wrapping a `for_each` over SPAs; with
  no domain it collapses to the loop, so the root instantiates `modules/spa`
  directly.
- In `modules/spa`, replace the `viewer_certificate` block with
  `cloudfront_default_certificate = true`, and drop `aliases`.
- Drop the `aws.us_east_1` provider alias — it exists only for ACM.
- Drop the Route 53 record and the `hosted_zone_id` / `dns_name` variables.

Phase 2 attaches a certificate and DNS alias to the **same** distributions.
Nothing is recreated, so this is a genuine phase rather than throwaway work.

## Step 3: Deploy scripts

**Templates, the credential model, and the cache and upload-ordering rules:
`references/deploy-scripts.md`.** Several of those rules exist because of
specific production failures and are not derivable — read them.

Two scripts are generated: a per-SPA `deploy.sh`, and a `deploy-smart.sh` that
skips apps already at `HEAD`.

### Version tracking

Inject git hash into builds: `VITE_GIT_HASH=$(git rev-parse --short HEAD)`. Render as `<meta name="version" content="...">` in index.html. This is what smart deploy compares, and what makes a deployed build identifiable in a bug report.

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

**Workflow template: `references/github-actions.md`.** The pipeline:

```
push to master
       |
  unit-tests ----> deploy-staging ----> e2e-tests ----> deploy-production ----> smoke-tests
```

One deployment at a time, queued rather than cancelled; secrets scoped per
GitHub Environment; migrations before edge functions before SPAs. The reference
has the reasons and the exact YAML.

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
2. Apply `terraform/iam` **as the admin identity** — this creates the Terraform
   and deploy users and outputs their access keys into that root's state
3. Configure local AWS profiles for both users from those outputs
4. Apply `terraform/clusters/*` **as the Terraform user**
5. Create or import Route 53 hosted zone, update domain registrar nameservers
   (skip for Phase 1 — no domain yet)
6. Set up GitHub secrets:
   - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` (from the **deploy** user)
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
