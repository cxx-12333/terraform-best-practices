# 标记敏感变量

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

密码和 API Key 等敏感值可能通过 CLI 输出、日志和 CI/CD 流水线泄露。将它们标记为敏感，且不要为必填密钥提供默认值。

## 错误示例

```hcl
variable "database_password" {
  type    = string
  default = "" # 空默认值允许跳过必填密钥
}

variable "api_key" {
  type = string
  # 未标记 sensitive - 会显示在 plan 输出中
}

# 在 terraform plan 输出中：
# + password = "actual-secret-value" # 泄露！
```

**问题：** 密钥在 Terraform 输出、CI 日志和状态文件中可见。

## 正确示例

```hcl
variable "database_password" {
  type        = string
  sensitive   = true
  description = "数据库主密码。通过 TF_VAR_database_password 设置。"
  # 无默认值 - 强制用户提供值
}

variable "api_key" {
  type        = string
  sensitive   = true
  description = "外部服务的 API 密钥。"
}

# 可选密钥配合自动生成
variable "random_password" {
  type      = string
  default   = null
  sensitive = true
  description = "密码。如果未提供，将自动生成。"
}

resource "random_password" "generated" {
  count   = var.random_password == null ? 1 : 0
  length  = 32
  special = true
}

locals {
  actual_password = coalesce(var.random_password, try(random_password.generated[0].result, null))
}
```

## Terraform 输出中的敏感值

```hcl
# 在 terraform plan 输出中：
# + password = (sensitive value) # 已保护！
```

## 传递敏感值

### 环境变量

```bash
export TF_VAR_database_password="secure-password-here"
terraform apply
```

### Terraform Cloud/Enterprise 变量

在 UI 或 API 中将变量标记为 "Sensitive"。

### 从密钥管理器获取

```hcl
# 从 AWS Secrets Manager 读取
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/database/password"
}

# 获取的值自动标记为敏感
resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

## 敏感输出

```hcl
# 将包含密钥的输出标记为敏感
output "database_connection_string" {
  value       = "postgres://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive   = true
  description = "数据库连接字符串（包含凭据）"
}
```

## 关于状态文件的警告

即使设置了 `sensitive = true`，密钥值仍以明文存储在 Terraform 状态文件中。通过以下方式保护状态文件：

1. 使用加密的远程后端
2. 限制状态存储的访问权限
3. 使用 Terraform Cloud 的状态加密
4. 永远不要将状态文件提交到版本控制

```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true # 启用服务端加密
  }
}
```

## 参考资料

- [敏感变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
- [敏感输出](https://developer.hashicorp.com/terraform/language/values/outputs#sensitive-suppressing-values-in-cli-output)
