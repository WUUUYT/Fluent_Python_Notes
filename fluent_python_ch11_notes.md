# Ch 11：A Pythonic Object

> 通过实现 Python 数据模型中的特殊方法，让自定义类的行为像内置类型一样自然。这不依赖继承，而是依赖鸭子类型——只需实现预期的方法即可。

---

## Object Representations

Python 提供多种方式来获取对象的字符串表示：

| 方法 | 触发方式 | 用途 |
|------|---------|------|
| `__repr__` | `repr()`、交互式控制台 | 面向开发者，应尽量可以用 `eval()` 重建对象 |
| `__str__` | `str()`、`print()` | 面向用户的友好表示 |
| `__bytes__` | `bytes()` | 二进制序列表示 |
| `__format__` | `format()`、f-string、`str.format()` | 自定义格式化输出 |

> Python 3 中 `__repr__`、`__str__`、`__format__` 必须返回 `str`（Unicode），只有 `__bytes__` 返回 `bytes`。

---

## Vector2d
```python
v1 = Vector2d(3, 4)
print(v1.x, v1.y)  # 3.0 4.0
x, y = v1  # (3.0, 4.0)

# __repr__ 的黄金准则：输出应该像构造该对象的源代码，让开发者一眼明白对象状态
v1_clone = eval(repr(v1))  # 可以用 eval 重新构建出相同对象
v1 == v1_clone             # True

print(v1)  # (3.0, 4.0)  ← 友好的元组形式
repr(v1)   # Vector2d(3.0, 4.0)  ← 精确的构造器形式
bytes(v1)  # b'd\x00\x00\x00\x00\x00\x00\x08@\x00\x00\x00\x00\x00\x00\x10@'
abs(v1)                    # 5.0（勾股定理：√(3²+4²)）
bool(v1), bool(Vector2d(0,0))  # (True, False)
```
**Source code**
```python
from array import array
import math

class Vector2d:
    typecode = 'd'  # 类属性，'d' 代表双精度浮点数, 用于 bytes 转换

    def __init__(self, x, y):
        self.x = float(x)  # 提前转换，尽早捕获错误
        self.y = float(y)

    def __iter__(self):
        return (i for i in (self.x, self.y))  # 支持解包: x, y = v1

    def __repr__(self):
        class_name = type(self).__name__  # 用 type(self) 而非硬编码，子类继承后 repr 依然正确
        return '{}({!r}, {!r})'.format(class_name, *self)
        # {!r} 表示对参数调用 repr()，确保字符串类型有引号，数字类型精确显示
        # *self 利用了 __iter__，自动展开为 x, y

    def __str__(self):
        return str(tuple(self))  # 输出 (3.0, 4.0)

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) + # 类型码字节
                bytes(array(self.typecode, self))) # 数据字节

    def __eq__(self, other):
        return tuple(self) == tuple(other)  # 注意：会与任何相同数值的可迭代对象相等

    def __abs__(self):
        return math.hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))  # 零向量为 False
```
- `__iter__` 使用生成器表达式，使 Vector2d 可迭代，从而支持解包
- `__repr__` 中用 `*self` 解包，利用了 `__iter__`
- `__eq__` 基于 tuple 比较，简单但有副作用（`Vector2d(3, 4) == [3, 4]` 为 True）

---

## An Alternative Constructor：@classmethod

```python
@classmethod
def frombytes(cls, octets):
    typecode = chr(octets[0])
    memv = memoryview(octets[1:]).cast(typecode)
    return cls(*memv)
```
- `@classmethod` 装饰器将方法绑定到类本身而不是实例。这意味着可以直接通过类调用，不需要先创建实例
- octets[0] 是第一个字节的整数值, `__bytes__` 的实现中，第一个字节存的就是 typecode（'d'），这里把它读回来。
- `memoryview` 是 Python 内置类型，允许在**不复制数据**的情况下访问字节序列的内存；`octets[1:]` 跳过第一个字节（类型码），只取数据部分
- `.cast(typecode)`把原始字节重新解释为指定类型（这里是 `'d'` = double）
- `cls(*memv)` 重建对象
### classmethod vs staticmethod

| 特性 | `@classmethod` | `@staticmethod` |
|------|---------------|-----------------|
| 第一个参数 | `cls`（类本身） | 无特殊参数 |
| 主要用途 | 备选构造器 | 很少有好的使用场景 |
| 推荐度 | 非常有用 | 通常用模块级函数替代更好 |

---

## Formatted Displays
Python 中三种格式化方式，本质上都是调用对象的 __format__ 方法
```python
format(obj, '0.4f')          # 直接调用
f'{obj:0.4f}'                # f-string
'{:0.4f}'.format(obj)        # str.format()

# 三种写法最终都等价于：
obj.__format__('0.4f')
```
- `'0.4f'` 就叫做 **格式规格说明符（format_spec）**，是冒号后面的部分。
  - 冒号左边：字段名，决定取哪个变量/属性
  - 冒号右边：格式规格说明符，决定怎么显示

### 内置类型的格式化代码
各类型独立解释自己的格式规格说明符——这就是格式规格说明符迷你语言（Format Specification Mini-Language）是可扩展的。
```python
# int：支持进制转换
format(42, 'b')    # '101010'  ← 二进制
format(42, 'x')    # '2a'      ← 十六进制
format(42, 'o')    # '52'      ← 八进制

# float：支持小数格式
format(0.667, '.1%')   # '66.7%'   ← 百分比
format(0.667, '.2f')   # '0.67'    ← 定点小数
format(0.667, '.2e')   # '6.67e-01'← 科学计数

# datetime：支持时间格式
from datetime import datetime
now = datetime.now()
format(now, '%H:%M:%S')          # '18:49:05'
"It's now {:%I:%M %p}".format(now)  # "It's now 06:49 PM"
```


### 1. 将 format_spec 应用到每个分量

```python
def __format__(self, fmt_spec=''):
    components = (format(c, fmt_spec) for c in self)
    return '({}, {})'.format(*components)
```

```python
>>> format(Vector2d(3, 4), '.2f')
'(3.00, 4.00)'
```

### 2. 扩展自定义代码 `'p'`（极坐标）

```python
def angle(self):
    return math.atan2(self.y, self.x)

def __format__(self, fmt_spec=''):
    if fmt_spec.endswith('p'):
        fmt_spec = fmt_spec[:-1]
        coords = (abs(self), self.angle())
        outer_fmt = '<{}, {}>'
    else:
        coords = self
        outer_fmt = '({}, {})'
    components = (format(c, fmt_spec) for c in coords)
    return outer_fmt.format(*components)
```

```python
>>> format(Vector2d(1, 1), '.3ep')
'<1.414e+00, 7.854e-01>'
```

> 选择自定义字母时应避免与已有类型冲突
> int 用`bcdoxXn`，float用 `eEfFgGn%`，str用`s`
> 虽然重用字母不会报错（每个类独立解释格式码），但会让用户困惑，应尽量避免。

---

## Hashable Vector2d
> 哈希是把任意对象转换成一个固定大小的整数的过程，这个整数叫做哈希值。

要让对象可散列（可放入 `set`、可作 `dict` 键），需要：
1. 实现 `__hash__`
2. 实现 `__eq__`（已有）
3. 保证对象不可变, 遵守约定：相等的对象必须有相同的哈希值

### `@property`实现只读
1. 双下划线强制属性变为私有，`v.__x # AttributeError: 找不到 '__x'`
   - `v._Vector2d__x # 3.0`: 实际存储名称，可以访问但不应该这么做
   - Python 把 `__x` 改写成 `_Vector2d__x`，目的是防止子类意外覆盖父类的私有属性，而不是真正的"加密"。
2.  `@property` 暴露只读接口

```python
class Vector2d:
    typecode = 'd'

    def __init__(self, x, y):
        self.__x = float(x)  # 双下划线前缀 → 名称改写（name mangling）
        self.__y = float(y)

    @property          # 把方法变成属性
    def x(self):       # 方法名就是属性名
        return self.__x  # 返回私有属性的值

    @property
    def y(self):
        return self.__y
```

### 实现 `__hash__`

```python
def __hash__(self):
    return hash((self.x, self.y))  # 把 x, y 打包成元组再哈希
```

> 严格来说，只需正确实现 `__hash__` 和 `__eq__` 就能创建可散列类型。
> 但可哈希对象的值不应改变，所以实现只读属性是好习惯。

---

## Positional Pattern Matching

**关键字模式（开箱即用）**
关键字模式不需要任何额外设置，开箱即用。
```python
match v:
    case Vector2d(x=0, y=0):    # v.x==0 且 v.y==0
        print(f'{v!r} is null')
    case Vector2d(x=x, y=y) if x==y:  # x==y 时，同时捕获值
        print(f'{v!r} is diagonal')
    case _:                      # 其他所有情况
        print(f'{v!r} is awesome')
```
**位置模式（需要配置）**
```python
case Vector2d(_, 0):   # 想表达"y==0，x随意"
```
直接使用会报错：Python 不知道第一个位置对应 x 还是 y，需要显式告知。
只需在类中添加一个类属性
```python
class Vector2d:
    __match_args__ = ('x', 'y')  # 声明位置模式匹配的顺序
```
- `__match_args__` 通常只包含必填参数对应的属性，可选参数留给关键字模式处理。
---

## Private and “Protected” Attributes
**Python 没有真正的私有属性**
### 机制

双下划线前缀的属性（如 `self.__x`）会被 Python 改写为 `_ClassName__x`，存储在 `__dict__` 中：

```python
>>> v1 = Vector2d(3, 4)
>>> v1.__dict__
{'_Vector2d__y': 4.0, '_Vector2d__x': 3.0}
>>> v1._Vector2d__x  # 仍然可以访问
3.0
```
命名规则：双下划线开头，最多一个下划线结尾
1. `__x`      → 触发改写 ✓
2. `__x_`     → 触发改写 ✓
3. `__x__`    → 不触发（魔法方法）✗
4. `_x`       → 不触发（单下划线）✗
### 设计哲学

名称改写是**安全机制**（防止子类意外覆盖），不是**安保机制**（无法阻止故意访问）。

**社区中的两种惯例：**
- `self.__x`：双下划线，触发名称改写，防止子类属性冲突
- `self._x`：单下划线，纯粹靠约定保护，更常见也更受推荐

---

## Saving Memory with `__slots__`

默认情况下，Python 用 `__dict__`（字典）存储实例属性。字典非常灵活，但内存开销大。

定义 `__slots__`，Python 看到 `__slots__` 后，会用一个**固定大小的数组**代替字典来存属性，内存占用显著降低。

```python
class Vector2d:
    __slots__ = ('__x', '__y')  # 声明允许存在的属性
    typecode = 'd'
```
1. 没有 `__dict__`
2. 不能动态添加属性

### 注意事项

1. **子类必须重新声明** `__slots__`，否则子类实例会有 `__dict__`
2. 实例只能拥有 `__slots__` 中列出的属性（除非加入 `'__dict__'`）
3. 加入 `'__dict__'` 到 `__slots__` 可同时支持动态属性，但可能抵消内存优势
4. 需要弱引用支持时，需加入 `'__weakref__'`
5. 使用 `@cached_property` 装饰器时，必须在 `__slots__` 中包含 `'__dict__'`

### 子类继承行为
**不声明`__slots__`**
```python
class OpenPixel(Pixel):       # 没有声明 __slots__
    pass                       # → 实例会有 __dict__，可添加任意属性
op = OpenPixel()
op.__dict__     # {}  ← 子类重新有了 __dict__！
op.x = 8        # x 存入隐藏数组（继承自父类的 __slots__）
op.color = 'green'  # color 存入 __dict__（子类自己的字典）
op.__dict__     # {'color': 'green'}
```
**声明空`__slots__`**
```python
class StrictPixel(Pixel):
    __slots__ = ()   # 空元组，不添加新属性，但继承父类的限制

sp = StrictPixel()
sp.__dict__   # AttributeError ← 没有 __dict__
sp.x = 1      # ✅ 继承父类的 x
sp.color = 'red'  # ❌ 不允许
```

**声明新 `__slots__`**
父子类的 `__slots__` **自动合并**，子类实例可以使用所有层级中声明的属性。
```python
class ColorPixel(Pixel):
    __slots__ = ('color',)     # 在父类基础上增加 color
                               # → 实例无 __dict__，只能设置 x, y, color
cp = ColorPixel()
cp.__dict__    # AttributeError ← 没有 __dict__
cp.x = 2       # ✅ 继承自 Pixel
cp.color = 'blue'  # ✅ 自己声明的
cp.flavor = 'banana'  # ❌ 不在任何 __slots__ 中
```

**把 `__dict__` 加入 `__slots__`**
```python
class FlexPixel:
    __slots__ = ('x', 'y', '__dict__')
fp = FlexPixel()
fp.x = 1           # 存在紧凑数组中
fp.color = 'red'   # 存在 __dict__ 中，允许动态属性
```
这样可以让固定属性走高效数组，同时保留动态添加属性的能力。

**`__weakref__`**
普通类默认支持弱引用，但使用 `__slots__` 后需要自己声明。
```python
class Pixel:
    __slots__ = ('x', 'y', '__weakref__')   # 手动添加
import weakref
p = Pixel()
weakref.ref(p)
```
---

## Overriding Class Attributes
Python 读取属性时遵循一个固定顺序：实例属性  →  类属性  →  父类属性
- 实例属性可以"遮蔽"同名的类属性，而不会破坏类属性本身。
- 类属性可以作为实例属性的默认值。

**修改单个实例**
```python
>>> v1 = Vector2d(1.1, 2.2)
>>> v1.typecode = 'f'     # 创建实例属性，遮蔽类属性
>>> Vector2d.typecode      # 类属性不受影响
'd'
```

**子类覆盖（最 Pythonic）**

```python
class ShortVector2d(Vector2d):
    typecode = 'f'  # 只覆盖这一个类属性
```

这也是为什么 `__repr__` 中使用 `type(self).__name__` 而非硬编码类名——子类自动获得正确的类名显示。

---

## 完整 Vector2d v3 清单

```python
from array import array
import math

class Vector2d:
    __match_args__ = ('x', 'y')
    typecode = 'd'

    def __init__(self, x, y):
        self.__x = float(x)
        self.__y = float(y)

    @property
    def x(self):
        return self.__x

    @property
    def y(self):
        return self.__y

    def __iter__(self):
        return (i for i in (self.x, self.y))

    def __repr__(self):
        class_name = type(self).__name__
        return '{}({!r}, {!r})'.format(class_name, *self)

    def __str__(self):
        return str(tuple(self))

    def __bytes__(self):
        return (bytes([ord(self.typecode)]) +
                bytes(array(self.typecode, self)))

    def __eq__(self, other):
        return tuple(self) == tuple(other)

    def __hash__(self):
        return hash((self.x, self.y))

    def __abs__(self):
        return math.hypot(self.x, self.y)

    def __bool__(self):
        return bool(abs(self))

    def angle(self):
        return math.atan2(self.y, self.x)

    def __format__(self, fmt_spec=''):
        if fmt_spec.endswith('p'):
            fmt_spec = fmt_spec[:-1]
            coords = (abs(self), self.angle())
            outer_fmt = '<{}, {}>'
        else:
            coords = self
            outer_fmt = '({}, {})'
        components = (format(c, fmt_spec) for c in coords)
        return outer_fmt.format(*components)

    @classmethod
    def frombytes(cls, octets):
        typecode = chr(octets[0])
        memv = memoryview(octets[1:]).cast(typecode)
        return cls(*memv)
```

---

## 总结：本章涉及的特殊方法

| 类别 | 方法 |
|------|------|
| 字符串/字节表示 | `__repr__`, `__str__`, `__format__`, `__bytes__` |
| 数值转换 | `__abs__`, `__bool__`, `__hash__` |
| 比较 | `__eq__` |
| 迭代 | `__iter__` |
| 构造 | `__init__`, `frombytes`（@classmethod） |

**核心原则：** 应用层代码追求简单够用，库/框架代码则应尽量符合 Python 用户的预期行为。

> "To build Pythonic objects, observe how real Python objects behave."
