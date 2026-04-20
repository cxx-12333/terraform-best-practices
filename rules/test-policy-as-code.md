# 使用策略即代码

**优先级：** 低
**分类：** 测试与验证

## 为什么重要

策略即代码自动执行组织标准，在部署前捕获安全违规和合规问题。它将安全左移并扩展治理能力。

## 错误示例

```hcl
# 无策略强制 - 安全问题进入生产环境
resource "aws_s3_bucket" "data" {
  bucket = "my-data"
  # 无加密 - 违反策略
  # 无日志 - 违反合规
  # 数月后在安全审计中才发现
}

resource "aws_security_group_rule" "ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  cidr_blocks = ["0.0.0.0/0"] # 对全世界开放 - 安全风险
}
```

## 正确示例

```bash
# CI/CD 中自动执行策略检查
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# 在 apply 前运行策略检查
conftest test tfplan.json -p policy/
checkov -d .
tfsec .
```

## 工具概览

| 工具 | 用途 | 集成方式 |
|------|------|----------|
| Sentinel | HashiCorp 企业版 | Terraform Cloud/Enterprise |
| OPA/Conftest | 开源 | CI/CD、本地 |
| Checkov | 安全扫描 | CI/CD、本地 |
| tfsec | 安全扫描 | CI/CD、本地 |
| Terramate | 堆栈策略 | CI/CD、本地 |

## OPA/Conftest 示例

```rego
# policy/terraform.rego
package main

# 拒绝未启用加密的 S3 存储桶
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_s3_bucket"
  resource.change.after.server_side_encryption_configuration == null

  msg := sprintf("S3 存储桶 '%s' 必须启用加密", [resource.name])
}

# 拒绝公开安全组
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
  resource.change.after.type == "ingress"

  msg := sprintf("安全组规则 '%s' 允许来自 0.0.0.0/0 的入站流量", [resource.name])
}

# 要求标签
deny[msg] {
  resource := input.resource_changes[_]
  required_tags := {"Environment", "Owner", "Project"}
  provided_tags := {tag | resource.change.after.tags[tag]}
  missing := required_tags - provided_tags
  count(missing) > 0

  msg := sprintf("资源 '%s' 缺少必需标签: %v", [resource.name, missing])
}
```

### 运行 Conftest

```bash
# 生成 Plan JSON
terraform plan -out=tfplan
terraform show -json tfplan > tfplan.json

# 运行策略检查
conftest test tfplan.json -p policy/
```

## Checkov 示例

```yaml
# .checkov.yaml
framework:
  - terraform

check:
  - CKV_AWS_18 # S3 存储桶日志
  - CKV_AWS_19 # S3 存储桶加密
  - CKV_AWS_21 # S3 存储桶版本控制

skip-check:
  - CKV_AWS_144 # 跳过跨区域复制（不需要）

soft-fail-on:
  - CKV_AWS_33 # KMS 轮换仅警告不失败
```

```bash
# 运行 Checkov
checkov -d . --config-file .checkov.yaml
```

## tfsec 示例

```yaml
# .tfsec/config.yml
severity_overrides:
  AWS002: ERROR   # S3 存储桶无日志
  AWS017: WARNING # S3 存储桶无版本控制

exclude:
  - AWS089 # CloudWatch 日志组加密
```

```bash
# 运行 tfsec
tfsec . --config-file .tfsec/config.yml
```

## CI/CD 集成

```yaml
# GitHub Actions
name: Terraform 策略检查

on: [pull_request]

jobs:
  policy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 安装 Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Plan
        run: |
          terraform init
          terraform plan -out=tfplan
          terraform show -json tfplan > tfplan.json

      - name: 运行 Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .

      - name: 运行 Conftest
        uses: instrumenta/conftest-action@master
        with:
          files: tfplan.json
          policy: policy/
```

## 参考资料

- [OPA Terraform](https://www.openpolicyagent.org/docs/latest/terraform/)
- [Checkov](https://www.checkov.io/)
- [tfsec](https://aquasecurity.github.io/tfsec/)
- [Sentinel](https://developer.hashicorp.com/sentinel)
