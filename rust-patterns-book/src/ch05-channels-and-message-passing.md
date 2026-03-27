# 5. 通道与消息传递 🟢

> **学习内容：**
> - `std::sync::mpsc` 基础以及何时应该升级到 crossbeam-channel
> - 使用 `select!` 进行多源消息处理
> - 有界通道 vs 无界通道以及背压策略
> - 使用 Actor 模式封装并发状态

## std::sync::mpsc — 标准通道

Rust 标准库提供了一个多生产者、单消费者的通道：

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    // Create a channel: tx (transmitter) and rx (receiver)
    let (tx, rx) = mpsc::channel();

    // Spawn a producer thread
    let tx1 = tx.clone(); // Clone for multiple producers
    thread::spawn(move || {
        for i in 0..5 {
            tx1.send(format!("producer-1: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(100));
        }
    });

    // Second producer
    thread::spawn(move || {
        for i in 0..5 {
            tx.send(format!("producer-2: msg {i}")).unwrap();
            thread::sleep(Duration::from_millis(150));
        }
    });

    // Consumer: receive all messages
    for msg in rx {
        // rx iterator ends when ALL senders are dropped
        println!("Received: {msg}");
    }
    println!("All producers done.");
}
```

**关键特性**：
- 默认情况下是**无界的**（如果消费者速度较慢，可能会撑爆内存）
- `mpsc::sync_channel(N)` 创建一个带有背压的**有界**通道
- `rx.recv()` 会阻塞当前线程直到收到消息
- `rx.try_recv()` 如果没有消息则立即返回 `Err(TryRecvError::Empty)`
- 当所有 `Sender` 被丢弃时通道关闭

```rust
// Bounded channel with backpressure:
let (tx, rx) = mpsc::sync_channel(10); // Buffer of 10 messages

thread::spawn(move || {
    for i in 0..1000 {
        tx.send(i).unwrap(); // BLOCKS if buffer is full — natural backpressure
    }
});
```

### crossbeam-channel — 生产级主力

`crossbeam-channel` 是生产环境中事实上的标准。它比 `std::sync::mpsc` 更快，并支持多消费者（`mpmc`）：

```rust,ignore
// Cargo.toml:
//   [dependencies]
//   crossbeam-channel = "0.5"
use crossbeam_channel::{bounded, unbounded, select, Sender, Receiver};
use std::thread;
use std::time::Duration;

fn main() {
    // Bounded MPMC channel
    let (tx, rx) = bounded::<String>(100);

    // Multiple producers
    for id in 0..4 {
        let tx = tx.clone();
        thread::spawn(move || {
            for i in 0..10 {
                tx.send(format!("worker-{id}: item-{i}")).unwrap();
            }
        });
    }
    drop(tx); // Drop the original sender so the channel can close

    // Multiple consumers (not possible with std::sync::mpsc!)
    let rx2 = rx.clone();
    let consumer1 = thread::spawn(move || {
        while let Ok(msg) = rx.recv() {
            println!("[consumer-1] {msg}");
        }
    });
    let consumer2 = thread::spawn(move || {
        while let Ok(msg) = rx2.recv() {
            println!("[consumer-2] {msg}");
        }
    });

    consumer1.join().unwrap();
    consumer2.join().unwrap();
}
```

### 通道选择（select!）

同时监听多个通道——就像 Go 中的 `select`：

```rust,ignore
use crossbeam_channel::{bounded, tick, after, select};
use std::time::Duration;

fn main() {
    let (work_tx, work_rx) = bounded::<String>(10);
    let ticker = tick(Duration::from_secs(1));        // Periodic tick
    let deadline = after(Duration::from_secs(10));     // One-shot timeout

    // Producer
    let tx = work_tx.clone();
    std::thread::spawn(move || {
        for i in 0..100 {
            tx.send(format!("job-{i}")).unwrap();
            std::thread::sleep(Duration::from_millis(500));
        }
    });
    drop(work_tx);

    loop {
        select! {
            recv(work_rx) -> msg => {
                match msg {
                    Ok(job) => println!("Processing: {job}"),
                    Err(_) => {
                        println!("Work channel closed");
                        break;
                    }
                }
            },
            recv(ticker) -> _ => {
                println!("Tick — heartbeat");
            },
            recv(deadline) -> _ => {
                println!("Deadline reached — shutting down");
                break;
            },
        }
    }
}
```

> **Go 对比**：这与 Go 的通道 `select` 语句完全相同。
> crossbeam 的 `select!` 宏随机化顺序以防止饥饿，这与 Go 一样。

### 有界 vs 无界以及背压

| 类型 | 满时的行为 | 内存 | 使用场景 |
|------|-----------|------|----------|
| **无界** | 从不阻塞（堆增长） | 无界 ⚠️ | 罕见——仅当生产者比消费者慢时 |
| **有界** | `send()` 阻塞直到有空间 | 固定 | 生产环境默认——防止 OOM |
| **Rendezvous**（bounded(0)） | `send()` 阻塞直到接收者就绪 | 无 | 同步/交接 |

```rust
// Rendezvous channel — zero capacity, direct handoff
let (tx, rx) = crossbeam_channel::bounded(0);
// tx.send(x) blocks until rx.recv() is called, and vice versa.
// This synchronizes the two threads precisely.
```

**原则**：在生产环境中始终使用有界通道，除非你能证明生产者永远不会超过消费者。

### 使用通道的 Actor 模式

Actor 模式使用通道来序列化对可变状态的访问——不需要互斥锁：

```rust
use std::sync::mpsc;
use std::thread;

// Messages the actor can receive
enum CounterMsg {
    Increment,
    Decrement,
    Get(mpsc::Sender<i64>), // Reply channel
}

struct CounterActor {
    count: i64,
    rx: mpsc::Receiver<CounterMsg>,
}

impl CounterActor {
    fn new(rx: mpsc::Receiver<CounterMsg>) -> Self {
        CounterActor { count: 0, rx }
    }

    fn run(mut self) {
        while let Ok(msg) = self.rx.recv() {
            match msg {
                CounterMsg::Increment => self.count += 1,
                CounterMsg::Decrement => self.count -= 1,
                CounterMsg::Get(reply) => {
                    let _ = reply.send(self.count);
                }
            }
        }
    }
}

// Actor handle — cheap to clone, Send + Sync
#[derive(Clone)]
struct Counter {
    tx: mpsc::Sender<CounterMsg>,
}

impl Counter {
    fn spawn() -> Self {
        let (tx, rx) = mpsc::channel();
        thread::spawn(move || CounterActor::new(rx).run());
        Counter { tx }
    }

    fn increment(&self) { let _ = self.tx.send(CounterMsg::Increment); }
    fn decrement(&self) { let _ = self.tx.send(CounterMsg::Decrement); }

    fn get(&self) -> i64 {
        let (reply_tx, reply_rx) = mpsc::channel();
        self.tx.send(CounterMsg::Get(reply_tx)).unwrap();
        reply_rx.recv().unwrap()
    }
}

fn main() {
    let counter = Counter::spawn();

    // Multiple threads can safely use the counter — no mutex!
    let handles: Vec<_> = (0..10).map(|_| {
        let counter = counter.clone();
        thread::spawn(move || {
            for _ in 0..1000 {
                counter.increment();
            }
        })
    }).collect();

    for h in handles { h.join().unwrap(); }
    println!("Final count: {}", counter.get()); // 10000
}
```

> **何时使用 Actor vs 互斥锁**：当状态有复杂的不可变条件、操作耗时较长，或者你想要序列化访问而无需考虑锁顺序时，Actor 非常好。互斥锁对于短的关键段更简单。

> **关键要点 — 通道**
> - `crossbeam-channel` 是生产级主力——比 `std::sync::mpsc` 更快、功能更丰富
> - `select!` 用声明式通道选择替代了复杂的多源轮询
> - 有界通道提供自然背压；无界通道有 OOM 风险

> **另见：** [第 6 章 — 并发](ch06-concurrency-vs-parallelism-vs-threads.md) 线程、互斥锁和共享状态。[第 15 章 — 异步](ch15-asyncawait-essentials.md) 异步通道（`tokio::sync::mpsc`）。

---

### 练习：基于通道的工作池 ★★★（约 45 分钟）

使用通道构建一个工作池，其中：
- 分发器通过通道发送 `Job` 结构体
- N 个 worker 消费任务并发送结果
- 使用 `std::sync::mpsc` 和 `Arc<Mutex<Receiver>>` 进行工作窃取

<details>
<summary>🔑 解答</summary>

```rust
use std::sync::mpsc;
use std::thread;

struct Job {
    id: u64,
    data: String,
}

struct JobResult {
    job_id: u64,
    output: String,
    worker_id: usize,
}

fn worker_pool(jobs: Vec<Job>, num_workers: usize) -> Vec<JobResult> {
    let (job_tx, job_rx) = mpsc::channel::<Job>();
    let (result_tx, result_rx) = mpsc::channel::<JobResult>();

    let job_rx = std::sync::Arc::new(std::sync::Mutex::new(job_rx));

    let mut handles = Vec::new();
    for worker_id in 0..num_workers {
        let job_rx = job_rx.clone();
        let result_tx = result_tx.clone();
        handles.push(thread::spawn(move || {
            loop {
                let job = {
                    let rx = job_rx.lock().unwrap();
                    rx.recv()
                };
                match job {
                    Ok(job) => {
                        let output = format!("processed '{}' by worker {worker_id}", job.data);
                        result_tx.send(JobResult {
                            job_id: job.id, output, worker_id,
                        }).unwrap();
                    }
                    Err(_) => break,
                }
            }
        }));
    }
    drop(result_tx);

    let num_jobs = jobs.len();
    for job in jobs {
        job_tx.send(job).unwrap();
    }
    drop(job_tx);

    let results: Vec<_> = result_rx.into_iter().collect();
    assert_eq!(results.len(), num_jobs);

    for h in handles { h.join().unwrap(); }
    results
}

fn main() {
    let jobs: Vec<Job> = (0..20).map(|i| Job {
        id: i, data: format!("task-{i}"),
    }).collect();

    let results = worker_pool(jobs, 4);
    for r in &results {
        println!("[worker {}] job {}: {}", r.worker_id, r.job_id, r.output);
    }
}
```

</details>

***

