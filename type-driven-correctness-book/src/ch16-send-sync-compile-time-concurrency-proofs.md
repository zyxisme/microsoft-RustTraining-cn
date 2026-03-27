# Send & Sync — 编译时并发证明 🟠

> **你将学到：** Rust 的 `Send` 和 `Sync` 自动 trait 如何将编译器变成并发审计员——在编译时证明哪些类型可以跨线程边界，哪些可以共享，零运行时成本。
>
> **交叉引用：** [ch04](ch04-capability-tokens-zero-cost-proof-of-aut.md)（能力令牌），[ch09](ch09-phantom-types-for-resource-tracking.md)（幽灵类型），[ch15](ch15-const-fn-compile-time-correctness-proofs.md)（const fn 证明）

## 问题：没有安全网的并发访问

在系统编程中，外设、共享缓冲区和全局状态从多个上下文访问——主循环、中断处理程序、DMA 回调和工作线程。在 C 中，编译器完全不提供强制执行：

```c
/* 共享传感器缓冲区 — 从主循环和 ISR 访问 */
volatile uint32_t sensor_buf[64];
volatile uint32_t buf_index = 0;

void SENSOR_IRQHandler(void) {
    sensor_buf[buf_index++] = read_sensor();  /* 竞态：buf_index 读 + 写 */
}

void process_sensors(void) {
    for (uint32_t i = 0; i < buf_index; i++) {  /* buf_index 在循环中变化 */
        process(sensor_buf[i]);                   /* 数据在读取中途被覆盖 */
    }
    buf_index = 0;                                /* ISR 在这两行之间触发 */
}
```

`volatile` 关键字防止编译器优化掉读取，但它对数据竞争 **什么也不做**。两个上下文可以同时读写 `buf_index`，产生撕裂值、丢失更新或缓冲区超限。相同的问题出现在 `pthread_mutex_t` — 编译器会愉快地让你忘记加锁：

```c
pthread_mutex_t lock;
int shared_counter;

void increment(void) {
    shared_counter++;  /* 哎呀 — 忘了 pthread_mutex_lock(&lock) */
}
```

**每个并发 bug 都在运行时被发现** — 通常是在负载下、生产中，而且是间歇性的。

## Send 和 Sync 证明什么

Rust 定义了两个标记 trait，编译器自动派生它们：

| Trait | 证明 | 非正式含义 |
|-------|-------|-------------------|
| `Send` | 类型 `T` 的值可以安全地 **移动** 到另一个线程 | "这可以跨线程边界" |
| `Sync` | **共享引用** `&T` 可以安全地被多个线程使用 | "这可以跨线程共享" |

这些是 **自动 trait** — 编译器通过检查每个字段来派生它们。结构体是 `Send` 如果所有字段都是 `Send`。结构体是 `Sync` 如果所有字段都是 `Sync`。如果任何字段选择退出，整个结构体就选择退出。不需要注解，没有运行时开销 — 证明是结构性的。

```mermaid
flowchart TD
    STRUCT["你的结构体"]
    INSPECT["编译器检查<br/>每个字段"]
    ALL_SEND{"所有字段<br/>Send?"}
    ALL_SYNC{"所有字段<br/>Sync?"}
    SEND_YES["Send ✅<br/><i>可以跨线程边界</i>"]
    SEND_NO["!Send ❌<br/><i>限制在一个线程</i>"]
    SYNC_YES["Sync ✅<br/><i>可跨线程共享</i>"]
    SYNC_NO["!Sync ❌<br/><i>无并发引用</i>"]

    STRUCT --> INSPECT
    INSPECT --> ALL_SEND
    INSPECT --> ALL_SYNC
    ALL_SEND -->|Yes| SEND_YES
    ALL_SEND -->|"Any field !Send<br/>(e.g., Rc, *const T)"| SEND_NO
    ALL_SYNC -->|Yes| SYNC_YES
    ALL_SYNC -->|"Any field !Sync<br/>(e.g., Cell, RefCell)"| SYNC_NO

    style SEND_YES fill:#c8e6c9,color:#000
    style SYNC_YES fill:#c8e6c9,color:#000
    style SEND_NO fill:#ffcdd2,color:#000
    style SYNC_NO fill:#ffcdd2,color:#000
```

> **编译器就是审计员。** 在 C 中，线程安全注解存在于注释和头文档中 — 建议性的，从不强制执行。在 Rust 中，`Send` 和 `Sync` 从类型本身的结构派生。添加一个 `Cell<f32>` 字段自动使包含结构体变为 `!Sync`。不需要程序员动作，无法忘记。

两个 trait 通过一个关键恒等式关联：

> **`T` 是 `Sync` 当且仅当 `&T` 是 `Send`。**

这很直观：如果共享引用可以安全发送到另一个线程，那么底层类型对并发读取是安全的。

### 选择退出的类型

某些类型故意是 `!Send` 或 `!Sync`：

| 类型 | Send | Sync | 为什么 |
|------|:----:|:----:|-----|
| `u32`, `String`, `Vec<T>` | ✅ | ✅ | 无内部可变性，无原始指针 |
| `Cell<T>`, `RefCell<T>` | ✅ | ❌ | 无同步的内部可变性 |
| `Rc<T>` | ❌ | ❌ | 引用计数不是原子的 |
| `*const T`, `*mut T` | ❌ | ❌ | 原始指针没有安全保证 |
| `Arc<T>`（其中 `T: Send + Sync`） | ✅ | ✅ | 原子引用计数 |
| `Mutex<T>`（其中 `T: Send`） | ✅ | ✅ | 锁序列化所有访问 |

表中每个 ❌ 都是 **编译时不变量**。你不能意外将 `Rc` 发送到另一个线程 — 编译器拒绝它。

## !Send 外设句柄

在嵌入式系统中，外设寄存器块位于固定内存地址，应该只从单个执行上下文访问。原始指针固有地是 `!Send` 和 `!Sync`，所以包装一个自动使包含类型退出两个 trait：

```rust
/// 内存映射 UART 外设的句柄。
/// 原始指针使其自动 !Send 和 !Sync。
pub struct Uart {
    regs: *const u32,
}

impl Uart {
    pub fn new(base: usize) -> Self {
        Self { regs: base as *const u32 }
    }

    pub fn write_byte(&self, byte: u8) {
        // 真实固件：unsafe { write_volatile(self.regs.add(DATA_OFFSET), byte as u32) }
        println!("UART TX: {:#04X}", byte);
    }
}

fn main() {
    let uart = Uart::new(0x4000_1000);
    uart.write_byte(b'A');  // ✅ 在创建线程上使用

    // ❌ 不会编译：Uart 是 !Send
    // std::thread::spawn(move || {
    //     uart.write_byte(b'B');
    // });
}
```

注释掉的 `thread::spawn` 会产生：

```text
error[E0277]: `*const u32` cannot be sent between threads safely
   |
   |     std::thread::spawn(move || {
   |     ^^^^^^^^^^^^^^^^^^ within `Uart`, the trait `Send` is not
   |                        implemented for `*const u32`
```

**没有原始指针？使用 `PhantomData`。** 有时类型没有原始指针但仍应限制在一个线程 — 例如，从 C 库获得的文件描述符索引或句柄：

```rust
use std::marker::PhantomData;

/// 来自 C 库的不透明句柄。PhantomData<*const ()> 使其
/// !Send + !Sync 即使内部 fd 只是普通整数。
pub struct LibHandle {
    fd: i32,
    _not_send: PhantomData<*const ()>,
}

impl LibHandle {
    pub fn open(path: &str) -> Self {
        let _ = path;
        Self { fd: 42, _not_send: PhantomData }
    }

    pub fn fd(&self) -> i32 { self.fd }
}

fn main() {
    let handle = LibHandle::open("/dev/sensor0");
    println!("fd = {}", handle.fd());

    // ❌ 不会编译：LibHandle 是 !Send
    // std::thread::spawn(move || { let _ = handle.fd(); });
}
```

这是 C 的编译时等价物："请阅读说这个句柄不是线程安全的文档。"在 Rust 中，编译器强制执行它。

## Mutex 将 !Sync 转换为 Sync

`Cell<T>` 和 `RefCell<T>` 提供无同步的内部可变性 — 所以它们是 `!Sync`。但有时你确实需要跨线程共享可变状态。`Mutex<T>` 添加了缺失的同步，编译器认可这一点：

> **如果 `T: Send`，则 `Mutex<T>: Send + Sync`。**

锁序列化所有访问，所以 `!Sync` 内部类型变得可以安全共享。编译器从结构上证明这一点 — 没有"程序员记得加锁了吗"的运行时检查：

```rust
use std::sync::{Arc, Mutex};
use std::cell::Cell;

/// 使用 Cell 进行内部可变的传感器缓存。
/// Cell<u32> 是 !Sync — 不能直接跨线程共享。
struct SensorCache {
    last_reading: Cell<u32>,
    reading_count: Cell<u32>,
}

fn main() {
    // Mutex 使 SensorCache 可以安全共享 — 编译器证明它
    let cache = Arc::new(Mutex::new(SensorCache {
        last_reading: Cell::new(0),
        reading_count: Cell::new(0),
    }));

    let handles: Vec<_> = (0..4).map(|i| {
        let c = Arc::clone(&cache);
        std::thread::spawn(move || {
            let guard = c.lock().unwrap();  // 访问前必须加锁
            guard.last_reading.set(i * 10);
            guard.reading_count.set(guard.reading_count.get() + 1);
        })
    }).collect();

    for h in handles { h.join().unwrap(); }

    let guard = cache.lock().unwrap();
    println!("Last reading: {}", guard.last_reading.get());
    println!("Total reads:  {}", guard.reading_count.get());
}
```

对比 C 版本：`pthread_mutex_lock` 是一个运行时调用，程序员可以忘记。在这里，类型系统使得不可能不通过 `Mutex` 访问 `SensorCache`。证明是结构性的 — 唯一运行时成本是锁本身。

> **`Mutex` 不仅仅同步 — 它证明同步。** `Mutex::lock()` 返回一个 `MutexGuard`，它 `Deref` 到 `&T`。无法不通过锁获取内部数据的引用。API 使得"忘记加锁"在结构上无法表示。

## 函数边界作为定理

`std::thread::spawn` 有这个签名：

```rust,ignore
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
```

`Send + 'static` 边界不仅仅是一个实现细节 — 它是一个 **定理**：

> "传递给 `spawn` 的任何闭包和返回值在编译时被证明可以安全地在另一个线程上运行，没有悬空引用。"

你可以将相同模式应用到你自己的 API：

```rust
use std::sync::mpsc;

/// 在后台线程上运行任务并返回其结果。
/// 边界证明：闭包及其结果是线程安全的。
fn run_on_background<F, T>(task: F) -> T
where
    F: FnOnce() -> T + Send + 'static,
    T: Send + 'static,
{
    let (tx, rx) = mpsc::channel();
    std::thread::spawn(move || {
        let _ = tx.send(task());
    });
    rx.recv().expect("background task panicked")
}

fn main() {
    // ✅ u32 是 Send，闭包不捕获非 Send 的东西
    let result = run_on_background(|| 6 * 7);
    println!("Result: {result}");

    // ✅ String 是 Send
    let greeting = run_on_background(|| String::from("hello from background"));
    println!("{greeting}");

    // ❌ 不会编译：Rc 是 !Send
    // use std::rc::Rc;
    // let data = Rc::new(42);
    // run_on_background(move || *data);
}
```

取消 `Rc` 示例的注释会产生精确诊断：

```text
error[E0277]: `Rc<i32>` cannot be sent between threads safely
   --> src/main.rs
    |
    |     run_on_background(move || *data);
    |     ^^^^^^^^^^^^^^^^^^ `Rc<i32>` cannot be sent between threads safely
    |
note: required by a bound in `run_on_background`
    |
    |     F: FnOnce() -> T + Send + 'static,
    |                        ^^^^ required by this bound
```

编译器将违反追溯到精确的边界 — 并告诉程序员 *为什么*。对比 C 的 `pthread_create`：

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                   void *(*start_routine)(void *), void *arg);
```

`void *arg` 接受任何东西 — 线程安全或不安全。C 编译器无法区分非原子引用计数和普通整数。Rust 的 trait 边界在类型级别做出区分。

## 何时使用 Send/Sync 证明

| 场景 | 方法 |
|----------|----------|
| 包装原始指针的外设句柄 | 自动 `!Send + !Sync` — 无事可做 |
| 来自 C 库的句柄（整数 fd/handle） | 添加 `PhantomData<*const ()>` 实现 `!Send + !Sync` |
| 锁后的共享配置 | `Arc<Mutex<T>>` — 编译器证明访问是安全的 |
| 跨线程消息传递 | `mpsc::channel` — `Send` 边界自动强制执行 |
| 任务生成器或线程池 API | 在签名中要求 `F: Send + 'static` |
| 单线程资源（例如 GPU 上下文） | `PhantomData<*const ()>` 防止共享 |
| 类型应该是 `Send` 但包含原始指针 | `unsafe impl Send` 带文档化的安全理由 |

### 成本摘要

| 什么 | 运行时成本 |
|------|:------:|
| `Send` / `Sync` 自动派生 | 仅编译时 — 0 字节 |
| `PhantomData<*const ()>` 字段 | 零大小 — 优化掉 |
| `!Send` / `!Sync` 强制执行 | 仅编译时 — 无运行时检查 |
| `F: Send + 'static` 函数边界 | 单态化 — 静态分派，无 boxing |
| `Mutex<T>` 锁 | 运行时锁（共享变更不可避免） |
| `Arc<T>` 引用计数 | 原子增量/减量（共享所有权不可避免） |

前四行是 **零成本** — 它们只存在于类型系统中，在编译后消失。`Mutex` 和 `Arc` 携带不可避免的运行时成本，但这些成本是任何正确的并发程序必须支付的 *最小值* — Rust 只是确保你支付它们。

## 练习：DMA 传输守卫

设计一个 `DmaTransfer<T>`，在 DMA 传输进行期间持有缓冲区。要求：

1. `DmaTransfer` 必须是 `!Send` — DMA 控制器使用与本核心内存总线绑定的物理地址
2. `DmaTransfer` 必须是 `!Sync` — 并发读取而 DMA 写入会看到撕裂数据
3. 提供一个 `wait()` 方法 **消耗** 守卫并返回缓冲区 — 所有权证明传输完成
4. 缓冲区类型 `T` 必须实现 `DmaSafe` 标记 trait

<details>
<summary>解答</summary>

```rust
use std::marker::PhantomData;

/// 用于可用作 DMA 缓冲区的类型的标记 trait。
/// 真实固件：类型必须是 repr(C) 且无填充。
trait DmaSafe {}

impl DmaSafe for [u8; 64] {}
impl DmaSafe for [u8; 256] {}

/// 表示进行中 DMA 传输的守卫。
/// !Send + !Sync：不能发送到另一个线程或共享。
pub struct DmaTransfer<T: DmaSafe> {
    buffer: T,
    channel: u8,
    _no_send_sync: PhantomData<*const ()>,
}

impl<T: DmaSafe> DmaTransfer<T> {
    /// 启动 DMA 传输。缓冲区被消耗 — 其他人不能触碰它。
    pub fn start(buffer: T, channel: u8) -> Self {
        // 真实固件：配置 DMA 通道，设置源/目的，启动传输
        println!("DMA channel {} started", channel);
        Self {
            buffer,
            channel,
            _no_send_sync: PhantomData,
        }
    }

    /// 等待传输完成并返回缓冲区。
    /// 消耗 self — 守卫在此之后不再存在。
    pub fn wait(self) -> T {
        // 真实固件：轮询 DMA 状态寄存器直到完成
        println!("DMA channel {} complete", self.channel);
        self.buffer
    }
}

fn main() {
    let buf = [0u8; 64];

    // 启动传输 — buf 被移动到守卫
    let transfer = DmaTransfer::start(buf, 2);

    // ❌ buf 不再可访问 — 所有权防止 DMA 期间使用
    // println!("{:?}", buf);

    // ❌ 不会编译：DmaTransfer 是 !Send
    // std::thread::spawn(move || { transfer.wait(); });

    // ✅ 在原始线程上等待，获取缓冲区
    let buf = transfer.wait();
    println!("Buffer recovered: {} bytes", buf.len());
}
```

</details>

```mermaid
flowchart TB
    subgraph compiler["编译时 — 自动派生证明"]
        direction TB
        SEND["Send<br/>✅ 可以安全跨线程移动"]
        SYNC["Sync<br/>✅ 可以安全共享引用"]
        NOTSEND["!Send<br/>❌ 限制在一个线程"]
        NOTSYNC["!Sync<br/>❌ 无并发共享"]
    end

    subgraph types["类型分类"]
        direction TB
        PLAIN["原语、String、Vec<br/>Send + Sync"]
        CELL["Cell、RefCell<br/>Send + !Sync"]
        RC["Rc、原始指针<br/>!Send + !Sync"]
        MUTEX["Mutex&lt;T&gt;<br/>恢复 Sync"]
        ARC["Arc&lt;T&gt;<br/>共享所有权 + Send"]
    end

    subgraph runtime["运行时"]
        SAFE["线程安全访问<br/>无数据竞争<br/>无忘记加锁"]
    end

    SEND --> PLAIN
    NOTSYNC --> CELL
    NOTSEND --> RC
    CELL --> MUTEX --> SAFE
    RC --> ARC --> SAFE
    PLAIN --> SAFE

    style SEND fill:#c8e6c9,color:#000
    style SYNC fill:#c8e6c9,color:#000
    style NOTSEND fill:#ffcdd2,color:#000
    style NOTSYNC fill:#ffcdd2,color:#000
    style PLAIN fill:#c8e6c9,color:#000
    style CELL fill:#fff3e0,color:#000
    style RC fill:#ffcdd2,color:#000
    style MUTEX fill:#e1f5fe,color:#000
    style ARC fill:#e1f5fe,color:#000
    style SAFE fill:#c8e6c9,color:#000
```

## 关键要点

1. **`Send` 和 `Sync` 是关于并发安全的编译时证明** — 编译器通过检查每个字段从结构上派生它们。不需要注解，不需要运行时成本，不需要选择加入。

2. **原始指针自动选择退出** — 任何包含 `*const T` 或 `*mut T` 的类型变为 `!Send + !Sync`。这使得外设句柄自然线程受限。

3. **`PhantomData<*const ()>` 是显式选择退出** — 当类型没有原始指针但仍应线程受限（C 库句柄、GPU 上下文）时，幽灵字段完成工作。

4. **`Mutex<T>` 用证明恢复 `Sync`** — 编译器从结构上证明所有访问都通过锁。与 C 的 `pthread_mutex_t` 不同，你不能忘记加锁。

5. **函数边界是定理** — spawner 签名中的 `F: Send + 'static` 是编译时证明义务：每个调用点必须证明其闭包是线程安全的。对比 C 的 `void *arg` 接受任何东西。

6. **该模式补充所有其他正确性技术** — 类型状态证明协议排序，幽灵类型证明权限，`const fn` 证明值不变量，而 `Send`/`Sync` 证明并发安全。它们一起覆盖完整的正确性表面。
