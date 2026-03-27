# Rust 工程实践 — 超越 `cargo build`

## 演讲者简介

- 微软 SCHIE（Silicon and Cloud Hardware Infrastructure Engineering）团队首席固件架构师
- 行业资深专家，在安全、系统编程（固件、操作系统、虚拟机监控器）、CPU 和平台架构以及 C++ 系统方面拥有专业知识
- 2017 年（@AWS EC2）开始使用 Rust 编程，从此爱上了这门语言

---

> 一份关于 Rust 工具链特性的实用指南，这些特性是大多数团队很晚才发现的：
> 构建脚本、交叉编译、性能基准测试、代码覆盖率以及使用 Miri 和 Valgrind 进行安全验证。
> 每一章都使用来自真实硬件诊断代码库的具体示例 —
> 一个大型多 crate 工作空间 — 因此每种技术都能直接映射到生产代码。

## 如何使用本书

本书专为**自主学习或团队研讨会**设计。每一章基本独立 — 可以按顺序阅读，也可以跳转到需要的主题。

### 难度图例

| 符号 | 级别 | 含义 |
|:------:|-------|---------|
| 🟢 | 入门级 | 简单直接的工具和清晰的模式 — 第一天就很有用 |
| 🟡 | 中级 | 需要理解工具链内部机制或平台概念 |
| 🔴 | 高级 | 深入的 toolchain 知识、nightly 特性或多工具编排 |

### 学习进度指南

| 部分 | 章节 | 预计时间 | 关键成果 |
|------|----------|:---------:|-------------|
| **I — 构建与发布** | ch01–02 | 3–4 小时 | 构建元数据、交叉编译、静态二进制文件 |
| **II — 测量与验证** | ch03–05 | 4–5 小时 | 统计基准测试、覆盖率检查点、Miri/ sanitizer |
| **III — 加固与优化** | ch06–10 | 6–8 小时 | 供应链安全、发布 profiles、编译时工具、`no_std`、Windows |
| **IV — 集成** | ch11–13 | 3–4 小时 | 生产 CI/CD 流水线、技巧、顶点练习 |
| | | **16–21 小时** | **完整的生产工程流水线** |

### 完成练习

每一章都包含**🏋️ 练习**，带有难度标识。解决方案在可展开的 `<details>` 块中提供 — 请先尝试练习，然后检查你的答案。

- 🟢 练习通常可以在 10–15 分钟内完成
- 🟡 练习需要 20–40 分钟，可能涉及本地运行工具
- 🔴 练习需要大量设置和实验（1+ 小时）

## 前置要求

| 概念 | 学习资源 |
|---------|-------------------|
| Cargo 工作空间布局 | [Rust Book ch14.3](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) |
| Feature flags | [Cargo Reference — Features](https://doc.rust-lang.org/cargo/reference/features.html) |
| `#[cfg(test)]` 和基础测试 | Rust Patterns ch12 |
| `unsafe` 代码块和 FFI 基础 | Rust Patterns ch10 |

## 章节依赖图

```text
                 ┌──────────┐
                 │ ch00     │
                 │  Intro   │
                 └────┬─────┘
        ┌─────┬───┬──┴──┬──────┬──────┐
        ▼     ▼   ▼     ▼      ▼      ▼
      ch01  ch03 ch04  ch05   ch06   ch09
      Build Bench Cov  Miri   Deps   no_std
        │     │    │    │      │      │
        │     └────┴────┘      │      ▼
        │          │           │    ch10
        ▼          ▼           ▼   Windows
       ch02      ch07        ch07    │
       Cross    RelProf     RelProf  │
        │          │           │     │
        │          ▼           │     │
        │        ch08          │     │
        │      CompTime        │     │
        └──────────┴───────────┴─────┘
                   │
                   ▼
                 ch11
               CI/CD Pipeline
                   │
                   ▼
                ch12 ─── ch13
              Tricks    Quick Ref
```

**可任意顺序阅读**：ch01、ch03、ch04、ch05、ch06、ch09 相互独立。
**需先阅读前置章节**：ch02（需要 ch01）、ch07–ch08（受益于 ch03–ch06）、ch10（受益于 ch09）。
**最后阅读**：ch11（将所有内容整合在一起）、ch12（技巧）、ch13（参考）。

## 带注释的目录

### 第一部分 — 构建与发布

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 1 | [构建脚本 — 深入理解 `build.rs`](ch01-build-scripts-buildrs-in-depth.md) | 🟢 | 编译时常量、C 代码编译、protobuf 生成、系统库链接、反模式 |
| 2 | [交叉编译 — 一份源码，多个目标平台](ch02-cross-compilation-one-source-many-target.md) | 🟡 | 目标三元组、musl 静态二进制文件、ARM 交叉编译、`cross` 工具、`cargo-zigbuild`、GitHub Actions |

### 第二部分 — 测量与验证

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 3 | [基准测试 — 测量重要的事情](ch03-benchmarking-measuring-what-matters.md) | 🟡 | Criterion.rs、Divan、`perf` flamegraphs、PGO、CI 中的持续基准测试 |
| 4 | [代码覆盖率 — 发现测试遗漏的内容](ch04-code-coverage-seeing-what-tests-miss.md) | 🟢 | `cargo-llvm-cov`、`cargo-tarpaulin`、`grcov`、Codecov/Coveralls CI 集成 |
| 5 | [Miri、Valgrind 和 Sanitizers](ch05-miri-valgrind-and-sanitizers-verifying-u.md) | 🔴 | MIR 解释器、Valgrind memcheck/Helgrind、ASan/MSan/TSan、cargo-fuzz、loom |

### 第三部分 — 加固与优化

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 6 | [依赖管理和供应链安全](ch06-dependency-management-and-supply-chain-s.md) | 🟢 | `cargo-audit`、`cargo-deny`、`cargo-vet`、`cargo-outdated`、`cargo-semver-checks` |
| 7 | [发布 Profiles 和二进制文件大小](ch07-release-profiles-and-binary-size.md) | 🟡 | 发布 profile 分析、LTO 权衡、`cargo-bloat`、`cargo-udeps` |
| 8 | [编译时和开发者工具](ch08-compile-time-and-developer-tools.md) | 🟡 | `sccache`、`mold`、`cargo-nextest`、`cargo-expand`、`cargo-geiger`、工作空间 lint、MSRV |
| 9 | [`no_std` 和 Feature 验证](ch09-no-std-and-feature-verification.md) | 🔴 | `cargo-hack`、`core`/`alloc`/`std` 层、自定义 panic 处理程序、测试 `no_std` 代码 |
| 10 | [Windows 和条件编译](ch10-windows-and-conditional-compilation.md) | 🟡 | `#[cfg]` 模式、`windows-sys`/`windows` crate、`cargo-xwin`、平台抽象 |

### 第四部分 — 集成

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 11 | [整合一切 — 生产 CI/CD 流水线](ch11-putting-it-all-together-a-production-cic.md) | 🟡 | GitHub Actions 工作流、`cargo-make`、pre-commit hooks、`cargo-dist`、顶点练习 |
| 12 | [来自前线的技巧](ch12-tricks-from-the-trenches.md) | 🟡 | 10 个经过实战验证的模式：`deny(warnings)` 陷阱、缓存调优、依赖去重、RUSTFLAGS 等 |
| 13 | [快速参考卡](ch13-quick-reference-card.md) | — | 命令速查、60+ 决策表条目、进一步阅读链接 |
