# 14. Crate 架构与 API 设计 🟡

> **学习目标：**
> - 模块布局约定和重导出策略
> - 公共 API 设计检查清单
> - 人体工学参数模式：`impl Into`、`AsRef`、`Cow`
> - "先解析，不要验证"：`TryFrom` 和验证类型
> - 功能标志、条件编译和工作区组织

## 模块布局约定

```text
my_crate/
├── Cargo.toml
├── src/
│   ├── lib.rs          # Crate 根 — 重导出和公共 API
│   ├── config.rs       # 功能模块
│   ├── parser/         # 复杂模块，包含子模块
│   │   ├── mod.rs      # 或 parser.rs（Rust 2018+）
│   │   ├── lexer.rs
│   │   └── ast.rs
│   ├── error.rs        # 错误类型
│   └── utils.rs        # 内部辅助函数（pub(crate)）
├── tests/
│   └── integration.rs  # 集成测试
├── benches/
│   └── perf.rs         # 基准测试
└── examples/
    └── basic.rs        # cargo run --example basic
```

```rust
// lib.rs — 通过重导出管理公共 API：
mod config;
mod error;
mod parser;
mod utils;

// 重导出用户需要的部分：
pub use config::Config;
pub use error::Error;
pub use parser::Parser;

// 公共类型在 crate 根级别 — 用户这样写：
// use my_crate::Config;
// 而不是：use my_crate::config::Config;
```

**可见性修饰符**：

| 修饰符 | 可见范围 |
|----------|-----------|
| `pub` | 所有地方 |
| `pub(crate)` | 仅当前 crate |
| `pub(super)` | 父模块 |
| `pub(in path)` | 特定祖先模块 |
| （无） | 当前模块及其子模块 |

### 公共 API 设计检查清单

1. **接收引用，返回所有权** — `fn process(input: &str) -> String`
2. **参数使用 `impl Trait`** — `fn read(r: impl Read)` 而不是 `fn read<R: Read>(r: R)`，签名更简洁
3. **返回 `Result`，不要 `panic!`** — 让调用者决定如何处理错误
4. **实现标准 trait** — `Debug`、`Display`、`Clone`、`Default`、`From`/`Into`
5. **使无效状态不可表示** — 使用类型状态和 newtype
6. **复杂配置使用构建器模式** — 如果字段是必需的，使用类型状态
7. **密封你不希望用户实现的 trait** — `pub trait Sealed: private::Sealed {}`
8. **为类型和函数标记 `#[must_use]`** — 防止重要 `Result`、守卫或值被无声丢弃。适用于任何忽略返回值几乎肯定是 bug 的类型：
   ```rust
   #[must_use = "dropping the guard immediately releases the lock"]
   pub struct LockGuard<'a, T> { /* ... */ }

   #[must_use]
   pub fn validate(input: &str) -> Result<ValidInput, ValidationError> { /* ... */ }
   ```

```rust
// 密封 trait 模式 — 用户可以使用但不能实现：
mod private {
    pub trait Sealed {}
}

pub trait DatabaseDriver: private::Sealed {
    fn connect(&self, url: &str) -> Connection;
}

// 只有这个 crate 内的类型可以实现 Sealed → 只有我们能实现 DatabaseDriver
pub struct PostgresDriver;
impl private::Sealed for PostgresDriver {}
impl DatabaseDriver for PostgresDriver {
    fn connect(&self, url: &str) -> Connection { /* ... */ }
}
```

> **`#[non_exhaustive]`** — 为公共枚举和结构体标记此属性，这样添加变体或字段不是破坏性变更。下游 crate 必须在 match 语句中使用通配符 arm（`_ =>`），并且不能使用结构体字面量语法构造该类型：
> ```rust
> #[non_exhaustive]
> pub enum DiagError {
>     Timeout,
>     HardwareFault,
>     // 在未来版本中添加新变体不是语义版本兼容突破。
> }
> ```

### 人体工学参数模式 — `impl Into`、`AsRef`、`Cow`

Rust 最有影响力的 API 模式之一是在函数参数中接受**最通用的类型**，这样调用者不需要在每个调用点重复 `.to_string()`、`&*s` 或 `.as_ref()`。这是 Rust 版本的"对你接受的要宽容"。

#### `impl Into<T>` — 接受任何可转换的类型

```rust
// ❌ 摩擦：调用者必须手动转换
fn connect(host: String, port: u16) -> Connection {
    // ...
}
connect("localhost".to_string(), 5432);  // 烦人的 .to_string()
connect(hostname.clone(), 5432);          // 如果已有 String 却不必要的 clone

// ✅ 人体工学：接受任何可以转换为 String 的类型
fn connect(host: impl Into<String>, port: u16) -> Connection {
    let host = host.into();  // 在函数内部转换一次
    // ...
}
connect("localhost", 5432);     // &str — 零摩擦
connect(hostname, 5432);        // String — 移动，无 clone
connect(arc_str, 5432);         // Arc<str> 如果实现了 From
```

这是可行的，因为 Rust 的 `From`/`Into` trait 对提供了一致的转换。当你接受 `impl Into<T>` 时，你是在说："给我任何知道如何变成 `T` 的东西"。

#### `AsRef<T>` — 作为引用借用

`AsRef<T>` 是 `Into<T>` 的借用对应物。当你只需要*读取*数据而不需要获取所有权时使用它：

```rust
use std::path::Path;

// ❌ 强制调用者转换为 &Path
fn file_exists(path: &Path) -> bool {
    path.exists()
}
file_exists(Path::new("/tmp/test.txt"));  // 尴尬

// ✅ 接受任何可以表现为 &Path 的类型
fn file_exists(path: impl AsRef<Path>) -> bool {
    path.as_ref().exists()
}
file_exists("/tmp/test.txt");                    // &str ✅
file_exists(String::from("/tmp/test.txt"));      // String ✅
file_exists(Path::new("/tmp/test.txt"));         // &Path ✅
file_exists(PathBuf::from("/tmp/test.txt"));     // PathBuf ✅

// 字符串参数的相同模式：
fn log_message(msg: impl AsRef<str>) {
    println!("[LOG] {}", msg.as_ref());
}
log_message("hello");                    // &str ✅
log_message(String::from("hello"));      // String ✅
```

#### `Cow<T>` — 写时克隆

`Cow<'a, T>`（Clone on Write）将内存分配延迟到需要修改时才进行。它持有借用的 `&T` 或有权的 `T::Owned`。这非常适合大多数调用不需要修改数据的场景：

```rust
use std::borrow::Cow;

/// 规范化诊断消息 — 仅在需要修改时分配。
fn normalize_message(msg: &str) -> Cow<'_, str> {
    if msg.contains('\t') || msg.contains('\r') {
        // 必须分配 — 我们需要修改内容
        Cow::Owned(msg.replace('\t', "    ").replace('\r', ""))
    } else {
        // 不分配 — 只需借用原始数据
        Cow::Borrowed(msg)
    }
}

// 大多数消息无需分配即可通过：
let clean = normalize_message("All tests passed");          // 借用 — 免费
let fixed = normalize_message("Error:\tfailed\r\n");        // 拥有 — 已分配

// Cow<str> 实现了 Deref<Target=str>，所以它像 &str 一样工作：
println!("{}", clean);
println!("{}", fixed.to_uppercase());
```

#### 快速参考：使用哪个

```text
你需要在函数内部获取数据的所有权吗？
├── 是 → impl Into<T>
│         "给我任何可以变成 T 的东西"
└── 否  → 你只需要读取它吗？
     ├── 是 → impl AsRef<T> 或 &T
     │         "给我任何我可以借用为 &T 的东西"
     └── 也许（有时需要修改？）
          └── Cow<'_, T>
              "尽可能借用，必须时再克隆"
```

| 模式 | 所有权 | 分配 | 使用场景 |
|---------|-----------|------------|-------------|
| `&str` | 借用 | 从不 | 简单字符串参数 |
| `impl AsRef<str>` | 借用 | 从不 | 接受 String、&str 等 — 只读 |
| `impl Into<String>` | 拥有 | 转换时 | 接受 &str、String — 将存储/拥有 |
| `Cow<'_, str>` | 两者皆可 | 仅修改时 | 通常不修改的处理 |
| `&[u8]` / `impl AsRef<[u8]>` | 借用 | 从不 | 字节导向 API |

> **`Borrow<T>` vs `AsRef<T>`**：两者都提供 `&T`，但 `Borrow<T>` 额外地保证原始形式和借用形式之间的 `Eq`、`Ord` 和 `Hash` 是**一致的**。这就是为什么 `HashMap<String, V>::get()` 接受 `&Q where String: Borrow<Q>` — 而不是 `AsRef`。当借用形式用作查找键时使用 `Borrow`；用于一般的"给我一个引用"参数时使用 `AsRef`。

#### 在 API 中组合转换

```rust
/// 一个使用人体工学参数的诊断 API：
pub struct DiagRunner {
    name: String,
    config_path: PathBuf,
}

impl DiagRunner {
    /// 接受任何类似字符串的类型作为名称，任何类似路径的类型作为配置。
    pub fn new(
        name: impl Into<String>,
        config_path: impl Into<PathBuf>,
    ) -> Self {
        DiagRunner {
            name: name.into(),
            config_path: config_path.into(),
        }
    }

    /// 接受任何 AsRef<str> 用于只读查找。
    pub fn get_result(&self, test_name: impl AsRef<str>) -> Option<&TestResult> {
        self.results.get(test_name.as_ref())
    }
}

// 所有这些都可以零摩擦地工作：
let runner = DiagRunner::new("GPU Diag", "/etc/diag_tool/config.json");
let runner = DiagRunner::new(format!("Diag-{}", node_id), config_path);
let runner = DiagRunner::new(name_string, path_buf);
```

***

## 案例研究：设计公共 Crate API — 前后对比

一个将字符串类型的内部 API 演进为符合人体工程学的、类型安全的公共 API 的真实例子。考虑一个配置解析 crate：

**之前**（字符串类型，容易误用）：

```rust
// ❌ 所有参数都是字符串 — 没有编译时验证
pub fn parse_config(path: &str, format: &str, strict: bool) -> Result<Config, String> {
    // 哪些格式是有效的？"json"？"JSON"？"Json"？
    // path 是文件路径还是 URL？
    // "strict" 到底是什么意思？
    todo!()
}
```

**之后**（类型安全、自文档化）：

```rust
use std::path::Path;

/// 支持的配置格式。
#[derive(Debug, Clone, Copy)]
#[non_exhaustive]  // 添加格式不会破坏下游
pub enum Format {
    Json,
    Toml,
    Yaml,
}

/// 控制解析严格性。
#[derive(Debug, Clone, Copy, Default)]
pub enum Strictness {
    /// 拒绝未知字段（库默认值）
    #[default]
    Strict,
    /// 忽略未知字段（对向前兼容的配置有用）
    Lenient,
}

pub fn parse_config(
    path: &Path,          // 类型强制：必须是文件系统路径
    format: Format,       // 枚举：不可能传递无效格式
    strictness: Strictness,  // 命名替代，不是裸露的 bool
) -> Result<Config, ConfigError> {
    todo!()
}
```

**改进点**：

| 方面 | 之前 | 之后 |
|--------|--------|-------|
| 格式验证 | 运行时字符串比较 | 编译时枚举 |
| 路径类型 | 原始 `&str`（可以是任何东西） | `&Path`（文件系统特定） |
| 严格性 | 神秘的 `bool` | 自文档化枚举 |
| 错误类型 | `String`（不透明） | `ConfigError`（结构化） |
| 可扩展性 | 破坏性变更 | `#[non_exhaustive]` |

> **经验法则**：如果你发现自己要对字符串值写 `match`，考虑用枚举替换该参数。如果参数是一个上下文不明显布尔值，使用双变体枚举代替。

***

### 先解析，不要验证 — `TryFrom` 和验证类型

"先解析，不要验证"是一个原则：**不要检查数据后再传递原始未检查形式——而是将其解析成一种只有在数据有效时才能存在的类型。** Rust 的 `TryFrom` trait 是实现这一点的标准工具。

#### 问题：没有强制执行的验证

```rust
// ❌ 验证后使用：检查后无法阻止使用无效值
fn process_port(port: u16) {
    if port == 0 || port > 65535 {
        panic!("Invalid port");           // 我们检查了，但是...
    }
    start_server(port);                    // 如果有人直接调用 start_server(0) 怎么办？
}

// ❌ 字符串类型：电子邮件只是一个 String — 任何垃圾都能通过
fn send_email(to: String, body: String) {
    // `to` 实际上是有效的电子邮件吗？我们不知道。
    // 有人可能传入 "not-an-email"，我们只在 SMTP 服务器才发现。
}
```

#### 解决方案：用 `TryFrom` 解析为验证过的 Newtype

```rust
use std::convert::TryFrom;
use std::fmt;

/// 验证过的 TCP 端口号（1–65535）。
/// 如果你有一个 `Port`，它就是有效的。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = PortError;

    fn try_from(value: u16) -> Result<Self, Self::Error> {
        if value == 0 {
            Err(PortError::Zero)
        } else {
            Ok(Port(value))
        }
    }
}

impl Port {
    pub fn get(&self) -> u16 { self.0 }
}

#[derive(Debug)]
pub enum PortError {
    Zero,
    InvalidFormat,
}

impl fmt::Display for PortError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            PortError::Zero => write!(f, "port must be non-zero"),
            PortError::InvalidFormat => write!(f, "invalid port format"),
        }
    }
}

impl std::error::Error for PortError {}

// 现在类型系统强制执行有效性：
fn start_server(port: Port) {
    // 不需要验证 — Port 只能通过 TryFrom 构造，
    // TryFrom 已经验证它是有效的。
    println!("Listening on port {}", port.get());
}

// 用法：
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let port = Port::try_from(8080)?;   // ✅ 在边界处验证一次
    start_server(port);                  // 下游无需重新验证

    let bad = Port::try_from(0);         // ❌ Err(PortError::Zero)
    Ok(())
}
```

#### 真实案例：验证过的 IPMI 地址

```rust
/// 验证过的 IPMI 从机地址（0x20–0xFE，仅偶数）。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct IpmiAddr(u8);

#[derive(Debug)]
pub enum IpmiAddrError {
    Odd(u8),
    OutOfRange(u8),
}

impl fmt::Display for IpmiAddrError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            IpmiAddrError::Odd(v) => write!(f, "IPMI address 0x{v:02X} must be even"),
            IpmiAddrError::OutOfRange(v) => {
                write!(f, "IPMI address 0x{v:02X} out of range (0x20..=0xFE)")
            }
        }
    }
}

impl TryFrom<u8> for IpmiAddr {
    type Error = IpmiAddrError;

    fn try_from(value: u8) -> Result<Self, Self::Error> {
        if value % 2 != 0 {
            Err(IpmiAddrError::Odd(value))
        } else if value < 0x20 || value > 0xFE {
            Err(IpmiAddrError::OutOfRange(value))
        } else {
            Ok(IpmiAddr(value))
        }
    }
}

impl IpmiAddr {
    pub fn get(&self) -> u8 { self.0 }
}

// 下游代码永远不需要重新检查：
fn send_ipmi_command(addr: IpmiAddr, cmd: u8, data: &[u8]) -> Result<Vec<u8>, IpmiError> {
    // addr.get() 保证是有效的、偶数的 IPMI 地址
    raw_ipmi_send(addr.get(), cmd, data)
}
```

#### 用 `FromStr` 解析字符串

对于常从文本解析的类型（CLI 参数、配置文件），实现 `FromStr`：

```rust
use std::str::FromStr;

impl FromStr for Port {
    type Err = PortError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        let n: u16 = s.parse().map_err(|_| PortError::InvalidFormat)?;
        Port::try_from(n)
    }
}

// 现在可以与 .parse() 一起使用：
let port: Port = "8080".parse()?;   // 一步验证

// 与 clap CLI 解析一起使用：
// #[derive(Parser)]
// struct Args {
//     #[arg(short, long)]
//     port: Port,   // clap 自动调用 FromStr
// }
```

#### `TryFrom` 链用于复杂验证

```rust
// 本例中的存根类型 — 生产环境中这些会在
// 各自模块中，有自己的 TryFrom 实现。
```

```rust
# struct Hostname(String);
# impl TryFrom<String> for Hostname {
#     type Error = String;
#     fn try_from(s: String) -> Result<Self, String> { Ok(Hostname(s)) }
# }
# struct Timeout(u64);
# impl TryFrom<u64> for Timeout {
#     type Error = String;
#     fn try_from(ms: u64) -> Result<Self, String> {
#         if ms == 0 { Err("timeout must be > 0".into()) } else { Ok(Timeout(ms)) }
#     }
# }
# struct RawConfig { host: String, port: u16, timeout_ms: u64 }
# #[derive(Debug)]
# enum ConfigError {
#     InvalidHost(String),
#     InvalidPort(PortError),
#     InvalidTimeout(String),
# }
# impl From<std::io::Error> for ConfigError {
#     fn from(e: std::io::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
# impl From<serde_json::Error> for ConfigError {
#     fn from(e: serde_json::Error) -> Self { ConfigError::InvalidHost(e.to_string()) }
# }
/// 只有所有字段都有效时才能存在的验证配置。
pub struct ValidConfig {
    pub host: Hostname,
    pub port: Port,
    pub timeout_ms: Timeout,
}

impl TryFrom<RawConfig> for ValidConfig {
    type Error = ConfigError;

    fn try_from(raw: RawConfig) -> Result<Self, Self::Error> {
        Ok(ValidConfig {
            host: Hostname::try_from(raw.host)
                .map_err(ConfigError::InvalidHost)?,
            port: Port::try_from(raw.port)
                .map_err(ConfigError::InvalidPort)?,
            timeout_ms: Timeout::try_from(raw.timeout_ms)
                .map_err(ConfigError::InvalidTimeout)?,
        })
    }
}

// 在边界处解析一次，在内部各处使用验证后的类型：
fn load_config(path: &str) -> Result<ValidConfig, ConfigError> {
    let raw: RawConfig = serde_json::from_str(&std::fs::read_to_string(path)?)?;
    ValidConfig::try_from(raw)  // 所有验证在此发生
}
```

#### 总结：验证 vs 解析

| 方法 | 数据已检查？ | 编译器强制有效性？ | 需要重新验证？ |
|----------|:---:|:---:|:---:|
| 运行时检查（if/assert） | ✅ | ❌ | 每个函数边界 |
| 验证过的 newtype + `TryFrom` | ✅ | ✅ | 从不 — 类型即证明 |

规则：**在边界处解析，在内部各处使用验证过的类型。** 原始字符串、整数和字节切片进入系统，通过 `TryFrom`/`FromStr` 解析成验证过的类型，从那时起类型系统保证它们是有效的。

### 功能标志和条件编译

```toml
```

# Cargo.toml
[features]
default = ["json"]          # 默认启用
json = ["dep:serde_json"]   # 启用 JSON 支持
xml = ["dep:quick-xml"]     # 启用 XML 支持
full = ["json", "xml"]      # 元功能：启用所有

[dependencies]
serde = "1"
serde_json = { version = "1", optional = true }
quick-xml = { version = "0.31", optional = true }

```rust
// 基于功能的条件编译：
#[cfg(feature = "json")]
pub fn to_json<T: serde::Serialize>(value: &T) -> String {
    serde_json::to_string(value).unwrap()
}

#[cfg(feature = "xml")]
pub fn to_xml<T: serde::Serialize>(value: &T) -> String {
    quick_xml::se::to_string(value).unwrap()
}

// 如果未启用所需功能则编译错误：
#[cfg(not(any(feature = "json", feature = "xml")))]
compile_error!("At least one format feature (json, xml) must be enabled");
```

**最佳实践**：
- 保持 `default` 功能最小化 — 用户可以选择加入
- 使用 `dep:` 语法（Rust 1.60+）处理可选依赖，避免创建隐式功能
- 在 README 和 crate 级文档中记录功能

### 工作区组织

对于大型项目，使用 Cargo 工作区共享依赖和构建产物：

```toml
```

# Root Cargo.toml
[workspace]
members = [
    "core",         # 共享类型和 trait
    "parser",       # 解析库
    "server",       # 二进制文件 — 主应用程序
    "client",       # 客户端库
    "cli",          # CLI 二进制文件
]

# 共享依赖版本：
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
tracing = "0.1"

# 在每个成员的 Cargo.toml 中：
# [dependencies]
# serde = { workspace = true }

```rust

**好处**：
```

- 单一 `Cargo.lock` — 所有 crate 使用相同的依赖版本
- `cargo test --workspace` 运行所有测试
- 共享构建缓存 — 编译一个 crate 惠及所有
- 组件之间干净的依赖边界

### `.cargo/config.toml`：项目级配置

`.cargo/config.toml` 文件（在工作区根目录或 `$HOME/.cargo/`）自定义 Cargo 行为，无需修改 `Cargo.toml`：

```toml
```

# .cargo/config.toml

# 此工作区的默认目标
[build]
target = "x86_64-unknown-linux-gnu"

# 自定义运行器 — 例如，通过 QEMU 运行交叉编译的二进制文件
[target.aarch64-unknown-linux-gnu]
runner = "qemu-aarch64-static"
linker = "aarch64-linux-gnu-gcc"

# Cargo 别名 — 自定义快捷命令
[alias]
xt = "test --workspace --release"        # cargo xt = 以 release 模式运行所有测试
ci = "clippy --workspace -- -D warnings" # cargo ci = 警告时 lint 出错
cov = "llvm-cov --workspace"             # cargo cov = 覆盖率（需要 cargo-llvm-cov）

# 构建脚本的环境变量
[env]
IPMI_LIB_PATH = "/usr/lib/bmc"

# 使用自定义注册表（用于内部包）
# [registries.internal]
# index = "https://gitlab.internal/crates/index"

```rust

常见配置模式：

```

| 设置 | 用途 | 示例 |
|---------|---------|---------|
| `[build] target` | 默认编译目标 | `x86_64-unknown-linux-musl` 用于静态构建 |
| `[target.X] runner` | 如何运行二进制文件 | `"qemu-aarch64-static"` 用于交叉编译 |
| `[target.X] linker` | 使用哪个链接器 | `"aarch64-linux-gnu-gcc"` |
| `[alias]` | 自定义 `cargo` 子命令 | `xt = "test --workspace"` |
| `[env]` | 构建时环境变量 | 库路径、功能切换 |
| `[net] offline` | 阻止网络访问 | `true` 用于气隙构建 |

### 编译时环境变量：`env!()` 和 `option_env!()`

Rust 可以在编译时将环境变量嵌入二进制文件 — 用于版本字符串、构建元数据和配置：

```rust
// env!() — 如果变量缺失则在编译时 panic
const VERSION: &str = env!("CARGO_PKG_VERSION"); // 来自 Cargo.toml 的 "0.1.0"
const PKG_NAME: &str = env!("CARGO_PKG_NAME");   // 来自 Cargo.toml 的 crate 名称

// option_env!() — 返回 Option<&str>，缺失时不会 panic
const BUILD_SHA: Option<&str> = option_env!("GIT_SHA");
const BUILD_TIME: Option<&str> = option_env!("BUILD_TIMESTAMP");

fn print_version() {
    println!("{PKG_NAME} v{VERSION}");
    if let Some(sha) = BUILD_SHA {
        println!("  commit: {sha}");
    }
    if let Some(time) = BUILD_TIME {
        println!("  built:  {time}");
    }
}
```

Cargo 自动设置许多有用的环境变量：

| 变量 | 值 | 使用场景 |
|----------|-------|----------|
| `CARGO_PKG_VERSION` | `"1.2.3"` | 版本报告 |
| `CARGO_PKG_NAME` | `"diag_tool"` | 二进制标识 |
| `CARGO_PKG_AUTHORS` | 来自 `Cargo.toml` | 关于/帮助文本 |
| `CARGO_MANIFEST_DIR` | `Cargo.toml` 的绝对路径 | 定位测试数据文件 |
| `OUT_DIR` | 构建输出目录 | `build.rs` 代码生成目标 |
| `TARGET` | 目标三元组 | `build.rs` 中的平台特定逻辑 |

你可以从 `build.rs` 设置自定义环境变量：
```rust
// build.rs
fn main() {
    println!("cargo::rustc-env=GIT_SHA={}", git_sha());
    println!("cargo::rustc-env=BUILD_TIMESTAMP={}", timestamp());
}
```

### `cfg_attr`：条件属性

`cfg_attr` 仅在条件为真时应用属性。这比 `#[cfg()]` 更有针对性，`#[cfg()]` 包含/排除整个条目：

```rust
// 仅当启用了 "serde" 功能时才派生 Serialize：
#[cfg_attr(feature = "serde", derive(serde::Serialize, serde::Deserialize))]
#[derive(Debug, Clone)]
pub struct DiagResult {
    pub fc: u32,
    pub passed: bool,
    pub message: String,
}
// 没有 "serde" 功能：根本不需要 serde 依赖
// 有 "serde" 功能：DiagResult 是可序列化的

// 测试的条件属性：
#[cfg_attr(test, derive(PartialEq))]  // 仅在测试构建中派生 PartialEq
pub struct LargeStruct { /* ... */ }

// 平台特定的函数属性：
#[cfg_attr(target_os = "linux", link_name = "ioctl")]
#[cfg_attr(target_os = "freebsd", link_name = "__ioctl")]
extern "C" fn platform_ioctl(fd: i32, request: u64) -> i32;
```

| 模式 | 作用 |
|---------|-------------|
| `#[cfg(feature = "x")]` | 包含/排除整个条目 |
| `#[cfg_attr(feature = "x", derive(Foo))]` | 仅当功能 "x" 启用时添加 `derive(Foo)` |
| `#[cfg_attr(test, allow(unused))]` | 仅在测试构建中抑制警告 |
| `#[cfg_attr(doc, doc = "...")]` | 仅在 `cargo doc` 中可见的文档 |

### `cargo deny` 和 `cargo audit`：供应链安全

```bash
```

# 安装安全审计工具
cargo install cargo-deny
cargo install cargo-audit

# 检查依赖中的已知漏洞
cargo audit

# 综合检查：许可证、禁止事项、建议、来源
cargo deny check

```rust

使用 `deny.toml` 配置 `cargo deny`（在工作区根目录）：

```

```toml
```

# deny.toml
[advisories]
vulnerability = "deny"      # 已知漏洞时失败
unmaintained = "warn"        # 未维护 crate 时警告

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-2-Clause", "BSD-3-Clause"]
deny = ["GPL-3.0"]          # 拒绝 copyleft 许可证

[bans]
multiple-versions = "warn"  # 同一 crate 多个版本时警告
deny = [

```rust
    { name = "openssl" },   # 强制使用 rustls
]

[sources]
allow-git = []              # 生产环境无 git 依赖
```

| 工具 | 用途 | 运行时间 |
|------|---------|---------|
| `cargo audit` | 检查依赖中的已知 CVE | CI 管道、发布前 |
| `cargo deny check` | 许可证、禁止事项、建议、来源 | CI 管道 |
| `cargo deny check licenses` | 仅许可证合规性 | 开源前 |
| `cargo deny check bans` | 阻止特定 crate | 强制执行架构决策 |

### 文档测试：文档内的测试

Rust 文档注释（`///`）可以包含**作为测试编译和运行的代码块**：

```rust
/// 从字符串解析诊断故障代码。
///
/// # 示例
///
/// ```
/// use my_crate::parse_fc;
///
/// let fc = parse_fc("FC:12345").unwrap();
/// assert_eq!(fc, 12345);
/// ```
///
/// 无效输入返回错误：
///
/// ```
/// use my_crate::parse_fc;
///
/// assert!(parse_fc("not-a-fc").is_err());
/// ```
pub fn parse_fc(input: &str) -> Result<u32, ParseError> {
    input.strip_prefix("FC:")
        .ok_or(ParseError::MissingPrefix)?
        .parse()
        .map_err(ParseError::InvalidNumber)
}
```

```bash
cargo test --doc  # 仅运行文档测试
cargo test        # 运行单元 + 集成 + 文档测试
```

**模块级文档**使用文件顶部的 `//!`：

```rust
//! # 诊断框架
//!
//! 此 crate 提供核心诊断执行引擎。
//! 它支持运行诊断测试、收集结果，
//! 并通过 IPMI 向 BMC 报告。
//!
//! ## 快速开始
//!
//! ```no_run
//! use diag_framework::Framework;
//!
//! let mut fw = Framework::new("config.json")?;
//! fw.run_all_tests()?;
//! ```
```

### 使用 Criterion 进行基准测试

> **完整覆盖**：关于完整的 `criterion` 设置、API 示例和与 `cargo bench` 的对比表，请参阅第 13 章（测试和基准测试模式）中的[使用 criterion 进行基准测试](ch13-testing-and-benchmarking-patterns.md#benchmarking-with-criterion)部分。
> 以下是架构特定用法的快速参考。

对 crate 的公共 API 进行基准测试时，将基准测试放在 `benches/` 中，并保持它们专注于热路径 — 通常是解析器、序列化器或验证边界：

```bash
cargo bench                  # 运行所有基准测试
cargo bench -- parse_config  # 运行特定基准测试
# 结果在 target/criterion/ 中，包含 HTML 报告
```

> **关键要点 — 架构与 API 设计**
> - 接受最通用的类型（`impl Into`、`impl AsRef`、`Cow`）；返回最具体的类型
> - 先解析，不要验证：使用 `TryFrom` 创建默认有效的类型
> - 公共枚举上的 `#[non_exhaustive]` 在添加变体时防止破坏性变更
> - `#[must_use]` 捕获重要值的无声丢弃

> **另请参阅：** [第 9 章 — 错误处理](ch09-error-handling-patterns.md) 了解公共 API 中的错误类型设计。[第 13 章 — 测试](ch13-testing-and-benchmarking-patterns.md) 了解如何测试 crate 的公共 API。

---

### 练习：Crate API 重构 ★★（约 30 分钟）

将以下"字符串类型"API 重构为使用 `TryFrom`、newtype 和构建器模式的 API：

```rust,ignore
// 之前：容易误用
fn create_server(host: &str, port: &str, max_conn: &str) -> Server { ... }
```

设计一个 `ServerConfig`，包含验证类型 `Host`、`Port`（1–65535）和 `MaxConnections`（1–10000），在解析时拒绝无效值。

<details>
<summary>🔑 解决方案</summary>

```rust
#[derive(Debug, Clone)]
struct Host(String);

impl TryFrom<&str> for Host {
    type Error = String;
    fn try_from(s: &str) -> Result<Self, String> {
        if s.is_empty() { return Err("host cannot be empty".into()); }
        if s.contains(' ') { return Err("host cannot contain spaces".into()); }
        Ok(Host(s.to_string()))
    }
}

#[derive(Debug, Clone, Copy)]
struct Port(u16);

impl TryFrom<u16> for Port {
    type Error = String;
    fn try_from(p: u16) -> Result<Self, String> {
        if p == 0 { return Err("port must be >= 1".into()); }
        Ok(Port(p))
    }
}

#[derive(Debug, Clone, Copy)]
struct MaxConnections(u32);

impl TryFrom<u32> for MaxConnections {
    type Error = String;
    fn try_from(n: u32) -> Result<Self, String> {
        if n == 0 || n > 10_000 {
            return Err(format!("max_connections must be 1–10000, got {n}"));
        }
        Ok(MaxConnections(n))
    }
}

#[derive(Debug)]
struct ServerConfig {
    host: Host,
    port: Port,
    max_connections: MaxConnections,
}

impl ServerConfig {
    fn new(host: Host, port: Port, max_connections: MaxConnections) -> Self {
        ServerConfig { host, port, max_connections }
    }
}

fn main() {
    let config = ServerConfig::new(
        Host::try_from("localhost").unwrap(),
        Port::try_from(8080).unwrap(),
        MaxConnections::try_from(100).unwrap(),
    );
    println!("{config:?}");

    // 无效值在解析时被捕获：
    assert!(Host::try_from("").is_err());
    assert!(Port::try_from(0).is_err());
    assert!(MaxConnections::try_from(99999).is_err());
}
```

</details>

***
