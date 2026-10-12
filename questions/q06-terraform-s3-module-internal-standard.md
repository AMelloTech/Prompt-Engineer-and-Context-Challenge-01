---
question: 6
title: "Terraform-S3-Module-Internal-Standard"
framework: "C-A-R-E (Context-Action-Result-Example)"
provider: "Anthropic"
model: "Claude Opus 5.5"
effort: "default"
status: "final"
---

## Scenario

Strickland, Head of Security & Compliance, published the internal IaC
standard that every new Terraform module must follow. Doc Brown asked for a
reusable Terraform module that creates S3 buckets compliant with that
standard. All teams in the company will consume it, so it needs a usage
example and must follow the style of the existing VPC module (see the general
context in [`context/hill-valley-tech.md`](../context/hill-valley-tech.md)).

## Prompt

```
CONTEXT: Strickland (Head of Security & Compliance) published the internal
IaC standard every new Terraform module must follow:
- Required tags on every resource: Owner, CostCenter, Environment
- Resource names prefixed with hvt-
- Every S3 bucket: encryption (SSE-S3 minimum), versioning enabled, full
  public access block, server access logging configured
- Every variable in variables.tf has description and type
Doc Brown wants a reusable S3 module that all teams will consume.

Action: We need a module to create buckets S3 follow the standard that have been set in the above lines for new modules, the module should also be reusable to all the teams in the company and have usage example.
Result: Create all the necessary files and exemples on how to use it and mapped rules and where its implemented and how to implement it.
Exemple: Match the style of the VPC module that it`s currently been used by the company.

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string
}

locals {
  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags = merge(local.common_tags, {
    Name = "hvt-vpc-${var.environment}"
  })
}
```

## Model

**Model used:** Claude Opus 5.5 (Anthropic)

**Why this model:** the task needs current AWS provider knowledge (v4+ split
S3 resources) and consistent application of a style reference across several
files. The output was checked with `terraform fmt` and `terraform validate`
(Terraform 1.16, AWS provider ≥ 5.0), and both passed.

## Output

### Structure

```
terraform-aws-s3-bucket/
├── versions.tf
├── variables.tf
├── main.tf
├── outputs.tf
└── examples/
    └── basic/
        └── main.tf
```

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

### `variables.tf`

```hcl
variable "name" {
  description = "Nome lógico do bucket, sem prefixo e sem ambiente (ex.: relatorios-financeiros)"
  type        = string

  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]{1,40}[a-z0-9]$", var.name))
    error_message = "Use apenas letras minúsculas, números e hífens (3 a 42 caracteres)."
  }
}

variable "environment" {
  description = "Nome do ambiente (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "O ambiente deve ser dev, staging ou production."
  }
}

variable "owner" {
  description = "Time ou pessoa responsável pelo bucket (tag Owner)"
  type        = string
}

variable "cost_center" {
  description = "Centro de custo para rateio da conta (tag CostCenter)"
  type        = string
}

variable "logging_target_bucket" {
  description = "Nome do bucket central, já existente, que recebe os access logs deste bucket"
  type        = string
}

variable "kms_key_arn" {
  description = "ARN de uma chave KMS para SSE-KMS. Se nulo, usa SSE-S3 (AES256), o mínimo exigido pelo padrão"
  type        = string
  default     = null
}

variable "force_destroy" {
  description = "Permite destruir o bucket mesmo com objetos dentro. Mantenha false em production"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags adicionais. As obrigatórias (Owner, CostCenter, Environment) não podem ser sobrescritas"
  type        = map(string)
  default     = {}
}
```

### `main.tf`

```hcl
locals {
  bucket_name = "hvt-${var.name}-${var.environment}"

  common_tags = {
    Owner       = var.owner
    CostCenter  = var.cost_center
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "this" {
  bucket        = local.bucket_name
  force_destroy = var.force_destroy

  # Mandatory tags are merged last so callers can't override them.
  tags = merge(var.tags, local.common_tags, {
    Name = local.bucket_name
  })
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket = aws_s3_bucket.this.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_ownership_controls" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    object_ownership = "BucketOwnerEnforced"
  }
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = var.kms_key_arn == null ? "AES256" : "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = var.kms_key_arn != null
  }
}

resource "aws_s3_bucket_logging" "this" {
  bucket = aws_s3_bucket.this.id

  target_bucket = var.logging_target_bucket
  target_prefix = "${local.bucket_name}/"
}
```

### `outputs.tf`

```hcl
output "bucket_id" {
  description = "Nome do bucket criado"
  value       = aws_s3_bucket.this.id
}

output "bucket_arn" {
  description = "ARN do bucket, para uso em políticas IAM"
  value       = aws_s3_bucket.this.arn
}

output "bucket_regional_domain_name" {
  description = "Domínio regional do bucket"
  value       = aws_s3_bucket.this.bucket_regional_domain_name
}
```

### `examples/basic/main.tf`

```hcl
provider "aws" {
  region = "us-east-1"
}

module "relatorios_financeiros" {
  # Other teams: source = "git::<module repository URL>?ref=v1.0.0"
  source = "../../"

  name                  = "relatorios-financeiros"
  environment           = "production"
  owner                 = "time-financas"
  cost_center           = "CC-1042"
  logging_target_bucket = "hvt-access-logs-production"

  tags = {
    DataClassification = "confidential"
  }
}

output "bucket_arn" {
  value = module.relatorios_financeiros.bucket_arn
}
```

### Rule mapping

| Standard rule | Where | How |
|---|---|---|
| Tags `Owner`, `CostCenter`, `Environment` | `main.tf`, `locals.common_tags` | Merged into `aws_s3_bucket.this`, last in `merge()` so callers can't override them. The other S3 sub-resources (versioning, encryption, etc.) can't be tagged in AWS. For extra coverage, also set `default_tags` in the provider. |
| `hvt-` prefix | `main.tf`, `local.bucket_name` | `hvt-${var.name}-${var.environment}`. Validation on `name` keeps the full name within S3's 63-character limit. |
| Encryption (SSE-S3 minimum) | `aws_s3_bucket_server_side_encryption_configuration.this` | AES256 (SSE-S3) by default. Passing `kms_key_arn` upgrades it to SSE-KMS. It can never be turned off. |
| Versioning enabled | `aws_s3_bucket_versioning.this` | Hardcoded to `Enabled` and not exposed as a variable. |
| Full public access block | `aws_s3_bucket_public_access_block.this` | All four flags `true`, not configurable. `BucketOwnerEnforced` also disables ACLs. |
| Server access logging | `aws_s3_bucket_logging.this` | Sends logs to `logging_target_bucket` (required, no default) under the prefix `<bucket name>/`. |
| Variables with `description` and `type` | `variables.tf` | All 8 variables have both, plus validation on `name` and `environment`. |

### How to implement

1. Publish the module in its own repository and tag a version (`v1.0.0`).
2. Teams consume it with `source = "git::<repo URL>?ref=v1.0.0"`, as in
   `examples/basic`.
3. Make sure the central log bucket exists and its bucket policy allows the
   `logging.s3.amazonaws.com` service principal to write to it.
4. Run `terraform init`, `terraform plan`, and review that every rule appears
   in the plan before `apply`.

### Design decisions

- **The log bucket is external, passed in as a variable.** If the module
  created its own log bucket, that bucket would also need logging under the
  standard, which loops back on itself. Companies also usually centralize
  access logs in a single bucket.
- **The compliance controls aren't configurable.** Versioning, the public
  access block, and encryption can't be switched off by a caller, so no team
  can create a bucket that breaks the standard.
- **Descriptions are in Portuguese**, matching the VPC module's style.

## Rationale

**Context** carries Strickland's rules as a list, and each rule became a
dedicated resource or local, mapped one-to-one in the rule table. **Action**
asks for a reusable module with a usage example, which produced input
variables with validation and `examples/basic/main.tf`. **Result** asks for
the files, usage, and rule mapping, which became the file structure, the
mapping table, and the "how to implement" steps. **Example** (the VPC module)
set the style: `common_tags` in `locals` merged into the resource, resources
named `this`, names in the form `hvt-<type>-${var.environment}`, and
descriptions in Portuguese. What could be better: the prompt left the logging
target and the file list to the model, and it chose an external log bucket
passed as a variable and explained why. Stating both in the prompt would
make the output less dependent on the model's own decisions.
