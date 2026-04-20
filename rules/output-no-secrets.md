# 不要在输出中暴露密钥

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

输出会被记录、显示在 CI/CD 流水线中、存储在状态中，并可通过 `terraform output` 访问。输出中的密钥很容易泄露给未授权方。

## 错误示例

```hcl
output "database_password" {
  value = random_password.db.result
  # 密钥在 terraform output 中暴露！
}

output "api_credentials" {
  value = {
    client_id     = var.client_id
    client_secret = var.client_secret # 密钥泄露！
  }
}

output "connection_string" {
  value = "postgres://admin:${random_password.db.result}@${aws_db_instance.main.endpoint}/mydb"
  # 密码嵌入在输出中！
}
```

**问题：** 运行 `terraform output` 或查看 CI 日志会暴露密钥。

## 正确示例

### 将密钥存储到密钥管理器

```hcl
# 生成密码
resource "random_password" "db" {
  length  = 32
  special = true
}

# 存储到 AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name = "${var.environment}/database/password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}

# 输出位置，而非值
output "database_password_secret_arn" {
  value       = aws_secretsmanager_secret.db_password.arn
  description = "包含数据库密码的 Secrets Manager 密钥 ARN"
}

output "database_password_secret_name" {
  value       = aws_secretsmanager_secret.db_password.name
  description = "包含数据库密码的 Secrets Manager 密钥名称"
}
```

### 存储到 SSM Parameter Store

```hcl
resource "aws_ssm_parameter" "db_password" {
  name        = "/${var.environment}/database/password"
  description = "数据库主密码"
  type        = "SecureString"
  value       = random_password.db.result

  tags = var.tags
}

output "database_password_parameter_name" {
  value       = aws_ssm_parameter.db_password.name
  description = "数据库密码的 SSM 参数名。获取方式：aws ssm get-parameter --name ${aws_ssm_parameter.db_password.name} --with-decryption"
}
```

### 提供指引而非值

```hcl
output "database_credentials_info" {
  value       = "数据库凭据存储在 SSM 的 /${var.environment}/database/*"
  description = "SSM Parameter Store 中数据库凭据的位置"
}

output "retrieve_password_command" {
  value       = "aws ssm get-parameter --name '/${var.environment}/database/password' --with-decryption --query 'Parameter.Value' --output text"
  description = "获取数据库密码的 AWS CLI 命令"
}
```

## 如果必须输出密钥

在极少数必须输出密钥的场景（如引导初始化），始终标记为敏感：

```hcl
output "initial_admin_password" {
  value       = random_password.admin.result
  sensitive   = true
  description = "初始管理员密码。首次登录后请立即修改。"
}
```

**注意：** 即使设置了 `sensitive = true`：
- 值仍然以明文存储在状态文件中
- 可以通过 `terraform output -json` 获取
- 最好完全避免

## 根模块 vs 可复用模块

- **根模块（组件）：** 永远不要输出密钥
- **可复用模块：** 如果组合需要，可以输出密钥，但必须标记为 `sensitive = true`

```hcl
# 可复用模块中 - 标记 sensitive 后可接受
output "generated_password" {
  value       = random_password.this.result
  sensitive   = true
  description = "供父模块使用的生成密码"
}

# 根模块中 - 存储而非输出
resource "aws_ssm_parameter" "password" {
  name  = "/app/password"
  type  = "SecureString"
  value = module.database.generated_password
}
```

## 参考资料

- [敏感输出](https://developer.hashicorp.com/terraform/language/values/outputs#sensitive-suppressing-values-in-cli-output)
- [AWS Secrets Manager](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret)
- [AWS SSM Parameter Store](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter)
