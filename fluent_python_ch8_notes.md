# Ch 8: Type Hints in Functions

> "Python will remain a dynamically typed language, and the authors have no desire to ever make type hints mandatory, even by convention." — PEP 484



## 渐进式类型 (Gradual Typing)

Python 的类型提示系统是**渐进式**的，具有三个核心特性：

- **可选的 (Optional)**：类型检查器对没有类型提示的代码默认假设为 `Any` 类型
- **不在运行时捕获类型错误**：类型提示仅供静态分析工具使用，运行时被忽略
- **不提升性能**：当前没有任何 Python 运行时利用类型注解进行优化

> 如果加标注反而让代码变复杂、API 变难用，直接不写
> 不要追求"100% 类型覆盖率"，这会导致为了满足指标而乱写标注，适得其反

---

## 实践入门：从 Mypy 开始
mypy 是什么？ 一个命令行工具，读你的 Python 代码后告诉你哪里类型不对。

**无类型提示的函数**
显示`Success: no issues found`, Mypy 默认忽略没有注解的函数。
```python
def show_count(count, word):
    if count == 1:
        return f'1 {word}'
    count_str = str(count) if count else 'no'
    return f'{count_str} {word}s'
```
**逐步添加类型提示**
使用 `--disallow-incomplete-defs`, 意思是："如果已经开始写类型标注了，就必须写完整。"
```python
# 第一步：先加返回类型，触发 Mypy 检查该函数
def show_count(count, word) -> str:
# error: Function is missing a type annotation for one or more arguments
# 第二步：补全参数类型
def show_count(count: int, word: str) -> str:
```
`--disallow-untyped-defs`，意思是："所有函数必须有完整类型标注。"

> 写 mypy.ini 配置文件：
```ini
ini[mypy]
python_version = 3.9
warn_unused_configs = True
disallow_incomplete_defs = True
```
>以后直接跑 mypy ***.py 就自动带上这些设置。

**带默认参数值的类型提示**
- `plural: str = ''`意思是：这个参数类型是 str，默认值是空字符串
- `plural: Optional[str] = None`意思就是：这个参数要么是 str，要么是 None。
- 更现代的写法（Python 3.10+）
  - `plural: str | None = None`
  - 用 `|` 表示"或"，不需要 import。
- `Optional[str]` 不代表参数可省略。是默认值 `= None` 让参数变成可选的。

### 类型提示代码风格

- 参数名和 `:` 之间无空格，`:` 之后一个空格：`name: str`
- 有类型提示时，`=` 两侧有空格：`plural: str = ''`
- 无类型提示时，`=` 两侧无空格：`plural=''`

---

## 什么是类型
类型由支持的操作定义，而非名字
```python
def double(x):
    return x * 2
```
`x`是什么类型：任何支持 * 2 操作的东西都行——int、str、list，甚至 numpy 数组，全都可以。
- **鸭子类型 (Duck typing)**：运行时只看是否支持某个操作，不关心声明类型
  - 错误在运行时才暴露；灵活，但危险
- **名义类型 (Nominal typing)**：静态类型检查器 只看类型声明
  - 错误在写代码时就提示；严格，但能提前发现 bug

### 对比示例

```python
class Bird:
    pass
class Duck(Bird):
    def quack(self):
        print('Quack!')
def alert(birdie):          # 无类型提示，Mypy 忽略
    birdie.quack()
def alert_duck(birdie: Duck) -> None:  # 接受 Duck
    birdie.quack()
def alert_bird(birdie: Bird) -> None:  # 接受 Bird
    birdie.quack()  # ❌ Mypy 报错："Bird" has no attribute "quack"
```

| 调用 | 运行时 | Mypy |
|------|--------|------|
| `alert(Duck())` | ✅ 正常 | 不检查（无类型提示） |
| `alert_duck(Duck())` | ✅ 正常 | ✅ 通过 |
| `alert_bird(Duck())` | ✅ 正常 | ⚠️ 报 `alert_bird` 函数体的错误 |
| `alert_bird(Bird())` | ❌ `AttributeError` | ⚠️ 同上 |

---

## Types Usable in Annotations

### `Any`

```python
def double(x):          # 等同于 def double(x: Any) -> Any:
    return x * 2
def double(x: object) -> object:
    return x * 2  # ❌ Mypy 报错：object 不支持 * 操作
```

- `Any` 兼容所有类型，支持所有操作（类型检查器不会报错）
- `object` 也接受所有类型，但只支持 `object` 定义的操作
  - 这是所有类型的基类，被严格检查
- 类型越通用，支持的操作越少


**`subtype-of` and `consistent-with`**
两种兼容关系：
- **subtype-of**：子类可以替换父类。Duck 继承自 Bird，所以 Duck is subtype-of Bird，凡是要求传 Bird 的地方，传 Duck 也行。反过来不行。
- **consistent-with**：渐进式类型系统专有的概念，在子类型关系基础上，额外给 Any 开了绿灯
  1. 子类型关系照旧成立（Duck consistent-with Bird）
  2. 任何类型都 consistent-with Any（传什么给 Any 参数都行）
  3. Any consistent-with 任何类型（Any 的值传给任何参数都行：函数返回值没标注默认为`Any`，又传入另外一个函数）

### Simple Types and Classes

`int`、`float`、`str`、`bytes` 以及具体类、ABC 都可直接用于类型提示。
- `int` consistent-with `float`，`float` consistent-with `complex`（虽然它们不是继承关系）。

**Optional 和 Union**
- `Optional[str]` 等同于 `Union[str, None]`
- 尽量避免返回 Union 类型，否则调用者需要检查返回值类型。
```python
from typing import Optional, Union
def show_count(count: int, singular: str, plural: Optional[str] = None) -> str:
# Union 示例
def ord(c: Union[str, bytes]) -> int: ...
# Python 3.10+ 新语法
plural: str | None = None       # 替代 Optional[str]
isinstance(x, int | str)        # 替代 isinstance(x, (int, str))
```

### Generic Collections: `list[str]`
```python
# Python ≥ 3.9 直接使用内置类型
def tokenize(text: str) -> list[str]:
    return text.upper().split()
```
```python
# Python 3.7-3.8 需要 __future__ import
    from __future__ import annotations
    def tokenize(text: str) -> list[str]: ...
# Python 3.5-3.6 使用 typing 模块
    from typing import List
    def tokenize(text: str) -> List[str]: ...
```

支持泛型的标准库集合（`container[item]` 形式）：

| 类型 | 泛型写法 |
|------|---------|
| `list` | `list[str]` |
| `set` / `frozenset` | `set[int]` |
| `collections.deque` | `deque[float]` |
| `abc.Sequence` | `Sequence[str]` |
| `abc.MutableSequence` | `MutableSequence[int]` |

### Tuple Types

三种用法：
1. 固定字段类型，记录用：tuple 里每个位置有明确含义，比如 (城市, 人口, 国家)
    ```python
    city: tuple[str, float, str] = ('Shanghai', 24.28, 'China')
    ```
2. 元组作有命名字段的记录
    ```python
    from typing import NamedTuple
    class Coordinate(NamedTuple):
        lat: float
        lon: float
    ```
   - Coordinate 本质上还是 tuple[float, float] 的子类，所以要求 tuple[float, float] 的地方可以传 Coordinate
3. 元组作不可变序列（不定长度）
   - `Tuple` 每个元素类型相同时，用`...` 表示任意长度
        >`list[str]` 列表本来就是可变长度的，不需要额外声明任意长度。
    ```python
    numbers: tuple[int, ...]       # 任意长度的 int 元组
    anything: tuple[Any, ...]      # 等同于 tuple
    ```

### Generic Mappings

```python
dict[str, int]           # 键是str，值是int
dict[str, set[str]]      # 键是str，值是str的集合
dict[int, list[float]]   # 键是int，值是float列表
def name_index(start: int = 32, end: int = STOP_CODE) -> dict[str, set[str]]:
    index: dict[str, set[str]] = {}  # 局部变量也可以加类型标注
    ...
```

### ABCs (Abstract Base Classes)

> "Be conservative in what you send, be liberal in what you accept." — Postel's law
> 接受参数时，要尽量宽松；返回结果时，要尽量具体

参数类型 → 用抽象类型（宽松），让调用方可以传更多种类的对象进来。
返回类型 → 用具体类型（严格），让调用方明确知道拿到的是什么。

```python
from collections.abc import Mapping

# ✅ 好：接受 dict, defaultdict, ChainMap, UserDict 等
def name2hex(name: str, color_map: Mapping[str, int]) -> str: ...

# ❌ 差：UserDict 不是 dict 的子类，会被拒绝
def name2hex(name: str, color_map: dict[str, int]) -> str: ...
```
`UserDict` 和 `dict` 都继承自 `abc.MutableMapping`

**该用 Mapping 还是 MutableMapping？** 取决于函数有没有修改传入的字典：
```python
# 只读取 → 用 Mapping（不需要有修改的方法，更宽松）
def name2hex(name: str, color_map: Mapping[str, int]) -> str:
    return color_map[name]
# 需要修改 → 用 MutableMapping
def register(name: str, color_map: MutableMapping[str, int]) -> None:
    color_map[name] = 0xFF0000
```
- 用 Mapping 的好处是：调用方不需要传一个支持 setdefault、pop、update 这些修改操作的对象，只要能读就行，限制更少。

| 类型 | 支持的操作 | 参数标注建议 |
|------|---------|---------|
|`abc.Mapping`|只读：`[]`取值、`get`、`keys`、`items`…|函数只读时用这个|
|`abc.MutableMapping`|读写：上面的+`setdefault`、`pop`、`update`…|函数需要修改时用这个|
|`dict`|读写 + `dict` 专有方法|尽量别用作参数类型|


**数字塔的问题**：`numbers` 模块的 ABC（Number, Complex, Real 等）不被静态类型检查支持。替代方案：
1. 使用具体类型 `int`/`float`/`complex`
2. 使用 `Union[float, Decimal, Fraction]`
3. 使用 `SupportsFloat` 等协议

### Iterable


`Iterable` vs `Sequence`
- Sequence：必须是有长度、可下标访问的对象
  - list、tuple、str 可以，但 generator 不行
  - 所以如果要获取长度`len()`，就只能用`Sequence`
- Iterable：只要能 for 循环遍历就行
  - list、tuple、str、generator、文件对象……都可以
- 都不推荐作为范围类型，要具体，如`list`
```python
from collections.abc import Iterable
FromTo = tuple[str, str] # 类型别名提高可读性
def zip_replace(text: str, changes: Iterable[FromTo]) -> str:
    for from_, to in changes:
        text = text.replace(from_, to)
    return text
```
```python
# Python 3.10+ 显式类型别名
from typing import TypeAlias
FromTo: TypeAlias = tuple[str, str]
```
### Parameterized Generics and TypeVar

`TypeVar` 让参数类型与返回类型关联，但不需要显式声明：
- 调用 `sample` 时，
  - 如果传入 `tuple[int, ...]`，返回 `list[int]`
  - 传入 `str`，返回 `list[str]`。
```python
from typing import TypeVar
from collections.abc import Sequence

T = TypeVar('T')

def sample(population: Sequence[T], size: int) -> list[T]:
    result = list(population)
    shuffle(result)
    return result[:size]
```

**Restricted TypeVar**
- T 只能被绑定为括号里列出的某一种，传其他类型 mypy 报错。
```python
NumberT = TypeVar('NumberT', float, Decimal, Fraction)

def mode(data: Iterable[NumberT]) -> NumberT:  # 只接受这三种类型之一
```

**Bounded TypeVar** (最推荐，最灵活又有约束)
- T 可以是 bound 指定的类型，或它的任意子类——不用穷举所有类型。
- 约束 T 必须满足某个条件（如：可哈希），但 T 本身还是会被推断为具体类型（int、str……），不会退化成 Hashable 这个抽象类型。
```python
from collections.abc import Hashable
HashableT = TypeVar('HashableT', bound=Hashable)
def mode(data: Iterable[HashableT]) -> HashableT:  # 接受任何 Hashable 的子类型
```

**AnyStr**
typing 里预定义好的 TypeVar，直接可以用
```python
AnyStr = TypeVar('AnyStr', bytes, str)  # 内置于 typing 模块
```

### Static Protocols
问题：要求传入的类型支持某个特定操作（比如`<`），但没有现成类型。
`typing.Protocol` 实现**静态鸭子类型**：
- 不需要继承或注册，只要实现了协议定义的方法就算兼容。
- 三个点，不需要实现，只声明接口
- 任何实现了 `__lt__` 的类型（`int`、`str`、`tuple` 等）都自动兼容 `SupportsLessThan`，无需显式声明。
```python
from typing import Protocol, Any

class SupportsLessThan(Protocol):
    def __lt__(self, other: Any) -> bool: ...
```

```python
from typing import TypeVar
from collections.abc import Iterable

LT = TypeVar('LT', bound=SupportsLessThan)

def top(series: Iterable[LT], length: int) -> list[LT]:
    ordered = sorted(series, reverse=True)
    return ordered[:length]
```

### Callable

```python
from collections.abc import Callable
# Callable[[参数类型列表], 返回类型]
Callable[[int, str], bool]

# 接受一个 Any 参数，返回 str 的函数
input_fn: Callable[[Any], str]
# 不接受任何参数，返回 float 的函数
probe: Callable[[], float]
# 接受一个 float 参数，返回 None 的函数
display: Callable[[float], None]
# 参数随意，只关心返回类型
callback: Callable[..., str]
```

**Variance in Callable**

```python
def update(
    probe: Callable[[], float],       # 返回 float
    display: Callable[[float], None]  # 接受 float
) -> None: ...

def probe_ok() -> int: ...            # ✅ 返回 int（int 是 float 的子类型）
def display_wrong(t: int) -> None: ...  # ❌ 接受 int 但要求能处理 float
def display_ok(t: complex) -> None: ... # ✅ 接受 complex（能处理 float）
```

- **返回类型是协变的 (covariant)**：`Callable[[], int]` 是 `Callable[[], float]` 的子类型
  - 自类型可接受
- **参数类型是逆变的 (contravariant)**：`Callable[[float], None]` 是 `Callable[[int], None]` 的子类型
  - 父类型可接受

### NoReturn

用于标注永远不会正常返回的函数（通常抛出异常）：

```python
from typing import NoReturn
def stop() -> NoReturn:
    raise RuntimeError("停止运行")
```

---

## Annotating Positional Only & Variadic Parameters
- 标注 *args 和 **kwargs 时，写的类型是单个元素的类型，不是整体容器的类型
- 如果 **kwargs 的值类型不统一，就用 Any
```python
from typing import Optional

def tag(
    name: str,
    /,                            # 之前的都是仅限位置参数
    *content: str,                # 可变位置参数 → 函数内类型为 tuple[str, ...]
    class_: Optional[str] = None,
    **attrs: str,                 # 可变关键字参数 → 函数内类型为 dict[str, str]
) -> str:
```

Python 3.7 及更早版本使用双下划线前缀表示仅限位置参数：`__name: str`。

---

## Imperfect Typing and Strong Testing

静态类型检查的局限：

- **假阳性**：工具报告实际正确代码的类型错误
- **假阴性**：工具未能报告实际错误代码的类型错误
- `config(**settings)` 等解包、元类、描述符这些高级用法，类型检查器跟不上。
- 跟不上新版本，Python 出了新语法，类型检查器可能要等一年多才能支持
- 无法表达业务约束（如"数量必须 > 0"）

> 静态类型检查器应被视为 CI 流水线中的工具之一，与测试运行器、linter 等并列，而非替代自动化测试。

---

## 要点速查

| 场景 | 推荐做法 |
|------|---------|
| 参数接受多种集合 | 用 ABC：`Mapping`、`Sequence`、`Iterable` |
| 返回集合 | 用具体类型：`list`、`dict`、`set` |
| 可选参数（不可变默认值） | `plural: str = ''` |
| 可选参数（可变默认值） | `items: Optional[list[str]] = None` |
| 输入输出类型关联 | 用 `TypeVar` |
| 约束操作（如要求可排序） | 用 `Protocol` + 有界 `TypeVar` |
| 回调函数 | `Callable[[Par数类型], 返回类型]` |
| 永不返回的函数 | `-> NoReturn` |

### 调试技巧

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    reveal_type(my_var)  # Mypy 专用，显示推断类型
```

`reveal_type()` 不是真正的函数，仅 Mypy 分析时输出类型信息，不可在运行时调用。

### Mypy 配置参考

```ini
# mypy.ini
[mypy]
python_version = 3.9
warn_unused_configs = True
disallow_incomplete_defs = True
```
