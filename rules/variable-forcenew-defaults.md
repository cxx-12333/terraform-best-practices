# ForceNew 参数：空值默认规则

**优先级：** 高
**分类：** 变量与输出模式

## 为什么重要

ForceNew 参数在变更时会触发资源重建。如果默认值非空（如 `"OFF"`），会被显式传递给 API，当 API 不返回该字段或返回不同值时，可能导致状态漂移。

## 问题：ForceNew 默认值导致状态漂移

### 错误代码

```hcl
# atomic/polardb/variables.tf
variable "strict_consistency" {
  description = "PolarDB 强一致性模式"
  type        = string
  default     = "OFF"  # ❌ 非空默认值导致状态漂移
}

# control/database-cluster/main.tf
strict_consistency = try(each.value.strict_consistency, "OFF")  # ❌ 默认值 "OFF"
```

**后果：**

1. 资源创建时 `strict_consistency = "OFF"`
2. API 在值为 "OFF" 时不返回该字段
3. State 记录 `strict_consistency = ""`（空）
4. 下次 plan：代码传递 "OFF"，state 为 ""
5. **ForceNew 触发资源重建！**

### 正确代码

```hcl
# atomic/polardb/variables.tf
variable "strict_consistency" {
  description = "PolarDB 强一致性模式。空 = 不设置，ON = 强一致性读"
  type        = string
  default     = ""  # ✅ 空 = 不设置

  validation {
    condition     = var.strict_consistency == "" || contains(["ON", "OFF"], var.strict_consistency)
    error_message = "strict_consistency 必须为空、ON 或 OFF。"
  }
}

# control/database-cluster/main.tf
strict_consistency = try(each.value.strict_consistency, "")  # ✅ 空默认值

# declarative/simple/04-database/terraform.tfvars
strict_consistency = ""  # ✅ 空 = 不设置，避免触发 ForceNew
```

## ForceNew 参数处理规则

| 场景 | 默认值 | 行为 |
|------|--------|------|
| ForceNew 参数 | ""（空） | ✅ 安全，仅明确设置时传递值 |
| ForceNew 参数 | "OFF" 或任何值 | ❌ 有状态漂移和重建风险 |

## 状态漂移流程

```
创建时：
  代码: strict_consistency = "" → API 不设置 → State = ""

Plan 时：
  代码: strict_consistency = "" → State = "" → 无漂移 ✓

如果默认值是 "OFF"：
创建时：
  代码: strict_consistency = "" → API 不设置 → State = ""
Plan 时：
  代码: strict_consistency = "OFF" → State = "" → ForceNew 触发重建 ❌
```

## 阿里云常见 ForceNew 参数

| 资源 | 参数 | ForceNew |
|------|------|----------|
| alicloud_polardb_cluster | strict_consistency | 是 |
| alicloud_polardb_cluster | db_type | 是 |
| alicloud_polardb_cluster | db_version | 是 |
| alicloud_db_instance | engine | 是 |
| alicloud_db_instance | engine_version | 是 |
| alicloud_instance | image_id | 是 |
| alicloud_kvstore_instance | engine_version | 是 |

## 决策规则

> **ForceNew 参数必须使用空默认值（""、[]、{}），确保仅在用户明确配置时传递，避免意外触发资源重建。**

## 检查清单

- [ ] 识别资源中所有 ForceNew 参数
- [ ] 设置默认值为空（字符串用 ""，列表用 []，map 用 {}）
- [ ] 如适用，添加验证规则
- [ ] 在描述中说明空值 = 不设置
- [ ] 运行 `terraform plan` 验证无 `# forces replacement`

## 参考资料

- `variable-atomic-defaults.md`（原子层默认值原则）
- `resource-lifecycle.md`（生命周期块）
- `provider-optional-api-mandatory.md`（Provider 参数 Schema 类型）
