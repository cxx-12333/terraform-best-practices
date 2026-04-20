# 声明层配置模式

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

声明层（terraform.tfvars）是 IaC 的"用户界面"。组织良好的配置可以提升：
- **可读性**：清晰结构，易于理解
- **可维护性**：一致模式，易于修改
- **完整性**：所有参数有文档，无隐藏缺口

## 配置结构模式

### 标准分区顺序（按优先级）

```
1. 必填参数 - Required, no defaults
2. 架构配置 - Architecture-specific (sub-type dependent)
3. 版本配置 - Version related
4. 计费配置 - Billing and payment
5. 可用区配置 - Zone and region
6. 安全配置 - Security (ips, groups, encryption)
7. 认证配置 - Authentication
8. 运维配置 - Operations and maintenance
9. 备份配置 - Backup and recovery
10. 高级配置 - Advanced/low-frequency
```

### 示例：Kafka 多类型配置

```hcl
kafka_instances = {
  #############################################################################
  # 实例 01：Serverless 版（推荐）- 弹性伸缩，按量计费
  #############################################################################
  "01" = {
    # ——— 必填参数 ———
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    deploy_type = 5

    # ——— Serverless 配置 ———
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }

    # ——— 版本配置 ———
    service_version = "" # Serverless 默认 3.3.1

    # ——— 计费配置 ———
    paid_type = "PostPaid"

    # ——— 可用区配置 ———
    zone_id = "" # 留空由系统自动选择

    # ——— 功能配置 ———
    enable_auto_group = false
    enable_auto_topic = "disable"
    default_topic_partition_num = 12

    # ——— 高级配置 ———
    config = ""
    kms_key_id = ""
    resource_group_id = ""
  }
}
```

## 产品文档模式

### 使用表格展示复杂信息

```hcl
# ═══════════════════════════════════════════════════════════════════════════════
# 一、Kafka 实例类型（instance_type）分类
# ═══════════════════════════════════════════════════════════════════════════════
# ┌─────────────────────────┬──────────────────────────────────────────────────┐
# │ 实例类型 │ 特点 │
# ├─────────────────────────┼──────────────────────────────────────────────────┤
# │ alikafka（传统版） │ 固定规格，需配置 disk_type/disk_size/io_max │
# │ │ 版本：2.2.0 / 2.6.2 │
# ├─────────────────────────┼──────────────────────────────────────────────────┤
# │ alikafka_serverless │ 弹性伸缩，按量计费（推荐） │
# │ （Serverless 版） │ 版本：3.3.1（默认） │
# └─────────────────────────┴──────────────────────────────────────────────────┘
```

### 使用表格展示参数约束

```hcl
# ┌─────────┬───────────────────┬──────────────────┐
# │ io_max │ partition_num 范围 │ disk_size 范围 │
# ├─────────┼───────────────────┼──────────────────┤
# │ 20 │ 50-450 │ 500-6100 GB │
# │ 30 │ 50-450 │ 800-6100 GB │
# │ 60 │ 80-450 │ 1400-6100 GB │
# └─────────┴───────────────────┴──────────────────┘
```

## 参数注释模式

### ForceNew 参数

```hcl
disk_type = 1 # 磁盘类型：0=高效云盘 / 1=SSD 云盘（ForceNew）
zone_id = "" # 可用区 ID（ForceNew）
```

### 枚举值

```hcl
enable_auto_topic = "disable" # enable=开启 / disable=关闭
paid_type = "PostPaid" # PostPaid=按量付费 / PrePaid=包年包月
```

### 范围约束

```hcl
partition_num = 50 # 分区数：50-450（与 io_max 联动）
disk_size = 500 # 磁盘容量 GB：500-6100
```

### 有默认值的可选参数

```hcl
zone_id = "" # 留空由系统自动选择
config = "" # 留空使用默认配置
```

## 多类型资源模式

当资源有多个子类型且参数不同时：

### 模式：只填写相关参数

```hcl
# Serverless 版 - 只填 serverless_config
"01" = {
  instance_type = "alikafka_serverless"
  serverless_config = { ... }
  # 不填 disk_type, disk_size - 与 Serverless 无关
}

# 传统版 - 填磁盘和分区参数
"02" = {
  instance_type = "alikafka"
  disk_type = 1
  disk_size = 500
  partition_num = 50
  # 不填 serverless_config
}

# Confluent 版 - 填 confluent_config
"03" = {
  instance_type = "alikafka_confluent"
  confluent_config = { ... }
  password = "xxx"
  # 不填 serverless_config, disk_type
}
```

## 嵌套对象配置

### serverless_config 模式

```hcl
serverless_config = {
  reserved_publish_capacity = 60 # 发布容量 CU（60-600，步长60）
  reserved_subscribe_capacity = 60 # 订阅容量 CU（60-600，步长60）
}
```

### confluent_config 模式

```hcl
confluent_config = {
  # ——— Kafka 集群（必填）———
  kafka_cu = 2
  kafka_storage = 500
  kafka_replica = 3

  # ——— Zookeeper（必填）———
  zookeeper_cu = 2
  zookeeper_storage = 20
  zookeeper_replica = 3 # ForceNew

  # ——— 可选组件 ———
  control_center_cu = 0
  schema_registry_cu = 0
  connect_cu = 0
  ksql_cu = 0
}
```

## Key Rules Summary

| Rule | Description |
|------|-------------|
| **Rule 1** | 使用标准化的区块顺序排列参数 |
| **Rule 2** | 使用表格展示产品说明和参数约束 |
| **Rule 3** | ForceNew 参数必须标注 `(ForceNew)` |
| **Rule 4** | 枚举类型参数标注所有可选值 |
| **Rule 5** | 范围约束参数标注取值范围 |
| **Rule 6** | 多子类型资源只填相关参数 |
| **Rule 7** | 使用中文分隔符 `———` 区分参数分组 |
| **Rule 8** | 每个实例使用 `####` 分隔符和标题注释 |
| **Rule 9** | **禁止使用 Terraform 函数调用** |

## terraform.tfvars 禁止函数调用

### 为什么禁止？

`terraform.tfvars` 文件只支持**静态值**，不支持 Terraform 函数。这是 Terraform 的设计限制。

### 错误示例

```hcl
# terraform.tfvars 中使用函数 - 报错！
config = jsonencode({ "enable.acl" = "true" }) # Error: Function calls not allowed

# 其他函数也不支持
instance_name = format("kafka-%s", var.env) # Error
tags = merge({a="1"}, {b="2"}) # Error
```

**错误信息**：
```
│ Error: Function calls not allowed
│ on terraform.tfvars line 822:
│ 822: config = jsonencode({ "enable.acl" = "true" })
│ Functions may not be called here.
```

### 正确写法

```hcl
# 直接写 JSON 字符串
config = "{\"enable.acl\":\"true\"}"

# 使用控制层或原子层的 locals 处理逻辑
# 声明层只提供原始值，逻辑处理在控制层
config = "" # 控制层有默认逻辑
```

### 函数处理位置对照表

| 需要的函数 | 正确处理位置 | 说明 |
|------------|--------------|------|
| `jsonencode()` | 直接写 JSON 字符串 | tfvars 只能写静态值 |
| `format()` | 控制层 locals | 自动命名逻辑 |
| `merge()` | 控制层 locals | tags 合并 |
| `coalesce()` | 控制层 main.tf | 默认值处理 |
| `try()` | 控制层 main.tf | 安全取值 |

### JSON 字符串转义规则

```hcl
# 单层 JSON - 使用 \" 转义
config = "{\"enable.acl\":\"true\"}"

# 多层 JSON - 注意嵌套转义
config = "{\"k1\":\"v1\",\"k2\":{\"nested\":\"value\"}}"
```

## 注释风格指南

```hcl
# ——— 必填参数 ———
# ——— 架构配置 ———
# ——— 版本配置 ———
# ——— 计费配置 ———
# ——— 可用区配置 ———
# ——— 安全配置 ———
# ——— 认证配置 ———
# ——— 运维配置 ———
# ——— 备份配置 ———
# ——— 高级配置 ———
```

## Provider 规格代码提取流程

### 从 Provider 源码提取有效值

当不确定参数的有效取值时，必须查看 Provider 源码中的 `ValidateFunc`：

```go
// 位置：terraform-provider-alicloud/alicloud/resource_alicloud_{resource}.go
"cluster_specification": {
  Type: schema.TypeString,
  Required: true,
  ValidateFunc: StringInSlice([]string{"MSE_SC_1_2_60_c", "MSE_SC_2_4_60_c", ...}, false),
},
```

**提取步骤**：
1. 定位资源文件：`resource_alicloud_{resource_name}.go`
2. 查找 `Schema: map[string]*schema.Schema{}` 块
3. 提取 `ValidateFunc: StringInSlice([...], false)` 中的有效值列表
4. 将有效值整理到声明层注释表格中

### 规格代码命名规律

| 产品 | 规格命名模式 | 示例 |
|------|-------------|------|
| MSE | `MSE_SC_{CPU}_{内存}_{连接数}_c` | `MSE_SC_1_2_60_c` = 1核2GB 60连接 |
| Redis | `redis.{架构}.{size}.{部署类型}` | `redis.shard.small.ce` = 云原生标准版 |
| PolarDB | `polar.mysql.{规格}.{后缀}` | `polar.mysql.g1.tiny.c` = 标准版 1核1G |
| NAS | 不使用规格代码 | 使用 `storage_type` + `protocol_type` 组合 |
| OSS | 不使用规格代码 | 使用 `storage_class` + `redundancy_type` 组合 |

## 最小规格实例化模板

### 原则

为每个子类型生成**最小规格**实例化代码，便于测试验证：

```hcl
# 最小规格选择原则：
# 1. 最小 CPU/内存
# 2. 最小存储/连接数
# 3. 开发版/测试版而非生产版
# 4. 单节点而非高可用（测试环境）
```

### 模板示例

```hcl
product_instances = {
  ##############################################################################
  # 实例01：{子类型名称} — 最小规格
  ##############################################################################
  "01" = {
    # ——— 必填参数 ———
    instance_type = "{最小规格代码}" # 最小规格：说明
    vswitch_key = "j-ecs" # 部署子网
    create = true # 开启测试

    # ——— 架构配置 ———
    # ... 最小配置 ...

    # ——— 高级配置 ———
    # ... 可选配置，留空或最小值 ...
  }

  ##############################################################################
  # 实例02：{另一子类型} — 最小规格（禁用）
  ##############################################################################
  "02" = {
    # ——— 必填参数 ———
    instance_type = "{最小规格代码}"
    vswitch_key = "j-ecs"
    create = false # 禁用，需要时开启

    # ... 该子类型特有参数 ...
  }
}
```

## 产品规格对照表标准格式

### 规格代码表格

```hcl
# ┌──────────────────┬─────────┬──────────┬────────────────────────────────┐
# │ 规格代码 │ CPU/内存 │ 连接数 │ 适用场景 │
# ├──────────────────┼─────────┼──────────┼────────────────────────────────┤
# │ {CODE_MIN} │ 最小配置 │ 最小值 │ 开发测试（最小规格） │
# │ {CODE_MID} │ 中等配置 │ 中等值 │ 小规模生产 │
# │ {CODE_HIGH} │ 高配置 │ 高值 │ 中规模生产 │
# │ {CODE_SERVERLESS} │ 弹性 │ 弹性 │ Serverless（按需弹性） │
# └──────────────────┴─────────┴──────────┴────────────────────────────────┘
```

### 产品类型差异表格

```hcl
# ┌───────────┬────────────────────────────────────────────────────────────┐
# │ 类型 │ 特点与约束 │
# ├───────────┼────────────────────────────────────────────────────────────┤
# │ {TYPE_A} │ 特点 A：支持参数列表，约束条件 │
# │ {TYPE_B} │ 特点 B：与 A 的差异说明 │
# │ {TYPE_C} │ 特点 C：特殊配置要求 │
# └───────────┴────────────────────────────────────────────────────────────┘
```

### 参数联动表格

```hcl
# ┌─────────────────────┬────────────┬──────────────────────────────────┐
# │ 主参数 │ 联动参数 │ 说明 │
# ├─────────────────────┼────────────┼──────────────────────────────────┤
# │ {PARAM}=VALUE_A │ 必填：X, Y │ VALUE_A 时的必填参数 │
# │ {PARAM}=VALUE_B │ 必填：Z │ VALUE_B 时的必填参数 │
# │ {PARAM}=VALUE_C │ 可选：W │ VALUE_C 时的可选参数 │
# └─────────────────────┴────────────┴──────────────────────────────────┘
```

## 自动化检查清单

在编写声明层配置时，确保：

- [ ] 从 Provider 源码提取了有效的规格代码
- [ ] 为每个子类型提供了最小规格实例
- [ ] 产品类型差异用表格清晰展示
- [ ] 参数联动关系用表格说明
- [ ] ForceNew 参数已标注 `(ForceNew)`
- [ ] 枚举值已标注所有可选项
- [ ] 第一个实例 `create = true` 用于测试验证

## 参考资料

- 相关规则：`variable-layered-type-design.md`、`variable-auto-naming.md`
