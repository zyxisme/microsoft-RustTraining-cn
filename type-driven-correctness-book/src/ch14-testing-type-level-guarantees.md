# 测试类型级保证 🟡

> **你将学到：** 如何测试无效代码 *无法编译*（trybuild）、对验证边界进行模糊测试（proptest）、验证 RAII 不变量，以及通过 `cargo-show-asm` 证明零成本抽象。
>
> **交叉引用：** [ch03](ch03-single-use-types-cryptographic-guarantee.md)（编译失败用于 nonce），[ch07](ch07-validated-boundaries-parse-dont-validate.md)（边界 proptest），[ch05](ch05-protocol-state-machines-type-state-for-r.md)（会话 RAII）

## 测试类型级保证

正确性构造模式将 bug 从运行时转移到编译时。但是你如何 **测试** 无效代码实际上无法编译？你如何确保验证边界在模糊测试下保持？这章涵盖了补充类型级正确性的测试工具。

### 使用 `trybuild` 进行编译失败测试

[`trybuild`](https://crates.io/crates/trybuild) crate 让你断言某些代码 **不应该编译**。这对于在重构中维护类型级不变量至关重要——如果有人意外地将 `Clone` 添加到你的单次使用 `Nonce`，编译失败测试会捕获它。

**设置：**

```toml
# Cargo.toml
[dev-dependencies]
trybuild = "1"
```

**测试文件（`tests/compile_fail.rs`）：**

```rust,ignore
#[test]
fn type_safety_tests() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

**测试用例：Nonce 重用不能编译（`tests/ui/nonce_reuse.rs`）：**

```rust,ignore
// tests/ui/nonce_reuse.rs
use my_crate::Nonce;

fn main() {
    let nonce = Nonce::new();
    encrypt(nonce);
    encrypt(nonce); // 应该失败：使用了已移动的值
}

fn encrypt(_n: Nonce) {}
```

**预期错误（`tests/ui/nonce_reuse.stderr`）：**

```text
error[E0382]: use of moved value: `nonce`
 --> tests/ui/nonce_reuse.rs:6:13
  |
4 |     let nonce = Nonce::new();
  |         ----- move occurs because `nonce` has type `Nonce`, which does not implement the `Copy` trait
5 |     encrypt(nonce);
  |             ----- value moved here
6 |     encrypt(nonce); // should fail: use of moved value
  |             ^^^^^ value used here after move
```

**每章更多编译失败测试用例：**

| 模式（章节） | 测试断言 | 文件 |
|-------------------|---------------|------|
| 单次使用 Nonce（ch03） | 不能两次使用 nonce | `nonce_reuse.rs` |
| 能力令牌（ch04） | 没有令牌不能调用 `admin_op()` | `missing_token.rs` |
| 类型状态（ch05） | 不能在 `Session<Idle>` 上 `send_command()` | `wrong_state.rs` |
| 量纲（ch06） | 不能添加 `Celsius + Rpm` | `unit_mismatch.rs` |
| 密封 Trait（技巧 2） | 外部 crate 不能实现密封 trait | `unseal_attempt.rs` |
| 非穷尽（技巧 3） | 外部匹配没有通配符失败 | `missing_wildcard.rs` |

**CI 集成：**

```yaml
# .github/workflows/ci.yml
- name: Run compile-fail tests
  run: cargo test --test compile_fail
```

### 验证边界的属性测试

验证边界（ch07）解析数据一次并拒绝无效输入。但是你如何知道你的验证捕获了 **所有** 无效输入？使用 [`proptest`](https://crates.io/crates/proptest) 的属性测试生成数千个随机输入来压力测试边界：

```toml
# Cargo.toml
[dev-dependencies]
proptest = "1"
```

```rust,ignore
use proptest::prelude::*;

/// 来自 ch07：ValidFru 包装符合规范的 FRU 有效载荷。
/// 这些测试使用完整的 ch07 ValidFru，带有 board_area()、
/// product_area() 和 format_version() 方法。
/// 注意：ch07 定义了 TryFrom<RawFruData>，所以我们先包装原始字节。

proptest! {
    /// 任何通过验证的字节序列必须可以在不 panic 的情况下使用。
    #[test]
    fn valid_fru_never_panics(data in proptest::collection::vec(any::<u8>(), 0..1024)) {
        if let Ok(fru) = ValidFru::try_from(RawFruData(data)) {
            // 这些在验证的 FRU 上绝不能 panic
            //（来自 ch07 的 ValidFru impl 的方法）：
            let _ = fru.format_version();
            let _ = fru.board_area();
            let _ = fru.product_area();
        }
    }

    /// 往返：format_version 在重新解析后保持不变。
    #[test]
    fn fru_round_trip(data in valid_fru_strategy()) {
        let raw = RawFruData(data.clone());
        let fru = ValidFru::try_from(raw).unwrap();
        let version = fru.format_version();
        // 重新解析相同的字节 — 版本必须相同
        let reparsed = ValidFru::try_from(RawFruData(data)).unwrap();
        prop_assert_eq!(version, reparsed.format_version());
    }
}

/// 自定义策略：生成满足 FRU 规范头部的字节向量。
/// 头部格式匹配 ch07 的 `TryFrom<RawFruData>` 验证：
///   - 字节 0：version = 0x01
///   - 字节 1-6：区域偏移量（×8 = 实际字节偏移量）
///   - 字节 7：校验和（字节 0-7 的和 = 0 mod 256）
/// 主体是随机的，但足够大以使偏移量在范围内。
fn valid_fru_strategy() -> impl Strategy<Value = Vec<u8>> {
    let header = vec![0x01, 0x00, 0x01, 0x02, 0x00, 0x00, 0x00];
    proptest::collection::vec(any::<u8>(), 64..256)
        .prop_map(move |body| {
            let mut fru = header.clone();
            let sum: u8 = fru.iter().fold(0u8, |a, &b| a.wrapping_add(b));
            fru.push(0u8.wrapping_sub(sum));
            fru.extend_from_slice(&body);
            fru
        })
}
```

**正确性构造代码的测试金字塔：**

```text
┌───────────────────────────────────┐
│    编译失败测试 (trybuild)  │ ← "无效代码必须不编译"
├───────────────────────────────────┤
│  属性测试 (proptest/quickcheck) │ ← "有效输入从不 panic"
├───────────────────────────────────┤
│    单元测试 (#[test])           │ ← "特定输入产生预期输出"
├───────────────────────────────────┤
│    类型系统 (模式 ch02–13) │ ← "整类 bug 不可能存在"
└───────────────────────────────────┘
```

### RAII 验证

RAII（技巧 12）保证清理。为了测试这一点，验证 `Drop` impl 实际触发：

```rust,ignore
use std::sync::atomic::{AtomicBool, Ordering};

// 注意：这些测试使用全局 AtomicBool，所以它们不能彼此并行运行。
// 使用 `#[serial_test::serial]` 或用
// `cargo test -- --test-threads=1` 运行。或者，使用每个测试的
// 通过闭包传递的 `Arc<AtomicBool>` 以完全避免全局。
static DROPPED: AtomicBool = AtomicBool::new(false);

struct TestSession;
impl Drop for TestSession {
    fn drop(&mut self) {
        DROPPED.store(true, Ordering::SeqCst);
    }
}

#[test]
fn session_drops_on_early_return() {
    DROPPED.store(false, Ordering::SeqCst);
    let result: Result<(), &str> = (|| {
        let _session = TestSession;
        Err("simulated failure")?;
        Ok(())
    })();
    assert!(result.is_err());
    assert!(DROPPED.load(Ordering::SeqCst), "Drop must fire on early return");
}

#[test]
fn session_drops_on_panic() {
    DROPPED.store(false, Ordering::SeqCst);
    let result = std::panic::catch_unwind(|| {
        let _session = TestSession;
        panic!("simulated panic");
    });
    assert!(result.is_err());
    assert!(DROPPED.load(Ordering::SeqCst), "Drop must fire on panic");
}
```

### 应用到你的代码库

这是向工作空间添类型级测试的优先级计划：

| Crate | 测试类型 | 测试什么 |
|-------|-----------|-------------|
| `protocol_lib` | 编译失败 | `Session<Idle>` 不能 `send_command()` |
| `protocol_lib` | 属性 | 任何字节序列 → `TryFrom` 要么成功要么返回 Err（不 panic） |
| `thermal_diag` | 编译失败 | 没有 `HasSpi` 混合不能构造 `FanReading` |
| `accel_diag` | 属性 | GPU 传感器解析：随机字节 → 验证或拒绝 |
| `config_loader` | 属性 | 随机字符串 → `FromStr` 用于 `DiagLevel` 从不 panic |
| `pci_topology` | 编译失败 | `Register<Width16>` 不能在期望 `Width32` 的地方传递 |
| `event_handler` | 编译失败 | 审计令牌不能被克隆 |
| `diag_framework` | 编译失败 | `DerBuilder<Missing, _>` 不能调用 `finish()` |

### 零成本抽象：通过汇编证明

一个常见担忧："newtype 和幽灵类型会增加运行时开销吗？"
答案是 **否** — 它们编译为与原始原语相同的汇编。以下是验证方法：

**设置：**

```bash
cargo install cargo-show-asm
```

**示例：Newtype vs 原始 u32：**

```rust,ignore
// src/lib.rs
#[derive(Clone, Copy)]
pub struct Rpm(pub u32);

#[derive(Clone, Copy)]
pub struct Celsius(pub f64);

// Newtype 算术
#[inline(never)]
pub fn add_rpm(a: Rpm, b: Rpm) -> Rpm {
    Rpm(a.0 + b.0)
}

// 原始算术（用于比较）
#[inline(never)]
pub fn add_raw(a: u32, b: u32) -> u32 {
    a + b
}
```

**运行：**

```bash
cargo asm my_crate::add_rpm
cargo asm my_crate::add_raw
```

**结果 — 相同的汇编：**

```asm
; add_rpm (newtype)           ; add_raw (raw u32)
my_crate::add_rpm:            my_crate::add_raw:
  lea eax, [rdi + rsi]         lea eax, [rdi + rsi]
  ret                          ret
```

`Rpm` 包装器在编译时完全擦除。这同样适用于 `PhantomData<S>`（零字节）、`ZST` 令牌（零字节）以及本指南中使用的所有其他类型级标记。

**验证你自己的类型：**

```bash
# 显示特定函数的汇编
cargo asm --lib ipmi_lib::session::execute

# 显示 PhantomData 添加零字节
cargo asm --lib --rust ipmi_lib::session::IpmiSession
```

> **关键要点：** 本指南中的每个模式都有 **零运行时成本**。
> 类型系统完成所有工作，在编译期间完全擦除。
> 你获得 Haskell 的安全性与 C 的性能。

## 关键要点

1. **trybuild 测试无效代码不会编译** — 对于在重构中维护类型级不变量至关重要。
2. **proptest 对验证边界进行模糊测试** — 生成数千个随机输入来压力测试 `TryFrom` 实现。
3. **RAII 验证测试 Drop 运行** — Arc 计数器或模拟标志证明清理发生了。
4. **cargo-show-asm 证明零成本** — 幽灵类型、ZST 和 newtype 产生与原始 C 相同的汇编。
5. **为每个"不可能"状态添加编译失败测试** — 如果有人意外地在单次使用类型上派生 `Clone`，测试会捕获它。

---

*Rust 类型驱动正确性 完*
