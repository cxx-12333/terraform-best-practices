# TypeSet 嵌套块 Computed-only 字段导致无限状态漂移

**优先级：** 高
**分类：** 常见陷阱 / 状态漂移

## 为什么重要

当 Provider 将嵌套块定义为 `schema.TypeSet` 且其内部包含 `Computed: true` only（无 `Optional`）的字段时，Terraform SDKv2 的 TypeSet hash 计算会包含这些不可配置字段，导致配置零值（`""`）与 API 返回值永远不等，产生**无限循环漂移**。

这类问题无法通过精简 dynamic content 解决，因为 TypeSet hash 基于 schema 全字段计算，不受 HCL content 赋值范围影响。

## 问题模式：TypeSet + Computed-only 嵌套字段

### 现象

```
每次 terraform plan 都检测到嵌套块变更：
~ servers {
  ~ server_group_id = "sgp-xxx" -> ""
  ~ status = "Available" -> ""
  ~ server_ip = "172.31.x.x" -> ""
}

apply 后再次 plan，仍然出现相同变更 → 无限循环！
```

### 根因链路

```
1. Provider schema 定义嵌套块为 TypeSet
↓
2. TypeSet hash 基于 schema 中所有字段计算（含 Computed only 字段）
↓
3. Computed only 字段（如 server_group_id、status）在 HCL 中不可赋值
↓
4. 配置中这些字段为零值（""），API 返回实际值（"sgp-xxx"、"Available"）
↓
5. 零值 ≠ API 返回值 → TypeSet hash 不匹配 → 判定元素变更
↓
6. apply 更新 state → 下次 plan 又检测到差异 → 无限循环漂移
```

### Provider 源码特征

```go
// 典型 TypeSet 嵌套块定义（含 Computed only 字段）
"servers": {
  Type: schema.TypeSet, // ← 关键：TypeSet
  Optional: true,
  Elem: &schema.Resource{
    Schema: map[string]*schema.Schema{
      // Computed only 字段 — 根因所在
      "status": {
        Type: schema.TypeString,
        Computed: true, // 无 Optional，HCL 不可赋值
      },
      "server_group_id": {
        Type: schema.TypeString,
        Computed: true, // 无 Optional，HCL 不可赋值
      },
      // Optional + Computed 字段 — 可能加剧漂移
      "server_ip": {
        Type: schema.TypeString,
        Optional: true,
        Computed: true, // 不赋值时 API 自动填充
      },
      // Required / Optional only 字段 — 正常
      "server_id": {
        Type: schema.TypeString,
        Required: true,
      },
    },
  },
}
```

## 判断标准

### 何时需要排查此问题

| 条件 | 说明 |
|------|------|
| 嵌套块定义为 `TypeSet`（非 `TypeList`） | TypeSet hash 包含所有 schema 字段 |
| 嵌套块内部存在 `Computed: true` only 字段 | HCL 不可赋值，零值必然与 API 返回值不同 |
| 使用 `dynamic` 块生成嵌套内容 | dynamic content 中无法控制 Computed only 字段 |

### TypeSet vs TypeList 对比

| 特性 | TypeSet | TypeList |
|------|---------|----------|
| 元素标识 | 基于 hash（包含所有 schema 字段） | 基于索引位置 |
| hash 计算范围 | schema 中**所有字段**（含 Computed only） | 同上，但对比逻辑不同 |
| 索引访问 | `Cannot index a set value` | 支持 `list[0]` |
| ignore_changes 粒度 | 只能忽略整个块 `ignore_changes = [block_name]` | 可忽略特定属性 `block_name[0].attr` |
| Computed only 字段影响 | **导致 hash 不匹配，无限漂移** | 通常不影响（按位置匹配） |

## 解决方案

### 唯一可行方案：ignore_changes 整个块

```hcl
resource "alicloud_alb_server_group" "this" {
  # ... 其他配置 ...

  dynamic "servers" {
    for_each = var.servers
    content {
      server_id = servers.value.server_id
      server_type = servers.value.server_type
      port = servers.value.port
      weight = lookup(servers.value, "weight", 100)
      description = lookup(servers.value, "description", null)
      # Computed only 字段（server_group_id、status）不赋值
      # Optional+Computed 字段（server_ip、remote_ip_enabled）不赋值
    }
  }

  # 忽略 servers 块整体变更（Provider TypeSet + Computed 字段漂移）
  # 变更 servers 配置时需临时注释此行，apply 后恢复
  lifecycle {
    ignore_changes = [
    servers,
    ]
  }
}
```

### 排除的方案及原因

| 方案 | 排除原因 | 验证方式 |
|------|---------|---------|
| `ignore_changes = [servers[0].server_ip]` | TypeSet 不支持索引访问，报 `Cannot index a set value` | terraform apply |
| 移除 content 中的 Optional+Computed 字段 | 不够——Computed only 字段（无 Optional）仍导致 hash 不匹配 | 第二次 plan 验证 |
| 在 content 中显式赋值 `server_ip = null` | 加剧漂移——null ≠ API 返回的实际 IP | plan 输出对比 |
| 修复 Provider（TypeSet→TypeList） | 不在 HCL 层面控制范围 | Provider 源码 |

### 副作用及应对

```
┌─────────────────────────────────────────────────────────────┐
│ servers 变更操作流程 │
├─────────────────────────────────────────────────────────────┤
│ │
│ 1. 注释 lifecycle { ignore_changes = [servers] } │
│ ↓ │
│ 2. 修改 dynamic content 或 var.servers │
│ ↓ │
│ 3. terraform plan 确认变更内容 │
│ ↓ │
│ 4. terraform apply │
│ ↓ │
│ 5. 恢复 lifecycle { ignore_changes = [servers] } │
│ ↓ │
│ 6. terraform plan 确认无额外漂移 │
│ │
└─────────────────────────────────────────────────────────────┘
```

## 排查路径总结（成功路径）

```
┌──────────────────────────────────────────────────────────────────────┐
│ TypeSet+Computed 漂移排查路径 │
├──────────────────────────────────────────────────────────────────────┤
│ │
│ Step 1: 确认漂移现象 │
│ ───────────────────── │
│ 第二次 terraform plan 仍检测到变更（非首次 apply 后的初始漂移） │
│ → 确认为无限循环漂移，非一次性问题 │
│ │
│ Step 2: Provider 源码逐字段分类 │
│ ────────────────────────────── │
│ 查看 resource_xxx.go 中嵌套块的 Schema 定义 │
│ 将每个字段标注为：Required / Optional only / Optional+Computed / │
│ Computed only │
│ → 识别出 Computed only 字段（根因候选） │
│ │
│ Step 3: 理解 TypeSet hash 机制 │
│ ───────────────────────────── │
│ Terraform SDKv2 TypeSet hash = schema 全字段参与计算 │
│ → Computed only 字段的零值 "" 必然 ≠ API 返回值 │
│ → 每次 plan 都判定元素变更 │
│ │
│ Step 4: 排除错误方案 │
│ ────────────────── │
│ 尝试 ignore_changes = [servers[0].xxx] → Cannot index a set │
│ 尝试移除 content 中 Optional+Computed 字段 → 仍漂移 │
│ → 确认 TypeSet hash 包含不可控的 Computed only 字段 │
│ │
│ Step 5: 确定最优解 │
│ ──────────────── │
│ ignore_changes = [servers] — 唯一可行方案 │
│ + 记录副作用：变更 servers 需临时注释 ignore_changes │
│ │
│ Step 6: 验证 │
│ ──────── │
│ terraform plan → No changes ✓ │
│ │
└──────────────────────────────────────────────────────────────────────┘
```

### 排查过程中的关键经验

1. **不要猜测，用源码实证**：每个字段的 Computed/Optional 属性必须从 Provider 源码确认，不能凭经验假设
2. **TypeSet 的 hash 机制是关键**：SDKv2 的 TypeSet hash 包含 schema 中所有字段，不仅限于 HCL content 中显式赋值的
3. **Computed only 字段是致命的**：无 `Optional` 的 `Computed: true` 字段在 HCL 中不可赋值，零值必然与 API 返回值不同
4. **逐步排除，不要跳步**：先尝试精确 ignore → 发现 TypeSet 不支持索引 → 尝试精简 content → 发现仍不够 → 确认只能 ignore 整个块

## 已知受影响资源

| 云厂商 | 资源 | 嵌套块 | Computed only 字段 |
|--------|------|--------|-------------------|
| 阿里云 | `alicloud_alb_server_group` | `servers` | `status`, `server_group_id` |

> 如发现新的受影响资源，请更新此表。

## 快速诊断检查清单

当 `dynamic` 嵌套块出现无限循环漂移时：

- [ ] 确认是否为第二次+ plan 仍检测到变更（排除首次 apply 的正常 state 回填）
- [ ] 查看 Provider 源码，确认嵌套块的 `Type` 是否为 `TypeSet`
- [ ] 逐字段标注 Computed/Optional 属性，识别 Computed only 字段
- [ ] 确认 TypeSet hash 包含 Computed only 字段（SDKv2 行为）
- [ ] 尝试 `ignore_changes = [block_name]`（整个块）
- [ ] 运行 `terraform plan` 验证漂移消除
- [ ] 记录副作用（变更时需临时注释 ignore_changes）

## 决策规则

> **当 Provider 将嵌套块定义为 TypeSet 且内部包含 Computed-only（无 Optional）字段时，HCL 层面无法规避漂移，必须使用 `ignore_changes = [block_name]` 忽略整个块。变更该块内容时需临时注释 ignore_changes。**

## 相关规则

- `provider-optional-api-mandatory.md` - Provider 参数 Schema 类型分类（顶层字段）
- `resource-lifecycle.md` - 生命周期块使用规范
- `test-drift-detection.md` - 部署验证漂移检测流程
- `resource-zoneid-format-drift.md` - API 返回值格式不一致漂移（不同根因）

## 参考资料

- `resource-lifecycle.md`（生命周期块）
- `resource-zoneid-format-drift.md`（ZoneId 格式漂移）
- [Terraform TypeSet 行为](https://developer.hashicorp.com/terraform/cli/state/resource-addressing)
