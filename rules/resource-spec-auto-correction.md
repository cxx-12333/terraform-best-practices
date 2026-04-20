# 资源规格自动对齐：防止状态漂移

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

某些云资源的参数由资源规格代码（instance_class）决定，而非用户配置。当用户设置的值与规格固定值不匹配时，会导致状态漂移，因为 API 会忽略用户设置，使用规格决定的值。

## 问题：忽略参数导致状态漂移

### 示例：阿里云 Redis 经典版集群

```
instance_class = "redis.sharding.basic.small.default"  # 固定：8分片 × 2GB
shard_count    = 2   # 用户设置 - API 忽略
capacity       = 2048 # 用户设置 - API 忽略
```

**后果：**

1. API 创建实例时使用 `shard_count=8, capacity=16384`（规格固定值）
2. State 记录 API 返回的实际值
3. 用户 tfvars 仍然是 `shard_count=2, capacity=2048`
4. 下次 plan：Code ≠ State → **触发重建！**

### 根本原因

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 经典版集群：shard_count 和 capacity 由 instance_class 决定                   │
│                                                                             │
│ redis.sharding.basic.small.default = 8分片 × 2GB = 16GB 总容量              │
│ redis.sharding.basic.medium.default = 8分片 × 4GB = 32GB 总容量             │
│                                                                             │
│ 用户参数被忽略 - 仅云原生版（.ce）支持自定义                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 解决方案：原子层自动对齐

### 模式：规格值映射

```hcl
# atomic/redis-community/main.tf

locals {
  # 检测部署类型
  is_classic = can(regex("\\.default$", var.instance_class))
  is_cloud_native = can(regex("\\.ce$", var.instance_class))
  is_cluster = can(regex("sharding|with\\.proxy", lower(var.instance_class)))
  is_classic_cluster = local.is_classic && local.is_cluster

  # 规格固定值映射
  classic_cluster_specs = {
    "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }
    "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }
    "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }
    "redis.sharding.basic.xlarge.default"   = { shard_count = 8,  capacity_mb = 131072 }
  }

  # 自动对齐：经典版集群使用规格固定值，其他使用用户值
  effective_shard_count = local.is_classic_cluster ? (
    try(local.classic_cluster_specs[var.instance_class].shard_count, var.shard_count)
  ) : (local.is_cluster ? var.shard_count : null)

  effective_capacity = local.is_classic_cluster ? (
    try(local.classic_cluster_specs[var.instance_class].capacity_mb, var.capacity)
  ) : var.capacity
}

resource "alicloud_kvstore_instance" "this" {
  shard_count = local.effective_shard_count
  capacity    = local.effective_capacity
}
```

## 决策矩阵

| 资源类型 | 参数控制 | 用户设置 | 原子层行为 |
|----------|----------|----------|------------|
| 经典版 `.default` | 规格固定 | 任意值 | **自动对齐**到规格值 |
| 云原生版 `.ce` | 用户控制 | 用户值 | 使用用户值 |
| 倚天版 `.y.ee` | 用户控制 | 用户值 | 使用用户值 |

## 何时应用此模式

1. **识别忽略参数**：检查 Provider 文档中"由 instance_class 决定"的参数
2. **创建规格映射**：建立 instance_class → 固定值的映射
3. **实现自动对齐**：使用 locals 将用户值覆盖为规格固定值
4. **记录限制**：添加注释说明行为

## 常见需要自动对齐的场景

| 云厂商 | 资源类型 | 忽略的参数 |
|--------|----------|------------|
| 阿里云 | Redis 经典版集群 | shard_count, capacity |
| 阿里云 | Redis 经典版标准版 | capacity（部分规格） |
| AWS | RDS 实例 | IOPS（部分实例类型） |
| AWS | EC2 实例 | bandwidth（部分实例类型） |

## 验证检查清单

- [ ] 识别由 instance_class/规格代码决定的参数
- [ ] 创建包含所有有效 instance_class 值的规格映射
- [ ] 在 locals 中实现自动对齐逻辑
- [ ] 添加清晰的注释说明行为
- [ ] 使用不匹配的用户值测试 - 不应导致漂移
- [ ] 记录哪些规格支持自定义值 vs 固定值

## 决策规则

> **当参数由资源规格代码（instance_class）决定时，原子层必须将用户值自动对齐为规格固定值，防止 API 忽略参数导致状态漂移。**

## 参考资料

- `resource-lifecycle.md`（生命周期块）
- `variable-atomic-defaults.md`（原子层默认值原则）
- `provider-documentation-lookup.md`（Provider 文档查找规范）
