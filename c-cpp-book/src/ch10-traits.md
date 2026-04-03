# Rust traits

> **你将学到什么：** Traits——Rust 对接口、抽象基类和运算符重载的回答。你将学习如何定义 traits、为你的类型实现它们，以及使用动态分发（`dyn Trait`）vs 静态分发（泛型）。对于 C++ 开发者：traits 替代虚函数、CRTP 和 concepts。对于 C 开发者：traits 是 Rust 实现多态的结构化方式。

- Rust traits 类似于其他语言中的接口
    - Traits 定义了必须由实现该 trait 的类型定义的方法。
```rust
fn main() {
    trait Pet {
        fn speak(&self);
    }
    struct Cat;
    struct Dog;
    impl Pet for Cat {
        fn speak(&self) {
            println!("Meow");
        }
    }
    impl Pet for Dog {
        fn speak(&self) {
            println!("Woof!")
        }
    }
    let c = Cat{};
    let d = Dog{};
    c.speak();  // There is no "is a" relationship between Cat and Dog
    d.speak(); // There is no "is a" relationship between Cat and Dog
}
```

## Traits vs C++ Concepts 和接口

### 传统 C++ 继承 vs Rust Traits

```cpp
// C++ - Inheritance-based polymorphism
class Animal {
public:
    virtual void speak() = 0;  // Pure virtual function
    virtual ~Animal() = default;
};

class Cat : public Animal {  // "Cat IS-A Animal"
public:
    void speak() override {
        std::cout << "Meow" << std::endl;
    }
};

void make_sound(Animal* animal) {  // Runtime polymorphism
    animal->speak();  // Virtual function call
}
```

```rust
// Rust - Composition over inheritance with traits
trait Animal {
    fn speak(&self);
}

struct Cat;  // Cat is NOT an Animal, but IMPLEMENTS Animal behavior

impl Animal for Cat {  // "Cat CAN-DO Animal behavior"
    fn speak(&self) {
        println!("Meow");
    }
}

fn make_sound<T: Animal>(animal: &T) {  // Static polymorphism
    animal.speak();  // Direct function call (zero cost)
}
```

```mermaid
graph TD
    subgraph "C++ Object-Oriented Hierarchy"
        CPP_ANIMAL["Animal<br/>(Abstract base class)"]
        CPP_CAT["Cat : public Animal<br/>(IS-A relationship)"]
        CPP_DOG["Dog : public Animal<br/>(IS-A relationship)"]
        
        CPP_ANIMAL --> CPP_CAT
        CPP_ANIMAL --> CPP_DOG
        
        CPP_VTABLE["Virtual function table<br/>(Runtime dispatch)"]
        CPP_HEAP["Often requires<br/>heap allocation"]
        CPP_ISSUES["[ERROR] Deep inheritance trees<br/>[ERROR] Diamond problem<br/>[ERROR] Runtime overhead<br/>[ERROR] Tight coupling"]
    end
    
    subgraph "Rust Trait-Based Composition"
        RUST_TRAIT["trait Animal<br/>(Behavior definition)"]
        RUST_CAT["struct Cat<br/>(Data only)"]
        RUST_DOG["struct Dog<br/>(Data only)"]
        
        RUST_CAT -.->|"impl Animal for Cat<br/>(CAN-DO behavior)"| RUST_TRAIT
        RUST_DOG -.->|"impl Animal for Dog<br/>(CAN-DO behavior)"| RUST_TRAIT
        
        RUST_STATIC["Static dispatch<br/>(Compile-time)"]
        RUST_STACK["Stack allocation<br/>possible"]
        RUST_BENEFITS["[OK] No inheritance hierarchy<br/>[OK] Multiple trait impls<br/>[OK] Zero runtime cost<br/>[OK] Loose coupling"]
    end
    
    style CPP_ISSUES fill:#ff6b6b,color:#000
    style RUST_BENEFITS fill:#91e5a3,color:#000
    style CPP_VTABLE fill:#ffa07a,color:#000
    style RUST_STATIC fill:#91e5a3,color:#000
```

### Trait Bounds and Generic Constraints

```rust
use std::fmt::Display;
use std::ops::Add;

// C++ template equivalent (less constrained)
// template<typename T>
// T add_and_print(T a, T b) {
//     // No guarantee T supports + or printing
//     return a + b;  // Might fail at compile time
// }

// Rust - explicit trait bounds
fn add_and_print<T>(a: T, b: T) -> T 
where 
    T: Display + Add<Output = T> + Copy,
{
    println!("Adding {} + {}", a, b);  // Display trait
    a + b  // Add trait
}
```

```mermaid
graph TD
    subgraph "Generic Constraints Evolution"
        UNCONSTRAINED["fn process<T>(data: T)<br/>[ERROR] T can be anything"]
        SINGLE_BOUND["fn process<T: Display>(data: T)<br/>[OK] T must implement Display"]
        MULTI_BOUND["fn process<T>(data: T)<br/>where T: Display + Clone + Debug<br/>[OK] Multiple requirements"]
        
        UNCONSTRAINED --> SINGLE_BOUND
        SINGLE_BOUND --> MULTI_BOUND
    end
    
    subgraph "Trait Bound Syntax"
        INLINE["fn func<T: Trait>(param: T)"]
        WHERE_CLAUSE["fn func<T>(param: T)<br/>where T: Trait"]
        IMPL_PARAM["fn func(param: impl Trait)"]
        
        COMPARISON["Inline: Simple cases<br/>Where: Complex bounds<br/>impl: Concise syntax"]
    end
    
    subgraph "Compile-time Magic"
        GENERIC_FUNC["Generic function<br/>with trait bounds"]
        TYPE_CHECK["Compiler verifies<br/>trait implementations"]
        MONOMORPH["Monomorphization<br/>(Create specialized versions)"]
        OPTIMIZED["Fully optimized<br/>machine code"]
        
        GENERIC_FUNC --> TYPE_CHECK
        TYPE_CHECK --> MONOMORPH
        MONOMORPH --> OPTIMIZED
        
        EXAMPLE["add_and_print::<i32><br/>add_and_print::<f64><br/>(Separate functions generated)"]
        MONOMORPH --> EXAMPLE
    end
    
    style UNCONSTRAINED fill:#ff6b6b,color:#000
    style SINGLE_BOUND fill:#ffa07a,color:#000
    style MULTI_BOUND fill:#91e5a3,color:#000
    style OPTIMIZED fill:#91e5a3,color:#000
```

### C++ Operator Overloading → Rust `std::ops` Traits

在 C++ 中，你可以通过编写带有特殊名称的自由函数或成员函数来重载运算符（`operator+`、`operator<<`、`operator[]` 等）。在 Rust 中，每个运算符都映射到 `std::ops` 中的一个 trait（输出使用 `std::fmt`）。你**实现 trait** 而不是编写具有魔法名称的函数。

#### Side-by-side: `+` operator

```cpp
// C++: operator overloading as a member or free function
struct Vec2 {
    double x, y;
    Vec2 operator+(const Vec2& rhs) const {
        return {x + rhs.x, y + rhs.y};
    }
};

Vec2 a{1.0, 2.0}, b{3.0, 4.0};
Vec2 c = a + b;  // calls a.operator+(b)
```

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Vec2;                     // Associated type — the result of +
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

let a = Vec2 { x: 1.0, y: 2.0 };
let b = Vec2 { x: 3.0, y: 4.0 };
let c = a + b;  // calls <Vec2 as Add>::add(a, b)
println!("{c:?}"); // Vec2 { x: 4.0, y: 6.0 }
```

#### Key differences from C++

| Aspect | C++ | Rust |
|--------|-----|------|
| **Mechanism** | Magic function names (`operator+`) | Implement a trait (`impl Add for T`) |
| **Discovery** | Grep for `operator+` or read the header | Look at trait impls — IDE support excellent |
| **Return type** | Free choice | Fixed by the `Output` associated type |
| **Receiver** | Usually takes `const T&` (borrows) | Takes `self` by value (moves!) by default |
| **Symmetry** | Can write `impl operator+(int, Vec2)` | Must add `impl Add<Vec2> for i32` (foreign trait rules apply) |
| **`<<` for printing** | `operator<<(ostream&, T)` — overload for *any* stream | `impl fmt::Display for T` — one canonical `to_string` representation |

#### The `self` by value gotcha

在 Rust 中，`Add::add(self, rhs)` 按**值**获取 `self`。对于 `Copy` 类型（如上面的 `Vec2`，它派生了 `Copy`）这没问题——编译器会复制。但对于非 `Copy` 类型，`+` **消耗**操作数：

```rust
let s1 = String::from("hello ");
let s2 = String::from("world");
let s3 = s1 + &s2;  // s1 is MOVED into s3!
// println!("{s1}");  // ❌ Compile error: value used after move
println!("{s2}");     // ✅ s2 was only borrowed (&s2)
```

这就是为什么 `String + &str` 有效而 `&str + &str` 无效——`Add` 只为 `String + &str` 实现了，消耗左边的 `String` 以重用其缓冲区。这没有 C++ 类比：`std::string::operator+` 总是创建一个新字符串。

#### Full mapping: C++ operators → Rust traits

| C++ Operator | Rust Trait | Notes |
|-------------|-----------|-------|
| `operator+` | `std::ops::Add` | `Output` associated type |
| `operator-` | `std::ops::Sub` | |
| `operator*` | `std::ops::Mul` | Not pointer deref — that's `Deref` |
| `operator/` | `std::ops::Div` | |
| `operator%` | `std::ops::Rem` | |
| `operator-` (unary) | `std::ops::Neg` | |
| `operator!` / `operator~` | `std::ops::Not` | Rust uses `!` for both logical and bitwise NOT (no `~` operator) |
| `operator&`, `\|`, `^` | `BitAnd`, `BitOr`, `BitXor` | |
| `operator<<`, `>>` (shift) | `Shl`, `Shr` | NOT stream I/O! |
| `operator+=` | `std::ops::AddAssign` | Takes `&mut self` (not `self`) |
| `operator[]` | `std::ops::Index` / `IndexMut` | Returns `&Output` / `&mut Output` |
| `operator()` | `Fn` / `FnMut` / `FnOnce` | Closures implement these; you cannot `impl Fn` directly |
| `operator==` | `PartialEq` (+ `Eq`) | In `std::cmp`, not `std::ops` |
| `operator<` | `PartialOrd` (+ `Ord`) | In `std::cmp` |
| `operator<<` (stream) | `fmt::Display` | `println!("{}", x)` |
| `operator<<` (debug) | `fmt::Debug` | `println!("{:?}", x)` |
| `operator bool` | No direct equivalent | Use `impl From<T> for bool` or a named method like `.is_empty()` |
| `operator T()` (implicit conversion) | No implicit conversions | Use `From`/`Into` traits (explicit) |

#### Guardrails: what Rust prevents

1. **无隐式转换**：C++ `operator int()` 可能导致静默的、令人惊讶的转换。Rust 没有隐式转换操作符——使用 `From`/`Into` 并显式调用 `.into()`。
2. **不能重载 `&&` / `||`**：C++ 允许（但会破坏短路语义！）。Rust 不允许。
3. **不能重载 `=`**：赋值总是移动或复制，从不是用户定义的。复合赋值（`+=`）可以通过 `AddAssign` 等重载。
4. **不能重载 `，`**：C++ 允许 `operator,()` —— 这是 C++ 最臭名昭著的危险特性之一。Rust 不允许。
5. **不能重载 `&`（取地址）**：另一个 C++ 危险特性（存在 `std::addressof` 来绕过它）。Rust 的 `&` 总是表示"借用"。
6. **一致性规则**：你只能为自己的类型实现 `Add<Foreign>`，或为外国类型实现 `Add<YourType>`——绝不能为外国类型实现 `Add<Foreign>`。这防止了跨 crate 的冲突运算符定义。

> **底线**：在 C++ 中，运算符重载很强大但基本上没有规范——你可以重载几乎任何东西，包括逗号和取地址，隐式转换可以静默触发。Rust 通过 trait 为算术和比较操作符提供了同样的表现力，但**阻止了历史上危险的重载**并强制所有转换都是显式的。

----
# Rust traits
- Rust 允许在甚至像 u32 这样的内置类型上实现用户定义的 trait，如本例所示。但是，trait 或类型必须属于 crate 之一
```rust
trait IsSecret {
  fn is_secret(&self);
}
// IsSecret trait 属于 crate，所以没问题
impl IsSecret for u32 {
  fn is_secret(&self) {
      if *self == 42 {
          println!("Is secret of life");
      }
  }
}

fn main() {
  42u32.is_secret();
  43u32.is_secret();
}
```


# Rust traits
- Traits 支持接口继承和默认实现
```rust
trait Animal {
  // Default implementation
  fn is_mammal(&self) -> bool {
    true
  }
}
trait Feline : Animal {
  // Default implementation
  fn is_feline(&self) -> bool {
    true
  }
}

struct Cat;
// 使用默认实现。注意，supertrait 的所有 traits 必须单独实现
impl Feline for Cat {}
impl Animal for Cat {}
fn main() {
  let c = Cat{};
  println!("{} {}", c.is_mammal(), c.is_feline());
}
```
----
# 练习：Logger trait 实现

🟡 **中级**

- 实现一个名为 `Log` 的 trait，它有一个接受 u64 的方法 `log()`
    - 实现两个不同的日志记录器 `SimpleLogger` 和 `ComplexLogger`，它们都实现 `Log` trait。一个应该输出"Simple logger"及其 `u64`，另一个应该输出"Complex logger"及其 `u64` 

<details><summary>Solution (click to expand)</summary>

```rust
trait Log {
    fn log(&self, value: u64);
}

struct SimpleLogger;
struct ComplexLogger;

impl Log for SimpleLogger {
    fn log(&self, value: u64) {
        println!("Simple logger: {value}");
    }
}

impl Log for ComplexLogger {
    fn log(&self, value: u64) {
        println!("Complex logger: {value} (hex: 0x{value:x}, binary: {value:b})");
    }
}

fn main() {
    let simple = SimpleLogger;
    let complex = ComplexLogger;
    simple.log(42);
    complex.log(42);
}
// Output:
// Simple logger: 42
// Complex logger: 42 (hex: 0x2a, binary: 101010)
```

</details>

----
# Rust trait associated types
```rust
#[derive(Debug)]
struct Small(u32);
#[derive(Debug)]
struct Big(u32);
trait Double {
    type T;
    fn double(&self) -> Self::T;
}

impl Double for Small {
    type T = Big;
    fn double(&self) -> Self::T {
        Big(self.0 * 2)
    }
}
fn main() {
    let a = Small(42);
    println!("{:?}", a.double());
}
```

# Rust trait impl
- ```impl``` 可以与 trait 一起使用，以接受任何实现了该 trait 的类型
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &impl Pet) {
    p.speak();
}
fn main() {
    let c = Cat {};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```

# Rust trait impl
- ```impl``` 也可以用于返回值
```rust
trait Pet {}
struct Dog;
struct Cat;
impl Pet for Cat {}
impl Pet for Dog {}
fn cat_as_pet() -> impl Pet {
    let c = Cat {};
    c
}
fn dog_as_pet() -> impl Pet {
    let d = Dog {};
    d
}
fn main() {
    let p = cat_as_pet();
    let d = dog_as_pet();
}
```
----
# Rust dynamic traits
- 动态 trait 可用于调用 trait 功能而无需知道底层类型。这称为```类型擦除``` 
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {x: u32}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &dyn Pet) {
    p.speak();
}
fn main() {
    let c = Cat {x: 42};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```
----

## 选择 `impl Trait`、`dyn Trait` 和 Enums

这三种方法都实现了多态，但有不同的权衡：

| Approach | Dispatch | Performance | Heterogeneous collections? | When to use |
|----------|----------|-------------|---------------------------|-------------|
| `impl Trait` / generics | Static (monomorphized) | Zero-cost — inlined at compile time | No — each slot has one concrete type | Default choice. Function arguments, return types |
| `dyn Trait` | Dynamic (vtable) | Small overhead per call (~1 pointer indirection) | Yes — `Vec<Box<dyn Trait>>` | When you need mixed types in a collection, or plugin-style extensibility |
| `enum` | Match | Zero-cost — known variants at compile time | Yes — but only known variants | When the set of variants is **closed** and known at compile time |

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { radius: f64 }
struct Rect { w: f64, h: f64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius } }
impl Shape for Rect   { fn area(&self) -> f64 { self.w * self.h } }

// Static dispatch — compiler generates separate code for each type
fn print_area(s: &impl Shape) { println!("{}", s.area()); }

// Dynamic dispatch — one function, works with any Shape behind a pointer
fn print_area_dyn(s: &dyn Shape) { println!("{}", s.area()); }

// Enum — closed set, no trait needed
enum ShapeEnum { Circle(f64), Rect(f64, f64) }
impl ShapeEnum {
    fn area(&self) -> f64 {
        match self {
            ShapeEnum::Circle(r) => std::f64::consts::PI * r * r,
            ShapeEnum::Rect(w, h) => w * h,
        }
    }
}
```

> **对于 C++ 开发者：** `impl Trait` 类似于 C++ 模板（单态化、零成本）。`dyn Trait` 类似于 C++ 虚函数（vtable 分发）。带有 `match` 的 Rust 枚举类似于 `std::variant` 与 `std::visit` —— 但穷举匹配由编译器强制执行。

> **经验法则**：从 `impl Trait`（静态分发）开始。只有当你需要异构集合或无法在编译时知道具体类型时才使用 `dyn Trait`。当你拥有所有变体时使用 `enum`。

