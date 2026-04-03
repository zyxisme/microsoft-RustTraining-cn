### Rust 数组类型

> **你将学到什么：** Rust 的核心数据结构——数组、元组、切片、字符串、结构体、`Vec` 和 `HashMap`。这是内容密集的一章；专注于理解 `String` vs `&str` 以及结构体如何工作。你将在第 7 章深入重温引用和借用。

- 数组包含固定数量的相同类型元素
    - 与所有其他 Rust 类型一样，数组默认是不可变的（除非使用 mut）
    - 数组使用 [] 索引，并有边界检查。可以使用 len() 方法获取数组长度
```rust
    fn get_index(y : usize) -> usize {
        y+1        
    }
    
    fn main() {
        // Initializes an array of 10 elements and sets all to 42
        let a : [u8; 3] = [42; 3];
        // Alternative syntax
        // let a = [42u8, 42u8, 42u8];
        for x in a {
            println!("{x}");
        }
        let y = get_index(a.len());
        // Commenting out the below will cause a panic
        //println!("{}", a[y]);
    }
```

----
### Rust 数组类型（续）
- 数组可以嵌套
    - Rust 有几种内置的打印格式化器。在下面，`:?` 是 `debug` 打印格式化器。`:#?` 格式化器可用于 `pretty print`。这些格式化器可以按类型自定义（更多内容稍后介绍）
```rust
    fn main() {
        let a = [
            [40, 0], // Define a nested array
            [41, 0],
            [42, 1],
        ];
        for x in a {
            println!("{x:?}");
        }
    }
```
----
### Rust 元组
- 元组有固定大小，可以将任意类型分组为单个复合类型
    - 组成类型可以通过其相对位置（.0、.1、.2 ...）索引。空元组，即 ()，称为单元值，相当于 void 返回值
    - Rust 支持元组解构，使绑定变量到各个元素变得容易
```rust
fn get_tuple() -> (u32, bool) {
    (42, true)        
}

fn main() {
   let t : (u8, bool) = (42, true);
   let u : (u32, bool) = (43, false);
   println!("{}, {}", t.0, t.1);
   println!("{}, {}", u.0, u.1);
   let (num, flag) = get_tuple(); // Tuple destructuring
   println!("{num}, {flag}");
}
```

### Rust 引用
- Rust 中的引用大致相当于 C 中的指针，但有一些关键差异
    - 在任何时刻，拥有任意数量的变量只读（不可变）引用是合法的。引用不能超出变量作用域（这是一个称为 **lifetime** 的关键概念；稍后将详细讨论）
    - 只允许对可变变量拥有单个可写（可变）引用，并且它不能与任何其他引用重叠。
```rust
fn main() {
    let mut a = 42;
    {
        let b = &a;
        let c = b;
        println!("{} {}", *b, *c); // The compiler automatically dereferences *c
        // Illegal because b and still are still in scope
        // let d = &mut a;
    }
    let d = &mut a; // Ok: b and c are not in scope
    *d = 43;
}
```

----
# Rust 切片
- Rust 引用可用于创建数组的子集
    - 与在编译时确定静态固定长度的数组不同，切片可以是任意大小。在内部，切片实现为"胖指针"，包含切片的长度和指向原始数组起始元素的指针
```rust
fn main() {
    let a = [40, 41, 42, 43];
    let b = &a[1..a.len()]; // A slice starting with the second element in the original
    let c = &a[1..]; // Same as the above
    let d = &a[..]; // Same as &a[0..] or &a[0..a.len()]
    println!("{b:?} {c:?} {d:?}");
}
```
----
# Rust 常量和静态变量
- `const` 关键字可用于定义常量值。常量值在**编译时**求值并内联到程序中
- `static` 关键字用于定义类似于 C/C++ 语言中全局变量的等价物。静态变量有一个可寻址的内存位置，创建一次并持续整个程序生命周期
```rust
const SECRET_OF_LIFE: u32 = 42;
static GLOBAL_VARIABLE : u32 = 2;
fn main() {
    println!("The secret of life is {}", SECRET_OF_LIFE);
    println!("Value of global variable is {GLOBAL_VARIABLE}")
}
```

----
# Rust 字符串：String vs &str

- Rust 有**两种**服务于不同目的的字符串类型
    - `String` — 拥有的、堆分配的、可增长的（类似于 C 的 `malloc`'d 缓冲区，或 C++ 的 `std::string`）
    - `&str` — 借用的、轻量级引用（类似于 C 的带有长度的 `const char*`，或 C++ 的 `std::string_view`——但 `&str` 是**经过 lifetime 检查的**，所以它永远不会悬空）
    - 与 C 的空字符终止字符串不同，Rust 字符串跟踪其长度并保证有效的 UTF-8

> **对于 C++ 开发者：** `String` ≈ `std::string`，`&str` ≈ `std::string_view`。与 `std::string_view` 不同，`&str` 由借用检查器保证在其整个生命周期内有效。

## String vs &str：拥有 vs 借用

> **生产模式**：参见 [JSON 处理：nlohmann::json → serde](ch17-2-avoiding-unchecked-indexing.md#json-handling-nlohmannjson--serde) 了解在生产代码中 serde 如何处理字符串。

| **方面** | **C `char*`** | **C++ `std::string`** | **Rust `String`** | **Rust `&str`** |
|------------|--------------|----------------------|-------------------|----------------|
| **内存** | 手动（`malloc`/`free`） | 堆分配，拥有缓冲区 | 堆分配，自动释放 | 借用引用（lifetime 检查） |
| **可变性** | 通过指针始终可变 | 可变 | 使用 `mut` 可变 | 始终不可变 |
| **大小信息** | 无（依赖 `'\0'`） | 跟踪长度和容量 | 跟踪长度和容量 | 跟踪长度（胖指针） |
| **编码** | 未指定（通常 ASCII） | 未指定（通常 ASCII） | 保证有效的 UTF-8 | 保证有效的 UTF-8 |
| **空字符终止符** | 需要 | 需要（`c_str()`） | 不使用 | 不使用 |

```rust
fn main() {
    // &str - string slice (borrowed, immutable, usually a string literal)
    let greeting: &str = "Hello";  // Points to read-only memory

    // String - owned, heap-allocated, growable
    let mut owned = String::from(greeting);  // Copies data to heap
    owned.push_str(", World!");        // Grow the string
    owned.push('!');                   // Append a single character

    // Converting between String and &str
    let slice: &str = &owned;          // String -> &str (free, just a borrow)
    let owned2: String = slice.to_string();  // &str -> String (allocates)
    let owned3: String = String::from(slice); // Same as above

    // String concatenation (note: + consumes the left operand)
    let hello = String::from("Hello");
    let world = String::from(", World!");
    let combined = hello + &world;  // hello is moved (consumed), world is borrowed
    // println!("{hello}");  // Won't compile: hello was moved

    // Use format! to avoid move issues
    let a = String::from("Hello");
    let b = String::from("World");
    let combined = format!("{a}, {b}!");  // Neither a nor b is consumed

    println!("{combined}");
}
```

## 为什么不能用 `[]` 索引字符串
```rust
fn main() {
    let s = String::from("hello");
    // let c = s[0];  // Won't compile! Rust strings are UTF-8, not byte arrays

    // Safe alternatives:
    let first_char = s.chars().next();           // Option<char>: Some('h')
    let as_bytes = s.as_bytes();                 // &[u8]: raw UTF-8 bytes
    let substring = &s[0..1];                    // &str: "h" (byte range, must be valid UTF-8 boundary)

    println!("First char: {:?}", first_char);
    println!("Bytes: {:?}", &as_bytes[..5]);
}
```

## 练习：字符串操作

🟢 **入门级**
- 编写一个函数 `fn count_words(text: &str) -> usize` 来计算字符串中由空白分隔的单词数
- 编写一个函数 `fn longest_word(text: &str) -> &str` 返回最长的单词（提示：你需要考虑 lifetimes——为什么返回类型需要是 `&str` 而不是 `String`？）

<details><summary>解决方案（点击展开）</summary>

```rust
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}

fn longest_word(text: &str) -> &str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

fn main() {
    let text = "the quick brown fox jumps over the lazy dog";
    println!("Word count: {}", count_words(text));       // 9
    println!("Longest word: {}", longest_word(text));     // "jumps"
}
```

</details>

# Rust 结构体
- `struct` 关键字声明一个用户定义的结构体类型
    - `struct` 成员可以是命名的或匿名的（元组结构体）
- 与 C++ 等语言不同，Rust 中没有"数据继承"的概念
```rust
fn main() {
    struct MyStruct {
        num: u32,
        is_secret_of_life: bool,
    }
    let x = MyStruct {
        num: 42,
        is_secret_of_life: true,
    };
    let y = MyStruct {
        num: x.num,
        is_secret_of_life: x.is_secret_of_life,
    };
    let z = MyStruct { num: x.num, ..x }; // The .. means copy remaining
    println!("{} {} {}", x.num, y.is_secret_of_life, z.num);
}
```

# Rust 元组结构体
- Rust 元组结构体类似于元组，各个字段没有名称
    - 像元组一样，各元素使用 .0、.1、.2 ... 访问。元组结构体的一个常见用例是包装原始类型以创建自定义类型。**这对于避免混合相同类型的不同值很有用**
```rust
struct WeightInGrams(u32);
struct WeightInMilligrams(u32);
fn to_weight_in_grams(kilograms: u32) -> WeightInGrams {
    WeightInGrams(kilograms * 1000)
}

fn to_weight_in_milligrams(w : WeightInGrams) -> WeightInMilligrams  {
    WeightInMilligrams(w.0 * 1000)
}

fn main() {
    let x = to_weight_in_grams(42);
    let y = to_weight_in_milligrams(x);
    // let z : WeightInGrams = x;  // Won't compile: x was moved into to_weight_in_milligrams()
    // let a : WeightInGrams = y;   // Won't compile: type mismatch (WeightInMilligrams vs WeightInGrams)
}
```


**注意**：`#[derive(...)]` 属性自动为结构体和枚举生成常见的 trait 实现。你将在整个课程中看到它的使用：
```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);           // Debug: works because of #[derive(Debug)]
    let p2 = p.clone();           // Clone: works because of #[derive(Clone)]
    assert_eq!(p, p2);            // PartialEq: works because of #[derive(PartialEq)]
}
```
我们稍后将深入介绍 trait 系统，但 `#[derive(Debug)]` 非常有用，你应该为你创建的几乎每个 `struct` 和 `enum` 添加它。

# Rust Vec 类型
- `Vec<T>` 类型实现动态堆分配缓冲区（类似于 C 中手动管理的 `malloc`/`realloc` 数组，或 C++ 的 `std::vector`）
    - 与固定大小的数组不同，`Vec` 可以在运行时增长和收缩
    - `Vec` 拥有其数据并自动管理内存分配/释放
- 常见操作：`push()`、`pop()`、`insert()`、`remove()`、`len()`、`capacity()`
```rust
fn main() {
    let mut v = Vec::new();    // Empty vector, type inferred from usage
    v.push(42);                // Add element to end - Vec<i32>
    v.push(43);                
    
    // Safe iteration (preferred)
    for x in &v {              // Borrow elements, don't consume vector
        println!("{x}");
    }
    
    // Initialization shortcuts
    let mut v2 = vec![1, 2, 3, 4, 5];           // Macro for initialization
    let v3 = vec![0; 10];                       // 10 zeros
    
    // Safe access methods (preferred over indexing)
    match v2.get(0) {
        Some(first) => println!("First: {first}"),
        None => println!("Empty vector"),
    }
    
    // Useful methods
    println!("Length: {}, Capacity: {}", v2.len(), v2.capacity());
    if let Some(last) = v2.pop() {             // Remove and return last element
        println!("Popped: {last}");
    }
    
    // Dangerous: direct indexing (can panic!)
    // println!("{}", v2[100]);  // Would panic at runtime
}
```
> **生产模式**：参见 [避免未检查的索引](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing) 了解生产 Rust 代码中安全的 `.get()` 模式。

# Rust HashMap 类型
- `HashMap` 实现通用的 `key` -> `value` 查找（又称 `dictionary` 或 `map`）
```rust
fn main() {
    use std::collections::HashMap;  // Need explicit import, unlike Vec
    let mut map = HashMap::new();       // Allocate an empty HashMap
    map.insert(40, false);  // Type is inferred as int -> bool
    map.insert(41, false);
    map.insert(42, true);
    for (key, value) in map {
        println!("{key} {value}");
    }
    let map = HashMap::from([(40, false), (41, false), (42, true)]);
    if let Some(x) = map.get(&43) {
        println!("43 was mapped to {x:?}");
    } else {
        println!("No mapping was found for 43");
    }
    let x = map.get(&43).or(Some(&false));  // Default value if key isn't found
    println!("{x:?}"); 
}
```

# 练习：Vec 和 HashMap

🟢 **入门级**
- 创建一个带有几个条目的 `HashMap<u32, bool>`（确保一些值是 `true`，一些是 `false`）。遍历哈希表中的所有元素，将键放入一个 `Vec`，将值放入另一个

<details><summary>解决方案（点击展开）</summary>

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::from([(1, true), (2, false), (3, true), (4, false)]);
    let mut keys = Vec::new();
    let mut values = Vec::new();
    for (k, v) in &map {
        keys.push(*k);
        values.push(*v);
    }
    println!("Keys:   {keys:?}");
    println!("Values: {values:?}");

    // Alternative: use iterators with unzip()
    let (keys2, values2): (Vec<u32>, Vec<bool>) = map.into_iter().unzip();
    println!("Keys (unzip):   {keys2:?}");
    println!("Values (unzip): {values2:?}");
}
```

</details>

---

## 深度探讨：C++ 引用 vs Rust 引用

> **对于 C++ 开发者：** C++ 程序员通常认为 Rust `&T` 的工作方式类似于 C++ `T&`。虽然表面上相似，但存在导致混淆的根本差异。C 开发者可以跳过本节——Rust 引用在 [所有权和借用](ch07-ownership-and-borrowing.md) 中介绍。

#### 1. 无右值引用或通用引用

在 C++ 中，`&&` 根据上下文有两种含义：

```cpp
// C++: && means different things:
int&& rref = 42;           // Rvalue reference — binds to temporaries
void process(Widget&& w);   // Rvalue reference — caller must std::move

// Universal (forwarding) reference — deduced template context:
template<typename T>
void forward(T&& arg) {     // NOT an rvalue ref! Deduced as T& or T&&
    inner(std::forward<T>(arg));  // Perfect forwarding
}
```

**在 Rust 中：这些都不存在。** `&&` 只是逻辑与运算符。

```rust
// Rust: && is just boolean AND
let a = true && false; // false

// Rust has NO rvalue references, no universal references, no perfect forwarding.
// Instead:
//   - Move is the default for non-Copy types (no std::move needed)
//   - Generics + trait bounds replace universal references
//   - No temporary-binding distinction — values are values

fn process(w: Widget) { }      // Takes ownership (like C++ value param + implicit move)
fn process_ref(w: &Widget) { } // Borrows immutably (like C++ const T&)
fn process_mut(w: &mut Widget) { } // Borrows mutably (like C++ T&, but exclusive)
```

| C++ 概念 | Rust 等价物 | 备注 |
|-------------|-----------------|-------|
| `T&`（左值引用） | `&T` 或 `&mut T` | Rust 分为共享 vs 独占 |
| `T&&`（右值引用） | 就是 `T` | 按值获取 = 获取所有权 |
| 模板中的 `T&&`（通用引用） | `impl Trait` 或 `<T: Trait>` | 泛型替代转发 |
| `std::move(x)` | `x`（直接使用） | 移动是默认值 |
| `std::forward<T>(x)` | 不需要等价物 | 没有要转发的通用引用 |

#### 2. 移动是位级的——无移动构造函数

在 C++ 中，移动是一个*用户定义的操作*（移动构造函数/移动赋值）。在 Rust 中，移动始终是值的**按位 memcpy**，源被标记为无效：

```rust
// Rust move = memcpy the bytes, mark source as invalid
let s1 = String::from("hello");
let s2 = s1; // Bytes of s1 are copied to s2's stack slot
              // s1 is now invalid — compiler enforces this
// println!("{s1}"); // ❌ Compile error: value used after move
```

```cpp
// C++ move = call the move constructor (user-defined!)
std::string s1 = "hello";
std::string s2 = std::move(s1); // Calls string's move ctor
// s1 is now a "valid but unspecified state" zombie
std::cout << s1; // Compiles! Prints... something (empty string, usually)
```

**后果**：
- Rust 没有五条规则（无需定义拷贝构造函数、移动构造函数、拷贝赋值、移动赋值、析构函数）
- 没有移动后的"僵尸"状态——编译器只是阻止访问
- 移动没有 `noexcept` 考虑——位拷贝不会抛出

#### 3. 自动解引用：编译器看穿间接寻址

Rust 通过 `Deref` trait 自动解引用多层指针/包装器。这在 C++ 中没有等价物：

```rust
use std::sync::{Arc, Mutex};

// Nested wrapping: Arc<Mutex<Vec<String>>>
let data = Arc::new(Mutex::new(vec!["hello".to_string()]));

// In C++, you'd need explicit unlocking and manual dereferencing at each layer.
// In Rust, the compiler auto-derefs through Arc → Mutex → MutexGuard → Vec:
let guard = data.lock().unwrap(); // Arc auto-derefs to Mutex
let first: &str = &guard[0];      // MutexGuard→Vec (Deref), Vec[0] (Index),
                                   // &String→&str (Deref coercion)
println!("First: {first}");

// Method calls also auto-deref:
let boxed_string = Box::new(String::from("hello"));
println!("Length: {}", boxed_string.len());  // Box→String, then String::len()
// No need for (*boxed_string).len() or boxed_string->len()
```

**Deref 强制转换**也适用于函数参数——编译器插入解引用以使类型匹配：

```rust
fn greet(name: &str) {
    println!("Hello, {name}");
}

fn main() {
    let owned = String::from("Alice");
    let boxed = Box::new(String::from("Bob"));
    let arced = std::sync::Arc::new(String::from("Carol"));

    greet(&owned);  // &String → &str  (1 deref coercion)
    greet(&boxed);  // &Box<String> → &String → &str  (2 deref coercions)
    greet(&arced);  // &Arc<String> → &String → &str  (2 deref coercions)
    greet("Dave");  // &str already — no coercion needed
}
// In C++ you'd need .c_str() or explicit conversions for each case.
```

**Deref 链**：当你调用 `x.method()` 时，Rust 的方法解析尝试接收者类型 `T`，然后是 `&T`，然后是 `&mut T`。如果没有匹配，它通过 `Deref` trait 解引用并用目标类型重复。这通过多层继续——这就是为什么 `Box<Vec<T>>` 像 `Vec<T>` 一样"正常工作"。Deref *强制转换*（对于函数参数）是一个单独的但相关的机制，通过链接 `Deref` impls 自动将 `&Box<String>` 转换为 `&str`。

#### 4. 无空引用，无可选引用

```cpp
// C++: references can't be null, but pointers can, and the distinction is blurry
Widget& ref = *ptr;  // If ptr is null → UB
Widget* opt = nullptr;  // "optional" reference via pointer
```

```rust
// Rust: references are ALWAYS valid — guaranteed by the borrow checker
// No way to create a null or dangling reference in safe code
let r: &i32 = &42; // Always valid

// "Optional reference" is explicit:
let opt: Option<&Widget> = None; // Clear intent, no null pointer
if let Some(w) = opt {
    w.do_something(); // Only reachable when present
}
```

#### 5. 引用不能重新绑定

```cpp
// C++: a reference is an alias — it can't be rebound
int a = 1, b = 2;
int& r = a;
r = b;  // This ASSIGNS b's value to a — it does NOT rebind r!
// a is now 2, r still refers to a
```

```rust
// Rust: let bindings can shadow, but references follow different rules
let a = 1;
let b = 2;
let r = &a;
// r = &b;   // ❌ Cannot assign to immutable variable
let r = &b;  // ✅ But you can SHADOW r with a new binding
             // The old binding is gone, not reseated

// With mut:
let mut r = &a;
r = &b;      // ✅ r now points to b — this IS rebinding (not assignment through)
```

> **心智模型**：在 C++ 中，引用是对一个对象的永久别名。
> 在 Rust 中，引用是一个值（带有生命周期保证的指针），遵循正常的变量绑定规则——默认不可变，仅在声明 `mut` 时可重新绑定。
