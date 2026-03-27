# 参考卡

> **所有 14+ 正确性构造模式的快速参考**，包含选择流程图、模式目录、组合规则、crate 映射和 Curry-Howard 速查表。
>
> **交叉引用：** 每个章节 — 这是整本书的查找表。

## 快速参考：正确性构造模式

### 模式选择指南

```text
如果错过，bug 是灾难性的吗？
├── 是 → 可以用类型编码吗？
│         ├── 是 → 使用正确性构造
│         └── 否 → 运行时检查 + 广泛测试
└── 否 → 运行时检查就够了
```

### 模式目录

| # | 模式 | 关键 Trait/类型 | 防止 | 运行时成本 | 章节 |
|---|---------|---------------|----------|:------:|---------|
| 1 | 类型化命令 | `trait IpmiCmd { type Response; }` | 错误的响应类型 | 零 | ch02 |
| 2 | 单次使用类型 | `struct Nonce`（非 Clone/Copy） | Nonce/密钥重用 | 零 | ch03 |
| 3 | 能力令牌 | `struct AdminToken { _private: () }` | 未授权访问 | 零 | ch04 |
| 4 | 类型状态 | `Session<Active>` | 协议违反 | 零 | ch05 |
| 5 | 量纲类型 | `struct Celsius(f64)` | 单位混淆 | 零 | ch06 |
| 6 | 验证边界 | `struct ValidFru`（通过 TryFrom） | 未验证数据使用 | 解析一次 | ch07 |
| 7 | 能力混合 | `trait FanDiagMixin: HasSpi + HasI2c` | 缺失总线访问 | 零 | ch08 |
| 8 | 幽灵类型 | `Register<Width16>` | 宽度/方向不匹配 | 零 | ch09 |
| 9 | 哨兵值 → Option | `Option<u8>`（不是 `0xFF`） | 哨兵值作为值的 bug | 零 | ch11 |
| 10 | 密封 Trait | `trait Cmd: private::Sealed` | 不安全的外部实现 | 零 | ch11 |
| 11 | 非穷尽枚举 | `#[non_exhaustive] enum Sku` | 静默 match 穿越 | 零 | ch11 |
| 12 | 类型状态构建器 | `DerBuilder<Set, Missing>` | 不完整构造 | 零 | ch11 |
| 13 | FromStr 验证 | `impl FromStr for DiagLevel` | 未验证的字符串输入 | 解析一次 | ch11 |
| 14 | Const 泛型大小 | `RegisterBank<const N: usize>` | 缓冲区大小不匹配 | 零 | ch11 |
| 15 | 安全的 `unsafe` 包装器 | `MmioRegion::read_u32()` | 未检查的 MMIO/FFI | 零 | ch11 |
| 16 | 异步类型状态 | `AsyncSession<Active>` | 异步协议违反 | 零 | ch11 |
| 17 | Const 断言 | `SdrSensorId<const N: u8>` | 无效的编译时 ID | 零 | ch11 |
| 18 | 会话类型 | `Chan<SendRequest>` | 乱序通道操作 | 零 | ch11 |
| 19 | Pin 自引用 | `Pin<Box<StreamParser>>` | 悬空结构内指针 | 零 | ch11 |
| 20 | RAII / Drop | `impl Drop for Session` | 任何退出路径上的资源泄漏 | 零 | ch11 |
| 21 | 错误类型层次结构 | `#[derive(Error)] enum DiagError` | 静默错误吞没 | 零 | ch11 |
| 22 | `#[must_use]` | `#[must_use] struct Token` | 静默丢弃的值 | 零 | ch11 |

### 组合规则

```text
能力令牌 + 类型状态 = 授权状态转换
类型化命令 + 量纲类型 = 物理类型化响应
验证边界 + 幽灵类型 = 验证配置上的类型化寄存器访问
能力混合 + 类型化命令 = 总线感知的类型化操作
单次使用类型 + 类型状态 = 消耗即转换协议
密封 Trait + 类型化命令 = 闭合、声音的命令集
哨兵值 → Option + 验证边界 = 清洁的解析一次管道
类型状态构建器 + 能力令牌 = 完整构造证明
FromStr + #[non_exhaustive] = 可演进、快速失败的枚举解析
Const 泛型大小 + 验证边界 = 有大小的验证协议缓冲区
安全 unsafe 包装器 + 幽灵类型 = 类型化、安全的 MMIO 访问
异步类型状态 + 能力令牌 = 授权异步转换
会话类型 + 类型化命令 = 完全类型化的请求-响应通道
Pin + 类型状态 = 不能移动的自引用状态机
RAII（Drop）+ 类型状态 = 状态相关清理保证
错误层次结构 + 验证边界 = 带穷尽处理的类型化解析错误
#[must_use] + 单次使用类型 = 难以忽略、难以重用的令牌
```

### 应避免的反模式

| 反模式 | 为什么错误 | 正确替代方案 |
|-------------|---------------|-------------------|
| `fn read_sensor() -> f64` | 无单位 — 可能是 °C、°F 或 RPM | `fn read_sensor() -> Celsius` |
| `fn encrypt(nonce: &[u8; 12])` | Nonce 可以重用（借用） | `fn encrypt(nonce: Nonce)`（移动） |
| `fn admin_op(is_admin: bool)` | 调用者可以撒谎（`true`） | `fn admin_op(_: &AdminToken)` |
| `fn send(session: &Session)` | 无状态保证 | `fn send(session: &Session<Active>)` |
| `fn process(data: &[u8])` | 未验证 | `fn process(data: &ValidFru)` |
| `Clone` 对短暂密钥 | 破坏单次使用保证 | 不要派生 Clone |
| `let vendor_id: u16 = 0xFFFF` | 哨兵值在内部携带 | `let vendor_id: Option<u16> = None` |
| `fn route(level: &str)` 带回退 | 拼写错误静默默认 | `let level: DiagLevel = s.parse()?` |
| `Builder::new().finish()` 没有字段 | 构造不完整对象 | 类型状态构建器：`finish()` 受 `Set` 门控 |
| `let buf: Vec<u8>` 用于固定大小硬件缓冲区 | 大小仅在运行时检查 | `RegisterBank<4096>`（const 泛型） |
| 散布的原始 `unsafe { ptr::read(...) }` | UB 风险、不可审计 | `MmioRegion::read_u32()` 安全包装器 |
| `async fn transition(&mut self)` | 可变借用不强制状态 | `async fn transition(self) -> NextState` |
| `fn cleanup()` 手动调用 | 在提前返回/panic 时被遗忘 | `impl Drop` — 编译器插入调用 |
| `fn op() -> Result<T, String>` | 不透明错误，无变体匹配 | `fn op() -> Result<T, DiagError>` 枚举 |

### 映射到诊断代码库

| 模块 | 适用模式 |
|---------------------|----------------------|
| `protocol_lib` | 类型化命令，类型状态会话 |
| `thermal_diag` | 能力混合，量纲类型 |
| `accel_diag` | 验证边界，幽灵寄存器 |
| `network_diag` | 类型状态（链路训练），能力令牌 |
| `pci_topology` | 幽灵类型（寄存器宽度），验证配置，哨兵值 → Option |
| `event_handler` | 单次使用审计令牌，能力令牌，FromStr（Component） |
| `event_log` | 验证边界（SEl 记录解析） |
| `compute_diag` | 量纲类型（温度、频率） |
| `memory_diag` | 验证边界（SPD 数据），量纲类型 |
| `switch_diag` | 类型状态（端口枚举），幽灵类型 |
| `config_loader` | FromStr（DiagLevel、FaultStatus、DiagAction） |
| `log_analyzer` | 验证边界（CompiledPatterns） |
| `diag_framework` | 类型状态构建器（DerBuilder），会话类型（orchestrator↔worker） |
| `topology_lib` | Const 泛型寄存器组，安全的 MMIO 包装器 |

### Curry-Howard 速查表

| 逻辑概念 | Rust 等价 | 示例 |
|--------------|----------------|---------|
| 命题 | 类型 | `AdminToken` |
| 证明 | 该类型的值 | `let tok = authenticate()?;` |
| 隐含（A → B） | 函数 `fn(A) -> B` | `fn activate(AdminToken) -> Session<Active>` |
| 合取（A ∧ B） | 元组 `(A, B)` 或多参数 | `fn op(a: &AdminToken, b: &LinkTrained)` |
| 析取（A ∨ B） | `enum { A(A), B(B) }` 或 `Result<A, B>` | `Result<Session<Active>, Error>` |
| 真 | `()`（单元类型） | 始终可构造 |
| 假 | `!`（never 类型）或 `enum Void {}` | 永远不能构造 |

---
