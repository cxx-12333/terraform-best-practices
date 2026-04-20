# 子组件型资源 Import 策略

**优先级：** 中高
**分类：** 状态管理

> 对于有子组件的复杂资源（如 MongoDB 分片集群），必须使用**专用资源类型** import 才能获取完整的组件规格。

## 为什么重要

复杂资源（如 MongoDB 分片集群）由多个子组件组成。使用错误的资源类型 import 可能只获取部分配置，导致后续管理不完整或状态损坏。

## 问题背景

阿里云部分资源有两种创建方式：

| 方式 | 资源类型 | 特点 | Import 结果 |
|------|----------|------|-------------|
| 简化模式 | `alicloud_xxx_instance` | 使用逻辑规格 | ❌ 只有拓扑，无组件规格 |
| 精细模式 | `alicloud_xxx_sharding_instance` | 逐组件配置规格 | ✅ 有完整组件规格 |

## 案例：MongoDB 分片集群

### 两种资源类型对比

```hcl
# 方式1：alicloud_mongodb_instance（简化模式）
# 规格代码：dds.mongo.logic（分片集群逻辑标识）
resource "alicloud_mongodb_instance" "this" {
  db_instance_class = "dds.mongo.logic"  # 逻辑规格，无组件详情
  db_instance_storage = 0                 # 存储在 shard 层
}

# 方式2：alicloud_mongodb_sharding_instance（精细模式）
# 规格代码：逐组件配置
resource "alicloud_mongodb_sharding_instance" "this" {
  mongo_list {
    node_class = "mdb.shard.2x.xlarge.d"  # Mongos 规格
  }
  shard_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # Shard 规格
    node_storage = 20                        # 存储大小
  }
  config_server_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # ConfigServer 规格
    node_storage = 20
  }
}
```

### Import 结果对比

**使用 `alicloud_mongodb_instance` import**：

```hcl
# ❌ 缺失组件规格！
resource "alicloud_mongodb_instance" "sharding_simple" {
  db_instance_class  = "dds.mongo.logic"  # 只有逻辑标识
  db_instance_storage = 0
  
  zone_infos = [  # 只有拓扑信息，无规格
    { node_type = "mongos", zone_id = "cn-hangzhou-j" },
    { node_type = "shard", zone_id = "cn-hangzhou-j" },
    { node_type = "configServer", zone_id = "cn-hangzhou-j" },
  ]
  
  # ❌ 缺失：mongo_list, shard_list, config_server_list
}
```

**使用 `alicloud_mongodb_sharding_instance` import**：

```hcl
# ✅ 有完整组件规格！
resource "alicloud_mongodb_sharding_instance" "sharding_fine" {
  engine_version = "8.0"
  
  mongo_list {
    node_class     = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    connect_string = "s-bp17eabc54c31a14.mongodb.rds.aliyuncs.com"
    port           = 3717
  }
  
  shard_list {
    node_class        = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    node_storage      = 20                        # ✅ 有存储！
    readonly_replicas = 0
  }
  
  config_server_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    node_storage = 20
  }
}
```

## Provider 源码分析

### API 调用差异

| 资源类型 | API 调用 | 返回数据 |
|----------|----------|----------|
| `alicloud_mongodb_instance` | `DescribeDBInstances` | 只有 `zone_infos`（拓扑） |
| `alicloud_mongodb_sharding_instance` | `DescribeMongoDBShardingInstance` | `MongosList` + `ShardList` + `ConfigserverList` |

### 源码位置

```
terraform-provider-alicloud/alicloud/resource_alicloud_mongodb_sharding_instance.go
```

关键代码（第 775-877 行）：

```go
// alicloud_mongodb_sharding_instance 的 Read 函数
if MongosListMap, ok := object["MongosList"].(map[string]interface{}); ok {
    if nodeClass, ok := MongosListArg["NodeClass"]; ok {
        MongosListItemMap["node_class"] = nodeClass  // ✅ 获取规格
    }
}

if shardListMap, ok := object["ShardList"].(map[string]interface{}); ok {
    if nodeClass, ok := shardListArg["NodeClass"]; ok {
        shardListItemMap["node_class"] = nodeClass    // ✅ 获取规格
    }
    if nodeStorage, ok := shardListArg["NodeStorage"]; ok {
        shardListItemMap["node_storage"] = formatInt(nodeStorage)  // ✅ 获取存储
    }
}
```

## 判断标准

### 如何识别子组件型资源

1. **查阅官方文档**：资源是否有多个组件（如 Mongos/Shard/ConfigServer）
2. **查阅 Provider 源码**：是否有专用资源类型（如 `xxx_sharding_instance`）
3. **查看 Schema 定义**：是否有 `xxx_list` 类型的参数（如 `mongo_list`）

### 常见子组件型资源

| 资源类型 | 简化模式 | 精细模式（推荐） |
|----------|----------|------------------|
| MongoDB 分片集群 | `alicloud_mongodb_instance` | `alicloud_mongodb_sharding_instance` |
| Redis 集群版 | `alicloud_kvstore_instance` | `alicloud_kvstore_instance`（通过 `node_type` 区分） |

## 最佳实践流程

### 步骤 1：识别资源类型

```bash
# 查看实例 ID 格式
dds-bp1698f5c4823ea4  # MongoDB 实例 ID

# 控制台确认架构类型
# 分片集群 → 使用 alicloud_mongodb_sharding_instance
# 副本集 → 使用 alicloud_mongodb_instance
```

### 步骤 2：使用专用资源类型 import

```bash
# ✅ 正确：使用专用资源类型
terraform import alicloud_mongodb_sharding_instance.this dds-bp1698f5c4823ea4

# ❌ 错误：使用通用资源类型
terraform import alicloud_mongodb_instance.this dds-bp1698f5c4823ea4
```

### 步骤 3：验证 import 结果

```bash
# 查看 state，确认组件规格是否完整
terraform state show alicloud_mongodb_sharding_instance.this

# 检查是否有 mongo_list / shard_list / config_server_list
# 如果有，说明 import 成功获取了组件规格
```

### 步骤 4：复制规格到声明层

```hcl
# 从 state 复制组件规格到声明层
mongodb_sharding_instances = {
  "01" = {
    mongos_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d" }  # 从 state 复制
      "02" = { node_class = "mdb.shard.2x.xlarge.d" }
    }
    shards_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
      "02" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
    }
    config_server_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
    }
  }
}
```

## 检查清单

Import 子组件型资源时：

- [ ] 已确认资源架构类型（分片集群 vs 副本集）
- [ ] 已选择专用资源类型（如 `xxx_sharding_instance`）
- [ ] 已验证 import 结果包含组件规格
- [ ] 已复制组件规格到声明层配置
- [ ] 已设置 `create = false` 避免重复创建

## 相关规则

- `state-analysis-vs-rebuild` - State 分析与配置重建规范
- `resource-sharding-instance-type` - 分片集群资源类型区分

## 参考资料

- `state-import.md`（资源导入规范）
- `state-analysis-vs-rebuild.md`（State 分析与重建）
- [Terraform Import](https://developer.hashicorp.com/terraform/cli/import)
