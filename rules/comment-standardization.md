# Terraform 注释标准化规范

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

注释的目标：让读者**快速理解参数填写规则和差异化限制**，而非展示所有细节。标准化的注释格式能减少沟通成本，提高团队协作效率。

## 核心原则

注释的目标：让读者**快速理解参数填写规则和差异化限制**，而非展示所有细节。

### 四大原则

1. **差异化优先** - 说明与其他配置/版本的关键差异
2. **简洁明了** - 一句话说清，避免冗长解释
3. **填写指引** - 告诉用户"怎么填"而非"这是什么"
4. **防漂提示** - 标注自动对齐机制，消除用户顾虑

---

## 注释分类与模板

### 1. 差异化注释（最重要）

用于说明参数在不同场景下的关键差异：

```hcl
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
```

**适用场景：**
- 经典版 vs 云原生版参数差异
- 不同产品版本的限制
- API 行为与用户预期不一致的情况

### 2. 行内简洁注释

参数后紧跟简短说明：

```hcl
# 正确示例
shard_count = 8       # 实际值由 instance_class 决定，原子层自动对齐
capacity    = 16384   # 实际值由 instance_class 决定，原子层自动对齐
engine_version = "5.0"  # 经典版仅支持 5.0

# 错误示例（过于冗长）
shard_count = 8  # 这是分片数量，经典版集群的分片数由规格代码决定，
                 # 用户设置的值会被API忽略，所以填写任意值都可以
```

**格式要求：**
- 一行内完成
- 说明关键限制或差异
- 无表情符号

### 3. 区块注释模板

用于资源/实例级别的差异化说明：

```hcl
#############################################################################
# 实例03：经典版集群版（切片架构）- 传统架构，仅支持 Redis 5.0
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
#############################################################################
```

### 4. 参数组注释模板

用于说明一组参数的联动关系：

```hcl
# ——— 架构配置 ———
# 经典版：shard_count/capacity 由规格决定，原子层自动对齐
deploy_type          = "classic"
shard_count          = 8       # 实际值由 instance_class 决定
capacity             = 16384   # 实际值由 instance_class 决定
read_only_count      = 0       # 经典版不支持读写分离
```

---

## 禁止的注释风格

### 禁止 1：过度校验注释

```hcl
# 错误
backup_period = ["Monday"]  # ✅ 已生效
enable_backup_log = 1       # ⚠️ State 实际值=0，可能需控制台确认

# 正确
backup_period = ["Monday"]
enable_backup_log = 1  # 经典版：API 可能忽略，需控制台确认
```

### 禁止 2：表情符号滥用

```hcl
# 错误
shard_count = 8  # ⚠️ 重要！API 会忽略这个值 ❗❗

# 正确
shard_count = 8  # 实际值由 instance_class 决定，原子层自动对齐
```

### 禁止 3：冗余解释

```hcl
# 错误
# 分片数量（shard_count）是指 Redis 集群中的数据分片个数，
# 每个分片包含一个主节点和若干从节点，经典版的分片数由规格代码决定...

# 正确
shard_count = 8  # 经典版：由规格决定，原子层自动对齐
```

---

## 产品版本差异化注释速查

### Redis 经典版 vs 云原生版

| 参数 | 经典版 | 云原生版 | 注释建议 |
|------|--------|----------|----------|
| `shard_count` | 由规格决定 | 用户自定义 | `# 经典版：由规格决定，原子层自动对齐` |
| `capacity` | 由规格决定 | 用户自定义 | `# 经典版：由规格决定，原子层自动对齐` |
| `engine_version` | 仅 5.0 | 5.0/6.0/7.0 | `# 经典版仅支持 5.0` |
| `read_only_count` | 有限制 | 无限制 | `# 经典版不支持读写分离` |

### PolarDB 标准版 vs 企业版

| 参数 | 标准版 | 企业版 | 注释建议 |
|------|--------|--------|----------|
| `db_node_count` | 1-16 | 1-15 | `# 标准版：1-16` |
| `creation_category` | SENormal | Normal | `# 标准版：与 .c 后缀规格一致` |
| `db_node_class` | .c 后缀 | 无后缀 | `# .c 后缀 = 标准版` |

---

## 自动对齐声明模板

当原子层实现了参数自动对齐时，使用以下标准注释：

```hcl
# 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
```

或行内简化：

```hcl
shard_count = 8  # 原子层自动对齐
```

---

## 注释更新规则

1. **发现新差异时** - 立即添加差异化注释
2. **实现自动对齐后** - 更新注释说明"原子层自动对齐"
3. **参数行为变化时** - 同步更新注释，删除过时说明
4. **避免重复** - 同一差异只在一处详细说明，其他位置引用

---

## 示例：完整实例注释

```hcl
#############################################################################
# 实例03：经典版集群版（切片架构）- 传统架构，仅支持 Redis 5.0
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
#############################################################################
"03" = {
  # ——— 必填参数 ———
  instance_class = "redis.sharding.basic.small.default"  # 经典版集群版（固定 8 分片）
  vswitch_key    = "j-data"
  create         = true

  # ——— 架构配置 ———
  # 经典版：shard_count/capacity 由规格决定，原子层自动对齐
  deploy_type          = "classic"
  shard_count          = 8       # 实际值由 instance_class 决定，原子层自动对齐
  capacity             = 16384   # 实际值由 instance_class 决定，原子层自动对齐
  read_only_count      = 0       # 经典版不支持读写分离
  slave_read_only_count = 0

  # ——— 版本与可用区 ———
  engine_version = "5.0"  # 经典版仅支持 5.0
  zone_id        = "cn-hangzhou-j"
  secondary_zone_id = ""

  # ——— 认证配置 ———
  vpc_auth_mode = "Close"  # Close=密码认证
  password      = "Redis@Classic#123"

  # ——— 运维配置 ———
  is_auto_upgrade_open = "Enable"  # 开启自动升级小版本

  # ——— 备份配置 ———
  backup_period = ["Monday", "Wednesday", "Friday"]
  backup_time   = "02:00Z-03:00Z"
  enable_backup_log = 1  # 经典版：API 可能忽略，需控制台确认

  # ——— SSL 配置 ———
  ssl_enable = "Enable"  # 经典版支持 SSL
}
```

## 参考资料

- `variable-descriptions.md`（变量描述规范）
- `layer-comment-standard.md`（三层注释规范）
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)
