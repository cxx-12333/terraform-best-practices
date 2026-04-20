# 原子层模块参数完整性检查

**优先级：** 高
**分类：** 模块设计

> 原子层模块变量必须覆盖 Provider Schema 中所有可配置参数，确保声明层可以控制所有资源属性。

## 为什么重要

原子层模块是声明层与云 API 之间的桥梁。如果原子层缺少参数，声明层将无法配置这些属性，导致：
1. 无法通过 IaC 管理部分资源属性
2. 依赖云厂商默认值，可能导致状态漂移
3. 模块功能不完整，需要后续补丁修复

## 检查流程

```
┌─────────────────────────────────────────────────────────────────┐
│              原子层模块参数完整性检查流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定位 Provider 源码                                          │
│     terraform-provider-alicloud/alicloud/resource_*.go          │
│     ↓                                                           │
│  2. 提取 Schema 中所有字段                                       │
│     Optional: true / Required: true                             │
│     ↓                                                           │
│  3. 对比原子层 variables.tf                                      │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ Provider 参数          │ 原子层变量        │ 状态       │ │
│     ├─────────────────────────────────────────────────────────┤ │
│     │ instance_name          │ instance_name    │ ✅ 已定义   │ │
│     │ period                 │ period           │ ❌ 缺失     │ │
│     │ auto_renew             │ auto_renew       │ ❌ 缺失     │ │
│     └─────────────────────────────────────────────────────────┘ │
│     ↓                                                           │
│  4. 补充缺失参数，遵循分层规范                                   │
│     - ForceNew 参数：注释标注                                    │
│     - 可选参数：设置安全默认值                                   │
│     - Computed only：不定义变量                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Provider Schema 字段分类处理

| Schema 定义 | 处理方式 | 原子层默认值 |
|-------------|---------|-------------|
| `Required: true` | 必须定义变量，无默认值 | 无 |
| `Optional: true, Computed: true` | 定义变量，`null` 默认值 | `null` |
| `Optional: true` | 定义变量，空值默认值 | `""` / `[]` / `{}` |
| `Computed: true` only | 不定义变量 | 不适用 |
| `ForceNew: true` | 注释标注 ForceNew | 视情况设置 |

## 案例：RocketMQ 参数补全

### 问题：原子层参数缺失

```hcl
# atomic/rocketmq/variables.tf（原始版本）
variable "series_code" { ... }
variable "payment_type" { ... }
# ❌ 缺少：period, auto_renew, commodity_code, flow_out_bandwidth...
```

### Provider Schema 字段

```go
// terraform-provider-alicloud/alicloud/resource_alicloud_rocketmq_instance.go
"period": {
    Type:     schema.TypeInt,
    Optional: true,  // 应定义变量
},
"auto_renew": {
    Type:     schema.TypeBool,
    Optional: true,  // 应定义变量
},
"commodity_code": {
    Type:     schema.TypeString,
    Optional: true,
    ForceNew: true,  // ForceNew 参数，注释标注
},
"flow_out_bandwidth": {
    Type:     schema.TypeInt,
    Optional: true,  // 应定义变量
},
```

### 修复：补充完整参数

```hcl
# atomic/rocketmq/variables.tf（修复后）

################################################################################\n# 包年包月计费配置（payment_type=Subscription 时使用）\n################################################################################\n
variable "period" {
  description = "购买周期（包年包月时使用）"
  type        = number
  default     = null
}

variable "auto_renew" {
  description = "是否自动续费（包年包月时使用）"
  type        = bool
  default     = false
}

variable "commodity_code" {
  description = "商品代码 ForceNew"
  type        = string
  default     = ""
}

variable "flow_out_bandwidth" {
  description = "公网带宽（MB），公网访问时使用"
  type        = number
  default     = null
}
```

## 检查清单

### 新增模块时

- [ ] 阅读 Provider 源码，提取 Schema 中所有 Optional/Required 字段
- [ ] 为每个字段创建对应变量
- [ ] ForceNew 参数在 description 中标注
- [ ] 按参数类型设置正确的默认值

### 审计现有模块时

- [ ] 对比 Provider Schema 与 variables.tf
- [ ] 列出缺失参数清单
- [ ] 评估缺失参数的业务影响
- [ ] 按优先级补充参数

## 常见遗漏参数

| 资源类型 | 常见遗漏参数 |
|----------|-------------|
| 数据库类 | `period`, `auto_renew`, `auto_renew_period` |
| 网络类 | `bandwidth`, `flow_out_bandwidth` |
| 安全类 | `security_group_ids`, `ip_whitelists` |
| 运维类 | `maintain_time`, `remark`, `resource_group_id` |

## 控制层类型定义完整性检查

> **控制层 `map(object({...}))` 类型定义必须覆盖 `main.tf` 中所有 `each.value.*` 引用，否则字段会被 Terraform 静默丢弃。**

### 检查流程

```
1. 提取 main.tf 中所有 each.value.XXX 引用
   grep -oP 'each\.value\.\w+' control/web-cluster/main.tf | sort -u
   ↓
2. 提取 variables.tf 中 map(object({...})) 的所有字段
   ↓
3. 对比：main.tf 引用字段 ⊆ variables.tf 类型定义字段
   ↓
4. 缺失字段 → 补充 optional() 声明
```

### 案例：NLB 安全组字段缺失

```hcl
# ❌ main.tf 引用了 security_group_key，但 variables.tf 类型定义中没有
# main.tf
security_group_ids = try(each.value.security_group_key, null) != null ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] : []

# variables.tf（缺失 security_group_key）
variable "nlbs" {
  type = map(object({
    address_type       = string
    cross_zone_enabled = optional(bool, true)
    create             = optional(bool, true)
    # ← security_group_key 缺失！
  }))
}
```

**结果**：Terraform 静默丢弃 `security_group_key`，`each.value.security_group_key` 永远为 `null`，安全组永远为空，`terraform plan` 显示 No changes。

### 检查清单（控制层）

- [ ] `main.tf` 中所有 `each.value.*` 引用的字段都在 `variables.tf` 的 `map(object)` 中声明
- [ ] 新增 `each.value.*` 引用时，同步更新 `variables.tf` 类型定义
- [ ] `terraform plan` 显示 No changes 但预期有变更时，优先检查字段是否被类型定义静默丢弃

## 相关规则

- `variable-atomic-defaults` - 原子层默认值：技术纯度原则
- `variable-layered-type-design` - 三层变量类型设计规范（含 Rule 5: 控制层字段对齐检查）
- `provider-optional-api-mandatory` - Provider 参数 Schema 类型与处理规范
- `code-reference-documentation` - 原子层代码添加官方文档参考链接

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `provider-optional-api-mandatory.md`（Provider 参数 Schema 类型）
- `openapi-reference-lookup.md`（OpenAPI 文档查找规范）
