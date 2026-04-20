# 每个环境每个配置使用独立工作区

**优先级：** 高
**分类：** 组织与工作流

## 为什么重要

将基础设施组织为独立工作区可以实现权限委派、控制爆炸半径并支持环境晋升。每个工作区应代表一个环境中的一个基础设施组件。

## 公式

```
Terraform 配置数 × 环境数 = 工作区数
```

## 错误示例

```hcl
# 一个巨型工作区管理所有内容
# terraform-monolith/
# - VPC
# - EKS 集群
# - RDS 数据库
# - ElastiCache
# - Lambda 函数
# - S3 存储桶
# - IAM 角色
# 所有环境共用一个状态文件
```

**问题：**
- 单点故障
- plan/apply 执行时间长
- 无法委派所有权
- 意外销毁的风险
- 状态文件冲突

## 正确示例

### 按组件和环境拆分

```bash
# 工作区命名：{组件}-{环境}
networking-dev
networking-staging
networking-prod

eks-cluster-dev
eks-cluster-staging
eks-cluster-prod

database-dev
database-staging
database-prod

app-frontend-dev
app-frontend-staging
app-frontend-prod
```

### 相同代码，不同变量

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "环境名称（dev、staging、prod）"
}

variable "instance_count" {
  type        = number
  description = "实例数量"
}

# main.tf - 所有环境使用相同代码
resource "aws_instance" "app" {
  count         = var.instance_count
  instance_type = var.environment == "prod" ? "m5.large" : "t3.micro"

  tags = {
    Environment = var.environment
  }
}
```

```hcl
# dev.tfvars
environment    = "dev"
instance_count = 1

# prod.tfvars
environment    = "prod"
instance_count = 3
```

### 通过远程状态实现工作区依赖

```hcl
# 在 eks-cluster 配置中
# 读取 networking 工作区的输出
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "terraform-state"
    key    = "networking-${var.environment}/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  name = "${var.project}-${var.environment}"

  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

### 每个工作区使用独立状态后端

```hcl
# 每个工作区使用不同的状态 key
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "networking-dev/terraform.tfstate" # 每个工作区不同
    region = "us-east-1"
  }
}
```

或使用工作区功能：

```bash
# Terraform 原生工作区
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select dev
terraform apply -var-file=dev.tfvars
```

## 工作区规模指南

| 规模 | 资源数量 | 适用场景 |
|------|----------|----------|
| 过小 | 1-3 个 | 管理开销不值得 |
| 合适 | 10-50 个 | 可管理，权责清晰 |
| 过大 | 200+ 个 | 应拆分为组件 |

## 优点

1. **爆炸半径可控** - 错误只影响一个组件/环境
2. **权限委派** - 不同团队负责不同工作区
3. **并行操作** - 团队独立工作互不干扰
4. **环境晋升** - 变更流程 dev → staging → prod
5. **审计追踪** - 每个组件有清晰的变更历史

## 参考资料

- [工作区结构](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part1#the-recommended-terraform-workspace-structure)
- [远程状态](https://developer.hashicorp.com/terraform/language/state/remote-state-data)
