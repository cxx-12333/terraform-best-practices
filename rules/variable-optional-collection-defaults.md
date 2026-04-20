# optional() 集合类型必须指定默认值

**优先级：** 高
**分类：** 变量与输出模式（层级专用）

## 为什么重要

在控制层变量中使用 `optional()` 时，省略参数会产生 `null`。这导致原子层的 `length()` 函数报错：`Invalid value for "value" parameter: argument must not be null.`

这是三层 IaC 架构中控制层向原子层传递参数时的关键问题。

## 错误示例

```hcl
# 控制层 - 未传入时产生 null
variable "kafka_instances" {
  type = map(object({
    vswitch_ids = optional(list(string)) # 不传 → null
    selected_zones = optional(list(string)) # 不传 → null
    tags = optional(map(string)) # 不传 → null
  }))
}

# 原子层 - 接收 null 时崩溃
resource "alicloud_alikafka_instance" "this" {
  vswitch_ids = length(var.vswitch_ids) > 0 ? var.vswitch_ids : null # length(null) 报错！
  selected_zones = length(var.selected_zones) > 0 ? var.selected_zones : null
}
```

**错误输出：**
```
│ Error: Invalid function argument
│ on main.tf line 60: length(var.vswitch_ids) > 0
│ var.vswitch_ids is null
│ Invalid value for "value" parameter: argument must not be null.
```

## 正确示例（Redis 模式）

```hcl
# Control layer - specify default values for collections
variable "kafka_instances" {
  type = map(object({
    vswitch_ids = optional(list(string), []) # 不传 → 空列表 []
    selected_zones = optional(list(string), []) # 不传 → 空列表 []
    tags = optional(map(string), {}) # 不传 → 空 map {}
  }))
}

# Atomic layer - now safe to use length()
resource "alicloud_alikafka_instance" "this" {
  vswitch_ids = length(var.vswitch_ids) > 0 ? var.vswitch_ids : null # length([]) = 0
  selected_zones = length(var.selected_zones) > 0 ? var.selected_zones : null #
}
```

## 默认值规则

| Type | Correct optional() Syntax | Value when not passed |
|------|---------------------------|----------------------|
| `list(string)` | `optional(list(string), [])` | 空列表 `[]` |
| `list(number)` | `optional(list(number), [])` | 空列表 `[]` |
| `map(string)` | `optional(map(string), {})` | 空 map `{}` |
| `string` | `optional(string)` | `null` (原子层用 `var.xxx != ""` 检查) |
| `string` (lookup key) | `optional(string, "")` | 空字符串 `""` |
| `number` | `optional(number)` | `null` (原子层用 `var.xxx == null` 检查) |
| `bool` (普通参数) | `optional(bool)` | `null` (原子层用 `var.xxx == null` 检查) |
| `bool` (控制开关) | `optional(bool, true)` | `true` (控制开关默认创建) |

## Computed 参数在 object 内部的特殊处理

### 问题场景

当 object 类型内部包含 `Optional + Computed` 参数时，**不应设置默认值**：

```hcl
# 错误：object 内部设置默认值
variable "product_info" {
  type = object({
    msg_process_spec = string
    send_receive_ratio = optional(number, 1.0) # 默认值 1.0 可能无效！
  })
}

# 当声明层传入 { msg_process_spec = "rmq.s2.2xlarge" } 时
# Terraform 自动填充 send_receive_ratio = 1.0
# API 收到 1.0 后报错：InvalidSendReceiveRatio.Range
```

### 正确处理

```hcl
# 正确：Computed 参数不设默认值
variable "product_info" {
  type = object({
    msg_process_spec = string
    send_receive_ratio = optional(number) # 不设默认值，返回 null
  })
}

# 原子层 main.tf
product_info {
  send_receive_ratio = try(product_info.value.send_receive_ratio, null) # null 时 API 自动填充
}
```

### 判断依据

查阅 Provider 源码 Schema 定义：
```go
"send_receive_ratio": {
  Type: schema.TypeFloat,
  Optional: true,
  Computed: true, // Computed = API 会返回默认值
},
```

**有 `Computed: true` 的参数，在 `optional()` 中不应设置默认值。**

## String 类型用于 lookup key 的特殊规则

### 问题场景

当 string 类型字段作为 `lookup()` 的 key 参数时，`null` 会导致报错：

```hcl
# 控制层变量定义 - optional(string) 默认返回 null
variable "kafka_instances" {
  type = map(object({
    security_group_key = optional(string) # 未设置时 → null
  }))
}

# 控制层 main.tf - lookup 接收 null 报错
security_group = try(each.value.security_group_key, "") != ""
? lookup(var.security_group_ids_map, each.value.security_group_key, "") # key=null 报错！
: ""
```

**错误输出：**
```
│ Error: Invalid function argument
│ on main.tf line 376: lookup(var.security_group_ids_map, each.value.security_group_key, "")
│ each.value.security_group_key is null
│ Invalid value for "key" parameter: argument must not be null.
```

### 根因分析

| 变量类型定义 | 未设置字段时 | `try()` 结果 | `lookup(map, key, "")` |
|-------------|------------|-------------|------------------------|
| `map(any)` | 字段**不存在** | `try` 捕获错误 → `""` | 不会执行到 lookup |
| `map(object)` + `optional(string)` | 字段存在，值为 `null` | `try(null, "")` → `""` | `lookup(map, null, "")` 报错 |
| `map(object)` + `optional(string, "")` | 字段存在，值为 `""` | `try("", "")` → `""` | `lookup(map, "", "")` 正常 |

**关键差异**：
- `map(any)`：未定义字段会报 "attribute not found"，`try()` 能捕获
- `map(object)`：字段已定义，值为 `null`，`try()` 无法阻止 `null` 传给 `lookup()`

### 正确写法

```hcl
# 控制层变量定义 - lookup key 字段必须设置空字符串默认值
variable "kafka_instances" {
  type = map(object({
    security_group_key = optional(string, "") # 未设置时 → "" (非 null)
    vswitch_key = optional(string, "") # 用于 vswitch_ids_map lookup
    zone_key = optional(string, "") # 用于 az_map lookup
  }))
}

# 控制层 main.tf - 现在可以安全使用
security_group = try(each.value.security_group_key, "") != ""
? lookup(var.security_group_ids_map, each.value.security_group_key, "") # key="" 或有效值，都不会报错
: try(each.value.security_group, "")

vswitch_id = try(each.value.vswitch_key, "") != ""
? var.vswitch_ids_map[each.value.vswitch_key] # key="" 不会执行到这里
: var.vswitch_ids_map["j-data"] # 默认值
```

### 三层架构中的最佳实践

| 场景 | 变量定义 | 说明 |
|------|----------|------|
| 用于 `lookup()` / `map[key]` 的 key | `optional(string, "")` | 必须设置空字符串默认值 |
| 普通字符串参数 | `optional(string)` | 可以保持 `null` 默认值 |
| 用于 `length()` 的 list | `optional(list(string), [])` | 必须设置空列表默认值 |
| 用于 `length()` 的 map | `optional(map(string), {})` | 必须设置空 map 默认值 |

## 控制开关参数特殊规则

### 为什么 `create` 参数必须设置默认值？

```hcl
# 错误 - optional(bool) 未设置默认值
create = optional(bool) # → null

# 控制层传递
create = try(each.value.create, true) # try(null, true) 返回 null！不是 true！

# 原子层报错
count = local.create ? 1 : 0 # null ? 1 : 0 → Error: Null condition
```

**关键知识点**：`try(null, default)` 返回 `null`，因为 `null` 不是错误，是合法值！

### 正确写法

```hcl
# 控制层变量定义 - 设置默认值
variable "kafka_instances" {
  type = map(object({
    create = optional(bool, true) # 默认 true = 创建
  }))
}

# 嵌套对象中的 create 也需要默认值
sasl_users = optional(map(object({
  create = optional(bool, true) # 默认创建用户
  password = optional(string)
})), {})

# 控制层传递（保持 try 作为双重保险）
create = try(each.value.create, true) # try(true, true) = true
```

### 三层 create 参数规范

| 层级 | 规范写法 | 说明 |
|------|----------|------|
| **控制层 variables.tf** | `create = optional(bool, true)` | 默认值在 optional 中设置 |
| **控制层 main.tf** | `create = try(each.value.create, true)` | try 作为双重保险 |
| **原子层 variables.tf** | `default = true` | 原子层保持默认 true |
| **原子层 main.tf** | `local.create = var.create` | 直接使用，无需 coalesce |

## 原子层协调

原子层验证规则应使用 null 安全模式：

```hcl
# Number type - check null first
variable "disk_type" {
  type = number
  default = 1

  validation {
    condition = var.disk_type == null || contains([0, 1], var.disk_type)
    error_message = "disk_type 取值：null（不设置）/ 0（高效云盘）/ 1（SSD 云盘）。"
  }
}

# String type - check null and empty
variable "enable_auto_topic" {
  type = string
  default = "disable"

  validation {
    condition = var.enable_auto_topic == null || var.enable_auto_topic == "" || contains(["enable", "disable"], var.enable_auto_topic)
    error_message = "enable_auto_topic 取值：null/空字符串（不设置）/ enable（开启）/ disable（关闭）。"
  }
}
```

## 总结规则

**一句话总结**：集合类型（list/map）和 lookup key 字段（string）在 `optional()` 中必须指定默认值，避免原子层函数接收到 `null`。

## 实际案例

### Kafka 模块修复

```hcl
# Before (error):
vswitch_ids = optional(list(string)) # → null
security_group_key = optional(string) # → null

# After (fixed):
vswitch_ids = optional(list(string), []) # → []
selected_zones = optional(list(string), []) # → []
tags = optional(map(string), {}) # → {}
security_group_key = optional(string, "") # → ""
```

### Redis 模块模式（参考）

```hcl
# Redis uses this pattern consistently
variable "backup_period" {
  type = list(string)
  default = ["Monday", "Wednesday", "Friday"] # Always has default
}

# In main.tf
backup_period = length(var.backup_period) > 0 ? var.backup_period : null # Safe
```

## 参考资料

- [Terraform 可选属性](https://developer.hashicorp.com/terraform/language/expressions/type-constraints#optional-attributes)
