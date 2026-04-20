# 锁定 Provider 版本

**优先级：** 中
**分类：** Provider 配置

## 为什么重要

未锁定版本的 Provider 在发布破坏性变更时可能导致意外行为。锁定版本以确保可复现性，同时允许受控升级。

## 错误示例

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      # 无版本约束 - 使用最新版本
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**问题：** 不同时间运行 `terraform init` 可能获取不同版本的 Provider，产生不同行为。

## 正确示例

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0" # 允许 5.x，不允许 6.0
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20.0, < 3.0.0"
    }
  }
}
```

## 版本约束策略

### 生产环境 - 保守策略

```hcl
# 锁定到精确版本
aws = {
  source  = "hashicorp/aws"
  version = "5.31.0"
}
```

### 开发环境 - 灵活策略

```hcl
# 允许小版本更新
aws = {
  source  = "hashicorp/aws"
  version = "~> 5.31" # 允许 5.31.x
}
```

### 平衡策略

```hcl
# 允许小版本更新，阻止大版本
aws = {
  source  = "hashicorp/aws"
  version = ">= 5.0.0, < 6.0.0"
}
```

## 锁定文件

Terraform 创建 `.terraform.lock.hcl` 记录精确版本：

```hcl
# .terraform.lock.hcl（自动生成）
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
  ]
}
```

**最佳实践：**
- 将 `.terraform.lock.hcl` 提交到版本控制
- 使用 `terraform init -upgrade` 更新
- 在 PR 中审查锁定文件变更

## 更新 Provider

```bash
# 在约束范围内更新所有 Provider
terraform init -upgrade

# 检查过时的 Provider
terraform version

# 更新特定 Provider
terraform providers lock -platform=linux_amd64 hashicorp/aws
```

## 多 Provider 配置

```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.west]
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

## 参考资料

- [Provider 要求](https://developer.hashicorp.com/terraform/language/providers/requirements)
- [依赖锁定文件](https://developer.hashicorp.com/terraform/language/files/dependency-lock)
