# 控制层 Map 访问安全：必须使用 lookup() 保护

**优先级：** 高
**分类：** 模块设计

## 为什么重要

控制层大量使用 map 变量（`vswitch_ids_map`、`security_group_ids_map` 等）进行 ID 映射。直接使用 `map[key]` 访问时，如果 key 不存在会报 `Invalid index` 错误，导致整个 `terraform apply` 失败。使用 `lookup()` 提供安全默认值，可以在 key 不存在时优雅降级。

## 问题：直接访问 map 无保护

```hcl
# ❌ 错误：直接访问 map，key 不存在时报错
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ❌ 如果 security_group_key 在 map 中不存在 → Invalid index
  security_group_ids = [var.security_group_ids_map[each.value.security_group_key]]

  # ❌ 如果 vswitch_key 在 map 中不存在 → Invalid index
  vswitch_id = var.vswitch_ids_map[each.value.vswitch_key]
}
```

**问题：**
1. 用户在 tfvars 中配置了错误的 key（如拼写错误 "dta" 而非 "data"）
2. 上游阶段未创建对应的资源，map 中缺少该 key
3. 错误信息是 Terraform 运行时的 `Invalid index`，难以定位到具体的 key 问题

## 正确：使用 lookup() 提供安全默认值

```hcl
# ✅ 正确：lookup() 保护 + 合理默认值
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ✅ key 不存在时返回空字符串，后续逻辑可安全处理
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    []

  # ✅ key 不存在时返回空字符串，原子层会因参数为空而跳过创建
  vswitch_id = lookup(var.vswitch_ids_map, try(each.value.vswitch_key, ""), "")
}
```

## lookup() 默认值选择

| 场景 | 默认值 | 说明 |
|------|--------|------|
| 资源 ID（vswitch_id, sg_id） | `""` | 原子层检测空字符串后跳过创建 |
| 嵌套 map 访问 | `null` | 用于条件判断 `!= null` |
| 安全组 ID | `""` | 空列表 `[]` 表示不绑定安全组 |
| 布尔守卫 | `false` | 用于 create 判断 |

## 完整模式：Key 存在性检查 + lookup 保护

```hcl
# 标准模式：先检查 key 是否配置，再安全查找
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  []  # 未配置 security_group_key 时不绑定安全组

# ID 映射模式：在 locals 中构建，过滤 null 值
ecs_ids_map = {
  for k, v in module.ecs : k => v.instance_id
  if v.instance_id != null    # 过滤未创建的实例
}
```

## Plan-time 可解析要求

**关键：** `for_each` 的 `count` 和 create 判断中使用的条件必须是 plan-time 可解析的。

```hcl
# ❌ 错误：create 条件依赖 apply-time 才知的 module output
create = try(each.value.create, true) && module.ssl_self_signed.cas_cert_id != null

# ✅ 正确：create 条件使用 plan-time 已知的变量
create = try(each.value.create, true) && var.ssl_create_self_signed
```

**原因：** `count` 必须在 plan 阶段确定。如果 create 条件依赖 apply-time 值，Terraform 无法计算 count，会报 `Invalid count argument` 错误。

## 检查清单

- [ ] 控制层所有 `map[key]` 直接访问已替换为 `lookup(map, key, default)`
- [ ] `lookup()` 默认值类型与 map value 类型一致
- [ ] `for_each`/`count` 条件不依赖 apply-time 才知的值
- [ ] ID 映射 locals 中过滤了 null 值（`if v.id != null`）
- [ ] 嵌套的 `try()` + `lookup()` 不超过 2 层

## 参考资料

- `control-key-mapping-pattern.md`（Key 映射模式）
- `layer-id-mapping.md`（ID 映射模式）
- `module-null-safe-dependency.md`（null-safe 依赖保障）
