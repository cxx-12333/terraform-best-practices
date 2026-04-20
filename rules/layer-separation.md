# 三层架构分离：原子层 / 控制层 / 声明层

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

清晰的三层分离是可维护基础设施即代码的基础。每层有明确的职责边界。职责混乱会导致隐藏依赖、代码难以维护、环境无法复现。

## 目录结构

```
modules/                          # 按云厂商组织（如 aws/、azure/、alicloud/）
├── atomic/          # 原子层：单资源封装
│   ├── vpc/
│   ├── vswitch/
│   ├── ecs/
│   ├── polardb/
│   ├── redis-community/
│   └── ... (30+ 模块)
├── control/         # 控制层：业务编排引擎
│   ├── network-topology/
│   ├── security/
│   ├── ack-cluster/
│   ├── database-cluster/
│   ├── middleware-cluster/
│   └── web-cluster/
└── declarative/     # 声明层：环境配置真相
    ├── simple/
    └── test33/
        ├── 00-shared/
        ├── 01-network/
        ├── 02-security/
        ├── 03-ack/
        ├── 04-database/
        ├── 05-middleware/
        └── 06-web/
```

## 层级职责

### 层级定义矩阵

| 层级 | 目录 | 职责 | 禁止事项 |
|------|------|------|----------|
| **原子层** | `atomic/<resource>/` | 单资源封装。暴露完整参数接口。一个模块 = 一种资源类型 | 禁止跨模块引用；禁止硬编码业务命名；禁止业务默认值 |
| **控制层** | `control/<scene>/` | 纯编排引擎。使用 for_each + locals 动态创建资源和 ID 映射。接收外部传入的 ID | 禁止硬编码业务值；禁止环境特定值；禁止创建共享资源（安全组/ACL） |
| **声明层** | `declarative/<env>/` | 环境配置的唯一真相来源。通过 tfvars 中的 map(object) 控制资源拓扑 | 禁止明文存储生产密钥；禁止直接调用原子模块 |

## 关键设计模式

### 声明层控制拓扑

```hcl
# declarative/test33/04-database/terraform.tfvars
# 声明你想要的 - 这一层控制一切

polardb_clusters = {
  "01" = {
    db_node_class = "polar.mysql.g2.xlarge"
    db_version    = "8.0"
    pay_type      = "PostPaid"
    vswitch_key   = "j-data"
    create        = true
  }
}

redis_instances = {
  "01" = {
    instance_class = "redis.shard.with.proxy.small.ce"
    shard_count    = 4
    capacity       = 4096
    engine_version = "7.0"
    vswitch_key    = "j-ecs"
    create         = true
  }
}
```

### 控制层动态编排

```hcl
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters  # 声明层控制数量

  # 必填输入
  vswitch_id    = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]
  db_node_class = each.value.db_node_class

  # 可选参数带默认值
  db_version = try(each.value.db_version, "8.0")
  pay_type   = try(each.value.pay_type, "PostPaid")

  # 自动命名
  cluster_description = coalesce(try(each.value.cluster_name, ""), "polar-${var.env}-${each.key}")

  # 安全配置
  security_ips       = try(each.value.security_ips, ["127.0.0.1"])  # API 默认值
  security_group_ids = can(each.value.security_group_key) ? 
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] : []

  tags = merge(local.common_tags, try(each.value.tags, {}))
}
```

### 原子层纯封装

```hcl
# atomic/polardb/main.tf

locals {
  create = var.create
}

resource "alicloud_polardb_cluster" "this" {
  count = local.create ? 1 : 0

  # 完整参数接口 - 必填字段无默认值
  db_node_class       = var.db_node_class
  vswitch_id          = var.vswitch_id
  zone_id             = var.zone_id
  db_version          = var.db_version

  # 可选字段 - 空默认值 = 让 API 决定
  pay_type            = var.pay_type != "" ? var.pay_type : null
  period              = var.period > 0 ? var.period : null
  cluster_description = var.cluster_description

  tags = merge({ "Name" = var.cluster_description }, var.tags,)
}
```

## 决策问题

**"这个配置属于哪一层？"**

| 问题 | 答案 | 所属层级 |
|------|------|----------|
| "这是什么资源，有哪些参数？" | 参数接口定义 | **原子层** |
| "这些资源如何组合，依赖顺序是什么？" | 编排逻辑 | **控制层** |
| "这个环境使用什么具体值？" | 具体赋值 | **声明层** |

**"新增资源时要改哪里？"**

| 场景 | 位置 |
|------|------|
| 新增 PolarDB 集群 | 编辑 04-database/terraform.tfvars 的 polardb_clusters map |
| 新增 Redis 实例 | 编辑 04-database/terraform.tfvars 的 redis_instances map |
| 新增 ECS 实例 | 编辑 06-web/terraform.tfvars 的 ecs_instances map |

> **所有场景都无需修改控制层代码。**

## 跨层数据流

```
declarative/test33/04-database/
    │
    │  tfvars: polardb_clusters = { "01" = {...} }
    │  tfvars: redis_instances = { "01" = {...} }
    ▼
control/database-cluster/
    │
    │  for_each = var.polardb_clusters
    │  for_each = var.redis_instances
    │  ID 映射: vswitch_ids_map["j-data"] → 实际vswitch_id
    ▼
atomic/polardb/
atomic/redis-community/
    │
    │  单资源创建
    ▼
alicloud_polardb_cluster.this
alicloud_kvstore_instance.this
```

## 检查清单

- [ ] 原子层：每个模块只封装一种资源类型
- [ ] 原子层：无跨模块引用
- [ ] 原子层：可选字段使用空默认值
- [ ] 控制层：所有动态资源使用 for_each + map
- [ ] 控制层：无硬编码业务值
- [ ] 控制层：通过 _map 变量实现 ID 映射
- [ ] 声明层：所有拓扑通过 tfvars 的 map(object) 控制
- [ ] 声明层：跨阶段引用使用 terraform_remote_state
- [ ] 新增/删除资源只需修改 tfvars

## 参考资料

