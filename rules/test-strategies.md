# 建立多层次测试策略

**优先级：** 中
**分类：** 测试与验证

## 为什么重要

测试在错误进入生产环境之前将其捕获。不同的测试策略验证 Terraform 代码的不同方面，从语法到实际基础设施行为。

## 错误示例

```bash
# 无测试 - 在生产环境才发现错误
vim main.tf
terraform apply -auto-approve
# 祈祷能正常工作！

# 部署后才发现的问题：
# - 语法错误
# - 安全配置错误
# - 缺少资源
# - 配置不正确
```

## 正确示例

```bash
# 多层次测试策略
terraform fmt -check       # 格式检查
terraform validate         # 语法验证
tflint --recursive         # 静态分析
tfsec .                    # 安全扫描
terraform plan -out=tfplan # Plan 审查
conftest test tfplan.json  # 策略检查
terraform test             # 原生测试（1.6+）
```

## 测试金字塔

```
       ┌─────────────┐
       │  集成测试    │ 最慢，最有信心
       └─────────────┘
      ┌───────────────┐
      │  Plan 测试    │
      └───────────────┘
     ┌─────────────────┐
     │  静态分析       │
     └─────────────────┘
    ┌───────────────────┐
    │  格式化与验证     │ 最快，基础检查
    └───────────────────┘
```

## 第一层：格式化和验证

```bash
# 格式检查（快速，捕获风格问题）
terraform fmt -check -recursive -diff

# 验证语法和配置
terraform init -backend=false
terraform validate
```

### CI 流水线

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: 格式检查
        run: terraform fmt -check -recursive

      - name: 验证
        run: |
          terraform init -backend=false
          terraform validate
```

## 第二层：静态分析

### tflint

```hcl
# .tflint.hcl
config {
  plugin_dir = "~/.tflint.d/plugins"
}

plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
}

rule "terraform_documented_variables" {
  enabled = true
}
```

```bash
tflint --init
tflint --recursive
```

### 安全扫描

```bash
# tfsec - 安全专用
tfsec .

# checkov - 合规和安全
checkov -d .

# trivy - 漏洞扫描
trivy config .
```

### 示例输出

```bash
$ tfsec .
结果: CRITICAL - 安全组规则允许所有流量
资源: aws_security_group_rule.bad_rule
位置: main.tf:15

$ checkov -d .
通过检查: 45
失败检查: 3
- CKV_AWS_23: "确保每个安全组规则都有描述"
- CKV_AWS_24: "确保没有安全组允许从 0.0.0.0/0 到端口 22 的入站"
```

## 第三层：Plan 测试

### Terraform Plan 分析

```bash
# 生成 Plan
terraform plan -out=tfplan

# 转换为 JSON 进行分析
terraform show -json tfplan > tfplan.json

# 使用自定义脚本或工具分析
```

### OPA/Conftest 验证 Plan

```rego
# policy/terraform.rego
package main

# 拒绝缺少必需标签的资源
deny[msg] {
  resource := input.resource_changes[_]
  not resource.change.after.tags.Environment
  msg := sprintf("资源 %s 缺少 'Environment' 标签", [resource.address])
}

# 拒绝过于宽松的安全组
deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group_rule"
  resource.change.after.cidr_blocks[_] == "0.0.0.0/0"
  resource.change.after.from_port == 22
  msg := sprintf("安全组 %s 允许从任意位置 SSH", [resource.address])
}
```

```bash
conftest test tfplan.json -p policy/
```

### Terraform Test（原生）

```hcl
# tests/basic.tftest.hcl
run "verify_vpc_created" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR 块不正确"
  }
}

run "verify_tags" {
  command = plan

  assert {
    condition     = aws_vpc.main.tags["Environment"] == var.environment
    error_message = "Environment 标签设置不正确"
  }
}
```

```bash
terraform test
```

## 测试框架决策矩阵

根据需求选择合适的测试方法：

| 标准 | 原生测试（Terraform 1.6+） | Terratest（Go） |
|------|---------------------------|-----------------|
| **语言** | HCL | Go |
| **配置复杂度** | 低（内置） | 中（Go 项目配置） |
| **测试速度** | 快（不创建真实资源） | 慢（创建真实资源） |
| **成本** | 免费（仅 Plan） | 花费（真实资源） |
| **适用场景** | Plan 验证、输出检查 | 集成测试、多步骤工作流 |
| **何时使用** | 简单断言、Plan 验证 | 复杂场景、跨资源验证 |

### 决策流程

```
起点：需要测试 Terraform 代码？
|
├─ 可以通过 Plan 输出测试？（不需要真实资源）
|   |
|   ├─ 是 → 使用原生测试（terraform test）
|   |       └─ 快速、免费、内置
|   |
|   └─ 否 → 需要真实资源？
|       |
|       ├─ 是 → 使用 Terratest
|       |       └─ 创建真实资源，验证行为
|       |
|       └─ 否 → 使用静态分析
|               └─ tflint、tfsec、checkov
```

## 第四层：集成测试

### Terratest（Go）

```go
// test/vpc_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
  t.Parallel()

  terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
    TerraformDir: "./fixtures/vpc",
    Vars: map[string]interface{}{
      "vpc_cidr":     "10.0.0.0/16",
      "environment":  "test",
    },
  })

  // 测试后清理
  defer terraform.Destroy(t, terraformOptions)

  // 部署基础设施
  terraform.InitAndApply(t, terraformOptions)

  // 验证输出
  vpcId := terraform.Output(t, terraformOptions, "vpc_id")
  assert.NotEmpty(t, vpcId)
}
```

### 测试固件

```hcl
# 使用被测模块的测试固件
module "vpc" {
  source = "../../" # 引用被测模块

  vpc_cidr    = "10.0.0.0/16"
  environment = "test"

  # 测试专用配置
  enable_nat_gateway = false
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

## 按环境划分测试策略

| 阶段 | 测试内容 | 时机 |
|------|----------|------|
| 本地 | 格式化、验证 | 提交前 |
| PR | 静态分析、Plan 测试 | 提交 PR 时 |
| 预发布 | 集成测试 | 合并后 |
| 生产 | 冒烟测试、漂移检测 | 部署后 |

## 测试检查清单

为 Terraform 代码建立测试时使用此清单：

- [ ] 格式检查（`terraform fmt -check`）
- [ ] 语法验证（`terraform validate`）
- [ ] 静态分析（`tflint`）
- [ ] 安全扫描（`tfsec`、`checkov`）
- [ ] Plan 生成（`terraform plan`）
- [ ] 策略检查（`conftest`、OPA）
- [ ] 原生测试（`terraform test`） - Terraform 1.6+
- [ ] 集成测试（Terratest） - 如需要
- [ ] CI/CD 集成
- [ ] Pre-commit 钩子

## 参考资料

- [Terraform Test](https://developer.hashicorp.com/terraform/language/tests)
- [Terratest](https://terratest.gruntwork.io/)
- [tfsec](https://github.com/aquasecurity/tfsec)
- [Checkov](https://www.checkov.io/)
- [Conftest](https://www.conftest.dev/)
- [测试最佳实践](https://developer.hashicorp.com/terraform/language/tests/best-practices)
