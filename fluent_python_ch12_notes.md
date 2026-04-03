# Ch 12：Special Methods for Sequences

通过实现序列协议（`__len__` + `__getitem__`），构建一个 N 维 `Vector` 类，展示如何让自定义类型表现得像标准 Python 序列。

---

## Protocol and Duck Typing

**协议（Protocol）** 是 Python 中的非正式接口，只在文档中定义，不在代码中强制。

序列协议的核心只有两个方法：`__len__` 和 `__getitem__`。任何实现了这两个方法的类都可以在需要序列的地方使用，无需继承任何基类。

```python
# FrenchDeck 没有继承任何序列类，但因为实现了协议，它就是序列
class FrenchDeck:
    def __len__(self): ...
    def __getitem__(self, position): ...
# 实现这两个接口之后自动获得的能力。
len(s)           # ✅ 调用 __len__
s[0]             # ✅ 调用 __getitem__
s[1:3]           # ✅ 调用 __getitem__ (传入slice对象)
for x in s:      # ✅ 自动迭代
    ...
x in s           # ✅ 自动支持 in 运算符
reversed(s)      # ✅ 自动支持反转
```

**部分实现也可以：** 比如只实现 `__getitem__` 就足以支持迭代，不一定需要 `__len__`。

> Python 3.8+ 引入了 `typing.Protocol`（静态协议），与传统的动态协议含义相关但不同。静态协议要求实现所有声明的方法，动态协议则更灵活。

```python
# EAFP（请求原谅比请求许可更容易）—— 推荐
try:
    n = len(obj)
except TypeError:
    n = None
```

**Python 迭代的回退机制：**
```
1. 先找 __iter__
2. 没有？再找 __getitem__（从索引0开始，直到 IndexError）
3. 都没有？抛出 TypeError
```
---

## Vector Take #1：基础实现

一个 任意维度的 Vector，并让它表现得像一个标准的 Python 不可变扁平序列（类似 tuple）

- 使用**组合**而非继承，内部用 `array('d', ...)` 存储分量
- 构造器接受**可迭代对象**（与内置序列一致），而非 `*args`
- 用 `reprlib.repr()` 限制大型向量的 repr 输出长度

```python
from array import array
import reprlib
import math

class Vector:
    typecode = 'd'
    # 类型码，作为类变量存在，子类可以覆盖它以使用不同精度。

    def __init__(self, components):
        self._components = array(self.typecode, components)

    def __iter__(self):
        return iter(self._components)

    def __repr__(self):
        # 步骤1：reprlib.repr 产生截断的字符串
        components = reprlib.repr(self._components)
        # → "array('d', [0.0, 1.0, 2.0, 3.0, 4.0, ...])"

        # 步骤2：找到 '[' 的位置，截取到倒数第2个字符
        components = components[components.find('['):-1]
        # find('[') 找到 '[' 的索引位置
        # [索引:-1] 取从 '[' 到最后一个 ')' 之前的部分
        # → "[0.0, 1.0, 2.0, 3.0, 4.0, ...]"

        # 步骤3：拼接成最终结果
        return f'Vector({components})'
        # → "Vector([0.0, 1.0, 2.0, 3.0, 4.0, ...])"

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) +
                bytes(self._components))

    def __eq__(self, other):
        return tuple(self) == tuple(other)

    def __abs__(self):
        return math.hypot(*self)  # Python 3.8+ 支持 N 维

    def __bool__(self):
        return bool(abs(self))

    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(memv)  # 直接传 memoryview，无需 * 解包
```
- `reprlib.repr()` 对大型集合自动截断并加 `...`，避免 repr 输出过长
- `repr()` 永远不应抛出异常——它用于调试，必须尽力产出可用输出
- `math.hypot(*self)` 在 Python 3.8+ 支持任意维度

---

## Vector Take #2：可切片序列

**问题**：简单委托给 `_components` 的 `__getitem__` 会让切片返回 `array` 而非 `Vector`：

```python
>>> v7 = Vector(range(7))
>>> v7[1:4]
array('d', [1.0, 2.0, 3.0])  # 应该返回 Vector！
```

**切片机制**
Python 将 `my_seq[1:4:2]` 转化为 `my_seq.__getitem__(slice(1, 4, 2))`：

```python
>>> class MySeq:
...     def __getitem__(self, index):
...         return index
>>> s = MySeq()
>>> s[1:4]        # → slice(1, 4, None)
>>> s[1:4:2, 9]   # → (slice(1, 4, 2), 9)  逗号产生 tuple
```

`slice` 对象有个实用方法 `indices(len)`，可将越界/负数索引规范化为合法的 `(start, stop, stride)` 三元组。

**感知切片的 `__getitem__`**

```python
import operator

def __len__(self):
    return len(self._components)

def __getitem__(self, key):
    if isinstance(key, slice):
        cls = type(self)
        return cls(self._components[key])  # 切片 → 返回新 Vector
    index = operator.index(key)            # 整数索引
    return self._components[index]
```

**`operator.index()` vs `int()`：**
- `operator.index(3.14)` → `TypeError`（float 不应作为索引）
- `int(3.14)` → `3`（静默截断，不安全）
- `operator.index()` 调用的是 `__index__` 特殊方法，专为索引场景设计



---

## Vector Take #3：动态属性访问

用 `v.x`, `v.y`, `v.z`, `v.t` 访问前 4 个分量，替代 `v[0]` ~ `v[3]`。

**`__getattr__` 实现**

```python
__match_args__ = ('x', 'y', 'z', 't')  # 同时支持位置模式匹配

def __getattr__(self, name):
    cls = type(self)
    try:
        pos = cls.__match_args__.index(name)
    except ValueError:
        pos = -1
    if 0 <= pos < len(self._components):
        return self._components[pos]
    msg = f'{cls.__name__!r} object has no attribute {name!r}'
    raise AttributeError(msg)
```

**陷阱：必须配套实现 `__setattr__`**

`__getattr__` 只在属性查找失败时被调用。
如果允许赋值 `v.x = 10`，会创建实例属性，之后读取 `v.x` 直接返回实例属性值（不再触发 `__getattr__`），但 `_components` 没有改变。

```python
def __setattr__(self, name, value):
    cls = type(self)
    if len(name) == 1:
        if name in cls.__match_args__:
            error = 'readonly attribute {attr_name!r}'
        elif name.islower():
            error = "can't set attributes 'a' to 'z' in {cls_name!r}"
        else:
            error = ''
        if error:
            msg = error.format(cls_name=cls.__name__, attr_name=name)
            raise AttributeError(msg)
    super().__setattr__(name, value)  # 把合法的赋值委托给父类处理

v = Vector(range(5))

v.x = 10        # ❌ AttributeError: readonly attribute 'x'
v.y = 10        # ❌ AttributeError: readonly attribute 'y'

v.a = 10        # ❌ AttributeError: can't set attributes 'a' to 'z'
v.k = 10        # ❌ AttributeError: can't set attributes 'a' to 'z'

v.X = 10        # ✅ 大写，允许（不会混淆）
v.MyAttr = 10   # ✅ 多字符，允许
v._data = 10    # ✅ 下划线开头，允许
```
```
__getattr__
  ├── 触发时机：正常查找失败后的兜底
  ├── 用途：为 x/y/z/t 动态映射到分量
  └── 陷阱：赋值后实例字典优先，导致不一致

__setattr__
  ├── 触发时机：每次赋值都触发
  ├── 用途：拦截非法赋值，保持不可变性
  └── 结尾必须调用 super().__setattr__ 处理合法赋值
```
> **核心教训：** 实现 `__getattr__` 时，几乎总是需要同时实现 `__setattr__`，否则会出现不一致行为。

---

## Vector Take #4：散列与高效

### __hash__：用 functools.reduce 做 map-reduce

对于 N 维向量，不能像 Vector2d 那样构建 tuple 再 hash（太浪费内存）。改用 xor 归约所有分量的 hash：

```python
import functools
import operator

def __hash__(self):
    hashes = (hash(x) for x in self._components)  # map: 惰性求 hash
    return functools.reduce(operator.xor, hashes, 0)  # reduce: xor 归约
```

**`reduce` 的三个参数：** `reduce(function, iterable, initializer)`
```
reduce(fn, [a, b, c, d]) 的计算过程：
第1步：fn(a, b) → r1
第2步：fn(r1, c) → r2
第3步：fn(r2, d) → r3（最终结果）
```
- `initializer` 防止空序列报错
- 对于 `+`, `|`, `^`，initializer 应为 `0`
- 对于 `*`, `&`，initializer 应为 `1`

**等价写法（用 `map`）：**
```python
def __hash__(self):
    hashes = map(hash, self._components)
    return functools.reduce(operator.xor, hashes, 0)
```

### __eq__：用 zip + all 替代 tuple 比较

原始版本为大向量构建两个完整 tuple 再比较，浪费内存：

```python
# 低效版本
def __eq__(self, other):
    return tuple(self) == tuple(other)
```

高效版本——提前退出，不构建临时对象：

```python
# 最终版本（一行）
def __eq__(self, other):
    return len(self) == len(other) and all(a == b for a, b in zip(self, other))
```

**为什么先检查 `len`：** `zip` 会在最短输入耗尽时静默停止，不检查长度可能将不等长序列判为相等。

> Python 3.10 新增 `zip(..., strict=True)` 参数，长度不等时抛出 `ValueError`。

---

## `zip`

```python
>>> list(zip(range(3), 'ABC'))
[(0, 'A'), (1, 'B'), (2, 'C')]

>>> list(zip(range(3), 'ABC', [0.0, 1.1, 2.2, 3.3]))
[(0, 'A', 0.0), (1, 'B', 1.1), (2, 'C', 2.2)]  # 3.3 被丢弃！

# 不想丢弃？用 itertools.zip_longest
>>> from itertools import zip_longest
>>> list(zip_longest(range(3), 'ABCD', fillvalue=-1))
[(0, 'A'), (1, 'B'), (2, 'C'), (-1, 'D')]

# Python 3.10+ zip(strict=True)：长度不同则报错
list(zip([1,2,3], [1,2], strict=True))
# → ValueError: zip() has arguments with different lengths
```

**zip 转置矩阵：**
```python
>>> matrix = [(1, 2, 3), (4, 5, 6)]
>>> list(zip(*matrix))
[(1, 4), (2, 5), (3, 6)]
```

---

## Vector Take #5：自定义格式化

扩展 Format Specification Mini-Language，用 `'h'` 后缀表示超球面坐标：

```python
>>> format(Vector([1, 1, 1]), 'h')
'<1.73205..., 0.95531..., 0.78539...>'
>>> format(Vector([1, 1, 1]), '.3eh')
'<1.732e+00, 9.553e-01, 7.854e-01>'
```

```python
import itertools

def angle(self, n):
    r = math.hypot(*self[n:])
    a = math.atan2(r, self[n-1])
    if (n == len(self) - 1) and (self[-1] < 0):
        return math.pi * 2 - a
    return a

def angles(self):
    return (self.angle(n) for n in range(1, len(self)))

def __format__(self, fmt_spec=''):
    if fmt_spec.endswith('h'):          # ① 检测 'h' 后缀
        fmt_spec = fmt_spec[:-1]        # ② 去掉 'h'，剩余是数字格式
        coords = itertools.chain(
            [abs(self)],                # ③ 第一个元素：模长 r
            self.angles()              # ④ 其余元素：各角度
        )
        outer_fmt = '<{}>'             # ⑤ 尖括号
    else:
        coords = self                  # ⑥ 直接用所有分量
        outer_fmt = '({})'            # ⑦ 圆括号

    components = (
        format(c, fmt_spec)            # ⑧ 对每个坐标应用数字格式
        for c in coords
    )
    return outer_fmt.format(
        ', '.join(components)          # ⑨ 拼接并套上括号
    )
```

**`itertools.chain`** 将多个可迭代对象无缝串联，这里用来拼接 `[magnitude]` 和角度生成器。

---

## 完整 Vector v5 关键代码

```python
from array import array
import reprlib
import math
import functools
import operator
import itertools

class Vector:
    typecode = 'd'
    __match_args__ = ('x', 'y', 'z', 't')

    def __init__(self, components):
        self._components = array(self.typecode, components)

    def __iter__(self):
        return iter(self._components)

    def __repr__(self):
        components = reprlib.repr(self._components)
        components = components[components.find('['):-1]
        return f'Vector({components})'

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) + bytes(self._components))

    def __eq__(self, other):
        return len(self) == len(other) and all(a == b for a, b in zip(self, other))

    def __hash__(self):
        hashes = (hash(x) for x in self)
        return functools.reduce(operator.xor, hashes, 0)

    def __abs__(self):
        return math.hypot(*self)

    def __bool__(self):
        return bool(abs(self))

    def __len__(self):
        return len(self._components)

    def __getitem__(self, key):
        if isinstance(key, slice):
            cls = type(self)
            return cls(self._components[key])
        index = operator.index(key)
        return self._components[index]

    def __getattr__(self, name):
        cls = type(self)
        try:
            pos = cls.__match_args__.index(name)
        except ValueError:
            pos = -1
        if 0 <= pos < len(self._components):
            return self._components[pos]
        msg = f'{cls.__name__!r} object has no attribute {name!r}'
        raise AttributeError(msg)

    def __setattr__(self, name, value):
        cls = type(self)
        if len(name) == 1:
            if name in cls.__match_args__:
                error = 'readonly attribute {attr_name!r}'
            elif name.islower():
                error = "can't set attributes 'a' to 'z' in {cls_name!r}"
            else:
                error = ''
            if error:
                msg = error.format(cls_name=cls.__name__, attr_name=name)
                raise AttributeError(msg)
        super().__setattr__(name, value)

    def angle(self, n):
        r = math.hypot(*self[n:])
        a = math.atan2(r, self[n-1])
        if (n == len(self) - 1) and (self[-1] < 0):
            return math.pi * 2 - a
        return a

    def angles(self):
        return (self.angle(n) for n in range(1, len(self)))

    def __format__(self, fmt_spec=''):
        if fmt_spec.endswith('h'):
            fmt_spec = fmt_spec[:-1]
            coords = itertools.chain([abs(self)], self.angles())
            outer_fmt = '<{}>'
        else:
            coords = self
            outer_fmt = '({})'
        components = (format(c, fmt_spec) for c in coords)
        return outer_fmt.format(', '.join(components))

    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(memv)
```

---

## 总结：特殊方法与技术

| 类别 | 方法/技术 | 要点 |
|------|---------|------|
| 序列协议 | `__len__`, `__getitem__` | 最小序列实现 |
| 切片支持 | `isinstance(key, slice)` | 切片返回同类型实例 |
| 安全索引 | `operator.index()` | 拒绝 float 等非整数索引 |
| 动态属性 | `__getattr__` + `__setattr__` | 必须成对实现 |
| 聚合散列 | `functools.reduce` + `operator.xor` | map-reduce 模式 |
| 高效比较 | `zip` + `all` | 惰性求值，提前退出 |
| 安全 repr | `reprlib.repr()` | 大型集合截断显示 |
| 格式扩展 | `__format__` + `itertools.chain` | 超球面坐标 `'h'` 后缀 |

- 实现 `__getattr__` 就要实现 `__setattr__`
- `reduce` 要提供 `initializer` 防止空序列错误
- 切片应返回同类型实例
- `repr()` 永远不应抛异常
