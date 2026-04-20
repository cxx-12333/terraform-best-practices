---
name: terraform-best-practices
description: Terraform 与基础设施即代码优化指南（多云通用）。适用于编写、审查或重构 Terraform/OpenTofu 代码，确保安全性、可维护性和可靠性。触发于 Terraform 模块、基础设施配置、状态管理或 IaC 优化任务。包含三层架构（原子层/控制层/声明层）专用规则。多云示例覆盖 AWS、Azure、GCP、阿里云。
license: MIT
metadata:
  author: terramate + tf-architecture
  version: "2.4.0"
---

# Terraform 最佳实践

Terraform 与基础设施即代码综合优化指南（多云通用）。包含 81 条规则，分为 12 个类别，按影响程度排序，指导自动化重构和代码生成。

## 适用场景

在以下情况参考本指南：
- 编写新的 Terraform 模块或配置
- 实现基础设施模式（AWS、Azure、GCP、阿里云等多云）
- 审查代码的安全性和可靠性问题
- 重构现有的 Terraform/OpenTofu 代码
- 优化状态管理和性能
- 建立团队工作流和治理规范
- 构建三层 IaC 架构（原子层/控制层/声明层）

## 规则分类（按优先级）

| 优先级 | 分类 | 影响程度 | 前缀 |
|--------|------|----------|------|
| 1 | 组织与工作流 | 关键 | `org-` |
| 2 | 状态管理 | 关键 | `state-` |
| 3 | 安全最佳实践 | 关键 | `security-` |
| 4 | 模块设计 | 高 | `module-`, `control-` |
| 5 | 资源组织 | 中高 | `resource-` |
| 6 | 变量与输出模式 | 中 | `variable-`, `output-` |
| 7 | 语言最佳实践 | 中 | `language-` |
| 8 | Provider 配置 | 中 | `provider-` |
| 9 | 性能优化 | 低中 | `perf-` |
| 10 | 测试与验证 | 关键 | `test-` |
| 11 | 三层架构 | 关键 | `layer-` |
| 12 | 元规则 | 中 | `meta-` |

## 快速参考

### 1. 组织与工作流（关键）- 5 条规则

- `org-version-control` - 所有 Terraform 代码纳入版本控制
- `org-workspaces` - 每个环境每个配置使用独立工作区
- `org-access-control` - 控制谁可以更改什么基础设施
- `org-change-workflow` - 基础设施变更的正式流程
- `org-audit-logging` - 追踪所有基础设施变更

### 2. 状态管理（关键）- 6 条规则

- `state-remote-backend` - 始终使用远程状态后端
- `state-locking` - 启用状态锁定防止损坏
- `state-import` - 将现有基础设施导入 Terraform
- `state-analysis-vs-rebuild` - State 分析仅作参考，重写配置需依据官方文档
- `import-component-resource` - 子组件型资源必须使用专用资源类型 import
- `resource-zoneid-format-drift` - ZoneId 格式漂移问题处理

### 3. 安全最佳实践（关键）- 4 条规则

- `security-no-hardcoded-secrets` - 永远不要在代码中硬编码密钥
- `security-no-hardcoded-passwords` - 禁止在 .tfvars 中硬编码密码，使用环境变量或 KMS（关键）
- `security-credentials` - 使用正确的凭证管理（OIDC、Vault、IAM 角色）
- `security-iam-least-privilege` - 遵循最小权限原则

### 4. 模块设计（高）- 14 条规则

- `module-single-responsibility` - 每个模块负责一个逻辑组件
- `module-naming` - 使用一致的命名规范（terraform-<PROVIDER>-<NAME>）
- `module-versioning` - 所有模块引用都要指定版本
- `module-composition` - 像搭积木一样组合模块
- `module-registry` - 使用现有的社区/共享模块
- `module-parameter-completeness` - 原子层模块参数完整性检查（覆盖 Provider Schema）
- `openapi-reference-lookup` - 阿里云 OpenAPI 与 Provider 文档查找规范（从源码反查文档地址）
- `module-dependency-prefilter` - 模块依赖预过滤：避免引用未创建的模块实例（关键）
- `module-null-safe-dependency` - 控制层 null-safe 依赖保障：父资源未创建时子资源自动安全降级
- `control-layer-map-lookup` - 控制层所有 map 访问必须使用 lookup() 保护（关键）
- `control-layer-dry-id-mapping` - 控制层 ID 映射命名一致性规范
- `control-nested-resource-flatten` - 控制层嵌套资源 for_each 展平模式（一对多关系安全编排）
- `control-key-mapping-pattern` - 控制层安全组/子网 Key 映射模式（逻辑名 → ID 解析）
- `control-backup-prefilter` - 控制层条件子资源预过滤模式（备份/快照等附加资源安全过滤）

### 5. 资源组织（中高）- 10 条规则（含多云特定补充）

- `resource-naming` - 使用一致的命名规范
- `resource-tagging` - 为所有资源打标签便于成本追踪
- `resource-lifecycle` - 使用生命周期块（prevent_destroy、ignore_changes）
- `resource-count-vs-foreach` - 优先使用 for_each 而非 count
- `resource-immutable` - 优先使用不可变基础设施模式
- `resource-spec-auto-correction` - 规格参数自动对齐防漂移
- `resource-foreach-unknown-value` - for_each 与 unknown 值：dynamic 块优先策略（关键）
- `resource-batch-dependency-serialization` - 同类资源批次依赖串行化：count/for_each 实例间无法建依赖时的批次分组方案（关键）
- `api-parameter-character-restrictions` - 云 API 参数字符限制处理规范
- `resource-typeset-computed-drift` - TypeSet 嵌套块 Computed-only 字段无限漂移问题

### 6. 变量与输出模式（中）- 12 条规则

- `variable-types` - 使用具体类型、正向命名、可空性
- `variable-validation` - 添加验证规则实现早期错误检测
- `variable-sensitive` - 将敏感信息标记为 sensitive，不设默认值
- `variable-descriptions` - 为所有变量添加描述文档
- `variable-forcenew-defaults` - ForceNew 参数：空值默认规则
- `variable-optional-collection-defaults` - optional 集合类型必须指定默认值（关键）
- `variable-layered-type-design` - 三层变量类型设计模式（关键：声明层 any → 控制层 map(object) + optional）
- `variable-auto-naming` - 三层架构自动命名机制（声明层省略 → 控制层自动生成）
- `output-descriptions` - 为所有输出添加描述文档
- `output-no-secrets` - 永远不要直接输出敏感信息
- `output-null-default` - 原子层 outputs 默认值：使用 null 而非空字符串
- `comment-standardization` - 参数差异化注释，简洁可操作

### 7. 语言最佳实践（中）- 5 条规则

- `language-no-heredoc-json` - 使用 jsonencode/yamlencode，而非 HEREDOC
- `language-locals` - 使用 locals 命名复杂表达式
- `language-linting` - 运行 terraform fmt 和 tflint
- `language-data-sources` - 使用数据源而非硬编码
- `language-dynamic-blocks` - 使用 dynamic 块实现 DRY 代码

### 8. Provider 配置（中）- 3 条规则

- `provider-version-constraints` - 锁定 Provider 版本
- `provider-optional-api-mandatory` - Provider 可选但 OpenAPI 必填参数设置有效默认值（多云特定）
- `provider-documentation-lookup` - Provider 文档与 API 查找规范（多云源码反查流程）

### 9. 性能优化（低中）- 2 条规则

- `perf-parallelism` - 为大型部署调整并行度
- `perf-debug` - 启用调试日志进行故障排查

### 10. 测试与验证（关键）- 4 条规则

- `test-drift-detection` - 部署后二次 Plan 验证（关键）
- `test-strategies` - 测试金字塔（validate、lint、plan、集成测试）
- `test-policy-as-code` - 实施策略检查（OPA、Checkov、tfsec）
- `test-six-step-method` - 六步测试法（声明层完整测试流程）

### 11. 三层架构（关键）- 14 条规则

- `layer-separation` - 原子层/控制层/声明层职责划分
- `layer-pure-declarative` - 声明层禁止包含 resource 块，只允许 module + data（关键）
- `layer-no-business-defaults` - 控制层不得设置业务默认值，区分技术默认 vs 业务默认（关键）
- `layer-for-each-boundary` - 原子层 for_each 边界：仅限同一主资源子组件（高）
- `layer-comment-standard` - 三层注释规范（Provider Schema 表格 + 职责域树 + 参数联动表格）
- `layer-id-mapping` - 通过 map 变量实现基于 key 的资源引用
- `layer-three-question` - 三问法确定资源归属
- `variable-atomic-defaults` - 原子层默认值：技术纯度原则
- `declaration-layer-configuration-patterns` - 声明层配置实践规范（参数排列、注释规范、多子类型模式）
- `declarative-staged-configuration` - 声明层分阶段配置与共享参数体系（Stage 目录 + shared.tfvars）
- `control-parameter-passthrough` - 控制层参数透传与业务默认值（四类透传模式）
- `code-reference-documentation` - 原子层代码添加官方文档参考链接
- `cross-stage-reference-validation` - 跨阶段引用完整性验证（中）
- `data-source-result-validation` - 数据源查询结果 precondition 验证（中）

### 12. 元规则（中）- 2 条规则

- `meta-how-to-add-rules` - 如何系统性地新增规则（模板、命名、优先级选择）
- `meta-skill-review` - 技能体系评审方法（完整性、格式、质量、优先级检查）

## 使用方法

阅读各规则文件获取详细说明和代码示例：

```
rules/state-remote-backend.md
rules/security-no-hardcoded-secrets.md
rules/module-versioning.md
```

每个规则文件包含：
- 为什么重要的简要说明
- 错误示例及解释
- 正确示例及解释
- 额外的上下文和参考资料

## 完整文档

所有规则展开的完整指南：`AGENTS.md`