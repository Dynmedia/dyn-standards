---
name: dyn-aws-tagging
description: >-
  Dyn's AWS resource tagging standard, how-to, and governance. Use whenever
  someone asks how to tag AWS resources at Dyn, what the required tags or
  allowed values are, why their resource is flagged non-compliant, what the
  AWS Config tagging rules or the organization tag policy do, or who owns AWS
  tagging/governance. Covers the six-key standard, Terraform/CLI examples,
  enforcement behaviour, exclusions, and contacts.
---

# Dyn AWS Tagging

The single source of truth for tagging AWS resources at Dyn. If you tag a
resource, this is the standard. If a resource is flagged non-compliant or a
tagging call is rejected, the answer is here.

## TL;DR — tag every resource with these six keys

| Key | Allowed values | Notes |
|-----|----------------|-------|
| `Owner` | any value (an email is best) | presence only; who to contact |
| `Environment` | `production`, `development`, `integration`, `staging`, `sandbox`, `shared`, `security`, `tools`, `management`, `sit` | long forms |
| `Project` | `networking`, `connectivity`, `shared-services`, `security-hub`, `audit`, `log-archive`, `infra-tools`, `api-toolkit`, `fast`, `business-intelligence`, `contentdesk`, `mimir-fileflows`, `blog`, `account-factory` | |
| `CostCenter` | `product-and-tech`, `editorial-team` | |
| `Stage` | `prod`, `dev`, `int`, `staging` | short forms |
| `Team` | `dcc`, `infra` | |

Rules that trip people up:

- **Keys are PascalCase and case-sensitive.** `CostCenter` is right; `costcenter`
  / `cost-center` are different (wrong) keys.
- **Values are matched exactly.** No trimming, no case-folding. `Production` is
  not `production`.
- **`Environment` vs `Stage` use different vocabularies.** `Environment` uses
  long forms (`development`), `Stage` uses short (`dev`). `Environment=dev` is a
  common mistake and is INVALID.
- **Empty values fail.** A key present with `""` is non-compliant and cannot be
  whitelisted.

## How to tag (copy-paste)

### Terraform — per resource
```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  tags = {
    Owner       = "you@dynmedia.com"
    Environment = "production"
    Project     = "shared-services"
    CostCenter  = "product-and-tech"
    Stage       = "prod"
    Team        = "infra"
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
      Environment = "development"
      Project     = "infra-tools"
      CostCenter  = "product-and-tech"
      Stage       = "dev"
      Team        = "infra"
    }
  }
}
```

### CLI
```bash
aws ec2 create-tags --resources i-0123456789abcdef0 \
  --tags Key=Owner,Value=you@dynmedia.com \
         Key=Environment,Value=production \
         Key=Project,Value=shared-services \
         Key=CostCenter,Value=product-and-tech \
         Key=Stage,Value=prod \
         Key=Team,Value=infra

# S3 put-bucket-tagging REPLACES the whole tag set - include every tag.
aws s3api put-bucket-tagging --bucket my-bucket --tagging 'TagSet=[
  {Key=Owner,Value=you@dynmedia.com},
  {Key=Environment,Value=development},
  {Key=Project,Value=business-intelligence},
  {Key=CostCenter,Value=editorial-team},
  {Key=Stage,Value=dev},
  {Key=Team,Value=dcc}]'
```

## Why tagging (the short version)

- **Cost:** `Project` + `CostCenter` are the only way spend is attributed and
  charged back. No tags = unallocated cost.
- **Ownership:** `Owner` is who to contact in an incident or cleanup. Untagged
  resources become orphans nobody dares delete.
- **Environment segregation:** `Environment` / `Stage` drive change control,
  backup, and alerting.
- **Operations:** consistent tags make inventory and automation possible.

## How this is governed (two mechanisms)

Both use the same six keys and allowed values, so they agree.

1. **AWS Config rules (detective).** Deployed org-wide from the delegated Config
   admin account `754348400096` (region `eu-central-1`). Rule
   `ou_rzmo_bjyh9b48_required_tags` reports resources missing keys or using bad
   values; `ou_rzmo_bjyh9b48_acm_certificate_expiration` warns 7 days before a
   TLS cert expires. Detection and email/CloudWatch only - never blocks. Code
   lives in the `security-account` repo (`modules/aws-config`).

2. **Organization tag policy (preventive).** Policy `Organization-Wide-Tagging`
   (`p-957g5s40o6`), managed from the management account `660571558619`.
   **Enforcement is LIVE.** It BLOCKS any create/tag call that sets a
   non-compliant VALUE for `Project`, `Environment`, `Stage`,
   `DataClassification`, or `Compliance`, across 53 AWS services
   (`<service>:ALL_SUPPORTED` - every resource type in those services that
   supports tag-policy enforcement). It is attached to every in-scope OU and
   account (16 targets), not the root. Code lives in the `shahriar-sajib` repo
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
scope. A newly created account is in scope by default.

## "My resource is flagged / my tagging call was rejected" - triage

1. **Rejected at create/tag time?** That is the tag policy enforcing a VALUE
   (typical error: `TagPolicyViolation` / "tags ... do not comply with the tag
   policy"; in Terraform the whole apply fails on that resource). Check the
   value against the table above - most often it is `Environment=dev` (use
   `development`) or a `Project` value not in the allowed list. Fix the value
   (ideally in provider `default_tags`) and re-run. Excluded accounts never see
   this.
2. **Flagged non-compliant in Config but created fine?** That is detection. Add
   the missing keys / fix the value; it re-evaluates automatically.
3. **Key present but still failing?** Check case (PascalCase) and for an empty
   value.
4. **Value genuinely missing from the allowed list** (e.g. a real new project)?
   That is a standard change - raise it with the owners below, do not work
   around it by using a wrong value.

## Best practices

- Set the six tags once via provider `default_tags` rather than per resource.
- Put a real email in `Owner`, not a team slug or placeholder.
- Keep `Environment` (long) and `Stage` (short) consistent with each other
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

> Keep this file the source of truth. If the standard changes (new allowed
> value, new enforced service, scope change), update this skill in the same
> change so the team always sees the current rules.
