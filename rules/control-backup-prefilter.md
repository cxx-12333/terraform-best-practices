# 控制层条件子资源预过滤模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

控制层常需要创建附加资源（备份计划、快照策略等），这些资源依赖主资源的 ID。当主资源未创建（create=false）时，附加资源的 `for_each` 必须排除这些实例，否则会因引用不存在的 module output 而 apply 失败。使用 locals 预过滤是最安全的方案。

## 模式说明

### 问题场景

```
mysql_ecs_instances = {
  "01" = { create = true,  hbr_backup_enabled = true  }  → 需要 HBR 备份
  "02" = { create = false, hbr_backup_enabled = true  }  → ECS 不创建，备份不应创建
  "03" = { create = true,  hbr_backup_enabled = false }  → ECS 创建，但不需要备份
}
```

### 预过滤模式

```hcl
locals {
  # 只保留同时满足两个条件的实例：
  #   1. create = true（ECS 实例需要创建）
  #   2. hbr_backup_enabled = true（HBR 备份启用）
  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}
```

然后在附加资源中使用过滤后的 map：

```hcl
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled  # 使用预过滤后的 map

  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}
```

## 关键设计要素

### 1. 过滤条件链

多个条件用 `&&` 连接，按优先级排列：

```hcl
# 条件 1：主资源是否创建
# 条件 2：附加功能是否启用
if try(v.create, true) && try(v.hbr_backup_enabled, false)
```

### 2. 双重保护

预过滤 + 模块内 create 检查：

```hcl
# 第一层保护：预过滤排除不需要的实例
for_each = local.mysql_ecs_hbr_backup_enabled

# 第二层保护：检查主资源是否真的创建成功
create = try(module.mysql_ecs[each.key].instance_id, "") != ""
```

### 3. key 保持一致

过滤后的 map 保持与原始 map 相同的 key，以便通过 `module.main[each.key]` 回引：

```hcl
for k, v in var.mysql_ecs_instances : k => v  # k 不变
```

## 多云场景适配

### AWS 备份预过滤

```hcl
locals {
  # AWS Backup：过滤启用备份的 RDS 实例
  rds_backup_enabled = {
    for k, v in var.rds_instances : k => v
    if try(v.create, true) && try(v.backup_enabled, false)
  }
}

resource "aws_backup_selection" "rds" {
  for_each = local.rds_backup_enabled

  plan_id      = var.backup_plan_id
  resources    = [module.rds[each.key].instance_arn]
}
```

### Azure 备份预过滤

```hcl
locals {
  # Azure Recovery Services Vault：过滤启用备份的 VM
  vm_backup_enabled = {
    for k, v in var.virtual_machines : k => v
    if try(v.create, true) && try(v.backup_enabled, false)
  }
}

resource "azurerm_backup_protected_vm" "vm" {
  for_each = local.vm_backup_enabled

  recovery_vault_name = var.recovery_vault_name
  resource_group_name = var.resource_group_name
  source_vm_id        = module.vm[each.key].vm_id
}
```

## 适用场景

| 场景 | 主资源 | 附加资源 | 过滤条件 |
|------|--------|----------|----------|
| ECS 备份 | ECS 实例 | HBR 备份客户端 + 计划 | create && hbr_backup_enabled |
| 磁盘快照 | ECS 实例 | 自动快照策略 | create && has_data_disks |
| 账号创建 | Redis/Kafka | 子账号 | create && create_account |
| 监控配置 | 任意资源 | CloudMonitor 告警 | create && monitoring_enabled |
| RDS 备份 | RDS 实例 | AWS Backup 策略 | create && backup_enabled |
| VM 备份 | Azure VM | Recovery Services | create && backup_enabled |

## 与嵌套字段的对比

### 方案 A：预过滤（推荐）

```hcl
# locals 中过滤
hbr_backup_enabled = {
  for k, v in var.instances : k => v
  if try(v.create, true) && try(v.hbr_backup_enabled, false)
}

# 模块引用
for_each = local.hbr_backup_enabled
```

### 方案 B：嵌套字段展平（适用于子资源有独立 key 时）

```hcl
# 展平子资源
for_each = {
  for pair in flatten([
    for k, v in var.instances : [
      for plan_key, plan in try(v.backup_plans, {}) : { ... }
    ] if try(v.create, true)
  ]) : "${pair.k}-${pair.plan_key}" => pair
}
```

**选择依据**：子资源是否需要独立的 key 管理。需要 → 方案 B；不需要 → 方案 A。

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤通用模式）
- `module-null-safe-dependency.md`（null-safe 依赖）
- `control-nested-resource-flatten.md`（嵌套资源展平模式）
- `control-parameter-passthrough.md`（控制层参数透传）
