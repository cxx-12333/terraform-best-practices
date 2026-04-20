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
