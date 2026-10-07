# tagging-kiro-skills

The Dyn AWS **tagging** knowledge, packaged as a Kiro skill so anyone at Dyn can
get correct, up-to-date tagging guidance from Kiro on demand.

## What's here

```
tagging-kiro-skills/
└── .kiro/skills/dyn-aws-tagging/SKILL.md   <- the skill
```

`SKILL.md` is the single source of truth for AWS tagging at Dyn: the six-key
standard and allowed values, the optional `Dyn-AIWorkload` tag for AI resources
(`developer`/`product`/`platform`), Terraform/CLI how-to, why tagging matters, how
governance works (AWS Config detection + the organization tag policy), the
org-wide-minus-6 scope, a "my resource is flagged / rejected" triage guide,
best practices, and contacts.

## How to use it

A Kiro skill is **pull, not push**: it teaches Kiro how to answer, and activates
when you ask a relevant question. It never emails or notifies anyone.

Two ways to make Kiro pick it up:

1. **Per-workspace (simplest):** open this folder (`tagging-kiro-skills/`) in
   Kiro. Kiro reads `.kiro/skills/` from the workspace root, discovers the
   skill, and activates it when you ask about Dyn tagging.

2. **Everywhere on your machine (recommended for daily use):** copy the skill
   into your user-level skills directory so it is active in *every* repo you
   open:
   ```bash
   mkdir -p ~/.kiro/skills
   cp -R .kiro/skills/dyn-aws-tagging ~/.kiro/skills/
   ```
   Re-run that copy after pulling updates to stay current.

Then just ask Kiro things like:
- "How do I tag my S3 bucket at Dyn?"
- "What are the allowed Dyn-Environment values?"
- "Why was my tagging call rejected?"
- "Which Dyn-AIWorkload value should my SageMaker endpoint have?"

## Keeping it current

This file is authoritative. If the tagging standard changes (a new allowed
value, a newly enforced service, a scope change), update `SKILL.md` in the same
change so the whole team sees the current rules. Changes go through a PR on this
repo.

## Owner

Ahmed (Shahriar) Sajib - ahmed.sajib@dynmedia.com
