# 遵循最小权限原则

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

过于宽松的 IAM 策略会在凭证泄露时扩大爆炸半径。始终只授予资源运行所需的最低权限。

## 错误示例

```hcl
resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"
        Resource = "*"
      }
    ]
  })
}
```

**问题：** Lambda 拥有对所有 AWS 服务的完全访问权限。一旦被攻破，攻击者将获得完全控制权。

## 正确示例

```hcl
resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadFromS3Bucket"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      },
      {
        Sid    = "WriteToSQSQueue"
        Effect = "Allow"
        Action = [
          "sqs:SendMessage"
        ]
        Resource = aws_sqs_queue.notifications.arn
      },
      {
        Sid    = "WriteLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/aws/lambda/${var.function_name}:*"
      }
    ]
  })
}
```

## 最佳实践

1. **精确的操作** - 列出所需的精确 API 操作
2. **精确的资源** - 引用精确的 ARN，不使用通配符
3. **使用条件** - 在适用时添加条件约束
4. **独立语句** - 按用途分组并使用 Sid 标识
5. **定期审计** - 定期审查权限设置

## 使用数据源获取 ARN

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

resource "aws_iam_role_policy" "specific_policy" {
  policy = jsonencode({
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["dynamodb:GetItem", "dynamodb:PutItem"]
        Resource = "arn:aws:dynamodb:${local.region}:${local.account_id}:table/${var.table_name}"
      }
    ]
  })
}
```

## 多云场景适配

### Azure: RBAC 角色分配

```hcl
# Azure: 使用 RBAC 精确授权
# 仅授予特定资源组的 Contributor 权限（而非整个订阅的 Owner）
resource "azurerm_role_assignment" "tf_plan" {
  scope                = azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = var.service_principal_id
}

# 自定义角色实现更细粒度控制
resource "azurerm_role_definition" "tf_custom" {
  name  = "TerraformCustomRole"
  scope = azurerm_resource_group.main.id
  permissions {
    actions     = ["Microsoft.Compute/virtualMachines/*"]
    not_actions = ["Microsoft.Compute/virtualMachines/delete"]
  }
  assignable_scopes = [azurerm_resource_group.main.id]
}
```

### 阿里云: RAM 策略

```hcl
# 阿里云: 使用 RAM 策略最小权限
resource "alicloud_ram_policy" "tf_plan" {
  policy_name = "TerraformPlanPolicy"
  policy_document = jsonencode({
    Statement = [
      {
        Action   = ["ecs:Describe*", "vpc:Describe*", "rds:Describe*"]
        Effect   = "Allow"
        Resource = ["*"]
      }
    ]
    Version = "1"
  })
  description = "Terraform plan 最小权限策略"
}
```

### 腾讯云: CAM 策略

```hcl
# 腾讯云: 使用 CAM 策略最小权限
resource "tencentcloud_cam_policy" "tf_plan" {
  name     = "TerraformPlanPolicy"
  document = jsonencode({
    version = "2.0"
    statement = [
      {
        effect   = "Allow"
        action   = ["cvm:Describe*", "vpc:Describe*", "cdb:Describe*"]
        resource = ["*"]
      }
    ]
  })
  description = "Terraform plan 最小权限策略"
}
```

### 权限模型对比

| 云厂商 | 权限服务 | 策略语言 | 模拟工具 |
|--------|----------|----------|----------|
| AWS | IAM | JSON Policy | IAM Policy Simulator |
| Azure | RBAC + Policy | JSON Role Definition | Azure Policy 测试 |
| 阿里云 | RAM | JSON Policy | RAM 策略模拟 |
| 腾讯云 | CAM | JSON Policy | CAM 策略模拟 |

## 参考资料

- [IAM 最佳实践](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)
- [Azure RBAC 最佳实践](https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices)
- [阿里云 RAM 最佳实践](https://help.aliyun.com/zh/ram/user-guide/best-practices/)
- [腾讯云 CAM 最佳实践](https://cloud.tencent.com/document/product/598/10592)
