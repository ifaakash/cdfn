# cdfn — learning AWS CloudFormation (from a Terraform background)

Hands-on lab + notes for understanding CloudFormation by building, then mapping every concept back to Terraform.

- **Notes (start here):** [`notes/README.md`](notes/README.md), with a table of contents, the Terraform ↔ CloudFormation map, and one note per concept
- **Region:** `us-east-1` · **Stack prefix:** `cfn-lab-*` · **Budget:** < $2

## Lab layout
| Folder | Module | What it covers |
|---|---|---|
| `01-basics/` | M1–M2 | First stack, events, rollback, change sets |
| `02-params/` | M3 | Parameters, Mappings, Conditions |
| `03-network/`, `04-app/` | M4–M5 | Cross-stack exports, EC2, drift, stack policy |
| `05-lambda/` | M6 | Packaging, IAM capabilities, custom resource |
| `06-nested/` | M7 | Nested stacks |
| `iam/`, `.github/workflows/` | M8 | CFN service role + GitHub Actions (OIDC) pipeline |
