# 控制层参数透传与控制层业务默认值

**优先级：** 中高
**分类：** 三层架构

## 为什么重要

控制层在声明层和原子层之间承担"透传 + 填充默认值"的角色。区分哪些默认值应该设在控制层、哪些在原子层，是三层架构的关键决策。错误放置默认值会导致隐藏配置或状态漂移。

## 核心原则

### 原子层：技术纯度默认值

```hcl
# 原子层 variables.tf — 技术纯度原则
variable "strict_consistency" {
  type    = string
  default = ""  # 空字符串 = 不设置，让 API 决定
  description = "ForceNew 参数。空字符串不传递给 API。"
}
```

### 控制层：业务默认值

```hcl
# 控制层 main.tf — 业务默认值
strict_consistency = try(each.value.strict_consistency, "")  # 不设默认
lower_case_table_names = try(each.value.lower_case_table_names, "1")  # 业务默认 "1"
```

### 声明层：显式配置

```hcl
# 声明层 terraform.tfvars — 只写需要覆盖的值
lower_case_table_names = "0"  # 显式覆盖默认值
```

## 透传模式

### 模式 1：纯透传（无默认值）

声明层传什么，原子层就收到什么：

```hcl
# 控制层
password = try(each.value.password, "")
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `password = "xxx"` | `"xxx"` | `"xxx"` |
| 不设置 | `""` | `""` → null（原子层处理） |

### 模式 2：透传 + 业务默认值

声明层不设置时使用控制层默认值：

```hcl
# 控制层
engine_version = try(each.value.engine_version, "7.0")
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `engine_version = "5.0"` | `"5.0"` | `"5.0"` |
| 不设置 | `"7.0"` | `"7.0"` |

### 模式 3：透传 + Key 解析

声明层给 key，控制层解析为 ID：

```hcl
# 控制层
vswitch_id = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `vswitch_key = "j-ecs"` | `vswitch_ids_map["j-ecs"]` | `"vsw-xxx"` |
| 不设置 | `vswitch_ids_map["j-data"]` | `"vsw-yyy"` |

### 模式 4：透传 + locals 预处理

声明层数据需要加工后再传给原子层：

```hcl
# 控制层 locals 预处理
mysql_ecs_data_disks_resolved = {
  for k, v in var.mysql_ecs_instances : k => [
    for disk in try(v.data_disks, []) : {
      name                    = disk.name
      auto_snapshot_policy_id = try(disk.auto_snapshot_policy_key, "") != ""
        ? lookup(var.ecs_snapshot_policy_ids_map, disk.auto_snapshot_policy_key, "")
        : try(disk.auto_snapshot_policy_id, "")
    }
  ]
}

# 控制层使用预处理结果
data_disks = local.mysql_ecs_data_disks_resolved[each.key]
```

## 默认值放置决策矩阵

| 场景 | 放置位置 | 说明 |
|------|----------|------|
| API 返回值决定的参数 | 原子层 `default = ""` | 让 API 自动填充 |
| ForceNew 参数 | 原子层 `default = ""` | 空值不触发重建 |
| 业务通用默认值（如 engine_version） | 控制层 `try(..., "7.0")` | 项目统一标准 |
| 基础设施默认值（如 vswitch_key） | 控制层 `try(..., "j-data")` | 环境级别默认 |
| 安全组 ID | 控制层 key 映射 | 跨模块引用 |
| 集合类型（list/map） | 控制层 `try(..., [])` | 避免 null |
| 标签 | 控制层 `merge(local.common_tags, ...)` | 统一标签策略 |

## 多云场景适配

### AWS 参数透传

```hcl
# AWS RDS 控制层透传
module "rds" {
  source = "../../atomic/aws-rds-instance"

  for_each = var.rds_instances

  engine               = try(each.value.engine, "mysql")
  engine_version       = try(each.value.engine_version, "8.0")
  instance_class       = try(each.value.instance_class, "db.t3.medium")
  allocated_storage    = try(each.value.allocated_storage, 20)
  subnet_group_name    = try(var.subnet_ids_map[each.value.subnet_key], "")
  vpc_security_group_ids = try(each.value.security_group_key, "") != ""
    ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
    : try(each.value.security_group_ids, [])
}
```

### Azure 参数透传

```hcl
# Azure MySQL 控制层透传
module "mysql" {
  source = "../../atomic/azurerm-mysql-flexible-server"

  for_each = var.mysql_servers

  sku_name             = try(each.value.sku_name, "B_Standard_B1s")
  version              = try(each.value.version, "8.0.21")
  resource_group_name  = var.resource_group_name
  location             = var.location
  subnet_id            = try(var.subnet_ids_map[each.value.subnet_key], "")
}
```

## 安全组默认值原则

```hcl
# 正确：控制层不设安全组业务默认值
# 声明层必须显式配置 security_group_key
security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])

# 错误：控制层硬编码安全组
security_group_ids = ["sg-xxx"]  # 不可移植
```

## security_ips 的特殊性

```hcl
# 正确：security_ips 的业务默认值是 ["127.0.0.1"]
# 这是 API 的默认行为，控制层需要与 API 保持一致
security_ips = try(each.value.security_ips, ["127.0.0.1"])

# 原因：如果不传 security_ips，API 仍然会设置 ["127.0.0.1"]
# 透传时也设置 ["127.0.0.1"]，才能避免状态漂移
```

## 参考资料

- `variable-atomic-defaults.md`（原子层默认值原则）
- `variable-forcenew-defaults.md`（ForceNew 参数空值规则）
- `layer-separation.md`（三层职责划分）
- `control-key-mapping-pattern.md`（安全组 Key 映射模式）
