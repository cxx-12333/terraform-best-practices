# 控制层嵌套资源 for_each 展平模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

当父资源有子资源时（如 Kafka 实例下的 SASL 用户），子资源依赖父资源的输出（instance_id）。使用 `flatten` + 嵌套 `for` 将两层 map 展平为一层，再通过复合 key (`"${parent_key}-${child_key}"`) 索引回父资源，实现一对多关系的安全编排。

## 模式说明

### 问题场景

```
Kafka 实例 "01" → SASL 用户 admin, reader
Kafka 实例 "02" → SASL 用户 admin
```

Terraform 的 `for_each` 只支持一层 map/list，无法直接表达"每个 Kafka 创建多个用户"。

### 展平模式

```hcl
module "kafka_sasl_user" {
  source = "../../atomic/kafka-sasl-user"

  # 展平：两层 map → 一层 map，复合 key
  for_each = {
    for pair in flatten([
      for kafka_key, kafka in var.kafka_instances : [
        for user_key, user in try(kafka.sasl_users, {}) : {
          kafka_key     = kafka_key
          user_key      = user_key
          user          = user
          kafka_created = try(kafka.create, true)
        }
      ] if try(kafka.create, true)  # 过滤掉不创建的 Kafka 实例
    ]) : "${pair.kafka_key}-${pair.user_key}" => pair
  }

  create    = try(each.value.user.create, true)
  instance_id = module.kafka[each.value.kafka_key].instance_id
  username  = each.value.user_key
  password  = try(each.value.user.password, "")
}
```

## 关键设计要素

### 1. 复合 Key 格式

```
${parent_key}-${child_key}
```

| 父子关系 | 复合 Key 示例 |
|----------|--------------|
| Kafka → SASL 用户 | `01-admin` |
| Kafka → ACL | `01-topic-write` |
| MongoDB → 账号 | `01-root` |
| RDS → 只读实例 | `01-reader-01` |

### 2. 父资源过滤

只展平已创建的父资源的子资源：

```hcl
if try(kafka.create, true)  # 父资源不创建 → 子资源也不创建
```

### 3. 回引父资源输出

通过 `each.value.kafka_key` 回引父模块：

```hcl
instance_id = module.kafka[each.value.kafka_key].instance_id
```

### 4. 子资源独立控制开关

每个子资源仍有自己的 `create` 标志：

```hcl
create = try(each.value.user.create, true)
```

## 声明层写法

```hcl
kafka_instances = {
  "01" = {
    create        = true
    instance_type = "alikafka_serverless"
    # ...

    # 子资源嵌套在父资源内部
    sasl_users = {
      admin = {
        password = "Admin123!"
        mechanism = "PLAIN"
      }
      reader = {
        password = "Reader123!"
        create   = false  # 暂不创建
      }
    }
  }
}
```

## 多云场景示例

### AWS MSK → SASL/IAM 用户

```hcl
module "msk_sasl_user" {
  source = "../../atomic/msk-sasl-user"

  for_each = {
    for pair in flatten([
      for cluster_key, cluster in var.msk_clusters : [
        for user_key, user in try(cluster.sasl_users, {}) : {
          cluster_key = cluster_key
          user_key    = user_key
          user        = user
        }
      ] if try(cluster.create, true)
    ]) : "${pair.cluster_key}-${pair.user_key}" => pair
  }

  create      = try(each.value.user.create, true)
  cluster_arn = module.msk[each.value.cluster_key].cluster_arn
  username    = each.value.user_key
  password    = try(each.value.user.password, "")
}
```

### Azure Event Hub → 授权规则

```hcl
module "eventhub_auth_rule" {
  source = "../../atomic/eventhub-authorization-rule"

  for_each = {
    for pair in flatten([
      for hub_key, hub in var.event_hubs : [
        for rule_key, rule in try(hub.authorization_rules, {}) : {
          hub_key  = hub_key
          rule_key = rule_key
          rule     = rule
        }
      ] if try(hub.create, true)
    ]) : "${pair.hub_key}-${pair.rule_key}" => pair
  }

  create           = try(each.value.rule.create, true)
  eventhub_id      = module.eventhub[each.value.hub_key].eventhub_id
  rule_name        = each.value.rule_key
  permissions      = try(each.value.rule.permissions, ["Listen"])
}
```

## 更多层级：三层展平

当存在三层关系（如 Kafka → ACL 依赖 SASL 用户）时，使用 `depends_on` 确保创建顺序：

```hcl
module "kafka_sasl_acl" {
  source = "../../atomic/kafka-sasl-acl"

  for_each = {
    for pair in flatten([
      for kafka_key, kafka in var.kafka_instances : [
        for acl_key, acl in try(kafka.sasl_acls, {}) : {
          kafka_key = kafka_key
          acl_key   = acl_key
          acl       = acl
        }
      ] if try(kafka.create, true)
    ]) : "${pair.kafka_key}-${pair.acl_key}" => pair
  }

  create      = try(each.value.acl.create, true)
  instance_id = module.kafka[each.value.kafka_key].instance_id

  # 确保 SASL 用户先创建
  depends_on = [module.kafka_sasl_user]
}
```

## 模式总结

| 要素 | 说明 |
|------|------|
| 展平函数 | `flatten([for parent: [for child: {复合对象}]])` |
| 复合 Key | `"${parent_key}-${child_key}"` |
| 父资源过滤 | `if try(parent.create, true)` |
| 回引父输出 | `module.parent[each.value.parent_key].output` |
| 创建顺序 | `depends_on = [module.parent_child]` |

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `module-null-safe-dependency.md`（null-safe 依赖）
- `control-backup-prefilter.md`（条件子资源预过滤模式）
