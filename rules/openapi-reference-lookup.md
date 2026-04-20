# 阿里云 OpenAPI 与 Provider 文档查找规范

**优先级：** 中
**分类：** 模块设计

> 从 Provider 源码反查 OpenAPI 文档地址，确保参数审计和模块开发时有权威参考。

## 为什么重要

在原子层参数审计、模块开发、参数约束确认时，需要查阅：
1. **OpenAPI 文档**：确认参数必填规则、取值范围、联动约束
2. **Provider 文档**：确认 Terraform 参数映射、默认值、ForceNew 行为
3. **Provider 源码**：最权威的参数定义和 API 调用逻辑

如果不知道如何系统性地查找这些文档，会导致：
- 凭猜测设置默认值，引发运行时错误
- 遗漏 API 级约束（条件必填、互斥参数）
- 无法确认参数是否为 ForceNew

## 查找流程

```
┌─────────────────────────────────────────────────────────────────┐
│              OpenAPI / Provider 文档查找流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 定位 Provider 源码                                          │
│     terraform-provider-alicloud/alicloud/resource_alicloud_*.go │
│     ↓                                                           │
│  2. 提取关键信息                                                 │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 信息           │ 提取位置                  │ 示例       │ │
│     ├─────────────────────────────────────────────────────────┤ │
│     │ Service Name  │ RpcPost/RpcGet 第1参数    │ "waf-openapi" │
│     │ API Version   │ RpcPost/RpcGet 第2参数    │ "2021-10-01"  │
│     │ Action Name   │ action := "..."           │ "CreatePostpaidInstance" │
│     │ Schema 字段   │ Schema: map[string]*Schema │ 见下文     │ │
│     └─────────────────────────────────────────────────────────┘ │
│     ↓                                                           │
│  3. 构造文档 URL                                                 │
│     ┌─────────────────────────────────────────────────────────┐ │
│     │ 文档类型    │ URL 模板                                   │ │
│     ├─────────────────────────────────────────────────────────┤ │
│     │ OpenAPI     │ 见下方 URL 构造规则                         │ │
│     │ Provider    │ 见下方 URL 构造规则                         │ │
│     └─────────────────────────────────────────────────────────┘ │
│     ↓                                                           │
│  4. 交叉验证                                                     │
│     Provider Schema ↔ OpenAPI 文档 ↔ 实际行为                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## URL 构造规则

### 1. Terraform Provider 文档

```
https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}
```

| Provider 资源名 | URL |
|----------------|-----|
| `alicloud_alb_load_balancer` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/alb_load_balancer |
| `alicloud_wafv3_instance` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/wafv3_instance |
| `alicloud_ssl_certificates_service_certificate` | https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/ssl_certificates_service_certificate |

**规则**：去掉 `alicloud_` 前缀即为 URL 路径

### 2. 阿里云 OpenAPI 文档

```
https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-{action}
```

#### 从 Provider 源码提取线索

Provider 源码中的 `RpcPost/RpcGet` 调用格式：

```go
response, err = client.RpcPost("waf-openapi", "2021-10-01", action, nil, request, false)
//                              ^^^^^^^^^^^^  ^^^^^^^^^^^
//                              Service Name   API Version
```

#### Service Name 到 product-slug 映射表

| Provider Service Name | product-slug | OpenAPI 文档基础 URL |
|----------------------|-------------|---------------------|
| `waf-openapi` | waf | https://help.aliyun.com/zh/waf/developer-reference/ |
| `cas` | ssl-certificates-service | https://help.aliyun.com/zh/ssl-certificates-service/developer-reference/ |
| `Alb` | alb | https://help.aliyun.com/zh/alb/developer-reference/ |
| `Slb` | slb | https://help.aliyun.com/zh/slb/developer-reference/ |
| `Nlb` | nlb | https://help.aliyun.com/zh/nlb/developer-reference/ |
| `Ecs` | ecs | https://help.aliyun.com/zh/ecs/developer-reference/ |
| `Rds` | rds | https://help.aliyun.com/zh/rds/developer-reference/ |
| `Dds` (MongoDB) | mongodb | https://help.aliyun.com/zh/mongodb/developer-reference/ |
| `Kvstore` (Redis) | redis | https://help.aliyun.com/zh/redis/developer-reference/ |
| `Polardb` | polardb | https://help.aliyun.com/zh/polardb/developer-reference/ |
| `Nas` | nas | https://help.aliyun.com/zh/nas/developer-reference/ |
| `Alikafka` | alikafka | https://help.aliyun.com/zh/alikafka/developer-reference/ |
| `Elasticsearch` | elasticsearch | https://help.aliyun.com/zh/elasticsearch/developer-reference/ |
| `Mse` | mse | https://help.aliyun.com/zh/mse/developer-reference/ |
| `Oss` | oss | https://help.aliyun.com/zh/oss/developer-reference/ |
| `Vpc` | vpc | https://help.aliyun.com/zh/vpc/developer-reference/ |

#### Action Name 到 API 文档 URL 转换

规则：将 Action 名从 PascalCase 转为 kebab-case，前面加 `api-`

```
CreatePostpaidInstance → api-waf-openapi-2021-10-01-createpostpaidinstance
UploadUserCertificate  → api-cas-2020-04-07-uploadusercertificate
CreateRule             → api-alb-2020-06-16-createrule
```

> **⚠️ 经验教训**：阿里云 OpenAPI 文档 URL 经常变动，精确 URL 大概率 404！
> 请使用以下 **可靠查找路径**（按优先级排序）：
>
> 1. **OpenAPI 在线调试台**（最可靠）：`https://next.api.aliyun.com/api/{service}/{version}/{action}`
>    - 可直接看到请求/响应参数
>    - WAF 3.0 示例：https://next.api.aliyun.com/api/waf-openapi/2021-10-01/CreatePostpaidInstance
> 2. **帮助中心搜索**：`https://help.aliyun.com/zh/` → 搜索产品名 → 开发者参考
>    - WAF 3.0 正确路径：https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/
>    - 注意：产品 slug 可能不是 Service Name 直译（如 waf-openapi → waf/web-application-firewall-3-0）
> 3. **API 目录页**（推荐中间页）：`https://help.aliyun.com/zh/{product-slug}/developer-reference/api-{service}-{version}-dir/`
>    - WAF 3.0：https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/api-waf-openapi-2021-10-01-dir/
>    - 从目录页可以逐级导航到具体的 Action 页面
> 4. **Provider 文档**（Terraform 侧）：`https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/{resource_name}`
>    - 有时包含 OpenAPI 参数对照
>    - WAF 示例：https://registry.terraform.io/providers/aliyun/alicloud/latest/docs/resources/wafv3_instance

### 3. 在线 OpenAPI 调试

```
https://next.api.aliyun.com/api/{service}/{version}/{action}
```

| 资源 | 在线调试 URL |
|------|-------------|
| WAF 创建实例 | https://next.api.aliyun.com/api/waf-openapi/2021-10-01/CreatePostpaidInstance |
| SSL 上传证书 | https://next.api.aliyun.com/api/cas/2020-04-07/UploadUserCertificate |
| ALB 创建规则 | https://next.api.aliyun.com/api/Alb/2020-06-16/CreateRule |

## Provider 源码关键位置速查

### 1. Schema 定义（参数完整性审计入口）

```go
// 位置：resource_alicloud_{product}.go 顶部
Schema: map[string]*schema.Schema{
    "param_name": {
        Type:     schema.TypeString,     // 参数类型
        Optional: true,                   // 可选
        Required: true,                   // 必填
        Computed: true,                   // API 返回
        ForceNew: true,                   // 修改触发重建
        Default:  "default_value",        // Provider 内置默认值
    },
}
```

### 2. Create 函数（API 调用逻辑）

```go
// 位置：resourceAlicloud{Product}Create()
action := "CreateXxx"                    // API Action 名
response, err = client.RpcPost(          // API 调用
    "service-name",                       // Service 标识
    "2021-10-01",                         // API 版本
    action, nil, request, false,          // Action + 请求体
)
```

### 3. Update 函数（参数更新行为）

```go
// 位置：resourceAlicloud{Product}Update()
// 关键：d.HasChange("param") 判断哪些参数变更触发 Update
if d.HasChange("param_name") {
    update = true  // 触发 API 调用
}
```

### 4. 数据源定义（查询类资源）

```go
// 位置：dataSource_alicloud_{product}.go
// 数据源用于查询已有资源，如 SSL 证书查询
```

## 审计时的标准操作步骤

### Step 1: 定位 Provider 源码

```bash
# 在项目内搜索
terraform-provider-alicloud/alicloud/resource_alicloud_{resource_name}.go
```

### Step 2: 提取 Schema 信息

对照 `module-parameter-completeness` 规则，提取所有 Optional/Required 字段

### Step 3: 查阅 OpenAPI 文档

1. 从 `RpcPost/RpcGet` 提取 Service + Version + Action
2. 构造 OpenAPI 文档 URL 或在线调试 URL
3. 确认参数约束（必填/选填、取值范围、联动规则）

### Step 4: 查阅 Provider 文档

1. 构造 Provider 文档 URL
2. 确认 Terraform 侧的参数映射和默认值

### Step 5: 交叉验证

| 验证项 | 方法 |
|--------|------|
| 参数是否存在 | Provider Schema ↔ OpenAPI 参数列表 |
| 默认值是否安全 | Provider Default ↔ OpenAPI 文档描述 |
| ForceNew 是否正确 | Provider ForceNew ↔ OpenAPI 重建行为 |
| 条件必填 | OpenAPI 文档 ↔ 原子层 validation |

## 产品 slug 完整映射表（实际验证）

> **关键发现**：阿里云帮助中心的 product-slug 往往不是 Service Name 的直译，
> 需要通过搜索或导航确认。以下为实际验证过的正确路径。

| Provider Service Name | 帮助中心 product-slug | 开发者参考完整路径 |
|----------------------|---------------------|----------------|
| `waf-openapi` | waf/web-application-firewall-3-0 | https://help.aliyun.com/zh/waf/web-application-firewall-3-0/developer-reference/ |
| `cas` | ssl-certificates-service | https://help.aliyun.com/zh/ssl-certificates-service/developer-reference/ |
| `Alb` | alb | https://help.aliyun.com/zh/alb/developer-reference/ |
| `Slb` | slb | https://help.aliyun.com/zh/slb/developer-reference/ |
| `Nlb` | nlb | https://help.aliyun.com/zh/nlb/developer-reference/ |
| `Ecs` | ecs | https://help.aliyun.com/zh/ecs/developer-reference/ |
| `Rds` | rds | https://help.aliyun.com/zh/rds/developer-reference/ |
| `Dds` (MongoDB) | mongodb | https://help.aliyun.com/zh/mongodb/developer-reference/ |
| `Kvstore` (Redis) | redis | https://help.aliyun.com/zh/redis/developer-reference/ |
| `Polardb` | polardb | https://help.aliyun.com/zh/polardb/developer-reference/ |
| `Nas` | nas | https://help.aliyun.com/zh/nas/developer-reference/ |
| `Alikafka` | alikafka | https://help.aliyun.com/zh/alikafka/developer-reference/ |
| `Elasticsearch` | elasticsearch | https://help.aliyun.com/zh/elasticsearch/developer-reference/ |
| `Mse` | mse | https://help.aliyun.com/zh/mse/developer-reference/ |
| `Oss` | oss | https://help.aliyun.com/zh/oss/developer-reference/ |
| `Vpc` | vpc | https://help.aliyun.com/zh/vpc/developer-reference/ |

## 当精确 URL 404 时的排查流程

```
精确 URL 404
    ↓
┌─── 1. 尝试 API 目录页 ────────────────────────────────────┐
│    https://help.aliyun.com/zh/{slug}/developer-reference/  │
│    → 从目录导航到具体 Action                                │
└──────────────────────────────────────────────────────────┘
    ↓ 仍然 404
┌─── 2. 使用搜索引擎 ──────────────────────────────────────┐
│    搜索关键词："阿里云 {Action名} {Service名} API"         │
│    → 搜索结果通常包含正确的文档链接                         │
└──────────────────────────────────────────────────────────┘
    ↓ 仍然找不到
┌─── 3. 使用在线调试台（最可靠）──────────────────────────┐
│    https://next.api.aliyun.com/api/{service}/{version}/   │
│    → 直接查看请求/响应参数定义                              │
└──────────────────────────────────────────────────────────┘
    ↓ 需要更多上下文
┌─── 4. 回归 Provider 源码（最权威）──────────────────────┐
│    Schema 定义 → 完整参数列表                              │
│    Create/Update 函数 → API 调用逻辑                      │
│    → 源码不会 404，永远可读                                │
└──────────────────────────────────────────────────────────┘
```

## 相关规则

- `module-parameter-completeness` - 原子层参数完整性检查
- `provider-optional-api-mandatory` - Provider 参数 Schema 类型与处理规范
- `code-reference-documentation` - 原子层代码参考规范

## 参考资料

- `provider-documentation-lookup.md`（多云 Provider 文档查找规范）
- `module-parameter-completeness.md`（参数完整性检查）
- [阿里云 OpenAPI 调试台](https://next.api.aliyun.com/)
