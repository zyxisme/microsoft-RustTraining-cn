## 异步编程：C# Task vs Rust Future

> **你将学到：** Rust 的惰性 `Future` 与 C# 的即时 `Task`，执行器模型（tokio），
> 通过 `Drop` + `select!` 实现取消 vs `CancellationToken`，以及并发请求的真实场景模式。
>
> **难度：** 🔴 高级

C# 开发者对 `async`/`await` 非常熟悉。Rust 使用相同的关键字，但执行模型有着本质的不同。

### 执行器模型

```csharp
// C# — 运行时提供了内置的线程池和任务调度器
// async/await 开箱即用
public async Task<string> FetchDataAsync(string url)
{
    using var client = new HttpClient();
    return await client.GetStringAsync(url);  // 由 .NET 线程池调度
}
// .NET 管理线程池、任务调度和同步上下文
```

```rust
// Rust — 没有内置的异步运行时。你需要选择一个执行器。
// 最流行的是 tokio。
async fn fetch_data(url: &str) -> Result<String, reqwest::Error> {
    let body = reqwest::get(url).await?.text().await?;
    Ok(body)
}

// 你必须有一个运行时来执行异步代码：
#[tokio::main]  // 这个宏设置了 tokio 运行时
async fn main() {
    let data = fetch_data("https://example.com").await.unwrap();
    println!("{}", &data[..100]);
}
```

### Future 与 Task

| | C# `Task<T>` | Rust `Future<Output = T>` |
|---|---|---|
| **执行** | 创建后立即开始 | **惰性** — 直到 `.await` 才执行 |
| **运行时** | 内置（CLR 线程池） | 外部（tokio、async-std 等） |
| **取消** | `CancellationToken` | 丢弃 `Future`（或 `tokio::select!`） |
| **状态机** | 编译器生成 | 编译器生成 |
| **大小** | 堆分配 | 栈分配直到装箱 |

```rust
// 重要：Future 在 Rust 中是惰性的！
async fn compute() -> i32 { println!("Computing!"); 42 }

let future = compute();  // 什么都没有打印！Future 还未被轮询。
let result = future.await; // 现在才打印 "Computing!"
```

```csharp
// C# Task 立即开始！
var task = ComputeAsync();  // "Computing!" 立即打印
var result = await task;    // 只是等待完成
```

### 取消：CancellationToken vs Drop / select!

```csharp
// C# — 通过 CancellationToken 进行协作取消
public async Task ProcessAsync(CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        await Task.Delay(1000, ct);  // 如果取消则抛出异常
        DoWork();
    }
}

var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await ProcessAsync(cts.Token);
```

```rust
// Rust — 通过丢弃 future 或使用 tokio::select! 来取消
use tokio::time::{sleep, Duration};

async fn process() {
    loop {
        sleep(Duration::from_secs(1)).await;
        do_work();
    }
}

// 使用 select! 的超时模式
async fn run_with_timeout() {
    tokio::select! {
        _ = process() => { println!("Completed"); }
        _ = sleep(Duration::from_secs(5)) => { println!("Timed out!"); }
    }
    // 当 select! 选择超时分支时，process() future 被丢弃
    // — 自动清理，无需 CancellationToken
}
```

### 真实场景模式：带超时的并发请求

```csharp
// C# — 带超时的并发 HTTP 请求
public async Task<string[]> FetchAllAsync(string[] urls, CancellationToken ct)
{
    var tasks = urls.Select(url => httpClient.GetStringAsync(url, ct));
    return await Task.WhenAll(tasks);
}
```

```rust
// Rust — 使用 tokio::join! 或 futures::join_all 实现并发请求
use futures::future::join_all;

async fn fetch_all(urls: &[&str]) -> Vec<Result<String, reqwest::Error>> {
    let futures = urls.iter().map(|url| reqwest::get(*url));
    let responses = join_all(futures).await;

    let mut results = Vec::new();
    for resp in responses {
        results.push(resp?.text().await);
    }
    results
}

// 带超时：
async fn fetch_all_with_timeout(urls: &[&str]) -> Result<Vec<String>, &'static str> {
    tokio::time::timeout(
        Duration::from_secs(10),
        async {
            let futures: Vec<_> = urls.iter()
                .map(|url| async { reqwest::get(*url).await?.text().await })
                .collect();
            let results = join_all(futures).await;
            results.into_iter().collect::<Result<Vec<_>, _>>()
        }
    )
    .await
    .map_err(|_| "Request timed out")?
    .map_err(|_| "Request failed")
}
```

<details>
<summary><strong>🏋️ 练习：异步超时模式</strong>（点击展开）</summary>

**挑战**：编写一个异步函数，从两个 URL 并发获取数据，返回最先响应的那个，并取消另一个。（这相当于 C# 中的 `Task.WhenAny`。）

<details>
<summary>🔑 解答</summary>

```rust
use tokio::time::{sleep, Duration};

// 模拟异步获取
async fn fetch(url: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("Response from {url}")
}

async fn fetch_first(url1: &str, url2: &str) -> String {
    tokio::select! {
        result = fetch(url1, 200) => {
            println!("URL 1 won");
            result
        }
        result = fetch(url2, 500) => {
            println!("URL 2 won");
            result
        }
    }
    // 失败分支的 future 会自动被丢弃（取消）
}

#[tokio::main]
async fn main() {
    let result = fetch_first("https://fast.api", "https://slow.api").await;
    println!("{result}");
}
```

**关键要点**：`tokio::select!` 是 Rust 中 `Task.WhenAny` 的等价物 — 它让多个 future 竞争，第一个完成时结束，并丢弃（取消）其余的。

</details>
</details>

### 使用 `tokio::spawn` 生成独立任务

在 C# 中，`Task.Run` 启动独立于调用者运行的工作。Rust 中的等价物是 `tokio::spawn`：

```rust
use tokio::task;

async fn background_work() {
    // 独立运行 — 即使调用者的 future 被丢弃
    let handle = task::spawn(async {
        tokio::time::sleep(Duration::from_secs(2)).await;
        42
    });

    // 在派生的任务运行的同时做其他工作...
    println!("Doing other work");

    // 需要时等待结果
    let result = handle.await.unwrap(); // 42
}
```

```csharp
// C# 等价代码
var task = Task.Run(async () => {
    await Task.Delay(2000);
    return 42;
});
// 做其他工作...
var result = await task;
```

**关键区别**：普通的 `async {}` 块是惰性的 — 直到被 await 才会执行。`tokio::spawn` 立即在运行时启动它，就像 C# 的 `Task.Run`。

### Pin：为什么 Rust 异步有一个 C# 没有的概念

C# 开发者从不遇到 `Pin` — CLR 的垃圾回收器自由地移动对象并自动更新所有引用。Rust 没有 GC。当编译器将 `async fn` 转换为状态机时，该结构体可能包含指向自身字段的内部指针。移动该结构体会使这些指针失效。

`Pin<T>` 是一个包装器，表示：**"这个值在内存中不会被移动。"**

```rust
// 你会在这些上下文中看到 Pin：
trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
    //           ^^^^^^^^^^^^^^ 被固定 — 内部引用保持有效
}

// 从 trait 返回装箱的 future：
fn make_future() -> Pin<Box<dyn Future<Output = i32> + Send>> {
    Box::pin(async { 42 })
}
```

**实际上，你几乎不需要自己写 `Pin`。** `async fn` 和 `.await` 语法会处理它。你只会在以下场景中遇到它：
- 编译器错误消息（按照建议操作）
- `tokio::select!`（使用 `pin!()` 宏）
- 返回 `dyn Future` 的 trait 方法（使用 `Box::pin(async { ... })`）

> **想要深入了解？** 配套的 [Async Rust Training](../../async-book/src/ch04-pin-and-unpin.md) 完整涵盖了 Pin、Unpin、自引用结构和结构化 pinning。

***

