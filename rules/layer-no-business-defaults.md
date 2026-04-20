# 控制层无业务默认值：技术默认 vs 业务默认

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

控制层是跨环境复用的编排引擎。如果控制层硬编码了业务默认值（如 `security_ips = ["127.0.0.1"]`），不同环境可能需要不同的默认值，导致控制层无法通用。区分"技术默认值"和"业务默认值"是保持控制层纯净的关键。

## 默认值分类

| 类型 | 定义 | 示例 | 允许层级 |
|------|------|------|----------|
| **技术默认值** | Terraform/Provider 运行所需的兜底值 | `try(v.xxx, "")`、`try(v.create, true)` | 控制层 ✅ |
| **业务默认值** | 与业务逻辑、安全策略、环境配置相关的值 | `security_ips = ["127.0.0.1"]`、`engine_version = "5.7"` | 仅声明层 ✅ |

## 问题：控制层包含业务默认值

```hcl
# ❌ 错误：控制层设置业务默认值
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ❌ 业务默认值 — "127.0.0.1" 是业务决策，不是技术默认值
  security_ips = try(each.value.security_ips, ["127.0.0.1"])

  # ❌ 业务默认值 — "data" 是业务决策
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    [lookup(var.security_group_ids_map, "data", "")]
}
```

**问题：**
1. "127.0.0.1" 是 API 的行为模拟，但控制层不应该替业务做这个决定
2. "data" 安全组是业务选择，test33 环境可能用 "data"，simple 环境可能不同
3. 当 API 默认值变更时，控制层硬编码值会导致 state 漂移

## 正确：区分技术默认值和业务默认值

### 技术默认值（控制层可设）

```hcl
# ✅ 技术默认值：控制层可设置
# 这些值不涉及业务决策，只是 Terraform 语言层面的空值处理

# try() 空字符串回退 — 纯技术守卫
vswitch_id    = lookup(var.vswitch_ids_map, each.value.vswitch_key, "")
resource_group_id = var.resource_group_id != "" ? var.resource_group_id : null

# null 传递 — 让 API 决定默认行为
pay_type   = try(each.value.pay_type, null)
period     = try(each.value.period, null)

# 条件守卫 — plan-time 可解析的 create 判断
create = try(each.value.create, true) && lookup(local.xxx_would_create, each.value.xxx_key, false)
```

### 业务默认值（声明层必须显式指定）

```hcl
# ✅ 业务默认值：声明层 tfvars 中显式指定

# declarative/test33/04-database/terraform.tfvars
polardb_clusters = {
  "01" = {
    security_ips       = ["127.0.0.1"]    # 业务决定：测试环境用本地回环
    security_group_key = "data"            # 业务决定：使用 data 安全组
  }
}

# declarative/prod/04-database/terraform.tfvars
polardb_clusters = {
  "01" = {
    security_ips       = ["172.31.192.0/18"]  # 业务决定：生产用 VPC 网段
    security_group_key = "data"               # 业务决定：使用 data 安全组
  }
}
```

## 判断矩阵

| 值类型 | 示例 | 控制层可设？ | 说明 |
|--------|------|-------------|------|
| 空字符串 `""` | `try(v.xxx, "")` | ✅ | 技术守卫，非业务决策 |
| `null` | `try(v.xxx, null)` | ✅ | 让 API 决定，不干预 |
| API 默认值 | `["127.0.0.1"]` | ❌ | API 行为可能变更，导致漂移 |
| 业务选择 | `"data"`, `"ecs"` | ❌ | 不同环境可能不同 |
| 计算公式 | `coalesce(v.a, v.b)` | ✅ | 技术逻辑，非业务值 |

## 特殊情况：API 默认值防漂移

某些 API 参数（如 `security_ips`）有服务端默认值。如果 Terraform 不设置该参数，API 会使用默认值，但 Terraform state 中记录为空，导致 state 漂移。

```hcl
# 方案 A（推荐）：声明层显式指定，对齐 API 默认值
# tfvars 中：security_ips = ["127.0.0.1"]

# 方案 B：控制层传递 null，让 API 决定（接受首次 apply 后可能有 diff）
security_ips = try(each.value.security_ips, null)
```

**选择原则：** 优先方案 A（声明层显式指定），仅在参数过多且用户不关心默认值时使用方案 B。

## 检查清单

- [ ] 控制层无硬编码的业务默认值（IP 网段、安全组名、规格等）
- [ ] `try()` 回退值仅使用 `""`、`null`、`true`/`false`（技术默认值）
- [ ] 业务决策（如安全组选择、白名单 IP）在声明层 tfvars 显式配置
- [ ] 控制层注释中不包含业务特定的示例值（如 `cn-hangzhou`）

## 参考资料

- `variable-atomic-defaults.md`（原子层默认值原则）
- `control-parameter-passthrough.md`（控制层参数透传）
- `layer-separation.md`（三层职责划分）
