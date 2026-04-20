# 每个模块负责一个逻辑组件

**优先级：** 高
**分类：** 模块设计

## 为什么重要

职责过多的模块难以理解、测试和复用。每个模块应有一个单一的、明确的职责。

## 错误示例

```hcl
# modules/everything/main.tf
# 这个模块做了太多事情

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  count  = 3
  vpc_id = aws_vpc.main.id
  # ...
}

resource "aws_eks_cluster" "cluster" {
  name = var.cluster_name
  # ...
}

resource "aws_rds_cluster" "database" {
  cluster_identifier = var.db_name
  # ...
}

resource "aws_elasticache_cluster" "cache" {
  cluster_id = var.cache_name
  # ...
}

resource "aws_lambda_function" "api" {
  function_name = var.lambda_name
  # ...
}
```

**问题：** 这个"万能模块"同时处理网络、计算、数据库、缓存和 Serverless。任何组件的变更都会影响整个模块。

## 正确示例

```hcl
# 网络模块 - 只做一件事：创建 VPC 和子网
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = var.tags
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

```hcl
# EKS 模块 - 只做一件事：创建 Kubernetes 集群
resource "aws_eks_cluster" "cluster" {
  name     = var.cluster_name
  role_arn = var.cluster_role_arn

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}
```

```hcl
# 根模块 - 组合各专注模块
module "networking" {
  source = "./modules/networking"

  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
}

module "eks" {
  source = "./modules/eks"

  cluster_name    = "prod-cluster"
  subnet_ids      = module.networking.public_subnet_ids
  cluster_role_arn = module.iam.eks_cluster_role_arn
}

module "database" {
  source = "./modules/rds"

  db_name   = "prod-db"
  vpc_id    = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}
```

## 设计原则

1. **每个模块一个逻辑组件** - VPC、EKS 集群、RDS 数据库各一个模块
2. **清晰的输入和输出** - 模块接口应该一目了然
3. **可组合** - 模块通过输出/输入协同工作
4. **可测试** - 每个模块可以独立测试
5. **可复用** - 同一模块可在不同环境中使用

## 模块规模参考

- **过小：** 单个资源且无逻辑
- **合适：** 5-20 个资源，职责明确
- **过大：** 50+ 个资源或包含多个不相关组件

## 参考资料

- [模块组合](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
