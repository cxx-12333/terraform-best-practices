# for_each 与 unknown 值：dynamic 块优先策略

**优先级：** 关键
**分类：** 资源组织

## 为什么重要

当控制层 `servers` 列表通过 `if` 过滤空 `server_id` 时，如果 `server_id` 来自 `lookup(local.ecs_ids_map, ...)`，在 ECS 首次创建时该值为 `known after apply`。Terraform 要求 `resource for_each` 的 map **key 集合**在 plan 阶段可确定——`if` 条件依赖 unknown 值会导致整个 map 变为 unknown，触发 `Invalid for_each argument` 错误。

**与 `module-dependency-prefilter` 的区别**：预过滤解决的是 `create=false` 导致模块实例不存在的问题（plan-time 可判断）；本规则解决的是 **value 为 unknown 导致 for_each key 集合不可确定**的问题（apply-time 才知道）。

## 根因分析

```
控制层: servers = [for s in var.servers : ... if lookup(local.ecs_ids_map, s.ecs_key, "") != ""]
                        ↑                              ↑
                   var.servers 长度已知           ECS ID 是 known after apply
                   (plan-time)                    (apply-time)

结果: if 条件依赖 unknown 值 → 列表长度 unknown → for_each map key 集合 unknown → 报错
```

| 求值阶段 | `if` 过滤 | `dynamic` 块 | `for_each`（无 if） |
|----------|----------|-------------|-------------------|
| plan | 列表长度 unknown → 报错 | 不影响 resource 数量 | 列表长度已知（来自 tfvars） |
| apply | 正常 | 正常 | server_id 已有值 |

## 决策矩阵：两种模式

### 模式 A：原子层改用 `dynamic` 块（优先推荐）

> 适用场景：后端服务器是资源的**嵌套属性**（如 ALB ServerGroup servers、SLB Backend servers）

**原理**：`dynamic` 块的 `for_each` 不影响 resource 数量，允许 unknown 值。控制层可安全使用 `if` 过滤。

```hcl
# ✅ 原子层：dynamic 块
resource "alicloud_slb_backend_server" "this" {
  count = local.create ? 1 : 0

  load_balancer_id = var.load_balancer_id

  dynamic "backend_servers" {
    for_each = [for s in var.servers : {
      server_id = s.server_id
      weight    = lookup(s, "weight", 100)
      type      = lookup(s, "type", "ecs")
    }]
    content {
      server_id = backend_servers.value.server_id
      weight    = backend_servers.value.weight
      type      = backend_servers.value.type
    }
  }
}
```

```hcl
# ✅ 控制层：可以安全使用 if 过滤（dynamic 块允许 unknown 的 for_each）
module "slb_backend_attachment" {
  source   = "../../atomic/slb-backend-attachment"
  for_each = var.slb_backend_attachments

  create = try(each.value.create, true) && lookup(local.slb_would_create, each.value.load_balancer_key, false)
  load_balancer_id = lookup(local.slb_ids_map, each.value.load_balancer_key, null)
  servers = [
    for s in try(each.value.servers, []) : merge(
      { for k, v in s : k => v if k != "ecs_key" },
      try(s.ecs_key, null) != null ? {
        server_id = lookup(local.ecs_ids_map, s.ecs_key, "")
      } : {}
    )
    if (try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")) != ""
  ]
  depends_on = [module.ecs]
}
```

### 模式 B：原子层保持 `resource for_each`，控制层用 `ecs_would_create` 预过滤

> 适用场景：后端服务器是**独立资源**（如 SLB Server Group Attachment、NLB Server Group Attachment）

**原理**：独立资源必须用 `resource for_each`，无法改为 `dynamic` 块。不能直接用 `lookup(local.ecs_ids_map, ...) != ""` 做 `if` 过滤（依赖 apply-time 值），但可以用 `var.ecs_instances` 的 **plan-time 已知属性** 预计算哪些 ECS 会创建，再用这个集合做 `if` 过滤。

**关键区分**：
- `lookup(local.ecs_ids_map, key, "") != ""` → 依赖 apply-time 值 → ❌ 不可用
- `contains(local.ecs_would_create, key)` → 纯 var 计算 → ✅ plan-time 可确定

```hcl
# ✅ 步骤 1：在 locals 中预计算 ECS 会创建的 key 集合
# 条件必须与 ECS 模块的 create 一致：try(v.create, try(v.image_id, "") != "")
locals {
  ecs_would_create = toset([
    for k, v in var.ecs_instances : k
    if try(v.create, try(v.image_id, "") != "")
  ])
}
```

```hcl
# ✅ 步骤 2：原子层用静态 key 的 for_each
locals {
  servers_map = local.create && length(var.servers) > 0 ? {
    for idx, obj in var.servers : "server-${idx}" => {
      server_id = obj.server_id
      port      = obj.port
      weight    = lookup(obj, "weight", 100)
    }
  } : {}
}

resource "alicloud_nlb_server_group_server_attachment" "this" {
  for_each = local.servers_map

  server_group_id = var.server_group_id
  server_id       = each.value.server_id
  port            = each.value.port
  weight          = each.value.weight
  server_type     = each.value.type
}
```

```hcl
# ✅ 步骤 3：控制层用 ecs_would_create 预过滤（plan-time 安全）
module "nlb_server_group_server_attachment" {
  source   = "../../atomic/nlb-server-group-server-attachment"
  for_each = var.nlb_server_groups

  create          = try(each.value.create, true)
  server_group_id = lookup(local.nlb_server_group_ids_map, each.key, null)
  servers = [
    for s in try(each.value.servers, []) : merge(s, {
      server_id = try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")
    })
    # ✅ 用 ecs_would_create 过滤 create=false 的 ECS（plan-time 安全）
    if try(s.ecs_key, null) != null ? contains(local.ecs_would_create, s.ecs_key) : try(s.server_id, "") != ""
  ]

  depends_on = [module.ecs, module.nlb_server_group]
}
```

## 分类速查表

| 资源 | 嵌套方式 | 推荐模式 | 控制层 if 过滤 | 原子层实现 |
|------|---------|---------|--------------|-----------|
| ALB ServerGroup servers | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "servers"` |
| SLB Backend servers | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "backend_servers"` |
| SLB Server Group Attachment | 独立 resource | B: for_each | **ecs_would_create** | `resource for_each` + 静态 key |
| NLB Server Group Attachment | 独立 resource | B: for_each | **ecs_would_create** | `resource for_each` + 静态 key |
| ALB Listener actions | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "actions"` |
| Route Entry (NAT nexthop) | 独立 resource | B: for_each | **nat_gateway_creates** | `resource for_each` + 静态 key |

## 判断流程

```
后端服务器配置是否是独立 resource？
├─ 否 → 是资源的嵌套属性 → 模式 A（dynamic 块）
│        控制层保留 if 过滤，原子层改用 dynamic
└─ 是 → 独立 resource for_each → 模式 B（ecs_would_create 预过滤）
         控制层用 ecs_would_create 做 if 过滤（plan-time 安全）
         原子层用 "server-${idx}" 作静态 key
         控制层添加 depends_on = [module.ecs]
```

## 常见陷阱

### 陷阱 1：在 resource for_each 场景用 apply-time 值做 if 过滤

```hcl
# ❌ 错误：if 条件依赖 apply-time 值，for_each key 集合 unknown
servers = [
  for s in var.servers : merge(s, {
    server_id = lookup(local.ecs_ids_map, s.ecs_key, "")
  })
  if lookup(local.ecs_ids_map, s.ecs_key, "") != ""  # ← known after apply
]
# 报错：Invalid for_each argument
```

**正确做法**：用 `ecs_would_create`（基于 var 的 plan-time 已知属性）替代 `ecs_ids_map`（apply-time 值）：

```hcl
# ✅ 正确：if 条件只依赖 plan-time 已知值
servers = [
  for s in try(each.value.servers, []) : merge(s, {
    server_id = try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")
  })
  if try(s.ecs_key, null) != null ? contains(local.ecs_would_create, s.ecs_key) : try(s.server_id, "") != ""
]
```

### 陷阱 2：忘记添加 depends_on

```hcl
# ❌ 错误：没有 depends_on，ECS 和 NLB Attachment 可能并行创建
# 即使 ecs_would_create 过滤了 create=false 的 ECS，首次创建时仍需保证 ECS 先完成
module "nlb_attachment" {
  source   = "../../atomic/nlb-server-group-server-attachment"
  for_each = var.nlb_server_groups
  # 缺少 depends_on = [module.ecs]
}
```

### 陷阱 2b：ecs_would_create 条件与 ECS 模块的 create 不一致

```hcl
# ❌ 错误：ecs_would_create 用了简化条件，与 ECS 模块实际 create 不一致
# ECS 模块的 create = try(each.value.create, each.value.image_id != "")
# 如果 ecs_would_create 只检查 create 字段，image_id 回退逻辑会被遗漏
ecs_would_create = toset([
  for k, v in var.ecs_instances : k
  if try(v.create, true)  # ← 遗漏了 image_id 回退条件
])
# 结果：image_id="" 但未显式设 create=false 的 ECS 不会被过滤，实际 ECS 不创建
```

### 陷阱 3：把 dynamic 块用于需要独立生命周期管理的资源

```hcl
# ❌ 错误：dynamic 块内的资源不能单独 import/replace/taint
# 如果需要独立管理每个 attachment，必须用 resource for_each
```

## 检查清单

- [ ] 识别原子层的后端服务器是嵌套属性还是独立资源
- [ ] 嵌套属性 → 原子层改用 `dynamic` 块，控制层可安全用 `if lookup(ecs_ids_map,...)` 过滤
- [ ] 独立资源 → 原子层用静态 key 的 `for_each`
- [ ] 独立资源场景 → 控制层用 `ecs_would_create`（plan-time 安全）做 `if` 过滤，不用 `ecs_ids_map`
- [ ] `ecs_would_create` 条件与 ECS 模块的 `create` 条件完全一致
- [ ] 所有场景 → 控制层添加 `depends_on = [module.ecs]`
- [ ] 原子层 header 注释说明设计选择（dynamic 或静态 key 的原因）

## 参考资料

