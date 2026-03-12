# Fluent Python 第2章：序列构成的数组

## 序列类型概览

- 按存储方式分类

| 类型         | 描述                               | 示例                                 |
| ------------ | ---------------------------------- | ------------------------------------ |
| **容器序列** | 存储对象引用，可嵌套，支持不同类型 | `list`, `tuple`, `collections.deque` |
| **扁平序列** | 直接存储值（C 原始类型），更紧凑   | `str`, `bytes`, `array.array`        |

> 扁平序列更紧凑高效，但只能存储数值、字节、字符等基础类型。

- 按可变性分类

-- **可变序列**：`list`, `bytearray`, `array.array`, `collections.deque`
-- **不可变序列**：`tuple`, `str`, `bytes`

---

## 列表推导式与生成器表达式
In Python, you can break lines inside parentheses (), brackets [], or braces {} without needing the \ character. Additionally, trailing commas (a comma after the last item) are ignored, which is a best practice for clean version control diffs.
### 列表推导式 (list comprehensions)

```python
# 普通写法
codes = []
for symbol in '$¢£¥€¤':
    codes.append(ord(symbol))

# 列表推导式（更清晰，意图明确）
codes = [ord(symbol) for symbol in '$¢£¥€¤']
```

- 列表推导式只做一件事：**构建新列表**
- 超过两行就应改写为普通 for 循环
- 不要用列表推导式产生副作用（不使用结果时别用它）

### 局部作用域

Python 3 中，列表推导式、生成器表达式、set/dict 推导式内部有**独立的局部作用域**：

```python
x = 'ABC'
codes = [ord(x) for x in x]
x  # 'ABC'，未被覆盖

# 海象运算符 := 赋值的变量会泄漏到外部
codes = [last := ord(c) for c in x]
last  # 67，仍然存在
c     # NameError，已销毁
```

### listcomp vs map/filter

```python
beyond_ascii = [ord(s) for s in symbols if ord(s) > 127]
# 等价于（但可读性更差）
beyond_ascii = list(filter(lambda c: c > 127, map(ord, symbols)))
```

性能上两者相近，推荐使用列表推导式。

### 笛卡尔积

```python
colors = ['black', 'white']
sizes = ['S', 'M', 'L']

# for 子句顺序决定结果排列顺序
tshirts = [(color, size) for color in colors for size in sizes]
# [('black','S'), ('black','M'), ('black','L'), ('white','S'), ...]
```

### 生成器表达式（generator expression）

- 语法与列表推导式相同，但用 `()`.
- **逐个产出元素**，节省内存，不构建完整列表
- 生成器表达式在被创建时，它并没有立即保存列表里的所有数据，而是保存了对这些对象的引用（Reference）。当你执行 for 循环并调用 next() 时，生成器才去读取列表当前的状态。

```python
tuple(ord(s) for s in '$¢£¥€¤')
# 当生成器表达式是唯一参数时，允许省略括号
array.array('I', (ord(s) for s in '$¢£¥€¤'))

# 笛卡尔积（不产生中间列表）
for tshirt in (f'{c} {s}' for c in colors for s in sizes):
    print(tshirt)
```

---

## 元组
1. **作为记录**：位置即含义，顺序重要
2. **作为不可变列表**：确保长度不变、节省内存

### 记录

```python
lax_coordinates = (33.9425, -118.408056)
city, year, pop, chg, area = ('Tokyo', 2003, 32_450, 0.66, 8014)

# 迭代时解包
for country, _ in traveler_ids:  # _ 丢弃不需要的字段
    print(country)
```

### 不可变列表
- **清晰性**：看到 tuple 即知长度不变
- **性能**：占用内存比同长度 list 少 (list需要预留空间来兼容append)；`tuple(t)` 直接返回引用，`list(l)` 必须复制

**陷阱**：元组的不可变性只针对引用本身，若引用指向可变对象，值仍可变 (tuple包含的list是可变的)。检测元组是否"真正不可变"：
```python
def fixed(o):
    try:
        hash(o)
        return True
    except TypeError:
        return False
```
- an object is only hashable if its value cannot ever change.
- An unhashable tuple cannot be inserted as a dict key, or a set element.
- tuple 支持所有不涉及“修改内容”的 list 方法，如 `count()`、`index()`、`__getitem__`
- 虽然没有 `__reversed__()`，但可以使用内置的 `reversed()` 来反向遍历元组。
- `__getnewargs__()` 是 tuple 特有的，它主要用于配合 pickle 模块进行序列化（将对象转换为二进制流以便存储或传输）时的性能优化。

---

## 序列解包

- 只要对象可以被遍历，就可以被解包。

```python
# Parallel Assignment
latitude, longitude = (33.9425, -118.408056)
b, a = a, b
# 函数调用时解包
divmod(*t)
# 忽略部分值
_, filename = os.path.split('/home/user/.ssh/id_rsa.pub')
```

- `*` 捕获多余元素

```python
a, b, *rest = range(5)     # rest = [2, 3, 4]
a, *body, c, d = range(5)  # body = [1, 2]
*head, b, c, d = range(5)  # head = [0, 1]
```

- 函数调用与字面量中的 `*`（PEP 448）

```python
fun(*[1, 2], 3, *range(4, 7))  # 多次使用 *
[*range(4), 4]                  # 列表字面量
{*range(4), 4, *(5, 6, 7)}     # 集合字面量
```

- 嵌套解包

```python
for name, _, _, (lat, lon) in metro_areas:
    if lon <= 0:
        print(f'{name:15} | {lat:9.4f} | {lon:9.4f}')
```
- 列表解包
-- 通过在赋值语句左侧添加 []，可以利用 Unpacking 同时提取与数量校验：
-- 强制校验：使用 [record] = ... 赋值时，Python 会严格校验右侧对象是否恰好包含一个元素。如果不匹配（0 个或 >1 个），将直接抛出 ValueError，防止程序在数据异常时静默运行。
- 嵌套解包：对于深层嵌套结构，可使用 `[[field]] = ...` 直接提取内部值。
- 避坑指南：虽然元组 `(record,)` 也可实现此功能，但必须注意末尾逗号。相比之下，使用列表语法 `[record]` 更清晰，且不易因漏写逗号产生逻辑错误。
---

## 模式匹配（Python 3.10+）

```python
match message:
    case ['BEEPER', frequency, times]:
        self.beep(times, frequency)
    case ['NECK', angle]:
        self.rotate_neck(angle)
    case ['LED', ident, red, green, blue]:
        self.leds[ident].set_color(ident, red, green, blue)
    case _:                         # 默认匹配，务必加上
        raise InvalidCommand(message)
```

匹配成功条件：
1. subject (这里是message) 是一个序列
2. 主题与模式项数相同
3. 每对对应项均匹配（含嵌套）

```python
metro_areas = [
    ('Tokyo', 'JP', 36.933, (35.689722, 139.691667)),
    ('Delhi NCR', 'IN', 21.935, (28.613889, 77.208889)),
    ('Mexico City', 'MX', 20.142, (19.433333, -99.133333)),
    ('New York-Newark', 'US', 20.104, (40.808611, -74.020386)),
    ('São Paulo', 'BR', 19.649, (-23.547778, -46.635833)),
]
def main():
    print(f'{"":15} | {"latitude":>9} | {"longitude":>9}')
    for record in metro_areas:
        match record:
            case [name, _, _, (lat, lon)] if lon <= 0:
                print(f'{name:15} | {lat:9.4f} | {lon:9.4f}')
```
- `()`和`[]`完全等价
- str、bytes 和 bytearray 在模式匹配中不被视为序列
  - 如果你确实想按字符匹配，必须手动转换，如 `match tuple(phone):`
- 特殊符号与关键字
    - `_` (通配符)：匹配任何单个项，但不绑定变量。它是唯一可以在一个模式中出现多次的变量名。
    - `*rest` 或 `*_`：匹配任意数量（包括 0 个）的剩余项。`*_` 只匹配不取值，*extra 会把匹配到的部分存入一个列表。
    - as 关键字：同时提取局部与整体。例如：case [name, ..., (lat, lon) as coord]。你既能得到拆解后的 lat 和 lon，也能得到完整的元组 coord。
- 类型检查：`case [str(name), float(lat), float(lon)]`。这里的 `str(name)` 是在进行 运行时类型检查。
  - 只有当第 0 个元素本来就是 str 类型时，才匹配成功并赋值给 name。如果它是整数，这个 case 就会失败。
- Guard: if 子句是最后的关卡。只有模式匹配成功，且 if 后面的条件为真（Truthy），这个 case 块才会执行。
```python
# 使用 as 绑定子模式
case [name, _, _, (lat, lon) as coord]:
# 类型检查
case [str(name), _, _, (float(lat), float(lon))]:
# * 匹配任意数量
case [str(name), *_, (float(lat), float(lon))]:
# guard 条件
case [name, _, _, (lat, lon)] if lon <= 0:
```

### 与 if/elif 对比（Norvig lis.py 示例）

```python
# 旧写法
elif exp[0] == 'if':
    (_, test, consequence, alternative) = exp
    ...

# match/case 写法（更简洁安全）
case ['if', test, consequence, alternative]:
    ...
case ['lambda', [*parms], *body] if body:
    return Procedure(parms, body, env)
case ['define', Symbol() as name, value_exp]:
    env[name] = evaluate(value_exp, env)
```

---

## Slicing

#### 为什么不含末尾元素

- `range(3)` 和 `my_list[:3]` 都产生 3 个元素，直观
- 长度 = `stop - start`
- 无重叠分割：`my_list[:x]` 和 `my_list[x:]`

### 切片对象与步长
- step > 0：从左往右走，默认 start=0
- step < 0：从右往左走，默认 start=-1
```python
s = 'bicycle'
s[::3]   # 'bye'
s[::-1]  # 'elcycib'（反转）
s[::-2]  # 'eccb'
```
- slice object: `slice(a, b, c)`
### 命名切片（提升可读性）
```python
SKU        = slice(0, 6)
DESCRIPTION = slice(6, 40)
UNIT_PRICE  = slice(40, 52)
for item in line_items:
    print(item[UNIT_PRICE], item[DESCRIPTION])
```
### 多维切片与省略号
```python
    pythona[i, j]     # 实际调用 a.__getitem__((i, j))
    a[m:n, k:l] # 二维切片，NumPy 用这个
```
- 但这个特性标准库的内置序列类型不支持（除了 memoryview），主要是为 NumPy 这类第三方库设计的扩展点。
- 省略号 ... 是一个真实的对象（Ellipsis，ellipsis 类的唯一实例），在 NumPy 里用作多维切片的简写：`x[i, ...]  # 等价于 x[i, :, :, :]（四维数组）`

### 切片赋值（就地修改可变序列）
```python
l = list(range(10))
l[2:5] = [20, 30]    # 替换
del l[5:7]           # 删除
l[3::2] = [11, 22]   # 步长赋值

# 注意：右侧必须是可迭代对象
l[2:5] = [100]       # OK
l[2:5] = 100         # TypeError
```
---
## `+` 和 `*` 操作
- `+` 和 `*` 总是**创建新对象**，不修改原序列
- `seq * n`：重复拼接

### 陷阱：列表嵌套

```python
# 错误！三个引用指向同一内层列表
weird_board = [['_'] * 3] * 3
weird_board[1][2] = 'O'  # 三行都被修改！

# 正确写法（列表推导式每次创建新列表）
board = [['_'] * 3 for i in range(3)]
board[1][2] = 'X'  # 只改第 1 行
```

### 增量赋值 `+=` 和 `*=`

- `+=` 会先找 `__iadd__`（就地加法），找不到才退而求其次用 `__add__`（普通加法）
- 可变序列：调用 `__iadd__` / `__imul__`，**就地修改**
- 不可变序列：等价于 `a = a + b`，**创建新对象**

```python
l = [1, 2, 3]; id(l)  # 原 id
l *= 2;         id(l)  # 相同 id（就地修改）

t = (1, 2, 3);  id(t)  # 原 id
t *= 2;         id(t)  # 新 id（创建新对象）
```
- 对不可变序列反复拼接（比如循环里做 +=）效率很低，因为每次都要把整个序列复制一遍再创建新对象。可变序列就没这个问题，直接在原地追加。
```python
t = (1, 2, [30, 40])
t[2] += [50, 60]
# 结果：t 变成 (1, 2, [30, 40, 50, 60])，同时抛出 TypeError！
```
- 不要把可变对象放入元组
- 增量赋值不是原子操作

---

## `list.sort` vs `sorted`

|                | `list.sort()`    | `sorted()`     |
| -------------- | ---------------- | -------------- |
| 作用对象       | list（就地排序） | 任意可迭代对象 |
| 返回值         | `None`           | 新列表         |
| 是否修改原对象 | 是               | 否             |

- `random.shuffle()` 也返回 `None`
### 共同参数
- `reverse=True`：降序
- `key=func`：单参数函数，用于提取比较键

```python
sorted(fruits, key=len)              # 按长度升序
sorted(fruits, key=len, reverse=True) # 按长度降序
sorted(fruits, key=str.lower)        # 大小写不敏感排序
```

Python 的排序算法（Timsort）是**稳定排序**。

---

## 何时不用 list
- 大量数值数据 → 用 array.array，更省内存
- 频繁在两端增删 → 用 deque
- 频繁做 item in collection 检查 → 用 set

### `array.array`

适合：**大量数值数据**（比 list 紧凑，I/O 快）
- list 存的是对象引用，每个数字都是一个完整的 Python float 对象，有额外的元数据开销。array 直接存原始的 C 类型值，没有这些开销，和 C 数组一样紧凑。

```python
from array import array
floats = array('d', (random() for i in range(10**7)))
floats.tofile(fp)      # 二进制写入，~7× 快于文本
floats2.fromfile(fp, 10**7)  # 读取，~60× 快于文本解析
```
- array 没有就地排序方法，需要这样排：`a = array.array(a.typecode, sorted(a))`
- To keep a sorted array sorted while adding items to it, use the
bisect.insort function.

常用 typecode：`'b'`(int8), `'h'`(int16), `'i'`(int32), `'f'`(float32), `'d'`(float64)

### `memoryview`
- 共享内存的类型，切片**不复制字节**
- 对同一块内存数据创建不同的"视角"，不复制数据。多个 memoryview 对象指向同一块内存，修改任何一个，底层数据都会变。
- 可通过 `cast()` 改变数据解读方式（如把 1D 字节数组当 2D 矩阵操作）

```python
octets = array('B', range(6))
m1 = memoryview(octets)
m2 = m1.cast('B', [2, 3])  # 2×3 视图
m3 = m1.cast('B', [3, 2])  # 3×2 视图
# 修改 m2/m3 会直接影响 octets（共享内存）
m2[1,1] = 22  # 修改 m2
m3[1,1] = 33  # 修改 m3
octets        # [0,1,2,33,22,5]，底层数据被改了
```

### Numpy
``` python
a = np.arange(12)      # 一维数组，0~11
a.shape = 3, 4         # 改成 3×4 二维数组，不复制数据
a[2]                   # 取第2行：[8, 9, 10, 11]
a[:, 1]                # 取所有行的第1列：[1, 5, 9]
a.transpose()          # 行列互换
```
```python
# 向量化运算
floats *= .5    # 所有元素同时乘以 0.5
floats /= 3     # 所有元素同时除以 3，1000万个数不到40毫秒
```
```python
# 普通加载：全部读入内存
floats = numpy.load('floats-10M.npy')
# 文件不完全载入内存，只在访问时读取对应部分，适合处理超大数据集。
floats2 = numpy.load('floats-10M.npy', 'r+')  # 内存映射模式加载
# 只要传入任意模式参数，就会开启内存映射
```

### Deques and other queues
- 为什么不用 list 做队列？
  - list 用 append 和 pop(0) 可以模拟队列，但从头部插入/删除需要把整个列表在内存中移位，数据量大时很慢。

#### `collections.deque`

适合：**两端频繁插入/删除**（双端队列），头尾增删都是 O(1)

```python
from collections import deque
dq = deque(range(10), maxlen=10)  # 有界队列，满后自动丢弃另一端
dq.rotate(3)       # 右旋：从右取3个放到左边 → [7,8,9,0,1,2,3,4,5,6]
dq.appendleft(-1)  # 左端追加，满了自动丢弃右端的0
dq.extend([11,22,33])      # 右端追加，满了丢弃左端
dq.extendleft([10,20,30])  # 逐个追加到左端，顺序会反转
```
`append` 和 `popleft` 是原子操作，多线程安全。

#### 其他队列

| 模块              | 特点                            | 场景             |
| ----------------- | ------------------------------- | ---------------- |
| `queue`           | 线程安全，支持阻塞              | 多线程通信       |
| `multiprocessing` | 进程间通信                      | 多进程通信       |
| `asyncio`         | 异步编程                        | 异步编程         |
| `heapq`           | 堆队列/优先队列（函数式，无类） | 需要按优先级出队 |

---

## 关键总结

1. **优先使用列表推导式**，而不是 `map`/`filter`
2. **生成器表达式**用于非 list 序列的初始化，节省内存
3. **元组**有双重用途：记录 & 不可变列表；含可变项时需谨慎
4. **解包**和 `*` 语法让代码更简洁，避免索引操作
5. **match/case** 是比 `if/elif` 更强大的结构化解包工具
6. **切片命名**可大幅提升代码可读性
7. `[x] * n` 嵌套列表是常见陷阱，用列表推导式替代
8. 数值密集型数据用 `array.array`；两端操作频繁用 `deque`
