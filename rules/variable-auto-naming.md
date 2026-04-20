# 控制层自动命名

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

在三层 IaC 架构中，声明层手动命名会导致冗余和不一致。控制层自动命名确保：
- **一致性**：所有资源遵循相同的命名模式
- **简洁性**：声明层只需 `key`，不需要 `instance_name`
- **可维护性**：修改命名模式只需更新控制层

## 模式：声明层 → 控制层自动命名

### 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Declaration Layer (terraform.tfvars) │
│ │
│ kafka_instances = { │
  │ "01" = { │
    │ vswitch_key = "j-data" │
    │ # instance_name NOT required! │
    │ } │
    │ } │
    └──────────────────────────────┬──────────────────────────────────────────┘
    │ key = "01"
    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │ Control Layer (main.tf) │
    │ │
    │ instance_name = coalesce( │
    │ try(each.value.instance_name, ""), │
    │ "kafka-${var.env}-${each.key}" # ← Auto-generated! │
    │ ) │
    │ │
    │ # Result: kafka-simple-01 │
    └─────────────────────────────────────────────────────────────────────────┘
```

### 控制层实现

```hcl
# control/database-cluster/main.tf（控制层模块）

module "kafka" {
  source = "../../atomic/kafka"
  for_each = var.kafka_instances

  # Auto-naming pattern: {resource_type}-{env}-{key}
  instance_name = coalesce(
  try(each.value.instance_name, ""), # Use custom name if provided
  "kafka-${var.env}-${each.key}" # Otherwise auto-generate
  )

  # Other parameters...
}

module "redis_community" {
  source = "../../atomic/redis-community"
  for_each = var.redis_community_instances

  # Auto-naming pattern: {resource_type}-{env}-{key}
  instance_name = coalesce(
  try(each.value.instance_name, ""),
  "redis-${var.env}-${each.key}"
  )
}

module "polardb" {
  source = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # Auto-naming pattern for cluster_description
  cluster_description = coalesce(
  try(each.value.cluster_name, ""),
  "polar-${var.env}-${each.key}"
  )
}
```

## 命名规范表

| Resource Type | Pattern | Example (env=simple, key=01) |
|---------------|---------|------------------------------|
| Kafka | `kafka-{env}-{key}` | `kafka-simple-01` |
| Redis Community | `redis-{env}-{key}` | `redis-simple-01` |
| Tair | `tair-{env}-{key}` | `tair-simple-01` |
| PolarDB | `polar-{env}-{key}` | `polar-simple-01` |
| Elasticsearch | `es-{env}-{key}` | `es-simple-01` |
| RocketMQ | `rmq-{env}-{key}` | `rmq-simple-01` |
| MySQL ECS | `ecs-{env}-mysql-{key}` | `ecs-simple-mysql-01` |

## 声明层示例

### 最小配置（推荐）

```hcl
# terraform.tfvars - Clean and simple
kafka_instances = {
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    paid_type = "PostPaid"
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
    # instance_name omitted - auto-generated!
  }
}
# Result: kafka-simple-01
```

### 自定义名称（按需）

```hcl
# terraform.tfvars - Override auto-naming when necessary
kafka_instances = {
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    instance_name = "kafka-orders-service" # Custom name for special case
    paid_type = "PostPaid"
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
  }
}
# Result: kafka-orders-service (custom name used)
```

## 实现最佳实践

### 1. 使用 `coalesce()` 提供回退

```hcl
# Correct - coalesce provides clean fallback
instance_name = coalesce(
try(each.value.instance_name, ""),
"kafka-${var.env}-${each.key}"
)

# Avoid - verbose and error-prone
instance_name = try(each.value.instance_name, "") != "" ? each.value.instance_name : "kafka-${var.env}-${each.key}"
```

### 2. 使用 `try()` 安全访问

```hcl
# Correct - handles missing key gracefully
try(each.value.instance_name, "")

# Avoid - fails if key doesn't exist
each.value.instance_name
```

### 3. 跨资源保持一致的命名模式

```hcl
# Consistent naming pattern for all resources
# Kafka: kafka-{env}-{key}
# Redis: redis-{env}-{key}
# PolarDB: polar-{env}-{key}
# ES: es-{env}-{key}

# Avoid inconsistent patterns
# Kafka: kafka-{env}-{key}
# Redis: {env}-redis-{key} # Different pattern
# PolarDB: polardb_cluster_{key} # Completely different
```

## 核心规则总结

| 规则 | 说明 |
|------|------|
| **规则 1** | 声明层默认省略 `instance_name` |
| **规则 2** | 控制层自动生成：`{resource_type}-{env}-{key}` |
| **规则 3** | 使用 `coalesce(try(...), auto_pattern)` 提供回退 |
| **规则 4** | 仅在业务需要特定命名时使用自定义名称 |
| **规则 5** | 所有资源保持命名模式一致 |

## 收益总结

| 之前（手动命名） | 之后（自动命名） |
|------------------|------------------|
| 每个实例都需要 `instance_name` | 仅在需要自定义名称时指定 |
| 命名模式不一致 | 强制一致的命名模式 |
| 命名变更需手动更新 | 单一位置更新模式 |
| 更多 tfvars 内容需要维护 | 更简洁、可读的 tfvars |

## 多云命名对照

自动命名模式（coalesce + try）在各云厂商通用，只需调整资源类型前缀：

| 云厂商 | 命名模板 | 示例输出 |
|--------|---------|---------|
| 阿里云 | `"{type}-${var.env}-${each.key}"` | `"polar-simple-01"` |
| AWS | `"{type}-${var.env}-${each.key}"` | `"rds-simple-01"` |
| 腾讯云 | `"{type}-${var.env}-${each.key}"` | `"tc-redis-simple-01"` |
| Azure | `"{type}-${var.env}-${each.key}"` | `"az-mysql-simple-01"` |

命名规则本身是架构层面的决策，与云厂商无关。

## 参考资料

- 相关规则：`variable-layered-type-design.md`