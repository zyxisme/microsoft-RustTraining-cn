## 元组与解构

> **你将学到：** Rust 元组 vs Python 元组、数组和切片、结构体（Rust 替代类的方式）、
> `Vec<T>` vs `list`、`HashMap<K,V>` vs `dict`，以及用于领域建模的 newtype 模式。
>
> **难度：** 🟢 初级

### Python 元组
```python
# Python — 元组是不可变的序列
point = (3.0, 4.0)
x, y = point                    # 解包
print(f"x={x}, y={y}")

# 元组可以存储混合类型
record = ("Alice", 30, True)
name, age, active = record

# 命名元组增加可读性
from typing import NamedTuple

class Point(NamedTuple):
    x: float
    y: float

p = Point(3.0, 4.0)
print(p.x)                      # 按名称访问
```

### Rust 元组
```rust
// Rust — 元组是固定大小的，有类型，可以存储混合类型
let point: (f64, f64) = (3.0, 4.0);
let (x, y) = point;              // 解构（与 Python 解包相同）
println!("x={x}, y={y}");

// 混合类型
let record: (&str, i32, bool) = ("Alice", 30, true);
let (name, age, active) = record;

// 按索引访问（与 Python 不同，使用 .0 .1 .2 语法）
let first = record.0;            // "Alice"
let second = record.1;           // 30

// Python: record[0]
// Rust:   record.0      ← 点索引，不是括号索引
```

### 何时使用元组 vs 结构体
```rust
// 元组：快速组合、函数返回、临时值
fn min_max(data: &[i32]) -> (i32, i32) {
    (*data.iter().min().unwrap(), *data.iter().max().unwrap())
}
let (lo, hi) = min_max(&[3, 1, 4, 1, 5]);

// 结构体：命名字段、意图清晰、需要方法
struct Point { x: f64, y: f64 }

// 经验法则：
// - 2-3 个同类型字段 → 元组即可
// - 需要命名字段  → 使用结构体
// - 需要方法       → 使用结构体
// （与 Python 指导原则相同：tuple vs namedtuple vs dataclass）
```

***

## 数组和切片

### Python 列表 vs Rust 数组
```python
# Python — 列表是动态的、异构的
numbers = [1, 2, 3, 4, 5]       # 可以增长、缩短、存储混合类型
numbers.append(6)
mixed = [1, "two", 3.0]         # 允许混合类型
```

```rust
// Rust 有两种固定大小 vs 动态的概念：

// 1. 数组 — 固定大小，栈分配（没有 Python 等价物）
let numbers: [i32; 5] = [1, 2, 3, 4, 5]; // 大小是类型的一部分！
// numbers.push(6);  // ❌ 数组不能增长

// 初始化所有元素为相同值：
let zeros = [0; 10];            // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

// 2. 切片 — 数组或 Vec 的视图（类似 Python 切片，但是被借用的）
let slice: &[i32] = &numbers[1..4]; // [2, 3, 4] — 是引用，不是拷贝！

// Python: numbers[1:4] 创建一个新列表（拷贝）
// Rust:   &numbers[1..4] 创建一个视图（无拷贝、无分配）
```

### 实用对比
```python
# Python 切片 — 创建拷贝
data = [10, 20, 30, 40, 50]
first_three = data[:3]          # 新列表：[10, 20, 30]
last_two = data[-2:]            # 新列表：[40, 50]
reversed_data = data[::-1]      # 新列表：[50, 40, 30, 20, 10]
```

```rust
// Rust 切片 — 创建视图（引用）
let data = [10, 20, 30, 40, 50];
let first_three = &data[..3];         // &[i32]，视图：[10, 20, 30]
let last_two = &data[3..];            // &[i32]，视图：[40, 50]

// 不支持负索引 — 使用 .len()
let last_two = &data[data.len()-2..]; // &[i32]，视图：[40, 50]

// 反转：使用迭代器
let reversed: Vec<i32> = data.iter().rev().copied().collect();
```

***

## 结构体 vs 类

### Python 类
```python
# Python — 带 __init__、方法、属性的类
from dataclasses import dataclass

@dataclass
class Rectangle:
    width: float
    height: float

    def area(self) -> float:
        return self.width * self.height

    def perimeter(self) -> float:
        return 2.0 * (self.width + self.height)

    def scale(self, factor: float) -> "Rectangle":
        return Rectangle(self.width * factor, self.height * factor)

    def __str__(self) -> str:
        return f"Rectangle({self.width} x {self.height})"

r = Rectangle(10.0, 5.0)
print(r.area())         # 50.0
print(r)                # Rectangle(10.0 x 5.0)
```

### Rust 结构体
```rust
// Rust — 结构体 + impl 块（无继承！）
#[derive(Debug, Clone)]
struct Rectangle {
    width: f64,
    height: f64,
}

impl Rectangle {
    // "构造函数" — 关联函数（无 self）
    fn new(width: f64, height: f64) -> Self {
        Rectangle { width, height }   // 字段简写（当名称匹配时）
    }

    fn area(&self) -> f64 {
        self.width * self.height
    }

    fn perimeter(&self) -> f64 {
        2.0 * (self.width + self.height)
    }

    fn scale(&self, factor: f64) -> Rectangle {
        Rectangle::new(self.width * factor, self.height * factor)
    }
}

// Display trait = Python 的 __str__
impl std::fmt::Display for Rectangle {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Rectangle({} x {})", self.width, self.height)
    }
}

fn main() {
    let r = Rectangle::new(10.0, 5.0);
    println!("{}", r.area());    // 50.0
    println!("{}", r);           // Rectangle(10 x 5)
}
```

```mermaid
flowchart LR
    subgraph Python ["Python 对象（堆）"]
        PH["PyObject 头\n(refcount + type ptr)"] --> PW["width: float 对象"]
        PH --> PHT["height: float 对象"]
        PH --> PD["__dict__"]
    end
    subgraph Rust ["Rust 结构体（栈）"]
        RW["width: f64\n(8 字节)"] --- RH["height: f64\n(8 字节)"]
    end
    style Python fill:#ffeeba
    style Rust fill:#d4edda
```

> **内存洞察**：Python 的 `Rectangle` 对象有 56 字节头 + 独立堆分配的 float 对象。Rust 的 `Rectangle` 在栈上正好是 16 字节 — 无间接引用，无 GC 压力。
>
> 📌 **另见**：[第 10 章 — Trait 和泛型](ch10-traits-and-generics.md) 涵盖了为结构体实现 trait，如 `Display`、`Debug` 和运算符重载。

### 关键映射：Python 魔术方法 → Rust Trait

| Python | Rust | 用途 |
|--------|------|------|
| `__str__` | `impl Display` | 人类可读字符串 |
| `__repr__` | `#[derive(Debug)]` | 调试表示 |
| `__eq__` | `#[derive(PartialEq)]` | 相等比较 |
| `__hash__` | `#[derive(Hash)]` | 可哈希（用于 dict 键 / HashSet） |
| `__lt__`、`__le__` 等 | `#[derive(PartialOrd, Ord)]` | 排序 |
| `__add__` | `impl Add` | `+` 运算符 |
| `__iter__` | `impl Iterator` | 迭代 |
| `__len__` | `.len()` 方法 | 长度 |
| `__enter__`/`__exit__` | `impl Drop` | 清理（Rust 中自动执行） |
| `__init__` | `fn new()`（约定） | 构造函数 |
| `__getitem__` | `impl Index` | 使用 `[]` 索引 |
| `__contains__` | `.contains()` 方法 | `in` 运算符 |

### 无继承 — 组合优先
```python
# Python — 继承
class Animal:
    def __init__(self, name: str):
        self.name = name
    def speak(self) -> str:
        raise NotImplementedError

class Dog(Animal):
    def speak(self) -> str:
        return f"{self.name} says Woof!"

class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} says Meow!"
```

```rust
// Rust — trait + 组合（无继承）
trait Animal {
    fn name(&self) -> &str;
    fn speak(&self) -> String;
}

struct Dog { name: String }
struct Cat { name: String }

impl Animal for Dog {
    fn name(&self) -> &str { &self.name }
    fn speak(&self) -> String {
        format!("{} says Woof!", self.name)
    }
}

impl Animal for Cat {
    fn name(&self) -> &str { &self.name }
    fn speak(&self) -> String {
        format!("{} says Meow!", self.name)
    }
}

// 使用 trait 对象实现多态（类似 Python 的鸭子类型）：
fn animal_roll_call(animals: &[&dyn Animal]) {
    for a in animals {
        println!("{}", a.speak());
    }
}
```

> **心智模型**：Python 说"继承行为"。Rust 说"实现契约"。
> 结果相似，但 Rust 避免了菱形继承问题和脆弱基类问题。

***

## Vec vs list

`Vec<T>` 是 Rust 的可增长、堆分配的数组 — 最接近 Python `list` 的等价物。

### 创建向量
```python
# Python
numbers = [1, 2, 3]
empty = []
repeated = [0] * 10
from_range = list(range(1, 6))
```

```rust
// Rust
let numbers = vec![1, 2, 3];            // vec! 宏（像列表字面量）
let empty: Vec<i32> = Vec::new();        // 空 vec（需要类型注解）
let repeated = vec![0; 10];              // [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
let from_range: Vec<i32> = (1..6).collect(); // [1, 2, 3, 4, 5]
```

### 常见操作
```python
# Python 列表操作
nums = [1, 2, 3]
nums.append(4)                   # [1, 2, 3, 4]
nums.extend([5, 6])             # [1, 2, 3, 4, 5, 6]
nums.insert(0, 0)               # [0, 1, 2, 3, 4, 5, 6]
last = nums.pop()               # 6, nums = [0, 1, 2, 3, 4, 5]
length = len(nums)              # 6
nums.sort()                     # 原地排序
sorted_copy = sorted(nums)     # 新排序列表
nums.reverse()                  # 原地反转
contains = 3 in nums           # True
index = nums.index(3)          # 第一个 3 的索引
```

```rust
// Rust Vec 操作
let mut nums = vec![1, 2, 3];
nums.push(4);                          // [1, 2, 3, 4]
nums.extend([5, 6]);                   // [1, 2, 3, 4, 5, 6]
nums.insert(0, 0);                     // [0, 1, 2, 3, 4, 5, 6]
let last = nums.pop();                 // Some(6), nums = [0, 1, 2, 3, 4, 5]
let length = nums.len();               // 6
nums.sort();                           // 原地排序
let mut sorted_copy = nums.clone();
sorted_copy.sort();                    // 对克隆排序
nums.reverse();                        // 原地反转
let contains = nums.contains(&3);      // true
let index = nums.iter().position(|&x| x == 3); // Some(index) 或 None
```

### 快速参考

| Python | Rust | 备注 |
|--------|------|------|
| `lst.append(x)` | `vec.push(x)` | |
| `lst.extend(other)` | `vec.extend(other)` | |
| `lst.pop()` | `vec.pop()` | 返回 `Option<T>` |
| `lst.insert(i, x)` | `vec.insert(i, x)` | |
| `lst.remove(x)` | `vec.retain(\|v\| v != &x)` | |
| `del lst[i]` | `vec.remove(i)` | 返回被移除的元素 |
| `len(lst)` | `vec.len()` | |
| `x in lst` | `vec.contains(&x)` | |
| `lst.sort()` | `vec.sort()` | |
| `sorted(lst)` | 克隆 + 排序，或迭代器 | |
| `lst[i]` | `vec[i]` | 越界时 panic |
| `lst.get(i, default)` | `vec.get(i)` | 返回 `Option<&T>` |
| `lst[1:3]` | `&vec[1..3]` | 返回切片（无拷贝） |

***

## HashMap vs dict

`HashMap<K, V>` 是 Rust 的哈希映射 — 等价于 Python 的 `dict`。

### 创建 HashMap
```python
# Python
scores = {"Alice": 100, "Bob": 85}
empty = {}
from_pairs = dict([("x", 1), ("y", 2)])
comprehension = {k: v for k, v in zip(keys, values)}
```

```rust
// Rust
use std::collections::HashMap;

let scores = HashMap::from([("Alice", 100), ("Bob", 85)]);
let empty: HashMap<String, i32> = HashMap::new();
let from_pairs: HashMap<&str, i32> = [("x", 1), ("y", 2)].into_iter().collect();
let comprehension: HashMap<_, _> = keys.iter().zip(values.iter()).collect();
```

### 常见操作
```python
# Python dict 操作
d = {"a": 1, "b": 2}
d["c"] = 3                      # 插入
val = d["a"]                     # 1（缺失时 KeyError）
val = d.get("z", 0)             # 0（缺失时默认值）
del d["b"]                       # 删除
exists = "a" in d               # True
keys = list(d.keys())           # ["a", "c"]
values = list(d.values())       # [1, 3]
items = list(d.items())         # [("a", 1), ("c", 3)]
length = len(d)                 # 2

# setdefault / defaultdict
from collections import defaultdict
word_count = defaultdict(int)
for word in words:
    word_count[word] += 1
```

```rust
// Rust HashMap 操作
use std::collections::HashMap;

let mut d = HashMap::new();
d.insert("a", 1);
d.insert("b", 2);
d.insert("c", 3);                       // 插入或覆盖

let val = d["a"];                        // 1（缺失时 panic）
let val = d.get("z").copied().unwrap_or(0); // 0（安全访问）
d.remove("b");                          // 删除
let exists = d.contains_key("a");       // true
let keys: Vec<_> = d.keys().collect();
let values: Vec<_> = d.values().collect();
let length = d.len();

// entry API = Python 的 setdefault / defaultdict 模式
let mut word_count: HashMap<&str, i32> = HashMap::new();
for word in words {
    *word_count.entry(word).or_insert(0) += 1;
}
```

### 快速参考

| Python | Rust | 备注 |
|--------|------|------|
| `d[key] = val` | `d.insert(key, val)` | 返回 `Option<V>`（旧值） |
| `d[key]` | `d[&key]` | 缺失时 panic |
| `d.get(key)` | `d.get(&key)` | 返回 `Option<&V>` |
| `d.get(key, default)` | `d.get(&key).unwrap_or(&default)` | |
| `key in d` | `d.contains_key(&key)` | |
| `del d[key]` | `d.remove(&key)` | 返回 `Option<V>` |
| `d.keys()` | `d.keys()` | 迭代器 |
| `d.values()` | `d.values()` | 迭代器 |
| `d.items()` | `d.iter()` | `(&K, &V)` 的迭代器 |
| `len(d)` | `d.len()` | |
| `d.update(other)` | `d.extend(other)` | |
| `defaultdict(int)` | `.entry().or_insert(0)` | Entry API |
| `d.setdefault(k, v)` | `d.entry(k).or_insert(v)` | Entry API |

***

### 其他集合

| Python | Rust | 备注 |
|--------|------|------|
| `set()` | `HashSet<T>` | `use std::collections::HashSet;` |
| `collections.deque` | `VecDeque<T>` | `use std::collections::VecDeque;` |
| `heapq` | `BinaryHeap<T>` | 默认最大堆 |
| `collections.OrderedDict` | `IndexMap`（crate） | HashMap 不保证顺序 |
| `sortedcontainers.SortedList` | `BTreeSet<T>` / `BTreeMap<K,V>` | 基于树、有序 |

---

## 练习

<details>
<summary><strong>🏋️ 练习：单词频率计数器</strong>（点击展开）</summary>

**挑战**：编写一个函数，接收 `&str` 句子并返回 `HashMap<String, usize>` 类型的单词频率（不区分大小写）。Python 中是 `Counter(s.lower().split())`。请翻译成 Rust。

<details>
<summary>🔑 解答</summary>

```rust
use std::collections::HashMap;

fn word_frequencies(text: &str) -> HashMap<String, usize> {
    let mut counts = HashMap::new();
    for word in text.split_whitespace() {
        let key = word.to_lowercase();
        *counts.entry(key).or_insert(0) += 1;
    }
    counts
}

fn main() {
    let text = "the quick brown fox jumps over the lazy fox";
    let freq = word_frequencies(text);
    for (word, count) in &freq {
        println!("{word}: {count}");
    }
}
```

**关键收获**：`HashMap::entry().or_insert()` 是 Rust 相当于 Python 的 `defaultdict` 或 `Counter`。需要解引用 `*`，因为 `or_insert` 返回 `&mut usize`。

</details>
</details>

***

