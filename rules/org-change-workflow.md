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
