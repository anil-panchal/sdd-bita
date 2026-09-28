# Terraform Security Reference

## Overview

When reviewing Terraform, focus on secret handling, state exposure, provider and module trust, command execution, and the security posture of the infrastructure being declared.

---

## Sensitive Data and State

### Safe Patterns (Do Not Flag)

```hcl
variable "database_password" {
  type      = string
  sensitive = true
}

variable "api_token" {
  type      = string
  sensitive = true
  ephemeral = true
}

terraform {
  backend "s3" {
    bucket  = "secure-tf-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true
  }
}
```

### Flag These

```hcl
variable "database_password" {
  default = "super-secret-password" # FLAG
}

provider "aws" {
  access_key = "AKIA..." # FLAG
  secret_key = "plain-text-secret" # FLAG
}

output "db_password" {
  value = var.database_password # CHECK if not marked sensitive
}
```

### Review Notes

- `sensitive = true` redacts values in CLI and UI, but does not prevent them from being stored in state.
- `ephemeral = true` is stronger for short-lived secrets because Terraform omits those values from state and plan files.
- Committed `.tfstate` files or secret defaults are high-confidence findings.

---

## Providers, Modules, and Dependency Trust

### Safe Patterns (Do Not Flag)

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.7.0"
}
```

### Check Context

```hcl
module "vpc" {
  source = "git::https://example.com/terraform-modules.git//vpc?ref=main" # CHECK
}

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0" # CHECK: broad constraint, rely on lock file review
    }
  }
}
```

### Review Notes

- Prefer pinned module versions and reviewed provider upgrades.
- `.terraform.lock.hcl` should be committed and reviewed when provider versions change.
- Floating git refs and broad version ranges increase supply-chain and change-management risk.

---

## Provisioners and Command Execution

### Always Flag

```hcl
resource "null_resource" "deploy" {
  provisioner "local-exec" {
    command = "bash -c '${var.user_input}'" # FLAG: command injection
  }
}
```

### Check Context

```hcl
resource "null_resource" "template" {
  provisioner "local-exec" {
    command = "scripts/render.sh ${var.environment}" # CHECK
  }
}

resource "aws_instance" "app" {
  provisioner "remote-exec" {
    inline = [
      "sudo systemctl restart app"
    ]
  }
}
```

### Review Notes

- Provisioners are operationally fragile and often bypass safer platform-native workflows.
- If command strings include attacker-controlled or unvalidated variables, treat them as command injection risk.

---

## Public Exposure and Access Control

### Flag These

```hcl
resource "aws_security_group_rule" "ssh_from_world" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"] # FLAG
}

resource "aws_iam_policy" "admin" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  }) # FLAG
}

resource "aws_s3_bucket_public_access_block" "bad" {
  bucket                  = aws_s3_bucket.app.id
  block_public_acls       = false # CHECK / FLAG
  block_public_policy     = false # CHECK / FLAG
  ignore_public_acls      = false # CHECK / FLAG
  restrict_public_buckets = false # CHECK / FLAG
}
```

### Check Context

```hcl
resource "aws_db_instance" "main" {
  publicly_accessible = true # CHECK
}

resource "azurerm_network_security_rule" "rdp" {
  access                     = "Allow"
  destination_port_range     = "3389"
  source_address_prefix      = "*" # FLAG
}

resource "google_compute_firewall" "wide_open" {
  source_ranges = ["0.0.0.0/0"] # CHECK
}
```

### Review Notes

- Public access is not always wrong, but admin ports, wildcard IAM, and disabled public-access protections are strong findings.
- Review cloud-provider equivalents even if the exact resource names differ.

---

## Secrets in Outputs and Logs

### Flag These

```hcl
output "connection_string" {
  value = "postgres://${var.db_user}:${var.database_password}@${aws_db_instance.main.address}/app"
}
```

### Safer Pattern

```hcl
output "connection_string" {
  value     = "postgres://${var.db_user}:${var.database_password}@${aws_db_instance.main.address}/app"
  sensitive = true
}
```

### Review Notes

- Sensitive outputs can still be retrieved by users with enough Terraform access, but they reduce accidental exposure in plans and logs.
- Root module outputs cannot use `ephemeral`, so access controls on state and runners still matter.

---

## Policy and Review Workflow

### Review Aids

```bash
terraform fmt -check
terraform validate
terraform plan
```

### Review Notes

- Always inspect plan output when provider versions, modules, or security-sensitive resources change.
- Policy-as-code, IaC scanning, and drift detection are strong complements, but the code review should still inspect trust boundaries and hardcoded secrets directly.

---

## rg Patterns

```bash
# Hardcoded secrets and provider credentials
rg -n 'access_key|secret_key|client_secret|token\s*=|password\s*=' -g '*.tf'

# Sensitive and ephemeral handling
rg -n 'sensitive\s*=\s*true|ephemeral\s*=\s*true|output "' -g '*.tf'

# Provisioners and shell execution
rg -n 'local-exec|remote-exec|null_resource' -g '*.tf'

# Broad public exposure
rg -n '0\.0\.0\.0/0|::/0|publicly_accessible\s*=\s*true|source_address_prefix\s*=\s*"\*"' -g '*.tf'

# Wildcard IAM or policy documents
rg -n 'Action"\s*:\s*"\*"|Resource"\s*:\s*"\*"|actions\s*=\s*\["\*"\]|resources\s*=\s*\["\*"\]' -g '*.tf'

# Provider and module pinning
rg -n 'required_providers|version\s*=|source\s*=' -g '*.tf' .terraform.lock.hcl
```

---

## References

- [Terraform Sensitive Data Guidance](https://developer.hashicorp.com/terraform/language/state/sensitive-data)
- [Terraform Provider Versioning and Lock Files](https://developer.hashicorp.com/terraform/tutorials/configuration-language/provider-versioning)
