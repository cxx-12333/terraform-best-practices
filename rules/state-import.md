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
