# 控制层 ID 映射命名一致性

**优先级：** 中
**分类：** 模块设计

## 为什么重要

控制层的 6 个模块都需要构建 ID 映射（`xxx_ids_map`），用于将逻辑 Key 转换为实际资源 ID。如果命名不一致（如 `sg_ids` vs `sg_ids_map`），会导致：
1. 声明层引用时混淆
2. 新开发者难以理解命名规律
3. 跨模块引用时需要记忆不同的命名

## 问题：命名不一致

```hcl
# ❌ 错误：不同控制层模块使用不同的命名风格

# control/security/locals.tf
locals {
  sg_ids = { ... }              # 缺少 _map 后缀
}

# control/security/outputs.tf
output "security_group_ids_map" {  # 有 _map 后缀
  value = local.sg_ids            # 与 locals 命名不一致
}

# control/web-cluster/locals.tf
locals {
  alb_ids_map = { ... }          # 有 _map 后缀
  slb_ids_map = { ... }          # 有 _map 后缀
}
```

**问题：** `sg_ids`（locals）vs `security_group_ids_map`（output）→ 引用者需要记住两套命名。

## 正确：统一命名规范

### 命名规范

| 位置 | 命名格式 | 示例 |
|------|----------|------|
| 控制层 locals | `xxx_ids_map` | `alb_ids_map`, `sg_ids_map`, `slb_ids_map` |
| 控制层 outputs | `xxx_ids_map` | `alb_ids_map`, `security_group_ids_map` |
| 声明层 outputs | `xxx_ids_map` | `vswitch_ids_map`, `security_group_ids_map` |
| 声明层 remote_state | `outputs.xxx_ids_map` | `data.terraform_remote_state.network.outputs.vswitch_ids_map` |

### 统一模式

```hcl
# ✅ 正确：所有层级统一使用 xxx_ids_map 后缀

# control/security/locals.tf
locals {
  sg_ids_map = {                   # ✅ _ids_map 后缀
    for k, v in module.security_group : k => v.sg_id
    if v.sg_id != null
  }
}

# control/security/outputs.tf
output "security_group_ids_map" {   # ✅ 与 locals 一致
  value = local.sg_ids_map
}
```

### ID 映射构建模式

所有控制层模块的 ID 映射遵循统一模式：

```hcl
# 标准模式：过滤 null + 使用 module 输出
locals {
  xxx_ids_map = {
    for k, v in module.xxx : k => v.xxx_id
    if v.xxx_id != null           # 过滤未创建的实例
  }
}
```

## 常见 ID 映射对照表

| 资源 | locals 变量名 | output 名 | 来源 |
|------|---------------|-----------|------|
| ALB | `alb_ids_map` | `alb_ids_map` | `module.alb` |
| SLB | `slb_ids_map` | `slb_ids_map` | `module.slb` |
| NLB | `nlb_ids_map` | `nlb_ids_map` | `module.nlb` |
| ECS | `ecs_ids_map` | `ecs_ids_map` | `module.ecs` |
| 安全组 | `sg_ids_map` | `security_group_ids_map` | `module.security_group` |
| VSwitch | — | `vswitch_ids_map` | `module.vswitches`（声明层直出） |

## 检查清单

- [ ] 所有 ID 映射 locals 使用 `xxx_ids_map` 后缀
- [ ] 所有 ID 映射 outputs 与对应 locals 命名一致
- [ ] ID 映射构建统一使用 `{ for k, v in ... : k => v.id if v.id != null }` 模式
- [ ] 无裸露的 `map[key]` 访问（使用 `lookup()` 保护）

## 参考资料

- `control-key-mapping-pattern.md`（Key 映射模式）
- `layer-id-mapping.md`（ID 映射模式）
- `control-parameter-passthrough.md`（参数透传模式）
