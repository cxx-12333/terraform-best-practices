# 声明层分阶段配置与共享参数体系

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

声明层按阶段（Stage）组织目录，每个阶段对应一个控制层模块。阶段间通过 `terraform_remote_state` 传递 ID map。环境级共享参数（region、az_map、tags）抽取到 `00-shared/shared.tfvars`，避免重复配置和跨阶段不一致。

## 目录结构模式

```
declarative/{environment}/
├── 00-shared/              # 共享参数（无 Terraform 资源）
│   └── shared.tfvars       # 环境级参数：env, region, az_map, tags
├── 01-network/             # 网络层：VPC、VSwitch / VNet、Subnet
├── 02-security/            # 安全层：安全组 / NSG、ACL
├── 03-compute/             # 计算层：ACK / EKS / AKS 集群
├── 04-database/            # 数据层：RDS、Redis、Kafka、MongoDB
├── 05-middleware/          # 中间件层：网关、NAS / EFS / File Share
└── 06-web/                 # Web 层：SLB / ALB / LB、ECS / EC2 / VM
```

### 阶段编号约定

| 编号 | 阶段名 | 职责 | 依赖上游 |
|------|--------|------|----------|
| 00 | shared | 环境级共享参数 | 无 |
| 01 | network | 网络拓扑（VPC/VNet、VSwitch/Subnet） | shared |
| 02 | security | 安全组 / NSG | network |
| 03 | compute | 容器集群（ACK/EKS/AKS） | network, security |
| 04 | database | 数据库/缓存/消息队列 | network, security |
| 05 | middleware | 中间件（网关、文件存储） | network, security |
| 06 | web | Web 服务（负载均衡、计算实例） | network, security, compute |

**依赖方向**：编号小的阶段先 apply，编号大的阶段通过 `terraform_remote_state` 引用上游输出。

## 双层配置体系

### 第一层：shared.tfvars（环境级共享参数）

```hcl
# 00-shared/shared.tfvars — 环境级参数，所有阶段共用

# ——— 基础标识 ———
env     = "simple"
project = "cc"

# ——— 区域配置（多云适配）———
# 阿里云
region = "cn-hangzhou"
# AWS
# region = "us-east-1"
# Azure
# location = "eastus"

# ——— 可用区映射（跨 region 迁移只需改这里）———
az_map = {
  j = "cn-hangzhou-j"
  k = "cn-hangzhou-k"
  # AWS:
  # a = "us-east-1a"
  # b = "us-east-1b"
  # Azure:
  # a = "eastus-1"
  # b = "eastus-2"
}

# ——— 统一标签 ———
tags = {
  "ops-env" = "simple"
  "project" = "cc"
}
```

### 第二层：terraform.tfvars（阶段级资源配置）

```hcl
# 01-network/terraform.tfvars — 仅本阶段资源配置

# 阿里云 VSwitch
vswitches = {
  "j-ecs" = {
    cidr_block = "172.31.192.0/24"
    zone_key   = "j"
    purpose    = "ECS"
  }
  "k-ecs" = {
    cidr_block = "172.31.193.0/24"
    zone_key   = "k"
    purpose    = "ECS"
  }
  "j-data" = {
    cidr_block = "172.31.196.0/24"
    zone_key   = "j"
    purpose    = "Data"
  }
}

# AWS Subnet（等价配置）
# subnets = {
#   "a-public" = {
#     cidr_block = "10.0.1.0/24"
#     zone_key   = "a"
#     purpose    = "public"
#   }
#   "b-private" = {
#     cidr_block = "10.0.2.0/24"
#     zone_key   = "b"
#     purpose    = "private"
#   }
# }
```

### 配置加载方式

```bash
# 每个阶段同时加载两个 tfvars
terraform plan \
  -var-file="../00-shared/shared.tfvars" \
  -var-file="terraform.tfvars"

# 或在 terraform.tf 中配置（如使用 Terramate）
```

## 参数归属决策

| 参数类型 | 放置位置 | 示例 |
|----------|----------|------|
| 环境标识（env） | shared.tfvars | `env = "simple"` |
| 区域/位置 | shared.tfvars | `region = "cn-hangzhou"` / `location = "eastus"` |
| 可用区映射（az_map） | shared.tfvars | `az_map = { j = "cn-hangzhou-j" }` |
| 统一标签（tags） | shared.tfvars | `tags = { "ops-env" = "simple" }` |
| 项目标识（project） | shared.tfvars | `project = "cc"` |
| 资源实例配置 | terraform.tfvars | `kafka_instances = { ... }` |
| 子网 CIDR | terraform.tfvars | `cidr_block = "172.31.192.0/24"` |
| 数据库规格 | terraform.tfvars | `db_node_class = "polar.mysql.g2.xlarge"` |
| 安全组规则 | terraform.tfvars | `security_group_rules = { ... }` |

## 阶段间引用模式

### 声明层 main.tf 结构

```hcl
# 04-database/main.tf

# 引用上游阶段状态
data "terraform_remote_state" "network" {
  backend = "local"
  config = { path = "../01-network/terraform.tfstate" }
}

data "terraform_remote_state" "security" {
  backend = "local"
  config = { path = "../02-security/terraform.tfstate" }
}

# 调用控制层模块
module "db" {
  source = "../../../control/database-cluster"

  # 从 shared.tfvars 传入
  env    = var.env
  region = var.region
  az_map = var.az_map
  tags   = var.tags

  # 从上游状态传入 ID map
  vswitch_ids_map         = data.terraform_remote_state.network.outputs.vswitch_ids_map
  security_group_ids_map  = data.terraform_remote_state.security.outputs.security_group_ids_map

  # 从本阶段 tfvars 传入
  kafka_instances = var.kafka_instances
  redis_instances = var.redis_instances
}
```

### 声明层 outputs.tf（传递给下游）

```hcl
# 01-network/outputs.tf — 输出供下游阶段引用

output "vswitch_ids_map" {
  description = "VSwitch/Subnet ID 映射（key = 逻辑名）"
  value = {
    "j-ecs"  = module.network.vswitch_ids["j-ecs"]
    "k-ecs"  = module.network.vswitch_ids["k-ecs"]
    "j-data" = module.network.vswitch_ids["j-data"]
    "j-pod"  = module.network.vswitch_ids["j-pod"]
    "k-pod"  = module.network.vswitch_ids["k-pod"]
  }
}

output "vpc_id" {
  description = "VPC/VNet ID"
  value = module.network.vpc_id
}
```

## 多云目录结构

```
declarative/
├── alicloud/             # 阿里云环境
│   ├── simple/
│   │   ├── 00-shared/shared.tfvars  # region="cn-hangzhou"
│   │   └── ...
│   └── prod/
│       ├── 00-shared/shared.tfvars  # region="cn-beijing"
│       └── ...
├── aws/                  # AWS 环境
│   ├── dev/
│   │   ├── 00-shared/shared.tfvars  # region="us-east-1"
│   │   └── ...
│   └── prod/
│       ├── 00-shared/shared.tfvars  # region="us-west-2"
│       └── ...
└── azure/                # Azure 环境
    ├── dev/
    │   ├── 00-shared/shared.tfvars  # location="eastus"
    │   └── ...
    └── prod/
        ├── 00-shared/shared.tfvars  # location="westeurope"
        └── ...
```

**环境差异只在 shared.tfvars 和 terraform.tfvars 中体现**，控制层和原子层代码尽量复用。

## 多环境管理（单云多环境）

```
declarative/
├── simple/              # 简单环境
│   ├── 00-shared/shared.tfvars    # env="simple", region="cn-hangzhou"
│   ├── 01-network/terraform.tfvars
│   └── ...
├── test33/              # 测试环境
│   ├── 00-shared/shared.tfvars    # env="test2", region="cn-hangzhou"
│   ├── 01-network/terraform.tfvars
│   └── ...
└── prod/                # 生产环境（未来）
    ├── 00-shared/shared.tfvars    # env="prod", region="cn-beijing"
    ├── 01-network/terraform.tfvars
    └── ...
```

## 检查清单

- [ ] 每个环境有 `00-shared/shared.tfvars` 包含 env、region/location、az_map、tags
- [ ] 阶段编号按依赖顺序排列（01 → 06）
- [ ] 下游阶段通过 `terraform_remote_state` 引用上游 ID map
- [ ] 每个阶段 outputs.tf 输出完整的 ID map 供下游使用
- [ ] 资源配置写在各阶段 `terraform.tfvars`，不写在 shared.tfvars
- [ ] 跨环境复用的参数在 shared.tfvars，环境特定的在 terraform.tfvars
- [ ] terraform plan 同时加载 shared.tfvars 和 terraform.tfvars
- [ ] 多云场景下，az_map 使用统一的逻辑 key（j/k 或 a/b）

## 参考资料

- `layer-id-mapping.md`（ID 映射模式）
- `declaration-layer-configuration-patterns.md`（声明层 tfvars 编写规范）
- `variable-layered-type-design.md`（三层变量类型设计）
- `control-parameter-passthrough.md`（控制层参数透传）
