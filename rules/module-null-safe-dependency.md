# 控制层 null-safe 依赖保障机制

**优先级：** 高
**分类：** 模块设计

> 控制层模块间存在依赖链路，当父资源未创建时，子资源必须安全降级（自动 create=false），避免 null 值传播导致 apply 失败。

## 为什么重要

三层架构中，声明层通过 `create=false` 控制资源是否创建。当父资源 `create=false` 时：
- 父资源 ID 为 null
- 子资源的 `for_each` 中引用父资源 ID 的参数会是 null
- 如果不处理，Terraform apply 会因 null 值传入必填参数而失败

null-safe 保障机制确保：**即使声明层配置了父资源 create=false，子资源也能安全跳过创建，不会报错。**

## 依赖链路

```
ECS → LB → ServerGroup → Listener → Rule/DomainExtension
                ↓                    ↓
           (LB ID)            (ServerGroup ID)
```

## 四种保障模式

### 模式1：双条件 create（推荐，最常用）

> 适用场景：子资源依赖单个父资源 ID

```hcl
module "alb_listener" {
  source   = "../../atomic/alb-listener"
  for_each = var.alb_listeners

  # 保障：父资源 ALB 不存在时自动跳过创建
  create = try(each.value.create, true) && lookup(local.alb_ids_map, each.value.load_balancer_key, null) != null
  # 保障：若 load_balancer_id 为 null（父 ALB 未创建），安全降级
  load_balancer_id = lookup(local.alb_ids_map, each.value.load_balancer_key, null)
}
```

**原理**：
1. `lookup(local.xxx_ids_map, key, null)` — 父资源未创建时返回 null
2. `create = ... && xxx != null` — 双条件：声明层 create=true 且父资源存在
3. 即使 `create=false`，`load_balancer_id = null` 也不会报错（因为模块内部 count=0）

### 模式2：预过滤 locals（用于跨模块依赖检查）

> 适用场景：子资源依赖需要跨多个变量判断的条件（如 HTTPS Listener 需要证书）

```hcl
locals {
  # 判断每个 Listener 是否会创建（用于 Rule 依赖检查）
  slb_listener_create = {
    for k, v in var.slb_listeners : k => (
      try(v.create, true) && (
        v.protocol != "https" ||
        (try(v.server_certificate_id, "") != "" || var.ssl_cert_id != "")
      )
    )
  }

  # Rule 依赖检查：对应的 Listener 是否会创建
  slb_rule_listener_exists = {
    for rule_key, rule_val in var.slb_rules : rule_key => (
      length([for k, v in var.slb_listeners : k
        if v.load_balancer_key == rule_val.load_balancer_key
        && v.frontend_port == rule_val.frontend_port
        && local.slb_listener_create[k]
      ]) > 0
    )
  }
}

module "slb_rule" {
  source   = "../../atomic/slb-rule"
  for_each = var.slb_rules

  # Rule 依赖 Listener，检查对应的 Listener 是否会创建
  create = try(each.value.create, true) && local.slb_rule_listener_exists[each.key]
}
```

**原理**：
1. 先在 locals 中计算每个 Listener 的创建条件
2. 再根据 Rule 的 `load_balancer_key` + `frontend_port` 匹配对应 Listener
3. 如果匹配的 Listener 不会创建，Rule 也自动跳过

### 模式3：plan-time create 标志预过滤（路由条目级别）

> 适用场景：子资源使用 `resource for_each`，且 `if` 过滤条件不能依赖 apply-time 值（如父资源 ID），必须用 plan-time 可确定的 `create` 标志

**问题根因**：当父资源（如 NAT Gateway）被禁用（`create=false`）时，子资源（如路由条目）的 `nexthop_id` 会解析为 null。如果用 `if lookup(local.nexthop_ids, ...) != null` 做过滤，会因为 `nexthop_ids` 依赖 apply-time 值（`nat_gateway_id` 是 `known after apply`）导致 `Invalid for_each argument` 错误。

```text
错误链路：
local.nexthop_ids → local.nat_gateway_ids_map → module.nat[...].nat_gateway_id
                                                              ↑
                                                        known after apply
if 条件依赖 unknown 值 → for_each key 集合 unknown → 报错
```

**解决方案**：用 `var.nat_gateways` 的 `create` 字段构建 plan-time 可确定的标志映射，而非引用 apply-time 的资源 ID。

```hcl
# ✅ 步骤 1：在 locals 中构建 create 标志映射（plan-time 可确定）
locals {
  nat_gateway_creates = {
    for k, v in var.nat_gateways : k => try(v.create, true)
  }
}
```

```hcl
# ✅ 步骤 2：控制层 route_entries 的 for 表达式使用 create 标志过滤
module "route_table" {
  source   = "../../atomic/route-table"
  for_each = var.route_tables

  create = try(each.value.create, false)

  # null-safe 过滤：通过 plan-time create 标志判断父资源是否存在
  # 注意：不能使用 lookup(local.nexthop_ids, ...) 做 if 过滤
  #       因为 nexthop_ids 依赖 nat_gateway_id（known after apply）
  route_entries = {
    for idx, entry in try(each.value.route_entries, []) : "entry-${idx}" => {
      destination_cidrblock = entry.destination_cidr_block
      nexthop_type          = entry.nexthop_type
      nexthop_id            = lookup(local.nexthop_ids, "${entry.nexthop_type}:${entry.nexthop_key}", null)
    }
    # ✅ 用 create 标志过滤（plan-time 安全），不用 nexthop_ids（apply-time）
    if entry.nexthop_type != "NatGateway" ? true : try(local.nat_gateway_creates[entry.nexthop_key], true) != false
  }
}
```

**原理**：
1. `nat_gateway_creates` 从 `var.nat_gateways` 计算，值在 plan 阶段完全确定
2. `if` 条件只引用 plan-time 已知值，不触发 `Invalid for_each argument`
3. `nexthop_id` 仍用 `lookup(local.nexthop_ids, ...)` 获取真实 ID（apply-time 解析，放入 value 不影响 key 集合）
4. 非 NatGateway 类型的路由条目（如 Instance/VPN）不受影响

**与 `resource-foreach-unknown-value` 的关系**：本模式是该规则「模式 B（ecs_would_create 预过滤）」在路由条目场景的具体应用。核心思路一致：**用 var 中的 plan-time 已知属性（create 标志）替代 apply-time 才知道的资源 ID 做 if 过滤**。

**声明层最佳实践**：虽然控制层有防御性保障，声明层仍应显式同步禁用依赖资源：

```hcl
# ✅ 推荐：声明层显式表达业务依赖
nat_gateways = {
  "main" = { create = false }  # 禁用 NAT
}
route_tables = {
  "proxy" = { create = false }  # 同步禁用路由表（业务语义：无 NAT 则无路由意义）
}

# ✅ 也可：只禁用 NAT，路由表留 create=true，控制层会自动过滤路由条目
# 但路由表本身仍会创建（空表），属于资源浪费
```

> 适用场景：子资源参数中嵌套了需要转换的引用（如 ALB Rule actions 中的 server_group_key）

```hcl
# ALB Rule actions 中 server_group_key 自动转换为 server_group_id
rule_actions = [
  for action in try(each.value.actions, []) : merge(
    { for k, v in action : k => v if k != "server_group_tuples" },
    {
      server_group_tuples = [
        for sgt in try(action.server_group_tuples, []) : merge(
          { for k, v in sgt : k => v if k != "server_group_key" },
          try(sgt.server_group_key, null) != null ? {
            server_group_id = lookup(local.alb_server_group_ids_map, sgt.server_group_key, "")
          } : {}
        )
      ]
    }
  )
]
```

**原理**：
1. 使用 `merge` + `{ for k, v in x : k => v if k != "排除key" }` 移除原始 key
2. 条件性地添加转换后的 key→ID
3. 如果 server_group_key 引用的资源不存在，`lookup` 返回空字符串，Terraform 会在 plan 阶段提示

## ID 映射 locals 规范

所有 ID 映射必须过滤 null 值：

```hcl
locals {
  # ✅ 正确：过滤 null 值
  alb_ids_map = { for k, v in module.alb : k => v.alb_id if v.alb_id != null }

  # ❌ 错误：不过滤 null 值，会导致 lookup 返回 null
  alb_ids_map = { for k, v in module.alb : k => v.alb_id }
}
```

## 常见 key→ID 转换模式

| 引用类型 | 声明层字段 | 控制层转换 |
|---------|-----------|-----------|
| 父资源 ID | `load_balancer_key` | `lookup(local.alb_ids_map, each.value.load_balancer_key, null)` |
| Server Group | `server_group_key` | `lookup(local.alb_server_group_ids_map, each.value.server_group_key, null)` |
| ECS 实例 | `ecs_key` | `lookup(local.ecs_ids_map, s.ecs_key, null)` |
| 快照策略 | `auto_snapshot_policy_key` | `lookup(var.ecs_snapshot_policy_ids_map, key, "")` |
| 安全组 | `security_group_key` | 由控制层固定映射，不需要 lookup |

## 完整保障检查清单

- [ ] 所有子资源模块的 `create` 使用双条件（`try(each.value.create, true) && lookup(...) != null`）
- [ ] 所有 ID 映射 locals 过滤 null 值（`if v.xxx_id != null`）
- [ ] 所有 `lookup` 调用使用三参数形式提供默认值（`lookup(map, key, null)`）
- [ ] HTTPS 监听器有证书检查条件
- [ ] 深层嵌套引用（actions/servers）使用 merge + for 表达式做 key→ID 转换
- [ ] 路由条目的 `if` 过滤使用 `create` 标志映射（plan-time 安全），不使用 `local.nexthop_ids`（apply-time 值）
- [ ] 声明层禁用父资源时同步禁用子资源（显式表达业务依赖）

## 相关规则

- `module-dependency-prefilter` - 模块依赖预过滤
- `layer-id-mapping` - 通过 map 变量实现基于 key 的资源引用
- `module-parameter-completeness` - 原子层参数完整性检查
- `resource-foreach-unknown-value` - for_each 与 unknown 值：模式 B 的路由条目实例

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `control-backup-prefilter.md`（条件子资源预过滤）
- `variable-forcenew-defaults.md`（ForceNew 空值规则）
