# Provider 文档与 API 查找规范

**优先级：** 中
**分类：** Provider 配置

> 从 Provider 源码和 API 文档中提取参数定义，确保原子层模块参数完整、默认值安全、ForceNew 行为正确。

## 为什么重要

在原子层参数审计、模块开发、参数约束确认时，需要查阅：
1. **API 文档**：确认参数必填规则、取值范围、联动约束
2. **Provider 文档**：确认 Terraform 参数映射、默认值、ForceNew 行为
3. **Provider 源码**：最权威的参数定义和 API 调用逻辑

如果不知道如何系统性地查找这些文档，会导致：
- 凭猜测设置默认值，引发运行时错误
- 遗漏 API 级约束（条件必填、互斥参数）
- 无法确认参数是否为 ForceNew

## 通用查找流程

```
┌─────────────────────────────────────────────────────────────────┐
│              Provider / API 文档查找流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定位 Provider 源码                                          │
│     terraform-provider-{provider}/                              │
│                                                                 │
│  2. 提取关键信息                                                │
│     ┌───────────────────────────────────────────────────────┐   │
│     │ 信息            │ 提取位置            │ 说明         │   │
│     ├───────────────────────────────────────────────────────┤   │
│     │ API Service     │ API 调用参数         │ 产品标识     │   │
│     │ API Version     │ API 调用参数         │ API 版本     │   │
│     │ Action Name     │ API 调用逻辑        │ 操作名       │   │
│     │ Schema 字段     │ Schema 定义块       │ 参数约束     │   │
│     └───────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 构造文档 URL                                                │
│     ┌───────────────────────────────────────────────────────┐   │
│     │ API 文档  → 各云厂商 API 文档中心                      │   │
│     │ Provider  → Terraform Registry                         │   │
│     └───────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 交叉验证                                                    │
│     Provider Schema ↔ API 文档 ↔ 实际行为                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 通用 Provider 源码关键位置

### 1. Schema 定义（参数完整性审计入口）

```go
// 所有 Provider 的 Schema 结构一致
Schema: map[string]*schema.Schema{
  "param_name": {
    Type:     schema.TypeString,
    Optional: true,   // 可选
    Required: true,   // 必填
    Computed: true,   // API 返回
    ForceNew: true,   // 修改触发重建
    Default:  "val",  // Provider 内置默认值
  },
}
```

### 2. Create 函数（API 调用逻辑）

```go
// 各厂商 Provider 的 Create 函数结构类似
// 提取 API Service、Version、Action 信息
action := "CreateXxx"   // API Action 名
// API 调用...
```

### 3. Update 函数（参数更新行为）

```go
// 关键：d.HasChange("param") 判断哪些参数变更触发 Update
if d.HasChange("param_name") {
  update = true
}
```

### 4. 数据源定义（查询类资源）

```go
// 数据源用于查询已有资源
// 各厂商命名：dataSource_{provider}_{resource}.go
```

## 审计标准操作步骤

### Step 1: 定位 Provider 源码

| 云厂商 | Provider 源码路径 |
|--------|-------------------|
| 阿里云 | `terraform-provider-alicloud/alicloud/resource_alicloud_*.go` |
| AWS | `terraform-provider-aws/internal/service/*/<resource>.go` |
| 腾讯云 | `terraform-provider-tencentcloud/tencentcloud/services/*/<resource>.go` |
| Azure | `terraform-provider-azurerm/internal/services/*/<resource>.go` |

### Step 2: 提取 Schema 信息

对照 `module-parameter-completeness` 规则，提取所有 Optional/Required 字段。

### Step 3: 查阅 API 文档

从 Provider 源码提取 Service + Version + Action，在对应云厂商 API 文档中心查找。

### Step 4: 查阅 Provider 文档

在 Terraform Registry 查找对应 Provider 的文档页面。

### Step 5: 交叉验证

| 验证项 | 方法 |
|--------|------|
| 参数是否存在 | Provider Schema ↔ API 参数列表 |
| 默认值是否安全 | Provider Default ↔ API 文档描述 |
| ForceNew 是否正确 | Provider ForceNew ↔ API 重建行为 |
| 条件必填 | API 文档 ↔ 原子层 validation |

---

## 附录 A：阿里云（Alibaba Cloud）

### Provider 文档 URL

```
https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}
```

**规则**：去掉 `alicloud_` 前缀即为 URL 路径

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `alicloud_alb_load_balancer` | `alb_load_balancer` |
| `alicloud_wafv3_instance` | `wafv3_instance` |

### OpenAPI 文档

**可靠查找路径**（按优先级排序）：

1. **OpenAPI 在线调试台**（最可靠）：`https://next.api.aliyun.com/api/{service}/{version}/{action}`
2. **帮助中心搜索**：`https://help.aliyun.com/zh/` → 搜索产品名 → 开发者参考
3. **API 目录页**：`https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-dir/`
4. **Provider 文档**：`https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}`

> **经验教训**：阿里云 OpenAPI 文档 URL 经常变动，精确 URL 大概率 404！请优先使用在线调试台。

### Provider 源码关键模式

```go
// API 调用格式
response, err = client.RpcPost("waf-openapi", "2021-10-01", action, nil, request, false)
//                       ^^^^^^^^^^^^  ^^^^^^^^^^^
//                       Service Name  API Version

// Action Name → API 文档 URL：PascalCase → kebab-case，前面加 api-
// CreatePostpaidInstance → api-waf-openapi-2021-10-01-createpostpaidinstance
```

### Service Name 到 product-slug 映射表

| Provider Service Name | product-slug | 帮助中心路径 |
|----------------------|-------------|-------------|
| `waf-openapi` | waf/web-application-firewall-3-0 | `help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/` |
| `cas` | ssl-certificates-service | `help.aliyun.com/zh/ssl-certificates-service/developer-reference/` |
| `Alb` | alb | `help.aliyun.com/zh/alb/developer-reference/` |
| `Ecs` | ecs | `help.aliyun.com/zh/ecs/developer-reference/` |
| `Rds` | rds | `help.aliyun.com/zh/rds/developer-reference/` |
| `Dds` (MongoDB) | mongodb | `help.aliyun.com/zh/mongodb/developer-reference/` |
| `Kvstore` (Redis) | redis | `help.aliyun.com/zh/redis/developer-reference/` |
| `Polardb` | polardb | `help.aliyun.com/zh/polardb/developer-reference/` |
| `Nas` | nas | `help.aliyun.com/zh/nas/developer-reference/` |
| `Alikafka` | alikafka | `help.aliyun.com/zh/alikafka/developer-reference/` |
| `Mse` | mse | `help.aliyun.com/zh/mse/developer-reference/` |
| `Vpc` | vpc | `help.aliyun.com/zh/vpc/developer-reference/` |

### 精确 URL 404 排查流程

```
精确 URL 404
→ 1. 尝试 API 目录页：help.aliyun.com/zh/{slug}/developer-reference/
→ 2. 搜索引擎："阿里云 {Action名} {Service名} API"
→ 3. 在线调试台：next.api.aliyun.com/api/{service}/{version}/
→ 4. 回归 Provider 源码（最权威，不会 404）
```

---

## 附录 B：AWS

### Provider 文档 URL

```
https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `aws_db_instance` | `db_instance` |
| `aws_elasticache_cluster` | `elasticache_cluster` |
| `aws_msk_cluster` | `msk_cluster` |

### API 文档

**查找路径**：
1. **AWS API Documentation**：`https://docs.aws.amazon.com/{service}/latest/APIReference/`
   - RDS: `docs.aws.amazon.com/AmazonRDS/latest/APIReference/`
   - ElastiCache: `docs.aws.amazon.com/AmazonElastiCache/latest/APIReference/`
2. **Terraform Provider 文档**：`registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/{resource}`
3. **Provider 源码**：`github.com/hashicorp/terraform-provider-aws`

### Provider 源码关键模式

```go
// AWS Provider 源码位置
internal/service/{service}/{resource}.go

// API 调用格式（使用 AWS SDK）
input := &rds.CreateDBInstanceInput{
  DBInstanceClass: aws.String("db.t3.micro"),
  Engine:         aws.String("mysql"),
}

// Schema 定义
"engine_version": {
  Type:     schema.TypeString,
  Optional: true,
  ForceNew: true,  // 修改触发重建
},
```

### 典型案例

| 资源 | 文档查找要点 |
|------|-------------|
| `aws_db_instance` | engine_version 是 ForceNew；storage_type 可在线修改 |
| `aws_msk_cluster` | kafka_version 需查 MSK API 确认可用版本 |
| `aws_elasticache_cluster` | engine_version_validations 在 API 端 |

---

## 附录 C：腾讯云（Tencent Cloud）

### Provider 文档 URL

```
https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `tencentcloud_mysql_instance` | `mysql_instance` |
| `tencentcloud_redis_instance` | `redis_instance` |
| `tencentcloud_ckafka_instance` | `ckafka_instance` |

### API 文档

**查找路径**：
1. **API Explorer**（最实用）：`https://console.cloud.tencent.com/api/explorer`
   - 可直接调试 API、查看请求/响应参数
2. **腾讯云 API 文档中心**：`https://cloud.tencent.com/document/api/{product}/{version}`
   - MySQL: `cloud.tencent.com/document/api/236/15880`
   - Redis: `cloud.tencent.com/document/api/239/20231`
3. **Provider 源码**：`github.com/TencentCloud/terraform-provider-tencentcloud`

### Provider 源码关键模式

```go
// 腾讯云 Provider 源码位置
tencentcloud/services/{service}/{resource}.go

// API 调用格式（使用腾讯云 SDK）
request := tcr.NewCreateRepositoryRequest()
request.RegistryId = &registryId

// Schema 定义（与通用结构一致）
"instance_name": {
  Type:     schema.TypeString,
  Required: true,
},
```

---

## 附录 D：Azure

### Provider 文档 URL

```
https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL 路径 |
|----------------|----------|
| `azurerm_mysql_flexible_server` | `mysql_flexible_server` |
| `azurerm_redis_cache` | `redis_cache` |
| `azurerm_kubernetes_cluster` | `kubernetes_cluster` |

### API 文档

**查找路径**：
1. **Azure REST API Docs**：`https://learn.microsoft.com/en-us/rest/api/{service}/`
   - MySQL: `learn.microsoft.com/en-us/rest/api/mysql/`
   - Redis: `learn.microsoft.com/en-us/rest/api/redis/`
2. **Terraform Provider 文档**：`registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/{resource}`
3. **Provider 源码**：`github.com/hashicorp/terraform-provider-azurerm`

### Provider 源码关键模式

```go
// Azure Provider 源码位置
internal/services/{service}/{resource}.go

// Schema 定义（使用 pluginsdk 或 hashicorp-go-azure-helpers）
"sku_name": {
  Type:     schema.TypeString,
  Required: true,
  ValidateFunc: validation.StringInSlice([]string{
    "B_Gen5_1", "GP_Gen5_2", "MO_Gen5_2",
  }, false),
},
```

---

## 相关规则

- `module-parameter-completeness` - 原子层参数完整性检查
- `provider-optional-api-mandatory` - Provider 参数 Schema 类型与处理规范
- `code-reference-documentation` - 原子层代码参考规范

## 参考资料

- `module-parameter-completeness.md`（参数完整性检查）
- `provider-optional-api-mandatory.md`（Provider 参数 Schema 类型）
- [AWS Provider 文档](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Azure Provider 文档](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
- [阿里云 Provider 文档](https://registry.terraform.io/providers/aliyun/alicloud/latest/docs)
- [腾讯云 Provider 文档](https://registry.terraform.io/providers/tencentcloudstack/tencentcloud/latest/docs)
