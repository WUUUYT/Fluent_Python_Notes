# Ch 5：Data Class Builders

> 数据类就像孩子——作为起点没问题，但要成为成熟的对象，它们需要承担一些责任。
> —— Martin Fowler & Kent Beck

---

## 三种数据类构建器
普通类的坏处
1. 每个字段要写三遍 (参数里、赋值左边、赋值右边)
2. 打印出来是垃圾 `# <coordinates.Coordinate object at 0x107142f10>`
3. 比较永远是 False，默认比较的是内存地址，不是内容

Python 提供了三种"偷懒"的方式来写只装数据、没什么逻辑的类，书里叫它"data class"（数据类）。

**为什么需要这些构建器？**
普通类不自动提供 `__repr__`、`__eq__` 等方法，三种构建器均会自动生成它们。


| 特性             |    `namedtuple`     |    `NamedTuple`     |           `@dataclass`           |
| ---------------- | :-----------------: | :-----------------: | :------------------------------: |
| 实例可变         |          ✗          |          ✗          |                ✓                 |
| class 语法       |          ✗          |          ✓          |                ✓                 |
| 转换为 dict      |    `x._asdict()`    |    `x._asdict()`    |     `dataclasses.asdict(x)`      |
| 获取字段名       |     `x._fields`     |     `x._fields`     |  `[f.name for f in fields(x)]`   |
| 获取默认值       | `x._field_defaults` | `x._field_defaults` | `[f.default for f in fields(x)]` |
| 获取字段类型     |         N/A         | `x.__annotations__` |       `x.__annotations__`        |
| 创建修改副本     |   `x._replace(…)`   |   `x._replace(…)`   |   `dataclasses.replace(x, …)`    |
| 运行时动态创建类 |   `namedtuple(…)`   |   `NamedTuple(…)`   | `dataclasses.make_dataclass(…)`  |


---

## ~~`collections.namedtuple`~~
有了打印和比较
```python
from collections import namedtuple
Coordinate = namedtuple('Coordinate', 'lat lon')
moscow = Coordinate(55.756, 37.617)
print(moscow)                          # Coordinate(lat=55.756, lon=37.617) ✓ 好看了
moscow == Coordinate(55.756, 37.617)   # True ✓ 比较内容了
```
---

## `typing.NamedTuple`

- 加了类型说明，支持 class 写法

```python
from typing import NamedTuple

Coordinate = NamedTuple('Coordinate', [('lat', float), ('lon', float)])
Coordinate = NamedTuple('Coordinate', lat=float, lon=float)

class Coordinate(NamedTuple):
    lat: float
    lon: float
    reference: str = 'WGS84'   # 带默认值的字段
    # 控制打印输出内容
    def __str__(self):          # 可以直接定义方法
        ns = 'N' if self.lat >= 0 else 'S'
        we = 'E' if self.lon >= 0 else 'W'
        return f'{abs(self.lat):.1f}°{ns}, {abs(self.lon):.1f}°{we}'
```

- 继承关系：`issubclass(Coordinate, tuple)` → `True`，但 `issubclass(Coordinate, NamedTuple)` → `False`. `NamedTuple` 通过**元类**机制工作，并非真正的父类
- 带类型注解的字段 `a`、`b` 成为**实例属性**（描述符），无类型注解的 `c = 'spam'` 仅为**类属性**

---

## `@dataclass`
```python
from dataclasses import dataclass

@dataclass(frozen=True)   # frozen=True 表示创建后不能修改，类似不可变
class Coordinate:
    lat: float
    lon: float

    def __str__(self):
        ...
```
`typing.NamedTuple`和`@dataclass`类的内部代码完全一样，区别只在第一行：
- `class Coordinate(NamedTuple):`
- `@dataclass + class Coordinate:`
实例可以修改，除非`@dataclass(frozen=True)`.
### 装饰器参数

```python
@dataclass(*, init=True, repr=True, eq=True, order=False, unsafe_hash=False, frozen=False)
```

| 参数          | 默认  | 说明                               |
| ------------- | ----- | ---------------------------------- |
| `init`        | True  | 生成 `__init__`                    |
| `repr`        | True  | 生成 `__repr__`                    |
| `eq`          | True  | 生成 `__eq__`                      |
| `order`       | False | 生成 `__lt__`、`__le__` 等比较方法 |
| `frozen`      | False | 禁止实例属性修改（模拟不可变）     |
| `unsafe_hash` | False | 强制生成 `__hash__`（谨慎使用）    |

> `frozen=True` + `eq=True` → 自动生成合适的 `__hash__`，实例可哈希。

### `field()`
不能直接使用list作为默认值。

```python
from dataclasses import dataclass, field

@dataclass
class ClubMember:
    name: str
    guests: list[str] = field(default_factory=list)   # 可变默认值必须用 factory
    athlete: bool = field(default=False, repr=False)  # 不显示在 repr 中
```

`field()` 的参数：

| 参数              | 默认 | 说明                                 |
| ----------------- | ---- | ------------------------------------ |
| `default`         | —    | 字段默认值（不可变类型用这个）                           |
| `default_factory` | —    | 0参数可调用，每次创建实例时调用      |
| `init`            | True | 是否包含在 `__init__` 参数中         |
| `repr`            | True | 是否包含在 `__repr__` 中             |
| `compare`         | True | 是否用于比较方法                     |
| `hash`            | None | 是否用于 `__hash__`                  |
| `metadata`        | None | 用户自定义元数据（不影响 dataclass） |

> `@dataclass` 会拒绝 `list`、`dict`、`set` 类型的字面量默认值（但不检查其他可变类型），必须使用 `default_factory`。

### `__post_init__`：初始化后处理
`@dataclass` 自动生成的 `__init__` 只会赋值，如果需要验证数据或计算衍生字段，用 `__post_init__`：
```python
@dataclass
class HackerClubMember(ClubMember):
    all_handles: ClassVar[set[str]] = set()  # 类属性，需用 ClassVar
    handle: str = ''

    def __post_init__(self):
    # __init__ 赋值完成后，自动调用这里
        cls = self.__class__
        if self.handle == '':
            self.handle = self.name.split()[0]
        if self.handle in cls.all_handles:
            raise ValueError(f'handle {self.handle!r} already exists.')
        cls.all_handles.add(self.handle)
```

### 类属性 vs 实例属性

- 普通类型注解 → 实例字段（被 `@dataclass` 处理）
- `ClassVar[T]` 注解 → 类属性（`@dataclass` 忽略）
- `InitVar[T]` → 仅在 `__init__` 和 `__post_init__` 中出现，不成为实例属性

```python
from typing import ClassVar
from dataclasses import InitVar

@dataclass
class C:
    i: int
    j: int = None
    database: InitVar[DatabaseType] = None   # 不生成实例属性

    def __post_init__(self, database):       # database 作为参数传入
        if self.j is None and database is not None:
            self.j = database.lookup('j')
```

### Example
```python
from dataclasses import dataclass, field, fields
from typing import Optional
from enum import Enum, auto
from datetime import date

class ResourceType(Enum):
    BOOK = auto()
    EBOOK = auto()
    VIDEO = auto()

@dataclass
class Resource:
    identifier: str                                    # 唯一必填
    title: str = '<untitled>'                          # 之后的字段都要有默认值
    creators: list[str] = field(default_factory=list)
    date: Optional[date] = None                        # 可以是日期或 None
    type: ResourceType = ResourceType.BOOK
    description: str = ''
    language: str = ''
    subjects: list[str] = field(default_factory=list)

    def __repr__(self):
        cls_name = self.__class__.__name__
        indent = '    '
        res = [f'{cls_name}(']
        for f in fields(self):                        # 遍历所有字段
            value = getattr(self, f.name)             # 取字段值
            res.append(f'{indent}{f.name} = {value!r},')
        res.append(')')
        return '\n'.join(res)
```
---

## 类型注解（Type Hints）

- 类型注解**不在运行时生效**，Python 解释器完全忽略类型检查
- 用于静态分析工具（Mypy、PyCharm 等）
- 不应直接读 `__annotations__`，推荐：
  - Python 3.10+：`inspect.get_annotations(MyClass)`
  - Python 3.5-3.9：`typing.get_type_hints(MyClass)`

### 注解语法
```python
name: str               # 字符串类型
age: int                # 整数类型
score: float            # 小数类型
tags: list[str]         # 字符串列表
score: Optional[float]  # float 或者 None 都行
name: str = 'unknown'   # 有默认值
```

### 三种class中type annotations的行为差异
```python
# 1. 普通类
class DemoPlainClass:
    a: int          # 仅存入 __annotations__，无类属性
    b: float = 1.1  # 存入 __annotations__ + 类属性 b=1.1
    c = 'spam'      # 仅类属性，无注解

# 2. NamedTuple
class DemoNTClass(NamedTuple):
    a: int          # 注解 + 实例属性（只读描述符）
    b: float = 1.1  # 注解 + 实例属性（默认值 1.1）
    c = 'spam'      # 仅类属性，无注解

# 3. @dataclass
@dataclass
class DemoDataClass:
    a: int          # 注解 + 实例属性（a 在类上不存在，但创建实例后存在于实例上）
    b: float = 1.1  # 注解 + 实例属性（类级别 b=1.1 作为默认值）
    c = 'spam'      # 仅类属性，无注解
```

---

## Pattern Matching
Python 3.10 引入的 `match/case`，类似其他语言的 switch，但功能强大得多：
```python
match 某个值:
    case 模式1:
        做这个
    case 模式2:
        做那个
```

### Simple Class Pattern：只检查类型
```python
match x:
    case float():       # ✅ 正确：检查 x 是不是 float 类型
        print('是小数')
match x:
    case float:         # ❌ 危险！这不是检查类型
        print('永远执行')  # Python 把 float 当变量名，x 赋值给它
```
- 没括号是把 float 当普通变量，会匹配任何东西。

可以匹配并捕获值：
```python
case [str(name), _, _, (float(lat), float(lon))]:
# 匹配一个四元素列表，同时：
# - 第一个元素必须是 str，赋值给 name
# - 最后一个必须是含两个 float 的元组，赋值给 lat 和 lon
```
这种 str(name)、float(lat) 的特殊语法只对9种内置类型有效：`bytes dict float frozenset int list set str tuple`


### Keyword Class Patterns: 按属性名匹配
```python
class City(typing.NamedTuple):
    continent: str
    name: str
    country: str
match city:
    case City(continent='Asia'):              # 匹配亚洲城市
        results.append(city)
match city:
    case City(continent='Asia', country=cc): # 同时捕获 country
        results.append(cc)
```
- 用属性名，顺序无所谓
- 只需要写你关心的属性，其他属性不影响匹配

### Positional Class Patterns

```python
match city:
    case City('Asia'):             # 匹配第一个属性 == 'Asia'
        results.append(city)
match city:
    case City('Asia', _, country): # 匹配第一个属性，捕获第三个属性
        results.append(country)
```

位置模式依赖 `__match_args__`（由类构建器自动创建）：

```python
City.__match_args__   # ('continent', 'name', 'country')
```

> 9种内置类型（`bytes`、`dict`、`float`、`frozenset`、`int`、`list`、`set`、`str`、`tuple`）支持特殊的简单模式，如 `float(x)` 将 x 绑定到整个匹配值。

---

## Data Class 作为代码坏味道

Martin Fowler 在《重构》中指出：**只有字段和 getter/setter 而没有行为的类**是一种代码坏味道。**合理使用场景（不算坏味道）：**

1. **脚手架（Scaffolding）**：项目初期快速搭建，后续迁移行为到类中
2. **中间数据表示**：用于 JSON 导入/导出的临时数据容器，此时应作为**不可变对象**处理

**核心原则**：面向对象编程要求数据和操作数据的行为放在同一个类中。

---

## 速查：如何选择构建器

```
需要不可变实例？
├── 是 → namedtuple 或 NamedTuple
│        需要类型注解/方法/文档字符串？
│        ├── 是 → NamedTuple（class 语法更清晰）
│        └── 否 → collections.namedtuple（最简单）
└── 否 → @dataclass（功能最丰富，可配置 frozen/order 等）
```
