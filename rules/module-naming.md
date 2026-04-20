# 使用一致的命名规范（terraform-<PROVIDER>-<NAME>）

**优先级：** 高
**分类：** 模块设计

## 为什么重要

一致的命名规范使模块易于发现、表明其用途，并遵循社区标准。命名良好的模块更容易查找、理解和复用。

## 错误示例

```hcl
# 命名模式不一致
module "vpc" {
  source = "./modules/vpc-module"
}

module "s3" {
  source = "./modules/storage"
}

module "db" {
  source = "./modules/database-module"
}

# 更糟的 - 没有清晰的命名模式
module "infra1" {
  source = "./modules/module1"
}
```

**问题：** 不一致的命名使模块难以发现和理解，无法表明 Provider 或用途。

## 正确示例

### 标准命名规范

可复用模块（尤其是公共/注册表模块）：

```
terraform-<PROVIDER>-<NAME>
```

**示例：**
- `terraform-aws-vpc` - AWS VPC 模块
- `terraform-aws-eks` - AWS EKS 集群模块
- `terraform-aws-rds` - AWS RDS 数据库模块
- `terraform-google-network` - GCP 网络模块
- `terraform-azurerm-aks` - Azure AKS 模块

### 目录结构

```
modules/
├── terraform-aws-vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
├── terraform-aws-eks/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
```

### 模块使用

```hcl
# 注册表模块
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# 遵循规范的本地模块
module "vpc" {
  source = "./modules/terraform-aws-vpc"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# Git 模块
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

### 内部/私有模块

内部模块可使用较短的名称，但应保持一致性：

```hcl
# 方案 1：保持完整规范
module "vpc" {
  source = "./modules/terraform-aws-vpc"
}

# 方案 2：较短的内部规范（需保持一致）
module "vpc" {
  source = "./modules/aws-vpc" # 仍标明 Provider
}

# 方案 3：组织特定前缀
module "vpc" {
  source = "./modules/acme-aws-vpc" # acme = 公司名
}
```

### 命名指南

1. **使用 kebab-case** - `terraform-aws-vpc`，而非 `terraform_aws_vpc` 或 `terraformAwsVpc`
2. **包含 Provider** - 明确使用哪个云厂商
3. **描述性强** - `terraform-aws-vpc` 优于 `terraform-aws-networking`
4. **避免缩写** - `terraform-aws-database` 而非 `terraform-aws-db`
5. **使用单数** - `terraform-aws-instance` 而非 `terraform-aws-instances`

## 多云场景适配

### 多云模块命名对照

| 云厂商 | Provider 前缀 | 模块命名示例 |
|--------|-------------|-------------|
| AWS | `aws` | `terraform-aws-vpc`, `terraform-aws-rds` |
| Azure | `azurerm` | `terraform-azurerm-vnet`, `terraform-azurerm-mysql` |
| GCP | `google` | `terraform-google-network`, `terraform-google-sql` |
| 阿里云 | `alicloud` | `terraform-alicloud-vpc`, `terraform-alicloud-rds` |
| 腾讯云 | `tencentcloud` | `terraform-tencentcloud-vpc`, `terraform-tencentcloud-mysql` |

### 多云内部模块命名规范

```
modules/
├── aws-vpc/
├── azurerm-vnet/
├── alicloud-vpc/
├── tencentcloud-vpc/
├── aws-rds/
├── azurerm-mysql/
├── alicloud-rds/
└── tencentcloud-mysql/
```

### 多云环境变量命名

```hcl
# 统一命名，按云厂商区分
variable "aws_region"    { type = string }
variable "azure_location" { type = string }
variable "ali_region"    { type = string }
variable "tc_region"     { type = string }
```

## 参考资料

- [Terraform 模块命名](https://developer.hashicorp.com/terraform/language/modules/develop/standard-structure)
- [Terraform Registry 命名规范](https://developer.hashicorp.com/terraform/registry/modules/publish)
