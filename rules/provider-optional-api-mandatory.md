# Provider 参数 Schema 类型与处理规范

**优先级：** 中
**分类：** Provider 配置

> 根据 Provider 源码中的 Schema 定义（Optional/Computed），正确处理不同类型参数。

## 为什么重要

Provider Schema 中标记为 Optional 的参数，在云厂商 API 中可能是必填的。如果不正确处理这些参数的默认值，会导致 apply 失败或状态漂移。

## Provider Schema 类型分类

| Schema 定义 | 类型 | 特征 | 原子层处理 |
|-------------|------|------|-----------|
| `Computed: true` only | **只读状态** | API 返回，不可设置 | 不定义变量 |
| `Optional: true` only | **动作参数** | 执行操作，有状态依赖 | `null` + `ignore_changes` |
| `Optional + Computed` | **状态参数** | 可设置或 API 返回 | 视联动关系处理 |

## 典型案例：阿里云 MongoDB SSL 参数

### Provider 源码分析

```go
// ssl_action - 动作参数（Optional only）
"ssl_action": {
    Type:         schema.TypeString,
    Optional:     true,           // 可设置
    // 无 Computed               // 不会从 API 返回
}

// force_encryption - 状态参数（Optional + Computed）
"force_encryption": {
    Type:         schema.TypeString,
    Optional:     true,           // 可设置
    Computed:     true,           // 会从 API 返回
}

// ssl_status - 只读状态（Computed only）
"ssl_status": {
    Type:     schema.TypeString,
    Computed: true,               // 只读，不可设置
}
```

### Update 函数联动逻辑

```go
// ModifyDBInstanceSSL API 触发条件
if d.HasChange("ssl_action") || d.HasChange("force_encryption") {
    update = true  // 任一变化都触发 API
}

if update {
    // 调用 ModifyDBInstanceSSL
    // API 要求 SSLAction 必填！
    // ssl_action=null 时报 MissingSSLAction
}
```

### 最优解

| 参数 | 默认值 | 处理方式 | 原因 |
|------|--------|---------|------|
| `ssl_action` | `null` | `ignore_changes` | 动作参数，API 必填但无法安全默认 |
| `force_encryption` | `null` | 不忽略 | Computed 自动填充，不触发 HasChange |
| `ssl_status` | 不定义 | 无 | 只读状态，无需变量 |

> **关键原理**：`force_encryption` 有 `Computed: true`，设置 `null` 时 Provider 不传参数，API 返回值自动写入 state，不会触发 `HasChange`。

## 实施规范

### 1. 原子层变量定义

```hcl
# atomic/mongodb/variables.tf

variable "ssl_action" {
  description = "SSL 操作（动作参数）。Open/Close/Update，null=不执行"
  type        = string
  default     = null  # 动作参数，ignore_changes

  validation {
    condition     = var.ssl_action == null || contains(["Open", "Close", "Update"], var.ssl_action)
    error_message = "ssl_action 取值：Open / Close / Update 或 null。"
  }
}

variable "force_encryption" {
  description = "是否强制加密。Computed 参数，由 API 返回"
  type        = string
  default     = null  # Computed 参数，null 不触发 HasChange
}

# ssl_status 不定义变量（只读状态）
```

### 2. 原子层资源配置

```hcl
# atomic/mongodb/main.tf

resource "alicloud_mongodb_instance" "this" {
  ssl_action       = var.ssl_action
  force_encryption = var.force_encryption

  lifecycle {
    ignore_changes = [ssl_action]  # 只忽略动作参数
  }
}
```

### 3. 控制层变量定义

```hcl
# control/database-cluster/variables.tf

variable "mongodb_instances" {
  type = map(object({
    ssl_action       = optional(string)  # null=不执行操作
    force_encryption = optional(string)  # Computed 参数，不设默认值
  }))
}
```

### 4. 控制层传值

```hcl
# control/database-cluster/main.tf

module "mongodb" {
  source = "../../atomic/mongodb"

  ssl_action       = try(each.value.ssl_action, null)
  force_encryption = try(each.value.force_encryption, null)
}
```

## 决策流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   参数处理决策流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 查阅 Provider 源码 Schema 定义                              │
│     ↓                                                           │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ 是否有 Computed?                                           │ │
│  └────────────────────────┬──────────────────────────────────┘ │
│                           │                                     │
│              ┌────────────┴────────────┐                       │
│              ↓                         ↓                       │
│          Computed only            Optional + Computed          │
│              │                         │                       │
│         只读状态                   状态参数                      │
│              │                         │                       │
│      不定义变量                  检查联动关系                   │
│                                   │                           │
│                        ┌──────────┴──────────┐                │
│                        ↓                     ↓                │
│                   有联动               无联动                 │
│                        │                     │                │
│                   default=null         设置有效默认值          │
│                   不触发HasChange                              │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ Optional only（无 Computed）                               │ │
│  └────────────────────────┬──────────────────────────────────┘ │
│                           │                                     │
│                      动作参数                                    │
│                           │                                     │
│              default=null + ignore_changes                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 已知陷阱参数

### 阿里云 MongoDB

| 参数 | Schema | 类型 | 正确处理 |
|------|--------|------|----------|
| `ssl_status` | `Computed: true` | 只读 | 不定义变量 |
| `ssl_action` | `Optional: true` | 动作 | `null` + `ignore_changes` |
| `force_encryption` | `Optional + Computed` | 状态 | `null`（Computed 自动填充） |
| `tde_status` | `Optional + Computed` | 状态 | 视联动关系处理 |

## 状态依赖型动作参数的特殊处理

> **定义**：动作参数的执行结果依赖于资源当前状态，无法设置安全的通用默认值。

### 典型特征

1. **API 必填**：动作参数在 API 调用时必填
2. **状态依赖**：执行结果取决于资源当前状态
3. **无安全默认值**：任何默认值都可能在某些状态下触发错误

### MongoDB ssl_action 状态依赖矩阵

```
┌─────────────────────────────────────────────────────────────┐
│                  ssl_action 执行结果                         │
├─────────────────────────────────────────────────────────────┤
│  当前 SSL 状态    │  ssl_action  │  结果                     │
│  ─────────────    │  ──────────  │  ──────                   │
│  未开启           │  null        │  ❌ MissingSSLAction     │
│  未开启           │  "Close"    │  ❌ SSLNotEnabled        │
│  未开启           │  "Open"     │  ✅ 成功开启              │
│  已开启           │  "Open"     │  ❌ SSLAlreadyEnabled    │
│  已开启           │  "Close"    │  ✅ 成功关闭              │
└─────────────────────────────────────────────────────────────┘
│  结论：无法设置安全默认值，必须 ignore_changes              │
└─────────────────────────────────────────────────────────────┘
```

### 处理方案

| 方案 | 说明 | 适用场景 |
|------|------|--------|
| `ignore_changes` | Terraform 忽略此参数，用户通过控制台手动管理 | 推荐，最安全 |
| 临时移除 ignore_changes | 需要变更时临时注释，apply 后恢复 | 需要通过 Terraform 执行动作 |

### 声明层注释模板

```hcl
# ssl_action = "Open"                      # ⚠️ 动作参数，无法通过 Terraform 纳管
#                                             # 原因：API 设计缺陷 - 状态依赖型动作参数
#                                             # - null → MissingSSLAction（API 必填）
#                                             # - "Close" → SSLNotEnabled（当 SSL 未开启时）
#                                             # - "Open" → 会实际开启 SSL（需要 ignore_changes 阻止）
#                                             # 解决方案：通过阿里云控制台手动管理 SSL，Terraform 忽略此参数
```

## 检查清单

- [ ] 查阅 Provider 源码，确认 Schema：`Optional only` / `Optional + Computed` / `Computed only`
- [ ] `Computed only` → 只读状态，不定义变量
- [ ] `Optional only` → 动作参数，`null` + `ignore_changes`
- [ ] `Optional + Computed` → 状态参数，检查联动关系
- [ ] 联动状态参数设置 `null`，让 Computed 自动填充
- [ ] 独立状态参数设置有效默认值

## 相关规则

- `variable-atomic-defaults` - 原子层默认值：技术纯度原则
- `resource-lifecycle` - 生命周期块使用规范

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `variable-forcenew-defaults.md`（ForceNew 空值规则）
- `variable-atomic-defaults.md`（原子层默认值原则）
