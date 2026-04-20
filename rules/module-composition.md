# 像搭积木一样组合模块

**优先级：** 高
**分类：** 模块设计

## 为什么重要

可组合的模块可以像积木一样拼装出复杂的基础设施。这提高了复用性、减少了重复代码，并使基础设施更易于理解和维护。

## 错误示例

```hcl
# 做所有事情的单体模块
module "infrastructure" {
  source = "./modules/infrastructure"

  # VPC 设置
  vpc_cidr = "10.0.0.0/16"

  # EKS 设置
  cluster_name = "prod"
  node_count   = 3

  # RDS 设置
  db_name          = "myapp"
  db_instance_class = "db.t3.medium"

  # ElastiCache 设置
  cache_node_type = "cache.t3.micro"

  # ... 还有 50 个变量
}
```

**问题：**
- 无法单独使用 VPC 而不创建 EKS、RDS 等
- 任何组件的变更都影响整个模块
- 测试需要部署所有内容
- 难以理解和维护

## 正确示例

### 设计具有清晰输出的模块

每个模块应暴露其他模块可以消费的输出：

```hcl
# 网络模块暴露 ID 供组合使用
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID，供其他模块使用"
}

output "public_subnet_ids" {
  value       = [for s in aws_subnet.public : s.id]
  description = "公有子网 ID 列表"
}

output "private_subnet_ids" {
  value       = [for s in aws_subnet.private : s.id]
  description = "私有子网 ID 列表"
}
```

### 通过输出组合模块

```hcl
# 根模块通过输出将模块连接起来
module "networking" {
  source = "./modules/networking"

  vpc_cidr            = "10.0.0.0/16"
  availability_zones = var.availability_zones
}

module "eks" {
  source = "./modules/eks"

  cluster_name = "${var.project}-cluster"
  vpc_id       = module.networking.vpc_id       # 输出 -> 输入
  subnet_ids   = module.networking.private_subnet_ids
}

module "database" {
  source = "./modules/rds"

  identifier = "${var.project}-db"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids

  # 跨模块引用
  allowed_security_groups = [module.eks.node_security_group_id]
}
```

## 组合模式

### 通过 Locals 共享数据

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
  }
}

module "vpc" {
  source = "./modules/vpc"
  tags   = local.common_tags
}

module "eks" {
  source = "./modules/eks"
  tags   = local.common_tags
}
```

### 使用 count 实现可选组件

```hcl
module "monitoring" {
  source = "./modules/monitoring"
  count  = var.enable_monitoring ? 1 : 0

  vpc_id = module.vpc.vpc_id
}
```

## 模块接口设计

保持接口最小化 - 只暴露需要的，隐藏实现细节：

```hcl
# 好的做法 - 聚焦的接口，合理的默认值
module "s3_bucket" {
  source      = "./modules/s3"
  name        = "my-bucket"
  versioning  = true
}

# 避免 - 泄漏抽象，暴露所有选项
module "s3_bucket" {
  source             = "./modules/s3"
  name               = "my-bucket"
  versioning         = true
  lifecycle_rules    = [...]
  cors_rules         = [...]
  replication_config = {...}
  # 暴露每个 S3 选项就失去了抽象的意义
}
```

## 参考资料

- [模块组合](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
- [模块结构](https://developer.hashicorp.com/terraform/language/modules/develop/structure)
