# 资源归属：三问法

**优先级：** 高
**分类：** 三层架构

## 为什么重要

构建控制层模块时，资源必须按**功能职责**组织，而非按资源类型。ECS 实例和托管服务属于不同模块，依据是它们**做什么**，而非它们**是什么**。归属错误会导致依赖混乱和安全边界不清。

## 三问法决策树

确定资源属于哪个控制模块时，问：

| 问题 | 如果是 | 归属模块 | 安全组 |
|------|--------|----------|--------|
| **1. 是否存储数据？** | MySQL/Kafka/ES/PolarDB/Redis | database-cluster | sg-data（自建）或 security_ips（托管） |
| **2. 是否为平台级基础设施？** | SkyWalking/ilogtail/MSE | middleware-cluster | sg-ecs |
| **3. 是否处理业务流量？** | Nginx/Jenkins/业务应用 | web-cluster | sg-ecs |

## 控制模块归属

```
control/
├── database-cluster/     # 问题 1：存储数据
│   ├── PolarDB          # 托管数据库
│   ├── Redis            # 托管缓存
│   ├── Kafka            # 托管消息（数据持久化）
│   ├── ES               # 托管搜索（索引数据）
│   └── MySQL ECS        # 自建数据库
│
├── middleware-cluster/   # 问题 2：平台基础设施
│   ├── MSE              # 服务注册中心
│   ├── NAS              # 共享存储
│   ├── OSS              # 对象存储
│   ├── SkyWalking ECS   # 可观测平台
│   └── ilogtail ECS     # 日志采集平台
│
└── web-cluster/          # 问题 3：业务流量
    ├── Nginx ECS        # 反向代理
    ├── Jenkins ECS      # CI/CD
    ├── App ECS×N        # 业务应用
    ├── ALB×N            # 七层负载均衡
    ├── SLB×N            # 四层负载均衡
    └── NLB×N            # 网络负载均衡
```

## 关键说明：托管服务与安全组

**托管服务（PolarDB/Redis/Kafka/ES）没有 ENI，无法"加入"安全组。**

security_group_ids 参数的含义是"自动将此安全组的成员 IP 添加到 security_ips 白名单"，而非防火墙。

```hcl
# ❌ 错误理解：托管服务加入安全组
# "PolarDB 加入 sg-test-data 安全组"

# ✅ 正确理解：托管服务使用 IP 白名单
# PolarDB 使用 security_ips 白名单
# security_group_ids 用于自动从安全组成员填充 security_ips
```

## 安全组归属

| 安全组 | 创建方 | 使用方 |
|--------|--------|--------|
| sg-ecs | security 层 | 所有包含 ECS 的模块 |
| sg-pod | security 层 | 仅 ack-cluster |
| sg-data | security 层 | database-cluster（自建 MySQL） |
| SLB ACL×N | web-cluster | web-cluster（生命周期与 SLB 对齐） |
| ALB ACL×1 | security 层 | ack-cluster |

## 决策示例

```hcl
# 问题 1：是否存储数据？
# MySQL ECS → database-cluster
# 创建 sg-data 保护自建 MySQL

mysql_ecs_instances = {
  "01" = {
    instance_type = "ecs.g5ne.xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：sg-data + sg-ecs
  }
}

# 问题 2：是否为平台基础设施？
# SkyWalking ECS → middleware-cluster
# 仅使用 sg-ecs（无需数据保护）

skywalking_ecs_instances = {
  "oap-01" = {
    instance_type = "ecs.g6a.2xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：仅 sg-ecs
  }
}

# 问题 3：是否处理业务流量？
# Nginx ECS → web-cluster
# 仅使用 sg-ecs（前端层）

ecs_instances = {
  "nginx-01" = {
    instance_type = "ecs.sn1ne.xlarge"
    vswitch_key   = "j-ecs"
    # 安全组：仅 sg-ecs
  }
}
```

## 反模式：按资源类型分组

```hcl
# ❌ 错误：ECS 按资源类型分组
control/
├── ecs-cluster/         # 所有 ECS 放在一个模块
│   ├── MySQL ECS
│   ├── SkyWalking ECS
│   ├── Nginx ECS
│   └── App ECS
└── load-balancers/      # 所有 LB 放在一个模块
    ├── ALB
    ├── SLB
    └── NLB
```

**问题：** 业务逻辑分散、归属不清、依赖混乱。

## 检查清单

- [ ] 所有 ECS 按功能职责分配到模块
- [ ] 自建数据库使用 sg-data（在 security 层创建）
- [ ] 托管服务接收 sg_ecs_id 用于白名单填充
- [ ] 无"所有 ECS 放一个模块"反模式
- [ ] 安全组归属已记录
- [ ] DAG 依赖关系与三问法对齐

## 参考资料

