# Ch 10：Design Patterns with First-Class Functions

> 在拥有一等函数的语言中，许多经典设计模式可以大幅简化甚至变得不必要。Peter Norvig 指出，GoF 的 23 个设计模式中有 16 个在动态语言中会变得"不可见或更简单"。本章以 **Strategy 模式**和 **Command 模式**为例，展示如何用函数替代只有单个方法的类。

---

## Strategy Pattern

> 定义一系列算法，将每个算法封装起来，并使它们可以互换。策略模式让算法的变化独立于使用它的客户端。

**业务场景**

一个在线商店有三种折扣规则（假设每次只能用一种）：

- **忠诚度折扣**：积分 ≥ 1000 的客户，订单享 5% 折扣
- **大批量折扣**：单个商品数量 ≥ 20 件，该商品享 10% 折扣
- **大订单折扣**：不同商品种类 ≥ 10 种，订单享 7% 折扣

### 经典实现：`class`
| 角色 | 说明 | 示例 |
|----|------|------|
| **Context** | 提供服务，将计算委托给策略 | `Order` |
| **Strategy** | 各算法的公共接口 | `Promotion`（ABC） |
| **Concrete Strategy** | 实现具体算法的子类 | `FidelityPromo` / `BulkItemPromo` / `LargeOrderPromo` |

**Data Model**
- NamedTuple 让数据类既有具名字段，又是不可变的，非常适合做值对象。
```python
from abc import ABC, abstractmethod
from decimal import Decimal
from typing import NamedTuple, Optional
from collections.abc import Sequence

class Customer(NamedTuple):
    name: str
    fidelity: int

class LineItem(NamedTuple):
    product: str
    quantity: int
    price: Decimal
    def total(self) -> Decimal:
        return self.price * self.quantity
```
**Context**
- 关键设计： Order 自己不知道折扣怎么算，它只负责调用 `promotion.discount(self)`，把计算工作委托给具体策略。
```python
class Order(NamedTuple):  # Context
    customer: Customer
    cart: Sequence[LineItem]
    promotion: Optional['Promotion'] = None

    def total(self) -> Decimal:
        return sum((item.total() for item in self.cart), start=Decimal(0))

    def due(self) -> Decimal:
        if self.promotion is None:
            discount = Decimal(0)
        else:
            discount = self.promotion.discount(self)
        return self.total() - discount
```
**Strategy & 3 concrete Strategies**
- 使用 ABC + @abstractmethod 强制子类必须实现 `discount()`，否则无法实例化。
```python
class Promotion(ABC):  # Strategy 接口
    @abstractmethod
    def discount(self, order: Order) -> Decimal:
        """返回折扣金额"""

class FidelityPromo(Promotion):
    """积分 ≥ 1000 → 5% 折扣"""
    def discount(self, order: Order) -> Decimal:
        if order.customer.fidelity >= 1000:
            return order.total() * Decimal('0.05')
        return Decimal(0)

class BulkItemPromo(Promotion):
    """单品 ≥ 20 件 → 该品 10% 折扣"""
    def discount(self, order: Order) -> Decimal:
        discount = Decimal(0)
        for item in order.cart:
            if item.quantity >= 20:
                discount += item.total() * Decimal('0.1')
        return discount

class LargeOrderPromo(Promotion):
    """品类 ≥ 10 种 → 7% 折扣"""
    def discount(self, order: Order) -> Decimal:
        distinct_items = {item.product for item in order.cart}
        if len(distinct_items) >= 10:
            return order.total() * Decimal('0.07')
        return Decimal(0)
```

使用时需要实例化策略对象：

```python
Order(ann, cart, FidelityPromo())   # 注意括号——要创建实例
Order(joe, banana_cart, BulkItemPromo())
```
**问题所在**
虽然代码**完全正确**，但存在明显冗余：
```
每个具体策略类：
  ├── 一个 class 定义
  ├── 继承 Promotion
  └── 一个 discount() 方法
        └── 里面只有 2~5 行真正的业务逻辑！
```
每个具体策略都是**只有一个方法、没有任何状态**的类。这本质上就是一个函数，却写了一堆样板代码（ABC、子类、实例化）。

---

## Function-Oriented Strategy
>没有状态、只有一个方法的类，完全可以用函数代替。
- promotion: `Optional['Promotion'] -> Optional[Callable[['Order'], Decimal]]`
```python
from typing import Callable

@dataclass(frozen=True)
class Order:  # Context
    customer: Customer
    cart: Sequence[LineItem]
    promotion: Optional[Callable[['Order'], Decimal]] = None  # 接受函数

    def due(self) -> Decimal:
        if self.promotion is None:
            discount = Decimal(0)
        else:
            discount = self.promotion(self)  # 直接调用函数
        return self.total() - discount

def fidelity_promo(order: Order) -> Decimal:
    if order.customer.fidelity >= 1000:
        return order.total() * Decimal('0.05')
    return Decimal(0)

def bulk_item_promo(order: Order) -> Decimal:
    discount = Decimal(0)
    for item in order.cart:
        if item.quantity >= 20:
            discount += item.total() * Decimal('0.1')
    return discount

def large_order_promo(order: Order) -> Decimal:
    distinct_items = {item.product for item in order.cart}
    if len(distinct_items) >= 10:
        return order.total() * Decimal('0.07')
    return Decimal(0)
```

使用时直接传函数名（不加括号）：

```python
Order(ann, cart, fidelity_promo)
Order(joe, banana_cart, bulk_item_promo)
```
**函数式方案的优势**

经典方案中，GoF 建议用 **Flyweight 模式**来避免重复创建策略对象。而函数天然就是"享元"——每个函数在模块加载时只创建一次，可以在多个上下文中共享，不需要额外模式。

---

## Choosing the Best Strategy
如何让系统自动找出并应用折扣最大的策略，同时避免手动维护策略列表带来的隐患。
### Simple Approach：手动维护列表

```python
promos = [fidelity_promo, bulk_item_promo, large_order_promo]

def best_promo(order: Order) -> Decimal:
    """尝试所有策略，返回最大折扣"""
    return max(promo(order) for promo in promos)
```

```python
Order(ann, cart, best_promo)  # 自动选出最优折扣
```

**问题：** 新增策略函数后，如果忘了加到 `promos` 列表，`best_promo` 会默默忽略它。

### 方案一：模块内省（globals）

利用 `globals()` 自动找出当前模块中所有以 `_promo` 结尾的函数：

```python
promos = [promo for name, promo in globals().items()
          if name.endswith('_promo') and
             name != 'best_promo']
```
`globals()` 返回当前模块的全局符号表（dict），键是名字，值是对象：
```python
{
  'fidelity_promo':    <function fidelity_promo>,
  'bulk_item_promo':   <function bulk_item_promo>,
  'large_order_promo': <function large_order_promo>,
  'best_promo':        <function best_promo>,
  'Order':             <class 'Order'>,
  # ... 其他全局变量
}
```
- 通过过滤 `name.endswith('_promo')`，自动收集所有策略函数，新增策略只需写函数，不用再碰列表。
- 局限：依赖命名约定（_promo 后缀），属于隐式契约。
### 方案二：独立模块 + inspect

把所有策略函数放在单独的 `promotions.py` 模块中，用 `inspect` 自动收集：

```python
import inspect
import promotions

promos = [func for _, func in
          inspect.getmembers(promotions, inspect.isfunction)]
```
- `inspect`获取模块中所有函数成员
    ```python
    inspect.getmembers(promotions, inspect.isfunction)
    # 返回：
    # [('bulk_item_promo',   <function>),
    #  ('fidelity_promo',    <function>),
    #  ('large_order_promo', <function>)]
    ```
  - 第二个参数是谓词函数，只保留通过测试的成员。这里用 inspect.isfunction 过滤出函数。
- 不依赖函数命名规则，只要求 `promotions` 模块里只放折扣函数。

### 方案三：装饰器注册（推荐）
> 前几种方案都有一个根本矛盾：函数定义和函数注册是两个分离的动作，容易脱节。

新装饰器方案：把"注册"这个动作，内嵌进"定义"这个动作里，两者永远同时发生。

```python
# 1. 类型别名：任何接受 Order、返回 Decimal 的可调用对象
Promotion = Callable[[Order], Decimal]
# 2. 全局策略列表，初始为空
promos: list[Promotion] = []
# 3. 注册装饰器
def promotion(promo: Promotion) -> Promotion:
    promos.append(promo)   # 把函数加入列表
    return promo           # 原样返回函数，不做任何修改

# 4. best_promo 完全不变
def best_promo(order: Order) -> Decimal:
    return max(promo(order) for promo in promos)

# 5. 策略函数：加上装饰器，定义即注册
@promotion
def fidelity(order: Order) -> Decimal:
    """积分 ≥ 1000，享 5% 折扣"""
    if order.customer.fidelity >= 1000:
        return order.total() * Decimal('0.05')
    return Decimal(0)
@promotion
def bulk_item(order: Order) -> Decimal:
    ...
@promotion
def large_order(order: Order) -> Decimal:
    ...
```

装饰器方案的优势：

- 函数命名自由
- `@promotion` 标记了用途，注释掉就能临时禁用
- 策略函数可以定义在任何模块中，只要加上装饰器就会被注册

---

## Command Pattern

### 经典定义

把"做什么"打包成一个对象（或函数），把"触发操作的人"（invoker）和"执行操作的人"（receiver）解耦。
- 做法是在两者之间插入一个 Command 对象，它只有一个 `execute()` 方法。
- GoF 原书的说法：命令是回调的面向对象替代品。

**Python 的简化**
如果 Command 只有一个 `execute()` 方法且不需要维护状态，直接用函数就行了：

```python
class MenuItem:
    def __init__(self, label, command):
        self.label = label
        self.command = command      # 只持有一个可调用对象

    def click(self):
        self.command()              # 就这一行，永远不变
    def set_shortcut(self, key):
        self.shortcut = key         # 快捷键也触发同一个 command
    def add_to_toolbar(self):
        toolbar.add(self.label, self.command)  # 工具栏复用同一个 command
paste_cmd = lambda: document.paste()
```

### MacroCommand：用 `__call__` 实现

需要批量执行多个命令时，可以用实现了 `__call__` 的类：

```python
class MacroCommand:
    """依次执行一系列命令"""

    def __init__(self, commands):
        self.commands = list(commands)  # 防御性拷贝

    def __call__(self):
        for command in self.commands:
            command()
```

`MacroCommand` 的实例本身也是可调用的，可以像普通函数一样传递。

### 更复杂的 Command 时

如果需要支持撤销（undo）等功能，可以考虑：

- 实现了 `__call__` 的类：可以保存状态，还能提供额外方法
- 闭包：用函数的自由变量保存调用之间的状态

---

## 总结

| 要点 | 说明 |
|------|------|
| 一等函数简化设计模式 | 只有一个方法、无状态的类 → 直接用函数 |
| Strategy 模式 | 函数替代策略类，省去 ABC + 子类 + 实例化 |
| 函数天然是享元 | 模块加载时创建一次，多处共享，不需要 Flyweight |
| 自动收集策略 | `globals()` / `inspect` / 装饰器注册 |
| Command 模式 | 简单命令用函数；需要状态时用 `__call__` 类或闭包 |
| 核心洞察 | 当 API 要求实现只有一个方法的接口（execute / run / do_it），在 Python 中通常可以用函数代替 |

> Ralph Johnson（GoF 作者之一）反思：设计模式的一个失败之处是"过于强调模式是终点，而不是设计过程中的一个步骤"。本章正是以 Strategy 为起点，用一等函数逐步简化的过程。
