# 1. 泛型 —— 全景图 🟢

> **你将学到：**
> - 单态化如何实现零成本泛型 —— 以及何时导致代码膨胀
> - 决策框架：泛型 vs 枚举 vs trait 对象
> - Const 泛型用于编译时数组大小和 `const fn` 用于编译时求值
> - 何时在冷路径上用动态分发换静态分发

## 单态化与零成本

Rust 中的泛型是**单态化**的 —— 编译器为每个使用的具体类型生成一个专用副本。这与 Java/C# 中泛型在运行时被擦除的做法相反。

```rust
fn max_of<T: PartialOrd>(a: T, b: T) -> T {
    if a >= b { a } else { b }
}

fn main() {
    max_of(3_i32, 5_i32);     // 编译器生成 max_of_i32
    max_of(2.0_f64, 7.0_f64); // 编译器生成 max_of_f64
    max_of("a", "z");         // 编译器生成 max_of_str
}
```

**编译器实际产生的内容**（概念上）：

```rust
// 三个独立函数 —— 无运行时分发，无虚表：
fn max_of_i32(a: i32, b: i32) -> i32 { if a >= b { a } else { b } }
fn max_of_f64(a: f64, b: f64) -> f64 { if a >= b { a } else { b } }
fn max_of_str<'a>(a: &'a str, b: &'a str) -> &'a str { if a >= b { a } else { b } }
```

> **为什么 `max_of_str` 需要 `<'a>` 但 `max_of_i32` 不需要？** `i32` 和 `f64`
> 是 `Copy` 类型 —— 函数返回一个拥有的值。但 `&str` 是一个引用，
> 所以编译器必须知道返回引用的生命周期。`<'a>` 注解表示
> "返回的 `&str` 至少与两个输入一样长寿。"

**优势**：零运行时成本 —— 与手写专用代码完全相同。优化器可以独立内联、向量化和专用化每个副本。

**与 C++ 对比**：Rust 泛型的工作方式类似于 C++ 模板，但有一个关键区别 —— **边界检查发生在定义时，而非实例化时**。在 C++ 中，模板只有在用于特定类型时才会编译，导致库代码深处出现神秘的错误信息。在 Rust 中，`T: PartialOrd` 在定义函数时就被检查了，所以错误被及早捕获，消息也很清晰。

```rust
// Rust：定义时的错误 —— "T 没有实现 Display"
fn broken<T>(val: T) {
    println!("{val}"); // ❌ 错误：T 没有实现 Display
}

// 修复：添加约束
fn fixed<T: std::fmt::Display>(val: T) {
    println!("{val}"); // ✅
}
```

### 泛型何时有害：代码膨胀

单态化有代价 —— 二进制大小。每个唯一实例化都会复制函数体：

```rust
// 这个无害的函数...
fn serialize<T: serde::Serialize>(value: &T) -> Vec<u8> {
    serde_json::to_vec(value).unwrap()
}

// ...用于 50 种不同类型 → 二进制中有 50 份副本。
```

**缓解策略**：

```rust
// 1. 提取非泛型核心（"outline" 模式）
fn serialize<T: serde::Serialize>(value: &T) -> Result<Vec<u8>, serde_json::Error> {
    // 泛型部分：只有序列化调用
    let json_value = serde_json::to_value(value)?;
    // 非泛型部分：提取到单独函数
    serialize_value(json_value)
}

fn serialize_value(value: serde_json::Value) -> Result<Vec<u8>, serde_json::Error> {
    // 这个函数在二进制中只存在一次
    serde_json::to_vec(&value)
}

// 2. 当内联不关键时使用 trait 对象（动态分发）
fn log_item(item: &dyn std::fmt::Display) {
    // 一份副本 —— 使用虚表进行分发
    println!("[LOG] {item}");
}
```

> **经验法则**：在需要内联的热点路径使用泛型。
> 在冷路径（错误处理、日志、配置）使用 `dyn Trait`，
> 虚表调用可以忽略不计。

### 泛型 vs 枚举 vs Trait 对象 —— 决策指南

在 Rust 中处理"不同类型，相同接口"的三种方式：

| 方法 | 分发 | 何时确定 | 可扩展？ | 开销 |
|----------|----------|----------|-------------|----------|
| **泛型**（`impl Trait` / `<T: Trait>`） | 静态（单态化） | 编译时 | ✅（开放集合） | 零 —— 内联 |
| **枚举** | Match 分支 | 编译时 | ❌（封闭集合） | 零 —— 无虚表 |
| **Trait 对象**（`dyn Trait`） | 动态（虚表） | 运行时 | ✅（开放集合） | 虚表指针 + 间接调用 |

```rust
// --- 泛型：开放集合，零成本，编译时 ---
fn process<H: Handler>(handler: H, request: Request) -> Response {
    handler.handle(request) // 单态化 —— 每个 H 一份副本
}

// --- 枚举：封闭集合，零成本，穷尽匹配 ---
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Triangle(f64, f64, f64),
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle(r) => std::f64::consts::PI * r * r,
            Shape::Rect(w, h) => w * h,
            Shape::Triangle(a, b, c) => {
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            }
        }
    }
}
// 添加新变体必须更新所有 match 分支 —— 编译器
// 强制穷尽性。适用于"我控制所有变体"的情况。

// --- Trait 对象：开放集合，运行时成本，可扩展 ---
fn log_all(items: &[Box<dyn std::fmt::Display>]) {
    for item in items {
        println!("{item}"); // 虚表分发
    }
}
```

**决策流程图**：

```mermaid
flowchart TD
    A["Do you know ALL<br>possible types at<br>compile time?"]
    A -->|"Yes, small<br>closed set"| B["Enum"]
    A -->|"Yes, but set<br>is open"| C["Generics<br>(monomorphized)"]
    A -->|"No — types<br>determined at runtime"| D["dyn Trait"]

    C --> E{"Hot path?<br>(millions of calls)"}
    E -->|Yes| F["Generics<br>(inlineable)"]
    E -->|No| G["dyn Trait<br>is fine"]

    D --> H{"Need mixed types<br>in one collection?"}
    H -->|Yes| I["Vec&lt;Box&lt;dyn Trait&gt;&gt;"]
    H -->|No| C

    style A fill:#e8f4f8,stroke:#2980b9,color:#000
    style B fill:#d4efdf,stroke:#27ae60,color:#000
    style C fill:#d4efdf,stroke:#27ae60,color:#000
    style D fill:#fdebd0,stroke:#e67e22,color:#000
    style F fill:#d4efdf,stroke:#27ae60,color:#000
    style G fill:#fdebd0,stroke:#e67e22,color:#000
    style I fill:#fdebd0,stroke:#e67e22,color:#000
    style E fill:#fef9e7,stroke:#f1c40f,color:#000
    style H fill:#fef9e7,stroke:#f1c40f,color:#000
```

### Const 泛型

自 Rust 1.51 以来，你可以将类型和函数参数化到**常量值**，而不仅仅是类型：

```rust
// 按大小参数化的数组包装器
struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f64; COLS]; ROWS],
}

impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    fn new() -> Self {
        Matrix { data: [[0.0; COLS]; ROWS] }
    }

    fn transpose(&self) -> Matrix<COLS, ROWS> {
        let mut result = Matrix::<COLS, ROWS>::new();
        for r in 0..ROWS {
            for c in 0..COLS {
                result.data[c][r] = self.data[r][c];
            }
        }
        result
    }
}

// 编译器强制维度正确性：
fn multiply<const M: usize, const N: usize, const P: usize>(
    a: &Matrix<M, N>,
    b: &Matrix<N, P>, // N 必须匹配！
) -> Matrix<M, P> {
    let mut result = Matrix::<M, P>::new();
    for i in 0..M {
        for j in 0..P {
            for k in 0..N {
                result.data[i][j] += a.data[i][k] * b.data[k][j];
            }
        }
    }
    result
}

// 用法：
let a = Matrix::<2, 3>::new(); // 2×3
let b = Matrix::<3, 4>::new(); // 3×4
let c = multiply(&a, &b);      // 2×4 ✅

// let d = Matrix::<5, 5>::new();
// multiply(&a, &d); // ❌ 编译错误：期望 Matrix<3, _>，得到 Matrix<5, 5>
```

> **C++ 对比**：这类似于 C++ 中的 `template<int N>`，但 Rust
> const 泛型会被急切地进行类型检查，不受 SFINAE 复杂性的困扰。

### Const 函数（const fn）

`const fn` 标记一个函数为可在编译时求值 —— Rust 的
C++ `constexpr` 等价物。结果可用于 `const` 和 `static` 上下文：

```rust
// 基本 const fn —— 在 const 上下文中使用时在编译时求值
const fn celsius_to_fahrenheit(c: f64) -> f64 {
    c * 9.0 / 5.0 + 32.0
}

const BOILING_F: f64 = celsius_to_fahrenheit(100.0); // 编译时计算
const FREEZING_F: f64 = celsius_to_fahrenheit(0.0);  // 32.0

// Const 构造函数 —— 创建 static 而无需 lazy_static！
struct BitMask(u32);

impl BitMask {
    const fn new(bit: u32) -> Self {
        BitMask(1 << bit)
    }

    const fn or(self, other: BitMask) -> Self {
        BitMask(self.0 | other.0)
    }

    const fn contains(&self, bit: u32) -> bool {
        self.0 & (1 << bit) != 0
    }
}

// 静态查找表 —— 无运行时成本，无延迟初始化
const GPIO_INPUT:  BitMask = BitMask::new(0);
const GPIO_OUTPUT: BitMask = BitMask::new(1);
const GPIO_IRQ:    BitMask = BitMask::new(2);
const GPIO_IO:     BitMask = GPIO_INPUT.or(GPIO_OUTPUT);

// 寄存器映射作为 const 数组：
const SENSOR_THRESHOLDS: [u16; 4] = {
    let mut table = [0u16; 4];
    table[0] = 50;   // 警告
    table[1] = 70;   // 高
    table[2] = 85;   // 严重
    table[3] = 100;  // 关闭
    table
};
// 整个表存在于二进制中 —— 无堆，无运行时初始化。
```

**你可以在 `const fn` 中做的事**（截至 Rust 1.79+）：
- 算术、位运算、比较
- `if`/`else`、`match`、`loop`、`while`（控制流）
- 创建和修改局部变量（`let mut`）
- 调用其他 `const fn`
- 引用（`&`、`&mut` —— 在 const 上下文中）
- `panic!()`（如果在编译时到达则成为编译错误）

**你不能做的事**（暂时）：
- 堆分配（`Box`、`Vec`、`String`）
- Trait 方法调用（只有固有方法）
- 某些上下文中的浮点数（基本运算已稳定）
- I/O 或副作用

```rust
// 带有 panic 的 const fn —— 在编译时成为错误：
const fn checked_div(a: u32, b: u32) -> u32 {
    if b == 0 {
        panic!("division by zero"); // 如果 b 在 const 时为 0 则编译错误
    }
    a / b
}

const RESULT: u32 = checked_div(100, 4);  // ✅ 25
// const BAD: u32 = checked_div(100, 0);  // ❌ 编译错误："division by zero"
```

> **C++ 对比**：`const fn` 是 Rust 的 `constexpr`。关键区别：
> Rust 版本是可选加入的，编译器严格验证只使用
> const 兼容的操作。在 C++ 中，`constexpr` 函数可能
> 静默回退到运行时求值 —— 在 Rust 中，const 上下文
> *要求*编译时求值，否则就是硬错误。

> **实践建议**：尽可能将构造函数和简单实用函数设为 `const fn`
> —— 这没有成本，并使调用者能在 const 上下文中使用它们。
> 对于硬件诊断代码，`const fn` 非常适合寄存器
> 定义、位掩码构造和阈值表。

> **关键要点 —— 泛型**
> - 单态化提供零成本抽象，但可能导致代码膨胀 —— 在冷路径使用 `dyn Trait`
> - Const 泛型（`[T; N]`）用编译时检查的数组大小取代 C++ 模板技巧
> - `const fn` 消除了对编译时可计算值的 `lazy_static!`

> **另请参阅：**[第 2 章 —— 深入 Trait](ch02-traits-in-depth.md) 了解 trait 约束、关联类型和 trait 对象。[第 4 章 —— PhantomData](ch04-phantomdata-types-that-carry-no-data.md) 了解零大小泛型标记。

---

### 练习：带驱逐策略的泛型缓存 ★★（约 30 分钟）

构建一个泛型 `Cache<K, V>` 结构体，存储键值对并具有可配置的最大容量。满时，最早的条目被驱逐（FIFO）。要求：

- `fn new(capacity: usize) -> Self`
- `fn insert(&mut self, key: K, value: V)` —— 满时驱逐最旧的
- `fn get(&self, key: &K) -> Option<&V>`
- `fn len(&self) -> usize`
- 约束 `K: Eq + Hash + Clone`

<details>
<summary>🔑 解决方案</summary>

```rust
use std::collections::{HashMap, VecDeque};
use std::hash::Hash;

struct Cache<K, V> {
    map: HashMap<K, V>,
    order: VecDeque<K>,
    capacity: usize,
}

impl<K: Eq + Hash + Clone, V> Cache<K, V> {
    fn new(capacity: usize) -> Self {
        Cache {
            map: HashMap::with_capacity(capacity),
            order: VecDeque::with_capacity(capacity),
            capacity,
        }
    }

    fn insert(&mut self, key: K, value: V) {
        if self.map.contains_key(&key) {
            self.map.insert(key, value);
            return;
        }
        if self.map.len() >= self.capacity {
            if let Some(oldest) = self.order.pop_front() {
                self.map.remove(&oldest);
            }
        }
        self.order.push_back(key.clone());
        self.map.insert(key, value);
    }

    fn get(&self, key: &K) -> Option<&V> {
        self.map.get(key)
    }

    fn len(&self) -> usize {
        self.map.len()
    }
}

fn main() {
    let mut cache = Cache::new(3);
    cache.insert("a", 1);
    cache.insert("b", 2);
    cache.insert("c", 3);
    assert_eq!(cache.len(), 3);

    cache.insert("d", 4); // 驱逐 "a"
    assert_eq!(cache.get(&"a"), None);
    assert_eq!(cache.get(&"d"), Some(&4));
    println!("Cache works! len = {}", cache.len());
}
```

</details>

***

