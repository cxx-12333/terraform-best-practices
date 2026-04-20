# 添加变量验证规则

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

验证规则在 Terraform 尝试创建资源之前及早捕获配置错误。这可以防止失败的部署并提供清晰的错误信息。

## 错误示例

```hcl
variable "environment" {
  type = string
  # 无验证 - 接受任何字符串
}

variable "instance_type" {
  type = string
}

variable "cidr_block" {
  type = string
}

# 错误只在 apply 时被 AWS 拒绝后才发现
```

## 正确示例

```hcl
variable "environment" {
  type        = string
  description = "部署环境"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "环境必须是 dev、staging 或 prod。"
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 实例类型"

  validation {
    condition     = can(regex("^[a-z][0-9]+\\.(nano|micro|small|medium|large|xlarge|[0-9]+xlarge)$", var.instance_type))
    error_message = "无效的实例类型格式。示例：t3.micro、m5.large"
  }
}

variable "cidr_block" {
  type        = string
  description = "VPC CIDR 块"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "必须是有效的 CIDR 块（如 10.0.0.0/16）。"
  }

  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) >= 16 && tonumber(split("/", var.cidr_block)[1]) <= 28
    error_message = "CIDR 块必须在 /16 到 /28 之间。"
  }
}
```

## 常用验证模式

### 字符串长度

```hcl
variable "bucket_name" {
  type = string

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "存储桶名称必须在 3 到 63 个字符之间。"
  }
}
```

### 数值范围

```hcl
variable "port" {
  type = number

  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "端口必须在 1 到 65535 之间。"
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "实例数量必须在 1 到 10 之间。"
  }
}
```

### 列表验证

```hcl
variable "availability_zones" {
  type = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "高可用至少需要 2 个可用区。"
  }
}
```

### Map 验证

```hcl
variable "tags" {
  type = map(string)

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "标签必须包含 'Environment' 键。"
  }
}
```

### 多重验证

```hcl
variable "db_name" {
  type = string

  validation {
    condition     = length(var.db_name) >= 1 && length(var.db_name) <= 63
    error_message = "数据库名称必须在 1 到 63 个字符之间。"
  }

  validation {
    condition     = can(regex("^[a-zA-Z][a-zA-Z0-9_]*$", var.db_name))
    error_message = "数据库名称必须以字母开头，只包含字母、数字和下划线。"
  }
}
```

## 多云场景适配

### 阿里云实例规格验证

```hcl
variable "instance_type" {
  type        = string
  description = "阿里云 ECS 实例规格"

  validation {
    condition     = can(regex("^ecs\\.[a-z]\\d+\\.", var.instance_type))
    error_message = "实例规格格式错误，示例: ecs.g7.xlarge"
  }
}

variable "engine" {
  type        = string
  description = "数据库引擎"

  validation {
    condition     = contains(["mysql", "postgresql", "mariadb"], var.engine)
    error_message = "数据库引擎必须是 mysql、postgresql 或 mariadb。"
  }
}
```

### Azure SKU 验证

```hcl
variable "vm_size" {
  type        = string
  description = "Azure 虚拟机规格"

  validation {
    condition     = can(regex("^Standard_", var.vm_size))
    error_message = "VM 规格必须以 Standard_ 开头，示例: Standard_D2s_v3"
  }
}

variable "storage_account_type" {
  type        = string
  description = "存储类型"

  validation {
    condition     = contains(["Premium_LRS", "Standard_LRS", "StandardSSD_LRS"], var.storage_account_type)
    error_message = "存储类型必须是 Premium_LRS、Standard_LRS 或 StandardSSD_LRS。"
  }
}
```

### 腾讯云实例验证

```hcl
variable "instance_type" {
  type        = string
  description = "腾讯云 CVM 实例规格"

  validation {
    condition     = can(regex("^[A-Z]\\w+\\.", var.instance_type))
    error_message = "实例规格格式错误，示例: SA2.MEDIUM4"
  }
}
```

## 参考资料

- [输入变量验证](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)
