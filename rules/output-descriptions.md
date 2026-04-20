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
