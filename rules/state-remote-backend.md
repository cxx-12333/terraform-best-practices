# 始终使用远程状态后端

**优先级：** 关键
**分类：** 状态管理

## 为什么重要

本地状态文件是单点故障源，且无法支持团队协作。远程后端提供持久化存储、状态锁定和团队共享访问。

## 错误示例

```hcl
# 无后端配置 - 状态存储在本地
terraform {
  required_version = ">= 1.0"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

**问题：** 状态存储在本地 `terraform.tfstate` 文件中。一旦丢失，Terraform 将失去对所有托管资源的跟踪。

## 正确示例

```hcl
terraform {
  required_version = ">= 1.0"

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

**优点：**
- 状态持久化存储在 S3 中
- 通过 DynamoDB 实现状态锁定，防止并发修改
- 静态数据加密保护敏感信息
- 团队成员可以访问共享状态

## 补充说明

常用的远程后端选项：
- **S3** - AWS 原生方案，配合 DynamoDB 实现锁定
- **GCS** - Google Cloud Storage，内置锁定机制
- **Azure Blob** - Azure 原生后端
- **Terraform Cloud** - 托管后端，提供额外功能
- **Terramate Cloud** - GitOps 原生状态管理

## 多云场景适配

### 多云后端配置

```hcl
# AWS: S3 + DynamoDB
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-lock"
    encrypt        = true
  }
}

# Azure: Blob Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "tf-state"
    storage_account_name = "tfstate1234"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# 阿里云: OSS
terraform {
  backend "oss" {
    bucket = "my-tfstate"
    prefix = "prod"
    key    = "terraform.tfstate"
    region = "cn-hangzhou"
  }
}

# 腾讯云: COS
terraform {
  backend "cos" {
    region = "ap-guangzhou"
    bucket = "my-tfstate-1250000000"
    prefix = "terraform/state"
  }
}
```

### 后端对比

| 云厂商 | 后端类型 | 存储服务 | 锁定机制 | 加密 |
|--------|----------|----------|----------|------|
| AWS | s3 | S3 + DynamoDB | DynamoDB | SSE-S3/KMS |
| Azure | azurerm | Blob Storage | Blob Lease | AES-256 |
| GCP | gcs | Cloud Storage | 内置 | Google-managed |
| 阿里云 | oss | OSS | Tablestore/OSS | AES-256/KMS |
| 腾讯云 | cos | COS | COS 内置 | AES-256/KMS |

## 参考资料

- [Terraform 后端配置](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)
- [S3 后端](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [Azure 后端](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)
- [OSS 后端](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/backend/oss)
- [COS 后端](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/backend/cos)
