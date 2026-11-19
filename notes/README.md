# CloudFormation Notes — through a Terraform lens

These notes are written for someone who already ships AWS with Terraform. They aren't meant to be memorised. Each note answers:

1. **What problem does this solve?** A situation you've probably hit already.
2. **How does CloudFormation solve it, and why that way?**
3. **Terraform lens:** what you already know that maps to it, and where the two differ.
4. **Real world:** where you'll actually run into it.
5. **See it yourself:** a 5-minute hands-on, linked to a Vikunja module.
6. **Takeaway:** the one or two lines worth keeping.

## Table of contents

| # | Note | The question it answers | Vikunja |
|---|------|------------------------|---------|
| 01 | [How CloudFormation works](01-how-cloudformation-works.md) | Who runs the engine and where does state live? | M0–M1 |
| 02 | [Templates](02-templates.md) | What is a template, and why is its language so limited? | M1 |
| 03 | [Intrinsic functions & pseudo parameters](03-intrinsic-functions.md) | How do resources reference each other? | M1 |
| 04 | [Parameters, Mappings, Conditions, Outputs](04-parameters-mappings-conditions-outputs.md) | How does one template serve dev and prod? | M3 |
| 05 | [Writing a custom template](05-writing-a-custom-template.md) | How do I go from "I need X" to a working template? | M1–M3 |
| 06 | [Stack lifecycle & rollback](06-stack-lifecycle-and-rollback.md) | What happens when a deploy fails halfway? | M1–M2 |
| 07 | [Change sets](07-change-sets.md) | What will this change do to prod *before* I run it? | M2 |
| 08 | [Update behaviour & protection](08-update-behaviour-and-protection.md) | Will this edit replace my database? How do I stop that? | M2, M5 |
| 09 | [Composing stacks](09-composing-stacks.md) | One big stack or many? How do they share values? | M4, M7 |
| 10 | [Drift detection](10-drift-detection.md) | Someone clicked in the console. Now what? | M5 |
| 11 | [Ecosystem](11-ecosystem.md) | CDK, SAM, StackSets, Hooks, cfn-lint: where they fit | M6–M8 |
| 12 | [Command cheat sheet](12-cheatsheet.md) | `terraform X` → `aws cloudformation Y` | all |
| 13 | [Gotchas](13-gotchas.md) | What bites people in practice | all |

## Why learn CloudFormation if you already know Terraform?

Three practical reasons, not "because it's AWS-native":

- **It sits underneath a lot of AWS tooling.** CDK, SAM, Amplify, Copilot and the Serverless Framework all compile to CloudFormation and deploy as stacks. When a `cdk deploy` fails, the error you debug is a CloudFormation stack event. You'll read CFN even if you never write it.
- **It needs no bootstrap.** Terraform needs an S3 bucket, a lock table and an OIDC role *before* it can run. That's a chicken-and-egg problem. A common real-world fix is a small CloudFormation stack that creates the Terraform backend, because CFN needs no backend of its own.
- **Some AWS features are built on it.** StackSets roll guardrails out across every account in an Organization, Service Catalog products are CFN templates, and many SaaS vendors onboard you with a "Launch Stack" button that is a CFN template creating a cross-account IAM role.

## The one-table mental map

| Concept | Terraform | CloudFormation | What actually differs |
|---|---|---|---|
| Unit of code | Root module (folder of `.tf`) | **Template** (one YAML/JSON file) | One document, not a folder |
| Deployed instance | Workspace + state | **Stack** | The stack *is* the state, held by AWS |
| State storage | `tfstate` in S3 + DynamoDB lock | CloudFormation service | Nothing to secure, lock or migrate |
| Engine | CLI on your machine / runner | AWS service, server-side | Your CLI only submits a request |
| Preview | `terraform plan` | **Change set** | Stored server-side, reviewable by others |
| Apply | `terraform apply` | `deploy` / `execute-change-set` | Asynchronous; you watch events |
| Failure | Partial apply, fix forward | **Automatic rollback** | All-or-nothing per operation |
| Inputs | `variable` | **Parameters** | Can be AWS-typed (`AWS::EC2::VPC::Id`) |
| Derived values | `locals`, functions | **Mappings** + intrinsic functions | No arbitrary expressions |
| Conditional resource | `count = x ? 1 : 0` | **Conditions** | Declarative only |
| Loops | `count`, `for_each` | None (except `Fn::ForEach` via a transform) | Copy or generate (CDK) |
| Outputs | `output` | **Outputs** (+ `Export`) | Exports lock the producer stack |
| Reuse | Modules | **Nested stacks** | Each child is a real stack |
| Cross-project refs | `terraform_remote_state` | `Fn::ImportValue`, SSM | Imports are hard dependencies |
| Drift | Every `plan` refreshes & shows it | **Drift detection** on demand | Reported, never auto-fixed |
| Delete protection | `prevent_destroy` | `DeletionPolicy`, termination protection, stack policy | `Retain` keeps the resource after the stack dies |
| Adopt existing infra | `terraform import` | Import change set, IaC generator | Same idea |
| Multi-account | Pipelines / Terragrunt | **StackSets** | Native fan-out |
