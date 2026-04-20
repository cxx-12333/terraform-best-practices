# 所有模块引用都要指定版本

**优先级：** 高
**分类：** 模块设计

## 为什么重要

未锁定版本的模块在上游变更时可能导致部署失败。始终锁定模块版本以确保可复现性和可控升级。

## 错误示例

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # 无版本约束 - 使用最新版本

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

module "internal_module" {
  source = "git::https://github.com/org/modules.git//vpc"
  # 无 ref - 使用默认分支的 HEAD
}
```

**问题：** 模块行为可能在两次运行之间意外变化。

## 正确示例

```hcl
# 注册表模块 - 使用版本约束
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2" # 锁定到精确版本

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# 允许小版本更新（5.1.x）
module "vpc_flexible" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# Git 模块 - 使用 ref
module "internal_module" {
  source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.3"
}

# Git 使用 commit SHA（最可复现）
module "internal_module_sha" {
  source = "git::https://github.com/org/modules.git//vpc?ref=abc123def456"
}
```

## 版本约束运算符

| 运算符 | 示例 | 含义 |
|--------|------|------|
| `=` | `= 1.2.3` | 精确版本 |
| `!=` | `!= 1.2.3` | 排除版本 |
| `>`、`>=`、`<`、`<=` | `>= 1.2.0` | 比较运算 |
| `~>` | `~> 1.2.0` | 悲观约束（允许 1.2.x，不允许 1.3.0） |

## 推荐策略

```hcl
# 生产环境 - 锁定精确版本
module "prod_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# 开发环境 - 允许补丁更新
module "dev_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"
}
```

## 更新版本

```bash
# 检查更新
terraform init -upgrade

# 锁定文件跟踪精确版本
cat .terraform.lock.hcl
```

## 参考资料

- [模块源](https://developer.hashicorp.com/terraform/language/modules/sources)
- [版本约束](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)
