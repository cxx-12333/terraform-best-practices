# 启用状态锁定防止损坏

**优先级：** 关键
**分类：** 状态管理

## 为什么重要

没有状态锁定，并发的 Terraform 操作可能损坏状态文件或引发竞态条件。始终启用锁定机制，防止多个用户或 CI 任务同时修改状态。

## 错误示例

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    # 未配置 DynamoDB 表用于锁定
  }
}
```

**问题：** 两位工程师同时执行 `terraform apply` 可能导致状态损坏。

## 正确示例

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**GCS 方案：**

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
    # GCS 内置锁定机制，无需额外配置
  }
}
```

## 创建锁定表

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Purpose = "Terraform state locking"
  }
}
```

## 多云场景适配

### 多云锁定机制

```hcl
# AWS: S3 + DynamoDB 锁定（最常用）
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    dynamodb_table = "tf-lock"  # DynamoDB 提供分布式锁
    encrypt        = true
  }
}

# Azure: Blob Storage 租约锁定
terraform {
  backend "azurerm" {
    resource_group_name  = "tf-state"
    storage_account_name = "tfstate1234"
    container_name       = "tfstate"
    # Blob Lease 自动提供锁定
  }
}

# 腾讯云: COS 内置锁定
terraform {
  backend "cos" {
    bucket = "my-tfstate-1250000000"
    # COS 原生支持状态锁定
  }
}
```

### 锁定机制对比

| 云厂商 | 锁定实现 | 可靠性 | 注意事项 |
|--------|----------|--------|----------|
| AWS | DynamoDB | 高 | 需额外创建 DynamoDB 表 |
| Azure | Blob Lease | 高 | 自动管理，无需额外资源 |
| GCP | GCS 内置 | 高 | 原生支持 |
| 阿里云 | OSS/Tablestore | 中 | 需配置 Tablestore 或 OSS 锁 |
| 腾讯云 | COS 内置 | 高 | 原生支持 |

## 参考资料

- [状态锁定](https://developer.hashicorp.com/terraform/language/state/locking)
- [S3 后端锁定](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [COS 后端](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/backend/cos)
