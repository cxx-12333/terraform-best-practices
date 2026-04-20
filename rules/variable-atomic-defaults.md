# 原子层默认值：技术纯度原则

**优先级：** 高
**分类：** 三层架构

## 为什么重要

当原子层变量包含业务默认值（如 `["127.0.0.1"]`）时，会造成隐藏配置，违反"灵魂文档"原则。声明层应明确控制所有资源属性，而非依赖底层的隐藏默认值。

## 问题：原子层默认值导致状态漂移

### 错误代码

```hcl
# atomic/polardb/variables.tf
variable "security_ips" {
  description = "PolarDB 白名单 IP 列表"
  type        = list(string)
  default     = ["127.0.0.1"]  # ❌ 原子层包含业务默认值
}
```

**后果：**

1. 声明层未设置 `security_ips`
2. 控制层传递 `[]`（空值）
3. 原子层使用默认值 `["127.0.0.1"]`
4. API 未收到该值，使用云厂商默认值
5. State 记录云厂商默认值
6. **状态漂移发生！**

### 正确代码

```hcl
# atomic/polardb/variables.tf
variable "security_ips" {
  description = "PolarDB 白名单 IP 列表。空列表 = 不设置（使用云厂商默认值）"
  type        = list(string)
  default     = []  # ✅ 空 = 不干预
}

# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # ✅ 业务默认值在控制层

# declarative/simple/04-database/terraform.tfvars
security_ips = ["127.0.0.1"]  # ✅ 明确配置
```

## 层级职责矩阵

| 层级 | 职责 | 默认值 |
|------|------|--------|
| 声明层 | 用户意图，明确配置 | 可选，可使用控制层默认值 |
| 控制层 | 业务逻辑，编排 | ✅ 业务默认值在这里设置 |
| 原子层 | 技术实现 | ❌ 无业务默认值，保持纯净 |

## 状态对比机制

```
Terraform 对比：传递给资源的最终值 vs 状态文件中的值

而非：声明层值 vs 状态文件

执行链：声明层(tfvars) → 控制层(main.tf) → 原子层 → Provider 资源
```

## 决策规则

> **区分参数类型，按类型设置默认值：**
> - **API 可选参数**：原子层 `""`/`[]`，控制层设 API 默认值
> - **技术规格参数**：原子层可设合理默认值
> - **业务敏感参数**：原子层 `""`/`[]`，声明层显式配置
> - **功能开关关联参数**：控制层根据开关状态动态计算，关闭时强制填 API 默认值

## 参数类型分类

### 1. API 可选参数（必须空值）

**特征**：不传时 API 有默认值，传递后覆盖 API 默认值

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `security_ips` | 白名单 IP | 原子层 `[]`，控制层设 API 默认值 |
| `backup_time` | 备份时间 | 原子层 `""`，控制层设 API 默认值 |

### 2. 技术规格参数（可设合理默认值）

**特征**：必填参数，有"合理推荐"的技术选择

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `engine_version` | 数据库版本 | 原子层设 `"7.0"`（最新稳定版）|
| `payment_type` | 付费类型 | 原子层设 `"PayAsYouGo"`（开发友好）|
| `series_code` | 实例系列 | 原子层设 `"standard"`（最常用）|
| `service_code` | 服务代码 | 原子层设 `"rmq"`（RocketMQ 5.0）|

**为什么可以设默认值？**
1. 这些是必填参数，不传会导致 API 报错
2. 默认值是"技术规格"而非"业务决策"
3. 设置默认值不会导致状态漂移（API 收到的值 = State 记录的值）

### 3. 业务敏感参数（必须空值）

**特征**：涉及安全、合规、成本的业务决策

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `password` | 数据库密码 | 原子层 `""`，声明层显式配置 |
| `security_group_ids` | 安全组 | 原子层 `""`，声明层显式配置 |
| `whitelist` | IP 白名单 | 原子层 `[]`，声明层显式配置 |

### 4. 功能开关关联参数（控制层动态计算）

**特征**：参数仅在某个功能开关开启时才有意义，关闭时 API 返回特定默认值

**问题**：若全链路默认值设为“合理值”（如 `7`），而功能关闭时 API 返回不同值（如 `-1`），会导致每次 `terraform plan` 检测到状态漂移。

#### 错误代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = 7  # ❌ 无论是否开启跨域复制都传 7

# control/security/main.tf
copied_snapshots_retention_days = try(each.value.copied_snapshots_retention_days, 7)
# ❌ enable_cross_region_copy=false 时也传 7，API 实际存的是 -1 → 状态漂移
```

#### 正确代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = -1  # ✅ 与 API 默认值一致（功能未开启时 API 返回 -1）

# control/security/main.tf — 控制层根据开关状态动态计算
# 未开启跨域 → 强制 -1（与 API 一致，零漂移）
# 已开启跨域 → 取用户配置或默认 7
copied_snapshots_retention_days = try(each.value.enable_cross_region_copy, false)
  ? try(each.value.copied_snapshots_retention_days, 7)
  : -1
```

#### 识别特征

| 判断条件 | 说明 |
|------|------|
| 参数有前置功能开关 | 如 `enable_xxx` 控制 `xxx_retention_days` |
| 功能关闭时 API 有固定默认值 | 如跨域复制关闭时 API 返回 `-1` |
| 功能开启时参数才有业务意义 | 关闭时参数无实际作用 |

#### 处理规范

| 层级 | 处理方式 |
|------|--------|
| 原子层 | 默认值 = API 在功能关闭时的返回值（如 `-1`） |
| 控制层 | 三元表达式根据开关动态计算：关闭→API默认值，开启→用户值/业务默认值 |
| 声明层 | 通常无需显式配置（控制层自动处理） |

#### 已知案例

| 资源 | 开关参数 | 关联参数 | 关闭时 API 值 | 开启时默认值 |
|------|---------|---------|-------------|-------------|
| ECS 快照策略 | `enable_cross_region_copy` | `copied_snapshots_retention_days` | `-1` | `7` |

> **参考文件**：`control/security/main.tf` L153-157 — 跨域复制保留天数动态计算

### 4. 功能开关关联参数（控制层动态计算）

**特征**：参数仅在某个功能开关开启时才有意义，关闭时 API 返回特定默认值

**问题**：若全链路默认值设为“合理值”（如 `7`），而功能关闭时 API 返回不同值（如 `-1`），会导致每次 `terraform plan` 检测到状态漂移。

#### 错误代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = 7  # ❌ 无论是否开启跨域复制都传 7

# control/security/main.tf
copied_snapshots_retention_days = try(each.value.copied_snapshots_retention_days, 7)
# ❌ enable_cross_region_copy=false 时也传 7，API 实际存的是 -1 → 状态漂移
```

#### 正确代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = -1  # ✅ 与 API 默认值一致（功能未开启时 API 返回 -1）

# control/security/main.tf — 控制层根据开关状态动态计算
# 未开启跨域 → 强制 -1（与 API 一致，零漂移）
# 已开启跨域 → 取用户配置或默认 7
copied_snapshots_retention_days = try(each.value.enable_cross_region_copy, false)
  ? try(each.value.copied_snapshots_retention_days, 7)
  : -1
```

#### 识别特征

| 判断条件 | 说明 |
|------|------|
| 参数有前置功能开关 | 如 `enable_xxx` 控制 `xxx_retention_days` |
| 功能关闭时 API 有固定默认值 | 如跨域复制关闭时 API 返回 `-1` |
| 功能开启时参数才有业务意义 | 关闭时参数无实际作用 |

#### 处理规范

| 层级 | 处理方式 |
|------|--------|
| 原子层 | 默认值 = API 在功能关闭时的返回值（如 `-1`） |
| 控制层 | 三元表达式根据开关动态计算：关闭→API默认值，开启→用户值/业务默认值 |
| 声明层 | 通常无需显式配置（控制层自动处理） |

#### 已知案例

| 资源 | 开关参数 | 关联参数 | 关闭时 API 值 | 开启时默认值 |
|------|---------|---------|-------------|-------------|
| ECS 快照策略 | `enable_cross_region_copy` | `copied_snapshots_retention_days` | `-1` | `7` |

> **参考文件**：`control/security/main.tf` L153-157 — 跨域复制保留天数动态计算

## API 默认值 vs 业务默认值

### 两种默认值

| 类型 | 说明 | 位置 |
|------|------|------|
| **API 默认值** | 云厂商的默认值（如 `["127.0.0.1"]`） | 控制层 `try(..., ["127.0.0.1"])` |
| **业务默认值** | 组织特定的值（如 VPC CIDR `["172.31.192.0/18"]`） | 声明层明确配置 |

### 示例：Redis security_ips

```hcl
# atomic/redis-community/variables.tf
variable "security_ips" {
  description = "Redis 白名单 IP 列表。空列表 = 不设置（使用控制层默认值）"
  type        = list(string)
  default     = []  # ✅ 空 = 纯技术层
}

# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # ✅ API 默认值

# declarative/simple/04-database/terraform.tfvars
security_ips = ["172.31.192.0/18"]  # ✅ 业务值：VPC 网段
```

### 为什么区分 API 默认值和业务默认值？

1. **API 默认值在控制层**：声明层未设置时防止状态漂移
2. **业务值在声明层**："灵魂文档"原则 - 业务决策明确可见
3. **灵活性**：不同环境可有不同业务值，控制层提供安全回退

## 检查清单

- [ ] 识别参数类型：API 可选 / 技术规格 / 业务敏感 / 功能开关关联
- [ ] API 可选参数：原子层 `""`/`[]`，控制层设 API 默认值
- [ ] 技术规格参数：原子层可设合理默认值
- [ ] 业务敏感参数：原子层 `""`/`[]`，声明层显式配置
- [ ] 功能开关关联参数：控制层根据开关状态动态计算，关闭时强制填 API 默认值
- [ ] 运行 `terraform plan` 验证无状态漂移

## 参考资料

- `variable-forcenew-defaults.md`（ForceNew 空值规则）
- `control-parameter-passthrough.md`（控制层参数透传）
- `layer-no-business-defaults.md`（控制层禁止业务默认值）
