# 03 — Intrinsic functions & pseudo parameters

[← Contents](README.md)

## The problem
When you write the template you don't know:
- the bucket's real name (CFN may generate it),
- the VPC ID (it doesn't exist yet),
- the account ID or region (the same template should work anywhere).

You need a way to say "use whatever value *that* resource ends up with". Intrinsic functions do two jobs:
1. **Reference** another resource. This also tells CFN the order to create things in: the dependency graph comes from `!Ref` and `!GetAtt`.
2. **Glue strings and values** together (`!Sub`, `!Join`, `!Select`).

That's the whole expression language. It's deliberately small (see [02](02-templates.md) for why).

## The four you'll use 90% of the time

### `!Ref`: "give me its identifier"
- On a **parameter**, returns its value. Terraform: `var.env`.
- On a **resource**, returns its *default identifier*. **What that is varies by type**:

| Resource | `!Ref` returns |
|---|---|
| `AWS::S3::Bucket` | bucket name |
| `AWS::SQS::Queue` | queue **URL** |
| `AWS::EC2::Instance` | instance ID |
| `AWS::IAM::Role` | role **name** (not ARN) |

Why it's inconsistent: each service team picked its "primary identifier" years ago, and it can't change without breaking existing templates. So check the resource's **Return values** section; don't guess.

### `!GetAtt`: "give me a specific attribute"
`!GetAtt LogsBucket.Arn`, `!GetAtt Role.Arn`. Terraform: `aws_s3_bucket.logs.arn`.
Rule: use `!Ref` when you want the name or ID, `!GetAtt` for anything else (usually the ARN).

### `!Sub`: string interpolation
```yaml
BucketName: !Sub "${AWS::StackName}-${Env}-logs"
PolicyArn:  !Sub "arn:${AWS::Partition}:s3:::${LogsBucket}/*"   # ${LogsBucket} = !Ref
RoleArn:    !Sub "${Role.Arn}"                                   # ${X.Attr} = !GetAtt
```
This is the most-used function. It replaces most uses of `!Join`.

### `!If` + Conditions
`!If [IsProd, 3, 1]`, the ternary operator. See [04](04-parameters-mappings-conditions-outputs.md).

## The rest, mapped to Terraform
| CFN | Terraform | When you reach for it |
|---|---|---|
| `!Join [",", [a, b]]` | `join()` | Building lists into strings |
| `!Split`, `!Select [0, list]` | `split()`, `list[0]` | Picking an AZ: `!Select [0, !GetAZs ""]` |
| `!GetAZs ""` | `data "aws_availability_zones"` | Spreading subnets across AZs |
| `!FindInMap` | `lookup(local.map, k)` | Per-env sizing (see [04](04-parameters-mappings-conditions-outputs.md)) |
| `!Equals`, `!And`, `!Or`, `!Not` | `==`, `&&`, `\|\|`, `!` | Defining Conditions |
| `!ImportValue` | `terraform_remote_state` | Reading another stack's export (see [09](09-composing-stacks.md)) |
| `Fn::Base64` | `base64encode()` | EC2 UserData |
| `!Cidr` | `cidrsubnets()` | Carving subnets from a VPC CIDR |

YAML gotcha: two short tags can't sit on one node. `!Base64 !Sub ...` is invalid, so write:
```yaml
UserData:
  Fn::Base64: !Sub |
    #!/bin/bash
    echo "stack=${AWS::StackName}"
```

## Pseudo parameters: why they exist
They're values AWS fills in without you declaring them. Their purpose is **portability**: the same template deploys into any account or region without edits. That's essential for StackSets, which push one template to 50 accounts.

| Pseudo parameter | Use it instead of | Terraform |
|---|---|---|
| `AWS::AccountId` | Hardcoded account IDs in ARNs | `data.aws_caller_identity` |
| `AWS::Region` | Hardcoded `us-east-1` | `data.aws_region` |
| `AWS::StackName` | Hardcoded name prefixes (lets you deploy the same template twice) | — |
| `AWS::Partition` | `arn:aws:` (breaks in GovCloud/China) | `data.aws_partition` |
| `AWS::NoValue` | — removes a property entirely when used in `!If` | `null` |

## Real world
- IAM policies built with `!Sub "arn:${AWS::Partition}:s3:::${Bucket}/*"`. They work in every partition and account.
- Name everything with `${AWS::StackName}` so `cfn-lab-dev` and `cfn-lab-prod` can sit in the same account without colliding.

## See it yourself (M1)
In your bucket template, output both `!Ref Bucket` and `!GetAtt Bucket.Arn`. **Predict both values before you deploy**, then check with `describe-stacks`.

## Takeaway
> `!Ref` gives the identifier (which varies by type, so check the docs), `!GetAtt` gives an attribute, and `!Sub` covers most string building. These references also build the dependency graph.
