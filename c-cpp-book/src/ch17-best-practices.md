# Rust 最佳实践总结

> **你将学到什么：** 编写惯用 Rust 的实用指南——代码组织、命名约定、错误处理模式和文档。这是一个你会经常返回的快速参考章节。

## 代码组织
- **优先使用小函数**：易于测试和推理
- **使用描述性名称**：`calculate_total_price()` vs `calc()`
- **分组相关功能**：使用模块和单独文件
- **编写文档**：为公共 API 使用 `///`

## 错误处理
- **除非确信不会失败，否则避免 `unwrap()`**：只有当你 100% 确定它不会 panic 时才使用
```rust
// Bad: Can panic
let value = some_option.unwrap();

// Good: Handle the None case
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // Uses Default trait

// For Result<T, E>
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("Error occurred: {err}");
    default_value
});
```
- **使用带有描述性消息的 `expect()`**：当 unwrap 是合理的时，解释原因
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("CONFIG_PATH environment variable must be set");
```
- **对可能失败的操作返回 `Result<T, E>`**：让调用者决定如何处理错误
- **使用 `thiserror` 定义自定义错误类型**：比手动实现更符合人体工程学
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {message}")]
    Parse { message: String },
    
    #[error("Value {value} is out of range")]
    OutOfRange { value: i32 },
}
```
- **用 `?` 操作符链接错误**：将错误向上传播到调用栈
- **优先使用 `thiserror` 而不是 `anyhow`**：我们的团队惯例是用 `#[derive(thiserror::Error)]` 定义显式错误枚举，以便调用者可以匹配特定变体。`anyhow::Error` 对于快速原型制作很方便，但会擦除错误类型，使调用者更难处理特定失败。为库和生产代码使用 `thiserror`；为一次性脚本或只需要打印错误的最顶层二进制文件保留 `anyhow`。
- **何时 `unwrap()` 是可接受的**：
  - **单元测试**：`assert_eq!(result.unwrap(), expected)`
  - **原型制作**：你会替换的快速而粗糙的代码
  - **不会失败的操作**：当你能够证明它不会失败时
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // Safe: we just created the vec with elements

// Better: Use expect() with explanation
let first = numbers.get(0).expect("numbers vec is non-empty by construction");
```
- **快速失败**：尽早检查前置条件并立即返回错误

## 内存管理
- **优先借用而非克隆**：尽可能使用 `&T` 而不是克隆
- **谨慎使用 `Rc<T>`**：只在需要共享所有权时使用
- **限制生命周期**：使用作用域 `{}` 来控制值的释放时机
- **避免在公共 API 中使用 `RefCell<T>`**：将内部可变性保持在内

## 性能
- **优化前先进行性能分析**：使用 `cargo bench` 和性能分析工具
- **优先使用迭代器而非循环**：更具可读性且通常更快
- **使用 `&str` 而非 `String`**：当你不需要所有权时
- **考虑使用 `Box<T>` 处理大型栈对象**：必要时将它们移动到堆上

## 应实现的基本 Trait

### 每个类型都应考虑的核心 Trait

在创建自定义类型时，考虑实现这些基本 trait，使你的类型在 Rust 中更加原生：

#### **Debug 和 Display**
```rust
use std::fmt;

#[derive(Debug)]  // 自动实现，用于调试
struct Person {
    name: String,
    age: u32,
}

// 手动实现 Display，用于用户面向的输出
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// 用法：
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice (age 30)
```

#### **Clone 和 Copy**
```rust
// Copy：小而简单类型的隐式复制
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone：复杂类型的显式复制
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String 不实现 Copy
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy（隐式）

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone（显式）
```

#### **PartialEq 和 Eq**
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 不实现 Eq（因为 NaN）
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // 因为 PartialEq 而工作

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // 通过 PartialEq 工作
```

#### **PartialOrd 和 Ord**
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // 数字越小 = 优先级越高

// 在集合中使用
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // 因为 Priority 实现了 Ord 所以工作
```

#### **Default**
```rust
#[derive(Debug, Default)]
struct Config {
    debug: bool,           // false（默认值）
    max_connections: u32,  // 0（默认值）
    timeout: Option<u64>,  // None（默认值）
}

// 自定义 Default 实现
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // 自定义默认值
            timeout: Some(30),     // 自定义默认值
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // 部分覆盖
```

#### **From 和 Into**
```rust
struct UserId(u64);
struct UserName(String);

// 实现 From，Into 自动获得
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// 用法：
let user_id: UserId = 123u64.into();         // 使用 Into
let user_id = UserId::from(123u64);          // 使用 From
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // 使用 Into
```

#### **TryFrom 和 TryInto**
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;

    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// 用法：
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde（用于序列化）**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// 自动 JSON 序列化/反序列化
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### Trait 实现检查清单

对于任何新类型，考虑此检查清单：

```rust
#[derive(
    Debug,          // [OK] Always implement for debugging
    Clone,          // [OK] If the type should be duplicatable
    PartialEq,      // [OK] If the type should be comparable
    Eq,             // [OK] If comparison is reflexive/transitive
    PartialOrd,     // [OK] If the type has ordering
    Ord,            // [OK] If ordering is total
    Hash,           // [OK] If type will be used as HashMap key
    Default,        // [OK] If there's a sensible default value
)]
struct MyType {
    // fields...
}

// Manual implementations to consider:
impl Display for MyType { /* user-facing representation */ }
impl From<OtherType> for MyType { /* convenient conversion */ }
impl TryFrom<FallibleType> for MyType { /* fallible conversion */ }
```

### 何时不实现 Trait

- **不要为包含堆数据的类型实现 Copy**：`String`、`Vec`、`HashMap` 等
- **不要为可能包含 NaN 的值实现 Eq**：包含 `f32`/`f64` 的类型
- **不要在没有合理默认值时实现 Default**：文件句柄、网络连接
- **不要在克隆代价高昂时实现 Clone**：大型数据结构（考虑改用 `Rc<T>`）

### Trait 好处总结

| Trait | 好处 | 何时使用 |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` | 始终（除极少数情况） |
| `Display` | `println!("{}", value)` | 用户面向的类型 |
| `Clone` | `value.clone()` | 当显式复制有意义时 |
| `Copy` | 隐式复制 | 小而简单的类型 |
| `PartialEq` | `==` 和 `!=` 运算符 | 大多数类型 |
| `Eq` | 自反相等性 | 当相等在数学上成立时 |
| `PartialOrd` | `<`、`>`、`<=`、`>=` | 有自然排序的类型 |
| `Ord` | `sort()`、`BinaryHeap` | 当排序是全序时 |
| `Hash` | `HashMap` 键 | 用作 map 键的类型 |
| `Default` | `Default::default()` | 有明显默认值的类型 |
| `From/Into` | 便捷转换 | 常见类型转换 |
| `TryFrom/TryInto` | 可失败转换 | 可能失败转换 |

----

----


