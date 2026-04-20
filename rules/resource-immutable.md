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
