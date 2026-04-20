# 声明层纯配置：禁止 resource 块

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

声明层的唯一职责是"声明我要什么"——通过 `module` 调用和 `data` 源组合控制层模块。在声明层直接创建资源（`resource` 块）违反了三层架构的职责边界，导致：
1. 声明层变成"半个控制层"，职责混乱
2. 资源创建逻辑散落在声明层和控制层两处，难以维护
3. 跨环境复用时，声明层的 resource 块可能产生副作用

## 层级允许的块类型

| 块类型 | 声明层 | 控制层 | 原子层 |
|--------|--------|--------|--------|
| `module` | ✅ 调用控制层 | ✅ 调用原子层 | ❌ |
| `data` | ✅ 数据源声明 | ✅ | ✅ |
| `resource` | ❌ **禁止** | ✅ 仅限原子模块内部 | ✅ |
| `locals` | ✅ 简单计算 | ✅ | ✅ |
| `terraform` | ✅ backend/provider | ✅ | ✅ |

## 问题：声明层包含 resource 块

```hcl
# ❌ 错误：声明层直接创建 SSL 证书资源
# declarative/simple/06-web/main.tf

resource "tls_private_key" "test" {               # 违反！
  count     = var.ssl_create_self_signed ? 1 : 0
  algorithm = "RSA"
}

resource "alicloud_ssl_certificates_service_certificate" "test" {  # 违反！
  count = var.ssl_create_self_signed ? 1 : 0
  cert  = tls_self_signed_cert.test[0].cert_pem
  key   = tls_private_key.test[0].private_key_pem
}

module "web" {
  source = "../../../control/web-cluster"
  depends_on = [alicloud_ssl_certificates_service_certificate.test]  # 依赖本层资源
}
```

**问题：**
1. 声明层变成了"资源创建者"，不再是"纯配置"
2. `depends_on` 引用本层资源，破坏了声明层→控制层的单向依赖
3. 多个声明层环境（simple/test33）需要复制相同的 resource 块

## 正确：声明层只传参数，控制层内部处理

```hcl
# ✅ 正确：声明层只传参数
# declarative/simple/06-web/main.tf

module "web" {
  source = "../../../control/web-cluster"

  # SSL 证书参数 — 控制层内部决定如何创建
  ssl_create_self_signed = var.ssl_create_self_signed
  ssl_cert_name          = var.ssl_cert_name
  ssl_cert_id            = var.ssl_cert_id
}
```

```hcl
# ✅ 正确：控制层通过原子模块创建资源
# control/web-cluster/main.tf

module "ssl_self_signed" {
  source = "../../atomic/ssl-self-signed-cert"
  create = var.ssl_create_self_signed
  env    = var.env
}
```

## 判断标准

**"这段代码属于声明层吗？"**
- 如果是 `resource` 块 → **不属于**，移到控制层或原子层
- 如果是 `module` 调用 → ✅ 属于
- 如果是 `data` 源 → ✅ 属于
- 如果是 `locals`（仅做简单值传递） → ✅ 属于
- 如果是 `locals`（做复杂的业务计算/ID解析） → **不属于**，移到控制层

## 例外情况

| 场景 | 是否允许 resource | 说明 |
|------|-------------------|------|
| `terraform_remote_state` | 这是 `data` 块，不是 resource | ✅ 允许 |
| 一次性导入工具（import/） | 临时方案，标注后可接受 | ⚠️ 临时豁免 |
| `tls_provider` 相关测试资源 | 迁移到控制层的原子模块 | ❌ 已有正确方案 |

## 检查清单

- [ ] 声明层 main.tf 中无 `resource` 块
- [ ] 声明层 module 调用无 `depends_on` 引用本层资源
- [ ] 所有资源创建逻辑在控制层或原子层
- [ ] 声明层 locals 只做简单值传递，不做复杂的 ID 解析或业务计算

## 参考资料

- `layer-separation.md`（三层职责划分）
- `declarative-staged-configuration.md`（声明层分阶段配置）
- `declaration-layer-configuration-patterns.md`（声明层配置模式）
