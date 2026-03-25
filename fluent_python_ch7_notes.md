# Ch 7: Functions as First-Class Objects

## First-Class Objects

满足以下条件的程序实体称为**first-class object**：
- 可在运行时创建 (嵌套函数)
  ```python
    def make_greeting(word):
        def greet(name):       # 在函数执行期间动态创建
            return f"{word}, {name}!"
        return greet
    hi = make_greeting("Hello")   # greet 在这里才被创建
    ```
- 可赋值给变量或数据结构中的元素
- 可作为参数传递给函数
- 可作为函数的返回值

Python 中，**所有函数都是一等对象**（整数、字符串、字典同理）。

---

## Function as Objects

```python
def factorial(n):
    """returns n!"""
    return 1 if n < 2 else n * factorial(n - 1)
# 查看文档字符串
factorial.__doc__       # 'returns n!'
# 查看类型
type(factorial)         # <class 'function'>
# 赋值给变量
fact = factorial
fact(5)                 # 120
# 作为参数传递
list(map(factorial, range(11)))
# [1, 1, 2, 6, 24, 120, 720, 5040, 40320, 362880, 3628800]
```
- `factorial` 是 `function` 类的一个实例，和 `42` 是 `int` 的实例、`"hello"` 是 `str` 的实例完全一样。
- 函数对象有属性，`__doc__` 只是其中之一。这意味着你可以像操作任何对象一样读取、修改函数的属性。
- fact = factorial：函数可以赋值给变量。fact 和 factorial 指向同一个函数对象，不是复制。

---

## Higher-Order Functions

Higher-Order Functions:
1. **接受函数作为参数**的函数
2. 或**返回函数作为结果**的函数

### 常用内置高阶函数

| 函数 | 说明 |
|------|------|
| `sorted(iterable, key=fn)` | 按 `key` 函数的返回值排序 |
| `map(fn, iterable)` | 对每个元素应用 `fn`，返回迭代器 |
| `filter(fn, iterable)` | 过滤元素，返回 `fn` 为真的元素迭代器 |
| `functools.reduce(fn, iterable)` | 累积计算，Python 3 移至 `functools` |

```python
fruits = ['strawberry', 'fig', 'apple', 'cherry', 'raspberry', 'banana']
# 按长度排序
sorted(fruits, key=len)
# ['fig', 'apple', 'cherry', 'banana', 'raspberry', 'strawberry']
# 按反转拼写排序
sorted(fruits, key=lambda word: word[::-1])
# ['banana', 'apple', 'fig', 'raspberry', 'strawberry', 'cherry']
```

### map / filter 的现代替代

**列表推导式**和**生成器表达式**通常比 `map`/`filter` 更可读：
- 列表推导式 (立即占用内存)
  - `my_list = [x * 2 for x in range(10)]`
- 生成器表达式 (省内存，惰性求值)
  - `my_gen = (x * 2 for x in range(10))`
```python
# 用 map + filter
list(map(factorial, filter(lambda n: n % 2, range(6))))  # [1, 6, 120]

# 列表推导（更清晰）
[factorial(n) for n in range(6) if n % 2]               # [1, 6, 120]
```

### reduce 的替代

```python
from functools import reduce
from operator import add

reduce(add, range(100))  # 4950
sum(range(100))          # 4950（更简洁，性能更好）
```

其他归约内置函数：
- `all(iterable)` — 所有元素为真返回 `True`（空序列返回 `True`）
- `any(iterable)` — 任一元素为真返回 `True`（空序列返回 `False`）

---

## Anonymous Functions: lambda

`lambda` 创建匿名函数:
- **函数体只能是纯表达式**（不允许 `while`、`try`、`=` 赋值等语句）
- 创建的函数对象和用 def 定义的完全一样，只是没有名字

```python
sorted(fruits, key=lambda word: word[::-1])
# 完全一样
def reverse(word): return word[::-1]
sorted(fruits, key=reverse)
```

**Fredrik Lundh 的 lambda 重构建议**

当 lambda 难以理解时，按以下步骤重构：

1. 写注释解释 lambda 做了什么
2. 根据注释想一个好名字
3. 用 `def` 定义同名函数
4. 删掉注释

> lambda 最好只用在高阶函数的参数位置，复杂逻辑应用 `def` 定义具名函数。

---

## 九种可调用对象（Callable Objects）

用 `callable()` 内置函数判断一个对象是否可调用。

| 类型 | 描述 |
|------|------|
| 用户定义函数 | `def` 或 `lambda` 创建 |
| 内置函数 | C 实现，如 `len`、`time.strftime` |
| 内置方法 | C 实现的方法，如 `dict.get` |
| 方法 | 类体中定义的函数 |
| 类 | 调用时执行 `__new__` 和 `__init__` |
| 类实例 | 定义了 `__call__` 方法的实例 |
| 生成器函数 | 包含 `yield` 的函数，调用返回生成器对象 |
| 原生协程函数 | `async def` 定义，调用返回协程对象（Python 3.5+） |
| 异步生成器函数 | `async def` + `yield`，配合 `async for` 使用（Python 3.6+） |
- 后3种和其他有本质区别：调用它们不会立即执行函数体，而是返回一个需要进一步驱动的对象。
  - 生成器要用 `next()` 或 `for` 驱动
  - 协程要用 `asyncio` 驱动。

**用户定义的可调用类型**

实现 `__call__` 方法，即可让类实例像函数一样被调用：

```python
import random

class BingoCage:
    def __init__(self, items):
        self._items = list(items)
        random.shuffle(self._items)

    def pick(self):
        try:
            return self._items.pop()
        except IndexError:
            raise LookupError('pick from empty BingoCage')

    def __call__(self):       # 让实例可直接调用
        return self.pick()
```

```python
bingo = BingoCage(range(3))
bingo.pick()      # 正常调用方法
bingo()           # 像函数一样调用
callable(bingo)   # True
```

**适用场景**：
1. 需要跨调用保持状态（如装饰器的记忆化缓存）。
2. 实现装饰器

---

## 函数参数

```python
def tag(name, *content, class_=None, **attrs):
    ...
```

| 参数形式 | 说明 |
|----------|------|
| `name` | 普通位置参数 |
| `*content` | 可变位置参数，捕获多余的位置参数为元组 |
| `class_=None` | 仅限关键字参数（keyword-only） |
| `**attrs` | 可变关键字参数，捕获多余的关键字参数为字典 |

**仅限关键字参数（Keyword-Only，Python 3+）**
放在 `*` 之后的参数只能通过关键字传递：

```python
def f(a, *, b):      # b 是 keyword-only，可以没有默认值（强制必填）
    return a, b

f(1, b=2)   # OK → (1, 2)
f(1, 2)     # TypeError
```
- `*` 单独出现时也只是分界线，不收集
- 强制调用方写清楚参数名，避免位置搞错

**仅限位置参数（Positional-Only，Python 3.8+）**

`/` 左侧的参数只能通过位置传递：

```python
def divmod(a, b, /):
    return (a // b, a % b)
```
- / 不收集任何参数
- 参数名是实现细节，不想暴露给外部
- 性能：内置函数（len、divmod 等）都是 positional-only，省去关键字匹配的开销。

### 参数顺序规则总结
- 普通参数: 签名里没有 `*` 和 `**` 修饰，不在 `/` 左边，不在 `*` 右边的参数
  - 可以用位置传，也可以用关键字传。

定义函数时，参数必须按这个顺序写：
```
positional-only | 普通参数 | *args | keyword-only | **kwargs
      /         |          |   *   |              |    **
def f(a, b,  /,  c, d, *args,  e, f,   **kwargs):
#     └──┘       └──┘  └───┘   └──┘    └──────┘
#    只能位置      普通   可变  keyword  可变关键字
```
---

## Packages for Functional Programming
### `operator` 模块

提供算术运算符对应的函数，避免写无聊的 lambda：

```python
from functools import reduce
from operator import mul
# 没有 operator：只能写 lambda
def factorial(n):
    return reduce(lambda a, b: a * b, range(1, n+1))

# 代替 lambda a, b: a*b
def factorial(n):
    return reduce(mul, range(1, n+1))
```

**`itemgetter`**

- 按索引取值（支持多索引，字典），适合排序
- 用的是 [] 运算符
```python
from operator import itemgetter

# 按第 1 个字段（国家代码）排序
for city in sorted(metro_data, key=itemgetter(1)):
    print(city)
# 提取多个字段
cc_name = itemgetter(1, 0)   # 返回 (field1, field0) 的元组
getter = itemgetter('name')
getter({'name': 'Alice', 'age': 30})   # 'Alice'
```

**`attrgetter`**
- 按属性名取值，用 . 取对象属性
- 支持点号嵌套访问
```python
from operator import attrgetter

name_lat = attrgetter('name', 'coord.lat')   # 支持嵌套属性
sorted(metro_areas, key=attrgetter('coord.lat'))
```

**`methodcaller`**
- 按名称调用方法，可预绑定参数
```python
from operator import methodcaller

upcase   = methodcaller('upper')
hyphenate = methodcaller('replace', ' ', '-')

upcase('hello world')      # 'HELLO WORLD'
hyphenate('hello world')   # 'hello-world'
```

### operator 模块的命名规律
```python
add    # a + b
mul    # a * b
sub    # a - b
mod    # a % b
neg    # -a

iadd   # a += b（in-place，如果对象可变就原地修改，否则等同于 add）
imul   # a *= b
isub   # a -= b

eq     # a == b
lt     # a < b
ge     # a >= b

and_   # a & b（加下划线是因为 and 是保留字）
or_    # a | b
not_   # not a
```
`i` 前缀对应 `+=` 这类增量赋值运算符
- 可变对象会原地修改
- 不可变对象则和不带 i 的版本效果相同。


### `functools.partial`
**核心思想**：把一个多参数函数，预先固定部分参数，生成一个参数更少的新函数。


```python
from operator import mul
from functools import partial

triple = partial(mul, 3)
triple(7)                        # 21
list(map(triple, range(1, 10)))  # [3, 6, 9, 12, 15, 18, 21, 24, 27]
```

**固定位置参数**：按顺序从左开始固定
```python
from operator import mul
triple = partial(mul, 3)
triple(7)    # mul(3, 7) = 21
triple(10)   # mul(3, 10) = 30
```
**固定关键字参数**
```python
# Unicode 标准化的例子
import unicodedata, functools
nfc = functools.partial(unicodedata.normalize, 'NFC')
# 等价于：每次调用 nfc(s) 就是调用 unicodedata.normalize('NFC', s)
s1 = 'café'
s2 = 'cafe\u0301'    # 看起来一样，但编码不同
s1 == s2             # False
nfc(s1) == nfc(s2)   # True，标准化后相等
```
**同时固定位置参数和关键字参数**
```python
# tag(name, *content, class_=None, **attrs)
picture = partial(tag, 'img', class_='pic-frame')
picture(src='wumpus.jpeg')
# 实际调用：tag('img', class_='pic-frame', src='wumpus.jpeg')
```


**`partial` 对象的属性**
partial 返回的是一个 functools.partial 对象，有三个属性可供检查：
```python
p = partial(tag, 'img', class_='pic-frame')
p.func      # 原始函数
p.args      # 已绑定的位置参数
p.keywords  # 已绑定的关键字参数
```

> `functools.partialmethod` 与 `partial` 相同，但专为**方法**设计。
> `Partialmethod` 专门解决了`self`处理的问题。
```python
from functools import partialmethod
class Button:
    def set_style(self, color, size):
        print(f"color={color}, size={size}")
    # 创建预固定了 color 的方法
    red   = partialmethod(set_style, color='red')
    large = partialmethod(set_style, size='large')
btn = Button()
btn.red(size='medium')    # color=red, size=medium
btn.large(color='blue')   # color=blue, size=large
```



---

## Summary

| 概念 | 要点 |
|------|------|
| 一等函数 | 可赋值、可传递、可存储、可返回 |
| 高阶函数 | `sorted`、`map`、`filter`、`reduce`、`functools.partial` |
| 现代替代 | 列表推导/生成器表达式替代 `map`/`filter`；`sum`/`all`/`any` 替代 `reduce` |
| 可调用类型 | 九种，均可用 `callable()` 检测 |
| `__call__` | 使任意类实例变为可调用对象 |
| 参数语法 | `*args`、`**kwargs`、keyword-only（`*` 后）、positional-only（`/` 前） |
| `operator` 模块 | `mul`、`itemgetter`、`attrgetter`、`methodcaller` |
| `functools.partial` | 冻结参数，适配需要更少参数的 API |
