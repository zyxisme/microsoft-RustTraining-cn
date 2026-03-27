## 安装和设置

> **你将学到：** 如何安装 Rust 及其工具链、Cargo 构建系统 vs pip/Poetry、
> IDE 设置、你的第一个 `Hello, world!` 程序，以及映射到 Python 等价物的 essential Rust 关键字。
>
> **难度：** 🟢 初级

### 安装 Rust
```bash
# 通过 rustup 安装 Rust（Linux/macOS/WSL）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 验证安装
rustc --version     # Rust 编译器
cargo --version     # 构建工具 + 包管理器（类似于 pip + setuptools 的组合）

# 更新 Rust
rustup update
```

### Rust 工具 vs Python 工具

| 用途 | Python | Rust |
|---------|--------|------|
| 语言运行时 | `python`（解释器） | `rustc`（编译器，很少直接调用） |
| 包管理器 | `pip` / `poetry` / `uv` | `cargo`（内置） |
| 项目配置 | `pyproject.toml` | `Cargo.toml` |
| 锁定文件 | `poetry.lock` / `requirements.txt` | `Cargo.lock` |
| 虚拟环境 | `venv` / `conda` | 不需要（依赖是按项目的） |
| 格式化工具 | `black` / `ruff format` | `rustfmt`（内置：`cargo fmt`） |
| 代码检查器 | `ruff` / `flake8` / `pylint` | `clippy`（内置：`cargo clippy`） |
| 类型检查器 | `mypy` / `pyright` | 内置到编译器（始终开启） |
| 测试运行器 | `pytest` | `cargo test`（内置） |
| 文档 | `sphinx` / `mkdocs` | `cargo doc`（内置） |
| REPL | `python` / `ipython` | 无（使用测试/`cargo run`） |

### IDE 设置

**VS Code**（推荐）：
```text
需要安装的扩展：
- rust-analyzer        ← 必需：IDE 功能、类型提示、自动补全
- Even Better TOML     ← Cargo.toml 语法高亮
- CodeLLDB             ← 调试器支持

# Python 等价映射：
# rust-analyzer ≈ Pylance（但 100% 类型覆盖，始终开启）
# cargo clippy  ≈ ruff（但检查正确性，而不仅仅是样式）
```

---

## 你的第一个 Rust 程序

### Python Hello World
```python
# hello.py — 直接运行
print("Hello, World!")

# 运行：
# python hello.py
```

### Rust Hello World
```rust
// src/main.rs — 必须先编译
fn main() {
    println!("Hello, World!");   // println! 是一个宏（感叹号很重要）
}

// 构建并运行：
// cargo run
```

### Python 开发者的关键差异

```text
Python:                              Rust:
─────────                            ─────
- 不需要 main()                      - fn main() 是入口点
- 缩进 = 代码块                      - 花括号 {} = 代码块
- print() 是一个函数                - println!() 是一个宏（! 很重要）
- 无分号                             - 分号结束语句
- 无类型声明                         - 类型推断但始终已知
- 解释型（直接运行）                 - 编译型（cargo build，然后运行）
- 运行时错误                         - 大多数错误在编译时
```

### 创建你的第一个项目
```bash
# Python                              # Rust
mkdir myproject                        cargo new myproject
cd myproject                           cd myproject
python -m venv .venv                   # 不需要虚拟环境
source .venv/bin/activate              # 不需要激活
# 手动创建文件                        # src/main.rs 已创建

# Python 项目结构：                   Rust 项目结构：
# myproject/                          myproject/
# ├── pyproject.toml                  ├── Cargo.toml        （类似于 pyproject.toml）
# ├── src/                            ├── src/
# │   └── myproject/                  │   └── main.rs       （入口点）
# │       ├── __init__.py             └── （不需要 __init__.py）
# │       └── main.py
# └── tests/
#     └── test_main.py
```

```mermaid
graph TD
    subgraph Python ["Python 项目"]
        PP["pyproject.toml"] --- PS["src/"]
        PS --- PM["myproject/"]
        PM --- PI["__init__.py"]
        PM --- PMN["main.py"]
        PP --- PT["tests/"]
    end
    subgraph Rust ["Rust 项目"]
        RC["Cargo.toml"] --- RS["src/"]
        RS --- RM["main.rs"]
        RC --- RTG["target/ (自动生成)"]
    end
    style Python fill:#ffeeba
    style Rust fill:#d4edda
```

> **关键差异**：Rust 项目更简单 — 无 `__init__.py`、无虚拟环境、无 `setup.py` vs `setup.cfg` vs `pyproject.toml` 的混乱。只有 `Cargo.toml` + `src/`。

---

## Cargo vs pip/Poetry

### 项目配置

```toml
# Python — pyproject.toml
[project]
name = "myproject"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "requests>=2.28",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]
```

```toml
# Rust — Cargo.toml
[package]
name = "myproject"
version = "0.1.0"
edition = "2021"          # Rust 版本（类似于 Python 版本）

[dependencies]
reqwest = "0.12"          # HTTP 客户端（类似于 requests）
serde = { version = "1.0", features = ["derive"] }  # 序列化（类似于 pydantic）

[dev-dependencies]
# 测试依赖 — 仅在为 `cargo test` 编译时
# （不需要单独的测试配置 — `cargo test` 是内置的）
```

### 常用 Cargo 命令
```bash
# Python 等价命令                # Rust
pip install requests               cargo add reqwest
pip install -r requirements.txt    cargo build           # 自动安装依赖
pip install -e .                   cargo build            # 始终是"可编辑的"
python -m pytest                   cargo test
python -m mypy .                   # 内置到编译器 — 始终运行
ruff check .                       cargo clippy
ruff format .                      cargo fmt
python main.py                     cargo run
python -c "..."                    # 无等价命令 — 使用 cargo run 或测试

# Rust 特有：
cargo new myproject                # 创建新项目
cargo build --release              # 优化构建（比调试快 10-100 倍）
cargo doc --open                   # 生成并浏览 API 文档
cargo update                       # 更新依赖（类似于 pip install --upgrade）
```

---

## Python 开发者的 essential Rust 关键字

### 变量和可变性关键字

```rust
// let — 声明变量（类似于 Python 赋值，但默认不可变）
let name = "Alice";          // Python: name = "Alice"（但 Python 是可变的）
// name = "Bob";             // ❌ 编译错误！默认不可变

// mut — 选择加入可变性
let mut count = 0;           // Python: count = 0（在 Python 中始终可变）
count += 1;                  // ✅ 允许因为有 `mut`

// const — 编译时常量（类似于 Python 的 UPPER_CASE 约定，但有强制执行）
const MAX_SIZE: usize = 1024;   // Python: MAX_SIZE = 1024（只是约定）
// static — 全局变量（少用；Python 有模块级全局变量）
static VERSION: &str = "1.0";
```

### 所有权和借用关键字

```rust
// 这些没有 Python 等价物 — 它们是 Rust 特有的概念

// & — 借用（只读引用）
fn print_name(name: &str) { }    // Python: def print_name(name: str) — 但 Python 始终传递引用

// &mut — 可变借用
fn append(list: &mut Vec<i32>) { }  // Python: def append(lst: list) — 在 Python 中始终可变

// move — 转移所有权（在 Rust 中隐式发生，在 Python 中永远不会）
let s1 = String::from("hello");
let s2 = s1;    // s1 被移动到 s2 — s1 不再有效
// println!("{}", s1);  // ❌ 编译错误：值已移动
```

### 类型定义关键字

```rust
// struct — 类似于 Python 的 dataclass 或 NamedTuple
struct Point {               // @dataclass
    x: f64,                  // class Point:
    y: f64,                  //     x: float
}                            //     y: float

// enum — 类似于 Python 的 enum 但强大得多（可以携带数据）
enum Shape {                 // 没有直接的 Python 等价物
    Circle(f64),             // 每个变体可以持有不同数据
    Rectangle(f64, f64),
}

// impl — 给类型附加方法（类似于在类中定义方法）
impl Point {                 // class Point:
    fn distance(&self) -> f64 {  //     def distance(self) -> float:
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

// trait — 类似于 Python 的 ABC 或 Protocol（PEP 544）
trait Drawable {             // class Drawable(Protocol):
    fn draw(&self);          //     def draw(self) -> None: ...
}

// type — 类型别名（类似于 Python 的 TypeAlias）
type UserId = i64;           // UserId = int  （或 TypeAlias）
```

### 控制流关键字

```rust
// match — 穷尽模式匹配（类似于 Python 3.10+ match，但有强制执行）
match value {
    1 => println!("one"),
    2 | 3 => println!("two or three"),
    _ => println!("other"),          // _ = 通配符（类似于 Python 的 case _:)
}

// if let — 解构 + 条件（Python 风格：if (m := regex.match(s)):）
if let Some(x) = optional_value {
    println!("{}", x);
}

// loop — 无限循环（类似于 while True:）
loop {
    break;  // 必须 break 才能退出
}

// for — 迭代（类似于 Python 的 for，但更经常需要 .iter()）
for item in collection.iter() {      // for item in collection:
    println!("{}", item);
}

// while let — 带解构的循环
while let Some(item) = stack.pop() {
    process(item);
}
```

### 可见性关键字

```rust
// pub — 公开（Python 没有真正的私有；使用 _ 约定）
pub fn greet() { }           // def greet():  — 在 Python 中一切都是"公开的"

// pub(crate) — 仅在 crate 内可见
pub(crate) fn internal() { } // def _internal():  — 单下划线约定

// （无关键字）— 对模块私有
fn private_helper() { }      // def __private():  — 双下划线名称改写

// 在 Python 中，"私有"是一种君子协定。
// 在 Rust 中，私有由编译器强制执行。
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：第一个 Rust 程序</strong>（点击展开）</summary>

**挑战**：创建一个新的 Rust 项目并编写一个程序：
1. 声明一个变量 `name` 包含你的名字（类型 `&str`）
2. 声明一个可变变量 `count`，初始值为 0
3. 使用 `for` 循环从 1..=5 来增加 `count` 并打印 `"Hello, {name}! (count: {count})"`
4. 循环结束后，使用 `match` 表达式打印 count 是奇数还是偶数

<details>
<summary>🔑 解决方案</summary>

```bash
cargo new hello_rust && cd hello_rust
```

```rust
// src/main.rs
fn main() {
    let name = "Pythonista";
    let mut count = 0u32;

    for _ in 1..=5 {
        count += 1;
        println!("Hello, {name}! (count: {count})");
    }

    let parity = match count % 2 {
        0 => "even",
        _ => "odd",
    };
    println!("Final count {count} is {parity}");
}
```

**关键收获**：
- `let` 默认不可变（需要 `mut` 才能改变 `count`）
- `1..=5` 是 inclusive 范围（Python 的 `range(1, 6)`）
- `match` 是一个返回值的表达式
- 无 `self`，无 `if __name__ == "__main__"` — 只有 `fn main()`

</details>
</details>

***
