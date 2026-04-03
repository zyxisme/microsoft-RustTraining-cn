## Rust 闭包

> **你将学到：** 支持所有权感知的捕获闭包（`Fn`/`FnMut`/`FnOnce`）与 C# lambda 的对比，
> Rust 迭代器作为 LINQ 的零成本替代品，惰性求值与积极求值，
> 以及使用 `rayon` 进行并行迭代。
>
> **难度：** 🟡 中级

Rust 中的闭包与 C# 的 lambda 和委托类似，但支持所有权感知的捕获。

### C# Lambda 和委托
```csharp
// C# - Lambda 按引用捕获
Func<int, int> doubler = x => x * 2;
Action<string> printer = msg => Console.WriteLine(msg);

// 闭包捕获外部变量
int multiplier = 3;
Func<int, int> multiply = x => x * multiplier;
Console.WriteLine(multiply(5)); // 15

// LINQ 大量使用 lambda
var evens = numbers.Where(n => n % 2 == 0).ToList();
```

### Rust 闭包
```rust
// Rust 闭包 - 所有权感知
let doubler = |x: i32| x * 2;
let printer = |msg: &str| println!("{}", msg);

// 闭包按引用捕获（默认情况下不可变）
let multiplier = 3;
let multiply = |x: i32| x * multiplier; // 借用 multiplier
println!("{}", multiply(5)); // 15
println!("{}", multiplier); // 仍然可访问

// 闭包按 move 捕获
let data = vec![1, 2, 3];
let owns_data = move || {
    println!("{:?}", data); // data 移动到闭包中
};
owns_data();
// println!("{:?}", data); // 错误：data 已被移动

// 使用闭包与迭代器
let numbers = vec![1, 2, 3, 4, 5];
let evens: Vec<&i32> = numbers.iter().filter(|&&n| n % 2 == 0).collect();
```

### 闭包类型
```rust
// Fn - 不可变借用捕获的值
fn apply_fn(f: impl Fn(i32) -> i32, x: i32) -> i32 {
    f(x)
}

// FnMut - 可变借用捕获的值
fn apply_fn_mut(mut f: impl FnMut(i32), values: &[i32]) {
    for &v in values {
        f(v);
    }
}

// FnOnce - 获取捕获值的所有权
fn apply_fn_once(f: impl FnOnce() -> Vec<i32>) -> Vec<i32> {
    f() // 只能调用一次
}

fn main() {
    // Fn 示例
    let multiplier = 3;
    let result = apply_fn(|x| x * multiplier, 5);
    
    // FnMut 示例
    let mut sum = 0;
    apply_fn_mut(|x| sum += x, &[1, 2, 3, 4, 5]);
    println!("Sum: {}", sum); // 15
    
    // FnOnce 示例
    let data = vec![1, 2, 3];
    let result = apply_fn_once(move || data); // 移动 data
}
```

***

## LINQ vs Rust 迭代器

### C# LINQ（语言集成查询）
```csharp
// C# LINQ - 声明式数据处理
var numbers = new[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

var result = numbers
    .Where(n => n % 2 == 0)           // 筛选偶数
    .Select(n => n * n)               // 平方
    .Where(n => n > 10)               // 筛选 > 10
    .OrderByDescending(n => n)        // 降序排序
    .Take(3)                          // 取前3个
    .ToList();                        // 物化

// LINQ 处理复杂对象
var users = GetUsers();
var activeAdults = users
    .Where(u => u.IsActive && u.Age >= 18)
    .GroupBy(u => u.Department)
    .Select(g => new {
        Department = g.Key,
        Count = g.Count(),
        AverageAge = g.Average(u => u.Age)
    })
    .OrderBy(x => x.Department)
    .ToList();

// 异步 LINQ（需要额外库支持）
var results = await users
    .ToAsyncEnumerable()
    .WhereAwait(async u => await IsActiveAsync(u.Id))
    .SelectAwait(async u => await EnrichUserAsync(u))
    .ToListAsync();
```

### Rust 迭代器
```rust
// Rust 迭代器 - 惰性的、零成本抽象
let numbers = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

let result: Vec<i32> = numbers
    .iter()
    .filter(|&&n| n % 2 == 0)        // 筛选偶数
    .map(|&n| n * n)                 // 平方
    .filter(|&n| n > 10)             // 筛选 > 10
    .collect::<Vec<_>>()             // 收集到 Vec
    .into_iter()
    .rev()                           // 反转（降序排序）
    .take(3)                         // 取前3个
    .collect();                      // 物化

// 复杂的迭代器链
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct User {
    name: String,
    age: u32,
    department: String,
    is_active: bool,
}

fn process_users(users: Vec<User>) -> HashMap<String, (usize, f64)> {
    users
        .into_iter()
        .filter(|u| u.is_active && u.age >= 18)
        .fold(HashMap::new(), |mut acc, user| {
            let entry = acc.entry(user.department.clone()).or_insert((0, 0.0));
            entry.0 += 1;  // 计数
            entry.1 += user.age as f64;  // 年龄总和
            acc
        })
        .into_iter()
        .map(|(dept, (count, sum))| (dept, (count, sum / count as f64)))  // 平均值
        .collect()
}

// 使用 rayon 并行处理
use rayon::prelude::*;

fn parallel_processing(numbers: Vec<i32>) -> Vec<i32> {
    numbers
        .par_iter()                  // 并行迭代器
        .filter(|&&n| n % 2 == 0)
        .map(|&n| expensive_computation(n))
        .collect()
}

fn expensive_computation(n: i32) -> i32 {
    // 模拟重计算
    (0..1000).fold(n, |acc, _| acc + 1)
}
```

```mermaid
graph TD
    subgraph "C# LINQ 特性"
        CS_LINQ["LINQ 表达式"]
        CS_EAGER["通常为积极求值<br/>(ToList(), ToArray())"]
        CS_REFLECTION["[错误] 一些运行时反射<br/>表达式树"]
        CS_ALLOCATIONS["[错误] 中间集合<br/>垃圾回收压力"]
        CS_ASYNC["[OK] 异步支持<br/>(需要额外库)"]
        CS_SQL["[OK] LINQ to SQL/EF 集成"]
        
        CS_LINQ --> CS_EAGER
        CS_LINQ --> CS_REFLECTION
        CS_LINQ --> CS_ALLOCATIONS
        CS_LINQ --> CS_ASYNC
        CS_LINQ --> CS_SQL
    end
    
    subgraph "Rust 迭代器特性"
        RUST_ITER["迭代器链"]
        RUST_LAZY["[OK] 惰性求值<br/>直到 .collect() 才执行"]
        RUST_ZERO["[OK] 零成本抽象<br/>编译为最优循环"]
        RUST_NO_ALLOC["[OK] 无中间分配<br/>基于栈处理"]
        RUST_PARALLEL["[OK] 易于并行化<br/>(rayon crate)"]
        RUST_FUNCTIONAL["[OK] 函数式编程<br/>默认不可变"]
        
        RUST_ITER --> RUST_LAZY
        RUST_ITER --> RUST_ZERO
        RUST_ITER --> RUST_NO_ALLOC
        RUST_ITER --> RUST_PARALLEL
        RUST_ITER --> RUST_FUNCTIONAL
    end
    
    subgraph "性能对比"
        CS_PERF["C# LINQ 性能<br/>[错误] 分配开销<br/>[错误] 虚函数派发<br/>[OK] 对大多数场景足够好"]
        RUST_PERF["Rust 迭代器性能<br/>[OK] 手写优化速度<br/>[OK] 无分配<br/>[OK] 编译时优化"]
    end
    
    style CS_REFLECTION fill:#ffcdd2,color:#000
    style CS_ALLOCATIONS fill:#fff3e0,color:#000
    style RUST_ZERO fill:#c8e6c9,color:#000
    style RUST_LAZY fill:#c8e6c9,color:#000
    style RUST_NO_ALLOC fill:#c8e6c9,color:#000
    style CS_PERF fill:#fff3e0,color:#000
    style RUST_PERF fill:#c8e6c9,color:#000
```

***


<details>
<summary><strong>🏋️ 练习：LINQ 到迭代器的翻译</strong>（点击展开）</summary>

**挑战**：将此 C# LINQ 管道翻译为惯用的 Rust 迭代器。

```csharp
// C# — 翻译为 Rust
record Employee(string Name, string Dept, int Salary);

var result = employees
    .Where(e => e.Salary > 50_000)
    .GroupBy(e => e.Dept)
    .Select(g => new {
        Department = g.Key,
        Count = g.Count(),
        AvgSalary = g.Average(e => e.Salary)
    })
    .OrderByDescending(x => x.AvgSalary)
    .ToList();
```

<details>
<summary>🔑 答案</summary>

```rust
use std::collections::HashMap;

struct Employee { name: String, dept: String, salary: u32 }

#[derive(Debug)]
struct DeptStats { department: String, count: usize, avg_salary: f64 }

fn department_stats(employees: &[Employee]) -> Vec<DeptStats> {
    let mut by_dept: HashMap<&str, Vec<u32>> = HashMap::new();
    for e in employees.iter().filter(|e| e.salary > 50_000) {
        by_dept.entry(&e.dept).or_default().push(e.salary);
    }

    let mut stats: Vec<DeptStats> = by_dept
        .into_iter()
        .map(|(dept, salaries)| {
            let count = salaries.len();
            let avg = salaries.iter().sum::<u32>() as f64 / count as f64;
            DeptStats { department: dept.to_string(), count, avg_salary: avg }
        })
        .collect();

    stats.sort_by(|a, b| b.avg_salary.partial_cmp(&a.avg_salary).unwrap());
    stats
}
```

**关键要点**：
- Rust 内置迭代器没有 `group_by` —— `HashMap` + `fold`/`for` 是惯用模式
- `itertools` crate 添加了 `.group_by()` 以提供更像 LINQ 的语法
- 迭代器链是零成本的 —— 编译器将它们优化为简单循环

</details>
</details>


<!-- ch12.0a: itertools — LINQ Power Tools -->
## itertools：缺失的 LINQ 操作

标准 Rust 迭代器覆盖了 `map`、`filter`、`fold`、`take` 和 `collect`。但使用过 `GroupBy`、`Zip`、`Chunk`、`SelectMany` 和 `Distinct` 的 C# 开发者会立即注意到差距。**`itertools`** crate 填补了这些空白。

```toml
# Cargo.toml
[dependencies]
itertools = "0.12"
```

### 并排对比：LINQ vs itertools

```csharp
// C# — GroupBy
var byDept = employees.GroupBy(e => e.Department)
    .Select(g => new { Dept = g.Key, Count = g.Count() });

// C# — Chunk（分批）
var batches = items.Chunk(100);  // IEnumerable<T[]>

// C# — Distinct / DistinctBy
var unique = users.DistinctBy(u => u.Email);

// C# — SelectMany（扁平化）
var allTags = posts.SelectMany(p => p.Tags);

// C# — Zip
var pairs = names.Zip(scores, (n, s) => new { Name = n, Score = s });

// C# — 滑动窗口
var windows = data.Zip(data.Skip(1), data.Skip(2))
    .Select(triple => (triple.First + triple.Second + triple.Third) / 3.0);
```

```rust
use itertools::Itertools;

// Rust — group_by（需要排序输入）
let by_dept = employees.iter()
    .sorted_by_key(|e| &e.department)
    .group_by(|e| &e.department);
for (dept, group) in &by_dept {
    println!("{}: {} employees", dept, group.count());
}

// Rust — chunks（分批）
let batches = items.iter().chunks(100);
for batch in &batches {
    process_batch(batch.collect::<Vec<_>>());
}

// Rust — unique / unique_by
let unique: Vec<_> = users.iter().unique_by(|u| &u.email).collect();

// Rust — flat_map（SelectMany 等价物 —— 内置！）
let all_tags: Vec<&str> = posts.iter().flat_map(|p| &p.tags).collect();

// Rust — zip（内置！）
let pairs: Vec<_> = names.iter().zip(scores.iter()).collect();

// Rust — tuple_windows（滑动窗口）
let moving_avg: Vec<f64> = data.iter()
    .tuple_windows::<(_, _, _)>()
    .map(|(a, b, c)| (*a + *b + *c) as f64 / 3.0)
    .collect();
```

### itertools 快速参考

| LINQ 方法 | itertools 等价物 | 备注 |
|------------|---------------------|------|
| `GroupBy(key)` | `.sorted_by_key().group_by()` | 需要排序输入（与 LINQ 不同） |
| `Chunk(n)` | `.chunks(n)` | 返回迭代器的迭代器 |
| `Distinct()` | `.unique()` | 需要 `Eq + Hash` |
| `DistinctBy(key)` | `.unique_by(key)` | |
| `SelectMany()` | `.flat_map()` | 内置于 std —— 无需额外 crate |
| `Zip()` | `.zip()` | 内置于 std |
| `Aggregate()` | `.fold()` | 内置于 std |
| `Any()` / `All()` | `.any()` / `.all()` | 内置于 std |
| `First()` / `Last()` | `.next()` / `.last()` | 内置于 std |
| `Skip(n)` / `Take(n)` | `.skip(n)` / `.take(n)` | 内置于 std |
| `OrderBy()` | `.sorted()` / `.sorted_by()` | `itertools`（std 无此功能） |
| `ThenBy()` | `.sorted_by(|a,b| a.x.cmp(&b.x).then(a.y.cmp(&b.y)))` | 链式 `Ordering::then` |
| `Intersect()` | `HashSet` 交集 | 无直接迭代器方法 |
| `Concat()` | `.chain()` | 内置于 std |
| 滑动窗口 | `.tuple_windows()` | 固定大小元组 |
| 笛卡尔积 | `.cartesian_product()` | `itertools` |
| 交叉合并 | `.interleave()` | `itertools` |
| 排列 | `.permutations(k)` | `itertools` |

### 真实案例：日志分析管道

```rust
use itertools::Itertools;
use std::collections::HashMap;

#[derive(Debug)]
struct LogEntry { level: String, module: String, message: String }

fn analyze_logs(entries: &[LogEntry]) {
    // 噪音最大的前5个模块（类似 LINQ GroupBy + OrderByDescending + Take）
    let noisy: Vec<_> = entries.iter()
        .into_group_map_by(|e| &e.module) // itertools：直接分组到 HashMap
        .into_iter()
        .sorted_by(|a, b| b.1.len().cmp(&a.1.len()))
        .take(5)
        .collect();

    for (module, entries) in &noisy {
        println!("{}: {} entries", module, entries.len());
    }

    // 每100条窗口的错误率（滑动窗口）
    let error_rates: Vec<f64> = entries.iter()
        .map(|e| if e.level == "ERROR" { 1.0 } else { 0.0 })
        .collect::<Vec<_>>()
        .windows(100)  // std slice 方法
        .map(|w| w.iter().sum::<f64>() / 100.0)
        .collect();

    // 去重连续相同的消息
    let deduped: Vec<_> = entries.iter().dedup_by(|a, b| a.message == b.message).collect();
    println!("Deduped {} → {} entries", entries.len(), deduped.len());
}
```

***


