# Terraform AWS Landing Zone

A reusable Terraform module for standing up a secure, production-ready AWS landing zone — VPC design, IAM baseline, centralized logging, and account guardrails. Maintained by [Sygitech](https://sygitech.com).

## What this module sets up

- **Networking** — multi-AZ VPC with public/private subnets, NAT gateways, route tables
- **IAM baseline** — least-privilege roles, break-glass access pattern, password policy
- **Logging & audit** — CloudTrail, VPC Flow Logs, centralized log bucket with lifecycle policies
- **Guardrails** — AWS Config rules and SCPs for common compliance baselines
- **Tagging strategy** — consistent resource tagging for cost allocation

## Why use this instead of building from scratch

Setting up a secure AWS foundation correctly takes real expertise — this module encodes patterns we use across client engagements so you don't have to reinvent them.

## Quick start

```hcl
module "landing_zone" {
  source  = "sygitech/landing-zone/aws"
  version = "~> 1.0"

  environment = "production"
  region      = "us-east-1"
  vpc_cidr    = "10.0.0.0/16"
}
```

```bash
terraform init
terraform plan
terraform apply
```

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.5 |
| aws provider | >= 5.0 |

## Structure

```
├── modules/
│   ├── networking/
│   ├── iam/
│   ├── logging/
│   └── guardrails/
├── examples/
│   └── basic/
└── variables.tf
```

## Contributing

Issues and PRs welcome, especially real-world edge cases from your own AWS environments.

## About Sygitech

Sygitech provides AWS cloud migration and managed services, including landing zone setup for teams starting fresh or cleaning up existing accounts. [sygitech.com](https://sygitech.com)

## License

MIT
