## Rust 闭包 vs Python Lambda

> **你将学到：** 多行闭包（不仅仅是一表达式 lambda），`Fn`/`FnMut`/`FnOnce` 捕获语义，
> 迭代器链 vs 列表推导式，`map`/`filter`/`fold`，以及 `macro_rules!` 基础。
>
> **难度：** 🟡 中级

### Python 闭包和 Lambda
```python
# Python — lambdas 是单表达式匿名函数
double = lambda x: x * 2
result = double(5)  # 10

# 完整闭包从封闭作用域捕获变量：
def make_adder(n):
    def adder(x):
        return x + n    # 从外部作用域捕获 `n`
    return adder

add_5 = make_adder(5)
print(add_5(10))  # 15

# 高阶函数：
numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))
evens = list(filter(lambda x: x % 2 == 0, numbers))
```

### Rust 闭包
```rust
// Rust — 闭包使用 |args| body 语法
let double = |x: i32| x * 2;
let result = double(5);  // 10

// 闭包从封闭作用域捕获变量：
fn make_adder(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x + n    // `move` 将 `n` 的所有权转移到闭包中
}

let add_5 = make_adder(5);
println!("{}", add_5(10));  // 15

// 使用迭代器的高阶函数：
let numbers = vec![1, 2, 3, 4, 5];
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
let evens: Vec<i32> = numbers.iter().filter(|&&x| x % 2 == 0).copied().collect();
```

### 闭包语法对比
```text
Python:                              Rust:
─────────                            ─────
lambda x: x * 2                      |x| x * 2
lambda x, y: x + y                   |x, y| x + y
lambda: 42                           || 42

# Multi-line
def f(x):                            |x| {
    y = x * 2                            let y = x * 2;
    return y + 1                         y + 1
                                      }
```

### 闭包捕获 — Rust 的不同之处
```python
# Python — 闭包通过引用捕获（延迟绑定！）
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])  # [2, 2, 2] — 出乎意料！所有闭包捕获了同一个 `i`

# 用默认参数技巧修复：
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])  # [0, 1, 2]
```

```rust
// Rust — 闭包正确捕获（没有延迟绑定的坑）
let funcs: Vec<Box<dyn Fn() -> i32>> = (0..3)
    .map(|i| Box::new(move || i) as Box<dyn Fn() -> i32>)
    .collect();

let results: Vec<i32> = funcs.iter().map(|f| f()).collect();
println!("{:?}", results);  // [0, 1, 2] — 正确！

// `move` 为每个闭包复制一份 `i` — 没有延迟绑定的意外。
```

### 三种闭包 Trait
```rust
// Rust 闭包实现以下一个或多个 trait：

// Fn — 可以多次调用，不改变捕获的变量（最常见）
fn apply(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(x) }

// FnMut — 可以多次调用，可能改变捕获的变量
fn apply_mut(mut f: impl FnMut(i32) -> i32, x: i32) -> i32 { f(x) }

// FnOnce — 只能调用一次（消耗捕获的变量）
fn apply_once(f: impl FnOnce() -> String) -> String { f() }

// Python 没有等价物 — 闭包总是 Fn 风格的。
// 在 Rust 中，编译器自动决定使用哪个 trait。
```

***

## 迭代器 vs 生成器

### Python 生成器
```python
# Python — 使用 yield 的生成器
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# 惰性求值 — 值按需计算
fib = fibonacci()
first_10 = [next(fib) for _ in range(10)]

# 生成器表达式 — 类似于惰性列表推导式
squares = (x ** 2 for x in range(1000000))  # 不分配内存
first_5 = [next(squares) for _ in range(5)]
```

### Rust 迭代器
```rust
// Rust — Iterator trait（类似概念，不同语法）
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { a: 0, b: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        let current = self.a;
        self.a = self.b;
        self.b = current + self.b;
        Some(current)
    }
}

// 惰性求值 — 值按需计算（与 Python 生成器相同）
let first_10: Vec<u64> = Fibonacci::new().take(10).collect();

// 迭代器链 — 类似于生成器表达式
let squares: Vec<u64> = (0..1_000_000u64).map(|x| x * x).take(5).collect();
```

***

## 推导式 vs 迭代器链

本节将 Python 的推导式语法映射到 Rust 的迭代器链。

### 列表推导式 → map/filter/collect
```python
# Python 推导式：
squares = [x ** 2 for x in range(10)]
evens = [x for x in range(20) if x % 2 == 0]
names = [user.name for user in users if user.active]
pairs = [(x, y) for x in range(3) for y in range(3)]
flat = [item for sublist in nested for item in sublist]
```

```mermaid
flowchart LR
    A["Source\n[1,2,3,4,5]"] -->|.iter()| B["Iterator"]
    B -->|.filter(|x| x%2==0)| C["[2, 4]"]
    C -->|.map(|x| x*x)| D["[4, 16]"]
    D -->|.collect()| E["Vec<i32>\n[4, 16]"]
    style A fill:#ffeeba
    style E fill:#d4edda
```

> **关键洞察**：Rust 迭代器是惰性的 — 在调用 `.collect()` 之前什么都不发生。Python 的生成器工作方式类似，但列表推导式是立即求值的。

```rust
// Rust 迭代器链：
let squares: Vec<i32> = (0..10).map(|x| x * x).collect();
let evens: Vec<i32> = (0..20).filter(|x| x % 2 == 0).collect();
let names: Vec<&str> = users.iter()
    .filter(|u| u.active)
    .map(|u| u.name.as_str())
    .collect();
let pairs: Vec<(i32, i32)> = (0..3)
    .flat_map(|x| (0..3).map(move |y| (x, y)))
    .collect();
let flat: Vec<i32> = nested.iter()
    .flat_map(|sublist| sublist.iter().copied())
    .collect();
```

### 字典推导式 → collect 成 HashMap
```python
# Python
word_lengths = {word: len(word) for word in words}
inverted = {v: k for k, v in mapping.items()}
```

```rust
// Rust
let word_lengths: HashMap<&str, usize> = words.iter()
    .map(|w| (*w, w.len()))
    .collect();
let inverted: HashMap<&V, &K> = mapping.iter()
    .map(|(k, v)| (v, k))
    .collect();
```

### 集合推导式 → collect 成 HashSet
```python
# Python
unique_lengths = {len(word) for word in words}
```

```rust
// Rust
let unique_lengths: HashSet<usize> = words.iter()
    .map(|w| w.len())
    .collect();
```

### 常用迭代器方法

| Python | Rust | 说明 |
|--------|------|------|
| `map(f, iter)` | `.map(f)` | 转换每个元素 |
| `filter(f, iter)` | `.filter(f)` | 保留匹配的元素 |
| `sum(iter)` | `.sum()` | 求和所有元素 |
| `min(iter)` / `max(iter)` | `.min()` / `.max()` | 返回 `Option` |
| `any(f(x) for x in iter)` | `.any(f)` | 任一匹配则为真 |
| `all(f(x) for x in iter)` | `.all(f)` | 全部匹配则为真 |
| `enumerate(iter)` | `.enumerate()` | 索引 + 值 |
| `zip(a, b)` | `a.zip(b)` | 配对元素 |
| `len(list)` | `.count()` (消耗!) 或 `.len()` | 计数元素 |
| `list(reversed(x))` | `.rev()` | 反向迭代 |
| `itertools.chain(a, b)` | `a.chain(b)` | 连接迭代器 |
| `next(iter)` | `.next()` | 获取下一个元素 |
| `next(iter, default)` | `.next().unwrap_or(default)` | 带默认值 |
| `list(iter)` | `.collect::<Vec<_>>()` | 物化为集合 |
| `sorted(iter)` | 收集后再 `.sort()` | 没有惰性排序迭代器 |
| `functools.reduce(f, iter)` | `.fold(init, f)` 或 `.reduce(f)` | 累积 |

### 关键区别
```text
Python 迭代器：                     Rust 迭代器：
─────────────────                  ──────────────
- 默认惰性（生成器）                 - 默认惰性（所有迭代器链）
- yield 创建生成器                  - impl Iterator { fn next() }
- StopIteration 结束               - None 结束
- 只能消耗一次                      - 只能消耗一次
- 无类型安全                        - 完全类型安全
- 稍慢（解释器）                     - 零成本（编译时消除）
```

***

<!-- ch12a: Macros -->

## 为什么 Rust 中存在宏

Python 没有宏系统 — 它使用装饰器、元类和运行时内省进行元编程。Rust 使用宏进行编译时代码生成。

### Python 元编程 vs Rust 宏
```python
# Python — 用于元编程的装饰器和元类
from dataclasses import dataclass
from functools import wraps

@dataclass              # 在导入时生成 __init__、__repr__、__eq__
class Point:
    x: float
    y: float

# 自定义装饰器
def log_calls(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@log_calls
def process(data):
    return data.upper()
```

```rust
// Rust — 用于代码生成的 derive 宏和声明式宏
#[derive(Debug, Clone, PartialEq)]  // 在编译时生成 Debug、Clone、PartialEq impl
struct Point {
    x: f64,
    y: f64,
}

// 声明式宏（类似于模板）
macro_rules! log_call {
    ($func_name:expr, $body:expr) => {
        println!("Calling {}", $func_name);
        $body
    };
}

fn process(data: &str) -> String {
    log_call!("process", data.to_uppercase())
}
```

### 常用内置宏
```rust
// 这些宏在 Rust 中到处使用：

println!("Hello, {}!", name);           // 带格式化的打印
format!("Value: {}", x);               // 创建格式化的 String
vec![1, 2, 3];                          // 创建 Vec
assert_eq!(2 + 2, 4);                  // 测试断言
assert!(value > 0, "must be positive"); // 布尔断言
dbg!(expression);                       // 调试打印：打印表达式和值
todo!();                                // 占位符 — 编译通过但到达时 panic
unimplemented!();                       // 标记代码未实现
panic!("something went wrong");         // 带消息的崩溃（类似于 raise RuntimeError）

// 为什么这些是宏而不是函数？
// - println! 接受可变参数（Rust 函数不能）
// - vec! 为任何类型和大小生成代码
// - assert_eq! 知道你比较的源代码
// - dbg! 知道文件名和行号
```

## 使用 macro_rules! 编写简单宏
```rust
// Python dict() 等价物
// Python: d = dict(a=1, b=2)
// Rust:   let d = hashmap!{ "a" => 1, "b" => 2 };

macro_rules! hashmap {
    ($($key:expr => $value:expr),* $(,)?) => {
        {
            let mut map = std::collections::HashMap::new();
            $(map.insert($key, $value);)*
            map
        }
    };
}

let scores = hashmap! {
    "Alice" => 100,
    "Bob" => 85,
    "Charlie" => 90,
};
```

## Derive 宏 — 自动实现 Trait
```rust
// #[derive(...)] 是 Rust 中 Python 的 @dataclass 装饰器的等价物

// Python:
// @dataclass(frozen=True, order=True)
// class Student:
//     name: str
//     grade: int

// Rust:
#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord, Hash)]
struct Student {
    name: String,
    grade: i32,
}

// 常用的 derive 宏：
// Debug         → {:?} 格式化（类似于 __repr__）
// Clone         → .clone() 深拷贝
// Copy          → 隐式拷贝（仅用于简单类型）
// PartialEq, Eq → == 比较（类似于 __eq__）
// PartialOrd, Ord → <, >, 排序（类似于 __lt__ 等）
// Hash          → 可用作 HashMap 键（类似于 __hash__）
// Default       → MyType::default()（类似于无参数的 __init__）

// crate 提供的 derive 宏：
// Serialize, Deserialize (serde) → JSON/YAML/TOML 序列化
//                                  （类似于 Python 的 json.dumps/loads 但类型安全）
```

### Python 装饰器 vs Rust Derive

| Python 装饰器 | Rust Derive | 用途 |
|----------------|-------------|------|
| `@dataclass` | `#[derive(Debug, Clone, PartialEq)]` | 数据类 |
| `@dataclass(frozen=True)` | 默认不可变 | 不可变性 |
| `@dataclass(order=True)` | `#[derive(Ord, PartialOrd)]` | 比较/排序 |
| `@total_ordering` | `#[derive(PartialOrd, Ord)]` | 完全排序 |
| JSON `json.dumps(obj.__dict__)` | `#[derive(Serialize)]` | 序列化 |
| JSON `MyClass(**json.loads(s))` | `#[derive(Deserialize)]` | 反序列化 |

---

## 练习

<details>
<summary><strong>🏋️ 练习：Derive 和自定义 Debug</strong>（点击展开）</summary>

**挑战**：创建一个 `User` 结构体，包含字段 `name: String`、`email: String` 和 `password_hash: String`。派生 `Clone` 和 `PartialEq`，但手动实现 `Debug` 以便打印姓名和电子邮件但隐藏密码（显示 `"***"` 而不是实际值）。

<details>
<summary>🔑 解决方案</summary>

```rust
use std::fmt;

#[derive(Clone, PartialEq)]
struct User {
    name: String,
    email: String,
    password_hash: String,
}

impl fmt::Debug for User {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("User")
            .field("name", &self.name)
            .field("email", &self.email)
            .field("password_hash", &"***")
            .finish()
    }
}

fn main() {
    let user = User {
        name: "Alice".into(),
        email: "alice@example.com".into(),
        password_hash: "a1b2c3d4e5f6".into(),
    };
    println!("{user:?}");
    // 输出：User { name: "Alice", email: "alice@example.com", password_hash: "***" }
}
```

**关键要点**：与 Python 的 `__repr__` 不同，Rust 可以免费派生 `Debug` — 但你可以为敏感字段覆盖它。这比 Python 更安全，因为在 Python 中 `print(user)` 可能意外泄露敏感信息。

</details>
</details>

***


