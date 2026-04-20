# 分层类型设计模式

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

Terraform 的 `map(any)` 要求所有 map 元素具有相同的字段结构。这与实际场景冲突，比如一种资源类型有多个子类型，各子类型有不同的参数集（如 Kafka Serverless/传统版/Confluent 版）。

本规则定义了**分层类型设计模式**，在声明层、控制层和原子层之间平衡灵活性与类型安全。

## 问题

```hcl
# Problem: map(any) requires identical structure
variable "kafka_instances" {
  type = map(any) # Error when elements have different fields!
}

# terraform.tfvars - three different sub-types
kafka_instances = {
  "01" = { serverless_config = { ... } } # Serverless type
  "02" = { disk_type = 1, disk_size = 500 } # Traditional type
  "03" = { confluent_config = { ... } } # Confluent type
}
# Error: all map elements must have the same type
```

## 解决方案：分层类型设计模式

### 架构概览

```
┌─────────────────┐ ┌─────────────────────────────┐ ┌─────────────────┐
│ Declaration │ │ Control Layer │ │ Atomic Layer │
│ Layer │ │ │ │ │
├─────────────────┤ ├─────────────────────────────┤ ├─────────────────┤
│ type = any │ ──► │ type = map(object({...})) │ ──► │ type = specific │
│ (most flexible) │ │ + optional() for each field │ │ (strict) │
└─────────────────┘ └─────────────────────────────┘ └─────────────────┘
│ │ │
Flexibility Type Safety Validation
(any sub-type) (optional constraints) (final check)
```

### 第 1 层：声明层（最灵活）

**使用 `any` 类型**，允许不同子类型在同一 map 中共存。

```hcl
# declarative/${ENV}/04-database/variables.tf（声明层）

variable "kafka_instances" {
  description = "Kafka 实例配置 map。key=逻辑名"
  type = any # 允许不同子类型共存
  default = {}
}
```

**优点：**
- 不同子类型可以省略不相关字段
- 干净、可读的 tfvars，无空字段
- 不会出现"所有元素必须具有相同类型"错误

### 第 2 层：控制层（类型约束 + 灵活性）

**使用 `map(object({...}))` 配合 `optional()`**，提供类型安全同时允许省略字段。

```hcl
# control/database-cluster/variables.tf（控制层）

variable "kafka_instances" {
  description = "Kafka 实例配置 map。key=逻辑名（如 01/02/03）"
  type = map(object({
    # Common fields
    create = optional(bool)
    vswitch_key = optional(string)
    zone_id = optional(string)
    instance_name = optional(string)
    instance_type = optional(string)
    deploy_type = optional(number)
    paid_type = optional(string)
    service_version = optional(string)

    # Traditional type fields
    spec_type = optional(string)
    partition_num = optional(number)
    io_max = optional(number)
    disk_type = optional(number)
    disk_size = optional(number)

    # Serverless type fields
    serverless_config = optional(object({
      reserved_publish_capacity = number
      reserved_subscribe_capacity = number
    }))

    # Confluent type fields
    confluent_config = optional(object({
      kafka_cu = optional(number)
      kafka_storage = optional(number)
      kafka_replica = optional(number)
      zookeeper_cu = optional(number)
      zookeeper_storage = optional(number)
      zookeeper_replica = optional(number)
    }))
    password = optional(string) # Confluent only

    # Collection types - MUST have defaults
    vswitch_ids = optional(list(string), [])
    selected_zones = optional(list(string), [])
    tags = optional(map(string), {})
  }))
  default = {}
}
```

**优点：**
- 类型安全：IDE 支持，早期错误检测
- 灵活性：`optional()` 允许省略字段
- 清晰契约：明确的字段定义即文档

### 第 3 层：原子层（最终验证）

**使用具体类型配合验证规则**，进行最终参数验证。

```hcl
# atomic/kafka/variables.tf（原子层）

variable "instance_type" {
  description = "Kafka 实例类型"
  type = string
  default = ""

  validation {
    condition = var.instance_type == "" || var.instance_type == null ||
    contains(["alikafka", "alikafka_serverless", "alikafka_confluent"], var.instance_type)
    error_message = "instance_type 取值：空字符串/alikafka/alikafka_serverless/alikafka_confluent。"
  }
}

variable "disk_type" {
  description = "Kafka 磁盘类型（传统版专用）"
  type = number
  default = 1

  validation {
    condition = var.disk_type == null || contains([0, 1], var.disk_type)
    error_message = "disk_type 取值：null/0/1。"
  }
}
```

## 完整示例：Kafka 多类型配置

### 声明层（terraform.tfvars）

```hcl
# Clean and readable - only relevant fields per sub-type
kafka_instances = {
  # Serverless type - elastic, pay-as-you-go
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    deploy_type = 5
    paid_type = "PostPaid"

    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
    instance_name = "kafka-serverless-01"
    # No disk_type, disk_size, confluent_config - irrelevant for Serverless
  }

  # Traditional type - fixed spec, high throughput
  "02" = {
    vswitch_key = "j-data"
    create = false
    instance_type = "alikafka"
    deploy_type = 5
    spec_type = "professional"
    partition_num = 50
    io_max = 20
    disk_type = 1
    disk_size = 500
    paid_type = "PostPaid"
    service_version = "2.6.2"
    instance_name = "kafka-traditional-02"
    # No serverless_config, confluent_config - irrelevant for Traditional
  }

  # Confluent type - enterprise features
  "03" = {
    vswitch_key = "j-data"
    create = false
    instance_type = "alikafka_confluent"
    deploy_type = 5
    spec_type = "professional"

    confluent_config = {
      kafka_cu = 2
      kafka_storage = 500
      kafka_replica = 3
      zookeeper_cu = 2
      zookeeper_storage = 20
      zookeeper_replica = 3
    }
    password = "KafkaTest123!"
    paid_type = "PostPaid"
    instance_name = "kafka-confluent-03"
    # No serverless_config, disk_type - irrelevant for Confluent
  }
}
```

## 类型设计决策矩阵

| Scenario | Declaration Layer | Control Layer | Atomic Layer |
|----------|------------------|---------------|--------------|
| **Multi sub-types** (Kafka, etc.) | `any` | `map(object)` + `optional()` | specific + validation |
| **Single type, all required** | `map(object)` | `map(object)` | specific |
| **Simple passthrough** | `map(any)` | `map(any)` | specific |

## 核心规则总结

### 规则 1：声明层灵活性
```hcl
# Use `any` when map elements have different structures
type = any

# Don't use map(any) - it requires identical structure
type = map(any) # Error when sub-types differ!
```

### 规则 2：控制层类型安全
```hcl
# Use map(object) with optional() for each field
type = map(object({
  field1 = optional(string)
  field2 = optional(number)
  list_field = optional(list(string), []) # Collection MUST have default!
}))

# Don't use map(any) - loses type safety and IDE support
type = map(any)
```

### 规则 3：集合类型必须有默认值
```hcl
# Collection types in optional() MUST specify defaults
vswitch_ids = optional(list(string), []) #
selected_zones = optional(list(string), []) #
tags = optional(map(string), {}) #

# Without defaults, null causes length() to fail
vswitch_ids = optional(list(string)) # null when not passed
# In atomic layer: length(var.vswitch_ids) > 0 → ERROR!
```

### 规则 4：原子层 Null 安全验证
```hcl
# Always check null first in validation
validation {
  condition = var.disk_type == null || contains([0, 1], var.disk_type)
  error_message = "..."
}

# contains() fails with null
validation {
  condition = contains([0, 1], var.disk_type) # ERROR when null!
  error_message = "..."
}
```

## 收益总结

| Aspect | Before (map(any) everywhere) | After (Layered Type Design) |
|--------|------------------------------|----------------------------|
| **Flexibility** | Requires identical structure | Different sub-types allowed |
| **Type Safety** | No IDE support, late errors | Early detection, IDE hints |
| **Readability** | Verbose with empty fields | Clean, only relevant fields |
| **Documentation** | No explicit contract | Control layer = contract |
| **Validation** | Scattered, inconsistent | Layered, systematic |

### 规则 5：控制层类型定义必须覆盖所有 `each.value.*` 引用（关键）

> **这是最常见的隐性错误：控制层 `main.tf` 中通过 `each.value.xxx` 引用的字段，必须在 `variables.tf` 的 `map(object({...}))` 类型定义中声明。否则 Terraform 会静默丢弃未声明的字段，导致配置不生效且无任何错误提示。**

```hcl
# 致命错误：main.tf 引用了 security_group_key，但类型定义中没有

# control/web-cluster/variables.tf
variable "nlbs" {
  type = map(object({
    address_type = string
    cross_zone_enabled = optional(bool, true)
    create = optional(bool, true)
    # 缺少 security_group_key！
  }))
}

# control/web-cluster/main.tf
module "nlb" {
  for_each = var.nlbs
  # each.value.security_group_key 永远为 null（被静默丢弃）
  security_group_ids = try(each.value.security_group_key, null) != null ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] : []
}
```

```hcl
# 正确：类型定义覆盖所有 each.value 引用

# control/web-cluster/variables.tf
variable "nlbs" {
  type = map(object({
    address_type = string
    cross_zone_enabled = optional(bool, true)
    create = optional(bool, true)
    security_group_key = optional(string) # 与 main.tf 引用对齐
  }))
}
```

**检测方法**：在控制层 `main.tf` 中搜索所有 `each.value.` 引用，提取字段名列表，然后逐一对照 `variables.tf` 中对应 `map(object({...}))` 的类型定义，确保每个引用字段都已声明。

```bash
# 检测命令：提取 main.tf 中所有 each.value.XXX 字段
grep -oP 'each\.value\.\w+' control/web-cluster/main.tf | sort -u
# 输出示例：
# each.value.address_type
# each.value.create
# each.value.cross_zone_enabled
# each.value.security_group_key

# 然后对照 variables.tf 中 map(object) 的字段列表，确认每个都有声明
```

## 迁移检查清单

将模块转换为此模式时：

- [ ] **声明层**：将 `map(any)` 改为 `any`
- [ ] **控制层**：定义 `map(object({...}))`，包含所有可能的字段
- [ ] **控制层**：为每个字段添加 `optional()`
- [ ] **控制层**：为集合类型添加默认值 `[]`、`{}`
- [ ] **控制层**：**`each.value.*` 引用与类型定义字段对齐检查（关键！）**
- [ ] **原子层**：添加 null 安全验证规则
- [ ] **terraform.tfvars**：移除空字段/不需要的字段，只保留相关字段

## 参考资料

- [Terraform 类型约束](https://developer.hashicorp.com/terraform/language/expressions/type-constraints)
- [Terraform 可选属性](https://developer.hashicorp.com/terraform/language/expressions/type-constraints#optional-attributes)
