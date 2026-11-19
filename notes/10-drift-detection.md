# 10 — Drift detection

[← Contents](README.md)

## The problem
It's an incident at 2 a.m. Someone opens port 22 on a security group in the console "just for a minute". It's still open three months later. Your template says one thing, reality says another. That gap is **drift**.

## How each tool handles it
| | Terraform | CloudFormation |
|---|---|---|
| Detects drift | Every `plan` (refresh reads real resources) | Only when you run **drift detection** |
| Fixes drift | `apply` reverts it to match the code | **Never automatically** |
| Change preview sees drift | Yes | No: change sets compare template to template |

### Why CloudFormation doesn't auto-fix
CFN only acts on **differences between the old template and the new template**. If you don't touch a property in the template, CFN doesn't touch it on the resource, even when it has drifted. So:
- Drift on a property you **later edit** in the template → overwritten by your new value.
- Drift on a property you **never edit** → persists indefinitely, invisible to change sets.

This is the biggest mental-model difference from Terraform. Without TF's refresh on every plan, drift has to be looked for on purpose.

## The workflow
```bash
# 1. start detection (asynchronous)
ID=$(aws cloudformation detect-stack-drift --stack-name cfn-lab-app --query StackDriftDetectionId --output text)

# 2. wait for it
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id "$ID"

# 3. see which resources drifted and how
aws cloudformation describe-stack-resource-drifts --stack-name cfn-lab-app \
  --stack-resource-drift-status-filters MODIFIED DELETED
```
Resource statuses: `IN_SYNC`, `MODIFIED`, `DELETED`, `NOT_CHECKED` (not every resource type supports drift detection). The output shows *expected* vs *actual* property values, a bit like a `plan` diff.

## Resolving drift: two choices
1. **Reality is right** (the manual fix was correct): put the change in the template and deploy. Template and reality now agree.
2. **The template is right** (the manual change was wrong): revert by hand, or change that property in the template so CFN rewrites it, then change it back if needed.

And if someone **created** a resource by hand that should be managed, use a **resource import** change set (`--change-set-type IMPORT`) to adopt it into a stack. That's CFN's `terraform import`.

## Real world
- **Compliance:** AWS Config has a managed rule (`cloudformation-stack-drift-detection-check`) that flags drifted stacks, so you don't have to remember to run it.
- **Scheduled detection:** a nightly job or EventBridge rule runs drift detection on prod stacks and alerts on `DRIFTED`.
- **Prevention beats detection:** restrict console write access in prod (SCPs / IAM) so changes have to go through the pipeline.

## See it yourself (M5)
1. Add an inbound rule to the app SG in the console.
2. Run drift detection and read the expected vs actual values.
3. Deploy the **unchanged** template. Does the rule disappear? (Predict first. The answer explains the whole note.)

## Takeaway
> CloudFormation only reacts to template changes, so drift is reported, never fixed. Look for it deliberately (or with AWS Config) and resolve it by updating either the template or reality.
