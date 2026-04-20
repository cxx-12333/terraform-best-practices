# 避免使用 HEREDOC 编写 JSON/YAML

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

HEREDOC 字符串编写 JSON、YAML 和 IAM 策略容易出错、难以验证，且无法享受 Terraform 的类型检查。应使用原生函数和资源代替。

## 错误示例

```hcl
resource "aws_iam_role_policy" "lambda" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  # HEREDOC JSON - 难以维护，无验证
  policy = <<-EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::${var.bucket_name}/*"
      }
    ]
  }
  EOF
}

resource "kubernetes_config_map" "config" {
  metadata {
    name = "app-config"
  }

  # HEREDOC YAML - 插值问题，无验证
  data = {
    "config.yaml" = <<-EOF
    database:
      host: ${var.db_host}
      port: 5432
    logging:
      level: info
    EOF
  }
}
```

**问题：**
- 直到 apply 时才能发现语法错误
- 复杂结构难以维护
- 插值可能破坏 JSON/YAML 语法
- 无 IDE 结构支持

## 正确示例

### 使用 jsonencode() 编写 JSON

```hcl
resource "aws_iam_role_policy" "lambda" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::${var.bucket_name}/*"
      }
    ]
  })
}
```

### 使用 IAM Policy Document 资源

```hcl
data "aws_iam_policy_document" "lambda" {
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject"
    ]
    resources = ["arn:aws:s3:::${var.bucket_name}/*"]
  }

  statement {
    effect = "Allow"
    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]
    resources = ["arn:aws:logs:*:*:*"]
  }
}

resource "aws_iam_role_policy" "lambda" {
  name   = "lambda-policy"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.lambda.json
}
```

### 使用 yamlencode() 编写 YAML

```hcl
resource "kubernetes_config_map" "config" {
  metadata {
    name = "app-config"
  }

  data = {
    "config.yaml" = yamlencode({
      database = {
        host = var.db_host
        port = 5432
      }
      logging = {
        level = "info"
      }
    })
  }
}
```

### 使用 templatefile() 处理复杂模板

```hcl
# templates/user-data.sh
#!/bin/bash
echo "Environment: ${environment}"
echo "Region: ${region}"
apt-get update && apt-get install -y ${packages}

# main.tf
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  user_data = templatefile("${path.module}/templates/user-data.sh", {
    environment = var.environment
    region      = var.region
    packages    = join(" ", var.packages)
  })
}
```

## 何时可以使用 HEREDOC

缩进式 HEREDOC（`<<-EOT`）适用于：
- 纯文本描述
- Shell 脚本（当 templatefile 过于复杂时）
- 无结构的多行字符串

```hcl
resource "aws_sns_topic" "alerts" {
  name = "alerts"
}

output "usage_instructions" {
  value = <<-EOT
  订阅告警的步骤：
  1. 进入 AWS 控制台
  2. 导航到 SNS
  3. 订阅主题：${aws_sns_topic.alerts.arn}
  EOT
  description = "告警订阅说明"
}
```

## 参考资料

- [jsonencode 函数](https://developer.hashicorp.com/terraform/language/functions/jsonencode)
- [yamlencode 函数](https://developer.hashicorp.com/terraform/language/functions/yamlencode)
- [templatefile 函数](https://developer.hashicorp.com/terraform/language/functions/templatefile)
- [aws_iam_policy_document](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)
