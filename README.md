<div style="background-color: #e6f4ea; padding: 16px; border-radius: 6px; color: #000000;">

**原文来源** [microsoft/rusttraining](https://github.com/microsoft/rusttraining)，由 **MiniMax-M2.7** 翻译

</div>

<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**许可证** 本项目采用 [MIT 许可证](LICENSE) 和 [知识共享署名 4.0 国际许可协议 (CC-BY-4.0)](LICENSE-DOCS) 双重许可。

</div>

<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**商标** 本项目可能包含项目、产品或服务的商标或标识。经授权使用 Microsoft 商标或标识须遵守并符合 [Microsoft 的商标与品牌指南](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general)。在本项目的修改版本中使用 Microsoft 商标或标识不得引起混淆或暗示 Microsoft 赞助。任何第三方商标或标识的使用须遵守各第三方政策。

</div>

# Rust 训练手册

七门训练课程，涵盖从不同编程背景学习 Rust，以及异步、进阶模式和工程实践的深度内容。

本教程结合了原创内容与 Rust 生态系统中一些优秀资源的思路和示例。目标是呈现一门深入、严谨的技术课程，将散落在书籍、博客、会议演讲和视频系列中的知识编织成连贯的、教学结构化的学习体验。

> **免责声明：** 这些书籍是训练材料，而非权威参考资料。虽然我们力求准确，但请始终通过 [官方 Rust 文档](https://doc.rust-lang.org/) 和 [Rust 参考](https://doc.rust-lang.org/reference/) 验证关键细节。

### 参考与致谢

- [**The Rust Programming Language**](https://doc.rust-lang.org/book/) — 一切的基础
- [**Jon Gjengset**](https://www.youtube.com/c/JonGjengset) — 深入讲解 Rust 内部机理的直播，`Crust of Rust` 系列
- [**withoutboats**](https://without.boats/blog/) — 异步设计、`Pin` 和 futures 模型
- [**fasterthanlime (Amos)**](https://fasterthanli.me/) — 从第一性原理出发的系统编程，引人入胜的长篇探索
- [**Mara Bos**](https://marabos.nl/) — *Rust Atomics and Locks*，并发原语
- [**Aleksey Kladov (matklad)**](https://matklad.github.io/) — Rust analyzer 洞察、API 设计、错误处理模式
- [**Niko Matsakis**](https://smallcultfollowing.com/babysteps/) — 语言设计、借用检查器内部机理、Polonius
- [**Rust by Example**](https://doc.rust-lang.org/rust-by-example/) 和 [**Rustonomicon**](https://doc.rust-lang.org/nomicon/) — 实用模式和 unsafe 深度讲解
- [**This Week in Rust**](https://this-week-in-rust.org/) — 社区发现塑造了许多示例
- ……以及 **Rust 社区** 中众多成员的博客文章、会议演讲、RFC 和论坛讨论——人数众多无法一一列出，但深表感谢

## 📖 开始阅读

选择与你背景相匹配的书籍。书籍按难度分组，方便你规划学习路径：

| 级别 | 说明 |
|-------|-------------|
| 🟢 **桥接** | 从另一门语言学习 Rust——从这里开始 |
| 🔵 **深度探索** | 深入探索 Rust 主要子系统 |
| 🟡 **进阶** | 为有经验的 Rustacean 准备的模式和技巧 |
| 🟣 **专家** | 前沿的类型级和正确性技术 |
| 🟤 **工程实践** | 工程、工具链和生产就绪 |

| 书籍 | 级别 | 适用人群 |
|------|-------|-------------|
| [**C/C++程序员的Rust**](c-cpp-book/src/SUMMARY.md) | 🟢 桥接 | 移动语义、RAII、FFI、嵌入式、no_std |
| [**C#程序员的Rust**](csharp-book/src/SUMMARY.md) | 🟢 桥接 | Swift / C# / Java → 所有权与类型系统 |
| [**Python程序员的Rust**](python-book/src/SUMMARY.md) | 🟢 桥接 | 动态→静态类型、GIL无关的并发 |
| [**异步Rust**](async-book/src/SUMMARY.md) | 🔵 深度探索 | Tokio、流、取消安全性 |
| [**Rust模式**](rust-patterns-book/src/SUMMARY.md) | 🟡 进阶 | Pin、分配器、无锁数据结构、unsafe |
| [**类型驱动的正确性**](type-driven-correctness-book/src/SUMMARY.md) | 🟣 专家 | 类型状态、幽灵类型、能力令牌 |
| [**Rust工程实践**](engineering-book/src/SUMMARY.md) | 🟤 工程实践 | 构建脚本、交叉编译、CI/CD、Miri |

每本书包含 15–16 章，配有 Mermaid 图表、可编辑的 Rust 练习场、习题和全文搜索。

> **提示：** 你可以直接在 GitHub 上阅读 Markdown 源码，也可以在 GitHub Pages 上浏览渲染后的站点（含侧边栏导航和搜索，链接见仓库 About 部分）。
>
> **本地服务：** 最佳阅读体验（章节间键盘导航、即时搜索、离线访问），请克隆仓库并运行：
> ```
> # 如果还没有 Rust，通过 rustup 安装：
> # https://rustup.rs/
>
> cargo install mdbook mdbook-mermaid
> cargo xtask serve          # 构建所有书籍并打开本地服务器
> ```

---

## 🔧 维护者指南

<details>
<summary>本地构建、服务和编辑书籍</summary>

### 前置要求

如果还没有 Rust，请先 [通过 **rustup** 安装 Rust](https://rustup.rs/)，然后：

```bash
cargo install mdbook mdbook-mermaid
```

### 构建与服务

```bash
cargo xtask build               # 构建所有书籍到 site/（本地预览）
cargo xtask serve               # 构建并在 http://localhost:3000 提供服务
cargo xtask deploy              # 构建所有书籍到 docs/（用于 GitHub Pages）
cargo xtask clean               # 删除 site/ 和 docs/
```

构建或服务单本书籍：

```bash
cd c-cpp-book && mdbook serve --open    # http://localhost:3000
```

### 部署

网站通过 `.github/workflows/pages.yml` 在推送到 `master` 时自动部署到 GitHub Pages。无需手动操作。

</details>
