# Muyu invoice domain

Terraform configuration for delegating subdomains of `muyuinvoices.click` to
participant-owned public Route 53 hosted zones.

```text
Namecheap registration
  -> Route 53: muyuinvoices.click
      -> NS delegation: alice.muyuinvoices.click
          -> Alice's Route 53 hosted zone and Terraform
```

## One-time setup

1. Create the S3 bucket configured in `backend.tf`, with encryption and
   versioning enabled.
2. Configure a GitHub OIDC provider and a role that can access the state
   object and manage the parent Route 53 zone.
3. Add the role ARN as the repository Actions variable
   `TERRAFORM_ROLE_ARN`.
4. Merge the initial configuration. The Apply workflow creates the parent
   hosted zone and prints `parent_name_servers`.
5. At Namecheap, set the domain's custom nameservers to those four Route 53
   nameservers. Copy any existing DNS records to Route 53 before changing
   nameservers.

## Add a participant subdomain

The participant first creates a **public** Route 53 hosted zone in their own
AWS account. For `alice.muyuinvoices.click`:

```hcl
resource "aws_route53_zone" "participant" {
  name = "alice.muyuinvoices.click"
}

output "name_servers" {
  value = aws_route53_zone.participant.name_servers
}
```

They then open a pull request that adds their label and all assigned
nameservers to `tenants.auto.tfvars.json`:

```json
{
  "tenants": {
    "alice": [
      "ns-123.awsdns-45.com",
      "ns-456.awsdns-78.net",
      "ns-789.awsdns-12.org",
      "ns-321.awsdns-34.co.uk"
    ]
  }
}
```

## Workflows

Pull requests validate Terraform; changes merged to `main` apply through
GitHub OIDC.

Do not run `terraform apply` locally. The remote state and all infrastructure
changes are owned by GitHub Actions.

## Local checks

```sh
terraform fmt -check
terraform init -backend=false
terraform validate
```

## Review policy

`.github/CODEOWNERS` assigns all repository files to the organizers team.
Enable **Require review from Code Owners** in the `main` branch ruleset to
enforce this policy.
