# Provider API 返回值格式不一致导致状态漂移

**优先级：** 高
**分类：** 常见陷阱

## 为什么重要

云厂商 Provider 的 API 可能存在**输入格式与返回格式不一致**的问题，导致 Terraform 状态漂移，触发 ForceNew 重建资源。

### 常见场景

| 云厂商 | 资源/字段 | 问题描述 |
|--------|----------|----------|
| 阿里云 | Kafka zone_id | 输入 `cn-hangzhou-j`，API 返回 `zonej` |
| 阿里云 | Kafka SASL mechanism | 输入 `PLAIN`，API 返回 `plain`（小写） |
| 阿里云 | Kafka SASL type | 输入 `user`，API 返回格式可能不一致 |
| 阿里云 | Kafka ACL acl_resource_pattern_type | 必须大写 `LITERAL`，小写会报错 |
| 阿里云 | Redis zone_id | 可能存在类似问题 |
| AWS | 某些 ARN 字段 | 输入简写，返回完整 ARN |
| 腾讯云 | CKafka zone | 输入 `ap-guangzhou-3`，API 可能返回不同格式 |
| 腾讯云 | Redis zone | 可用区格式可能不一致 |
| 通用 | 时间格式 | 输入 `2024-01-01`，返回 `2024-01-01T00:00:00Z` |

## 问题模式：API 返回值 ≠ 输入值

### 现象

```
Terraform 检测到变更：
~ field_name = "返回值" -> "输入值" # forces replacement
```

### 根因分析

```
输入配置：field_name = "格式A"
↓
API 接受多种格式，正常创建
↓
API 返回：field_name = "格式B" ← 不同格式！
↓
Terraform 对比：
配置值 "格式A" ≠ State 值 "格式B"
↓
触发 ForceNew，资源被重建！
```

### Provider 层面的问题

```go
// Provider 直接使用 API 返回值，未做格式标准化
d.Set("field_name", object["FieldName"]) // 直接赋值
```

## 解决方案一：不传该字段（推荐）

### 原则

> **如果字段的值可从其他参数自动推断，不要显式传递，消除格式差异风险。**

### 示例：Kafka zone_id

```hcl
# 正确：不传 zone_id
resource "alicloud_alikafka_instance" "this" {
  vswitch_id = var.vswitch_id # vswitch 已绑定 zone
  # zone_id 不传，API 自动推断
}

# 错误：显式传递会导致漂移
resource "alicloud_alikafka_instance" "this" {
  vswitch_id = var.vswitch_id
  zone_id = "cn-hangzhou-j" # API 返回 "zonej"，触发漂移
}
```

### 其他适用场景

| 场景 | 替代方案 |
|------|----------|
| zone_id 可从 vswitch 推断 | 不传 zone_id |
| region 可从 provider 推断 | 不传 region |
| account_id 可从 provider 推断 | 不传 account_id |

## 解决方案二：ignore_changes

如果必须保留字段参数：

```hcl
resource "alicloud_xxx_instance" "this" {
  field_name = var.field_value

  lifecycle {
    ignore_changes = [field_name] # 忽略格式差异
  }
}
```

## 解决方案三：使用 API 返回格式

确保输入格式与 API 返回格式一致：

```hcl
# 如果 API 返回简化格式，输入也用简化格式
variable "az_map" {
  default = {
    j = "zonej" # 使用 API 返回的格式
    k = "zonek"
  }
}
```

## 阿里云 zone_id 格式不一致详解

### 问题确认

```
阿里云不同产品 API 返回格式：

VPC/VSwitch API：
输入：cn-hangzhou-j
返回：cn-hangzhou-j 一致

Kafka API：
输入：cn-hangzhou-j
返回：zonej 简化格式，触发漂移！
```

### 源码证据

```go
// terraform-provider-alicloud/alicloud/resource_alicloud_alikafka_instance.go
d.Set("zone_id", object["ZoneId"]) // 直接使用，无格式转换
```

### 修复方案

```hcl
# 控制层 - 不传 zone_id
module "kafka" {
  vswitch_id = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]
  # zone_id 不传，消除漂移
}
```

## 检查清单

- [ ] 识别资源中可能存在格式差异的字段
- [ ] 检查该字段是否可从其他参数推断
- [ ] 优先采用"不传该字段"方案
- [ ] 运行 `terraform plan` 验证无漂移
- [ ] 如果必须传，考虑 `ignore_changes`

## 决策规则

> **当 Provider API 返回格式与输入格式可能不一致时，优先让 API 自动推断，避免显式传递导致状态漂移。**

## 多云 lifecycle ignore_changes 示例

各云厂商均可使用 lifecycle ignore_changes 处理格式漂移：

```hcl
# 阿里云 — Kafka zone_id 格式漂移
lifecycle {
  ignore_changes = [zone_id]
}

# AWS — 某些 ARN 字段格式差异
lifecycle {
  ignore_changes = [arn]
}

# 腾讯云 — 可用区格式不一致
lifecycle {
  ignore_changes = [zone_id]
}
```

**注意**：`ignore_changes` 是最后手段。优先使用 `state_analysis-vs-rebuild` 中描述的配置对齐方案。

## 参考资料

- `resource-typeset-computed-drift.md`（TypeSet 计算漂移）
- `resource-lifecycle.md`（生命周期块）
- `provider-documentation-lookup.md`（Provider 文档查找规范）
