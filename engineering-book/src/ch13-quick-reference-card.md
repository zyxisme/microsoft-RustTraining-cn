# 快速参考卡

### 速查表：一目了然的命令

```bash
# ─── 构建脚本 ───
cargo build                          # 先编译 build.rs，然后编译 crate
cargo build -vv                      # 详细 — 显示 build.rs 输出

# ─── 交叉编译 ───
rustup target add x86_64-unknown-linux-musl
cargo build --release --target x86_64-unknown-linux-musl
cargo zigbuild --release --target x86_64-unknown-linux-gnu.2.17
cross build --release --target aarch64-unknown-linux-gnu

# ─── 基准测试 ───
cargo bench                          # 运行所有基准测试
cargo bench -- parse                 # 运行匹配 "parse" 的基准测试
cargo flamegraph -- --args           # 从二进制文件生成 flamegraph
perf record -g ./target/release/bin  # 记录 perf 数据
perf report                          # 交互式查看 perf 数据

# ─── 覆盖率 ───
cargo llvm-cov --html                # HTML 报告
cargo llvm-cov --lcov --output-path lcov.info
cargo llvm-cov --workspace --fail-under-lines 80
cargo tarpaulin --out Html           # 替代工具

# ─── 安全验证 ───
cargo +nightly miri test             # 在 Miri 下运行测试
MIRIFLAGS="-Zmiri-disable-isolation" cargo +nightly miri test
valgrind --leak-check=full ./target/debug/binary
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu

# ─── 审计和供应链 ───
cargo audit                          # 已知漏洞扫描
cargo audit --deny warnings          # 任何公告时 CI 失败
cargo deny check                     # 许可证 + 公告 + 禁令 + 来源检查
cargo deny list                      # 列出 dep 树中的所有许可证
cargo vet                            # 供应链信任验证
cargo outdated --workspace           # 查找过时的依赖
cargo semver-checks                  # 检测破坏性 API 变更
cargo geiger                         # 统计依赖树中的 unsafe

# ─── 二进制优化 ───
cargo bloat --release --crates       # 每个 crate 的大小贡献
cargo bloat --release -n 20          # 20 个最大函数
cargo +nightly udeps --workspace     # 查找未使用的依赖
cargo machete                        # 快速未使用依赖检测
cargo expand --lib module::name     # 查看宏展开
cargo msrv find                     # 发现最低 Rust 版本
cargo clippy --fix --workspace --allow-dirty  # 自动修复 lint 警告

# ─── 编译时优化 ───
export RUSTC_WRAPPER=sccache         # 共享编译缓存
sccache --show-stats                # 缓存命中统计
cargo nextest run                   # 更快的测试运行器
cargo nextest run --retries 2       # 重试不稳定的测试

# ─── 平台工程 ───
cargo check --target thumbv7em-none-eabihf   # 验证 no_std 构建
cargo build --target x86_64-pc-windows-gnu   # 交叉编译到 Windows
cargo xwin build --target x86_64-pc-windows-msvc  # MSVC ABI 交叉编译
cfg!(target_os = "linux")                     # 编译时 cfg（求值为 bool）

# ─── 发布 ───
cargo release patch --dry-run        # 预览发布
cargo release patch --execute        # 递增、提交、标签、发布
cargo dist plan                      # 预览分发产物
```

### 决策表：何时使用什么工具

| 目标 | 工具 | 何时使用 |
|------|------|-------------|
| 嵌入 git 哈希 / 构建信息 | `build.rs` | 二进制需要可追溯性 |
| 用 Rust 编译 C 代码 | `build.rs` 中的 `cc` crate | FFI 到小型 C 库 |
| 从模式生成代码 | `prost-build` / `tonic-build` | Protobuf、gRPC、FlatBuffers |
| 链接系统库 | `build.rs` 中的 `pkg-config` | OpenSSL、libpci、systemd |
| 静态 Linux 二进制文件 | `--target x86_64-unknown-linux-musl` | 容器/云部署 |
| 面向旧 glibc | `cargo-zigbuild` | RHEL 7、CentOS 7 兼容性 |
| ARM 服务器二进制文件 | `cross` 或 `cargo-zigbuild` | Graviton/Ampere 部署 |
| 统计基准测试 | Criterion.rs | 性能回归检测 |
| 快速性能检查 | Divan | 开发时性能分析 |
| 找到热点 | `cargo flamegraph` / `perf` | 基准测试识别慢代码之后 |
| 行/分支覆盖率 | `cargo-llvm-cov` | CI 覆盖率门控、差距分析 |
| 快速覆盖率检查 | `cargo-tarpaulin` | 本地开发 |
| Rust UB 检测 | Miri | 纯 Rust `unsafe` 代码 |
| C FFI 内存安全 | Valgrind memcheck | 混合 Rust/C 代码库 |
| 数据竞争检测 | TSan 或 Miri | 并发 `unsafe` 代码 |
| 缓冲区溢出检测 | ASan | `unsafe` 指针算术 |
| 泄漏检测 | Valgrind 或 LSan | 长运行服务 |
| 本地 CI 等价物 | `cargo-make` | 开发者工作流自动化 |
| Pre-commit 检查 | `cargo-husky` 或 git hooks | 在推送之前捕获问题 |
| 自动发布 | `cargo-release` + `cargo-dist` | 版本管理 + 分发 |
| 依赖审计 | `cargo-audit` / `cargo-deny` | 供应链安全 |
| 许可证合规性 | `cargo-deny`（许可证） | 商业 / 企业项目 |
| 供应链信任 | `cargo-vet` | 高安全环境 |
| 查找过时的 dep | `cargo-outdated` | 计划维护 |
| 检测破坏性变更 | `cargo-semver-checks` | 库 crate 发布 |
| 依赖树分析 | `cargo tree --duplicates` | 去重和修整 dep 图 |
| 二进制大小分析 | `cargo-bloat` | 大小受限部署 |
| 查找未使用的 dep | `cargo-udeps` / `cargo-machete` | 修整编译时间和大小 |
| LTO 调优 | `lto = true` 或 `"thin"` | 发布二进制优化 |
| 大小优化二进制文件 | `opt-level = "z"` + `strip = true` | 嵌入式 / WASM / 容器 |
| Unsafe 使用审计 | `cargo-geiger` | 安全策略执行 |
| 宏调试 | `cargo-expand` | Derive / macro_rules 调试 |
| 更快链接 | `mold` 链接器 | 开发者内循环 |
| 编译缓存 | `sccache` | CI 和本地构建速度 |
| 更快的测试 | `cargo-nextest` | CI 和本地测试速度 |
| MSRV 合规性 | `cargo-msrv` | 库发布 |
| `no_std` 库 | `#![no_std]` + `default-features = false` | 嵌入式、UEFI、WASM |
| Windows 交叉编译 | `cargo-xwin` / MinGW | Linux → Windows 构建 |
| 平台抽象 | `#[cfg]` + trait 模式 | 多 OS 代码库 |
| Windows API 调用 | `windows-sys` / `windows` crate | 原生 Windows 功能 |
| 端到端计时 | `hyperfine` | 整体二进制基准测试、前后比较 |
| 属性驱动测试 | `proptest` | 边缘情况发现、解析器健壮性 |
| 快照测试 | `insta` | 大型结构化输出验证 |
| 覆盖率引导模糊测试 | `cargo-fuzz` | 解析器中的崩溃发现 |
| 并发模型检查 | `loom` | 无锁数据结构、原子顺序 |
| 特性组合测试 | `cargo-hack` | 有多个 `#[cfg]` 特性的 crate |
| 快速 UB 检查（接近原生） | `cargo-careful` | CI 安全门控，比 Miri 更轻 |
| 保存时自动重建 | `cargo-watch` | 开发者内循环、紧密反馈 |
| 工作空间文档 | `cargo doc` + rustdoc | API 发现、入职、doc-link CI |
| 可重现构建 | `--locked` + `SOURCE_DATE_EPOCH` | 发布完整性验证 |
| CI 缓存调优 | `Swatinem/rust-cache@v2` | 构建时间减少（冷 → 缓存） |
| 工作空间 lint 策略 | Cargo.toml 中的 `[workspace.lints]` | 跨所有 crate 的一致 Clippy/编译器 lint |
| 自动修复 lint 警告 | `cargo clippy --fix` | 繁琐问题的自动清理 |

### 进一步阅读

| 主题 | 资源 |
|-------|----------|
| Cargo 构建脚本 | [Cargo Book — Build Scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) |
| 交叉编译 | [Rust Cross-Compilation](https://rust-lang.github.io/rustup/cross-compilation.html) |
| `cross` 工具 | [cross-rs/cross](https://github.com/cross-rs/cross) |
| `cargo-zigbuild` | [cargo-zigbuild docs](https://github.com/rust-cross/cargo-zigbuild) |
| Criterion.rs | [Criterion User Guide](https://bheisler.github.io/criterion.rs/book/) |
| Divan | [Divan docs](https://github.com/nvzqz/divan) |
| `cargo-llvm-cov` | [cargo-llvm-cov](https://github.com/taiki-e/cargo-llvm-cov) |
| `cargo-tarpaulin` | [tarpaulin docs](https://github.com/xd009642/tarpaulin) |
| Miri | [Miri GitHub](https://github.com/rust-lang/miri) |
| Rust 中的 Sanitizer | [rustc Sanitizer docs](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html) |
| `cargo-make` | [cargo-make book](https://sagiegurari.github.io/cargo-make/) |
| `cargo-release` | [cargo-release docs](https://github.com/crate-ci/cargo-release) |
| `cargo-dist` | [cargo-dist docs](https://axodotdev.github.io/cargo-dist/book/) |
| 配置文件引导优化 | [Rust PGO guide](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) |
| Flamegraphs | [cargo-flamegraph](https://github.com/flamegraph-rs/flamegraph) |
| `cargo-deny` | [cargo-deny docs](https://embarkstudios.github.io/cargo-deny/) |
| `cargo-vet` | [cargo-vet docs](https://mozilla.github.io/cargo-vet/) |
| `cargo-audit` | [cargo-audit](https://github.com/rustsec/rustsec/tree/main/cargo-audit) |
| `cargo-bloat` | [cargo-bloat](https://github.com/RazrFalcon/cargo-bloat) |
| `cargo-udeps` | [cargo-udeps](https://github.com/est31/cargo-udeps) |
| `cargo-geiger` | [cargo-geiger](https://github.com/geiger-rs/cargo-geiger) |
| `cargo-semver-checks` | [cargo-semver-checks](https://github.com/obi1kenobi/cargo-semver-checks) |
| `cargo-nextest` | [nextest docs](https://nexte.st/) |
| `sccache` | [sccache](https://github.com/mozilla/sccache) |
| `mold` 链接器 | [mold](https://github.com/rui314/mold) |
| `cargo-msrv` | [cargo-msrv](https://github.com/foresterre/cargo-msrv) |
| LTO | [rustc Codegen Options](https://doc.rust-lang.org/rustc/codegen-options/index.html) |
| Cargo Profiles | [Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) |
| `no_std` | [Rust Embedded Book](https://docs.rust-embedded.org/book/) |
| `windows-sys` crate | [windows-rs](https://github.com/microsoft/windows-rs) |
| `cargo-xwin` | [cargo-xwin docs](https://github.com/rust-cross/cargo-xwin) |
| `cargo-hack` | [cargo-hack](https://github.com/taiki-e/cargo-hack) |
| `cargo-careful` | [cargo-careful](https://github.com/RalfJung/cargo-careful) |
| `cargo-watch` | [cargo-watch](https://github.com/watchexec/cargo-watch) |
| Rust CI 缓存 | [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) |
| Rustdoc book | [Rustdoc Book](https://doc.rust-lang.org/rustdoc/) |
| 条件编译 | [Rust Reference — cfg](https://doc.rust-lang.org/reference/conditional-compilation.html) |
| 嵌入式 Rust | [Awesome Embedded Rust](https://github.com/rust-embedded/awesome-embedded-rust) |
| `hyperfine` | [hyperfine](https://github.com/sharkdp/hyperfine) |
| `proptest` | [proptest](https://github.com/proptest-rs/proptest) |
| `insta` | [insta snapshot testing](https://insta.rs/) |
| `cargo-fuzz` | [cargo-fuzz](https://github.com/rust-fuzz/cargo-fuzz) |
| `loom` | [loom concurrency testing](https://github.com/tokio-rs/loom) |

---

*作为配套参考生成 — Rust 模式和类型驱动正确性的配套参考。*

*版本 1.3 — 为完整性添加了 cargo-hack、cargo-careful、cargo-watch、cargo doc、
可重现构建、CI 缓存策略、顶点练习和章节依赖图。*
