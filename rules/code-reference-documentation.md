# 原子层代码参考规范

**优先级：** 中
**分类：** 三层架构

> 原子层代码中应在关键配置处添加官方文档参考链接，便于查阅和理解。

## 为什么重要

在原子层代码中添加官方文档参考链接有以下好处：

1. **可追溯性**：配置参数有据可查，方便后续维护
2. **学习效率**：新人阅读代码时能快速了解参数含义
3. **准确性验证**：遇到疑问可直接查阅官方文档确认
4. **知识沉淀**：将分散的官方文档整合到代码中

## 推荐格式

### 1. 文件头部参考

```hcl
################################################################################
# MongoDB 云数据库
# 设计模式：create 开关 + locals 前置 + try 安全输出
# 实例类型：副本集（Replica Set）/ 分片集群（Sharded Cluster）
# 官方文档：https://help.aliyun.com/zh/mongodb/product-overview/instance-types
# Provider 文档：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/mongodb_instance
################################################################################
```

### 2. 关键配置块参考

```hcl
# ——— Mongos 节点规格 ———
# 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
# 必填：node_class（Mongos 规格）
# 可选：无（Mongos 无存储配置）
mongo_list {
  node_class = var.mongos_node_class
}

# ——— Shard 节点规格 ———
# 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
# 必填：node_class, node_storage
# 可选：readonly_replicas（0-5，默认 0）
shard_list {
  node_class        = var.shard_node_class
  node_storage      = var.shard_node_storage
  readonly_replicas = var.shard_readonly_replicas
}
```

### 3. 规格映射表参考

```hcl
# 经典版集群规格固定值映射（防止状态漂移）
# 格式：shard_count = 分片数, capacity_mb = 总容量(MB)
# 参考文档：https://help.aliyun.com/document_detail/26350.html
classic_cluster_specs = {
  "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }  # small = 8分片×2GB
  "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }  # medium = 8分片×4GB
  "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }  # large = 8分片×8GB
}
```

### 4. 特殊逻辑说明参考

```hcl
# ——— 架构类型判断 ———
# 判断依据：replication_factor = 0 表示分片集群
# 参考：https://help.aliyun.com/zh/mongodb/developer-reference/api-dds-2015-12-01-createdbinstance
is_sharding = var.replication_factor == 0

# ——— ForceNew 参数处理 ———
# storage_engine 是 ForceNew 参数，修改会触发重建
# 参考：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/mongodb_instance#storage_engine
storage_engine = var.storage_engine
```

## 最佳实践示例

### Redis 原子层示例

```hcl
locals {
  # ——— 架构类型判断 ———
  # 判断依据：deploy_type 后缀决定架构类型
  # 参考文档：https://help.aliyun.com/document_detail/26350.html
  is_classic    = var.deploy_type == "classic"
  is_cloud_native = var.deploy_type == "cloud_native"
  is_yitian     = var.deploy_type == "yitian"
  is_cluster    = var.instance_class != "" && (
    can(regex("sharding", var.instance_class)) ||
    can(regex("shard\\.with\\.proxy", var.instance_class))
  )

  # 经典版集群规格固定值映射（防止状态漂移）
  # 格式：shard_count = 分片数, capacity_mb = 总容量(MB)
  # 参考文档：https://help.aliyun.com/document_detail/26350.html
  classic_cluster_specs = {
    "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }
    "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }
    "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }
    "redis.sharding.basic.xlarge.default"   = { shard_count = 8,  capacity_mb = 131072 }
    "redis.sharding.2xlarge.default"        = { shard_count = 8,  capacity_mb = 262144 }
  }
}
```

### MongoDB 分片集群原子层示例

```hcl
resource "alicloud_mongodb_sharding_instance" "this" {
  count = local.create ? 1 : 0

  # ——— 基础配置 ———
  engine_version = var.engine_version
  
  # ——— Mongos 节点规格 ———
  # 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  # Mongos 规格：2核8GB / 2核16GB / 4核8GB / 4核16GB 等
  # 数量范围：2-32 个
  dynamic "mongo_list" {
    for_each = var.mongos_list
    content {
      node_class = mongo_list.value.node_class
    }
  }

  # ——— Shard 节点规格 ———
  # 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  # Shard 规格：2核8GB / 2核16GB / 4核8GB / 4核16GB 等
  # 存储范围：20-16000 GB
  # 数量范围：2-32 个
  dynamic "shard_list" {
    for_each = var.shards_list
    content {
      node_class        = shard_list.value.node_class
      node_storage      = shard_list.value.node_storage
      readonly_replicas = try(shard_list.value.readonly_replicas, 0)
    }
  }

  # ——— ConfigServer 规格（可选）——
  # 默认值：dds.cs.mid（1核2GB）
  # 参考文档：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  dynamic "config_server_list" {
    for_each = var.config_server_list
    content {
      node_class   = config_server_list.value.node_class
      node_storage = config_server_list.value.node_storage
    }
  }
}
```

## 常用参考链接

### 阿里云产品文档

| 产品 | 文档地址 |
|------|---------|
| MongoDB | https://help.aliyun.com/zh/mongodb/ |
| Redis | https://help.aliyun.com/zh/redis/ |
| PolarDB | https://help.aliyun.com/zh/polardb/ |
| RDS | https://help.aliyun.com/zh/rds/ |
| Kafka | https://help.aliyun.com/zh/alikafka/ |
| Elasticsearch | https://help.aliyun.com/zh/elasticsearch/ |

### 规格表参考

| 产品 | 规格表地址 |
|------|-----------|
| MongoDB 分片集群 | https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types |
| MongoDB 副本集 | https://help.aliyun.com/zh/mongodb/product-overview/replica-set-instance-types |
| Redis 社区版 | https://help.aliyun.com/document_detail/26350.html |
| Redis Tair | https://help.aliyun.com/zh/redis/product-overview/tair-instance-types |
| PolarDB MySQL | https://help.aliyun.com/zh/polardb/polardb-for-mysql/user-guide/primary-mp-instance-types |

### Terraform Provider 文档

| Provider | 文档地址 |
|---------|---------|
| alicloud | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs |

## 相关规则

- `comment-standardization` - 参数差异化注释规范
- `state-analysis-vs-rebuild` - State 分析与配置重建规范

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [Terraform Registry](https://registry.terraform.io/)
