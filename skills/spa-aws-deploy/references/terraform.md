# Terraform Module Templates

Complete templates for the Terraform infrastructure. Replace placeholders in `{CURLY_BRACES}` with project-specific values.

## File: `terraform/providers.tf`

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket  = "{PROJECT}-terraform-state"
    key     = "infra/terraform.tfstate"
    region  = "{AWS_REGION}"
    encrypt = true
  }
}

# Primary provider — all infrastructure goes here
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile  # Optional; remove if using env vars
}

# us-east-1 alias — required for CloudFront ACM certificates
provider "aws" {
  alias   = "us_east_1"
  region  = "us-east-1"
  profile = var.aws_profile
}
```

## File: `terraform/variables.tf`

```hcl
variable "aws_region" {
  description = "Primary AWS region for all resources"
  type        = string
  default     = "{AWS_REGION}"
}

variable "aws_profile" {
  description = "AWS CLI profile name (optional)"
  type        = string
  default     = null
}

variable "domain_name" {
  description = "Root domain (e.g. example.com)"
  type        = string
}

# Per-environment Supabase variables
variable "prod_supabase_url" {
  type      = string
  sensitive = true
}

variable "stg_supabase_url" {
  type      = string
  sensitive = true
}
```

## File: `terraform/main.tf`

```hcl
# Look up the existing hosted zone (must be created out-of-band or imported)
data "aws_route53_zone" "this" {
  name         = var.domain_name
  private_zone = false
}

# ============================================================================
# Production environment
# ============================================================================
module "production" {
  source = "./modules/environment"

  providers = {
    aws           = aws
    aws.us_east_1 = aws.us_east_1
  }

  environment    = "production"
  bucket_prefix  = "{PROJECT}-"
  domain_suffix  = var.domain_name
  hosted_zone_id = data.aws_route53_zone.this.zone_id

  spas = {
    # Repeat per SPA. Each entry creates one S3 bucket + CloudFront distribution + DNS record.
    app = {
      subdomain     = "app"
      bucket_suffix = "app-spa"
      origin_id     = "s3-app"
    }
    admin = {
      subdomain     = "admin"
      bucket_suffix = "admin-spa"
      origin_id     = "s3-admin"
    }
  }

  supabase_url = var.prod_supabase_url
}

# ============================================================================
# Staging environment (optional — comment out if single-env)
# ============================================================================
module "staging" {
  source = "./modules/environment"

  providers = {
    aws           = aws
    aws.us_east_1 = aws.us_east_1
  }

  environment    = "staging"
  bucket_prefix  = "{PROJECT}-stg-"
  domain_suffix  = "stg.${var.domain_name}"
  hosted_zone_id = data.aws_route53_zone.this.zone_id

  spas = {
    app = {
      subdomain     = "app"
      bucket_suffix = "app-spa"
      origin_id     = "s3-app"
    }
    admin = {
      subdomain     = "admin"
      bucket_suffix = "admin-spa"
      origin_id     = "s3-admin"
    }
  }

  supabase_url = var.stg_supabase_url
}

# ============================================================================
# Homepage at apex domain (optional — only if there's a marketing site)
# ============================================================================
module "homepage_production" {
  source = "./modules/homepage"

  providers = {
    aws           = aws
    aws.us_east_1 = aws.us_east_1
  }

  domain_name      = var.domain_name
  bucket_name      = "{PROJECT}-homepage"
  hosted_zone_id   = data.aws_route53_zone.this.zone_id
  include_www      = true  # Also create www.example.com -> apex redirect
}

# ============================================================================
# Deploy IAM user — used by CI/CD and local deploys
# ============================================================================
resource "aws_iam_user" "deploy" {
  name = "{PROJECT}-deploy"
}

resource "aws_iam_policy" "deploy" {
  name = "{PROJECT}-deploy-policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3SyncBuckets"
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetObject",
          "s3:DeleteObject",
          "s3:ListBucket",
        ]
        Resource = concat(
          [for arn in module.production.spa_s3_bucket_arns : arn],
          [for arn in module.production.spa_s3_bucket_arns : "${arn}/*"],
          [for arn in module.staging.spa_s3_bucket_arns : arn],
          [for arn in module.staging.spa_s3_bucket_arns : "${arn}/*"],
        )
      },
      {
        Sid    = "CloudFrontInvalidation"
        Effect = "Allow"
        Action = ["cloudfront:CreateInvalidation", "cloudfront:GetInvalidation"]
        Resource = "*"
      },
    ]
  })
}

resource "aws_iam_user_policy_attachment" "deploy" {
  user       = aws_iam_user.deploy.name
  policy_arn = aws_iam_policy.deploy.arn
}
```

## File: `terraform/outputs.tf`

These outputs are consumed by deploy scripts via `terraform output -raw <key>`.

```hcl
# Production
output "app_cloudfront_distribution_id" {
  value = module.production.spa_cloudfront_distribution_ids["app"]
}

output "app_s3_bucket_name" {
  value = module.production.spa_s3_bucket_names["app"]
}

output "admin_cloudfront_distribution_id" {
  value = module.production.spa_cloudfront_distribution_ids["admin"]
}

output "admin_s3_bucket_name" {
  value = module.production.spa_s3_bucket_names["admin"]
}

# Staging — prefixed with stg_
output "stg_app_cloudfront_distribution_id" {
  value = module.staging.spa_cloudfront_distribution_ids["app"]
}

output "stg_app_s3_bucket_name" {
  value = module.staging.spa_s3_bucket_names["app"]
}

# Homepage
output "homepage_cloudfront_distribution_id" {
  value = module.homepage_production.cloudfront_distribution_id
}

output "homepage_s3_bucket_name" {
  value = module.homepage_production.s3_bucket_name
}
```

## Module: `modules/spa/` (the reusable SPA building block)

### `modules/spa/variables.tf`
```hcl
variable "bucket_name" {
  type = string
}

variable "aliases" {
  description = "CloudFront aliases (e.g. ['app.example.com'])"
  type        = list(string)
}

variable "acm_certificate_arn" {
  type = string
}

variable "origin_id" {
  description = "Unique CloudFront origin ID — keep stable to avoid drift"
  type        = string
}

variable "hosted_zone_id" {
  description = "Route 53 hosted zone ID. Pass empty string to skip DNS record creation."
  type        = string
  default     = ""
}

variable "dns_name" {
  description = "DNS A record name. Empty string skips creation (e.g. for apex managed elsewhere)."
  type        = string
  default     = ""
}
```

### `modules/spa/main.tf`
```hcl
# S3 bucket
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

# CloudFront Origin Access Control — secure S3 access without public bucket
resource "aws_cloudfront_origin_access_control" "this" {
  name                              = "${var.bucket_name}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront distribution
resource "aws_cloudfront_distribution" "this" {
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  aliases             = var.aliases
  price_class         = "PriceClass_100"  # NA + EU + Asia. Use 200 for cheaper.

  origin {
    domain_name              = aws_s3_bucket.this.bucket_regional_domain_name
    origin_id                = var.origin_id
    origin_access_control_id = aws_cloudfront_origin_access_control.this.id
  }

  default_cache_behavior {
    target_origin_id       = var.origin_id
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    # Managed CachingOptimized policy
    cache_policy_id = "658327ea-f89d-4fab-a63d-7e88639e58f6"
  }

  # SPA routing: 404/403 -> /index.html with status 200
  # This is what makes React Router work for deep links.
  custom_error_response {
    error_code            = 404
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  custom_error_response {
    error_code            = 403
    response_code         = 200
    response_page_path    = "/index.html"
    error_caching_min_ttl = 0
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    acm_certificate_arn      = var.acm_certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
}

# S3 bucket policy: only this CloudFront distribution can read
resource "aws_s3_bucket_policy" "this" {
  bucket = aws_s3_bucket.this.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AllowCloudFrontOAC"
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.this.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.this.arn
        }
      }
    }]
  })
}

# Optional Route 53 A record (skip when caller manages DNS — e.g. apex domain)
resource "aws_route53_record" "this" {
  count = var.dns_name != "" && var.hosted_zone_id != "" ? 1 : 0

  zone_id = var.hosted_zone_id
  name    = var.dns_name
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.this.domain_name
    zone_id                = aws_cloudfront_distribution.this.hosted_zone_id
    evaluate_target_health = false
  }
}
```

### `modules/spa/outputs.tf`
```hcl
output "cloudfront_distribution_id" { value = aws_cloudfront_distribution.this.id }
output "cloudfront_domain_name"     { value = aws_cloudfront_distribution.this.domain_name }
output "cloudfront_arn"             { value = aws_cloudfront_distribution.this.arn }
output "s3_bucket_name"             { value = aws_s3_bucket.this.id }
output "s3_bucket_arn"              { value = aws_s3_bucket.this.arn }
```

## Module: `modules/environment/` (per-env wrapper)

### `modules/environment/variables.tf`
```hcl
variable "environment"    { type = string }
variable "bucket_prefix"  { type = string }
variable "domain_suffix"  { type = string }
variable "hosted_zone_id" { type = string }

variable "spas" {
  description = "Map of SPAs to create. Key = logical name."
  type = map(object({
    subdomain     = string
    bucket_suffix = string
    origin_id     = string
  }))
}

variable "supabase_url" {
  type      = string
  sensitive = true
}
```

### `modules/environment/main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.us_east_1]
    }
  }
}

# Collect all subdomains for the SAN list
locals {
  subdomains = [for k, v in var.spas : "${v.subdomain}.${var.domain_suffix}"]
}

# ACM certificate (must be in us-east-1 for CloudFront)
resource "aws_acm_certificate" "this" {
  provider                  = aws.us_east_1
  domain_name               = var.domain_suffix
  subject_alternative_names = local.subdomains
  validation_method         = "DNS"

  lifecycle { create_before_destroy = true }
}

# DNS validation records
resource "aws_route53_record" "validation" {
  for_each = {
    for dvo in aws_acm_certificate.this.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  zone_id         = var.hosted_zone_id
  name            = each.value.name
  type            = each.value.type
  records         = [each.value.record]
  ttl             = 60
  allow_overwrite = true
}

resource "aws_acm_certificate_validation" "this" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.this.arn
  validation_record_fqdns = [for r in aws_route53_record.validation : r.fqdn]
}

# Create one SPA per entry in var.spas
module "spas" {
  source = "../spa"

  for_each = var.spas

  bucket_name         = "${var.bucket_prefix}${each.value.bucket_suffix}"
  aliases             = ["${each.value.subdomain}.${var.domain_suffix}"]
  acm_certificate_arn = aws_acm_certificate_validation.this.certificate_arn
  origin_id           = each.value.origin_id
  hosted_zone_id      = var.hosted_zone_id
  dns_name            = "${each.value.subdomain}.${var.domain_suffix}"
}
```

### `modules/environment/outputs.tf`
```hcl
output "spa_cloudfront_distribution_ids" {
  value = { for k, v in module.spas : k => v.cloudfront_distribution_id }
}

output "spa_s3_bucket_names" {
  value = { for k, v in module.spas : k => v.s3_bucket_name }
}

output "spa_s3_bucket_arns" {
  value = { for k, v in module.spas : k => v.s3_bucket_arn }
}
```

## Module: `modules/homepage/` (apex domain — optional)

Use this only when there's a homepage at the apex domain (e.g. `example.com`). It wraps the SPA module but manages the apex A record + optional `www` redirect itself, since the SPA module's DNS handling assumes a subdomain.

### `modules/homepage/main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      configuration_aliases = [aws.us_east_1]
    }
  }
}

variable "domain_name"    { type = string }
variable "bucket_name"    { type = string }
variable "hosted_zone_id" { type = string }
variable "include_www"    { type = bool; default = true }

# ACM certificate for apex (+ www if requested)
resource "aws_acm_certificate" "this" {
  provider                  = aws.us_east_1
  domain_name               = var.domain_name
  subject_alternative_names = var.include_www ? ["www.${var.domain_name}"] : []
  validation_method         = "DNS"

  lifecycle { create_before_destroy = true }
}

resource "aws_route53_record" "validation" {
  for_each = {
    for dvo in aws_acm_certificate.this.domain_validation_options :
    dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }
  zone_id         = var.hosted_zone_id
  name            = each.value.name
  type            = each.value.type
  records         = [each.value.record]
  ttl             = 60
  allow_overwrite = true
}

resource "aws_acm_certificate_validation" "this" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.this.arn
  validation_record_fqdns = [for r in aws_route53_record.validation : r.fqdn]
}

# Use the SPA module for S3 + CloudFront, but skip its DNS handling
module "spa" {
  source = "../spa"

  bucket_name         = var.bucket_name
  aliases             = var.include_www ? [var.domain_name, "www.${var.domain_name}"] : [var.domain_name]
  acm_certificate_arn = aws_acm_certificate_validation.this.certificate_arn
  origin_id           = "s3-homepage"
  dns_name            = ""  # Don't let the SPA module create DNS
}

# Apex A record
resource "aws_route53_record" "apex" {
  zone_id = var.hosted_zone_id
  name    = var.domain_name
  type    = "A"

  alias {
    name                   = module.spa.cloudfront_domain_name
    zone_id                = module.spa.cloudfront_distribution_id == "" ? "" : "Z2FDTNDATAQYW2"  # CloudFront global zone
    evaluate_target_health = false
  }
}

# www CNAME -> apex
resource "aws_route53_record" "www" {
  count = var.include_www ? 1 : 0

  zone_id = var.hosted_zone_id
  name    = "www.${var.domain_name}"
  type    = "A"

  alias {
    name                   = module.spa.cloudfront_domain_name
    zone_id                = "Z2FDTNDATAQYW2"
    evaluate_target_health = false
  }
}

output "cloudfront_distribution_id" { value = module.spa.cloudfront_distribution_id }
output "s3_bucket_name"             { value = module.spa.s3_bucket_name }
```

## Bootstrap: creating the state bucket

The Terraform state bucket must exist before `terraform init`. Create it manually one time:

```bash
aws s3api create-bucket \
  --bucket {PROJECT}-terraform-state \
  --region {AWS_REGION} \
  --create-bucket-configuration LocationConstraint={AWS_REGION}

aws s3api put-bucket-versioning \
  --bucket {PROJECT}-terraform-state \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket {PROJECT}-terraform-state \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

## Common gotchas

- **ACM cert in wrong region**: CloudFront only accepts certs from `us-east-1`. The `aws.us_east_1` provider alias is mandatory.
- **CloudFront global zone ID**: For Route 53 alias records pointing to CloudFront, the alias zone_id is always `Z2FDTNDATAQYW2` (not the hosted zone ID).
- **SPA routing without error pages**: Without the 404 -> /index.html error response, deep links return real 404s. Easy to forget, painful to debug.
- **Origin ID drift**: Changing `origin_id` causes CloudFront to recreate the origin, which can cause downtime. Set it once and leave it.
- **Bucket policy after distribution**: The bucket policy references the distribution ARN, so the distribution must exist first. Terraform handles ordering, but if you destroy and recreate, you may see ordering errors.
