# 13 — Gotchas

[← Contents](README.md)

Each one has a *why*, so you can reason your way out of the next one instead of memorising this list. Add your own as you hit them.

| # | Gotcha | Why it happens | What to do |
|---|---|---|---|
| 1 | Stack stuck in `ROLLBACK_COMPLETE`, can't update | The first create failed, so there's no good state to update from | Delete the stack, fix, recreate. In CI, detect this and delete before retrying. |
| 2 | "Custom-named resource requires replacing" | Replacement creates the new resource before deleting the old one; both can't share a name | Don't hardcode names, or include `${AWS::StackName}` in them |
| 3 | Can't delete stack: bucket not empty | CFN won't delete your data for you | Empty the bucket first (all versions, if versioned), or use `DeletionPolicy: Retain` |
| 4 | `deploy` fails in CI with "No changes to deploy" | Default is a non-zero exit when the change set is empty | Add `--no-fail-on-empty-changeset` |
| 5 | `Requires capabilities: [CAPABILITY_NAMED_IAM]` | CFN makes you explicitly acknowledge IAM changes, which are privilege escalation risks | Add the capability, after reading the IAM in the template |
| 6 | Can't delete or update the network stack | Another stack imports one of its exports | `list-imports`, update the consumers first. Or use SSM for loose coupling |
| 7 | Stack hangs ~1 hour on a custom resource | The Lambda crashed without sending a response | Always send SUCCESS/FAILED in try/finally, including on Delete |
| 8 | `UPDATE_ROLLBACK_FAILED` | Something was changed or deleted outside CFN, so the rollback can't restore it | Fix the cause, then `continue-update-rollback --resources-to-skip` |
| 9 | Change set looked clean, but deploy failed | Change sets diff templates; they don't check permissions, limits or name collisions | Expect it. Read the first FAILED event |
| 10 | Manual console change "survives" a redeploy | CFN only touches properties that changed in the template | Run drift detection regularly; see [10](10-drift-detection.md) |
| 11 | `!Ref` gave a URL / name instead of an ARN | `!Ref` returns a per-type default identifier | Check **Return values**; use `!GetAtt X.Arn` |
| 12 | `!Base64 !Sub` YAML error | One YAML node can't carry two short tags | `Fn::Base64: !Sub \|` |
| 13 | `DeletionPolicy: Retain` resource still costing money | Retain *orphans* the resource; it no longer belongs to any stack | Track and clean up retained resources deliberately |
| 14 | Stack policy blocks *every* update | Once a policy exists, everything is denied unless explicitly allowed | Start the policy with `Allow Update:*` on `*`, then add the Denys |
| 15 | Template too large | 51,200-byte inline limit | `--s3-bucket` on deploy (allows up to 1 MB), or split into nested stacks |
| 16 | Second deploy rejected while the first is running | The stack is locked during `*_IN_PROGRESS` | That's the built-in lock. Serialise deploys per stack in CI (`concurrency:` in GitHub Actions) |
