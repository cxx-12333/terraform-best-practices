# 正确使用生命周期管理

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

生命周期块控制 Terraform 如何创建、更新和销毁资源。正确使用可以防止意外销毁、实现零停机部署，并处理资源管理的边缘情况。

## 错误示例

```hcl
# 无生命周期规则 - 对关键资源很危险
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"
  # 可能被 terraform destroy 意外删除！
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  # 默认行为：先销毁再创建 = 停机
}
```

## 正确示例

```hcl
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true # 防止意外删除
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true # 零停机替换
  }
}
```

## 生命周期参数

| 参数 | 用途 |
|------|------|
| `create_before_destroy` | 在销毁原始资源前先创建替换 |
| `prevent_destroy` | 防止意外删除 |
| `ignore_changes` | 忽略特定属性的变更 |
| `replace_triggered_by` | 依赖变更时强制替换 |

## create_before_destroy

### 问题

```hcl
# 默认行为：先销毁再创建
# 替换资源时导致停机

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
}

# AMI 变更时：
# 1. 销毁旧实例（停机开始）
# 2. 创建新实例（停机结束）
```

### 解决方案

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}

# AMI 变更时：
# 1. 创建新实例
# 2. 更新引用（负载均衡器等）
# 3. 销毁旧实例
# 零停机！
```

### 常用场景

```hcl
# 安全组（被实例引用）
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"

  lifecycle {
    create_before_destroy = true
  }
}

# IAM 角色（被服务引用）
resource "aws_iam_role" "app" {
  name_prefix = "app-role-"

  lifecycle {
    create_before_destroy = true
  }
}

# 启动模板（被 ASG 引用）
resource "aws_launch_template" "web" {
  name_prefix = "web-lt-"

  lifecycle {
    create_before_destroy = true
  }
}
```

## prevent_destroy

### 保护关键资源

```hcl
# 防止意外删除生产数据库
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true
  }
}

# 防止删除状态存储桶
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

# 防止删除加密密钥
resource "aws_kms_key" "main" {
  description = "主加密密钥"

  lifecycle {
    prevent_destroy = true
  }
}
```

### terraform destroy 行为

```bash
# 设置了 prevent_destroy = true 时
terraform destroy

# Error: Instance cannot be destroyed
# Resource aws_db_instance.production has lifecycle.prevent_destroy set

# 要实际销毁，先从配置中移除 prevent_destroy
```

## ignore_changes

### 忽略外部修改

```hcl
# 忽略由外部自动化管理的标签
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    ignore_changes = [
      tags["LastBackup"],     # 由备份系统更新
      tags["CostAllocation"], # 由成本工具更新
    ]
  }
}

# 忽略 ASG 期望容量（由自动伸缩管理）
resource "aws_autoscaling_group" "web" {
  name             = "web-asg"
  min_size         = 2
  max_size         = 10
  desired_capacity = 2 # 初始值

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

### 忽略所有变更

```hcl
# 资源由外部管理，仅在状态中跟踪
resource "aws_instance" "legacy" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = all
  }
}
```

### 常用 ignore_changes 模式

```hcl
# EKS 集群版本（通过控制台/CLI 升级）
resource "aws_eks_cluster" "main" {
  name    = "main"
  version = "1.27"

  lifecycle {
    ignore_changes = [version]
  }
}

# Lambda 函数代码（单独部署）
resource "aws_lambda_function" "app" {
  function_name = "app"
  filename      = "placeholder.zip"

  lifecycle {
    ignore_changes = [
      filename,
      source_code_hash,
    ]
  }
}

# RDS 密码（在 Terraform 外管理）
resource "aws_db_instance" "main" {
  identifier = "main"
  password   = "initial-password"

  lifecycle {
    ignore_changes = [password]
  }
}
```

## replace_triggered_by

### 依赖变更时强制替换

```hcl
# 用户数据脚本变更时替换实例
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  user_data     = file("${path.module}/user-data.sh")

  lifecycle {
    replace_triggered_by = [
      null_resource.user_data_version
    ]
  }
}

resource "null_resource" "user_data_version" {
  triggers = {
    user_data_hash = filemd5("${path.module}/user-data.sh")
  }
}
```

### 模块输出变更时替换

```hcl
resource "aws_instance" "web" {
  ami           = module.ami.latest_id
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      module.ami # 当任何模块输出变更时替换
    ]
  }
}
```

## 组合生命周期规则

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "prod"

    ignore_changes = [
      tags["LastBackup"],
    ]

    replace_triggered_by = [
      null_resource.deployment_trigger
    ]
  }
}
```

## 前置条件和后置条件

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.instance_type != "t3.micro"
      error_message = "生产环境实例必须大于 t3.micro"
    }

    postcondition {
      condition     = self.public_ip != null
      error_message = "实例必须有公网 IP"
    }
  }
}
```

## TypeSet 嵌套块的 ignore_changes 限制

### 问题：TypeSet 不支持索引访问

当嵌套块在 Provider schema 中定义为 `TypeSet` 时，`ignore_changes` **不能使用索引语法**：

```hcl
# 错误：TypeSet 不支持索引
lifecycle {
  ignore_changes = [
    servers[0].server_ip, # Cannot index a set value
  ]
}

# 正确：只能忽略整个 TypeSet 块
lifecycle {
  ignore_changes = [
    servers, # 忽略整个 servers 块
  ]
}
```

### 根因：TypeSet hash 包含所有 schema 字段

TypeSet 的 hash 计算包含 schema 中**所有字段**（含 Computed-only 字段），因此：
- 移除 dynamic content 中的字段赋值**不能**消除漂移
- Computed-only 字段（无 Optional）在 HCL 中不可赋值，零值永远不等于 API 返回值
- 唯一方案是 `ignore_changes = [block_name]`

> **详见**: `resource-typeset-computed-drift.md` — TypeSet 嵌套块 Computed-only 字段导致无限状态漂移的完整排查路径

## 参考资料

- [生命周期元参数](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- [前置条件和后置条件](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions)
- `resource-typeset-computed-drift.md` - TypeSet 嵌套块 Computed 字段漂移专题
