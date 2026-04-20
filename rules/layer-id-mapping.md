# 层级 ID 映射：基于 Key 的资源引用

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

控制层模块需要引用其他层创建的资源（VSwitch、安全组）。使用原始 ID 会造成紧耦合，导致环境无法复现。基于 Key 的 ID 映射将逻辑名与物理 ID 解耦。

## 问题：硬编码 ID 引用

```hcl
# ❌ 错误：控制层硬编码 ID
module "polardb" {
  source = "../../atomic/polardb"

  vswitch_id = "vsw-bp1ilacjy9xed5e9vygeb"  # 硬编码！
  security_group_ids = ["sg-bp123456"]        # 硬编码！
}
```

**问题：**
1. 无法复现到新环境（ID 不存在）
2. 无法更换 VSwitch，必须修改代码
3. 违反"控制层无硬编码值"原则

## 解决方案：基于 Key 的 ID 映射

### 模式 1：Map 变量用于 ID 查找

```hcl
# control/database-cluster/variables.tf

variable "vswitch_ids_map" {
  description = "VSwitch ID 映射。key = 逻辑名(j-ecs/k-ecs/j-data), value = 实际 ID"
  type        = map(string)
}

variable "security_group_ids_map" {
  description = "安全组 ID 映射。key = 逻辑名(ecs/pod/data), value = 实际 ID"
  type        = map(string)
  default     = {}
}

variable "az_map" {
  description = "可用区 ID 映射。key = 逻辑名(j/k/l), value = 实际可用区 ID(cn-hangzhou-j)"
  type        = map(string)
  default     = {}
}
```

### 模式 2：声明层控制 Key 选择

```hcl
# declarative/test33/04-database/terraform.tfvars

polardb_clusters = {
  "01" = {
    db_node_class = "polar.mysql.g2.xlarge"
    vswitch_key   = "j-data"           # 逻辑 key，非实际 ID
    security_group_key = "data"        # 逻辑 key，非实际 ID
  }
}

redis_instances = {
  "01" = {
    instance_class     = "redis.shard.with.proxy.small.ce"
    vswitch_key        = "j-ecs"       # 不同的 VSwitch
    security_group_key = "data"        # 相同的安全组
    zone_key           = "j"           # 主可用区（逻辑 key）
    secondary_zone_key = "k"           # 备可用区（跨 AZ 高可用）
  }
}
```

### 模式 3：控制层将 Key 映射为 ID

```hcl
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # Key → ID 映射
  vswitch_id = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]

  # 可用区 Key → ID 映射
  # ⚠️ 注意：can() 对空字符串返回 true，需用 try(..., "") != "" 判断
  zone_id           = try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "")
  secondary_zone_id = try(each.value.secondary_zone_key, "") != "" ? var.az_map[each.value.secondary_zone_key] : try(each.value.secondary_zone_id, "")
  hidden_zone_id    = try(each.value.hidden_zone_key, "") != "" ? var.az_map[each.value.hidden_zone_key] : try(each.value.hidden_zone_id, "")

  # 安全组：控制层不设业务默认值
  # ⚠️ 原则：控制层是共用模块，不应硬编码业务值（如 "data"）
  # 业务决策应在声明层显式配置 security_group_key
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    []  # 空值，声明层必须显式配置
}
```

### 模式 4：通过 Remote State 实现跨阶段引用

```hcl
# declarative/test33/04-database/main.tf

data "terraform_remote_state" "network" {
  backend = "local"
  config = { path = "../01-network/terraform.tfstate" }
}

data "terraform_remote_state" "security" {
  backend = "local"
  config = { path = "../02-security/terraform.tfstate" }
}

module "db" {
  source = "../../../control/database-cluster"

  # 从上游阶段传递完整的 map
  vswitch_ids_map        = data.terraform_remote_state.network.outputs.vswitch_ids_map
  security_group_ids_map = data.terraform_remote_state.security.outputs.security_group_ids_map

  # 可用区 map 来自 shared.tfvars（跨环境共享）
  az_map = var.az_map
}
```

### 模式 5：az_map 定义在 shared.tfvars（跨环境共享）

```hcl
# declarative/test33/00-shared/shared.tfvars

# 可用区映射 — key 为逻辑名，value 为实际可用区 ID
# 跨 region 迁移只需修改此 map，声明层所有 zone_key 保持不变
az_map = {
  j = "cn-hangzhou-j"     # 主可用区
  k = "cn-hangzhou-k"     # 备可用区
}
```

**跨 Region 迁移优势：**

| Region | az_map 定义 | zone_key 不变 |
|--------|-------------|---------------|
| 杭州 | `{j = "cn-hangzhou-j", k = "cn-hangzhou-k"}` | `zone_key = "j"` |
| 北京 | `{j = "cn-beijing-a", k = "cn-beijing-b"}` | `zone_key = "j"` |
| 上海 | `{j = "cn-shanghai-a", k = "cn-shanghai-b"}` | `zone_key = "j"` |
```

## ID Map 输出模式

每个声明阶段输出完整的 ID map：

```hcl
# declarative/test33/01-network/outputs.tf

output "vswitch_ids_map" {
  description = "VSwitch ID 映射（key = 逻辑名）"
  value = {
    "j-ecs"  = module.network.vswitch_ids["j-ecs"]
    "k-ecs"  = module.network.vswitch_ids["k-ecs"]
    "j-data" = module.network.vswitch_ids["j-data"]
    "j-pod"  = module.network.vswitch_ids["j-pod"]
    "k-pod"  = module.network.vswitch_ids["k-pod"]
  }
}

# declarative/test33/02-security/outputs.tf

output "security_group_ids_map" {
  description = "安全组 ID 映射（key = 逻辑名）"
  value = {
    "ecs"  = module.security.sg_ecs_id
    "pod"  = module.security.sg_pod_id
    "data" = module.security.sg_data_id
  }
}
```

## Key 命名规范

| 资源类型 | Key 模式 | 示例 |
|----------|----------|------|
| 可用区 | `{az_letter}` | j, k, l, m |
| VSwitch | {az}-{purpose} | j-ecs, k-ecs, j-data, j-pod |
| 安全组 | {purpose} | ecs, pod, data |
| ALB ACL | {purpose} | nginx, ack-ingress |
| SLB ACL | {purpose} | waf, internal |

## 决策矩阵

| 场景 | 使用方式 |
|------|----------|
| 控制层需要 VSwitch ID | var.vswitch_ids_map[each.value.vswitch_key] |
| 控制层需要安全组 ID | lookup(var.security_group_ids_map, each.value.security_group_key, "") |
| 控制层需要可用区 ID | try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "") |
| 声明阶段传递 ID 给控制层 | data.terraform_remote_state.xxx.outputs.xxx_map |
| 声明 tfvars 指定资源 | 使用 vswitch_key、security_group_key、zone_key（非实际 ID） |

## 控制层无业务默认值原则

**原则：控制层是共用模块，不应硬编码业务默认值。**

| 层级 | 职责 | 业务默认值 |
|------|------|----------|
| 原子层 | 技术实现 | ❌ 无 |
| 控制层 | 编排逻辑 | ❌ **不应有**（共用模块）|
| 声明层 | 业务配置 | ✅ 显式配置 |

```hcl
# ❌ 错误：控制层硬编码业务默认值
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  compact([lookup(var.security_group_ids_map, "data", "")])  # 硬编码 "data"！

# ✅ 正确：控制层不设默认值，声明层必须显式配置
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  []  # 空值，强制声明层配置

# 声明层（terraform.tfvars）— 业务决策在这里
security_group_key = "data"  # 显式配置，非隐式默认
```

**原因：**
1. 控制层可能被多个声明层复用（test33/test34/demo）
2. 不同环境的安全组策略可能不同
3. 显式配置更安全，避免隐藏默认值导致的意外

## 常见陷阱

### 陷阱：can() 函数对空字符串返回 true

```hcl
# ❌ 错误写法：can() 对空字符串返回 true，导致用 "" 作为 key 查找 map
zone_id = can(each.value.zone_key) ? var.az_map[each.value.zone_key] : "default"
# 当 zone_key = "" 时，can() 返回 true，然后 az_map[""] 报 Invalid index

# ✅ 正确写法：显式检查非空
zone_id = try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "")
```

### 陷阱：声明层直接使用 zone_id

```hcl
# ❌ 错误：声明层硬编码可用区 ID，无法统一管理
zone_id           = "cn-hangzhou-j"
secondary_zone_id = "cn-hangzhou-k"

# ✅ 正确：使用 zone_key 通过 az_map 统一入口
zone_key           = "j"    # 由 az_map 映射到 cn-hangzhou-j
secondary_zone_key = "k"    # 由 az_map 映射到 cn-hangzhou-k
```

**优势：**
- 跨 Region 迁移只需修改 az_map，声明层无需改动
- 统一入口，便于管理和审计
- 避免因不同产品 zone_id 格式差异导致的问题

## 检查清单

- [ ] 控制层变量使用 map(string) 接收 ID 输入（vswitch_ids_map, security_group_ids_map, az_map）
- [ ] 声明 tfvars 使用 _key 字段（逻辑名：vswitch_key, security_group_key, zone_key）
- [ ] 控制层通过 map 查找将 _key 映射为 _id
- [ ] 跨阶段引用使用 terraform_remote_state
- [ ] 每个阶段输出完整的 ID map 供下游使用
- [ ] 控制层不设业务默认值（如安全组 "data"），声明层必须显式配置
- [ ] 控制层和声明层无硬编码资源 ID
- [ ] az_map 定义在 shared.tfvars（跨环境共享）
- [ ] zone_key 使用单字母逻辑名（j/k/l/m）

## 参考资料

