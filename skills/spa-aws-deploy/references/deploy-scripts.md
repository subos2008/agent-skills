# Deploy Script Templates

Bash scripts for building, deploying SPAs to S3, and invalidating CloudFront. Replace placeholders in `{CURLY_BRACES}` with project-specific values.

All scripts use Terraform outputs as the source of truth for AWS resource IDs — no hardcoding.

## Script: per-SPA deploy (`web-apps/{name}/deploy.sh`)

One copy per SPA. Identical structure, only the Terraform output keys differ.

```bash
#!/usr/bin/env bash
# Deploy a single SPA to S3 + invalidate CloudFront.
# Usage: ./web-apps/{name}/deploy.sh <staging|production>

set -euo pipefail

ENV="${1:-}"
if [[ "$ENV" != "staging" && "$ENV" != "production" ]]; then
  echo "Usage: $0 <staging|production>" >&2
  exit 1
fi

# Resolve repo paths
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
APP_DIR="$SCRIPT_DIR"
REPO_ROOT="$(cd "$SCRIPT_DIR/../.." && pwd)"
APP_NAME="$(basename "$APP_DIR")"

# Read Terraform outputs (no API calls — just reads local state)
tf_out() { terraform -chdir="$REPO_ROOT/terraform" output -raw "$1"; }

if [[ "$ENV" == "staging" ]]; then
  DISTRIBUTION_ID="$(tf_out "stg_${APP_NAME}_cloudfront_distribution_id")"
  BUCKET_NAME="$(tf_out "stg_${APP_NAME}_s3_bucket_name")"
else
  DISTRIBUTION_ID="$(tf_out "${APP_NAME}_cloudfront_distribution_id")"
  BUCKET_NAME="$(tf_out "${APP_NAME}_s3_bucket_name")"
fi

# Use the deploy IAM user profile (set up via aws configure --profile)
export AWS_PROFILE="${AWS_PROFILE:-{PROJECT}-deploy}"

# Inject git hash into the build (used by deploy-smart.sh for change detection)
export VITE_GIT_HASH="$(git rev-parse --short HEAD)"

echo "==> Building $APP_NAME ($ENV)"
cd "$APP_DIR"
npm run build -- --mode "$ENV"

echo "==> Syncing to s3://$BUCKET_NAME"
aws s3 sync "$APP_DIR/dist" "s3://$BUCKET_NAME" \
  --delete \
  --cache-control "max-age=300"

# index.html must NOT be cached — always serve fresh HTML.
# Static assets have hashed filenames so they're safe to cache.
echo "==> Setting index.html cache headers"
aws s3 cp "s3://$BUCKET_NAME/index.html" "s3://$BUCKET_NAME/index.html" \
  --content-type "text/html" \
  --cache-control "no-cache" \
  --metadata-directive REPLACE

echo "==> Invalidating CloudFront ($DISTRIBUTION_ID)"
INVALIDATION_ID="$(aws cloudfront create-invalidation \
  --distribution-id "$DISTRIBUTION_ID" \
  --paths "/*" \
  --query 'Invalidation.Id' \
  --output text)"

echo "    Invalidation $INVALIDATION_ID — waiting for completion..."
aws cloudfront wait invalidation-completed \
  --distribution-id "$DISTRIBUTION_ID" \
  --id "$INVALIDATION_ID"

echo "==> Done. $APP_NAME deployed to $ENV."
```

### Why each step matters

- **`set -euo pipefail`**: Fail fast on errors, undefined vars, and pipe failures.
- **`tf_out` reads Terraform outputs**: Single source of truth. If you change the bucket name in Terraform, deploy scripts pick it up automatically.
- **`VITE_GIT_HASH`**: Injected into the build so the SPA can render `<meta name="version">`. Smart deploy uses this for change detection.
- **`aws s3 sync --delete`**: Removes files in S3 that don't exist locally — keeps the bucket clean.
- **`--cache-control max-age=300`**: 5-minute cache for static assets (their filenames are hashed, so this is safe).
- **index.html `no-cache`**: HTML must always be fresh, otherwise users get old asset references after a deploy.
- **`wait invalidation-completed`**: Don't return until CloudFront has actually purged its cache. Otherwise the deploy looks complete but users might still see the old version.

## Script: full deploy (`deploy-all.sh`)

Sequential deploy of every SPA + every edge function. Simple, reliable, slow. Use for first-time setup or when smart deploy isn't appropriate.

```bash
#!/usr/bin/env bash
# Deploy everything to a single environment.
# Usage: ./deploy-all.sh <staging|production>

set -euo pipefail

ENV="${1:-}"
if [[ "$ENV" != "staging" && "$ENV" != "production" ]]; then
  echo "Usage: $0 <staging|production>" >&2
  exit 1
fi

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Load env-specific Supabase project ref
# shellcheck source=/dev/null
source "$REPO_ROOT/env/$ENV.env"  # exports SUPABASE_PROJECT_REF

# === SPAs ===
SPAS=({SPA_LIST})  # e.g. (homepage app admin)

for spa in "${SPAS[@]}"; do
  echo
  echo "############################################"
  echo "# Deploying $spa to $ENV"
  echo "############################################"
  bash "$REPO_ROOT/web-apps/$spa/deploy.sh" "$ENV"
done

# === Edge functions ===
FUNCTIONS=({EDGE_FUNCTION_LIST})  # e.g. (stripe-webhook create-checkout)

for fn in "${FUNCTIONS[@]}"; do
  echo
  echo "==> Deploying edge function: $fn"
  npx supabase functions deploy "$fn" --project-ref "$SUPABASE_PROJECT_REF"
done

echo
echo "All deploys to $ENV complete."
```

## Script: smart deploy (`deploy-smart.sh`)

Only redeploys what changed. Parallel, fast, with change detection.

```bash
#!/usr/bin/env bash
# Smart deploy: only redeploy SPAs that differ from HEAD.
# Usage: ./deploy-smart.sh <staging|production> [--dry-run] [--force]

set -euo pipefail

ENV="${1:-}"
shift || true

DRY_RUN=false
FORCE=false
for arg in "$@"; do
  case "$arg" in
    --dry-run) DRY_RUN=true ;;
    --force|-f) FORCE=true ;;
    *) echo "Unknown arg: $arg" >&2; exit 1 ;;
  esac
done

if [[ "$ENV" != "staging" && "$ENV" != "production" ]]; then
  echo "Usage: $0 <staging|production> [--dry-run] [--force]" >&2
  exit 1
fi

REPO_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Refuse to deploy a dirty working tree (you'd ship uncommitted code).
if [[ -n "$(git -C "$REPO_ROOT" status --porcelain)" ]]; then
  echo "Working tree is dirty. Commit or stash before deploying." >&2
  exit 1
fi

HEAD_HASH="$(git -C "$REPO_ROOT" rev-parse --short HEAD)"
echo "=== Smart deploy to $ENV (HEAD: $HEAD_HASH) ==="
echo

# === Configuration: SPAs and their deployed URLs ===
# Map: spa_name => deployed URL to check version on
declare -A SPA_URLS_PRODUCTION=(
  [homepage]="https://{DOMAIN}/"
  [app]="https://app.{DOMAIN}/"
  [admin]="https://admin.{DOMAIN}/"
)
declare -A SPA_URLS_STAGING=(
  [homepage]="https://stg.{DOMAIN}/"
  [app]="https://app.stg.{DOMAIN}/"
  [admin]="https://admin.stg.{DOMAIN}/"
)

if [[ "$ENV" == "production" ]]; then
  declare -n SPA_URLS=SPA_URLS_PRODUCTION
else
  declare -n SPA_URLS=SPA_URLS_STAGING
fi

# === Detect which SPAs need deploying ===
APPS_TO_DEPLOY=()
APPS_SKIPPED=()

echo "Checking deployed versions..."
for spa in "${!SPA_URLS[@]}"; do
  url="${SPA_URLS[$spa]}"

  if [[ "$FORCE" == "true" ]]; then
    APPS_TO_DEPLOY+=("$spa")
    printf "  %-12s -> force deploy\n" "$spa"
    continue
  fi

  # Extract <meta name="version" content="..."> from deployed index.html
  deployed=$(curl -sf --max-time 10 "$url" 2>/dev/null \
    | grep -oE '<meta name="version" content="[^"]+"' \
    | sed -E 's/.*content="([^"]+)".*/\1/' \
    || echo "unknown")

  if [[ "$deployed" == "$HEAD_HASH" ]]; then
    APPS_SKIPPED+=("$spa")
    printf "  %-12s -> up to date (%s)\n" "$spa" "$deployed"
  else
    APPS_TO_DEPLOY+=("$spa")
    printf "  %-12s -> needs deploy (deployed: %s, HEAD: %s)\n" "$spa" "$deployed" "$HEAD_HASH"
  fi
done

echo

if [[ "$DRY_RUN" == "true" ]]; then
  echo "(dry run — exiting)"
  exit 0
fi

if [[ ${#APPS_TO_DEPLOY[@]} -eq 0 ]]; then
  echo "Nothing to deploy. Everything is at $HEAD_HASH."
  exit 0
fi

# === Deploy in parallel ===
echo "Deploying ${#APPS_TO_DEPLOY[@]} SPAs in parallel..."
PIDS=()
LOG_DIR="$(mktemp -d)"

for spa in "${APPS_TO_DEPLOY[@]}"; do
  bash "$REPO_ROOT/web-apps/$spa/deploy.sh" "$ENV" \
    > "$LOG_DIR/$spa.log" 2>&1 &
  PIDS+=("$!:$spa")
done

# === Edge functions in parallel ===
# shellcheck source=/dev/null
source "$REPO_ROOT/env/$ENV.env"  # SUPABASE_PROJECT_REF
FUNCTIONS=({EDGE_FUNCTION_LIST})

for fn in "${FUNCTIONS[@]}"; do
  npx supabase functions deploy "$fn" --project-ref "$SUPABASE_PROJECT_REF" \
    > "$LOG_DIR/fn-$fn.log" 2>&1 &
  PIDS+=("$!:fn-$fn")
done

# === Wait for all, collect failures ===
FAILED=()
for entry in "${PIDS[@]}"; do
  pid="${entry%%:*}"
  name="${entry#*:}"
  if wait "$pid"; then
    echo "  $name — done"
  else
    echo "  $name — FAILED (see $LOG_DIR/$name.log)"
    FAILED+=("$name")
  fi
done

echo
echo "=== Summary ==="
echo "Environment: $ENV"
echo "Git hash:    $HEAD_HASH"
echo "Deployed:    ${#APPS_TO_DEPLOY[@]} SPAs, ${#FUNCTIONS[@]} edge functions"
[[ ${#APPS_SKIPPED[@]} -gt 0 ]] && echo "Skipped:     ${APPS_SKIPPED[*]} (already at HEAD)"

if [[ ${#FAILED[@]} -gt 0 ]]; then
  echo
  echo "FAILED: ${FAILED[*]}"
  exit 1
fi
echo
echo "All deployments successful."
```

### Why parallel + change detection matters

For projects with 4+ SPAs, sequential deploys take ages even when you only changed one SPA. Smart deploy makes "deploy after every commit" actually pleasant — typical iteration is 30-60 seconds instead of 5+ minutes.

The change detection works by curl'ing each deployed index.html and comparing the embedded `<meta name="version">` to `git rev-parse HEAD`. This requires:
1. Each SPA injects `VITE_GIT_HASH` into its build
2. index.html renders `<meta name="version" content={import.meta.env.VITE_GIT_HASH} />`
3. index.html is served `no-cache` so the curl always sees the latest

## File: `env/staging.env` and `env/production.env`

Per-environment shell variables consumed by deploy scripts.

```bash
# env/staging.env
export SUPABASE_PROJECT_REF="{STAGING_PROJECT_REF}"
```

```bash
# env/production.env
export SUPABASE_PROJECT_REF="{PRODUCTION_PROJECT_REF}"
```

Both should be checked into git (they contain only project refs, not secrets).

## Rendering the version meta tag in your SPA

Each SPA's `index.html` should render the git hash so smart deploy can detect changes. With Vite + React:

**`index.html`**:
```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="version" content="%VITE_GIT_HASH%" />
    <title>{App Name}</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

Vite replaces `%VITE_GIT_HASH%` with the value of `VITE_GIT_HASH` env var at build time. If `VITE_GIT_HASH=local` (the default in `.env.staging`), local builds will show "local" — that's fine, only deployed builds need real hashes.

## Setting up the AWS deploy profile

After Terraform creates the deploy IAM user, generate access keys and configure a local profile:

```bash
# Generate access keys (one time)
aws iam create-access-key --user-name {PROJECT}-deploy

# Configure named profile
aws configure --profile {PROJECT}-deploy
# Paste the access key + secret + region
```

Deploy scripts use this profile via `export AWS_PROFILE={PROJECT}-deploy`. CI/CD uses the same credentials but as env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`).
