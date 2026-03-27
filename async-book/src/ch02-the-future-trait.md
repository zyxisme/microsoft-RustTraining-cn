# 2. Future Trait 🟡

> **你将学到：**
> - `Future` trait：`Output`、`poll()`、`Context`、`Waker`
> - Waker 如何告诉执行器"再次轮询我"
> - 契约：不调用 `wake()` = 程序静默挂起
> - 手动实现一个真实的 future（`Delay`）

## Future 的结构

异步 Rust 中的一切最终都实现这个 trait：

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),   // Future 已完成，值为 T
    Pending,    // Future 还未就绪 — 稍后回调
}
```

就这样。`Future` 是任何可以被*轮询*的东西——被问到"你完成了吗？"——然后回答"是的，这是结果"或"还没有，我准备好时会唤醒你"。

### Output、poll()、Context、Waker

```mermaid
sequenceDiagram
    participant E as Executor
    participant F as Future
    participant R as Resource (I/O)

    E->>F: poll(cx)
    F->>R: Check: is data ready?
    R-->>F: Not yet
    F->>R: Register waker from cx
    F-->>E: Poll::Pending

    Note over R: ... time passes, data arrives ...

    R->>E: waker.wake() — "I'm ready!"
    E->>F: poll(cx) — try again
    F->>R: Check: is data ready?
    R-->>F: Yes! Here's the data
    F-->>E: Poll::Ready(data)
```

让我们分解每个部分：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// 一个立即返回 42 的 future
struct Ready42;

impl Future for Ready42 {
    type Output = i32; // Future 最终产生的值

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<i32> {
        Poll::Ready(42) // 总是就绪 — 不需要等待
    }
}
```

**组件说明**：

- **`Output`** — Future 完成时产生的值的类型
- **`poll()``** — 由执行器调用以检查进度；返回 `Ready(value)` 或 `Pending`
- **`Pin<&mut Self>`** — 确保 future 不会被移动到内存中的其他位置（我们将在第 4 章介绍原因）
- **`Context`** — 携带 `Waker`，以便 Future 可以在准备好继续时通知执行器

### Waker 契约

`Waker` 是回调机制。当一个 future 返回 `Pending` 时，它*必须*安排稍后调用 `waker.wake()`——否则执行器永远不会再次轮询它，程序就会挂起。

```rust
use std::task::{Context, Poll, Waker};
use std::pin::Pin;
use std::future::Future;
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

/// 一个在延迟后完成的 future（玩具实现）
struct Delay {
    completed: Arc<Mutex<bool>>,
    waker_stored: Arc<Mutex<Option<Waker>>>,
    duration: Duration,
    started: bool,
}

impl Delay {
    fn new(duration: Duration) -> Self {
        Delay {
            completed: Arc::new(Mutex::new(false)),
            waker_stored: Arc::new(Mutex::new(None)),
            duration,
            started: false,
        }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // 检查是否已完成
        if *self.completed.lock().unwrap() {
            return Poll::Ready(());
        }

        // 存储 waker 以便后台线程可以唤醒我们
        *self.waker_stored.lock().unwrap() = Some(cx.waker().clone());

        // 首次轮询时启动后台计时器
        if !self.started {
            self.started = true;
            let completed = Arc::clone(&self.completed);
            let waker = Arc::clone(&self.waker_stored);
            let duration = self.duration;

            thread::spawn(move || {
                thread::sleep(duration);
                *completed.lock().unwrap() = true;

                // 关键：唤醒执行器以便再次轮询我们
                if let Some(w) = waker.lock().unwrap().take() {
                    w.wake(); // "嘿执行器，我准备好了——再次轮询我！"
                }
            });
        }

        Poll::Pending // 还未完成
    }
}
```

> **关键洞察**：在 C# 中，TaskScheduler 自动处理唤醒。在 Rust 中，**你**（或你使用的 I/O 库）负责调用 `waker.wake()`。忘记这一点，你的程序就会静默挂起。

### 练习：实现一个 CountdownFuture

<details>
<summary>🏋️ 练习（点击展开）</summary>

**挑战**：实现一个 `CountdownFuture`，从 N倒数到 0，每次被轮询时打印当前计数。当到达 0 时，以 `Ready("Liftoff!")` 完成。

*提示*：future 需要存储当前计数并在每次轮询时递减。记得总是重新注册 waker！

<details>
<summary>🔑 答案</summary>

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct CountdownFuture {
    count: u32,
}

impl CountdownFuture {
    fn new(start: u32) -> Self {
        CountdownFuture { count: start }
    }
}

impl Future for CountdownFuture {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.count == 0 {
            println!("Liftoff!");
            Poll::Ready("Liftoff!")
        } else {
            println!("{}...", self.count);
            self.count -= 1;
            cx.waker().wake_by_ref(); // 立即调度重新轮询
            Poll::Pending
        }
    }
}
```

**关键要点**：这个 future 每次计数被轮询一次。每次返回 `Pending` 时，它会立即唤醒自己以便再次被轮询。在生产环境中，你会使用定时器而不是忙等待。

</details>
</details>

> **核心要点 — Future Trait**
> - `Future::poll()` 返回 `Poll::Ready(value)` 或 `Poll::Pending`
> - Future 在返回 `Pending` 前必须注册一个 `Waker`——执行器用它来知道何时重新轮询
> - `Pin<&mut Self>` 保证 future 不会被移动到内存中（这对自引用状态机是必需的——见第 4 章）
> - 异步 Rust 中的一切——`async fn`、`.await`、组合器——都建立在这一个 trait 之上

> **另见：** [第 3 章 — Poll 如何工作](ch03-how-poll-works.md) 了解执行器循环，[第 6 章 — 手动构建 Futures](ch06-building-futures-by-hand.md) 了解更多复杂实现

***
