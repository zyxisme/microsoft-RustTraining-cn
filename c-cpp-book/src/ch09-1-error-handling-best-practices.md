# Rust Option 和 Result 关键要点

> **你将学到什么：** 惯用的错误处理模式——`unwrap()` 的安全替代品、`?` 操作符用于传播、自定义错误类型，以及在生产代码中何时使用 `anyhow` vs `thiserror`。

- `Option` 和 `Result` 是惯用 Rust 的组成部分
- **`unwrap()` 的安全替代品**：
```rust
// Option<T> safe alternatives
let value = opt.unwrap_or(default);              // Provide fallback value
let value = opt.unwrap_or_else(|| compute());    // Lazy computation for fallback
let value = opt.unwrap_or_default();             // Use Default trait implementation
let value = opt.expect("descriptive message");   // Only when panic is acceptable

// Result<T, E> safe alternatives  
let value = result.unwrap_or(fallback);          // Ignore error, use fallback
let value = result.unwrap_or_else(|e| handle(e)); // Handle error, return fallback
let value = result.unwrap_or_default();          // Use Default trait
```
- **用于显式控制的模式匹配**：
```rust
match some_option {
    Some(value) => println!("Got: {}", value),
    None => println!("No value found"),
}

match some_result {
    Ok(value) => process(value),
    Err(error) => log_error(error),
}
```
- **使用 `?` 操作符进行错误传播**：短路并向上冒泡错误
```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?; // Automatically returns error
    Ok(content.to_uppercase())
}
```
- **转换方法**：
    - `map()`：转换成功值 `Ok(T)` -> `Ok(U)` 或 `Some(T)` -> `Some(U)`
    - `map_err()`：转换错误类型 `Err(E)` -> `Err(F)`
    - `and_then()`：链接可能失败的操作
- **在你自己的 API 中使用**：优先使用 `Result<T, E>` 而不是异常或错误码
- **参考**：[Option 文档](https://doc.rust-lang.org/std/option/enum.Option.html) | [Result 文档](https://doc.rust-lang.org/std/result/enum.Result.html)

# Rust 常见陷阱和调试技巧
- **借用问题**：最常见的初学者错误
    - "cannot borrow as mutable" -> 一次只允许一个可变引用
    - "borrowed value does not live long enough" -> 引用比它指向的数据存活更久
    - **修复**：使用作用域 `{}` 限制引用生命周期，或在需要时克隆数据
- **缺少 trait 实现**："method not found" 错误
    - **修复**：为常见 traits 添加 `#[derive(Debug, Clone, PartialEq)]`
    - 使用 `cargo check` 而不是 `cargo run` 获取更好的错误消息
- **调试模式下的整数溢出**：Rust 在溢出时 panic
    - **修复**：使用 `wrapping_add()`、`saturating_add()` 或 `checked_add()` 获得显式行为
- **String vs &str 混淆**：不同类型用于不同用例
    - 使用 `&str` 作为字符串切片（借用的），`String` 作为拥有的字符串
    - **修复**：使用 `.to_string()` 或 `String::from()` 将 `&str` 转换为 `String`
- **与借用检查器斗争**：不要试图比它更聪明
    - **修复**：重构代码以符合所有权规则而不是违背它们
    - 在复杂的共享场景中考虑使用 `Rc<RefCell<T>>`（谨慎使用）

## 错误处理示例：好与坏
```rust
// [ERROR] BAD: Can panic unexpectedly
fn bad_config_reader() -> String {
    let config = std::env::var("CONFIG_FILE").unwrap(); // Panic if not set!
    std::fs::read_to_string(config).unwrap()           // Panic if file missing!
}

// [OK] GOOD: Handles errors gracefully
fn good_config_reader() -> Result<String, ConfigError> {
    let config_path = std::env::var("CONFIG_FILE")
        .unwrap_or_else(|_| "default.conf".to_string()); // Fallback to default
    
    let content = std::fs::read_to_string(config_path)
        .map_err(ConfigError::FileRead)?;                // Convert and propagate error
    
    Ok(content)
}

// [OK] EVEN BETTER: With proper error types
use thiserror::Error;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("Failed to read config file: {0}")]
    FileRead(#[from] std::io::Error),
    
    #[error("Invalid configuration: {message}")]
    Invalid { message: String },
}
```

让我们分解这里发生了什么。`ConfigError` 只有**两个变体**——一个用于 I/O 错误，一个用于验证错误。这是大多数模块的正确起点：

| `ConfigError` 变体 | 包含 | 创建方式 |
|----------------------|-------|-----------|
| `FileRead(io::Error)` | 原始 I/O 错误 | `#[from]` 通过 `?` 自动转换 |
| `Invalid { message }` | 人类可读的解释 | 你的验证代码 |

现在你可以编写返回 `Result<T, ConfigError>` 的函数：

```rust
fn read_config(path: &str) -> Result<String, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError::FileRead
    if content.is_empty() {
        return Err(ConfigError::Invalid {
            message: "config file is empty".to_string(),
        });
    }
    Ok(content)
}
```

> **🟢 自主学习检查点：** 在继续之前，确保你能回答：
> 1. 为什么 `read_to_string` 调用上的 `?` 有效？（因为 `#[from]` 生成了 `impl From<io::Error> for ConfigError`）
> 2. 如果你添加第三个变体 `MissingKey(String)` 会发生什么——需要什么代码更改？（只需添加变体；现有代码仍然编译）

## Crate 级错误类型和 Result 别名

随着你的项目发展超过单个文件，你将把多个模块级错误组合成一个 **crate 级错误类型**。这是生产 Rust 中的标准模式。让我们从上面的 `ConfigError` 构建。

在现实世界的 Rust 项目中，每个 crate（或重要模块）定义自己的 `Error` 枚举和一个 `Result` 类型别名。这是惯用模式——类似于在 C++ 中你会定义每个库的异常层次结构和 `using Result = std::expected<T, Error>`。

### 模式

```rust
// src/error.rs  (or at the top of lib.rs)
use thiserror::Error;

/// Every error this crate can produce.
#[derive(Error, Debug)]
pub enum Error {
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),          // auto-converts via From

    #[error("JSON parse error: {0}")]
    Json(#[from] serde_json::Error),     // auto-converts via From

    #[error("Invalid sensor id: {0}")]
    InvalidSensor(u32),                  // domain-specific variant

    #[error("Timeout after {ms} ms")]
    Timeout { ms: u64 },
}

/// Crate-wide Result alias — saves typing throughout the crate.
pub type Result<T> = core::result::Result<T, Error>;
```

### 它如何简化每个函数

没有别名你会写：

```rust
// 冗长——错误类型到处重复
fn read_sensor(id: u32) -> Result<f64, crate::Error> { ... }
fn parse_config(path: &str) -> Result<Config, crate::Error> { ... }
```

有别名：

```rust
// 简洁——只是 `Result<T>`
use crate::{Error, Result};

fn read_sensor(id: u32) -> Result<f64> {
    if id > 128 {
        return Err(Error::InvalidSensor(id));
    }
    let raw = std::fs::read_to_string(format!("/dev/sensor/{id}"))?; // io::Error → Error::Io
    let value: f64 = raw.trim().parse()
        .map_err(|_| Error::InvalidSensor(id))?;
    Ok(value)
}
```

`Io` 上的 `#[from]` 属性免费生成这个 `impl`：

```rust
// 由 thiserror 的 #[from] 自动生成
impl From<std::io::Error> for Error {
    fn from(source: std::io::Error) -> Self {
        Error::Io(source)
    }
}
```

这就是使 `?` 工作的原因：当函数返回 `std::io::Error` 而你的函数返回 `Result<T>`（你的别名）时，编译器调用 `From::from()` 自动转换它。

### 组合模块级错误

较大的 crate 按模块拆分错误，然后在 crate 根级别组合它们：

```rust
// src/config/error.rs
#[derive(thiserror::Error, Debug)]
pub enum ConfigError {
    #[error("Missing key: {0}")]
    MissingKey(String),
    #[error("Invalid value for '{key}': {reason}")]
    InvalidValue { key: String, reason: String },
}

// src/error.rs  (crate-level)
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error(transparent)]               // delegates Display to inner error
    Config(#[from] crate::config::ConfigError),

    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
}
pub type Result<T> = core::result::Result<T, Error>;
```

调用者仍然可以匹配特定的配置错误：

```rust
match result {
    Err(Error::Config(ConfigError::MissingKey(k))) => eprintln!("Add '{k}' to config"),
    Err(e) => eprintln!("Other error: {e}"),
    Ok(v) => use_value(v),
}
```

### C++ 比较

| 概念 | C++ | Rust |
|---------|-----|------|
| 错误层次结构 | `class AppError : public std::runtime_error` | `#[derive(thiserror::Error)] enum Error { ... }` |
| 返回错误 | `std::expected<T, Error>` 或 `throw` | `fn foo() -> Result<T>` |
| 转换错误 | 手动 `try/catch` + 重新抛出 | `#[from]` + `?`——零样板代码 |
| Result 别名 | `template<class T> using Result = std::expected<T, Error>;` | `pub type Result<T> = core::result::Result<T, Error>;` |
| 错误消息 | 重写 `what()` | `#[error("...")]`——编译为 `Display` impl |


