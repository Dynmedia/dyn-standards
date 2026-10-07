---
name: dyn-aws-tagging
description: >-
  Dyn's AWS resource tagging standard, how-to, and governance. Use whenever
  someone asks how to tag AWS resources at Dyn, what the required tags or
  allowed values are, why their resource is flagged non-compliant, what the
  AWS Config tagging rules or the organization tag policy do, or who owns AWS
  tagging/governance. Also covers the optional Dyn-AIWorkload tag for classifying
  AI resources and AI cost attribution. Covers the six-key standard, the
  Dyn-AIWorkload classification tag, Terraform/CLI examples, enforcement behaviour,
  exclusions, and contacts.
---

# Dyn AWS Tagging

The single source of truth for tagging AWS resources at Dyn. If you tag a
resource, this is the standard. If a resource is flagged non-compliant or a
tagging call is rejected, the answer is here.

## TL;DR — tag every resource with these six keys

| Key | Allowed values | Notes |
|-----|----------------|-------|
| `Owner` | any value (an email is best) | presence only; who to contact |
| `Dyn-Environment` | `production`, `development`, `integration`, `staging`, `sandbox`, `shared`, `security`, `tools`, `management`, `sit` | long forms |
| `Dyn-Project` | `networking`, `connectivity`, `shared-services`, `security-hub`, `audit`, `log-archive`, `infra-tools`, `api-toolkit`, `fast`, `business-intelligence`, `contentdesk`, `mimir-fileflows`, `blog`, `account-factory` | |
| `Dyn-CostCenter` | `product-and-tech`, `editorial-team` | |
| `Dyn-Stage` | `prod`, `dev`, `int`, `staging` | short forms |
| `Dyn-Team` | `dcc`, `infra` | |

Rules that trip people up:

- **Keys are PascalCase with a `Dyn-` prefix, and matched exactly.**
  `Dyn-CostCenter` is right; `costcenter`, `CostCenter` and
  `Dyn-CostCenter` are different (wrong) keys. All seven: `Owner`,
  `Dyn-Environment`, `Dyn-Stage`, `Dyn-Project`, `Dyn-Team`,
  `Dyn-CostCenter`, `Dyn-AIWorkload` — `Owner` is the ONLY key without the
  prefix.
- **Values are lowercase.**
- **Values are matched exactly.** No trimming, no case-folding. `Production` is
  not `production`.
- **`Dyn-Environment` vs `Dyn-Stage` use different vocabularies.** `Dyn-Environment` uses
  long forms (`development`), `Dyn-Stage` uses short (`dev`). `Dyn-Environment=dev` is a
  common mistake and is INVALID.
- **Empty values fail.** A key present with `""` is non-compliant and cannot be
  whitelisted.

## Optional: `Dyn-AIWorkload` — classify AI resources

`Dyn-AIWorkload` is a **7th, OPTIONAL** tag. It is **not** part of the six-key
"tag everything" standard — apply it **only to AI resources**, in addition to
the six keys.

| Key | Allowed values | Notes |
|-----|----------------|-------|
| `Dyn-AIWorkload` | `developer`, `product`, `platform` | optional; AI resources only; PascalCase `Dyn-` key, lowercase exact-match values |

- `developer` — internal dev/tooling AI (copilots, experimentation, AI dev infra)
- `product` — AI embedded in a customer-facing product
- `platform` — shared AI infrastructure (model hosting, knowledge bases, gateways)

Why it exists: it powers **AI cost attribution** — spend on AI resources is
grouped into developer / product / platform in Cost Explorer, and drives the AI
budget alerts. Combined with `Dyn-Project`, it answers "how much AI, for
which product, and is it dev or production?"

When to apply it:

- **Only when a resource is genuinely AI** and is *taggable* — e.g. SageMaker
  endpoints/jobs, Bedrock agents / knowledge bases / provisioned throughput,
  EC2/ECS/EKS hosting your own models.
- **Not** on ordinary resources. A non-AI S3 bucket or RDS instance should not
  carry `Dyn-AIWorkload`.
- Note: usage-based AI with no resource (e.g. on-demand Bedrock model calls,
  SaaS AI subscriptions) **cannot** be tagged; that spend is attributed by
  account instead, not by this tag.

Governance: like every key in the tag policy, its **value** is validated when
present (only `developer`/`product`/`platform` are allowed), but its **presence
is never required** — non-AI resources are simply not evaluated for it.

Example (AI resource — six keys PLUS Dyn-AIWorkload):

```hcl
resource "aws_sagemaker_endpoint" "inference" {
  # ...
  tags = {
    Owner       = "you@dynmedia.com"
    "Dyn-Environment" = "production"
    "Dyn-Project"     = "business-intelligence"
    "Dyn-CostCenter"  = "product-and-tech"
    "Dyn-Stage"       = "prod"
    "Dyn-Team"        = "dcc"
    "Dyn-AIWorkload"  = "product" # <- only because this is an AI resource
  }
}
```

### For Kiro: applying `Dyn-AIWorkload` when generating AI infrastructure

When you (Kiro) scaffold or modify a **taggable AI resource** (SageMaker,
Bedrock agent/knowledge base/provisioned throughput, or compute whose purpose is
hosting/serving a model), add `Dyn-AIWorkload` alongside the six standard tags:

- Choose the value from context: internal tooling/experimentation → `developer`;
  customer-facing product feature → `product`; shared AI infra used by several
  teams → `platform`.
- If the intent is ambiguous, **ask the user which of developer/product/platform
  applies** rather than guessing.
- Do **not** add `Dyn-AIWorkload` to non-AI resources.
- Prefer setting it in provider `default_tags` only when the whole stack is AI;
  otherwise set it per AI resource.

## How to tag (copy-paste)

### Terraform — per resource
```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  tags = {
    Owner       = "you@dynmedia.com"
    "Dyn-Environment" = "production"
    "Dyn-Project"     = "shared-services"
    "Dyn-CostCenter"  = "product-and-tech"
    "Dyn-Stage"       = "prod"
    "Dyn-Team"        = "infra"
  }
}
```

### Terraform — once per stack (preferred)
```hcl
provider "aws" {
  region = "eu-central-1"
  default_tags {
    tags = {
      Owner       = "you@dynmedia.com"
      "Dyn-Environment" = "development"
      "Dyn-Project"     = "infra-tools"
      "Dyn-CostCenter"  = "product-and-tech"
      "Dyn-Stage"       = "dev"
      "Dyn-Team"        = "infra"
    }
  }
}
```

### CLI
```bash
aws ec2 create-tags --resources i-0123456789abcdef0 \
  --tags Key=Owner,Value=you@dynmedia.com \
         Key=Dyn-Environment,Value=production \
         Key=Dyn-Project,Value=shared-services \
         Key=Dyn-CostCenter,Value=product-and-tech \
         Key=Dyn-Stage,Value=prod \
         Key=Dyn-Team,Value=infra

# S3 put-bucket-tagging REPLACES the whole tag set - include every tag.
aws s3api put-bucket-tagging --bucket my-bucket --tagging 'TagSet=[
  {Key=Owner,Value=you@dynmedia.com},
  {Key=Dyn-Environment,Value=development},
  {Key=Dyn-Project,Value=business-intelligence},
  {Key=Dyn-CostCenter,Value=editorial-team},
  {Key=Dyn-Stage,Value=dev},
  {Key=Dyn-Team,Value=dcc}]'
```

## Why tagging (the short version)

- **Cost:** `Dyn-Project` + `Dyn-CostCenter` are the only way spend is attributed and
  charged back. No tags = unallocated cost.
- **Ownership:** `Owner` is who to contact in an incident or cleanup. Untagged
  resources become orphans nobody dares delete.
- **Environment segregation:** `Dyn-Environment` / `Dyn-Stage` drive change control,
  backup, and alerting.
- **Operations:** consistent tags make inventory and automation possible.

## How this is governed (two mechanisms)

Both use the same six keys and allowed values, so they agree. The optional
`Dyn-AIWorkload` key is only in the tag policy: Config doesn't check it, so a
missing or wrong `Dyn-AIWorkload` is never reported by Config (a wrong value is
still blocked by the tag policy).

1. **AWS Config rules (detective).** Deployed org-wide from the delegated Config
   admin account `754348400096` (region `eu-central-1`). Rule
   `org_wide_required_tags` reports resources missing keys or using bad
   values; `org_wide_acm_certificate_expiration` warns 7 days before a
   TLS cert expires. Detection and email/CloudWatch only - never blocks. Code
   lives in the `security-account` repo (`modules/aws-config`).

2. **Organization tag policy (preventive).** Policy `Organization-Wide-Tagging`
   (`p-957g5s40o6`), managed from the management account `660571558619`.
   **Enforcement is currently OFF (observe-only)** while the org moves to
   the `Dyn-` keys; non-compliant tags are reported, not blocked. When it is
   switched on it BLOCKS any create/tag call that sets a non-compliant VALUE
   for `Dyn-Project`, `Dyn-Environment`, `Dyn-Stage`, `Dyn-CostCenter`, `Dyn-Team` or
   `Dyn-AIWorkload` (every key except `Owner`), across 53 AWS services
   (`<service>:ALL_SUPPORTED` - every resource type in those services that
   supports tag-policy enforcement). The optional `Dyn-AIWorkload` key is governed
   the same way: if present, its value must be `developer`/`product`/`platform`;
   it is never required. It is attached to every in-scope OU and account (16
   targets), not the root. Code lives in the `shahriar-sajib` repo
   (`Tagging/`); state is in S3 in the management account.

Two things the tag policy does NOT do, worth knowing:

- It does **not** block untagged resources - only wrong values on tagged ones.
  (Blocking untagged creation would need an SCP.)
- `Owner` is presence-only, so its value is never enforced.
- Services outside the 53 enforced ones (or resource types AWS does not
  support for enforcement) are not blocked - Config still reports them.
- Existing resources with bad values are not changed or deleted; the next
  tagging call on them must use a valid value.

## Scope: who is covered

Governance applies **organization-wide except six accounts**:
`dyn-contentdesk-prod`, `dyn-contentdesk-dev`, `dyn-contentdesk-stg`,
`dyn-deltatreaxis-prod`, `voscustomer1002.vos`, `dyn-mmo`. Everything else is in
scope.

New accounts:

- **AWS Config** monitors every new account automatically.
- **The tag policy** covers a new account only if it lands in one of the 13
  OUs the policy is attached to. A new OU, the Contentdesk OU, or the
  organization root gets **no** tag enforcement until it is added to the
  attachment list in `shahriar-sajib/Tagging`. Moving an account into the
  Contentdesk OU or to the root removes its enforcement.

## "My resource is flagged / my tagging call was rejected" - triage

1. **Rejected at create/tag time?** That is the tag policy enforcing a VALUE
   (exact error: `TagPolicyError: The tag policy does not allow the specified
   value for the following tag key: '<Key>'.`; in Terraform the apply fails on
   that resource). Check the
   value against the table above - most often it is `Dyn-Environment=dev` (use
   `development`) or a `Dyn-Project` value not in the allowed list. Fix the value
   (ideally in provider `default_tags`) and re-run. Excluded accounts never see
   this.
2. **Flagged non-compliant in Config but created fine?** That is detection. Add
   the missing keys / fix the value; it re-evaluates automatically.
3. **Key present but still failing?** Check the key spelling (lowercase with the
   `Dyn-` prefix: `Dyn-Environment`, not `environment`, `Environment` or `dyn-environment`) and for an empty value.
4. **Value genuinely missing from the allowed list** (e.g. a real new project)?
   That is a standard change - raise it with the owners below, do not work
   around it by using a wrong value.

## Best practices

- Set the six tags once via provider `default_tags` rather than per resource.
- Put a real email in `Owner`, not a team slug or placeholder.
- Keep `Dyn-Environment` (long) and `Dyn-Stage` (short) consistent with each other
  (`production`/`prod`, `development`/`dev`, `integration`/`int`).
- New account? It is monitored automatically - tag from day one.
- Need a new allowed value? Change the standard, do not misuse an existing one.

## Contacts / ownership

- **AWS tagging standard & governance:** Ahmed (Shahriar) Sajib -
  ahmed.sajib@dynmedia.com
- **Config compliance stack:** `security-account` repo (`modules/aws-config`),
  runs in account `754348400096`.
- **Tag policy / enforcement:** `shahriar-sajib` repo (`Tagging/`), runs in the
  management account `660571558619`.
- **Compliance notifications:** SNS topic `config-compliance-notifications`.

> Keys renamed to PascalCase with a `Dyn-` prefix in Oct 2026. The Tag
> Policy board in the Cloud Infra Portal (Security -> Organization Wide
> Tagging) is the source of truth for keys, values and their meanings (tag policy in `shahriar-sajib`,
> Config rule in `security-account`).

> Keep this file the source of truth. If the standard changes (new allowed
> value, new enforced service, scope change), update this skill in the same
> change so the team always sees the current rules.
