# Rust for Python Programmers: Complete Training Guide

面向有 Python 开发经验的开发者的 Rust 完整学习指南。本指南涵盖从基础语法到高级模式的所有内容，重点介绍从动态类型、垃圾回收语言转变为静态类型、在编译时保证内存安全的系统编程语言所需的概念转变。

## 如何使用本书

**自学格式**：先学习第一部分（第1-6章）——这些与你已知的 Python 概念紧密对应。第二部分（第7-12章）介绍 Rust 特有的概念，如所有权和 trait。第三部分（第13-16章）涵盖高级主题和迁移。

**学习进度建议：**

| 章节 | 主题 | 建议时间 | 里程碑 |
|------|------|----------|--------|
| 1–4 | 环境配置、类型、控制流 | 1天 | 能在 Rust 中编写 CLI 温度转换器 |
| 5–6 | 数据结构、枚举、模式匹配 | 1–2天 | 能定义带数据的枚举并穷尽匹配 |
| 7 | 所有权与借用 | 1–2天 | 能解释为什么 `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1天 | 能创建使用 `?` 传播错误的多文件项目 |
| 10–12 | Trait、泛型、闭包、迭代器 | 1–2天 | 能将列表推导式转换为迭代器链 |
| 13 | 并发 | 1天 | 能用 `Arc<Mutex<T>>` 编写线程安全计数器 |
| 14 | 不安全代码、PyO3、测试 | 1天 | 能通过 PyO3 从 Python 调用 Rust 函数 |
| 15–16 | 迁移、最佳实践 | 按自己节奏 | 参考资料——写实际代码时查阅 |
| 17 | 终极项目 | 2–3天 | 构建一个整合所有内容的完整 CLI 应用 |

**如何使用练习：**
- 各章节包含可折叠的 `<details>` 块中的实践练习，附有答案
- **展开答案之前先尝试练习。** 与借用检查器的斗争是学习的一部分——编译器的错误信息就是你的老师
- 如果卡住超过15分钟，展开答案，学习它，然后关闭它从头再试
- [Rust Playground](https://play.rust-lang.org/) 让你无需本地安装即可运行代码

**难度标识：**
- 🟢 **初级** — 从 Python 概念的直接转换
- 🟡 **中级** — 需要理解所有权或 trait
- 🔴 **高级** — 生命周期、异步内部原理或不安全代码

**当你遇到障碍时：**
- 仔细阅读编译器错误信息——Rust 的错误提示非常有用
- 重读相关章节；所有权（第7章）等概念在第二遍阅读时往往会豁然开朗
- [Rust 标准库文档](https://doc.rust-lang.org/std/) 非常完善——可以搜索任何类型或方法
- 想要更深入的异步模式，请参阅配套的 [Async Rust Training](../async-book/)

---

## 目录

### 第一部分 — 基础

#### 1. 引言与动机 🟢
- [为什么 Python 开发者应该学 Rust](ch01-introduction-and-motivation.md#the-case-for-rust-for-python-developers)
- [Rust 解决的 Python 常见痛点](ch01-introduction-and-motivation.md#common-python-pain-points-that-rust-addresses)
- [何时选择 Rust 而非 Python](ch01-introduction-and-motivation.md#when-to-choose-rust-over-python)

#### 2. 入门指南 🟢
- [安装与配置](ch02-getting-started.md#installation-and-setup)
- [你的第一个 Rust 程序](ch02-getting-started.md#your-first-rust-program)
- [Cargo vs pip/Poetry](ch02-getting-started.md#cargo-vs-pippoetry)

#### 3. 内置类型与变量 🟢
- [变量与可变性](ch03-built-in-types-and-variables.md#variables-and-mutability)
- [基本类型对比](ch03-built-in-types-and-variables.md#primitive-types-comparison)
- [字符串类型：String vs &str](ch03-built-in-types-and-variables.md#string-types-string-vs-str)

#### 4. 控制流 🟢
- [条件语句](ch04-control-flow.md#conditional-statements)
- [循环与迭代](ch04-control-flow.md#loops-and-iteration)
- [表达式块](ch04-control-flow.md#expression-blocks)
- [函数与类型签名](ch04-control-flow.md#functions-and-type-signatures)

#### 5. 数据结构与集合 🟢
- [元组、解构](ch05-data-structures-and-collections.md#tuples-and-destructuring)
- [结构体 vs 类](ch05-data-structures-and-collections.md#structs-vs-classes)
- [Vec vs list、HashMap vs dict](ch05-data-structures-and-collections.md#vec-vs-list)

#### 6. 枚举与模式匹配 🟡
- [代数数据类型 vs 联合类型](ch06-enums-and-pattern-matching.md#algebraic-data-types-vs-union-types)
- [穷尽模式匹配](ch06-enums-and-pattern-matching.md#exhaustive-pattern-matching)
- [Option 处理 None 安全](ch06-enums-and-pattern-matching.md#option-for-none-safety)

### 第二部分 — 核心概念

#### 7. 所有权与借用 🟡
- [理解所有权](ch07-ownership-and-borrowing.md#understanding-ownership)
- [移动语义 vs 引用计数](ch07-ownership-and-borrowing.md#move-semantics-vs-reference-counting)
- [借用与生命周期](ch07-ownership-and-borrowing.md#borrowing-and-lifetimes)
- [智能指针](ch07-ownership-and-borrowing.md#smart-pointers)

#### 8. 包与模块 🟢
- [Rust 模块 vs Python 包](ch08-crates-and-modules.md#rust-modules-vs-python-packages)
- [Crate vs PyPI 包](ch08-crates-and-modules.md#crates-vs-pypi-packages)

#### 9. 错误处理 🟡
- [异常 vs Result](ch09-error-handling.md#exceptions-vs-result)
- [? 操作符](ch09-error-handling.md#the--operator)
- [使用 thiserror 自定义错误类型](ch09-error-handling.md#custom-error-types-with-thiserror)

#### 10. Trait 与泛型 🟡
- [Trait vs 鸭子类型](ch10-traits-and-generics.md#traits-vs-duck-typing)
- [协议（PEP 544）vs Trait](ch10-traits-and-generics.md#protocols-pep-544-vs-traits)
- [泛型约束](ch10-traits-and-generics.md#generic-constraints)

#### 11. From 与 Into Trait 🟡
- [Rust 中的类型转换](ch11-from-and-into-traits.md#type-conversions-in-rust)
- [From、Into、TryFrom](ch11-from-and-into-traits.md#rust-frominto)
- [字符串转换模式](ch11-from-and-into-traits.md#string-conversions)

#### 12. 闭包与迭代器 🟡
- [闭包 vs Lambda](ch12-closures-and-iterators.md#rust-closures-vs-python-lambdas)
- [迭代器 vs 生成器](ch12-closures-and-iterators.md#iterators-vs-generators)
- [宏：编写代码的代码](ch12-closures-and-iterators.md#why-macros-exist-in-rust)

### 第三部分 — 高级主题与迁移

#### 13. 并发 🔴
- [无 GIL：真正的并行](ch13-concurrency.md#no-gil-true-parallelism)
- [线程安全：类型系统保证](ch13-concurrency.md#thread-safety-type-system-guarantees)
- [async/await 对比](ch13-concurrency.md#asyncawait-comparison)

#### 14. 不安全 Rust、FFI 与测试 🔴
- [何时以及为何使用 unsafe](ch14-unsafe-rust-and-ffi.md#when-and-why-to-use-unsafe)
- [PyO3：Python 的 Rust 扩展](ch14-unsafe-rust-and-ffi.md#pyo3-rust-extensions-for-python)
- [单元测试 vs pytest](ch14-unsafe-rust-and-ffi.md#unit-tests-vs-pytest)

#### 15. 迁移模式 🟡
- [Rust 中的常见 Python 模式](ch15-migration-patterns.md#common-python-patterns-in-rust)
- [Python 开发者的必备 Crate](ch08-crates-and-modules.md#essential-crates-for-python-developers)
- [增量采用策略](ch15-migration-patterns.md#incremental-adoption-strategy)

#### 16. 最佳实践 🟡
- [Python 开发者的惯用 Rust](ch16-best-practices.md#idiomatic-rust-for-python-developers)
- [常见陷阱与解决方案](ch16-best-practices.md#common-pitfalls-and-solutions)
- [Python→Rust Rosetta Stone](ch16-best-practices.md#rosetta-stone-python-to-rust)
- [学习路径与资源](ch16-best-practices.md#learning-path-and-resources)

---

### 第四部分 — 终极项目

#### 17. 终极项目：CLI 任务管理器 🔴
- [项目：`rustdo`](ch17-capstone-project.md#the-project-rustdo)
- [数据模型、存储、命令、业务逻辑](ch17-capstone-project.md#step-1-define-the-data-model-ch-3-6-10-11)
- [测试与延伸目标](ch17-capstone-project.md#step-7-tests-ch-14)

***
