# Ch 1. Python 数据模型

Python 数据模型（Data Model）是 Python 作为框架的描述，定义了语言构建块的接口。通过实现**特殊方法（Special Methods / Dunder Methods）**，自定义对象可以与语言核心特性无缝集成。

> **Dunder** = Double Underscore，如 `__getitem__` 读作 "dunder-getitem"

---

## 特殊方法的作用范围

实现特殊方法可让对象支持：

- 集合操作（Collections）
- 属性访问（Attribute access）
- 迭代，含异步迭代 `async for`
- 运算符重载（Operator overloading）
- 函数与方法调用
- 字符串表示与格式化
- 异步编程 `await`
- 对象创建与销毁
- 上下文管理 `with` / `async with`

---

## 示例一：FrenchDeck（序列模拟）

只需实现 `__len__` 和 `__getitem__` 两个方法，即可获得大量"免费"功能：

```python
import collections

Card = collections.namedtuple('Card', ['rank', 'suit'])

class FrenchDeck:
    ranks = [str(n) for n in range(2, 11)] + list('JQKA')
    suits = 'spades diamonds clubs hearts'.split()

    def __init__(self):
        self._cards = [Card(rank, suit) for suit in self.suits
                                        for rank in self.ranks]

    def __len__(self):
        return len(self._cards)

    def __getitem__(self, position):
        return self._cards[position]
```

**自动获得的能力：**

| 能力 | 触发方式 |
|------|----------|
| `len(deck)` → 52 | `__len__` |
| `deck[0]`、`deck[-1]` | `__getitem__` |
| `deck[:3]`、`deck[12::13]` 切片 | `__getitem__` 委托给 list |
| `for card in deck` 迭代 | `__getitem__` |
| `for card in reversed(deck)` | `__getitem__` |
| `Card('Q','hearts') in deck` | 无 `__contains__` 时顺序扫描 |
| `sorted(deck, key=spades_high)` | 可迭代即可排序 |
| `random.choice(deck)` | 序列协议 |

**结论：** 功能来自**数据模型 + 组合**，而非继承。

---

## 示例二：Vector（数值类型模拟）

```python
import math

class Vector:
    def __init__(self, x=0, y=0):
        self.x = x
        self.y = y

    def __repr__(self):
        return f'Vector({self.x!r}, {self.y!r})'

    def __abs__(self):
        return math.hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))      # 或更快: bool(self.x or self.y)

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar):
        return Vector(self.x * scalar, self.y * scalar)
```

---

## 关键特殊方法详解

### `__repr__` vs `__str__`

| 方法 | 调用时机 | 目标受众 |
|------|----------|----------|
| `__repr__` | `repr(obj)`、调试器、f-string `!r` | 开发者，应尽量能重建对象 |
| `__str__` | `str(obj)`、`print()` | 终端用户 |

> 若只实现一个，选 `__repr__`。`__str__` 缺失时会回退到 `__repr__`。

### `__bool__`

- 默认：用户自定义类实例均为 **truthy**
- `bool(x)` → 先调 `x.__bool__()` → 再调 `x.__len__()`（返回 0 则 False）

---

## Collection A(bstract) B(ase) C(lass) 层次结构
最顶层有三个最基础的能力：

- Iterable — 能被 for 循环遍历（实现 __iter__）
- Sized — 能用 len() 取长度（实现 __len__）
- Container — 能用 in 判断包含（实现 __contains__）

Collection 是这三者的组合，意思是"一个像样的集合，应该同时具备这三种能力"。

```
Iterable(__iter__)   Sized(__len__)   Container(__contains__)
         \               |               /
           \             |             /
             \           |           /
               \         |         /
                 \       |       /
                  Collection(3.6+)
                   /        \
        Sequence        Mapping        Set
      (__getitem__)   (__getitem__)  (isdisjoint...)
```

- **Sequence**：支持 Reversible，如 `list`、`str`
- **Mapping**：`dict`、`defaultdict`（Python 3.7+ 保留插入顺序，但不可任意重排）
- **Set**：`set`、`frozenset`，所有方法均为中缀运算符实现

> Python **不强制继承** ABC，实现了对应方法即满足接口。

---

## 特殊方法速查（非运算符类）

| 类别 | 方法 |
|------|------|
| 字符串/字节表示 | `__repr__` `__str__` `__format__` `__bytes__` |
| 数值转换 | `__bool__` `__int__` `__float__` `__hash__` `__index__` |
| 模拟集合 | `__len__` `__getitem__` `__setitem__` `__delitem__` `__contains__` |
| 迭代 | `__iter__` `__next__` `__reversed__` `__aiter__` `__anext__` |
| 可调用/协程 | `__call__` `__await__` |
| 上下文管理 | `__enter__` `__exit__` `__aenter__` `__aexit__` |
| 实例生命周期 | `__new__` `__init__` `__del__` |
| 属性管理 | `__getattr__` `__getattribute__` `__setattr__` `__delattr__` `__dir__` |
| 描述符 | `__get__` `__set__` `__delete__` `__set_name__` |
| 类元编程 | `__init_subclass__` `__class_getitem__` `__prepare__` |


## 运算符特殊方法速查

| 类别 | 符号 | 方法 |
|------|------|------|
| 一元 | `-` `+` `abs()` | `__neg__` `__pos__` `__abs__` |
| 比较 | `< <= == != > >=` | `__lt__` `__le__` `__eq__` `__ne__` `__gt__` `__ge__` |
| 算术 | `+ - * / // % @ **` | `__add__` `__sub__` `__mul__` `__truediv__` `__floordiv__` `__mod__` `__matmul__` `__pow__` |
| 反射算术 | （操作数互换时触发） | `__radd__` `__rsub__` `__rmul__` ... |
| 增量赋值 | `+= -= *=` ... | `__iadd__` `__isub__` `__imul__` ... |
| 位运算 | `& \| ^ << >> ~` | `__and__` `__or__` `__xor__` `__lshift__` `__rshift__` `__invert__` |

---

## 为什么 `len` 不是方法？

- 对内置类型（`list`、`str` 等），`len(x)` 直接读取 C 结构体的 `ob_size` 字段，极快
- 通过 `__len__` 特殊方法，自定义类也能让 `len()` 生效
- 这是**效率**与**一致性**之间的合理折衷

---


## `__iter__` 与迭代协议

### 执行流程

`for x in obj` 背后分三步：

1. `iter(obj)` → 触发 `obj.__iter__()`，返回一个**迭代器对象**
2. 反复 `next(迭代器)` → 触发 `__next__()`，每次取一个值
3. `__next__` 抛出 `StopIteration` → 循环结束

> `__getitem__` 是兜底机制：没有 `__iter__` 时，Python 从索引 0 开始逐个调用 `__getitem__`，直到 `IndexError`。

---

### 完整实现：容器与迭代器分离

```python
class MyRange:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        return MyRangeIterator(self)   # 返回独立的迭代器对象


class MyRangeIterator:
    def __init__(self, obj):
        self.obj = obj
        self.current = 0

    def __iter__(self):
        return self                    # 迭代器自身也要实现 __iter__，返回自己

    def __next__(self):
        if self.current >= self.obj.n:
            raise StopIteration
        val = self.current
        self.current += 1
        return val
```

**为什么容器和迭代器要分开？**

同一容器可能同时跑多个循环，每个循环需要独立的"当前位置"。若状态存在容器本身，嵌套循环会共享位置导致混乱：

```python
r = MyRange(3)
for x in r:
    for y in r:   # 两个迭代器互相独立，互不干扰
        print(x, y)
```

---

### 实际常用 `yield` 代替手写迭代器

```python
class MyRange:
    def __init__(self, n):
        self.n = n

    def __iter__(self):
        current = 0
        while current < self.n:
            yield current    # 自动处理 __next__ 和 StopIteration
            current += 1
```

`yield` 让 `__iter__` 变成**生成器函数**，Python 自动创建迭代器对象，省去手写 `MyRangeIterator` 的麻烦，效果完全一致。

---


## 总结

1. 特殊方法由 **Python 解释器调用**，而不是用户代码直接调用
2. 应调用内置函数（`len()`、`iter()`），而非直接调用 `__len__()`、`__iter__()`
3. 用户代码中唯一常见的直接调用是 `__init__`（在子类中调用父类初始化）
4. 实现特殊方法 = 让对象融入语言生态，而非重复造轮子
5. 功能优先来自**组合（Composition）**，而非继承
