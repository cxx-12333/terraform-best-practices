# 使用 for_each 替代 count 管理多资源

**优先级：** 中高
**分类：** 资源组织

## 为什么重要

使用 `count` 配合列表时，资源通过索引标识。从列表中间删除元素会导致后续资源被销毁并重建。`for_each` 使用稳定的键，不受列表修改影响。

## 错误示例

```hcl
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "subnets" {
  count      = length(var.subnet_cidrs)
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index]

  tags = {
    Name = "subnet-${count.index}"
  }
}
```

**问题：** 如果从中间删除 `"10.0.2.0/24"`：
- `subnet[1]`（原来是 10.0.2.0/24）变成 10.0.3.0/24
- `subnet[2]`（原来是 10.0.3.0/24）被销毁
- subnet[1] 中的资源受到干扰

## 正确示例

```hcl
variable "subnets" {
  default = {
    "public-a" = "10.0.1.0/24"
    "public-b" = "10.0.2.0/24"
    "public-c" = "10.0.3.0/24"
  }
}

resource "aws_subnet" "subnets" {
  for_each   = var.subnets
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value

  tags = {
    Name = each.key
  }
}

# 引用特定子网
output "public_a_id" {
  value = aws_subnet.subnets["public-a"].id
}
```

## 列表转 Map

```hcl
variable "availability_zones" {
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "subnets" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.value
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, index(var.availability_zones, each.value))

  tags = {
    Name = "subnet-${each.value}"
  }
}
```

## 何时使用 count

`count` 仍然适用于以下场景：

```hcl
# 布尔条件 - 创建或不创建
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? 1 : 0

  domain = "vpc"
}

# 固定数量的相同资源
resource "aws_nat_gateway" "gw" {
  count = var.nat_gateway_count # 例如 3

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

## 快速参考

| 场景 | 使用 |
|------|------|
| 创建 0 或 1 个资源 | `count` |
| 固定数量的相同资源 | `count` |
| 按名称/键标识的资源 | `for_each` |
| 可能删除元素的列表 | `for_each` + `toset()` |

## 参考资料

- [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)
- [count](https://developer.hashicorp.com/terraform/language/meta-arguments/count)
