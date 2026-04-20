# 使用明确的类型约束

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

明确的类型约束能及早捕获错误、提供文档，并改善 IDE 支持。使用 `any` 或省略类型会失去这些优势，使模块更难正确使用。

## 错误示例

```hcl
# 无类型 - 接受任何值
variable "instance_count" {}

# 已知具体类型时使用 'any'
variable "tags" {
  type = any
}

# 过于宽松
variable "port" {
  type = any
}
```

**问题：** 类型错误只能在 apply 时发现，甚至可能无法发现。

## 正确示例

```hcl
# 明确的基本类型
variable "instance_count" {
  type        = number
  description = "要创建的实例数量"
}

variable "environment" {
  type        = string
  description = "部署环境"
}

variable "enable_monitoring" {
  type        = bool
  default     = true
  description = "启用 CloudWatch 监控"
}

# 明确的集合类型
variable "tags" {
  type        = map(string)
  default     = {}
  description = "资源标签"
}

variable "subnet_ids" {
  type        = list(string)
  description = "子网 ID 列表"
}

variable "availability_zones" {
  type        = set(string)
  description = "可用区集合"
}
```

## 使用 Object 的复杂类型

```hcl
# 包含必填字段的对象
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
  })
  description = "数据库配置"
}

# 包含可选字段和默认值的对象
variable "scaling_config" {
  type = object({
    min_size          = optional(number, 1)
    max_size          = optional(number, 10)
    desired_capacity  = optional(number, 2)
    cooldown          = optional(number, 300)
  })
  default     = {}
  description = "自动伸缩配置"
}
```

## 使用正向变量名

避免双重否定，使用正向名称：

```hcl
# 错误 - 设为 false 时产生双重否定
variable "disable_encryption" {
  type    = bool
  default = false # !disable = enable... 令人困惑
}

# 正确 - 清晰的意图
variable "encryption_enabled" {
  type        = bool
  default     = true
  description = "启用静态加密"
}

# 错误
variable "no_public_ip" {
  type = bool
}

# 正确
variable "associate_public_ip" {
  type        = bool
  default     = false
  description = "关联公网 IP 地址"
}
```

## 适当使用 `nullable = false`

```hcl
# 防止集合类型接收 null 值
variable "subnet_cidrs" {
  type     = list(string)
  default  = []
  nullable = false
  description = "子网 CIDR 块列表"
}

variable "tags" {
  type     = map(string)
  default  = {}
  nullable = false
  description = "资源标签"
}

# 确保字符串有值
variable "environment" {
  type     = string
  nullable = false
  description = "部署环境（必填）"
}
```

## 何时使用 `any`

将 `type = any` 保留给特殊情况：

```hcl
# 可接受：来自外部源的高度可变结构
variable "datadog_monitor_config" {
  type        = any
  description = "匹配 Datadog API Schema 的监控配置"
}

# 可接受：透传到动态资源
variable "container_definitions" {
  type        = any
  description = "ECS 容器定义 JSON"
}
```

## 多云场景适配

### 阿里云实例规格类型

```hcl
# 阿里云 ECS 实例配置
variable "ecs_instances" {
  type = map(object({
    instance_type = string    # 如 "ecs.g7.xlarge"
    image_id      = string    # 如 "ubuntu_22_04_x64"
    vswitch_key   = string    # 逻辑名映射
    security_group_key = optional(string)
    data_disks = optional(list(object({
      name     = string
      size     = number
      category = optional(string, "cloud_essd")
    })), [])
  }))
}
```

### Azure VM 规格类型

```hcl
# Azure 虚拟机配置
variable "virtual_machines" {
  type = map(object({
    vm_size    = string       # 如 "Standard_D2s_v3"
    os_disk    = object({
      caching              = string
      storage_account_type = string  # 如 "Premium_LRS"
    })
    subnet_key = string       # 逻辑名映射
    nsg_key    = optional(string)
  }))
}
```

### 腾讯云 CVM 实例类型

```hcl
# 腾讯云 CVM 实例配置
variable "cvm_instances" {
  type = map(object({
    instance_type = string    # 如 "SA2.MEDIUM4"
    image_id      = string    # 如 "img-xxx"
    subnet_key    = string
    security_group_key = optional(string)
  }))
}
```

## 参考资料

- [类型约束](https://developer.hashicorp.com/terraform/language/expressions/type-constraints)
- [变量类型](https://developer.hashicorp.com/terraform/language/values/variables#type-constraints)
