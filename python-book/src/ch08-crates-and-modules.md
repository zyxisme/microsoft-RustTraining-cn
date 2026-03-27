## Rust 模块 vs Python 包

> **你将学到：** `mod` 和 `use` vs `import`、可见性（`pub`）vs Python 基于约定的私有、
> Cargo.toml vs pyproject.toml、crates.io vs PyPI，以及 workspace vs monorepo。
>
> **难度：** 🟢 初级

### Python 模块系统
```python
# Python — 文件是模块，带 __init__.py 的目录是包

# myproject/
# ├── __init__.py          # 使其成为包
# ├── main.py
# ├── utils/
# │   ├── __init__.py      # 使 utils 成为子包
# │   ├── helpers.py
# │   └── validators.py
# └── models/
#     ├── __init__.py
#     ├── user.py
#     └── product.py

# 导入方式：
from myproject.utils.helpers import format_name
from myproject.models.user import User
import myproject.utils.validators as validators
```

### Rust 模块系统
```rust
// Rust — mod 声明创建模块树，文件提供内容

// src/
// ├── main.rs             # 包根 — 声明模块
// ├── utils/
// │   ├── mod.rs           # 模块声明（类似 __init__.py）
// │   ├── helpers.rs
// │   └── validators.rs
// └── models/
//     ├── mod.rs
//     ├── user.rs
//     └── product.rs

// 在 src/main.rs 中：
mod utils;       # 告诉 Rust 查找 src/utils/mod.rs
mod models;      # 告诉 Rust 查找 src/models/mod.rs

use utils::helpers::format_name;
use models::user::User;

// 在 src/utils/mod.rs 中：
pub mod helpers;      # 声明并重新导出 helpers.rs
pub mod validators;   # 声明并重新导出 validators.rs
```

```mermaid
graph TD
    A["main.rs<br/>(crate root)"] --> B["mod utils"]
    A --> C["mod models"]
    B --> D["utils/mod.rs"]
    D --> E["helpers.rs"]
    D --> F["validators.rs"]
    C --> G["models/mod.rs"]
    G --> H["user.rs"]
    G --> I["product.rs"]
    style A fill:#d4edda,stroke:#28a745
    style D fill:#fff3cd,stroke:#ffc107
    style G fill:#fff3cd,stroke:#ffc107
```

> **Python 等价物**：把 `mod.rs` 看作 `__init__.py` — 它声明模块导出什么。
> 包根（`main.rs` / `lib.rs`）像你的顶层包 `__init__.py`。

### 关键区别

| 概念 | Python | Rust |
|---------|--------|------|
| 模块 = 文件 | ✅ 自动 | 必须用 `mod` 声明 |
| 包 = 目录 | `__init__.py` | `mod.rs` |
| 默认公开 | ✅ 一切都是 | ❌ 默认私有 |
| 公开方式 | `_前缀` 约定 | `pub` 关键字 |
| 导入语法 | `from x import y` | `use x::y;` |
| 通配符导入 | `from x import *` | `use x::*;`（不推荐） |
| 相对导入 | `from . import sibling` | `use super::sibling;` |
| 重新导出 | `__all__` 或显式 | `pub use inner::Thing;` |

### 可见性 — 默认私有
```python
# Python — "我们都是成年人"
class User:
    def __init__(self):
        self.name = "Alice"       # 公开（按约定）
        self._age = 30            # "私有"（约定：单下划线）
        self.__secret = "shhh"    # 名称重整（不是真正私有）

# 没什么能阻止你访问 _age 甚至 __secret
print(user._age)                  # 正常工作
print(user._User__secret)        # 也能工作（名称重整）
```

```rust
// Rust — 私有由编译器强制
pub struct User {
    pub name: String,      // 公开 — 任何人都能访问
    age: i32,              // 私有 — 只有这个模块能访问
}

impl User {
    pub fn new(name: &str, age: i32) -> Self {
        User { name: name.to_string(), age }
    }

    pub fn age(&self) -> i32 {   // 公开 getter
        self.age
    }

    fn validate(&self) -> bool { // 私有方法
        self.age > 0
    }
}

// 在模块外：
let user = User::new("Alice", 30);
println!("{}", user.name);        // ✅ 公开
// println!("{}", user.age);      // ❌ 编译错误：字段是私有的
println!("{}", user.age());       // ✅ 公开方法（getter）
```

***

## Crates vs PyPI 包

### Python 包（PyPI）
```bash
# Python
pip install requests           # 从 PyPI 安装
pip install "requests>=2.28"   # 版本约束
pip freeze > requirements.txt  # 锁定版本
pip install -r requirements.txt # 复现环境
```

### Rust Crates（crates.io）
```bash
# Rust
cargo add reqwest              # 从 crates.io 安装（添加到 Cargo.toml）
cargo add reqwest@0.12         # 版本约束
# Cargo.lock 自动生成 — 无需手动步骤
cargo build                    # 下载并编译依赖
```

### Cargo.toml vs pyproject.toml
```toml
# Rust — Cargo.toml
[package]
name = "my-project"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }  # 带特性标志
reqwest = { version = "0.12", features = ["json"] }
tokio = { version = "1", features = ["full"] }
log = "0.4"

[dev-dependencies]
mockall = "0.13"
```

### Python 开发者的必备 Crates

| Python 库 | Rust Crate | 用途 |
|---------------|------------|---------|
| `requests` | `reqwest` | HTTP 客户端 |
| `json`（标准库） | `serde_json` | JSON 解析 |
| `pydantic` | `serde` | 序列化/验证 |
| `pathlib` | `std::path`（标准库） | 路径处理 |
| `os` / `shutil` | `std::fs`（标准库） | 文件操作 |
| `re` | `regex` | 正则表达式 |
| `logging` | `tracing` / `log` | 日志 |
| `click` / `argparse` | `clap` | CLI 参数解析 |
| `asyncio` | `tokio` | 异步运行时 |
| `datetime` | `chrono` | 日期和时间 |
| `pytest` | 内置 + `rstest` | 测试 |
| `dataclasses` | `#[derive(...)]` | 数据结构 |
| `typing.Protocol` | Trait | 结构化类型 |
| `subprocess` | `std::process`（标准库） | 运行外部命令 |
| `sqlite3` | `rusqlite` | SQLite |
| `sqlalchemy` | `diesel` / `sqlx` | ORM / SQL 工具包 |
| `fastapi` | `axum` / `actix-web` | Web 框架 |

***

## Workspace vs Monorepo

### Python Monorepo（典型）
```text
# Python monorepo（多种方式，无标准）
myproject/
├── pyproject.toml           # 根项目
├── packages/
│   ├── core/
│   │   ├── pyproject.toml   # 每个包有自己的配置
│   │   └── src/core/...
│   ├── api/
│   │   ├── pyproject.toml
│   │   └── src/api/...
│   └── cli/
│       ├── pyproject.toml
│       └── src/cli/...
# 工具：poetry workspaces、pip -e .、uv workspaces — 无标准
```

### Rust Workspace
```toml
# Rust — 根目录的 Cargo.toml
[workspace]
members = [
    "core",
    "api",
    "cli",
]

# 跨 workspace 共享依赖
[workspace.dependencies]
serde = { version = "1.0", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

```text
# Rust workspace 结构 — 标准化，内置在 Cargo 中
myproject/
├── Cargo.toml               # Workspace 根
├── Cargo.lock               # 所有 crate 的单一锁文件
├── core/
│   ├── Cargo.toml            # [dependencies] serde.workspace = true
│   └── src/lib.rs
├── api/
│   ├── Cargo.toml
│   └── src/lib.rs
└── cli/
    ├── Cargo.toml
    └── src/main.rs
```

```bash
# Workspace 命令
cargo build                  # 构建一切
cargo test                   # 测试一切
cargo build -p core          # 只构建 core crate
cargo test -p api            # 只测试 api crate
cargo clippy --all           # 检查一切
```

> **关键洞察**：Rust workspace 是一流的、内置在 Cargo 中的。
> Python monorepo 需要第三方工具（poetry、uv、pants），支持程度各异。
> 在 Rust workspace 中，所有 crate 共享一个 `Cargo.lock`，确保项目中的依赖版本一致。

---

## 练习

<details>
<summary><strong>🏋️ 练习：模块可见性</strong>（点击展开）</summary>

**挑战**：给定以下模块结构，预测哪些行编译，哪些不编译：

```rust
mod kitchen {
    fn secret_recipe() -> &'static str { "42 spices" }
    pub fn menu() -> &'static str { "Today's special" }

    pub mod staff {
        pub fn cook() -> String {
            format!("Cooking with {}", super::secret_recipe())
        }
    }
}

fn main() {
    println!("{}", kitchen::menu());             // A 行
    println!("{}", kitchen::secret_recipe());     # B 行
    println!("{}", kitchen::staff::cook());       # C 行
}
```

<details>
<summary>🔑 解答</summary>

- **A 行**：✅ 编译 — `menu()` 是 `pub`
- **B 行**：❌ 编译错误 — `secret_recipe()` 对 `kitchen` 是私有的
- **C 行**：✅ 编译 — `staff::cook()` 是 `pub`，且 `cook()` 能通过 `super::` 访问 `secret_recipe()`（子模块可以访问父模块的私有项）

**关键收获**：在 Rust 中，子模块可以看到父模块的私有项（类似 Python 的 `_private` 约定，但强制执行）。外部人员不能。这与 Python 相反，Python 的 `_private` 只是一个提示。

</details>
</details>

***

