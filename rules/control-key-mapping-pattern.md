# 控制层安全组 Key 映射模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

安全组 ID 是跨控制层模块共享的基础设施引用。直接硬编码安全组 ID 会导致环境不可移植。通过 `security_group_key` + `security_group_ids_map` 的 key 映射模式，声明层只需指定逻辑名称，控制层负责解析为实际 ID。

## 模式说明

### 映射架构

```
声明层                    控制层                     安全组模块
security_group_key    →  lookup(security_group_ids_map, key)  →  security_group_id
= "data"                  = "sg-xxx"                       = "sg-xxx"
```

### 控制层变量定义

```hcl
variable "security_group_ids_map" {
  description = "安全组 ID 映射表。key=逻辑名, value=安全组 ID"
  type        = map(string)
  default     = {}
}
```

### 控制层使用模式

#### 模式 1：单个安全组 Key

```hcl
# 最常用：单个安全组 key → 单个安全组 ID
security_group_id = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : ""
```

#### 模式 2：安全组 Key → 安全组 ID 列表

```hcl
# 资源需要安全组 ID 列表
security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

#### 模式 3：多个安全组 Key

```hcl
# ECS 绑定多个安全组（data + ecs）
security_group_ids = [
  var.security_group_ids_map[try(each.value.security_group_key, "data")]
]
```

#### 模式 4：优先 Key 回退直接 ID

```hcl
# 优先使用 key 查找，否则使用直接 ID
security_group = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : try(each.value.security_group, "")
```

## 设计原则

### 1. 控制层不设安全组业务默认值

```hcl
# 错误：控制层硬编码安全组 ID
security_group_id = "sg-xxx"

# 正确：控制层通过 key 映射查找
security_group_id = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : ""
```

> 安全组由独立的 `security` 控制层模块管理，其他控制层通过 key 引用。

### 2. Key 使用短逻辑名

| Key 名 | 含义 | 说明 |
|--------|------|------|
| `data` | 数据层安全组 | 数据库、缓存通用 |
| `web` | Web 层安全组 | ECS、SLB 通用 |
| `ecs` | ECS 通用安全组 | 自建服务 |
| `kafka` | Kafka 专用安全组 | 消息队列 |

### 3. lookup 保护的必要性

```hcl
# 安全：key 不存在时返回空字符串
lookup(var.security_group_ids_map, each.value.security_group_key, "")

# 不安全：key 不存在时报错
var.security_group_ids_map[each.value.security_group_key]
```

## 声明层写法

```hcl
# terraform.tfvars
kafka_instances = {
  "01" = {
    security_group_key = "data"  # 引用逻辑名
    # ...
  }
}

redis_community_instances = {
  "01" = {
    security_group_key = "data"  # 同一逻辑名，不同资源
    # ...
  }
}
```

## 多云场景适配

### AWS 安全组映射

```hcl
# AWS 使用 vpc_security_group_ids
vpc_security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

### Azure NSG 映射

```hcl
# Azure 使用 network_interface_security_group_association
# key 映射到 NSG ID
network_security_group_id = try(each.value.nsg_key, "") != ""
  ? lookup(var.nsg_ids_map, each.value.nsg_key, "")
  : try(each.value.network_security_group_id, "")
```

### 腾讯云安全组映射

```hcl
# 腾讯云使用 security_groups
security_groups = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

## 相同模式的其他 Key 映射

| Key 字段 | Map 变量 | 用途 |
|----------|----------|------|
| `vswitch_key` / `subnet_key` | `vswitch_ids_map` / `subnet_ids_map` | 子网 ID 查找 |
| `zone_key` | `az_map` | 可用区 ID 查找 |
| `security_group_key` / `nsg_key` | `security_group_ids_map` / `nsg_ids_map` | 安全组 ID 查找 |
| `hbr_vault_key` / `backup_vault_key` | `hbr_vault_ids_map` / `backup_vault_ids_map` | 备份库 ID 查找 |
| `system_disk_auto_snapshot_policy_key` | `ecs_snapshot_policy_ids_map` | 快照策略 ID 查找 |

所有 key 映射遵循统一模式：

```hcl
try(each.value.{key}_field, "") != ""
  ? lookup(var.{resource}_ids_map, each.value.{key}_field, "")
  : try(each.value.{direct_id_field}, "")
```

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `layer-id-mapping.md`（ID 映射模式）
- `control-parameter-passthrough.md`（控制层参数透传）
