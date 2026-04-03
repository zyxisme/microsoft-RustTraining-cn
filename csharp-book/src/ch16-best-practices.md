## C# 开发者的最佳实践

> **学习内容：** 五种关键思维转变（GC→所有权、异常→Result、继承→组合）、
> 符合语言习惯的项目组织方式、错误处理策略、测试模式，以及 C# 开发者
> 在 Rust 中最常犯的错误。
>
> **难度：** 🟡 中级

### 1. **思维转变**
- **从 GC 到所有权**：思考谁拥有数据以及何时释放
- **从异常到 Result**：使错误处理显式且可见
- **从继承到组合**：使用 trait 来组合行为
- **从 Null 到 Option**：在类型系统中明确表示值的缺失

### 2. **代码组织**
```rust
// Structure projects like C# solutions
src/
├── main.rs          // Program.cs equivalent
├── lib.rs           // Library entry point
├── models/          // Like Models/ folder in C#
│   ├── mod.rs
│   ├── user.rs
│   └── product.rs
├── services/        // Like Services/ folder
│   ├── mod.rs
│   ├── user_service.rs
│   └── product_service.rs
├── controllers/     // Like Controllers/ (for web apps)
├── repositories/    // Like Repositories/
└── utils/          // Like Utilities/
```

### 3. **错误处理策略**
```rust
// Create a common Result type for your application
pub type AppResult<T> = Result<T, AppError>;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("Validation error: {message}")]
    Validation { message: String },

    #[error("Business logic error: {message}")]
    Business { message: String },
}

// Use throughout your application
pub async fn create_user(data: CreateUserRequest) -> AppResult<User> {
    validate_user_data(&data)?;  // Returns AppError::Validation
    let user = repository.create_user(data).await?;  // Returns AppError::Database
    Ok(user)
}
```

### 4. **测试模式**
```rust
// Structure tests like C# unit tests
#[cfg(test)]
mod tests {
    use super::*;
    use rstest::*;  // For parameterized tests like C# [Theory]

    #[test]
    fn test_basic_functionality() {
        // Arrange
        let input = "test data";

        // Act
        let result = process_data(input);

        // Assert
        assert_eq!(result, "expected output");
    }

    #[rstest]
    #[case(1, 2, 3)]
    #[case(5, 5, 10)]
    #[case(0, 0, 0)]
    fn test_addition(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
        assert_eq!(add(a, b), expected);
    }

    #[tokio::test]  // For async tests
    async fn test_async_functionality() {
        let result = async_function().await;
        assert!(result.is_ok());
    }
}
```

### 5. **应避免的常见错误**
```rust
// [ERROR] Don't try to implement inheritance
// Instead of:
// struct Manager : Employee  // This doesn't exist in Rust

// [OK] Use composition with traits
trait Employee {
    fn get_salary(&self) -> u32;
}

trait Manager: Employee {
    fn get_team_size(&self) -> usize;
}

// [ERROR] Don't use unwrap() everywhere (like ignoring exceptions)
let value = might_fail().unwrap();  // Can panic!

// [OK] Handle errors properly
let value = match might_fail() {
    Ok(v) => v,
    Err(e) => {
        log::error!("Operation failed: {}", e);
        return Err(e.into());
    }
};

// [ERROR] Don't clone everything (like copying objects unnecessarily)
let data = expensive_data.clone();  // Expensive!

// [OK] Use borrowing when possible
let data = &expensive_data;  // Just a reference

// [ERROR] Don't use RefCell everywhere (like making everything mutable)
struct Data {
    value: RefCell<i32>,  // Interior mutability - use sparingly
}

// [OK] Prefer owned or borrowed data
struct Data {
    value: i32,  // Simple and clear
}
```

本指南为 C# 开发者提供了将现有知识迁移到 Rust 的全面理解，突出了两者的相似之处
以及方法上的根本差异。关键在于理解 Rust 的约束（如所有权机制）旨在防止
C# 中可能出现的一整类 bug，代价是一些初始的复杂性。

---

### 6. **避免过度使用 `clone()`** 🟡

C# 开发者本能地 clone 数据，因为 GC 会处理成本。在 Rust 中，每一次 `.clone()` 都是
一次显式的内存分配。大多数 clone 可以通过借用来消除。

```rust
// [ERROR] C# habit: cloning strings to pass around
fn greet(name: String) {
    println!("Hello, {name}");
}

let user_name = String::from("Alice");
greet(user_name.clone());  // unnecessary allocation
greet(user_name.clone());  // and again

// [OK] Borrow instead — zero allocation
fn greet(name: &str) {
    println!("Hello, {name}");
}

let user_name = String::from("Alice");
greet(&user_name);  // borrows
greet(&user_name);  // borrows again — no cost
```

**适合使用 clone 的场景：**
- 将数据移动到线程或 `'static` 闭包中（`Arc::clone` 很便宜——只是增加引用计数）
- 缓存：确实需要一个独立的副本
- 原型开发：先让它工作起来，之后再移除 clone

**决策清单：**
1. 能否改用 `&T` 或 `&str`？→ 用这个
2. 被调用者需要所有权吗？→ 通过移动传递，而不是 clone
3. 是否在多个线程间共享？→ 使用 `Arc<T>`（clone 只是增加引用计数）
4. 以上都不是？→ `clone()` 是合理的

---

### 7. **避免在生产代码中使用 `unwrap()`** 🟡

忽略异常的 C# 开发者在 Rust 中会到处写 `.unwrap()`。两者同样危险。

```rust
// [ERROR] The "I'll fix this later" trap
let config = std::fs::read_to_string("config.toml").unwrap();
let port: u16 = config_value.parse().unwrap();
let conn = db_pool.get().await.unwrap();

// [OK] Propagate with ? in application code
let config = std::fs::read_to_string("config.toml")?;
let port: u16 = config_value.parse()?;
let conn = db_pool.get().await?;

// [OK] Use expect() only when failure is truly a bug
let home = std::env::var("HOME")
    .expect("HOME environment variable must be set");  // documents the invariant
```

**经验法则：**
| 方法 | 使用时机 |
|--------|------------|
| `?` | 应用/库代码——传播给调用者 |
| `expect("reason")` | 启动时的断言、*必须*成立的不变量 |
| `unwrap()` | 仅在测试中，或在 `is_some()`/`is_ok()` 检查之后 |
| `unwrap_or(default)` | 当你有合理的默认值时 |
| `unwrap_or_else(|| ...)` | 当回退计算代价较高时 |

---

### 8. **与借用检查器斗争（以及如何停止）** 🟡

每位 C# 开发者都会经历借用检查器拒绝看似有效代码的阶段。解决方案通常是
结构性改变，而不是变通方法。

```rust
// [ERROR] Trying to mutate while iterating (C# foreach + modify pattern)
let mut items = vec![1, 2, 3, 4, 5];
for item in &items {
    if *item > 3 {
        items.push(*item * 2);  // ERROR: can't borrow items as mutable
    }
}

// [OK] Collect first, then mutate
let extras: Vec<i32> = items.iter()
    .filter(|&&x| x > 3)
    .map(|&x| x * 2)
    .collect();
items.extend(extras);
```

```rust
// [ERROR] Returning a reference to a local (C# returns references freely via GC)
fn get_greeting() -> &str {
    let s = String::from("hello");
    &s  // ERROR: s is dropped at end of function
}

// [OK] Return owned data
fn get_greeting() -> String {
    String::from("hello")  // caller owns it
}
```

**解决借用检查器冲突的常见模式：**

| C# 习惯 | Rust 解决方案 |
|----------|--------------|
| 在结构体中存储引用 | 使用拥有所有权的，或添加生命周期参数 |
| 自由地改变共享状态 | 使用 `Arc<Mutex<T>>` 或重构以避免共享 |
| 返回局部变量的引用 | 返回拥有所有权的值 |
| 迭代时修改集合 | 先收集修改，再应用 |
| 多个可变引用 | 将结构体拆分为独立的部分 |

---

### 9. **消除赋值金字塔** 🟢

C# 开发者写 `if (x != null) { if (x.Value > 0) { ... } }` 这样的链式调用。
Rust 的 `match`、`if let` 和 `?` 可以将这些扁平化。

```rust
// [ERROR] Nested null-checking style from C#
fn process(input: Option<String>) -> Option<usize> {
    match input {
        Some(s) => {
            if !s.is_empty() {
                match s.parse::<usize>() {
                    Ok(n) => {
                        if n > 0 {
                            Some(n * 2)
                        } else {
                            None
                        }
                    }
                    Err(_) => None,
                }
            } else {
                None
            }
        }
        None => None,
    }
}

// [OK] Flatten with combinators
fn process(input: Option<String>) -> Option<usize> {
    input
        .filter(|s| !s.is_empty())
        .and_then(|s| s.parse::<usize>().ok())
        .filter(|&n| n > 0)
        .map(|n| n * 2)
}
```

**每位 C# 开发者都应掌握的关键组合器：**

| 组合器 | 作用 | C# 等价 |
|-----------|-------------|---------------|
| `map` | 转换内部值 | `Select` / 空条件 `?.` |
| `and_then` | 链接返回 Option/Result 的操作 | `SelectMany` / `?.Method()` |
| `filter` | 仅在谓词通过时保留值 | `Where` |
| `unwrap_or` | 提供默认值 | `?? defaultValue` |
| `ok()` | 将 `Result` 转换为 `Option`（丢弃错误） | — |
| `transpose` | 将 `Option<Result>` 翻转为 `Result<Option>` | — |

