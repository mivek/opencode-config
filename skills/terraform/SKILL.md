---
name: terraform
description: Conventions for Terraform infrastructure code. Load when writing or modifying .tf files, working with providers, modules, state, or planning infrastructure changes.
compatibility: opencode
metadata:
  tool: terraform
  scope: infrastructure
---

# Terraform conventions

## Always check state before suggesting changes

Before modifying any resource, verify current state :
```bash
terraform state list
terraform state show <resource>
```

Don't assume the configured code matches deployed reality — drift happens.

## File organization

Standard layout :
```
main.tf              # primary resources
variables.tf         # input variables
outputs.tf           # outputs
versions.tf          # provider/terraform version constraints
providers.tf         # provider configurations
data.tf              # data sources (optional)
locals.tf            # locals (optional)
<feature>.tf         # split by domain for larger configs
```

Don't put everything in `main.tf`. Split by logical domain.

## Variables and outputs

Every variable needs :
- A `type` (always specify, never let Terraform infer)
- A `description` (humans will read this)
- A `default` only if optional
- A `validation` block when the input has constraints (length, regex, enum)

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

Every output needs a `description`. Mark `sensitive = true` for secrets.

## Resource naming

- Resource labels in snake_case : `aws_s3_bucket.app_logs` not `app-logs`
- Real names (the `name` attribute) follow the cloud provider's convention (often kebab-case)
- Include environment in real names : `myapp-prod-database` not `database`

## Modules

- Use a module when the same pattern repeats 2+ times
- Don't use a module for a one-off — adds indirection without benefit
- Pin module versions : `source = "..."  version = "1.2.3"` (never use `latest`)
- Local modules : `./modules/<name>/` relative to root

## State management

- **Never commit state files** (`*.tfstate`, `*.tfstate.backup`) — add to `.gitignore`
- Use a remote backend (S3, GCS, Terraform Cloud, or for homelab : a private gitea repo with the `http` backend)
- Lock state during apply (DynamoDB for S3, or built-in for Terraform Cloud)
- `terraform init` after every backend config change

## Standard workflow

| Step | Command | Purpose |
|---|---|---|
| Init | `terraform init` | Download providers, configure backend |
| Format | `terraform fmt -recursive` | Format files |
| Validate | `terraform validate` | Syntax + reference check |
| Plan | `terraform plan -out=plan.tfplan` | Show changes, save for apply |
| Apply | `terraform apply plan.tfplan` | Apply saved plan |
| Destroy | `terraform destroy` | Tear down — DANGER |

**Never run `apply` without a saved plan file** in production. Plan + review + apply that exact plan.

## Pre-commit checks

Before every commit :
1. `terraform fmt -recursive` (idempotent)
2. `terraform validate`
3. Optionally `tflint` if installed
4. Optionally `tfsec` or `checkov` for security

## Provider versioning

In `versions.tf` :
```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"  # pessimistic constraint
    }
  }
}
```

Pin to a minor version with `~>`. Major version bumps require testing.

## Common pitfalls

- **`count` vs `for_each`** : use `for_each` (map/set) for stable IDs, `count` (number) only for "n identical copies"
- **Sensitive in outputs** : if an output exposes a secret, set `sensitive = true` — otherwise it leaks in logs
- **Forgetting `depends_on`** : Terraform infers dependencies from references, but some implicit deps (e.g., IAM policy must exist before role uses it) need explicit `depends_on`
- **Hardcoded ARNs/IDs** : never hardcode resource IDs. Reference them : `aws_iam_role.app.arn`
- **Apply on dirty state** : if `plan` shows unexpected diff, investigate before applying. Probably someone edited resources in the cloud console.

## When generating new code

1. Check `versions.tf` first to know what provider versions are pinned
2. Check existing resource naming patterns and match them
3. Add variables for anything that varies across environments
4. Add outputs for anything other modules/resources will reference
5. Always run `terraform fmt` and `terraform validate` on generated code
