# API 参数字符限制处理规范

**优先级：** 高
**分类：** 云 API 兼容性

## 为什么重要

云产品 API 对某些参数有严格的字符限制（如不允许空格、横杠等），但 Terraform 自动生成的命名可能包含这些字符。如果不处理，会导致 `IllegalCharacters` 错误。

## 已知限制

### 1. NAS Access Group — Description 不允许空格和横杠

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 产品                        │ NAS（所有类型：standard/extreme/cpfs）          │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 参数                        │ Description                                   │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 限制                        │ 仅允许字母、数字、下划线（不允许空格、横杠）      │
├────────────────────────────┼───────────────────────────────────────────────┤
│ AccessGroupName            │ 允许字母、数字、下划线、横杠                    │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 影响范围                    │ CreateAccessGroup / ModifyAccessGroup          │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 2. NAS Access Rule — 需要 file_system_type 关联

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ Access Rule 必须传 file_system_type 才能找到    │
│                             │ 对应类型的 Access Group                        │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 错误                        │ InvalidAccessGroup.NotFound (404)              │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 原因                        │ 不传 file_system_type 时，Provider 按 default  │
│                             │ 类型查找，找不到 extreme 类型的 Access Group    │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 3. Tair Account — instance_id 代理模式漂移

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ 代理模式下 Provider 读取 account 返回的         │
│                             │ instance_id 带后缀 "-db-{N}"（节点 ID）         │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 错误                        │ 每次 plan 都触发 replacement                   │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 修复                        │ lifecycle { ignore_changes = [instance_id] }   │
└────────────────────────────┴───────────────────────────────────────────────┘
```

### 4. NAS effective_access_group_name — try() 安全访问

```
┌────────────────────────────┬───────────────────────────────────────────────┐
│ 问题                        │ access_group 未创建时，引用 this[0] 会报        │
│                             │ Invalid index（空列表）                         │
├────────────────────────────┼───────────────────────────────────────────────┤
│ 修复                        │ try(alicloud_nas_access_group.this[0].name,    │
│                             │     var.access_group_name)                     │
└────────────────────────────┴───────────────────────────────────────────────┘
```

## 通用修复模式

### 模式 1：字符净化（replace）

```hcl
# 原子层：将横杠替换为下划线，确保 API 兼容
description = "${replace(var.nas_name, "-", "_")}_access_group"
# 结果: "nas_simple_02_access_group" (无横杠、无空格)
```

### 模式 2：补充关联参数

```hcl
# 子资源必须传递父资源的类型参数，确保 API 查找正确
resource "alicloud_nas_access_rule" "this" {
  access_group_name = alicloud_nas_access_group.this[0].access_group_name
  file_system_type  = var.file_system_type != "" ? var.file_system_type : null  # 关键！
  ...
}
```

### 模式 3：ignore_changes 处理 Provider 漂移

```hcl
# Provider 读取的值与创建时传入的值不一致（Computed 差异）
resource "alicloud_kvstore_account" "this" {
  instance_id = alicloud_redis_tair_instance.this[0].id
  ...
  lifecycle {
    ignore_changes = [instance_id]  # 防止 "-db-{N}" 后缀导致无限漂移
  }
}
```

### 模式 4：try() 安全引用

```hcl
# 条件创建的资源，引用前用 try() 防止空列表索引错误
effective_name = local.create_sub_resource 
  ? try(parent_resource.this[0].name, var.default_name) 
  : var.default_name
```

## 调试方法论

1. **CLI 对比验证**：先用 aliyun CLI 测试 API 调用，排除 Provider 问题
2. **逐步排除参数**：逐个参数测试，确定哪个参数导致 IllegalCharacters
3. **Provider TRACE 日志**：`TF_LOG=DEBUG` 查看 Provider 实际发送的 API 请求
4. **Skills 三合一**：结合官方 API 文档 + Provider 源码 + 实际测试

## 发现时间

- 2026-04-17: NAS Access Group Description 限制 + Access Rule file_system_type 缺失
- 2026-04-17: Tair Account instance_id 代理模式漂移
- 2026-04-17: NAS effective_access_group_name try() 修复

## 参考资料

- `provider-documentation-lookup.md`（Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [阿里云 OpenAPI 调试台](https://next.api.aliyun.com/)
