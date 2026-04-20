# 原子层 Outputs 默认值规范：使用 null 而非空字符串

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

原子层模块支持 `create = false` 控制开关。当资源未创建时，outputs.tf 的默认值决定了控制层过滤逻辑是否正常工作。

## 核心原则

**原子层 outputs.tf 未创建时应返回 `null`，而非空字符串 `""`。**

| 层级 | outputs.tf 默认值 | 控制层过滤 | 最终输出 |
|------|------------------|-----------|----------|
| 原子层 | `try(..., null)` | `if v.xxx != null` | 不出现（被过滤）|
| 原子层 | `try(..., "")` | `if v.xxx != null` | **错误出现**（空字符串不被过滤）|

## 问题：空字符串不被 null 过滤

```hcl
# ❌ 错误：原子层使用空字符串默认值
output "instance_id" {
  value = try(alicloud_rocketmq_instance.this[0].id, "")  # 默认值 ""
}

# 控制层过滤逻辑
output "instance_ids_map" {
  value = { for k, v in module.xxx : k => v.instance_id if v.instance_id != null }
}

# create = false 时：
# - 原子层返回 "" (空字符串)
# - "" != null 为 true（空字符串不是 null！）
# - 结果：{"02" = ""} ← 错误！未创建的实例出现了
```

## 正确做法

```hcl
# ✅ 正确：原子层使用 null 默认值
output "instance_id" {
  value = try(alicloud_rocketmq_instance.this[0].id, null)  # 默认值 null
}

# 控制层过滤逻辑（不变）
output "instance_ids_map" {
  value = { for k, v in module.xxx : k => v.instance_id if v.instance_id != null }
}

# create = false 时：
# - 原子层返回 null
# - null != null 为 false
# - 结果：{} ← 正确！未创建的实例不出现
```

## 三层架构中的体现

```
┌─────────────────────────────────────────────────────────────────────┐
│  声明层 (outputs.tf)                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ # 直接透传控制层输出，无需额外过滤                               ││
│  │ output "nas_file_system_ids_map" {                              ││
│  │   value = module.middleware-cluster.nas_file_system_ids_map      ││
│  │ }                                                               ││
│  │ # 期望结果：{}（空集合）而非 {"01" = null}                       ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  控制层 (outputs.tf) — 关键过滤层                                   │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ # ✅ 正确：使用 != null 过滤（推荐）                              ││
│  │ value = { for k, v in module.nas :                              ││
│  │   k => v.file_system_id if v.file_system_id != null }           ││
│  │                                                                 ││
│  │ # ⚠️ 备选：使用 try + != "" 过滤（兼容旧代码）                    ││
│  │ value = { for k, v in module.nas :                              ││
│  │   k => try(v.file_system_id, "")                                ││
│  │   if try(v.file_system_id, "") != "" }                          ││
│  │                                                                 ││
│  │ # 两种写法等效，但推荐第一种（更简洁）                            ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  原子层 (outputs.tf)                                                │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ output "file_system_id" {                                       ││
│  │   value = try(resource.this[0].id, null)  # ← 必须是 null       ││
│  │ }                                                               ││
│  └─────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

## 最终输出要求

| 场景 | 期望输出 | 错误输出 |
|------|----------|----------|
| 所有实例未创建 | `{}` （空集合） | `{"01" = null, "02" = null}` |
| 部分创建 | `{"01" = "xxx"}` | `{"01" = "xxx", "02" = null}` |
| 全部创建 | `{"01" = "xxx", "02" = "yyy"}` | 同 |

**核心原则：未创建的实例不应出现在输出 map 中，而非值为 null。**

## 统一规范

所有原子层模块的 outputs.tf 应统一使用 `null`：

```hcl
# ✅ 正确模式
output "instance_id" {
  value = try(resource.this[0].id, null)
}

output "instance_name" {
  value = try(resource.this[0].name, null)
}

output "connection_string" {
  value = try(resource.this[0].connection_string, null)
}
```

## 实际案例

### 修复前（RocketMQ）

```hcl
# atomic/rocketmq/outputs.tf - 错误
output "instance_id" {
  value = try(alicloud_rocketmq_instance.this[0].id, "")
}

# 输出结果
rocketmq_instance_ids_map = {
  "01" = "rmq-cn-cq74q3dl606"
  "02" = ""   # ← 错误：未创建但出现了
}
```

### 修复后

```hcl
# atomic/rocketmq/outputs.tf - 正确
output "instance_id" {
  value = try(alicloud_rocketmq_instance.this[0].id, null)
}

# 输出结果
rocketmq_instance_ids_map = {
  "01" = "rmq-cn-cq74q3dl606"
  # "02" 不出现
}
```

## 检查清单

- [ ] 原子层 outputs.tf 所有 ID 类输出使用 `try(..., null)`
- [ ] 原子层 outputs.tf 所有名称类输出使用 `try(..., null)`
- [ ] 原子层 outputs.tf 所有连接地址类输出使用 `try(..., null)`
- [ ] 控制层 outputs.tf 使用 `if v.xxx != null` 过滤（推荐）或 `if try(v.xxx, "") != ""`（备选）
- [ ] 声明层 outputs.tf 直接透传控制层输出
- [ ] `create = false` 的实例不出现在最终输出中
- [ ] 所有实例未创建时，输出为 `{}`（空集合），而非 `{"01" = null}`

## 常见陷阱

### 陷阱：认为空字符串会被 null 过滤

```hcl
# ❌ 错误理解
"" != null  # 这是 true！空字符串不等于 null

# ✅ 正确理解
null != null  # 这是 false，才能被过滤
```

### 陷阱：不同模块使用不同默认值

| 模块 | 修复前 | 修复后 |
|------|--------|--------|
| redis-community | `try(..., null)` ✅ | 不需修改 |
| kafka | `try(..., null)` ✅ | 不需修改 |
| polardb | `try(..., null)` ✅ | 不需修改 |
| rocketmq | `try(..., "")` ❌ | 需修改为 `null` |

**一致性原则：所有原子层模块应使用相同的默认值模式。**

## 参考资料

