# Rust 生命周期和借用

> **你将学到什么：** Rust 的生命周期系统如何确保引用永远不会悬空——从隐式生命周期到显式注解，再到使大多数代码无需注解的三个省略规则。在进入下一节的智能指针之前，理解这里的生命周期至关重要。

- Rust 强制执行一个可变引用和任意数量的不可变引用的规则
    - 任何引用的生命周期必须至少与该引用所属的生命周期一样长。这些是隐式生命周期，由编译器推断（见 https://doc.rust-lang.org/nomicon/lifetime-elision.html）
```rust
fn borrow_mut(x: &mut u32) {
    *x = 43;
}
fn main() {
    let mut x = 42;
    let y = &mut x;
    borrow_mut(y);
    let _z = &x; // Permitted because the compiler knows y isn't subsequently used
    //println!("{y}"); // Will not compile if this is uncommented
    borrow_mut(&mut x); // Permitted because _z isn't used 
    let z = &x; // Ok -- mutable borrow of x ended after foo() returned
    println!("{z}");
}
```

# Rust 生命周期注解
- 处理多个生命周期时需要显式生命周期注解
    - 生命周期用 `'` 表示，可以是任何标识符（`'a`、`'b`、`'static` 等）
    - 当编译器无法弄清楚引用应该存活多久时，需要你来帮助
- **常见场景**：函数返回引用，但它来自哪个输入？
```rust
#[derive(Debug)]
struct Point {x: u32, y: u32}

// Without lifetime annotation, this won't compile:
// fn left_or_right(pick_left: bool, left: &Point, right: &Point) -> &Point

// With lifetime annotation - all references share the same lifetime 'a
fn left_or_right<'a>(pick_left: bool, left: &'a Point, right: &'a Point) -> &'a Point {
    if pick_left { left } else { right }
}

// More complex: different lifetimes for inputs
fn get_x_coordinate<'a, 'b>(p1: &'a Point, _p2: &'b Point) -> &'a u32 {
    &p1.x  // Return value lifetime tied to p1, not p2
}

fn main() {
    let p1 = Point {x: 20, y: 30};
    let result;
    {
        let p2 = Point {x: 42, y: 50};
        result = left_or_right(true, &p1, &p2);
        // This works because we use result before p2 goes out of scope
        println!("Selected: {result:?}");
    }
    // This would NOT work - result references p2 which is now gone:
    // println!("After scope: {result:?}");
}
```

# Rust 生命周期注解
- 数据结构中的引用也需要生命周期注解
```rust
use std::collections::HashMap;
#[derive(Debug)]
struct Point {x: u32, y: u32}
struct Lookup<'a> {
    map: HashMap<u32, &'a Point>,
}
fn main() {
    let p = Point{x: 42, y: 42};
    let p1 = Point{x: 50, y: 60};
    let mut m = Lookup {map : HashMap::new()};
    m.map.insert(0, &p);
    m.map.insert(1, &p1);
    {
        let p3 = Point{x: 60, y:70};
        //m.map.insert(3, &p3); // Will not compile
        // p3 is dropped here, but m will outlive
    }
    for (k, v) in m.map {
        println!("{v:?}");
    }
    // m is dropped here
    // p1 and p are dropped here in that order
} 
```

# 练习：带生命周期的第一个单词

🟢 **入门级** — 练习生命周期省略的实际应用

编写一个函数 `fn first_word(s: &str) -> &str`，返回字符串中第一个由空白分隔的单词。思考为什么这可以在没有显式生命周期注解的情况下编译（提示：省略规则 #1 和 #2）。

<details><summary>答案（点击展开）</summary>

```rust
fn first_word(s: &str) -> &str {
    // The compiler applies elision rules:
    // Rule 1: input &str gets lifetime 'a → fn first_word(s: &'a str) -> &str
    // Rule 2: single input lifetime → output gets same → fn first_word(s: &'a str) -> &'a str
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}

fn main() {
    let text = "hello world foo";
    let word = first_word(text);
    println!("First word: {word}");  // "hello"
    
    let single = "onlyone";
    println!("First word: {}", first_word(single));  // "onlyone"
}
```

</details>

# 练习：带生命周期的切片存储

🟡 **中级** — 你第一次遇到生命周期注解
- 创建一个存储对 `&str` 切片引用的结构体
    - 创建一个长的 `&str`，并从中存储引用切片到结构体内部
    - 编写一个接受该结构体并返回包含的切片的函数
```rust
// TODO: Create a structure to store a reference to a slice
struct SliceStore {

}
fn main() {
    let s = "This is long string";
    let s1 = &s[0..];
    let s2 = &s[1..2];
    // let slice = struct SliceStore {...};
    // let slice2 = struct SliceStore {...};
}
```

<details><summary>答案（点击展开）</summary>

```rust
struct SliceStore<'a> {
    slice: &'a str,
}

impl<'a> SliceStore<'a> {
    fn new(slice: &'a str) -> Self {
        SliceStore { slice }
    }

    fn get_slice(&self) -> &'a str {
        self.slice
    }
}

fn main() {
    let s = "This is a long string";
    let store1 = SliceStore::new(&s[0..4]);   // "This"
    let store2 = SliceStore::new(&s[5..7]);   // "is"
    println!("store1: {}", store1.get_slice());
    println!("store2: {}", store2.get_slice());
}
// Output:
// store1: This
// store2: is
```

</details>

---

## 生命周期省略规则深度探讨

C 程序员经常问："如果生命周期如此重要，为什么大多数 Rust 函数没有 `'a` 注解？" 答案是**生命周期省略**——编译器应用三个确定性规则来自动推断生命周期。

### 三个省略规则

Rust 编译器按**顺序**将这些规则应用于函数签名。如果在应用规则后所有输出生命周期都被确定，则不需要注解。

```mermaid
flowchart TD
    A["Function signature with references"] --> R1
    R1["Rule 1: Each input reference<br/>gets its own lifetime<br/><br/>fn f(&str, &str)<br/>→ fn f<'a,'b>(&'a str, &'b str)"]
    R1 --> R2
    R2["Rule 2: If exactly ONE input<br/>lifetime, assign it to ALL outputs<br/><br/>fn f(&str) → &str<br/>→ fn f<'a>(&'a str) → &'a str"]
    R2 --> R3
    R3["Rule 3: If one input is &self<br/>or &mut self, assign its lifetime<br/>to ALL outputs<br/><br/>fn f(&self, &str) → &str<br/>→ fn f<'a>(&'a self, &str) → &'a str"]
    R3 --> CHECK{{"All output lifetimes<br/>determined?"}}
    CHECK -->|"Yes"| OK["✅ No annotations needed"]
    CHECK -->|"No"| ERR["❌ Compile error:<br/>must annotate manually"]
    
    style OK fill:#91e5a3,color:#000
    style ERR fill:#ff6b6b,color:#000
```

### 逐规则详解

**规则 1** — 每个输入引用获得自己的生命周期参数：
```rust
// 你写的代码:
fn first_word(s: &str) -> &str { ... }

// 编译器应用规则 1 后看到的:
fn first_word<'a>(s: &'a str) -> &str { ... }
// 只有一个输入生命周期 → 规则 2 适用
```

**规则 2** — 单个输入生命周期传播到所有输出：
```rust
// 应用规则 2 后:
fn first_word<'a>(s: &'a str) -> &'a str { ... }
// ✅ 所有输出生命周期都已确定 — 不需要注解！
```

**规则 3** — `&self` 生命周期传播到输出：
```rust
// 你写的代码:
impl SliceStore<'_> {
    fn get_slice(&self) -> &str { self.slice }
}

// 编译器应用规则 1 + 3 后看到的:
impl SliceStore<'_> {
    fn get_slice<'a>(&'a self) -> &'a str { self.slice }
}
// ✅ 不需要注解 — &self 的生命周期用于输出
```

**当省略失败时** — 必须添加注解：
```rust
// Two input references, no &self → Rules 2 and 3 don't apply
// fn longest(a: &str, b: &str) -> &str  ← WON'T COMPILE

// Fix: tell the compiler which input the output borrows from
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

### C 程序员的思维模型

在 C 中，每个指针都是独立的——程序员在脑子里追踪每个指针指向哪个分配，编译器完全信任你。在 Rust 中，生命周期使这种追踪变得**显式且经过编译器验证**：

| C | Rust | 发生了什么 |
|---|------|-------------|
| `char* get_name(struct User* u)` | `fn get_name(&self) -> &str` | 规则 3 省略：输出借用了 `self` |
| `char* concat(char* a, char* b)` | `fn concat<'a>(a: &'a str, b: &'a str) -> &'a str` | 必须注解 — 两个输入 |
| `void process(char* in, char* out)` | `fn process(input: &str, output: &mut String)` | 没有输出引用 — 不需要生命周期 |
| `char* buf; /* 谁拥有这个？ */` | 如果生命周期错误则编译错误 | 编译器捕获悬空指针 |

### `'static` 生命周期

`'static` 意味着引用在**整个程序运行期间**有效。它是 C 全局变量或字符串字面量的 Rust 等价物：

```rust
// 字符串字面量总是 'static — 它们存在于二进制文件的只读段中
let s: &'static str = "hello";  // 等同于 C 中的: static const char* s = "hello";

// 常量也是 'static
static GREETING: &str = "hello";

// 在线程生成的 trait bound 中很常见:
fn spawn<F: FnOnce() + Send + 'static>(f: F) { /* ... */ }
// 这里的 'static 意味着: "闭包不能借用任何局部变量"
// (要么将它们移入，要么只使用 'static 数据)
```

### 练习：预测省略

🟡 **中级**

对于下面的每个函数签名，预测编译器是否可以省略生命周期。
如果没有，添加必要的注解：

```rust
// 1. 编译器可以省略吗？
fn trim_prefix(s: &str) -> &str { &s[1..] }

// 2. 编译器可以省略吗？
fn pick(flag: bool, a: &str, b: &str) -> &str {
    if flag { a } else { b }
}

// 3. 编译器可以省略吗？
struct Parser { data: String }
impl Parser {
    fn next_token(&self) -> &str { &self.data[..5] }
}

// 4. 编译器可以省略吗？
fn split_at(s: &str, pos: usize) -> (&str, &str) {
    (&s[..pos], &s[pos..])
}
```

<details><summary>答案（点击展开）</summary>

```rust,ignore
// 1. 可以 — 规则 1 给 s 分配 'a，规则 2 传播到输出
fn trim_prefix(s: &str) -> &str { &s[1..] }

// 2. 不可以 — 两个输入引用，没有 &self。必须注解:
fn pick<'a>(flag: bool, a: &'a str, b: &'a str) -> &'a str {
    if flag { a } else { b }
}

// 3. 可以 — 规则 1 给 &self 分配 'a，规则 3 传播到输出
impl Parser {
    fn next_token(&self) -> &str { &self.data[..5] }
}

// 4. 可以 — 规则 1 给 s 分配 'a（只有一个输入引用），
//    规则 2 传播到两个输出。两个切片都借用了 s。
fn split_at(s: &str, pos: usize) -> (&str, &str) {
    (&s[..pos], &s[pos..])
}
```

</details>
