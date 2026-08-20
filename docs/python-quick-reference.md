# Python 专家指南

> 面向 20 年经验的跨平台资深开发者（C++ / ObjC / Swift / Dart / Flutter）
> 这不是入门教程，而是从专家视角对 Python 的深度解剖

---

## 目录

### 第一部分：思维与语法
1. [思维模型重塑](#1-思维模型重塑)
2. [语法深度解析](#2-语法深度解析)
3. [类型系统全解](#3-类型系统全解)
4. [函数、闭包与装饰器](#4-函数闭包与装饰器)
5. [面向对象深度剖析](#5-面向对象深度剖析)

### 第二部分：内存与性能
6. [内存管理内幕](#6-内存管理内幕)
7. [性能分析与优化](#7-性能分析与优化)
8. [C 扩展与 FFI](#8-c-扩展与-ffi)

### 第三部分：并发与异步
9. [并发模型深度解析](#9-并发模型深度解析)
10. [asyncio 完全指南](#10-asyncio-完全指南)
11. [并发设计模式](#11-并发设计模式)

### 第四部分：元编程
12. [描述符协议](#12-描述符协议)
13. [元类编程](#13-元类编程)
14. [装饰器大全](#14-装饰器大全)
15. [上下文管理器深入](#15-上下文管理器深入)

### 第五部分：标准库精通
16. [数据结构与算法](#16-数据结构与算法)
17. [文件 IO 与序列化](#17-文件-io-与序列化)
18. [正则与文本处理](#18-正则与文本处理)
19. [日期时间与日志](#19-日期时间与日志)

### 第六部分：工程实践
20. [项目架构与包管理](#20-项目架构与包管理)
21. [测试策略与 CI/CD](#21-测试策略与-cicd)
22. [设计模式 Python 版](#22-设计模式-python-版)
23. [Web 开发全栈](#23-web-开发全栈)
24. [数据库与 ORM](#24-数据库与-orm)

### 第七部分：Python 内部机制
25. [对象模型](#25-对象模型)
26. [字节码与解释器](#26-字节码与解释器)
27. [GIL 的过去与未来](#27-gil-的过去与未来)
28. [Import 系统](#28-import-系统)

### 附录
- [A. 快速参考卡片](#a-快速参考卡片)
- [B. 版本迁移指南](#b-版本迁移指南)
- [C. 学习路线](#c-学习路线)

---

## 1. 思维模型重塑

### 1.1 范式对比表

| 维度 | C++ | ObjC/Swift | Dart | Python |
|------|-----|-----------|------|--------|
| 编译 | AOT → 机器码 | AOT/LLVM | AOT+JIT | 字节码 → 解释执行（CPython） |
| 类型 | 静态强类型 | 静态强类型 | 静态强类型（可 null-safety） | 动态强类型 + 可选 Gradual Typing |
| 内存 | 手动/RAII/smart ptr | ARC（ObjC）/ ARC（Swift） | GC（分代） | 引用计数 + 分代 GC（循环检测） |
| 并发 | 真并行（线程） | GCD/async-await | Isolate + async | GIL + 多进程 + asyncio |
| 范式 | 多范式（偏底层） | 面向协议/面向对象 | 面向对象 | 多范式（偏脚本/面向对象） |
| 文件 | `.h` + `.cpp` | `.h` + `.m` / `.swift` | `.dart` | `.py`（一个文件就是模块） |
| 访问控制 | `public`/`private`/`protected` | `open`/`public`/`internal`/`private` | `_` 前缀约定 | `_` 前缀约定（无强制） |
| 构建 | CMake/MSBuild | Xcode/SPM | pub.dev | pip/uv/poetry |

### 1.2 核心心态转变

**你之前的思维**：代码写出来是给编译器看的，编译器帮你把关。

**Python 的思维**：代码写出来是**给人看的**。解释器是你忠实的执行者——它不审判你的类型，不限制你的风格，但会在运行时精准地告诉你哪里错了。

```python
import this  # 在 Python REPL 中运行，阅读"Python 之禅"
```

关键原则：
- **Explicit is better than implicit** —— 别耍小聪明，写清楚的代码
- **Simple is better than complex** —— 有 C++ 模板元编程 PTSD 的请放心
- **There should be one obvious way to do it** —— 别炫技
- **Namespaces are one honking great idea** —— 模块化是王道

### 1.3 Python 不是"慢语言"

你可能会下意识觉得 Python 慢。实际上：

1. **80% 的代码不跑在 CPython 上**：NumPy、Pandas、TensorFlow 底层全是 C/C++/Fortran
2. **Python 是胶水语言**：你写的是"指挥层"逻辑，计算密集型部分用 C 扩展
3. **PyPy JIT**：对纯 Python 代码可提速 4-10 倍
4. **Python 3.11~3.13**：CPython 本身快了 25-60%（Faster CPython 项目）

---

## 2. 语法深度解析

### 2.1 一切都是对象

```python
# C++ 中 int 是原始类型，Python 中 int 是对象（PyLongObject）
x = 42
print(type(x))           # <class 'int'>
print(dir(x))            # 列出所有方法和属性，包括 __add__, __mul__...
print(x.__class__)       # <class 'int'>
print(x.bit_length())    # 6 —— int 有方法！

# 函数也是对象
def foo():
    pass
foo.custom_attr = "hello"  # 可以给函数添加属性
print(foo.__name__)         # 'foo'
print(foo.__code__)         # 字节码对象
```

### 2.2 可变 vs 不可变（核心概念）

```python
# ─── 不可变类型 ───
# int, float, str, tuple, frozenset, bytes
# 类似 C++ const 对象，但这是 Python 级别的语义
a = "hello"
# a[0] = "H"             # ❌ TypeError: 'str' object does not support item assignment
a = "Hello"              # ✅ 这其实是创建了新对象，a 指向新地址

# ─── 可变类型 ───
# list, dict, set, bytearray, 自定义类实例
b = [1, 2, 3]
b[0] = 10                # ✅ 原地修改

# ─── 这个区别影响一切 ───
def add_item(lst, item):  # 传入可变对象
    lst.append(item)      # 函数外也生效！

my_list = [1, 2]
add_item(my_list, 3)
print(my_list)            # [1, 2, 3] —— "传引用"效果

def increment(n):         # 传入不可变对象
    n += 1                # 只在函数内创建了新对象

x = 0
increment(x)
print(x)                  # 0 —— 未改变
```

### 2.3 切片：比任何语言都强大

```python
# 切片语法 s[start:end:step]
# 这不仅是语法糖，它由 __getitem__ 的 slice 对象实现

s = list(range(10))         # [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# 基础切片
s[2:5]       # [2, 3, 4]          —— C++ 半开区间风格
s[:5]        # [0, 1, 2, 3, 4]
s[5:]        # [5, 6, 7, 8, 9]
s[-3:]       # [7, 8, 9]          —— 最后三个

# step 步长
s[::2]       # [0, 2, 4, 6, 8]    —— 每隔一个
s[::-1]      # [9, 8, 7, 6, 5, 4, 3, 2, 1, 0] —— 反转

# 原地修改切片
s[2:5] = [20, 30]            # [0, 1, 20, 30, 5, 6, 7, 8, 9]
del s[2:4]                   # [0, 1, 5, 6, 7, 8, 9]

# slice 对象（可复用）
my_slice = slice(2, 8, 2)    # 等价于 [2:8:2]
print(s[my_slice])           # [1, 5, 7]

# 切片用于自定义类
class MySequence:
    def __getitem__(self, idx):
        if isinstance(idx, slice):
            return f"slice({idx.start}, {idx.stop}, {idx.step})"
        return f"index {idx}"
```

### 2.4 推导式（Comprehension）完全指南

```python
# ─── 列表推导式 ───
# C++:  需要 std::copy_if + std::transform
# Dart: list.where().map().toList()
# Python: 一行搞定
squares = [x**2 for x in range(10) if x % 2 == 0]
# → [0, 4, 16, 36, 64]

# 嵌套循环（从左到右 = 外层到内层）
pairs = [(x, y) for x in range(3) for y in range(2)]
# → [(0,0), (0,1), (1,0), (1,1), (2,0), (2,1)]

# ─── 字典推导式 ───
square_map = {x: x**2 for x in range(5)}
# → {0:0, 1:1, 2:4, 3:9, 4:16}

# 翻转字典
inverted = {v: k for k, v in original.items()}

# ─── 集合推导式 ───
unique_lengths = {len(w) for w in words}

# ─── 生成器表达式（惰性） ───
# 用小括号代替方括号，不一次性分配内存
gen = (x**2 for x in range(10**9))  # 几乎零内存
sum(gen)  # 边算边累加

# ─── 性能对比 ───
# 推导式 > map/filter > for 循环手工 append
# 原因：推导式在 C 级别执行，避免 Python 级别的循环开销
```

### 2.5 模式匹配（Python 3.10+）

```python
# match/case 比 C++ switch、Dart switch、Swift switch 都强大
# 它是结构匹配，不是值匹配

def process(value):
    match value:
        # 字面量匹配
        case 0:
            return "zero"

        # 类型匹配 + 守卫条件
        case int(n) if n > 0:
            return f"positive {n}"

        case str(s) if len(s) > 10:
            return f"long string: {s[:10]}..."

        # 序列解构匹配
        case [first, second, *rest]:
            return f"list: {first}, {second}, rest={len(rest)}"

        case (x, y):
            return f"point: ({x}, {y})"

        # 字典/映射匹配
        case {"type": "error", "message": str(msg)}:
            return f"Error: {msg}"

        case {"type": "success", "data": data}:
            return f"Success: {data}"

        # 类实例匹配（属性解构）
        case Point(x=0, y=0):
            return "origin point"

        case Point(x=x, y=y):
            return f"point at ({x}, {y})"

        # 或模式
        case "yes" | "y" | "ok":
            return "affirmative"

        # 通配
        case _:
            return "unknown"
```

### 2.6 字符串完全手册

```python
# ─── f-string（3.6+） —— 最常用的字符串格式化 ───
name = "World"
n = 42
pi = 3.14159265

f"Hello {name}"                    # 'Hello World'
f"{n:04d}"                         # '0042' —— 填充
f"{pi:.2f}"                        # '3.14' —— 精度
f"{n:#x}"                          # '0x2a' —— 十六进制
f"{n:b}"                           # '101010' —— 二进制
f"{name:*^20}"                     # '*******World********' —— 居中填充

# f-string 中直接调用表达式
f"2 + 2 = {2 + 2}"
f"len = {len(name)}"
f"upper = {name.upper()!r}"        # !r = repr(), !s = str(), !a = ascii()

# ─── f-string 调试模式（3.8+） ───
f"{name=}"                         # "name='World'" —— 变量名 + 值
f"{n + 1=}"                        # "n + 1=43"

# ─── 多行字符串 ───
sql = """
    SELECT *
    FROM users
    WHERE active = 1
"""

# textwrap.dedent 去除共同缩进
from textwrap import dedent
sql = dedent("""
    SELECT *
    FROM users
    WHERE active = 1
""").strip()

# ─── 字符串方法 ───
"hello world".split()              # ['hello', 'world']
"a,b,c".split(",")                 # ['a', 'b', 'c']
", ".join(["a", "b", "c"])         # 'a, b, c'
"  trim  ".strip()                  # 'trim'
"hello".startswith("he")           # True
"hello".endswith("lo")             # True
"hello world".removeprefix("hello ")  # 'world' (3.9+)
"file.txt".removesuffix(".txt")    # 'file' (3.9+)

# ─── 编码与解码 ───
"hello".encode("utf-8")            # b'hello'
b"hello".decode("utf-8")           # 'hello'
"中文".encode("utf-8")             # b'\xe4\xb8\xad\xe6\x96\x87'
```

### 2.7 异常处理进阶

```python
# ─── 异常链（__cause__ vs __context__） ───
# 类似 C++ std::nested_exception

try:
    try:
        int("abc")
    except ValueError as e:
        raise RuntimeError("Parse failed") from e   # __cause__ 显式链
except RuntimeError as e:
    print(e.__cause__)    # ValueError

# raise ... from None —— 隐藏原因链
try:
    int("abc")
except ValueError:
    raise RuntimeError("Parse failed") from None

# ─── except*（Python 3.11+ ExceptionGroup） ───
# 类似 Swift 的并行任务错误聚合
try:
    raise ExceptionGroup("issues", [
        ValueError("bad value"),
        TypeError("bad type"),
    ])
except* ValueError as eg:
    print(f"Value errors: {eg.exceptions}")
except* TypeError as eg:
    print(f"Type errors: {eg.exceptions}")

# ─── 自定义异常 ───
class AppError(Exception):
    def __init__(self, message: str, code: int = 500):
        self.code = code
        super().__init__(message)

# ─── 常见内置异常层次 ───
# BaseException
#  ├── SystemExit
#  ├── KeyboardInterrupt
#  └── Exception
#       ├── ValueError
#       ├── TypeError
#       ├── KeyError
#       ├── IndexError
#       ├── FileNotFoundError
#       ├── RuntimeError
#       └── ...
```

---

## 3. 类型系统全解

### 3.1 Gradual Typing 哲学

```python
# Python 类型系统是"渐进式"的：
# - 不加注解 = 动态类型（Python 初心）
# - 加注解   = 需要 mypy/pyright 静态检查
# - 运行时完全不检查类型注解！

# 获取类型注解的方式
def greet(name: str) -> str:
    return f"Hello {name}"

print(greet.__annotations__)  # {'name': <class 'str'>, 'return': <class 'str'>}

# typing.get_type_hints() —— 解析字符串注解
from typing import get_type_hints
```

### 3.2 现代类型注解（Python 3.10+）

```python
# ─── 联合类型 ───
# 旧: from typing import Union, Optional
# 新: 直接用 |
def maybe_int(value: str) -> int | None:   # ≈ Dart int?
    try:
        return int(value)
    except ValueError:
        return None

# ─── 泛型 ───
# 旧: from typing import List, Dict, Tuple
# 新: 直接用 list, dict, tuple (3.9+)
def process(items: list[int], mapping: dict[str, float]) -> tuple[int, float]:
    return len(items), sum(mapping.values())

# ─── 可调用对象 ───
from collections.abc import Callable
def apply(func: Callable[[int, int], int], x: int, y: int) -> int:
    return func(x, y)

# ─── 迭代器与生成器 ───
from collections.abc import Iterator, Generator

def count_up(n: int) -> Iterator[int]:
    for i in range(n):
        yield i

def coro() -> Generator[int, str, None]:
    # 生成 int，接收 str，返回 None
    received = yield 42
    yield len(received)

# ─── TypedDict（有类型的字典） ───
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int
    email: str | None

def create_user(data: User) -> None:
    print(data["name"])        # 类型检查器知道有 name 键

# ─── Protocol（结构化子类型 = Swift Protocol / C++ Concepts） ───
from typing import Protocol

class Flyable(Protocol):
    def fly(self) -> None: ...

def make_it_fly(obj: Flyable) -> None:
    obj.fly()

class Bird:
    def fly(self) -> None:
        print("Bird flying")

class Plane:
    def fly(self) -> None:
        print("Plane flying")

make_it_fly(Bird())   # ✅ 无需显式实现 Flyable
make_it_fly(Plane())  # ✅ 鸭子类型 + 静态检查
```

### 3.3 高级类型技巧

```python
# ─── Literal（枚举值类型） ───
from typing import Literal
def set_mode(mode: Literal["read", "write", "append"]) -> None: ...

# ─── TypeGuard / TypeIs（类型收窄） ───
from typing import TypeGuard

def is_str_list(val: list[object]) -> TypeGuard[list[str]]:
    return all(isinstance(x, str) for x in val)

items: list[object] = ["a", "b"]
if is_str_list(items):
    items[0].upper()  # 类型收窄为 list[str]

# ─── Final（禁止重写/重赋值） ───
from typing import Final
MAX_SIZE: Final = 1024
# MAX_SIZE = 2048  # mypy 报错

class Base:
    def method(self) -> None: ...

class Derived(Base):
    @final  # 禁止子类重写
    def method(self) -> None: ...

# ─── overload（函数重载） ───
from typing import overload

@overload
def get_value(key: str) -> str: ...
@overload
def get_value(key: str, default: int) -> str | int: ...
def get_value(key: str, default=None):
    # 实际实现
    return config.get(key, default)

# ─── NamedTuple —— 不可变的 typed tuple ───
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float
    label: str = ""

p = Point(3.0, 4.0)
print(p.x, p.y)        # 3.0 4.0
# p.x = 5.0             # ❌ AttributeError (不可变)

# ─── Self 类型（3.11+） ───
from typing import Self

class Builder:
    def set_x(self, x: int) -> Self:
        self.x = x
        return self
```

### 3.4 类型对应速查表

| C++ | ObjC | Swift | Dart | Python |
|-----|------|-------|------|--------|
| `int`/`long long` | `NSInteger` | `Int` | `int` | `int`（任意精度） |
| `double` | `CGFloat` | `Double` | `double` | `float`（64-bit） |
| `std::complex` | — | — | — | `complex`（内置） |
| `bool` | `BOOL` | `Bool` | `bool` | `bool` |
| `std::string` | `NSString` | `String` | `String` | `str`（不可变 Unicode） |
| `std::vector<T>` | `NSArray< T >` | `[T]` | `List<T>` | `list[T]` |
| `std::map<K,V>` | `NSDictionary` | `[K:V]` | `Map<K,V>` | `dict[K,V]` |
| `std::set<T>` | `NSSet< T >` | `Set<T>` | `Set<T>` | `set[T]` |
| `std::tuple<...>` | — | `(T, U)` | `(T, U)` | `tuple[T, ...]` |
| `nullptr` | `nil` | `nil` | `null` | `None` |
| `std::optional<T>` | `nullable` | `T?` | `T?` | `T \| None` |
| `std::variant<...>` | — | `enum` w/ assoc. | `sealed class` | `T \| U`（联合类型） |
| `enum class` | `NS_ENUM` | `enum` | `enum` | `enum` / `Literal` |
| `std::function<...>` | Block | closure | `Function` | `Callable[[...], R]` |
| `size_t` | `NSUInteger` | `UInt` | `int` | `int` |

### 3.5 dataclass：告别样板代码

```python
from dataclasses import dataclass, field

# 类似 Swift struct / Dart data class / C++ struct
# 自动生成 __init__, __repr__, __eq__

@dataclass(order=True, frozen=True)  # frozen = 不可变
class Person:
    name: str
    age: int
    tags: list[str] = field(default_factory=list)  # 可变默认值必须用 default_factory
    _id: int = field(default=0, repr=False)       # repr=False 不打印

    @property
    def is_adult(self) -> bool:
        return self.age >= 18

# 使用
p1 = Person("Alice", 30)
p2 = Person("Bob", 25)
print(p1)               # Person(name='Alice', age=30)
print(p1 == p2)         # False
# p1.age = 31           # ❌ frozen=True 禁止修改

# 用 replace 创建修改后的副本
p3 = p2  # 直接引用
# 没有 replace。frozen 时用:
import dataclasses
p3 = dataclasses.replace(p1, age=31)  # 创建新实例
```

---

## 4. 函数、闭包与装饰器

### 4.1 一等公民：函数是对象

```python
# 函数可以赋值、传参、返回、存储
def add(a, b):
    return a + b

ops = {"+": add, "-": lambda a, b: a - b}
result = ops["+"](1, 2)   # 3

# 函数属性
add.author = "shelton"
print(add.__name__)        # 'add'
print(add.__doc__)         # None（可设置 docstring）
print(add.__defaults__)    # 默认参数值元组
print(add.__code__)        # 字节码对象
print(add.__closure__)     # 闭包变量（如有）
```

### 4.2 参数传递全解

```python
# 参数分类（按 PEP 3102/570）：
# 1. 位置参数（positional-only，3.8+ / 之前的参数）
# 2. 位置或关键字参数（默认）
# 3. 仅限关键字参数（keyword-only，* 之后）
# 4. 可变位置参数（*args）
# 5. 可变关键字参数（**kwargs）

def f(pos_only, /, pos_or_kw, *, kw_only, **kwargs):
    """
    pos_only: 必须按位置传递（C 扩展常用）
    pos_or_kw: 位置或关键字皆可
    kw_only: 必须按关键字传递
    """
    pass

f(1, 2, kw_only=3)            # ✅
f(1, pos_or_kw=2, kw_only=3)  # ✅
# f(pos_only=1, ...)           # ❌ pos_only 不能用关键字

# ─── 解包传递 ───
args = [1, 2, 3]
func(*args)              # 解包列表为位置参数
kwargs = {"a": 1, "b": 2}
func(**kwargs)           # 解包字典为关键字参数

# ─── 仅限位置参数的实际用途 ───
# 避免参数名冲突（如 dict 的 setdefault）
dict.setdefault(key, default)  # key 不能用关键字
```

### 4.3 闭包与作用域（LEGB 详解）

```python
# LEGB 变量查找规则：
# Local → Enclosing → Global → Built-in

x = "global"          # G
def outer():
    x = "enclosing"   # E
    def inner():
        x = "local"   # L
        print(x)      # → "local"
    inner()

# ─── 闭包的陷阱 ───
# ❌ 经典错误：循环中创建闭包
funcs = []
for i in range(3):
    funcs.append(lambda: i)  # 所有 lambda 都引用同一个 i

print([f() for f in funcs])  # [2, 2, 2] 不是 [0, 1, 2]！

# ✅ 修复：默认参数在定义时求值
funcs = []
for i in range(3):
    funcs.append(lambda i=i: i)  # i=i 在定义时捕获值

print([f() for f in funcs])  # [0, 1, 2]

# ─── nonlocal 详解 ───
def make_counter():
    count = 0
    def counter():
        nonlocal count      # 没有 nonlocal，count += 1 会创建新的 local 变量
        count += 1          # （因为不可变类型 += 实际上是 rebind）
        return count
    return counter

c = make_counter()
print(c())  # 1
print(c())  # 2
print(c())  # 3
```

### 4.4 装饰器基础

```python
# 装饰器本质：高阶函数，接受函数，返回函数
# 语法 @decorator 等价于 func = decorator(func)

# ─── 最简单的装饰器 ───
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"{func.__name__} returned {result}")
        return result
    return wrapper

@log_call
def add(a, b):
    return a + b

add(1, 2)
# Calling add
# add returned 3

# ─── 保留函数元数据 ───
from functools import wraps

def log_call(func):
    @wraps(func)  # 复制 __name__, __doc__, __module__ 等
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

# ─── 带参数的装饰器 ───
# 本质：写一个返回装饰器的函数
def repeat(times: int):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def greet(name):
    print(f"Hello {name}")

greet("World")   # 打印三次

# ─── 类装饰器 ───
# 装饰器也可以是类，__init__ 接收 func，__call__ 执行逻辑
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0
        wraps(func)(self)

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call {self.count} of {self.func.__name__}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hello():
    print("hello!")
```

### 4.5 常用内置装饰器

```python
# ─── @staticmethod ───
# C++/Dart/Swift 中的静态方法
class Math:
    @staticmethod
    def add(a, b):
        return a + b

# ─── @classmethod ───
# 第一个参数是类本身(cls)，常用于工厂方法
class Date:
    def __init__(self, year, month, day):
        self.year, self.month, self.day = year, month, day

    @classmethod
    def from_string(cls, date_str: str):
        year, month, day = map(int, date_str.split("-"))
        return cls(year, month, day)

# ─── @property / @x.setter / @x.deleter ───
# ≈ Swift computed property / Dart getter+setter / ObjC @property
class Temperature:
    def __init__(self, celsius: float):
        self._celsius = celsius

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value: float):
        self._celsius = (value - 32) * 5/9

# ─── @cached_property（3.8+） ───
# 惰性计算 + 只计算一次
from functools import cached_property

class DataLoader:
    @cached_property
    def data(self):
        print("Loading...")  # 只打印一次
        return [1, 2, 3]

dl = DataLoader()
dl.data  # Loading... [1, 2, 3]
dl.data  # [1, 2, 3]  —— 不再打印

# ─── @dataclass（3.7+） —— 见 3.5 节 ───
```

---

## 5. 面向对象深度剖析

### 5.1 Python 对象模型

```python
# Python 对象 = __dict__（属性字典） + 类型指针 + 引用计数
# 所有对象的属性默认存储在 __dict__ 中（字典开销 ~96 bytes）

class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

p = Point(1, 2)
print(p.__dict__)    # {'x': 1, 'y': 2}

# ─── __slots__：去掉 __dict__，节省内存 ───
# 类似 C++ 固定布局 struct，没有动态属性能力
class PointCompact:
    __slots__ = ('x', 'y')   # 只允许这两个属性

    def __init__(self, x, y):
        self.x = x
        self.y = y

pc = PointCompact(1, 2)
# pc.z = 3                 # ❌ AttributeError
# print(pc.__dict__)       # ❌ 没有 __dict__

# 内存对比（粗略）：
# 普通对象：~56 bytes (PyObject) + 96 bytes (__dict__ overhead)
# __slots__：~56 bytes + 8 bytes × N slots
```

### 5.2 继承与 MRO（方法解析顺序）

```python
# Python 多继承用 C3 线性化算法
# MRO = Method Resolution Order

class A:
    def method(self):
        return "A"

class B(A):
    def method(self):
        return "B"

class C(A):
    def method(self):
        return "C"

class D(B, C):
    pass

# 查看 MRO
print(D.__mro__)   # D → B → C → A → object
print(D().method())  # "B" —— 按 MRO 第一个找到的

# ─── super() 的精妙之处 ───
# super() 不是"调用父类"，而是"按 MRO 找到下一个"
class A:
    def __init__(self):
        print("A")
        super().__init__()  # 不是调用 object！按 MRO 继续

class B:
    def __init__(self):
        print("B")
        super().__init__()

class C(A, B):
    pass

C()  # 输出: A B —— MRO 是 C→A→B→object，A 的 super() 调到了 B

# ─── Mixin 模式 ───
# 类似 Swift Protocol Extension / Dart Mixin
class JSONMixin:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)

class LoggingMixin:
    def log(self, msg: str) -> None:
        print(f"[{self.__class__.__name__}] {msg}")

class User(JSONMixin, LoggingMixin):
    def __init__(self, name: str):
        self.name = name
```

### 5.3 魔术方法完全手册

```python
# Python 魔术方法（dunder methods）决定了对象的"协议"
# 实现对应方法 = 支持对应语法/操作

class Matrix:
    def __init__(self, data: list[list[float]]):
        self.data = data

    # ─── 表示 ───
    def __repr__(self) -> str:
        return f"Matrix({self.data!r})"

    def __str__(self) -> str:
        return "\n".join(" ".join(f"{x:4}" for x in row) for row in self.data)

    # ─── 长度/布尔 ───
    def __len__(self) -> int:
        return len(self.data)

    def __bool__(self) -> bool:
        return any(any(row) for row in self.data)

    # ─── 比较 ───
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Matrix):
            return NotImplemented
        return self.data == other.data

    def __hash__(self) -> int:  # 如果定义了 __eq__ 且需要可哈希
        return hash(tuple(tuple(row) for row in self.data))

    # ─── 运算符 ───
    def __add__(self, other: "Matrix") -> "Matrix":
        return Matrix([
            [a + b for a, b in zip(row1, row2)]
            for row1, row2 in zip(self.data, other.data)
        ])

    def __neg__(self) -> "Matrix":
        return Matrix([[-x for x in row] for row in self.data])

    def __matmul__(self, other: "Matrix") -> "Matrix":  # @ 运算符
        # 矩阵乘法...
        pass

    # ─── 下标访问 ───
    def __getitem__(self, idx: tuple[int, int]) -> float:
        row, col = idx
        return self.data[row][col]

    def __setitem__(self, idx: tuple[int, int], value: float):
        row, col = idx
        self.data[row][col] = value

    # ─── 迭代器协议 ───
    def __iter__(self):
        return iter(self.data)  # 迭代行

    # ─── 可调用 ───
    def __call__(self, vector):
        """使实例可以像函数一样调用"""
        return [sum(a * b for a, b in zip(row, vector)) for row in self.data]

    # ─── 上下文管理器 ───
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        pass
```

完整魔术方法速查：

| 类别 | 方法 | 触发 |
|------|------|------|
| 创建/销毁 | `__init__`, `__new__`, `__del__` | 构造、创建、析构 |
| 表示 | `__repr__`, `__str__`, `__format__` | `repr()`, `str()`, f-string |
| 比较 | `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__`, `__hash__` | `==`, `!=`, `<`, `<=`, `>`, `>=`, `hash()` |
| 算术 | `__add__`, `__sub__`, `__mul__`, `__truediv__`, `__floordiv__`, `__mod__`, `__pow__`, `__matmul__` | `+`, `-`, `*`, `/`, `//`, `%`, `**`, `@` |
| 增强赋值 | `__iadd__`, `__isub__`, ... | `+=`, `-=`, ... |
| 一元 | `__neg__`, `__pos__`, `__abs__`, `__invert__` | `-x`, `+x`, `abs(x)`, `~x` |
| 类型转换 | `__int__`, `__float__`, `__bool__`, `__bytes__` | `int()`, `float()`, `bool()`, `bytes()` |
| 容器 | `__len__`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__` | `len(x)`, `x[i]`, `x[i]=v`, `del x[i]`, `in` |
| 迭代 | `__iter__`, `__next__`, `__reversed__` | `for`, `next()`, `reversed()` |
| 属性 | `__getattr__`, `__setattr__`, `__delattr__`, `__getattribute__` | `obj.x`, `obj.x=v`, `del obj.x` |
| 描述符 | `__get__`, `__set__`, `__delete__` | 见第 12 章 |
| 上下文 | `__enter__`, `__exit__` | `with obj:` |
| 调用 | `__call__` | `obj()` |

### 5.4 访问控制真相

```python
# Python 没有真正的 private！
# 约定（靠自觉）：
# - _name      : "保护"属性，别碰（但能访问）
# - __name     : 名称改写（name mangling），_ClassName__name
# - __name__   : 系统魔术方法，别自己定义这种名字

class MyClass:
    def __init__(self):
        self.public = 1
        self._protected = 2          # 约定：内部使用
        self.__private = 3           # 名称改写为 _MyClass__private

obj = MyClass()
print(obj.public)                    # 1
print(obj._protected)                # 2 —— 能访问，但不应该
print(obj._MyClass__private)         # 3 —— 名称改写不是安全机制！
# print(obj.__private)               # ❌ AttributeError

# 名称改写的目的：避免子类属性名冲突，不是安全性
```

---

## 6. 内存管理内幕

### 6.1 引用计数机制

```python
import sys

# 每个对象都有一个引用计数（ob_refcnt）
a = []          # 空列表，refcnt = 1
b = a           # refcnt = 2
c = b           # refcnt = 3
del a           # refcnt = 2
del b           # refcnt = 1

# 查看引用计数（注意：getrefcount 的参数本身加 1）
print(sys.getrefcount(c))  # 2 （c + getrefcount 的参数）

# ─── 引用计数的代价 ───
# 每次赋值、传参、返回值都涉及引用计数操作
# 这就是为什么 CPython 的单线程比 C++ 慢很多的基础原因之一

# ─── 引用计数的线程安全 ───
# GIL 保证了引用计数操作的原子性
# 这也是移除 GIL 的最大难点之一
```

### 6.2 循环引用与分代 GC

```python
import gc

# ─── 循环引用 ───
# 引用计数无法处理循环引用：
class Node:
    def __init__(self):
        self.ref = None

a = Node()
b = Node()
a.ref = b   # a→b, b refcnt=2 (b 变量 + a.ref)
b.ref = a   # b→a, a refcnt=2 (a 变量 + b.ref)
del a       # a refcnt=1 (b.ref 还指着)
del b       # b refcnt=1 (a.ref 还指着)
# 此时 a 和 b 形成环，都不可达但 refcnt > 0

# CPython 的解决方案：分代垃圾回收
# - Gen 0：新生对象（最频繁回收）
# - Gen 1：存活过一次 GC 的对象
# - Gen 2：长期存活的对象（最少回收）

# ─── 手动控制 GC ───
gc.collect()       # 立即触发完整 GC
gc.disable()       # 禁用自动 GC（适合短期高吞吐场景）
gc.enable()        # 重新启用
print(gc.get_threshold())  # (700, 10, 10) —— Gen0/1/2 的阈值

# ─── __del__ 的坑 ───
# 不要依赖 __del__！调用时机不确定，且在循环引用中可能永远不会被调用
# 标准做法：用 with 语句（上下文管理器）或显式 close()
```

### 6.3 弱引用

```python
import weakref

# 弱引用：不增加引用计数
# 类似 ObjC 的 weak / Swift 的 weak

class BigObject:
    def __del__(self):
        print("BigObject deleted")

obj = BigObject()
weak = weakref.ref(obj)   # 创建弱引用
print(weak())              # <BigObject ...> —— 弱引用可调用取回对象

del obj                    # 引用计数归零，对象释放
print(weak())              # None —— 对象已释放

# ─── WeakKeyDictionary / WeakValueDictionary ───
# 缓存场景：当键/值不再被其他地方引用时自动清除
cache = weakref.WeakValueDictionary()
cache["key"] = BigObject()
# 当 BigObject 没有其他强引用时，自动从 cache 中移除

# ─── WeakSet ───
import weakref
registry = weakref.WeakSet()
registry.add(obj)
# obj 释放时自动从 registry 移除
```

### 6.4 上下文管理器深入

```python
# C++ RAII 是最优美的资源管理模式之一
# Python 的 with 语句实现了类似效果，但机制不同

# ─── 类方式 ───
class ManagedResource:
    def __init__(self, name):
        self.name = name

    def __enter__(self):
        print(f"Acquiring {self.name}")
        return self          # 返回给 as 变量

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"Releasing {self.name}")
        if exc_type:
            print(f"Exception occurred: {exc_val}")
            # return True 会吞掉异常（不推荐）
        return False          # False = 不吞异常

# ─── 生成器方式（contextlib） ───
from contextlib import contextmanager

@contextmanager
def managed(name):
    print(f"Acquiring {name}")
    try:
        yield name           # ← 执行点，值传给 as
    finally:
        print(f"Releasing {name}")  # 无论如何都执行（≈ RAII 析构）

# ─── 实用组合器 ───
from contextlib import ExitStack

with ExitStack() as stack:
    files = [stack.enter_context(open(f"file_{i}.txt")) for i in range(10)]
    # 退出时自动关闭所有文件（逆序）

# contextlib.closing —— 给不实现 context manager 的对象加 close
from contextlib import closing
with closing(SomeLegacyObject()) as obj:
    obj.do_work()

# contextlib.suppress —— 忽略特定异常
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove("maybe_not_exist.txt")

# contextlib.nullcontext —— 可选上下文
with (contextmanager(lambda: iter([]))() if condition else managed("real")):
    pass
```

### 6.5 内存分析工具

```python
# ─── sys.getsizeof ───
import sys
print(sys.getsizeof([1, 2, 3]))    # 列表对象本身的大小（不含元素）
print(sys.getsizeof(42))           # 28 bytes —— 一个 int 的大小！

# ─── tracemalloc ───
import tracemalloc
tracemalloc.start()
# ... 运行代码 ...
snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:5]:
    print(stat)

# ─── objgraph ───
# pip install objgraph
import objgraph
objgraph.show_most_common_types(limit=10)
objgraph.show_backrefs([obj], max_depth=3, filename="backrefs.png")

# ─── memory_profiler ───
# pip install memory-profiler
# @profile
# def my_func():
#     ...
```

---

## 7. 性能分析与优化

### 7.1 性能分析工具链

```python
# ─── timeit（微基准测试） ───
import timeit
timeit.timeit("'-'.join(str(n) for n in range(100))", number=10000)

# ─── cProfile（确定性 profiler） ───
python -m cProfile -s cumulative my_script.py

# 在代码中使用：
import cProfile, pstats
profiler = cProfile.Profile()
profiler.enable()
# ... 被测试代码 ...
profiler.disable()
pstats.Stats(profiler).sort_stats('cumtime').print_stats(20)

# ─── py-spy（采样 profiler，可附加到运行中的进程） ───
# pip install py-spy
# py-spy top -- python my_script.py
# py-spy record -o profile.svg -- python my_script.py

# ─── line_profiler（逐行分析） ───
# pip install line_profiler
# @profile
# def slow_function():
#     ...
# kernprof -l -v my_script.py
```

### 7.2 编写高性能 Python

```python
# ─── 1. 使用局部变量 ───
# 局部变量查找是数组索引（LOAD_FAST），全局变量查找是字典查找（LOAD_GLOBAL）
import math
def compute_slow():
    return math.sqrt(2)        # LOAD_GLOBAL 'math' + LOAD_ATTR 'sqrt'

def compute_fast():
    sqrt = math.sqrt           # 捕获到局部
    return sqrt(2)             # LOAD_FAST 'sqrt'

# ─── 2. 推导式 > map/filter > for 循环 ───
# 推导式在 C 级别执行，避开 Python 循环开销
import timeit
# 最慢
def with_for():
    result = []
    for x in range(1000):
        result.append(x * x)
    return result

# 中等
def with_map():
    return list(map(lambda x: x * x, range(1000)))

# 最快
with_comp = lambda: [x * x for x in range(1000)]

# ─── 3. 字符串拼接：用 ''.join() ───
# ❌ result = ""
#    for s in strings:
#        result += s        # 每次创建新字符串（不可变）

# ✅ result = "".join(strings)  # C 级别一次性分配

# ─── 4. 避免属性访问在循环中 ───
# ❌ for i in range(len(items)):
#        items[i].name       # 每次循环都做属性查找

# ✅ for item in items:
#        item.name

# ─── 5. __slots__ 减少内存 ───
# 见 5.1 节

# ─── 6. 使用 array 模块（同类型数组） ───
import array
numbers = array.array('i', [1, 2, 3, 4, 5])  # 紧凑 C 数组

# ─── 7. functools.lru_cache ───
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive(n):
    return n ** n
```

### 7.3 NumPy 入门（计算引擎）

```python
# NumPy 是 Python 科学计算的基础，底层是 C/Fortran
# C++ 开发者应该把它理解为"带 Python 接口的向量化计算库"

import numpy as np

# ─── 数组创建 ───
a = np.array([1, 2, 3])         # 从列表
b = np.zeros((3, 4))             # 3×4 零矩阵
c = np.ones((2, 3))              # 2×3 全一矩阵
d = np.arange(10)                # [0, 1, ..., 9]
e = np.linspace(0, 1, 100)       # 0 到 1 等间距 100 点
f = np.random.randn(1000)        # 正态分布

# ─── 向量化操作（没有 Python 循环！） ───
# ❌ result = [x * 2 for x in data]     # Python 循环
# ✅ result = data * 2                  # C 级别广播

# ─── 广播（Broadcasting） ───
a = np.array([[1, 2, 3], [4, 5, 6]])   # (2, 3)
b = np.array([10, 20, 30])              # (3,)
print(a + b)   # [[11, 22, 33], [14, 25, 36]]

# ─── 索引与切片 ───
a = np.arange(12).reshape(3, 4)
a[1, 2]           # 单个元素
a[:, 1]           # 第二列
a[a > 5]          # 布尔索引

# 与 C++ 的对比：
# C++:    需要 for 循环处理 std::vector
# NumPy:  arr[arr > 5] 一行搞定，且运行在 C 速度
```

---

## 8. C 扩展与 FFI

### 8.1 何时需要 C 扩展

- 纯 Python 计算的循环无法优化
- 需要调用现有 C/C++ 库
- CPU 密集型且无法用 NumPy 向量化
- 需要绕过 GIL

### 8.2 Cython（推荐首选）

```cython
# setup.py
# from setuptools import setup
# from Cython.Build import cythonize
# setup(ext_modules=cythonize("my_module.pyx"))

# my_module.pyx
def fibonacci(int n):
    cdef int a = 0, b = 1, i
    cdef list result = []
    for i in range(n):
        result.append(a)
        a, b = b, a + b
    return result

# 带类型的 Cython 可以接近 C 速度
# 类型声明用 cdef，Python 对象用 def
```

### 8.3 ctypes / cffi（调用 C 库）

```python
# ─── ctypes（标准库） ───
import ctypes

# 加载 C 标准库
libc = ctypes.CDLL("libc.dylib")  # macOS
# libc = ctypes.CDLL("libc.so.6")  # Linux

# 调用 printf
libc.printf(b"Hello %s\n", b"World")

# ─── cffi（更快、更 Pythonic） ───
# pip install cffi
from cffi import FFI
ffi = FFI()

ffi.cdef("""
    int printf(const char *format, ...);
""")
C = ffi.dlopen(None)
C.printf(b"Hello %s\n", b"World")
```

### 8.4 pybind11（C++ → Python）

```cpp
// C++ 代码用 pybind11 导出
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>

double add(double a, double b) {
    return a + b;
}

PYBIND11_MODULE(my_module, m) {
    m.doc() = "My module";
    m.def("add", &add, "A function that adds two numbers");
}
```

### 8.5 Numba（JIT 编译）

```python
from numba import jit, njit

# 在纯 Python 函数上加装饰器即可 JIT 编译
@njit
def monte_carlo_pi(nsamples: int) -> float:
    import random, math  # Numba 有自己的 random 实现
    acc = 0
    for _ in range(nsamples):
        x = random.random()
        y = random.random()
        if (x**2 + y**2) < 1.0:
            acc += 1
    return 4.0 * acc / nsamples

# 首次调用会编译，后续调用接近 C 速度
print(monte_carlo_pi(10_000_000))
```

---

## 9. 并发模型深度解析

### 9.1 GIL 真相

```python
# GIL（Global Interpreter Lock）是 CPython 最被误解的特性

# ─── 什么是 GIL ───
# 一个互斥锁，保证同一时刻只有一个线程执行 Python 字节码
# 原因：CPython 的引用计数不是线程安全的，GIL 是最简单的解决方案

# ─── GIL 影响什么 ───
# ✅ IO 密集型（网络请求、文件读写）：GIL 在 IO 时释放，多线程有效
# ❌ CPU 密集型（计算、解析）：多线程反而比单线程慢（上下文切换 + 锁竞争）

# ─── 绕过 GIL 的方式 ───
# 1. 多进程（multiprocessing）—— 每个进程有独立 GIL
# 2. C 扩展中可以手动释放 GIL
# 3. 用 Cython/Numba 编译
# 4. 用其他 Python 实现：Jython（无 GIL）、PyPy-STM
# 5. Python 3.13 引入的实验性 no-GIL 模式（PEP 703）

# ─── GIL 释放时机 ───
# - IO 操作（文件读写、socket、sleep）
# - 显式释放：在 C 扩展中调用 Py_BEGIN_ALLOW_THREADS
# - 每执行 N 条字节码指令释放一次（sys.setswitchinterval）
```

### 9.2 threading 模块

```python
import threading
import time

# ─── 基本线程 ───
def worker(name: str, delay: float):
    for i in range(3):
        time.sleep(delay)        # sleep 时释放 GIL
        print(f"{name}: step {i}")

t1 = threading.Thread(target=worker, args=("A", 0.5))
t2 = threading.Thread(target=worker, args=("B", 0.3))
t1.start()
t2.start()
t1.join()
t2.join()

# ─── Lock ───
lock = threading.Lock()
shared = 0

def increment():
    global shared
    for _ in range(1_000_000):
        with lock:
            shared += 1

# ─── RLock（可重入锁） ───
rlock = threading.RLock()
rlock.acquire()
rlock.acquire()    # 同一线程可以重复获取
rlock.release()
rlock.release()

# ─── Condition（条件变量 ≈ C++ std::condition_variable） ───
cv = threading.Condition()
items = []

def consumer():
    with cv:
        while not items:
            cv.wait()       # 等待通知
        item = items.pop()
        print(f"Got {item}")

def producer():
    with cv:
        items.append(42)
        cv.notify()         # 通知一个等待者

# ─── Semaphore ───
sem = threading.Semaphore(5)  # 最多 5 个并发

# ─── Event（≈ Dart Completer 用于一次信号） ───
event = threading.Event()

def waiter():
    print("Waiting...")
    event.wait()
    print("Done!")

def notifier():
    time.sleep(2)
    event.set()

# ─── local（线程本地存储） ───
local_data = threading.local()
local_data.user_id = 123   # 每个线程独立

# ─── Barrier（同步屏障） ───
barrier = threading.Barrier(3)   # 等 3 个线程都到达
```

### 9.3 concurrent.futures（高层 API）

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor, as_completed
import time

# ─── ThreadPoolExecutor ───
# 适合 IO 密集型
def fetch_url(url: str) -> str:
    time.sleep(1)  # 模拟 IO
    return f"Response from {url}"

with ThreadPoolExecutor(max_workers=5) as executor:
    urls = [f"http://api/{i}" for i in range(10)]

    # map 方式：保持顺序
    results = executor.map(fetch_url, urls)

    # submit 方式：按完成顺序
    futures = {executor.submit(fetch_url, url): url for url in urls}
    for future in as_completed(futures):
        url = futures[future]
        try:
            result = future.result(timeout=10)
            print(f"{url} → {result}")
        except Exception as e:
            print(f"{url} failed: {e}")

# ─── ProcessPoolExecutor ───
# 适合 CPU 密集型（绕过 GIL）
def cpu_intensive(n: int) -> int:
    return sum(i * i for i in range(n))

with ProcessPoolExecutor(max_workers=4) as executor:
    results = executor.map(cpu_intensive, [10**6, 10**6, 10**6, 10**6])
```

### 9.4 multiprocessing 模块

```python
from multiprocessing import Process, Queue, Pool, Manager, Value, Array
import os

# ─── 子进程 ───
def child(name: str):
    print(f"Child {name}, PID={os.getpid()}")

p = Process(target=child, args=("worker",))
p.start()
p.join()

# ─── Queue（进程间通信） ───
# 基于管道 + 序列化（pickle）
q = Queue()

def producer(q: Queue):
    for i in range(5):
        q.put(i)

def consumer(q: Queue):
    while True:
        item = q.get()
        if item is None:  # 哨兵值
            break
        print(item)

# ─── Pipe（双向通信） ───
parent_conn, child_conn = Pipe()

# ─── 共享内存 ───
# Value：单个值；Array：数组
counter = Value('i', 0)      # 'i' = signed int
arr = Array('d', [0.0] * 10) # 'd' = double

# ─── Manager（共享 Python 对象） ───
# 比 Queue/Pipe 慢，但支持任意 Python 对象
with Manager() as manager:
    shared_dict = manager.dict()
    shared_list = manager.list()

# ─── Pool（进程池） ───
with Pool(processes=4) as pool:
    results = pool.map(cpu_intensive, [10**6, 10**6, 10**6])
    # 异步版本：
    async_result = pool.map_async(cpu_intensive, [10**6] * 4)
    results = async_result.get(timeout=60)
```

---

## 10. asyncio 完全指南

### 10.1 事件循环与协程

```python
import asyncio

# ─── 协程对象 vs 协程函数 ───
async def coro_function():    # 协程函数（返回协程对象）
    return 42

coro = coro_function()        # 调用返回协程对象，不会执行！

# ─── 三种运行方式 ───
# 1. asyncio.run() —— 顶层入口（3.7+）
asyncio.run(coro_function())

# 2. await —— 在另一个协程中
async def main():
    result = await coro_function()
    print(result)

# 3. create_task —— 在事件循环中调度
async def main2():
    task = asyncio.create_task(coro_function())
    result = await task

# ─── await 的本质 ───
# await 是协作式多任务的核心：
# - 遇到 await 时，当前协程暂停
# - 将控制权交还给事件循环
# - 事件循环找下一个可执行的协程
# - 被 await 的 Future 完成时，当前协程恢复

# 类比：
# C++20: co_await
# Dart:   await（几乎一样）
# Swift:  await（Swift 5.5+，结构化并发）
```

### 10.2 Task 与并发

```python
import asyncio
import time

async def fetch(id: int) -> str:
    await asyncio.sleep(1)   # 模拟 IO（释放控制权）
    return f"Result {id}"

async def main():
    # ─── 并发执行 ───
    # gather：等待所有完成，返回结果列表
    results = await asyncio.gather(
        fetch(1), fetch(2), fetch(3)
    )   # 约 1 秒（不是 3 秒）
    print(results)  # ['Result 1', 'Result 2', 'Result 3']

    # ─── 按完成顺序处理 ───
    tasks = [asyncio.create_task(fetch(i)) for i in range(5)]
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)

    # ─── wait：更灵活的控制 ───
    done, pending = await asyncio.wait(
        tasks, timeout=2, return_when=asyncio.FIRST_COMPLETED
    )

    # ─── TaskGroup（3.11+，结构化并发 ≈ Swift TaskGroup） ───
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(fetch(1))
        task2 = tg.create_task(fetch(2))
        # with 块退出时自动等待所有 task

    # ─── 超时 ───
    try:
        result = await asyncio.wait_for(slow_operation(), timeout=5.0)
    except asyncio.TimeoutError:
        print("Timed out!")

asyncio.run(main())
```

### 10.3 同步原语（协程安全版本）

```python
# asyncio 提供了协程安全的同步原语（不同于 threading）
# 在 await 处释放控制权，而非阻塞

# ─── Lock ───
lock = asyncio.Lock()

async def critical_section():
    async with lock:
        await do_io()  # 持有锁期间可以 await！

# ─── Semaphore（限流） ───
sem = asyncio.Semaphore(10)  # 最多 10 个并发协程
async with sem:
    await fetch_url(url)

# ─── Event ───
event = asyncio.Event()
await event.wait()    # 等待
event.set()           # 通知所有等待者

# ─── Condition ───
cond = asyncio.Condition()
async with cond:
    await cond.wait()   # 等待通知
    # ...

# ─── Queue（生产者消费者） ───
queue = asyncio.Queue(maxsize=100)
await queue.put(item)
item = await queue.get()

# ─── BoundedSemaphore ───
# 同上但限制 release() 次数不超过 acquire()
```

### 10.4 流与异步迭代

```python
import asyncio

# ─── 异步生成器（PEP 525） ───
async def ticker(delay, to):
    """异步生成器：生成值之间有 await"""
    for i in range(to):
        yield i
        await asyncio.sleep(delay)

async def main():
    async for i in ticker(0.5, 5):  # 每 0.5 秒打印一个数
        print(i)

# ─── 异步推导式（3.6+） ───
async def fetch_items():
    results = [await fetch(i) async for i in async_gen()]
    # 或
    results = [x async for x in async_gen() if x > 0]

# ─── 异步上下文管理器 ───
class AsyncResource:
    async def __aenter__(self):
        await self.connect()
        return self
    async def __aexit__(self, *args):
        await self.disconnect()

async with AsyncResource() as res:
    await res.use()
```

### 10.5 aiohttp（异步 HTTP）

```python
# pip install aiohttp
import aiohttp
import asyncio

async def fetch_url(session: aiohttp.ClientSession, url: str) -> str:
    async with session.get(url) as resp:
        return await resp.text()

async def main():
    async with aiohttp.ClientSession() as session:
        urls = [f"https://httpbin.org/delay/1" for _ in range(10)]

        # 并发 10 个请求 ≈ 1 秒完成
        tasks = [fetch_url(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        print(f"Fetched {len(results)} URLs")

# ─── 连接池与限流 ───
conn = aiohttp.TCPConnector(limit=100, limit_per_host=10)
async with aiohttp.ClientSession(connector=conn) as session:
    ...

# ─── 服务端 ───
from aiohttp import web

async def handle(request):
    return web.Response(text="Hello, async world!")

app = web.Application()
app.add_routes([web.get('/', handle)])
web.run_app(app, port=8080)
```

---

## 11. 并发设计模式

### 11.1 生产者-消费者模式

```python
import asyncio
from asyncio import Queue

async def producer(queue: Queue, n: int):
    for i in range(n):
        await asyncio.sleep(0.1)   # 模拟生产
        await queue.put(i)
        print(f"Produced {i}")
    await queue.put(None)           # 哨兵值

async def consumer(queue: Queue, name: str):
    while True:
        item = await queue.get()
        if item is None:
            await queue.put(None)   # 传给下一个消费者
            break
        await asyncio.sleep(0.3)   # 模拟消费
        print(f"Consumer-{name} processed {item}")
        queue.task_done()

async def main():
    queue = Queue(maxsize=5)
    await asyncio.gather(
        producer(queue, 10),
        consumer(queue, "A"),
        consumer(queue, "B"),
    )
```

### 11.2 扇出/扇入模式

```python
# 一个任务分裂成多个子任务并行执行，结果汇总
async def fan_out_fan_in(items, worker, max_concurrency=10):
    sem = asyncio.Semaphore(max_concurrency)

    async def bounded_worker(item):
        async with sem:
            return await worker(item)

    tasks = [bounded_worker(item) for item in items]
    return await asyncio.gather(*tasks)  # 扇入
```

### 11.3 请求-响应与超时重试

```python
import asyncio

async def with_retry(coro_factory, max_retries=3, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return await asyncio.wait_for(
                coro_factory(),
                timeout=5.0
            )
        except (asyncio.TimeoutError, ConnectionError) as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # 指数退避
            print(f"Retry {attempt+1}/{max_retries} after {delay}s: {e}")
            await asyncio.sleep(delay)
```

### 11.4 Pub/Sub（发布订阅）

```python
import asyncio
from typing import Callable, Awaitable

class PubSub:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = {}

    def subscribe(self, topic: str, callback: Callable):
        self._subscribers.setdefault(topic, []).append(callback)

    async def publish(self, topic: str, message):
        if topic in self._subscribers:
            await asyncio.gather(*[
                cb(message) for cb in self._subscribers[topic]
            ])
```

---

## 12. 描述符协议

### 12.1 描述符本质

```python
# 描述符是实现了 __get__ / __set__ / __delete__ 的对象
# @property、@classmethod、@staticmethod 都是描述符！
# 类似 Swift 的 Property Wrapper

# ─── 非数据描述符：(只有 __get__) ───
# ─── 数据描述符：(有 __get__ + __set__/__delete__) ───
# 数据描述符优先于实例 __dict__

class TypedField:
    """一个类型检查的描述符"""
    def __init__(self, type_: type, default=None):
        self.type = type_
        self.default = default
        self._data = {}  # 不要在描述符实例上存值！用名字索引

    def __set_name__(self, owner, name):  # Python 3.6+
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name, self.default)

    def __set__(self, obj, value):
        if not isinstance(value, self.type):
            raise TypeError(f"{self.name} must be {self.type.__name__}")
        obj.__dict__[self.name] = value

class Person:
    name = TypedField(str)
    age = TypedField(int, default=0)

p = Person()
p.name = "Alice"
p.age = 30
# p.age = "thirty"   # ❌ TypeError
```

### 12.2 描述符查找优先级

```python
# 1. 数据描述符（有 __set__ 或 __delete__）
# 2. 实例 __dict__
# 3. 非数据描述符（只有 __get__）
# 4. 类 __dict__ 中的普通值
# 5. __getattr__ （最后的回退）
```

---

## 13. 元类编程

### 13.1 type 就是元类

```python
# 在 Python 中，类也是对象（type 的实例）
# type 是默认的元类

class Foo:
    pass

print(type(Foo))    # <class 'type'>
print(type(int))    # <class 'type'>
print(type(type))   # <class 'type'> —— type 是它自己的实例！

# ─── 动态创建类 ───
# class Foo: ... 等价于：
Foo = type('Foo', (object,), {'bar': 42})
f = Foo()
print(f.bar)    # 42
```

### 13.2 自定义元类

```python
# 元类控制类的创建过程
# 类似 C++ 模板元编程，但更直接

class SingletonMeta(type):
    """使所有实例化的类成为单例"""
    _instances = {}

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Config(metaclass=SingletonMeta):
    def __init__(self):
        self.settings = {}
```

### 13.3 `__init_subclass__`（Python 3.6+）

```python
# 子类注册钩子——比元类简单得多
class PluginBase:
    plugins = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        name = kwargs.get('name', cls.__name__)
        PluginBase.plugins[name] = cls

class ImagePlugin(PluginBase, name="image"):
    pass

class AudioPlugin(PluginBase, name="audio"):
    pass

print(PluginBase.plugins)
# {'image': <class 'ImagePlugin'>, 'audio': <class 'AudioPlugin'>}
```

---

## 14. 装饰器大全

### 14.1 常用场景

```python
# ─── 计时 ───
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__qualname__}: {elapsed:.4f}s")
        return result
    return wrapper

# ─── 重试 ───
def retry(max_attempts=3, delay=1.0, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts - 1:
                        raise
                    time.sleep(delay * (2 ** attempt))
            return None
        return wrapper
    return decorator

# ─── 缓存 ───
from functools import lru_cache

@lru_cache(maxsize=256)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# ─── 单例 ───
def singleton(cls):
    instances = {}
    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

# ─── 权限检查 ───
def require_role(role: str):
    def decorator(func):
        @wraps(func)
        def wrapper(user, *args, **kwargs):
            if user.role != role:
                raise PermissionError(f"Requires {role}")
            return func(user, *args, **kwargs)
        return wrapper
    return decorator

# ─── 废弃警告 ───
import warnings

def deprecated(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        warnings.warn(f"{func.__name__} is deprecated", DeprecationWarning, stacklevel=2)
        return func(*args, **kwargs)
    return wrapper
```

---

## 15. 上下文管理器深入

### 15.1 实际场景

```python
# ─── 临时目录 ───
import tempfile
from pathlib import Path
from contextlib import contextmanager

@contextmanager
def temp_dir():
    import shutil
    path = Path(tempfile.mkdtemp())
    try:
        yield path
    finally:
        shutil.rmtree(path)

# ─── chdir ───
@contextmanager
def chdir(path: Path):
    import os
    old = os.getcwd()
    os.chdir(path)
    try:
        yield
    finally:
        os.chdir(old)

# ─── 计时器（增强版） ───
@contextmanager
def measure_time(name: str = "block"):
    import time
    start = time.perf_counter_ns()
    yield
    ns = time.perf_counter_ns() - start
    print(f"[{name}] {ns / 1e6:.2f}ms")

# ─── 临时环境变量 ───
@contextmanager
def set_env(**env_vars):
    import os
    old = {k: os.environ.get(k) for k in env_vars}
    os.environ.update(env_vars)
    try:
        yield
    finally:
        for k, v in old.items():
            if v is None:
                del os.environ[k]
            else:
                os.environ[k] = v

# ─── ExitStack：动态管理多个上下文 ───
from contextlib import ExitStack

def process_files(filenames):
    with ExitStack() as stack:
        files = [stack.enter_context(open(f)) for f in filenames]
        # 退出时自动关闭所有文件
        for f in files:
            process(f)
```

---

## 16. 数据结构与算法

### 16.1 collections 模块

```python
from collections import (
    namedtuple, deque, Counter, defaultdict,
    OrderedDict, ChainMap,
)

# ─── namedtuple ───
# 轻量级不可变对象，比 class 更省内存
Point = namedtuple('Point', ['x', 'y'])
p = Point(10, 20)
print(p.x, p.y, p[0], p[1])  # 10 20 10 20
# p.x = 30                     # ❌ 不可变

# ─── deque ───
# 双向队列，O(1) 头尾操作（vs list 的 O(n) 头部操作）
dq = deque([1, 2, 3])
dq.appendleft(0); dq.append(4)
dq.popleft(); dq.pop()
dq.rotate(2)    # 向右旋转

# ─── Counter ───
c = Counter("abracadabra")
print(c.most_common(3))    # [('a', 5), ('b', 2), ('r', 2)]
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
print(c1 + c2)             # Counter({'a': 4, 'b': 3})
print(c1 - c2)             # Counter({'a': 2})
print(c1 & c2)             # min: Counter({'a': 1, 'b': 1})
print(c1 | c2)             # max: Counter({'a': 3, 'b': 2})

# ─── defaultdict ───
# 访问不存在的 key 时自动用工厂函数创建默认值
dd = defaultdict(list)
dd["key"].append(1)       # 不会 KeyError，自动创建空 list

# 嵌套字典
nested = defaultdict(lambda: defaultdict(int))
nested["user"]["score"] += 10

# ─── OrderedDict ───
# 保持插入顺序（Python 3.7+ 普通 dict 也保证顺序，但 OrderedDict
# 额外支持 reorder: move_to_end, popitem(last=False)）
od = OrderedDict([("b", 2), ("a", 1)])
od.move_to_end("b")
od.popitem(last=False)    # FIFO 弹出

# ─── ChainMap ───
# 多个字典的视图，按顺序查找
defaults = {"color": "red", "size": "M"}
user = {"color": "blue"}
combined = ChainMap(user, defaults)
print(combined["color"])  # 'blue' (user 优先)
print(combined["size"])   # 'M' (回退到 defaults)
```

### 16.2 itertools 模块

```python
import itertools

# ─── 无限迭代器 ───
itertools.count(10, 2)      # 10, 12, 14, 16, ...
itertools.cycle("ABC")      # A, B, C, A, B, C, ...
itertools.repeat(42, 3)     # 42, 42, 42

# ─── 组合 ───
items = ['A', 'B', 'C']
list(itertools.permutations(items, 2))   # 排列: AB AC BA BC CA CB
list(itertools.combinations(items, 2))   # 组合: AB AC BC
list(itertools.combinations_with_replacement(items, 2))  # AA AB AC BB BC CC
list(itertools.product('AB', '12'))      # 笛卡尔积: A1 A2 B1 B2

# ─── 处理迭代器 ───
itertools.chain([1,2], [3,4])            # 1, 2, 3, 4
itertools.chain.from_iterable([[1,2], [3,4]])  # 同上
list(itertools.islice(range(100), 10, 20))     # [10..19]

# ─── 分组与过滤 ───
# groupby：按键分组（需要预先排序）
data = [('A', 1), ('A', 2), ('B', 3)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))

# compress：按掩码过滤
itertools.compress('ABCDEF', [1, 0, 1, 0, 1, 1])  # A C E F

# dropwhile / takewhile
itertools.dropwhile(lambda x: x < 5, [1,4,6,4,1])  # 6 4 1
itertools.takewhile(lambda x: x < 5, [1,4,6,4,1])  # 1 4

# ─── 成对迭代（常用技巧） ───
def pairwise(iterable):
    a, b = itertools.tee(iterable)
    next(b, None)
    return zip(a, b)
# itertools.pairwise (Python 3.10+)
```

### 16.3 functools 模块

```python
from functools import (
    lru_cache, cached_property, partial,
    reduce, singledispatch, total_ordering, wraps,
)

# ─── partial（部分应用） ───
# ≈ C++ std::bind
from functools import partial

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
cube = partial(power, exp=3)
print(square(3))   # 9
print(cube(3))     # 27

# ─── reduce ───
from functools import reduce
reduce(lambda a, b: a * b, [1, 2, 3, 4], 1)  # 24

# ─── singledispatch（函数重载） ───
from functools import singledispatch

@singledispatch
def process(value):
    raise NotImplementedError(f"Unsupported type: {type(value)}")

@process.register(int)
def _(value):
    return f"Integer: {value}"

@process.register(str)
def _(value):
    return f"String: {value}"

@process.register(list)
def _(value):
    return f"List with {len(value)} items"

# ─── total_ordering ───
# 只定义 __eq__ 和 __lt__，自动生成其他比较方法
```

### 16.4 heapq（堆）

```python
import heapq

# 最小堆（无最大堆，用负号反转）
heap = [3, 1, 4, 1, 5]
heapq.heapify(heap)     # 原地转为堆 O(n)
heapq.heappush(heap, 0) # 加入
heapq.heappop(heap)     # 弹出最小值

# 找最大/最小的 N 个
heapq.nlargest(3, [1, 3, 5, 7, 9, 2, 4])  # [9, 7, 5]
heapq.nsmallest(3, [1, 3, 5, 7, 9, 2, 4]) # [1, 2, 3]

# 优先队列
pq = []
heapq.heappush(pq, (1, "low priority"))
heapq.heappush(pq, (0, "high priority"))
heapq.heappop(pq)  # (0, "high priority")
```

### 16.5 bisect（二分查找）

```python
import bisect

# 在有序列表中查找插入位置
arr = [1, 3, 5, 7, 9]
bisect.bisect_left(arr, 5)    # 2 —— 插在左边
bisect.bisect_right(arr, 5)   # 3 —— 插在右边

# 插入并保持有序
bisect.insort(arr, 6)         # [1, 3, 5, 6, 7, 9]

# 用 bisect 实现二分查找
def binary_search(arr, target):
    i = bisect.bisect_left(arr, target)
    if i != len(arr) and arr[i] == target:
        return i
    return -1
```

---

## 17. 文件 IO 与序列化

### 17.1 文件操作

```python
from pathlib import Path

# ─── pathlib（现代方式，3.4+） ───
# 比 os.path 好用得多

p = Path("/Users/sheltonwan/Documents/report.txt")

# 路径信息
p.name          # 'report.txt'
p.stem          # 'report'
p.suffix        # '.txt'
p.parent        # Path('/Users/sheltonwan/Documents')
p.parts         # ('/', 'Users', 'sheltonwan', 'Documents', 'report.txt')

# 路径操作
config = Path.home() / ".config" / "myapp"   # / 运算符拼接！
config.mkdir(parents=True, exist_ok=True)

# 遍历
for f in Path("src").rglob("*.py"):          # 递归通配
    print(f)

# 读写（快捷方式）
text = Path("file.txt").read_text(encoding="utf-8")
Path("file.txt").write_text("hello", encoding="utf-8")
data = Path("file.bin").read_bytes()
Path("output.bin").write_bytes(data)

# 判断
p.exists(), p.is_file(), p.is_dir(), p.is_symlink()

# ─── 传统 open ───
with open("file.txt", "r", encoding="utf-8") as f:
    for line in f:             # 逐行迭代（省内存）
        process(line)

with open("output.txt", "w", encoding="utf-8") as f:
    f.write("hello\n")
```

### 17.2 JSON

```python
import json

# 序列化
data = {"name": "Alice", "age": 30, "tags": ["dev", "python"]}
json_str = json.dumps(data, indent=2, ensure_ascii=False)

# 反序列化
obj = json.loads(json_str)

# 自定义编码器
from datetime import datetime

class DateTimeEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        return super().default(obj)

json.dumps({"time": datetime.now()}, cls=DateTimeEncoder)

# JSON ↔ dataclass
from dataclasses import dataclass, asdict
@dataclass
class User:
    name: str
    age: int

user = User("Bob", 25)
json.dumps(asdict(user))
```

### 17.3 pickle / cloudpickle

```python
import pickle

# pickle 序列化任意 Python 对象（仅限 Python！）
# 安全性：不要 unpickle 不可信数据！

data = {"func": lambda x: x + 1, "set": {1, 2, 3}}
serialized = pickle.dumps(data)
restored = pickle.loads(serialized)

# cloudpickle（支持更多类型：lambda、内部函数等）
# pip install cloudpickle
import cloudpickle
```

### 17.4 CSV / Parquet / Arrow

```python
import csv

# 读 CSV
with open("data.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row["name"], row["age"])

# 写 CSV
with open("out.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["name", "age"])
    writer.writeheader()
    writer.writerows([{"name": "Alice", "age": 30}])

# Parquet（高效列式存储）
# pip install pyarrow
import pyarrow.parquet as pq
table = pq.read_table("data.parquet")
pq.write_table(table, "output.parquet", compression="snappy")
```

---

## 18. 正则与文本处理

### 18.1 re 模块

```python
import re

# ─── 匹配 ───
re.search(r"\d+", "abc123def")    # 第一个匹配 (Match object)
re.match(r"\d+", "abc123")        # 从开头匹配（None）
re.fullmatch(r"\d+", "123")       # 整个字符串匹配

# ─── 查找所有 ───
re.findall(r"\d+", "a1b2c3")     # ['1', '2', '3']
re.finditer(r"\d+", "a1b2c3")    # 迭代器（省内存）

# ─── 替换 ───
re.sub(r"\d+", "#", "a1b2c3")    # 'a#b#c#'
# 使用函数替换
re.sub(r"\d+", lambda m: str(int(m.group()) * 2), "a1b2")  # 'a2b4'

# ─── 分割 ───
re.split(r"\s+", "a  b   c")     # ['a', 'b', 'c']

# ─── 编译（重复使用时提高性能） ───
pattern = re.compile(r"(?P<name>\w+)@(?P<domain>\w+\.\w+)")
match = pattern.search("Email: alice@example.com")
match.group("name")    # 'alice'
match.group("domain")  # 'example.com'
match.groupdict()      # {'name': 'alice', 'domain': 'example.com'}

# ─── 常用标志 ───
re.IGNORECASE  # 忽略大小写
re.MULTILINE   # ^$ 匹配每行首尾
re.DOTALL      # . 匹配包括换行符
re.VERBOSE     # 允许注释和空白
```

---

## 19. 日期时间与日志

### 19.1 datetime

```python
from datetime import datetime, date, time, timedelta, timezone

# ─── 基本操作 ───
now = datetime.now()
today = date.today()
dt = datetime(2024, 1, 15, 14, 30, 0)

# ─── 字符串 ↔ datetime ───
dt_str = dt.strftime("%Y-%m-%d %H:%M:%S")   # '2024-01-15 14:30:00'
dt = datetime.strptime("2024-01-15", "%Y-%m-%d")

# ─── ISO 格式 ───
now.isoformat()              # '2024-01-15T14:30:00.123456'
datetime.fromisoformat("2024-01-15T14:30:00")

# ─── 时区（3.9+ zoneinfo） ───
from zoneinfo import ZoneInfo
ny_time = datetime.now(ZoneInfo("America/New_York"))
tokyo_time = datetime.now(ZoneInfo("Asia/Tokyo"))

# ─── timedelta ───
delta = timedelta(days=1, hours=2, minutes=30)
tomorrow = now + delta
diff = datetime(2024, 1, 1) - datetime(2023, 1, 1)  # timedelta(days=365)
diff.days, diff.total_seconds()

# ─── Unix 时间戳 ───
now.timestamp()                          # float
datetime.fromtimestamp(1700000000.0)

# ─── 第三方：pendulum（更人性化的时间库） ───
# pip install pendulum
# import pendulum
# now = pendulum.now()
# now.diff_for_humans()  # "1 hour ago"
```

### 19.2 logging

```python
import logging

# ─── 基础配置 ───
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("app.log"),
    ],
)

logger = logging.getLogger(__name__)

# ─── 日志级别（递增） ───
# DEBUG → INFO → WARNING → ERROR → CRITICAL
logger.debug("调试信息")
logger.info("一般信息")
logger.warning("警告")
logger.error("错误")
logger.exception("异常（自动附带 traceback）")

# ─── 结构化日志（JSON） ───
# pip install structlog
# import structlog
# logger = structlog.get_logger()
# logger.info("user_login", user_id=123, ip="1.2.3.4")
```

---

## 20. 项目架构与包管理

### 20.1 现代项目结构

```
myproject/
├── pyproject.toml          # 项目元数据（PEP 621）
├── src/
│   └── mypackage/
│       ├── __init__.py     # 导出公共 API
│       ├── __main__.py     # python -m mypackage 入口
│       ├── core.py
│       ├── services/
│       │   ├── __init__.py
│       │   └── auth.py
│       └── utils.py
├── tests/
│   ├── __init__.py         # 空文件
│   ├── conftest.py         # pytest fixtures
│   ├── test_core.py
│   └── services/
│       └── test_auth.py
├── scripts/                # 开发脚本
├── docs/
├── Dockerfile
├── .env.example
├── .gitignore
└── README.md
```

### 20.2 pyproject.toml

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "My awesome project"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.100",
    "sqlalchemy>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7",
    "ruff>=0.1",
    "mypy>=1",
]
test = [
    "pytest-cov",
    "httpx",
]

[tool.ruff]
line-length = 100
select = ["E", "F", "I", "N", "W"]

[tool.mypy]
python_version = "3.11"
strict = true

[tool.pytest.ini_options]
testpaths = ["tests"]
pythonpath = ["src"]
```

### 20.3 包管理工具对比

| 工具 | 速度 | 锁定文件 | 特点 |
|------|------|---------|------|
| pip | 慢 | requirements.txt | 标准，无依赖解析 |
| uv | 极快 (Rust) | requirements.txt / uv.lock | 兼容 pip 工作流 |
| poetry | 中等 | poetry.lock | 完整项目管理 |
| PDM | 快 | pdm.lock | PEP 621 标准，类 npm |
| pipenv | 慢 | Pipfile.lock | 历史原因居多 |

推荐：**uv**（日常快速）+ **poetry**（发布包）。

### 20.4 `__init__.py` 设计

```python
# src/mypackage/__init__.py
"""
MyPackage - Public API
"""

# 控制导出
__all__ = ["UserService", "Config", "create_app"]

# 懒加载（推荐，大型包避免启动过慢）
def __getattr__(name):
    if name == "UserService":
        from .services.auth import UserService
        return UserService
    raise AttributeError(f"module {__name__!r} has no attribute {name!r}")

# 版本
from importlib.metadata import version
__version__ = version("mypackage")
```

---

## 21. 测试策略与 CI/CD

### 21.1 pytest 全面指南

```python
# conftest.py —— 共享 fixtures
import pytest

@pytest.fixture
def db_session():
    """创建测试数据库会话，测试后回滚"""
    session = create_session()
    yield session
    session.rollback()
    session.close()

@pytest.fixture(params=["sqlite", "postgresql"])
def db_backend(request):
    return request.param

# test_user.py
import pytest

def test_create_user(db_session):
    user = db_session.create(User(name="Alice"))
    assert user.name == "Alice"

# ─── 参数化测试 ───
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(a, b, expected):
    assert add(a, b) == expected

# ─── 异常测试 ───
def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError, match="division by zero"):
        1 / 0

# ─── Mock ───
from unittest.mock import Mock, patch, AsyncMock

def test_service_with_mock():
    mock_db = Mock()
    mock_db.query.return_value = [User("Alice")]

    service = UserService(mock_db)
    assert len(service.list_users()) == 1
    mock_db.query.assert_called_once()

# ─── 快照测试 ───
# pip install syrupy
# def test_output(snapshot):
#     result = render_template("email.html", user=user)
#     assert result == snapshot()
```

### 21.2 覆盖率与 CI

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install uv && uv pip install -e ".[dev]"
      - run: ruff check .
      - run: mypy src/
      - run: pytest --cov=src --cov-report=xml
```

```bash
# 本地运行
pytest -xvs                      # 详细 + 首错停止
pytest --lf                      # 只运行上次失败的
pytest -k "test_user"            # 按关键字筛选
pytest --cov=src --cov-report=html  # 覆盖率 HTML 报告
```

---

## 22. 设计模式 Python 版

### 22.1 单例的 N 种写法

```python
# 1. 模块单例（最 Pythonic）
# config.py
class Config:
    def __init__(self):
        self.settings = {}

config = Config()  # 模块级实例 = 天然单例

# 2. 装饰器单例
def singleton(cls):
    instances = {}
    def get(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get

# 3. 元类单例
class SingletonMeta(type):
    _instances = {}
    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

### 22.2 工厂模式

```python
# 用字典调度代替 switch/if-else
class PDFExporter:
    def export(self, data): ...

class CSVExporter:
    def export(self, data): ...

class HTMLExporter:
    def export(self, data): ...

exporters = {
    "pdf": PDFExporter,
    "csv": CSVExporter,
    "html": HTMLExporter,
}

def export_data(fmt: str, data):
    exporter_cls = exporters.get(fmt)
    if not exporter_cls:
        raise ValueError(f"Unknown format: {fmt}")
    return exporter_cls().export(data)
```

### 22.3 策略模式

```python
# Python 中策略模式 = 传入函数/可调用对象
# 不需要像 C++ 那样定义 Strategy 基类和子类

def sort_by(items, key_func):
    return sorted(items, key=key_func)

# 策略即函数
by_name = lambda x: x.name
by_age = lambda x: x.age

sort_by(users, by_name)
sort_by(users, by_age)
```

### 22.4 观察者模式

```python
from typing import Callable

class EventEmitter:
    def __init__(self):
        self._listeners: dict[str, list[Callable]] = {}

    def on(self, event: str, callback: Callable):
        self._listeners.setdefault(event, []).append(callback)

    def emit(self, event: str, *args, **kwargs):
        for cb in self._listeners.get(event, []):
            cb(*args, **kwargs)

# 使用
emitter = EventEmitter()
emitter.on("data", lambda data: print(f"Got: {data}"))
emitter.emit("data", "hello")
```

### 22.5 建造者模式

```python
# Python 不需要经典的 Builder 模式
# 用可选参数 / dataclass / fluent interface 即可

@dataclass
class Request:
    url: str
    method: str = "GET"
    headers: dict = field(default_factory=dict)
    timeout: float = 30.0

    def with_header(self, key, value):
        self.headers[key] = value
        return self

    def with_timeout(self, seconds):
        self.timeout = seconds
        return self

req = Request("https://api.example.com") \
    .with_header("Authorization", "Bearer ...") \
    .with_timeout(60)
```

---

## 23. Web 开发全栈

### 23.1 FastAPI（推荐）

```python
# pip install fastapi uvicorn
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from typing import Annotated

app = FastAPI(title="My API", version="1.0.0")

# ─── 数据模型（Pydantic v2） ───
class UserCreate(BaseModel):
    name: str = Field(min_length=2, max_length=50)
    email: str = Field(pattern=r"[^@]+@[^@]+\.[^@]+")
    age: int = Field(ge=0, le=150)

class UserResponse(BaseModel):
    id: int
    name: str
    email: str

# ─── 路由 ───
@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: int):
    user = await db.get_user(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.post("/users", response_model=UserResponse, status_code=201)
async def create_user(user: UserCreate):
    return await db.create_user(user)

# ─── 依赖注入 ───
async def get_db():
    db = Database()
    try:
        yield db
    finally:
        await db.close()

@app.get("/items")
async def list_items(db: Annotated[Database, Depends(get_db)]):
    return await db.fetch_all()

# 运行：uvicorn main:app --reload
```

### 23.2 Flask（轻量级）

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

@app.route("/hello/<name>")
def hello(name):
    return jsonify({"message": f"Hello, {name}!"})

@app.route("/data", methods=["POST"])
def receive_data():
    data = request.get_json()
    return jsonify({"received": data}), 201
```

### 23.3 HTTP 客户端

```python
# httpx —— 支持 async/await 的现代 HTTP 客户端
# pip install httpx
import httpx

# 同步
with httpx.Client(base_url="https://api.example.com") as client:
    r = client.get("/users/1")
    print(r.json())

# 异步
async with httpx.AsyncClient() as client:
    tasks = [client.get(f"https://api.example.com/users/{i}") for i in range(10)]
    responses = await asyncio.gather(*tasks)
```

---

## 24. 数据库与 ORM

### 24.1 SQLAlchemy 2.0

```python
# pip install sqlalchemy
from sqlalchemy import create_engine, select, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, Session
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession

# ─── 模型定义（2.0 风格） ───
class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = "users"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column()
    email: Mapped[str] = mapped_column(unique=True)
    age: Mapped[int | None] = mapped_column(default=None)

# ─── 同步引擎 ───
engine = create_engine("sqlite:///app.db", echo=True)
Base.metadata.create_all(engine)

# ─── 查询（2.0 风格用 select()） ───
with Session(engine) as session:
    # INSERT
    user = User(name="Alice", email="alice@example.com")
    session.add(user)
    session.commit()

    # SELECT
    stmt = select(User).where(User.name == "Alice")
    result = session.execute(stmt)
    users = result.scalars().all()

    # UPDATE
    stmt = select(User).where(User.id == 1)
    user = session.execute(stmt).scalar_one()
    user.age = 31
    session.commit()

# ─── 异步引擎 ───
async_engine = create_async_engine("sqlite+aiosqlite:///app.db")

async with AsyncSession(async_engine) as session:
    stmt = select(User).where(User.age > 18)
    result = await session.execute(stmt)
    users = result.scalars().all()
```

### 24.2 原始 SQL

```python
import sqlite3

# 同步
conn = sqlite3.connect("app.db")
conn.row_factory = sqlite3.Row  # 按列名访问
rows = conn.execute("SELECT * FROM users WHERE age > ?", (18,)).fetchall()
for row in rows:
    print(row["name"])
conn.close()

# 异步（aiosqlite）
import aiosqlite

async with aiosqlite.connect("app.db") as db:
    db.row_factory = aiosqlite.Row
    async with db.execute("SELECT * FROM users") as cursor:
        async for row in cursor:
            print(row["name"])
```

---

## 25. 对象模型

### 25.1 CPython 对象结构

```c
// CPython 中每个对象都以 PyObject 开头
typedef struct _object {
    Py_ssize_t ob_refcnt;       // 引用计数
    PyTypeObject *ob_type;      // 指向类型对象
} PyObject;

// 整数（PyLongObject）
// 浮点数（PyFloatObject）
// 列表（PyListObject）—— 底层是 PyObject* 数组
// 字典（PyDictObject）—— 开放寻址哈希表
```

```python
# Python 层面观察
import sys

x = 42
print(sys.getsizeof(x))         # 28 bytes（64-bit）
print(sys.getsizeof(42))        # 同上，小整数被 intern

# 小整数缓存（-5 ~ 256）
a = 256
b = 256
print(a is b)  # True —— 同一对象！

a = 257
b = 257
print(a is b)  # False —— 不同对象！

# 字符串 intern
a = "hello"
b = "hello"
print(a is b)  # True —— 自动 intern
```

### 25.2 属性查找链路

```python
# obj.attr 的查找顺序：
# 1. type(obj).__mro__[0].__dict__ 中的 数据描述符
# 2. obj.__dict__
# 3. type(obj).__mro__[0].__dict__ 中的 非数据描述符
# 4. type(obj).__mro__[1:].__dict__ 中的 数据描述符
# 5. ... 沿 MRO 继续
# 6. __getattr__ 回退

# __getattribute__ 是每次属性访问都调用的入口
class LogAccess:
    def __getattribute__(self, name):
        print(f"Accessing: {name}")
        return super().__getattribute__(name)
```

---

## 26. 字节码与解释器

### 26.1 查看字节码

```python
import dis

def add(a, b):
    return a + b

dis.dis(add)
#   2           0 LOAD_FAST                0 (a)
#               2 LOAD_FAST                1 (b)
#               4 BINARY_OP                0 (+)
#               8 RETURN_VALUE

# ─── 字节码指令类别 ───
# LOAD_FAST / STORE_FAST  : 局部变量（数组索引，快）
# LOAD_GLOBAL             : 全局变量（字典查找，慢）
# LOAD_ATTR               : 属性访问
# LOAD_CONST              : 常量
# BINARY_OP / BINARY_SUBSCR
# CALL / CALL_FUNCTION_EX
# JUMP_ABSOLUTE / POP_JUMP_IF_FALSE (循环/条件)

# ─── Python 3.11+ 的新字节码 ───
# 3.11 引入了自适应指令和零开销异常处理
# 3.12 进一步优化了推导式和生成器
# 3.13 实验性 JIT 编译器（copy-and-patch）
```

### 26.2 compile() 与 exec()

```python
# 动态编译和执行
code = compile("x = 1 + 2", "<string>", "exec")
exec(code)
print(x)  # 3

# eval: 表达式求值（不能有语句）
result = eval("2 + 3 * 4")
print(result)  # 14

# 安全性警告：永远不要 eval/exec 用户输入！
```

---

## 27. GIL 的过去与未来

### 27.1 GIL 为什么存在

1. **引用计数不是线程安全的**：每次赋值都涉及 `Py_INCREF`/`Py_DECREF`，没有 GIL 需要原子操作（性能损失）
2. **C 扩展生态假设有 GIL**：数十年的 C 扩展不是线程安全的
3. **移除 GIL 的历史尝试**：多次尝试（free-threading 等）都因性能退化或兼容性放弃

### 27.2 Python 3.13 的 free-threaded 模式

```bash
# Python 3.13+ 可以编译为无 GIL 模式（实验性）
# 使用 --disable-gil 编译 CPython
# 或安装 python3.13t（free-threaded 版本）

# 此时多线程真正并行，但：
# - 引用计数改为偏向引用计数（性能有影响）
# - 部分 C 扩展可能不兼容
# - 单线程性能约 30-40% 退化（当前状态）
```

### 27.3 当前最佳实践

```
IO 密集型    → asyncio / threading（使用 async 库如 aiohttp）
CPU 密集型   → multiprocessing / ProcessPoolExecutor
混合场景     → 主进程 + asyncio + 子进程池
需要高性能   → Cython / Numba / Rust（PyO3）扩展
分布式       → Celery / Ray / Dask
```

---

## 28. Import 系统

### 28.1 import 机制

```python
# import 执行以下步骤：
# 1. 检查 sys.modules 是否已导入（缓存）
# 2. 未缓存：查找模块（sys.path）
# 3. 创建模块对象
# 4. 执行模块代码
# 5. 存入 sys.modules

import sys
print(sys.path)        # 模块搜索路径
print(sys.modules)     # 已导入模块字典

# ─── 相对导入 ───
# 仅包内有效
from . import sibling        # 同包
from .sibling import func    # 同包某模块
from .. import parent        # 父包
from ..parent.module import Class
```

### 28.2 动态导入

```python
# importlib
import importlib

module = importlib.import_module("json")  # 等价于 import json

# 按路径导入
spec = importlib.util.spec_from_file_location("my_module", "/path/to/file.py")
module = importlib.util.module_from_spec(spec)
spec.loader.exec_module(module)

# ─── importlib.metadata（3.8+） ───
from importlib.metadata import version, packages_distributions
print(version("requests"))
```

### 28.3 自定义导入器

```python
# 可以实现从数据库、网络、ZIP 导入模块
# 使用 importlib.abc 中的 Loader 协议

import sys
from importlib.abc import Loader, MetaPathFinder

class MyFinder(MetaPathFinder):
    def find_spec(self, fullname, path, target=None):
        # 自定义查找逻辑
        pass

# 注册到 sys.meta_path
# sys.meta_path.insert(0, MyFinder())
```

---

## A. 快速参考卡片

### 常用内置函数

```python
# 类型相关
type(x)            isinstance(x, T)   issubclass(A, B)
# 数学
abs(x)             round(x, n)        pow(x, y)         divmod(a, b)
# 序列
len(x)             range(n)           enumerate(seq)    zip(*seqs)
map(fn, seq)       filter(fn, seq)    sorted(seq, key=) reversed(seq)
# 字符串
repr(x)            str(x)             format(x, spec)   ord(c) / chr(i)
# 迭代
iter(x)            next(it)           any(seq) / all(seq)
# I/O
open(path)         print(*args)       input(prompt)
# 属性
getattr(obj, n)    setattr(obj,n,v)   hasattr(obj, n)   delattr(obj,n)
dir(obj)           vars(obj)          help(obj)
# 代码
compile(s,f,m)     exec(code)         eval(expr)
# 调试
breakpoint()       __import__(name)   globals() / locals()
```

### 常用 pip 命令

```bash
pip install <pkg>              # 安装
pip install -r requirements.txt # 批量安装
pip uninstall <pkg>            # 卸载
pip list                       # 列出已安装
pip show <pkg>                 # 显示详情
pip freeze > requirements.txt  # 导出依赖
pip install -e .               # 开发模式安装
```

### 虚拟环境速查

```bash
python -m venv .venv           # 创建
source .venv/bin/activate      # 激活 (macOS/Linux)
deactivate                     # 退出
uv venv                        # uv 创建
uv pip install <pkg>           # uv 安装
```

---

## B. 版本迁移指南

| 版本 | 关键特性 |
|------|---------|
| 3.6 | f-string, 类型注解, async gen |
| 3.7 | dataclass, breakpoint(), asyncio.run() |
| 3.8 | walrus `:=`, `=` in f-string, positional-only `/` |
| 3.9 | `dict | dict`, `list[str]` 泛型, zoneinfo |
| 3.10 | match/case, `X | Y` 联合类型, `strict` zip |
| 3.11 | ExceptionGroup, 25-60% 提速, TOML stdlib |
| 3.12 | 进一步提升推导式, `@override`, perf 改进 |
| 3.13 | 实验性 no-GIL, 实验性 JIT, `@deprecated` |

---

## C. 学习路线

### 对 C++/ObjC/Swift/Dart 专家的建议

1. **第一周**：通读本指南第 1-6 章，建立核心概念映射
2. **第二周**：阅读第 16-24 章，掌握标准库和工程化
3. **第三周**：深入第 9-11 章（并发/异步）+ 第 12-15 章（元编程）
4. **第四周**：探索第 25-28 章（内部机制），读 CPython 源码
5. **长期**：
   - 用 Python 重写一个你熟悉的中型项目
   - 贡献开源 Python 项目
   - 学习 Python 的科学计算栈（NumPy/SciPy/Pandas）
   - 探索 Python 的 AI/ML 生态（PyTorch/JAX/scikit-learn）

### 关键心态

> Python 不是"去掉类型的 C++"，也不是"更慢的 Dart"。Python 是一门让你用最少代码做最多事情的语言。你的 C++ 经验让你理解底层发生了什么，你的 ObjC/Swift 经验让你习惯 ARC 和协议编程，你的 Dart/Flutter 经验让你熟悉 async/await。Python 把所有这些概念融合在一起，用一种极简的语法呈现。

> `import this` —— 每天读一遍。

---

> **编写日期**：2024-06 · **目标读者**：20 年跨平台资深开发者 · **推荐 Python 版本**：3.12+
