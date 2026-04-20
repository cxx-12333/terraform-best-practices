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
