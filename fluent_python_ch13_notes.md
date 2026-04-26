# Ch 13: Interfaces, Protocols, and ABCs

## 四种类型检查方式（Typing Map）

| | 结构类型 (Structural) | 名义类型 (Nominal) |
|---|---|---|
| **运行时检查** | **Duck typing** — 不用 `isinstance`，只看方法 | **Goose typing** — 用 `isinstance` 检查 ABC |
| **静态检查** | **Static duck typing** — `typing.Protocol` (Python ≥ 3.8) | **Static typing** — 传统类型注解 (Python ≥ 3.5) |

四种方式互补，不应排斥任何一种。

---

## 一、两种协议（Two Kinds of Protocols）

### 动态协议 (Dynamic Protocol)
- Python 自始至终的非正式协议，靠**约定和文档**定义
- 对象可以**只实现部分**协议仍然可用
- 静态类型检查器**无法验证**

### 静态协议 (Static Protocol)
- PEP 544 引入，通过 `typing.Protocol` 子类**显式定义**
- 必须实现协议类中声明的**所有方法**
- 可被静态类型检查器验证

**共同特征**：类永远不需要通过继承来声明它支持某个协议。

---

## 二、Programming Ducks（动态协议实践）

### Python 对序列协议的特殊支持

```python
class Vowels:
    def __getitem__(self, i):
        return 'AEIOU'[i]

v = Vowels()
v[0]        # 'A' — 索引访问
for c in v: print(c)  # 可迭代（回退机制：用 __getitem__ 从 0 开始）
'E' in v    # True — 可用 in（回退：顺序扫描）
```

**关键点**：只实现 `__getitem__` 就能获得索引、迭代、`in` 操作符支持。Python 解释器会尽力配合。

### Monkey Patching：运行时实现协议

`FrenchDeck` 只有 `__getitem__` 和 `__len__`，不能 shuffle（缺少 `__setitem__`）：

```python
def set_card(deck, position, card):
    deck._cards[position] = card

FrenchDeck.__setitem__ = set_card  # 运行时注入方法
shuffle(deck)  # 现在可以了！
```

`random.shuffle` 不关心对象的类，只需要对象实现可变序列协议的方法。

### 防御性编程与 Fail Fast (尽早报错)

**接受序列参数时**：不要做类型检查，直接转换：
```python
def __init__(self, iterable):
    self._balls = list(iterable)  # 不可迭代会立即报 TypeError
```

**处理字符串或可迭代对象**（EAFP 风格）：
```python
try:
    field_names = field_names.replace(',', ' ').split()  # 当字符串处理
except AttributeError:
    pass  # 不是字符串，假设已经是可迭代的名称序列
field_names = tuple(field_names)
```

---

## 三、Goose Typing（鹅类型）

**核心思想**：用 ABC（抽象基类）来定义接口，然后用 `isinstance()` 检查。

### 鹅类型 vs 纯鸭子类型

**纯鸭子类型的问题**：

```python
class Artist:
    def draw(self): print("画画")

class Gunslinger:
    def draw(self): print("拔枪")

# 两个都有 draw()，但完全不同的意思
# 鸭子类型分不出区别
```

**鹅类型的解决方案**：

```python
from collections.abc import Sequence

def process(data):
    if isinstance(data, Sequence):  # 检查接口，不检查具体类
        print(f"这是个序列")
        return data[0]

process([1, 2, 3])   # ✅ list 是 Sequence
process((1, 2, 3))   # ✅ tuple 也是 Sequence
process("hello")     # ✅ str 也是 Sequence
process({1: 2})      # ❌ dict 不是 Sequence，报错
```

### 两种使用 ABC 的方式

#### 方式 1：继承 ABC

```python
from collections.abc import Sequence

class MyList(Sequence):
    def __getitem__(self, i):
        return ...

    def __len__(self):
        return ...

    # 自动获得：__iter__, __contains__, index(), count() 等

# 现在能用
isinstance(MyList(), Sequence)  # True
```

**代价**：必须实现所有抽象方法
**好处**：免费获得很多现成的方法

#### 方式 2：虚拟子类（不继承，只声明）

```python
from collections.abc import Sequence

class WeirdList:
    def __getitem__(self, i): ...
    def __len__(self): ...
    # 不继承任何东西

# 告诉 Python："信任我，我实现了 Sequence"
Sequence.register(WeirdList)

# 现在：
isinstance(WeirdList(), Sequence)  # ✅ True
# 但 WeirdList 还是不继承任何方法！
```

**优势**：不需要改自己的代码，一行 `register()` 就搞定


### 标准库的 ABC 有哪些？

最常用的在 `collections.abc` 里：

```python
Sequence      # 序列（list, tuple, str 都是）
Mapping       # 字典类（dict）
Set           # 集合（set）
Iterable      # 可迭代（for 循环要求）
Callable      # 可调用（函数、对象等）
```

**简单规则：** 如果你的函数需要某种数据结构，就检查 ABC，而不是具体的类

```python
# ❌ 不要这样（太严格）
if type(data) is list:
    ...

# ✅ 要这样（更灵活）
if isinstance(data, Sequence):
    ...
```


### 自定义 ABC 的简单例子

大多数情况你**不需要**自定义 ABC，但如果要写框架：

```python
import abc

class Drawable(abc.ABC):
    @abc.abstractmethod
    def draw(self):
        """必须实现这个方法"""
        pass

# 用户的代码
class Circle(Drawable):
    def draw(self):
        print("画圆形")

# 框架的代码
def render(shape):
    if not isinstance(shape, Drawable):
        raise TypeError("必须是 Drawable 对象")
    shape.draw()

render(Circle())  # ✅ 工作
```

**关键点**：`@abc.abstractmethod` 标记"子类必须实现这个方法"

### 自动识别（无需声明）

某些 ABC 甚至不需要你主动注册：

```python
class MyClass:
    def __len__(self):
        return 42

from collections import abc
isinstance(MyClass(), abc.Sized)  # ✅ True！
# 因为你有 __len__，Python 自动认你是 Sized
```

### ⚠️ 两个警告

**1. 不要过度使用 isinstance 检查**

```python
# ❌ 坏
if isinstance(obj, Dog):
    obj.bark()
elif isinstance(obj, Cat):
    obj.meow()

# ✅ 好
class Animal:
    def make_sound(self): ...

obj.make_sound()  # 不管是什么，都调用这个方法
```

**2. 不要在生产代码中自定义 ABC**

- 除非你在写框架/库
- 日常代码？直接用现成的 ABC


## 四、Static Protocols（静态协议）

### 基本用法：给 `double()` 加类型注解

```python
from typing import TypeVar, Protocol

T = TypeVar('T')

class Repeatable(Protocol):
    def __mul__(self: T, repeat_count: int) -> T: ...

RT = TypeVar('RT', bound=Repeatable)

def double(x: RT) -> RT:
    return x * 2
```

类无需继承 `Repeatable`，只需实现 `__mul__` 即可通过类型检查。

### `@runtime_checkable` 运行时检查

```python
from typing import runtime_checkable, Protocol

@runtime_checkable
class RandomPicker(Protocol):
    def pick(self) -> Any: ...

class SimplePicker:  # 不继承 RandomPicker
    def pick(self) -> Any: return self._items.pop()

isinstance(SimplePicker([1]), RandomPicker)  # True
# 最好还是
try:
    c = complex(obj)
except TypeError:
    raise TypeError("obj must be convertible to complex")

# 或者更简洁：
c = complex(obj)
```

### 运行时检查的局限性

- 只检查方法**是否存在**，不检查签名和返回类型
- Python 3.9 中 `complex` 有 `__float__` 方法（但只是为了抛 TypeError），导致 `isinstance(3+4j, SupportsFloat)` 误报为 `True`

### 设计静态协议的最佳实践

1. **窄协议优先**：单方法协议最灵活（借鉴 Go 经验）
2. **在客户端代码附近定义**：便于扩展和测试
3. **命名约定**：
   - 清晰概念 → 直接命名（`Iterator`, `Container`）
   - 提供可调用方法 → `SupportsX`（`SupportsInt`）
   - 有可读/写属性 → `HasX`（`HasItems`）
4. **扩展协议**时派生新协议，而非修改原协议：

```python
@runtime_checkable
class LoadableRandomPicker(RandomPicker, Protocol):  # 必须显式写 Protocol
    def load(self, Iterable) -> None: ...
```

> 注意：`@runtime_checkable` 不被继承，需重新标注；必须显式列出 `Protocol` 作为基类。

---

## 五、数值类型的类型检查

| 方式 | 运行时 | 静态检查 |
|---|---|---|
| `numbers` ABC (`numbers.Integral` 等) | ✅ 好用 | ❌ Mypy 不支持 |
| `typing` 协议 (`SupportsFloat` 等) | ⚠️ complex 相关有坑 | ✅ 好用 |
| `Union[float, Decimal, Fraction]` | ✅ | ✅ 但不支持第三方数值类型 |

---

## 要点总结

1. **Duck typing**：Python 默认方式，最灵活，解释器会尽力配合（如 `__getitem__` 回退迭代）
2. **Goose typing**：用 ABC + `isinstance` 做运行时检查，适合框架和插件架构
3. **Static duck typing**：`typing.Protocol` 让静态检查器理解鸭子类型，窄协议最有用
4. **防御性编程**：优先用 EAFP（try/except），而非 LBYL（isinstance/hasattr）
5. **Monkey patching**：强大但危险，可在运行时实现协议
6. **虚拟子类**：通过 `register` 声明实现 ABC 接口，不继承方法
7. 四种类型方式互补，大型 Python 项目中都会用到

```python
# 日常脚本 → Duck Typing
def process(items):
    for item in items:
        print(item)

# 框架/库的 API → Goose Typing + Static Protocol
from collections.abc import Sequence
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

def render(shape: Drawable) -> None:
    shape.draw()

# 大型项目 → Static Type Hints
def calculate(x: float, y: float) -> float:
    return x + y

# 特殊情况 → Monkey Patch
if not hasattr(MyClass, 'new_feature'):
    MyClass.new_feature = add_feature
```
