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
