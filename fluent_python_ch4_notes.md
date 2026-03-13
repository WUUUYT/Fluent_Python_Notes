# Ch 4：Unicode 文本与字节序列

## 字符、码位与字节

**字符的本质**：Python 3 中的字符 = Unicode 字符，不是原始字节。
> Unicode 是一个标准/系统，不是"一个东西"。它做了两件事：
> 1. 给世界上所有字符分配一个唯一的编号 → 这个编号就叫 code point（码位）
> 2. 定义这些字符的各种属性（名称、类别、方向性等）
>
- 视觉字符  →  code point 数量 可能有1或者多个
- code point 数量  →  字节数量 取决于编码
- Python 的 str 存的就是 code point，显示的时候终端/系统负责把它渲染成视觉字符。

| 概念                   | 说明                                                  |
| ---------------------- | ----------------------------------------------------- |
| **码位（Code Point）** | 字符的唯一标识，0 ~ 1,114,111 的整数，格式为 `U+XXXX` |
| **编码（Encoding）**   | 码位 → 字节序列的算法（`str.encode()`）               |
| **解码（Decoding）**   | 字节序列 → 码位的算法（`bytes.decode()`）             |

```python
s = 'café'   # len(s)=4（Unicode 字符数）
b = s.encode('utf8') # b = b'caf\xc3\xa9'
len(b)          # 5（字节数，é 用 2 个字节）
b.decode('utf8')  # 'café'
```
---

## 二进制序列类型

Python 3 有两种内置二进制序列类型：

| 类型        | 可变性 | 说明            |
| ----------- | ------ | --------------- |
| `bytes`     | 不可变 | Python 3 引入   |
| `bytearray` | 可变   | Python 2.6 引入 |

- bytes 的元素是`0~255` 的整数，不是字符
```python
cafe = bytes('café', encoding='utf_8')
cafe[0]    # 99（整数，不是 'c'）
cafe[:1]   # b'c'（bytes 类型）
```
- **切片**返回同类型的序列（包括长度为 1 的切片）
  - 这才是 Python 序列的通用行为，str 是特例。
- `my_bytes[0]` → `int`；`my_bytes[:1]` → `bytes`

**字面量显示规则：**
- 为了可读性，显示时会把 ASCII 范围内的字节直接显示成字符
  - ASCII 可打印字符（32~126）
- `\t \n \r \\`：使用转义序列
- 其他：十六进制转义 `\xNN`

**构造方法**
```python
bytes('café', encoding='utf-8')        # 从 str
bytes([99, 97, 102, 195, 169])         # 从整数列表
bytes(array.array('h', [-2, -1, 0]))   # 从缓冲区对象（会复制字节）
bytes.fromhex('31 4B CE A9')           # 从十六进制字符串（str 没有这个方法）
# Typecode 'h' creates an array of short integers (16 bits).
```

---

## 常见编解码器

Python 内置 100+ 种编解码器，常见的有：

| 编码                 | 特点                                          |
| -------------------- | --------------------------------------------- |
| `latin1` (iso8859_1) | 其他编码的基础，包括 cp1252 和 Unicode        |
| `cp1252`             | Windows 常用，latin1 超集，含欧元符号等       |
| `cp437`              | IBM PC 原始字符集，含制表符，与 latin1 不兼容 |
| `gb2312`             | 简体中文编码                                  |
| `utf-8`              | Web 最主流编码，2021 年占 97% 网站            |
| `utf-16le`           | 16 位 UTF 编码，支持代理对（surrogate pairs） |

> UTF 编码能处理所有 Unicode 码位，其他编码只覆盖部分字符。

---

## 编解码错误处理

### UnicodeEncodeError（str → bytes）

非 UTF 编解码器遇到无法表示的字符时抛出，可通过 `errors` 参数处理：

```python
city = 'São Paulo'
city.encode('cp437')                          # 报错！
city.encode('cp437', errors='ignore')         # b'So Paulo'（静默丢失数据）
city.encode('cp437', errors='replace')        # b'S?o Paulo'（用?替换）
city.encode('cp437', errors='xmlcharrefreplace')  # b'S&#227;o Paulo'
```

> `errors='ignore'` 通常不好，会导致静默数据丢失。

### UnicodeDecodeError（bytes → str）

- UTF-8/16 遇到非法字节序列时抛出。
- Legacy 8 位编码（cp1252、latin1 等）则**不会报错，会静默产生乱码（mojibake，文字化け）**：

```python
octets = b'Montr\xe9al'
octets.decode('cp1252')          # 'Montréal'（正确）
octets.decode('iso8859_7')       # 'Montrιal'（希腊字母，乱码）
octets.decode('utf_8')           # UnicodeDecodeError
octets.decode('utf_8', errors='replace')  # 'Montr\ufffdal'（替换字符）
```

### SyntaxError（加载模块时）

Python 3 默认源码编码为 UTF-8，若文件含非 UTF-8 字节且无声明，则报 `SyntaxError`。修复方法：在文件顶部添加 coding 注释：

```python
# coding: cp1252
```

> 更好的做法：直接将文件转换为 UTF-8。

### 检测字节序列的编码

**无法确定性地检测编码**，只能依靠协议头（HTTP、XML）或启发式推断。
可用 `chardet` 库进行猜测：

```bash
$ chardetect file.txt   # 输出编码及置信度
```

### BOM（字节顺序标记）
核心问题：UTF-16 用 2 个字节表示一个码位，但 2 个字节谁在前谁在后有歧义。
- UTF-16 编码时自动在开头添加 BOM（`b'\xff\xfe'` = 小端）
- UTF-8 一般不需要 BOM，但 Windows Notepad 会添加（UTF-8-SIG）
- 建议读取 UTF-8 文件时用 `utf-8-sig` 编解码器，兼容有无 BOM 的情况
```python
# 读文件时用 utf-8-sig，有没有 BOM 都能正确处理
open('file.txt', encoding='utf-8-sig').read()
```
---

## 处理文本文件

- Unicode 三明治原则
```
输入层：bytes → str（尽早解码）
处理层：100% str（业务逻辑只用 str）
输出层：str → bytes（尽晚编码）
```

### 关键原则

**永远显式指定编码，不要依赖默认值！**

```python
# 错误示例（依赖默认编码，Windows 上会出问题）
open('cafe.txt', 'w').write('café')
open('cafe.txt').read()  # Windows 上可能返回 'cafÃ©'

# 正确示例
open('cafe.txt', 'w', encoding='utf-8').write('café')
# 在磁盘上打开 cafe.txt，如果不存在就创建，'w' 表示写模式（会清空已有内容）。
# 之后所有写入操作都会自动用 UTF-8 把 str 转成字节再写进磁盘
open('cafe.txt', encoding='utf-8').read()  # 'café'
```
- 各平台默认编码情况

| 平台              | 默认编码                                              |
| ----------------- | ----------------------------------------------------- |
| GNU/Linux / macOS | 全部 UTF-8                                            |
| Windows           | 文件 I/O 用 cp1252，控制台输出用 utf-8（Python 3.6+） |

- `locale.getpreferredencoding()` → 打开文本文件时的默认编码（**最重要**）。不写 encoding= 时，open() 就用这个值
- `sys.getdefaultencoding()` → Python 内部隐式转换用的编码
- `sys.getfilesystemencoding()` → 文件名编解码

---

## Unicode 规范化

- 为什么需要规范化：同一字符可用不同码位序列表示：
```python
s1 = 'café'                        # 4 个码位（é = U+00E9）
s2 = 'cafe\N{COMBINING ACUTE ACCENT}'  # 5 个码位（e + ́ 组合）
s1 == s2   # False（虽然显示一样！）
```

### 四种规范化形式

| 形式     | 说明                              | 使用场景                     |
| -------- | --------------------------------- | ---------------------------- |
| **NFC**  | 组合字符，产生最短等价字符串      | 通用存储、比较（推荐）       |
| **NFD**  | 分解字符，展开为基字符 + 组合字符 | 需要分析字符组成时           |
| **NFKC** | 兼容分解后组合，替换兼容字符      | 搜索、索引（会丢失格式信息） |
| **NFKD** | 兼容分解                          | 搜索、索引（会丢失格式信息） |

```python
from unicodedata import normalize
normalize('NFC', s1) == normalize('NFC', s2)   # True
normalize('NFD', s1) == normalize('NFD', s2)   # True
```
- NFKC/NFKD 会导致信息丢失（如 `½` → `1⁄2`，`4²` → `42`）
  - 仅用于搜索索引，不要用于永久存储。

### Case Folding

`str.casefold()` 用于大小写不敏感的比较（比 `lower()` 更彻底），大多数情况下两者一样
  - `lower()` 只是单纯转小写，`casefold()` 还处理了跨语言的等价关系，有近 300 个码位的处理结果不同。


```python
'µ'.casefold()   # 'μ'（MICRO SIGN → GREEK SMALL LETTER MU）
'ß'.casefold()   # 'ss'
```

### 实用工具函数

```python
from unicodedata import normalize

def nfc_equal(str1, str2):
    """大小写敏感的 Unicode 比较"""
    return normalize('NFC', str1) == normalize('NFC', str2)

def fold_equal(str1, str2):
    """大小写不敏感的 Unicode 比较"""
    return normalize('NFC', str1).casefold() == normalize('NFC', str2).casefold()
```

### 去除变音符号（Diacritics）

```python
import unicodedata

def shave_marks(txt):
    """去除所有变音符号"""
    norm = unicodedata.normalize('NFD', txt)         # 分解
    shaved = ''.join(c for c in norm if not unicodedata.combining(c))
    # 过滤组合字符
    return unicodedata.normalize('NFC', shaved)      # 重新组合
```

---

## Unicode 文本排序

Python 默认按码位排序，对非 ASCII 字符排序结果不正确：

```python
fruits = ['caju', 'atemoia', 'cajá', 'açaí', 'acerola']
sorted(fruits)  # ['acerola', 'atemoia', 'açaí', 'caju', 'cajá']（错误）
# 期望：['açaí', 'acerola', 'atemoia', 'cajá', 'caju']
```

### 方案一：locale.strxfrm（有局限性）

```python
import locale
locale.setlocale(locale.LC_COLLATE, 'pt_BR.UTF-8')
sorted(fruits, key=locale.strxfrm)  # 正确
```

缺点：依赖 OS locale 配置，macOS 上可能无效，不推荐在库中使用；locale 是全局设置，在库里用会影响整个进程。

### 方案二：pyuca（推荐）

```python
import pyuca
coll = pyuca.Collator()
sorted(fruits, key=coll.sort_key)  # 跨平台正确
```
- 跨平台，不依赖 OS locale，用的是 Unicode 官方的排序规则表。
- 但也处理不了语言级别精确排序

---

## Unicode 数据库

`unicodedata` 模块提供字符元数据：

```python
import unicodedata

unicodedata.name('A')           # 'LATIN CAPITAL LETTER A'
unicodedata.category('A')       # 'Lu'（大写字母）
unicodedata.numeric('½')        # 0.5
unicodedata.combining('\u0301') # 230（非零 = 组合字符）

# str 方法底层依赖 Unicode 数据库
'A'.isalpha()    # True
'½'.isnumeric()  # True
'2'.isdecimal()  # True
```

**cf.py：按名字搜索字符**

核心逻辑：遍历所有码位，看名字里是否包含搜索词：
```python
def find(*query_words, start=START, end=END):
    query = {w.upper() for w in query_words}  # 搜索词转大写集合
    for code in range(start, end):
        char = chr(code)                       # 码位 → 字符
        name = unicodedata.name(char, None)    # 获取字符名，未分配返回 None
        if name and query.issubset(name.split()):  # 搜索词全部出现在名字里
            print(f'U+{code:04X}\t{char}\t{name}')
```
```bash
$ python cf.py cat smiling
U+1F638  😸  GRINNING CAT FACE WITH SMILING EYES
U+1F63A  😺  SMILING CAT FACE WITH OPEN MOUTH
```
`query.issubset(name.split())` 把名字按空格拆成单词列表，用集合的 issubset 判断所有搜索词是否都在其中，避免了嵌套循环。

### 字符的数字含义
**三种判断方式，标准越来越宽松**
| 方法          | 接受范围                 | 典型例子                     |
| ------------- | ------------------------ | ---------------------------- |
| `isdecimal()` | 只认十进制数字系统       | `1` `٣`（阿拉伯）`३`（梵文） |
| `isdigit()`   | 还包括上标等数字形式     | `²` `③`                      |
| `isnumeric()` | 最宽，包括分数、罗马数字 | `½` `Ⅻ`                      |
```python
'1'.isdecimal()   # True
'²'.isdecimal()   # False  ← 上标不算十进制数字
'²'.isdigit()     # True   ← 但算 digit
'Ⅻ'.isdigit()    # False  ← 罗马数字12不算 digit
'Ⅻ'.isnumeric()  # True   ← 但算 numeric
```

**unicodedata.numeric() 能返回实际数值**
- Unicode 数据库里直接存了这些符号对应的数学值，不需要自己解析。
```python
unicodedata.numeric('²')   # 2.0
unicodedata.numeric('½')   # 0.5
unicodedata.numeric('Ⅻ')  # 12.0
```

**正则 `\d` 比 `isdigit()` 更严格**
```python
re.match(r'\d', '²')   # None，不匹配
'²'.isdigit()          # True
```
`re` 模块对 Unicode 的支持不够完善，`\d` 认不了所有 Unicode 数字字符。
如果需要更准确的 Unicode 正则匹配，用 PyPI 上的 `regex` 库替代。


---

## 双模式 API（str 与 bytes）
标准库里有些函数接受 str 或 bytes，行为会不同
### 正则表达式
- str 模式识别 Unicode，bytes 模式只认 ASCII
```python
import re
# str 模式：匹配 Unicode 字符（含泰米尔数字、上标等）
re.compile(r'\d+').findall('1²Tamil:௧')   # ['1', '2', '௧']
# bytes 模式：仅匹配 ASCII
re.compile(rb'\d+').findall(b'1\xb2')      # [b'1']
# str 模式但只匹配 ASCII 数字
re.compile(r'\d+', re.ASCII)
```

### os 模块文件名
- 正常情况下传 str 就可以，os 会自动用 sys.getfilesystemencoding() 编解码
- 但 Linux 内核本身不是 Unicode 的，文件名本质是字节序列，可能存在无法被任何编码合法解码的文件名。这时传 bytes 可以绕过解码，直接拿到原始字节
```python
import os
os.listdir('.')    # 返回 str 列表（自动解码）
os.listdir(b'.')   # 返回 bytes 列表（不解码，处理乱码文件名时用）
```

---

## 常见坑与解决方案

| 问题                             | 原因                    | 解决方案                                       |
| -------------------------------- | ----------------------- | ---------------------------------------------- |
| `s1 == s2` 为 False 但看起来一样 | 不同 Unicode 规范化形式 | `normalize('NFC', s1) == normalize('NFC', s2)` |
| 读文件出现乱码                   | 编码不一致              | 始终显式指定 `encoding=`                       |
| Windows 脚本跨平台乱码           | 依赖 locale 默认编码    | 始终显式指定 `encoding='utf-8'`                |
| `UnicodeEncodeError`             | 编码不支持该字符        | 换用 UTF-8，或用 `errors=` 参数                |
| `UnicodeDecodeError`             | 字节不符合指定编码      | 确认文件实际编码，或用 `chardet` 猜测          |
| 非 ASCII 文本排序错误            | Python 按码位排序       | 使用 `pyuca` 库                                |


1. Python 3 严格区分 `str`（文本）与 `bytes`（二进制），禁止隐式转换
2. **编码** = str → bytes；**解码** = bytes → str
3. 处理文本遵循 **Unicode 三明治**：输入立即解码，输出最晚编码，中间只用 str
4. **永远显式指定编码**，不依赖平台默认值
5. 比较 Unicode 文本前先 **NFC 规范化**，大小写不敏感比较用 **`casefold()`**
6. 跨平台 Unicode 排序用 **`pyuca`**
