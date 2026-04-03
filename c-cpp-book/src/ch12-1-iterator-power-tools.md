## 迭代器强大工具参考

> **你将学到什么：** 除 `filter`/`map`/`collect` 之外的高级迭代器组合器——`enumerate`、`zip`、`chain`、`flat_map`、`scan`、`windows` 和 `chunks`。对于用安全、表达力强的 Rust 迭代器替代 C 风格的索引 `for` 循环至关重要。

基本的 `filter`/`map`/`collect` 链涵盖许多情况，但 Rust 的迭代器库要丰富得多。本节介绍你每天都会使用的工具——特别是当转换手动跟踪索引、累积结果或处理固定大小块数据的 C 循环时。

### 快速参考表

| 方法 | C等价 | 作用 | 返回 |
|--------|-------------|-------------|---------|
| `enumerate()` | `for (int i=0; ...)` | 将每个元素与其索引配对 | `(usize, T)` |
| `zip(other)` | 并行数组使用相同索引 | 将两个迭代器的元素配对 | `(A, B)` |
| `chain(other)` | 先处理array1再处理array2 | 连接两个迭代器 | `T` |
| `flat_map(f)` | 嵌套循环 | 映射后展平一层 | `U` |
| `windows(n)` | `for (int i=0; i<len-n+1; i++) &arr[i..i+n]` | 大小为`n`的重叠切片 | `&[T]` |
| `chunks(n)` | 每次处理`n`个元素 | 大小为`n`的非重叠切片 | `&[T]` |
| `fold(init, f)` | `int acc = init; for (...) acc = f(acc, x);` | 归约为单个值 | `Acc` |
| `scan(init, f)` | 带输出的运行累积器 | 类似`fold`但产生中间结果 | `Option<B>` |
| `take(n)` / `skip(n)` | 从偏移量开始循环 / 限制 | 前`n`个 / 跳过前`n`个元素 | `T` |
| `take_while(f)` / `skip_while(f)` | `while (pred) {...}` | 条件满足时获取/跳过 | `T` |
| `peekable()` | 用`arr[i+1]` lookahead | 允许`.peek()`而不消费 | `T` |
| `step_by(n)` | `for (i=0; i<len; i+=n)` | 取每第n个元素 | `T` |
| `unzip()` | 分割并行数组 | 将配对收集到两个集合 | `(A, B)` |
| `sum()` / `product()` | 累积求和/乘积 | 用`+`或`*`归约 | `T` |
| `min()` / `max()` | 找极值 | 返回`Option<T>` | `Option<T>` |
| `any(f)` / `all(f)` | `bool found = false; for (...) ...` | 短路布尔搜索 | `bool` |
| `position(f)` | `for (i=0; ...) if (pred) return i;` | 第一个匹配的位置 | `Option<usize>` |

### `enumerate` — 索引 + 值（替代 C 索引循环）

```rust
fn main() {
    let sensors = ["GPU_TEMP", "CPU_TEMP", "FAN_RPM", "PSU_WATT"];

    // C style: for (int i = 0; i < 4; i++) printf("[%d] %s\n", i, sensors[i]);
    for (i, name) in sensors.iter().enumerate() {
        println!("[{i}] {name}");
    }

    // Find the index of a specific sensor
    let gpu_idx = sensors.iter().position(|&s| s == "GPU_TEMP");
    println!("GPU sensor at index: {gpu_idx:?}");  // Some(0)
}
```

### `zip` — 并行迭代（替代并行数组循环）

```rust
fn main() {
    let names = ["accel_diag", "nic_diag", "cpu_diag"];
    let statuses = [true, false, true];
    let durations_ms = [1200, 850, 3400];

    // C: for (int i=0; i<3; i++) printf("%s: %s (%d ms)\n", names[i], ...);
    for ((name, passed), ms) in names.iter().zip(&statuses).zip(&durations_ms) {
        let status = if *passed { "PASS" } else { "FAIL" };
        println!("{name}: {status} ({ms} ms)");
    }
}
```

### `chain` — 连接迭代器

```rust
fn main() {
    let critical = vec!["ECC error", "Thermal shutdown"];
    let warnings = vec!["Link degraded", "Fan slow"];

    // Process all events in priority order
    let all_events: Vec<_> = critical.iter().chain(warnings.iter()).collect();
    println!("{all_events:?}");
    // ["ECC error", "Thermal shutdown", "Link degraded", "Fan slow"]
}
```

### `flat_map` — 展平嵌套结果

```rust
fn main() {
    let lines = vec!["gpu:42:ok", "nic:99:fail", "cpu:7:ok"];

    // Extract all numeric values from colon-separated lines
    let numbers: Vec<u32> = lines.iter()
        .flat_map(|line| line.split(':'))
        .filter_map(|token| token.parse::<u32>().ok())
        .collect();
    println!("{numbers:?}");  // [42, 99, 7]
}
```

### `windows` and `chunks` — 滑动和固定大小分组

```rust
fn main() {
    let temps = [65, 68, 72, 71, 75, 80, 78, 76];

    // windows(3): overlapping groups of 3 (like a sliding average)
    // C: for (int i = 0; i <= len-3; i++) avg(arr[i], arr[i+1], arr[i+2]);
    let moving_avg: Vec<f64> = temps.windows(3)
        .map(|w| w.iter().sum::<i32>() as f64 / 3.0)
        .collect();
    println!("Moving avg: {moving_avg:.1?}");

    // chunks(2): non-overlapping groups of 2
    // C: for (int i = 0; i < len; i += 2) process(arr[i], arr[i+1]);
    for pair in temps.chunks(2) {
        println!("Chunk: {pair:?}");
    }

    // chunks_exact(2): same but panics if remainder exists
    // Also: .remainder() gives leftover elements
}
```

### `fold` and `scan` — 累积

```rust
fn main() {
    let values = [10, 20, 30, 40, 50];

    // fold: single final result (like C's accumulator loop)
    let sum = values.iter().fold(0, |acc, &x| acc + x);
    println!("Sum: {sum}");  // 150

    // Build a string with fold
    let csv = values.iter()
        .fold(String::new(), |acc, x| {
            if acc.is_empty() { format!("{x}") }
            else { format!("{acc},{x}") }
        });
    println!("CSV: {csv}");  // "10,20,30,40,50"

    // scan: like fold but yields intermediate results
    let running_sum: Vec<i32> = values.iter()
        .scan(0, |state, &x| {
            *state += x;
            Some(*state)
        })
        .collect();
    println!("Running sum: {running_sum:?}");  // [10, 30, 60, 100, 150]
}
```

### 练习：传感器数据管道

给定原始传感器读数（每行一个，格式为`sensor_name:value:unit"`），编写一个
迭代器管道来：
1. 将每行解析为`(name, f64, unit)`
2. 过滤掉低于阈值的读数
3. 使用`fold`按传感器名称分组到`HashMap`
4. 打印每个传感器的平均读数

```rust
// Starter code
fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;
    // TODO: Parse, filter values >= threshold, group by name, compute averages
}
```

<details><summary>Solution (click to expand)</summary>

```rust
use std::collections::HashMap;

fn main() {
    let raw_data = vec![
        "gpu_temp:72.5:C",
        "cpu_temp:65.0:C",
        "gpu_temp:74.2:C",
        "fan_rpm:1200.0:RPM",
        "cpu_temp:63.8:C",
        "gpu_temp:80.1:C",
        "fan_rpm:1150.0:RPM",
    ];
    let threshold = 70.0;

    // Parse → filter → group → average
    let grouped = raw_data.iter()
        .filter_map(|line| {
            let parts: Vec<&str> = line.splitn(3, ':').collect();
            if parts.len() == 3 {
                let value: f64 = parts[1].parse().ok()?;
                Some((parts[0], value, parts[2]))
            } else {
                None
            }
        })
        .filter(|(_, value, _)| *value >= threshold)
        .fold(HashMap::<&str, Vec<f64>>::new(), |mut acc, (name, value, _)| {
            acc.entry(name).or_default().push(value);
            acc
        });

    for (name, values) in &grouped {
        let avg = values.iter().sum::<f64>() / values.len() as f64;
        println!("{name}: avg={avg:.1} ({} readings)", values.len());
    }
}
// Output (order may vary):
// gpu_temp: avg=75.6 (3 readings)
// fan_rpm: avg=1175.0 (2 readings)
```

</details>


# Rust 迭代器
- ```Iterator``` trait 用于实现用户定义类型的迭代（https://doc.rust-lang.org/std/iter/trait.IntoIterator.html）
    - 在示例中，我们将实现斐波那契序列的迭代器，它以1, 1, 2, ...开始，后继是前两个数字的和
    - ```Iterator```中的```associated type```（```type Item = u32;```）定义了迭代器的输出类型（```u32```）
    - ```next()```方法只包含实现迭代器的逻辑。在这种情况下，所有状态信息都可在```Fibonacci```结构中获得
    - 我们也可以实现另一个叫```IntoIterator```的trait来为更专门的迭代器实现```into_iter()```方法
    - https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=ab367dc2611e1b5a0bf98f1185b38f3f


