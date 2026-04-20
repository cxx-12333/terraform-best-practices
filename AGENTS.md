# Terraform 最佳实践 - 完整参考指南

Terraform 与基础设施即代码综合优化指南（多云通用）。包含 81 条规则，分为 12 个类别。多云示例覆盖 AWS、Azure、GCP、阿里云、腾讯云。

> **说明：** 本文档由 `rules/` 目录下的各规则文件合并生成。每条规则在独立文件中维护以便更新。最新版本请参考 `rules/<rule-name>.md`。

---


---

# 所有 Terraform 代码纳入版本控制

**优先级：** 关键
**分类：** 组织与工作流

## 为什么重要

版本控制提供完整的基础设施变更历史，支持团队协作、代码审查和回滚到先前状态。所有 Terraform 代码都应纳入版本控制。

## 错误示例

```bash
# Terraform 代码存储在本地或共享驱动器
/shared-drive/terraform/
├── main.tf
├── main-backup.tf
├── main-old.tf
└── main-DONT-USE.tf

# 通过邮件或聊天工具分享代码
# 没有谁在何时改了什么的变更记录
```

**问题：**
- 没有变更审计记录
- 无法回滚
- 多人同时编辑时产生冲突
- 没有代码审查流程

## 正确示例

### 所有 Terraform 代码使用 Git

```bash
# 每个配置都纳入版本控制
git init
git add .
git commit -m "Initial infrastructure configuration"
git push origin main
```

### 仓库组织方式

组织 Terraform 仓库有多种有效方案：

- **单体仓库（Monorepo）** - 所有基础设施放在一个仓库
- **多仓库（Polyrepo）** - 每个组件或团队使用独立仓库
- **混合方式（Hybrid）** - 共享模块在一个仓库，配置在独立仓库

选择适合组织需求的方案。核心原则：
- 代码有版本且可审计
- 团队可通过代码审查协作
- 变更可回滚

### 分支策略

```bash
# 功能分支工作流
git checkout -b feature/add-cache-layer
# 进行修改
git add .
git commit -m "Add ElastiCache for session storage"
git push origin feature/add-cache-layer
# 创建 Pull Request 进行审查
```

### 保护主分支

配置分支保护规则：
- 合并前需要 Pull Request 审查
- 需要 CI/CD 状态检查通过
- 需要解决所有讨论
- 禁止强制推送

### 提交信息规范

```bash
# 好的提交信息
git commit -m "Add RDS read replica for reporting queries

- Creates read replica in us-east-1b
- Configures security group for app servers
- Updates outputs for connection string

Refs: INFRA-1234"

# 不好的提交信息
git commit -m "updates"
git commit -m "fix"
git commit -m "wip"
```

## Terraform .gitignore 配置

```gitignore
# 本地 .terraform 目录
**/.terraform/*

# .tfstate 文件
*.tfstate
*.tfstate.*

# 崩溃日志
crash.log
crash.*.log

# 排除 override 文件
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# 排除 CLI 配置文件
.terraformrc
terraform.rc

# 排除敏感变量文件
*.tfvars
*.tfvars.json
!example.tfvars

# 锁文件应该提交
# 不要将 .terraform.lock.hcl 加入 .gitignore
```

## 锁文件管理

```bash
# 提交锁文件以确保可复现性
git add .terraform.lock.hcl
git commit -m "Update provider lock file"

# 更新 Provider 时
terraform init -upgrade
git add .terraform.lock.hcl
git commit -m "Upgrade AWS provider to 5.32.0"
```

## 参考资料

- [版本控制最佳实践](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part3.2)
- [Git 工作流策略](https://www.atlassian.com/git/tutorials/comparing-workflows)

---

# 每个环境每个配置使用独立工作区

**优先级：** 高
**分类：** 组织与工作流

## 为什么重要

将基础设施组织为独立工作区可以实现权限委派、控制爆炸半径并支持环境晋升。每个工作区应代表一个环境中的一个基础设施组件。

## 公式

```
Terraform 配置数 × 环境数 = 工作区数
```

## 错误示例

```hcl
# 一个巨型工作区管理所有内容
# terraform-monolith/
# - VPC
# - EKS 集群
# - RDS 数据库
# - ElastiCache
# - Lambda 函数
# - S3 存储桶
# - IAM 角色
# 所有环境共用一个状态文件
```

**问题：**
- 单点故障
- plan/apply 执行时间长
- 无法委派所有权
- 意外销毁的风险
- 状态文件冲突

## 正确示例

### 按组件和环境拆分

```bash
# 工作区命名：{组件}-{环境}
networking-dev
networking-staging
networking-prod

eks-cluster-dev
eks-cluster-staging
eks-cluster-prod

database-dev
database-staging
database-prod

app-frontend-dev
app-frontend-staging
app-frontend-prod
```

### 相同代码，不同变量

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "环境名称（dev、staging、prod）"
}

variable "instance_count" {
  type        = number
  description = "实例数量"
}

# main.tf - 所有环境使用相同代码
resource "aws_instance" "app" {
  count         = var.instance_count
  instance_type = var.environment == "prod" ? "m5.large" : "t3.micro"

  tags = {
    Environment = var.environment
  }
}
```

```hcl
# dev.tfvars
environment    = "dev"
instance_count = 1

# prod.tfvars
environment    = "prod"
instance_count = 3
```

### 通过远程状态实现工作区依赖

```hcl
# 在 eks-cluster 配置中
# 读取 networking 工作区的输出
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "terraform-state"
    key    = "networking-${var.environment}/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_eks_cluster" "main" {
  name = "${var.project}-${var.environment}"

  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

### 每个工作区使用独立状态后端

```hcl
# 每个工作区使用不同的状态 key
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "networking-dev/terraform.tfstate" # 每个工作区不同
    region = "us-east-1"
  }
}
```

或使用工作区功能：

```bash
# Terraform 原生工作区
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace select dev
terraform apply -var-file=dev.tfvars
```

## 工作区规模指南

| 规模 | 资源数量 | 适用场景 |
|------|----------|----------|
| 过小 | 1-3 个 | 管理开销不值得 |
| 合适 | 10-50 个 | 可管理，权责清晰 |
| 过大 | 200+ 个 | 应拆分为组件 |

## 优点

1. **爆炸半径可控** - 错误只影响一个组件/环境
2. **权限委派** - 不同团队负责不同工作区
3. **并行操作** - 团队独立工作互不干扰
4. **环境晋升** - 变更流程 dev → staging → prod
5. **审计追踪** - 每个组件有清晰的变更历史

## 参考资料

- [工作区结构](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part1#the-recommended-terraform-workspace-structure)
- [远程状态](https://developer.hashicorp.com/terraform/language/state/remote-state-data)

---

# 控制谁可以更改什么基础设施

**优先级：** 高
**分类：** 组织与工作流

## 为什么重要

并非所有人都应该能够修改所有基础设施。访问控制确保变更由授权人员执行，保护生产环境，并建立问责机制。

## 错误示例

```bash
# 每个人对所有资源都有管理员权限
# 共享的 AWS 凭证放在 wiki 上
# 开发和生产环境的访问权限没有区分
# 任何人都可以在生产环境运行 terraform apply
```

**问题：**
- 变更无问责
- 意外修改生产环境
- 没有职责分离
- 合规违规

## 正确示例

### 最小权限原则

为每个角色授予最低必要权限：

| 角色 | 开发环境 | 预发布环境 | 生产环境 |
|------|----------|-----------|----------|
| 初级工程师 | Apply | 仅 Plan | 只读 |
| 高级工程师 | Apply | Apply | Plan + 审查 |
| 技术负责人 | Admin | Admin | Apply |
| 平台团队 | Admin | Admin | Admin |

### 每个环境使用独立凭证

```hcl
# 开发账号
provider "aws" {
  alias   = "dev"
  region  = "us-east-1"
  profile = "company-dev" # 有限权限
}

# 生产账号
provider "aws" {
  alias   = "prod"
  region  = "us-east-1"
  profile = "company-prod" # 需要 MFA，更严格的控制
}
```

### 使用 IAM 角色，而非长期密钥

```hcl
# CI/CD 通过角色假设获取有限权限
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::123456789:role/TerraformDeployRole"
    session_name = "terraform-ci"
  }
}
```

### 环境级 IAM 策略

```hcl
# 开发环境角色 - 较宽泛的权限
resource "aws_iam_role" "terraform_dev" {
  name = "terraform-dev-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.dev_account_id}:root" }
      Action    = "sts:AssumeRole"
    }]
  })
}

# 生产环境角色 - 受限权限
resource "aws_iam_role" "terraform_prod" {
  name = "terraform-prod-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.prod_account_id}:root" }
      Action    = "sts:AssumeRole"
      Condition = {
        Bool = { "aws:MultiFactorAuthPresent" = "true" } # 要求 MFA
      }
    }]
  })
}
```

### 限制直接控制台/CLI 访问

基础设施由 Terraform 管理后：

1. 移除大多数用户的直接控制台访问权限
2. 使用 Terraform 作为变更的唯一工作流
3. 启用只读控制台访问用于故障排查
4. 审计任何手动变更

```hcl
# 控制台只读访问策略
data "aws_iam_policy_document" "readonly" {
  statement {
    effect = "Allow"
    actions = [
      "ec2:Describe*",
      "rds:Describe*",
      "s3:Get*",
      "s3:List*"
    ]
    resources = ["*"]
  }

  statement {
    effect = "Deny"
    actions = [
      "ec2:*",
      "rds:*",
      "s3:Put*",
      "s3:Delete*"
    ]
    resources = ["*"]
  }
}
```

### Git 分支保护

```yaml
# .github/CODEOWNERS
# 生产环境变更需要平台团队审查
*prod* @platform-team
# 共享模块需要审查
**/modules/** @platform-team @senior-engineers
```

### CI/CD 访问控制

```yaml
# GitHub Actions - 不同环境不同权限
jobs:
  deploy-dev:
    if: github.ref == 'refs/heads/develop'
    environment: dev # 无需审批

  deploy-prod:
    if: github.ref == 'refs/heads/main'
    environment: prod
    # 需要在 GitHub 环境设置中手动审批
```

### 多账号策略

```
组织
├── 管理账号（账单、组织策略）
├── 安全账号（审计日志、安全工具）
├── 共享服务账号（CI/CD、制品存储）
├── 开发账号（开发工作负载）
├── 预发布账号（预发布测试）
└── 生产账号（生产工作负载）
```

## 支持访问控制的后端选项

- **S3 + IAM** - AWS 原生，使用 IAM 策略
- **GCS + IAM** - GCP 原生，使用 IAM 策略
- **Azure Blob + RBAC** - Azure 原生
- **Terraform Cloud/Enterprise** - 内置团队权限
- **Terramate Cloud** - GitOps 原生，栈级权限

## 参考资料

- [AWS 多账号策略](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices.html)
- [HashiCorp 访问控制建议](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part1#personas-responsibilities-and-desired-user-experiences)

---

# 基础设施变更的正式流程

**优先级：** 高
**分类：** 组织与工作流

## 为什么重要

正式的变更工作流可以减少故障、支持回滚、防止冲突并建立审计追踪。基础设施变更应遵循可预测、可审查的流程。

## 错误示例

```bash
# 野路子工作流
cd terraform/
vim main.tf
terraform apply -auto-approve
# 祈祷它能正常工作！
```

**问题：**
- 变更前没有审查
- 没有变更记录
- 无法轻松回滚
- 团队成员之间产生冲突

## 正确示例

### 标准变更工作流

```
1. 创建分支
2. 进行修改
3. 运行 terraform plan
4. 创建 Pull Request
5. 审查 plan 输出
6. 审批并合并
7. 应用变更（自动化或手动）
8. 验证部署
```

### 基于分支的工作流

```bash
# 1. 创建功能分支
git checkout -b feature/add-redis-cache

# 2. 进行修改
vim main.tf

# 3. 格式化和验证
terraform fmt
terraform validate

# 4. 生成 plan
terraform plan -out=tfplan

# 5. 提交并推送
git add .
git commit -m "Add Redis cache for session storage"
git push origin feature/add-redis-cache

# 6. 创建包含 plan 输出的 PR
# 7. 获取审查和审批
# 8. 合并到主分支
# 9. 通过 CI/CD 或手动 apply
```

### Pull Request 模板

```markdown
<!-- .github/pull_request_template.md -->
## 描述
<!-- 本 PR 做了哪些基础设施变更？ -->

## 动机
<!-- 为什么需要这些变更？ -->

## Terraform Plan
<details>
<summary>点击展开 plan 输出</summary>

```
<!-- 在此粘贴 terraform plan 输出 -->
```

</details>

## 检查清单
- [ ] 已运行 `terraform fmt`
- [ ] `terraform validate` 通过
- [ ] 已审查 plan 输出
- [ ] 代码中无密钥
- [ ] 文档已更新（如需要）
- [ ] 已在开发环境测试
```

### 环境晋升

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│  Dev    │───▶│ Staging │───▶│  Prod   │
└─────────┘    └─────────┘    └─────────┘
     │              │              │
  自动 apply    自动 apply     手动审批
     │              │              │
  功能分支     主分支合并    标签发布或审批
```

### Makefile 统一操作

```makefile
.PHONY: init plan apply destroy

ENV ?= dev

init:
	terraform init

plan:
	terraform plan -var-file=$(ENV).tfvars -out=tfplan

apply:
	terraform apply tfplan

destroy:
	terraform destroy -var-file=$(ENV).tfvars

# 用法：make plan ENV=prod
```

### 变更文档

```hcl
# 在代码中记录重要变更
# CHANGELOG: 2024-01-15 - 添加只读副本用于报表
# Ticket: INFRA-1234
# Author: @engineer

resource "aws_db_instance" "read_replica" {
  # ...
}
```

### 回滚策略

```bash
# 方案 1：回退提交
git revert HEAD
git push
# CI/CD 会应用回退后的状态

# 方案 2：应用先前状态
terraform apply -target=aws_instance.web -var="ami_id=ami-previous"

# 方案 3：使用状态恢复
terraform state pull > backup.tfstate
# 需要时从备份恢复
```

## 成熟度四级模型

| 级别 | 实践 |
|------|------|
| 手动 | 通过控制台/CLI 变更，无跟踪 |
| 半自动化 | 部分 IaC，流程不一致 |
| 基础设施即代码 | 所有变更通过 Terraform、VCS、审查 |
| 协作式 IaC | 权限委派、访问控制、自动化晋升 |

## 参考资料

- [HashiCorp 变更工作流](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part2#your-current-change-control-workflow)
- [GitOps 原则](https://opengitops.dev/)

---

# 追踪所有基础设施变更

**优先级：** 中高
**分类：** 组织与工作流

## 为什么重要

审计日志提供问责依据、支持故障排查、满足合规要求，并辅助安全调查。追踪所有基础设施变更及其执行者。

## 错误示例

```bash
# 没有任何日志
# "上周谁改了安全组？"
# "不知道，去问问团队所有人"

# 手动变更日志
# 共享文档但没有人持续更新
```

**问题：**
- 无法确定谁做了变更
- 无法排查问题
- 合规违规
- 安全盲区

## 正确示例

### 第一层：版本控制历史

```bash
# Git 日志显示谁修改了基础设施代码
git log --oneline --all -- '*.tf'

# 查看特定文件的变更
git log -p -- modules/networking/main.tf

# 查找谁引入了特定变更
git blame main.tf
```

### 第二层：云厂商审计日志

```hcl
# AWS CloudTrail
resource "aws_cloudtrail" "main" {
  name                       = "infrastructure-audit"
  s3_bucket_name             = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail      = true
  enable_log_file_validation = true

  event_selector {
    read_write_type     = "All"
    include_management_events = true
  }
}
```

### 第三层：Terraform 状态变更

```hcl
# 在状态存储桶上启用版本控制
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# 记录状态存储桶的访问日志
resource "aws_s3_bucket_logging" "state" {
  bucket        = aws_s3_bucket.terraform_state.id

  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "state-access-logs/"
}
```

### 第四层：CI/CD 流水线日志

```yaml
# GitHub Actions - 自动保存日志
- name: Terraform Apply
  run: terraform apply -auto-approve
  # 输出自动记录在 workflow 运行日志中

# 在日志中包含元数据
- name: Log Deployment Info
  run: |
    echo "Deployer: ${{ github.actor }}"
    echo "Commit: ${{ github.sha }}"
    echo "Ref: ${{ github.ref }}"
    echo "Timestamp: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
```

### 集中日志

```hcl
# 将 CloudTrail 发送到 CloudWatch Logs
resource "aws_cloudtrail" "main" {
  # ...
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail.arn
}

# 为重要事件创建告警
resource "aws_cloudwatch_metric_alarm" "unauthorized_api_calls" {
  alarm_name          = "unauthorized-api-calls"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnauthorizedAPICalls"
  namespace           = "CloudTrailMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "检测到未授权的 API 调用"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

### 日志保留

```hcl
# 按合规要求保留日志
resource "aws_cloudwatch_log_group" "terraform" {
  name              = "/terraform/deployments"
  retention_in_days = 365 # 根据合规要求调整
}

resource "aws_s3_bucket_lifecycle_configuration" "audit_logs" {
  bucket = aws_s3_bucket.audit_logs.id

  rule {
    id     = "archive-old-logs"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 2555 # 7 年，满足合规要求
    }
  }
}
```

### 查询审计日志

```bash
# AWS CloudTrail - 查找谁修改了安全组
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AuthorizeSecurityGroupIngress \
  --start-time 2024-01-01 \
  --end-time 2024-01-31

# 查找所有 Terraform 发起的变更（通过 user agent）
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=ec2.amazonaws.com \
  | jq '.Events[] | select(.CloudTrailEvent | contains("terraform"))'
```

## 应追踪的事件

| 事件 | 来源 | 保留期 |
|------|------|--------|
| 代码变更 | Git | 永久 |
| API 调用 | CloudTrail/Stackdriver | 1-7 年 |
| 状态变更 | S3 版本控制 | 1 年 |
| 流水线运行 | CI/CD 日志 | 90 天 |
| 访问尝试 | CloudTrail | 1 年 |

## 参考资料

- [AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)
- [GCP 审计日志](https://cloud.google.com/logging/docs/audit)
- [Azure 活动日志](https://docs.microsoft.com/en-us/azure/azure-monitor/essentials/activity-log)

---

# 始终使用远程状态后端

**优先级：** 关键
**分类：** 状态管理

## 为什么重要

本地状态文件是单点故障源，且无法支持团队协作。远程后端提供持久化存储、状态锁定和团队共享访问。

## 错误示例

```hcl
# 无后端配置 - 状态存储在本地
terraform {
  required_version = ">= 1.0"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

**问题：** 状态存储在本地 `terraform.tfstate` 文件中。一旦丢失，Terraform 将失去对所有托管资源的跟踪。

## 正确示例

```hcl
terraform {
  required_version = ">= 1.0"

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

**优点：**
- 状态持久化存储在 S3 中
- 通过 DynamoDB 实现状态锁定，防止并发修改
- 静态数据加密保护敏感信息
- 团队成员可以访问共享状态

## 补充说明

常用的远程后端选项：
- **S3** - AWS 原生方案，配合 DynamoDB 实现锁定
- **GCS** - Google Cloud Storage，内置锁定机制
- **Azure Blob** - Azure 原生后端
- **Terraform Cloud** - 托管后端，提供额外功能
- **Terramate Cloud** - GitOps 原生状态管理

## 多云场景适配

### 多云后端配置

```hcl
# AWS: S3 + DynamoDB
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-lock"
    encrypt        = true
  }
}

# Azure: Blob Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "tf-state"
    storage_account_name = "tfstate1234"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# 阿里云: OSS
terraform {
  backend "oss" {
    bucket = "my-tfstate"
    prefix = "prod"
    key    = "terraform.tfstate"
    region = "cn-hangzhou"
  }
}

# 腾讯云: COS
terraform {
  backend "cos" {
    region = "ap-guangzhou"
    bucket = "my-tfstate-1250000000"
    prefix = "terraform/state"
  }
}
```

### 后端对比

| 云厂商 | 后端类型 | 存储服务 | 锁定机制 | 加密 |
|--------|----------|----------|----------|------|
| AWS | s3 | S3 + DynamoDB | DynamoDB | SSE-S3/KMS |
| Azure | azurerm | Blob Storage | Blob Lease | AES-256 |
| GCP | gcs | Cloud Storage | 内置 | Google-managed |
| 阿里云 | oss | OSS | Tablestore/OSS | AES-256/KMS |
| 腾讯云 | cos | COS | COS 内置 | AES-256/KMS |

## 参考资料

- [Terraform 后端配置](https://developer.hashicorp.com/terraform/language/settings/backends/configuration)
- [S3 后端](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [Azure 后端](https://developer.hashicorp.com/terraform/language/settings/backends/azurerm)
- [OSS 后端](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/backend/oss)
- [COS 后端](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/backend/cos)

---

# 启用状态锁定防止损坏

**优先级：** 关键
**分类：** 状态管理

## 为什么重要

没有状态锁定，并发的 Terraform 操作可能损坏状态文件或引发竞态条件。始终启用锁定机制，防止多个用户或 CI 任务同时修改状态。

## 错误示例

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    # 未配置 DynamoDB 表用于锁定
  }
}
```

**问题：** 两位工程师同时执行 `terraform apply` 可能导致状态损坏。

## 正确示例

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**GCS 方案：**

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "prod"
    # GCS 内置锁定机制，无需额外配置
  }
}
```

## 创建锁定表

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  tags = {
    Purpose = "Terraform state locking"
  }
}
```

## 多云场景适配

### 多云锁定机制

```hcl
# AWS: S3 + DynamoDB 锁定（最常用）
terraform {
  backend "s3" {
    bucket         = "my-tfstate"
    dynamodb_table = "tf-lock"  # DynamoDB 提供分布式锁
    encrypt        = true
  }
}

# Azure: Blob Storage 租约锁定
terraform {
  backend "azurerm" {
    resource_group_name  = "tf-state"
    storage_account_name = "tfstate1234"
    container_name       = "tfstate"
    # Blob Lease 自动提供锁定
  }
}

# 腾讯云: COS 内置锁定
terraform {
  backend "cos" {
    bucket = "my-tfstate-1250000000"
    # COS 原生支持状态锁定
  }
}
```

### 锁定机制对比

| 云厂商 | 锁定实现 | 可靠性 | 注意事项 |
|--------|----------|--------|----------|
| AWS | DynamoDB | 高 | 需额外创建 DynamoDB 表 |
| Azure | Blob Lease | 高 | 自动管理，无需额外资源 |
| GCP | GCS 内置 | 高 | 原生支持 |
| 阿里云 | OSS/Tablestore | 中 | 需配置 Tablestore 或 OSS 锁 |
| 腾讯云 | COS 内置 | 高 | 原生支持 |

## 参考资料

- [状态锁定](https://developer.hashicorp.com/terraform/language/state/locking)
- [S3 后端锁定](https://developer.hashicorp.com/terraform/language/settings/backends/s3)
- [COS 后端](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/backend/cos)

---

# 将现有基础设施导入 Terraform

**优先级：** 中高
**分类：** 状态管理

## 为什么重要

大多数组织都有通过手动操作或其他工具创建的现有基础设施。导入功能将这些资源纳入 Terraform 管理，提供唯一的真相来源，防止配置漂移。

## 错误示例

```bash
# 基础设施已存在但 Terraform 不知道
terraform plan
# Plan: 5 to add, 0 to change, 0 to destroy

# Terraform 试图创建已存在的资源！
# 这将失败或创建重复资源
```

## 正确示例

### 步骤 1：编写配置

```hcl
# 首先，编写资源配置
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  subnet_id     = "subnet-abc123"

  tags = {
    Name = "production-web-server"
  }
}
```

### 步骤 2：导入资源

```bash
# 将已有资源导入状态
terraform import aws_instance.web i-1234567890abcdef0

# 验证导入
terraform state show aws_instance.web
```

### 步骤 3：调整配置

```bash
# 运行 plan 查看差异
terraform plan

# 更新配置以匹配实际资源
# 重复此过程直到 plan 显示无变更
```

### Import 块（Terraform 1.5+）

```hcl
# 声明式导入 - 推荐方式
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  # ...
}
```

```bash
# 从导入生成配置
terraform plan -generate-config-out=generated.tf
```

### 使用 for_each 批量导入

```hcl
# 导入多个资源
locals {
  existing_instances = {
    "web-1" = "i-111111111"
    "web-2" = "i-222222222"
    "web-3" = "i-333333333"
  }
}

import {
  for_each = local.existing_instances
  to       = aws_instance.web[each.key]
  id       = each.value
}

resource "aws_instance" "web" {
  for_each      = local.existing_instances
  ami           = "ami-12345678"
  instance_type = "t3.micro"
}
```

### 常见导入命令

```bash
# AWS 示例
terraform import aws_vpc.main vpc-abc123
terraform import aws_subnet.public subnet-abc123
terraform import aws_security_group.web sg-abc123
terraform import aws_db_instance.main mydb
terraform import aws_s3_bucket.data my-bucket-name
terraform import aws_iam_role.app my-role-name

# 模块资源
terraform import module.vpc.aws_vpc.this vpc-abc123

# 复杂 ID 的资源
terraform import 'aws_route53_record.www["A"]' 'Z123456_example.com_A'
```

### 大规模环境的导入策略

```bash
# 1. 盘点现有资源
aws ec2 describe-instances --query 'Reservations[].Instances[].InstanceId'

# 2. 创建导入脚本
#!/bin/bash
terraform import aws_instance.web[0] i-111111111
terraform import aws_instance.web[1] i-222222222
terraform import aws_instance.web[2] i-333333333

# 3. 执行导入
chmod +x import.sh
./import.sh

# 4. 生成配置
terraform plan -generate-config-out=imported.tf

# 5. 审查并重构生成的代码
```

### 批量导入工具

以下工具可辅助大规模导入：

```bash
# terraformer - 从现有基础设施生成 Terraform 配置
terraformer import aws --resources=vpc,subnet,security_group

# former2 - AWS CloudFormation/Terraform 生成器
# 通过 Web 界面选择资源
```

### 处理导入错误

```bash
# 资源未找到
Error: Cannot import non-existent remote object
# 请验证资源 ID 是否正确

# 资源已被管理
Error: Resource already managed by Terraform
# 检查状态：terraform state list | grep resource_name

# 资源类型错误
Error: resource address does not match
# 确保资源类型匹配（如 aws_instance vs aws_spot_instance_request）
```

### 导入后检查清单

```markdown
- [ ] 导入成功（terraform import）
- [ ] 配置与实际资源匹配（terraform plan 显示无变更）
- [ ] 敏感值已迁移到变量
- [ ] 标签和命名规范已应用
- [ ] 文档已更新
- [ ] 团队已获知新纳入管理的资源
```

### 防止意外销毁

```hcl
# 过渡期保护已导入的资源
resource "aws_instance" "web" {
  # ... 配置 ...

  lifecycle {
    prevent_destroy = true # 确认导入正确后移除此行
  }
}
```

## 多云场景适配

### 阿里云资源导入

```bash
# 阿里云常用资源导入
terraform import alicloud_instance.main i-2ze4...
terraform import alicloud_db_instance.main rm-2ze4...
terraform import alicloud_vpc.main vpc-2ze4...
terraform import alicloud_slb_load_balancer.main lb-2ze4...
terraform import alicloud_redis_instance.main r-2ze4...
```

### Azure 资源导入

```bash
# Azure 使用完整资源 ID 导入
terraform import azurerm_virtual_machine.main /subscriptions/00000000-0000-0000-0000-000000000000/resourceGroups/mygroup/providers/Microsoft.Compute/virtualMachines/myvm

terraform import azurerm_mysql_flexible_server.main /subscriptions/.../providers/Microsoft.DBforMySQL/flexibleServers/myserver

terraform import azurerm_storage_account.main /subscriptions/.../providers/Microsoft.Storage/storageAccounts/mystorage

# 使用 Azure CLI 查询资源 ID
az resource list --resource-group mygroup --query "[].id" -o tsv
```

### 腾讯云资源导入

```bash
# 腾讯云常用资源导入
terraform import tencentcloud_instance.main ins-xxxx
terraform import tencentcloud_mysql_instance.main cdb-xxxx
terraform import tencentcloud_vpc.main vpc-xxxx
terraform import tencentcloud_redis_instance.main crs-xxxx
```

### 多云导入对比

| 云厂商 | ID 格式 | 发现工具 | 批量导入 |
|--------|---------|---------|---------|
| AWS | `i-xxx`, `vpc-xxx` | AWS CLI `aws resource list` | terraform import 循环 |
| Azure | 完整资源 ID | Azure CLI `az resource list` | terraform import 循环 |
| 阿里云 | `i-xxx`, `vpc-xxx` | aliyun CLI `aliyun ecs DescribeInstances` | terraform import 循环 |
| 腾讯云 | `ins-xxx`, `vpc-xxx` | tccli `tccli cvm DescribeInstances` | terraform import 循环 |

## 参考资料

- [Terraform 导入](https://developer.hashicorp.com/terraform/cli/import)
- [Import 块](https://developer.hashicorp.com/terraform/language/import)
- [生成配置](https://developer.hashicorp.com/terraform/language/import/generating-configuration)
- [阿里云 Provider 导入](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs)
- [Azure Provider 导入](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)

---

# State 分析与配置重建规范

**优先级：** 中高
**分类：** 状态管理

> Import state 仅用于分析现有资源配置，重写/重建配置必须依据官方文档和 Provider 源码。

## 为什么重要

在实际项目中，我们经常需要：
1. **Import 现有资源** → 获取 state 分析实际配置
2. **重写声明层配置** → 实现可重建性

但这两个步骤的信息来源**不能混为一谈**：

| 阶段 | 信息来源 | 用途 |
|------|---------|------|
| 分析阶段 | State（import 结果） | 了解现有资源的实际配置 |
| 重建阶段 | 官方文档 + Provider 源码 | 确保配置准确、完整、可重建 |

## State 的局限性

Import 得到的 state 存在以下问题：

### 1. 参数不完整

```hcl
# State 中可能缺失的参数示例
# MongoDB 分片集群使用错误的资源类型 import：
db_instance_class = "dds.mongo.logic"  # 只有逻辑标识，无组件规格
# 缺失：mongo_list, shard_list, config_server_list
```

### 2. 类型信息丢失

```hcl
# State 中某些字段可能为空或默认值
encrypted = false          # 可能被 Provider 填充默认值
storage_type = ""          # 可能未在 state 中体现
```

### 3. 资源类型错误

```hcl
# 分片集群可能被错误 import 为副本集资源
# State 显示：alicloud_mongodb_instance
# 实际应该：alicloud_mongodb_sharding_instance
```

## 正确的做法

### 步骤 1：State 分析（了解现状）

```bash
# Import 现有资源
terraform import alicloud_mongodb_instance.this dds-bp1bbce1bc9d8df4

# 查看 state
terraform state show alicloud_mongodb_instance.this
```

从 state 获取：
- 实例 ID、规格代码、版本号
- 网络配置（VPC、VSwitch、Zone）
- 可用区分布
- 组件拓扑（从 zone_infos 推断）

### 步骤 2：查阅官方文档（确认参数）

```
# 官方文档来源
1. 产品文档：https://help.aliyun.com/zh/mongodb/
2. API 文档：https://help.aliyun.com/zh/mongodb/developer-reference/
3. 规格表：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
```

### 步骤 3：查阅 Provider 源码（确认 schema）

```
# Provider 源码位置
terraform-provider-alicloud/alicloud/resource_alicloud_mongodb_sharding_instance.go

# 确认：
- 必填参数（Required）
- 可选参数（Optional）
- ForceNew 参数（修改会重建）
- 参数验证规则（ValidateFunc）
- Computed 参数（由 API 返回）
```

### 步骤 4：重写配置（可重建性）

```hcl
# ✅ 基于官方文档 + Provider 源码重写的完整配置
resource "alicloud_mongodb_sharding_instance" "this" {
  # 从 state 获取：engine_version = "8.0"
  engine_version = "8.0"
  
  # 从官方文档确认：mongo_list 是 Required
  # 从 Provider 源码确认：node_class 是 Required
  mongo_list {
    node_class = "mdb.shard.4x.large.d"  # 从规格表确认
  }
  mongo_list {
    node_class = "mdb.shard.4x.large.d"
  }
  
  # 从官方文档确认：shard_list 是 Required
  shard_list {
    node_class        = "mdb.shard.4x.large.d"
    node_storage      = 470    # 从 state 分析或控制台确认
    readonly_replicas = 1
  }
  
  # 从 Provider 源码确认：config_server_list 有默认值
  # 可选配置，使用默认值或显式指定
}
```

## 实践案例

### 案例：MongoDB 分片集群配置重建

**State 分析发现的问题**：
```
db_instance_class = "dds.mongo.logic"  # 逻辑标识，非规格
replication_factor = 0                  # 分片集群特征
db_instance_storage = 0                 # 存储在 shard 层
zone_infos 有多个 mongos/shard 节点    # 分片集群拓扑
```

**查阅官方文档确认**：
- 分片集群需要使用 `alicloud_mongodb_sharding_instance`
- `mongo_list` 和 `shard_list` 是必填参数
- 每个组件需要指定 `node_class` 和 `node_storage`

**查阅 Provider 源码确认**：
- `mongo_list` 的 `node_class` 是 Required
- `shard_list` 的 `node_storage` 是 Required
- `config_server_list` 是 Optional，有默认值

**最终配置**：
```hcl
# 使用正确的资源类型 + 完整的组件规格配置
resource "alicloud_mongodb_sharding_instance" "this" {
  # ... 完整的可重建配置
}
```

## 检查清单

重写配置前，确认：

- [ ] 已从 state 分析现有配置
- [ ] 已查阅官方产品文档确认参数含义
- [ ] 已查阅 Provider 源码确认 schema 定义
- [ ] 已查阅 API 文档确认参数约束
- [ ] 已查阅规格表确认规格代码
- [ ] 已区分 Required / Optional / Computed 参数
- [ ] 已识别 ForceNew 参数并设置合理默认值

## 参数逆向验证法（参数不确定时）

当 Terraform 报错且无法从文档确定正确参数时，采用逆向验证。

### 适用场景

- `COMMODITY.INVALID_COMPONENT` 等参数校验错误
- 地域/可用区规格差异（文档未明确）
- 参数互斥规则不清楚
- Provider 行为与文档不一致

### 验证步骤

```bash
# 1. 控制台创建资源（使用已知可用的参数组合）
#    选择目标地域/可用区、目标规格，完成创建后记录资源 ID

# 2. 编写最小配置（仅必填参数）
resource "alicloud_nas_file_system" "test" {
  protocol_type     = "cpfs"
  storage_type      = "advance_200"  # 待验证
  file_system_type  = "cpfs"
  capacity          = 3600
  zone_id           = "cn-hangzhou-j"
  vswitch_id        = "vsw-xxx"
}

# 3. Import 实际资源
terraform import alicloud_nas_file_system.test cpfs-xxx

# 4. State 分析查看实际参数
terraform state show alicloud_nas_file_system.test

# 输出示例：
# storage_type           = "advance_100"   ← 实际值！文档写的是 advance_200
# redundancy_type        = null           ← API 默认不传！显式传 LRS 会报错
# redundancy_vswitch_ids = []             ← 空列表，非 null

# 5. 修正配置并重新 terraform plan 验证
```

### 实践案例：CPFS 参数校验错误

**问题**：创建 CPFS 报错 `COMMODITY.INVALID_COMPONENT`

**逆向验证发现**：
- 杭州j区实际只支持 `advance_100`，文档写的 `advance_200` 不适用
- CPFS 类型不应传递 `redundancy_type`，API 默认 LRS

**修复**：
```hcl
# 正确配置
resource "alicloud_nas_file_system" "this" {
  file_system_type = "cpfs"
  storage_type     = "advance_100"  # 验证后修正
  # redundancy_type 不传递（API 默认 LRS）
}
```

### 注意事项

- 逆向验证仅用于**参数校验错误诊断**，不用于生产资源配置
- 验证后需删除测试资源，使用正式 Terraform 配置重建
- 测试资源配置应放在 `modules/test/` 目录，与生产隔离

## 相关规则

- `resource-sharding-instance-type` - 分片集群资源类型区分
- `variable-forcenew-defaults` - ForceNew 参数空值默认规则
- `test-drift-detection` - 部署后二次 Plan 验证

## 参考资料

- `state-import.md`（资源导入规范）
- `import-component-resource.md`（子组件 Import 策略）
- `provider-documentation-lookup.md`（Provider 文档查找规范）

---

# 子组件型资源 Import 策略

**优先级：** 中高
**分类：** 状态管理

> 对于有子组件的复杂资源（如 MongoDB 分片集群），必须使用**专用资源类型** import 才能获取完整的组件规格。

## 为什么重要

复杂资源（如 MongoDB 分片集群）由多个子组件组成。使用错误的资源类型 import 可能只获取部分配置，导致后续管理不完整或状态损坏。

## 问题背景

阿里云部分资源有两种创建方式：

| 方式 | 资源类型 | 特点 | Import 结果 |
|------|----------|------|-------------|
| 简化模式 | `alicloud_xxx_instance` | 使用逻辑规格 | ❌ 只有拓扑，无组件规格 |
| 精细模式 | `alicloud_xxx_sharding_instance` | 逐组件配置规格 | ✅ 有完整组件规格 |

## 案例：MongoDB 分片集群

### 两种资源类型对比

```hcl
# 方式1：alicloud_mongodb_instance（简化模式）
# 规格代码：dds.mongo.logic（分片集群逻辑标识）
resource "alicloud_mongodb_instance" "this" {
  db_instance_class = "dds.mongo.logic"  # 逻辑规格，无组件详情
  db_instance_storage = 0                 # 存储在 shard 层
}

# 方式2：alicloud_mongodb_sharding_instance（精细模式）
# 规格代码：逐组件配置
resource "alicloud_mongodb_sharding_instance" "this" {
  mongo_list {
    node_class = "mdb.shard.2x.xlarge.d"  # Mongos 规格
  }
  shard_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # Shard 规格
    node_storage = 20                        # 存储大小
  }
  config_server_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # ConfigServer 规格
    node_storage = 20
  }
}
```

### Import 结果对比

**使用 `alicloud_mongodb_instance` import**：

```hcl
# ❌ 缺失组件规格！
resource "alicloud_mongodb_instance" "sharding_simple" {
  db_instance_class  = "dds.mongo.logic"  # 只有逻辑标识
  db_instance_storage = 0
  
  zone_infos = [  # 只有拓扑信息，无规格
    { node_type = "mongos", zone_id = "cn-hangzhou-j" },
    { node_type = "shard", zone_id = "cn-hangzhou-j" },
    { node_type = "configServer", zone_id = "cn-hangzhou-j" },
  ]
  
  # ❌ 缺失：mongo_list, shard_list, config_server_list
}
```

**使用 `alicloud_mongodb_sharding_instance` import**：

```hcl
# ✅ 有完整组件规格！
resource "alicloud_mongodb_sharding_instance" "sharding_fine" {
  engine_version = "8.0"
  
  mongo_list {
    node_class     = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    connect_string = "s-bp17eabc54c31a14.mongodb.rds.aliyuncs.com"
    port           = 3717
  }
  
  shard_list {
    node_class        = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    node_storage      = 20                        # ✅ 有存储！
    readonly_replicas = 0
  }
  
  config_server_list {
    node_class   = "mdb.shard.2x.xlarge.d"  # ✅ 有规格！
    node_storage = 20
  }
}
```

## Provider 源码分析

### API 调用差异

| 资源类型 | API 调用 | 返回数据 |
|----------|----------|----------|
| `alicloud_mongodb_instance` | `DescribeDBInstances` | 只有 `zone_infos`（拓扑） |
| `alicloud_mongodb_sharding_instance` | `DescribeMongoDBShardingInstance` | `MongosList` + `ShardList` + `ConfigserverList` |

### 源码位置

```
terraform-provider-alicloud/alicloud/resource_alicloud_mongodb_sharding_instance.go
```

关键代码（第 775-877 行）：

```go
// alicloud_mongodb_sharding_instance 的 Read 函数
if MongosListMap, ok := object["MongosList"].(map[string]interface{}); ok {
    if nodeClass, ok := MongosListArg["NodeClass"]; ok {
        MongosListItemMap["node_class"] = nodeClass  // ✅ 获取规格
    }
}

if shardListMap, ok := object["ShardList"].(map[string]interface{}); ok {
    if nodeClass, ok := shardListArg["NodeClass"]; ok {
        shardListItemMap["node_class"] = nodeClass    // ✅ 获取规格
    }
    if nodeStorage, ok := shardListArg["NodeStorage"]; ok {
        shardListItemMap["node_storage"] = formatInt(nodeStorage)  // ✅ 获取存储
    }
}
```

## 判断标准

### 如何识别子组件型资源

1. **查阅官方文档**：资源是否有多个组件（如 Mongos/Shard/ConfigServer）
2. **查阅 Provider 源码**：是否有专用资源类型（如 `xxx_sharding_instance`）
3. **查看 Schema 定义**：是否有 `xxx_list` 类型的参数（如 `mongo_list`）

### 常见子组件型资源

| 资源类型 | 简化模式 | 精细模式（推荐） |
|----------|----------|------------------|
| MongoDB 分片集群 | `alicloud_mongodb_instance` | `alicloud_mongodb_sharding_instance` |
| Redis 集群版 | `alicloud_kvstore_instance` | `alicloud_kvstore_instance`（通过 `node_type` 区分） |

## 最佳实践流程

### 步骤 1：识别资源类型

```bash
# 查看实例 ID 格式
dds-bp1698f5c4823ea4  # MongoDB 实例 ID

# 控制台确认架构类型
# 分片集群 → 使用 alicloud_mongodb_sharding_instance
# 副本集 → 使用 alicloud_mongodb_instance
```

### 步骤 2：使用专用资源类型 import

```bash
# ✅ 正确：使用专用资源类型
terraform import alicloud_mongodb_sharding_instance.this dds-bp1698f5c4823ea4

# ❌ 错误：使用通用资源类型
terraform import alicloud_mongodb_instance.this dds-bp1698f5c4823ea4
```

### 步骤 3：验证 import 结果

```bash
# 查看 state，确认组件规格是否完整
terraform state show alicloud_mongodb_sharding_instance.this

# 检查是否有 mongo_list / shard_list / config_server_list
# 如果有，说明 import 成功获取了组件规格
```

### 步骤 4：复制规格到声明层

```hcl
# 从 state 复制组件规格到声明层
mongodb_sharding_instances = {
  "01" = {
    mongos_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d" }  # 从 state 复制
      "02" = { node_class = "mdb.shard.2x.xlarge.d" }
    }
    shards_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
      "02" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
    }
    config_server_list = {
      "01" = { node_class = "mdb.shard.2x.xlarge.d", node_storage = 20 }
    }
  }
}
```

## 检查清单

Import 子组件型资源时：

- [ ] 已确认资源架构类型（分片集群 vs 副本集）
- [ ] 已选择专用资源类型（如 `xxx_sharding_instance`）
- [ ] 已验证 import 结果包含组件规格
- [ ] 已复制组件规格到声明层配置
- [ ] 已设置 `create = false` 避免重复创建

## 相关规则

- `state-analysis-vs-rebuild` - State 分析与配置重建规范
- `resource-sharding-instance-type` - 分片集群资源类型区分

## 参考资料

- `state-import.md`（资源导入规范）
- `state-analysis-vs-rebuild.md`（State 分析与重建）
- [Terraform Import](https://developer.hashicorp.com/terraform/cli/import)

---

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

---

# 永远不要在代码中硬编码密钥

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

硬编码在 Terraform 代码中的密钥会被提交到版本控制系统、记录在 CI 输出日志中，并存储在状态文件里。这会造成严重的安全漏洞。

## 错误示例

```hcl
resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"

  # 绝对不要这样做
  username = "admin"
  password = "SuperSecret123!"
}

provider "aws" {
  region = "us-east-1"
  # 绝对不要这样做
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

**问题：** 凭证暴露在代码、状态文件和 git 历史中。

## 正确示例

**方案 1：环境变量**

```hcl
variable "db_password" {
  type        = string
  sensitive   = true
  description = "数据库密码 - 通过 TF_VAR_db_password 设置"
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "admin"
  password       = var.db_password
}
```

**方案 2：密钥管理服务**

```hcl
data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = "prod/database/credentials"
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = local.db_creds.username
  password       = local.db_creds.password
}
```

**方案 3：随机密码生成**

```hcl
resource "random_password" "db_password" {
  length  = 32
  special = true
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

resource "aws_db_instance" "database" {
  identifier     = "prod-database"
  engine         = "postgres"
  instance_class = "db.t3.micro"
  username       = "admin"
  password       = random_password.db_password.result
}
```

## 补充说明

- 将敏感变量标记为 `sensitive = true`
- 使用 `.gitignore` 排除包含密钥的 `.tfvars` 文件
- 考虑使用 SOPS 或 sealed-secrets 加密敏感值

## 参考资料

- [敏感变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
- [AWS Secrets Manager](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/secretsmanager_secret_version)

---

# 禁止硬编码密码：使用环境变量或 KMS 加密

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

在 `.tfvars` 或 `.tf` 文件中硬编码明文密码，会导致密码随代码仓库泄露。代码仓库的访问范围远大于基础设施的访问范围，一旦泄露，攻击者可直接获取数据库、中间件的管理密码。

## 问题：tfvars 中硬编码明文密码

```hcl
# ❌ 错误：terraform.tfvars 中硬编码密码
# declarative/simple/04-database/terraform.tfvars

rds_instances = {
  "mysql" = {
    master_password = "Caocao123!@#"          # 明文密码！
  }
}

redis_instances = {
  "cache" = {
    password        = "Redis@Classic#123"      # 明文密码！
    vpc_auth_mode   = "Open"                   # VPC 免密，更危险
  }
}

kafka_instances = {
  "producer" = {
    password = "KafkaProd123!"                 # 明文密码！
  }
}
```

**风险：**
1. 密码随 `git push` 进入版本控制，永久留存
2. 代码仓库的协作者、CI/CD 系统都能看到密码
3. 即使后续删除，git 历史中仍然存在
4. 违反安全合规要求（等保、SOC2 等）

## 解决方案

### 方案 1：环境变量注入（推荐测试环境）

```hcl
# tfvars 中使用变量占位
# declarative/simple/04-database/terraform.tfvars

rds_instances = {
  "mysql" = {
    master_password = var.rds_master_password   # 从环境变量读取
  }
}
```

```bash
# 执行时通过环境变量注入
export TF_VAR_rds_master_password="your-secure-password"
terraform apply -var-file="terraform.tfvars"
```

### 方案 2：KMS 加密密码（推荐生产环境）

```hcl
# 原子模块中使用 kms_encrypted_password
# atomic/rds/main.tf

resource "alicloud_db_instance" "this" {
  # 使用 KMS 加密的密码，Terraform state 中不存储明文
  kms_encrypted_password = var.kms_encrypted_password != "" ? var.kms_encrypted_password : null
}
```

```hcl
# tfvars 中使用 KMS 加密后的密文
rds_instances = {
  "mysql" = {
    kms_encrypted_password = "AQICAHxxxx..."   # KMS 加密密文
  }
}
```

### 方案 3：separate secrets 文件（.gitignore 排除）

```hcl
# secrets.tfvars（加入 .gitignore）
rds_master_password = "actual-password-here"
redis_password      = "actual-password-here"
```

```bash
# 执行时合并 tfvars
terraform apply \
  -var-file="terraform.tfvars" \
  -var-file="secrets.tfvars"
```

## VPC 免密模式风险

```hcl
# ❌ 危险：Redis VPC 免密模式
redis_instances = {
  "cache" = {
    vpc_auth_mode = "Open"    # VPC 内任何实例无需密码即可访问
  }
}
```

**风险链：** ECS 被入侵 → 攻击者利用 ECS 元数据获取 VPC 信息 → 无密码访问 Redis → 数据泄露

```hcl
# ✅ 正确：使用密码认证
redis_instances = {
  "cache" = {
    password      = var.redis_password   # 环境变量注入
    vpc_auth_mode = "Close"              # 强制密码认证
  }
}
```

## .gitignore 配置

```gitignore
# 敏感文件
*.tfstate
*.tfstate.backup
*secrets*.tfvars
*secret*.tfvars
.env
```

## 检查清单

- [ ] `.tfvars` 中无明文密码（搜索模式：`password\s*=\s*"[^var]"`）
- [ ] 密码通过环境变量 `TF_VAR_xxx` 或 KMS 加密注入
- [ ] `secrets*.tfvars` 已加入 `.gitignore`
- [ ] Redis/Tair 未使用 `vpc_auth_mode = "Open"`（生产环境）
- [ ] `*.tfstate.backup` 已加入 `.gitignore`（可能包含密码明文）
- [ ] `tls_private_key` 的 `private_key_pem` 未被输出或存储到可访问位置

## 参考资料

- `security-no-hardcoded-secrets.md`（禁止硬编码密钥）
- `security-credentials.md`（凭证管理规范）
- [Terraform Sensitive 变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)

---

# 使用正确的凭证管理（OIDC、Vault、IAM 角色）

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

基础设施 Provider 凭证拥有对系统的强大访问权限。不当的凭证管理会导致数据泄露、未授权访问和合规违规。

## 错误示例

```hcl
# 代码中硬编码凭证
provider "aws" {
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# 凭证写在 terraform.tfvars 中并提交到 git
aws_access_key = "AKIAIOSFODNN7EXAMPLE"
aws_secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"

# 团队共享凭证
# "用 wiki 上的管理员凭证就行"
```

**问题：**
- 凭证永久留在版本控制历史中
- 无问责机制（共享凭证）
- 无轮换机制
- 权限过宽（所有人使用相同凭证）

## 正确示例

### 优先级排序（从最佳到可接受）

1. **短期令牌**（OIDC、STS）
2. **密钥管理系统**（Vault、AWS Secrets Manager）
3. **实例/工作负载身份**（IAM 角色、服务账号）
4. **环境变量**（不写入代码）

### 方案 1：OIDC 联邦（推荐用于 CI/CD）

```yaml
# GitHub Actions 使用 OIDC - 无需存储凭证
name: Deploy
on: push

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
          # 无需 Access Key！

      - run: terraform apply -auto-approve
```

### 方案 2：密钥管理系统（Vault）

```hcl
# 从 Vault 获取凭证
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_aws_access_credentials" "creds" {
  backend = "aws"
  role    = "terraform-role"
  type    = "sts" # 短期 STS 凭证
}

provider "aws" {
  region     = "us-east-1"
  access_key = data.vault_aws_access_credentials.creds.access_key
  secret_key = data.vault_aws_access_credentials.creds.secret_key
  token      = data.vault_aws_access_credentials.creds.security_token
}
```

### 方案 3：实例/工作负载身份

```hcl
# 在 EC2 上使用实例配置文件 - 代码中无需凭证
provider "aws" {
  region = "us-east-1"
  # 自动使用实例配置文件凭证
}

# 在 GKE 上使用 Workload Identity
provider "google" {
  project = "my-project"
  region  = "us-central1"
  # 自动使用绑定到 Pod 的服务账号
}
```

### 方案 4：环境变量

```bash
# 在环境中设置凭证（不写入代码）
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_REGION="us-east-1"

# 或使用 AWS CLI 配置文件
export AWS_PROFILE="company-prod"

terraform plan
```

```hcl
# Provider 自动使用环境变量
provider "aws" {
  region = "us-east-1"
  # 未指定凭证 - 自动使用 AWS_* 环境变量
}
```

### 凭证轮换

```hcl
# 使用 IAM Access Key 轮换
resource "aws_iam_access_key" "terraform" {
  user = aws_iam_user.terraform.name

  lifecycle {
    create_before_destroy = true
  }
}

# 存储到 Secrets Manager 并配置轮换
resource "aws_secretsmanager_secret_rotation" "terraform_creds" {
  secret_id           = aws_secretsmanager_secret.terraform_creds.id
  rotation_lambda_arn = aws_lambda_function.rotate_creds.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

### 多账号 AssumeRole

```hcl
# 中央身份账号，假设到目标账号
provider "aws" {
  alias  = "prod"
  region = "us-east-1"

  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT:role/TerraformRole"
    session_name = "terraform-prod"
    external_id  = var.external_id # 额外安全保障
  }
}
```

### 绝不提交凭证

```gitignore
# .gitignore
*.tfvars
!example.tfvars
.env
.env.*
credentials*
*.pem
*.key
```

## 多云场景适配

### Azure: Managed Identity + Key Vault

```hcl
# Azure: 使用 Managed Identity 自动认证
provider "azurerm" {
  features {}
  use_msi = true  # 在 Azure VM/ACI 上自动获取凭证
}

# 从 Key Vault 读取密钥
data "azurerm_key_vault_secret" "db_password" {
  name         = "db-password"
  key_vault_id = var.key_vault_id
}
```

### 阿里云: RAM 角色 + KMS

```hcl
# 阿里云: 通过 ECS 实例 RAM 角色自动获取临时凭证
# 或通过 OIDC 联邦身份认证（类似 AWS）
provider "alicloud" {
  # RAM 角色模式下无需配置 access_key
}

# 使用 KMS 密钥管理服务
data "alicloud_kms_secret" "db_password" {
  secret_name = "db-password"
}
```

### 腾讯云: CAM 角色 + Secrets Manager

```hcl
# 腾讯云: 在 CVM/TKE 上通过 CAM 角色自动获取临时凭证
provider "tencentcloud" {
  # CAM 角色模式下无需配置 secret_id/secret_key
}

# 使用 Secrets Manager 管理密钥
data "tencentcloud_ssm_secret" "db_password" {
  secret_name = "db-password"
}
```

### 凭证管理多云对比

| 云厂商 | 身份认证 | 密钥管理 | OIDC 支持 |
|--------|----------|----------|-----------|
| AWS | IAM Role | Secrets Manager / Parameter Store | 支持 |
| Azure | Managed Identity | Key Vault | 支持 |
| 阿里云 | RAM 角色 | KMS 密钥管理 | 支持 |
| 腾讯云 | CAM 角色 | Secrets Manager | 支持 |
| 通用 | Vault | Vault Secrets Engine | 支持 |

## 参考资料

- [HashiCorp 凭证管理](https://developer.hashicorp.com/terraform/cloud-docs/recommended-practices/part2#q9-how-are-infrastructure-service-provider-credentials-managed)
- [AWS OIDC Provider](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)
- [Vault AWS Secrets Engine](https://developer.hashicorp.com/vault/docs/secrets/aws)
- [Azure Managed Identity](https://learn.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/)
- [阿里云 RAM OIDC](https://help.aliyun.com/zh/ram/user-guide/oidc-federation/)

---

# 遵循最小权限原则

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

过于宽松的 IAM 策略会在凭证泄露时扩大爆炸半径。始终只授予资源运行所需的最低权限。

## 错误示例

```hcl
resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = "*"
        Resource = "*"
      }
    ]
  })
}
```

**问题：** Lambda 拥有对所有 AWS 服务的完全访问权限。一旦被攻破，攻击者将获得完全控制权。

## 正确示例

```hcl
resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ReadFromS3Bucket"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      },
      {
        Sid    = "WriteToSQSQueue"
        Effect = "Allow"
        Action = [
          "sqs:SendMessage"
        ]
        Resource = aws_sqs_queue.notifications.arn
      },
      {
        Sid    = "WriteLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:log-group:/aws/lambda/${var.function_name}:*"
      }
    ]
  })
}
```

## 最佳实践

1. **精确的操作** - 列出所需的精确 API 操作
2. **精确的资源** - 引用精确的 ARN，不使用通配符
3. **使用条件** - 在适用时添加条件约束
4. **独立语句** - 按用途分组并使用 Sid 标识
5. **定期审计** - 定期审查权限设置

## 使用数据源获取 ARN

```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

resource "aws_iam_role_policy" "specific_policy" {
  policy = jsonencode({
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["dynamodb:GetItem", "dynamodb:PutItem"]
        Resource = "arn:aws:dynamodb:${local.region}:${local.account_id}:table/${var.table_name}"
      }
    ]
  })
}
```

## 多云场景适配

### Azure: RBAC 角色分配

```hcl
# Azure: 使用 RBAC 精确授权
# 仅授予特定资源组的 Contributor 权限（而非整个订阅的 Owner）
resource "azurerm_role_assignment" "tf_plan" {
  scope                = azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = var.service_principal_id
}

# 自定义角色实现更细粒度控制
resource "azurerm_role_definition" "tf_custom" {
  name  = "TerraformCustomRole"
  scope = azurerm_resource_group.main.id
  permissions {
    actions     = ["Microsoft.Compute/virtualMachines/*"]
    not_actions = ["Microsoft.Compute/virtualMachines/delete"]
  }
  assignable_scopes = [azurerm_resource_group.main.id]
}
```

### 阿里云: RAM 策略

```hcl
# 阿里云: 使用 RAM 策略最小权限
resource "alicloud_ram_policy" "tf_plan" {
  policy_name = "TerraformPlanPolicy"
  policy_document = jsonencode({
    Statement = [
      {
        Action   = ["ecs:Describe*", "vpc:Describe*", "rds:Describe*"]
        Effect   = "Allow"
        Resource = ["*"]
      }
    ]
    Version = "1"
  })
  description = "Terraform plan 最小权限策略"
}
```

### 腾讯云: CAM 策略

```hcl
# 腾讯云: 使用 CAM 策略最小权限
resource "tencentcloud_cam_policy" "tf_plan" {
  name     = "TerraformPlanPolicy"
  document = jsonencode({
    version = "2.0"
    statement = [
      {
        effect   = "Allow"
        action   = ["cvm:Describe*", "vpc:Describe*", "cdb:Describe*"]
        resource = ["*"]
      }
    ]
  })
  description = "Terraform plan 最小权限策略"
}
```

### 权限模型对比

| 云厂商 | 权限服务 | 策略语言 | 模拟工具 |
|--------|----------|----------|----------|
| AWS | IAM | JSON Policy | IAM Policy Simulator |
| Azure | RBAC + Policy | JSON Role Definition | Azure Policy 测试 |
| 阿里云 | RAM | JSON Policy | RAM 策略模拟 |
| 腾讯云 | CAM | JSON Policy | CAM 策略模拟 |

## 参考资料

- [IAM 最佳实践](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)
- [Azure RBAC 最佳实践](https://learn.microsoft.com/en-us/azure/role-based-access-control/best-practices)
- [阿里云 RAM 最佳实践](https://help.aliyun.com/zh/ram/user-guide/best-practices/)
- [腾讯云 CAM 最佳实践](https://cloud.tencent.com/document/product/598/10592)

---

# 每个模块负责一个逻辑组件

**优先级：** 高
**分类：** 模块设计

## 为什么重要

职责过多的模块难以理解、测试和复用。每个模块应有一个单一的、明确的职责。

## 错误示例

```hcl
# modules/everything/main.tf
# 这个模块做了太多事情

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  count  = 3
  vpc_id = aws_vpc.main.id
  # ...
}

resource "aws_eks_cluster" "cluster" {
  name = var.cluster_name
  # ...
}

resource "aws_rds_cluster" "database" {
  cluster_identifier = var.db_name
  # ...
}

resource "aws_elasticache_cluster" "cache" {
  cluster_id = var.cache_name
  # ...
}

resource "aws_lambda_function" "api" {
  function_name = var.lambda_name
  # ...
}
```

**问题：** 这个"万能模块"同时处理网络、计算、数据库、缓存和 Serverless。任何组件的变更都会影响整个模块。

## 正确示例

```hcl
# 网络模块 - 只做一件事：创建 VPC 和子网
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags       = var.tags
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}
```

```hcl
# EKS 模块 - 只做一件事：创建 Kubernetes 集群
resource "aws_eks_cluster" "cluster" {
  name     = var.cluster_name
  role_arn = var.cluster_role_arn

  vpc_config {
    subnet_ids = var.subnet_ids
  }
}
```

```hcl
# 根模块 - 组合各专注模块
module "networking" {
  source = "./modules/networking"

  vpc_cidr            = "10.0.0.0/16"
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
}

module "eks" {
  source = "./modules/eks"

  cluster_name    = "prod-cluster"
  subnet_ids      = module.networking.public_subnet_ids
  cluster_role_arn = module.iam.eks_cluster_role_arn
}

module "database" {
  source = "./modules/rds"

  db_name   = "prod-db"
  vpc_id    = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}
```

## 设计原则

1. **每个模块一个逻辑组件** - VPC、EKS 集群、RDS 数据库各一个模块
2. **清晰的输入和输出** - 模块接口应该一目了然
3. **可组合** - 模块通过输出/输入协同工作
4. **可测试** - 每个模块可以独立测试
5. **可复用** - 同一模块可在不同环境中使用

## 模块规模参考

- **过小：** 单个资源且无逻辑
- **合适：** 5-20 个资源，职责明确
- **过大：** 50+ 个资源或包含多个不相关组件

## 参考资料

- [模块组合](https://developer.hashicorp.com/terraform/language/modules/develop/composition)

---

# 使用一致的命名规范（terraform-<PROVIDER>-<NAME>）

**优先级：** 高
**分类：** 模块设计

## 为什么重要

一致的命名规范使模块易于发现、表明其用途，并遵循社区标准。命名良好的模块更容易查找、理解和复用。

## 错误示例

```hcl
# 命名模式不一致
module "vpc" {
  source = "./modules/vpc-module"
}

module "s3" {
  source = "./modules/storage"
}

module "db" {
  source = "./modules/database-module"
}

# 更糟的 - 没有清晰的命名模式
module "infra1" {
  source = "./modules/module1"
}
```

**问题：** 不一致的命名使模块难以发现和理解，无法表明 Provider 或用途。

## 正确示例

### 标准命名规范

可复用模块（尤其是公共/注册表模块）：

```
terraform-<PROVIDER>-<NAME>
```

**示例：**
- `terraform-aws-vpc` - AWS VPC 模块
- `terraform-aws-eks` - AWS EKS 集群模块
- `terraform-aws-rds` - AWS RDS 数据库模块
- `terraform-google-network` - GCP 网络模块
- `terraform-azurerm-aks` - Azure AKS 模块

### 目录结构

```
modules/
├── terraform-aws-vpc/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── README.md
├── terraform-aws-eks/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── README.md
```

### 模块使用

```hcl
# 注册表模块
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# 遵循规范的本地模块
module "vpc" {
  source = "./modules/terraform-aws-vpc"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# Git 模块
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-vpc.git?ref=v1.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}
```

### 内部/私有模块

内部模块可使用较短的名称，但应保持一致性：

```hcl
# 方案 1：保持完整规范
module "vpc" {
  source = "./modules/terraform-aws-vpc"
}

# 方案 2：较短的内部规范（需保持一致）
module "vpc" {
  source = "./modules/aws-vpc" # 仍标明 Provider
}

# 方案 3：组织特定前缀
module "vpc" {
  source = "./modules/acme-aws-vpc" # acme = 公司名
}
```

### 命名指南

1. **使用 kebab-case** - `terraform-aws-vpc`，而非 `terraform_aws_vpc` 或 `terraformAwsVpc`
2. **包含 Provider** - 明确使用哪个云厂商
3. **描述性强** - `terraform-aws-vpc` 优于 `terraform-aws-networking`
4. **避免缩写** - `terraform-aws-database` 而非 `terraform-aws-db`
5. **使用单数** - `terraform-aws-instance` 而非 `terraform-aws-instances`

## 多云场景适配

### 多云模块命名对照

| 云厂商 | Provider 前缀 | 模块命名示例 |
|--------|-------------|-------------|
| AWS | `aws` | `terraform-aws-vpc`, `terraform-aws-rds` |
| Azure | `azurerm` | `terraform-azurerm-vnet`, `terraform-azurerm-mysql` |
| GCP | `google` | `terraform-google-network`, `terraform-google-sql` |
| 阿里云 | `alicloud` | `terraform-alicloud-vpc`, `terraform-alicloud-rds` |
| 腾讯云 | `tencentcloud` | `terraform-tencentcloud-vpc`, `terraform-tencentcloud-mysql` |

### 多云内部模块命名规范

```
modules/
├── aws-vpc/
├── azurerm-vnet/
├── alicloud-vpc/
├── tencentcloud-vpc/
├── aws-rds/
├── azurerm-mysql/
├── alicloud-rds/
└── tencentcloud-mysql/
```

### 多云环境变量命名

```hcl
# 统一命名，按云厂商区分
variable "aws_region"    { type = string }
variable "azure_location" { type = string }
variable "ali_region"    { type = string }
variable "tc_region"     { type = string }
```

## 参考资料

- [Terraform 模块命名](https://developer.hashicorp.com/terraform/language/modules/develop/standard-structure)
- [Terraform Registry 命名规范](https://developer.hashicorp.com/terraform/registry/modules/publish)

---

# 所有模块引用都要指定版本

**优先级：** 高
**分类：** 模块设计

## 为什么重要

未锁定版本的模块在上游变更时可能导致部署失败。始终锁定模块版本以确保可复现性和可控升级。

## 错误示例

```hcl
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # 无版本约束 - 使用最新版本

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

module "internal_module" {
  source = "git::https://github.com/org/modules.git//vpc"
  # 无 ref - 使用默认分支的 HEAD
}
```

**问题：** 模块行为可能在两次运行之间意外变化。

## 正确示例

```hcl
# 注册表模块 - 使用版本约束
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2" # 锁定到精确版本

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# 允许小版本更新（5.1.x）
module "vpc_flexible" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
}

# Git 模块 - 使用 ref
module "internal_module" {
  source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.3"
}

# Git 使用 commit SHA（最可复现）
module "internal_module_sha" {
  source = "git::https://github.com/org/modules.git//vpc?ref=abc123def456"
}
```

## 版本约束运算符

| 运算符 | 示例 | 含义 |
|--------|------|------|
| `=` | `= 1.2.3` | 精确版本 |
| `!=` | `!= 1.2.3` | 排除版本 |
| `>`、`>=`、`<`、`<=` | `>= 1.2.0` | 比较运算 |
| `~>` | `~> 1.2.0` | 悲观约束（允许 1.2.x，不允许 1.3.0） |

## 推荐策略

```hcl
# 生产环境 - 锁定精确版本
module "prod_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}

# 开发环境 - 允许补丁更新
module "dev_vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"
}
```

## 更新版本

```bash
# 检查更新
terraform init -upgrade

# 锁定文件跟踪精确版本
cat .terraform.lock.hcl
```

## 参考资料

- [模块源](https://developer.hashicorp.com/terraform/language/modules/sources)
- [版本约束](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)

---

# 像搭积木一样组合模块

**优先级：** 高
**分类：** 模块设计

## 为什么重要

可组合的模块可以像积木一样拼装出复杂的基础设施。这提高了复用性、减少了重复代码，并使基础设施更易于理解和维护。

## 错误示例

```hcl
# 做所有事情的单体模块
module "infrastructure" {
  source = "./modules/infrastructure"

  # VPC 设置
  vpc_cidr = "10.0.0.0/16"

  # EKS 设置
  cluster_name = "prod"
  node_count   = 3

  # RDS 设置
  db_name          = "myapp"
  db_instance_class = "db.t3.medium"

  # ElastiCache 设置
  cache_node_type = "cache.t3.micro"

  # ... 还有 50 个变量
}
```

**问题：**
- 无法单独使用 VPC 而不创建 EKS、RDS 等
- 任何组件的变更都影响整个模块
- 测试需要部署所有内容
- 难以理解和维护

## 正确示例

### 设计具有清晰输出的模块

每个模块应暴露其他模块可以消费的输出：

```hcl
# 网络模块暴露 ID 供组合使用
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC ID，供其他模块使用"
}

output "public_subnet_ids" {
  value       = [for s in aws_subnet.public : s.id]
  description = "公有子网 ID 列表"
}

output "private_subnet_ids" {
  value       = [for s in aws_subnet.private : s.id]
  description = "私有子网 ID 列表"
}
```

### 通过输出组合模块

```hcl
# 根模块通过输出将模块连接起来
module "networking" {
  source = "./modules/networking"

  vpc_cidr            = "10.0.0.0/16"
  availability_zones = var.availability_zones
}

module "eks" {
  source = "./modules/eks"

  cluster_name = "${var.project}-cluster"
  vpc_id       = module.networking.vpc_id       # 输出 -> 输入
  subnet_ids   = module.networking.private_subnet_ids
}

module "database" {
  source = "./modules/rds"

  identifier = "${var.project}-db"
  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids

  # 跨模块引用
  allowed_security_groups = [module.eks.node_security_group_id]
}
```

## 组合模式

### 通过 Locals 共享数据

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project
  }
}

module "vpc" {
  source = "./modules/vpc"
  tags   = local.common_tags
}

module "eks" {
  source = "./modules/eks"
  tags   = local.common_tags
}
```

### 使用 count 实现可选组件

```hcl
module "monitoring" {
  source = "./modules/monitoring"
  count  = var.enable_monitoring ? 1 : 0

  vpc_id = module.vpc.vpc_id
}
```

## 模块接口设计

保持接口最小化 - 只暴露需要的，隐藏实现细节：

```hcl
# 好的做法 - 聚焦的接口，合理的默认值
module "s3_bucket" {
  source      = "./modules/s3"
  name        = "my-bucket"
  versioning  = true
}

# 避免 - 泄漏抽象，暴露所有选项
module "s3_bucket" {
  source             = "./modules/s3"
  name               = "my-bucket"
  versioning         = true
  lifecycle_rules    = [...]
  cors_rules         = [...]
  replication_config = {...}
  # 暴露每个 S3 选项就失去了抽象的意义
}
```

## 参考资料

- [模块组合](https://developer.hashicorp.com/terraform/language/modules/develop/composition)
- [模块结构](https://developer.hashicorp.com/terraform/language/modules/develop/structure)

---

# 使用现有的社区/共享模块

**优先级：** 高
**分类：** 模块设计

## 为什么重要

不要重复造轮子。社区和共享模块经过实战检验、持续维护，能节省大量开发时间。使用现有模块处理常见模式，将精力集中在业务特定的基础设施上。

## 错误示例

```hcl
# 已有维护良好的模块时仍从零编写 VPC
resource "aws_vpc" "main" {
  cidr_block         = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
}

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  # ... 还有 200 行网络代码
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}
```

**问题：**
- 在已解决的问题上浪费时间
- 遗漏社区已处理的边界情况
- 团队维护负担
- 潜在的安全漏洞

## 正确示例

### 使用 Terraform Registry 模块

```hcl
# 内置最佳实践的 VPC 模块
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "${var.project}-${var.environment}"
  cidr = var.vpc_cidr

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = var.environment != "prod"

  tags = local.common_tags
}
```

### 热门社区模块

| 用途 | 模块 |
|------|------|
| AWS VPC | `terraform-aws-modules/vpc/aws` |
| AWS EKS | `terraform-aws-modules/eks/aws` |
| AWS RDS | `terraform-aws-modules/rds/aws` |
| AWS Lambda | `terraform-aws-modules/lambda/aws` |
| AWS S3 | `terraform-aws-modules/s3-bucket/aws` |
| GCP 网络 | `terraform-google-modules/network/google` |
| GCP GKE | `terraform-google-modules/kubernetes-engine/google` |
| Azure VNet | `Azure/vnet/azurerm` |
| Azure AKS | `Azure/aks/azurerm` |

### 使用前评估

```bash
# 检查模块质量
# 1. 注册表上的星标/下载数
# 2. 最近更新（是否积极维护？）
# 3. 未解决的问题数量
# 4. 文档质量
# 5. 测试覆盖率
```

### 何时编写自定义模块

在以下情况编写自定义模块：
- 没有现有模块满足需求
- 安全要求禁止使用外部依赖
- 需要对实现进行严格控制
- 社区模块已停止维护

### 私有模块注册表

```hcl
# Terraform Cloud/Enterprise 私有注册表
module "internal_vpc" {
  source  = "app.terraform.io/my-org/vpc/aws"
  version = "1.0.0"
}

# Git 源私有模块
module "internal_vpc" {
  source = "git::https://github.com/my-org/terraform-modules.git//vpc?ref=v1.0.0"
}
```

### 包装社区模块

在社区模块之上添加组织默认值：

```hcl
# 薄包装模块，强制执行公司标准
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = var.name
  cidr = var.cidr

  # 公司标准：始终启用 DNS
  enable_dns_hostnames = true
  enable_dns_support   = true

  # 公司标准：必须开启流日志
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group  = true
  create_flow_log_cloudwatch_iam_role   = true

  # 透传其他变量
  azs             = var.azs
  private_subnets = var.private_subnets
  public_subnets  = var.public_subnets

  tags = merge(var.tags, {
    ManagedBy = "terraform"
  })
}
```

## 多云场景适配

### 多云 Registry 模块对比

| 云厂商 | 模块命名空间 | 推荐模块 | 质量 |
|--------|-------------|---------|------|
| AWS | `terraform-aws-modules` | VPC, RDS, EKS, S3 | 高（官方维护） |
| Azure | `Azure` | AKS, SQL, Key Vault | 高（微软维护） |
| GCP | `terraform-google-modules` | GKE, Cloud SQL, VPC | 高（Google 维护） |
| 阿里云 | `aliyun` | VPC, ECS, RDS, OSS | 中（阿里维护） |
| 腾讯云 | `tencentcloudstack` | VPC, CVM, MySQL | 中（腾讯维护） |

### 腾讯云 Registry 使用

```hcl
# 腾讯云 VPC 模块
module "vpc" {
  source  = "tencentcloudstack/vpc/tencentcloud"
  version = "~> 1.0"

  vpc_name    = "${var.project}-${var.environment}"
  cidr_block  = "10.0.0.0/16"
}
```

### Registry 模块选择标准（多云通用）

1. **下载量** > 10,000（说明社区验证充分）
2. **最近更新** < 6个月（活跃维护）
3. **兼容 Terraform >= 1.0**
4. **有完整 examples/ 目录**
5. **有 CI/CD 自动化测试**

## 参考资料

- [Terraform Registry](https://registry.terraform.io/)
- [AWS 模块](https://registry.terraform.io/namespaces/terraform-aws-modules)
- [Google 模块](https://registry.terraform.io/namespaces/terraform-google-modules)
- [Azure 模块](https://registry.terraform.io/namespaces/Azure)
- [阿里云模块](https://registry.terraform.io/namespaces/aliyun)
- [腾讯云模块](https://registry.terraform.io/namespaces/tencentcloudstack)

---

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

---

# 阿里云 OpenAPI 与 Provider 文档查找规范

**优先级：** 中
**分类：** 模块设计

> 从 Provider 源码反查 OpenAPI 文档地址，确保参数审计和模块开发时有权威参考。

## 为什么重要

在原子层参数审计、模块开发、参数约束确认时，需要查阅：
1. **OpenAPI 文档**：确认参数必填规则、取值范围、联动约束
2. **Provider 文档**：确认 Terraform 参数映射、默认值、ForceNew 行为
3. **Provider 源码**：最权威的参数定义和 API 调用逻辑

如果不知道如何系统性地查找这些文档，会导致：
- 凭猜测设置默认值，引发运行时错误
- 遗漏 API 级约束（条件必填、互斥参数）
- 无法确认参数是否为 ForceNew

## 查找流程

```
┌─────────────────────────────────────────────────────────────────┐
│              OpenAPI / Provider 文档查找流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定位 Provider 源码                                          │
│     terraform-provider-alicloud/alicloud/resource_alicloud_*.go │
│     ↓                                                           │
│  2. 提取关键信息                                                 │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 信息           │ 提取位置                  │ 示例       │ │
│     ├─────────────────────────────────────────────────────────┤ │
│     │ Service Name  │ RpcPost/RpcGet 第1参数    │ "waf-openapi" │
│     │ API Version   │ RpcPost/RpcGet 第2参数    │ "2021-10-01"  │
│     │ Action Name   │ action := "..."           │ "CreatePostpaidInstance" │
│     │ Schema 字段   │ Schema: map[string]*Schema │ 见下文     │ │
│     └─────────────────────────────────────────────────────────┘ │
│     ↓                                                           │
│  3. 构造文档 URL                                                 │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 文档类型    │ URL 模板                                   │ │
│     ├─────────────────────────────────────────────────────────┤ │
│     │ OpenAPI     │ 见下方 URL 构造规则                         │ │
│     │ Provider    │ 见下方 URL 构造规则                         │ │
│     └─────────────────────────────────────────────────────────┘ │
│     ↓                                                           │
│  4. 交叉验证                                                     │
│     Provider Schema ↔ OpenAPI 文档 ↔ 实际行为                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## URL 构造规则

### 1. Terraform Provider 文档

```
https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL |
|----------------|-----|
| `alicloud_alb_load_balancer` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/alb_load_balancer |
| `alicloud_wafv3_instance` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/wafv3_instance |
| `alicloud_ssl_certificates_service_certificate` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/ssl_certificates_service_certificate |

**规则**：去掉 `alicloud_` 前缀即为 URL 路径

### 2. 阿里云 OpenAPI 文档

```
https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-{action}
```

#### 从 Provider 源码提取线索

Provider 源码中的 `RpcPost/RpcGet` 调用格式：

```go
response, err = client.RpcPost("waf-openapi", "2021-10-01", action, nil, request, false)
//                              ^^^^^^^^^^^^  ^^^^^^^^^^^
//                              Service Name   API Version
```

#### Service Name 到 product-slug 映射表

| Provider Service Name | product-slug | OpenAPI 文档基础 URL |
|----------------------|-------------|---------------------|
| `waf-openapi` | waf | https://help.aliyun.com/zh/waf/developer-reference/ |
| `cas` | ssl-certificates-service | https://help.aliyun.com/zh/ssl-certificates-service/developer-reference/ |
| `Alb` | alb | https://help.aliyun.com/zh/alb/developer-reference/ |
| `Slb` | slb | https://help.aliyun.com/zh/slb/developer-reference/ |
| `Nlb` | nlb | https://help.aliyun.com/zh/nlb/developer-reference/ |
| `Ecs` | ecs | https://help.aliyun.com/zh/ecs/developer-reference/ |
| `Rds` | rds | https://help.aliyun.com/zh/rds/developer-reference/ |
| `Dds` (MongoDB) | mongodb | https://help.aliyun.com/zh/mongodb/developer-reference/ |
| `Kvstore` (Redis) | redis | https://help.aliyun.com/zh/redis/developer-reference/ |
| `Polardb` | polardb | https://help.aliyun.com/zh/polardb/developer-reference/ |
| `Nas` | nas | https://help.aliyun.com/zh/nas/developer-reference/ |
| `Alikafka` | alikafka | https://help.aliyun.com/zh/alikafka/developer-reference/ |
| `Elasticsearch` | elasticsearch | https://help.aliyun.com/zh/elasticsearch/developer-reference/ |
| `Mse` | mse | https://help.aliyun.com/zh/mse/developer-reference/ |
| `Oss` | oss | https://help.aliyun.com/zh/oss/developer-reference/ |
| `Vpc` | vpc | https://help.aliyun.com/zh/vpc/developer-reference/ |

#### Action Name 到 API 文档 URL 转换

规则：将 Action 名从 PascalCase 转为 kebab-case，前面加 `api-`

```
CreatePostpaidInstance → api-waf-openapi-2021-10-01-createpostpaidinstance
UploadUserCertificate  → api-cas-2020-04-07-uploadusercertificate
CreateRule             → api-alb-2020-06-16-createrule
```

> **⚠️ 经验教训**：阿里云 OpenAPI 文档 URL 经常变动，精确 URL 大概率 404！
> 请使用以下 **可靠查找路径**（按优先级排序）：
>
> 1. **OpenAPI 在线调试台**（最可靠）：`https://next.api.aliyun.com/api/{service}/{version}/{action}`
>    - 可直接看到请求/响应参数
>    - WAF 3.0 示例：https://next.api.aliyun.com/api/waf-openapi/2021-10-01/CreatePostpaidInstance
> 2. **帮助中心搜索**：`https://help.aliyun.com/zh/` → 搜索产品名 → 开发者参考
>    - WAF 3.0 正确路径：https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/
>    - 注意：产品 slug 可能不是 Service Name 直译（如 waf-openapi → waf/web-application-firewall-3-0）
> 3. **API 目录页**（推荐中间页）：`https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-dir/`
>    - WAF 3.0：https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/api-waf-openapi-2021-10-01-dir/
>    - 从目录页可以逐级导航到具体的 Action 页面
> 4. **Provider 文档**（Terraform 侧）：`https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}`
>    - 有时包含 OpenAPI 参数对照
>    - WAF 示例：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/wafv3_instance

### 3. 在线 OpenAPI 调试

```
https://next.api.aliyun.com/api/{service}/{version}/{action}
```

| 资源 | 在线调试 URL |
|------|-------------|
| WAF 创建实例 | https://next.api.aliyun.com/api/waf-openapi/2021-10-01/CreatePostpaidInstance |
| SSL 上传证书 | https://next.api.aliyun.com/api/cas/2020-04-07/UploadUserCertificate |
| ALB 创建规则 | https://next.api.aliyun.com/api/Alb/2020-06-16/CreateRule |

## Provider 源码关键位置速查

### 1. Schema 定义（参数完整性审计入口）

```go
// 位置：resource_alicloud_{product}.go 顶部
Schema: map[string]*schema.Schema{
    "param_name": {
        Type:     schema.TypeString,     // 参数类型
        Optional: true,                   // 可选
        Required: true,                   // 必填
        Computed: true,                   // API 返回
        ForceNew: true,                   // 修改触发重建
        Default:  "default_value",        // Provider 内置默认值
    },
}
```

### 2. Create 函数（API 调用逻辑）

```go
// 位置：resourceAlicloud{Product}Create()
action := "CreateXxx"                    // API Action 名
response, err = client.RpcPost(          // API 调用
    "service-name",                       // Service 标识
    "2021-10-01",                         // API 版本
    action, nil, request, false,          // Action + 请求体
)
```

### 3. Update 函数（参数更新行为）

```go
// 位置：resourceAlicloud{Product}Update()
// 关键：d.HasChange("param") 判断哪些参数变更触发 Update
if d.HasChange("param_name") {
    update = true  // 触发 API 调用
}
```

### 4. 数据源定义（查询类资源）

```go
// 位置：dataSource_alicloud_{product}.go
// 数据源用于查询已有资源，如 SSL 证书查询
```

## 审计时的标准操作步骤

### Step 1: 定位 Provider 源码

```bash
# 在项目内搜索
terraform-provider-alicloud/alicloud/resource_alicloud_{resource_name}.go
```

### Step 2: 提取 Schema 信息

对照 `module-parameter-completeness` 规则，提取所有 Optional/Required 字段

### Step 3: 查阅 OpenAPI 文档

1. 从 `RpcPost/RpcGet` 提取 Service + Version + Action
2. 构造 OpenAPI 文档 URL 或在线调试 URL
3. 确认参数约束（必填/选填、取值范围、联动规则）

### Step 4: 查阅 Provider 文档

1. 构造 Provider 文档 URL
2. 确认 Terraform 侧的参数映射和默认值

### Step 5: 交叉验证

| 验证项 | 方法 |
|--------|------|
| 参数是否存在 | Provider Schema ↔ OpenAPI 参数列表 |
| 默认值是否安全 | Provider Default ↔ OpenAPI 文档描述 |
| ForceNew 是否正确 | Provider ForceNew ↔ OpenAPI 重建行为 |
| 条件必填 | OpenAPI 文档 ↔ 原子层 validation |

## 产品 slug 完整映射表（实际验证）

> **关键发现**：阿里云帮助中心的 product-slug 往往不是 Service Name 的直译，
> 需要通过搜索或导航确认。以下为实际验证过的正确路径。

| Provider Service Name | 帮助中心 product-slug | 开发者参考完整路径 |
|----------------------|---------------------|----------------|
| `waf-openapi` | waf/web-application-firewall-3-0 | https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/ |
| `cas` | ssl-certificates-service | https://help.aliyun.com/zh/ssl-certificates-service/developer-reference/ |
| `Alb` | alb | https://help.aliyun.com/zh/alb/developer-reference/ |
| `Slb` | slb | https://help.aliyun.com/zh/slb/developer-reference/ |
| `Nlb` | nlb | https://help.aliyun.com/zh/nlb/developer-reference/ |
| `Ecs` | ecs | https://help.aliyun.com/zh/ecs/developer-reference/ |
| `Rds` | rds | https://help.aliyun.com/zh/rds/developer-reference/ |
| `Dds` (MongoDB) | mongodb | https://help.aliyun.com/zh/mongodb/developer-reference/ |
| `Kvstore` (Redis) | redis | https://help.aliyun.com/zh/redis/developer-reference/ |
| `Polardb` | polardb | https://help.aliyun.com/zh/polardb/developer-reference/ |
| `Nas` | nas | https://help.aliyun.com/zh/nas/developer-reference/ |
| `Alikafka` | alikafka | https://help.aliyun.com/zh/alikafka/developer-reference/ |
| `Elasticsearch` | elasticsearch | https://help.aliyun.com/zh/elasticsearch/developer-reference/ |
| `Mse` | mse | https://help.aliyun.com/zh/mse/developer-reference/ |
| `Oss` | oss | https://help.aliyun.com/zh/oss/developer-reference/ |
| `Vpc` | vpc | https://help.aliyun.com/zh/vpc/developer-reference/ |

## 当精确 URL 404 时的排查流程

```
精确 URL 404
    ↓
┌─── 1. 尝试 API 目录页 ────────────────────────────────────┐
│    https://help.aliyun.com/zh/{slug}/developer-reference/  │
│    → 从目录导航到具体 Action                                │
└──────────────────────────────────────────────────────────┘
    ↓ 仍然 404
┌─── 2. 使用搜索引擎 ──────────────────────────────────────┐
│    搜索关键词："阿里云 {Action名} {Service名} API"         │
│    → 搜索结果通常包含正确的文档链接                         │
└──────────────────────────────────────────────────────────┘
    ↓ 仍然找不到
┌─── 3. 使用在线调试台（最可靠）──────────────────────────┐
│    https://next.api.aliyun.com/api/{service}/{version}/   │
│    → 直接查看请求/响应参数定义                              │
└──────────────────────────────────────────────────────────┘
    ↓ 需要更多上下文
┌─── 4. 回归 Provider 源码（最权威）──────────────────────┐
│    Schema 定义 → 完整参数列表                              │
│    Create/Update 函数 → API 调用逻辑                      │
│    → 源码不会 404，永远可读                                │
└──────────────────────────────────────────────────────────┘
```

## 相关规则

- `module-parameter-completeness` - 原子层参数完整性检查
- `provider-optional-api-mandatory` - Provider 参数 Schema 类型与处理规范
- `code-reference-documentation` - 原子层代码参考规范

## 参考资料

- `provider-documentation-lookup.md`（多云 Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [阿里云 OpenAPI 调试台](https://next.api.aliyun.com/)

---

# 模块依赖预过滤：避免引用未创建的模块实例

**优先级：** 关键
**分类：** 模块设计

## 为什么重要

当一个模块（如 ECS）使用 `create = false` 跳过创建时，依赖它的模块（如 HBR 备份）如果仍然遍历原始 map，会因为引用不存在的模块实例而报错：

```
Error: "instance_id": required field is not set
```

这是 Terraform for_each 求值顺序导致的问题：**for_each 先求值，create 条件后求值**。

## 问题场景

```hcl
# 声明层配置
mysql_ecs_instances = {
  "01" = {
    create             = false    # ❌ ECS 不创建
    hbr_backup_enabled = true     # HBR 备份启用
  }
}

# 控制层代码（错误写法）
module "mysql_ecs" {
  source   = "../../atomic/ecs"
  for_each = var.mysql_ecs_instances
  create   = try(each.value.create, true)
  # ... 其他配置 ...
}

module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = var.mysql_ecs_instances    # ❌ 遍历原始 map

  # 当 ECS create = false 时，module.mysql_ecs["01"] 不存在！
  create      = try(each.value.create, true) && try(each.value.hbr_backup_enabled, false)
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")  # ❌ 报错！
}
```

**报错信息：**
```
Error: "instance_id": required field is not set
with module.db.module.mysql_ecs_backup_client["01"].alicloud_hbr_ecs_backup_client.this[0]
```

## 根因分析

| 求值顺序 | 说明 |
|----------|------|
| 1. for_each | 先确定要创建哪些模块实例 |
| 2. 模块引用 | module.mysql_ecs[each.key] 需要对应实例存在 |
| 3. create 条件 | 最后才判断是否真正创建资源 |

**关键问题**：当 ECS `create = false` 时，`module.mysql_ecs["01"]` 这个模块实例根本不存在，引用它的 `instance_id` 会直接报错，而不是返回空值。

## 解决方案：locals 预过滤

### 步骤 1：在 locals 中预过滤

```hcl
locals {
  # 预过滤：只包含同时满足以下条件的实例
  # 1. create = true（ECS 实例需要创建）
  # 2. hbr_backup_enabled = true（HBR 备份启用）
  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}
```

### 步骤 2：依赖模块使用预过滤后的 map

```hcl
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled    # ✅ 使用预过滤后的 map

  # 只需检查 ECS 实例是否创建成功（create 和 hbr_backup_enabled 已在 locals 中过滤）
  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}

module "mysql_ecs_backup_plan" {
  source   = "../../atomic/hbr-ecs-backup-plan"
  for_each = local.mysql_ecs_hbr_backup_enabled    # ✅ 使用预过滤后的 map

  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  # ... 其他配置 ...
}
```

## 决策矩阵

| ECS create | hbr_backup_enabled | 是否进入预过滤 map | HBR 模块行为 |
|------------|-------------------|-------------------|--------------|
| true | true | ✅ 进入 | 尝试创建 HBR 资源 |
| true | false | ❌ 不进入 | 不创建（跳过） |
| false | true | ❌ 不进入 | 不创建（跳过） |
| false | false | ❌ 不进入 | 不创建（跳过） |

## 完整示例

```hcl
# control/database-cluster/main.tf

locals {
  common_tags = merge(
    { "ops-env" = var.env, "project" = var.project },
    var.tags,
  )

  ################################################################################
  # MySQL ECS HBR 备份预过滤
  # 只包含同时满足以下条件的实例：
  #   1. create = true（ECS 实例需要创建）
  #   2. hbr_backup_enabled = true（HBR 备份启用）
  # 用途：避免 ECS 未创建时 HBR 模块引用不存在的 instance_id
  ################################################################################

  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}

# ECS 模块（正常遍历原始 map）
module "mysql_ecs" {
  source   = "../../atomic/ecs"
  for_each = var.mysql_ecs_instances

  create        = try(each.value.create, true)
  instance_name = coalesce(try(each.value.instance_name, ""), "ecs-${var.env}-mysql-${each.key}")
  # ... 其他配置 ...
}

# HBR 备份客户端（使用预过滤后的 map）
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled

  # create 和 hbr_backup_enabled 已在 local.mysql_ecs_hbr_backup_enabled 中过滤
  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}

# HBR 备份计划（使用预过滤后的 map）
module "mysql_ecs_backup_plan" {
  source   = "../../atomic/hbr-ecs-backup-plan"
  for_each = local.mysql_ecs_hbr_backup_enabled

  create                = try(module.mysql_ecs[each.key].instance_id, "") != ""
  ecs_backup_plan_name  = "plan-${var.env}-mysql-${each.key}"
  instance_id           = try(module.mysql_ecs[each.key].instance_id, "")
  vault_id              = try(each.value.hbr_vault_key, "") != "" ? lookup(var.hbr_vault_ids_map, each.value.hbr_vault_key, "") : ""
  backup_type           = try(each.value.hbr_backup_type, "COMPLETE")
  retention             = try(each.value.hbr_retention_days, "7")
  schedule              = try(each.value.hbr_schedule, "0 0 4 * * *")
  path                  = try(each.value.hbr_backup_paths, [])
  disabled              = try(each.value.hbr_backup_disabled, false)
}
```

## 声明层配置示例

```hcl
# declarative/simple/04-database/terraform.tfvars

mysql_ecs_instances = {
  "01" = {
    # ECS 实例配置
    create        = true    # ✅ ECS 创建
    instance_type = "ecs.g7a.xlarge"
    image_id      = "ubuntu_22_04_x64_20G_alibase_20240808.vhd"
  
    # HBR 云备份配置
    hbr_backup_enabled  = true              # ✅ 启用 HBR 备份
    hbr_vault_key       = "default"
    hbr_retention_days  = "7"
    hbr_schedule        = "0 0 4 * * *"
    hbr_backup_paths    = ["/data/mysql"]
  }

  "02" = {
    create              = false    # ❌ ECS 不创建
    hbr_backup_enabled  = true     # 即使启用也会被预过滤跳过
    # ...
  }
}
```

## 常见陷阱

### 陷阱 1：只检查 create 条件，忽略 for_each

```hcl
# ❌ 错误：create 条件正确，但 for_each 仍遍历原始 map
module "backup" {
  for_each = var.mysql_ecs_instances    # 这里仍然包含 create=false 的实例
  create   = try(each.value.create, true) && try(each.value.hbr_backup_enabled, false)
  # 当 create=false 时，引用 module.mysql_ecs[each.key] 会报错
}
```

### 陷阱 2：使用 count 而非 for_each

```hcl
# ❌ 错误：使用 count 会导致索引错乱
module "backup" {
  count = var.mysql_ecs.create && var.mysql_ecs.hbr_backup_enabled ? 1 : 0
  instance_id = module.mysql_ecs[0].instance_id    # 索引可能不存在
}
```

## 检查清单

- [ ] 依赖模块使用 locals 预过滤后的 map 作为 for_each
- [ ] 预过滤条件包含 create = true 和功能启用开关
- [ ] 预过滤注释说明过滤条件
- [ ] 依赖模块的 create 只检查资源是否创建成功（无需重复检查 create 条件）
- [ ] 测试 create = false 场景，确认依赖模块不会报错

## 适用场景

| 场景 | 被依赖资源 | 依赖资源 | 预过滤条件 |
|------|-----------|----------|-----------|
| ECS + HBR 备份 | ECS 实例 | HBR Client/Plan | create && hbr_backup_enabled |
| ECS + EIP 绑定 | ECS 实例 | EIP Association | create && eip_enabled |
| ECS + SLB 后端 | ECS 实例 | SLB Attachment | create && attach_to_slb |
| RDS + 只读实例 | RDS 主实例 | RDS Read Replica | create && create_read_replica |

## 参考资料

- Terraform for_each 求值顺序文档
---

# 控制层 null-safe 依赖保障机制

**优先级：** 高
**分类：** 模块设计

> 控制层模块间存在依赖链路，当父资源未创建时，子资源必须安全降级（自动 create=false），避免 null 值传播导致 apply 失败。

## 为什么重要

三层架构中，声明层通过 `create=false` 控制资源是否创建。当父资源 `create=false` 时：
- 父资源 ID 为 null
- 子资源的 `for_each` 中引用父资源 ID 的参数会是 null
- 如果不处理，Terraform apply 会因 null 值传入必填参数而失败

null-safe 保障机制确保：**即使声明层配置了父资源 create=false，子资源也能安全跳过创建，不会报错。**

## 依赖链路

```
ECS → LB → ServerGroup → Listener → Rule/DomainExtension
                ↓                    ↓
           (LB ID)            (ServerGroup ID)
```

## 四种保障模式

### 模式1：双条件 create（推荐，最常用）

> 适用场景：子资源依赖单个父资源 ID

```hcl
module "alb_listener" {
  source   = "../../atomic/alb-listener"
  for_each = var.alb_listeners

  # 保障：父资源 ALB 不存在时自动跳过创建
  create = try(each.value.create, true) && lookup(local.alb_ids_map, each.value.load_balancer_key, null) != null
  # 保障：若 load_balancer_id 为 null（父 ALB 未创建），安全降级
  load_balancer_id = lookup(local.alb_ids_map, each.value.load_balancer_key, null)
}
```

**原理**：
1. `lookup(local.xxx_ids_map, key, null)` — 父资源未创建时返回 null
2. `create = ... && xxx != null` — 双条件：声明层 create=true 且父资源存在
3. 即使 `create=false`，`load_balancer_id = null` 也不会报错（因为模块内部 count=0）

### 模式2：预过滤 locals（用于跨模块依赖检查）

> 适用场景：子资源依赖需要跨多个变量判断的条件（如 HTTPS Listener 需要证书）

```hcl
locals {
  # 判断每个 Listener 是否会创建（用于 Rule 依赖检查）
  slb_listener_create = {
    for k, v in var.slb_listeners : k => (
      try(v.create, true) && (
        v.protocol != "https" ||
        (try(v.server_certificate_id, "") != "" || var.ssl_cert_id != "")
      )
    )
  }

  # Rule 依赖检查：对应的 Listener 是否会创建
  slb_rule_listener_exists = {
    for rule_key, rule_val in var.slb_rules : rule_key => (
      length([for k, v in var.slb_listeners : k
        if v.load_balancer_key == rule_val.load_balancer_key
        && v.frontend_port == rule_val.frontend_port
        && local.slb_listener_create[k]
      ]) > 0
    )
  }
}

module "slb_rule" {
  source   = "../../atomic/slb-rule"
  for_each = var.slb_rules

  # Rule 依赖 Listener，检查对应的 Listener 是否会创建
  create = try(each.value.create, true) && local.slb_rule_listener_exists[each.key]
}
```

**原理**：
1. 先在 locals 中计算每个 Listener 的创建条件
2. 再根据 Rule 的 `load_balancer_key` + `frontend_port` 匹配对应 Listener
3. 如果匹配的 Listener 不会创建，Rule 也自动跳过

### 模式3：plan-time create 标志预过滤（路由条目级别）

> 适用场景：子资源使用 `resource for_each`，且 `if` 过滤条件不能依赖 apply-time 值（如父资源 ID），必须用 plan-time 可确定的 `create` 标志

**问题根因**：当父资源（如 NAT Gateway）被禁用（`create=false`）时，子资源（如路由条目）的 `nexthop_id` 会解析为 null。如果用 `if lookup(local.nexthop_ids, ...) != null` 做过滤，会因为 `nexthop_ids` 依赖 apply-time 值（`nat_gateway_id` 是 `known after apply`）导致 `Invalid for_each argument` 错误。

```text
错误链路：
local.nexthop_ids → local.nat_gateway_ids_map → module.nat[...].nat_gateway_id
                                                              ↑
                                                        known after apply
if 条件依赖 unknown 值 → for_each key 集合 unknown → 报错
```

**解决方案**：用 `var.nat_gateways` 的 `create` 字段构建 plan-time 可确定的标志映射，而非引用 apply-time 的资源 ID。

```hcl
# ✅ 步骤 1：在 locals 中构建 create 标志映射（plan-time 可确定）
locals {
  nat_gateway_creates = {
    for k, v in var.nat_gateways : k => try(v.create, true)
  }
}
```

```hcl
# ✅ 步骤 2：控制层 route_entries 的 for 表达式使用 create 标志过滤
module "route_table" {
  source   = "../../atomic/route-table"
  for_each = var.route_tables

  create = try(each.value.create, false)

  # null-safe 过滤：通过 plan-time create 标志判断父资源是否存在
  # 注意：不能使用 lookup(local.nexthop_ids, ...) 做 if 过滤
  #       因为 nexthop_ids 依赖 nat_gateway_id（known after apply）
  route_entries = {
    for idx, entry in try(each.value.route_entries, []) : "entry-${idx}" => {
      destination_cidrblock = entry.destination_cidr_block
      nexthop_type          = entry.nexthop_type
      nexthop_id            = lookup(local.nexthop_ids, "${entry.nexthop_type}:${entry.nexthop_key}", null)
    }
    # ✅ 用 create 标志过滤（plan-time 安全），不用 nexthop_ids（apply-time）
    if entry.nexthop_type != "NatGateway" ? true : try(local.nat_gateway_creates[entry.nexthop_key], true) != false
  }
}
```

**原理**：
1. `nat_gateway_creates` 从 `var.nat_gateways` 计算，值在 plan 阶段完全确定
2. `if` 条件只引用 plan-time 已知值，不触发 `Invalid for_each argument`
3. `nexthop_id` 仍用 `lookup(local.nexthop_ids, ...)` 获取真实 ID（apply-time 解析，放入 value 不影响 key 集合）
4. 非 NatGateway 类型的路由条目（如 Instance/VPN）不受影响

**与 `resource-foreach-unknown-value` 的关系**：本模式是该规则「模式 B（ecs_would_create 预过滤）」在路由条目场景的具体应用。核心思路一致：**用 var 中的 plan-time 已知属性（create 标志）替代 apply-time 才知道的资源 ID 做 if 过滤**。

**声明层最佳实践**：虽然控制层有防御性保障，声明层仍应显式同步禁用依赖资源：

```hcl
# ✅ 推荐：声明层显式表达业务依赖
nat_gateways = {
  "main" = { create = false }  # 禁用 NAT
}
route_tables = {
  "proxy" = { create = false }  # 同步禁用路由表（业务语义：无 NAT 则无路由意义）
}

# ✅ 也可：只禁用 NAT，路由表留 create=true，控制层会自动过滤路由条目
# 但路由表本身仍会创建（空表），属于资源浪费
```

> 适用场景：子资源参数中嵌套了需要转换的引用（如 ALB Rule actions 中的 server_group_key）

```hcl
# ALB Rule actions 中 server_group_key 自动转换为 server_group_id
rule_actions = [
  for action in try(each.value.actions, []) : merge(
    { for k, v in action : k => v if k != "server_group_tuples" },
    {
      server_group_tuples = [
        for sgt in try(action.server_group_tuples, []) : merge(
          { for k, v in sgt : k => v if k != "server_group_key" },
          try(sgt.server_group_key, null) != null ? {
            server_group_id = lookup(local.alb_server_group_ids_map, sgt.server_group_key, "")
          } : {}
        )
      ]
    }
  )
]
```

**原理**：
1. 使用 `merge` + `{ for k, v in x : k => v if k != "排除key" }` 移除原始 key
2. 条件性地添加转换后的 key→ID
3. 如果 server_group_key 引用的资源不存在，`lookup` 返回空字符串，Terraform 会在 plan 阶段提示

## ID 映射 locals 规范

所有 ID 映射必须过滤 null 值：

```hcl
locals {
  # ✅ 正确：过滤 null 值
  alb_ids_map = { for k, v in module.alb : k => v.alb_id if v.alb_id != null }

  # ❌ 错误：不过滤 null 值，会导致 lookup 返回 null
  alb_ids_map = { for k, v in module.alb : k => v.alb_id }
}
```

## 常见 key→ID 转换模式

| 引用类型 | 声明层字段 | 控制层转换 |
|---------|-----------|-----------|
| 父资源 ID | `load_balancer_key` | `lookup(local.alb_ids_map, each.value.load_balancer_key, null)` |
| Server Group | `server_group_key` | `lookup(local.alb_server_group_ids_map, each.value.server_group_key, null)` |
| ECS 实例 | `ecs_key` | `lookup(local.ecs_ids_map, s.ecs_key, null)` |
| 快照策略 | `auto_snapshot_policy_key` | `lookup(var.ecs_snapshot_policy_ids_map, key, "")` |
| 安全组 | `security_group_key` | 由控制层固定映射，不需要 lookup |

## 完整保障检查清单

- [ ] 所有子资源模块的 `create` 使用双条件（`try(each.value.create, true) && lookup(...) != null`）
- [ ] 所有 ID 映射 locals 过滤 null 值（`if v.xxx_id != null`）
- [ ] 所有 `lookup` 调用使用三参数形式提供默认值（`lookup(map, key, null)`）
- [ ] HTTPS 监听器有证书检查条件
- [ ] 深层嵌套引用（actions/servers）使用 merge + for 表达式做 key→ID 转换
- [ ] 路由条目的 `if` 过滤使用 `create` 标志映射（plan-time 安全），不使用 `local.nexthop_ids`（apply-time 值）
- [ ] 声明层禁用父资源时同步禁用子资源（显式表达业务依赖）

## 相关规则

- `module-dependency-prefilter` - 模块依赖预过滤
- `layer-id-mapping` - 通过 map 变量实现基于 key 的资源引用
- `module-parameter-completeness` - 原子层参数完整性检查
- `resource-foreach-unknown-value` - for_each 与 unknown 值：模式 B 的路由条目实例

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `control-backup-prefilter.md`（条件子资源预过滤）
- `variable-forcenew-defaults.md`（ForceNew 空值规则）

---

# 控制层 Map 访问安全：必须使用 lookup() 保护

**优先级：** 高
**分类：** 模块设计

## 为什么重要

控制层大量使用 map 变量（`vswitch_ids_map`、`security_group_ids_map` 等）进行 ID 映射。直接使用 `map[key]` 访问时，如果 key 不存在会报 `Invalid index` 错误，导致整个 `terraform apply` 失败。使用 `lookup()` 提供安全默认值，可以在 key 不存在时优雅降级。

## 问题：直接访问 map 无保护

```hcl
# ❌ 错误：直接访问 map，key 不存在时报错
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ❌ 如果 security_group_key 在 map 中不存在 → Invalid index
  security_group_ids = [var.security_group_ids_map[each.value.security_group_key]]

  # ❌ 如果 vswitch_key 在 map 中不存在 → Invalid index
  vswitch_id = var.vswitch_ids_map[each.value.vswitch_key]
}
```

**问题：**
1. 用户在 tfvars 中配置了错误的 key（如拼写错误 "dta" 而非 "data"）
2. 上游阶段未创建对应的资源，map 中缺少该 key
3. 错误信息是 Terraform 运行时的 `Invalid index`，难以定位到具体的 key 问题

## 正确：使用 lookup() 提供安全默认值

```hcl
# ✅ 正确：lookup() 保护 + 合理默认值
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ✅ key 不存在时返回空字符串，后续逻辑可安全处理
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    []

  # ✅ key 不存在时返回空字符串，原子层会因参数为空而跳过创建
  vswitch_id = lookup(var.vswitch_ids_map, try(each.value.vswitch_key, ""), "")
}
```

## lookup() 默认值选择

| 场景 | 默认值 | 说明 |
|------|--------|------|
| 资源 ID（vswitch_id, sg_id） | `""` | 原子层检测空字符串后跳过创建 |
| 嵌套 map 访问 | `null` | 用于条件判断 `!= null` |
| 安全组 ID | `""` | 空列表 `[]` 表示不绑定安全组 |
| 布尔守卫 | `false` | 用于 create 判断 |

## 完整模式：Key 存在性检查 + lookup 保护

```hcl
# 标准模式：先检查 key 是否配置，再安全查找
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  []  # 未配置 security_group_key 时不绑定安全组

# ID 映射模式：在 locals 中构建，过滤 null 值
ecs_ids_map = {
  for k, v in module.ecs : k => v.instance_id
  if v.instance_id != null    # 过滤未创建的实例
}
```

## Plan-time 可解析要求

**关键：** `for_each` 的 `count` 和 create 判断中使用的条件必须是 plan-time 可解析的。

```hcl
# ❌ 错误：create 条件依赖 apply-time 才知的 module output
create = try(each.value.create, true) && module.ssl_self_signed.cas_cert_id != null

# ✅ 正确：create 条件使用 plan-time 已知的变量
create = try(each.value.create, true) && var.ssl_create_self_signed
```

**原因：** `count` 必须在 plan 阶段确定。如果 create 条件依赖 apply-time 值，Terraform 无法计算 count，会报 `Invalid count argument` 错误。

## 检查清单

- [ ] 控制层所有 `map[key]` 直接访问已替换为 `lookup(map, key, default)`
- [ ] `lookup()` 默认值类型与 map value 类型一致
- [ ] `for_each`/`count` 条件不依赖 apply-time 才知的值
- [ ] ID 映射 locals 中过滤了 null 值（`if v.id != null`）
- [ ] 嵌套的 `try()` + `lookup()` 不超过 2 层

## 参考资料

- `control-key-mapping-pattern.md`（Key 映射模式）
- `layer-id-mapping.md`（ID 映射模式）
- `module-null-safe-dependency.md`（null-safe 依赖保障）

---

# 控制层 ID 映射命名一致性

**优先级：** 中
**分类：** 模块设计

## 为什么重要

控制层的 6 个模块都需要构建 ID 映射（`xxx_ids_map`），用于将逻辑 Key 转换为实际资源 ID。如果命名不一致（如 `sg_ids` vs `sg_ids_map`），会导致：
1. 声明层引用时混淆
2. 新开发者难以理解命名规律
3. 跨模块引用时需要记忆不同的命名

## 问题：命名不一致

```hcl
# ❌ 错误：不同控制层模块使用不同的命名风格

# control/security/locals.tf
locals {
  sg_ids = { ... }              # 缺少 _map 后缀
}

# control/security/outputs.tf
output "security_group_ids_map" {  # 有 _map 后缀
  value = local.sg_ids            # 与 locals 命名不一致
}

# control/web-cluster/locals.tf
locals {
  alb_ids_map = { ... }          # 有 _map 后缀
  slb_ids_map = { ... }          # 有 _map 后缀
}
```

**问题：** `sg_ids`（locals）vs `security_group_ids_map`（output）→ 引用者需要记住两套命名。

## 正确：统一命名规范

### 命名规范

| 位置 | 命名格式 | 示例 |
|------|----------|------|
| 控制层 locals | `xxx_ids_map` | `alb_ids_map`, `sg_ids_map`, `slb_ids_map` |
| 控制层 outputs | `xxx_ids_map` | `alb_ids_map`, `security_group_ids_map` |
| 声明层 outputs | `xxx_ids_map` | `vswitch_ids_map`, `security_group_ids_map` |
| 声明层 remote_state | `outputs.xxx_ids_map` | `data.terraform_remote_state.network.outputs.vswitch_ids_map` |

### 统一模式

```hcl
# ✅ 正确：所有层级统一使用 xxx_ids_map 后缀

# control/security/locals.tf
locals {
  sg_ids_map = {                   # ✅ _ids_map 后缀
    for k, v in module.security_group : k => v.sg_id
    if v.sg_id != null
  }
}

# control/security/outputs.tf
output "security_group_ids_map" {   # ✅ 与 locals 一致
  value = local.sg_ids_map
}
```

### ID 映射构建模式

所有控制层模块的 ID 映射遵循统一模式：

```hcl
# 标准模式：过滤 null + 使用 module 输出
locals {
  xxx_ids_map = {
    for k, v in module.xxx : k => v.xxx_id
    if v.xxx_id != null           # 过滤未创建的实例
  }
}
```

## 常见 ID 映射对照表

| 资源 | locals 变量名 | output 名 | 来源 |
|------|---------------|-----------|------|
| ALB | `alb_ids_map` | `alb_ids_map` | `module.alb` |
| SLB | `slb_ids_map` | `slb_ids_map` | `module.slb` |
| NLB | `nlb_ids_map` | `nlb_ids_map` | `module.nlb` |
| ECS | `ecs_ids_map` | `ecs_ids_map` | `module.ecs` |
| 安全组 | `sg_ids_map` | `security_group_ids_map` | `module.security_group` |
| VSwitch | — | `vswitch_ids_map` | `module.vswitches`（声明层直出） |

## 检查清单

- [ ] 所有 ID 映射 locals 使用 `xxx_ids_map` 后缀
- [ ] 所有 ID 映射 outputs 与对应 locals 命名一致
- [ ] ID 映射构建统一使用 `{ for k, v in ... : k => v.id if v.id != null }` 模式
- [ ] 无裸露的 `map[key]` 访问（使用 `lookup()` 保护）

## 参考资料

- `control-key-mapping-pattern.md`（Key 映射模式）
- `layer-id-mapping.md`（ID 映射模式）
- `control-parameter-passthrough.md`（参数透传模式）

---

# 控制层嵌套资源 for_each 展平模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

当父资源有子资源时（如 Kafka 实例下的 SASL 用户），子资源依赖父资源的输出（instance_id）。使用 `flatten` + 嵌套 `for` 将两层 map 展平为一层，再通过复合 key (`"${parent_key}-${child_key}"`) 索引回父资源，实现一对多关系的安全编排。

## 模式说明

### 问题场景

```
Kafka 实例 "01" → SASL 用户 admin, reader
Kafka 实例 "02" → SASL 用户 admin
```

Terraform 的 `for_each` 只支持一层 map/list，无法直接表达"每个 Kafka 创建多个用户"。

### 展平模式

```hcl
module "kafka_sasl_user" {
  source = "../../atomic/kafka-sasl-user"

  # 展平：两层 map → 一层 map，复合 key
  for_each = {
    for pair in flatten([
      for kafka_key, kafka in var.kafka_instances : [
        for user_key, user in try(kafka.sasl_users, {}) : {
          kafka_key     = kafka_key
          user_key      = user_key
          user          = user
          kafka_created = try(kafka.create, true)
        }
      ] if try(kafka.create, true)  # 过滤掉不创建的 Kafka 实例
    ]) : "${pair.kafka_key}-${pair.user_key}" => pair
  }

  create    = try(each.value.user.create, true)
  instance_id = module.kafka[each.value.kafka_key].instance_id
  username  = each.value.user_key
  password  = try(each.value.user.password, "")
}
```

## 关键设计要素

### 1. 复合 Key 格式

```
${parent_key}-${child_key}
```

| 父子关系 | 复合 Key 示例 |
|----------|--------------|
| Kafka → SASL 用户 | `01-admin` |
| Kafka → ACL | `01-topic-write` |
| MongoDB → 账号 | `01-root` |
| RDS → 只读实例 | `01-reader-01` |

### 2. 父资源过滤

只展平已创建的父资源的子资源：

```hcl
if try(kafka.create, true)  # 父资源不创建 → 子资源也不创建
```

### 3. 回引父资源输出

通过 `each.value.kafka_key` 回引父模块：

```hcl
instance_id = module.kafka[each.value.kafka_key].instance_id
```

### 4. 子资源独立控制开关

每个子资源仍有自己的 `create` 标志：

```hcl
create = try(each.value.user.create, true)
```

## 声明层写法

```hcl
kafka_instances = {
  "01" = {
    create        = true
    instance_type = "alikafka_serverless"
    # ...

    # 子资源嵌套在父资源内部
    sasl_users = {
      admin = {
        password = "Admin123!"
        mechanism = "PLAIN"
      }
      reader = {
        password = "Reader123!"
        create   = false  # 暂不创建
      }
    }
  }
}
```

## 多云场景示例

### AWS MSK → SASL/IAM 用户

```hcl
module "msk_sasl_user" {
  source = "../../atomic/msk-sasl-user"

  for_each = {
    for pair in flatten([
      for cluster_key, cluster in var.msk_clusters : [
        for user_key, user in try(cluster.sasl_users, {}) : {
          cluster_key = cluster_key
          user_key    = user_key
          user        = user
        }
      ] if try(cluster.create, true)
    ]) : "${pair.cluster_key}-${pair.user_key}" => pair
  }

  create      = try(each.value.user.create, true)
  cluster_arn = module.msk[each.value.cluster_key].cluster_arn
  username    = each.value.user_key
  password    = try(each.value.user.password, "")
}
```

### Azure Event Hub → 授权规则

```hcl
module "eventhub_auth_rule" {
  source = "../../atomic/eventhub-authorization-rule"

  for_each = {
    for pair in flatten([
      for hub_key, hub in var.event_hubs : [
        for rule_key, rule in try(hub.authorization_rules, {}) : {
          hub_key  = hub_key
          rule_key = rule_key
          rule     = rule
        }
      ] if try(hub.create, true)
    ]) : "${pair.hub_key}-${pair.rule_key}" => pair
  }

  create           = try(each.value.rule.create, true)
  eventhub_id      = module.eventhub[each.value.hub_key].eventhub_id
  rule_name        = each.value.rule_key
  permissions      = try(each.value.rule.permissions, ["Listen"])
}
```

## 更多层级：三层展平

当存在三层关系（如 Kafka → ACL 依赖 SASL 用户）时，使用 `depends_on` 确保创建顺序：

```hcl
module "kafka_sasl_acl" {
  source = "../../atomic/kafka-sasl-acl"

  for_each = {
    for pair in flatten([
      for kafka_key, kafka in var.kafka_instances : [
        for acl_key, acl in try(kafka.sasl_acls, {}) : {
          kafka_key = kafka_key
          acl_key   = acl_key
          acl       = acl
        }
      ] if try(kafka.create, true)
    ]) : "${pair.kafka_key}-${pair.acl_key}" => pair
  }

  create      = try(each.value.acl.create, true)
  instance_id = module.kafka[each.value.kafka_key].instance_id

  # 确保 SASL 用户先创建
  depends_on = [module.kafka_sasl_user]
}
```

## 模式总结

| 要素 | 说明 |
|------|------|
| 展平函数 | `flatten([for parent: [for child: {复合对象}]])` |
| 复合 Key | `"${parent_key}-${child_key}"` |
| 父资源过滤 | `if try(parent.create, true)` |
| 回引父输出 | `module.parent[each.value.parent_key].output` |
| 创建顺序 | `depends_on = [module.parent_child]` |

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `module-null-safe-dependency.md`（null-safe 依赖）
- `control-backup-prefilter.md`（条件子资源预过滤模式）

---

# 控制层安全组 Key 映射模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

安全组 ID 是跨控制层模块共享的基础设施引用。直接硬编码安全组 ID 会导致环境不可移植。通过 `security_group_key` + `security_group_ids_map` 的 key 映射模式，声明层只需指定逻辑名称，控制层负责解析为实际 ID。

## 模式说明

### 映射架构

```
声明层                    控制层                     安全组模块
security_group_key    →  lookup(security_group_ids_map, key)  →  security_group_id
= "data"                  = "sg-xxx"                       = "sg-xxx"
```

### 控制层变量定义

```hcl
variable "security_group_ids_map" {
  description = "安全组 ID 映射表。key=逻辑名, value=安全组 ID"
  type        = map(string)
  default     = {}
}
```

### 控制层使用模式

#### 模式 1：单个安全组 Key

```hcl
# 最常用：单个安全组 key → 单个安全组 ID
security_group_id = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : ""
```

#### 模式 2：安全组 Key → 安全组 ID 列表

```hcl
# 资源需要安全组 ID 列表
security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

#### 模式 3：多个安全组 Key

```hcl
# ECS 绑定多个安全组（data + ecs）
security_group_ids = [
  var.security_group_ids_map[try(each.value.security_group_key, "data")]
]
```

#### 模式 4：优先 Key 回退直接 ID

```hcl
# 优先使用 key 查找，否则使用直接 ID
security_group = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : try(each.value.security_group, "")
```

## 设计原则

### 1. 控制层不设安全组业务默认值

```hcl
# 错误：控制层硬编码安全组 ID
security_group_id = "sg-xxx"

# 正确：控制层通过 key 映射查找
security_group_id = try(each.value.security_group_key, "") != ""
  ? lookup(var.security_group_ids_map, each.value.security_group_key, "")
  : ""
```

> 安全组由独立的 `security` 控制层模块管理，其他控制层通过 key 引用。

### 2. Key 使用短逻辑名

| Key 名 | 含义 | 说明 |
|--------|------|------|
| `data` | 数据层安全组 | 数据库、缓存通用 |
| `web` | Web 层安全组 | ECS、SLB 通用 |
| `ecs` | ECS 通用安全组 | 自建服务 |
| `kafka` | Kafka 专用安全组 | 消息队列 |

### 3. lookup 保护的必要性

```hcl
# 安全：key 不存在时返回空字符串
lookup(var.security_group_ids_map, each.value.security_group_key, "")

# 不安全：key 不存在时报错
var.security_group_ids_map[each.value.security_group_key]
```

## 声明层写法

```hcl
# terraform.tfvars
kafka_instances = {
  "01" = {
    security_group_key = "data"  # 引用逻辑名
    # ...
  }
}

redis_community_instances = {
  "01" = {
    security_group_key = "data"  # 同一逻辑名，不同资源
    # ...
  }
}
```

## 多云场景适配

### AWS 安全组映射

```hcl
# AWS 使用 vpc_security_group_ids
vpc_security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

### Azure NSG 映射

```hcl
# Azure 使用 network_interface_security_group_association
# key 映射到 NSG ID
network_security_group_id = try(each.value.nsg_key, "") != ""
  ? lookup(var.nsg_ids_map, each.value.nsg_key, "")
  : try(each.value.network_security_group_id, "")
```

### 腾讯云安全组映射

```hcl
# 腾讯云使用 security_groups
security_groups = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])
```

## 相同模式的其他 Key 映射

| Key 字段 | Map 变量 | 用途 |
|----------|----------|------|
| `vswitch_key` / `subnet_key` | `vswitch_ids_map` / `subnet_ids_map` | 子网 ID 查找 |
| `zone_key` | `az_map` | 可用区 ID 查找 |
| `security_group_key` / `nsg_key` | `security_group_ids_map` / `nsg_ids_map` | 安全组 ID 查找 |
| `hbr_vault_key` / `backup_vault_key` | `hbr_vault_ids_map` / `backup_vault_ids_map` | 备份库 ID 查找 |
| `system_disk_auto_snapshot_policy_key` | `ecs_snapshot_policy_ids_map` | 快照策略 ID 查找 |

所有 key 映射遵循统一模式：

```hcl
try(each.value.{key}_field, "") != ""
  ? lookup(var.{resource}_ids_map, each.value.{key}_field, "")
  : try(each.value.{direct_id_field}, "")
```

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤）
- `layer-id-mapping.md`（ID 映射模式）
- `control-parameter-passthrough.md`（控制层参数透传）

---

# 控制层条件子资源预过滤模式

**优先级：** 高
**分类：** 模块设计

## 为什么重要

控制层常需要创建附加资源（备份计划、快照策略等），这些资源依赖主资源的 ID。当主资源未创建（create=false）时，附加资源的 `for_each` 必须排除这些实例，否则会因引用不存在的 module output 而 apply 失败。使用 locals 预过滤是最安全的方案。

## 模式说明

### 问题场景

```
mysql_ecs_instances = {
  "01" = { create = true,  hbr_backup_enabled = true  }  → 需要 HBR 备份
  "02" = { create = false, hbr_backup_enabled = true  }  → ECS 不创建，备份不应创建
  "03" = { create = true,  hbr_backup_enabled = false }  → ECS 创建，但不需要备份
}
```

### 预过滤模式

```hcl
locals {
  # 只保留同时满足两个条件的实例：
  #   1. create = true（ECS 实例需要创建）
  #   2. hbr_backup_enabled = true（HBR 备份启用）
  mysql_ecs_hbr_backup_enabled = {
    for k, v in var.mysql_ecs_instances : k => v
    if try(v.create, true) && try(v.hbr_backup_enabled, false)
  }
}
```

然后在附加资源中使用过滤后的 map：

```hcl
module "mysql_ecs_backup_client" {
  source   = "../../atomic/hbr-ecs-backup-client"
  for_each = local.mysql_ecs_hbr_backup_enabled  # 使用预过滤后的 map

  create      = try(module.mysql_ecs[each.key].instance_id, "") != ""
  instance_id = try(module.mysql_ecs[each.key].instance_id, "")
}
```

## 关键设计要素

### 1. 过滤条件链

多个条件用 `&&` 连接，按优先级排列：

```hcl
# 条件 1：主资源是否创建
# 条件 2：附加功能是否启用
if try(v.create, true) && try(v.hbr_backup_enabled, false)
```

### 2. 双重保护

预过滤 + 模块内 create 检查：

```hcl
# 第一层保护：预过滤排除不需要的实例
for_each = local.mysql_ecs_hbr_backup_enabled

# 第二层保护：检查主资源是否真的创建成功
create = try(module.mysql_ecs[each.key].instance_id, "") != ""
```

### 3. key 保持一致

过滤后的 map 保持与原始 map 相同的 key，以便通过 `module.main[each.key]` 回引：

```hcl
for k, v in var.mysql_ecs_instances : k => v  # k 不变
```

## 多云场景适配

### AWS 备份预过滤

```hcl
locals {
  # AWS Backup：过滤启用备份的 RDS 实例
  rds_backup_enabled = {
    for k, v in var.rds_instances : k => v
    if try(v.create, true) && try(v.backup_enabled, false)
  }
}

resource "aws_backup_selection" "rds" {
  for_each = local.rds_backup_enabled

  plan_id      = var.backup_plan_id
  resources    = [module.rds[each.key].instance_arn]
}
```

### Azure 备份预过滤

```hcl
locals {
  # Azure Recovery Services Vault：过滤启用备份的 VM
  vm_backup_enabled = {
    for k, v in var.virtual_machines : k => v
    if try(v.create, true) && try(v.backup_enabled, false)
  }
}

resource "azurerm_backup_protected_vm" "vm" {
  for_each = local.vm_backup_enabled

  recovery_vault_name = var.recovery_vault_name
  resource_group_name = var.resource_group_name
  source_vm_id        = module.vm[each.key].vm_id
}
```

## 适用场景

| 场景 | 主资源 | 附加资源 | 过滤条件 |
|------|--------|----------|----------|
| ECS 备份 | ECS 实例 | HBR 备份客户端 + 计划 | create && hbr_backup_enabled |
| 磁盘快照 | ECS 实例 | 自动快照策略 | create && has_data_disks |
| 账号创建 | Redis/Kafka | 子账号 | create && create_account |
| 监控配置 | 任意资源 | CloudMonitor 告警 | create && monitoring_enabled |
| RDS 备份 | RDS 实例 | AWS Backup 策略 | create && backup_enabled |
| VM 备份 | Azure VM | Recovery Services | create && backup_enabled |

## 与嵌套字段的对比

### 方案 A：预过滤（推荐）

```hcl
# locals 中过滤
hbr_backup_enabled = {
  for k, v in var.instances : k => v
  if try(v.create, true) && try(v.hbr_backup_enabled, false)
}

# 模块引用
for_each = local.hbr_backup_enabled
```

### 方案 B：嵌套字段展平（适用于子资源有独立 key 时）

```hcl
# 展平子资源
for_each = {
  for pair in flatten([
    for k, v in var.instances : [
      for plan_key, plan in try(v.backup_plans, {}) : { ... }
    ] if try(v.create, true)
  ]) : "${pair.k}-${pair.plan_key}" => pair
}
```

**选择依据**：子资源是否需要独立的 key 管理。需要 → 方案 B；不需要 → 方案 A。

## 参考资料

- `module-dependency-prefilter.md`（依赖预过滤通用模式）
- `module-null-safe-dependency.md`（null-safe 依赖）
- `control-nested-resource-flatten.md`（嵌套资源展平模式）
- `control-parameter-passthrough.md`（控制层参数透传）

---

# 使用一致的命名规范

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

一致的命名规范提高可读性、便于查找资源，并有助于成本分摊和合规。尽早建立命名模式并在整个基础设施中强制执行。

## 错误示例

```hcl
resource "aws_instance" "my_server" {
  # ...
}

resource "aws_instance" "WebServer1" {
  # ...
}

resource "aws_s3_bucket" "Data_Bucket" {
  bucket = "mycompany-stuff"
}

resource "aws_lambda_function" "func" {
  function_name = "DoTheThing"
}
```

**问题：**
- 大小写不一致（snake_case、PascalCase、混合使用）
- 通用名称无法表达用途
- 资源名称与云资源名称不匹配

## 正确示例

```hcl
# Terraform 资源名称：snake_case，具有描述性
resource "aws_instance" "web_server" {
  tags = {
    Name = "prod-web-server-001" # 云资源名：环境-用途-编号
  }
}

resource "aws_s3_bucket" "application_data" {
  bucket = "acme-prod-application-data-us-east-1" # 组织-环境-用途-区域
}

resource "aws_lambda_function" "order_processor" {
  function_name = "prod-order-processor"
}

resource "aws_security_group" "web_server_sg" {
  name        = "prod-web-server-sg"
  description = "生产环境 Web 服务器安全组"
}
```

## 推荐命名模式

```
{org}-{environment}-{purpose}-{region}-{instance}
```

| 组成部分 | 示例 | 是否必填 |
|----------|------|----------|
| org | acme, myco | 可选 |
| environment | prod, staging, dev | 是 |
| purpose | web, api, db, cache | 是 |
| region | use1, usw2, euw1 | 视情况 |
| instance | 001, blue, primary | 视情况 |

## Terraform 资源名称

```hcl
# 使用 snake_case 命名 Terraform 资源
resource "aws_vpc" "main" {}             # 简单，单个 VPC
resource "aws_vpc" "application" {}      # 描述性用途
resource "aws_subnet" "public_a" {}      # 包含区域/用途
resource "aws_subnet" "private_database" {}

# 避免
resource "aws_vpc" "VPC" {}              # 冗余，大小写不当
resource "aws_subnet" "subnet1" {}       # 无描述性
```

## 使用 Locals 保持一致性

```hcl
locals {
  name_prefix = "${var.org}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-server"
  })
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs-${data.aws_region.current.name}"
  tags   = local.common_tags
}
```

## 参考资料

- [AWS 标签最佳实践](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html)
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)

---

# 为所有资源添加标签

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

标签是实现成本分摊、资源组织、自动化和合规的基础。没有一致的标签策略，你将无法按团队追踪成本、确定资源所有者或实现运维自动化。

## 错误示例

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  # 无标签 - 无法追踪归属或成本
}

resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"

  tags = {
    Name = "data" # 最小化、不一致的标签
  }
}
```

**问题：**
- 无法确定资源所有者
- 无法将成本分摊到团队/项目
- 无法识别用于自动化的资源
- 合规违规

## 正确示例

### 定义标准标签

```hcl
locals {
  required_tags = {
    Environment = var.environment
    Project     = var.project
    Owner       = var.owner
    ManagedBy   = "terraform"
    CostCenter  = var.cost_center
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = merge(local.required_tags, {
    Name = "${var.project}-${var.environment}-web"
    Role = "webserver"
  })
}

resource "aws_s3_bucket" "data" {
  bucket = "${var.project}-${var.environment}-data"

  tags = merge(local.required_tags, {
    Name     = "${var.project}-${var.environment}-data"
    DataClass = "internal"
    Backup   = "daily"
  })
}
```

### 标准标签模式

| 标签 | 是否必填 | 说明 | 示例 |
|------|----------|------|------|
| Environment | 是 | 部署环境 | prod, staging, dev |
| Project | 是 | 项目或应用名称 | myapp |
| Owner | 是 | 团队或个人负责人 | platform-team |
| ManagedBy | 是 | 资源管理方式 | terraform |
| CostCenter | 是 | 成本中心代码 | CC-12345 |
| Name | 推荐 | 人类可读名称 | myapp-prod-web |

### 使用默认标签（AWS Provider 3.38+）

```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      Owner       = var.owner
      ManagedBy   = "terraform"
      CostCenter  = var.cost_center
    }
  }
}

# 所有资源自动获得默认标签
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "${var.project}-${var.environment}-web"
    Role = "webserver"
  }
}
```

### 标签模块模式

创建可复用模块以保持标签一致性：

```hcl
# 标签模块的变量
variable "environment" {
  type        = string
  description = "环境名称"
}

variable "project" {
  type        = string
  description = "项目名称"
}

variable "owner" {
  type        = string
  description = "资源所有者"
}

variable "cost_center" {
  type        = string
  description = "成本中心代码"
}

variable "additional_tags" {
  type        = map(string)
  default     = {}
  description = "额外的标签"
}

# 输出合并后的标签
output "tags" {
  value = merge({
    Environment = var.environment
    Project     = var.project
    Owner       = var.owner
    ManagedBy   = "terraform"
    CostCenter  = var.cost_center
  }, var.additional_tags)
}
```

### 在根模块中使用

```hcl
module "tags" {
  source = "./modules/tags"

  environment = "prod"
  project     = "myapp"
  owner       = "platform-team"
  cost_center = "CC-12345"
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = merge(module.tags.tags, {
    Name = "myapp-prod-web"
  })
}
```

### 使用策略强制标签

```hcl
# AWS Config 规则强制标签
resource "aws_config_config_rule" "required_tags" {
  name = "required-tags"

  source {
    owner             = "AWS"
    source_identifier = "REQUIRED_TAGS"
  }

  input_parameters = jsonencode({
    tag1Key = "Environment"
    tag2Key = "Project"
    tag3Key = "Owner"
    tag4Key = "CostCenter"
  })
}
```

### 在 CI 中验证标签

```yaml
# .github/workflows/terraform.yml
- name: Check Required Tags
  run: |
    # 使用 tfsec 或自定义脚本验证标签
    tfsec . --tfvars-file=prod.tfvars
```

## 参考资料

- [AWS 标签最佳实践](https://docs.aws.amazon.com/general/latest/gr/aws_tagging.html)
- [AWS 默认标签](https://registry.terraform.io/providers/hashicorp/aws/latest/docs#default_tags)
- [成本分摊标签](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/cost-alloc-tags.html)

---

# 正确使用生命周期管理

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

生命周期块控制 Terraform 如何创建、更新和销毁资源。正确使用可以防止意外销毁、实现零停机部署，并处理资源管理的边缘情况。

## 错误示例

```hcl
# 无生命周期规则 - 对关键资源很危险
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"
  # 可能被 terraform destroy 意外删除！
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  # 默认行为：先销毁再创建 = 停机
}
```

## 正确示例

```hcl
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true # 防止意外删除
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true # 零停机替换
  }
}
```

## 生命周期参数

| 参数 | 用途 |
|------|------|
| `create_before_destroy` | 在销毁原始资源前先创建替换 |
| `prevent_destroy` | 防止意外删除 |
| `ignore_changes` | 忽略特定属性的变更 |
| `replace_triggered_by` | 依赖变更时强制替换 |

## create_before_destroy

### 问题

```hcl
# 默认行为：先销毁再创建
# 替换资源时导致停机

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
}

# AMI 变更时：
# 1. 销毁旧实例（停机开始）
# 2. 创建新实例（停机结束）
```

### 解决方案

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    create_before_destroy = true
  }
}

# AMI 变更时：
# 1. 创建新实例
# 2. 更新引用（负载均衡器等）
# 3. 销毁旧实例
# 零停机！
```

### 常用场景

```hcl
# 安全组（被实例引用）
resource "aws_security_group" "web" {
  name_prefix = "web-sg-"

  lifecycle {
    create_before_destroy = true
  }
}

# IAM 角色（被服务引用）
resource "aws_iam_role" "app" {
  name_prefix = "app-role-"

  lifecycle {
    create_before_destroy = true
  }
}

# 启动模板（被 ASG 引用）
resource "aws_launch_template" "web" {
  name_prefix = "web-lt-"

  lifecycle {
    create_before_destroy = true
  }
}
```

## prevent_destroy

### 保护关键资源

```hcl
# 防止意外删除生产数据库
resource "aws_db_instance" "production" {
  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.r5.large"

  lifecycle {
    prevent_destroy = true
  }
}

# 防止删除状态存储桶
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

# 防止删除加密密钥
resource "aws_kms_key" "main" {
  description = "主加密密钥"

  lifecycle {
    prevent_destroy = true
  }
}
```

### terraform destroy 行为

```bash
# 设置了 prevent_destroy = true 时
terraform destroy

# Error: Instance cannot be destroyed
# Resource aws_db_instance.production has lifecycle.prevent_destroy set

# 要实际销毁，先从配置中移除 prevent_destroy
```

## ignore_changes

### 忽略外部修改

```hcl
# 忽略由外部自动化管理的标签
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    ignore_changes = [
      tags["LastBackup"],     # 由备份系统更新
      tags["CostAllocation"], # 由成本工具更新
    ]
  }
}

# 忽略 ASG 期望容量（由自动伸缩管理）
resource "aws_autoscaling_group" "web" {
  name             = "web-asg"
  min_size         = 2
  max_size         = 10
  desired_capacity = 2 # 初始值

  lifecycle {
    ignore_changes = [desired_capacity]
  }
}
```

### 忽略所有变更

```hcl
# 资源由外部管理，仅在状态中跟踪
resource "aws_instance" "legacy" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  lifecycle {
    ignore_changes = all
  }
}
```

### 常用 ignore_changes 模式

```hcl
# EKS 集群版本（通过控制台/CLI 升级）
resource "aws_eks_cluster" "main" {
  name    = "main"
  version = "1.27"

  lifecycle {
    ignore_changes = [version]
  }
}

# Lambda 函数代码（单独部署）
resource "aws_lambda_function" "app" {
  function_name = "app"
  filename      = "placeholder.zip"

  lifecycle {
    ignore_changes = [
      filename,
      source_code_hash,
    ]
  }
}

# RDS 密码（在 Terraform 外管理）
resource "aws_db_instance" "main" {
  identifier = "main"
  password   = "initial-password"

  lifecycle {
    ignore_changes = [password]
  }
}
```

## replace_triggered_by

### 依赖变更时强制替换

```hcl
# 用户数据脚本变更时替换实例
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"
  user_data     = file("${path.module}/user-data.sh")

  lifecycle {
    replace_triggered_by = [
      null_resource.user_data_version
    ]
  }
}

resource "null_resource" "user_data_version" {
  triggers = {
    user_data_hash = filemd5("${path.module}/user-data.sh")
  }
}
```

### 模块输出变更时替换

```hcl
resource "aws_instance" "web" {
  ami           = module.ami.latest_id
  instance_type = "t3.micro"

  lifecycle {
    replace_triggered_by = [
      module.ami # 当任何模块输出变更时替换
    ]
  }
}
```

## 组合生命周期规则

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "prod"

    ignore_changes = [
      tags["LastBackup"],
    ]

    replace_triggered_by = [
      null_resource.deployment_trigger
    ]
  }
}
```

## 前置条件和后置条件

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    precondition {
      condition     = var.environment != "prod" || var.instance_type != "t3.micro"
      error_message = "生产环境实例必须大于 t3.micro"
    }

    postcondition {
      condition     = self.public_ip != null
      error_message = "实例必须有公网 IP"
    }
  }
}
```

## TypeSet 嵌套块的 ignore_changes 限制

### 问题：TypeSet 不支持索引访问

当嵌套块在 Provider schema 中定义为 `TypeSet` 时，`ignore_changes` **不能使用索引语法**：

```hcl
# 错误：TypeSet 不支持索引
lifecycle {
  ignore_changes = [
    servers[0].server_ip, # Cannot index a set value
  ]
}

# 正确：只能忽略整个 TypeSet 块
lifecycle {
  ignore_changes = [
    servers, # 忽略整个 servers 块
  ]
}
```

### 根因：TypeSet hash 包含所有 schema 字段

TypeSet 的 hash 计算包含 schema 中**所有字段**（含 Computed-only 字段），因此：
- 移除 dynamic content 中的字段赋值**不能**消除漂移
- Computed-only 字段（无 Optional）在 HCL 中不可赋值，零值永远不等于 API 返回值
- 唯一方案是 `ignore_changes = [block_name]`

> **详见**: `resource-typeset-computed-drift.md` — TypeSet 嵌套块 Computed-only 字段导致无限状态漂移的完整排查路径

## 参考资料

- [生命周期元参数](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)
- [前置条件和后置条件](https://developer.hashicorp.com/terraform/language/expressions/custom-conditions)
- `resource-typeset-computed-drift.md` - TypeSet 嵌套块 Computed 字段漂移专题

---

# 使用 for_each 替代 count 管理多资源

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

使用 `count` 配合列表时，资源通过索引标识。从列表中间删除元素会导致后续资源被销毁并重建。`for_each` 使用稳定的键，不受列表修改影响。

## 错误示例

```hcl
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "subnets" {
  count      = length(var.subnet_cidrs)
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index]

  tags = {
    Name = "subnet-${count.index}"
  }
}
```

**问题：** 如果从中间删除 `"10.0.2.0/24"`：
- `subnet[1]`（原来是 10.0.2.0/24）变成 10.0.3.0/24
- `subnet[2]`（原来是 10.0.3.0/24）被销毁
- subnet[1] 中的资源受到干扰

## 正确示例

```hcl
variable "subnets" {
  default = {
    "public-a" = "10.0.1.0/24"
    "public-b" = "10.0.2.0/24"
    "public-c" = "10.0.3.0/24"
  }
}

resource "aws_subnet" "subnets" {
  for_each   = var.subnets
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value

  tags = {
    Name = each.key
  }
}

# 引用特定子网
output "public_a_id" {
  value = aws_subnet.subnets["public-a"].id
}
```

## 列表转 Map

```hcl
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "subnets" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.value
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, index(var.availability_zones, each.value))

  tags = {
    Name = "subnet-${each.value}"
  }
}
```

## 何时使用 count

`count` 仍然适用于以下场景：

```hcl
# 布尔条件 - 创建或不创建
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0

  domain = "vpc"
}

# 固定数量的相同资源
resource "aws_nat_gateway" "gw" {
  count = var.nat_gateway_count # 例如 3

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

## 快速参考

| 场景 | 使用 |
|------|------|
| 创建 0 或 1 个资源 | `count` |
| 固定数量的相同资源 | `count` |
| 按名称/键标识的资源 | `for_each` |
| 可能删除元素的列表 | `for_each` + `toset()` |

## 参考资料

- [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
- [count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)

---

# 优先使用不可变基础设施

**优先级：** 中
**分类：** 资源组织

## 为什么重要

不可变基础设施通过替换组件而非就地修改来实现部署。这使部署更可预测、回滚更简单，并消除配置漂移。

## 错误示例

```hcl
# 可变实例 - SSH 进去修改
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"

  # 通过 SSH 执行命令部署
  provisioner "remote-exec" {
    inline = [
      "apt-get update",
      "apt-get install -y nginx",
      "systemctl start nginx"
    ]
  }
}

# 随时间推移产生配置漂移：
# - 应用了手动热修复
# - 安装了不同的包
# - 状态未知
```

**问题：**
- 实例间配置漂移
- 回滚需要复杂的状态管理
- 难以复现问题
- "在我机器上能用"但在生产环境不行

## 正确示例

### 使用 AMI/镜像实现不可变

```hcl
# 使用 Packer 构建不可变镜像
# packer/web-server.pkr.hcl
source "amazon-ebs" "web" {
  ami_name      = "web-server-${timestamp()}"
  instance_type = "t3.micro"
  source_ami_filter {
    filters = {
      name = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    }
    owners     = ["099720109477"]
    most_recent = true
  }
  ssh_username = "ubuntu"
}

build {
  sources = ["source.amazon-ebs.web"]

  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl enable nginx"
    ]
  }
}
```

```hcl
# 部署不可变镜像
data "aws_ami" "web" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["web-server-*"]
  }
}

resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = data.aws_ami.web.id
  instance_type = "t3.micro"

  # 无 provisioner - 镜像已预配置
}

resource "aws_autoscaling_group" "web" {
  desired_capacity = 3
  max_size         = 6
  min_size         = 3

  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }

  # 滚动更新 - 用新镜像替换实例
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }
}
```

### 使用容器实现不可变

```hcl
# 构建并推送容器镜像
resource "null_resource" "docker_build" {
  triggers = {
    dockerfile_hash = filemd5("${path.module}/Dockerfile")
    app_hash        = filemd5("${path.module}/app.py")
  }

  provisioner "local-exec" {
    command = <<-EOT
      docker build -t ${var.ecr_repo}:${var.app_version} .
      docker push ${var.ecr_repo}:${var.app_version}
    EOT
  }
}

# 部署不可变容器
resource "aws_ecs_task_definition" "app" {
  family = "app"

  container_definitions = jsonencode([{
    name  = "app"
    image = "${var.ecr_repo}:${var.app_version}" # 指定版本，不是 :latest
    # ...
  }])
}

resource "aws_ecs_service" "app" {
  name            = "app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 50
  }
}
```

### 蓝绿部署

```hcl
variable "active_color" {
  description = "当前活跃的部署（blue 或 green）"
  default     = "blue"
}

resource "aws_launch_template" "blue" {
  name_prefix = "blue-"
  image_id    = var.blue_ami_id
  # ...
}

resource "aws_launch_template" "green" {
  name_prefix = "green-"
  image_id    = var.green_ami_id
  # ...
}

resource "aws_lb_target_group" "blue" {
  name = "blue-tg"
  # ...
}

resource "aws_lb_target_group" "green" {
  name = "green-tg"
  # ...
}

# 通过切换活跃颜色来切换流量
resource "aws_lb_listener_rule" "app" {
  listener_arn = aws_lb_listener.front_end.arn

  action {
    type             = "forward"
    target_group_arn = var.active_color == "blue" ? aws_lb_target_group.blue.arn : aws_lb_target_group.green.arn
  }

  condition {
    path_pattern {
      values = ["/*"]
    }
  }
}
```

### 不可变性的生命周期规则

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  lifecycle {
    # 先创建新实例再销毁旧实例
    create_before_destroy = true

    # AMI 变更时强制替换
    replace_triggered_by = [
      null_resource.ami_version
    ]
  }
}

# 版本变更时触发替换
resource "null_resource" "ami_version" {
  triggers = {
    ami_id = var.ami_id
  }
}
```

### 何时可以使用可变方式

有些资源天生是有状态的：

```hcl
# 数据库 - 使用生命周期规则，而非替换
resource "aws_db_instance" "main" {
  identifier = "mydb"
  # ...

  lifecycle {
    prevent_destroy = true       # 防止意外删除
    ignore_changes  = [password] # 在 Terraform 外管理
  }
}

# 使用快照实现"类不可变"的数据库更新
resource "aws_db_instance" "main" {
  snapshot_identifier = var.restore_from_snapshot # 从快照部署
}
```

## 不可变性光谱

| 级别 | 说明 | 示例 |
|------|------|------|
| 完全可变 | 就地修改 | SSH 编辑配置 |
| 配置管理 | 自动化就地更新 | Ansible/Chef 执行 |
| 不可变镜像 | 替换而非修改 | AMI、Docker 镜像 |
| 不可变基础设施 | 替换整个堆栈 | 蓝绿部署、金丝雀发布 |

## 参考资料

- [HashiCorp Packer](https://developer.hashicorp.com/packer)
- [不可变基础设施](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure)
- [蓝绿部署](https://martinfowler.com/bliki/BlueGreenDeployment.html)

---

# 资源规格自动对齐：防止状态漂移

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

某些云资源的参数由资源规格代码（instance_class）决定，而非用户配置。当用户设置的值与规格固定值不匹配时，会导致状态漂移，因为 API 会忽略用户设置，使用规格决定的值。

## 问题：忽略参数导致状态漂移

### 示例：阿里云 Redis 经典版集群

```
instance_class = "redis.sharding.basic.small.default"  # 固定：8分片 × 2GB
shard_count    = 2   # 用户设置 - API 忽略
capacity       = 2048 # 用户设置 - API 忽略
```

**后果：**

1. API 创建实例时使用 `shard_count=8, capacity=16384`（规格固定值）
2. State 记录 API 返回的实际值
3. 用户 tfvars 仍然是 `shard_count=2, capacity=2048`
4. 下次 plan：Code ≠ State → **触发重建！**

### 根本原因

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 经典版集群：shard_count 和 capacity 由 instance_class 决定                   │
│                                                                             │
│ redis.sharding.basic.small.default = 8分片 × 2GB = 16GB 总容量              │
│ redis.sharding.basic.medium.default = 8分片 × 4GB = 32GB 总容量             │
│                                                                             │
│ 用户参数被忽略 - 仅云原生版（.ce）支持自定义                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 解决方案：原子层自动对齐

### 模式：规格值映射

```hcl
# atomic/redis-community/main.tf

locals {
  # 检测部署类型
  is_classic = can(regex("\\.default$", var.instance_class))
  is_cloud_native = can(regex("\\.ce$", var.instance_class))
  is_cluster = can(regex("sharding|with\\.proxy", lower(var.instance_class)))
  is_classic_cluster = local.is_classic && local.is_cluster

  # 规格固定值映射
  classic_cluster_specs = {
    "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }
    "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }
    "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }
    "redis.sharding.basic.xlarge.default"   = { shard_count = 8,  capacity_mb = 131072 }
  }

  # 自动对齐：经典版集群使用规格固定值，其他使用用户值
  effective_shard_count = local.is_classic_cluster ? (
    try(local.classic_cluster_specs[var.instance_class].shard_count, var.shard_count)
  ) : (local.is_cluster ? var.shard_count : null)

  effective_capacity = local.is_classic_cluster ? (
    try(local.classic_cluster_specs[var.instance_class].capacity_mb, var.capacity)
  ) : var.capacity
}

resource "alicloud_kvstore_instance" "this" {
  shard_count = local.effective_shard_count
  capacity    = local.effective_capacity
}
```

## 决策矩阵

| 资源类型 | 参数控制 | 用户设置 | 原子层行为 |
|----------|----------|----------|------------|
| 经典版 `.default` | 规格固定 | 任意值 | **自动对齐**到规格值 |
| 云原生版 `.ce` | 用户控制 | 用户值 | 使用用户值 |
| 倚天版 `.y.ee` | 用户控制 | 用户值 | 使用用户值 |

## 何时应用此模式

1. **识别忽略参数**：检查 Provider 文档中"由 instance_class 决定"的参数
2. **创建规格映射**：建立 instance_class → 固定值的映射
3. **实现自动对齐**：使用 locals 将用户值覆盖为规格固定值
4. **记录限制**：添加注释说明行为

## 常见需要自动对齐的场景

| 云厂商 | 资源类型 | 忽略的参数 |
|--------|----------|------------|
| 阿里云 | Redis 经典版集群 | shard_count, capacity |
| 阿里云 | Redis 经典版标准版 | capacity（部分规格） |
| AWS | RDS 实例 | IOPS（部分实例类型） |
| AWS | EC2 实例 | bandwidth（部分实例类型） |

## 验证检查清单

- [ ] 识别由 instance_class/规格代码决定的参数
- [ ] 创建包含所有有效 instance_class 值的规格映射
- [ ] 在 locals 中实现自动对齐逻辑
- [ ] 添加清晰的注释说明行为
- [ ] 使用不匹配的用户值测试 - 不应导致漂移
- [ ] 记录哪些规格支持自定义值 vs 固定值

## 决策规则

> **当参数由资源规格代码（instance_class）决定时，原子层必须将用户值自动对齐为规格固定值，防止 API 忽略参数导致状态漂移。**

## 参考资料

- `resource-lifecycle.md`（生命周期块）
- `variable-atomic-defaults.md`（原子层默认值原则）
- `provider-documentation-lookup.md`（Provider 文档查找规范）

---

# for_each 与 unknown 值：dynamic 块优先策略

**优先级：** 关键
**分类：** 资源组织

## 为什么重要

当控制层 `servers` 列表通过 `if` 过滤空 `server_id` 时，如果 `server_id` 来自 `lookup(local.ecs_ids_map, ...)`，在 ECS 首次创建时该值为 `known after apply`。Terraform 要求 `resource for_each` 的 map **key 集合**在 plan 阶段可确定——`if` 条件依赖 unknown 值会导致整个 map 变为 unknown，触发 `Invalid for_each argument` 错误。

**与 `module-dependency-prefilter` 的区别**：预过滤解决的是 `create=false` 导致模块实例不存在的问题（plan-time 可判断）；本规则解决的是 **value 为 unknown 导致 for_each key 集合不可确定**的问题（apply-time 才知道）。

## 根因分析

```
控制层: servers = [for s in var.servers : ... if lookup(local.ecs_ids_map, s.ecs_key, "") != ""]
                        ↑                              ↑
                   var.servers 长度已知           ECS ID 是 known after apply
                   (plan-time)                    (apply-time)

结果: if 条件依赖 unknown 值 → 列表长度 unknown → for_each map key 集合 unknown → 报错
```

| 求值阶段 | `if` 过滤 | `dynamic` 块 | `for_each`（无 if） |
|----------|----------|-------------|-------------------|
| plan | 列表长度 unknown → 报错 | 不影响 resource 数量 | 列表长度已知（来自 tfvars） |
| apply | 正常 | 正常 | server_id 已有值 |

## 决策矩阵：两种模式

### 模式 A：原子层改用 `dynamic` 块（优先推荐）

> 适用场景：后端服务器是资源的**嵌套属性**（如 ALB ServerGroup servers、SLB Backend servers）

**原理**：`dynamic` 块的 `for_each` 不影响 resource 数量，允许 unknown 值。控制层可安全使用 `if` 过滤。

```hcl
# ✅ 原子层：dynamic 块
resource "alicloud_slb_backend_server" "this" {
  count = local.create ? 1 : 0

  load_balancer_id = var.load_balancer_id

  dynamic "backend_servers" {
    for_each = [for s in var.servers : {
      server_id = s.server_id
      weight    = lookup(s, "weight", 100)
      type      = lookup(s, "type", "ecs")
    }]
    content {
      server_id = backend_servers.value.server_id
      weight    = backend_servers.value.weight
      type      = backend_servers.value.type
    }
  }
}
```

```hcl
# ✅ 控制层：可以安全使用 if 过滤（dynamic 块允许 unknown 的 for_each）
module "slb_backend_attachment" {
  source   = "../../atomic/slb-backend-attachment"
  for_each = var.slb_backend_attachments

  create = try(each.value.create, true) && lookup(local.slb_would_create, each.value.load_balancer_key, false)
  load_balancer_id = lookup(local.slb_ids_map, each.value.load_balancer_key, null)
  servers = [
    for s in try(each.value.servers, []) : merge(
      { for k, v in s : k => v if k != "ecs_key" },
      try(s.ecs_key, null) != null ? {
        server_id = lookup(local.ecs_ids_map, s.ecs_key, "")
      } : {}
    )
    if (try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")) != ""
  ]
  depends_on = [module.ecs]
}
```

### 模式 B：原子层保持 `resource for_each`，控制层用 `ecs_would_create` 预过滤

> 适用场景：后端服务器是**独立资源**（如 SLB Server Group Attachment、NLB Server Group Attachment）

**原理**：独立资源必须用 `resource for_each`，无法改为 `dynamic` 块。不能直接用 `lookup(local.ecs_ids_map, ...) != ""` 做 `if` 过滤（依赖 apply-time 值），但可以用 `var.ecs_instances` 的 **plan-time 已知属性** 预计算哪些 ECS 会创建，再用这个集合做 `if` 过滤。

**关键区分**：
- `lookup(local.ecs_ids_map, key, "") != ""` → 依赖 apply-time 值 → ❌ 不可用
- `contains(local.ecs_would_create, key)` → 纯 var 计算 → ✅ plan-time 可确定

```hcl
# ✅ 步骤 1：在 locals 中预计算 ECS 会创建的 key 集合
# 条件必须与 ECS 模块的 create 一致：try(v.create, try(v.image_id, "") != "")
locals {
  ecs_would_create = toset([
    for k, v in var.ecs_instances : k
    if try(v.create, try(v.image_id, "") != "")
  ])
}
```

```hcl
# ✅ 步骤 2：原子层用静态 key 的 for_each
locals {
  servers_map = local.create && length(var.servers) > 0 ? {
    for idx, obj in var.servers : "server-${idx}" => {
      server_id = obj.server_id
      port      = obj.port
      weight    = lookup(obj, "weight", 100)
    }
  } : {}
}

resource "alicloud_nlb_server_group_server_attachment" "this" {
  for_each = local.servers_map

  server_group_id = var.server_group_id
  server_id       = each.value.server_id
  port            = each.value.port
  weight          = each.value.weight
  server_type     = each.value.type
}
```

```hcl
# ✅ 步骤 3：控制层用 ecs_would_create 预过滤（plan-time 安全）
module "nlb_server_group_server_attachment" {
  source   = "../../atomic/nlb-server-group-server-attachment"
  for_each = var.nlb_server_groups

  create          = try(each.value.create, true)
  server_group_id = lookup(local.nlb_server_group_ids_map, each.key, null)
  servers = [
    for s in try(each.value.servers, []) : merge(s, {
      server_id = try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")
    })
    # ✅ 用 ecs_would_create 过滤 create=false 的 ECS（plan-time 安全）
    if try(s.ecs_key, null) != null ? contains(local.ecs_would_create, s.ecs_key) : try(s.server_id, "") != ""
  ]

  depends_on = [module.ecs, module.nlb_server_group]
}
```

## 分类速查表

| 资源 | 嵌套方式 | 推荐模式 | 控制层 if 过滤 | 原子层实现 |
|------|---------|---------|--------------|-----------|
| ALB ServerGroup servers | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "servers"` |
| SLB Backend servers | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "backend_servers"` |
| SLB Server Group Attachment | 独立 resource | B: for_each | **ecs_would_create** | `resource for_each` + 静态 key |
| NLB Server Group Attachment | 独立 resource | B: for_each | **ecs_would_create** | `resource for_each` + 静态 key |
| ALB Listener actions | `dynamic` 嵌套块 | A: dynamic | 保留 | `dynamic "actions"` |
| Route Entry (NAT nexthop) | 独立 resource | B: for_each | **nat_gateway_creates** | `resource for_each` + 静态 key |

## 判断流程

```
后端服务器配置是否是独立 resource？
├─ 否 → 是资源的嵌套属性 → 模式 A（dynamic 块）
│        控制层保留 if 过滤，原子层改用 dynamic
└─ 是 → 独立 resource for_each → 模式 B（ecs_would_create 预过滤）
         控制层用 ecs_would_create 做 if 过滤（plan-time 安全）
         原子层用 "server-${idx}" 作静态 key
         控制层添加 depends_on = [module.ecs]
```

## 常见陷阱

### 陷阱 1：在 resource for_each 场景用 apply-time 值做 if 过滤

```hcl
# ❌ 错误：if 条件依赖 apply-time 值，for_each key 集合 unknown
servers = [
  for s in var.servers : merge(s, {
    server_id = lookup(local.ecs_ids_map, s.ecs_key, "")
  })
  if lookup(local.ecs_ids_map, s.ecs_key, "") != ""  # ← known after apply
]
# 报错：Invalid for_each argument
```

**正确做法**：用 `ecs_would_create`（基于 var 的 plan-time 已知属性）替代 `ecs_ids_map`（apply-time 值）：

```hcl
# ✅ 正确：if 条件只依赖 plan-time 已知值
servers = [
  for s in try(each.value.servers, []) : merge(s, {
    server_id = try(s.ecs_key, null) != null ? lookup(local.ecs_ids_map, s.ecs_key, "") : try(s.server_id, "")
  })
  if try(s.ecs_key, null) != null ? contains(local.ecs_would_create, s.ecs_key) : try(s.server_id, "") != ""
]
```

### 陷阱 2：忘记添加 depends_on

```hcl
# ❌ 错误：没有 depends_on，ECS 和 NLB Attachment 可能并行创建
# 即使 ecs_would_create 过滤了 create=false 的 ECS，首次创建时仍需保证 ECS 先完成
module "nlb_attachment" {
  source   = "../../atomic/nlb-server-group-server-attachment"
  for_each = var.nlb_server_groups
  # 缺少 depends_on = [module.ecs]
}
```

### 陷阱 2b：ecs_would_create 条件与 ECS 模块的 create 不一致

```hcl
# ❌ 错误：ecs_would_create 用了简化条件，与 ECS 模块实际 create 不一致
# ECS 模块的 create = try(each.value.create, each.value.image_id != "")
# 如果 ecs_would_create 只检查 create 字段，image_id 回退逻辑会被遗漏
ecs_would_create = toset([
  for k, v in var.ecs_instances : k
  if try(v.create, true)  # ← 遗漏了 image_id 回退条件
])
# 结果：image_id="" 但未显式设 create=false 的 ECS 不会被过滤，实际 ECS 不创建
```

### 陷阱 3：把 dynamic 块用于需要独立生命周期管理的资源

```hcl
# ❌ 错误：dynamic 块内的资源不能单独 import/replace/taint
# 如果需要独立管理每个 attachment，必须用 resource for_each
```

## 检查清单

- [ ] 识别原子层的后端服务器是嵌套属性还是独立资源
- [ ] 嵌套属性 → 原子层改用 `dynamic` 块，控制层可安全用 `if lookup(ecs_ids_map,...)` 过滤
- [ ] 独立资源 → 原子层用静态 key 的 `for_each`
- [ ] 独立资源场景 → 控制层用 `ecs_would_create`（plan-time 安全）做 `if` 过滤，不用 `ecs_ids_map`
- [ ] `ecs_would_create` 条件与 ECS 模块的 `create` 条件完全一致
- [ ] 所有场景 → 控制层添加 `depends_on = [module.ecs]`
- [ ] 原子层 header 注释说明设计选择（dynamic 或静态 key 的原因）

## 参考资料


---

# 同类资源批次依赖串行化（Batch Dependency Serialization）

**优先级：** 关键
**分类：** 资源组织

## 为什么重要

当云 API 对同一父资源的子操作加资源锁（如 ALB 同一实例的 ACL attachment 不允许并发），而 Terraform 的 `count`/`for_each` **所有实例共享一个依赖图节点**，无法创建实例间依赖。这会导致并发操作触发云 API 错误（如 `LockFailed`），且无法通过 Terraform 层面的单资源块解决。

**核心矛盾**：
- 云 API：同一父资源的子操作必须串行
- Terraform：`count`/`for_each` 实例间无法建立依赖（`this[count.index-1]` 自引用 → Cycle）
- Provider：create 路径可能不重试锁冲突错误

## 错误方案：coalesce 自引用（Cycle）

```hcl
# ❌ 错误：同一 resource block 内实例间引用 → Terraform 检测为 Cycle
resource "alicloud_alb_listener_acl_attachment" "this" {
  count = length(local.attachments)

  listener_id = coalesce(
    lookup(local.listener_ids_map, local.attachments[count.index].listener_key, null),
    # ↓ 自引用：alicloud_alb_listener_acl_attachment.this 引用自身 → Cycle Error
    count.index > 0 ? alicloud_alb_listener_acl_attachment.this[count.index - 1].listener_id : null,
  )
}
```

**错误信息**：`Error: Cycle: module.xxx.resource_name.this[1], module.xxx.resource_name.this[0]`

**根因**：Terraform 将 `count`/`for_each` 资源视为单个依赖图节点，所有实例属于同一节点，节点内无法建立顺序依赖。

**依据**：HashiCorp Issue [#10802](https://github.com/hashicorp/terraform/issues/10802) — 官方确认的唯一解决方案是拆分为多个独立 resource block。

## 正确方案：Batch 分组 + 多 resource block

**核心思路**：按父资源分组，同一父资源内的操作按序分配到不同 batch（0/1/2），每个 batch 用独立 resource block + `for_each`，`batch[n]` 通过 `depends_on` 依赖 `batch[n-1]`。

**效果**：不同父资源的操作并发（性能无损），同一父资源内串行（避免锁冲突）。

### Step 1：展平 + 分组 locals

```hcl
locals {
  # 展平所有操作项，带 load_balancer_key 用于父资源分组
  acl_flat = flatten([
    for lk, lv in var.listeners : [
      for ak in try(lv.acl_keys, []) : {
        listener_key      = lk
        load_balancer_key = lv.load_balancer_key  # 父资源标识，用于分组
        acl_key           = ak
        acl_type          = try(lv.acl_type, "White")
        key               = "${lk}-${ak}"          # for_each 的稳定 key
      }
      if try(lv.acl_keys, null) != null
    ]
    if try(lv.acl_keys, null) != null
      && lookup(local.listener_would_create, lk, false)
  ])

  # 为同一父资源的操作分配 batch 序号
  # batch = 同一 load_balancer_key 内、展平顺序在此项之前的操作数量
  acl_batched = [
    for i, v in local.acl_flat : merge(v, {
      batch = length([
        for j, w in local.acl_flat : w
        if w.load_balancer_key == v.load_balancer_key && j < i
      ])
    })
  ]

  # 按 batch 分组为 map（for_each 需要 map）
  acl_batch0 = { for v in local.acl_batched : v.key => v if v.batch == 0 }
  acl_batch1 = { for v in local.acl_batched : v.key => v if v.batch == 1 }
  acl_batch2 = { for v in local.acl_batched : v.key => v if v.batch == 2 }
}
```

### Step 2：多个独立 resource block

```hcl
# Batch 0：首批，所有父资源的第一个操作并发执行
resource "alicloud_alb_listener_acl_attachment" "batch0" {
  for_each = local.acl_batch0

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [module.alb_listener]
}

# Batch 1：同一父资源的第 2 个操作，等 batch0 全部完成后执行
resource "alicloud_alb_listener_acl_attachment" "batch1" {
  for_each = local.acl_batch1

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [alicloud_alb_listener_acl_attachment.batch0]
}

# Batch 2：同一父资源的第 3 个操作，等 batch1 全部完成后执行
resource "alicloud_alb_listener_acl_attachment" "batch2" {
  for_each = local.acl_batch2

  listener_id = lookup(local.listener_ids_map, each.value.listener_key, null)
  acl_id      = lookup(local.acl_ids_map, each.value.acl_key, null)
  acl_type    = each.value.acl_type

  depends_on = [alicloud_alb_listener_acl_attachment.batch1]
}
```

### 执行时序图

```
                     时间轴 →
ALB-A:  [batch0: attach-1] ──→ [batch1: attach-2] ──→ [batch2: attach-3]
ALB-B:  [batch0: attach-1] ──→ [batch1: attach-2]
             ↓ 并发                  ↓ 等batch0完成          ↓ 等batch1完成
```

## batch 序号计算原理

```hcl
# 示例：2个 ALB，每个有 2-3 个 ACL attachment
# 展平顺序：A-1, A-2, B-1, B-2
#
# A-1: load_balancer_key=A, 之前同 ALB 的数量=0 → batch=0
# A-2: load_balancer_key=A, 之前同 ALB 的数量=1 → batch=1
# B-1: load_balancer_key=B, 之前同 ALB 的数量=0 → batch=0
# B-2: load_balancer_key=B, 之前同 ALB 的数量=1 → batch=1
#
# 结果：
# batch0 = { A-1, B-1 }  → 并发
# batch1 = { A-2, B-2 }  → 等 batch0 后并发
```

## 方案决策矩阵

| 方案 | 可行性 | 性能 | 复杂度 | 问题 |
|------|--------|------|--------|------|
| count + coalesce 自引用 | ❌ Cycle | - | - | Terraform 禁止同资源实例间引用 |
| 限制 -parallelism=1 | ✅ | ❌ 全局串行 | 低 | 所有资源串行，性能严重退化 |
| 批次分组（本方案） | ✅ | ✅ 按需串行 | 中 | batch 数量固定（通常 3 个足够） |
| 修改 Provider 重试逻辑 | ✅ | ✅ | 高 | 需维护 fork 版本，不可控 |
| 分多次 apply | ✅ | ❌ | 高 | 运维成本高，不适合自动化 |

## 适用场景判断

满足以下所有条件时使用本方案：
1. **云 API 资源锁**：同一父资源的子操作不允许并发（如 LockFailed、Conflict 等错误）
2. **Provider 不重试**：create 路径的重试列表不包含锁冲突错误码
3. **操作数量可控**：同一父资源的最大子操作数 ≤ batch 数量（本方案固定 3）

## 实战案例：ALB ACL Attachment LockFailed

**错误场景**：阿里云 ALB 同一实例的多个 Listener 并发绑定 ACL 时触发 `LockFailed`（StatusCode: 400）。

**Provider 源码依据**：
- create 路径（`resource_alicloud_alb_listener_acl_attachment.go:73`）：重试列表 `["ResourceInConfiguring.Listener", "IncorrectStatus.Listener", "Conflict.Acl"]`，**不含** `LockFailed`
- delete 路径（同文件 L158）：重试列表**包含** `LockFailed`

**改造范围**：
1. 原子层 `alb-listener`：移除 `alicloud_alb_listener_acl_attachment` 资源，`acl_config` 改为透传输出
2. 控制层 `web-cluster`：新增 locals 批次分组 + 3 个独立 resource block

## 注意事项

- batch 数量（通常 3 个）基于 `alicloud_alb_listener_acl_attachment` 文档限制："You can associate at most three ACLs with a listener"
- `for_each` 使用稳定 key（`${listener_key}-${acl_key}`），避免 count 的索引漂移问题
- `depends_on` 跨 resource block 建立串行链，是 Terraform 依赖图中合法的操作

## 相关规则

- `resource-count-vs-foreach` - for_each 优先于 count
- `module-null-safe-dependency` - 父资源未创建时子资源安全降级
- `module-dependency-prefilter` - 模块依赖预过滤
- `layer-id-mapping` - 通过 map 变量实现基于 key 的资源引用

## 参考资料

- `resource-count-vs-foreach.md`（for_each vs count）
- `module-dependency-prefilter.md`（依赖预过滤）
- `control-nested-resource-flatten.md`（嵌套资源展平）

---

# API 参数字符限制处理规范

**优先级：** 高
**分类：** 云 API 兼容性

## 为什么重要

云产品 API 对某些参数有严格的字符限制（如不允许空格、横杠等），但 Terraform 自动生成的命名可能包含这些字符。如果不处理，会导致 `IllegalCharacters` 错误。

## 已知限制

### 1. NAS Access Group — Description 不允许空格和横杠

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 产品                        │ NAS（所有类型：standard/extreme/cpfs）          │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 参数                        │ Description                                   │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 限制                        │ 仅允许字母、数字、下划线（不允许空格、横杠）      │
├────────────────────────────┼───────────────────────────────────────────────┤
│ AccessGroupName            │ 允许字母、数字、下划线、横杠                    │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 影响范围                    │ CreateAccessGroup / ModifyAccessGroup          │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 2. NAS Access Rule — 需要 file_system_type 关联

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ Access Rule 必须传 file_system_type 才能找到    │
│                             │ 对应类型的 Access Group                        │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 错误                        │ InvalidAccessGroup.NotFound (404)              │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 原因                        │ 不传 file_system_type 时，Provider 按 default  │
│                             │ 类型查找，找不到 extreme 类型的 Access Group    │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 3. Tair Account — instance_id 代理模式漂移

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ 代理模式下 Provider 读取 account 返回的         │
│                             │ instance_id 带后缀 "-db-{N}"（节点 ID）         │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 错误                        │ 每次 plan 都触发 replacement                   │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 修复                        │ lifecycle { ignore_changes = [instance_id] }   │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 4. NAS effective_access_group_name — try() 安全访问

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ access_group 未创建时，引用 this[0] 会报        │
│                             │ Invalid index（空列表）                         │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 修复                        │ try(alicloud_nas_access_group.this[0].name,    │
│                             │     var.access_group_name)                     │
└────────────────────────────┴───────────────────────────────────────────────┘
```

## 通用修复模式

### 模式 1：字符净化（replace）

```hcl
# 原子层：将横杠替换为下划线，确保 API 兼容
description = "${replace(var.nas_name, "-", "_")}_access_group"
# 结果: "nas_simple_02_access_group" (无横杠、无空格)
```

### 模式 2：补充关联参数

```hcl
# 子资源必须传递父资源的类型参数，确保 API 查找正确
resource "alicloud_nas_access_rule" "this" {
  access_group_name = alicloud_nas_access_group.this[0].access_group_name
  file_system_type  = var.file_system_type != "" ? var.file_system_type : null  # 关键！
  ...
}
```

### 模式 3：ignore_changes 处理 Provider 漂移

```hcl
# Provider 读取的值与创建时传入的值不一致（Computed 差异）
resource "alicloud_kvstore_account" "this" {
  instance_id = alicloud_redis_tair_instance.this[0].id
  ...
  lifecycle {
    ignore_changes = [instance_id]  # 防止 "-db-{N}" 后缀导致无限漂移
  }
}
```

### 模式 4：try() 安全引用

```hcl
# 条件创建的资源，引用前用 try() 防止空列表索引错误
effective_name = local.create_sub_resource 
  ? try(parent_resource.this[0].name, var.default_name) 
  : var.default_name
```

## 调试方法论

1. **CLI 对比验证**：先用 aliyun CLI 测试 API 调用，排除 Provider 问题
2. **逐步排除参数**：逐个参数测试，确定哪个参数导致 IllegalCharacters
3. **Provider TRACE 日志**：`TF_LOG=DEBUG` 查看 Provider 实际发送的 API 请求
4. **Skills 三合一**：结合官方 API 文档 + Provider 源码 + 实际测试

## 发现时间

- 2026-04-17: NAS Access Group Description 限制 + Access Rule file_system_type 缺失
- 2026-04-17: Tair Account instance_id 代理模式漂移
- 2026-04-17: NAS effective_access_group_name try() 修复

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [阿里云 OpenAPI 调试台](https://next.api.aliyun.com/)

---

# TypeSet 嵌套块 Computed-only 字段导致无限状态漂移

**优先级：** 高
**分类：** 常见陷阱 / 状态漂移

## 为什么重要

当 Provider 将嵌套块定义为 `schema.TypeSet` 且其内部包含 `Computed: true` only（无 `Optional`）的字段时，Terraform SDKv2 的 TypeSet hash 计算会包含这些不可配置字段，导致配置零值（`""`）与 API 返回值永远不等，产生**无限循环漂移**。

这类问题无法通过精简 dynamic content 解决，因为 TypeSet hash 基于 schema 全字段计算，不受 HCL content 赋值范围影响。

## 问题模式：TypeSet + Computed-only 嵌套字段

### 现象

```
每次 terraform plan 都检测到嵌套块变更：
~ servers {
  ~ server_group_id = "sgp-xxx" -> ""
  ~ status = "Available" -> ""
  ~ server_ip = "172.31.x.x" -> ""
}

apply 后再次 plan，仍然出现相同变更 → 无限循环！
```

### 根因链路

```
1. Provider schema 定义嵌套块为 TypeSet
↓
2. TypeSet hash 基于 schema 中所有字段计算（含 Computed only 字段）
↓
3. Computed only 字段（如 server_group_id、status）在 HCL 中不可赋值
↓
4. 配置中这些字段为零值（""），API 返回实际值（"sgp-xxx"、"Available"）
↓
5. 零值 ≠ API 返回值 → TypeSet hash 不匹配 → 判定元素变更
↓
6. apply 更新 state → 下次 plan 又检测到差异 → 无限循环漂移
```

### Provider 源码特征

```go
// 典型 TypeSet 嵌套块定义（含 Computed only 字段）
"servers": {
  Type: schema.TypeSet, // ← 关键：TypeSet
  Optional: true,
  Elem: &schema.Resource{
    Schema: map[string]*schema.Schema{
      // Computed only 字段 — 根因所在
      "status": {
        Type: schema.TypeString,
        Computed: true, // 无 Optional，HCL 不可赋值
      },
      "server_group_id": {
        Type: schema.TypeString,
        Computed: true, // 无 Optional，HCL 不可赋值
      },
      // Optional + Computed 字段 — 可能加剧漂移
      "server_ip": {
        Type: schema.TypeString,
        Optional: true,
        Computed: true, // 不赋值时 API 自动填充
      },
      // Required / Optional only 字段 — 正常
      "server_id": {
        Type: schema.TypeString,
        Required: true,
      },
    },
  },
}
```

## 判断标准

### 何时需要排查此问题

| 条件 | 说明 |
|------|------|
| 嵌套块定义为 `TypeSet`（非 `TypeList`） | TypeSet hash 包含所有 schema 字段 |
| 嵌套块内部存在 `Computed: true` only 字段 | HCL 不可赋值，零值必然与 API 返回值不同 |
| 使用 `dynamic` 块生成嵌套内容 | dynamic content 中无法控制 Computed only 字段 |

### TypeSet vs TypeList 对比

| 特性 | TypeSet | TypeList |
|------|---------|----------|
| 元素标识 | 基于 hash（包含所有 schema 字段） | 基于索引位置 |
| hash 计算范围 | schema 中**所有字段**（含 Computed only） | 同上，但对比逻辑不同 |
| 索引访问 | `Cannot index a set value` | 支持 `list[0]` |
| ignore_changes 粒度 | 只能忽略整个块 `ignore_changes = [block_name]` | 可忽略特定属性 `block_name[0].attr` |
| Computed only 字段影响 | **导致 hash 不匹配，无限漂移** | 通常不影响（按位置匹配） |

## 解决方案

### 唯一可行方案：ignore_changes 整个块

```hcl
resource "alicloud_alb_server_group" "this" {
  # ... 其他配置 ...

  dynamic "servers" {
    for_each = var.servers
    content {
      server_id = servers.value.server_id
      server_type = servers.value.server_type
      port = servers.value.port
      weight = lookup(servers.value, "weight", 100)
      description = lookup(servers.value, "description", null)
      # Computed only 字段（server_group_id、status）不赋值
      # Optional+Computed 字段（server_ip、remote_ip_enabled）不赋值
    }
  }

  # 忽略 servers 块整体变更（Provider TypeSet + Computed 字段漂移）
  # 变更 servers 配置时需临时注释此行，apply 后恢复
  lifecycle {
    ignore_changes = [
    servers,
    ]
  }
}
```

### 排除的方案及原因

| 方案 | 排除原因 | 验证方式 |
|------|---------|---------|
| `ignore_changes = [servers[0].server_ip]` | TypeSet 不支持索引访问，报 `Cannot index a set value` | terraform apply |
| 移除 content 中的 Optional+Computed 字段 | 不够——Computed only 字段（无 Optional）仍导致 hash 不匹配 | 第二次 plan 验证 |
| 在 content 中显式赋值 `server_ip = null` | 加剧漂移——null ≠ API 返回的实际 IP | plan 输出对比 |
| 修复 Provider（TypeSet→TypeList） | 不在 HCL 层面控制范围 | Provider 源码 |

### 副作用及应对

```
┌─────────────────────────────────────────────────────────────┐
│ servers 变更操作流程 │
├─────────────────────────────────────────────────────────────┤
│ │
│ 1. 注释 lifecycle { ignore_changes = [servers] } │
│ ↓ │
│ 2. 修改 dynamic content 或 var.servers │
│ ↓ │
│ 3. terraform plan 确认变更内容 │
│ ↓ │
│ 4. terraform apply │
│ ↓ │
│ 5. 恢复 lifecycle { ignore_changes = [servers] } │
│ ↓ │
│ 6. terraform plan 确认无额外漂移 │
│ │
└─────────────────────────────────────────────────────────────┘
```

## 排查路径总结（成功路径）

```
┌──────────────────────────────────────────────────────────────────────┐
│ TypeSet+Computed 漂移排查路径 │
├──────────────────────────────────────────────────────────────────────┤
│ │
│ Step 1: 确认漂移现象 │
│ ───────────────────── │
│ 第二次 terraform plan 仍检测到变更（非首次 apply 后的初始漂移） │
│ → 确认为无限循环漂移，非一次性问题 │
│ │
│ Step 2: Provider 源码逐字段分类 │
│ ────────────────────────────── │
│ 查看 resource_xxx.go 中嵌套块的 Schema 定义 │
│ 将每个字段标注为：Required / Optional only / Optional+Computed / │
│ Computed only │
│ → 识别出 Computed only 字段（根因候选） │
│ │
│ Step 3: 理解 TypeSet hash 机制 │
│ ───────────────────────────── │
│ Terraform SDKv2 TypeSet hash = schema 全字段参与计算 │
│ → Computed only 字段的零值 "" 必然 ≠ API 返回值 │
│ → 每次 plan 都判定元素变更 │
│ │
│ Step 4: 排除错误方案 │
│ ────────────────── │
│ 尝试 ignore_changes = [servers[0].xxx] → Cannot index a set │
│ 尝试移除 content 中 Optional+Computed 字段 → 仍漂移 │
│ → 确认 TypeSet hash 包含不可控的 Computed only 字段 │
│ │
│ Step 5: 确定最优解 │
│ ──────────────── │
│ ignore_changes = [servers] — 唯一可行方案 │
│ + 记录副作用：变更 servers 需临时注释 ignore_changes │
│ │
│ Step 6: 验证 │
│ ──────── │
│ terraform plan → No changes ✓ │
│ │
└──────────────────────────────────────────────────────────────────────┘
```

### 排查过程中的关键经验

1. **不要猜测，用源码实证**：每个字段的 Computed/Optional 属性必须从 Provider 源码确认，不能凭经验假设
2. **TypeSet 的 hash 机制是关键**：SDKv2 的 TypeSet hash 包含 schema 中所有字段，不仅限于 HCL content 中显式赋值的
3. **Computed only 字段是致命的**：无 `Optional` 的 `Computed: true` 字段在 HCL 中不可赋值，零值必然与 API 返回值不同
4. **逐步排除，不要跳步**：先尝试精确 ignore → 发现 TypeSet 不支持索引 → 尝试精简 content → 发现仍不够 → 确认只能 ignore 整个块

## 已知受影响资源

| 云厂商 | 资源 | 嵌套块 | Computed only 字段 |
|--------|------|--------|-------------------|
| 阿里云 | `alicloud_alb_server_group` | `servers` | `status`, `server_group_id` |

> 如发现新的受影响资源，请更新此表。

## 快速诊断检查清单

当 `dynamic` 嵌套块出现无限循环漂移时：

- [ ] 确认是否为第二次+ plan 仍检测到变更（排除首次 apply 的正常 state 回填）
- [ ] 查看 Provider 源码，确认嵌套块的 `Type` 是否为 `TypeSet`
- [ ] 逐字段标注 Computed/Optional 属性，识别 Computed only 字段
- [ ] 确认 TypeSet hash 包含 Computed only 字段（SDKv2 行为）
- [ ] 尝试 `ignore_changes = [block_name]`（整个块）
- [ ] 运行 `terraform plan` 验证漂移消除
- [ ] 记录副作用（变更时需临时注释 ignore_changes）

## 决策规则

> **当 Provider 将嵌套块定义为 TypeSet 且内部包含 Computed-only（无 Optional）字段时，HCL 层面无法规避漂移，必须使用 `ignore_changes = [block_name]` 忽略整个块。变更该块内容时需临时注释 ignore_changes。**

## 相关规则

- `provider-optional-api-mandatory.md` - Provider 参数 Schema 类型分类（顶层字段）
- `resource-lifecycle.md` - 生命周期块使用规范
- `test-drift-detection.md` - 部署验证漂移检测流程
- `resource-zoneid-format-drift.md` - API 返回值格式不一致漂移（不同根因）

## 参考资料

- `resource-lifecycle.md`（生命周期块）
- `resource-zoneid-format-drift.md`（ZoneId 格式漂移）
- [Terraform TypeSet 行为](https://developer.hashicorp.com/terraform/cli/state/resource-addressing)

---

# 使用明确的类型约束

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

明确的类型约束能及早捕获错误、提供文档，并改善 IDE 支持。使用 `any` 或省略类型会失去这些优势，使模块更难正确使用。

## 错误示例

```hcl
# 无类型 - 接受任何值
variable "instance_count" {}

# 已知具体类型时使用 'any'
variable "tags" {
  type = any
}

# 过于宽松
variable "port" {
  type = any
}
```

**问题：** 类型错误只能在 apply 时发现，甚至可能无法发现。

## 正确示例

```hcl
# 明确的基本类型
variable "instance_count" {
  type        = number
  description = "要创建的实例数量"
}

variable "environment" {
  type        = string
  description = "部署环境"
}

variable "enable_monitoring" {
  type        = bool
  default     = true
  description = "启用 CloudWatch 监控"
}

# 明确的集合类型
variable "tags" {
  type        = map(string)
  default     = {}
  description = "资源标签"
}

variable "subnet_ids" {
  type        = list(string)
  description = "子网 ID 列表"
}

variable "availability_zones" {
  type        = set(string)
  description = "可用区集合"
}
```

## 使用 Object 的复杂类型

```hcl
# 包含必填字段的对象
variable "database_config" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
  })
  description = "数据库配置"
}

# 包含可选字段和默认值的对象
variable "scaling_config" {
  type = object({
    min_size          = optional(number, 1)
    max_size          = optional(number, 10)
    desired_capacity  = optional(number, 2)
    cooldown          = optional(number, 300)
  })
  default     = {}
  description = "自动伸缩配置"
}
```

## 使用正向变量名

避免双重否定，使用正向名称：

```hcl
# 错误 - 设为 false 时产生双重否定
variable "disable_encryption" {
  type    = bool
  default = false # !disable = enable... 令人困惑
}

# 正确 - 清晰的意图
variable "encryption_enabled" {
  type        = bool
  default     = true
  description = "启用静态加密"
}

# 错误
variable "no_public_ip" {
  type = bool
}

# 正确
variable "associate_public_ip" {
  type        = bool
  default     = false
  description = "关联公网 IP 地址"
}
```

## 适当使用 `nullable = false`

```hcl
# 防止集合类型接收 null 值
variable "subnet_cidrs" {
  type     = list(string)
  default  = []
  nullable = false
  description = "子网 CIDR 块列表"
}

variable "tags" {
  type     = map(string)
  default  = {}
  nullable = false
  description = "资源标签"
}

# 确保字符串有值
variable "environment" {
  type     = string
  nullable = false
  description = "部署环境（必填）"
}
```

## 何时使用 `any`

将 `type = any` 保留给特殊情况：

```hcl
# 可接受：来自外部源的高度可变结构
variable "datadog_monitor_config" {
  type        = any
  description = "匹配 Datadog API Schema 的监控配置"
}

# 可接受：透传到动态资源
variable "container_definitions" {
  type        = any
  description = "ECS 容器定义 JSON"
}
```

## 多云场景适配

### 阿里云实例规格类型

```hcl
# 阿里云 ECS 实例配置
variable "ecs_instances" {
  type = map(object({
    instance_type = string    # 如 "ecs.g7.xlarge"
    image_id      = string    # 如 "ubuntu_22_04_x64"
    vswitch_key   = string    # 逻辑名映射
    security_group_key = optional(string)
    data_disks = optional(list(object({
      name     = string
      size     = number
      category = optional(string, "cloud_essd")
    })), [])
  }))
}
```

### Azure VM 规格类型

```hcl
# Azure 虚拟机配置
variable "virtual_machines" {
  type = map(object({
    vm_size    = string       # 如 "Standard_D2s_v3"
    os_disk    = object({
      caching              = string
      storage_account_type = string  # 如 "Premium_LRS"
    })
    subnet_key = string       # 逻辑名映射
    nsg_key    = optional(string)
  }))
}
```

### 腾讯云 CVM 实例类型

```hcl
# 腾讯云 CVM 实例配置
variable "cvm_instances" {
  type = map(object({
    instance_type = string    # 如 "SA2.MEDIUM4"
    image_id      = string    # 如 "img-xxx"
    subnet_key    = string
    security_group_key = optional(string)
  }))
}
```

## 参考资料

- [类型约束](https://developer.hashicorp.com/terraform/language/expressions/type-constraints)
- [变量类型](https://developer.hashicorp.com/terraform/language/values/variables#type-constraints)

---

# 添加变量验证规则

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

验证规则在 Terraform 尝试创建资源之前及早捕获配置错误。这可以防止失败的部署并提供清晰的错误信息。

## 错误示例

```hcl
variable "environment" {
  type = string
  # 无验证 - 接受任何字符串
}

variable "instance_type" {
  type = string
}

variable "cidr_block" {
  type = string
}

# 错误只在 apply 时被 AWS 拒绝后才发现
```

## 正确示例

```hcl
variable "environment" {
  type        = string
  description = "部署环境"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "环境必须是 dev、staging 或 prod。"
  }
}

variable "instance_type" {
  type        = string
  description = "EC2 实例类型"

  validation {
    condition     = can(regex("^[a-z][0-9]+\\.(nano|micro|small|medium|large|xlarge|[0-9]+xlarge)$", var.instance_type))
    error_message = "无效的实例类型格式。示例：t3.micro、m5.large"
  }
}

variable "cidr_block" {
  type        = string
  description = "VPC CIDR 块"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "必须是有效的 CIDR 块（如 10.0.0.0/16）。"
  }

  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) >= 16 && tonumber(split("/", var.cidr_block)[1]) <= 28
    error_message = "CIDR 块必须在 /16 到 /28 之间。"
  }
}
```

## 常用验证模式

### 字符串长度

```hcl
variable "bucket_name" {
  type = string

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "存储桶名称必须在 3 到 63 个字符之间。"
  }
}
```

### 数值范围

```hcl
variable "port" {
  type = number

  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "端口必须在 1 到 65535 之间。"
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "实例数量必须在 1 到 10 之间。"
  }
}
```

### 列表验证

```hcl
variable "availability_zones" {
  type = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "高可用至少需要 2 个可用区。"
  }
}
```

### Map 验证

```hcl
variable "tags" {
  type = map(string)

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "标签必须包含 'Environment' 键。"
  }
}
```

### 多重验证

```hcl
variable "db_name" {
  type = string

  validation {
    condition     = length(var.db_name) >= 1 && length(var.db_name) <= 63
    error_message = "数据库名称必须在 1 到 63 个字符之间。"
  }

  validation {
    condition     = can(regex("^[a-zA-Z][a-zA-Z0-9_]*$", var.db_name))
    error_message = "数据库名称必须以字母开头，只包含字母、数字和下划线。"
  }
}
```

## 多云场景适配

### 阿里云实例规格验证

```hcl
variable "instance_type" {
  type        = string
  description = "阿里云 ECS 实例规格"

  validation {
    condition     = can(regex("^ecs\\.[a-z]\\d+\\.", var.instance_type))
    error_message = "实例规格格式错误，示例: ecs.g7.xlarge"
  }
}

variable "engine" {
  type        = string
  description = "数据库引擎"

  validation {
    condition     = contains(["mysql", "postgresql", "mariadb"], var.engine)
    error_message = "数据库引擎必须是 mysql、postgresql 或 mariadb。"
  }
}
```

### Azure SKU 验证

```hcl
variable "vm_size" {
  type        = string
  description = "Azure 虚拟机规格"

  validation {
    condition     = can(regex("^Standard_", var.vm_size))
    error_message = "VM 规格必须以 Standard_ 开头，示例: Standard_D2s_v3"
  }
}

variable "storage_account_type" {
  type        = string
  description = "存储类型"

  validation {
    condition     = contains(["Premium_LRS", "Standard_LRS", "StandardSSD_LRS"], var.storage_account_type)
    error_message = "存储类型必须是 Premium_LRS、Standard_LRS 或 StandardSSD_LRS。"
  }
}
```

### 腾讯云实例验证

```hcl
variable "instance_type" {
  type        = string
  description = "腾讯云 CVM 实例规格"

  validation {
    condition     = can(regex("^[A-Z]\\w+\\.", var.instance_type))
    error_message = "实例规格格式错误，示例: SA2.MEDIUM4"
  }
}
```

## 参考资料

- [输入变量验证](https://developer.hashicorp.com/terraform/language/values/variables#custom-validation-rules)

---

# 标记敏感变量

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

密码和 API Key 等敏感值可能通过 CLI 输出、日志和 CI/CD 流水线泄露。将它们标记为敏感，且不要为必填密钥提供默认值。

## 错误示例

```hcl
variable "database_password" {
  type    = string
  default = "" # 空默认值允许跳过必填密钥
}

variable "api_key" {
  type = string
  # 未标记 sensitive - 会显示在 plan 输出中
}

# 在 terraform plan 输出中：
# + password = "actual-secret-value" # 泄露！
```

**问题：** 密钥在 Terraform 输出、CI 日志和状态文件中可见。

## 正确示例

```hcl
variable "database_password" {
  type        = string
  sensitive   = true
  description = "数据库主密码。通过 TF_VAR_database_password 设置。"
  # 无默认值 - 强制用户提供值
}

variable "api_key" {
  type        = string
  sensitive   = true
  description = "外部服务的 API 密钥。"
}

# 可选密钥配合自动生成
variable "random_password" {
  type      = string
  default   = null
  sensitive = true
  description = "密码。如果未提供，将自动生成。"
}

resource "random_password" "generated" {
  count   = var.random_password == null ? 1 : 0
  length  = 32
  special = true
}

locals {
  actual_password = coalesce(var.random_password, try(random_password.generated[0].result, null))
}
```

## Terraform 输出中的敏感值

```hcl
# 在 terraform plan 输出中：
# + password = (sensitive value) # 已保护！
```

## 传递敏感值

### 环境变量

```bash
export TF_VAR_database_password="secure-password-here"
terraform apply
```

### Terraform Cloud/Enterprise 变量

在 UI 或 API 中将变量标记为 "Sensitive"。

### 从密钥管理器获取

```hcl
# 从 AWS Secrets Manager 读取
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/database/password"
}

# 获取的值自动标记为敏感
resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

## 敏感输出

```hcl
# 将包含密钥的输出标记为敏感
output "database_connection_string" {
  value       = "postgres://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive   = true
  description = "数据库连接字符串（包含凭据）"
}
```

## 关于状态文件的警告

即使设置了 `sensitive = true`，密钥值仍以明文存储在 Terraform 状态文件中。通过以下方式保护状态文件：

1. 使用加密的远程后端
2. 限制状态存储的访问权限
3. 使用 Terraform Cloud 的状态加密
4. 永远不要将状态文件提交到版本控制

```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true # 启用服务端加密
  }
}
```

## 参考资料

- [敏感变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
- [敏感输出](https://developer.hashicorp.com/terraform/language/values/outputs#sensitive-suppressing-values-in-cli-output)

---

# 为变量添加描述

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

没有描述的变量使模块难以使用。描述作为内联文档，出现在生成的文档、`terraform plan` 输出和 IDE 提示中。

## 错误示例

```hcl
variable "instance_type" {
  type    = string
  default = "t3.micro"
}

variable "vpc_cidr" {
  type = string
}

variable "enable_monitoring" {
  type    = bool
  default = true
}
```

**问题：** 用户必须阅读代码或猜测每个变量的用途。

## 正确示例

```hcl
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "Web 服务器的 EC2 实例类型。参见 https://aws.amazon.com/ec2/instance-types/"
}

variable "vpc_cidr" {
  type        = string
  description = "VPC 的 CIDR 块。必须是 /16 到 /28 范围。"
}

variable "enable_monitoring" {
  type        = bool
  default     = true
  description = "启用 EC2 实例的详细 CloudWatch 监控。"
}
```

## 最佳实践

### 使用上游 Provider 描述

封装 Provider 资源时，使用与上游 Provider 文档相同的描述：

```hcl
# 来自 AWS Provider 文档的 aws_instance 描述
variable "associate_public_ip_address" {
  type        = bool
  default     = false
  description = "是否为 VPC 中的实例关联公网 IP 地址。"
}
```

### 在描述中包含约束

```hcl
variable "retention_days" {
  type        = number
  default     = 30
  description = "日志保留天数。必须在 1 到 365 之间。"

  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 365
    error_message = "保留天数必须在 1 到 365 之间。"
  }
}
```

### 记录默认行为

```hcl
variable "tags" {
  type        = map(string)
  default     = {}
  description = "应用到所有资源的额外标签。与默认标签合并。"
}

variable "subnet_ids" {
  type        = list(string)
  default     = null
  description = "子网 ID 列表。如果未提供，将自动创建子网。"
}
```

## 参考资料

- [输入变量](https://developer.hashicorp.com/terraform/language/values/variables)
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)

---

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

---

# optional() 集合类型必须指定默认值

**优先级：** 高
**分类：** 变量与输出模式（层级专用）

## 为什么重要

在控制层变量中使用 `optional()` 时，省略参数会产生 `null`。这导致原子层的 `length()` 函数报错：`Invalid value for "value" parameter: argument must not be null.`

这是三层 IaC 架构中控制层向原子层传递参数时的关键问题。

## 错误示例

```hcl
# 控制层 - 未传入时产生 null
variable "kafka_instances" {
  type = map(object({
    vswitch_ids = optional(list(string)) # 不传 → null
    selected_zones = optional(list(string)) # 不传 → null
    tags = optional(map(string)) # 不传 → null
  }))
}

# 原子层 - 接收 null 时崩溃
resource "alicloud_alikafka_instance" "this" {
  vswitch_ids = length(var.vswitch_ids) > 0 ? var.vswitch_ids : null # length(null) 报错！
  selected_zones = length(var.selected_zones) > 0 ? var.selected_zones : null
}
```

**错误输出：**
```
│ Error: Invalid function argument
│ on main.tf line 60: length(var.vswitch_ids) > 0
│ var.vswitch_ids is null
│ Invalid value for "value" parameter: argument must not be null.
```

## 正确示例（Redis 模式）

```hcl
# Control layer - specify default values for collections
variable "kafka_instances" {
  type = map(object({
    vswitch_ids = optional(list(string), []) # 不传 → 空列表 []
    selected_zones = optional(list(string), []) # 不传 → 空列表 []
    tags = optional(map(string), {}) # 不传 → 空 map {}
  }))
}

# Atomic layer - now safe to use length()
resource "alicloud_alikafka_instance" "this" {
  vswitch_ids = length(var.vswitch_ids) > 0 ? var.vswitch_ids : null # length([]) = 0
  selected_zones = length(var.selected_zones) > 0 ? var.selected_zones : null #
}
```

## 默认值规则

| Type | Correct optional() Syntax | Value when not passed |
|------|---------------------------|----------------------|
| `list(string)` | `optional(list(string), [])` | 空列表 `[]` |
| `list(number)` | `optional(list(number), [])` | 空列表 `[]` |
| `map(string)` | `optional(map(string), {})` | 空 map `{}` |
| `string` | `optional(string)` | `null` (原子层用 `var.xxx != ""` 检查) |
| `string` (lookup key) | `optional(string, "")` | 空字符串 `""` |
| `number` | `optional(number)` | `null` (原子层用 `var.xxx == null` 检查) |
| `bool` (普通参数) | `optional(bool)` | `null` (原子层用 `var.xxx == null` 检查) |
| `bool` (控制开关) | `optional(bool, true)` | `true` (控制开关默认创建) |

## Computed 参数在 object 内部的特殊处理

### 问题场景

当 object 类型内部包含 `Optional + Computed` 参数时，**不应设置默认值**：

```hcl
# 错误：object 内部设置默认值
variable "product_info" {
  type = object({
    msg_process_spec = string
    send_receive_ratio = optional(number, 1.0) # 默认值 1.0 可能无效！
  })
}

# 当声明层传入 { msg_process_spec = "rmq.s2.2xlarge" } 时
# Terraform 自动填充 send_receive_ratio = 1.0
# API 收到 1.0 后报错：InvalidSendReceiveRatio.Range
```

### 正确处理

```hcl
# 正确：Computed 参数不设默认值
variable "product_info" {
  type = object({
    msg_process_spec = string
    send_receive_ratio = optional(number) # 不设默认值，返回 null
  })
}

# 原子层 main.tf
product_info {
  send_receive_ratio = try(product_info.value.send_receive_ratio, null) # null 时 API 自动填充
}
```

### 判断依据

查阅 Provider 源码 Schema 定义：
```go
"send_receive_ratio": {
  Type: schema.TypeFloat,
  Optional: true,
  Computed: true, // Computed = API 会返回默认值
},
```

**有 `Computed: true` 的参数，在 `optional()` 中不应设置默认值。**

## String 类型用于 lookup key 的特殊规则

### 问题场景

当 string 类型字段作为 `lookup()` 的 key 参数时，`null` 会导致报错：

```hcl
# 控制层变量定义 - optional(string) 默认返回 null
variable "kafka_instances" {
  type = map(object({
    security_group_key = optional(string) # 未设置时 → null
  }))
}

# 控制层 main.tf - lookup 接收 null 报错
security_group = try(each.value.security_group_key, "") != ""
? lookup(var.security_group_ids_map, each.value.security_group_key, "") # key=null 报错！
: ""
```

**错误输出：**
```
│ Error: Invalid function argument
│ on main.tf line 376: lookup(var.security_group_ids_map, each.value.security_group_key, "")
│ each.value.security_group_key is null
│ Invalid value for "key" parameter: argument must not be null.
```

### 根因分析

| 变量类型定义 | 未设置字段时 | `try()` 结果 | `lookup(map, key, "")` |
|-------------|------------|-------------|------------------------|
| `map(any)` | 字段**不存在** | `try` 捕获错误 → `""` | 不会执行到 lookup |
| `map(object)` + `optional(string)` | 字段存在，值为 `null` | `try(null, "")` → `""` | `lookup(map, null, "")` 报错 |
| `map(object)` + `optional(string, "")` | 字段存在，值为 `""` | `try("", "")` → `""` | `lookup(map, "", "")` 正常 |

**关键差异**：
- `map(any)`：未定义字段会报 "attribute not found"，`try()` 能捕获
- `map(object)`：字段已定义，值为 `null`，`try()` 无法阻止 `null` 传给 `lookup()`

### 正确写法

```hcl
# 控制层变量定义 - lookup key 字段必须设置空字符串默认值
variable "kafka_instances" {
  type = map(object({
    security_group_key = optional(string, "") # 未设置时 → "" (非 null)
    vswitch_key = optional(string, "") # 用于 vswitch_ids_map lookup
    zone_key = optional(string, "") # 用于 az_map lookup
  }))
}

# 控制层 main.tf - 现在可以安全使用
security_group = try(each.value.security_group_key, "") != ""
? lookup(var.security_group_ids_map, each.value.security_group_key, "") # key="" 或有效值，都不会报错
: try(each.value.security_group, "")

vswitch_id = try(each.value.vswitch_key, "") != ""
? var.vswitch_ids_map[each.value.vswitch_key] # key="" 不会执行到这里
: var.vswitch_ids_map["j-data"] # 默认值
```

### 三层架构中的最佳实践

| 场景 | 变量定义 | 说明 |
|------|----------|------|
| 用于 `lookup()` / `map[key]` 的 key | `optional(string, "")` | 必须设置空字符串默认值 |
| 普通字符串参数 | `optional(string)` | 可以保持 `null` 默认值 |
| 用于 `length()` 的 list | `optional(list(string), [])` | 必须设置空列表默认值 |
| 用于 `length()` 的 map | `optional(map(string), {})` | 必须设置空 map 默认值 |

## 控制开关参数特殊规则

### 为什么 `create` 参数必须设置默认值？

```hcl
# 错误 - optional(bool) 未设置默认值
create = optional(bool) # → null

# 控制层传递
create = try(each.value.create, true) # try(null, true) 返回 null！不是 true！

# 原子层报错
count = local.create ? 1 : 0 # null ? 1 : 0 → Error: Null condition
```

**关键知识点**：`try(null, default)` 返回 `null`，因为 `null` 不是错误，是合法值！

### 正确写法

```hcl
# 控制层变量定义 - 设置默认值
variable "kafka_instances" {
  type = map(object({
    create = optional(bool, true) # 默认 true = 创建
  }))
}

# 嵌套对象中的 create 也需要默认值
sasl_users = optional(map(object({
  create = optional(bool, true) # 默认创建用户
  password = optional(string)
})), {})

# 控制层传递（保持 try 作为双重保险）
create = try(each.value.create, true) # try(true, true) = true
```

### 三层 create 参数规范

| 层级 | 规范写法 | 说明 |
|------|----------|------|
| **控制层 variables.tf** | `create = optional(bool, true)` | 默认值在 optional 中设置 |
| **控制层 main.tf** | `create = try(each.value.create, true)` | try 作为双重保险 |
| **原子层 variables.tf** | `default = true` | 原子层保持默认 true |
| **原子层 main.tf** | `local.create = var.create` | 直接使用，无需 coalesce |

## 原子层协调

原子层验证规则应使用 null 安全模式：

```hcl
# Number type - check null first
variable "disk_type" {
  type = number
  default = 1

  validation {
    condition = var.disk_type == null || contains([0, 1], var.disk_type)
    error_message = "disk_type 取值：null（不设置）/ 0（高效云盘）/ 1（SSD 云盘）。"
  }
}

# String type - check null and empty
variable "enable_auto_topic" {
  type = string
  default = "disable"

  validation {
    condition = var.enable_auto_topic == null || var.enable_auto_topic == "" || contains(["enable", "disable"], var.enable_auto_topic)
    error_message = "enable_auto_topic 取值：null/空字符串（不设置）/ enable（开启）/ disable（关闭）。"
  }
}
```

## 总结规则

**一句话总结**：集合类型（list/map）和 lookup key 字段（string）在 `optional()` 中必须指定默认值，避免原子层函数接收到 `null`。

## 实际案例

### Kafka 模块修复

```hcl
# Before (error):
vswitch_ids = optional(list(string)) # → null
security_group_key = optional(string) # → null

# After (fixed):
vswitch_ids = optional(list(string), []) # → []
selected_zones = optional(list(string), []) # → []
tags = optional(map(string), {}) # → {}
security_group_key = optional(string, "") # → ""
```

### Redis 模块模式（参考）

```hcl
# Redis uses this pattern consistently
variable "backup_period" {
  type = list(string)
  default = ["Monday", "Wednesday", "Friday"] # Always has default
}

# In main.tf
backup_period = length(var.backup_period) > 0 ? var.backup_period : null # Safe
```

## 参考资料

- [Terraform 可选属性](https://developer.hashicorp.com/terraform/language/expressions/type-constraints#optional-attributes)

---

# 分层类型设计模式

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

Terraform 的 `map(any)` 要求所有 map 元素具有相同的字段结构。这与实际场景冲突，比如一种资源类型有多个子类型，各子类型有不同的参数集（如 Kafka Serverless/传统版/Confluent 版）。

本规则定义了**分层类型设计模式**，在声明层、控制层和原子层之间平衡灵活性与类型安全。

## 问题

```hcl
# Problem: map(any) requires identical structure
variable "kafka_instances" {
  type = map(any) # Error when elements have different fields!
}

# terraform.tfvars - three different sub-types
kafka_instances = {
  "01" = { serverless_config = { ... } } # Serverless type
  "02" = { disk_type = 1, disk_size = 500 } # Traditional type
  "03" = { confluent_config = { ... } } # Confluent type
}
# Error: all map elements must have the same type
```

## 解决方案：分层类型设计模式

### 架构概览

```
┌─────────────────┐ ┌─────────────────────────────┐ ┌─────────────────┐
│ Declaration │ │ Control Layer │ │ Atomic Layer │
│ Layer │ │ │ │ │
├─────────────────┤ ├─────────────────────────────┤ ├─────────────────┤
│ type = any │ ──► │ type = map(object({...})) │ ──► │ type = specific │
│ (most flexible) │ │ + optional() for each field │ │ (strict) │
└─────────────────┘ └─────────────────────────────┘ └─────────────────┘
│ │ │
Flexibility Type Safety Validation
(any sub-type) (optional constraints) (final check)
```

### 第 1 层：声明层（最灵活）

**使用 `any` 类型**，允许不同子类型在同一 map 中共存。

```hcl
# declarative/${ENV}/04-database/variables.tf（声明层）

variable "kafka_instances" {
  description = "Kafka 实例配置 map。key=逻辑名"
  type = any # 允许不同子类型共存
  default = {}
}
```

**优点：**
- 不同子类型可以省略不相关字段
- 干净、可读的 tfvars，无空字段
- 不会出现"所有元素必须具有相同类型"错误

### 第 2 层：控制层（类型约束 + 灵活性）

**使用 `map(object({...}))` 配合 `optional()`**，提供类型安全同时允许省略字段。

```hcl
# control/database-cluster/variables.tf（控制层）

variable "kafka_instances" {
  description = "Kafka 实例配置 map。key=逻辑名（如 01/02/03）"
  type = map(object({
    # Common fields
    create = optional(bool)
    vswitch_key = optional(string)
    zone_id = optional(string)
    instance_name = optional(string)
    instance_type = optional(string)
    deploy_type = optional(number)
    paid_type = optional(string)
    service_version = optional(string)

    # Traditional type fields
    spec_type = optional(string)
    partition_num = optional(number)
    io_max = optional(number)
    disk_type = optional(number)
    disk_size = optional(number)

    # Serverless type fields
    serverless_config = optional(object({
      reserved_publish_capacity = number
      reserved_subscribe_capacity = number
    }))

    # Confluent type fields
    confluent_config = optional(object({
      kafka_cu = optional(number)
      kafka_storage = optional(number)
      kafka_replica = optional(number)
      zookeeper_cu = optional(number)
      zookeeper_storage = optional(number)
      zookeeper_replica = optional(number)
    }))
    password = optional(string) # Confluent only

    # Collection types - MUST have defaults
    vswitch_ids = optional(list(string), [])
    selected_zones = optional(list(string), [])
    tags = optional(map(string), {})
  }))
  default = {}
}
```

**优点：**
- 类型安全：IDE 支持，早期错误检测
- 灵活性：`optional()` 允许省略字段
- 清晰契约：明确的字段定义即文档

### 第 3 层：原子层（最终验证）

**使用具体类型配合验证规则**，进行最终参数验证。

```hcl
# atomic/kafka/variables.tf（原子层）

variable "instance_type" {
  description = "Kafka 实例类型"
  type = string
  default = ""

  validation {
    condition = var.instance_type == "" || var.instance_type == null ||
    contains(["alikafka", "alikafka_serverless", "alikafka_confluent"], var.instance_type)
    error_message = "instance_type 取值：空字符串/alikafka/alikafka_serverless/alikafka_confluent。"
  }
}

variable "disk_type" {
  description = "Kafka 磁盘类型（传统版专用）"
  type = number
  default = 1

  validation {
    condition = var.disk_type == null || contains([0, 1], var.disk_type)
    error_message = "disk_type 取值：null/0/1。"
  }
}
```

## 完整示例：Kafka 多类型配置

### 声明层（terraform.tfvars）

```hcl
# Clean and readable - only relevant fields per sub-type
kafka_instances = {
  # Serverless type - elastic, pay-as-you-go
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    deploy_type = 5
    paid_type = "PostPaid"

    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
    instance_name = "kafka-serverless-01"
    # No disk_type, disk_size, confluent_config - irrelevant for Serverless
  }

  # Traditional type - fixed spec, high throughput
  "02" = {
    vswitch_key = "j-data"
    create = false
    instance_type = "alikafka"
    deploy_type = 5
    spec_type = "professional"
    partition_num = 50
    io_max = 20
    disk_type = 1
    disk_size = 500
    paid_type = "PostPaid"
    service_version = "2.6.2"
    instance_name = "kafka-traditional-02"
    # No serverless_config, confluent_config - irrelevant for Traditional
  }

  # Confluent type - enterprise features
  "03" = {
    vswitch_key = "j-data"
    create = false
    instance_type = "alikafka_confluent"
    deploy_type = 5
    spec_type = "professional"

    confluent_config = {
      kafka_cu = 2
      kafka_storage = 500
      kafka_replica = 3
      zookeeper_cu = 2
      zookeeper_storage = 20
      zookeeper_replica = 3
    }
    password = "KafkaTest123!"
    paid_type = "PostPaid"
    instance_name = "kafka-confluent-03"
    # No serverless_config, disk_type - irrelevant for Confluent
  }
}
```

## 类型设计决策矩阵

| Scenario | Declaration Layer | Control Layer | Atomic Layer |
|----------|------------------|---------------|--------------|
| **Multi sub-types** (Kafka, etc.) | `any` | `map(object)` + `optional()` | specific + validation |
| **Single type, all required** | `map(object)` | `map(object)` | specific |
| **Simple passthrough** | `map(any)` | `map(any)` | specific |

## 核心规则总结

### 规则 1：声明层灵活性
```hcl
# Use `any` when map elements have different structures
type = any

# Don't use map(any) - it requires identical structure
type = map(any) # Error when sub-types differ!
```

### 规则 2：控制层类型安全
```hcl
# Use map(object) with optional() for each field
type = map(object({
  field1 = optional(string)
  field2 = optional(number)
  list_field = optional(list(string), []) # Collection MUST have default!
}))

# Don't use map(any) - loses type safety and IDE support
type = map(any)
```

### 规则 3：集合类型必须有默认值
```hcl
# Collection types in optional() MUST specify defaults
vswitch_ids = optional(list(string), []) #
selected_zones = optional(list(string), []) #
tags = optional(map(string), {}) #

# Without defaults, null causes length() to fail
vswitch_ids = optional(list(string)) # null when not passed
# In atomic layer: length(var.vswitch_ids) > 0 → ERROR!
```

### 规则 4：原子层 Null 安全验证
```hcl
# Always check null first in validation
validation {
  condition = var.disk_type == null || contains([0, 1], var.disk_type)
  error_message = "..."
}

# contains() fails with null
validation {
  condition = contains([0, 1], var.disk_type) # ERROR when null!
  error_message = "..."
}
```

## 收益总结

| Aspect | Before (map(any) everywhere) | After (Layered Type Design) |
|--------|------------------------------|----------------------------|
| **Flexibility** | Requires identical structure | Different sub-types allowed |
| **Type Safety** | No IDE support, late errors | Early detection, IDE hints |
| **Readability** | Verbose with empty fields | Clean, only relevant fields |
| **Documentation** | No explicit contract | Control layer = contract |
| **Validation** | Scattered, inconsistent | Layered, systematic |

### 规则 5：控制层类型定义必须覆盖所有 `each.value.*` 引用（关键）

> **这是最常见的隐性错误：控制层 `main.tf` 中通过 `each.value.xxx` 引用的字段，必须在 `variables.tf` 的 `map(object({...}))` 类型定义中声明。否则 Terraform 会静默丢弃未声明的字段，导致配置不生效且无任何错误提示。**

```hcl
# 致命错误：main.tf 引用了 security_group_key，但类型定义中没有

# control/web-cluster/variables.tf
variable "nlbs" {
  type = map(object({
    address_type = string
    cross_zone_enabled = optional(bool, true)
    create = optional(bool, true)
    # 缺少 security_group_key！
  }))
}

# control/web-cluster/main.tf
module "nlb" {
  for_each = var.nlbs
  # each.value.security_group_key 永远为 null（被静默丢弃）
  security_group_ids = try(each.value.security_group_key, null) != null ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] : []
}
```

```hcl
# 正确：类型定义覆盖所有 each.value 引用

# control/web-cluster/variables.tf
variable "nlbs" {
  type = map(object({
    address_type = string
    cross_zone_enabled = optional(bool, true)
    create = optional(bool, true)
    security_group_key = optional(string) # 与 main.tf 引用对齐
  }))
}
```

**检测方法**：在控制层 `main.tf` 中搜索所有 `each.value.` 引用，提取字段名列表，然后逐一对照 `variables.tf` 中对应 `map(object({...}))` 的类型定义，确保每个引用字段都已声明。

```bash
# 检测命令：提取 main.tf 中所有 each.value.XXX 字段
grep -oP 'each\.value\.\w+' control/web-cluster/main.tf | sort -u
# 输出示例：
# each.value.address_type
# each.value.create
# each.value.cross_zone_enabled
# each.value.security_group_key

# 然后对照 variables.tf 中 map(object) 的字段列表，确认每个都有声明
```

## 迁移检查清单

将模块转换为此模式时：

- [ ] **声明层**：将 `map(any)` 改为 `any`
- [ ] **控制层**：定义 `map(object({...}))`，包含所有可能的字段
- [ ] **控制层**：为每个字段添加 `optional()`
- [ ] **控制层**：为集合类型添加默认值 `[]`、`{}`
- [ ] **控制层**：**`each.value.*` 引用与类型定义字段对齐检查（关键！）**
- [ ] **原子层**：添加 null 安全验证规则
- [ ] **terraform.tfvars**：移除空字段/不需要的字段，只保留相关字段

## 参考资料

- [Terraform 类型约束](https://developer.hashicorp.com/terraform/language/expressions/type-constraints)
- [Terraform 可选属性](https://developer.hashicorp.com/terraform/language/expressions/type-constraints#optional-attributes)

---

# 控制层自动命名

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

在三层 IaC 架构中，声明层手动命名会导致冗余和不一致。控制层自动命名确保：
- **一致性**：所有资源遵循相同的命名模式
- **简洁性**：声明层只需 `key`，不需要 `instance_name`
- **可维护性**：修改命名模式只需更新控制层

## 模式：声明层 → 控制层自动命名

### 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│ Declaration Layer (terraform.tfvars) │
│ │
│ kafka_instances = { │
  │ "01" = { │
    │ vswitch_key = "j-data" │
    │ # instance_name NOT required! │
    │ } │
    │ } │
    └──────────────────────────────┬──────────────────────────────────────────┘
    │ key = "01"
    ▼
    ┌─────────────────────────────────────────────────────────────────────────┐
    │ Control Layer (main.tf) │
    │ │
    │ instance_name = coalesce( │
    │ try(each.value.instance_name, ""), │
    │ "kafka-${var.env}-${each.key}" # ← Auto-generated! │
    │ ) │
    │ │
    │ # Result: kafka-simple-01 │
    └─────────────────────────────────────────────────────────────────────────┘
```

### 控制层实现

```hcl
# control/database-cluster/main.tf（控制层模块）

module "kafka" {
  source = "../../atomic/kafka"
  for_each = var.kafka_instances

  # Auto-naming pattern: {resource_type}-{env}-{key}
  instance_name = coalesce(
  try(each.value.instance_name, ""), # Use custom name if provided
  "kafka-${var.env}-${each.key}" # Otherwise auto-generate
  )

  # Other parameters...
}

module "redis_community" {
  source = "../../atomic/redis-community"
  for_each = var.redis_community_instances

  # Auto-naming pattern: {resource_type}-{env}-{key}
  instance_name = coalesce(
  try(each.value.instance_name, ""),
  "redis-${var.env}-${each.key}"
  )
}

module "polardb" {
  source = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # Auto-naming pattern for cluster_description
  cluster_description = coalesce(
  try(each.value.cluster_name, ""),
  "polar-${var.env}-${each.key}"
  )
}
```

## 命名规范表

| Resource Type | Pattern | Example (env=simple, key=01) |
|---------------|---------|------------------------------|
| Kafka | `kafka-{env}-{key}` | `kafka-simple-01` |
| Redis Community | `redis-{env}-{key}` | `redis-simple-01` |
| Tair | `tair-{env}-{key}` | `tair-simple-01` |
| PolarDB | `polar-{env}-{key}` | `polar-simple-01` |
| Elasticsearch | `es-{env}-{key}` | `es-simple-01` |
| RocketMQ | `rmq-{env}-{key}` | `rmq-simple-01` |
| MySQL ECS | `ecs-{env}-mysql-{key}` | `ecs-simple-mysql-01` |

## 声明层示例

### 最小配置（推荐）

```hcl
# terraform.tfvars - Clean and simple
kafka_instances = {
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    paid_type = "PostPaid"
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
    # instance_name omitted - auto-generated!
  }
}
# Result: kafka-simple-01
```

### 自定义名称（按需）

```hcl
# terraform.tfvars - Override auto-naming when necessary
kafka_instances = {
  "01" = {
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    instance_name = "kafka-orders-service" # Custom name for special case
    paid_type = "PostPaid"
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }
  }
}
# Result: kafka-orders-service (custom name used)
```

## 实现最佳实践

### 1. 使用 `coalesce()` 提供回退

```hcl
# Correct - coalesce provides clean fallback
instance_name = coalesce(
try(each.value.instance_name, ""),
"kafka-${var.env}-${each.key}"
)

# Avoid - verbose and error-prone
instance_name = try(each.value.instance_name, "") != "" ? each.value.instance_name : "kafka-${var.env}-${each.key}"
```

### 2. 使用 `try()` 安全访问

```hcl
# Correct - handles missing key gracefully
try(each.value.instance_name, "")

# Avoid - fails if key doesn't exist
each.value.instance_name
```

### 3. 跨资源保持一致的命名模式

```hcl
# Consistent naming pattern for all resources
# Kafka: kafka-{env}-{key}
# Redis: redis-{env}-{key}
# PolarDB: polar-{env}-{key}
# ES: es-{env}-{key}

# Avoid inconsistent patterns
# Kafka: kafka-{env}-{key}
# Redis: {env}-redis-{key} # Different pattern
# PolarDB: polardb_cluster_{key} # Completely different
```

## 核心规则总结

| 规则 | 说明 |
|------|------|
| **规则 1** | 声明层默认省略 `instance_name` |
| **规则 2** | 控制层自动生成：`{resource_type}-{env}-{key}` |
| **规则 3** | 使用 `coalesce(try(...), auto_pattern)` 提供回退 |
| **规则 4** | 仅在业务需要特定命名时使用自定义名称 |
| **规则 5** | 所有资源保持命名模式一致 |

## 收益总结

| 之前（手动命名） | 之后（自动命名） |
|------------------|------------------|
| 每个实例都需要 `instance_name` | 仅在需要自定义名称时指定 |
| 命名模式不一致 | 强制一致的命名模式 |
| 命名变更需手动更新 | 单一位置更新模式 |
| 更多 tfvars 内容需要维护 | 更简洁、可读的 tfvars |

## 多云命名对照

自动命名模式（coalesce + try）在各云厂商通用，只需调整资源类型前缀：

| 云厂商 | 命名模板 | 示例输出 |
|--------|---------|---------|
| 阿里云 | `"{type}-${var.env}-${each.key}"` | `"polar-simple-01"` |
| AWS | `"{type}-${var.env}-${each.key}"` | `"rds-simple-01"` |
| 腾讯云 | `"{type}-${var.env}-${each.key}"` | `"tc-redis-simple-01"` |
| Azure | `"{type}-${var.env}-${each.key}"` | `"az-mysql-simple-01"` |

命名规则本身是架构层面的决策，与云厂商无关。

## 参考资料

- 相关规则：`variable-layered-type-design.md`
---

# 原子层默认值：技术纯度原则

**优先级：** 高
**分类：** 三层架构

## 为什么重要

当原子层变量包含业务默认值（如 `["127.0.0.1"]`）时，会造成隐藏配置，违反"灵魂文档"原则。声明层应明确控制所有资源属性，而非依赖底层的隐藏默认值。

## 问题：原子层默认值导致状态漂移

### 错误代码

```hcl
# atomic/polardb/variables.tf
variable "security_ips" {
  description = "PolarDB 白名单 IP 列表"
  type        = list(string)
  default     = ["127.0.0.1"]  # ❌ 原子层包含业务默认值
}
```

**后果：**

1. 声明层未设置 `security_ips`
2. 控制层传递 `[]`（空值）
3. 原子层使用默认值 `["127.0.0.1"]`
4. API 未收到该值，使用云厂商默认值
5. State 记录云厂商默认值
6. **状态漂移发生！**

### 正确代码

```hcl
# atomic/polardb/variables.tf
variable "security_ips" {
  description = "PolarDB 白名单 IP 列表。空列表 = 不设置（使用云厂商默认值）"
  type        = list(string)
  default     = []  # ✅ 空 = 不干预
}

# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # ✅ 业务默认值在控制层

# declarative/simple/04-database/terraform.tfvars
security_ips = ["127.0.0.1"]  # ✅ 明确配置
```

## 层级职责矩阵

| 层级 | 职责 | 默认值 |
|------|------|--------|
| 声明层 | 用户意图，明确配置 | 可选，可使用控制层默认值 |
| 控制层 | 业务逻辑，编排 | ✅ 业务默认值在这里设置 |
| 原子层 | 技术实现 | ❌ 无业务默认值，保持纯净 |

## 状态对比机制

```
Terraform 对比：传递给资源的最终值 vs 状态文件中的值

而非：声明层值 vs 状态文件

执行链：声明层(tfvars) → 控制层(main.tf) → 原子层 → Provider 资源
```

## 决策规则

> **区分参数类型，按类型设置默认值：**
> - **API 可选参数**：原子层 `""`/`[]`，控制层设 API 默认值
> - **技术规格参数**：原子层可设合理默认值
> - **业务敏感参数**：原子层 `""`/`[]`，声明层显式配置
> - **功能开关关联参数**：控制层根据开关状态动态计算，关闭时强制填 API 默认值

## 参数类型分类

### 1. API 可选参数（必须空值）

**特征**：不传时 API 有默认值，传递后覆盖 API 默认值

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `security_ips` | 白名单 IP | 原子层 `[]`，控制层设 API 默认值 |
| `backup_time` | 备份时间 | 原子层 `""`，控制层设 API 默认值 |

### 2. 技术规格参数（可设合理默认值）

**特征**：必填参数，有"合理推荐"的技术选择

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `engine_version` | 数据库版本 | 原子层设 `"7.0"`（最新稳定版）|
| `payment_type` | 付费类型 | 原子层设 `"PayAsYouGo"`（开发友好）|
| `series_code` | 实例系列 | 原子层设 `"standard"`（最常用）|
| `service_code` | 服务代码 | 原子层设 `"rmq"`（RocketMQ 5.0）|

**为什么可以设默认值？**
1. 这些是必填参数，不传会导致 API 报错
2. 默认值是"技术规格"而非"业务决策"
3. 设置默认值不会导致状态漂移（API 收到的值 = State 记录的值）

### 3. 业务敏感参数（必须空值）

**特征**：涉及安全、合规、成本的业务决策

| 示例 | 说明 | 正确处理 |
|------|------|----------|
| `password` | 数据库密码 | 原子层 `""`，声明层显式配置 |
| `security_group_ids` | 安全组 | 原子层 `""`，声明层显式配置 |
| `whitelist` | IP 白名单 | 原子层 `[]`，声明层显式配置 |

### 4. 功能开关关联参数（控制层动态计算）

**特征**：参数仅在某个功能开关开启时才有意义，关闭时 API 返回特定默认值

**问题**：若全链路默认值设为“合理值”（如 `7`），而功能关闭时 API 返回不同值（如 `-1`），会导致每次 `terraform plan` 检测到状态漂移。

#### 错误代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = 7  # ❌ 无论是否开启跨域复制都传 7

# control/security/main.tf
copied_snapshots_retention_days = try(each.value.copied_snapshots_retention_days, 7)
# ❌ enable_cross_region_copy=false 时也传 7，API 实际存的是 -1 → 状态漂移
```

#### 正确代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = -1  # ✅ 与 API 默认值一致（功能未开启时 API 返回 -1）

# control/security/main.tf — 控制层根据开关状态动态计算
# 未开启跨域 → 强制 -1（与 API 一致，零漂移）
# 已开启跨域 → 取用户配置或默认 7
copied_snapshots_retention_days = try(each.value.enable_cross_region_copy, false)
  ? try(each.value.copied_snapshots_retention_days, 7)
  : -1
```

#### 识别特征

| 判断条件 | 说明 |
|------|------|
| 参数有前置功能开关 | 如 `enable_xxx` 控制 `xxx_retention_days` |
| 功能关闭时 API 有固定默认值 | 如跨域复制关闭时 API 返回 `-1` |
| 功能开启时参数才有业务意义 | 关闭时参数无实际作用 |

#### 处理规范

| 层级 | 处理方式 |
|------|--------|
| 原子层 | 默认值 = API 在功能关闭时的返回值（如 `-1`） |
| 控制层 | 三元表达式根据开关动态计算：关闭→API默认值，开启→用户值/业务默认值 |
| 声明层 | 通常无需显式配置（控制层自动处理） |

#### 已知案例

| 资源 | 开关参数 | 关联参数 | 关闭时 API 值 | 开启时默认值 |
|------|---------|---------|-------------|-------------|
| ECS 快照策略 | `enable_cross_region_copy` | `copied_snapshots_retention_days` | `-1` | `7` |

> **参考文件**：`control/security/main.tf` L153-157 — 跨域复制保留天数动态计算

### 4. 功能开关关联参数（控制层动态计算）

**特征**：参数仅在某个功能开关开启时才有意义，关闭时 API 返回特定默认值

**问题**：若全链路默认值设为“合理值”（如 `7`），而功能关闭时 API 返回不同值（如 `-1`），会导致每次 `terraform plan` 检测到状态漂移。

#### 错误代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = 7  # ❌ 无论是否开启跨域复制都传 7

# control/security/main.tf
copied_snapshots_retention_days = try(each.value.copied_snapshots_retention_days, 7)
# ❌ enable_cross_region_copy=false 时也传 7，API 实际存的是 -1 → 状态漂移
```

#### 正确代码

```hcl
# atomic/ecs-auto-snapshot-policy/variables.tf
default = -1  # ✅ 与 API 默认值一致（功能未开启时 API 返回 -1）

# control/security/main.tf — 控制层根据开关状态动态计算
# 未开启跨域 → 强制 -1（与 API 一致，零漂移）
# 已开启跨域 → 取用户配置或默认 7
copied_snapshots_retention_days = try(each.value.enable_cross_region_copy, false)
  ? try(each.value.copied_snapshots_retention_days, 7)
  : -1
```

#### 识别特征

| 判断条件 | 说明 |
|------|------|
| 参数有前置功能开关 | 如 `enable_xxx` 控制 `xxx_retention_days` |
| 功能关闭时 API 有固定默认值 | 如跨域复制关闭时 API 返回 `-1` |
| 功能开启时参数才有业务意义 | 关闭时参数无实际作用 |

#### 处理规范

| 层级 | 处理方式 |
|------|--------|
| 原子层 | 默认值 = API 在功能关闭时的返回值（如 `-1`） |
| 控制层 | 三元表达式根据开关动态计算：关闭→API默认值，开启→用户值/业务默认值 |
| 声明层 | 通常无需显式配置（控制层自动处理） |

#### 已知案例

| 资源 | 开关参数 | 关联参数 | 关闭时 API 值 | 开启时默认值 |
|------|---------|---------|-------------|-------------|
| ECS 快照策略 | `enable_cross_region_copy` | `copied_snapshots_retention_days` | `-1` | `7` |

> **参考文件**：`control/security/main.tf` L153-157 — 跨域复制保留天数动态计算

## API 默认值 vs 业务默认值

### 两种默认值

| 类型 | 说明 | 位置 |
|------|------|------|
| **API 默认值** | 云厂商的默认值（如 `["127.0.0.1"]`） | 控制层 `try(..., ["127.0.0.1"])` |
| **业务默认值** | 组织特定的值（如 VPC CIDR `["172.31.192.0/18"]`） | 声明层明确配置 |

### 示例：Redis security_ips

```hcl
# atomic/redis-community/variables.tf
variable "security_ips" {
  description = "Redis 白名单 IP 列表。空列表 = 不设置（使用控制层默认值）"
  type        = list(string)
  default     = []  # ✅ 空 = 纯技术层
}

# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # ✅ API 默认值

# declarative/simple/04-database/terraform.tfvars
security_ips = ["172.31.192.0/18"]  # ✅ 业务值：VPC 网段
```

### 为什么区分 API 默认值和业务默认值？

1. **API 默认值在控制层**：声明层未设置时防止状态漂移
2. **业务值在声明层**："灵魂文档"原则 - 业务决策明确可见
3. **灵活性**：不同环境可有不同业务值，控制层提供安全回退

## 检查清单

- [ ] 识别参数类型：API 可选 / 技术规格 / 业务敏感 / 功能开关关联
- [ ] API 可选参数：原子层 `""`/`[]`，控制层设 API 默认值
- [ ] 技术规格参数：原子层可设合理默认值
- [ ] 业务敏感参数：原子层 `""`/`[]`，声明层显式配置
- [ ] 功能开关关联参数：控制层根据开关状态动态计算，关闭时强制填 API 默认值
- [ ] 运行 `terraform plan` 验证无状态漂移

## 参考资料

- `variable-forcenew-defaults.md`（ForceNew 空值规则）
- `control-parameter-passthrough.md`（控制层参数透传）
- `layer-no-business-defaults.md`（控制层禁止业务默认值）

---

# 为输出添加描述

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

输出描述记录了模块可提供的数据及其使用方式。它们出现在生成的文档中，帮助用户理解模块接口。

## 错误示例

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.private[*].id
}

output "endpoint" {
  value = aws_db_instance.main.endpoint
}
```

**问题：** 用户必须阅读源代码才能理解每个输出提供什么。

## 正确示例

```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "VPC 的 ID"
}

output "subnet_ids" {
  value       = aws_subnet.private[*].id
  description = "私有子网 ID 列表"
}

output "endpoint" {
  value       = aws_db_instance.main.endpoint
  description = "连接端点，格式为 address:port"
}
```

## 使用对称命名

保持输出名称与上游资源属性名一致：

```hcl
# 好的做法 - 与 aws_iam_user 资源属性名匹配
output "user_arn" {
  value       = aws_iam_user.this.arn
  description = "IAM 用户的 ARN"
}

output "user_name" {
  value       = aws_iam_user.this.name
  description = "IAM 用户的名称"
}

output "user_unique_id" {
  value       = aws_iam_user.this.unique_id
  description = "AWS 分配的唯一 ID"
}

# 不好的做法 - 命名不一致
output "arn" {
  value = aws_iam_user.this.arn
}

output "username" { # 与 'name' 属性不匹配
  value = aws_iam_user.this.name
}
```

## 导出完整资源对象

为提高灵活性，在导出特定属性的同时导出完整资源：

```hcl
# 特定的常用输出
output "instance_id" {
  value       = aws_instance.web.id
  description = "EC2 实例的 ID"
}

output "instance_public_ip" {
  value       = aws_instance.web.public_ip
  description = "实例的公网 IP 地址"
}

# 完整资源供高级场景使用
output "instance" {
  value       = aws_instance.web
  description = "EC2 实例的所有属性。参见 https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/instance#attribute-reference"
}
```

## 文档化复杂输出

```hcl
output "load_balancer" {
  value = {
    arn       = aws_lb.main.arn
    dns_name  = aws_lb.main.dns_name
    zone_id   = aws_lb.main.zone_id
  }
  description = <<-EOT
  负载均衡器属性：
  - arn: 负载均衡器的 ARN
  - dns_name: 负载均衡器的 DNS 名称
  - zone_id: 用于别名记录的 Route 53 区域 ID
  EOT
}

output "subnets" {
  value = {
    for k, v in aws_subnet.this : k => {
      id         = v.id
      arn        = v.arn
      cidr_block = v.cidr_block
    }
  }
  description = "以子网名称为键的子网对象 Map"
}
```

## 使用 snake_case

```hcl
# 正确 - snake_case
output "security_group_id" {
  value       = aws_security_group.main.id
  description = "安全组的 ID"
}

# 错误 - 其他规范
output "securityGroupId" { # camelCase
  value = aws_security_group.main.id
}

output "SecurityGroupID" { # PascalCase
  value = aws_security_group.main.id
}
```

## 参考资料

- [输出值](https://developer.hashicorp.com/terraform/language/values/outputs)
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)

---

# 不要在输出中暴露密钥

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

输出会被记录、显示在 CI/CD 流水线中、存储在状态中，并可通过 `terraform output` 访问。输出中的密钥很容易泄露给未授权方。

## 错误示例

```hcl
output "database_password" {
  value = random_password.db.result
  # 密钥在 terraform output 中暴露！
}

output "api_credentials" {
  value = {
    client_id     = var.client_id
    client_secret = var.client_secret # 密钥泄露！
  }
}

output "connection_string" {
  value = "postgres://admin:${random_password.db.result}@${aws_db_instance.main.endpoint}/mydb"
  # 密码嵌入在输出中！
}
```

**问题：** 运行 `terraform output` 或查看 CI 日志会暴露密钥。

## 正确示例

### 将密钥存储到密钥管理器

```hcl
# 生成密码
resource "random_password" "db" {
  length  = 32
  special = true
}

# 存储到 AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name = "${var.environment}/database/password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db.result
}

# 输出位置，而非值
output "database_password_secret_arn" {
  value       = aws_secretsmanager_secret.db_password.arn
  description = "包含数据库密码的 Secrets Manager 密钥 ARN"
}

output "database_password_secret_name" {
  value       = aws_secretsmanager_secret.db_password.name
  description = "包含数据库密码的 Secrets Manager 密钥名称"
}
```

### 存储到 SSM Parameter Store

```hcl
resource "aws_ssm_parameter" "db_password" {
  name        = "/${var.environment}/database/password"
  description = "数据库主密码"
  type        = "SecureString"
  value       = random_password.db.result

  tags = var.tags
}

output "database_password_parameter_name" {
  value       = aws_ssm_parameter.db_password.name
  description = "数据库密码的 SSM 参数名。获取方式：aws ssm get-parameter --name ${aws_ssm_parameter.db_password.name} --with-decryption"
}
```

### 提供指引而非值

```hcl
output "database_credentials_info" {
  value       = "数据库凭据存储在 SSM 的 /${var.environment}/database/*"
  description = "SSM Parameter Store 中数据库凭据的位置"
}

output "retrieve_password_command" {
  value       = "aws ssm get-parameter --name '/${var.environment}/database/password' --with-decryption --query 'Parameter.Value' --output text"
  description = "获取数据库密码的 AWS CLI 命令"
}
```

## 如果必须输出密钥

在极少数必须输出密钥的场景（如引导初始化），始终标记为敏感：

```hcl
output "initial_admin_password" {
  value       = random_password.admin.result
  sensitive   = true
  description = "初始管理员密码。首次登录后请立即修改。"
}
```

**注意：** 即使设置了 `sensitive = true`：
- 值仍然以明文存储在状态文件中
- 可以通过 `terraform output -json` 获取
- 最好完全避免

## 根模块 vs 可复用模块

- **根模块（组件）：** 永远不要输出密钥
- **可复用模块：** 如果组合需要，可以输出密钥，但必须标记为 `sensitive = true`

```hcl
# 可复用模块中 - 标记 sensitive 后可接受
output "generated_password" {
  value       = random_password.this.result
  sensitive   = true
  description = "供父模块使用的生成密码"
}

# 根模块中 - 存储而非输出
resource "aws_ssm_parameter" "password" {
  name  = "/app/password"
  type  = "SecureString"
  value = module.database.generated_password
}
```

## 参考资料

- [敏感输出](https://developer.hashicorp.com/terraform/language/values/outputs#sensitive-suppressing-values-in-cli-output)
- [AWS Secrets Manager](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/secretsmanager_secret)
- [AWS SSM Parameter Store](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/ssm_parameter)

---

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


---

# Terraform 注释标准化规范

**优先级：** 中
**分类：** 变量与输出模式

## 为什么重要

注释的目标：让读者**快速理解参数填写规则和差异化限制**，而非展示所有细节。标准化的注释格式能减少沟通成本，提高团队协作效率。

## 核心原则

注释的目标：让读者**快速理解参数填写规则和差异化限制**，而非展示所有细节。

### 四大原则

1. **差异化优先** - 说明与其他配置/版本的关键差异
2. **简洁明了** - 一句话说清，避免冗长解释
3. **填写指引** - 告诉用户"怎么填"而非"这是什么"
4. **防漂提示** - 标注自动对齐机制，消除用户顾虑

---

## 注释分类与模板

### 1. 差异化注释（最重要）

用于说明参数在不同场景下的关键差异：

```hcl
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
```

**适用场景：**
- 经典版 vs 云原生版参数差异
- 不同产品版本的限制
- API 行为与用户预期不一致的情况

### 2. 行内简洁注释

参数后紧跟简短说明：

```hcl
# 正确示例
shard_count = 8       # 实际值由 instance_class 决定，原子层自动对齐
capacity    = 16384   # 实际值由 instance_class 决定，原子层自动对齐
engine_version = "5.0"  # 经典版仅支持 5.0

# 错误示例（过于冗长）
shard_count = 8  # 这是分片数量，经典版集群的分片数由规格代码决定，
                 # 用户设置的值会被API忽略，所以填写任意值都可以
```

**格式要求：**
- 一行内完成
- 说明关键限制或差异
- 无表情符号

### 3. 区块注释模板

用于资源/实例级别的差异化说明：

```hcl
#############################################################################
# 实例03：经典版集群版（切片架构）- 传统架构，仅支持 Redis 5.0
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
#############################################################################
```

### 4. 参数组注释模板

用于说明一组参数的联动关系：

```hcl
# ——— 架构配置 ———
# 经典版：shard_count/capacity 由规格决定，原子层自动对齐
deploy_type          = "classic"
shard_count          = 8       # 实际值由 instance_class 决定
capacity             = 16384   # 实际值由 instance_class 决定
read_only_count      = 0       # 经典版不支持读写分离
```

---

## 禁止的注释风格

### 禁止 1：过度校验注释

```hcl
# 错误
backup_period = ["Monday"]  # ✅ 已生效
enable_backup_log = 1       # ⚠️ State 实际值=0，可能需控制台确认

# 正确
backup_period = ["Monday"]
enable_backup_log = 1  # 经典版：API 可能忽略，需控制台确认
```

### 禁止 2：表情符号滥用

```hcl
# 错误
shard_count = 8  # ⚠️ 重要！API 会忽略这个值 ❗❗

# 正确
shard_count = 8  # 实际值由 instance_class 决定，原子层自动对齐
```

### 禁止 3：冗余解释

```hcl
# 错误
# 分片数量（shard_count）是指 Redis 集群中的数据分片个数，
# 每个分片包含一个主节点和若干从节点，经典版的分片数由规格代码决定...

# 正确
shard_count = 8  # 经典版：由规格决定，原子层自动对齐
```

---

## 产品版本差异化注释速查

### Redis 经典版 vs 云原生版

| 参数 | 经典版 | 云原生版 | 注释建议 |
|------|--------|----------|----------|
| `shard_count` | 由规格决定 | 用户自定义 | `# 经典版：由规格决定，原子层自动对齐` |
| `capacity` | 由规格决定 | 用户自定义 | `# 经典版：由规格决定，原子层自动对齐` |
| `engine_version` | 仅 5.0 | 5.0/6.0/7.0 | `# 经典版仅支持 5.0` |
| `read_only_count` | 有限制 | 无限制 | `# 经典版不支持读写分离` |

### PolarDB 标准版 vs 企业版

| 参数 | 标准版 | 企业版 | 注释建议 |
|------|--------|--------|----------|
| `db_node_count` | 1-16 | 1-15 | `# 标准版：1-16` |
| `creation_category` | SENormal | Normal | `# 标准版：与 .c 后缀规格一致` |
| `db_node_class` | .c 后缀 | 无后缀 | `# .c 后缀 = 标准版` |

---

## 自动对齐声明模板

当原子层实现了参数自动对齐时，使用以下标准注释：

```hcl
# 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
```

或行内简化：

```hcl
shard_count = 8  # 原子层自动对齐
```

---

## 注释更新规则

1. **发现新差异时** - 立即添加差异化注释
2. **实现自动对齐后** - 更新注释说明"原子层自动对齐"
3. **参数行为变化时** - 同步更新注释，删除过时说明
4. **避免重复** - 同一差异只在一处详细说明，其他位置引用

---

## 示例：完整实例注释

```hcl
#############################################################################
# 实例03：经典版集群版（切片架构）- 传统架构，仅支持 Redis 5.0
# 差异化说明：
#   - shard_count/capacity 由 instance_class 决定，API 忽略用户设置
#   - redis.sharding.basic.small.default = 8分片 × 2GB = 16GB
#   - 原子层已实现自动对齐防漂移，填写任意值不影响实际结果
#############################################################################
"03" = {
  # ——— 必填参数 ———
  instance_class = "redis.sharding.basic.small.default"  # 经典版集群版（固定 8 分片）
  vswitch_key    = "j-data"
  create         = true

  # ——— 架构配置 ———
  # 经典版：shard_count/capacity 由规格决定，原子层自动对齐
  deploy_type          = "classic"
  shard_count          = 8       # 实际值由 instance_class 决定，原子层自动对齐
  capacity             = 16384   # 实际值由 instance_class 决定，原子层自动对齐
  read_only_count      = 0       # 经典版不支持读写分离
  slave_read_only_count = 0

  # ——— 版本与可用区 ———
  engine_version = "5.0"  # 经典版仅支持 5.0
  zone_id        = "cn-hangzhou-j"
  secondary_zone_id = ""

  # ——— 认证配置 ———
  vpc_auth_mode = "Close"  # Close=密码认证
  password      = "Redis@Classic#123"

  # ——— 运维配置 ———
  is_auto_upgrade_open = "Enable"  # 开启自动升级小版本

  # ——— 备份配置 ———
  backup_period = ["Monday", "Wednesday", "Friday"]
  backup_time   = "02:00Z-03:00Z"
  enable_backup_log = 1  # 经典版：API 可能忽略，需控制台确认

  # ——— SSL 配置 ———
  ssl_enable = "Enable"  # 经典版支持 SSL
}
```

## 参考资料

- `variable-descriptions.md`（变量描述规范）
- `layer-comment-standard.md`（三层注释规范）
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)

---

# 避免使用 HEREDOC 编写 JSON/YAML

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

HEREDOC 字符串编写 JSON、YAML 和 IAM 策略容易出错、难以验证，且无法享受 Terraform 的类型检查。应使用原生函数和资源代替。

## 错误示例

```hcl
resource "aws_iam_role_policy" "lambda" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  # HEREDOC JSON - 难以维护，无验证
  policy = <<-EOF
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject"
        ],
        "Resource": "arn:aws:s3:::${var.bucket_name}/*"
      }
    ]
  }
  EOF
}

resource "kubernetes_config_map" "config" {
  metadata {
    name = "app-config"
  }

  # HEREDOC YAML - 插值问题，无验证
  data = {
    "config.yaml" = <<-EOF
    database:
      host: ${var.db_host}
      port: 5432
    logging:
      level: info
    EOF
  }
}
```

**问题：**
- 直到 apply 时才能发现语法错误
- 复杂结构难以维护
- 插值可能破坏 JSON/YAML 语法
- 无 IDE 结构支持

## 正确示例

### 使用 jsonencode() 编写 JSON

```hcl
resource "aws_iam_role_policy" "lambda" {
  name = "lambda-policy"
  role = aws_iam_role.lambda.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject"
        ]
        Resource = "arn:aws:s3:::${var.bucket_name}/*"
      }
    ]
  })
}
```

### 使用 IAM Policy Document 资源

```hcl
data "aws_iam_policy_document" "lambda" {
  statement {
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:PutObject"
    ]
    resources = ["arn:aws:s3:::${var.bucket_name}/*"]
  }

  statement {
    effect = "Allow"
    actions = [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]
    resources = ["arn:aws:logs:*:*:*"]
  }
}

resource "aws_iam_role_policy" "lambda" {
  name   = "lambda-policy"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.lambda.json
}
```

### 使用 yamlencode() 编写 YAML

```hcl
resource "kubernetes_config_map" "config" {
  metadata {
    name = "app-config"
  }

  data = {
    "config.yaml" = yamlencode({
      database = {
        host = var.db_host
        port = 5432
      }
      logging = {
        level = "info"
      }
    })
  }
}
```

### 使用 templatefile() 处理复杂模板

```hcl
# templates/user-data.sh
#!/bin/bash
echo "Environment: ${environment}"
echo "Region: ${region}"
apt-get update && apt-get install -y ${packages}

# main.tf
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  user_data = templatefile("${path.module}/templates/user-data.sh", {
    environment = var.environment
    region      = var.region
    packages    = join(" ", var.packages)
  })
}
```

## 何时可以使用 HEREDOC

缩进式 HEREDOC（`<<-EOT`）适用于：
- 纯文本描述
- Shell 脚本（当 templatefile 过于复杂时）
- 无结构的多行字符串

```hcl
resource "aws_sns_topic" "alerts" {
  name = "alerts"
}

output "usage_instructions" {
  value = <<-EOT
  订阅告警的步骤：
  1. 进入 AWS 控制台
  2. 导航到 SNS
  3. 订阅主题：${aws_sns_topic.alerts.arn}
  EOT
  description = "告警订阅说明"
}
```

## 参考资料

- [jsonencode 函数](https://developer.hashicorp.com/terraform/language/functions/jsonencode)
- [yamlencode 函数](https://developer.hashicorp.com/terraform/language/functions/yamlencode)
- [templatefile 函数](https://developer.hashicorp.com/terraform/language/functions/templatefile)
- [aws_iam_policy_document](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)

---

# 使用 Locals 提高代码可读性

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

`locals` 通过为复杂表达式命名来提高代码可读性。它减少重复、使代码自文档化，并集中管理计算值。

## 错误示例

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = {
    Name        = "${var.project}-${var.environment}-web"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_security_group" "web" {
  name = "${var.project}-${var.environment}-web-sg"

  tags = {
    Name        = "${var.project}-${var.environment}-web-sg"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_lb" "web" {
  name = "${var.project}-${var.environment}-alb"

  tags = {
    Name        = "${var.project}-${var.environment}-alb"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}
```

**问题：**
- 标签块在多个资源间重复
- 命名模式重复多次
- 修改需要更新多处

## 正确示例

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}

resource "aws_security_group" "web" {
  name = "${local.name_prefix}-web-sg"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web-sg"
  })
}

resource "aws_lb" "web" {
  name = "${local.name_prefix}-alb"

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-alb"
  })
}
```

## Locals 使用场景

### 为复杂表达式命名

```hcl
locals {
  # 代替重复此表达式
  is_production = var.environment == "prod" || var.environment == "production"

  # 计算子网 CIDR
  subnet_cidrs = [for i in range(var.subnet_count) : cidrsubnet(var.vpc_cidr, 8, i)]

  # 构建 ARN 模式
  log_group_arn = "arn:aws:logs:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:log-group:/aws/lambda/${var.function_name}:*"
}

resource "aws_lambda_function" "main" {
  function_name = var.function_name
  # ...

  environment {
    variables = {
      LOG_LEVEL = local.is_production ? "INFO" : "DEBUG"
    }
  }
}
```

### 集中管理计算值

```hcl
data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name

  # 构建资源 ARN
  bucket_arn = "arn:aws:s3:::${var.bucket_name}"
  table_arn  = "arn:aws:dynamodb:${local.region}:${local.account_id}:table/${var.table_name}"
}
```

### 转换输入数据

```hcl
variable "users" {
  type = list(object({
    name  = string
    email = string
    role  = string
  }))
}

locals {
  # 创建查找 map
  users_by_name = { for user in var.users : user.name => user }
  users_by_role = { for user in var.users : user.role => user... }

  # 过滤列表
  admin_users = [for user in var.users : user if user.role == "admin"]

  # 提取值
  all_emails = [for user in var.users : user.email]
}
```

### 条件逻辑

```hcl
variable "enable_https" {
  type    = bool
  default = true
}

variable "custom_domain" {
  type    = string
  default = null
}

locals {
  # 计算派生值
  protocol     = var.enable_https ? "https" : "http"
  default_port = var.enable_https ? 443 : 80

  # 处理可选值
  domain = coalesce(var.custom_domain, "${var.app_name}.example.com")

  # 构建 URL
  app_url = "${local.protocol}://${local.domain}"
}
```

## 组织 Locals

```hcl
locals {
  # 命名
  name_prefix = "${var.project}-${var.environment}"

  # 标签
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

locals {
  # 网络计算
  vpc_cidr        = var.vpc_cidr
  public_subnets  = [for i in range(3) : cidrsubnet(local.vpc_cidr, 4, i)]
  private_subnets = [for i in range(3) : cidrsubnet(local.vpc_cidr, 4, i + 3)]
}

locals {
  # 功能开关
  is_production     = var.environment == "prod"
  enable_monitoring = local.is_production
  enable_backups    = local.is_production
}
```

## 多云场景适配

### 阿里云: Locals 抽象多云命名

```hcl
locals {
  # 多云统一命名前缀
  name_prefix = "${var.project}-${var.environment}"

  # 阿里云常用 tags
  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "terraform"
  }
}

resource "alicloud_instance" "this" {
  name          = "${local.name_prefix}-web"
  tags          = merge(local.common_tags, { Role = "web" })
}
```

### Azure: Locals 抽象 SKU 和区域

```hcl
locals {
  # Azure SKU 按环境映射
  sku_by_env = {
    dev  = "B_Gen5_1"
    prod = "GP_Gen5_4"
  }

  # 区域别名映射
  region_alias = {
    east  = "eastus"
    west  = "westus2"
    china = "chinaeast2"
  }
}

resource "azurerm_mysql_flexible_server" "this" {
  name                = "${local.name_prefix}-mysql"
  location            = local.region_alias[var.region_alias]
  sku_name            = local.sku_by_env[var.environment]
  resource_group_name = var.resource_group_name
}
```

### 腾讯云: Locals 统一配置

```hcl
locals {
  # 腾讯云可用区映射
  az_map = {
    a = "ap-guangzhou-3"
    b = "ap-guangzhou-4"
  }

  # 按实例规格映射
  instance_types = {
    small  = "SA2.MEDIUM4"
    medium = "SA2.LARGE8"
    large  = "SA2.2XLARGE16"
  }
}
```

## 参考资料

- [Local Values](https://developer.hashicorp.com/terraform/language/values/locals)
- [表达式](https://developer.hashicorp.com/terraform/language/expressions)

---

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

---

# 使用数据源替代硬编码

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

数据源动态获取信息，而非硬编码值。这使配置更具可移植性、自文档化，并减少因过时或错误值导致的问题。

## 错误示例

```hcl
# 硬编码值可能过时或不正确
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0" # 什么区域？什么操作系统？还有效吗？
  instance_type = "t3.micro"
  subnet_id     = "subnet-abc123def456"   # 如果变了怎么办？
}

resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::my-bucket/*" # 假设了硬编码的账号
    }]
  })
}

# 硬编码的账号 ID
locals {
  account_id = "123456789012" # 复制粘贴，容易出错
}
```

**问题：**
- AMI ID 因区域而异
- 值会随时间过时
- 硬编码 ID 可能错误
- 无法跨环境移植

## 正确示例

### 动态 AMI 查找

```hcl
# 始终获取当前区域最新的 Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}
```

### 当前账号和区域

```hcl
# 动态获取当前 AWS 账号 ID
data "aws_caller_identity" "current" {}

# 获取当前区域
data "aws_region" "current" {}

locals {
  account_id = data.aws_caller_identity.current.account_id
  region     = data.aws_region.current.name
}

resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "arn:aws:s3:::${local.account_id}-app-data/*"
    }]
  })
}
```

### 可用区

```hcl
# 获取当前区域的可用区
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "private" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### 引用现有资源

```hcl
# 通过标签查找现有 VPC
data "aws_vpc" "main" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}

# 查找现有子网
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  filter {
    name   = "tag:Tier"
    values = ["private"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
  subnet_id     = data.aws_subnets.private.ids[0]
}
```

### IAM 策略文档

```hcl
# 使用数据源代替 JSON 字符串
data "aws_iam_policy_document" "s3_read" {
  statement {
    effect = "Allow"

    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]

    resources = [
      aws_s3_bucket.data.arn,
      "${aws_s3_bucket.data.arn}/*"
    ]
  }
}

resource "aws_iam_role_policy" "app" {
  name   = "s3-read-policy"
  role   = aws_iam_role.app.id
  policy = data.aws_iam_policy_document.s3_read.json
}
```

### 跨账号数据

```hcl
# 引用其他账号的资源
data "aws_secretsmanager_secret" "shared" {
  provider = aws.shared_services
  name     = "shared/api-key"
}

# 从其他 Terraform 状态引用
data "terraform_remote_state" "networking" {
  backend = "s3"

  config = {
    bucket = "terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

### 常用数据源

| 用途 | 数据源 |
|------|--------|
| 当前账号 | `aws_caller_identity` |
| 当前区域 | `aws_region` |
| 可用区 | `aws_availability_zones` |
| 最新 AMI | `aws_ami` |
| 现有 VPC | `aws_vpc` |
| 现有子网 | `aws_subnets` |
| IAM 策略 | `aws_iam_policy_document` |
| 密钥 | `aws_secretsmanager_secret_version` |
| SSM 参数 | `aws_ssm_parameter` |
| Route53 区域 | `aws_route53_zone` |
| ACM 证书 | `aws_acm_certificate` |

### GCP 数据源

```hcl
data "google_project" "current" {}

data "google_compute_zones" "available" {
  region = var.region
}

data "google_compute_image" "debian" {
  family  = "debian-11"
  project = "debian-cloud"
}
```

### Azure 数据源

```hcl
data "azurerm_subscription" "current" {}

data "azurerm_client_config" "current" {}

data "azurerm_resource_group" "existing" {
  name = "my-resource-group"
}
```

## 参考资料

- [数据源](https://developer.hashicorp.com/terraform/language/data-sources)
- [AWS 数据源](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)

---

# 使用动态块减少重复

**优先级：** 中
**分类：** 语言最佳实践

## 为什么重要

动态块根据变量生成重复的嵌套块，消除代码重复并实现灵活配置。它们是实现 DRY Terraform 代码的关键手段。

## 错误示例

```hcl
# 硬编码的入站规则 - 不灵活
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }

  # 添加新规则需要修改资源
}
```

**问题：**
- 添加规则需要修改代码
- 无法按环境变化
- 代码重复
- 不可复用

## 正确示例

### 基本动态块

```hcl
variable "ingress_rules" {
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP"
    },
    {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS"
    }
  ]
}

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web 服务器安全组"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 条件动态块

```hcl
variable "enable_https" {
  type    = bool
  default = true
}

variable "allowed_ssh_cidrs" {
  type    = list(string)
  default = [] # 空列表 = 无 SSH 访问
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  # 始终允许 HTTP
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # 条件允许 HTTPS
  dynamic "ingress" {
    for_each = var.enable_https ? [1] : []
    content {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  # 条件允许 SSH（仅在提供了 CIDR 时）
  dynamic "ingress" {
    for_each = length(var.allowed_ssh_cidrs) > 0 ? [1] : []
    content {
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = var.allowed_ssh_cidrs
    }
  }
}
```

### 使用 Map 的动态块

```hcl
variable "ingress_rules" {
  type = map(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = {
    http = {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
    https = {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.key # 使用 map 的 key 作为描述
    }
  }
}
```

### 嵌套动态块

```hcl
variable "load_balancer_config" {
  type = object({
    listeners = list(object({
      port     = number
      protocol = string
      actions  = list(object({
        type              = string
        target_group_arn  = string
      }))
    }))
  })
}

resource "aws_lb_listener" "main" {
  for_each = { for l in var.load_balancer_config.listeners : l.port => l }

  load_balancer_arn = aws_lb.main.arn
  port              = each.value.port
  protocol          = each.value.protocol

  dynamic "default_action" {
    for_each = each.value.actions
    content {
      type              = default_action.value.type
      target_group_arn  = default_action.value.target_group_arn
    }
  }
}
```

### 动态块用于设置

```hcl
variable "enable_encryption" {
  type    = bool
  default = true
}

variable "kms_key_id" {
  type    = string
  default = null
}

resource "aws_db_instance" "main" {
  identifier     = "mydb"
  engine         = "postgres"
  instance_class = "db.t3.micro"

  # 条件加密块
  dynamic "restore_to_point_in_time" {
    for_each = var.restore_from_snapshot != null ? [1] : []
    content {
      source_db_instance_identifier = var.restore_from_snapshot
      restore_time                  = var.restore_time
    }
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  dynamic "rule" {
    for_each = var.enable_encryption ? [1] : []
    content {
      apply_server_side_encryption_by_default {
        sse_algorithm     = var.kms_key_id != null ? "aws:kms" : "AES256"
        kms_master_key_id = var.kms_key_id
      }
    }
  }
}
```

### ECS 容器定义

```hcl
variable "containers" {
  type = list(object({
    name    = string
    image   = string
    cpu     = number
    memory  = number
    ports   = list(number)
    env     = map(string)
  }))
}

resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([
    for container in var.containers : {
      name      = container.name
      image     = container.image
      cpu       = container.cpu
      memory    = container.memory
      essential = true

      portMappings = [
        for port in container.ports : {
          containerPort = port
          protocol      = "tcp"
        }
      ]

      environment = [
        for key, value in container.env : {
          name  = key
          value = value
        }
      ]
    }
  ])
}
```

## 何时不应使用动态块

```hcl
# 如果只有 1-2 个静态块，直接写出来
# 动态块增加了复杂性 - 仅在需要时使用

resource "aws_security_group" "simple" {
  name = "simple-sg"

  # 只有两条规则？直接写
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## TypeMap 不能使用 Dynamic 块

### 问题：Provider schema 决定属性类型

Dynamic 块只能用于**嵌套块（Nested Block）**，不能用于 **TypeMap（属性类型）**。这由 Provider 的 schema 定义决定，不是 Terraform 语言特性。

### 错误示例

```hcl
# 错误：kms_encryption_context 是 TypeMap，不是嵌套块
resource "alicloud_alikafka_sasl_user" "this" {
  instance_id = var.instance_id
  username    = var.username
  password    = var.password

  # TypeMap 不能用 dynamic 块！
  dynamic "kms_encryption_context" {
    for_each = var.kms_encryption_context
    content {
      key   = kms_encryption_context.key
      value = kms_encryption_context.value
    }
  }
}
```

**错误信息**：
```
│ Error: Unsupported block type
│ on main.tf line 15:
│ Blocks of type "kms_encryption_context" are not expected here.
```

### 正确写法

```hcl
# TypeMap 直接赋值
resource "alicloud_alikafka_sasl_user" "this" {
  instance_id = var.instance_id
  username    = var.username
  password    = var.password

  # TypeMap 直接赋值 map
  kms_encryption_context = length(var.kms_encryption_context) > 0 ? var.kms_encryption_context : null
}
```

### 如何判断是 TypeMap 还是嵌套块？

| 判断方法 | TypeMap | Nested Block |
|----------|---------|--------------|
| Provider 文档 | 标注 `(TypeMap)` | 标注 `(Block)` |
| 语法 | `key = value` 直接赋值 | `block_name { ... }` 块语法 |
| Dynamic 支持 | 不支持 | 支持 |

### 常见 TypeMap 属性（阿里云）

| 资源 | 属性名 | 类型 |
|------|--------|------|
| `alicloud_alikafka_sasl_user` | `kms_encryption_context` | TypeMap |
| `alicloud_kms_key` | `tags` | TypeMap |
| `alicloud_instance` | `user_data` | TypeMap（部分场景） |

### 判断流程

```
需要动态配置一个属性
    │
    ▼
查看 Provider 文档 schema
    │
    ┌────┴────┐
    │         │
  TypeMap  Nested Block
    │         │
    ▼         ▼
  直接赋值  dynamic 块
```

## 多云场景适配

### 阿里云: SLB 监听器动态配置

```hcl
# 阿里云 SLB 动态监听规则
resource "alicloud_slb_listener" "this" {
  load_balancer_id = var.slb_id
  frontend_port    = 443
  protocol         = "https"

  dynamic "extension_blocks" {
    for_each = var.listener_rules
    content {
      rule_name  = extension_blocks.value.name
      domain     = extension_blocks.value.domain
      url        = extension_blocks.value.url
    }
  }
}
```

### Azure: NSG 动态安全规则

```hcl
# Azure NSG 动态入站规则
resource "azurerm_network_security_group" "this" {
  name                = "nsg-web"
  location            = var.location
  resource_group_name = var.resource_group_name

  dynamic "security_rule" {
    for_each = var.inbound_rules
    content {
      name                       = security_rule.value.name
      priority                   = security_rule.value.priority
      direction                  = "Inbound"
      access                     = "Allow"
      protocol                   = "Tcp"
      source_port_range          = "*"
      destination_port_range     = security_rule.value.port
      source_address_prefix      = security_rule.value.source_cidr
      destination_address_prefix = "*"
    }
  }
}
```

### 腾讯云: 安全组动态规则

```hcl
# 腾讯云安全组动态入站规则
resource "tencentcloud_security_group" "this" {
  name        = "sg-web"
  description = "Web security group"
}

resource "tencentcloud_security_group_rule" "ingress" {
  for_each = var.inbound_rules

  security_group_id = tencentcloud_security_group.this.id
  type              = "ingress"
  cidr_ip           = each.value.source_cidr
  ip_protocol       = "tcp"
  port_range        = each.value.port
  policy            = "accept"
}
```

## 参考资料

- [动态块](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks)
- [for_each 元参数](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)

---

# 锁定 Provider 版本

**优先级：** 中
**分类：** Provider 配置

## 为什么重要

未锁定版本的 Provider 在发布破坏性变更时可能导致意外行为。锁定版本以确保可复现性，同时允许受控升级。

## 错误示例

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      # 无版本约束 - 使用最新版本
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

**问题：** 不同时间运行 `terraform init` 可能获取不同版本的 Provider，产生不同行为。

## 正确示例

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0" # 允许 5.x，不允许 6.0
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20.0, < 3.0.0"
    }
  }
}
```

## 版本约束策略

### 生产环境 - 保守策略

```hcl
# 锁定到精确版本
aws = {
  source  = "hashicorp/aws"
  version = "5.31.0"
}
```

### 开发环境 - 灵活策略

```hcl
# 允许小版本更新
aws = {
  source  = "hashicorp/aws"
  version = "~> 5.31" # 允许 5.31.x
}
```

### 平衡策略

```hcl
# 允许小版本更新，阻止大版本
aws = {
  source  = "hashicorp/aws"
  version = ">= 5.0.0, < 6.0.0"
}
```

## 锁定文件

Terraform 创建 `.terraform.lock.hcl` 记录精确版本：

```hcl
# .terraform.lock.hcl（自动生成）
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
  ]
}
```

**最佳实践：**
- 将 `.terraform.lock.hcl` 提交到版本控制
- 使用 `terraform init -upgrade` 更新
- 在 PR 中审查锁定文件变更

## 更新 Provider

```bash
# 在约束范围内更新所有 Provider
terraform init -upgrade

# 检查过时的 Provider
terraform version

# 更新特定 Provider
terraform providers lock -platform=linux_amd64 hashicorp/aws
```

## 多 Provider 配置

```hcl
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = "~> 5.0"
      configuration_aliases = [aws.west]
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
```

## 参考资料

- [Provider 要求](https://developer.hashicorp.com/terraform/language/providers/requirements)
- [依赖锁定文件](https://developer.hashicorp.com/terraform/language/files/dependency-lock)

---

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

---

# Provider 文档与 API 查找规范

**优先级：** 中
**分类：** Provider 配置

> 从 Provider 源码和 API 文档中提取参数定义，确保原子层模块参数完整、默认值安全、ForceNew 行为正确。

## 为什么重要

在原子层参数审计、模块开发、参数约束确认时，需要查阅：
1. **API 文档**：确认参数必填规则、取值范围、联动约束
2. **Provider 文档**：确认 Terraform 参数映射、默认值、ForceNew 行为
3. **Provider 源码**：最权威的参数定义和 API 调用逻辑

如果不知道如何系统性地查找这些文档，会导致：
- 凭猜测设置默认值，引发运行时错误
- 遗漏 API 级约束（条件必填、互斥参数）
- 无法确认参数是否为 ForceNew

## 通用查找流程

```
┌─────────────────────────────────────────────────────────────────┐
│              Provider / API 文档查找流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定位 Provider 源码                                          │
│     terraform-provider-{provider}/                              │
│                                                                 │
│  2. 提取关键信息                                                │
│     ┌───────────────────────────────────────────────────────┐   │
│     │ 信息            │ 提取位置            │ 说明         │   │
│     ├───────────────────────────────────────────────────────┤   │
│     │ API Service     │ API 调用参数         │ 产品标识     │   │
│     │ API Version     │ API 调用参数         │ API 版本     │   │
│     │ Action Name     │ API 调用逻辑        │ 操作名       │   │
│     │ Schema 字段     │ Schema 定义块       │ 参数约束     │   │
│     └───────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 构造文档 URL                                                │
│     ┌───────────────────────────────────────────────────────┐   │
│     │ API 文档  → 各云厂商 API 文档中心                      │   │
│     │ Provider  → Terraform Registry                         │   │
│     └───────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 交叉验证                                                    │
│     Provider Schema ↔ API 文档 ↔ 实际行为                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 通用 Provider 源码关键位置

### 1. Schema 定义（参数完整性审计入口）

```go
// 所有 Provider 的 Schema 结构一致
Schema: map[string]*schema.Schema{
  "param_name": {
    Type:     schema.TypeString,
    Optional: true,   // 可选
    Required: true,   // 必填
    Computed: true,   // API 返回
    ForceNew: true,   // 修改触发重建
    Default:  "val",  // Provider 内置默认值
  },
}
```

### 2. Create 函数（API 调用逻辑）

```go
// 各厂商 Provider 的 Create 函数结构类似
// 提取 API Service、Version、Action 信息
action := "CreateXxx"   // API Action 名
// API 调用...
```

### 3. Update 函数（参数更新行为）

```go
// 关键：d.HasChange("param") 判断哪些参数变更触发 Update
if d.HasChange("param_name") {
  update = true
}
```

### 4. 数据源定义（查询类资源）

```go
// 数据源用于查询已有资源
// 各厂商命名：dataSource_{provider}_{resource}.go
```

## 审计标准操作步骤

### Step 1: 定位 Provider 源码

| 云厂商 | Provider 源码路径 |
|--------|-------------------|
| 阿里云 | `terraform-provider-alicloud/alicloud/resource_alicloud_*.go` |
| AWS | `terraform-provider-aws/internal/service/*/<resource>.go` |
| 腾讯云 | `terraform-provider-tencentcloud/tencentcloud/services/*/<resource>.go` |
| Azure | `terraform-provider-azurerm/internal/services/*/<resource>.go` |

### Step 2: 提取 Schema 信息

对照 `module-parameter-completeness` 规则，提取所有 Optional/Required 字段。

### Step 3: 查阅 API 文档

从 Provider 源码提取 Service + Version + Action，在对应云厂商 API 文档中心查找。

### Step 4: 查阅 Provider 文档

在 Terraform Registry 查找对应 Provider 的文档页面。

### Step 5: 交叉验证

| 验证项 | 方法 |
|--------|------|
| 参数是否存在 | Provider Schema ↔ API 参数列表 |
| 默认值是否安全 | Provider Default ↔ API 文档描述 |
| ForceNew 是否正确 | Provider ForceNew ↔ API 重建行为 |
| 条件必填 | API 文档 ↔ 原子层 validation |

---

## 附录 A：阿里云（Alibaba Cloud）

### Provider 文档 URL

```
https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}
```

**规则**：去掉 `alicloud_` 前缀即为 URL 路径

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `alicloud_alb_load_balancer` | `alb_load_balancer` |
| `alicloud_wafv3_instance` | `wafv3_instance` |

### OpenAPI 文档

**可靠查找路径**（按优先级排序）：

1. **OpenAPI 在线调试台**（最可靠）：`https://next.api.aliyun.com/api/{service}/{version}/{action}`
2. **帮助中心搜索**：`https://help.aliyun.com/zh/` → 搜索产品名 → 开发者参考
3. **API 目录页**：`https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-dir/`
4. **Provider 文档**：`https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}`

> **经验教训**：阿里云 OpenAPI 文档 URL 经常变动，精确 URL 大概率 404！请优先使用在线调试台。

### Provider 源码关键模式

```go
// API 调用格式
response, err = client.RpcPost("waf-openapi", "2021-10-01", action, nil, request, false)
//                       ^^^^^^^^^^^^  ^^^^^^^^^^^
//                       Service Name  API Version

// Action Name → API 文档 URL：PascalCase → kebab-case，前面加 api-
// CreatePostpaidInstance → api-waf-openapi-2021-10-01-createpostpaidinstance
```

### Service Name 到 product-slug 映射表

| Provider Service Name | product-slug | 帮助中心路径 |
|----------------------|-------------|-------------|
| `waf-openapi` | waf/web-application-firewall-3-0 | `help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/` |
| `cas` | ssl-certificates-service | `help.aliyun.com/zh/ssl-certificates-service/developer-reference/` |
| `Alb` | alb | `help.aliyun.com/zh/alb/developer-reference/` |
| `Ecs` | ecs | `help.aliyun.com/zh/ecs/developer-reference/` |
| `Rds` | rds | `help.aliyun.com/zh/rds/developer-reference/` |
| `Dds` (MongoDB) | mongodb | `help.aliyun.com/zh/mongodb/developer-reference/` |
| `Kvstore` (Redis) | redis | `help.aliyun.com/zh/redis/developer-reference/` |
| `Polardb` | polardb | `help.aliyun.com/zh/polardb/developer-reference/` |
| `Nas` | nas | `help.aliyun.com/zh/nas/developer-reference/` |
| `Alikafka` | alikafka | `help.aliyun.com/zh/alikafka/developer-reference/` |
| `Mse` | mse | `help.aliyun.com/zh/mse/developer-reference/` |
| `Vpc` | vpc | `help.aliyun.com/zh/vpc/developer-reference/` |

### 精确 URL 404 排查流程

```
精确 URL 404
→ 1. 尝试 API 目录页：help.aliyun.com/zh/{slug}/developer-reference/
→ 2. 搜索引擎："阿里云 {Action名} {Service名} API"
→ 3. 在线调试台：next.api.aliyun.com/api/{service}/{version}/
→ 4. 回归 Provider 源码（最权威，不会 404）
```

---

## 附录 B：AWS

### Provider 文档 URL

```
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `aws_db_instance` | `db_instance` |
| `aws_elasticache_cluster` | `elasticache_cluster` |
| `aws_msk_cluster` | `msk_cluster` |

### API 文档

**查找路径**：
1. **AWS API Documentation**：`https://docs.aws.amazon.com/{service}/latest/APIReference/`
   - RDS: `docs.aws.amazon.com/AmazonRDS/latest/APIReference/`
   - ElastiCache: `docs.aws.amazon.com/AmazonElastiCache/latest/APIReference/`
2. **Terraform Provider 文档**：`registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/{resource}`
3. **Provider 源码**：`github.com/hashicorp/terraform-provider-aws`

### Provider 源码关键模式

```go
// AWS Provider 源码位置
internal/service/{service}/{resource}.go

// API 调用格式（使用 AWS SDK）
input := &rds.CreateDBInstanceInput{
  DBInstanceClass: aws.String("db.t3.micro"),
  Engine:         aws.String("mysql"),
}

// Schema 定义
"engine_version": {
  Type:     schema.TypeString,
  Optional: true,
  ForceNew: true,  // 修改触发重建
},
```

### 典型案例

| 资源 | 文档查找要点 |
|------|-------------|
| `aws_db_instance` | engine_version 是 ForceNew；storage_type 可在线修改 |
| `aws_msk_cluster` | kafka_version 需查 MSK API 确认可用版本 |
| `aws_elasticache_cluster` | engine_version_validations 在 API 端 |

---

## 附录 C：腾讯云（Tencent Cloud）

### Provider 文档 URL

```
https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `tencentcloud_mysql_instance` | `mysql_instance` |
| `tencentcloud_redis_instance` | `redis_instance` |
| `tencentcloud_ckafka_instance` | `ckafka_instance` |

### API 文档

**查找路径**：
1. **API Explorer**（最实用）：`https://console.cloud.tencent.com/api/explorer`
   - 可直接调试 API、查看请求/响应参数
2. **腾讯云 API 文档中心**：`https://cloud.tencent.com/document/api/{product}/{version}`
   - MySQL: `cloud.tencent.com/document/api/236/15880`
   - Redis: `cloud.tencent.com/document/api/239/20231`
3. **Provider 源码**：`github.com/TencentCloud/terraform-provider-tencentcloud`

### Provider 源码关键模式

```go
// 腾讯云 Provider 源码位置
tencentcloud/services/{service}/{resource}.go

// API 调用格式（使用腾讯云 SDK）
request := tcr.NewCreateRepositoryRequest()
request.RegistryId = &registryId

// Schema 定义（与通用结构一致）
"instance_name": {
  Type:     schema.TypeString,
  Required: true,
},
```

---

## 附录 D：Azure

### Provider 文档 URL

```
https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `azurerm_mysql_flexible_server` | `mysql_flexible_server` |
| `azurerm_redis_cache` | `redis_cache` |
| `azurerm_kubernetes_cluster` | `kubernetes_cluster` |

### API 文档

**查找路径**：
1. **Azure REST API Docs**：`https://learn.microsoft.com/en-us/rest/api/{service}/`
   - MySQL: `learn.microsoft.com/en-us/rest/api/mysql/`
   - Redis: `learn.microsoft.com/en-us/rest/api/redis/`
2. **Terraform Provider 文档**：`registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/{resource}`
3. **Provider 源码**：`github.com/hashicorp/terraform-provider-azurerm`

### Provider 源码关键模式

```go
// Azure Provider 源码位置
internal/services/{service}/{resource}.go

// Schema 定义（使用 pluginsdk 或 hashicorp-go-azure-helpers）
"sku_name": {
  Type:     schema.TypeString,
  Required: true,
  ValidateFunc: validation.StringInSlice([]string{
    "B_Gen5_1", "GP_Gen5_2", "MO_Gen5_2",
  }, false),
},
```

---

## 相关规则

- `module-parameter-completeness` - 原子层参数完整性检查
- `provider-optional-api-mandatory` - Provider 参数 Schema 类型与处理规范
- `code-reference-documentation` - 原子层代码参考规范

## 参考资料

- `module-parameter-completeness.md`（参数完整性检查）
- `provider-optional-api-mandatory.md`（Provider 参数 Schema 类型）
- [AWS Provider 文档](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Azure Provider 文档](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [阿里云 Provider 文档](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs)
- [腾讯云 Provider 文档](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs)

---

# 调整并行度参数

**优先级：** 低中
**分类：** 性能优化

## 为什么重要

Terraform 默认 10 个并发操作适用于大多数场景，但大规模部署或 API 限频的情况可能需要调整。

## 错误示例

```bash
# 遇到 API 限频时使用默认并行度
terraform apply
# Error: Rate limit exceeded
# Error: Too many requests

# 或大规模部署时使用默认值（很慢）
terraform apply # 500+ 资源需要很长时间
```

## 正确示例

```bash
# 降低并行度以应对 API 限频（GitHub、Cloudflare 等）
terraform apply -parallelism=3

# 提高并行度用于大规模部署
terraform apply -parallelism=20

# 串行执行用于调试
terraform apply -parallelism=1
```

## 默认行为

```bash
# 默认：10 个并发操作
terraform apply
```

## 提高并行度

适用于具有大量独立资源的大规模部署：

```bash
# 提高并行度加快 apply
terraform apply -parallelism=20
```

**适用场景：**
- 100+ 个独立资源
- 无 API 限频问题
- 资源间无互相依赖
- 到 Provider 的网络连接速度快

## 降低并行度

适用于 API 限频或调试：

```bash
# 降低并行度以应对 API 限频
terraform apply -parallelism=5

# 串行执行用于调试
terraform apply -parallelism=1
```

**适用场景：**
- Provider API 限频（GitHub、Cloudflare 常见）
- 调试资源创建顺序
- 共享资源争用
- 内存受限环境

## Provider 特定注意事项

### AWS

```hcl
# AWS 通常能处理较高并行度
# 但某些服务有限制（如 IAM）
terraform apply -parallelism=15
```

### GitHub

```hcl
# GitHub API 限频严格
terraform apply -parallelism=3
```

### Kubernetes

```hcl
# Kubernetes API Server 可能过载
terraform apply -parallelism=5
```

## 使用环境变量

```bash
# 设置默认并行度
export TF_CLI_ARGS_apply="-parallelism=15"
export TF_CLI_ARGS_plan="-parallelism=15"

terraform apply # 使用 15
```

## CI/CD 配置

```yaml
# GitHub Actions 示例
jobs:
  terraform:
    steps:
      - name: Terraform Apply
        run: terraform apply -parallelism=10 -auto-approve
        env:
          TF_IN_AUTOMATION: true
```

## 衡量影响

```bash
# 测试不同并行度的执行时间
time terraform apply -parallelism=5 -auto-approve
time terraform apply -parallelism=10 -auto-approve
time terraform apply -parallelism=20 -auto-approve
```

## 依赖关系覆盖并行度

有依赖关系的资源无论并行度设置如何都会串行执行：

```hcl
resource "aws_vpc" "main" {
  # 先创建
}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id # 等待 VPC
}

resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id # 等待子网
}
```

## 参考资料

- [Terraform CLI 配置](https://developer.hashicorp.com/terraform/cli/commands/apply#parallelism-n)

---

# 掌握调试技巧

**优先级：** 低中
**分类：** 性能优化

## 为什么重要

当 Terraform 行为异常时，调试日志有助于定位根因。掌握如何启用和解读调试输出可以加速排障。

## 错误示例

```bash
# 出错时不带调试信息运行
terraform apply
# Error: something went wrong
# 不知道发生了什么或为什么
```

## 正确示例

```bash
# 启用调试日志以理解失败原因
TF_LOG=DEBUG terraform apply

# 或记录到文件供后续分析
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform.log"
terraform apply
```

## 启用调试日志

### 临时（单次命令）

```bash
# 完整调试输出
TF_LOG=DEBUG terraform plan

# Trace 级别（最详细）
TF_LOG=TRACE terraform apply

# 可用级别：TRACE、DEBUG、INFO、WARN、ERROR
TF_LOG=INFO terraform plan
```

### 持久化到文件

```bash
# 日志输出到文件而非标准输出
export TF_LOG=DEBUG
export TF_LOG_PATH="./terraform.log"

terraform plan

# 查看日志
cat terraform.log
```

### Provider 专用日志

```bash
# 仅记录 Terraform 核心
TF_LOG_CORE=DEBUG terraform plan

# 仅记录 Provider 操作
TF_LOG_PROVIDER=DEBUG terraform plan

# 组合使用
TF_LOG_CORE=WARN TF_LOG_PROVIDER=DEBUG terraform plan
```

## 常见调试场景

### 慢 Plan

```bash
# 计时并识别慢资源
TF_LOG=DEBUG terraform plan 2>&1 | tee plan.log

# 查找慢操作
grep -i "seconds" plan.log
grep -i "waiting" plan.log
```

### 认证问题

```bash
# 调试凭据问题
TF_LOG=DEBUG terraform plan 2>&1 | grep -i "auth\|credential\|token\|401\|403"

# AWS 专用
AWS_DEBUG=1 TF_LOG=DEBUG terraform plan
```

### Provider 错误

```bash
# 捕获完整 API 响应
TF_LOG=TRACE terraform apply 2>&1 | tee apply.log

# 搜索错误
grep -i "error\|failed\|invalid" apply.log
```

## Terraform Plan 调试

### 理解 Plan 输出

```bash
# 详细 Plan 及变更原因
terraform plan -detailed-exitcode

# 退出码：
# 0 = 无变更
# 1 = 错误
# 2 = 有变更
```

### 查看变更内容

```bash
# JSON 输出供程序化分析
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | select(.change.actions != ["no-op"])'
```

### 刷新和状态问题

```bash
# 跳过刷新以隔离问题
terraform plan -refresh=false

# 与刷新对比
terraform plan -refresh=true

# 定向特定资源
terraform plan -target=aws_instance.web
```

## 状态调试

```bash
# 列出状态中的所有资源
terraform state list

# 查看特定资源
terraform state show aws_instance.web

# 拉取状态供检查
terraform state pull > state.json
jq '.resources[] | select(.type == "aws_instance")' state.json
```

## 崩溃调试

```bash
# Terraform 在 panic 时创建 crash.log
cat crash.log

# 启用 core dump
export TF_LOG=TRACE
export TF_LOG_PATH="./crash_debug.log"
terraform apply
```

## 网络调试

```bash
# 调试 HTTP 请求
export TF_LOG=TRACE
export HTTPS_PROXY=http://localhost:8080 # 配合 mitmproxy/Fiddler 使用

terraform plan
```

## 常见问题及解决方案

### 卡在 "Refreshing state..."

```bash
# 可能原因：API 限频或网络问题
# 解决方案：降低并行度
terraform plan -parallelism=1

# 或临时跳过刷新
terraform plan -refresh=false
```

### "Resource already exists"

```bash
# 导入已存在的资源
terraform import aws_instance.web i-1234567890abcdef0

# 或检查命名冲突
terraform state list | grep instance
```

### "Error acquiring state lock"

```bash
# 查找并释放卡住的锁
terraform force-unlock LOCK_ID

# 检查 DynamoDB 中的锁
aws dynamodb scan --table-name terraform-locks
```

### Provider 插件错误

```bash
# 清除插件缓存
rm -rf .terraform/

# 重新初始化
terraform init -upgrade

# 检查 Provider 版本
terraform providers
```

## 调试工作流

```bash
#!/bin/bash
# debug-terraform.sh

set -e

echo "=== Terraform 调试会话 ==="
echo "时间戳: $(date)"
echo "目录: $(pwd)"
echo ""

# 捕获环境信息
echo "=== 环境信息 ==="
terraform version
echo ""

# 启用调试
export TF_LOG=DEBUG
export TF_LOG_PATH="./debug_$(date +%Y%m%d_%H%M%S).log"

# 运行命令
echo "=== 执行: terraform $@ ==="
terraform "$@"

echo ""
echo "=== 调试日志已保存到: $TF_LOG_PATH ==="
```

## 参考资料

- [调试 Terraform](https://developer.hashicorp.com/terraform/internals/debugging)
- [Terraform 日志](https://developer.hashicorp.com/terraform/cli/config/environment-variables#tf_log)

---

# 部署验证：漂移检测流程

**优先级：** 关键
**分类：** 测试与验证

## 为什么重要

`terraform apply` 后，云厂商 API 可能设置与代码不同的值，原因包括：
- API 默认值与 Terraform 默认值不同
- API 忽略的参数（由实例规格决定）
- Terraform 未预料的自动生成值
- Provider 端的标准化处理

这会导致**状态漂移**：`terraform.tfvars` ≠ `terraform.tfstate`

## 验证流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           部署验证流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. terraform plan                                                          │
│     │                                                                       │
│     ▼                                                                       │
│  2. terraform apply（首次）                                                  │
│     │                                                                       │
│     ▼                                                                       │
│  3. 等待 apply 完成（检查资源状态）                                            │
│     │                                                                       │
│     ▼                                                                       │
│  4. terraform plan（第二次 - 漂移检测）                                       │
│     │                                                                       │
│     ├── 无变更？ ──────────────────────► 5. 验证 state = code               │
│     │                                              │                        │
│     │                                              ▼                        │
│     │                                         6. 最终版本 ✓               │
│     │                                                                       │
│     └── 检测到变更？                                                          │
│              │                                                              │
│              ▼                                                              │
│         分析漂移原因：                                                        │
│         • API 忽略了参数？                                                    │
│         • API 默认值与代码不同？                                               │
│         • 规格决定的值？                                                      │
│              │                                                              │
│              ▼                                                              │
│         在原子层修复（自动对齐）                                                │
│              │                                                              │
│              ▼                                                              │
│         返回步骤 4                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 分步流程

### 步骤 1：初始 Plan

```bash
cd environments/${ENV}/04-database
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

检查：
- [ ] 无意外的资源销毁
- [ ] 资源数量符合预期
- [ ] Plan 输出中无敏感数据

### 步骤 2：首次 Apply

```bash
terraform apply -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

**重要：** 必须 100% 完成才可继续。

### 步骤 3：等待完成

对于长时间运行的资源，验证状态：

| 资源类型 | 检查命令 | 预期状态 |
|----------|----------|----------|
| ACK 集群 | aliyun cs GET /clusters/<id> | running |
| PolarDB | aliyun polardb DescribeDBClusters | Running |
| Redis | aliyun r-kvstore DescribeInstances | Normal |
| ES | aliyun elasticsearch DescribeInstance | active |

### 步骤 4：第二次 Plan（漂移检测）

```bash
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

**这是关键验证步骤！**

| Plan 输出 | 含义 | 操作 |
|-----------|------|------|
| No changes | 状态与代码一致 | ✓ 进入步骤 5 |
| X to update | 检测到漂移 | ✗ 分析并修复 |

### 步骤 5：State-Code 一致性检查

```bash
# 显示当前状态
terraform show -json > current-state.json

# 与代码对比
# 需验证的关键字段：
# - instance_class 与 tfvars 一致
# - shard_count 与 tfvars 一致
# - capacity 与 tfvars 一致
# - 无意外的 null 值
```

### 步骤 6：最终版本

所有检查通过后：
```bash
# 提交验证后的状态
git add terraform.tfstate
git commit -m "feat: 已验证部署 - 无漂移"
```

## 常见漂移原因与修复

### 原因 1：API 默认值 ≠ 代码默认值

```hcl
# tfvars
security_ips = []  # 用户期望：不设置

# API 默认值
security_ips = ["127.0.0.1"]  # API 设置此值

# 结果：漂移！Code ≠ State
```

**修复：** 在控制层设置 API 默认值

```hcl
# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # API 默认值
```

### 原因 2：API 忽略参数（规格决定）

```hcl
# tfvars
shard_count = 2
capacity    = 4096

# instance_class = "redis.sharding.basic.small.default"
# API 忽略 shard_count/capacity，使用规格固定值：
#   shard_count = 8（此规格固定值）
#   capacity    = 16384（此规格固定值）

# 结果：漂移！Code ≠ State
```

**修复：** 原子层自动对齐

```hcl
# atomic/redis-community/main.tf
locals {
  is_classic_cluster = can(regex("\\.default$", var.instance_class)) && 
                       can(regex("sharding", lower(var.instance_class)))

  classic_specs = {
    "redis.sharding.basic.small.default"  = { shard_count = 8,  capacity = 16384 }
    "redis.sharding.basic.medium.default" = { shard_count = 8,  capacity = 32768 }
  }

  effective_shard_count = local.is_classic_cluster ? 
    local.classic_specs[var.instance_class].shard_count : var.shard_count

  effective_capacity = local.is_classic_cluster ? 
    local.classic_specs[var.instance_class].capacity : var.capacity
}
```

### 原因 3：ForceNew 参数变更

```hcl
# 修改 ForceNew 参数会导致资源替换
# terraform plan 显示："forces replacement"

# 常见 ForceNew 参数：
# - alicloud_polardb_cluster: zone_id, db_version
# - alicloud_kvstore_instance: engine_version（有时）
# - alicloud_instance: image_id, instance_type
```

**修复：** ForceNew 参数使用空字符串默认值

```hcl
# atomic/polardb/variables.tf
variable "strict_consistency" {
  description = "ForceNew 参数。空 = 不设置（避免触发替换）"
  type        = string
  default     = ""  # 空 = 不设置，避免 ForceNew
}

# atomic/polardb/main.tf
strict_consistency = var.strict_consistency != "" ? var.strict_consistency : null
```

## 漂移分析检查清单

当第二次 plan 显示变更时：

- [ ] 确定哪个资源有漂移
- [ ] 确定哪个参数不同
- [ ] 检查参数是否为 ForceNew
- [ ] 检查参数是否由规格决定
- [ ] 检查 API 默认值是否与代码默认值不同
- [ ] 在原子层/控制层应用适当修复
- [ ] 重新运行步骤 4 直到无变更

## 验证脚本模板

```bash
#!/bin/bash
# verify-deployment.sh

set -e

PHASE=$1
TFVARS_DIR="environments/${ENV}/${PHASE}"

echo "=== 步骤 1：初始 Plan ==="
cd $TFVARS_DIR
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"

echo ""
echo "=== 步骤 2：Apply ==="
read -p "继续 apply？(y/n) " confirm
if [ "$confirm" = "y" ]; then
  terraform apply -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars" -auto-approve
fi

echo ""
echo "=== 步骤 3：等待完成 ==="
echo "检查控制台确认资源状态..."

echo ""
echo "=== 步骤 4：第二次 Plan（漂移检测） ==="
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"

echo ""
echo "=== 验证完成 ==="
```

## 检查清单

- [ ] 步骤 1：初始 terraform plan 已审查
- [ ] 步骤 2：首次 terraform apply 已完成
- [ ] 步骤 3：资源已完全配置（状态已验证）
- [ ] 步骤 4：第二次 terraform plan 显示 **无变更**
- [ ] 步骤 5：State-Code 一致性已验证
- [ ] 步骤 6：最终版本已提交

## 决策规则

> **在第二次 terraform plan 显示"无变更"之前，永远不要认为部署完成。首次 apply 可能因 API 默认值、规格决定值或忽略参数引入状态漂移。在标记任务完成前，务必验证无漂移状态。**

## 参考资料

- resource-spec-auto-correction.md - 规格决定值的自动对齐
- variable-atomic-defaults.md - 原子层默认值策略
- variable-forcenew-defaults.md - ForceNew 参数处理
---

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

---

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

---

# 六步测试法 — 声明层完整测试流程

**优先级：** 高
**分类：** 测试与验证

> 适用范围：三层架构（声明层 → 控制层 → 原子层）的声明层环境测试

## 为什么重要

声明层是整个三层架构的最终配置入口。六步测试法提供系统化的测试流程，确保从代码审计到部署验证的每个环节都不遗漏，减少生产事故风险。

## 测试六步流程

### Step 1 — 规格一致性审计（不 apply，只读代码）

自上向下检查变量穿透：声明层 tfvars → 控制层 variables.tf → 原子层 variable + resource

检查项：
- 声明层的每个字段在控制层是否有对应定义（optional 或 required）
- 控制层的每个字段在原子层是否被正确引用
- 字段类型是否一致（string/number/bool/list/map）
- required 字段是否都有赋值

### Step 2 — 全 false 空跑

tfvars: 所有 `create = false`

```
terraform apply → 确认空跑无报错
```

目的：验证控制层 → 原子层的 `count = local.create ? 1 : 0` 逻辑正确，无依赖报错。

### Step 3 — 主资源开启（整层 apply）

tfvars: 主资源 `create = true` / 子资源 `create = false`

```
terraform apply → 创建所有主资源
terraform apply → 双 apply 确认 0 drift
```

双 apply 规则：**每次 apply 后必须再 apply 一次**，检测 Provider 读取与 API 实际状态不一致导致的漂移。

### Step 4 — 子资源逐个开启（逐产品 apply）

每开一个产品的子资源：

```
修改 tfvars 开启该子资源 → terraform apply → 验证成功 → terraform apply → 确认 0 drift
```

逐产品顺序示例：
1. Tair `create_backup_policy` → apply × 2
2. Tair `create_account` → apply × 2
3. NAS `create_access_group` + `access_rules` → apply × 2
4. Kafka `sasl_users` → apply × 2
5. Kafka `sasl_acls` → apply × 2
6. MySQL ECS `hbr_backup_enabled` → apply × 2

### Step 5 — 关联性验证（不 apply，查 state）

`terraform state show` 验证资源间引用关系：

- 网络：VSwitch 引用 ID、跨 AZ 部署是否正确
- 安全组：`security_group_key` → 控制层解析为实际安全组 ID
- LB 后端：`ecs_key` / `ecs_keys` → ECS 实例 ID 绑定
- 连接地址：域名、端口生成规则

### Step 6 — 整层销毁测试

```
terraform destroy  # 自上而下：Web → Middleware → Database
```

验证资源能完整销毁、无依赖阻塞。

---

## 双 Apply 规则

> **每次 apply 后必须再 apply 一次，确认 0 change。**

原因：Provider 在 `read` 阶段可能返回与 API 实际状态不一致的值（computed drift），例如：
- Tair Account 代理模式下 `instance_id` 返回 `-db-{N}` 后缀
- NAS computed 字段在首次创建后值不同
- 某些资源 creation_time / status 字段仅在第二次 read 时才稳定

如果第二次 apply 出现 drift，需要用 `lifecycle { ignore_changes = [...] }` 修复。

---

## Apply 节奏总结

```
全 false 空跑 → 主资源全 true（apply×2）→ 逐产品子资源（各 apply×2）→ 无漂移收尾 → 整层 destroy
```

## 每层测试顺序

按层级自上而下：Web(06) → Middleware(05) → Database(04) → Security(02) → ACK(03) → Network(01)

销毁也按此顺序，从上层开始销毁避免依赖阻塞。

## 参考资料

- `test-drift-detection.md`（部署后二次 Plan 验证）
- `test-strategies.md`（测试金字塔）
- `test-policy-as-code.md`（策略即代码）

---

# 三层架构分离：原子层 / 控制层 / 声明层

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

清晰的三层分离是可维护基础设施即代码的基础。每层有明确的职责边界。职责混乱会导致隐藏依赖、代码难以维护、环境无法复现。

## 目录结构

```
modules/                          # 按云厂商组织（如 aws/、azure/、alicloud/）
├── atomic/          # 原子层：单资源封装
│   ├── vpc/
│   ├── vswitch/
│   ├── ecs/
│   ├── polardb/
│   ├── redis-community/
│   └── ... (30+ 模块)
├── control/         # 控制层：业务编排引擎
│   ├── network-topology/
│   ├── security/
│   ├── ack-cluster/
│   ├── database-cluster/
│   ├── middleware-cluster/
│   └── web-cluster/
└── declarative/     # 声明层：环境配置真相
    ├── simple/
    └── test33/
        ├── 00-shared/
        ├── 01-network/
        ├── 02-security/
        ├── 03-ack/
        ├── 04-database/
        ├── 05-middleware/
        └── 06-web/
```

## 层级职责

### 层级定义矩阵

| 层级 | 目录 | 职责 | 禁止事项 |
|------|------|------|----------|
| **原子层** | `atomic/<resource>/` | 单资源封装。暴露完整参数接口。一个模块 = 一种资源类型 | 禁止跨模块引用；禁止硬编码业务命名；禁止业务默认值 |
| **控制层** | `control/<scene>/` | 纯编排引擎。使用 for_each + locals 动态创建资源和 ID 映射。接收外部传入的 ID | 禁止硬编码业务值；禁止环境特定值；禁止创建共享资源（安全组/ACL） |
| **声明层** | `declarative/<env>/` | 环境配置的唯一真相来源。通过 tfvars 中的 map(object) 控制资源拓扑 | 禁止明文存储生产密钥；禁止直接调用原子模块 |

## 关键设计模式

### 声明层控制拓扑

```hcl
# declarative/test33/04-database/terraform.tfvars
# 声明你想要的 - 这一层控制一切

polardb_clusters = {
  "01" = {
    db_node_class = "polar.mysql.g2.xlarge"
    db_version    = "8.0"
    pay_type      = "PostPaid"
    vswitch_key   = "j-data"
    create        = true
  }
}

redis_instances = {
  "01" = {
    instance_class = "redis.shard.with.proxy.small.ce"
    shard_count    = 4
    capacity       = 4096
    engine_version = "7.0"
    vswitch_key    = "j-ecs"
    create         = true
  }
}
```

### 控制层动态编排

```hcl
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters  # 声明层控制数量

  # 必填输入
  vswitch_id    = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]
  db_node_class = each.value.db_node_class

  # 可选参数带默认值
  db_version = try(each.value.db_version, "8.0")
  pay_type   = try(each.value.pay_type, "PostPaid")

  # 自动命名
  cluster_description = coalesce(try(each.value.cluster_name, ""), "polar-${var.env}-${each.key}")

  # 安全配置
  security_ips       = try(each.value.security_ips, ["127.0.0.1"])  # API 默认值
  security_group_ids = can(each.value.security_group_key) ? 
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] : []

  tags = merge(local.common_tags, try(each.value.tags, {}))
}
```

### 原子层纯封装

```hcl
# atomic/polardb/main.tf

locals {
  create = var.create
}

resource "alicloud_polardb_cluster" "this" {
  count = local.create ? 1 : 0

  # 完整参数接口 - 必填字段无默认值
  db_node_class       = var.db_node_class
  vswitch_id          = var.vswitch_id
  zone_id             = var.zone_id
  db_version          = var.db_version

  # 可选字段 - 空默认值 = 让 API 决定
  pay_type            = var.pay_type != "" ? var.pay_type : null
  period              = var.period > 0 ? var.period : null
  cluster_description = var.cluster_description

  tags = merge({ "Name" = var.cluster_description }, var.tags,)
}
```

## 决策问题

**"这个配置属于哪一层？"**

| 问题 | 答案 | 所属层级 |
|------|------|----------|
| "这是什么资源，有哪些参数？" | 参数接口定义 | **原子层** |
| "这些资源如何组合，依赖顺序是什么？" | 编排逻辑 | **控制层** |
| "这个环境使用什么具体值？" | 具体赋值 | **声明层** |

**"新增资源时要改哪里？"**

| 场景 | 位置 |
|------|------|
| 新增 PolarDB 集群 | 编辑 04-database/terraform.tfvars 的 polardb_clusters map |
| 新增 Redis 实例 | 编辑 04-database/terraform.tfvars 的 redis_instances map |
| 新增 ECS 实例 | 编辑 06-web/terraform.tfvars 的 ecs_instances map |

> **所有场景都无需修改控制层代码。**

## 跨层数据流

```
declarative/test33/04-database/
    │
    │  tfvars: polardb_clusters = { "01" = {...} }
    │  tfvars: redis_instances = { "01" = {...} }
    ▼
control/database-cluster/
    │
    │  for_each = var.polardb_clusters
    │  for_each = var.redis_instances
    │  ID 映射: vswitch_ids_map["j-data"] → 实际vswitch_id
    ▼
atomic/polardb/
atomic/redis-community/
    │
    │  单资源创建
    ▼
alicloud_polardb_cluster.this
alicloud_kvstore_instance.this
```

## 检查清单

- [ ] 原子层：每个模块只封装一种资源类型
- [ ] 原子层：无跨模块引用
- [ ] 原子层：可选字段使用空默认值
- [ ] 控制层：所有动态资源使用 for_each + map
- [ ] 控制层：无硬编码业务值
- [ ] 控制层：通过 _map 变量实现 ID 映射
- [ ] 声明层：所有拓扑通过 tfvars 的 map(object) 控制
- [ ] 声明层：跨阶段引用使用 terraform_remote_state
- [ ] 新增/删除资源只需修改 tfvars

## 参考资料


---

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

---

# 控制层无业务默认值：技术默认 vs 业务默认

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

控制层是跨环境复用的编排引擎。如果控制层硬编码了业务默认值（如 `security_ips = ["127.0.0.1"]`），不同环境可能需要不同的默认值，导致控制层无法通用。区分"技术默认值"和"业务默认值"是保持控制层纯净的关键。

## 默认值分类

| 类型 | 定义 | 示例 | 允许层级 |
|------|------|------|----------|
| **技术默认值** | Terraform/Provider 运行所需的兜底值 | `try(v.xxx, "")`、`try(v.create, true)` | 控制层 ✅ |
| **业务默认值** | 与业务逻辑、安全策略、环境配置相关的值 | `security_ips = ["127.0.0.1"]`、`engine_version = "5.7"` | 仅声明层 ✅ |

## 问题：控制层包含业务默认值

```hcl
# ❌ 错误：控制层设置业务默认值
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # ❌ 业务默认值 — "127.0.0.1" 是业务决策，不是技术默认值
  security_ips = try(each.value.security_ips, ["127.0.0.1"])

  # ❌ 业务默认值 — "data" 是业务决策
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    [lookup(var.security_group_ids_map, "data", "")]
}
```

**问题：**
1. "127.0.0.1" 是 API 的行为模拟，但控制层不应该替业务做这个决定
2. "data" 安全组是业务选择，test33 环境可能用 "data"，simple 环境可能不同
3. 当 API 默认值变更时，控制层硬编码值会导致 state 漂移

## 正确：区分技术默认值和业务默认值

### 技术默认值（控制层可设）

```hcl
# ✅ 技术默认值：控制层可设置
# 这些值不涉及业务决策，只是 Terraform 语言层面的空值处理

# try() 空字符串回退 — 纯技术守卫
vswitch_id    = lookup(var.vswitch_ids_map, each.value.vswitch_key, "")
resource_group_id = var.resource_group_id != "" ? var.resource_group_id : null

# null 传递 — 让 API 决定默认行为
pay_type   = try(each.value.pay_type, null)
period     = try(each.value.period, null)

# 条件守卫 — plan-time 可解析的 create 判断
create = try(each.value.create, true) && lookup(local.xxx_would_create, each.value.xxx_key, false)
```

### 业务默认值（声明层必须显式指定）

```hcl
# ✅ 业务默认值：声明层 tfvars 中显式指定

# declarative/test33/04-database/terraform.tfvars
polardb_clusters = {
  "01" = {
    security_ips       = ["127.0.0.1"]    # 业务决定：测试环境用本地回环
    security_group_key = "data"            # 业务决定：使用 data 安全组
  }
}

# declarative/prod/04-database/terraform.tfvars
polardb_clusters = {
  "01" = {
    security_ips       = ["172.31.192.0/18"]  # 业务决定：生产用 VPC 网段
    security_group_key = "data"               # 业务决定：使用 data 安全组
  }
}
```

## 判断矩阵

| 值类型 | 示例 | 控制层可设？ | 说明 |
|--------|------|-------------|------|
| 空字符串 `""` | `try(v.xxx, "")` | ✅ | 技术守卫，非业务决策 |
| `null` | `try(v.xxx, null)` | ✅ | 让 API 决定，不干预 |
| API 默认值 | `["127.0.0.1"]` | ❌ | API 行为可能变更，导致漂移 |
| 业务选择 | `"data"`, `"ecs"` | ❌ | 不同环境可能不同 |
| 计算公式 | `coalesce(v.a, v.b)` | ✅ | 技术逻辑，非业务值 |

## 特殊情况：API 默认值防漂移

某些 API 参数（如 `security_ips`）有服务端默认值。如果 Terraform 不设置该参数，API 会使用默认值，但 Terraform state 中记录为空，导致 state 漂移。

```hcl
# 方案 A（推荐）：声明层显式指定，对齐 API 默认值
# tfvars 中：security_ips = ["127.0.0.1"]

# 方案 B：控制层传递 null，让 API 决定（接受首次 apply 后可能有 diff）
security_ips = try(each.value.security_ips, null)
```

**选择原则：** 优先方案 A（声明层显式指定），仅在参数过多且用户不关心默认值时使用方案 B。

## 检查清单

- [ ] 控制层无硬编码的业务默认值（IP 网段、安全组名、规格等）
- [ ] `try()` 回退值仅使用 `""`、`null`、`true`/`false`（技术默认值）
- [ ] 业务决策（如安全组选择、白名单 IP）在声明层 tfvars 显式配置
- [ ] 控制层注释中不包含业务特定的示例值（如 `cn-hangzhou`）

## 参考资料

- `variable-atomic-defaults.md`（原子层默认值原则）
- `control-parameter-passthrough.md`（控制层参数透传）
- `layer-separation.md`（三层职责划分）

---

# 原子层 for_each 边界：仅限同一主资源子组件

**优先级：** 高
**分类：** 三层架构

## 为什么重要

原子层的职责是"封装一种资源类型"。`for_each` 是编排逻辑，理论上属于控制层。但某些场景下，同一主资源有多个子组件（如 ALB 的多个 Rule），在原子层内使用 `for_each` 处理这些子组件是合理的。需要明确边界，防止 for_each 被滥用。

## 原子层 for_each 边界定义

### 允许：同一主资源的子组件

```hcl
# ✅ 允许：ALB Rule 是 ALB Listener 的子组件，与主资源生命周期绑定
# atomic/alb-rule/main.tf

resource "alicloud_alb_rule" "this" {
  count = local.create ? 1 : 0

  listener_id = var.listener_id    # 依赖父资源 ID
  rule_name   = var.rule_name
  # ... 单个 Rule 的配置
}
```

**判断标准：**
1. 子组件不能独立存在（必须有父资源 ID）
2. 子组件与父资源是 1:N 关系（一个 Listener 有多个 Rule）
3. 子组件的生命周期跟随父资源（父资源删除，子组件自动删除）

### 允许：同一资源的可变数量配置项

```hcl
# ✅ 允许：Security Group 的多条规则是同一资源的配置项
# atomic/security-group/main.tf

resource "alicloud_security_group_rule" "ingress" {
  count = local.create ? length(var.ingress_rules) : 0

  security_group_id = alicloud_security_group.this[0].id
  # ... 规则配置
}
```

### 禁止：跨资源的编排逻辑

```hcl
# ❌ 禁止：原子层不应包含多资源的编排逻辑
# atomic/web-stack/main.tf（假设的错误示例）

resource "alicloud_alb" "this" {
  count = local.create ? 1 : 0
}

resource "alicloud_alb_listener" "this" {
  count = local.create ? 1 : 0
  load_balancer_id = alicloud_alb.this[0].id
}

# ❌ 这应该在控制层用 for_each + module 调用来编排
```

## 决策矩阵

| 场景 | for_each 位置 | 说明 |
|------|---------------|------|
| ALB 有多个 Rule | **控制层** for_each 调用 `module alb-rule` | Rule 是独立模块，控制层编排数量 |
| Security Group 有多条规则 | **原子层** count 遍历规则列表 | 规则是同一资源的配置项 |
| SLB 有多个 Backend | **控制层** for_each 调用 `module slb-backend-attachment` | Backend 是独立资源 |
| ECS 有多块数据盘 | **原子层** dynamic block 或 count | 数据盘是 ECS 的子组件 |
| RDS 有多个数据库 | **控制层** for_each 调用 `module rds-database` | 数据库是独立资源 |

## 当前项目中的边界实践

### 符合边界的设计

```
atomic/alb/                 → 单个 ALB 实例（无 for_each）
atomic/alb-listener/        → 单个 Listener（无 for_each，控制层 for_each 编排）
atomic/alb-rule/            → 单个 Rule（无 for_each，控制层 for_each 编排）
atomic/alb-server-group/    → 单个 Server Group（无 for_each）
atomic/security-group/      → 单个 SG + count 遍历规则列表 ✅ 原子层 for_each 允许
atomic/ecs/                 → 单个 ECS + dynamic block 遍历数据盘 ✅ 允许
```

### 需要注意的设计

```
atomic/alb/main.tf:253      → ALB 的 load_balancer_id 转发（检查是否为同一资源子组件）
atomic/slb/main.tf:154-268  → SLB Server Group Attachment（已拆为独立模块 ✅）
```

## 拆分指南

当原子层模块包含多种资源类型的 `for_each` 时，应拆分为独立原子模块：

```
# 拆分前（违反边界）
atomic/alb/  → 包含 ALB + Listener + Rule + Server Group

# 拆分后（符合边界）
atomic/alb/                → 仅 ALB 实例
atomic/alb-listener/       → 仅 Listener
atomic/alb-rule/           → 仅 Rule
atomic/alb-server-group/   → 仅 Server Group
```

**控制层负责编排它们之间的关系：**

```hcl
# control/web-cluster/main.tf
module "alb" { for_each = var.albs ... }
module "alb_listener" { for_each = var.alb_listeners ... }
module "alb_rule" { for_each = var.alb_rules ... }
```

## 检查清单

- [ ] 原子层模块只封装一种主资源类型
- [ ] 原子层中的 `for_each`/`count` 仅用于同一主资源的子组件（如 SG 规则、ECS 数据盘）
- [ ] 跨资源的 `for_each` 编排在控制层完成
- [ ] 可独立存在/可独立删除的子资源已拆分为独立原子模块

## 参考资料

- `layer-separation.md`（三层职责划分）
- `control-nested-resource-flatten.md`（嵌套资源展平）
- `control-backup-prefilter.md`（条件子资源预过滤）

---

# 三层注释规范：原子层 / 控制层 / 声明层

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

统一的注释规范是代码可读性和可维护性的基础。三层架构各层职责不同，注释重点也应有所差异：
- **原子层**：关注 Provider Schema 完整性、参数约束说明
- **控制层**：关注职责域定义、设计原则
- **声明层**：关注参数联动表格、规格选择指南

---

## 一、原子层注释规范

### 1.1 文件头注释模板

```hcl
################################################################################
# {资源名称} — 原子层
# 资源类型：alicloud_{resource}_{type}
# 设计模式：create 开关 + locals 前置 + try 安全输出
#
# {产品体系说明（可选，复杂产品需要）}
#   - 实例类型 A：xxx
#   - 实例类型 B：xxx
#
# Provider Schema 参考（resource_alicloud_{resource}.go）:
#   ┌─ Required 参数 ────────────────────────────────────────────────────────
#   │ param_1   (Type)  - 参数说明
#   │ param_2   (Type)  - 参数说明
#   ├─ Optional 参数 ────────────────────────────────────────────────────────
#   │ param_3   (Type)  - 参数说明
#   │ param_4   (Type)  - 参数说明
#   ├─ Computed 属性 ────────────────────────────────────────────────────────
#   │ id        - 资源 ID
#   │ status    - 状态
#   └────────────────────────────────────────────────────────────────────────
#
# API 文档：https://help.aliyun.com/zh/{product}/developer-reference/api-{action}
################################################################################
```

### 1.2 资源块内分节注释

```hcl
resource "alicloud_instance" "this" {
  count = local.create ? 1 : 0

  # ——— 基础配置 ———
  name        = var.instance_name
  zone_id     = var.zone_id
  vswitch_id  = var.vswitch_id

  # ——— 网络配置 ———
  security_group_ids = var.security_group_ids
  vpc_id             = var.vpc_id

  # ——— 认证配置 ———
  password               = var.password != "" ? var.password : null
  kms_encrypted_password = var.kms_encrypted_password != "" ? var.kms_encrypted_password : null

  # ——— 计费配置 ———
  instance_charge_type = var.instance_charge_type
  period               = var.instance_charge_type == "PrePaid" ? var.period : null
}
```

### 1.3 原子层注释要点

| 元素 | 规范 | 示例 |
|------|------|------|
| 文件头 | Provider Schema 表格 + API 文档链接 | `resource_alicloud_polardb_cluster.go` |
| 分节注释 | `# ——— xxx 配置 ———` | `# ——— 认证配置 ———` |
| 行尾注释 | 仅关键参数、ForceNew 参数 | `period = var.period  # ForceNew` |
| 多子类型 | 用表格说明版本约束 | Redis 经典版/云原生版/倚天版表格 |

### 1.4 完整示例（Redis）

```hcl
#################################################################################
# Redis Community（开源版）
# 资源类型：alicloud_kvstore_instance
# 设计模式：create 开关 + locals 前置 + try 安全输出
#
# 部署类型（deploy_type）：
#   - classic：经典版（.default 后缀），仅支持 Redis 5.0
#   - cloud_native：云原生版（.ce 后缀），支持 Redis 5.0/6.0/7.0
#   - yitian：倚天版（.y.ee 后缀），ARM架构，支持 Redis 5.0/6.0/7.0
#
# 版本约束：
#   ┌───────────┬─────────────────┬──────────────────┬────────────────┐
#   │ 部署类型   │ 规格代码特征     │ 支持版本          │ CPU架构        │
#   ├───────────┼─────────────────┼──────────────────┼────────────────┤
#   │ classic   │ .default 后缀   │ 仅 5.0           │ X86           │
#   │ cloud_native│ .ce 后缀       │ 5.0 / 6.0 / 7.0  │ X86           │
#   │ yitian    │ .y.ee 后缀      │ 5.0 / 6.0 / 7.0  │ ARM（倚天）    │
#   └───────────┴─────────────────┴──────────────────┴────────────────┘
#################################################################################

locals {
  create = var.create
}

resource "alicloud_kvstore_instance" "this" {
  count = local.create ? 1 : 0

  # ——— 基础配置 ———
  instance_name  = var.instance_name
  instance_class = var.instance_class
  engine_version = var.engine_version

  # ——— 网络配置 ———
  vswitch_id = var.vswitch_id
  zone_id    = var.zone_id != "" ? var.zone_id : null
}
```

---

## 二、控制层注释规范

### 2.1 文件头注释模板

```hcl
################################################################################
# {模块名} — {功能描述} 控制层
# 职责域（三问法：{问题}？→ {模块名}）：
#   ├─ {资源类型 A}    {功能描述}（{变量名} map）
#   ├─ {资源类型 B}    {功能描述}（{变量名} map）
#   └─ {资源类型 C}    {功能描述}（{变量名} map）
#
# 设计原则：
#   - 声明层通过 map 完全控制拓扑
#   - 控制层只做 for_each 编排，拒绝硬编码默认值
#
# 安全编码规范：
#   - ID 映射过滤 null 值
#   - Map Key 访问使用 try() 保护
################################################################################
```

### 2.2 控制层注释要点

| 元素 | 规范 | 说明 |
|------|------|------|
| 职责域 | 树形结构 `├─ └─` | 清晰展示资源分类 |
| 设计原则 | 2-3 条精简 | 避免冗长描述 |
| 安全规范 | 2-3 条 | ID 映射 + try 保护 |
| 模块调用前 | 分类标识 + 命名规范 | `[Database] polar-{env}-{key}` |

### 2.3 完整示例

```hcl
################################################################################
# security — 安全组 + ACL + 快照策略 控制层
# 职责域（三问法：安全边界在哪里？→ security）：
#   ├─ Security Groups    安全组（security_groups map）
#   ├─ SLB ACL            负载均衡访问控制（slb_acls map）
#   ├─ ALB ACL            应用负载均衡访问控制（alb_acls map）
#   └─ ECS Snapshot Policy 自动快照策略（ecs_auto_snapshot_policies map）
#
# 设计原则：
#   - 声明层通过 map 完全控制拓扑
#   - 控制层只做 for_each 编排，拒绝硬编码默认值
#
# 安全编码规范：
#   - ID 映射过滤 null 值
#   - Map Key 访问使用 try() 保护
################################################################################

locals {
  common_tags = merge(
    { "ops-env" = var.env, "project" = var.project },
    var.tags,
  )
}

################################################################################
# [Security] Security Groups — sg-{env}-{key}
################################################################################

module "security_groups" {
  source   = "../../atomic/security-group"
  for_each = var.security_groups

  create = try(each.value.create, true)
  # ...
}
```

---

## 三、声明层注释规范

### 3.1 文件头注释模板

```hcl
################################################################################
# {环境名} — {阶段名} 环境参数
# 公共变量来自 ../00-shared/shared.tfvars
# 执行方式：terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
#
# 设计说明：
#   - 每类资源是一个 map，key = 逻辑名（自动生成命名）
#   - 增删实例：只需增删 map entry，无需改控制层代码
#   - 安全组由 02-security 层管理，此处指定 key
################################################################################
```

### 3.2 资源块前注释模板

```hcl
################################################################################
# [{分类}] {资源名} — {命名规范}
# {产品说明/导入实例/注意事项}
#
# ┌─────────────────────────────────────────────────────────────────────────────┐
# │              {参数联动表格标题}                                              │
# ├─────────────────────┬────────────┬──────────────────────────────────────────┤
# │ {参数1}              │ {参数2}     │ 说明                                     │
# ├─────────────────────┼────────────┼──────────────────────────────────────────┤
# │ value_1             │ value_a    │ 说明 A                                   │
# │ value_2             │ value_b    │ 说明 B                                   │
# └─────────────────────┴────────────┴──────────────────────────────────────────┘
################################################################################
```

### 3.3 实例内分节注释

```hcl
"01" = {
  # ——— 必填参数 ———
  db_node_class = "polar.mysql.g1.tiny.c"  # .c后缀=标准版，1核1G入门型
  db_version    = "8.0"                    # MySQL 8.0
  pay_type      = "PostPaid"               # 按量付费
  vswitch_key   = "j-data"                 # 部署在数据层子网

  # ——— 可选参数 ———
  db_node_count       = 2                  # 默认1主1备高可用
  security_group_key  = "data"             # 数据层安全组

  # ——— 开关配置 ———
  create = false  # 已测试通过，暂时禁用
}
```

### 3.4 声明层注释要点

| 元素 | 规范 | 说明 |
|------|------|------|
| 文件头 | 环境标识 + 执行方式 | 明确上下文 |
| 资源块前 | 分类标识 + 命名规范 + 联动表格 | 帮助选择参数 |
| 实例内 | 分节注释 + 行尾注释 | 每个参数有说明 |
| create 参数 | 行尾注释状态 | `create = false  # 已测试通过` |

---

## 四、三层注释对比矩阵

| 维度 | 原子层 | 控制层 | 声明层 |
|------|--------|--------|--------|
| **文件头重点** | Provider Schema + API 文档 | 职责域树 + 设计原则 | 环境标识 + 执行方式 |
| **分节注释** | `# ——— xxx 配置 ———` | 模块调用前分类标识 | `# ——— xxx 参数 ———` |
| **行尾注释** | ForceNew / 关键约束 | 命名规范 | 每个参数说明 |
| **表格用途** | 版本约束 / 规格后缀规则 | 无 | 参数联动 / 规格选择 |
| **文档链接** | Provider 源码 + API 文档 | 无 | 无 |
| **长度控制** | 20-40 行（含 Schema 表格） | 10-15 行 | 15-50 行（含产品表格） |

---

## 五、禁止事项

### 5.1 原子层禁止

```hcl
# ❌ 禁止：无 Provider Schema 参考
################################################################################
# ECS Instance
################################################################################

# ❌ 禁止：无 API 文档链接
# ❌ 禁止：无版本约束表格（复杂产品）
```

### 5.2 控制层禁止

```hcl
# ❌ 禁止：冗长设计原则描述
# 设计原则（灵魂文档原则）：
#   - 原子层 = 组件：单一职责，可复用
#   - 控制层 = 捆绑的一组：只保留编排逻辑，拒绝硬编码
#   - 声明层 = 完全控制：通过 map 控制资源拓扑，增删资源无需改控制层
#   - 控制层直接使用 for_each，移除原子层 wrapper 依赖
#   - 条件逻辑在 locals 中显式定义

# ✅ 正确：精简 2-3 条
# 设计原则：
#   - 声明层通过 map 完全控制拓扑
#   - 控制层只做 for_each 编排，拒绝硬编码默认值
```

### 5.3 声明层禁止

```hcl
# ❌ 禁止：参数无注释
"01" = {
  db_node_class = "polar.mysql.g1.tiny.c"
  db_version    = "8.0"
  pay_type      = "PostPaid"
}

# ✅ 正确：每个参数有说明
"01" = {
  db_node_class = "polar.mysql.g1.tiny.c"  # .c后缀=标准版，1核1G入门型
  db_version    = "8.0"                    # MySQL 8.0
  pay_type      = "PostPaid"               # 按量付费
}
```

---

---

## 六、版本一致性规范

### 6.1 原子层 versions.tf 统一格式

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    alicloud = { source = "aliyun/alicloud", version = "1.273.0" }
  }
}
```

### 6.2 版本一致性检查清单

| 检查项 | 统一值 | 说明 |
|--------|--------|------|
| `required_version` | `">= 1.3.0"` | Terraform 最低版本 |
| `alicloud.version` | `"1.273.0"` | Provider 固定版本 |
| 格式 | 单行紧凑 | `{ source = "xxx", version = "x.x.x" }` |

### 6.3 禁止事项

```hcl
# ❌ 禁止：使用范围版本
version = ">= 1.200.0"

# ❌ 禁止：多行展开格式
alicloud = {
  source  = "aliyun/alicloud"
  version = "1.273.0"
}

# ✅ 正确：固定版本 + 单行格式
alicloud = { source = "aliyun/alicloud", version = "1.273.0" }
```

---

## 检查清单

### 原子层
- [ ] 文件头包含 Provider Schema 参考表格
- [ ] 文件头包含 API 文档链接
- [ ] 复杂产品包含版本约束/规格后缀表格
- [ ] 资源块使用分节注释 `# ——— xxx 配置 ———`
- [ ] ForceNew 参数有行尾注释

### 控制层
- [ ] 文件头包含职责域树形结构
- [ ] 设计原则精简（2-3 条）
- [ ] 包含安全编码规范
- [ ] 模块调用前有分类标识和命名规范

### 声明层
- [ ] 文件头包含环境标识和执行方式
- [ ] 资源块前包含参数联动表格（如适用）
- [ ] 实例内参数有行尾注释
- [ ] create 参数有状态注释

---

## 参考资料

- API 文档格式：`https://help.aliyun.com/zh/{product}/developer-reference/api-{action}`
- 三层架构规范：`layer-separation.md`
---

# 层级 ID 映射：基于 Key 的资源引用

**优先级：** 关键
**分类：** 三层架构

## 为什么重要

控制层模块需要引用其他层创建的资源（VSwitch、安全组）。使用原始 ID 会造成紧耦合，导致环境无法复现。基于 Key 的 ID 映射将逻辑名与物理 ID 解耦。

## 问题：硬编码 ID 引用

```hcl
# ❌ 错误：控制层硬编码 ID
module "polardb" {
  source = "../../atomic/polardb"

  vswitch_id = "vsw-bp1ilacjy9xed5e9vygeb"  # 硬编码！
  security_group_ids = ["sg-bp123456"]        # 硬编码！
}
```

**问题：**
1. 无法复现到新环境（ID 不存在）
2. 无法更换 VSwitch，必须修改代码
3. 违反"控制层无硬编码值"原则

## 解决方案：基于 Key 的 ID 映射

### 模式 1：Map 变量用于 ID 查找

```hcl
# control/database-cluster/variables.tf

variable "vswitch_ids_map" {
  description = "VSwitch ID 映射。key = 逻辑名(j-ecs/k-ecs/j-data), value = 实际 ID"
  type        = map(string)
}

variable "security_group_ids_map" {
  description = "安全组 ID 映射。key = 逻辑名(ecs/pod/data), value = 实际 ID"
  type        = map(string)
  default     = {}
}

variable "az_map" {
  description = "可用区 ID 映射。key = 逻辑名(j/k/l), value = 实际可用区 ID(cn-hangzhou-j)"
  type        = map(string)
  default     = {}
}
```

### 模式 2：声明层控制 Key 选择

```hcl
# declarative/test33/04-database/terraform.tfvars

polardb_clusters = {
  "01" = {
    db_node_class = "polar.mysql.g2.xlarge"
    vswitch_key   = "j-data"           # 逻辑 key，非实际 ID
    security_group_key = "data"        # 逻辑 key，非实际 ID
  }
}

redis_instances = {
  "01" = {
    instance_class     = "redis.shard.with.proxy.small.ce"
    vswitch_key        = "j-ecs"       # 不同的 VSwitch
    security_group_key = "data"        # 相同的安全组
    zone_key           = "j"           # 主可用区（逻辑 key）
    secondary_zone_key = "k"           # 备可用区（跨 AZ 高可用）
  }
}
```

### 模式 3：控制层将 Key 映射为 ID

```hcl
# control/database-cluster/main.tf

module "polardb" {
  source   = "../../atomic/polardb"
  for_each = var.polardb_clusters

  # Key → ID 映射
  vswitch_id = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]

  # 可用区 Key → ID 映射
  # ⚠️ 注意：can() 对空字符串返回 true，需用 try(..., "") != "" 判断
  zone_id           = try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "")
  secondary_zone_id = try(each.value.secondary_zone_key, "") != "" ? var.az_map[each.value.secondary_zone_key] : try(each.value.secondary_zone_id, "")
  hidden_zone_id    = try(each.value.hidden_zone_key, "") != "" ? var.az_map[each.value.hidden_zone_key] : try(each.value.hidden_zone_id, "")

  # 安全组：控制层不设业务默认值
  # ⚠️ 原则：控制层是共用模块，不应硬编码业务值（如 "data"）
  # 业务决策应在声明层显式配置 security_group_key
  security_group_ids = try(each.value.security_group_key, "") != "" ?
    [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
    []  # 空值，声明层必须显式配置
}
```

### 模式 4：通过 Remote State 实现跨阶段引用

```hcl
# declarative/test33/04-database/main.tf

data "terraform_remote_state" "network" {
  backend = "local"
  config = { path = "../01-network/terraform.tfstate" }
}

data "terraform_remote_state" "security" {
  backend = "local"
  config = { path = "../02-security/terraform.tfstate" }
}

module "db" {
  source = "../../../control/database-cluster"

  # 从上游阶段传递完整的 map
  vswitch_ids_map        = data.terraform_remote_state.network.outputs.vswitch_ids_map
  security_group_ids_map = data.terraform_remote_state.security.outputs.security_group_ids_map

  # 可用区 map 来自 shared.tfvars（跨环境共享）
  az_map = var.az_map
}
```

### 模式 5：az_map 定义在 shared.tfvars（跨环境共享）

```hcl
# declarative/test33/00-shared/shared.tfvars

# 可用区映射 — key 为逻辑名，value 为实际可用区 ID
# 跨 region 迁移只需修改此 map，声明层所有 zone_key 保持不变
az_map = {
  j = "cn-hangzhou-j"     # 主可用区
  k = "cn-hangzhou-k"     # 备可用区
}
```

**跨 Region 迁移优势：**

| Region | az_map 定义 | zone_key 不变 |
|--------|-------------|---------------|
| 杭州 | `{j = "cn-hangzhou-j", k = "cn-hangzhou-k"}` | `zone_key = "j"` |
| 北京 | `{j = "cn-beijing-a", k = "cn-beijing-b"}` | `zone_key = "j"` |
| 上海 | `{j = "cn-shanghai-a", k = "cn-shanghai-b"}` | `zone_key = "j"` |
```

## ID Map 输出模式

每个声明阶段输出完整的 ID map：

```hcl
# declarative/test33/01-network/outputs.tf

output "vswitch_ids_map" {
  description = "VSwitch ID 映射（key = 逻辑名）"
  value = {
    "j-ecs"  = module.network.vswitch_ids["j-ecs"]
    "k-ecs"  = module.network.vswitch_ids["k-ecs"]
    "j-data" = module.network.vswitch_ids["j-data"]
    "j-pod"  = module.network.vswitch_ids["j-pod"]
    "k-pod"  = module.network.vswitch_ids["k-pod"]
  }
}

# declarative/test33/02-security/outputs.tf

output "security_group_ids_map" {
  description = "安全组 ID 映射（key = 逻辑名）"
  value = {
    "ecs"  = module.security.sg_ecs_id
    "pod"  = module.security.sg_pod_id
    "data" = module.security.sg_data_id
  }
}
```

## Key 命名规范

| 资源类型 | Key 模式 | 示例 |
|----------|----------|------|
| 可用区 | `{az_letter}` | j, k, l, m |
| VSwitch | {az}-{purpose} | j-ecs, k-ecs, j-data, j-pod |
| 安全组 | {purpose} | ecs, pod, data |
| ALB ACL | {purpose} | nginx, ack-ingress |
| SLB ACL | {purpose} | waf, internal |

## 决策矩阵

| 场景 | 使用方式 |
|------|----------|
| 控制层需要 VSwitch ID | var.vswitch_ids_map[each.value.vswitch_key] |
| 控制层需要安全组 ID | lookup(var.security_group_ids_map, each.value.security_group_key, "") |
| 控制层需要可用区 ID | try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "") |
| 声明阶段传递 ID 给控制层 | data.terraform_remote_state.xxx.outputs.xxx_map |
| 声明 tfvars 指定资源 | 使用 vswitch_key、security_group_key、zone_key（非实际 ID） |

## 控制层无业务默认值原则

**原则：控制层是共用模块，不应硬编码业务默认值。**

| 层级 | 职责 | 业务默认值 |
|------|------|----------|
| 原子层 | 技术实现 | ❌ 无 |
| 控制层 | 编排逻辑 | ❌ **不应有**（共用模块）|
| 声明层 | 业务配置 | ✅ 显式配置 |

```hcl
# ❌ 错误：控制层硬编码业务默认值
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  compact([lookup(var.security_group_ids_map, "data", "")])  # 硬编码 "data"！

# ✅ 正确：控制层不设默认值，声明层必须显式配置
security_group_ids = try(each.value.security_group_key, "") != "" ?
  [lookup(var.security_group_ids_map, each.value.security_group_key, "")] :
  []  # 空值，强制声明层配置

# 声明层（terraform.tfvars）— 业务决策在这里
security_group_key = "data"  # 显式配置，非隐式默认
```

**原因：**
1. 控制层可能被多个声明层复用（test33/test34/demo）
2. 不同环境的安全组策略可能不同
3. 显式配置更安全，避免隐藏默认值导致的意外

## 常见陷阱

### 陷阱：can() 函数对空字符串返回 true

```hcl
# ❌ 错误写法：can() 对空字符串返回 true，导致用 "" 作为 key 查找 map
zone_id = can(each.value.zone_key) ? var.az_map[each.value.zone_key] : "default"
# 当 zone_key = "" 时，can() 返回 true，然后 az_map[""] 报 Invalid index

# ✅ 正确写法：显式检查非空
zone_id = try(each.value.zone_key, "") != "" ? var.az_map[each.value.zone_key] : try(each.value.zone_id, "")
```

### 陷阱：声明层直接使用 zone_id

```hcl
# ❌ 错误：声明层硬编码可用区 ID，无法统一管理
zone_id           = "cn-hangzhou-j"
secondary_zone_id = "cn-hangzhou-k"

# ✅ 正确：使用 zone_key 通过 az_map 统一入口
zone_key           = "j"    # 由 az_map 映射到 cn-hangzhou-j
secondary_zone_key = "k"    # 由 az_map 映射到 cn-hangzhou-k
```

**优势：**
- 跨 Region 迁移只需修改 az_map，声明层无需改动
- 统一入口，便于管理和审计
- 避免因不同产品 zone_id 格式差异导致的问题

## 检查清单

- [ ] 控制层变量使用 map(string) 接收 ID 输入（vswitch_ids_map, security_group_ids_map, az_map）
- [ ] 声明 tfvars 使用 _key 字段（逻辑名：vswitch_key, security_group_key, zone_key）
- [ ] 控制层通过 map 查找将 _key 映射为 _id
- [ ] 跨阶段引用使用 terraform_remote_state
- [ ] 每个阶段输出完整的 ID map 供下游使用
- [ ] 控制层不设业务默认值（如安全组 "data"），声明层必须显式配置
- [ ] 控制层和声明层无硬编码资源 ID
- [ ] az_map 定义在 shared.tfvars（跨环境共享）
- [ ] zone_key 使用单字母逻辑名（j/k/l/m）

## 参考资料


---

# 资源归属：三问法

**优先级：** 高
**分类：** 三层架构

## 为什么重要

构建控制层模块时，资源必须按**功能职责**组织，而非按资源类型。ECS 实例和托管服务属于不同模块，依据是它们**做什么**，而非它们**是什么**。归属错误会导致依赖混乱和安全边界不清。

## 三问法决策树

确定资源属于哪个控制模块时，问：

| 问题 | 如果是 | 归属模块 | 安全组 |
|------|--------|----------|--------|
| **1. 是否存储数据？** | MySQL/Kafka/ES/PolarDB/Redis | database-cluster | sg-data（自建）或 security_ips（托管） |
| **2. 是否为平台级基础设施？** | SkyWalking/ilogtail/MSE | middleware-cluster | sg-ecs |
| **3. 是否处理业务流量？** | Nginx/Jenkins/业务应用 | web-cluster | sg-ecs |

## 控制模块归属

```
control/
├── database-cluster/     # 问题 1：存储数据
│   ├── PolarDB          # 托管数据库
│   ├── Redis            # 托管缓存
│   ├── Kafka            # 托管消息（数据持久化）
│   ├── ES               # 托管搜索（索引数据）
│   └── MySQL ECS        # 自建数据库
│
├── middleware-cluster/   # 问题 2：平台基础设施
│   ├── MSE              # 服务注册中心
│   ├── NAS              # 共享存储
│   ├── OSS              # 对象存储
│   ├── SkyWalking ECS   # 可观测平台
│   └── ilogtail ECS     # 日志采集平台
│
└── web-cluster/          # 问题 3：业务流量
    ├── Nginx ECS        # 反向代理
    ├── Jenkins ECS      # CI/CD
    ├── App ECS×N        # 业务应用
    ├── ALB×N            # 七层负载均衡
    ├── SLB×N            # 四层负载均衡
    └── NLB×N            # 网络负载均衡
```

## 关键说明：托管服务与安全组

**托管服务（PolarDB/Redis/Kafka/ES）没有 ENI，无法"加入"安全组。**

security_group_ids 参数的含义是"自动将此安全组的成员 IP 添加到 security_ips 白名单"，而非防火墙。

```hcl
# ❌ 错误理解：托管服务加入安全组
# "PolarDB 加入 sg-test-data 安全组"

# ✅ 正确理解：托管服务使用 IP 白名单
# PolarDB 使用 security_ips 白名单
# security_group_ids 用于自动从安全组成员填充 security_ips
```

## 安全组归属

| 安全组 | 创建方 | 使用方 |
|--------|--------|--------|
| sg-ecs | security 层 | 所有包含 ECS 的模块 |
| sg-pod | security 层 | 仅 ack-cluster |
| sg-data | security 层 | database-cluster（自建 MySQL） |
| SLB ACL×N | web-cluster | web-cluster（生命周期与 SLB 对齐） |
| ALB ACL×1 | security 层 | ack-cluster |

## 决策示例

```hcl
# 问题 1：是否存储数据？
# MySQL ECS → database-cluster
# 创建 sg-data 保护自建 MySQL

mysql_ecs_instances = {
  "01" = {
    instance_type = "ecs.g5ne.xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：sg-data + sg-ecs
  }
}

# 问题 2：是否为平台基础设施？
# SkyWalking ECS → middleware-cluster
# 仅使用 sg-ecs（无需数据保护）

skywalking_ecs_instances = {
  "oap-01" = {
    instance_type = "ecs.g6a.2xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：仅 sg-ecs
  }
}

# 问题 3：是否处理业务流量？
# Nginx ECS → web-cluster
# 仅使用 sg-ecs（前端层）

ecs_instances = {
  "nginx-01" = {
    instance_type = "ecs.sn1ne.xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：仅 sg-ecs
  }
}
```

## 反模式：按资源类型分组

```hcl
# ❌ 错误：ECS 按资源类型分组
control/
├── ecs-cluster/         # 所有 ECS 放在一个模块
│   ├── MySQL ECS
│   ├── SkyWalking ECS
│   ├── Nginx ECS
│   └── App ECS
└── load-balancers/      # 所有 LB 放在一个模块
    ├── ALB
    ├── SLB
    └── NLB
```

**问题：** 业务逻辑分散、归属不清、依赖混乱。

## 检查清单

- [ ] 所有 ECS 按功能职责分配到模块
- [ ] 自建数据库使用 sg-data（在 security 层创建）
- [ ] 托管服务接收 sg_ecs_id 用于白名单填充
- [ ] 无"所有 ECS 放一个模块"反模式
- [ ] 安全组归属已记录
- [ ] DAG 依赖关系与三问法对齐

## 参考资料


---

# 声明层配置模式

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

声明层（terraform.tfvars）是 IaC 的"用户界面"。组织良好的配置可以提升：
- **可读性**：清晰结构，易于理解
- **可维护性**：一致模式，易于修改
- **完整性**：所有参数有文档，无隐藏缺口

## 配置结构模式

### 标准分区顺序（按优先级）

```
1. 必填参数 - Required, no defaults
2. 架构配置 - Architecture-specific (sub-type dependent)
3. 版本配置 - Version related
4. 计费配置 - Billing and payment
5. 可用区配置 - Zone and region
6. 安全配置 - Security (ips, groups, encryption)
7. 认证配置 - Authentication
8. 运维配置 - Operations and maintenance
9. 备份配置 - Backup and recovery
10. 高级配置 - Advanced/low-frequency
```

### 示例：Kafka 多类型配置

```hcl
kafka_instances = {
  #############################################################################
  # 实例 01：Serverless 版（推荐）- 弹性伸缩，按量计费
  #############################################################################
  "01" = {
    # ——— 必填参数 ———
    vswitch_key = "j-data"
    create = true
    instance_type = "alikafka_serverless"
    deploy_type = 5

    # ——— Serverless 配置 ———
    serverless_config = {
      reserved_publish_capacity = 60
      reserved_subscribe_capacity = 60
    }

    # ——— 版本配置 ———
    service_version = "" # Serverless 默认 3.3.1

    # ——— 计费配置 ———
    paid_type = "PostPaid"

    # ——— 可用区配置 ———
    zone_id = "" # 留空由系统自动选择

    # ——— 功能配置 ———
    enable_auto_group = false
    enable_auto_topic = "disable"
    default_topic_partition_num = 12

    # ——— 高级配置 ———
    config = ""
    kms_key_id = ""
    resource_group_id = ""
  }
}
```

## 产品文档模式

### 使用表格展示复杂信息

```hcl
# ═══════════════════════════════════════════════════════════════════════════════
# 一、Kafka 实例类型（instance_type）分类
# ═══════════════════════════════════════════════════════════════════════════════
# ┌─────────────────────────┬──────────────────────────────────────────────────┐
# │ 实例类型 │ 特点 │
# ├─────────────────────────┼──────────────────────────────────────────────────┤
# │ alikafka（传统版） │ 固定规格，需配置 disk_type/disk_size/io_max │
# │ │ 版本：2.2.0 / 2.6.2 │
# ├─────────────────────────┼──────────────────────────────────────────────────┤
# │ alikafka_serverless │ 弹性伸缩，按量计费（推荐） │
# │ （Serverless 版） │ 版本：3.3.1（默认） │
# └─────────────────────────┴──────────────────────────────────────────────────┘
```

### 使用表格展示参数约束

```hcl
# ┌─────────┬───────────────────┬──────────────────┐
# │ io_max │ partition_num 范围 │ disk_size 范围 │
# ├─────────┼───────────────────┼──────────────────┤
# │ 20 │ 50-450 │ 500-6100 GB │
# │ 30 │ 50-450 │ 800-6100 GB │
# │ 60 │ 80-450 │ 1400-6100 GB │
# └─────────┴───────────────────┴──────────────────┘
```

## 参数注释模式

### ForceNew 参数

```hcl
disk_type = 1 # 磁盘类型：0=高效云盘 / 1=SSD 云盘（ForceNew）
zone_id = "" # 可用区 ID（ForceNew）
```

### 枚举值

```hcl
enable_auto_topic = "disable" # enable=开启 / disable=关闭
paid_type = "PostPaid" # PostPaid=按量付费 / PrePaid=包年包月
```

### 范围约束

```hcl
partition_num = 50 # 分区数：50-450（与 io_max 联动）
disk_size = 500 # 磁盘容量 GB：500-6100
```

### 有默认值的可选参数

```hcl
zone_id = "" # 留空由系统自动选择
config = "" # 留空使用默认配置
```

## 多类型资源模式

当资源有多个子类型且参数不同时：

### 模式：只填写相关参数

```hcl
# Serverless 版 - 只填 serverless_config
"01" = {
  instance_type = "alikafka_serverless"
  serverless_config = { ... }
  # 不填 disk_type, disk_size - 与 Serverless 无关
}

# 传统版 - 填磁盘和分区参数
"02" = {
  instance_type = "alikafka"
  disk_type = 1
  disk_size = 500
  partition_num = 50
  # 不填 serverless_config
}

# Confluent 版 - 填 confluent_config
"03" = {
  instance_type = "alikafka_confluent"
  confluent_config = { ... }
  password = "xxx"
  # 不填 serverless_config, disk_type
}
```

## 嵌套对象配置

### serverless_config 模式

```hcl
serverless_config = {
  reserved_publish_capacity = 60 # 发布容量 CU（60-600，步长60）
  reserved_subscribe_capacity = 60 # 订阅容量 CU（60-600，步长60）
}
```

### confluent_config 模式

```hcl
confluent_config = {
  # ——— Kafka 集群（必填）———
  kafka_cu = 2
  kafka_storage = 500
  kafka_replica = 3

  # ——— Zookeeper（必填）———
  zookeeper_cu = 2
  zookeeper_storage = 20
  zookeeper_replica = 3 # ForceNew

  # ——— 可选组件 ———
  control_center_cu = 0
  schema_registry_cu = 0
  connect_cu = 0
  ksql_cu = 0
}
```

## Key Rules Summary

| Rule | Description |
|------|-------------|
| **Rule 1** | 使用标准化的区块顺序排列参数 |
| **Rule 2** | 使用表格展示产品说明和参数约束 |
| **Rule 3** | ForceNew 参数必须标注 `(ForceNew)` |
| **Rule 4** | 枚举类型参数标注所有可选值 |
| **Rule 5** | 范围约束参数标注取值范围 |
| **Rule 6** | 多子类型资源只填相关参数 |
| **Rule 7** | 使用中文分隔符 `———` 区分参数分组 |
| **Rule 8** | 每个实例使用 `####` 分隔符和标题注释 |
| **Rule 9** | **禁止使用 Terraform 函数调用** |

## terraform.tfvars 禁止函数调用

### 为什么禁止？

`terraform.tfvars` 文件只支持**静态值**，不支持 Terraform 函数。这是 Terraform 的设计限制。

### 错误示例

```hcl
# terraform.tfvars 中使用函数 - 报错！
config = jsonencode({ "enable.acl" = "true" }) # Error: Function calls not allowed

# 其他函数也不支持
instance_name = format("kafka-%s", var.env) # Error
tags = merge({a="1"}, {b="2"}) # Error
```

**错误信息**：
```
│ Error: Function calls not allowed
│ on terraform.tfvars line 822:
│ 822: config = jsonencode({ "enable.acl" = "true" })
│ Functions may not be called here.
```

### 正确写法

```hcl
# 直接写 JSON 字符串
config = "{\"enable.acl\":\"true\"}"

# 使用控制层或原子层的 locals 处理逻辑
# 声明层只提供原始值，逻辑处理在控制层
config = "" # 控制层有默认逻辑
```

### 函数处理位置对照表

| 需要的函数 | 正确处理位置 | 说明 |
|------------|--------------|------|
| `jsonencode()` | 直接写 JSON 字符串 | tfvars 只能写静态值 |
| `format()` | 控制层 locals | 自动命名逻辑 |
| `merge()` | 控制层 locals | tags 合并 |
| `coalesce()` | 控制层 main.tf | 默认值处理 |
| `try()` | 控制层 main.tf | 安全取值 |

### JSON 字符串转义规则

```hcl
# 单层 JSON - 使用 \" 转义
config = "{\"enable.acl\":\"true\"}"

# 多层 JSON - 注意嵌套转义
config = "{\"k1\":\"v1\",\"k2\":{\"nested\":\"value\"}}"
```

## 注释风格指南

```hcl
# ——— 必填参数 ———
# ——— 架构配置 ———
# ——— 版本配置 ———
# ——— 计费配置 ———
# ——— 可用区配置 ———
# ——— 安全配置 ———
# ——— 认证配置 ———
# ——— 运维配置 ———
# ——— 备份配置 ———
# ——— 高级配置 ———
```

## Provider 规格代码提取流程

### 从 Provider 源码提取有效值

当不确定参数的有效取值时，必须查看 Provider 源码中的 `ValidateFunc`：

```go
// 位置：terraform-provider-alicloud/alicloud/resource_alicloud_{resource}.go
"cluster_specification": {
  Type: schema.TypeString,
  Required: true,
  ValidateFunc: StringInSlice([]string{"MSE_SC_1_2_60_c", "MSE_SC_2_4_60_c", ...}, false),
},
```

**提取步骤**：
1. 定位资源文件：`resource_alicloud_{resource_name}.go`
2. 查找 `Schema: map[string]*schema.Schema{}` 块
3. 提取 `ValidateFunc: StringInSlice([...], false)` 中的有效值列表
4. 将有效值整理到声明层注释表格中

### 规格代码命名规律

| 产品 | 规格命名模式 | 示例 |
|------|-------------|------|
| MSE | `MSE_SC_{CPU}_{内存}_{连接数}_c` | `MSE_SC_1_2_60_c` = 1核2GB 60连接 |
| Redis | `redis.{架构}.{size}.{部署类型}` | `redis.shard.small.ce` = 云原生标准版 |
| PolarDB | `polar.mysql.{规格}.{后缀}` | `polar.mysql.g1.tiny.c` = 标准版 1核1G |
| NAS | 不使用规格代码 | 使用 `storage_type` + `protocol_type` 组合 |
| OSS | 不使用规格代码 | 使用 `storage_class` + `redundancy_type` 组合 |

## 最小规格实例化模板

### 原则

为每个子类型生成**最小规格**实例化代码，便于测试验证：

```hcl
# 最小规格选择原则：
# 1. 最小 CPU/内存
# 2. 最小存储/连接数
# 3. 开发版/测试版而非生产版
# 4. 单节点而非高可用（测试环境）
```

### 模板示例

```hcl
product_instances = {
  ##############################################################################
  # 实例01：{子类型名称} — 最小规格
  ##############################################################################
  "01" = {
    # ——— 必填参数 ———
    instance_type = "{最小规格代码}" # 最小规格：说明
    vswitch_key = "j-ecs" # 部署子网
    create = true # 开启测试

    # ——— 架构配置 ———
    # ... 最小配置 ...

    # ——— 高级配置 ———
    # ... 可选配置，留空或最小值 ...
  }

  ##############################################################################
  # 实例02：{另一子类型} — 最小规格（禁用）
  ##############################################################################
  "02" = {
    # ——— 必填参数 ———
    instance_type = "{最小规格代码}"
    vswitch_key = "j-ecs"
    create = false # 禁用，需要时开启

    # ... 该子类型特有参数 ...
  }
}
```

## 产品规格对照表标准格式

### 规格代码表格

```hcl
# ┌──────────────────┬─────────┬──────────┬────────────────────────────────┐
# │ 规格代码 │ CPU/内存 │ 连接数 │ 适用场景 │
# ├──────────────────┼─────────┼──────────┼────────────────────────────────┤
# │ {CODE_MIN} │ 最小配置 │ 最小值 │ 开发测试（最小规格） │
# │ {CODE_MID} │ 中等配置 │ 中等值 │ 小规模生产 │
# │ {CODE_HIGH} │ 高配置 │ 高值 │ 中规模生产 │
# │ {CODE_SERVERLESS} │ 弹性 │ 弹性 │ Serverless（按需弹性） │
# └──────────────────┴─────────┴──────────┴────────────────────────────────┘
```

### 产品类型差异表格

```hcl
# ┌───────────┬────────────────────────────────────────────────────────────┐
# │ 类型 │ 特点与约束 │
# ├───────────┼────────────────────────────────────────────────────────────┤
# │ {TYPE_A} │ 特点 A：支持参数列表，约束条件 │
# │ {TYPE_B} │ 特点 B：与 A 的差异说明 │
# │ {TYPE_C} │ 特点 C：特殊配置要求 │
# └───────────┴────────────────────────────────────────────────────────────┘
```

### 参数联动表格

```hcl
# ┌─────────────────────┬────────────┬──────────────────────────────────┐
# │ 主参数 │ 联动参数 │ 说明 │
# ├─────────────────────┼────────────┼──────────────────────────────────┤
# │ {PARAM}=VALUE_A │ 必填：X, Y │ VALUE_A 时的必填参数 │
# │ {PARAM}=VALUE_B │ 必填：Z │ VALUE_B 时的必填参数 │
# │ {PARAM}=VALUE_C │ 可选：W │ VALUE_C 时的可选参数 │
# └─────────────────────┴────────────┴──────────────────────────────────┘
```

## 自动化检查清单

在编写声明层配置时，确保：

- [ ] 从 Provider 源码提取了有效的规格代码
- [ ] 为每个子类型提供了最小规格实例
- [ ] 产品类型差异用表格清晰展示
- [ ] 参数联动关系用表格说明
- [ ] ForceNew 参数已标注 `(ForceNew)`
- [ ] 枚举值已标注所有可选项
- [ ] 第一个实例 `create = true` 用于测试验证

## 参考资料

- 相关规则：`variable-layered-type-design.md`、`variable-auto-naming.md`

---

# 声明层分阶段配置与共享参数体系

**优先级：** 高
**分类：** 三层架构（层级专用）

## 为什么重要

声明层按阶段（Stage）组织目录，每个阶段对应一个控制层模块。阶段间通过 `terraform_remote_state` 传递 ID map。环境级共享参数（region、az_map、tags）抽取到 `00-shared/shared.tfvars`，避免重复配置和跨阶段不一致。

## 目录结构模式

```
declarative/{environment}/
├── 00-shared/              # 共享参数（无 Terraform 资源）
│   └── shared.tfvars       # 环境级参数：env, region, az_map, tags
├── 01-network/             # 网络层：VPC、VSwitch / VNet、Subnet
├── 02-security/            # 安全层：安全组 / NSG、ACL
├── 03-compute/             # 计算层：ACK / EKS / AKS 集群
├── 04-database/            # 数据层：RDS、Redis、Kafka、MongoDB
├── 05-middleware/          # 中间件层：网关、NAS / EFS / File Share
└── 06-web/                 # Web 层：SLB / ALB / LB、ECS / EC2 / VM
```

### 阶段编号约定

| 编号 | 阶段名 | 职责 | 依赖上游 |
|------|--------|------|----------|
| 00 | shared | 环境级共享参数 | 无 |
| 01 | network | 网络拓扑（VPC/VNet、VSwitch/Subnet） | shared |
| 02 | security | 安全组 / NSG | network |
| 03 | compute | 容器集群（ACK/EKS/AKS） | network, security |
| 04 | database | 数据库/缓存/消息队列 | network, security |
| 05 | middleware | 中间件（网关、文件存储） | network, security |
| 06 | web | Web 服务（负载均衡、计算实例） | network, security, compute |

**依赖方向**：编号小的阶段先 apply，编号大的阶段通过 `terraform_remote_state` 引用上游输出。

## 双层配置体系

### 第一层：shared.tfvars（环境级共享参数）

```hcl
# 00-shared/shared.tfvars — 环境级参数，所有阶段共用

# ——— 基础标识 ———
env     = "simple"
project = "cc"

# ——— 区域配置（多云适配）———
# 阿里云
region = "cn-hangzhou"
# AWS
# region = "us-east-1"
# Azure
# location = "eastus"

# ——— 可用区映射（跨 region 迁移只需改这里）———
az_map = {
  j = "cn-hangzhou-j"
  k = "cn-hangzhou-k"
  # AWS:
  # a = "us-east-1a"
  # b = "us-east-1b"
  # Azure:
  # a = "eastus-1"
  # b = "eastus-2"
}

# ——— 统一标签 ———
tags = {
  "ops-env" = "simple"
  "project" = "cc"
}
```

### 第二层：terraform.tfvars（阶段级资源配置）

```hcl
# 01-network/terraform.tfvars — 仅本阶段资源配置

# 阿里云 VSwitch
vswitches = {
  "j-ecs" = {
    cidr_block = "172.31.192.0/24"
    zone_key   = "j"
    purpose    = "ECS"
  }
  "k-ecs" = {
    cidr_block = "172.31.193.0/24"
    zone_key   = "k"
    purpose    = "ECS"
  }
  "j-data" = {
    cidr_block = "172.31.196.0/24"
    zone_key   = "j"
    purpose    = "Data"
  }
}

# AWS Subnet（等价配置）
# subnets = {
#   "a-public" = {
#     cidr_block = "10.0.1.0/24"
#     zone_key   = "a"
#     purpose    = "public"
#   }
#   "b-private" = {
#     cidr_block = "10.0.2.0/24"
#     zone_key   = "b"
#     purpose    = "private"
#   }
# }
```

### 配置加载方式

```bash
# 每个阶段同时加载两个 tfvars
terraform plan \
  -var-file="../00-shared/shared.tfvars" \
  -var-file="terraform.tfvars"

# 或在 terraform.tf 中配置（如使用 Terramate）
```

## 参数归属决策

| 参数类型 | 放置位置 | 示例 |
|----------|----------|------|
| 环境标识（env） | shared.tfvars | `env = "simple"` |
| 区域/位置 | shared.tfvars | `region = "cn-hangzhou"` / `location = "eastus"` |
| 可用区映射（az_map） | shared.tfvars | `az_map = { j = "cn-hangzhou-j" }` |
| 统一标签（tags） | shared.tfvars | `tags = { "ops-env" = "simple" }` |
| 项目标识（project） | shared.tfvars | `project = "cc"` |
| 资源实例配置 | terraform.tfvars | `kafka_instances = { ... }` |
| 子网 CIDR | terraform.tfvars | `cidr_block = "172.31.192.0/24"` |
| 数据库规格 | terraform.tfvars | `db_node_class = "polar.mysql.g2.xlarge"` |
| 安全组规则 | terraform.tfvars | `security_group_rules = { ... }` |

## 阶段间引用模式

### 声明层 main.tf 结构

```hcl
# 04-database/main.tf

# 引用上游阶段状态
data "terraform_remote_state" "network" {
  backend = "local"
  config = { path = "../01-network/terraform.tfstate" }
}

data "terraform_remote_state" "security" {
  backend = "local"
  config = { path = "../02-security/terraform.tfstate" }
}

# 调用控制层模块
module "db" {
  source = "../../../control/database-cluster"

  # 从 shared.tfvars 传入
  env    = var.env
  region = var.region
  az_map = var.az_map
  tags   = var.tags

  # 从上游状态传入 ID map
  vswitch_ids_map         = data.terraform_remote_state.network.outputs.vswitch_ids_map
  security_group_ids_map  = data.terraform_remote_state.security.outputs.security_group_ids_map

  # 从本阶段 tfvars 传入
  kafka_instances = var.kafka_instances
  redis_instances = var.redis_instances
}
```

### 声明层 outputs.tf（传递给下游）

```hcl
# 01-network/outputs.tf — 输出供下游阶段引用

output "vswitch_ids_map" {
  description = "VSwitch/Subnet ID 映射（key = 逻辑名）"
  value = {
    "j-ecs"  = module.network.vswitch_ids["j-ecs"]
    "k-ecs"  = module.network.vswitch_ids["k-ecs"]
    "j-data" = module.network.vswitch_ids["j-data"]
    "j-pod"  = module.network.vswitch_ids["j-pod"]
    "k-pod"  = module.network.vswitch_ids["k-pod"]
  }
}

output "vpc_id" {
  description = "VPC/VNet ID"
  value = module.network.vpc_id
}
```

## 多云目录结构

```
declarative/
├── alicloud/             # 阿里云环境
│   ├── simple/
│   │   ├── 00-shared/shared.tfvars  # region="cn-hangzhou"
│   │   └── ...
│   └── prod/
│       ├── 00-shared/shared.tfvars  # region="cn-beijing"
│       └── ...
├── aws/                  # AWS 环境
│   ├── dev/
│   │   ├── 00-shared/shared.tfvars  # region="us-east-1"
│   │   └── ...
│   └── prod/
│       ├── 00-shared/shared.tfvars  # region="us-west-2"
│       └── ...
└── azure/                # Azure 环境
    ├── dev/
    │   ├── 00-shared/shared.tfvars  # location="eastus"
    │   └── ...
    └── prod/
        ├── 00-shared/shared.tfvars  # location="westeurope"
        └── ...
```

**环境差异只在 shared.tfvars 和 terraform.tfvars 中体现**，控制层和原子层代码尽量复用。

## 多环境管理（单云多环境）

```
declarative/
├── simple/              # 简单环境
│   ├── 00-shared/shared.tfvars    # env="simple", region="cn-hangzhou"
│   ├── 01-network/terraform.tfvars
│   └── ...
├── test33/              # 测试环境
│   ├── 00-shared/shared.tfvars    # env="test2", region="cn-hangzhou"
│   ├── 01-network/terraform.tfvars
│   └── ...
└── prod/                # 生产环境（未来）
    ├── 00-shared/shared.tfvars    # env="prod", region="cn-beijing"
    ├── 01-network/terraform.tfvars
    └── ...
```

## 检查清单

- [ ] 每个环境有 `00-shared/shared.tfvars` 包含 env、region/location、az_map、tags
- [ ] 阶段编号按依赖顺序排列（01 → 06）
- [ ] 下游阶段通过 `terraform_remote_state` 引用上游 ID map
- [ ] 每个阶段 outputs.tf 输出完整的 ID map 供下游使用
- [ ] 资源配置写在各阶段 `terraform.tfvars`，不写在 shared.tfvars
- [ ] 跨环境复用的参数在 shared.tfvars，环境特定的在 terraform.tfvars
- [ ] terraform plan 同时加载 shared.tfvars 和 terraform.tfvars
- [ ] 多云场景下，az_map 使用统一的逻辑 key（j/k 或 a/b）

## 参考资料

- `layer-id-mapping.md`（ID 映射模式）
- `declaration-layer-configuration-patterns.md`（声明层 tfvars 编写规范）
- `variable-layered-type-design.md`（三层变量类型设计）
- `control-parameter-passthrough.md`（控制层参数透传）

---

# 控制层参数透传与控制层业务默认值

**优先级：** 中高
**分类：** 三层架构

## 为什么重要

控制层在声明层和原子层之间承担"透传 + 填充默认值"的角色。区分哪些默认值应该设在控制层、哪些在原子层，是三层架构的关键决策。错误放置默认值会导致隐藏配置或状态漂移。

## 核心原则

### 原子层：技术纯度默认值

```hcl
# 原子层 variables.tf — 技术纯度原则
variable "strict_consistency" {
  type    = string
  default = ""  # 空字符串 = 不设置，让 API 决定
  description = "ForceNew 参数。空字符串不传递给 API。"
}
```

### 控制层：业务默认值

```hcl
# 控制层 main.tf — 业务默认值
strict_consistency = try(each.value.strict_consistency, "")  # 不设默认
lower_case_table_names = try(each.value.lower_case_table_names, "1")  # 业务默认 "1"
```

### 声明层：显式配置

```hcl
# 声明层 terraform.tfvars — 只写需要覆盖的值
lower_case_table_names = "0"  # 显式覆盖默认值
```

## 透传模式

### 模式 1：纯透传（无默认值）

声明层传什么，原子层就收到什么：

```hcl
# 控制层
password = try(each.value.password, "")
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `password = "xxx"` | `"xxx"` | `"xxx"` |
| 不设置 | `""` | `""` → null（原子层处理） |

### 模式 2：透传 + 业务默认值

声明层不设置时使用控制层默认值：

```hcl
# 控制层
engine_version = try(each.value.engine_version, "7.0")
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `engine_version = "5.0"` | `"5.0"` | `"5.0"` |
| 不设置 | `"7.0"` | `"7.0"` |

### 模式 3：透传 + Key 解析

声明层给 key，控制层解析为 ID：

```hcl
# 控制层
vswitch_id = var.vswitch_ids_map[try(each.value.vswitch_key, "j-data")]
```

| 声明层 | 控制层 | 原子层 |
|--------|--------|--------|
| `vswitch_key = "j-ecs"` | `vswitch_ids_map["j-ecs"]` | `"vsw-xxx"` |
| 不设置 | `vswitch_ids_map["j-data"]` | `"vsw-yyy"` |

### 模式 4：透传 + locals 预处理

声明层数据需要加工后再传给原子层：

```hcl
# 控制层 locals 预处理
mysql_ecs_data_disks_resolved = {
  for k, v in var.mysql_ecs_instances : k => [
    for disk in try(v.data_disks, []) : {
      name                    = disk.name
      auto_snapshot_policy_id = try(disk.auto_snapshot_policy_key, "") != ""
        ? lookup(var.ecs_snapshot_policy_ids_map, disk.auto_snapshot_policy_key, "")
        : try(disk.auto_snapshot_policy_id, "")
    }
  ]
}

# 控制层使用预处理结果
data_disks = local.mysql_ecs_data_disks_resolved[each.key]
```

## 默认值放置决策矩阵

| 场景 | 放置位置 | 说明 |
|------|----------|------|
| API 返回值决定的参数 | 原子层 `default = ""` | 让 API 自动填充 |
| ForceNew 参数 | 原子层 `default = ""` | 空值不触发重建 |
| 业务通用默认值（如 engine_version） | 控制层 `try(..., "7.0")` | 项目统一标准 |
| 基础设施默认值（如 vswitch_key） | 控制层 `try(..., "j-data")` | 环境级别默认 |
| 安全组 ID | 控制层 key 映射 | 跨模块引用 |
| 集合类型（list/map） | 控制层 `try(..., [])` | 避免 null |
| 标签 | 控制层 `merge(local.common_tags, ...)` | 统一标签策略 |

## 多云场景适配

### AWS 参数透传

```hcl
# AWS RDS 控制层透传
module "rds" {
  source = "../../atomic/aws-rds-instance"

  for_each = var.rds_instances

  engine               = try(each.value.engine, "mysql")
  engine_version       = try(each.value.engine_version, "8.0")
  instance_class       = try(each.value.instance_class, "db.t3.medium")
  allocated_storage    = try(each.value.allocated_storage, 20)
  subnet_group_name    = try(var.subnet_ids_map[each.value.subnet_key], "")
  vpc_security_group_ids = try(each.value.security_group_key, "") != ""
    ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
    : try(each.value.security_group_ids, [])
}
```

### Azure 参数透传

```hcl
# Azure MySQL 控制层透传
module "mysql" {
  source = "../../atomic/azurerm-mysql-flexible-server"

  for_each = var.mysql_servers

  sku_name             = try(each.value.sku_name, "B_Standard_B1s")
  version              = try(each.value.version, "8.0.21")
  resource_group_name  = var.resource_group_name
  location             = var.location
  subnet_id            = try(var.subnet_ids_map[each.value.subnet_key], "")
}
```

## 安全组默认值原则

```hcl
# 正确：控制层不设安全组业务默认值
# 声明层必须显式配置 security_group_key
security_group_ids = try(each.value.security_group_key, "") != ""
  ? [lookup(var.security_group_ids_map, each.value.security_group_key, "")]
  : try(each.value.security_group_ids, [])

# 错误：控制层硬编码安全组
security_group_ids = ["sg-xxx"]  # 不可移植
```

## security_ips 的特殊性

```hcl
# 正确：security_ips 的业务默认值是 ["127.0.0.1"]
# 这是 API 的默认行为，控制层需要与 API 保持一致
security_ips = try(each.value.security_ips, ["127.0.0.1"])

# 原因：如果不传 security_ips，API 仍然会设置 ["127.0.0.1"]
# 透传时也设置 ["127.0.0.1"]，才能避免状态漂移
```

## 参考资料

- `variable-atomic-defaults.md`（原子层默认值原则）
- `variable-forcenew-defaults.md`（ForceNew 参数空值规则）
- `layer-separation.md`（三层职责划分）
- `control-key-mapping-pattern.md`（安全组 Key 映射模式）

---

# 原子层代码参考规范

**优先级：** 中
**分类：** 三层架构

> 原子层代码中应在关键配置处添加官方文档参考链接，便于查阅和理解。

## 为什么重要

在原子层代码中添加官方文档参考链接有以下好处：

1. **可追溯性**：配置参数有据可查，方便后续维护
2. **学习效率**：新人阅读代码时能快速了解参数含义
3. **准确性验证**：遇到疑问可直接查阅官方文档确认
4. **知识沉淀**：将分散的官方文档整合到代码中

## 推荐格式

### 1. 文件头部参考

```hcl
################################################################################
# MongoDB 云数据库
# 设计模式：create 开关 + locals 前置 + try 安全输出
# 实例类型：副本集（Replica Set）/ 分片集群（Sharded Cluster）
# 官方文档：https://help.aliyun.com/zh/mongodb/product-overview/instance-types
# Provider 文档：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/mongodb_instance
################################################################################
```

### 2. 关键配置块参考

```hcl
# ——— Mongos 节点规格 ———
# 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
# 必填：node_class（Mongos 规格）
# 可选：无（Mongos 无存储配置）
mongo_list {
  node_class = var.mongos_node_class
}

# ——— Shard 节点规格 ———
# 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
# 必填：node_class, node_storage
# 可选：readonly_replicas（0-5，默认 0）
shard_list {
  node_class        = var.shard_node_class
  node_storage      = var.shard_node_storage
  readonly_replicas = var.shard_readonly_replicas
}
```

### 3. 规格映射表参考

```hcl
# 经典版集群规格固定值映射（防止状态漂移）
# 格式：shard_count = 分片数, capacity_mb = 总容量(MB)
# 参考文档：https://help.aliyun.com/document_detail/26350.html
classic_cluster_specs = {
  "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }  # small = 8分片×2GB
  "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }  # medium = 8分片×4GB
  "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }  # large = 8分片×8GB
}
```

### 4. 特殊逻辑说明参考

```hcl
# ——— 架构类型判断 ———
# 判断依据：replication_factor = 0 表示分片集群
# 参考：https://help.aliyun.com/zh/mongodb/developer-reference/api-dds-2015-12-01-createdbinstance
is_sharding = var.replication_factor == 0

# ——— ForceNew 参数处理 ———
# storage_engine 是 ForceNew 参数，修改会触发重建
# 参考：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/mongodb_instance#storage_engine
storage_engine = var.storage_engine
```

## 最佳实践示例

### Redis 原子层示例

```hcl
locals {
  # ——— 架构类型判断 ———
  # 判断依据：deploy_type 后缀决定架构类型
  # 参考文档：https://help.aliyun.com/document_detail/26350.html
  is_classic    = var.deploy_type == "classic"
  is_cloud_native = var.deploy_type == "cloud_native"
  is_yitian     = var.deploy_type == "yitian"
  is_cluster    = var.instance_class != "" && (
    can(regex("sharding", var.instance_class)) ||
    can(regex("shard\\.with\\.proxy", var.instance_class))
  )

  # 经典版集群规格固定值映射（防止状态漂移）
  # 格式：shard_count = 分片数, capacity_mb = 总容量(MB)
  # 参考文档：https://help.aliyun.com/document_detail/26350.html
  classic_cluster_specs = {
    "redis.sharding.basic.small.default"    = { shard_count = 8,  capacity_mb = 16384 }
    "redis.sharding.basic.medium.default"   = { shard_count = 8,  capacity_mb = 32768 }
    "redis.sharding.basic.large.default"    = { shard_count = 8,  capacity_mb = 65536 }
    "redis.sharding.basic.xlarge.default"   = { shard_count = 8,  capacity_mb = 131072 }
    "redis.sharding.2xlarge.default"        = { shard_count = 8,  capacity_mb = 262144 }
  }
}
```

### MongoDB 分片集群原子层示例

```hcl
resource "alicloud_mongodb_sharding_instance" "this" {
  count = local.create ? 1 : 0

  # ——— 基础配置 ———
  engine_version = var.engine_version
  
  # ——— Mongos 节点规格 ———
  # 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  # Mongos 规格：2核8GB / 2核16GB / 4核8GB / 4核16GB 等
  # 数量范围：2-32 个
  dynamic "mongo_list" {
    for_each = var.mongos_list
    content {
      node_class = mongo_list.value.node_class
    }
  }

  # ——— Shard 节点规格 ———
  # 规格参考：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  # Shard 规格：2核8GB / 2核16GB / 4核8GB / 4核16GB 等
  # 存储范围：20-16000 GB
  # 数量范围：2-32 个
  dynamic "shard_list" {
    for_each = var.shards_list
    content {
      node_class        = shard_list.value.node_class
      node_storage      = shard_list.value.node_storage
      readonly_replicas = try(shard_list.value.readonly_replicas, 0)
    }
  }

  # ——— ConfigServer 规格（可选）——
  # 默认值：dds.cs.mid（1核2GB）
  # 参考文档：https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types
  dynamic "config_server_list" {
    for_each = var.config_server_list
    content {
      node_class   = config_server_list.value.node_class
      node_storage = config_server_list.value.node_storage
    }
  }
}
```

## 常用参考链接

### 阿里云产品文档

| 产品 | 文档地址 |
|------|---------|
| MongoDB | https://help.aliyun.com/zh/mongodb/ |
| Redis | https://help.aliyun.com/zh/redis/ |
| PolarDB | https://help.aliyun.com/zh/polardb/ |
| RDS | https://help.aliyun.com/zh/rds/ |
| Kafka | https://help.aliyun.com/zh/alikafka/ |
| Elasticsearch | https://help.aliyun.com/zh/elasticsearch/ |

### 规格表参考

| 产品 | 规格表地址 |
|------|-----------|
| MongoDB 分片集群 | https://help.aliyun.com/zh/mongodb/product-overview/sharded-cluster-instance-types |
| MongoDB 副本集 | https://help.aliyun.com/zh/mongodb/product-overview/replica-set-instance-types |
| Redis 社区版 | https://help.aliyun.com/document_detail/26350.html |
| Redis Tair | https://help.aliyun.com/zh/redis/product-overview/tair-instance-types |
| PolarDB MySQL | https://help.aliyun.com/zh/polardb/polardb-for-mysql/user-guide/primary-mp-instance-types |

### Terraform Provider 文档

| Provider | 文档地址 |
|---------|---------|
| alicloud | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs |

## 相关规则

- `comment-standardization` - 参数差异化注释规范
- `state-analysis-vs-rebuild` - State 分析与配置重建规范

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [Terraform Registry](https://registry.terraform.io/)

---

# 跨阶段引用完整性验证

**优先级：** 中
**分类：** 三层架构

## 为什么重要

声明层各阶段（01-network → 02-security → 04-database → 06-web）通过 `terraform_remote_state` 引用上游输出。如果上游删除或重命名了某个 output，下游的 `remote_state.outputs.xxx` 会在 `terraform apply` 时才报错，而非 `terraform plan` 时。这种延迟报错增加了排查成本。

## 问题：上游输出变更导致下游运行时失败

```hcl
# 01-network/outputs.tf 中删除了 vpc_cidr_block 输出
# output "vpc_cidr_block" { value = local.vpc_cidr_block }  # 被删除

# 04-database/main.tf 仍然引用
data "terraform_remote_state" "network" {
  backend = "local"
  config  = { path = "../01-network/terraform.tfstate" }
}

# ❌ apply 时才报错：This object does not have an attribute named "vpc_cidr_block"
```

**问题：**
1. 删除上游 output 时，无法立即知道哪些下游依赖它
2. 错误在 apply 阶段才暴露，而非 plan 阶段
3. 多阶段部署时，可能在中途失败，需要回滚

## 当前跨阶段引用映射

```
01-network outputs:
  ├── vpc_id
  ├── vswitch_ids_map
  ├── az_map
  ├── vpc_cidr_block
  ├── nat_gateway_ids_map
  └── nat_eip_addresses_map
      ↓
02-security inputs:  vpc_id, vswitch_ids_map
02-security outputs:
  ├── security_group_ids_map
  ├── slb_acl_ids_map
  ├── alb_acl_ids_map
  ├── ecs_snapshot_policy_ids_map
  └── hbr_vault_ids_map
      ↓
04-database inputs:  vpc_id, vswitch_ids_map, security_group_ids_map, ...
05-middleware inputs: vpc_id, vswitch_ids_map, az_map, security_group_ids_map, ...
06-web inputs:       vpc_id, vswitch_ids_map, az_map, security_group_ids_map,
                     slb_acl_ids_map, alb_acl_ids_map, ecs_snapshot_policy_ids_map, ...
```

## 防护方案

### 方案 1：pre-commit 钩子自动验证

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: validate-cross-stage-refs
        name: Validate cross-stage remote_state references
        entry: scripts/validate-cross-stage-refs.sh
        language: script
        files: 'declarative/.*/main\.tf$'
```

```bash
# scripts/validate-cross-stage-refs.sh
# 检查每个 main.tf 中的 remote_state.outputs.xxx 是否在上游 outputs.tf 中定义

for stage_dir in declarative/simple/0*; do
  # 提取引用的 output keys
  refs=$(grep -oP 'remote_state\.\w+\.outputs\.\K\w+' "$stage_dir/main.tf" | sort -u)

  # 提取上游定义的 output keys
  upstream=$(grep -oP 'output "\K[^"]+' "../01-network/outputs.tf" | sort -u)

  # 检查每个引用是否都有定义
  for ref in $refs; do
    if ! echo "$upstream" | grep -q "^$ref$"; then
      echo "ERROR: $stage_dir references '$ref' but upstream doesn't define it"
    fi
  done
done
```

### 方案 2：在 outputs.tf 中添加注释标记

```hcl
# 01-network/outputs.tf

# ⚠️ 下游引用：02-security, 03-ack, 04-database, 05-middleware, 06-web
output "vpc_id" {
  description = "VPC ID（供所有下游阶段引用）"
  value       = module.network.vpc_id
}

# ⚠️ 下游引用：04-database, 06-web
output "vswitch_ids_map" {
  description = "VSwitch ID 映射（供 04-database、06-web 引用）"
  value       = module.network.vswitch_ids_map
}
```

### 方案 3：terraform plan 前置检查

```bash
# 部署前检查上游 state 是否包含需要的 outputs
terraform plan -var-file="..." -var-file="..." 2>&1 | grep "does not have an attribute"
```

## 变更上游 output 时的检查流程

1. 搜索下游所有 `main.tf` 中对该 output 的引用
2. 确认没有下游依赖后再删除
3. 如果有，删除或重命名
3. 更新下游引用
4. 运行 `terraform plan` 验证

```bash
# 搜索所有引用
grep -r "remote_state.*outputs\.vpc_cidr_block" modules/declarative/
```

## 检查清单

- [ ] 每个声明阶段的 `main.tf` 引用的 `remote_state.outputs.xxx` 在上游 `outputs.tf` 中有定义
- [ ] 删除上游 output 前搜索下游引用
- [ ] outputs.tf 中标注下游引用范围（注释或 description）
- [ ] 考虑添加 pre-commit 钩子自动验证

## 参考资料

- `declarative-staged-configuration.md`（声明层分阶段配置）
- `layer-id-mapping.md`（ID 映射模式）
- `data-source-result-validation.md`（数据源结果验证）

---

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

---

# 如何新增规则

**优先级：** 中
**分类：** 元规则（技能维护）

## 为什么重要

新增规则时遵循统一的格式和流程，确保技能体系的一致性和可维护性。本指南总结了已有 63+ 条规则中提炼的最佳实践。

## 规则文件模板

创建新规则文件时，使用以下模板：

```markdown
# [中文标题，简洁明了]

**优先级：** [关键/高/中高/中/低中/低]
**分类：** [对应分类名]

## 为什么重要

[1-3 句话说明为什么这条规则重要，不遵守会有什么后果]

## 错误示例

[如果适用] 展示违反规则的代码和问题说明

\```hcl
# 违反规则的代码
\```

**问题：**
- [具体问题 1]
- [具体问题 2]

## 正确示例

[展示遵循规则的代码和解释]

\```hcl
# 遵循规则的代码
\```

## 参考资料

- [相关官方文档链接]
- [相关规则文件引用]
```

## 文件命名规范

### 格式

```
{分类前缀}-{简短描述}.md
```

### 分类前缀对照表

| 分类 | 前缀 | 示例 |
|------|------|------|
| 组织与工作流 | `org-` | `org-version-control.md` |
| 状态管理 | `state-` | `state-remote-backend.md` |
| 安全最佳实践 | `security-` | `security-credentials.md` |
| 三层架构 | `layer-` 或混合前缀 | `layer-separation.md` |
| 测试与验证 | `test-` | `test-strategies.md` |
| 模块设计 | `module-` | `module-versioning.md` |
| 资源组织 | `resource-` | `resource-naming.md` |
| 变量与输出模式 | `variable-` / `output-` | `variable-types.md` |
| 语言最佳实践 | `language-` | `language-locals.md` |
| Provider 配置 | `provider-` | `provider-version-constraints.md` |
| 性能优化 | `perf-` | `perf-parallelism.md` |
| 元规则 | `meta-` | `meta-how-to-add-rules.md` |

### 命名规则

- 使用小写字母和连字符（kebab-case）
- 名称应简短但能说明规则内容
- 避免使用缩写，除非是公认的（如 `vpc`、`eks`）

## 优先级选择指南

选择优先级时，问自己："不遵守这条规则，最坏会发生什么？"

| 优先级 | 判断标准 | 示例 |
|--------|----------|------|
| **关键** | 安全事故、数据丢失、生产故障 | 硬编码密钥、无状态锁定 |
| **高** | 部署失败、状态损坏、重大损失 | 模块无版本、ForceNew 漂移 |
| **中高** | 可维护性下降、运维成本上升 | count 代替 for_each、无标签 |
| **中** | 代码质量下降、可读性变差 | 无变量描述、硬编码数据源 |
| **低中** | 性能次优、工具配置不完善 | 并行度未调优 |
| **低** | 可选的最佳实践 | 策略即代码 |

## 分类选择指南

### 决策流程

```
新规则是关于什么的？
│
├─ 团队协作流程？ → 组织与工作流 (org-)
├─ 状态文件管理？ → 状态管理 (state-)
├─ 安全/密钥/权限？ → 安全最佳实践 (security-)
├─ 三层架构设计？ → 三层架构 (layer-/混合)
├─ 测试/验证/策略？ → 测试与验证 (test-)
├─ 模块设计/复用？ → 模块设计 (module-)
├─ 资源管理/组织？ → 资源组织 (resource-)
├─ 变量/输出定义？ → 变量与输出模式 (variable-/output-)
├─ HCL 语言特性？ → 语言最佳实践 (language-)
├─ Provider 配置？ → Provider 配置 (provider-)
├─ 执行性能？ → 性能优化 (perf-)
└─ 技能体系本身？ → 元规则 (meta-)
```

## 代码示例编写指南

### 好的代码示例

```hcl
# 好的示例：完整、有上下文、能直接理解
variable "environment" {
  type        = string
  description = "部署环境"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "环境必须是 dev、staging 或 prod。"
  }
}
```

### 避免的写法

```hcl
# 不好的示例：片段、无上下文、需要猜测
validation {
  condition = contains([...], var.x)
}
```

### 代码示例原则

1. **完整可理解**：不需要额外上下文就能理解示例
2. **使用真实场景**：用多云资源（aws_instance、azurerm_mysql_flexible_server、tencentcloud_mysql_instance 等）
3. **注释说明意图**：在关键行加注释说明为什么这样做
4. **不要用省略号**：避免 `# ... 还有 200 行` 这种写法，除非是在说明代码量
5. **HCL 缩进 2 空格**：遵循 `terraform fmt` 标准

## 新增规则的完整流程

### 步骤 1：确认必要性

- [ ] 搜索现有规则，确认没有覆盖相同内容
- [ ] 确认该规则在实际项目中发生过问题
- [ ] 确认该规则有明确的"正确做法"

### 步骤 2：创建文件

- [ ] 使用模板创建 `{前缀}-{名称}.md`
- [ ] 填写标题、优先级、分类
- [ ] 编写"为什么重要"
- [ ] 编写错误示例（如适用）
- [ ] 编写正确示例
- [ ] 添加参考资料链接

### 步骤 3：更新索引

在 `SKILL.md` 中：

1. 找到对应的分类段落
2. 按逻辑顺序插入新规则条目
3. 更新该分类的规则计数
4. 更新顶部的总规则数

### 步骤 4：验证

```bash
# 检查文件格式
head -5 rules/{新文件名}.md

# 检查 SKILL.md 中的引用
grep "{新文件名}" SKILL.md

# 检查总数量
echo "SKILL.md 声明: $(grep -oP '\d+(?= 条规则)' SKILL.md | head -1)"
echo "实际文件数: $(ls rules/*.md | wc -l)"
```

## 常见错误

| 错误 | 说明 | 正确做法 |
|------|------|----------|
| 无优先级 | 省略 `**优先级：**` 行 | 必须填写 |
| 无分类 | 省略 `**分类：**` 行 | 必须填写 |
| 英文标题 | `## Why It Matters` | 使用 `## 为什么重要` |
| 过于笼统 | "不要写糟糕的代码" | 具体到可操作的规则 |
| 无代码示例 | 纯文字描述 | 配合 HCL 代码示例 |
| 示例不可执行 | 语法错误、缺少变量定义 | 确保 `terraform validate` 通过 |
| 未更新索引 | 创建文件但忘了更新 SKILL.md | 同步更新 |

## 参考资料

- SKILL.md（技能索引文件）
- `meta-skill-review.md`（评审方法）
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)

---

# 技能体系评审方法

**优先级：** 中
**分类：** 元规则（技能维护）

## 为什么重要

技能体系需要定期评审以确保质量、一致性和实用性。系统化的评审方法能发现隐藏缺陷，避免问题积累。

## 评审维度

### 1. 完整性检查

- **索引一致性**：SKILL.md 中列出的每条规则是否都有对应的文件？文件是否都在 SKILL.md 中列出？
- **数量核对**：声明的规则总数与实际文件数是否一致
- **分类核对**：每个文件的分类标签与 SKILL.md 中的归类是否一致
- **必填字段**：每个规则文件是否都有标题、优先级、分类、为什么重要、参考资料

### 2. 格式一致性

检查所有文件的格式是否遵循统一标准：

```
# 标题

**优先级：** [关键/高/中高/中/低中/低]
**分类：** [分类名]

## 为什么重要

[说明内容]

## 错误示例 / ## 正确示例 / ## 模式说明

[代码和解释]

## 参考资料

[链接列表]
```

### 3. 内容质量

- **代码示例正确性**：HCL 语法是否正确
- **示例实用性**：是否贴近生产场景
- **交叉引用有效性**：引用的其他规则文件是否存在
- **参考资料可用性**：外部链接是否有效

### 4. 优先级合理性

检查同类规则的优先级是否一致：

| 优先级 | 适用场景 |
|--------|----------|
| 关键 | 不遵守会导致安全事故、数据丢失 |
| 高 | 不遵守会导致部署失败、状态损坏 |
| 中高 | 不遵守会降低可维护性、增加运维成本 |
| 中 | 影响代码质量和可读性 |
| 低中 | 性能优化、工具配置 |
| 低 | 可选的最佳实践 |

## 评审检查清单

```markdown
## 评审检查清单

### 索引层
- [ ] SKILL.md 规则数 == 实际文件数
- [ ] SKILL.md 分类数与实际一致
- [ ] 每条规则在 SKILL.md 和文件中都有对应

### 文件层
- [ ] 所有文件有 优先级/分类 元数据
- [ ] 所有文件使用中文小节标题
- [ ] 所有文件有参考资料小节（或内容本身即参考）

### 内容层
- [ ] 代码示例语法正确
- [ ] 错误示例/正确示例对比清晰
- [ ] 交叉引用的文件存在
- [ ] 优先级在同类规则中一致

### 体系层
- [ ] 无重复规则
- [ ] 无孤立规则（不属于任何分类）
- [ ] 分类边界清晰（无规则归属模糊）
```

## 评审工具脚本

快速扫描缺失字段的脚本：

```bash
#!/bin/bash
# 评审扫描脚本
RULES_DIR="rules"

echo "=== 缺少 优先级 的文件 ==="
for f in $RULES_DIR/*.md; do
  grep -q '^\*\*优先级：' "$f" || echo "  $(basename $f)"
done

echo ""
echo "=== 缺少 分类 的文件 ==="
for f in $RULES_DIR/*.md; do
  grep -q '^\*\*分类：' "$f" || echo "  $(basename $f)"
done

echo ""
echo "=== 缺少 参考资料小节 的文件 ==="
for f in $RULES_DIR/*.md; do
  grep -q '^## 参考资料' "$f" || echo "  $(basename $f)"
done

echo ""
echo "=== 文件总数 ==="
ls $RULES_DIR/*.md | wc -l
```

## 常见缺陷模式

| 缺陷类型 | 表现 | 修复方式 |
|----------|------|----------|
| 幽灵规则 | SKILL.md 列出但无文件 | 创建文件或从 SKILL.md 移除 |
| 孤儿规则 | 文件存在但未在 SKILL.md 列出 | 添加到 SKILL.md 对应分类 |
| 重复规则 | 两条规则内容高度重叠 | 合并或明确区分 |
| 分类漂移 | 文件分类与 SKILL.md 不一致 | 统一到更合理的分类 |
| 元数据缺失 | 缺少优先级或分类行 | 按规则重要性补齐 |
| 语言混杂 | 同一文件中英文标题混用 | 统一为中文标题 |

## 参考资料

- SKILL.md（技能索引文件）
- `meta-how-to-add-rules.md`（新增规则指南）
