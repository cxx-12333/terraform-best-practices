# 部署验证：漂移检测流程

**优先级：** 关键
**分类：** 测试与验证

## 为什么重要

`terraform apply` 后，云厂商 API 可能设置与代码不同的值，原因包括：
- API 默认值与 Terraform 默认值不同
- API 忽略的参数（由实例规格决定）
- Terraform 未预料的自动生成值
- Provider 端的标准化处理

这会导致**状态漂移**：`terraform.tfvars` ≠ `terraform.tfstate`

## 验证流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           部署验证流程                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. terraform plan                                                          │
│     │                                                                       │
│     ▼                                                                       │
│  2. terraform apply（首次）                                                  │
│     │                                                                       │
│     ▼                                                                       │
│  3. 等待 apply 完成（检查资源状态）                                            │
│     │                                                                       │
│     ▼                                                                       │
│  4. terraform plan（第二次 - 漂移检测）                                       │
│     │                                                                       │
│     ├── 无变更？ ──────────────────────► 5. 验证 state = code               │
│     │                                              │                        │
│     │                                              ▼                        │
│     │                                         6. 最终版本 ✓               │
│     │                                                                       │
│     └── 检测到变更？                                                          │
│              │                                                              │
│              ▼                                                              │
│         分析漂移原因：                                                        │
│         • API 忽略了参数？                                                    │
│         • API 默认值与代码不同？                                               │
│         • 规格决定的值？                                                      │
│              │                                                              │
│              ▼                                                              │
│         在原子层修复（自动对齐）                                                │
│              │                                                              │
│              ▼                                                              │
│         返回步骤 4                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 分步流程

### 步骤 1：初始 Plan

```bash
cd environments/${ENV}/04-database
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

检查：
- [ ] 无意外的资源销毁
- [ ] 资源数量符合预期
- [ ] Plan 输出中无敏感数据

### 步骤 2：首次 Apply

```bash
terraform apply -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

**重要：** 必须 100% 完成才可继续。

### 步骤 3：等待完成

对于长时间运行的资源，验证状态：

| 资源类型 | 检查命令 | 预期状态 |
|----------|----------|----------|
| ACK 集群 | aliyun cs GET /clusters/<id> | running |
| PolarDB | aliyun polardb DescribeDBClusters | Running |
| Redis | aliyun r-kvstore DescribeInstances | Normal |
| ES | aliyun elasticsearch DescribeInstance | active |

### 步骤 4：第二次 Plan（漂移检测）

```bash
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"
```

**这是关键验证步骤！**

| Plan 输出 | 含义 | 操作 |
|-----------|------|------|
| No changes | 状态与代码一致 | ✓ 进入步骤 5 |
| X to update | 检测到漂移 | ✗ 分析并修复 |

### 步骤 5：State-Code 一致性检查

```bash
# 显示当前状态
terraform show -json > current-state.json

# 与代码对比
# 需验证的关键字段：
# - instance_class 与 tfvars 一致
# - shard_count 与 tfvars 一致
# - capacity 与 tfvars 一致
# - 无意外的 null 值
```

### 步骤 6：最终版本

所有检查通过后：
```bash
# 提交验证后的状态
git add terraform.tfstate
git commit -m "feat: 已验证部署 - 无漂移"
```

## 常见漂移原因与修复

### 原因 1：API 默认值 ≠ 代码默认值

```hcl
# tfvars
security_ips = []  # 用户期望：不设置

# API 默认值
security_ips = ["127.0.0.1"]  # API 设置此值

# 结果：漂移！Code ≠ State
```

**修复：** 在控制层设置 API 默认值

```hcl
# control/database-cluster/main.tf
security_ips = try(each.value.security_ips, ["127.0.0.1"])  # API 默认值
```

### 原因 2：API 忽略参数（规格决定）

```hcl
# tfvars
shard_count = 2
capacity    = 4096

# instance_class = "redis.sharding.basic.small.default"
# API 忽略 shard_count/capacity，使用规格固定值：
#   shard_count = 8（此规格固定值）
#   capacity    = 16384（此规格固定值）

# 结果：漂移！Code ≠ State
```

**修复：** 原子层自动对齐

```hcl
# atomic/redis-community/main.tf
locals {
  is_classic_cluster = can(regex("\\.default$", var.instance_class)) && 
                       can(regex("sharding", lower(var.instance_class)))

  classic_specs = {
    "redis.sharding.basic.small.default"  = { shard_count = 8,  capacity = 16384 }
    "redis.sharding.basic.medium.default" = { shard_count = 8,  capacity = 32768 }
  }

  effective_shard_count = local.is_classic_cluster ? 
    local.classic_specs[var.instance_class].shard_count : var.shard_count

  effective_capacity = local.is_classic_cluster ? 
    local.classic_specs[var.instance_class].capacity : var.capacity
}
```

### 原因 3：ForceNew 参数变更

```hcl
# 修改 ForceNew 参数会导致资源替换
# terraform plan 显示："forces replacement"

# 常见 ForceNew 参数：
# - alicloud_polardb_cluster: zone_id, db_version
# - alicloud_kvstore_instance: engine_version（有时）
# - alicloud_instance: image_id, instance_type
```

**修复：** ForceNew 参数使用空字符串默认值

```hcl
# atomic/polardb/variables.tf
variable "strict_consistency" {
  description = "ForceNew 参数。空 = 不设置（避免触发替换）"
  type        = string
  default     = ""  # 空 = 不设置，避免 ForceNew
}

# atomic/polardb/main.tf
strict_consistency = var.strict_consistency != "" ? var.strict_consistency : null
```

## 漂移分析检查清单

当第二次 plan 显示变更时：

- [ ] 确定哪个资源有漂移
- [ ] 确定哪个参数不同
- [ ] 检查参数是否为 ForceNew
- [ ] 检查参数是否由规格决定
- [ ] 检查 API 默认值是否与代码默认值不同
- [ ] 在原子层/控制层应用适当修复
- [ ] 重新运行步骤 4 直到无变更

## 验证脚本模板

```bash
#!/bin/bash
# verify-deployment.sh

set -e

PHASE=$1
TFVARS_DIR="environments/${ENV}/${PHASE}"

echo "=== 步骤 1：初始 Plan ==="
cd $TFVARS_DIR
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"

echo ""
echo "=== 步骤 2：Apply ==="
read -p "继续 apply？(y/n) " confirm
if [ "$confirm" = "y" ]; then
  terraform apply -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars" -auto-approve
fi

echo ""
echo "=== 步骤 3：等待完成 ==="
echo "检查控制台确认资源状态..."

echo ""
echo "=== 步骤 4：第二次 Plan（漂移检测） ==="
terraform plan -var-file="../00-shared/shared.tfvars" -var-file="terraform.tfvars"

echo ""
echo "=== 验证完成 ==="
```

## 检查清单

- [ ] 步骤 1：初始 terraform plan 已审查
- [ ] 步骤 2：首次 terraform apply 已完成
- [ ] 步骤 3：资源已完全配置（状态已验证）
- [ ] 步骤 4：第二次 terraform plan 显示 **无变更**
- [ ] 步骤 5：State-Code 一致性已验证
- [ ] 步骤 6：最终版本已提交

## 决策规则

> **在第二次 terraform plan 显示"无变更"之前，永远不要认为部署完成。首次 apply 可能因 API 默认值、规格决定值或忽略参数引入状态漂移。在标记任务完成前，务必验证无漂移状态。**

## 参考资料

- resource-spec-auto-correction.md - 规格决定值的自动对齐
- variable-atomic-defaults.md - 原子层默认值策略
- variable-forcenew-defaults.md - ForceNew 参数处理