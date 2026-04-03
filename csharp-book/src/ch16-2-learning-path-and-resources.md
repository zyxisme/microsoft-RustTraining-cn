## 学习路径与后续步骤

> **你将学到：** 结构化的学习路线图（第1-2周、第1-3个月及以上）、推荐的书籍和资源、C#开发者的常见陷阱（所有权混淆、与借用检查器的斗争），以及使用 `tracing` 与 `ILogger` 的结构化可观测性。
>
> **难度：** 🟢 初级

### 近期后续步骤（第1-2周）
1. **设置你的环境**
   - 通过 [rustup.rs](https://rustup.rs/) 安装 Rust
   - 配置 VS Code 的 rust-analyzer 扩展
   - 创建你的第一个 `cargo new hello_world` 项目

2. **掌握基础知识**
   - 通过简单练习练习所有权
   - 编写具有不同参数类型的函数（`&str`、`String`、`&mut`）
   - 实现基本的结构体和方法

3. **错误处理练习**
   - 将 C# 的 try-catch 代码转换为 Result 模式
   - 练习使用 `?` 操作符和 `match` 语句
   - 实现自定义错误类型

### 中期目标（第1-2个月）
1. **集合和迭代器**
   - 掌握 `Vec<T>`、`HashMap<K,V>` 和 `HashSet<T>`
   - 学习迭代器方法：`map`、`filter`、`collect`、`fold`
   - 练习使用 `for` 循环与迭代器链

2. **trait 和泛型**
   - 实现常见的 trait：`Debug`、`Clone`、`PartialEq`
   - 编写泛型函数和结构体
   - 理解 trait 约束和 where 子句

3. **项目结构**
   - 将代码组织成模块
   - 理解 `pub` 可见性
   - 使用 crates.io 上的外部 crate

### 高级主题（第3个月及以上）
1. **并发**
   - 学习 `Send` 和 `Sync` trait
   - 使用 `std::thread` 进行基本并行处理
   - 探索 `tokio` 进行异步编程

2. **内存管理**
   - 理解 `Rc<T>` 和 `Arc<T>` 用于共享所有权
   - 学习何时使用 `Box<T>` 进行堆分配
   - 掌握复杂场景下的生命周期

3. **实际项目**
   - 使用 `clap` 构建 CLI 工具
   - 使用 `axum` 或 `warp` 创建 Web API
   - 编写库并发布到 crates.io

### 推荐学习资源

#### 书籍
- **"The Rust Programming Language"**（免费在线）—— 官方书籍
- **"Rust by Example"**（免费在线）—— 实践示例
- **"Programming Rust"** by Jim Blandy —— 深入的技术覆盖

#### 在线资源
- [Rust Playground](https://play.rust-lang.org/) —— 在浏览器中尝试代码
- [Rustlings](https://github.com/rust-lang/rustlings) —— 交互式练习
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/) —— 实践示例

#### 练习项目
1. **命令行计算器** —— 练习枚举和模式匹配
2. **文件整理器** —— 练习文件系统和错误处理
3. **JSON 处理器** —— 学习 serde 和数据转换
4. **HTTP 服务器** —— 理解异步编程和网络
5. **数据库库** —— 掌握 trait、泛型和错误处理

### C# 开发者的常见陷阱

#### 所有权混淆
```rust
// 不要：错误地使用已移动的值
fn wrong_way() {
    let s = String::from("hello");
    takes_ownership(s);
    // println!("{}", s); // 错误：s 已被移动
}

// 应该：必要时使用引用或克隆
fn right_way() {
    let s = String::from("hello");
    borrows_string(&s);
    println!("{}", s); // OK：s 仍然在这里被拥有
}

fn takes_ownership(s: String) { /* s 在这里被移动 */ }
fn borrows_string(s: &str) { /* s 在这里被借用 */ }
```

#### 与借用检查器斗争
```rust
// 不要：多个可变引用
fn wrong_borrowing() {
    let mut v = vec![1, 2, 3];
    let r1 = &mut v;
    // let r2 = &mut v; // 错误：不能多次可变借用
}

// 应该：限制可变借用的作用域
fn right_borrowing() {
    let mut v = vec![1, 2, 3];
    {
        let r1 = &mut v;
        r1.push(4);
    } // r1 在这里超出作用域

    let r2 = &mut v; // OK：没有其他可变借用存在
    r2.push(5);
}
```

#### 期望空值
```rust
// 不要：期望类似 null 的行为
fn no_null_in_rust() {
    // let s: String = null; // Rust 中没有 null！
}

// 应该：显式使用 Option<T>
fn use_option_instead() {
    let maybe_string: Option<String> = None;

    match maybe_string {
        Some(s) => println!("Got string: {}", s),
        None => println!("No string available"),
    }
}
```

### 最后提示

1. **拥抱编译器** —— Rust 的编译器错误是有帮助的，而不是敌对的
2. **从小开始** —— 从简单的程序开始，逐渐增加复杂度
3. **阅读别人的代码** —— 学习 GitHub 上流行的 crate
4. **寻求帮助** —— Rust 社区是友好且乐于助人的
5. **定期练习** —— Rust 的概念会随着练习变得自然

记住：Rust 有学习曲线，但它带来的内存安全、性能和无畏并发是值得的。最初看起来受限的所有权系统会成为编写正确、高效程序的强大工具。

---

**恭喜！** 你现在已经有了从 C# 过渡到 Rust 的坚实基础。从简单的项目开始，对学习过程保持耐心，逐步向更复杂的应用程序迈进。Rust 的安全性和性能优势使初始的学习投入是值得的。


<!-- ch16.2a: Structured Observability with tracing -->
## 结构化可观测性：`tracing` vs ILogger 和 Serilog

C# 开发者习惯于通过 `ILogger`、`Serilog` 或 `NLog` 进行**结构化日志记录** —— 日志消息携带类型化的键值属性。Rust 的 `log` crate 提供了基本的分级日志，但 **`tracing`** 是结构化可观测性的生产标准，支持 span、异步感知和分布式追踪。

### 为什么选择 `tracing` 而不是 `log`

| 特性 | `log` crate | `tracing` crate | C# 等价物 |
|---------|------------|-----------------|----------------|
| 分级消息 | ✅ `info!()`、`error!()` | ✅ `info!()`、`error!()` | `ILogger.LogInformation()` |
| 结构化字段 | ❌ 仅字符串插值 | ✅ 类型化的键值字段 | Serilog `Log.Information("{User}", user)` |
| Span（作用域上下文） | ❌ | ✅ `#[instrument]`、`span!()` | `ILogger.BeginScope()` |
| 异步感知 | ❌ 在 `.await` 上丢失上下文 | ✅ Span 跨 `.await` 跟随 | `Activity` / `DiagnosticSource` |
| 分布式追踪 | ❌ | ✅ OpenTelemetry 集成 | `System.Diagnostics.Activity` |
| 多种输出格式 | 基本 | JSON、pretty、compact、OTLP | Serilog sinks |

### 入门

```toml
# Cargo.toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
```

### 基本用法：结构化日志记录
```csharp
// C# Serilog
Log.Information("Processing order {OrderId} for {Customer}, total {Total:C}",
    orderId, customer.Name, order.Total);
// Output: Processing order 12345 for Alice, total $99.95
// JSON:  {"OrderId": 12345, "Customer": "Alice", "Total": 99.95, ...}
```

```rust
use tracing::{info, warn, error, debug, instrument};

// 结构化字段 —— 类型化的，而不是字符串插值的
info!(order_id = 12345, customer = "Alice", total = 99.95,
      "Processing order");
// Output: INFO Processing order order_id=12345 customer="Alice" total=99.95
// JSON:  {"order_id": 12345, "customer": "Alice", "total": 99.95, ...}

// 动态值
let order_id = 12345;
info!(order_id, "Order received");  // 字段名 = 变量名简写

// 条件字段
if let Some(promo) = promo_code {
    info!(order_id, promo_code = %promo, "Promo applied");
    //                        ^ % 表示使用 Display 格式化
    //                        ? 表示使用 Debug 格式化
}
```

### Span：异步代码的杀手级特性

Span 是作用域上下文，在函数调用和 `.await` 点之间携带字段 —— 类似于 `ILogger.BeginScope()`，但它是异步安全的。

```csharp
// C# — Activity / BeginScope
using var activity = new Activity("ProcessOrder").Start();
activity.SetTag("order_id", orderId);

using (_logger.BeginScope(new Dictionary<string, object> { ["OrderId"] = orderId }))
{
    _logger.LogInformation("Starting processing");
    await ProcessPaymentAsync();
    _logger.LogInformation("Payment complete");  // OrderId still in scope
}
```

```rust
use tracing::{info, instrument, Instrument};

// #[instrument] 自动创建带有函数参数的 span 作为字段
#[instrument(skip(db), fields(customer_name))]
async fn process_order(order_id: u64, db: &Database) -> Result<(), AppError> {
    let order = db.get_order(order_id).await?;

    // 动态地向当前 span 添加字段
    tracing::Span::current().record("customer_name", &order.customer_name.as_str());

    info!("Starting processing");
    process_payment(&order).await?;        // span 上下文跨 .await 保留！
    info!(items = order.items.len(), "Payment complete");
    Ok(())
}
// 此函数内部的每条日志消息自动包含：
//   order_id=12345 customer_name="Alice"
// 即使在嵌套的异步调用中！

// 手动创建 span（类似于 BeginScope）
async fn batch_process(orders: Vec<u64>, db: &Database) {
    for order_id in orders {
        let span = tracing::info_span!("process_order", order_id);

        // .instrument(span) 将 span 附加到 future
        process_order(order_id, db)
            .instrument(span)
            .await
            .unwrap_or_else(|e| error!("Failed: {e}"));
    }
}
```

### Subscriber 配置（类似于 Serilog Sinks）

```rust
use tracing_subscriber::{fmt, EnvFilter, layer::SubscriberExt, util::SubscriberInitExt};

fn init_tracing() {
    // 开发环境：人类可读的、带颜色的输出
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| "my_app=debug,tower_http=info".into()))
        .with(fmt::layer().pretty())  // 带颜色的、缩进的 span
        .init();
}

fn init_tracing_production() {
    // 生产环境：用于日志聚合的 JSON 输出（类似于 Serilog JSON sink）
    tracing_subscriber::registry()
        .with(EnvFilter::new("my_app=info"))
        .with(fmt::layer().json())  // 结构化 JSON
        .init();
    // Output: {"timestamp":"...","level":"INFO","fields":{"order_id":123},...}
}
```

```bash
# 通过环境变量控制日志级别（类似于 Serilog MinimumLevel）
RUST_LOG=my_app=debug,hyper=warn cargo run
RUST_LOG=trace cargo run  # 记录所有内容
```

### Serilog → tracing 迁移速查表

| Serilog / ILogger | tracing | 备注 |
|-------------------|---------|-------|
| `Log.Information("{Key}", val)` | `info!(key = val, "message")` | 字段是类型化的，而不是插值的 |
| `Log.ForContext("Key", val)` | `span.record("key", val)` | 向当前 span 添加字段 |
| `using BeginScope(...)` | `#[instrument]` 或 `info_span!()` | `#[instrument]` 自动处理 |
| `.WriteTo.Console()` | `fmt::layer()` | 人类可读 |
| `.WriteTo.Seq()` / `.File()` | `fmt::layer().json()` + 文件重定向 | 或使用 `tracing-appender` |
| `.Enrich.WithProperty()` | `span!(Level::INFO, "name", key = val)` | Span 字段 |
| `LogEventLevel.Debug` | `tracing::Level::DEBUG` | 相同概念 |
| `{@Object}` 解构 | `field = ?value`（Debug）或 `%value`（Display） | `?` = Debug，`%` = Display |

### OpenTelemetry 集成

```toml
# 用于分布式追踪（类似于 System.Diagnostics + OTLP exporter）
[dependencies]
tracing-opentelemetry = "0.22"
opentelemetry = "0.21"
opentelemetry-otlp = "0.14"
```

```rust
// 在控制台输出的基础上添加 OpenTelemetry 层
use tracing_opentelemetry::OpenTelemetryLayer;

fn init_otel() {
    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(opentelemetry_otlp::new_exporter().tonic())
        .install_batch(opentelemetry_sdk::runtime::Tokio)
        .expect("Failed to create OTLP tracer");

    tracing_subscriber::registry()
        .with(OpenTelemetryLayer::new(tracer))  // 发送 span 到 Jaeger/Tempo
        .with(fmt::layer())                      // 同时打印到控制台
        .init();
}
// 现在 #[instrument] span 自动成为分布式追踪！
```

***

