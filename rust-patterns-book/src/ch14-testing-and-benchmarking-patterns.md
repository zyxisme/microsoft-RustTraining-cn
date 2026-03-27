# 13. 测试和基准测试模式 🟢

> **你将学到：**
> - Rust 的三层测试：单元测试、集成测试和文档测试
> - 使用 proptest 进行属性测试以发现边界情况
> - 使用 criterion 进行可靠的性能基准测试
> - 无需重量级框架的 Mock 策略

## 单元测试、集成测试、文档测试

Rust 有三层测试内置于语言中：

```rust
// --- 单元测试：与代码在同一文件中 ---
pub fn factorial(n: u64) -> u64 {
    (1..=n).product()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_factorial_zero() {
        // (1..=0).product() 返回 1 — 空范围的乘法恒等式
        assert_eq!(factorial(0), 1);
    }

    #[test]
    fn test_factorial_five() {
        assert_eq!(factorial(5), 120);
    }

    #[test]
    #[cfg(debug_assertions)] // 溢出检查仅在调试模式下启用
    #[should_panic(expected = "overflow")]
    fn test_factorial_overflow() {
        // ⚠️ 此测试仅在调试模式下通过（启用溢出检查）。
        // 在发布模式（`cargo test --release`）下，u64 算术会静默环绕
        // 不会发生 panic。若要在发布模式下安全，请使用 `checked_mul` 或
        // `overflow-checks = true` 配置。
        factorial(100); // 应该在溢出时 panic
    }

    #[test]
    fn test_with_result() -> Result<(), Box<dyn std::error::Error>> {
        // 测试可以返回 Result — ? 在内部工作！
        let value: u64 = "42".parse()?;
        assert_eq!(value, 42);
        Ok(())
    }
}
```

```rust
// --- 集成测试：在 tests/ 目录中 ---
// tests/integration_test.rs
// 这些只测试你的 crate 的公共 API

use my_crate::factorial;

#[test]
fn test_factorial_from_outside() {
    assert_eq!(factorial(10), 3_628_800);
}
```

```rust
// --- 文档测试：在文档注释中 ---
/// 计算 `n` 的阶乘。
///
/// # 示例
///
/// ```
/// use my_crate::factorial;
/// assert_eq!(factorial(5), 120);
/// ```
///
/// # Panic
///
/// 如果结果溢出 `u64` 则 panic。
///
/// ```should_panic
/// my_crate::factorial(100);
/// ```
pub fn factorial(n: u64) -> u64 {
    (1..=n).product()
}
// 文档测试由 `cargo test` 编译和运行 — 它们保持示例的正确性。
```

### 测试夹具和设置

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // 共享设置 — 创建一个辅助函数
    fn setup_database() -> TestDb {
        let db = TestDb::new_in_memory();
        db.run_migrations();
        db.seed_test_data();
        db
    }

    #[test]
    fn test_user_creation() {
        let db = setup_database();
        let user = db.create_user("Alice", "alice@test.com").unwrap();
        assert_eq!(user.name, "Alice");
    }

    #[test]
    fn test_user_deletion() {
        let db = setup_database();
        db.create_user("Bob", "bob@test.com").unwrap();
        assert!(db.delete_user("Bob").is_ok());
        assert!(db.get_user("Bob").is_none());
    }

    // 使用 Drop 进行清理（RAII）：
    struct TempDir {
        path: std::path::PathBuf,
    }

    impl TempDir {
        fn new() -> Self {
            // Cargo.toml: rand = "0.8"
            let path = std::env::temp_dir().join(format!("test_{}", rand::random::<u32>()));
            std::fs::create_dir_all(&path).unwrap();
            TempDir { path }
        }
    }

    impl Drop for TempDir {
        fn drop(&mut self) {
            let _ = std::fs::remove_dir_all(&self.path);
        }
    }

    #[test]
    fn test_file_operations() {
        let dir = TempDir::new(); // 已创建
        std::fs::write(dir.path.join("test.txt"), "hello").unwrap();
        assert!(dir.path.join("test.txt").exists());
    } // dir 在这里 drop → 临时目录被清理
}
```

### 属性测试（proptest）

不是测试特定值，而是测试*属性*，这些属性应该始终成立：

```rust
// Cargo.toml: proptest = "1"
use proptest::prelude::*;

fn reverse(v: &[i32]) -> Vec<i32> {
    v.iter().rev().cloned().collect()
}

proptest! {
    #[test]
    fn test_reverse_twice_is_identity(v in prop::collection::vec(any::<i32>(), 0..100)) {
        // 属性：反转两次得到原始值
        assert_eq!(reverse(&reverse(&v)), v);
    }

    #[test]
    fn test_reverse_preserves_length(v in prop::collection::vec(any::<i32>(), 0..100)) {
        assert_eq!(reverse(&v).len(), v.len());
    }

    #[test]
    fn test_sort_is_idempotent(mut v in prop::collection::vec(any::<i32>(), 0..100)) {
        v.sort();
        let sorted_once = v.clone();
        v.sort();
        assert_eq!(v, sorted_once); // 排序两次 = 排序一次
    }

    #[test]
    fn test_parse_roundtrip(x in any::<f64>().prop_filter("finite", |x| x.is_finite())) {
        // 属性：格式化后再解析得到相同的值
        let s = format!("{x}");
        let parsed: f64 = s.parse().unwrap();
        prop_assert!((x - parsed).abs() < f64::EPSILON);
    }
}
```

> **何时使用 proptest**：当你要测试的函数输入空间很大，
> 并且希望确信它对你没想到的边界情况也能工作。
> proptest 生成数百个随机输入并将失败缩小到最小复现案例。

### 使用 criterion 进行基准测试

```rust
// Cargo.toml:
// [dev-dependencies]
// criterion = { version = "0.5", features = ["html_reports"] }
//
// [[bench]]
// name = "my_benchmarks"
// harness = false

// benches/my_benchmarks.rs
use criterion::{criterion_group, criterion_main, Criterion, black_box};

fn fibonacci(n: u64) -> u64 {
    match n {
        0 | 1 => n,
        _ => fibonacci(n - 1) + fibonacci(n - 2),
    }
}

fn bench_fibonacci(c: &mut Criterion) {
    c.bench_function("fibonacci 20", |b| {
        b.iter(|| fibonacci(black_box(20)))
    });

    // 比较不同实现：
    let mut group = c.benchmark_group("fibonacci_compare");
    for size in [10, 15, 20, 25] {
        group.bench_with_input(
            criterion::BenchmarkId::from_parameter(size),
            &size,
            |b, &size| b.iter(|| fibonacci(black_box(size))),
        );
    }
    group.finish();
}

criterion_group!(benches, bench_fibonacci);
criterion_main!(benches);

// 运行：cargo bench
// 在 target/criterion/ 生成 HTML 报告
```

### 无框架的 Mock 策略

Rust 的 trait 系统提供自然的依赖注入 — 无需 mock 框架：

```rust
// 将行为定义为 trait
trait Clock {
    fn now(&self) -> std::time::Instant;
}

trait HttpClient {
    fn get(&self, url: &str) -> Result<String, String>;
}

// 生产实现
struct RealClock;
impl Clock for RealClock {
    fn now(&self) -> std::time::Instant { std::time::Instant::now() }
}

// 服务依赖于抽象
struct CacheService<C: Clock, H: HttpClient> {
    clock: C,
    client: H,
    ttl: std::time::Duration,
}

impl<C: Clock, H: HttpClient> CacheService<C, H> {
    fn fetch(&self, url: &str) -> Result<String, String> {
        // 使用 self.clock 和 self.client — 可注入的
        self.client.get(url)
    }
}

// 使用 mock 实现进行测试 — 无需框架！
#[cfg(test)]
mod tests {
    use super::*;

    struct MockClock {
        fixed_time: std::time::Instant,
    }
    impl Clock for MockClock {
        fn now(&self) -> std::time::Instant { self.fixed_time }
    }

    struct MockHttpClient {
        response: String,
    }
    impl HttpClient for MockHttpClient {
        fn get(&self, _url: &str) -> Result<String, String> {
            Ok(self.response.clone())
        }
    }

    #[test]
    fn test_cache_service() {
        let service = CacheService {
            clock: MockClock { fixed_time: std::time::Instant::now() },
            client: MockHttpClient { response: "cached data".into() },
            ttl: std::time::Duration::from_secs(300),
        };

        assert_eq!(service.fetch("http://example.com").unwrap(), "cached data");
    }
}
```

> **测试理念**：在集成测试中优先使用真实依赖，
> 在单元测试中使用基于 trait 的 mock。
> 除非依赖图复杂，否则避免 mock 框架 —
> Rust 的 trait 泛型在大多情况下自然处理。

> **关键要点 — 测试**
> - 文档测试（`///`）既是文档又是回归测试 — 它们被编译和运行
> - `proptest` 生成随机输入以发现你永远不会手动编写的边界情况
> - `criterion` 提供带有 HTML 报告的统计严格基准
> - 通过 trait 泛型 + 测试替身进行 mock，而不是 mock 框架

> **另请参阅：** [第 12 章 — 宏](ch12-macros-code-that-writes-code.md) 用于测试宏生成的代码。[第 14 章 — API 设计](ch14-crate-architecture-and-api-design.md) 用于模块布局如何影响测试组织。

---

### 练习：使用 proptest 进行属性测试 ★★（约 25 分钟）

编写一个维护排序不变量的 `SortedVec<T: Ord>` 包装器。
使用 `proptest` 验证：
1. 在任何插入序列之后，内部 vec 始终是排序的
2. `contains()` 与 stdlib `Vec::contains()` 一致
3. 长度等于插入次数

<details>
<summary>🔑 解决方案</summary>

```rust,ignore
#[derive(Debug)]
struct SortedVec<T: Ord> {
    inner: Vec<T>,
}

impl<T: Ord> SortedVec<T> {
    fn new() -> Self { SortedVec { inner: Vec::new() } }

    fn insert(&mut self, value: T) {
        let pos = self.inner.binary_search(&value).unwrap_or_else(|p| p);
        self.inner.insert(pos, value);
    }

    fn contains(&self, value: &T) -> bool {
        self.inner.binary_search(value).is_ok()
    }

    fn len(&self) -> usize { self.inner.len() }
    fn as_slice(&self) -> &[T] { &self.inner }
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn always_sorted(values in proptest::collection::vec(-1000i32..1000, 0..100)) {
            let mut sv = SortedVec::new();
            for v in &values {
                sv.insert(*v);
            }
            for w in sv.as_slice().windows(2) {
                prop_assert!(w[0] <= w[1]);
            }
            prop_assert_eq!(sv.len(), values.len());
        }

        #[test]
        fn contains_matches_stdlib(values in proptest::collection::vec(0i32..50, 1..30)) {
            let mut sv = SortedVec::new();
            for v in &values {
                sv.insert(*v);
            }
            for v in &values {
                prop_assert!(sv.contains(v));
            }
            prop_assert!(!sv.contains(&9999));
        }
    }
}
```

</details>

***

