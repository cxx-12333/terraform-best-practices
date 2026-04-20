# 使用正确的凭证管理（OIDC、Vault、IAM 角色）

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

基础设施 Provider 凭证拥有对系统的强大访问权限。不当的凭证管理会导致数据泄露、未授权访问和合规违规。

## 错误示例

```hcl
# 代码中硬编码凭证
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# 凭证写在 terraform.tfvars 中并提交到 git
aws_access_key = "AKIAIOSFODNN7EXAMPLE"
aws_secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# 团队共享凭证
# "用 wiki 上的管理员凭证就行"
```

**问题：**
- 凭证永久留在版本控制历史中
- 无问责机制（共享凭证）
- 无轮换机制
- 权限过宽（所有人使用相同凭证）

## 正确示例

### 优先级排序（从最佳到可接受）

1. **短期令牌**（OIDC、STS）
2. **密钥管理系统**（Vault、AWS Secrets Manager）
3. **实例/工作负载身份**（IAM 角色、服务账号）
4. **环境变量**（不写入代码）

### 方案 1：OIDC 联邦（推荐用于 CI/CD）

```yaml
# GitHub Actions 使用 OIDC - 无需存储凭证
name: Deploy
on: push

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
          # 无需 Access Key！

      - run: terraform apply -auto-approve
```

### 方案 2：密钥管理系统（Vault）

```hcl
# 从 Vault 获取凭证
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "terraform-role"
  type    = "sts" # 短期 STS 凭证
}

provider "aws" {
  region     = "us-east-1"
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  token      = data.vault_aws_access_credentials.creds.security_token
}
```

### 方案 3：实例/工作负载身份

```hcl
# 在 EC2 上使用实例配置文件 - 代码中无需凭证
provider "aws" {
  region = "us-east-1"
  # 自动使用实例配置文件凭证
}

# 在 GKE 上使用 Workload Identity
provider "google" {
  project = "my-project"
  region  = "us-central1"
  # 自动使用绑定到 Pod 的服务账号
}
```

### 方案 4：环境变量

```bash
# 在环境中设置凭证（不写入代码）
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"

# 或使用 AWS CLI 配置文件
export AWS_PROFILE="company-prod"

terraform plan
```

```hcl
# Provider 自动使用环境变量
provider "aws" {
  region = "us-east-1"
  # 未指定凭证 - 自动使用 AWS_* 环境变量
}
```

### 凭证轮换

```hcl
# 使用 IAM Access Key 轮换
resource "aws_iam_access_key" "terraform" {
  user = aws_iam_user.terraform.name

  lifecycle {
    create_before_destroy = true
  }
}

# 存储到 Secrets Manager 并配置轮换
resource "aws_secretsmanager_secret_rotation" "terraform_creds" {
  secret_id           = aws_secretsmanager_secret.terraform_creds.id
  rotation_lambda_arn = aws_lambda_function.rotate_creds.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

### 多账号 AssumeRole

```hcl
# 中央身份账号，假设到目标账号
provider "aws" {
  alias  = "prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT:role/TerraformRole"
    session_name = "terraform-prod"
    external_id  = var.external_id # 额外安全保障
  }
}
```

### 绝不提交凭证

```gitignore
# .gitignore
*.tfvars
!example.tfvars
.env
.env.*
credentials*
*.pem
*.key
```

## 多云场景适配

### Azure: Managed Identity + Key Vault

```hcl
# Azure: 使用 Managed Identity 自动认证
provider "azurerm" {
  features {}
  use_msi = true  # 在 Azure VM/ACI 上自动获取凭证
}

# 从 Key Vault 读取密钥
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = var.key_vault_id
}
```

### 阿里云: RAM 角色 + KMS

```hcl
# 阿里云: 通过 ECS 实例 RAM 角色自动获取临时凭证
# 或通过 OIDC 联邦身份认证（类似 AWS）
provider "alicloud" {
  # RAM 角色模式下无需配置 access_key
}

# 使用 KMS 密钥管理服务
data "alicloud_kms_secret" "db_password" {
  secret_name = "db-password"
}
```

### 腾讯云: CAM 角色 + Secrets Manager

```hcl
# 腾讯云: 在 CVM/TKE 上通过 CAM 角色自动获取临时凭证
provider "tencentcloud" {
  # CAM 角色模式下无需配置 secret_id/secret_key
}

# 使用 Secrets Manager 管理密钥
data "tencentcloud_ssm_secret" "db_password" {
  secret_name = "db-password"
}
```

### 凭证管理多云对比

| 云厂商 | 身份认证 | 密钥管理 | OIDC 支持 |
|--------|----------|----------|-----------|
| AWS | IAM Role | Secrets Manager / Parameter Store | 支持 |
| Azure | Managed Identity | Key Vault | 支持 |
| 阿里云 | RAM 角色 | KMS 密钥管理 | 支持 |
| 腾讯云 | CAM 角色 | Secrets Manager | 支持 |
| 通用 | Vault | Vault Secrets Engine | 支持 |

## 参考资料

- [HashiCorp 凭证管理](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part2#q9-how-are-infrastructure-service-provider-credentials-managed)
- [AWS OIDC Provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [Vault AWS Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/aws)
- [Azure Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
- [阿里云 RAM OIDC](https://help.aliyun.com/zh/ram/user-guide/oidc-federation/)
