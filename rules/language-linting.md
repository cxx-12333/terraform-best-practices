# 使用格式化和 Lint 工具

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

一致的格式化提高可读性并减少合并冲突。Lint 在错误进入生产环境之前捕获常见问题。在 CI/CD 和 pre-commit 钩子中自动化这些检查。

## 错误示例

```hcl
# 不一致的格式，无 Lint
resource "aws_instance" "web" {
  ami = var.ami_id
  instance_type="t3.micro"
  tags={Name="web"}
}

# 无 pre-commit 钩子
# 无 CI 检查
# 错误在生产环境才发现
```

## 正确示例

```bash
# 每次提交前运行格式化和 Lint
terraform fmt -recursive
tflint --recursive
terraform validate
```

## terraform fmt

每次提交前运行 `terraform fmt` 以确保格式一致。

```bash
# 格式化当前目录
terraform fmt

# 递归格式化
terraform fmt -recursive

# 仅检查格式，不修改文件（适用于 CI）
terraform fmt -check -recursive

# 显示变更差异
terraform fmt -diff
```

### Pre-commit 钩子

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.5
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
```

安装并运行：

```bash
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

## terraform validate

验证配置语法和内部一致性：

```bash
terraform init -backend=false
terraform validate
```

### CI 流水线

```yaml
# .github/workflows/terraform.yml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform 格式检查
        run: terraform fmt -check -recursive

      - name: Terraform 初始化
        run: terraform init -backend=false

      - name: Terraform 验证
        run: terraform validate
```

## tflint

TFLint 捕获 `terraform validate` 遗漏的错误：

```bash
# 安装（多平台）
# macOS
brew install tflint
# Linux
curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
# Windows
choco install tflint

# 运行
tflint --init
tflint
```

### 配置

```hcl
# .tflint.hcl
config {
  plugin_dir = "~/.tflint.d/plugins"

  # 启用模块检查
  module = true
}

# AWS 专用规则
plugin "aws" {
  enabled = true
  version = "0.27.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

# 强制命名规范
rule "terraform_naming_convention" {
  enabled = true
}

# 要求变量描述
rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}

# 要求类型声明
rule "terraform_typed_variables" {
  enabled = true
}
```

### 常用 tflint 规则

```hcl
# 捕获无效实例类型
rule "aws_instance_invalid_type" {
  enabled = true
}

# 警告已弃用的资源
rule "terraform_deprecated_interpolation" {
  enabled = true
}

# 强制标准模块结构
rule "terraform_standard_module_structure" {
  enabled = true
}
```

## .editorconfig

确保跨编辑器的空白字符一致：

```ini
# .editorconfig
root = true

[*]
indent_style = space
indent_size = 2
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.tf]
indent_size = 2

[*.md]
trim_trailing_whitespace = false

[Makefile]
indent_style = tab
```

## 完整 CI 工作流

```yaml
name: Terraform CI

on:
  pull_request:
    paths:
      - '**.tf'
      - '**.tfvars'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.6.0

      - name: Terraform 格式化
        run: terraform fmt -check -recursive -diff

      - name: 安装 TFLint
        uses: terraform-linters/setup-tflint@v4

      - name: 初始化 TFLint
        run: tflint --init

      - name: 运行 TFLint
        run: tflint --recursive

      - name: Terraform 初始化
        run: terraform init -backend=false

      - name: Terraform 验证
        run: terraform validate
```

## 本地开发 Makefile

```makefile
.PHONY: fmt lint validate

fmt:
  terraform fmt -recursive

lint: fmt
  tflint --recursive

validate: lint
  terraform init -backend=false
  terraform validate

check: validate
  @echo "All checks passed!"
```

## 参考资料

- [terraform fmt](https://developer.hashicorp.com/terraform/cli/commands/fmt)
- [terraform validate](https://developer.hashicorp.com/terraform/cli/commands/validate)
- [TFLint](https://github.com/terraform-linters/tflint)
- [pre-commit-terraform](https://github.com/antonbabenko/pre-commit-terraform)
