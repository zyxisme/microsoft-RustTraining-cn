## 何时以及为何使用 Unsafe

> **你将学到：** `unsafe` 允许什么以及为什么存在，用 PyO3 编写 Python 扩展（Python 开发者的杀手级功能），
> Rust 的测试框架 vs pytest，用 mockall 进行模拟，以及基准测试。
>
> **难度：** 🔴 高级

Rust 中的 `unsafe` 是一个逃生舱 — 它告诉编译器："我做了一些你无法验证的事情，但我保证它是正确的。" Python 没有等价物，因为
Python 从不给你直接内存访问的权限。

```mermaid
flowchart TB
    subgraph Safe ["安全 Rust（99% 的代码）"]
        S1["你的应用逻辑"]
        S2["pub fn safe_api\(&self\) -> Result"]
    end
    subgraph Unsafe ["unsafe 块（最小化，审计过）"]
        U1["原始指针解引用"]
        U2["FFI 调用 C/Python"]
    end
    subgraph External ["外部（C / Python / OS）"]
        E1["libc / PyO3 / 系统调用"]
    end
    S1 --> S2
    S2 --> U1
    S2 --> U2
    U1 --> E1
    U2 --> E1
    style Safe fill:#d4edda,stroke:#28a745
    style Unsafe fill:#fff3cd,stroke:#ffc107
    style External fill:#f8d7da,stroke:#dc3545
```

> **模式**：安全 API 包装一个小 `unsafe` 块。调用者永远看不到 `unsafe`。Python 的 `ctypes` 没有这种边界 — 每个 FFI 调用都隐式是 unsafe。
>
> 📌 **另见**：[第 13 章 — 并发](ch13-concurrency.md) 涵盖了 `Send`/`Sync` trait，它们是 `unsafe` 自动 trait，编译器检查它们以保证线程安全。

### unsafe 允许什么
```rust
// unsafe 允许你做安全 Rust 禁止的五件事：
// 1. 解引用原始指针
// 2. 调用 unsafe 函数/方法
// 3. 访问可变静态变量
// 4. 实现 unsafe trait
// 5. 访问 union 字段

// 示例：调用 C 函数
extern "C" {
    fn abs(input: i32) -> i32;
}

fn main() {
    let result = unsafe { abs(-42) };  // 安全 Rust 无法验证 C 代码
    println!("{result}");               // 42
}
```

### 何时使用 unsafe
```rust
// 1. FFI — 调用 C 库（最常见的原因）
// 2. 性能关键的内部循环（罕见）
// 3. 借用检查器无法表达的数据结构（罕见）

// 作为 Python 开发者，你主要会在以下地方遇到 unsafe：
// - PyO3 内部（Python ↔ Rust 桥接）
// - C 库绑定
// - 低级系统调用

// 经验法则：如果你在写应用代码（不是库代码），
// 你几乎永远不需要 unsafe。如果你认为自己需要，
// 先问问 Rust 社区 — 通常有安全的替代方案。
```

***

## PyO3：Python 的 Rust 扩展

PyO3 是 Python 和 Rust 之间的桥梁。它让你写的 Rust 函数和类可以从 Python 调用 — 非常适合替换慢速的 Python 热点代码。

### 在 Rust 中创建 Python 扩展
```bash
# 设置
pip install maturin    # Rust Python 扩展的构建工具
maturin init           # 创建项目结构

# 项目结构：
# my_extension/
# ├── Cargo.toml
# ├── pyproject.toml
# └── src/
#     └── lib.rs
```

```toml
# Cargo.toml
[package]
name = "my_extension"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]    # Python 的共享库

[dependencies]
pyo3 = { version = "0.22", features = ["extension-module"] }
```

```rust
// src/lib.rs — 可从 Python 调用的 Rust 函数
use pyo3::prelude::*;

/// 一个用 Rust 编写的高速斐波那契函数。
#[pyfunction]
fn fibonacci(n: u64) -> u64 {
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 0..n {
        let temp = b;
        b = a.wrapping_add(b);
        a = temp;
    }
    a
}

/// 找出 n 以内的所有质数（埃拉托斯特尼筛法）。
#[pyfunction]
fn primes_up_to(n: usize) -> Vec<usize> {
    let mut is_prime = vec![true; n + 1];
    is_prime[0] = false;
    if n > 0 { is_prime[1] = false; }
    for i in 2..=((n as f64).sqrt() as usize) {
        if is_prime[i] {
            for j in (i * i..=n).step_by(i) {
                is_prime[j] = false;
            }
        }
    }
    (2..=n).filter(|&i| is_prime[i]).collect()
}

/// 一个可从 Python 使用的 Rust 类。
#[pyclass]
struct Counter {
    value: i64,
}

#[pymethods]
impl Counter {
    #[new]
    fn new(start: i64) -> Self {
        Counter { value: start }
    }

    fn increment(&mut self) {
        self.value += 1;
    }

    fn get_value(&self) -> i64 {
        self.value
    }

    fn __repr__(&self) -> String {
        format!("Counter(value={})", self.value)
    }
}

/// Python 模块定义。
#[pymodule]
fn my_extension(m: &Bound<'_, PyModule>) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(fibonacci, m)?)?;
    m.add_function(wrap_pyfunction!(primes_up_to, m)?)?;
    m.add_class::<Counter>()?;
    Ok(())
}
```

### 从 Python 使用
```bash
# 构建和安装：
maturin develop --release   # 构建并安装到当前 venv
```

```python
# Python — 像使用任何 Python 模块一样使用 Rust 扩展
import my_extension

# 调用 Rust 函数
result = my_extension.fibonacci(50)
print(result)  # 12586269025 — 微秒级计算

# 使用 Rust 类
counter = my_extension.Counter(0)
counter.increment()
counter.increment()
print(counter.get_value())  # 2
print(counter)              # Counter(value=2)

# 性能对比：
import time

# Python 版本
def py_primes(n):
    sieve = [True] * (n + 1)
    for i in range(2, int(n**0.5) + 1):
        if sieve[i]:
            for j in range(i*i, n+1, i):
                sieve[j] = False
    return [i for i in range(2, n+1) if sieve[i]]

start = time.perf_counter()
py_result = py_primes(10_000_000)
py_time = time.perf_counter() - start

start = time.perf_counter()
rs_result = my_extension.primes_up_to(10_000_000)
rs_time = time.perf_counter() - start

print(f"Python: {py_time:.3f}s")    # 约 3.5s
print(f"Rust:   {rs_time:.3f}s")    # 约 0.05s — 70 倍快！
print(f"Same results: {py_result == rs_result}")  # True
```

### PyO3 快速参考

| Python 概念 | PyO3 属性 | 说明 |
|-------------|----------------|------|
| 函数 | `#[pyfunction]` | 暴露给 Python |
| 类 | `#[pyclass]` | Python 可见的类 |
| 方法 | `#[pymethods]` | pyclass 上的方法 |
| `__init__` | `#[new]` | 构造函数 |
| `__repr__` | `fn __repr__()` | 字符串表示 |
| `__str__` | `fn __str__()` | 显示字符串 |
| `__len__` | `fn __len__()` | 长度 |
| `__getitem__` | `fn __getitem__()` | 索引 |
| 属性 | `#[getter]` / `#[setter]` | 属性访问 |
| 静态方法 | `#[staticmethod]` | 无 self |
| 类方法 | `#[classmethod]` | 接收 cls |

### FFI 安全模式

当将 Rust 暴露给 Python（通过 PyO3 或原始 C FFI）时，这些规则可以防止最常见的 bug：

1. **永远不要让 panic 跨越 FFI 边界** — Rust panic 解栈到 Python（或 C）是 **未定义行为**。PyO3 自动为 `#[pyfunction]` 处理这个问题，但原始 `extern "C"` 函数需要显式保护：
    ```rust
    #[no_mangle]
    pub extern "C" fn raw_ffi_function() -> i32 {
        match std::panic::catch_unwind(|| {
            // 实际逻辑
            42
        }) {
            Ok(result) => result,
            Err(_) => -1,  // 返回错误码而不是 panic 到 C/Python
        }
    }
    ```

2. **`#[repr(C)]` 用于共享结构** — 如果 Python/C 直接读取结构字段，你 **必须** 使用 `#[repr(C)]` 来保证 C 兼容的布局。如果你传递 opaque 指针（PyO3 对 `#[pyclass]` 这样做），就不需要。

3. **`extern "C"`** — 原始 FFI 函数需要它，以便调用约定与 C/Python 期望的匹配。PyO3 的 `#[pyfunction]` 为你处理这个。

> **PyO3 优势**：PyO3 为你包装了大部分安全考虑 — panic 捕获、类型转换、GIL 管理。除非你有特定原因不要使用 PyO3，否则优先选择 PyO3 而不是原始 FFI。

***


<!-- ch14a: Testing -->
## 单元测试 vs pytest

### Python 用 pytest 测试
```python
# test_calculator.py
import pytest
from calculator import add, divide

def test_add():
    assert add(2, 3) == 5

def test_add_negative():
    assert add(-1, 1) == 0

def test_divide():
    assert divide(10, 2) == 5.0

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError):
        divide(1, 0)

# 参数化测试
@pytest.mark.parametrize("a,b,expected", [
    (1, 2, 3),
    (0, 0, 0),
    (-1, -1, -2),
    (100, 200, 300),
])
def test_add_parametrized(a, b, expected):
    assert add(a, b) == expected

# Fixtures
@pytest.fixture
def sample_data():
    return [1, 2, 3, 4, 5]

def test_sum(sample_data):
    assert sum(sample_data) == 15
```

```bash
# 运行测试
pytest                      # 运行所有测试
pytest test_calculator.py   # 运行一个文件
pytest -k "test_add"        # 运行匹配的测试
pytest -v                   # 详细输出
pytest --tb=short           # 简短回溯
```

### Rust 内置测试
```rust
// src/calculator.rs — 测试生活在同一个文件中！
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// 测试放在 #[cfg(test)] 模块中 — 只在 `cargo test` 时编译
#[cfg(test)]
mod tests {
    use super::*;  // 从父模块导入一切

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_negative() {
        assert_eq!(add(-1, 1), 0);
    }

    #[test]
    fn test_divide() {
        assert_eq!(divide(10.0, 2.0), Ok(5.0));
    }

    #[test]
    fn test_divide_by_zero() {
        assert!(divide(1.0, 0.0).is_err());
    }

    // 测试某些东西 panic（像 pytest.raises）
    #[test]
    #[should_panic(expected = "out of bounds")]
    fn test_out_of_bounds() {
        let v = vec![1, 2, 3];
        let _ = v[99];  // Panic
    }
}
```

```bash
# 运行测试
cargo test                         # 运行所有测试
cargo test test_add                # 运行匹配的测试
cargo test -- --nocapture          # 显示 println! 输出
cargo test -p my_crate             # 测试工作区中的一个 crate
cargo test -- --test-threads=1     # 顺序运行（用于有副作用的测试）
```

### 测试快速参考

| pytest | Rust | 说明 |
|--------|------|------|
| `assert x == y` | `assert_eq!(x, y)` | 相等 |
| `assert x != y` | `assert_ne!(x, y)` | 不等 |
| `assert condition` | `assert!(condition)` | 布尔 |
| `assert condition, "msg"` | `assert!(condition, "msg")` | 带消息 |
| `pytest.raises(E)` | `#[should_panic]` | 期望 panic |
| `@pytest.fixture` | 在测试或辅助函数中设置 | 无内置 fixtures |
| `@pytest.mark.parametrize` | `rstest` crate | 参数化测试 |
| `conftest.py` | `tests/common/mod.rs` | 共享测试辅助 |
| `pytest.skip()` | `#[ignore]` | 跳过测试 |
| `tmp_path` fixture | `tempfile` crate | 临时目录 |

***

## 使用 rstest 的参数化测试
```rust
// Cargo.toml: rstest = "0.23"

use rstest::rstest;

// 像 @pytest.mark.parametrize
#[rstest]
#[case(1, 2, 3)]
#[case(0, 0, 0)]
#[case(-1, -1, -2)]
#[case(100, 200, 300)]
fn test_add(#[case] a: i32, #[case] b: i32, #[case] expected: i32) {
    assert_eq!(add(a, b), expected);
}

// 像 @pytest.fixture
use rstest::fixture;

#[fixture]
fn sample_data() -> Vec<i32> {
    vec![1, 2, 3, 4, 5]
}

#[rstest]
fn test_sum(sample_data: Vec<i32>) {
    assert_eq!(sample_data.iter().sum::<i32>(), 15);
}
```

***

## 使用 mockall 进行模拟
```python
# Python — 用 unittest.mock 进行模拟
from unittest.mock import Mock, patch

def test_fetch_user():
    mock_db = Mock()
    mock_db.get_user.return_value = {"name": "Alice"}

    result = fetch_user_name(mock_db, 1)
    assert result == "Alice"
    mock_db.get_user.assert_called_once_with(1)
```

```rust
// Rust — 用 mockall crate 进行模拟
// Cargo.toml: mockall = "0.13"

use mockall::{automock, predicate::*};

#[automock]                          // 自动生成 MockDatabase
trait Database {
    fn get_user(&self, id: i64) -> Option<User>;
}

fn fetch_user_name(db: &dyn Database, id: i64) -> Option<String> {
    db.get_user(id).map(|u| u.name)
}

#[test]
fn test_fetch_user() {
    let mut mock = MockDatabase::new();
    mock.expect_get_user()
        .with(eq(1))                   // assert_called_with(1)
        .times(1)                      // assert_called_once
        .returning(|_| Some(User { name: "Alice".into() }));

    let result = fetch_user_name(&mock, 1);
    assert_eq!(result, Some("Alice".to_string()));
}
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：unsafe 外的安全包装器</strong>（点击展开）</summary>

**挑战**：编写一个安全函数 `split_at_mid`，接受一个 `&mut [i32]`，返回两个在中间点分割的可变切片 `(&mut [i32], &mut [i32])`。内部使用 `unsafe` 和原始指针（模拟 `split_at_mut` 的行为）。然后包装在安全的 API 中。

<details>
<summary>🔑 解决方案</summary>

```rust
fn split_at_mid(slice: &mut [i32]) -> (&mut [i32], &mut [i32]) {
    let mid = slice.len() / 2;
    let ptr = slice.as_mut_ptr();
    let len = slice.len();

    assert!(mid <= len); // unsafe 之前的安全检查

    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut data = vec![1, 2, 3, 4, 5, 6];
    let (left, right) = split_at_mid(&mut data);
    left[0] = 99;
    right[0] = 88;
    println!("left: {left:?}, right: {right:?}");
    // left: [99, 2, 3], right: [88, 5, 6]
}
```

**关键要点**：`unsafe` 块很小，受 `assert!` 保护。公共 API 完全安全 — 调用者永远看不到 `unsafe`。这是 Rust 模式：unsafe 内部实现，安全接口。Python 的 `ctypes` 不能给你这样的保证。

</details>
</details>

***


