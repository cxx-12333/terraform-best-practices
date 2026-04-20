# 模块依赖预过滤：避免引用未创建的模块实例

**优先级：** 关键
**分类：** 模块设计

## 为什么重要

当一个模块（如 ECS）使用 `create = false` 跳过创建时，依赖它的模块（如 HBR 备份）如果仍然遍历原始 map，会因为引用不存在的模块实例而报错：

```
Error: "instance_id": required field is not set
```

这是 Terraform for_each 求值顺序导致的问题：**for_each 先求值，create 条件后求值**。

## 问题场景

```hcl
# 声明层配置
mysql_ecs_instances = {
  "01" = {
    create             = false    # ❌ ECS 不创建
    hbr_backup_enabled = true     # HBR 备份启用
  }
}

# 控制层代码（错误写法）
module "mysql_ecs" {
  source   = "../../atomic/ecs"
  for_each = var.mysql_ecs_instances
  create   = try(each.value.create, true)
  # ... 其他配置 ...
}

module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = var.mysql_ecs_instances    # ❌ 遍历原始 map

  # 当 ECS create = false 时，module.mysql_ecs["01"] 不存在！
  create      = try(each.value.create, true) && try(each.value.hbr_backup_enabled, false)
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")  # ❌ 报错！
}
```

**报错信息：**
```
Error: "instance_id": required field is not set
with module.db.module.mysql_ecs_backup_client["01"].alicloud_hbr_ecs_backup_client.this[0]
```

## 根因分析

| 求值顺序 | 说明 |
|----------|------|
| 1. for_each | 先确定要创建哪些模块实例 |
| 2. 模块引用 | module.mysql_ecs[each.key] 需要对应实例存在 |
| 3. create 条件 | 最后才判断是否真正创建资源 |

**关键问题**：当 ECS `create = false` 时，`module.mysql_ecs["01"]` 这个模块实例根本不存在，引用它的 `instance_id` 会直接报错，而不是返回空值。

## 解决方案：locals 预过滤

### 步骤 1：在 locals 中预过滤

```hcl
locals {
  # 预过滤：只包含同时满足以下条件的实例
  # 1. create = true（ECS 实例需要创建）
  # 2. hbr_backup_enabled = true（HBR 备份启用）
  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}
```

### 步骤 2：依赖模块使用预过滤后的 map

```hcl
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled    # ✅ 使用预过滤后的 map

  # 只需检查 ECS 实例是否创建成功（create 和 hbr_backup_enabled 已在 locals 中过滤）
  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}

module "mysql_ecs_backup_plan" {
  source   = "../../atomic/hbr-ecs-backup-plan"
  for_each = local.mysql_ecs_hbr_backup_enabled    # ✅ 使用预过滤后的 map

  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  # ... 其他配置 ...
}
```

## 决策矩阵

| ECS create | hbr_backup_enabled | 是否进入预过滤 map | HBR 模块行为 |
|------------|-------------------|-------------------|--------------|
| true | true | ✅ 进入 | 尝试创建 HBR 资源 |
| true | false | ❌ 不进入 | 不创建（跳过） |
| false | true | ❌ 不进入 | 不创建（跳过） |
| false | false | ❌ 不进入 | 不创建（跳过） |

## 完整示例

```hcl
# control/database-cluster/main.tf

locals {
  common_tags = merge(
    { "ops-env" = var.env, "project" = var.project },
    var.tags,
  )

  ################################################################################
  # MySQL ECS HBR 备份预过滤
  # 只包含同时满足以下条件的实例：
  #   1. create = true（ECS 实例需要创建）
  #   2. hbr_backup_enabled = true（HBR 备份启用）
  # 用途：避免 ECS 未创建时 HBR 模块引用不存在的 instance_id
  ################################################################################

  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}

# ECS 模块（正常遍历原始 map）
module "mysql_ecs" {
  source   = "../../atomic/ecs"
  for_each = var.mysql_ecs_instances

  create        = try(each.value.create, true)
  instance_name = coalesce(try(each.value.instance_name, ""), "ecs-${var.env}-mysql-${each.key}")
  # ... 其他配置 ...
}

# HBR 备份客户端（使用预过滤后的 map）
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled

  # create 和 hbr_backup_enabled 已在 local.mysql_ecs_hbr_backup_enabled 中过滤
  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}

# HBR 备份计划（使用预过滤后的 map）
module "mysql_ecs_backup_plan" {
  source   = "../../atomic/hbr-ecs-backup-plan"
  for_each = local.mysql_ecs_hbr_backup_enabled

  create                = try(module.mysql_ecs[each.key].instance_id, "") != ""
  ecs_backup_plan_name  = "plan-${var.env}-mysql-${each.key}"
  instance_id           = try(module.mysql_ecs[each.key].instance_id, "")
  vault_id              = try(each.value.hbr_vault_key, "") != "" ? lookup(var.hbr_vault_ids_map, each.value.hbr_vault_key, "") : ""
  backup_type           = try(each.value.hbr_backup_type, "COMPLETE")
  retention             = try(each.value.hbr_retention_days, "7")
  schedule              = try(each.value.hbr_schedule, "0 0 4 * * *")
  path                  = try(each.value.hbr_backup_paths, [])
  disabled              = try(each.value.hbr_backup_disabled, false)
}
```

## 声明层配置示例

```hcl
# declarative/simple/04-database/terraform.tfvars

mysql_ecs_instances = {
  "01" = {
    # ECS 实例配置
    create        = true    # ✅ ECS 创建
    instance_type = "ecs.g7a.xlarge"
    image_id      = "ubuntu_22_04_x64_20G_alibase_20240808.vhd"
  
    # HBR 云备份配置
    hbr_backup_enabled  = true              # ✅ 启用 HBR 备份
    hbr_vault_key       = "default"
    hbr_retention_days  = "7"
    hbr_schedule        = "0 0 4 * * *"
    hbr_backup_paths    = ["/data/mysql"]
  }

  "02" = {
    create              = false    # ❌ ECS 不创建
    hbr_backup_enabled  = true     # 即使启用也会被预过滤跳过
    # ...
  }
}
```

## 常见陷阱

### 陷阱 1：只检查 create 条件，忽略 for_each

```hcl
# ❌ 错误：create 条件正确，但 for_each 仍遍历原始 map
module "backup" {
  for_each = var.mysql_ecs_instances    # 这里仍然包含 create=false 的实例
  create   = try(each.value.create, true) && try(each.value.hbr_backup_enabled, false)
  # 当 create=false 时，引用 module.mysql_ecs[each.key] 会报错
}
```

### 陷阱 2：使用 count 而非 for_each

```hcl
# ❌ 错误：使用 count 会导致索引错乱
module "backup" {
  count = var.mysql_ecs.create && var.mysql_ecs.hbr_backup_enabled ? 1 : 0
  instance_id = module.mysql_ecs[0].instance_id    # 索引可能不存在
}
```

## 检查清单

- [ ] 依赖模块使用 locals 预过滤后的 map 作为 for_each
- [ ] 预过滤条件包含 create = true 和功能启用开关
- [ ] 预过滤注释说明过滤条件
- [ ] 依赖模块的 create 只检查资源是否创建成功（无需重复检查 create 条件）
- [ ] 测试 create = false 场景，确认依赖模块不会报错

## 适用场景

| 场景 | 被依赖资源 | 依赖资源 | 预过滤条件 |
|------|-----------|----------|-----------|
| ECS + HBR 备份 | ECS 实例 | HBR Client/Plan | create && hbr_backup_enabled |
| ECS + EIP 绑定 | ECS 实例 | EIP Association | create && eip_enabled |
| ECS + SLB 后端 | ECS 实例 | SLB Attachment | create && attach_to_slb |
| RDS + 只读实例 | RDS 主实例 | RDS Read Replica | create && create_read_replica |

## 参考资料

- Terraform for_each 求值顺序文档