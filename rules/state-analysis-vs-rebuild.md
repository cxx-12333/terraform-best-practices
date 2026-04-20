# State 分析与配置重建规范

**优先级：** 中高
**分类：** 状态管理

> Import state 仅用于分析现有资源配置，重写/重建配置必须依据官方文档和 Provider 源码。

## 为什么重要

在实际项目中，我们经常需要：
1. **Import 现有资源** → 获取 state 分析实际配置
2. **重写声明层配置** → 实现可重建性

但这两个步骤的信息来源**不能混为一谈**：

| 阶段 | 信息来源 | 用途 |
|------|---------|------|
| 分析阶段 | State（import 结果） | 了解现有资源的实际配置 |
| 重建阶段 | 官方文档 + Provider 源码 | 确保配置准确、完整、可重建 |

## State 的局限性

Import 得到的 state 存在以下问题：

### 1. 参数不完整

```hcl
# State 中可能缺失的参数示例
# MongoDB 分片集群使用错误的资源类型 import：
db_instance_class = "dds.mongo.logic"  # 只有逻辑标识，无组件规格
# 缺失：mongo_list, shard_list, config_server_list
```

### 2. 类型信息丢失

```hcl
# State 中某些字段可能为空或默认值
encrypted = false          # 可能被 Provider 填充默认值
storage_type = ""          # 可能未在 state 中体现
```

### 3. 资源类型错误

```hcl
# 分片集群可能被错误 import 为副本集资源
# State 显示：alicloud_mongodb_instance
# 实际应该：alicloud_mongodb_sharding_instance
```

## 正确的做法

### 步骤 1：State 分析（了解现状）

```bash
# Import 现有资源
terraform import alicloud_mongodb_instance.this dds-bp1bbce1bc9d8df4

# 查看 state
terraform state show alicloud_mongodb_instance.this
```

从 state 获取：
- 实例 ID、规格代码、版本号
- 网络配置（VPC、VSwitch、Zone）
- 可用区分布
- 组件拓扑（从 zone_infos 推断）

### 步骤 2：查阅官方文档（确认参数）

```
# 官方文档来源
1. 产品文档：https://help.aliyun.com/zh/mongodb/
2. API 文档：https://help.aliyun.com/zh/mongodb/developer-reference/
3. 规格表：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
```

### 步骤 3：查阅 Provider 源码（确认 schema）

```
# Provider 源码位置
terraform-provider-alicloud/alicloud/resource_alicloud_mongodb_sharding_instance.go

# 确认：
- 必填参数（Required）
- 可选参数（Optional）
- ForceNew 参数（修改会重建）
- 参数验证规则（ValidateFunc）
- Computed 参数（由 API 返回）
```

### 步骤 4：重写配置（可重建性）

```hcl
# ✅ 基于官方文档 + Provider 源码重写的完整配置
resource "alicloud_mongodb_sharding_instance" "this" {
  # 从 state 获取：engine_version = "8.0"
  engine_version = "8.0"
  
  # 从官方文档确认：mongo_list 是 Required
  # 从 Provider 源码确认：node_class 是 Required
  mongo_list {
    node_class = "mdb.shard.4x.large.d"  # 从规格表确认
  }
  mongo_list {
    node_class = "mdb.shard.4x.large.d"
  }
  
  # 从官方文档确认：shard_list 是 Required
  shard_list {
    node_class        = "mdb.shard.4x.large.d"
    node_storage      = 470    # 从 state 分析或控制台确认
    readonly_replicas = 1
  }
  
  # 从 Provider 源码确认：config_server_list 有默认值
  # 可选配置，使用默认值或显式指定
}
```

## 实践案例

### 案例：MongoDB 分片集群配置重建

**State 分析发现的问题**：
```
db_instance_class = "dds.mongo.logic"  # 逻辑标识，非规格
replication_factor = 0                  # 分片集群特征
db_instance_storage = 0                 # 存储在 shard 层
zone_infos 有多个 mongos/shard 节点    # 分片集群拓扑
```

**查阅官方文档确认**：
- 分片集群需要使用 `alicloud_mongodb_sharding_instance`
- `mongo_list` 和 `shard_list` 是必填参数
- 每个组件需要指定 `node_class` 和 `node_storage`

**查阅 Provider 源码确认**：
- `mongo_list` 的 `node_class` 是 Required
- `shard_list` 的 `node_storage` 是 Required
- `config_server_list` 是 Optional，有默认值

**最终配置**：
```hcl
# 使用正确的资源类型 + 完整的组件规格配置
resource "alicloud_mongodb_sharding_instance" "this" {
  # ... 完整的可重建配置
}
```

## 检查清单

重写配置前，确认：

- [ ] 已从 state 分析现有配置
- [ ] 已查阅官方产品文档确认参数含义
- [ ] 已查阅 Provider 源码确认 schema 定义
- [ ] 已查阅 API 文档确认参数约束
- [ ] 已查阅规格表确认规格代码
- [ ] 已区分 Required / Optional / Computed 参数
- [ ] 已识别 ForceNew 参数并设置合理默认值

## 参数逆向验证法（参数不确定时）

当 Terraform 报错且无法从文档确定正确参数时，采用逆向验证。

### 适用场景

- `COMMODITY.INVALID_COMPONENT` 等参数校验错误
- 地域/可用区规格差异（文档未明确）
- 参数互斥规则不清楚
- Provider 行为与文档不一致

### 验证步骤

```bash
# 1. 控制台创建资源（使用已知可用的参数组合）
#    选择目标地域/可用区、目标规格，完成创建后记录资源 ID

# 2. 编写最小配置（仅必填参数）
resource "alicloud_nas_file_system" "test" {
  protocol_type     = "cpfs"
  storage_type      = "advance_200"  # 待验证
  file_system_type  = "cpfs"
  capacity          = 3600
  zone_id           = "cn-hangzhou-j"
  vswitch_id        = "vsw-xxx"
}

# 3. Import 实际资源
terraform import alicloud_nas_file_system.test cpfs-xxx

# 4. State 分析查看实际参数
terraform state show alicloud_nas_file_system.test

# 输出示例：
# storage_type           = "advance_100"   ← 实际值！文档写的是 advance_200
# redundancy_type        = null           ← API 默认不传！显式传 LRS 会报错
# redundancy_vswitch_ids = []             ← 空列表，非 null

# 5. 修正配置并重新 terraform plan 验证
```

### 实践案例：CPFS 参数校验错误

**问题**：创建 CPFS 报错 `COMMODITY.INVALID_COMPONENT`

**逆向验证发现**：
- 杭州j区实际只支持 `advance_100`，文档写的 `advance_200` 不适用
- CPFS 类型不应传递 `redundancy_type`，API 默认 LRS

**修复**：
```hcl
# 正确配置
resource "alicloud_nas_file_system" "this" {
  file_system_type = "cpfs"
  storage_type     = "advance_100"  # 验证后修正
  # redundancy_type 不传递（API 默认 LRS）
}
```

### 注意事项

- 逆向验证仅用于**参数校验错误诊断**，不用于生产资源配置
- 验证后需删除测试资源，使用正式 Terraform 配置重建
- 测试资源配置应放在 `modules/test/` 目录，与生产隔离

## 相关规则

- `resource-sharding-instance-type` - 分片集群资源类型区分
- `variable-forcenew-defaults` - ForceNew 参数空值默认规则
- `test-drift-detection` - 部署后二次 Plan 验证

## 参考资料

- `state-import.md`（资源导入规范）
- `import-component-resource.md`（子组件 Import 策略）
- `provider-documentation-lookup.md`（Provider 文档查找规范）
