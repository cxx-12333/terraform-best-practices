# 为所有资源添加标签

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

标签是实现成本分摊、资源组织、自动化和合规的基础。没有一致的标签策略，你将无法按团队追踪成本、确定资源所有者或实现运维自动化。

## 错误示例

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  # 无标签 - 无法追踪归属或成本
}

resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"

  tags = {
    Name = "data" # 最小化、不一致的标签
  }
}
```

**问题：**
- 无法确定资源所有者
- 无法将成本分摊到团队/项目
- 无法识别用于自动化的资源
- 合规违规

## 正确示例

### 定义标准标签

```hcl
locals {
  required_tags = {
    Environment = var.environment
    Project     = var.project
    Owner       = var.owner
    ManagedBy   = "terraform"
    CostCenter  = var.cost_center
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = merge(local.required_tags, {
    Name = "${var.project}-${var.environment}-web"
    Role = "webserver"
  })
}

resource "aws_s3_bucket" "data" {
  bucket = "${var.project}-${var.environment}-data"

  tags = merge(local.required_tags, {
    Name     = "${var.project}-${var.environment}-data"
    DataClass = "internal"
    Backup   = "daily"
  })
}
```

### 标准标签模式

| 标签 | 是否必填 | 说明 | 示例 |
|------|----------|------|------|
| Environment | 是 | 部署环境 | prod, staging, dev |
| Project | 是 | 项目或应用名称 | myapp |
| Owner | 是 | 团队或个人负责人 | platform-team |
| ManagedBy | 是 | 资源管理方式 | terraform |
| CostCenter | 是 | 成本中心代码 | CC-12345 |
| Name | 推荐 | 人类可读名称 | myapp-prod-web |

### 使用默认标签（AWS Provider 3.38+）

```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      Owner       = var.owner
      ManagedBy   = "terraform"
      CostCenter  = var.cost_center
    }
  }
}

# 所有资源自动获得默认标签
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "${var.project}-${var.environment}-web"
    Role = "webserver"
  }
}
```

### 标签模块模式

创建可复用模块以保持标签一致性：

```hcl
# 标签模块的变量
variable "environment" {
  type        = string
  description = "环境名称"
}

variable "project" {
  type        = string
  description = "项目名称"
}

variable "owner" {
  type        = string
  description = "资源所有者"
}

variable "cost_center" {
  type        = string
  description = "成本中心代码"
}

variable "additional_tags" {
  type        = map(string)
  default     = {}
  description = "额外的标签"
}

# 输出合并后的标签
output "tags" {
  value = merge({
    Environment = var.environment
    Project     = var.project
    Owner       = var.owner
    ManagedBy   = "terraform"
    CostCenter  = var.cost_center
  }, var.additional_tags)
}
```

### 在根模块中使用

```hcl
module "tags" {
  source = "./modules/tags"

  environment = "prod"
  project     = "myapp"
  owner       = "platform-team"
  cost_center = "CC-12345"
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = merge(module.tags.tags, {
    Name = "myapp-prod-web"
  })
}
```

### 使用策略强制标签

```hcl
# AWS Config 规则强制标签
resource "aws_config_config_rule" "required_tags" {
  name = "required-tags"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    tag1Key = "Environment"
    tag2Key = "Project"
    tag3Key = "Owner"
    tag4Key = "CostCenter"
  })
}
```

### 在 CI 中验证标签

```yaml
# .github/workflows/terraform.yml
- name: Check Required Tags
  run: |
    # 使用 tfsec 或自定义脚本验证标签
    tfsec . --tfvars-file=prod.tfvars
```

## 参考资料

- [AWS 标签最佳实践](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html)
- [AWS 默认标签](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#default_tags)
- [成本分摊标签](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)
