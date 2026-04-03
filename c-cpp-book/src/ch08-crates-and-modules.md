# Rust crate 和模块

> **你将学到什么：** Rust 如何将代码组织成模块和 crate——默认私有的可见性、`pub` 修饰符、工作区，以及 `crates.io` 生态系统。替代 C/C++ 头文件、`#include` 和 CMake 依赖管理。

- 模块是 crate 内代码的基本组织单位
    - 每个源文件（.rs）都是它自己的模块，可以使用 `mod` 关键字创建嵌套模块
    - （子）模块中的所有类型默认是**私有的**，在同一 crate 中，除非显式标记为 `pub`（公开），否则外部不可见。`pub` 的作用域可以进一步限制为 `pub(crate)` 等
    - 即使一个类型是公开的，它也不会自动在同一模块的另一个作用域中可见，除非使用 `use` 关键字导入。子子模块可以使用 `use super::` 引用父作用域中的类型
    - 源文件（.rs）**除非**在 `main.rs`（可执行文件）或 `lib.rs` 中明确列出，否则不会自动包含在 crate 中

# 练习：模块和函数
- 我们将看看修改我们的 [hello world](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=522d86dbb8c4af71ff2ec081fb76aee7) 来调用另一个函数
    - 如前所述，函数用 `fn` 关键字定义。`->` 关键字声明函数返回一个值（默认是 void），类型为 `u32`（无符号 32 位整数）
    - 函数按模块作用域化，即两个模块中完全同名的两个函数不会有名称冲突
        - 模块作用域扩展到所有类型（例如，`mod a { struct foo; }` 中的 `struct foo` 是一个独特的类型（`a::foo`），不同于 `mod b { struct foo; }`（`b::foo`））

**Starter code** — complete the functions:
```rust
mod math {
    // TODO: implement pub fn add(a: u32, b: u32) -> u32
}

fn greet(name: &str) -> String {
    // TODO: return "Hello, <name>! The secret number is <math::add(21,21)>"
    todo!()
}

fn main() {
    println!("{}", greet("Rustacean"));
}
```

<details><summary>Solution (click to expand)</summary>

```rust
mod math {
    pub fn add(a: u32, b: u32) -> u32 {
        a + b
    }
}

fn greet(name: &str) -> String {
    format!("Hello, {}! The secret number is {}", name, math::add(21, 21))
}

fn main() {
    println!("{}", greet("Rustacean"));
}
// Output: Hello, Rustacean! The secret number is 42
```

</details>
## 工作区和 crate（包）

- 任何重要的 Rust 项目都应该使用工作区来组织组件 crate
    - 工作区只是将用于构建目标二进制文件的本地 crate 集合。workspace 根目录的 `Cargo.toml` 应该指向组成包（crate）

```toml
[workspace]
resolver = "2"
members = ["package1", "package2"]
```

```text
workspace_root/
|-- Cargo.toml      # Workspace configuration
|-- package1/
|   |-- Cargo.toml  # Package 1 configuration
|   `-- src/
|       `-- lib.rs  # Package 1 source code
|-- package2/
|   |-- Cargo.toml  # Package 2 configuration
|   `-- src/
|       `-- main.rs # Package 2 source code
```

---
## 练习：使用工作区和包依赖
- 我们将创建一个简单的包并从我们的 `hello world` 程序中使用它`
- 创建工作区目录
```bash
mkdir workspace
cd workspace
```
- 创建一个名为 Cargo.toml 的文件并在其中添加以下内容。这会创建一个空的工作区
```toml
[workspace]
resolver = "2"
members = []
```
- 添加包（`cargo new --lib` 指定一个库而不是可执行文件`）
```bash
cargo new hello
cargo new --lib hellolib
```

## 练习：使用工作区和包依赖
- 查看 `hello` 和 `hellolib` 中生成的 Cargo.toml。注意它们都被添加到了上层 `Cargo.toml`
- `hellolib` 中 `lib.rs` 的存在意味着是一个库包（参见 https://doc.rust-lang.org/cargo/reference/cargo-targets.html 了解自定义选项）
- 在 `hello` 的 `Cargo.toml` 中添加对 `hellolib` 的依赖
```toml
[dependencies]
hellolib = {path = "../hellolib"}
```
- 使用 `hellolib` 中的 `add()`
```rust
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
```

<details><summary>Solution (click to expand)</summary>

完整的工作区设置：

```bash
# Terminal commands
mkdir workspace && cd workspace

# Create workspace Cargo.toml
cat > Cargo.toml << 'EOF'
[workspace]
resolver = "2"
members = ["hello", "hellolib"]
EOF

cargo new hello
cargo new --lib hellolib
```

```toml
# hello/Cargo.toml — add dependency
[dependencies]
hellolib = {path = "../hellolib"}
```

```rust
// hellolib/src/lib.rs — already has add() from cargo new --lib
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
```

```rust,ignore
// hello/src/main.rs
fn main() {
    println!("Hello, world! {}", hellolib::add(21, 21));
}
// Output: Hello, world! 42
```

</details>

# 使用来自 crates.io 的社区 crate
- Rust 拥有充满活力的社区 crate 生态系统（见 https://crates.io/）
    - Rust 的理念是保持标准库紧凑，将功能外包给社区 crate
    - 关于使用社区 crate 没有硬性规定，但经验法则应该是确保 crate 具有相当成熟的水平（由版本号表明），并且正在被积极维护。如果对某个 crate 有疑问，请联系内部来源
- 发布在 `crates.io` 上的每个 crate 都有主版本和次版本
    - crate 应该遵守这里定义的 major 和 minor `SemVer` 指南：https://doc.rust-lang.org/cargo/reference/semver.html
    - TL;DR 版本是，对于相同的次版本，不应该有破坏性更改。例如，v0.11 必须与 v0.15 兼容（但 v0.20 可能有破坏性更改）

# Crate 依赖和 SemVer
- Crate 可以定义对特定版本、特定次版本或主版本或不关心版本的依赖。以下示例显示了声明对 `rand` crate 依赖的 `Cargo.toml` 条目
- 至少 `0.10.0`，但任何 `< 0.11.0` 都可以
```toml
[dependencies]
rand = { version = "0.10.0"}
```
- 仅 `0.10.0`，没有别的
```toml
[dependencies]
rand = { version = "=0.10.0"}
```
- 不关心；`cargo` 将选择最新版本
```toml
[dependencies]
rand = { version = "*"}
```
- 参考：https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
----
# 练习：使用 rand crate
- 修改 `helloworld` 示例以打印随机数
- 使用 `cargo add rand` 添加依赖
- 使用 `https://docs.rs/rand/latest/rand/` 作为 API 参考

**Starter code** — add this to `main.rs` after running `cargo add rand`:
```rust,ignore
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    // TODO: Generate and print a random u32 in 1..=100
    // TODO: Generate and print a random bool
    // TODO: Generate and print a random f64
}
```

<details><summary>Solution (click to expand)</summary>

```rust
use rand::RngExt;

fn main() {
    let mut rng = rand::rng();
    let n: u32 = rng.random_range(1..=100);
    println!("Random number (1-100): {n}");

    // Generate a random boolean
    let b: bool = rng.random();
    println!("Random bool: {b}");

    // Generate a random float between 0.0 and 1.0
    let f: f64 = rng.random();
    println!("Random float: {f:.4}");
}
```

</details>

# Cargo.toml 和 Cargo.lock
- 如前所述，Cargo.lock 是从 Cargo.toml 自动生成的
    - Cargo.lock 背后的主要思想是确保可重现的构建。例如，如果 `Cargo.toml` 指定了版本 `0.10.0`，cargo 可以自由选择任何 `< 0.11.0` 的版本
    - Cargo.lock 包含构建期间使用的 rand crate 的*特定*版本。
    - 建议将 `Cargo.lock` 包含在 git 仓库中以确保可重现的构建

## Cargo test 功能
- Rust 单元测试驻留在同一源文件中（按惯例），通常分组到单独的模块中
    - 测试代码从不包含在实际的二进制文件中。这是通过 `cfg`（配置）功能实现的。配置对于创建特定平台代码（例如 `Linux` vs. `Windows`）很有用
    - 可以使用 `cargo test` 执行测试。参考：https://doc.rust-lang.org/reference/conditional-compilation.html

```rust
pub fn add(left: u64, right: u64) -> u64 {
    left + right
}
// Will be included only during testing
#[cfg(test)]
mod tests {
    use super::*; // This makes all types in the parent scope visible
    #[test]
    fn it_works() {
        let result = add(2, 2); // Alternatively, super::add(2, 2);
        assert_eq!(result, 4);
    }
}
```

# 其他 Cargo 功能
- `cargo` 还有几个其他有用的功能，包括：
    - `cargo clippy` 是检查 Rust 代码的好方法。通常，应该修复警告（或者在真正有理由时很少抑制）
    - `cargo format` 执行 `rustfmt` 工具来格式化源代码。使用该工具确保检查过代码的标准格式，并结束关于样式的争论
    - `cargo doc` 可用于从 `///` 风格注释生成文档。`crates.io` 上所有 crate 的文档都是使用这种方法生成的

### 构建配置文件：控制优化

在 C 中，你向 `gcc`/`clang` 传递 `-O0`、`-O2`、`-Os`、`-flto`。在 Rust 中，你在 `Cargo.toml` 中配置构建配置文件：

```toml
# Cargo.toml — build profile configuration

[profile.dev]
opt-level = 0          # No optimization (fast compile, like -O0)
debug = true           # Full debug symbols (like -g)

[profile.release]
opt-level = 3          # Maximum optimization (like -O3)
lto = "fat"            # Link-Time Optimization (like -flto)
strip = true           # Strip symbols (like the strip command)
codegen-units = 1      # Single codegen unit — slower compile, better optimization
panic = "abort"        # No unwind tables (smaller binary)
```

| C/GCC 标志 | Cargo.toml 键 | 值 |
|------------|---------------|--------|
| `-O0` / `-O2` / `-O3` | `opt-level` | `0`、`1`、`2`、`3`、`"s"`、`"z"` |
| `-flto` | `lto` | `false`、`"thin"`、`"fat"` |
| `-g` / 无 `-g` | `debug` | `true`、`false`、`"line-tables-only"` |
| `strip` 命令 | `strip` | `"none"`、`"debuginfo"`、`"symbols"`、`true`/`false` |
| — | `codegen-units` | `1` = 最佳优化，最慢编译 |

```bash
cargo build              # Uses [profile.dev]
cargo build --release    # Uses [profile.release]
```

### 构建脚本（`build.rs`）：链接 C 库

在 C 中，你使用 Makefiles 或 CMake 链接库并运行代码生成。
Rust 在 crate 根目录使用 `build.rs` 文件：

```rust
// build.rs — runs before compiling the crate

fn main() {
    // Link a system C library (like -lbmc_ipmi in gcc)
    println!("cargo::rustc-link-lib=bmc_ipmi");

    // Where to find the library (like -L/usr/lib/bmc)
    println!("cargo::rustc-link-search=/usr/lib/bmc");

    // Re-run if the C header changes
    println!("cargo::rerun-if-changed=wrapper.h");
}
```

你甚至可以直接从 Rust crate 中编译 C 源文件：

```toml
# Cargo.toml
[build-dependencies]
cc = "1"  # C compiler integration
```

```rust
// build.rs
fn main() {
    cc::Build::new()
        .file("src/c_helpers/ipmi_raw.c")
        .include("/usr/include/bmc")
        .compile("ipmi_raw");   // Produces libipmi_raw.a, linked automatically
    println!("cargo::rerun-if-changed=src/c_helpers/ipmi_raw.c");
}
```

| C / Make / CMake | Rust `build.rs` |
|-----------------|-----------------|
| `-lfoo` | `println!("cargo::rustc-link-lib=foo")` |
| `-L/path` | `println!("cargo::rustc-link-search=/path")` |
| 编译 C 源文件 | `cc::Build::new().file("foo.c").compile("foo")` |
| 生成代码 | 将文件写入 `$OUT_DIR`，然后 `include!()` |

### 交叉编译

在 C 中，交叉编译需要安装单独的工具链（`arm-linux-gnueabihf-gcc`）
并配置 Make/CMake。在 Rust 中：

```bash
# Install a cross-compilation target
rustup target add aarch64-unknown-linux-gnu

# Cross-compile
cargo build --target aarch64-unknown-linux-gnu --release
```

在 `.cargo/config.toml` 中指定链接器：

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
```

| C 交叉编译 | Rust 等价物 |
|-----------------|-----------------|
| `apt install gcc-aarch64-linux-gnu` | `rustup target add aarch64-unknown-linux-gnu` + 安装链接器 |
| `CC=aarch64-linux-gnu-gcc make` | `.cargo/config.toml` `[target.X] linker = "..."` |
| `#ifdef __aarch64__` | `#[cfg(target_arch = "aarch64")]` |
| 单独的 Makefile 目标 | `cargo build --target ...` |

### 功能标志：条件编译

C 使用 `#ifdef` 和 `-DFOO` 进行条件编译。Rust 使用在 `Cargo.toml` 中定义的功能标志：

```toml
# Cargo.toml
[features]
default = ["json"]         # Enabled by default
json = ["dep:serde_json"]  # Optional dependency
verbose = []               # Flag with no dependency
gpu = ["dep:cuda-sys"]     # Optional GPU support
```

```rust
// Code gated on features:
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose"))]
macro_rules! verbose {
    ($($arg:tt)*) => {}; // Compiles to nothing
}
```

| C 预处理器 | Rust 功能标志 |
|---------------|-------------------|
| `gcc -DDEBUG` | `cargo build --features verbose` |
| `#ifdef DEBUG` | `#[cfg(feature = "verbose")]` |
| `#define MAX 100` | `const MAX: u32 = 100;` |
| `#ifdef __linux__` | `#[cfg(target_os = "linux")]` |

### 集成测试 vs 单元测试

单元测试与代码相邻，使用 `#[cfg(test)]`。**集成测试**位于 `tests/` 中，仅测试你 crate 的**公共 API**：

```rust
// tests/smoke_test.rs — no #[cfg(test)] needed
use my_crate::parse_config;

#[test]
fn parse_valid_config() {
    let config = parse_config("test_data/valid.json").unwrap();
    assert_eq!(config.max_retries, 5);
}
```

| 方面 | 单元测试（`#[cfg(test)]`） | 集成测试（`tests/`） |
|--------|----------------------------|------------------------------|
| 位置 | 与代码同一文件 | 单独的 `tests/` 目录 |
| 访问 | 私有 + 公共项 | **仅公共 API** |
| 运行命令 | `cargo test` | `cargo test --test smoke_test` |


### 测试模式和策略

C 固件团队通常使用大量样板代码在 CUnit、CMocka 或自定义框架中编写测试。Rust 内置的测试工具功能更强大。本节介绍生产代码所需的模式。

#### `#[should_panic]` — 测试预期失败

```rust
// Test that certain conditions cause panics (like C's assert failures)
#[test]
#[should_panic(expected = "index out of bounds")]
fn test_bounds_check() {
    let v = vec![1, 2, 3];
    let _ = v[10];  // Should panic
}

#[test]
#[should_panic(expected = "temperature exceeds safe limit")]
fn test_thermal_shutdown() {
    fn check_temperature(celsius: f64) {
        if celsius > 105.0 {
            panic!("temperature exceeds safe limit: {celsius}°C");
        }
    }
    check_temperature(110.0);
}
```

#### `#[ignore]` — 慢速或硬件相关测试

```rust
// Mark tests that require special conditions (like C's #ifdef HARDWARE_TEST)
#[test]
#[ignore = "requires GPU hardware"]
fn test_gpu_ecc_scrub() {
    // This test only runs on machines with GPUs
    // Run with: cargo test -- --ignored
    // Run with: cargo test -- --include-ignored  (runs ALL tests)
}
```

#### 返回 Result 的测试（替代 `unwrap` 链）

```rust
// Instead of many unwrap() calls that hide the actual failure:
#[test]
fn test_config_parsing() -> Result<(), Box<dyn std::error::Error>> {
    let json = r#"{"hostname": "node-01", "port": 8080}"#;
    let config: ServerConfig = serde_json::from_str(json)?;  // ? instead of unwrap()
    assert_eq!(config.hostname, "node-01");
    assert_eq!(config.port, 8080);
    Ok(())  // Test passes if we reach here without error
}
```

#### 使用 Builder 函数的测试固件

C 使用 `setUp()`/`tearDown()` 函数。Rust 使用辅助函数和 `Drop`：

```rust
struct TestFixture {
    temp_dir: std::path::PathBuf,
    config: Config,
}

impl TestFixture {
    fn new() -> Self {
        let temp_dir = std::env::temp_dir().join(format!("test_{}", std::process::id()));
        std::fs::create_dir_all(&temp_dir).unwrap();
        let config = Config {
            log_dir: temp_dir.clone(),
            max_retries: 3,
            ..Default::default()
        };
        Self { temp_dir, config }
    }
}

impl Drop for TestFixture {
    fn drop(&mut self) {
        // Automatic cleanup — like C's tearDown() but can't be forgotten
        let _ = std::fs::remove_dir_all(&self.temp_dir);
    }
}

#[test]
fn test_with_fixture() {
    let fixture = TestFixture::new();
    // Use fixture.config, fixture.temp_dir...
    assert!(fixture.temp_dir.exists());
    // fixture is automatically dropped here → cleanup runs
}
```

#### 用于硬件接口的 Trait 模拟

在 C 中，模拟硬件需要预处理器技巧或函数指针交换。
在 Rust 中，traits 使这变得自然：

```rust
// Production trait for IPMI communication
trait IpmiTransport {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String>;
}

// Real implementation (used in production)
struct RealIpmi { /* BMC connection details */ }
impl IpmiTransport for RealIpmi {
    fn send_command(&self, cmd: u8, data: &[u8]) -> Result<Vec<u8>, String> {
        // Actually talks to BMC hardware
        todo!("Real IPMI call")
    }
}

// Mock implementation (used in tests)
struct MockIpmi {
    responses: std::collections::HashMap<u8, Vec<u8>>,
}
impl IpmiTransport for MockIpmi {
    fn send_command(&self, cmd: u8, _data: &[u8]) -> Result<Vec<u8>, String> {
        self.responses.get(&cmd)
            .cloned()
            .ok_or_else(|| format!("No mock response for cmd 0x{cmd:02x}"))
    }
}

// Generic function that works with both real and mock
fn read_sensor_temperature(transport: &dyn IpmiTransport) -> Result<f64, String> {
    let response = transport.send_command(0x2D, &[])?;
    if response.len() < 2 {
        return Err("Response too short".into());
    }
    Ok(response[0] as f64 + (response[1] as f64 / 256.0))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_temperature_reading() {
        let mut mock = MockIpmi { responses: std::collections::HashMap::new() };
        mock.responses.insert(0x2D, vec![72, 128]); // 72.5°C

        let temp = read_sensor_temperature(&mock).unwrap();
        assert!((temp - 72.5).abs() < 0.01);
    }

    #[test]
    fn test_short_response() {
        let mock = MockIpmi { responses: std::collections::HashMap::new() };
        // No response configured → error
        assert!(read_sensor_temperature(&mock).is_err());
    }
}
```

#### 使用 `proptest` 的基于属性的测试

不是测试特定值，而是测试必须始终成立的**属性**：

```rust
// Cargo.toml: [dev-dependencies] proptest = "1"
use proptest::prelude::*;

fn parse_sensor_id(s: &str) -> Option<u32> {
    s.strip_prefix("sensor_")?.parse().ok()
}

fn format_sensor_id(id: u32) -> String {
    format!("sensor_{id}")
}

proptest! {
    #[test]
    fn roundtrip_sensor_id(id in 0u32..10000) {
        // Property: format then parse should give back the original
        let formatted = format_sensor_id(id);
        let parsed = parse_sensor_id(&formatted);
        prop_assert_eq!(parsed, Some(id));
    }

    #[test]
    fn parse_rejects_garbage(s in "[^s].*") {
        // Property: strings not starting with 's' should never parse
        let result = parse_sensor_id(&s);
        prop_assert!(result.is_none());
    }
}
```

#### C vs Rust 测试比较

| C 测试 | Rust 等价物 |
|-----------|----------------|
| `CUnit`、`CMocka`、自定义框架 | 内置 `#[test]` + `cargo test` |
| `setUp()` / `tearDown()` | Builder 函数 + `Drop` trait |
| `#ifdef TEST` 模拟函数 | 基于 trait 的依赖注入 |
| `assert(x == y)` | `assert_eq!(x, y)` 带自动差异输出 |
| 单独的测试可执行文件 | 同一二进制文件，使用 `#[cfg(test)]` 条件编译 |
| `valgrind --leak-check=full ./test` | `cargo test`（默认内存安全）+ `cargo miri test` |
| 代码覆盖率：`gcov` / `lcov` | `cargo tarpaulin` 或 `cargo llvm-cov` |
| 测试发现：手动注册 | 自动——任何 `#[test]` fn 都会被发现的 |



