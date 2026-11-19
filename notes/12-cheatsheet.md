# 12 — Command cheat sheet

[← Contents](README.md)

All commands assume `export AWS_REGION=us-east-1`.

## Terraform → CloudFormation
| Intent | Terraform | CloudFormation |
|---|---|---|
| Validate | `terraform validate` | `cfn-lint template.yaml` (or `aws cloudformation validate-template --template-body file://template.yaml`, syntax only) |
| Plan | `terraform plan` | `aws cloudformation deploy ... --no-execute-changeset`, then `describe-change-set` |
| Apply a reviewed plan | `terraform apply tfplan` | `aws cloudformation execute-change-set --stack-name S --change-set-name CS` |
| Plan + apply | `terraform apply` | `aws cloudformation deploy --template-file t.yaml --stack-name S` |
| Show state | `terraform state list` | `aws cloudformation describe-stack-resources --stack-name S` |
| Outputs | `terraform output` | `aws cloudformation describe-stacks --stack-name S --query "Stacks[0].Outputs"` |
| Logs of the apply | terminal output | `aws cloudformation describe-stack-events --stack-name S` |
| Refresh / drift | `terraform plan` | `detect-stack-drift`, then `describe-stack-resource-drifts` |
| Import | `terraform import` | `create-change-set --change-set-type IMPORT --resources-to-import ...` |
| Destroy | `terraform destroy` | `aws cloudformation delete-stack --stack-name S` |
| Wait for completion | (synchronous) | `aws cloudformation wait stack-create-complete \| stack-update-complete \| stack-delete-complete --stack-name S` |
| Package code/modules | — | `aws cloudformation package --template-file t.yaml --s3-bucket B --output-template-file packaged.yaml` |

## `deploy` flags you'll actually use
| Flag | Why |
|---|---|
| `--parameter-overrides Env=dev Size=small` | Supply parameters |
| `--capabilities CAPABILITY_IAM` / `CAPABILITY_NAMED_IAM` | Acknowledge the template creates IAM (named IAM = you set the role name) |
| `--capabilities CAPABILITY_AUTO_EXPAND` | Acknowledge macros/transforms (SAM, nested stacks with transforms) |
| `--no-execute-changeset` | Create the change set only, i.e. "plan" |
| `--no-fail-on-empty-changeset` | Exit 0 when nothing changed. **Needed in CI**, because the default is a non-zero exit on no changes |
| `--role-arn arn:aws:iam::...:role/cfn-service-role` | CFN acts as this role, not as you |
| `--tags owner=aakash project=cfn-lab` | Tags propagate to supported resources |
| `--s3-bucket B` | Required when the template exceeds 51,200 bytes |
| `--disable-rollback` | Keep failed resources for debugging (dev only) |

## Handy queries
```bash
# Only the failures, with reasons
aws cloudformation describe-stack-events --stack-name S \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[Timestamp,LogicalResourceId,ResourceStatusReason]" --output table

# Change set summary
aws cloudformation describe-change-set --stack-name S --change-set-name CS \
  --query "Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]" --output table

# All my lab stacks still alive
aws cloudformation list-stacks --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE \
  --query "StackSummaries[?starts_with(StackName,'cfn-lab')].[StackName,StackStatus]" --output table

# Who imports this export?
aws cloudformation list-imports --export-name cfn-lab-network-VpcId

# Protection
aws cloudformation update-termination-protection --enable-termination-protection --stack-name S
aws cloudformation set-stack-policy --stack-name S --stack-policy-body file://policy.json

# Recover a stuck rollback
aws cloudformation continue-update-rollback --stack-name S [--resources-to-skip LogicalId]
```
