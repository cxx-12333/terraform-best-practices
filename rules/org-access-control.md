# 控制谁可以更改什么基础设施

**优先级：** 高
**分类：** 组织与工作流

## 为什么重要

并非所有人都应该能够修改所有基础设施。访问控制确保变更由授权人员执行，保护生产环境，并建立问责机制。

## 错误示例

```bash
# 每个人对所有资源都有管理员权限
# 共享的 AWS 凭证放在 wiki 上
# 开发和生产环境的访问权限没有区分
# 任何人都可以在生产环境运行 terraform apply
```

**问题：**
- 变更无问责
- 意外修改生产环境
- 没有职责分离
- 合规违规

## 正确示例

### 最小权限原则

为每个角色授予最低必要权限：

| 角色 | 开发环境 | 预发布环境 | 生产环境 |
|------|----------|-----------|----------|
| 初级工程师 | Apply | 仅 Plan | 只读 |
| 高级工程师 | Apply | Apply | Plan + 审查 |
| 技术负责人 | Admin | Admin | Apply |
| 平台团队 | Admin | Admin | Admin |

### 每个环境使用独立凭证

```hcl
# 开发账号
provider "aws" {
  alias   = "dev"
  region  = "us-east-1"
  profile = "company-dev" # 有限权限
}

# 生产账号
provider "aws" {
  alias   = "prod"
  region  = "us-east-1"
  profile = "company-prod" # 需要 MFA，更严格的控制
}
```

### 使用 IAM 角色，而非长期密钥

```hcl
# CI/CD 通过角色假设获取有限权限
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789:role/TerraformDeployRole"
    session_name = "terraform-ci"
  }
}
```

### 环境级 IAM 策略

```hcl
# 开发环境角色 - 较宽泛的权限
resource "aws_iam_role" "terraform_dev" {
  name = "terraform-dev-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.dev_account_id}:root" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# 生产环境角色 - 受限权限
resource "aws_iam_role" "terraform_prod" {
  name = "terraform-prod-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.prod_account_id}:root" }
      Action    = "sts:AssumeRole"
      Condition = {
        Bool = { "aws:MultiFactorAuthPresent" = "true" } # 要求 MFA
      }
    }]
  })
}
```

### 限制直接控制台/CLI 访问

基础设施由 Terraform 管理后：

1. 移除大多数用户的直接控制台访问权限
2. 使用 Terraform 作为变更的唯一工作流
3. 启用只读控制台访问用于故障排查
4. 审计任何手动变更

```hcl
# 控制台只读访问策略
data "aws_iam_policy_document" "readonly" {
  statement {
    effect = "Allow"
    actions = [
      "ec2:Describe*",
      "rds:Describe*",
      "s3:Get*",
      "s3:List*"
    ]
    resources = ["*"]
  }

  statement {
    effect = "Deny"
    actions = [
      "ec2:*",
      "rds:*",
      "s3:Put*",
      "s3:Delete*"
    ]
    resources = ["*"]
  }
}
```

### Git 分支保护

```yaml
# .github/CODEOWNERS
# 生产环境变更需要平台团队审查
*prod* @platform-team
# 共享模块需要审查
**/modules/** @platform-team @senior-engineers
```

### CI/CD 访问控制

```yaml
# GitHub Actions - 不同环境不同权限
jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    environment: dev # 无需审批

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    environment: prod
    # 需要在 GitHub 环境设置中手动审批
```

### 多账号策略

```
组织
├── 管理账号（账单、组织策略）
├── 安全账号（审计日志、安全工具）
├── 共享服务账号（CI/CD、制品存储）
├── 开发账号（开发工作负载）
├── 预发布账号（预发布测试）
└── 生产账号（生产工作负载）
```

## 支持访问控制的后端选项

- **S3 + IAM** - AWS 原生，使用 IAM 策略
- **GCS + IAM** - GCP 原生，使用 IAM 策略
- **Azure Blob + RBAC** - Azure 原生
- **Terraform Cloud/Enterprise** - 内置团队权限
- **Terramate Cloud** - GitOps 原生，栈级权限

## 参考资料

- [AWS 多账号策略](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- [HashiCorp 访问控制建议](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part1#personas-responsibilities-and-desired-user-experiences)
