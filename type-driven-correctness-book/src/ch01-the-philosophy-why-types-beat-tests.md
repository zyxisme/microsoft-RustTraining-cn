# 哲学理念 — 为什么类型比测试更好 🟢

> **你将学到：** 编译时正确性的三个级别（值、状态、协议），类型级证明背后的 Curry-Howard 直观理解，以及正确性构造模式何时值得——何时不值得——投入。
>
> **交叉引用：** [ch02](ch02-typed-command-interfaces-request-determi.md)（类型化命令），[ch05](ch05-protocol-state-machines-type-state-for-r.md)（类型状态），[ch13](ch13-reference-card.md)（参考卡）

## 运行时检查的代价

考虑一个典型的诊断代码库中的运行时保护：

```rust,ignore
fn read_sensor(sensor_type: &str, raw: &[u8]) -> f64 {
    match sensor_type {
        "temperature" => raw[0] as i8 as f64,          // 有符号字节
        "fan_speed"   => u16::from_le_bytes([raw[0], raw[1]]) as f64,
        "voltage"     => u16::from_le_bytes([raw[0], raw[1]]) as f64 / 1000.0,
        _             => panic!("unknown sensor type: {sensor_type}"),
    }
}
```

这个函数有 **四种编译器无法捕获的失败模式**：

1. 拼写错误：`"temperture"` → 运行时 panic
2. 错误的 `raw` 长度：`fan_speed` 只有 1 字节 → 运行时 panic
3. 调用者将返回的 `f64` 用作 RPM 而实际上是 °C → 逻辑错误，静默发生
4. 添加了新传感器类型但未更新此 `match` → 运行时 panic

每种失败模式都是在 **部署后** 才发现的。测试有帮助，但只覆盖了有人想到去写的情况。类型系统覆盖 **所有** 情况，包括没人想到的情况。

## 正确性的三个级别

### 第一级 — 值正确性
**使无效值不可表示。**

```rust,ignore
// ❌ 任何 u16 都可以是"端口"——0 是无效的但可以编译
fn connect(port: u16) { /* ... */ }

// ✅ 只有经过验证的端口才能存在
pub struct Port(u16);  // 私有字段

impl TryFrom<u16> for Port {
    type Error = &'static str;
    fn try_from(v: u16) -> Result<Self, Self::Error> {
        if v > 0 { Ok(Port(v)) } else { Err("port must be > 0") }
    }
}

fn connect(port: Port) { /* ... */ }
// Port(0) 永远无法构造——不变量在所有地方都成立
```

**硬件示例：** `SensorId(u8)` — 包装原始传感器编号并验证其在 SDR 范围内。

### 第二级 — 状态正确性
**使无效转换不可表示。**

```rust,ignore
use std::marker::PhantomData;

struct Disconnected;
struct Connected;

struct Socket<State> {
    fd: i32,
    _state: PhantomData<State>,
}

impl Socket<Disconnected> {
    fn connect(self, addr: &str) -> Socket<Connected> {
        // ... 连接逻辑 ...
        Socket { fd: self.fd, _state: PhantomData }
    }
}

impl Socket<Connected> {
    fn send(&mut self, data: &[u8]) { /* ... */ }
    fn disconnect(self) -> Socket<Disconnected> {
        Socket { fd: self.fd, _state: PhantomData }
    }
}

// Socket<Disconnected> 没有 send() 方法——如果你尝试调用会得到编译错误
```

**硬件示例：** GPIO 引脚模式 — `Pin<Input>` 有 `read()` 但没有 `write()`。

### 第三级 — 协议正确性
**使无效交互不可表示。**

```rust,ignore
use std::io;

trait IpmiCmd {
    type Response;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

// 为简化说明——完整的 trait 包含 net_fn()、cmd_byte()、
// payload() 和 parse_response()，见 ch02。

struct ReadTemp { sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        Ok(Celsius(raw[0] as i8 as f64))
    }
}

# #[derive(Debug)] struct Celsius(f64);

fn execute<C: IpmiCmd>(cmd: &C, raw: &[u8]) -> io::Result<C::Response> {
    cmd.parse_response(raw)
}
// ReadTemp 总是返回 Celsius——不可能意外得到 Rpm
```

**硬件示例：** IPMI、Redfish、NVMe Admin 命令——请求类型决定响应类型。

## Curry-Howard 联系（简化版）

在编程语言理论中，**Curry-Howard 对应** 表明：类型是命题，程序是证明。当你写：

```rust,ignore
fn execute<C: IpmiCmd>(cmd: &C) -> io::Result<C::Response>
```

你不仅仅是写了一个函数——你是在陈述一个 **定理**："对于任何实现了 `IpmiCmd` 的命令类型 `C`，执行它会产生恰好是 `C::Response` 的结果。" 编译器每次编译你的代码时都会 **证明** 这个定理。如果证明失败，程序就不可能存在。

你不需要理解理论才能使用这些模式。但这解释了他 Rust 的类型系统为何如此强大——它不仅仅是捕获错误，而是在 **证明正确性**。

## 何时不使用这些模式

正确性构造并不总是正确的选择：

| 情况 | 建议 |
|-----------|---------------|
| 安全关键边界（电源排序、加密） | ✅ 始终使用——这里的 bug 会熔化硬件或泄露秘密 |
| 跨模块公共 API | ✅ 通常使用——误用应该是编译错误 |
| 3+ 状态的状态机 | ✅ 通常使用——类型状态防止错误转换 |
| 一个 50 行函数内部的辅助函数 | ❌ 过度设计——简单的 `assert!` 就够了 |
| 原型设计/探索未知硬件 | ❌ 先用原始类型——在理解行为后进行细化 |
| 用户面向的 CLI 解析 | ⚠️ 在边界处使用 `clap` + `TryFrom`，内部使用原始类型即可 |

关键问题是：**"如果这个 bug 发生在生产环境中，会有多糟糕？"**

- 风扇停止 → GPU 熔化 → **使用类型**
- 错误的 DER 记录 → 客户得到坏数据 → **使用类型**
- 调试日志消息略有错误 → **使用 `assert!`**

## 关键要点

1. **正确性的三个级别** — 值（newtype）、状态（类型状态）、协议（关联类型）——每个级别消除更广泛的 bug 类别。
2. **实践中的 Curry-Howard** — 每个泛型函数签名都是一个定理，编译器在每次构建时都会证明它。
3. **代价问题** — "如果这个 bug 发布，会有多糟糕？"决定了类型还是测试是正确的工具。
4. **类型补充测试** — 它们消除了整个 *类别*；测试覆盖特定的 *值* 和边界情况。
5. **知道何时停止** — 内部辅助函数和一次性原型很少需要类型级强制执行。

---

