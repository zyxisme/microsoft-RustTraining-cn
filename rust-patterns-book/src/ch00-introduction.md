# Rust 模式与工程实践

## 讲师简介

- 微软 SCHIE（硅与云硬件基础设施工程）团队首席固件架构师
- 拥有安全、系统编程（固件、操作系统、虚拟机监控器）、CPU 和平台架构以及 C++ 系统领域专业经验的行业资深专家
- 2017 年在 AWS EC2 开始使用 Rust 编程，此后一直热爱这门语言

---

这是一本关于真实代码库中出现的中级及以上 Rust 模式的实用指南。这不是一门语言教程 —— 它假设你已经能够编写基本的 Rust 并希望提升水平。每一章都围绕一个概念展开，解释何时以及为什么使用它，并提供带有嵌入式练习的可编译示例。

## 适用对象

- 已经学完《Rust 编程语言》但在实际设计中遇到困难的开发者
- 将生产系统翻译成 Rust 的 C++/C# 工程师
- 在泛型、trait 约束或生命周期错误面前碰壁，希望获得系统化工具包的任何人

## 前置知识

开始之前，你应该熟悉以下内容：
- 所有权、借用和生命周期（基础级别）
- 枚举、模式匹配和 `Option`/`Result`
- 结构体、方法和基本 trait（`Display`、`Debug`、`Clone`）
- Cargo 基础：`cargo build`、`cargo test`、`cargo run`

## 如何使用本书

### 难度图例

每一章都标有难度级别：

| 符号 | 级别 | 含义 |
|--------|-------|---------|
| 🟢 | 基础 | 每个 Rust 开发者都需要掌握的核心概念 |
| 🟡 | 中级 | 生产代码库中使用的模式 |
| 🔴 | 高级 | 深度语言机制 —— 需要时回顾即可 |

### 学习进度指南

| 章节 | 主题 | 建议时间 | 里程碑 |
|----------|-------|----------------|------------|
| **第一部分：类型级模式** | | | |
| 1. 泛型 🟢 | 单态化、代码膨胀的权衡、泛型 vs 枚举 vs trait 对象、const 泛型、`const fn` | 1–2 小时 | 能解释何时 `dyn Trait` 优于泛型 |
| 2. Traits 🟡 | 关联类型、GAT、 blanket 实现、虚表 | 3–4 小时 | 能设计带有关联类型的 trait |
| 3. Newtype 与类型状态 🟡 | 零成本安全、编译时 FSM、构建器模式 | 2–3 小时 | 能构建类型状态构建器模式 |
| 4. PhantomData 🔴 | 生命周期标记、方差、drop 检查 | 2–3 小时 | 能解释为何 `PhantomData<fn(T)>` 不同于 `PhantomData<T>` |
| **第二部分：并发与运行时** | | | |
| 5. 通道 🟢 | `mpsc`、crossbeam、`select!`、actor | 1–2 小时 | 能实现基于通道的工作池 |
| 6. 并发 🟡 | 线程、rayon、Mutex、RwLock、原子类型 | 2–3 小时 | 能为场景选择正确的同步原语 |
| 7. 闭包 🟢 | `Fn`/`FnMut`/`FnOnce`、组合器 | 1–2 小时 | 能编写接受闭包的高阶函数 |
| 8. 智能指针 🟡 | Box、Rc、Arc、RefCell、Cow、Pin | 2–3 小时 | 能解释何时使用每种智能指针 |
| **第三部分：系统与生产环境** | | | |
| 9. 错误处理 🟢 | thiserror、anyhow、`?` 操作符 | 1–2 小时 | 能设计错误类型层次结构 |
| 10. 序列化 🟡 | serde、零拷贝、二进制数据 | 2–3 小时 | 能编写自定义 serde 反序列化器 |
| 11. Unsafe 🔴 | 超能力、FFI、UB 陷阱、分配器 | 2–3 小时 | 能将 unsafe 代码包装在 sound 的安全 API 中 |
| 12. 宏 🟡 | `macro_rules!`、proc 宏、`syn`/`quote` | 2–3 小时 | 能编写带有 `tt` 吞噬的声明式宏 |
| 13. 测试 🟢 | 单元/集成/文档测试、proptest、criterion | 1–2 小时 | 能设置基于属性的测试 |
| 14. API 设计 🟡 | 模块布局、人体工程学 API、功能标志 | 2–3 小时 | 能应用"解析，不要验证"模式 |
| 15. Async 🔴 | Future、Tokio、常见陷阱 | 1–2 小时 | 能识别 async 反模式 |
| **附录** | | | |
| 速查卡 | 快速查看 trait 约束、生命周期、模式 | 按需 | — |
| Capstone 项目 | 类型安全的任务调度器 | 4–6 小时 | 提交一个可工作的实现 |

**预计总时间**：认真学习和做练习需要 30–45 小时。

### 完成练习的方法

每一章末尾都有实践练习。为了获得最大学习效果：

1. **先自己尝试** —— 打开解决方案前至少花 15 分钟
2. **亲手敲代码** —— 不要复制粘贴；打字能建立肌肉记忆
3. **修改解决方案** —— 添加一个功能、改变一个约束、有意破坏一些东西
4. **检查交叉引用** —— 大多数练习结合了多章的模式

Capstone 项目（附录）将书中的模式整合到一个完整的生产级系统中。

## 目录

### 第一部分：类型级模式

**[1. 泛型 —— 全景图](ch01-generics-the-full-picture.md)** 🟢
单态化、代码膨胀权衡、泛型 vs 枚举 vs trait 对象、const 泛型、`const fn`。

**[2. 深入 Trait](ch02-traits-in-depth.md)** 🟡
关联类型、GAT、blanket 实现、标记 trait、虚表、HRTB、扩展 trait、枚举分发。

**[3. Newtype 与类型状态模式](ch03-the-newtype-and-type-state-patterns.md)** 🟡
零成本类型安全、编译时状态机、构建器模式、配置 trait。

**[4. PhantomData —— 不携带数据的类型](ch04-phantomdata-types-that-carry-no-data.md)** 🔴
生命周期标记、单位量模式、drop 检查、方差。

### 第二部分：并发与运行时

**[5. 通道与消息传递](ch05-channels-and-message-passing.md)** 🟢
`std::sync::mpsc`、crossbeam、`select!`、背压、actor 模式。

**[6. 并发 vs 并行 vs 线程](ch06-concurrency-vs-parallelism-vs-threads.md)** 🟡
操作系统线程、作用域线程、rayon、Mutex/RwLock/原子类型、Condvar、OnceLock、无锁模式。

**[7. 闭包与高阶函数](ch07-closures-and-higher-order-functions.md)** 🟢
`Fn`/`FnMut`/`FnOnce`、闭包作为参数/返回值、组合器、高阶 API。

**[8. 智能指针与内部可变性](ch08-smart-pointers-and-interior-mutability.md)** 🟡
Box、Rc、Arc、Weak、Cell/RefCell、Cow、Pin、ManuallyDrop。

### 第三部分：系统与生产环境

**[9. 错误处理模式](ch09-error-handling-patterns.md)** 🟢
thiserror vs anyhow、`#[from]`、`.context()`、`?` 操作符、panic。

**[10. 序列化、零拷贝与二进制数据](ch10-serialization-zero-copy-and-binary-data.md)** 🟡
serde 基础、枚举表示、零拷贝反序列化、`repr(C)`、`bytes::Bytes`。

**[11. Unsafe Rust —— 受控的危险](ch11-unsafe-rust-controlled-danger.md)** 🔴
五大超能力、sound 抽象、FFI、UB 陷阱、arena/slab 分配器。

**[12. 宏 —— 编写代码的代码](ch12-macros-code-that-writes-code.md)** 🟡
`macro_rules!`、何时（不）使用宏、proc 宏、derive 宏、`syn`/`quote`。

**[13. 测试与基准测试模式](ch13-testing-and-benchmarking-patterns.md)** 🟢
单元/集成/文档测试、proptest、criterion、模拟策略。

**[14. Crate 架构与 API 设计](ch14-crate-architecture-and-api-design.md)** 🟡
模块布局、API 设计清单、人体工程学参数、功能标志、工作空间。

**[15. Async/Await  Essentials](ch15-asyncawait-essentials.md)** 🔴
Future、Tokio 快速入门、常见陷阱。（深度 async 覆盖，请参阅我们的 Async Rust Training。）

### 附录

**[总结与速查卡](ch17-summary-and-reference-card.md)**
模式决策指南、trait 约束速查表、生命周期省略规则、延伸阅读。

**[Capstone 项目：类型安全的任务调度器](ch18-capstone-project.md)**
将泛型、trait、类型状态、通道、错误处理和测试整合到一个完整系统中。

***
