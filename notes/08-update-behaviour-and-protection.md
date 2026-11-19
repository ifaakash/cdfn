# 08 — Update behaviour & protection

[← Contents](README.md)

## The problem
Two different worries:
1. **"Will this edit hurt the running resource?"** Some changes apply in place, some cause a reboot, and some destroy and recreate.
2. **"How do I make sure nobody, including me, accidentally deletes the prod database?"**

## Part 1: the three update behaviours
Every property on every resource type documents one of these under **"Update requires"**:

| Behaviour | What happens | Example | Terraform equivalent |
|---|---|---|---|
| **No interruption** | Changed in place, resource keeps running | Tags, S3 versioning, SG rules | `~ update in-place` |
| **Some interruption** | Changed in place but the resource restarts | EC2 `InstanceType` (stop, then start) | `~ update in-place` (TF doesn't warn you about the restart either) |
| **Replacement** | New resource created, old one deleted | S3 `BucketName`, RDS `DBInstanceIdentifier`, EC2 `AvailabilityZone` | `-/+ forces replacement` |

**Why replacement exists at all:** AWS APIs can't change some attributes after creation. A bucket can't be renamed, and an instance can't move AZ. The only way to "change" them is to make a new one.

### The custom-name trap
On replacement, CFN creates the new resource **before** deleting the old one (see [06](06-stack-lifecycle-and-rollback.md)). If you set a fixed name (`BucketName: my-bucket`), the new resource needs the same name as the one that still exists, so the update fails with:
> *CloudFormation cannot update a stack when a custom-named resource requires replacing.*

**Real-world rule:** let CFN generate names, or include `${AWS::StackName}` in them. Only hardcode a name when something external depends on it (for example a bucket name in a partner's config), and then accept that replacing it will be a manual job.

## Part 2: layers of protection
Each layer guards against a *different* accident. Production stacks usually use several.

| Layer | Protects against | Scope | Terraform equivalent |
|---|---|---|---|
| `DeletionPolicy: Retain` | Resource deleted when **removed from the template** or the **stack is deleted** | Per resource | `prevent_destroy` blocks the delete; `Retain` lets it proceed but keeps the resource |
| `DeletionPolicy: Snapshot` | Losing data on delete (RDS, EBS, ElastiCache, ...) | Per resource | `final_snapshot_identifier` (resource-specific) |
| `DeletionPolicy: RetainExceptOnCreate` | Keeps resources on delete, but still cleans up if the *first create* fails | Per resource | — |
| `UpdateReplacePolicy: Retain` | The **old** resource being deleted during a **replacement** | Per resource | `create_before_destroy` + manual handling |
| Termination protection | `delete-stack` on the whole stack | Per stack | — |
| **Stack policy** | **Updates** to specific resources (e.g. deny `Update:Replace` on the DB) | Per stack, by resource | — (closest: Sentinel/OPA policy) |
| IAM / service role | Who can call CFN at all, and what CFN may do | Account | Same |

### `Retain` vs `prevent_destroy`: an important difference
- Terraform `prevent_destroy` **refuses** the plan. Nothing happens.
- CFN `Retain` **lets the operation go ahead** but **orphans** the resource: it's removed from the stack and keeps running in your account, unmanaged, and you keep paying for it.

That's the right behaviour for "never lose this data", but you need to clean up retained resources deliberately.

### Stack policy: how it actually works
```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*",       "Principal": "*", "Resource": "*" },
    { "Effect": "Deny",  "Action": "Update:Replace", "Principal": "*", "Resource": "LogicalResourceId/ProdDb" }
  ]
}
```
- Once a stack policy exists, **everything is denied unless allowed**. That's why the first statement allows all updates.
- It applies to everyone, including admins. To override it for one deploy you pass `--stack-policy-during-update-body`, which forces a deliberate act.
- You can't remove a stack policy, only replace it.

## Real world: a typical production database
```yaml
ProdDb:
  Type: AWS::RDS::DBInstance
  DeletionPolicy: Snapshot          # stack deleted or resource removed → final snapshot
  UpdateReplacePolicy: Snapshot     # replaced → snapshot the old one first
  Properties: { ... }
```
Add to that termination protection on the stack, and a stack policy that denies `Update:Replace` on `ProdDb`. Three independent safety nets, each covering a different mistake.

## See it yourself (M2, M5)
1. Add `DeletionPolicy: Retain` to your bucket, delete the stack, and confirm the bucket still exists in S3. Then clean it up by hand.
2. Apply the stack policy above to the network stack (deny `Update:Replace` on the VPC) and try to change the VPC CIDR.

## Takeaway
> Check **"Update requires"** before editing a property. Avoid fixed names so replacement stays possible. Layer `DeletionPolicy` / `UpdateReplacePolicy` (data), termination protection (the stack) and stack policy (dangerous updates). `Retain` orphans resources; it doesn't block the operation.
