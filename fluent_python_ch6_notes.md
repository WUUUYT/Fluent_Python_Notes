# Ch 6：Object References, Mutability, and Recycling

## Variables Are Not Boxes

Python 变量不是存储数据的"盒子"，而是贴在对象上的**标签（label）**。

```python
a = [1, 2, 3]
b = a          # b 贴到 a 所引用的同一个对象上
a.append(4)
b              # [1, 2, 3, 4]  —— b 也看到了变化
```

**核心原则：** 赋值语句先执行右边（创建或获取对象），再把左边的变量名绑定到该对象上。

```python
class Gizmo:
    def __init__(self):
        print(f'Gizmo id: {id(self)}')

x = Gizmo()           # 成功，x 绑定到新对象
y = Gizmo() * 10      # Gizmo 已创建，但乘法异常 → y 从未被绑定
```

---

## Identity, Equality, and Aliases

### 别名（Aliasing）

多个变量绑定到同一个对象：

```python
charles = {'name': 'Charles L. Dodgson', 'born': 1832}
lewis = charles        # lewis 是 charles 的别名
lewis is charles       # True — 同一对象
lewis['balance'] = 950
charles                # {'name': ..., 'born': 1832, 'balance': 950}
```

### 相等 vs 标识

```python
alex = {'name': 'Charles L. Dodgson', 'born': 1832, 'balance': 950}
alex == charles        # True  — 值相等（__eq__）
alex is not charles    # True  — 不同对象（不同 id）
```

| 操作符 | 比较内容 | 说明 |
|-------|---------|------|
| `==` | **值**（调用 `__eq__`） | 日常编程中更常用 |
| `is` | **标识**（比较 `id()`） | 更快，不可重载 |

**`is` 的正确用法**
```python
x is None          # ✅ 推荐：检查 None
x is not None      # ✅ 推荐：否定写法

# 哨兵对象
END_OF_DATA = object()
if node is END_OF_DATA:
    return
```

**经验法则：** 除了和 `None` / 哨兵对象比较，其他情况用 `==`。

---

## Relative Immutability of Tuples

元组本身不可变（不能增删元素引用），但如果元素是可变对象，其**值可以改变**：

```python
t1 = (1, 2, [30, 40])
t2 = (1, 2, [30, 40])
t1 == t2               # True

t1[-1].append(99)      # 修改列表（id 不变）
t1                     # (1, 2, [30, 40, 99])
t1 == t2               # False
```

不可变的是元组中各元素的**标识（引用）**，不是元素的**值**。所以含可变元素的元组不可哈希。

---

## 浅拷贝&深拷贝
拷贝的目的是创建一个值相等但 id 不同的独立对象。
### 默认是浅拷贝

```python
l1 = [3, [66, 55, 44], (7, 8, 9)]
l2 = list(l1)
l2 = l1[:] # 效果相同
l2 is l1             # False — 外层是新列表
l2[1] is l1[1]       # True  — 内部元素共享引用
```

浅拷贝只复制最外层容器，内部对象仍共享：

```python
l1[1].remove(55)     # l2[1] 也受影响（同一个列表）
l2[1] += [33, 22]    # 列表原地修改 → l1[1] 也变了
l2[2] += (10, 11)    # 元组不可变 → 创建新元组，l1[2] 不受影响
```
- `+=` 行为取决于类型，不可变类型创建新对象；可变类型原地修改
### `copy` 模块

```python
import copy

bus1 = Bus(['Alice', 'Bill', 'Claire', 'David'])
bus2 = copy.copy(bus1)       # 浅拷贝：共享 passengers 列表
bus3 = copy.deepcopy(bus1)   # 深拷贝：passengers 是独立副本

bus1.drop('Bill')
bus2.passengers              # ['Alice', 'Claire', 'David'] — Bill 也没了
bus3.passengers              # ['Alice', 'Bill', 'Claire', 'David'] — 不受影响
```
- deepcopy递归地复制所有嵌套对象**，完全独立，互不干扰。
  - `deepcopy` 能处理循环引用，会记住已拷贝的对象避免无限递归。可通过 `__copy__()` / `__deepcopy__()` 自定义拷贝行为。

---

## Function Parameters as References
不是按值/引用传递，Python 的参数传递方式是 **call by sharing**：函数的形参获得实参引用的**副本**，即形参成为实参的别名。

- **能**修改传入的**可变对象**
- **不能**替换实参本身（因为形参只是引用的副本，重新绑定不影响外部）

### ❌ Mutable Types as Parameter Defaults
```python
class HauntedBus:
    def __init__(self, passengers=[]):  # ⚠️ 危险！
        self.passengers = passengers    # 直接别名

    def pick(self, name):
        self.passengers.append(name)

    def drop(self, name):
        self.passengers.remove(name)
```

默认值在函数**定义时**求值（通常是模块加载时），之后作为函数对象的属性永久保存在 `__defaults__` 中。所有未传参的实例共享同一个列表。

```python
bus2 = HauntedBus()
bus2.pick('Carrie')
bus3 = HauntedBus()
bus3.passengers         # ['Carrie'] ← 幽灵乘客！
bus2.passengers is bus3.passengers  # True
```

### 正确做法

```python
def __init__(self, passengers=None):
    if passengers is None:
        self.passengers = []
    else:
        self.passengers = list(passengers)  # 拷贝！
```

---

## Defensive Programming：保护可变参数

```python
class TwilightBus:
    def __init__(self, passengers=None):
        if passengers is None:
            self.passengers = []
        else:
            self.passengers = passengers  # ⚠️ 直接别名！

basketball_team = ['Sue', 'Tina', 'Maya', 'Diana', 'Pat']
bus = TwilightBus(basketball_team)
bus.drop('Tina')
basketball_team         # ['Sue', 'Maya', 'Diana', 'Pat'] ← Tina 从球队消失了！
```
**修复：** `self.passengers = list(passengers)` — 创建副本。
**原则：** 应拷贝参数再使用（除非方法明确要修改传入对象）。
1. 默认参数可能是可变对象 -> 用`None`，在函数内部创建新对象
2. 接收列表/集合并在内部使用 -> 用 `list()` 或 `copy()` 复制一份
3. 如果明确是"共享并修改" -> 直接别名化，但必须在文档中说明

---

## `del` 和垃圾回收

- `del` 删除的是引用，不是对象
- `del` 是语句，不是函数
```python
a = [1, 2]
b = a
del a       # 删除引用 a
b           # [1, 2] — 对象仍在（b 还引用着）
b = [3]     # 最后一个引用消失 → [1, 2] 可被回收
```

### CPython 垃圾回收机制

- **引用计数** — 引用数归零时立即销毁
- **辅助: 分代垃圾回收**（CPython 2.0+）— 检测引用循环

### `weakref.finalize` 观察对象销毁

```python
import weakref
s1 = {1, 2, 3}
s2 = s1
ender = weakref.finalize(s1, lambda: print('对象被销毁'))
del s1              # ender.alive 仍为 True（s2 还在）
s2 = 'spam'         # 输出"对象被销毁"，ender.alive → False
```

`finalize` 持有的是**弱引用** (refcount 不变)，不会阻止对象被回收。

### `__del__` 注意事项
- `del` 只删引用，`__del__` 在对象真正销毁时才调用
- 不要主动调用 `__del__`
- 它不等于析构函数，只是对象销毁前的步骤：可以关闭文件，断开连接。
- 很少需要自己实现


---

## Python 对不可变对象的优化技巧
> 不可变对象可以安全共享。CPython 利用这一点做了性能优化：直接复用同一个对象。

**元组/字符串的共享优化（假拷贝）**
```python
t1 = (1, 2, 3)
t2 = tuple(t1)
t2 is t1            # True — 同一对象！（元组不可变，无需复制）

s1 = 'ABC'
s2 = 'ABC'
s2 is s1            # True — 字符串驻留（interning）
```

- `tuple(t)` 和 `t[:]` 对元组返回同一对象
- `frozenset.copy()` 也返回同一对象
- 从零创建内容相同的tuple，是不同对象。
- 驻留优化（String Interning）：CPython内部维护一个字符串池，对于**符合条件的字符串字面量**，直接复用已有对象。
- 小整数缓存：CPython 预先创建了 -5 到 256 的整数对象并永久缓存，使用这些数字时直接复用
> **警告：** 比较字符串和整数时始终用 `==`，不用 `is`（不依赖驻留）。

