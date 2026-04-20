# 使用数据源替代硬编码

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

数据源动态获取信息，而非硬编码值。这使配置更具可移植性、自文档化，并减少因过时或错误值导致的问题。

## 错误示例

```hcl
# 硬编码值可能过时或不正确
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0" # 什么区域？什么操作系统？还有效吗？
  instance_type = "t3.micro"
  subnet_id     = "subnet-abc123def456"   # 如果变了怎么办？
}

resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::my-bucket/*" # 假设了硬编码的账号
    }]
  })
}

# 硬编码的账号 ID
locals {
  account_id = "123456789012" # 复制粘贴，容易出错
}
```

**问题：**
- AMI ID 因区域而异
- 值会随时间过时
- 硬编码 ID 可能错误
- 无法跨环境移植

## 正确示例

### 动态 AMI 查找

```hcl
# 始终获取当前区域最新的 Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

### 当前账号和区域

```hcl
# 动态获取当前 AWS 账号 ID
data "aws_caller_identity" "current" {}

# 获取当前区域
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::${local.account_id}-app-data/*"
    }]
  })
}
```

### 可用区

```hcl
# 获取当前区域的可用区
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### 引用现有资源

```hcl
# 通过标签查找现有 VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# 查找现有子网
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  filter {
    name   = "tag:Tier"
    values = ["private"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnets.private.ids[0]
}
```

### IAM 策略文档

```hcl
# 使用数据源代替 JSON 字符串
data "aws_iam_policy_document" "s3_read" {
  statement {
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]

    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*"
    ]
  }
}

resource "aws_iam_role_policy" "app" {
  name   = "s3-read-policy"
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.s3_read.json
}
```

### 跨账号数据

```hcl
# 引用其他账号的资源
data "aws_secretsmanager_secret" "shared" {
  provider = aws.shared_services
  name     = "shared/api-key"
}

# 从其他 Terraform 状态引用
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

### 常用数据源

| 用途 | 数据源 |
|------|--------|
| 当前账号 | `aws_caller_identity` |
| 当前区域 | `aws_region` |
| 可用区 | `aws_availability_zones` |
| 最新 AMI | `aws_ami` |
| 现有 VPC | `aws_vpc` |
| 现有子网 | `aws_subnets` |
| IAM 策略 | `aws_iam_policy_document` |
| 密钥 | `aws_secretsmanager_secret_version` |
| SSM 参数 | `aws_ssm_parameter` |
| Route53 区域 | `aws_route53_zone` |
| ACM 证书 | `aws_acm_certificate` |

### GCP 数据源

```hcl
data "google_project" "current" {}

data "google_compute_zones" "available" {
  region = var.region
}

data "google_compute_image" "debian" {
  family  = "debian-11"
  project = "debian-cloud"
}
```

### Azure 数据源

```hcl
data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "existing" {
  name = "my-resource-group"
}
```

## 参考资料

- [数据源](https://developer.hashicorp.com/terraform/language/data-sources)
- [AWS 数据源](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
