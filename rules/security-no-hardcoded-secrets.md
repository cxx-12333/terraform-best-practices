# 永远不要在代码中硬编码密钥

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

硬编码在 Terraform 代码中的密钥会被提交到版本控制系统、记录在 CI 输出日志中，并存储在状态文件里。这会造成严重的安全漏洞。

## 错误示例

```hcl
resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"

  # 绝对不要这样做
  username = "admin"
  password = "SuperSecret123!"
}

provider "aws" {
  region = "us-east-1"
  # 绝对不要这样做
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

**问题：** 凭证暴露在代码、状态文件和 git 历史中。

## 正确示例

**方案 1：环境变量**

```hcl
variable "db_password" {
  type        = string
  sensitive   = true
  description = "数据库密码 - 通过 TF_VAR_db_password 设置"
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "admin"
  password       = var.db_password
}
```

**方案 2：密钥管理服务**

```hcl
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "prod/database/credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = local.db_creds.username
  password       = local.db_creds.password
}
```

**方案 3：随机密码生成**

```hcl
resource "random_password" "db_password" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "admin"
  password       = random_password.db_password.result
}
```

## 补充说明

- 将敏感变量标记为 `sensitive = true`
- 使用 `.gitignore` 排除包含密钥的 `.tfvars` 文件
- 考虑使用 SOPS 或 sealed-secrets 加密敏感值

## 参考资料

- [敏感变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
- [AWS Secrets Manager](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/secretsmanager_secret_version)
