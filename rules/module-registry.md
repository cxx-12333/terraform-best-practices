# 使用现有的社区/共享模块

**优先级：** 高
**分类：** 模块设计

## 为什么重要

不要重复造轮子。社区和共享模块经过实战检验、持续维护，能节省大量开发时间。使用现有模块处理常见模式，将精力集中在业务特定的基础设施上。

## 错误示例

```hcl
# 已有维护良好的模块时仍从零编写 VPC
resource "aws_vpc" "main" {
  cidr_block         = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  # ... 还有 200 行网络代码
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```

**问题：**
- 在已解决的问题上浪费时间
- 遗漏社区已处理的边界情况
- 团队维护负担
- 潜在的安全漏洞

## 正确示例

### 使用 Terraform Registry 模块

```hcl
# 内置最佳实践的 VPC 模块
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "${var.project}-${var.environment}"
  cidr = var.vpc_cidr

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = var.environment != "prod"

  tags = local.common_tags
}
```

### 热门社区模块

| 用途 | 模块 |
|------|------|
| AWS VPC | `terraform-aws-modules/vpc/aws` |
| AWS EKS | `terraform-aws-modules/eks/aws` |
| AWS RDS | `terraform-aws-modules/rds/aws` |
| AWS Lambda | `terraform-aws-modules/lambda/aws` |
| AWS S3 | `terraform-aws-modules/s3-bucket/aws` |
| GCP 网络 | `terraform-google-modules/network/google` |
| GCP GKE | `terraform-google-modules/kubernetes-engine/google` |
| Azure VNet | `Azure/vnet/azurerm` |
| Azure AKS | `Azure/aks/azurerm` |

### 使用前评估

```bash
# 检查模块质量
# 1. 注册表上的星标/下载数
# 2. 最近更新（是否积极维护？）
# 3. 未解决的问题数量
# 4. 文档质量
# 5. 测试覆盖率
```

### 何时编写自定义模块

在以下情况编写自定义模块：
- 没有现有模块满足需求
- 安全要求禁止使用外部依赖
- 需要对实现进行严格控制
- 社区模块已停止维护

### 私有模块注册表

```hcl
# Terraform Cloud/Enterprise 私有注册表
module "internal_vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}

# Git 源私有模块
module "internal_vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//vpc?ref=v1.0.0"
}
```

### 包装社区模块

在社区模块之上添加组织默认值：

```hcl
# 薄包装模块，强制执行公司标准
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = var.name
  cidr = var.cidr

  # 公司标准：始终启用 DNS
  enable_dns_hostnames = true
  enable_dns_support   = true

  # 公司标准：必须开启流日志
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group  = true
  create_flow_log_cloudwatch_iam_role   = true

  # 透传其他变量
  azs             = var.azs
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  tags = merge(var.tags, {
    ManagedBy = "terraform"
  })
}
```

## 多云场景适配

### 多云 Registry 模块对比

| 云厂商 | 模块命名空间 | 推荐模块 | 质量 |
|--------|-------------|---------|------|
| AWS | `terraform-aws-modules` | VPC, RDS, EKS, S3 | 高（官方维护） |
| Azure | `Azure` | AKS, SQL, Key Vault | 高（微软维护） |
| GCP | `terraform-google-modules` | GKE, Cloud SQL, VPC | 高（Google 维护） |
| 阿里云 | `aliyun` | VPC, ECS, RDS, OSS | 中（阿里维护） |
| 腾讯云 | `tencentcloudstack` | VPC, CVM, MySQL | 中（腾讯维护） |

### 腾讯云 Registry 使用

```hcl
# 腾讯云 VPC 模块
module "vpc" {
  source  = "tencentcloudstack/vpc/tencentcloud"
  version = "~> 1.0"

  vpc_name    = "${var.project}-${var.environment}"
  cidr_block  = "10.0.0.0/16"
}
```

### Registry 模块选择标准（多云通用）

1. **下载量** > 10,000（说明社区验证充分）
2. **最近更新** < 6个月（活跃维护）
3. **兼容 Terraform >= 1.0**
4. **有完整 examples/ 目录**
5. **有 CI/CD 自动化测试**

## 参考资料

- [Terraform Registry](https://registry.terraform.io/)
- [AWS 模块](https://registry.terraform.io/namespaces/terraform-aws-modules)
- [Google 模块](https://registry.terraform.io/namespaces/terraform-google-modules)
- [Azure 模块](https://registry.terraform.io/namespaces/Azure)
- [阿里云模块](https://registry.terraform.io/namespaces/aliyun)
- [腾讯云模块](https://registry.terraform.io/namespaces/tencentcloudstack)
