## Rust 与 C# 中的测试对比

> **你将学到：** 内置的 `#[test]` 与 xUnit 的对比，`rstest` 的参数化测试（类似 `[Theory]`），
> 使用 `proptest` 的属性测试，使用 `mockall` 的模拟，以及异步测试模式。
>
> **难度：** 🟡 中级

### 单元测试
```csharp
// C# — xUnit
using Xunit;

public class CalculatorTests
{
    [Fact]
    public void Add_ReturnsSum()
    {
        var calc = new Calculator();
        Assert.Equal(5, calc.Add(2, 3));
    }

    [Theory]
    [InlineData(1, 2, 3)]
    [InlineData(0, 0, 0)]
    [InlineData(-1, 1, 0)]
    public void Add_Theory(int a, int b, int expected)
    {
        Assert.Equal(expected, new Calculator().Add(a, b));
    }
}
```

```rust
// Rust — built-in testing, no external framework needed
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]  // Only compiled during `cargo test`
mod tests {
    use super::*;  // Import from parent module

    #[test]
    fn add_returns_sum() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn add_negative_numbers() {
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    #[should_panic(expected = "overflow")]
    fn add_overflow_panics() {
        let _ = add(i32::MAX, 1); // panics in debug mode
    }
}
```

### 参数化测试（类似 `[Theory]`）
```rust
// Use the `rstest` crate for parameterized tests
use rstest::rstest;

#[rstest]
#[case(1, 2, 3)]
#[case(0, 0, 0)]
#[case(-1, 1, 0)]
fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
    assert_eq!(add(a, b), expected);
}

// Fixtures — like test setup methods
#[rstest]
fn test_with_fixture(#[values(1, 2, 3)] x: i32) {
    assert!(x > 0);
}
```

### 断言对比

| C# (xUnit) | Rust | Notes |
|-------------|------|-------|
| `Assert.Equal(expected, actual)` | `assert_eq!(expected, actual)` | Prints diff on failure |
| `Assert.NotEqual(a, b)` | `assert_ne!(a, b)` | |
| `Assert.True(condition)` | `assert!(condition)` | |
| `Assert.Contains("sub", str)` | `assert!(str.contains("sub"))` | |
| `Assert.Throws<T>(() => ...)` | `#[should_panic]` | Or use `std::panic::catch_unwind` |
| `Assert.Null(obj)` | `assert!(option.is_none())` | No nulls — use `Option` |

### 测试组织

```text
my_crate/
├── src/
│   ├── lib.rs          # Unit tests in #[cfg(test)] mod tests { }
│   └── parser.rs       # Each module can have its own test module
├── tests/              # Integration tests (each file is a separate crate)
│   ├── parser_test.rs  # Tests the public API as an external consumer
│   └── api_test.rs
└── benches/            # Benchmarks (with criterion crate)
    └── my_benchmark.rs
```

```rust
// tests/parser_test.rs — integration test
// Can only access PUBLIC API (like testing from outside the assembly)
use my_crate::parser;

#[test]
fn test_parse_valid_input() {
    let result = parser::parse("valid input");
    assert!(result.is_ok());
}
```

### 异步测试
```csharp
// C# — async test with xUnit
[Fact]
public async Task GetUser_ReturnsUser()
{
    var service = new UserService();
    var user = await service.GetUserAsync(1);
    Assert.Equal("Alice", user.Name);
}
```

```rust
// Rust — async test with tokio
#[tokio::test]
async fn get_user_returns_user() {
    let service = UserService::new();
    let user = service.get_user(1).await.unwrap();
    assert_eq!(user.name, "Alice");
}
```

### 使用 mockall 进行模拟
```rust
use mockall::automock;

#[automock]                         // Generates MockUserRepo struct
trait UserRepo {
    fn find_by_id(&self, id: u32) -> Option<User>;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn service_returns_user_from_repo() {
        let mut mock = MockUserRepo::new();
        mock.expect_find_by_id()
            .with(mockall::predicate::eq(1))
            .returning(|_| Some(User { name: "Alice".into() }));

        let service = UserService::new(mock);
        let user = service.get_user(1).unwrap();
        assert_eq!(user.name, "Alice");
    }
}
```

```csharp
// C# — Moq equivalent
var mock = new Mock<IUserRepo>();
mock.Setup(r => r.FindById(1)).Returns(new User { Name = "Alice" });
var service = new UserService(mock.Object);
Assert.Equal("Alice", service.GetUser(1).Name);
```

<details>
<summary><strong>🏋️ 练习：编写全面的测试</strong>（点击展开）</summary>

**挑战**：给定这个函数，编写覆盖以下场景的测试：正常路径、空输入、数字字符串和 Unicode。

```rust
pub fn title_case(input: &str) -> String {
    input.split_whitespace()
        .map(|word| {
            let mut chars = word.chars();
            match chars.next() {
                Some(c) => format!("{}{}", c.to_uppercase(), chars.as_str().to_lowercase()),
                None => String::new(),
            }
        })
        .collect::<Vec<_>>()
        .join(" ")
}
```

<details>
<summary>🔑 解答</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn happy_path() {
        assert_eq!(title_case("hello world"), "Hello World");
    }

    #[test]
    fn empty_input() {
        assert_eq!(title_case(""), "");
    }

    #[test]
    fn single_word() {
        assert_eq!(title_case("rust"), "Rust");
    }

    #[test]
    fn already_title_case() {
        assert_eq!(title_case("Hello World"), "Hello World");
    }

    #[test]
    fn all_caps() {
        assert_eq!(title_case("HELLO WORLD"), "Hello World");
    }

    #[test]
    fn extra_whitespace() {
        // split_whitespace handles multiple spaces
        assert_eq!(title_case("  hello   world  "), "Hello World");
    }

    #[test]
    fn unicode() {
        assert_eq!(title_case("café résumé"), "Café Résumé");
    }

    #[test]
    fn numeric_words() {
        assert_eq!(title_case("hello 42 world"), "Hello 42 World");
    }
}
```

**关键收获**：Rust 内置的测试框架可以处理大多数单元测试需求。使用 `rstest` 进行参数化测试，使用 `mockall` 进行模拟——无需像 xUnit 那样的大型测试框架。

</details>
</details>


<!-- ch14a.1: Property Testing with proptest -->
## 属性测试：用规模证明正确性

熟悉 **FsCheck** 的 C# 开发者会认知识属性测试：不是编写单独的测试用例，而是描述**所有可能输入**都必须满足的*属性*，然后框架生成数千个随机输入来尝试打破它们。

### 为什么属性测试很重要
```csharp
// C# — Hand-written unit tests check specific cases
[Fact]
public void Reverse_Twice_Returns_Original()
{
    var list = new List<int> { 1, 2, 3 };
    list.Reverse();
    list.Reverse();
    Assert.Equal(new[] { 1, 2, 3 }, list);
}
// But what about empty lists? Single elements? 10,000 elements? Negative numbers?
// You'd need dozens of hand-written cases.
```

```rust
// Rust — proptest generates thousands of inputs automatically
use proptest::prelude::*;

fn reverse<T: Clone>(v: &[T]) -> Vec<T> {
    v.iter().rev().cloned().collect()
}

proptest! {
    #[test]
    fn reverse_twice_is_identity(ref v in prop::collection::vec(any::<i32>(), 0..1000)) {
        let reversed_twice = reverse(&reverse(v));
        prop_assert_eq!(v, &reversed_twice);
    }
    // proptest runs this with hundreds of random Vec<i32> values:
    // [], [0], [i32::MIN, i32::MAX], [42; 999], random sequences...
    // If it fails, it SHRINKS to the smallest failing input!
}
```

### proptest 入门
```toml
# Cargo.toml
[dev-dependencies]
proptest = "1.4"
```

### C# 开发者的常用模式

```rust
use proptest::prelude::*;

// 1. Roundtrip property: serialize → deserialize = identity
// (Like testing JsonSerializer.Serialize → Deserialize)
proptest! {
    #[test]
    fn json_roundtrip(name in "[a-zA-Z]{1,50}", age in 0u32..150) {
        let user = User { name: name.clone(), age };
        let json = serde_json::to_string(&user).unwrap();
        let parsed: User = serde_json::from_str(&json).unwrap();
        prop_assert_eq!(user, parsed);
    }
}

// 2. Invariant property: output always satisfies a condition
proptest! {
    #[test]
    fn sort_output_is_sorted(ref v in prop::collection::vec(any::<i32>(), 0..500)) {
        let mut sorted = v.clone();
        sorted.sort();
        // Every adjacent pair must be in order
        for window in sorted.windows(2) {
            prop_assert!(window[0] <= window[1]);
        }
    }
}

// 3. Oracle property: compare two implementations
proptest! {
    #[test]
    fn fast_path_matches_slow_path(input in "[0-9a-f]{1,100}") {
        let result_fast = parse_hex_fast(&input);
        let result_slow = parse_hex_slow(&input);
        prop_assert_eq!(result_fast, result_slow);
    }
}

// 4. Custom strategies: generate domain-specific test data
fn valid_email() -> impl Strategy<Value = String> {
    ("[a-z]{1,20}", "[a-z]{1,10}", prop::sample::select(vec!["com", "org", "io"]))
        .prop_map(|(user, domain, tld)| format!("{}@{}.{}", user, domain, tld))
}

proptest! {
    #[test]
    fn email_parsing_accepts_valid_emails(email in valid_email()) {
        let result = Email::new(&email);
        prop_assert!(result.is_ok(), "Failed to parse: {}", email);
    }
}
```

### proptest 与 FsCheck 对比

| Feature | C# FsCheck | Rust proptest |
|---------|-----------|---------------|
| Random input generation | `Arb.Generate<T>()` | `any::<T>()` |
| Custom generators | `Arb.Register<T>()` | `impl Strategy<Value = T>` |
| Shrinking on failure | Automatic | Automatic |
| String patterns | Manual | `"[regex]"` strategy |
| Collection generation | `Gen.ListOf` | `prop::collection::vec(strategy, range)` |
| Composing generators | `Gen.Select` | `.prop_map()`, `.prop_flat_map()` |
| Config (# of cases) | `Config.MaxTest` | `#![proptest_config(ProptestConfig::with_cases(10000))]` inside `proptest!` block |

### 何时使用属性测试 vs 单元测试

| Use **unit tests** when | Use **proptest** when |
|------------------------|----------------------|
| Testing specific edge cases | Verifying invariants across all inputs |
| Testing error messages/codes | Roundtrip properties (parse ↔ format) |
| Integration/mock tests | Comparing two implementations |
| Behavior depends on exact values | "For all X, property P holds" |

---

## 集成测试：`tests/` 目录

单元测试位于 `src/` 内部，使用 `#[cfg(test)]`。集成测试位于单独的 `tests/` 目录中，测试你的 crate 的**公共 API**——就像 C# 集成测试将项目作为外部程序集引用一样。

```
my_crate/
├── src/
│   ├── lib.rs          // public API
│   └── internal.rs     // private implementation
├── tests/
│   ├── smoke.rs        // each file is a separate test binary
│   ├── api_tests.rs
│   └── common/
│       └── mod.rs      // shared test helpers
└── Cargo.toml
```

### 编写集成测试

`tests/` 中的每个文件都作为单独的 crate 编译，依赖你的库：

```rust
// tests/smoke.rs — can only access pub items from my_crate
use my_crate::{process_order, Order, OrderResult};

#[test]
fn process_valid_order_returns_confirmation() {
    let order = Order::new("SKU-001", 3);
    let result = process_order(order);
    assert!(matches!(result, OrderResult::Confirmed { .. }));
}
```

### 共享测试辅助函数

将共享的设置代码放在 `tests/common/mod.rs` 中（不是 `tests/common.rs`，后者会被视为独立的测试文件）：

```rust
// tests/common/mod.rs
use my_crate::Config;

pub fn test_config() -> Config {
    Config::builder()
        .database_url("sqlite::memory:")
        .build()
        .expect("test config must be valid")
}
```

```rust
// tests/api_tests.rs
mod common;

use my_crate::App;

#[test]
fn app_starts_with_test_config() {
    let config = common::test_config();
    let app = App::new(config);
    assert!(app.is_healthy());
}
```

### 运行特定类型的测试

```bash
cargo test                  # 运行所有测试（单元 + 集成）
cargo test --lib            # 仅单元测试（类似于 dotnet test --filter Category=Unit）
cargo test --test smoke     # 仅运行 tests/smoke.rs
cargo test --test api_tests # 仅运行 tests/api_tests.rs
```

**与 C# 的关键区别：** 集成测试文件只能访问你的 crate 的 `pub` API。私有函数不可见——这强制你通过公共接口进行测试，这是更好的测试设计。

***


