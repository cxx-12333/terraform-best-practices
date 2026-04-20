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
