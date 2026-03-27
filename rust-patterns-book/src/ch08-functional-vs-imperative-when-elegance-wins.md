# 第 8 章 — 函数式 vs 命令式：何时优雅胜出（以及何时不）

> **难度：** 🟡 中级 | **时间：** 2-3 小时 | **前置知识：** [第 7 章 — 闭包](ch07-closures-and-higher-order-functions.md)

Rust 让函数式和命令式风格之间获得了真正的对等性。与 Haskell（强制函数式）或 C（默认命令式）不同，Rust 让你选择——正确的选择取决于你要表达的内容。本章培养判断何时选择好的能力。

**核心原则：** 当你要**通过管道转换数据**时，函数式风格大放异彩。当你要**管理有副作用的状态转换**时，命令式风格大放异彩。大多数实际代码两者都有，技巧在于知道边界在哪里。

---

## 8.1 你不知道你想要的组合子

许多 Rust 开发者这样写：

```rust
let value = if let Some(x) = maybe_config() {
    x
} else {
    default_config()
};
process(value);
```

他们可以这样写：

```rust
process(maybe_config().unwrap_or_else(default_config));
```

或者这个常见模式：

```rust
let display_name = if let Some(name) = user.nickname() {
    name.to_uppercase()
} else {
    "ANONYMOUS".to_string()
};
```

这可以写成：

```rust
let display_name = user.nickname()
    .map(|n| n.to_uppercase())
    .unwrap_or_else(|| "ANONYMOUS".to_string());
```

函数式版本不仅仅是更短——它告诉你*发生了什么*（转换，然后默认值）而不必让你追踪控制流。`if let` 版本让你阅读分支才能弄清楚两条路径最终到达同一个地方。

### Option 组合子家族

这是心智模型：`Option<T>` 是一个单元素或空集合。`Option` 上的每个组合子都与集合操作有类比。

| 你写... | 而不是... | 传达什么 |
|---------|----------|----------|
| `opt.unwrap_or(default)` | `if let Some(x) = opt { x } else { default }` | "使用此值或回退" |
| `opt.unwrap_or_else(\|\| expensive())` | `if let Some(x) = opt { x } else { expensive() }` | 相同，但默认值是惰性的 |
| `opt.map(f)` | `match opt { Some(x) => Some(f(x)), None => None }` | "转换内部，传播空值" |
| `opt.and_then(f)` | `match opt { Some(x) => f(x), None => None }` | "链接可能失败的操作"（flatmap） |
| `opt.filter(\|x\| pred(x))` | `match opt { Some(x) if pred(&x) => Some(x), _ => None }` | "只保留通过的" |
| `opt.zip(other)` | `if let (Some(a), Some(b)) = (opt, other) { Some((a,b)) } else { None }` | "两者都有或都没有" |
| `opt.or(fallback)` | `if opt.is_some() { opt } else { fallback }` | "第一个可用的" |
| `opt.or_else(\|\| try_another())` | `if opt.is_some() { opt } else { try_another() }` | "按顺序尝试替代方案" |
| `opt.map_or(default, f)` | `if let Some(x) = opt { f(x) } else { default }` | "转换或默认值"——一行搞定 |
| `opt.map_or_else(default_fn, f)` | `if let Some(x) = opt { f(x) } else { default_fn() }` | 相同，两边都是闭包 |
| `opt?` | `match opt { Some(x) => x, None => return None }` | "向上传播空值" |

### Result 组合子家族

相同的模式适用于 `Result<T, E>`：

| 你写... | 而不是... | 传达什么 |
|---------|----------|----------|
| `res.map(f)` | `match res { Ok(x) => Ok(f(x)), Err(e) => Err(e) }` | 转换成功路径 |
| `res.map_err(f)` | `match res { Ok(x) => Ok(x), Err(e) => Err(f(e)) }` | 转换错误 |
| `res.and_then(f)` | `match res { Ok(x) => f(x), Err(e) => Err(e) }` | 链接可能失败的操作 |
| `res.unwrap_or_else(\|e\| default(e))` | `match res { Ok(x) => x, Err(e) => default(e) }` | 从错误中恢复 |
| `res.ok()` | `match res { Ok(x) => Some(x), Err(_) => None }` | "我不在乎错误" |
| `res?` | `match res { Ok(x) => x, Err(e) => return Err(e.into()) }` | 向上传播错误 |

### 何时 `if let` 更好

当以下情况时组合子会失效：

- **你需要 `Some` 分支中的多个语句。** 带有 5 行代码的 map 闭包比带有 5 行代码的 `if let` 更糟糕。
- **控制流才是重点。** `if let Some(connection) = pool.try_get() { /* 使用它 */ } else { /* 记录、重试、警告 */ }`——两个分支是真正不同的代码路径，而不是转换或默认值。
- **副作用占主导。** 如果两个分支都做 I/O 且有不同错误处理，组合子版本会掩盖重要差异。

**经验法则：** 如果 `else` 分支产生与 `Some` 分支相同的类型且函数体很短，使用组合子。如果分支做根本不同的事情，使用 `if let` 或 `match`。

---

## 8.2 Bool 组合子：`.then()` 和 `.then_some()`

另一个比应有更常见的模式：

```rust
let label = if is_admin {
    Some("ADMIN")
} else {
    None
};
```

Rust 1.62+ 给了你：

```rust
let label = is_admin.then_some("ADMIN");
```

或者带计算值：

```rust
let permissions = is_admin.then(|| compute_admin_permissions());
```

这在链式中特别强大：

```rust
// Imperative
let mut tags = Vec::new();
if user.is_admin { tags.push("admin"); }
if user.is_verified { tags.push("verified"); }
if user.score > 100 { tags.push("power-user"); }

// Functional
let tags: Vec<&str> = [
    user.is_admin.then_some("admin"),
    user.is_verified.then_some("verified"),
    (user.score > 100).then_some("power-user"),
]
.into_iter()
.flatten()
.collect();
```

函数式版本使模式明确："从条件元素构建列表。"命令式版本让你阅读每个 `if` 来确认它们都做同样的事情（推送标签）。

---

## 8.3 迭代器链 vs 循环：决策框架

第 7 章展示了机制。本节培养判断力。

### 迭代器胜出的时候

**数据管道**——通过一系列步骤转换集合：

```rust
// Imperative: 8 lines, 2 mutable variables
let mut results = Vec::new();
for item in inventory {
    if item.category == Category::Server {
        if let Some(temp) = item.last_temperature() {
            if temp > 80.0 {
                results.push((item.id, temp));
            }
        }
    }
}

// Functional: 6 lines, 0 mutable variables, one pipeline
let results: Vec<_> = inventory.iter()
    .filter(|item| item.category == Category::Server)
    .filter_map(|item| item.last_temperature().map(|t| (item.id, t)))
    .filter(|(_, temp)| *temp > 80.0)
    .collect();
```

函数式版本胜出因为：
- 每个过滤器独立可读
- 无 `mut`——数据单向流动
- 你可以添加/删除/重新排序管道阶段而不重构
- LLVM 将迭代器适配器内联到与循环相同的机器码

**聚合**——从集合计算单一值：

```rust
// Imperative
let mut total_power = 0.0;
let mut count = 0;
for server in fleet {
    total_power += server.power_draw();
    count += 1;
}
let avg = total_power / count as f64;

// Functional
let (total_power, count) = fleet.iter()
    .map(|s| s.power_draw())
    .fold((0.0, 0usize), |(sum, n), p| (sum + p, n + 1));
let avg = total_power / count as f64;
```

或者如果你只需要总和，甚至更简单：

```rust
let total: f64 = fleet.iter().map(|s| s.power_draw()).sum();
```

### 循环胜出的时候

**带复杂状态的提前退出：**

```rust
// This is clear and direct
let mut best_candidate = None;
for server in fleet {
    let score = evaluate(server);
    if score > threshold {
        if server.is_available() {
            best_candidate = Some(server);
            break; // Found one — stop immediately
        }
    }
}

// The functional version is strained
let best_candidate = fleet.iter()
    .filter(|s| evaluate(s) > threshold)
    .find(|s| s.is_available());
```

等等——那个函数式版本实际上相当简洁。让我们尝试一个它真正失效的情况：

**同时构建多个输出：**

```rust
// Imperative: clear, each branch does something different
let mut warnings = Vec::new();
let mut errors = Vec::new();
let mut stats = Stats::default();

for event in log_stream {
    match event.severity {
        Severity::Warn => {
            warnings.push(event.clone());
            stats.warn_count += 1;
        }
        Severity::Error => {
            errors.push(event.clone());
            stats.error_count += 1;
            if event.is_critical() {
                alert_oncall(&event);
            }
        }
        _ => stats.other_count += 1,
    }
}

// Functional version: forced, awkward, nobody wants to read this
let (warnings, errors, stats) = log_stream.iter().fold(
    (Vec::new(), Vec::new(), Stats::default()),
    |(mut w, mut e, mut s), event| {
        match event.severity {
            Severity::Warn => { w.push(event.clone()); s.warn_count += 1; }
            Severity::Error => {
                e.push(event.clone()); s.error_count += 1;
                if event.is_critical() { alert_oncall(event); }
            }
            _ => s.other_count += 1,
        }
        (w, e, s)
    },
);
```

fold 版本*更长*、*更难读*，而且有突变（`mut` 解构的累加器）。循环胜出因为：
- 多个输出并行构建
- 副作用（警告）混入逻辑
- 分支体是语句而非表达式

**带 I/O 的状态机：**

```rust
// A parser that reads tokens — the loop IS the algorithm
let mut state = ParseState::Start;
loop {
    let token = lexer.next_token()?;
    state = match state {
        ParseState::Start => match token {
            Token::Keyword(k) => ParseState::GotKeyword(k),
            Token::Eof => break,
            _ => return Err(ParseError::UnexpectedToken(token)),
        },
        ParseState::GotKeyword(k) => match token {
            Token::Ident(name) => ParseState::GotName(k, name),
            _ => return Err(ParseError::ExpectedIdentifier),
        },
        // ...more states
    };
}
```

没有更简洁的函数式等价物。带 `match state` 的循环是状态机的自然表达。

### 决策流程图

```mermaid
flowchart TB
    START{What are you doing?}

    START -->|"Transforming a collection\ninto another collection"| PIPE[Use iterator chain]
    START -->|"Computing a single value\nfrom a collection"| AGG{How complex?}
    START -->|"Multiple outputs from\none pass"| LOOP[Use a for loop]
    START -->|"State machine with\nI/O or side effects"| LOOP
    START -->|"One Option/Result\ntransform + default"| COMB[Use combinators]

    AGG -->|"Sum, count, min, max"| BUILTIN["Use .sum(), .count(),\n.min(), .max()"]
    AGG -->|"Custom accumulation"| FOLD{Accumulator has mutation\nor side effects?}
    FOLD -->|"No"| FOLDF["Use .fold()"]
    FOLD -->|"Yes"| LOOP

    style PIPE fill:#d4efdf,stroke:#27ae60,color:#000
    style COMB fill:#d4efdf,stroke:#27ae60,color:#000
    style BUILTIN fill:#d4efdf,stroke:#27ae60,color:#000
    style FOLDF fill:#d4efdf,stroke:#27ae60,color:#000
    style LOOP fill:#fef9e7,stroke:#f1c40f,color:#000
```

### 旁注：作用域可变性——内部命令式，外部函数式

Rust 块是表达式。这让你将突变限制在构造阶段并不可变地绑定结果：

```rust
use rand::random;

let samples = {
    let mut buf = Vec::with_capacity(10);
    while buf.len() < 10 {
        let reading: f64 = random();
        buf.push(reading);
        if random::<u8>() % 3 == 0 { break; } // randomly stop early
    }
    buf
};
// samples is immutable — contains between 1 and 10 elements
```

内部的 `buf` 只在块内可变。一旦块产生结果，外部绑定 `samples` 是不可变的，编译器将拒绝任何后续的 `samples.push(...)`。

**为什么不是迭代器链？** 你可能尝试：

```rust
let samples: Vec<f64> = std::iter::from_fn(|| Some(random()))
    .take(10)
    .take_while(|_| random::<u8>() % 3 != 0)
    .collect();
```

但 `take_while` *排除*不满足谓词的元素，产生 0 到 9 个元素而不是命令式版本保证的至少 1 个。你可以用 `scan` 或 `chain` 来解决，但命令式版本更清晰。

**作用域可变性真正胜出的时候：**

| 场景 | 为什么迭代器挣扎 |
|------|-----------------|
| **排序然后冻结**（`sort_unstable()` + `dedup()`） | 两者都返回 `()`——没有可链式处理的输出（itertools 提供 `.sorted().dedup()` 如果可用） |
| **有状态终止**（在与数据无关的条件上停止） | `take_while` 丢弃边界元素 |
| **多步骤结构填充**（来自不同源的字段按字段） | 没有自然的单一管道 |

**诚实的校准：** 对于大多数集合构建任务，迭代器链或 [itertools](https://docs.rs/itertools) 是首选。当构造逻辑有分支、提前退出或不适合单一管道的中途变化时，寻求作用域可变性。这个模式的真正价值在于教学*突变作用域可以小于变量生命周期*——这是让来自 C++、C# 和 Python 的开发者惊讶的 Rust 基础。

---

## 8.4 `?` 操作符：函数式遇见命令式的地方

`?` 操作符是 Rust 最优雅的两种风格融合。它本质上是 `.and_then()` 与提前返回的结合：

```rust
// This chain of and_then...
fn load_config() -> Result<Config, Error> {
    read_file("config.toml")
        .and_then(|contents| parse_toml(&contents))
        .and_then(|table| validate_config(table))
        .and_then(|valid| Config::from_validated(valid))
}

// ...is exactly equivalent to this
fn load_config() -> Result<Config, Error> {
    let contents = read_file("config.toml")?;
    let table = parse_toml(&contents)?;
    let valid = validate_config(table)?;
    Config::from_validated(valid)
}
```

两者在精神上都是函数式的（它们自动传播错误），但 `?` 版本给你命名的中间变量，这在以下情况很重要：

- 你需要再次使用 `contents`
- 你想为每步添加 `.context("while parsing config")?`
- 你在调试并想检查中间值

**反模式：** 当 `?` 可用时使用长的 `.and_then()` 链。如果链中每个闭包都是 `|x| next_step(x)`，你重新发明了 `?` 而没有可读性。

**何时 `.and_then()` 比 `?` 更好：**

```rust
// Transforming inside an Option, without early return
let port: Option<u16> = config.get("port")
    .and_then(|v| v.parse::<u16>().ok())
    .filter(|&p| p > 0 && p < 65535);
```

你不能在这里使用 `?`，因为没有要返回的封闭函数——你正在构建一个 `Option` 而不是传播它。

---

## 8.5 集合构建：`collect()` vs 推送循环

`collect()` 比大多数开发者意识到的更强大：

### 收集到 Result

```rust
// Imperative: parse a list, fail on first error
let mut numbers = Vec::new();
for s in input_strings {
    let n: i64 = s.parse().map_err(|_| Error::BadInput(s.clone()))?;
    numbers.push(n);
}

// Functional: collect into Result<Vec<_>, _>
let numbers: Vec<i64> = input_strings.iter()
    .map(|s| s.parse::<i64>().map_err(|_| Error::BadInput(s.clone())))
    .collect::<Result(_, _>>()?;
```

`collect::<Result<Vec<_>, _>>()` 技巧有效，因为 `Result` 实现了 `FromIterator`。它在第一个 `Err` 时短路，就像带 `?` 的循环一样。

### 收集到 HashMap

```rust
// Imperative
let mut index = HashMap::new();
for server in fleet {
    index.insert(server.id.clone(), server);
}

// Functional
let index: HashMap<_, _> = fleet.into_iter()
    .map(|s| (s.id.clone(), s))
    .collect();
```

### 收集到 String

```rust
// Imperative
let mut csv = String::new();
for (i, field) in fields.iter().enumerate() {
    if i > 0 { csv.push(','); }
    csv.push_str(field);
}

// Functional
let csv = fields.join(",");

// Or for more complex formatting:
let csv: String = fields.iter()
    .map(|f| format!("\"{f}\""))
    .collect::<Vec<_>>()
    .join(",");
```

### 循环版本胜出的时候

`collect()` 分配一个新集合。如果你要*原地修改*，循环既更清晰也更高效：

```rust
// In-place update — no functional equivalent that's better
for server in &mut fleet {
    if server.needs_refresh() {
        server.refresh_telemetry()?;
    }
}
```

函数式版本需要 `.iter_mut().for_each(|s| { ... })`，这只是带有额外语法的循环。

---

## 8.6 模式匹配作为函数分派

Rust 的 `match` 是大多数开发者命令式使用的函数式构造。以下是函数式视角：

### Match 作为查找表

```rust
// Imperative thinking: "check each case"
fn status_message(code: StatusCode) -> &'static str {
    if code == StatusCode::OK { "Success" }
    else if code == StatusCode::NOT_FOUND { "Not found" }
    else if code == StatusCode::INTERNAL { "Server error" }
    else { "Unknown" }
}

// Functional thinking: "map from domain to range"
fn status_message(code: StatusCode) -> &'static str {
    match code {
        StatusCode::OK => "Success",
        StatusCode::NOT_FOUND => "Not found",
        StatusCode::INTERNAL => "Server error",
        _ => "Unknown",
    }
}
```

`match` 版本不仅仅是风格——编译器验证穷举性。添加一个新变体，每个不处理它的 `match` 都会成为编译错误。`if/else` 链静默地落入默认。

### Match + 解构作为管道

```rust
// Parsing a command — each arm extracts and transforms
fn execute(cmd: Command) -> Result<Response, Error> {
    match cmd {
        Command::Get { key } => db.get(&key).map(Response::Value),
        Command::Set { key, value } => db.set(key, value).map(|_| Response::Ok),
        Command::Delete { key } => db.delete(&key).map(|_| Response::Ok),
        Command::Batch(cmds) => cmds.into_iter()
            .map(execute)
            .collect::<Result<Vec<_>, _>>()
            .map(Response::Batch),
    }
}
```

每个分支是返回相同类型的表达式。这是作为函数分派的模式匹配——`match` 分支本质上是按枚举变体索引的函数表。

---

## 8.7 在自定义类型上链式调用方法

函数式风格超越了标准库类型。Builder 模式和流式 API 是变相的函数式编程：

```rust
// This is a combinator chain over your own type
let query = QueryBuilder::new("servers")
    .filter("status", Eq, "active")
    .filter("rack", In, &["A1", "A2", "B1"])
    .order_by("temperature", Desc)
    .limit(50)
    .build();
```

**关键洞察：** 如果你的类型有接受 `self` 并返回 `Self`（或转换类型）的方法，你就构建了一个组合子。相同的函数式/命令式判断适用：

```rust
// Good: chainable because each step is a simple transform
let config = Config::default()
    .with_timeout(Duration::from_secs(30))
    .with_retries(3)
    .with_tls(true);

// Bad: chainable but the chain is doing too many unrelated things
let result = processor
    .load_data(path)?       // I/O
    .validate()             // Pure
    .transform(rule_set)    // Pure
    .save_to_disk(output)?  // I/O
    .notify_downstream()?;  // Side effect

// Better: separate the pure pipeline from the I/O bookends
let data = load_data(path)?;
let processed = data.validate().transform(rule_set);
save_to_disk(output, &processed)?;
notify_downstream()?;
```

当链混合纯转换与 I/O 时会失效。读者无法判断哪些调用可能失败，哪些有副作用，以及实际的数据转换发生在哪里。

---

## 8.8 性能：它们是一样的

一个常见误解："函数式风格更慢，因为所有闭包和分配。"

在 Rust 中，**迭代器链编译成与手写循环相同的机器码。** LLVM 内联闭包调用，消除迭代器适配器结构，通常产生相同的汇编。这叫做*零成本抽象*，这不是 aspiration——这是可测量的。

```rust
// These produce identical assembly on release builds:

// Functional
let sum: i64 = (0..1000).filter(|n| n % 2 == 0).map(|n| n * n).sum();

// Imperative
let mut sum: i64 = 0;
for n in 0..1000 {
    if n % 2 == 0 {
        sum += n * n;
    }
}
```

**一个例外：** `.collect()` 会分配。如果你要链 `.map().collect().iter().map().collect()` 并带有中间集合，你正在为循环版本避免的分配付出代价。修复方法：通过直接链式适配器消除中间 collect，或者如果你需要中间集合用于其他原因则使用循环。

---

## 8.9 品味测试：转换目录

这是一个最常见的"我写了 6 行但有一个一行式"模式的参考表：

| 命令式模式 | 函数式等价 | 何时偏好函数式 |
|-----------|-----------|--------------|
| `if let Some(x) = opt { f(x) } else { default }` | `opt.map_or(default, f)` | 两边都是短表达式 |
| `if let Some(x) = opt { Some(g(x)) } else { None }` | `opt.map(g)` | 始终——这就是 `map` 的用途 |
| `if condition { Some(x) } else { None }` | `condition.then_some(x)` | 始终 |
| `if condition { Some(compute()) } else { None }` | `condition.then(compute)` | 始终 |
| `match opt { Some(x) if pred(x) => Some(x), _ => None }` | `opt.filter(pred)` | 始终 |
| `for x in iter { if pred(x) { result.push(f(x)); } }` | `iter.filter(pred).map(f).collect()` | 管道在一屏内可读时 |
| `if a.is_some() && b.is_some() { Some((a?, b?)) }` | `a.zip(b)` | 始终——`.zip()` 就是这样 |
| `match (a, b) { (Some(x), Some(y)) => x + y, _ => 0 }` | `a.zip(b).map(\|(x,y)\| x + y).unwrap_or(0)` | 判断调用——取决于复杂性 |
| `iter.map(f).collect::<Vec<_>>()[0]` | `iter.map(f).next().unwrap()` | 不要为单个元素分配 Vec |
| `let mut v = vec; v.sort(); v` | `{ let mut v = vec; v.sort(); v }` | Rust std 没有 `.sorted()`（使用 itertools） |

---

## 8.10 反模式

### 过度函数化：没人能读的 5 层深度链

```rust
// This is not elegant. This is a puzzle.
let result = data.iter()
    .filter_map(|x| x.metadata.as_ref())
    .flat_map(|m| m.tags.iter())
    .filter(|t| t.starts_with("env:"))
    .map(|t| t.strip_prefix("env:").unwrap())
    .filter(|env| allowed_envs.contains(env))
    .map(|env| env.to_uppercase())
    .collect::<HashSet<_>>()
    .into_iter()
    .sorted()
    .collect::<Vec<_>>();
```

当链超过约 4 个适配器时，用命名的中间变量分解或提取辅助函数：

```rust
let env_tags = data.iter()
    .filter_map(|x| x.metadata.as_ref())
    .flat_map(|m| m.tags.iter());

let allowed: Vec<_> = env_tags
    .filter_map(|t| t.strip_prefix("env:"))
    .filter(|env| allowed_envs.contains(env))
    .map(|env| env.to_uppercase())
    .sorted()
    .collect();
```

### 函数化不足：Rust 有对应词的 C 风格循环

```rust
// This is just .any()
let mut found = false;
for item in &list {
    if item.is_expired() {
        found = true;
        break;
    }
}

// Write this instead
let found = list.iter().any(|item| item.is_expired());
```

```rust
// This is just .find()
let mut target = None;
for server in &fleet {
    if server.id == target_id {
        target = Some(server);
        break;
    }
}

// Write this instead
let target = fleet.iter().find(|s| s.id == target_id);
```

```rust
// This is just .all()
let mut all_healthy = true;
for server in &fleet {
    if !server.is_healthy() {
        all_healthy = false;
        break;
    }
}

// Write this instead
let all_healthy = fleet.iter().all(|s| s.is_healthy());
```

标准库有这些是有原因的。学习词汇，模式就变得显而易见。

---

## 关键要点

> - **Option 和 Result 是单元素集合。** 它们的组合子（`.map()`、`.and_then()`、`.unwrap_or_else()`、`.filter()`、`.zip()`）替换大多数 `if let` / `match` 样板代码。
> - **使用 `bool::then_some()`**——它替换所有情况下的 `if cond { Some(x) } else { None }`。
> - **迭代器链在数据管道上胜出**——filter/map/collect，零可变状态。它们编译成与循环相同的机器码。
> - **循环在多输出状态机上胜出**——当你构建多个集合、在分支中做 I/O 或管理状态转换时。
> - **`?` 操作符是两全其美的**——函数式错误传播与命令式可读性。
> - **在约 4 个适配器处分解链**——为可读性使用命名中间项。过度函数化和函数化不足一样糟糕。
> - **学习标准库词汇**——`.any()`、`.all()`、`.find()`、`.position()`、`.sum()`、`.min_by_key()`——每个都替换多行循环与单个揭示意图的调用。

> **另见：** [第 7 章](ch07-closures-and-higher-order-functions.md) 闭包机制和 `Fn` trait 层次结构。[第 10 章](ch10-error-handling-patterns.md) 错误组合子模式。[第 15 章](ch15-crate-architecture-and-api-design.md) 流式 API 设计。

---

### 练习：从命令式到函数式的重构 ★★（约 30 分钟）

将以下函数从命令式重构为函数式。然后找出一个函数式版本*更差*的地方并解释为什么。

```rust
fn summarize_fleet(fleet: &[Server]) -> FleetSummary {
    let mut healthy = Vec::new();
    let mut degraded = Vec::new();
    let mut failed = Vec::new();
    let mut total_power = 0.0;
    let mut max_temp = f64::NEG_INFINITY;

    for server in fleet {
        match server.health_status() {
            Health::Healthy => healthy.push(server.id.clone()),
            Health::Degraded(reason) => degraded.push((server.id.clone(), reason)),
            Health::Failed(err) => failed.push((server.id.clone(), err)),
        }
        total_power += server.power_draw();
        if server.max_temperature() > max_temp {
            max_temp = server.max_temperature();
        }
    }

    FleetSummary {
        healthy,
        degraded,
        failed,
        avg_power: total_power / fleet.len() as f64,
        max_temp,
    }
}
```

<details>
<summary>🔑 解答</summary>

`total_power` 和 `max_temp` 是简洁的函数式重写：

```rust
fn summarize_fleet(fleet: &[Server]) -> FleetSummary {
    let avg_power: f64 = fleet.iter().map(|s| s.power_draw()).sum::<f64>()
        / fleet.len() as f64;

    let max_temp = fleet.iter()
        .map(|s| s.max_temperature())
        .fold(f64::NEG_INFINITY, f64::max);

    // But the three-way partition is BETTER as a loop.
    // Functional version would require three separate passes
    // or an awkward fold with three mutable accumulators.
    let mut healthy = Vec::new();
    let mut degraded = Vec::new();
    let mut failed = Vec::new();

    for server in fleet {
        match server.health_status() {
            Health::Healthy => healthy.push(server.id.clone()),
            Health::Degraded(reason) => degraded.push((server.id.clone(), reason)),
            Health::Failed(err) => failed.push((server.id.clone(), err)),
        }
    }

    FleetSummary { healthy, degraded, failed, avg_power, max_temp }
}
```

**为什么三路分区用循环更好：** 函数式版本要么需要三个单独的 `.filter().collect()` passes（3 倍迭代），要么需要一个带有三个 `mut Vec` 累加器在元组内的 `.fold()`——这只是用更差的语法重写的循环。命令式单次遍历循环更清晰、更高效、更易于扩展。

</details>

***

