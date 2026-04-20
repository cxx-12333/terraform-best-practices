# 数据源查询结果验证：precondition 防护

**优先级：** 中
**分类：** 资源组织

## 为什么重要

`data` 源查询（如按名称查询 SSL 证书）可能返回空结果。如果不验证查询结果，后续逻辑会静默使用空值，导致资源创建被跳过或配置不完整。用户难以诊断问题——不知道是证书不存在还是配置有误。

## 问题：数据源查询结果未验证

```hcl
# ❌ 错误：查询证书但未验证结果是否为空
data "alicloud_slb_server_certificates" "default" {
  count      = var.ssl_cert_name != "" ? 1 : 0
  name_regex = var.ssl_cert_name
}

# 如果 ssl_cert_name 拼写错误，查询返回空列表
# 但 try(data...[0].certificates[0].id, "") 会静默返回 ""
# 用户不知道证书未找到
locals {
  ssl_cert_id = try(data.alicloud_slb_server_certificates.default[0].certificates[0].id, "")
}
```

**问题：**
1. `name_regex` 拼写错误 → 查询返回空 → 证书 ID 为空
2. HTTPS 监听器因证书为空被跳过创建 → 用户以为创建了实际没有
3. 无明确的错误信息指导用户修复

## 正确：使用 precondition 或 lifecycle 验证

### 方案 1：precondition 验证（Terraform 1.2+）

```hcl
# ✅ 正确：使用 precondition 验证查询结果
data "alicloud_slb_server_certificates" "default" {
  count      = var.ssl_cert_name != "" ? 1 : 0
  name_regex = var.ssl_cert_name

  # 验证查询结果非空
  lifecycle {
    precondition {
      condition     = length(self.certificates) > 0
      error_message = "未找到名称匹配 '${var.ssl_cert_name}' 的 SSL 证书。请检查 ssl_cert_name 是否正确，或改用 ssl_cert_id 直接指定证书 ID"
    }
  }
}
```

### 方案 2：locals 中显式检查 + 明确错误信息

```hcl
# ✅ 正确：在 locals 中检查并提供诊断信息
locals {
  ssl_cert_found = var.ssl_cert_name != "" && length(try(data.alicloud_slb_server_certificates.default[0].certificates, [])) > 0

  ssl_cert_id = var.ssl_cert_id != "" ? var.ssl_cert_id : (
    local.ssl_cert_found ? data.alicloud_slb_server_certificates.default[0].certificates[0].id : ""
  )
}
```

### 方案 3：变量 validation 防护互斥配置

```hcl
# ✅ 正确：验证 SSL 证书配置互斥性
variable "ssl_create_self_signed" {
  type    = bool
  default = false
}

# 在声明层 variables.tf 中添加验证
variable "ssl_cert_name" {
  type    = string
  default = ""
  validation {
    condition = !(var.ssl_cert_name != "" && var.ssl_create_self_signed)
    error_message = "不能同时指定 ssl_cert_name 和 ssl_create_self_signed=true。请选择一种证书获取方式"
  }
}
```

## 常见需要验证的数据源

| 数据源 | 风险 | 验证方式 |
|--------|------|----------|
| `alicloud_slb_server_certificates` | 证书名称不存在 | precondition 检查 `length(certificates) > 0` |
| `alicloud_images` | 镜像 ID 不存在 | precondition 检查 `length(images) > 0` |
| `alicloud_zones` | 可用区 key 不存在 | locals 中 `can(az_map[key])` 检查 |
| `alicloud_account` | 账号信息获取失败 | 通常不会失败，无需特殊验证 |

## 输出诊断信息

```hcl
# 在控制层输出诊断信息，帮助用户排查问题
output "ssl_cert_diagnostic" {
  description = "SSL 证书诊断信息"
  value = {
    source = var.ssl_cert_id != "" ? "manual_id" : (
      var.ssl_cert_name != "" ? "name_query" : (
        var.ssl_create_self_signed ? "self_signed" : "none"
      )
    )
    cert_found = local.ssl_cert_id_resolved != ""
  }
}
```

## 检查清单

- [ ] 按名称查询的数据源使用 `precondition` 验证结果非空
- [ ] `error_message` 包含具体的修复建议（如"请检查 xxx 是否正确"）
- [ ] 变量 validation 检查互斥配置（如不能同时指定 cert_id 和 cert_name）
- [ ] 输出诊断信息帮助排查证书/查询问题
- [ ] `try()` 包裹的数据源访问不会静默吞掉"未找到"的情况

## 参考资料

- `language-data-sources.md`（数据源使用规范）
- `cross-stage-reference-validation.md`（跨阶段引用验证）
- `declarative-staged-configuration.md`（声明层分阶段配置）
