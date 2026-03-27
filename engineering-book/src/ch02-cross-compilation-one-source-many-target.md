# 交叉编译 — 一份源码，多个目标平台 🟡

> **你将学到：**
> - Rust 目标三元组如何工作以及如何使用 `rustup` 添加它们
> - 为容器/云部署构建静态 musl 二进制文件
> - 使用原生工具链、`cross` 和 `cargo-zigbuild` 交叉编译到 ARM (aarch64)
> - 为多架构 CI 设置 GitHub Actions 矩阵构建
>
> **交叉引用：** [构建脚本](ch01-build-scripts-buildrs-in-depth.md) — build.rs 在交叉编译期间在 HOST 上运行 · [发布 Profiles](ch07-release-profiles-and-binary-size.md) — 交叉编译发布二进制文件的 LTO 和 strip 设置 · [Windows](ch10-windows-and-conditional-compilation.md) — Windows 交叉编译和 `no_std` 目标

交叉编译是指在一台机器（**主机**）上构建可在另一台机器（**目标**）上运行的可执行文件。
主机可能是你的 x86_64 笔记本电脑；目标可能是 ARM 服务器、基于 musl 的容器，甚至是 Windows 机器。
Rust 使这变得非常可行，因为 `rustc` 本身就是一个交叉编译器——
它只需要正确的目标库和兼容的链接器。

### 目标三元组解剖

每个 Rust 编译目标都由一个**目标三元组**标识（尽管名称如此，它通常有四个部分）：

```text
<arch>-<vendor>-<os>-<env>

示例：
  x86_64  - unknown - linux  - gnu      ← 标准 Linux (glibc)
  x86_64  - unknown - linux  - musl     ← 静态 Linux (musl libc)
  aarch64 - unknown - linux  - gnu      ← ARM 64位 Linux
  x86_64  - pc      - windows- msvc     ← Windows + MSVC
  aarch64 - apple   - darwin             ← Apple Silicon 上的 macOS
  x86_64  - unknown - none              ← 裸机（无 OS）
```

列出所有可用目标：

```bash
# 显示 rustc 可以编译的所有目标（约 250 个目标）
rustc --print target-list | wc -l

# 显示系统上已安装的目标
rustup target list --installed

# 显示当前默认目标
rustc -vV | grep host
```

### 使用 rustup 安装工具链

```bash
# 添加目标库（该目标的 Rust std、core 和 alloc）
rustup target add x86_64-unknown-linux-musl
rustup target add aarch64-unknown-linux-gnu

# 现在你可以交叉编译了：
cargo build --target x86_64-unknown-linux-musl
cargo build --target aarch64-unknown-linux-gnu  # 需要链接器 — 见下文
```

**`rustup target add` 给你的是**：该目标的预编译 `std`、`core` 和 `alloc`
库。它*不会*给你 C 链接器或 C 库。对于需要 C 工具链的目标（大多数 `gnu` 目标），
你需要单独安装一个。

```bash
# Ubuntu/Debian — 为 aarch64 安装交叉链接器
sudo apt install gcc-aarch64-linux-gnu

# Ubuntu/Debian — 为静态构建安装 musl 工具链
sudo apt install musl-tools

# Fedora
sudo dnf install gcc-aarch64-linux-gnu
```

### `.cargo/config.toml` — 每目标配置

不要在每个命令上传递 `--target`，而是在项目根目录或主目录的
`.cargo/config.toml` 中配置默认值：

```toml
# .cargo/config.toml

# 此项目的默认目标（可选 — 省略以保持原生默认）
# [build]
# target = "x86_64-unknown-linux-musl"

# aarch64 交叉编译的链接器
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
rustflags = ["-C", "target-feature=+crc"]

# musl 静态构建的链接器（通常系统 gcc 就可以）
[target.x86_64-unknown-linux-musl]
linker = "musl-gcc"
rustflags = ["-C", "target-feature=+crc,+aes"]

# ARM 32位（Raspberry Pi、嵌入式）
[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"

# 所有目标的环境变量
[env]
# 示例：设置自定义 sysroot
# SYSROOT = "/opt/cross/sysroot"
```

**配置文件搜索顺序**（首先匹配获胜）：
1. `<project>/.cargo/config.toml`
2. `<project>/../.cargo/config.toml`（父目录，向上遍历）
3. `$CARGO_HOME/config.toml`（通常为 `~/.cargo/config.toml`）

### 使用 musl 的静态二进制文件

对于部署到最小容器（Alpine、scratch Docker 镜像）或你无法控制 glibc
版本的系统，使用 musl 构建：

```bash
# 安装 musl 目标
rustup target add x86_64-unknown-linux-musl
sudo apt install musl-tools  # 提供 musl-gcc

# 构建完全静态的二进制文件
cargo build --release --target x86_64-unknown-linux-musl

# 验证它是静态的
file target/x86_64-unknown-linux-musl/release/diag_tool
# → ELF 64-bit LSB executable, x86-64, statically linked

ldd target/x86_64-unknown-linux-musl/release/diag_tool
# → not a dynamic executable
```

**静态与动态的权衡：**

| 方面 | glibc（动态） | musl（静态） |
|--------|-----------------|---------------|
| 二进制文件大小 | 更小（共享库） | 更大（约增加 5-15 MB） |
| 可移植性 | 需要匹配版本的 glibc | 可在任何 Linux 上运行 |
| DNS 解析 | 完整的 `nsswitch` 支持 | 基本解析器（无 mDNS） |
| 部署 | 需要 sysroot 或容器 | 单个二进制文件，无依赖 |
| 性能 | malloc 略快 | malloc 略慢 |
| `dlopen()` 支持 | 有 | 无 |

> **对于本项目**：静态 musl 构建非常适合部署到你无法保证主机 OS 版本的多样化
> 服务器硬件。单二进制文件部署模型消除了"works on my machine"问题。

### 交叉编译到 ARM (aarch64)

ARM 服务器（AWS Graviton、Ampere Altra、Grace）在数据中心越来越常见。
从 x86_64 主机交叉编译到 aarch64：

```bash
# 步骤 1：安装目标 + 交叉链接器
rustup target add aarch64-unknown-linux-gnu
sudo apt install gcc-aarch64-linux-gnu

# 步骤 2：在 .cargo/config.toml 中配置链接器（见上文）

# 步骤 3：构建
cargo build --release --target aarch64-unknown-linux-gnu

# 步骤 4：验证二进制文件
file target/aarch64-unknown-linux-gnu/release/diag_tool
# → ELF 64-bit LSB executable, ARM aarch64
```

**为目标架构运行测试**需要：
- 一台实际的 ARM 机器
- QEMU 用户模式模拟

```bash
# 安装 QEMU 用户模式（在 x86_64 上运行 ARM 二进制文件）
sudo apt install qemu-user qemu-user-static binfmt-support

# 现在 cargo test 可以通过 QEMU 运行交叉编译的测试
cargo test --target aarch64-unknown-linux-gnu
#（很慢——每个测试二进制文件都被模拟。仅用于 CI 验证，不适合日常开发。）
```

在 `.cargo/config.toml` 中将 QEMU 配置为测试运行器：

```toml
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"
runner = "qemu-aarch64-static -L /usr/aarch64-linux-gnu"
```

### `cross` 工具 — 基于 Docker 的交叉编译

[`cross`](https://github.com/cross-rs/cross) 工具使用预配置的 Docker 镜像
提供了零设置交叉编译体验：

```bash
# 安装 cross（从 crates.io — 稳定版本）
cargo install cross
# 或从 git 获取最新功能（不太稳定）：
# cargo install cross --git https://github.com/cross-rs/cross

# 交叉编译 — 无需工具链设置！
cross build --release --target aarch64-unknown-linux-gnu
cross build --release --target x86_64-unknown-linux-musl
cross build --release --target armv7-unknown-linux-gnueabihf

# 交叉测试 — QEMU 包含在 Docker 镜像中
cross test --target aarch64-unknown-linux-gnu
```

**它如何工作**：`cross` 替换 `cargo` 并在预安装了正确交叉编译工具链的
Docker 容器内运行构建。你的源码被挂载到容器中，输出进入你正常的 `target/`
目录。

**使用 `Cross.toml` 自定义 Docker 镜像**：

```toml
# Cross.toml
[target.aarch64-unknown-linux-gnu]
# 使用带有额外系统库的自定义 Docker 镜像
image = "my-registry/cross-aarch64:latest"

# 预安装系统包
pre-build = [
    "dpkg --add-architecture arm64",
    "apt-get update && apt-get install -y libpci-dev:arm64"
]

[target.aarch64-unknown-linux-gnu.env]
# 将环境变量传递到容器中
passthrough = ["CI", "GITHUB_TOKEN"]
```

`cross` 需要 Docker（或 Podman），但消除了手动安装交叉编译器、sysroot 和 QEMU 的需要。
这是 CI 的推荐方法。

### 使用 Zig 作为交叉编译链接器

[Zig](https://ziglang.org/) 将 C 编译器和约 40 个目标的交叉编译 sysroot
打包在单个约 40 MB 的下载中。这使得它成为 Rust 的一个非常方便的
交叉链接器：

```bash
# 安装 Zig（单个二进制文件，无需包管理器）
# 从 https://ziglang.org/download/ 下载
# 或通过包管理器：
sudo snap install zig --classic --beta  # Ubuntu
brew install zig                          # macOS

# 安装 cargo-zigbuild
cargo install cargo-zigbuild
```

**为什么选择 Zig？** 关键优势是 **glibc 版本定位**。Zig 让你指定
要链接的确切 glibc 版本，确保你的二进制文件在较旧的 Linux
发行版上运行：

```bash
# 为 glibc 2.17 构建（CentOS 7 / RHEL 7 兼容性）
cargo zigbuild --release --target x86_64-unknown-linux-gnu.2.17

# 为带有 glibc 2.28 的 aarch64 构建（Ubuntu 18.04+）
cargo zigbuild --release --target aarch64-unknown-linux-gnu.2.28

# 为 musl 构建（完全静态）
cargo zigbuild --release --target x86_64-unknown-linux-musl
```

`.2.17` 后缀是 Zig 扩展——它告诉 Zig 的链接器使用 glibc 2.17
符号版本，因此生成的二进制文件可以在 CentOS 7 及更高版本上运行。
无需 Docker、无需 sysroot 管理、无需交叉编译器安装。

**比较：cross vs cargo-zigbuild vs 手动：**

| 功能 | 手动 | cross | cargo-zigbuild |
|---------|--------|-------|----------------|
| 设置工作量 | 高（每个目标安装工具链） | 低（需要 Docker） | 低（单个二进制文件） |
| 需要 Docker | 否 | 是 | 否 |
| glibc 版本定位 | 否（使用主机 glibc） | 否（使用容器 glibc） | 是（精确版本） |
| 测试执行 | 需要 QEMU | 包含在内 | 需要 QEMU |
| macOS → Linux | 困难 | 简单 | 简单 |
| Linux → macOS | 非常困难 | 不支持 | 有限 |
| 二进制文件大小开销 | 无 | 无 | 无 |

### CI 流水线：GitHub Actions 矩阵

为多个目标构建的生产级 CI 工作流：

```yaml
# .github/workflows/cross-build.yml
name: Cross-Platform Build

on: [push, pull_request]

env:
  CARGO_TERM_COLOR: always

jobs:
  build:
    strategy:
      matrix:
        include:
          - target: x86_64-unknown-linux-gnu
            os: ubuntu-latest
            name: linux-x86_64
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
            name: linux-x86_64-static
          - target: aarch64-unknown-linux-gnu
            os: ubuntu-latest
            name: linux-aarch64
            use_cross: true
          - target: x86_64-pc-windows-msvc
            os: windows-latest
            name: windows-x86_64

    runs-on: ${{ matrix.os }}
    name: Build (${{ matrix.name }})

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install musl tools
        if: matrix.target == 'x86_64-unknown-linux-musl'
        run: sudo apt-get install -y musl-tools

      - name: Install cross
        if: matrix.use_cross
        run: cargo install cross

      - name: Build (native)
        if: "!matrix.use_cross"
        run: cargo build --release --target ${{ matrix.target }}

      - name: Build (cross)
        if: matrix.use_cross
        run: cross build --release --target ${{ matrix.target }}

      - name: Run tests
        if: "!matrix.use_cross"
        run: cargo test --target ${{ matrix.target }}

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: diag_tool-${{ matrix.name }}
          path: target/${{ matrix.target }}/release/diag_tool*
```

### 应用：多架构服务器构建

该二进制文件目前没有交叉编译设置。对于部署在不同服务器集群中的硬件
诊断工具，建议的添加：

```text
my_workspace/
├── .cargo/
│   └── config.toml          ← 每个目标的链接器配置
├── Cross.toml                ← cross 工具配置
└── .github/workflows/
    └── cross-build.yml       ← 3个目标的 CI 矩阵
```

**推荐的 `.cargo/config.toml`：**

```toml
# 项目的 .cargo/config.toml

# 发布 profile 优化（已在 Cargo.toml 中，显示供参考）
# [profile.release]
# lto = true
# codegen-units = 1
# panic = "abort"
# strip = true

# aarch64 用于 ARM 服务器（Graviton、Ampere、Grace）
[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

# musl 用于可移植静态二进制文件
[target.x86_64-unknown-linux-musl]
linker = "musl-gcc"
```

**推荐的构建目标：**

| 目标 | 使用场景 | 部署到 |
|--------|----------|-----------|
| `x86_64-unknown-linux-gnu` | 默认原生构建 | 标准 x86 服务器 |
| `x86_64-unknown-linux-musl` | 静态二进制文件，任何发行版 | 容器、最小主机 |
| `aarch64-unknown-linux-gnu` | ARM 服务器 | Graviton、Ampere、Grace |

> **关键见解**：工作空间根 `Cargo.toml` 中的 `[profile.release]`
> 已经有 `lto = true`、`codegen-units = 1`、`panic = "abort"`
> 和 `strip = true`——这是交叉编译部署二进制文件的理想发布 profile
>（参见[发布 Profiles](ch07-release-profiles-and-binary-size.md) 了解完整影响表）。
> 与 musl 结合，这会生成一个约 10 MB 的单一静态二进制文件，无运行时依赖。

### 交叉编译故障排除

| 症状 | 原因 | 修复 |
|---------|-------|-----|
| `linker 'aarch64-linux-gnu-gcc' not found` | 缺少交叉链接器工具链 | `sudo apt install gcc-aarch64-linux-gnu` |
| `cannot find -lssl`（musl 目标） | 系统 OpenSSL 是 glibc 链接的 | 使用 `vendored` 特性：`openssl = { version = "0.10", features = ["vendored"] }` |
| `build.rs` 运行了错误的二进制文件 | build.rs 在 HOST 上运行，而不是目标 | 在 build.rs 中检查 `CARGO_CFG_TARGET_OS`，而不是 `cfg!(target_os)` |
| 测试在本地通过，在 `cross` 中失败 | Docker 镜像缺少测试固件 | 通过 `Cross.toml` 挂载测试数据：`[build.env] volumes = ["./test_area:/test_area"]` |
| `undefined reference to __cxa_thread_atexit_impl` | 目标上的 glibc 太旧 | 使用带有精确 glibc 版本的 `cargo-zigbuild`：`--target x86_64-unknown-linux-gnu.2.17` |
| 二进制文件在 ARM 上 segfault | 为错误的 ARM 变体编译 | 验证目标三元组与硬件匹配：64位 ARM 使用 `aarch64-unknown-linux-gnu` |
| 运行时出现 `GLIBC_2.XX not found` | 构建机器有较新的 glibc | 使用 musl 进行静态构建，或使用 `cargo-zigbuild` 进行 glibc 版本固定 |

### 交叉编译决策树

```mermaid
flowchart TD
    START["Need to cross-compile?"] --> STATIC{"Static binary?"}

    STATIC -->|Yes| MUSL["musl target\n--target x86_64-unknown-linux-musl"]
    STATIC -->|No| GLIBC{"Need old glibc?"}

    GLIBC -->|Yes| ZIG["cargo-zigbuild\n--target x86_64-unknown-linux-gnu.2.17"]
    GLIBC -->|No| ARCH{"Target arch?"}

    ARCH -->|"Same arch"| NATIVE["Native toolchain\nrustup target add + linker"]
    ARCH -->|"ARM/other"| DOCKER{"Docker available?"}

    DOCKER -->|Yes| CROSS["cross build\nDocker-based, zero setup"]
    DOCKER -->|No| MANUAL["Manual sysroot\napt install gcc-aarch64-linux-gnu"]

    style MUSL fill:#91e5a3,color:#000
    style ZIG fill:#91e5a3,color:#000
    style CROSS fill:#91e5a3,color:#000
    style NATIVE fill:#e3f2fd,color:#000
    style MANUAL fill:#ffd43b,color:#000
```

### 🏋️ 练习

#### 🟢 练习 1：静态 musl 二进制文件

为 `x86_64-unknown-linux-musl` 构建任何 Rust 二进制文件。
使用 `file` 和 `ldd` 验证它是静态链接的。

<details>
<summary>解决方案</summary>

```bash
rustup target add x86_64-unknown-linux-musl
cargo new hello-static && cd hello-static
cargo build --release --target x86_64-unknown-linux-musl

# 验证
file target/x86_64-unknown-linux-musl/release/hello-static
# Output: ... statically linked ...

ldd target/x86_64-unknown-linux-musl/release/hello-static
# Output: not a dynamic executable
```
</details>

#### 🟡 练习 2：GitHub Actions 交叉构建矩阵

编写一个 GitHub Actions 工作流，为三个目标构建 Rust 项目：
`x86_64-unknown-linux-gnu`、`x86_64-unknown-linux-musl` 和 `aarch64-unknown-linux-gnu`。
使用矩阵策略。

<details>
<summary>解决方案</summary>

```yaml
name: Cross-build
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        target:
          - x86_64-unknown-linux-gnu
          - x86_64-unknown-linux-musl
          - aarch64-unknown-linux-gnu
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}
      - name: Install cross
        run: cargo install cross --locked
      - name: Build
        run: cross build --release --target ${{ matrix.target }}
      - uses: actions/upload-artifact@v4
        with:
          name: binary-${{ matrix.target }}
          path: target/${{ matrix.target }}/release/my-binary
```
</details>

### 关键要点

- Rust 的 `rustc` 本身就是一个交叉编译器——你只需要正确的目标和链接器
- **musl** 生成完全静态的二进制文件，零运行时依赖——非常适合容器
- **`cargo-zigbuild`** 解决了企业 Linux 目标的"哪个 glibc 版本"问题
- **`cross`** 是 ARM 和其他特殊目标的最简单路径——Docker 处理 sysroot
- 始终使用 `file` 和 `ldd` 测试以验证二进制文件与你的部署目标匹配

---

