# 基准测试 — 测量重要的事情 🟡

> **你将学到：**
> - 为什么用 `Instant::now()` 进行朴素计时会产生不可靠的结果
> - 使用 Criterion.rs 和更轻量的 Divan 进行统计基准测试
> - 使用 `perf`、flamegraph 和 PGO 分析热点
> - 在 CI 中设置持续基准测试以自动捕获性能回归
>
> **交叉引用：** [发布 Profiles](ch07-release-profiles-and-binary-size.md) — 一旦找到热点，优化二进制文件 · [CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) — 流水线中的基准测试作业 · [代码覆盖率](ch04-code-coverage-seeing-what-tests-miss.md) — 覆盖率告诉你测试了什么，基准测试告诉你什么很快

"我们应该忘记小效率，说大约 97% 的时间：过早优化是万恶之源。
然而我们不应该放弃那关键 3% 中的机会。" — Donald Knuth

困难的部分不是*编写*基准测试——而是编写能产生**有意义的、可重现的、可操作的**
数字的基准测试。本章涵盖的工具和技术让你从"它看起来很快"
到"我们有统计证据表明 PR #347 使解析吞吐量下降了 4.2%"。

### 为什么不用 `std::time::Instant`？

诱惑：

```rust
// ❌ 朴素基准测试 — 不可靠的结果
use std::time::Instant;

fn main() {
    let start = Instant::now();
    let result = parse_device_query_output(&sample_data);
    let elapsed = start.elapsed();
    println!("Parsing took {:?}", elapsed);
    // 问题 1：编译器可能会优化掉 `result`（死代码消除）
    // 问题 2：单个样本 — 无统计意义
    // 问题 3：CPU 频率缩放、热节流、其他进程
    // 问题 4：冷缓存 vs 热缓存未受控制
}
```

手动计时的问题：
1. **死代码消除** — 如果结果未被使用，编译器可能会完全跳过计算。
2. **没有预热** — 第一次运行包括缓存未命中、JIT 效果（Rust 中不相关，但 OS 页面错误适用）和延迟初始化。
3. **无统计分析** — 单次测量不能告诉你方差、异常值或置信区间。
4. **无回归检测** — 你无法与之前的运行进行比较。

### Criterion.rs — 统计基准测试

[Criterion.rs](https://bheisler.github.io/criterion.rs/book/) 是 Rust 微基准测试的事实标准。
它使用统计方法产生可靠的测量并自动检测性能回归。

**设置：**

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports", "cargo_bench_support"] }

[[bench]]
name = "parsing_bench"
harness = false  # 使用 Criterion 的 harness，而不是内置的测试 harness
```

**一个完整的基准测试：**

```rust
// benches/parsing_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion, BenchmarkId};

/// 解析后的 GPU 信息数据类型
#[derive(Debug, Clone)]
struct GpuInfo {
    index: u32,
    name: String,
    temp_c: u32,
    power_w: f64,
}

/// 被测试的函数 — 模拟解析 device-query CSV 输出
fn parse_gpu_csv(input: &str) -> Vec<GpuInfo> {
    input
        .lines()
        .filter(|line| !line.starts_with('#'))
        .filter_map(|line| {
            let fields: Vec<&str> = line.split(", ").collect();
            if fields.len() >= 4 {
                Some(GpuInfo {
                    index: fields[0].parse().ok()?,
                    name: fields[1].to_string(),
                    temp_c: fields[2].parse().ok()?,
                    power_w: fields[3].parse().ok()?,
                })
            } else {
                None
            }
        })
        .collect()
}

fn bench_parse_gpu_csv(c: &mut Criterion) {
    // 有代表性的测试数据
    let small_input = "0, Acme Accel-V1-80GB, 32, 65.5\n\
                       1, Acme Accel-V1-80GB, 34, 67.2\n";

    let large_input = (0..64)
        .map(|i| format!("{i}, Acme Accel-X1-80GB, {}, {:.1}\n", 30 + i % 20, 60.0 + i as f64))
        .collect::<String>();

    c.bench_function("parse_2_gpus", |b| {
        b.iter(|| parse_gpu_csv(black_box(small_input)))
    });

    c.bench_function("parse_64_gpus", |b| {
        b.iter(|| parse_gpu_csv(black_box(&large_input)))
    });
}

criterion_group!(benches, bench_parse_gpu_csv);
criterion_main!(benches);
```

**运行和阅读结果：**

```bash
# 运行所有基准测试
cargo bench

# 按名称运行特定基准测试
cargo bench -- parse_64

# 输出：
# parse_2_gpus        time:   [1.2345 µs  1.2456 µs  1.2578 µs]
#                      ▲            ▲           ▲
#                      │       confidence interval
#                   lower 95%    median    upper 95%
#
# parse_64_gpus       time:   [38.123 µs  38.456 µs  38.812 µs]
#                     change: [-1.2345% -0.5678% +0.1234%] (p = 0.12 > 0.05)
#                     No change in performance detected.
```

**`black_box()` 做什么**：它是一个编译器提示，防止死代码
消除和过度积极的常量折叠。编译器不能透过 `black_box` 看过去，
所以它必须实际计算结果。

### 参数化基准测试和基准测试组

比较多种实现或输入大小：

```rust
// benches/comparison_bench.rs
use criterion::{criterion_group, criterion_main, Criterion, BenchmarkId, Throughput};

fn bench_parsing_strategies(c: &mut Criterion) {
    let mut group = c.benchmark_group("csv_parsing");

    // 测试不同的输入大小
    for num_gpus in [1, 8, 32, 64, 128] {
        let input = generate_gpu_csv(num_gpus);

        // 设置吞吐量以报告字节/秒
        group.throughput(Throughput::Bytes(input.len() as u64));

        group.bench_with_input(
            BenchmarkId::new("split_based", num_gpus),
            &input,
            |b, input| b.iter(|| parse_split(input)),
        );

        group.bench_with_input(
            BenchmarkId::new("regex_based", num_gpus),
            &input,
            |b, input| b.iter(|| parse_regex(input)),
        );

        group.bench_with_input(
            BenchmarkId::new("nom_based", num_gpus),
            &input,
            |b, input| b.iter(|| parse_nom(input)),
        );
    }
    group.finish();
}

criterion_group!(benches, bench_parsing_strategies);
criterion_main!(benches);
```

**输出**：Criterion 在 `target/criterion/report/index.html` 生成一个 HTML 报告，
包含小提琴图、比较图表和回归分析——在浏览器中打开。

### Divan — 更轻量的替代方案

[Divan](https://github.com/nvzqz/divan) 是一个更新的基准测试框架，
使用属性宏而不是 Criterion 的宏 DSL：

```toml
# Cargo.toml
[dev-dependencies]
divan = "0.1"

[[bench]]
name = "parsing_bench"
harness = false
```

```rust
// benches/parsing_bench.rs
use divan::black_box;

const SMALL_INPUT: &str = "0, Acme Accel-V1-80GB, 32, 65.5\n\
                          1, Acme Accel-V1-80GB, 34, 67.2\n";

fn generate_gpu_csv(n: usize) -> String {
    (0..n)
        .map(|i| format!("{i}, Acme Accel-X1-80GB, {}, {:.1}\n", 30 + i % 20, 60.0 + i as f64))
        .collect()
}

fn main() {
    divan::main();
}

#[divan::bench]
fn parse_2_gpus() -> Vec<GpuInfo> {
    parse_gpu_csv(black_box(SMALL_INPUT))
}

#[divan::bench(args = [1, 8, 32, 64, 128])]
fn parse_n_gpus(n: usize) -> Vec<GpuInfo> {
    let input = generate_gpu_csv(n);
    parse_gpu_csv(black_box(&input))
}

// Divan 输出是一个干净的表格：
// ╰─ parse_2_gpus   fastest  │ slowest  │ median   │ mean     │ samples │ iters
//                   1.234 µs │ 1.567 µs │ 1.345 µs │ 1.350 µs │ 100     │ 1600
```

**何时选择 Divan 而不是 Criterion：**
- 更简单的 API（属性宏，更少的样板代码）
- 编译更快（依赖更少）
- 适合开发期间的快速性能检查

**何时选择 Criterion：**
- 跨运行的统计回归检测
- 带图表的 HTML 报告
-成熟的生态系统，更多 CI 集成

### 使用 `perf` 和 Flamegraph 进行性能分析

基准测试告诉你*有多快*——性能分析告诉你*时间花在哪里*。

```bash
# 步骤 1：使用调试信息构建（发布速度，调试符号）
cargo build --release
# 确保调试信息可用：
# [profile.release]
# debug = true          # 临时添加这个以进行性能分析

# 步骤 2：用 perf 记录
perf record -g --call-graph=dwarf ./target/release/diag_tool --run-diagnostics

# 步骤 3：生成 flamegraph
# 安装：cargo install flamegraph
cargo flamegraph --root -- --run-diagnostics
# 打开一个交互式 SVG flamegraph

# 替代：使用 perf + inferno
perf script | inferno-collapse-perf | inferno-flamegraph > flamegraph.svg
```

**阅读 flamegraph：**
- **宽度** = 在该函数中花费的时间（越宽越慢）
- **高度** = 调用栈深度（越高 ≠ 越慢，只是越深）
- **底部** = 入口点，**顶部** = 做实际工作的叶子函数
- 寻找顶部的宽平台——那些是你的热点

**配置文件引导优化 (PGO)：**

```bash
# 步骤 1：用插桩构建
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# 步骤 2：运行有代表性的工作负载
./target/release/diag_tool --run-full   # 生成性能数据

# 步骤 3：合并性能数据
# 使用与 rustc 的 LLVM 版本匹配的 llvm-profdata：
# $(rustc --print sysroot)/lib/rustlib/x86_64-unknown-linux-gnu/bin/llvm-profdata
# 或者如果安装了 llvm-tools：rustup component add llvm-tools
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data/

# 步骤 4：用性能反馈重新构建
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
# 对于计算密集型代码（解析、加密、代码生成）的典型改进：5-20%。
# 对于 I/O 密集型或系统调用密集型代码（如大型项目）改进会小得多，
# 因为 CPU 大部分时间在等待，而不是执行热点循环。
```

> **提示**：在花时间在 PGO 之前，确保你的[发布 profile](ch07-release-profiles-and-binary-size.md)
> 已经启用了 LTO——它通常以更少的努力提供更大的收益。

### `hyperfine` — 快速端到端计时

[`hyperfine`](https://github.com/sharkdp/hyperfine) 对整个命令进行基准测试，
而不是单个函数。它非常适合测量整体二进制文件性能：

```bash
# 安装
cargo install hyperfine
# 或者：sudo apt install hyperfine  (Ubuntu 23.04+)

# 基本基准测试
hyperfine './target/release/diag_tool --run-diagnostics'

# 比较两种实现
hyperfine './target/release/diag_tool_v1 --run-diagnostics' \
          './target/release/diag_tool_v2 --run-diagnostics'

# 预热运行 + 最小迭代次数
hyperfine --warmup 3 --min-runs 10 './target/release/diag_tool --run-all'

# 将结果导出为 JSON 以便 CI 比较
hyperfine --export-json bench.json './target/release/diag_tool --run-all'
```

**何时使用 `hyperfine` vs Criterion：**
- `hyperfine`：整体二进制计时、重构前后的比较、I/O 密集型工作负载
- Criterion：单个函数的微基准测试、统计回归检测

### CI 中的持续基准测试

在性能回归发货之前检测到它们：

```yaml
# .github/workflows/bench.yml
name: Benchmarks

on:
  pull_request:
    paths: ['**/*.rs', 'Cargo.toml', 'Cargo.lock']

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Run benchmarks
        # 需要 criterion = { features = ["cargo_bench_support"] } 用于 --output-format
        run: cargo bench -- --output-format bencher | tee bench_output.txt

      - name: Store benchmark result
        uses: benchmark-action/github-action-benchmark@v1
        with:
          tool: 'cargo'
          output-file-path: bench_output.txt
          github-token: ${{ secrets.GITHUB_TOKEN }}
          auto-push: true
          alert-threshold: '120%'    # 如果慢 20% 则警报
          comment-on-alert: true
          fail-on-alert: true        # 如果检测到回归则阻止 PR
```

**关键 CI 考虑：**
- 使用**专用基准测试运行器**（而不是共享 CI）以获得一致的结果
- 如果使用云 CI，将运行器固定到特定机器类型
- 存储历史数据以检测渐进回归
- 根据工作负载的容差设置阈值（热点 5%，冷路径 20%）

### 应用：解析性能

项目中有几个对性能敏感的解析路径，可以从基准测试中受益：

| 解析热点 | Crate | 为什么重要 |
|------------------|-------|----------------|
| accelerator-query CSV/XML 输出 | `device_diag` | 每次 GPU 调用，最多每次运行 8 次 |
| 传感器事件解析 | `event_log` | 繁忙服务器上的数千条记录 |
| PCIe 拓扑 JSON | `topology_lib` | 复杂的嵌套结构，金色文件验证 |
| 报告 JSON 序列化 | `diag_framework` | 最终报告输出，大小敏感 |
| 配置 JSON 加载 | `config_loader` | 启动延迟 |

**推荐的第一个基准测试** — 已经有金色文件测试数据的拓扑解析器：

```rust
// topology_lib/benches/parse_bench.rs (提议)
use criterion::{criterion_group, criterion_main, Criterion, Throughput};
use std::fs;

fn bench_topology_parse(c: &mut Criterion) {
    let mut group = c.benchmark_group("topology_parse");

    for golden_file in ["S2001", "S1015", "S1035", "S1080"] {
        let path = format!("tests/test_data/{golden_file}.json");
        let data = fs::read_to_string(&path).expect("golden file not found");
        group.throughput(Throughput::Bytes(data.len() as u64));

        group.bench_function(golden_file, |b| {
            b.iter(|| {
                topology_lib::TopologyProfile::from_json_str(
                    criterion::black_box(&data)
                )
            });
        });
    }
    group.finish();
}

criterion_group!(benches, bench_topology_parse);
criterion_main!(benches);
```

### 亲身体验

1. **编写一个 Criterion 基准测试**：选择代码库中的任何解析函数。
   创建一个 `benches/` 目录，设置一个测量以字节/秒为吞吐量的 Criterion 基准测试。
   运行 `cargo bench` 并检查 HTML 报告。

2. **生成一个 flamegraph**：用 `[profile.release]` 中的 `debug = true` 构建你的项目，
   然后运行 `cargo flamegraph -- <your-args>`。识别 flamegraph 顶部最宽的三个栈——那些是你的热点。

3. **用 `hyperfine` 比较**：安装 `hyperfine` 并用不同标志对二进制文件的整体执行时间进行基准测试。
   与 Criterion 的每函数时间进行比较。Criterion 看不到的时间花在哪里？
   （答案：I/O、系统调用、进程启动。）

### 基准测试工具选择

```mermaid
flowchart TD
    START["Want to measure performance?"] --> WHAT{"What level?"}

    WHAT -->|"Single function"| CRITERION["Criterion.rs\nStatistical, regression detection"]
    WHAT -->|"Quick function check"| DIVAN["Divan\nLighter, attribute macros"]
    WHAT -->|"Whole binary"| HYPERFINE["hyperfine\nEnd-to-end, wall-clock"]
    WHAT -->|"Find hot spots"| PERF["perf + flamegraph\nCPU sampling profiler"]

    CRITERION --> CI_BENCH["Continuous benchmarking\nin GitHub Actions"]
    PERF --> OPTIMIZE["Profile-Guided\nOptimization (PGO)"]

    style CRITERION fill:#91e5a3,color:#000
    style DIVAN fill:#91e5a3,color:#000
    style HYPERFINE fill:#e3f2fd,color:#000
    style PERF fill:#ffd43b,color:#000
    style CI_BENCH fill:#e3f2fd,color:#000
    style OPTIMIZE fill:#ffd43b,color:#000
```

### 🏋️ 练习

#### 🟢 练习 1：第一个 Criterion 基准测试

创建一个包含对 10,000 个随机元素的 `Vec<u64>` 进行排序的函数的 crate。
为它编写一个 Criterion 基准测试，然后切换到 `.sort_unstable()` 并在 HTML 报告中观察性能差异。

<details>
<summary>解决方案</summary>

```toml
# Cargo.toml
[[bench]]
name = "sort_bench"
harness = false

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }
rand = "0.8"
```

```rust
// benches/sort_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use rand::Rng;

fn generate_data(n: usize) -> Vec<u64> {
    let mut rng = rand::thread_rng();
    (0..n).map(|_| rng.gen()).collect()
}

fn bench_sort(c: &mut Criterion) {
    let mut group = c.benchmark_group("sort-10k");

    group.bench_function("stable", |b| {
        b.iter_batched(
            || generate_data(10_000),
            |mut data| { data.sort(); black_box(&data); },
            criterion::BatchSize::SmallInput,
        )
    });

    group.bench_function("unstable", |b| {
        b.iter_batched(
            || generate_data(10_000),
            |mut data| { data.sort_unstable(); black_box(&data); },
            criterion::BatchSize::SmallInput,
        )
    });

    group.finish();
}

criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

```bash
cargo bench
open target/criterion/sort-10k/report/index.html
```
</details>

#### 🟡 练习 2：Flamegraph 热点

用 `[profile.release]` 中的 `debug = true` 构建一个项目，然后生成一个 flamegraph。
识别最宽的三个栈。

<details>
<summary>解决方案</summary>

```toml
# Cargo.toml
[profile.release]
debug = true  # 为 flamegraph 保留符号
```

```bash
cargo install flamegraph
cargo flamegraph --release -- <your-args>
# 在浏览器中打开 flamegraph.svg
# 顶部最宽的栈就是你的热点
```
</details>

### 关键要点

- 永远不要用 `Instant::now()` 进行基准测试——使用 Criterion.rs 以获得统计严谨性和回归检测
- `black_box()` 防止编译器优化掉你的基准测试目标
- `hyperfine` 测量整个二进制文件的墙上时间；Criterion 测量单个函数——两者都用
- Flamegraph 显示时间*花在哪里*；基准测试显示*花了多少*时间
- CI 中的持续基准测试在性能回归发货之前捕获它们

---

