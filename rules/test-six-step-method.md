# 六步测试法 — 声明层完整测试流程

**优先级：** 高
**分类：** 测试与验证

> 适用范围：三层架构（声明层 → 控制层 → 原子层）的声明层环境测试

## 为什么重要

声明层是整个三层架构的最终配置入口。六步测试法提供系统化的测试流程，确保从代码审计到部署验证的每个环节都不遗漏，减少生产事故风险。

## 测试六步流程

### Step 1 — 规格一致性审计（不 apply，只读代码）

自上向下检查变量穿透：声明层 tfvars → 控制层 variables.tf → 原子层 variable + resource

检查项：
- 声明层的每个字段在控制层是否有对应定义（optional 或 required）
- 控制层的每个字段在原子层是否被正确引用
- 字段类型是否一致（string/number/bool/list/map）
- required 字段是否都有赋值

### Step 2 — 全 false 空跑

tfvars: 所有 `create = false`

```
terraform apply → 确认空跑无报错
```

目的：验证控制层 → 原子层的 `count = local.create ? 1 : 0` 逻辑正确，无依赖报错。

### Step 3 — 主资源开启（整层 apply）

tfvars: 主资源 `create = true` / 子资源 `create = false`

```
terraform apply → 创建所有主资源
terraform apply → 双 apply 确认 0 drift
```

双 apply 规则：**每次 apply 后必须再 apply 一次**，检测 Provider 读取与 API 实际状态不一致导致的漂移。

### Step 4 — 子资源逐个开启（逐产品 apply）

每开一个产品的子资源：

```
修改 tfvars 开启该子资源 → terraform apply → 验证成功 → terraform apply → 确认 0 drift
```

逐产品顺序示例：
1. Tair `create_backup_policy` → apply × 2
2. Tair `create_account` → apply × 2
3. NAS `create_access_group` + `access_rules` → apply × 2
4. Kafka `sasl_users` → apply × 2
5. Kafka `sasl_acls` → apply × 2
6. MySQL ECS `hbr_backup_enabled` → apply × 2

### Step 5 — 关联性验证（不 apply，查 state）

`terraform state show` 验证资源间引用关系：

- 网络：VSwitch 引用 ID、跨 AZ 部署是否正确
- 安全组：`security_group_key` → 控制层解析为实际安全组 ID
- LB 后端：`ecs_key` / `ecs_keys` → ECS 实例 ID 绑定
- 连接地址：域名、端口生成规则

### Step 6 — 整层销毁测试

```
terraform destroy  # 自上而下：Web → Middleware → Database
```

验证资源能完整销毁、无依赖阻塞。

---

## 双 Apply 规则

> **每次 apply 后必须再 apply 一次，确认 0 change。**

原因：Provider 在 `read` 阶段可能返回与 API 实际状态不一致的值（computed drift），例如：
- Tair Account 代理模式下 `instance_id` 返回 `-db-{N}` 后缀
- NAS computed 字段在首次创建后值不同
- 某些资源 creation_time / status 字段仅在第二次 read 时才稳定

如果第二次 apply 出现 drift，需要用 `lifecycle { ignore_changes = [...] }` 修复。

---

## Apply 节奏总结

```
全 false 空跑 → 主资源全 true（apply×2）→ 逐产品子资源（各 apply×2）→ 无漂移收尾 → 整层 destroy
```

## 每层测试顺序

按层级自上而下：Web(06) → Middleware(05) → Database(04) → Security(02) → ACK(03) → Network(01)

销毁也按此顺序，从上层开始销毁避免依赖阻塞。

## 参考资料

- `test-drift-detection.md`（部署后二次 Plan 验证）
- `test-strategies.md`（测试金字塔）
- `test-policy-as-code.md`（策略即代码）
