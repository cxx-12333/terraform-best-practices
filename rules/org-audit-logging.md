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
