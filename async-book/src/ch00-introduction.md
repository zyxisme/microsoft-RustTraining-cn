# 异步 Rust：从 Future 到生产环境

## 作者简介

- 微软 SCHIE（硅与云硬件基础设施工程）团队首席固件架构师
- 拥有安全、系统编程（固件、操作系统、虚拟机监控程序）、CPU 和平台架构以及 C++ 系统方面专业经验的行业资深人士
- 2017 年在 AWS EC2 开始使用 Rust 编程，从此爱上了这门语言

---

一本深入探讨 Rust 异步编程的指南。与大多数从 `tokio::main` 开始并绕过内部原理的异步教程不同，本指南从第一性原理出发——`Future` trait、轮询、状态机——然后逐步深入到实际生产中的模式、运行时选择和生产环境中的陷阱。

## 适用人群

- 能够编写同步 Rust 但觉得异步复杂的 Rust 开发者
- 来自 C#、Go、Python 或 JavaScript 的开发者，熟悉 `async/await` 但不了解 Rust 的异步模型
- 曾被 `Future is not Send`、`Pin<Box<dyn Future>>` 或"为什么我的程序卡住了"困扰过的任何人

## 前置要求

你应该熟悉以下内容：

- 所有权、借用和生命周期
- Trait 和泛型（包括 `impl Trait`）
- 使用 `Result<T, E>` 和 `?` 操作符
- 基础多线程（`std::thread::spawn`、`Arc`、`Mutex`）

不需要任何异步 Rust 经验。

## 如何使用本书

**第一次阅读时请按顺序阅读。** 第一至第三部分内容层层递进。每章包含：

| 符号 | 含义 |
|------|------|
| 🟢 | 初级 — 基础概念 |
| 🟡 | 中级 — 需要前面章节的基础 |
| 🔴 | 高级 — 深层内部原理或生产模式 |

每章包含：
- 顶部的**"你将学到什么"**板块
- **Mermaid 图表**，供视觉学习者使用
- **嵌入式练习**，附有隐藏答案
- **核心要点**，总结核心思想
- 与相关章节的**交叉引用**

## 学习进度指南

| 章节 | 主题 | 建议时长 | 里程碑 |
|------|------|----------|--------|
| 1–5 | 异步工作原理 | 6–8 小时 | 能够解释 `Future`、`Poll`、`Pin`，以及为什么 Rust 没有内置运行时 |
| 6–10 | 异步生态系统 | 6–8 小时 | 能够手动构建 futures、选择运行时，并使用 tokio 的 API |
| 11–13 | 生产级异步 | 6–8 小时 | 能够编写包含 streams、正确错误处理和优雅关闭的生产级异步代码 |
| 终极项目 | 聊天服务器 | 4–6 小时 | 已构建一个整合所有概念的完整异步应用程序 |

**预计总时长：22–30 小时**

## 完成练习

每个内容章节都有嵌入式练习。终极项目（第 16 章）将所有内容整合到一个项目中。为了最大化学习效果：

1. **展开答案之前先尝试练习**——思考的过程才是真正学习发生的地方
2. **亲手敲代码，不要复制粘贴**——对于 Rust 的语法，肌肉记忆很重要
3. **运行每个示例**——`cargo new async-exercises`，边学边测

## 目录

### 第一部分：异步工作原理

- [1. 为什么 Rust 的异步与众不同](ch01-why-async-is-different-in-rust.md) 🟢 — 根本区别：Rust 没有内置运行时
- [2. Future Trait](ch02-the-future-trait.md) 🟡 — `poll()`、`Waker`，以及使这一切工作的契约
- [3. Poll 如何工作](ch03-how-poll-works.md) 🟡 — 轮询状态机和一个最小执行器
- [4. Pin 和 Unpin](ch04-pin-and-unpin.md) 🔴 — 为什么自引用结构体需要 pinning
- [5. 状态机揭秘](ch05-the-state-machine-reveal.md) 🟢 — 编译器从 `async fn` 实际生成的代码

### 第二部分：异步生态系统

- [6. 手动构建 Futures](ch06-building-futures-by-hand.md) 🟡 — 从零开始实现 TimerFuture、Join、Select
- [7. 执行器和运行时](ch07-executors-and-runtimes.md) 🟡 — tokio、smol、async-std、embassy——如何选择
- [8. Tokio 深度探讨](ch08-tokio-deep-dive.md) 🟡 — 运行时变体、spawn、通道、同步原语
- [9. 何时不应使用 Tokio](ch09-when-tokio-isnt-the-right-fit.md) 🟡 — LocalSet、FuturesUnordered、运行时无关设计
- [10. 异步 Traits](ch10-async-traits.md) 🟡 — RPITIT、dyn dispatch、trait_variant、异步闭包

### 第三部分：生产级异步

- [11. Streams 和 AsyncIterator](ch11-streams-and-asynciterator.md) 🟡 — 异步迭代、AsyncRead/Write、stream 组合器
- [12. 常见陷阱](ch12-common-pitfalls.md) 🔴 — 9 个生产环境 bug 及如何避免
- [13. 生产模式](ch13-production-patterns.md) 🔴 — 优雅关闭、背压、Tower 中间件

### 附录

- [摘要和参考卡片](ch15-summary-and-reference-card.md) — 快速查询表和决策树
- [终极项目：异步聊天服务器](ch16-capstone-project.md) — 构建一个完整的异步应用程序

***
