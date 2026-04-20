# 如何新增规则

**优先级：** 中
**分类：** 元规则（技能维护）

## 为什么重要

新增规则时遵循统一的格式和流程，确保技能体系的一致性和可维护性。本指南总结了已有 63+ 条规则中提炼的最佳实践。

## 规则文件模板

创建新规则文件时，使用以下模板：

```markdown
# [中文标题，简洁明了]

**优先级：** [关键/高/中高/中/低中/低]
**分类：** [对应分类名]

## 为什么重要

[1-3 句话说明为什么这条规则重要，不遵守会有什么后果]

## 错误示例

[如果适用] 展示违反规则的代码和问题说明

\```hcl
# 违反规则的代码
\```

**问题：**
- [具体问题 1]
- [具体问题 2]

## 正确示例

[展示遵循规则的代码和解释]

\```hcl
# 遵循规则的代码
\```

## 参考资料

- [相关官方文档链接]
- [相关规则文件引用]
```

## 文件命名规范

### 格式

```
{分类前缀}-{简短描述}.md
```

### 分类前缀对照表

| 分类 | 前缀 | 示例 |
|------|------|------|
| 组织与工作流 | `org-` | `org-version-control.md` |
| 状态管理 | `state-` | `state-remote-backend.md` |
| 安全最佳实践 | `security-` | `security-credentials.md` |
| 三层架构 | `layer-` 或混合前缀 | `layer-separation.md` |
| 测试与验证 | `test-` | `test-strategies.md` |
| 模块设计 | `module-` | `module-versioning.md` |
| 资源组织 | `resource-` | `resource-naming.md` |
| 变量与输出模式 | `variable-` / `output-` | `variable-types.md` |
| 语言最佳实践 | `language-` | `language-locals.md` |
| Provider 配置 | `provider-` | `provider-version-constraints.md` |
| 性能优化 | `perf-` | `perf-parallelism.md` |
| 元规则 | `meta-` | `meta-how-to-add-rules.md` |

### 命名规则

- 使用小写字母和连字符（kebab-case）
- 名称应简短但能说明规则内容
- 避免使用缩写，除非是公认的（如 `vpc`、`eks`）

## 优先级选择指南

选择优先级时，问自己："不遵守这条规则，最坏会发生什么？"

| 优先级 | 判断标准 | 示例 |
|--------|----------|------|
| **关键** | 安全事故、数据丢失、生产故障 | 硬编码密钥、无状态锁定 |
| **高** | 部署失败、状态损坏、重大损失 | 模块无版本、ForceNew 漂移 |
| **中高** | 可维护性下降、运维成本上升 | count 代替 for_each、无标签 |
| **中** | 代码质量下降、可读性变差 | 无变量描述、硬编码数据源 |
| **低中** | 性能次优、工具配置不完善 | 并行度未调优 |
| **低** | 可选的最佳实践 | 策略即代码 |

## 分类选择指南

### 决策流程

```
新规则是关于什么的？
│
├─ 团队协作流程？ → 组织与工作流 (org-)
├─ 状态文件管理？ → 状态管理 (state-)
├─ 安全/密钥/权限？ → 安全最佳实践 (security-)
├─ 三层架构设计？ → 三层架构 (layer-/混合)
├─ 测试/验证/策略？ → 测试与验证 (test-)
├─ 模块设计/复用？ → 模块设计 (module-)
├─ 资源管理/组织？ → 资源组织 (resource-)
├─ 变量/输出定义？ → 变量与输出模式 (variable-/output-)
├─ HCL 语言特性？ → 语言最佳实践 (language-)
├─ Provider 配置？ → Provider 配置 (provider-)
├─ 执行性能？ → 性能优化 (perf-)
└─ 技能体系本身？ → 元规则 (meta-)
```

## 代码示例编写指南

### 好的代码示例

```hcl
# 好的示例：完整、有上下文、能直接理解
variable "environment" {
  type        = string
  description = "部署环境"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "环境必须是 dev、staging 或 prod。"
  }
}
```

### 避免的写法

```hcl
# 不好的示例：片段、无上下文、需要猜测
validation {
  condition = contains([...], var.x)
}
```

### 代码示例原则

1. **完整可理解**：不需要额外上下文就能理解示例
2. **使用真实场景**：用多云资源（aws_instance、azurerm_mysql_flexible_server、tencentcloud_mysql_instance 等）
3. **注释说明意图**：在关键行加注释说明为什么这样做
4. **不要用省略号**：避免 `# ... 还有 200 行` 这种写法，除非是在说明代码量
5. **HCL 缩进 2 空格**：遵循 `terraform fmt` 标准

## 新增规则的完整流程

### 步骤 1：确认必要性

- [ ] 搜索现有规则，确认没有覆盖相同内容
- [ ] 确认该规则在实际项目中发生过问题
- [ ] 确认该规则有明确的"正确做法"

### 步骤 2：创建文件

- [ ] 使用模板创建 `{前缀}-{名称}.md`
- [ ] 填写标题、优先级、分类
- [ ] 编写"为什么重要"
- [ ] 编写错误示例（如适用）
- [ ] 编写正确示例
- [ ] 添加参考资料链接

### 步骤 3：更新索引

在 `SKILL.md` 中：

1. 找到对应的分类段落
2. 按逻辑顺序插入新规则条目
3. 更新该分类的规则计数
4. 更新顶部的总规则数

### 步骤 4：验证

```bash
# 检查文件格式
head -5 rules/{新文件名}.md

# 检查 SKILL.md 中的引用
grep "{新文件名}" SKILL.md

# 检查总数量
echo "SKILL.md 声明: $(grep -oP '\d+(?= 条规则)' SKILL.md | head -1)"
echo "实际文件数: $(ls rules/*.md | wc -l)"
```

## 常见错误

| 错误 | 说明 | 正确做法 |
|------|------|----------|
| 无优先级 | 省略 `**优先级：**` 行 | 必须填写 |
| 无分类 | 省略 `**分类：**` 行 | 必须填写 |
| 英文标题 | `## Why It Matters` | 使用 `## 为什么重要` |
| 过于笼统 | "不要写糟糕的代码" | 具体到可操作的规则 |
| 无代码示例 | 纯文字描述 | 配合 HCL 代码示例 |
| 示例不可执行 | 语法错误、缺少变量定义 | 确保 `terraform validate` 通过 |
| 未更新索引 | 创建文件但忘了更新 SKILL.md | 同步更新 |

## 参考资料

- SKILL.md（技能索引文件）
- `meta-skill-review.md`（评审方法）
- [Terraform 风格指南](https://developer.hashicorp.com/terraform/language/style)
