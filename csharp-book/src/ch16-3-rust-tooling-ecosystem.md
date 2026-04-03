## C# 开发者的 Rust 必备工具

> **你将学到：** Rust 开发工具及其对应的 C# 工具 — Clippy（对应 Roslyn 分析器）、
> rustfmt（对应 dotnet format）、cargo doc（对应 XML 文档）、cargo watch（对应 dotnet watch）以及 VS Code 扩展。
>
> **难度：** 🟢 入门级

### 工具对比

| C# 工具 | Rust 对应工具 | 安装方式 | 用途 |
|---------|----------------|---------|---------|
| Roslyn 分析器 | **Clippy** | `rustup component add clippy` | 代码检查 + 风格建议 |
| `dotnet format` | **rustfmt** | `rustup component add rustfmt` | 自动格式化 |
| XML 文档注释 | **`cargo doc`** | 内置 | 生成 HTML 文档 |
| OmniSharp / Roslyn | **rust-analyzer** | VS Code 扩展 | IDE 支持 |
| `dotnet watch` | **cargo-watch** | `cargo install cargo-watch` | 保存时自动重新构建 |
| — | **cargo-expand** | `cargo install cargo-expand` | 查看宏展开结果 |
| `dotnet audit` | **cargo-audit** | `cargo install cargo-audit` | 安全漏洞扫描 |

### Clippy：你的自动化代码审查工具
```bash
# Run Clippy on your project
cargo clippy

# Treat warnings as errors (CI/CD)
cargo clippy -- -D warnings

# Auto-fix suggestions
cargo clippy --fix
```

```rust
// Clippy catches hundreds of anti-patterns:

// Before Clippy:
if x == true { }           // warning: equality check with bool
let _ = vec.len() == 0;    // warning: use .is_empty() instead
for i in 0..vec.len() { }  // warning: use .iter().enumerate()

// After Clippy suggestions:
if x { }
let _ = vec.is_empty();
for (i, item) in vec.iter().enumerate() { }
```

### rustfmt：一致的格式化
```bash
# Format all files
cargo fmt

# Check formatting without changing (CI/CD)
cargo fmt -- --check
```

```toml
# rustfmt.toml — customize formatting (like .editorconfig)
max_width = 100
tab_spaces = 4
use_field_init_shorthand = true
```

### cargo doc：文档生成
```bash
# Generate and open docs (including dependencies)
cargo doc --open

# Run documentation tests
cargo test --doc
```

```rust
/// Calculate the area of a circle.
///
/// # Arguments
/// * `radius` - The radius of the circle (must be non-negative)
///
/// # Examples
/// ```
/// let area = my_crate::circle_area(5.0);
/// assert!((area - 78.54).abs() < 0.01);
/// ```
///
/// # Panics
/// Panics if `radius` is negative.
pub fn circle_area(radius: f64) -> f64 {
    assert!(radius >= 0.0, "radius must be non-negative");
    std::f64::consts::PI * radius * radius
}
// The code in /// ``` blocks is compiled and run during `cargo test`!
```

### cargo watch：自动重新构建
```bash
# Rebuild on file changes (like dotnet watch)
cargo watch -x check          # Type-check only (fastest)
cargo watch -x test           # Run tests on save
cargo watch -x 'run -- args'  # Run program on save
cargo watch -x clippy         # Lint on save
```

### cargo expand：查看宏生成的内容
```bash
# See the expanded output of derive macros
cargo expand --lib            # Expand lib.rs
cargo expand module_name      # Expand specific module
```

### 推荐的 VS Code 扩展

| 扩展 | 用途 |
|-----------|---------|
| **rust-analyzer** | 代码补全、内联错误提示、重构 |
| **CodeLLDB** | 调试器（类似于 Visual Studio 调试器） |
| **Even Better TOML** | Cargo.toml 语法高亮 |
| **crates** | 在 Cargo.toml 中显示最新 crate 版本 |
| **Error Lens** | 内联错误/警告显示 |

***

如需深入探索本指南中提到的进阶主题，请参阅配套培训文档：

- **[Rust 模式](../../source-docs/RUST_PATTERNS.md)** — Pin 投影、自定义分配器、arena 模式、无锁数据结构以及进阶 unsafe 模式
- **[异步 Rust 培训](../../source-docs/ASYNC_RUST_TRAINING.md)** — 深入探索 tokio、异步取消安全性、流处理以及生产级异步架构
- **[面向 C++ 开发者的 Rust 培训](./RUST_TRAINING_FOR_CPP.md)** — 如果你的团队也有 C++ 经验，则很有用；涵盖移动语义映射、RAII 差异以及模板与泛型的对比
- **[面向 C 开发者的 Rust 培训](./RUST_TRAINING_FOR_C.md)** — 适用于互操作场景；涵盖 FFI 模式、嵌入式 Rust 调试以及 `no_std` 编程
