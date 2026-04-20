# 禁止硬编码密码：使用环境变量或 KMS 加密

**优先级：** 关键
**分类：** 安全最佳实践

## 为什么重要

在 `.tfvars` 或 `.tf` 文件中硬编码明文密码，会导致密码随代码仓库泄露。代码仓库的访问范围远大于基础设施的访问范围，一旦泄露，攻击者可直接获取数据库、中间件的管理密码。

## 问题：tfvars 中硬编码明文密码

```hcl
# ❌ 错误：terraform.tfvars 中硬编码密码
# declarative/simple/04-database/terraform.tfvars

rds_instances = {
  "mysql" = {
    master_password = "Caocao123!@#"          # 明文密码！
  }
}

redis_instances = {
  "cache" = {
    password        = "Redis@Classic#123"      # 明文密码！
    vpc_auth_mode   = "Open"                   # VPC 免密，更危险
  }
}

kafka_instances = {
  "producer" = {
    password = "KafkaProd123!"                 # 明文密码！
  }
}
```

**风险：**
1. 密码随 `git push` 进入版本控制，永久留存
2. 代码仓库的协作者、CI/CD 系统都能看到密码
3. 即使后续删除，git 历史中仍然存在
4. 违反安全合规要求（等保、SOC2 等）

## 解决方案

### 方案 1：环境变量注入（推荐测试环境）

```hcl
# tfvars 中使用变量占位
# declarative/simple/04-database/terraform.tfvars

rds_instances = {
  "mysql" = {
    master_password = var.rds_master_password   # 从环境变量读取
  }
}
```

```bash
# 执行时通过环境变量注入
export TF_VAR_rds_master_password="your-secure-password"
terraform apply -var-file="terraform.tfvars"
```

### 方案 2：KMS 加密密码（推荐生产环境）

```hcl
# 原子模块中使用 kms_encrypted_password
# atomic/rds/main.tf

resource "alicloud_db_instance" "this" {
  # 使用 KMS 加密的密码，Terraform state 中不存储明文
  kms_encrypted_password = var.kms_encrypted_password != "" ? var.kms_encrypted_password : null
}
```

```hcl
# tfvars 中使用 KMS 加密后的密文
rds_instances = {
  "mysql" = {
    kms_encrypted_password = "AQICAHxxxx..."   # KMS 加密密文
  }
}
```

### 方案 3：separate secrets 文件（.gitignore 排除）

```hcl
# secrets.tfvars（加入 .gitignore）
rds_master_password = "actual-password-here"
redis_password      = "actual-password-here"
```

```bash
# 执行时合并 tfvars
terraform apply \
  -var-file="terraform.tfvars" \
  -var-file="secrets.tfvars"
```

## VPC 免密模式风险

```hcl
# ❌ 危险：Redis VPC 免密模式
redis_instances = {
  "cache" = {
    vpc_auth_mode = "Open"    # VPC 内任何实例无需密码即可访问
  }
}
```

**风险链：** ECS 被入侵 → 攻击者利用 ECS 元数据获取 VPC 信息 → 无密码访问 Redis → 数据泄露

```hcl
# ✅ 正确：使用密码认证
redis_instances = {
  "cache" = {
    password      = var.redis_password   # 环境变量注入
    vpc_auth_mode = "Close"              # 强制密码认证
  }
}
```

## .gitignore 配置

```gitignore
# 敏感文件
*.tfstate
*.tfstate.backup
*secrets*.tfvars
*secret*.tfvars
.env
```

## 检查清单

- [ ] `.tfvars` 中无明文密码（搜索模式：`password\s*=\s*"[^var]"`）
- [ ] 密码通过环境变量 `TF_VAR_xxx` 或 KMS 加密注入
- [ ] `secrets*.tfvars` 已加入 `.gitignore`
- [ ] Redis/Tair 未使用 `vpc_auth_mode = "Open"`（生产环境）
- [ ] `*.tfstate.backup` 已加入 `.gitignore`（可能包含密码明文）
- [ ] `tls_private_key` 的 `private_key_pem` 未被输出或存储到可访问位置

## 参考资料

- `security-no-hardcoded-secrets.md`（禁止硬编码密钥）
- `security-credentials.md`（凭证管理规范）
- [Terraform Sensitive 变量](https://developer.hashicorp.com/terraform/language/values/variables#suppressing-values-in-cli-output)
