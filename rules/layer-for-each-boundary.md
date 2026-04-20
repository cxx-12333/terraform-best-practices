# 原子层 for_each 边界：仅限同一主资源子组件

**优先级：** 高
**分类：** 三层架构

## 为什么重要

原子层的职责是"封装一种资源类型"。`for_each` 是编排逻辑，理论上属于控制层。但某些场景下，同一主资源有多个子组件（如 ALB 的多个 Rule），在原子层内使用 `for_each` 处理这些子组件是合理的。需要明确边界，防止 for_each 被滥用。

## 原子层 for_each 边界定义

### 允许：同一主资源的子组件

```hcl
# ✅ 允许：ALB Rule 是 ALB Listener 的子组件，与主资源生命周期绑定
# atomic/alb-rule/main.tf

resource "alicloud_alb_rule" "this" {
  count = local.create ? 1 : 0

  listener_id = var.listener_id    # 依赖父资源 ID
  rule_name   = var.rule_name
  # ... 单个 Rule 的配置
}
```

**判断标准：**
1. 子组件不能独立存在（必须有父资源 ID）
2. 子组件与父资源是 1:N 关系（一个 Listener 有多个 Rule）
3. 子组件的生命周期跟随父资源（父资源删除，子组件自动删除）

### 允许：同一资源的可变数量配置项

```hcl
# ✅ 允许：Security Group 的多条规则是同一资源的配置项
# atomic/security-group/main.tf

resource "alicloud_security_group_rule" "ingress" {
  count = local.create ? length(var.ingress_rules) : 0

  security_group_id = alicloud_security_group.this[0].id
  # ... 规则配置
}
```

### 禁止：跨资源的编排逻辑

```hcl
# ❌ 禁止：原子层不应包含多资源的编排逻辑
# atomic/web-stack/main.tf（假设的错误示例）

resource "alicloud_alb" "this" {
  count = local.create ? 1 : 0
}

resource "alicloud_alb_listener" "this" {
  count = local.create ? 1 : 0
  load_balancer_id = alicloud_alb.this[0].id
}

# ❌ 这应该在控制层用 for_each + module 调用来编排
```

## 决策矩阵

| 场景 | for_each 位置 | 说明 |
|------|---------------|------|
| ALB 有多个 Rule | **控制层** for_each 调用 `module alb-rule` | Rule 是独立模块，控制层编排数量 |
| Security Group 有多条规则 | **原子层** count 遍历规则列表 | 规则是同一资源的配置项 |
| SLB 有多个 Backend | **控制层** for_each 调用 `module slb-backend-attachment` | Backend 是独立资源 |
| ECS 有多块数据盘 | **原子层** dynamic block 或 count | 数据盘是 ECS 的子组件 |
| RDS 有多个数据库 | **控制层** for_each 调用 `module rds-database` | 数据库是独立资源 |

## 当前项目中的边界实践

### 符合边界的设计

```
atomic/alb/                 → 单个 ALB 实例（无 for_each）
atomic/alb-listener/        → 单个 Listener（无 for_each，控制层 for_each 编排）
atomic/alb-rule/            → 单个 Rule（无 for_each，控制层 for_each 编排）
atomic/alb-server-group/    → 单个 Server Group（无 for_each）
atomic/security-group/      → 单个 SG + count 遍历规则列表 ✅ 原子层 for_each 允许
atomic/ecs/                 → 单个 ECS + dynamic block 遍历数据盘 ✅ 允许
```

### 需要注意的设计

```
atomic/alb/main.tf:253      → ALB 的 load_balancer_id 转发（检查是否为同一资源子组件）
atomic/slb/main.tf:154-268  → SLB Server Group Attachment（已拆为独立模块 ✅）
```

## 拆分指南

当原子层模块包含多种资源类型的 `for_each` 时，应拆分为独立原子模块：

```
# 拆分前（违反边界）
atomic/alb/  → 包含 ALB + Listener + Rule + Server Group

# 拆分后（符合边界）
atomic/alb/                → 仅 ALB 实例
atomic/alb-listener/       → 仅 Listener
atomic/alb-rule/           → 仅 Rule
atomic/alb-server-group/   → 仅 Server Group
```

**控制层负责编排它们之间的关系：**

```hcl
# control/web-cluster/main.tf
module "alb" { for_each = var.albs ... }
module "alb_listener" { for_each = var.alb_listeners ... }
module "alb_rule" { for_each = var.alb_rules ... }
```

## 检查清单

- [ ] 原子层模块只封装一种主资源类型
- [ ] 原子层中的 `for_each`/`count` 仅用于同一主资源的子组件（如 SG 规则、ECS 数据盘）
- [ ] 跨资源的 `for_each` 编排在控制层完成
- [ ] 可独立存在/可独立删除的子资源已拆分为独立原子模块

## 参考资料

- `layer-separation.md`（三层职责划分）
- `control-nested-resource-flatten.md`（嵌套资源展平）
- `control-backup-prefilter.md`（条件子资源预过滤）
