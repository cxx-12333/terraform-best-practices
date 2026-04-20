# 同类资源批次依赖串行化（Batch Dependency Serialization）

**优先级：** 关键
**分类：** 资源组织

## 为什么重要

当云 API 对同一父资源的子操作加资源锁（如 ALB 同一实例的 ACL attachment 不允许并发），而 Terraform 的 `count`/`for_each` **所有实例共享一个依赖图节点**，无法创建实例间依赖。这会导致并发操作触发云 API 错误（如 `LockFailed`），且无法通过 Terraform 层面的单资源块解决。

**核心矛盾**：
- 云 API：同一父资源的子操作必须串行
- Terraform：`count`/`for_each` 实例间无法建立依赖（`this[count.index-1]` 自引用 → Cycle）
- Provider：create 路径可能不重试锁冲突错误

## 错误方案：coalesce 自引用（Cycle）

```hcl
# ❌ 错误：同一 resource block 内实例间引用 → Terraform 检测为 Cycle
resource "alicloud_alb_listener_acl_attachment" "this" {
  count = length(local.attachments)

  listener_id = coalesce(
    lookup(local.listener_ids_map, local.attachments[count.index].listener_key, null),
    # ↓ 自引用：alicloud_alb_listener_acl_attachment.this 引用自身 → Cycle Error
    count.index > 0 ? alicloud_alb_listener_acl_attachment.this[count.index - 1].listener_id : null,
  )
}
```

**错误信息**：`Error: Cycle: module.xxx.resource_name.this[1], module.xxx.resource_name.this[0]`

**根因**：Terraform 将 `count`/`for_each` 资源视为单个依赖图节点，所有实例属于同一节点，节点内无法建立顺序依赖。

**依据**：HashiCorp Issue [#10802](https://github.com/hashicorp/terraform/issues/10802) — 官方确认的唯一解决方案是拆分为多个独立 resource block。

## 正确方案：Batch 分组 + 多 resource block

**核心思路**：按父资源分组，同一父资源内的操作按序分配到不同 batch（0/1/2），每个 batch 用独立 resource block + `for_each`，`batch[n]` 通过 `depends_on` 依赖 `batch[n-1]`。

**效果**：不同父资源的操作并发（性能无损），同一父资源内串行（避免锁冲突）。

### Step 1：展平 + 分组 locals

```hcl
locals {
  # 展平所有操作项，带 load_balancer_key 用于父资源分组
  acl_flat = flatten([
    for lk, lv in var.listeners : [
      for ak in try(lv.acl_keys, []) : {
        listener_key      = lk
        load_balancer_key = lv.load_balancer_key  # 父资源标识，用于分组
        acl_key           = ak
        acl_type          = try(lv.acl_type, "White")
        key               = "${lk}-${ak}"          # for_each 的稳定 key
      }
      if try(lv.acl_keys, null) != null
    ]
    if try(lv.acl_keys, null) != null
      && lookup(local.listener_would_create, lk, false)
  ])

  # 为同一父资源的操作分配 batch 序号
  # batch = 同一 load_balancer_key 内、展平顺序在此项之前的操作数量
  acl_batched = [
    for i, v in local.acl_flat : merge(v, {
      batch = length([
        for j, w in local.acl_flat : w
        if w.load_balancer_key == v.load_balancer_key && j < i
      ])
    })
  ]

  # 按 batch 分组为 map（for_each 需要 map）
  acl_batch0 = { for v in local.acl_batched : v.key => v if v.batch == 0 }
  acl_batch1 = { for v in local.acl_batched : v.key => v if v.batch == 1 }
  acl_batch2 = { for v in local.acl_batched : v.key => v if v.batch == 2 }
}
```

### Step 2：多个独立 resource block

```hcl
# Batch 0：首批，所有父资源的第一个操作并发执行
resource "alicloud_alb_listener_acl_attachment" "batch0" {
  for_each = local.acl_batch0

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [module.alb_listener]
}

# Batch 1：同一父资源的第 2 个操作，等 batch0 全部完成后执行
resource "alicloud_alb_listener_acl_attachment" "batch1" {
  for_each = local.acl_batch1

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [alicloud_alb_listener_acl_attachment.batch0]
}

# Batch 2：同一父资源的第 3 个操作，等 batch1 全部完成后执行
resource "alicloud_alb_listener_acl_attachment" "batch2" {
  for_each = local.acl_batch2

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [alicloud_alb_listener_acl_attachment.batch1]
}
```

### 执行时序图

```
                     时间轴 →
ALB-A:  [batch0: attach-1] ──→ [batch1: attach-2] ──→ [batch2: attach-3]
ALB-B:  [batch0: attach-1] ──→ [batch1: attach-2]
             ↓ 并发                  ↓ 等batch0完成          ↓ 等batch1完成
```

## batch 序号计算原理

```hcl
# 示例：2个 ALB，每个有 2-3 个 ACL attachment
# 展平顺序：A-1, A-2, B-1, B-2
#
# A-1: load_balancer_key=A, 之前同 ALB 的数量=0 → batch=0
# A-2: load_balancer_key=A, 之前同 ALB 的数量=1 → batch=1
# B-1: load_balancer_key=B, 之前同 ALB 的数量=0 → batch=0
# B-2: load_balancer_key=B, 之前同 ALB 的数量=1 → batch=1
#
# 结果：
# batch0 = { A-1, B-1 }  → 并发
# batch1 = { A-2, B-2 }  → 等 batch0 后并发
```

## 方案决策矩阵

| 方案 | 可行性 | 性能 | 复杂度 | 问题 |
|------|--------|------|--------|------|
| count + coalesce 自引用 | ❌ Cycle | - | - | Terraform 禁止同资源实例间引用 |
| 限制 -parallelism=1 | ✅ | ❌ 全局串行 | 低 | 所有资源串行，性能严重退化 |
| 批次分组（本方案） | ✅ | ✅ 按需串行 | 中 | batch 数量固定（通常 3 个足够） |
| 修改 Provider 重试逻辑 | ✅ | ✅ | 高 | 需维护 fork 版本，不可控 |
| 分多次 apply | ✅ | ❌ | 高 | 运维成本高，不适合自动化 |

## 适用场景判断

满足以下所有条件时使用本方案：
1. **云 API 资源锁**：同一父资源的子操作不允许并发（如 LockFailed、Conflict 等错误）
2. **Provider 不重试**：create 路径的重试列表不包含锁冲突错误码
3. **操作数量可控**：同一父资源的最大子操作数 ≤ batch 数量（本方案固定 3）

## 实战案例：ALB ACL Attachment LockFailed

**错误场景**：阿里云 ALB 同一实例的多个 Listener 并发绑定 ACL 时触发 `LockFailed`（StatusCode: 400）。

**Provider 源码依据**：
- create 路径（`resource_alicloud_alb_listener_acl_attachment.go:73`）：重试列表 `["ResourceInConfiguring.Listener", "IncorrectStatus.Listener", "Conflict.Acl"]`，**不含** `LockFailed`
- delete 路径（同文件 L158）：重试列表**包含** `LockFailed`

**改造范围**：
1. 原子层 `alb-listener`：移除 `alicloud_alb_listener_acl_attachment` 资源，`acl_config` 改为透传输出
2. 控制层 `web-cluster`：新增 locals 批次分组 + 3 个独立 resource block

## 注意事项

- batch 数量（通常 3 个）基于 `alicloud_alb_listener_acl_attachment` 文档限制："You can associate at most three ACLs with a listener"
- `for_each` 使用稳定 key（`${listener_key}-${acl_key}`），避免 count 的索引漂移问题
- `depends_on` 跨 resource block 建立串行链，是 Terraform 依赖图中合法的操作

## 相关规则

- `resource-count-vs-foreach` - for_each 优先于 count
- `module-null-safe-dependency` - 父资源未创建时子资源安全降级
- `module-dependency-prefilter` - 模块依赖预过滤
- `layer-id-mapping` - 通过 map 变量实现基于 key 的资源引用

## 参考资料

- `resource-count-vs-foreach.md`（for_each vs count）
- `module-dependency-prefilter.md`（依赖预过滤）
- `control-nested-resource-flatten.md`（嵌套资源展平）
