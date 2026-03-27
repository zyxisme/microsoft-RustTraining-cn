# 来自前线的技巧 🟡

> **你将学到：**
> - 经过实战测试的模式，不能整齐地放入一章
> - 常见陷阱及其修复 — 从 CI 抖动到二进制文件膨胀
> - 今天可以应用于任何 Rust 项目的快速技巧
>
> **交叉引用：** 本书每一章 — 这些技巧贯穿所有主题

本章收集了在生产 Rust 代码库中反复出现的工程模式。每个技巧都是独立的——可以按任意顺序阅读。

---

### 1. `deny(warnings)` 陷阱

**问题**：源代码中的 `#![deny(warnings)]` 在 Clippy 添加新 lint 时破坏构建 —
昨天编译的代码今天失败。

**修复**：在 CI 中使用 `CARGO_ENCODED_RUSTFLAGS` 而不是源级别属性：

```yaml
# CI：在不触及源代码的情况下将警告视为错误
env:
  CARGO_ENCODED_RUSTFLAGS: "-Dwarnings"
```

或者使用 `[workspace.lints]` 进行更细粒度的控制：

```toml
# Cargo.toml
[workspace.lints.rust]
unsafe_code = "deny"

[workspace.lints.clippy]
all = { level = "deny", priority = -1 }
pedantic = { level = "warn", priority = -1 }
```

> 参见[编译时工具，工作空间 Lint](ch08-compile-time-and-developer-tools.md) 了解完整模式。

---

### 2. 编译一次，测试到处

**问题**：`cargo test` 在 `--lib`、`--doc` 和 `--test` 之间切换时重新编译，因为它们使用不同的 profile。

**修复**：使用 `cargo nextest` 运行单元/集成测试，单独运行 doc-tests：

```bash
cargo nextest run --workspace        # 快：并行，缓存
cargo test --workspace --doc         # Doc-tests（nextest 无法运行这些）
```

> 参见[编译时工具](ch08-compile-time-and-developer-tools.md) 了解 `cargo-nextest` 设置。

---

### 3. 特性标志卫生性

**问题**：一个库 crate 有 `default = ["std"]` 但没有人用 `--no-default-features` 测试。
有一天嵌入式用户报告它无法编译。

**修复**：在 CI 中添加 `cargo-hack`：

```yaml
- name: Feature matrix
  run: |
    cargo hack check --each-feature --no-dev-deps
    cargo check --no-default-features
    cargo check --all-features
```

> 参见[`no_std` 和特性验证](ch09-no-std-and-feature-verification.md) 了解完整模式。

---

### 4. 锁文件辩论 — 提交还是忽略？

**经验法则：**

| Crate 类型 | 提交 `Cargo.lock`？ | 为什么 |
|------------|---------------------|---------|
| 二进制 / 应用程序 | **是** | 可重现构建 |
| 库 | **否**（`.gitignore`） | 让下游选择版本 |
| 同时包含两者的工作空间 | **是** | 二进制优先 |

添加 CI 检查以确保锁文件保持最新：

```yaml
- name: Check lock file
  run: cargo update --locked  # 如果 Cargo.lock 过时则失败
```

---

### 5. 优化依赖的 Debug 构建

**问题**：Debug 构建很慢，因为依赖（特别是 `serde`、`regex`）没有被优化。

**修复**：在 dev profile 中优化依赖，同时保持你的代码未优化以实现快速重新编译：

```toml
# Cargo.toml
[profile.dev.package."*"]
opt-level = 2  # 在 dev 模式下优化所有依赖
```

这会稍微减慢第一次构建，但使开发期间的运行时显著更快。对于数据库支持的服务和解析器特别有效。

> 参见[发布 Profiles](ch07-release-profiles-and-binary-size.md) 了解每个 crate 的 profile 覆盖。

---

### 6. CI 缓存抖动

**问题**：`Swatinem/rust-cache@v2` 在每个 PR 上保存新缓存，膨胀存储并减慢恢复时间。

**修复**：只从 `main` 保存缓存，从任何地方恢复：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

对于有多个二进制文件的工作空间，添加 `shared-key`：

```yaml
- uses: Swatinem/rust-cache@v2
  with:
    shared-key: "ci-${{ matrix.target }}"
    save-if: ${{ github.ref == 'refs/heads/main' }}
```

> 参见 [CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) 了解完整工作流。

---

### 7. `RUSTFLAGS` vs `CARGO_ENCODED_RUSTFLAGS`

**问题**：`RUSTFLAGS="-Dwarnings"` 适用于*一切* — 包括构建脚本和 proc-macros。
`serde_derive` 的 build.rs 中的警告会破坏你的 CI。

**修复**：使用 `CARGO_ENCODED_RUSTFLAGS`，它只适用于顶层 crate：

```bash
# 糟糕 — 因第三方构建脚本警告而中断
RUSTFLAGS="-Dwarnings" cargo build

# 好 — 只影响你的 crate
CARGO_ENCODED_RUSTFLAGS="-Dwarnings" cargo build

# 同样好 — 工作空间 lint（Cargo.toml）
[workspace.lints.rust]
warnings = "deny"
```

---

### 8. 使用 `SOURCE_DATE_EPOCH` 实现可重现构建

**问题**：在 `build.rs` 中嵌入 `chrono::Utc::now()` 使构建不可重现 — 每次构建产生不同的二进制哈希。

**修复**：尊重 `SOURCE_DATE_EPOCH`：

```rust
// build.rs
let timestamp = std::env::var("SOURCE_DATE_EPOCH")
    .ok()
    .and_then(|s| s.parse::<i64>().ok())
    .unwrap_or_else(|| chrono::Utc::now().timestamp());
println!("cargo:rustc-env=BUILD_TIMESTAMP={timestamp}");
```

> 参见[构建脚本](ch01-build-scripts-buildrs-in-depth.md) 了解完整的 build.rs 模式。

---

### 9. `cargo tree` 去重工作流

**问题**：`cargo tree --duplicates` 显示 5 个版本的 `syn` 和 3 个版本的 `tokio-util`。
编译时间很痛苦。

**修复**：系统性去重：

```bash
# 步骤 1：查找重复项
cargo tree --duplicates

# 步骤 2：找出谁拉入了旧版本
cargo tree --invert --package syn@1.0.109

# 步骤 3：更新罪魁祸首
cargo update -p serde_derive  # 可能拉入 syn 2.x

# 步骤 4：如果没有可用更新，在 [patch] 中固定
# [patch.crates-io]
# old-crate = { git = "...", branch = "syn2-migration" }

# 步骤 5：验证
cargo tree --duplicates  # 应该更短
```

> 参见[依赖管理](ch06-dependency-management-and-supply-chain-s.md) 了解 `cargo-deny` 和供应链安全。

---

### 10. 推送前冒烟测试

**问题**：你推送，CI 需要 10 分钟，在格式化问题上失败。

**修复**：推送前在本地运行快速检查：

```toml
# Makefile.toml (cargo-make)
[tasks.pre-push]
description = "推送前的本地冒烟测试"
script = '''
cargo fmt --all -- --check
cargo clippy --workspace --all-targets -- -D warnings
cargo test --workspace --lib
'''
```

```bash
cargo make pre-push  # < 30 秒
git push
```

或者使用 git pre-push hook：

```bash
#!/bin/sh
# .git/hooks/pre-push
cargo fmt --all -- --check && cargo clippy --workspace -- -D warnings
```

> 参见 [CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) 了解 `Makefile.toml` 模式。

---

### 🏋️ 练习

#### 🟢 练习 1：应用三个技巧

从本章选择三个技巧并将它们应用到现有的 Rust 项目中。哪个影响最大？

<details>
<summary>解决方案</summary>

典型的高影响组合：

1. **`[profile.dev.package."*"] opt-level = 2`** — 开发模式运行时的即时改进（解析密集型代码快 2-10 倍）

2. **`CARGO_ENCODED_RUSTFLAGS`** — 消除因第三方警告导致的虚假 CI 失败

3. **`cargo-hack --each-feature`** — 通常在任何有 3+ 特性的项目中找到至少一个损坏的特性组合

```bash
# 应用技巧 5：
echo '[profile.dev.package."*"]' >> Cargo.toml
echo 'opt-level = 2' >> Cargo.toml

# 在 CI 中应用技巧 7：
# 用 CARGO_ENCODED_RUSTFLAGS 替换 RUSTFLAGS

# 应用技巧 3：
cargo install cargo-hack
cargo hack check --each-feature --no-dev-deps
```
</details>

#### 🟡 练习 2：去重你的依赖树

在真实项目上运行 `cargo tree --duplicates`。消除至少一个重复项。测量前后的编译时间。

<details>
<summary>解决方案</summary>

```bash
# 之前
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 计算重复行数

# 查找并修复一个重复项
cargo tree --duplicates
cargo tree --invert --package <duplicate-crate>@<old-version>
cargo update -p <parent-crate>

# 之后
time cargo build --release 2>&1 | tail -1
cargo tree --duplicates | wc -l  # 应该更少

# 典型结果：每个消除的重复项减少 5-15% 编译时间
#（尤其是像 syn、tokio 这样的重 crate）
```
</details>

### 关键要点

- 使用 `CARGO_ENCODED_RUSTFLAGS` 而不是 `RUSTFLAGS` 以避免破坏第三方构建脚本
- `[profile.dev.package."*"] opt-level = 2` 是单一最高影响的开发体验技巧
- 缓存调优（只在 main 上 `save-if`）防止活跃仓库上的 CI 缓存膨胀
- `cargo tree --duplicates` + `cargo update` 是免费的编译时间改进 — 每月做一次
- 用 `cargo make pre-push` 在本地运行快速检查以避免 CI 往返浪费

---

