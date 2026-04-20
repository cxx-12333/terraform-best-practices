# 使用 Locals 提高代码可读性

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

`locals` 通过为复杂表达式命名来提高代码可读性。它减少重复、使代码自文档化，并集中管理计算值。

## 错误示例

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name        = "${var.project}-${var.environment}-web"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_security_group" "web" {
  name = "${var.project}-${var.environment}-web-sg"

  tags = {
    Name        = "${var.project}-${var.environment}-web-sg"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_lb" "web" {
  name = "${var.project}-${var.environment}-alb"

  tags = {
    Name        = "${var.project}-${var.environment}-alb"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}
```

**问题：**
- 标签块在多个资源间重复
- 命名模式重复多次
- 修改需要更新多处

## 正确示例

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}

resource "aws_security_group" "web" {
  name = "${local.name_prefix}-web-sg"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-sg"
  })
}

resource "aws_lb" "web" {
  name = "${local.name_prefix}-alb"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-alb"
  })
}
```

## Locals 使用场景

### 为复杂表达式命名

```hcl
locals {
  # 代替重复此表达式
  is_production = var.environment == "prod" || var.environment == "production"

  # 计算子网 CIDR
  subnet_cidrs = [for i in range(var.subnet_count) : cidrsubnet(var.vpc_cidr, 8, i)]

  # 构建 ARN 模式
  log_group_arn = "arn:aws:logs:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:log-group:/aws/lambda/${var.function_name}:*"
}

resource "aws_lambda_function" "main" {
  function_name = var.function_name
  # ...

  environment {
    variables = {
      LOG_LEVEL = local.is_production ? "INFO" : "DEBUG"
    }
  }
}
```

### 集中管理计算值

```hcl
data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name

  # 构建资源 ARN
  bucket_arn = "arn:aws:s3:::${var.bucket_name}"
  table_arn  = "arn:aws:dynamodb:${local.region}:${local.account_id}:table/${var.table_name}"
}
```

### 转换输入数据

```hcl
variable "users" {
  type = list(object({
    name  = string
    email = string
    role  = string
  }))
}

locals {
  # 创建查找 map
  users_by_name = { for user in var.users : user.name => user }
  users_by_role = { for user in var.users : user.role => user... }

  # 过滤列表
  admin_users = [for user in var.users : user if user.role == "admin"]

  # 提取值
  all_emails = [for user in var.users : user.email]
}
```

### 条件逻辑

```hcl
variable "enable_https" {
  type    = bool
  default = true
}

variable "custom_domain" {
  type    = string
  default = null
}

locals {
  # 计算派生值
  protocol     = var.enable_https ? "https" : "http"
  default_port = var.enable_https ? 443 : 80

  # 处理可选值
  domain = coalesce(var.custom_domain, "${var.app_name}.example.com")

  # 构建 URL
  app_url = "${local.protocol}://${local.domain}"
}
```

## 组织 Locals

```hcl
locals {
  # 命名
  name_prefix = "${var.project}-${var.environment}"

  # 标签
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

locals {
  # 网络计算
  vpc_cidr        = var.vpc_cidr
  public_subnets  = [for i in range(3) : cidrsubnet(local.vpc_cidr, 4, i)]
  private_subnets = [for i in range(3) : cidrsubnet(local.vpc_cidr, 4, i + 3)]
}

locals {
  # 功能开关
  is_production     = var.environment == "prod"
  enable_monitoring = local.is_production
  enable_backups    = local.is_production
}
```

## 多云场景适配

### 阿里云: Locals 抽象多云命名

```hcl
locals {
  # 多云统一命名前缀
  name_prefix = "${var.project}-${var.environment}"

  # 阿里云常用 tags
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "alicloud_instance" "this" {
  name          = "${local.name_prefix}-web"
  tags          = merge(local.common_tags, { Role = "web" })
}
```

### Azure: Locals 抽象 SKU 和区域

```hcl
locals {
  # Azure SKU 按环境映射
  sku_by_env = {
    dev  = "B_Gen5_1"
    prod = "GP_Gen5_4"
  }

  # 区域别名映射
  region_alias = {
    east  = "eastus"
    west  = "westus2"
    china = "chinaeast2"
  }
}

resource "azurerm_mysql_flexible_server" "this" {
  name                = "${local.name_prefix}-mysql"
  location            = local.region_alias[var.region_alias]
  sku_name            = local.sku_by_env[var.environment]
  resource_group_name = var.resource_group_name
}
```

### 腾讯云: Locals 统一配置

```hcl
locals {
  # 腾讯云可用区映射
  az_map = {
    a = "ap-guangzhou-3"
    b = "ap-guangzhou-4"
  }

  # 按实例规格映射
  instance_types = {
    small  = "SA2.MEDIUM4"
    medium = "SA2.LARGE8"
    large  = "SA2.2XLARGE16"
  }
}
```

## 参考资料

- [Local Values](https://developer.hashicorp.com/terraform/language/values/locals)
- [表达式](https://developer.hashicorp.com/terraform/language/expressions)
