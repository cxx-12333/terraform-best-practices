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
