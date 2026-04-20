# 使用一致的命名规范

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

一致的命名规范提高可读性、便于查找资源，并有助于成本分摊和合规。尽早建立命名模式并在整个基础设施中强制执行。

## 错误示例

```hcl
resource "aws_instance" "my_server" {
  # ...
}

resource "aws_instance" "WebServer1" {
  # ...
}

resource "aws_s3_bucket" "Data_Bucket" {
  bucket = "mycompany-stuff"
}

resource "aws_lambda_function" "func" {
  function_name = "DoTheThing"
}
```

**问题：**
- 大小写不一致（snake_case、PascalCase、混合使用）
- 通用名称无法表达用途
- 资源名称与云资源名称不匹配

## 正确示例

```hcl
# Terraform 资源名称：snake_case，具有描述性
resource "aws_instance" "web_server" {
  tags = {
    Name = "prod-web-server-001" # 云资源名：环境-用途-编号
  }
}

resource "aws_s3_bucket" "application_data" {
  bucket = "acme-prod-application-data-us-east-1" # 组织-环境-用途-区域
}

resource "aws_lambda_function" "order_processor" {
  function_name = "prod-order-processor"
}

resource "aws_security_group" "web_server_sg" {
  name        = "prod-web-server-sg"
  description = "生产环境 Web 服务器安全组"
}
```

## 推荐命名模式

```
{org}-{environment}-{purpose}-{region}-{instance}
```

| 组成部分 | 示例 | 是否必填 |
|----------|------|----------|
| org | acme, myco | 可选 |
| environment | prod, staging, dev | 是 |
| purpose | web, api, db, cache | 是 |
| region | use1, usw2, euw1 | 视情况 |
| instance | 001, blue, primary | 视情况 |

## Terraform 资源名称

```hcl
# 使用 snake_case 命名 Terraform 资源
resource "aws_vpc" "main" {}             # 简单，单个 VPC
resource "aws_vpc" "application" {}      # 描述性用途
resource "aws_subnet" "public_a" {}      # 包含区域/用途
resource "aws_subnet" "private_database" {}

# 避免
resource "aws_vpc" "VPC" {}              # 冗余，大小写不当
resource "aws_subnet" "subnet1" {}       # 无描述性
```

## 使用 Locals 保持一致性

```hcl
locals {
  name_prefix = "${var.org}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-server"
  })
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs-${data.aws_region.current.name}"
  tags   = local.common_tags
}
```

## 参考资料

- [AWS 标签最佳实践](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html)
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)
