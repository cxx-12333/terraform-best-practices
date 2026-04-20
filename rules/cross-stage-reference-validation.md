# 跨阶段引用完整性验证

**优先级：** 中
**分类：** 三层架构

## 为什么重要

声明层各阶段（01-network → 02-security → 04-database → 06-web）通过 `terraform_remote_state` 引用上游输出。如果上游删除或重命名了某个 output，下游的 `remote_state.outputs.xxx` 会在 `terraform apply` 时才报错，而非 `terraform plan` 时。这种延迟报错增加了排查成本。

## 问题：上游输出变更导致下游运行时失败

```hcl
# 01-network/outputs.tf 中删除了 vpc_cidr_block 输出
# output "vpc_cidr_block" { value = local.vpc_cidr_block }  # 被删除

# 04-database/main.tf 仍然引用
data "terraform_remote_state" "network" {
  backend = "local"
  config  = { path = "../01-network/terraform.tfstate" }
}

# ❌ apply 时才报错：This object does not have an attribute named "vpc_cidr_block"
```

**问题：**
1. 删除上游 output 时，无法立即知道哪些下游依赖它
2. 错误在 apply 阶段才暴露，而非 plan 阶段
3. 多阶段部署时，可能在中途失败，需要回滚

## 当前跨阶段引用映射

```
01-network outputs:
  ├── vpc_id
  ├── vswitch_ids_map
  ├── az_map
  ├── vpc_cidr_block
  ├── nat_gateway_ids_map
  └── nat_eip_addresses_map
      ↓
02-security inputs:  vpc_id, vswitch_ids_map
02-security outputs:
  ├── security_group_ids_map
  ├── slb_acl_ids_map
  ├── alb_acl_ids_map
  ├── ecs_snapshot_policy_ids_map
  └── hbr_vault_ids_map
      ↓
04-database inputs:  vpc_id, vswitch_ids_map, security_group_ids_map, ...
05-middleware inputs: vpc_id, vswitch_ids_map, az_map, security_group_ids_map, ...
06-web inputs:       vpc_id, vswitch_ids_map, az_map, security_group_ids_map,
                     slb_acl_ids_map, alb_acl_ids_map, ecs_snapshot_policy_ids_map, ...
```

## 防护方案

### 方案 1：pre-commit 钩子自动验证

```yaml
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: validate-cross-stage-refs
        name: Validate cross-stage remote_state references
        entry: scripts/validate-cross-stage-refs.sh
        language: script
        files: 'declarative/.*/main\.tf$'
```

```bash
# scripts/validate-cross-stage-refs.sh
# 检查每个 main.tf 中的 remote_state.outputs.xxx 是否在上游 outputs.tf 中定义

for stage_dir in declarative/simple/0*; do
  # 提取引用的 output keys
  refs=$(grep -oP 'remote_state\.\w+\.outputs\.\K\w+' "$stage_dir/main.tf" | sort -u)

  # 提取上游定义的 output keys
  upstream=$(grep -oP 'output "\K[^"]+' "../01-network/outputs.tf" | sort -u)

  # 检查每个引用是否都有定义
  for ref in $refs; do
    if ! echo "$upstream" | grep -q "^$ref$"; then
      echo "ERROR: $stage_dir references '$ref' but upstream doesn't define it"
    fi
  done
done
```

### 方案 2：在 outputs.tf 中添加注释标记

```hcl
# 01-network/outputs.tf

# ⚠️ 下游引用：02-security, 03-ack, 04-database, 05-middleware, 06-web
output "vpc_id" {
  description = "VPC ID（供所有下游阶段引用）"
  value       = module.network.vpc_id
}

# ⚠️ 下游引用：04-database, 06-web
output "vswitch_ids_map" {
  description = "VSwitch ID 映射（供 04-database、06-web 引用）"
  value       = module.network.vswitch_ids_map
}
```

### 方案 3：terraform plan 前置检查

```bash
# 部署前检查上游 state 是否包含需要的 outputs
terraform plan -var-file="..." -var-file="..." 2>&1 | grep "does not have an attribute"
```

## 变更上游 output 时的检查流程

1. 搜索下游所有 `main.tf` 中对该 output 的引用
2. 确认没有下游依赖后再删除
3. 如果有，删除或重命名
3. 更新下游引用
4. 运行 `terraform plan` 验证

```bash
# 搜索所有引用
grep -r "remote_state.*outputs\.vpc_cidr_block" modules/declarative/
```

## 检查清单

- [ ] 每个声明阶段的 `main.tf` 引用的 `remote_state.outputs.xxx` 在上游 `outputs.tf` 中有定义
- [ ] 删除上游 output 前搜索下游引用
- [ ] outputs.tf 中标注下游引用范围（注释或 description）
- [ ] 考虑添加 pre-commit 钩子自动验证

## 参考资料

- `declarative-staged-configuration.md`（声明层分阶段配置）
- `layer-id-mapping.md`（ID 映射模式）
- `data-source-result-validation.md`（数据源结果验证）
