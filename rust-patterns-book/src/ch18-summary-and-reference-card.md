## 快速参考卡

### 模式决策指南

```text
需要对原语进行类型安全？
└── Newtype 模式（第 3 章）

需要编译时状态强制？
└── 类型状态模式（第 3 章）

需要"标签"但无运行时数据？
└── PhantomData（第 4 章）

需要打破 Rc/Arc 引用循环？
└── Weak<T> / sync::Weak<T>（第 8 章）

需要等待条件而不忙等待？
└── Condvar + Mutex（第 6 章）

需要处理"N 种类型之一"？
├── 已知封闭集合 → 枚举
├── 开放集合，热路径 → 泛型
├── 开放集合，冷路径 → dyn Trait
└── 完全未知类型 → Any + TypeId（第 2 章）

需要跨线程共享状态？
├── 简单计数器/标志 → 原子类型
├── 短临界区 → Mutex
├── 读密集型 → RwLock
├── 惰性一次性初始化 → OnceLock / LazyLock（第 6 章）
└── 复杂状态 → Actor + 通道

需要并行化计算？
├── 集合处理 → rayon::par_iter
├── 后台任务 → thread::spawn
└── 借用本地数据 → thread::scope

需要 async I/O 或并发网络？
├── 基础 → tokio + async/await（第 15 章）
└── 高级（流、中间件）→ 参见 Async Rust Training

需要错误处理？
├── 库 → thiserror（#[derive(Error)]）
└── 应用程序 → anyhow（Result<T>）

需要防止值被移动？
└── Pin<T>（第 8 章）— Futures、自引用类型所需
```

### Trait 约束速查表

| 约束 | 含义 |
|-------|---------|
| `T: Clone` | 可以复制 |
| `T: Send` | 可以移动到另一个线程 |
| `T: Sync` | `&T` 可以在线程间共享 |
| `T: 'static` | 不包含非静态引用 |
| `T: Sized` | 编译时知道大小（默认） |
| `T: ?Sized` | 大小可能未知（`[T]`、`dyn Trait`） |
| `T: Unpin` | 固定后可以安全移动 |
| `T: Default` | 有默认值 |
| `T: Into<U>` | 可以转换为 `U` |
| `T: AsRef<U>` | 可以借用为 `&U` |
| `T: Deref<Target = U>` | 自动解引用为 `&U` |
| `F: Fn(A) -> B` | 可调用，借用状态不可变 |
| `F: FnMut(A) -> B` | 可调用，可能修改状态 |
| `F: FnOnce(A) -> B` | 仅可调用一次，可能消费状态 |

### 生命周期省略规则

编译器在三种情况下自动插入生命周期（所以你不必手动写）：

```rust
// 规则 1：每个引用参数获得自己的生命周期
// fn foo(x: &str, y: &str)  →  fn foo<'a, 'b>(x: &'a str, y: &'b str)

// 规则 2：如果恰好有一个输入生命周期，它用于所有输出
// fn foo(x: &str) -> &str   →  fn foo<'a>(x: &'a str) -> &'a str

// 规则 3：如果一个参数是 &self 或 &mut self，其生命周期被使用
// fn foo(&self, x: &str) -> &str  →  fn foo<'a>(&'a self, x: &str) -> &'a str
```

**当你必须写显式生命周期时**：
- 多个输入引用和一个引用输出（编译器无法猜测哪个输入）
- 持有引用的结构体字段：`struct Ref<'a> { data: &'a str }`
- 当你需要没有借用引用的数据时的 `'static` 约束

### 常用派生 Trait

```rust
#[derive(
    Debug,          // {:?} 格式化
    Clone,          // .clone()
    Copy,           // 隐式复制（仅用于简单类型）
    PartialEq, Eq,  // == 比较
    PartialOrd, Ord, // < > 比较和排序
    Hash,           // HashMap/HashSet 键
    Default,        // Type::default()
)]
struct MyType { /* ... */ }
```

### 模块可见性快速参考

```text
pub           → 所有地方可见
pub(crate)    → crate 内可见
pub(super)    → 对父模块可见
pub(in path)  → 在特定路径内可见
（无）        → 对当前模块及其子模块私有
```

### 进阶阅读

| 资源 | 原因 |
|----------|-----|
| [Rust Design Patterns](https://rust-unofficial.github.io/patterns/) | 惯用模式和反模式目录 |
| [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/) | 完善的公共 API 官方检查清单 |
| [Rust Atomics and Locks](https://marabos.nl/atomics/) | Mara Bos 对并发原语的深入探讨 |
| [The Rustonomicon](https://doc.rust-lang.org/nomicon/) | unsafe Rust 和黑暗角落的官方指南 |
| [Error Handling in Rust](https://blog.burntsushi.net/rust-error-handling/) | Andrew Gallant 的全面指南 |
| [Jon Gjengset — Crust of Rust series](https://www.youtube.com/playlist?list=PLqbS7AVVErFiWDOAVrPt7aYmnuuOLYvOa) | 迭代器、生命周期、通道等深度探讨 |
| [Effective Rust](https://www.lurklurk.org/effective-rust/) | 改进 Rust 代码的 35 种具体方法 |

***

*Rust Patterns & Engineering How-Tos 完*
