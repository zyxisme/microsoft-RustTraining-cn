# 来自实战的十四个技巧 🟡

> **你将学到：** 十四个较小的正确性构造技术——从哨兵值消除、密封 trait 到会话类型、`Pin`、RAII 和 `#[must_use]`——每个都能以几乎零成本消除特定的 bug 类别。
>
> **交叉引用：** [ch02](ch02-typed-command-interfaces-request-determi.md)（密封 trait 扩展 ch02），[ch05](ch05-protocol-state-machines-type-state-for-r.md)（类型状态构建器扩展 ch05），[ch07](ch07-validated-boundaries-parse-dont-validate.md)（FromStr 扩展 ch07）

## 来自实战的十四个技巧

八个核心模式（ch02–ch09）涵盖了主要的正确性构造技术。本章收集了十四个 **较小但高价值的技巧**，它们在生产 Rust 代码中反复出现——每个都能以零或几乎零成本消除特定的 bug 类别。

### 技巧1 — 边界处的哨兵值 → `Option`

硬件协议充满了哨兵值：IPMI 使用 `0xFF` 表示"传感器不存在"，PCI 使用 `0xFFFF` 表示"无设备"，SMBIOS 使用 `0x00` 表示"未知"。如果你将这些哨兵值作为普通整数带入代码，每个使用者都必须记得检查这个魔法值。如果忘记一个比较，你会得到一个虚假的 255°C 读数或一个伪造的供应商 ID 匹配。

**规则：** 在最开始的解析边界将哨兵值转换为 `Option`，仅在序列化边界转换回哨兵值。

#### 反模式（来自 `pcie_tree/src/lspci.rs`）

```rust,ignore
// 哨兵值在内部携带 — 每个比较都必须记住
let mut current_vendor_id: u16 = 0xFFFF;
let mut current_device_id: u16 = 0xFFFF;

// ... 稍后，解析静默失败 ...
current_vendor_id = u16::from_str_radix(hex, 16)
    .unwrap_or(0xFFFF);  // 哨兵值隐藏了错误
```

每个接收 `current_vendor_id` 的函数都必须知道 `0xFFFF` 是特殊的。如果有人在没有先检查 `0xFFFF` 的情况下写 `if vendor_id == target_id`，当目标恰好也从坏输入解析为 `0xFFFF` 时，缺失的设备会静默匹配。

#### 正确的模式（来自 `nic_sel/src/events.rs`）

```rust,ignore
pub struct ThermalEvent {
    pub record_id: u16,
    pub temperature: Option<u8>,  // 如果传感器报告 0xFF 则为 None
}

impl ThermalEvent {
    pub fn from_raw(record_id: u16, raw_temp: u8) -> Self {
        ThermalEvent {
            record_id,
            temperature: if raw_temp != 0xFF {
                Some(raw_temp)
            } else {
                None
            },
        }
    }
}
```

现在每个使用者 *必须* 处理 `None` 情况——编译器强制它：

```rust,ignore
// 安全 — 编译器确保我们处理缺失的温度
fn is_overtemp(temp: Option<u8>, threshold: u8) -> bool {
    temp.map_or(false, |t| t > threshold)
}

// 忘记处理 None 是编译错误：
// fn bad_check(temp: Option<u8>, threshold: u8) -> bool {
//     temp > threshold  // 错误：不能将 Option<u8> 与 u8 比较
// }
```

#### 实际影响

`inventory/src/events.rs` 对 GPU 热警报使用相同的模式：
```rust,ignore
temperature: if data[1] != 0xFF {
    Some(data[1] as i8)
} else {
    None
},
```

`pcie_tree/src/lspci.rs` 的重构很简单：将 `current_vendor_id: u16` 改为 `current_vendor_id: Option<u16>`，将 `0xFFFF` 替换为 `None`，然后让编译器找到每个需要更新的地方。

| 之前 | 之后 |
|--------|-------|
| `let mut vendor_id: u16 = 0xFFFF` | `let mut vendor_id: Option<u16> = None` |
| `.unwrap_or(0xFFFF)` | `.ok()`（已经返回 `Option`） |
| `if vendor_id != 0xFFFF { ... }` | `if let Some(vid) = vendor_id { ... }` |
| 序列化：`vendor_id` | `vendor_id.unwrap_or(0xFFFF)` |

***

### 技巧2 — 密封 Trait

第2章介绍了带有关联类型的 `IpmiCmd`，将每个命令绑定到其响应。但有一个漏洞： 如果 *任何* 代码可以实现 `IpmiCmd`，有人可以写一个 `MaliciousCmd`，其 `parse_response` 返回错误的类型或 panic。整个系统的类型安全性建立在每个实现都是正确的基础上。

**密封 trait** 关闭了这个漏洞。想法很简单：让 trait 需要一个 *私有的* 超 trait，只有你的 crate 可以实现它。

```rust,ignore
// — 私有模块：从 crate 导出 —
mod private {
    pub trait Sealed {}
}

// — 公共 trait：需要 Sealed，外部人员无法实现 —
pub trait IpmiCmd: private::Sealed {
    type Response;
    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}
```

在 crate 内部，你为每个批准的命令类型实现 `Sealed`：

```rust,ignore
pub struct ReadTemp { pub sensor_id: u8 }
impl private::Sealed for ReadTemp {}

impl IpmiCmd for ReadTemp {
    type Response = Celsius;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        if raw.is_empty() { return Err(io::Error::new(io::ErrorKind::InvalidData, "empty")); }
        Ok(Celsius(raw[0] as f64))
    }
}
```

外部代码看到 `IpmiCmd` 并可以调用 `execute()`，但不能实现它：

```rust,ignore
// 在另一个 crate 中：
struct EvilCmd;
// impl private::Sealed for EvilCmd {}  // 错误：模块 `private` 是私有的
// impl IpmiCmd for EvilCmd { ... }     // 错误：`Sealed` 未满足
```

#### 何时密封

| 密封当… | 不要密封当… |
|-----------|-----------------|
| 安全性取决于正确实现（IpmiCmd、DiagModule） | 用户应该扩展系统（自定义报告格式化器） |
| 关联类型必须满足不变量 | trait 是一个简单的能力标记（HasIpmi） |
| 你拥有规范的实现集 | 第三方插件是设计目标 |

#### 实际候选者

- `IpmiCmd` — 错误的解析可能损坏类型化响应
- `DiagModule` — 框架假设 `run()` 返回有效的 DER 记录
- `SelEventFilter` — 坏的过滤器可能吞掉关键 SEL 事件

***

### 技巧3 — `#[non_exhaustive]` 用于演进的枚举

今天 `inventory/src/types.rs` 中的 `SkuVariant` 有五个变体：

```rust,ignore
pub enum SkuVariant {
    S1001, S2001, S2002, S2003, S3001,
}
```

当下一代产品发布并添加 `S4001` 时，任何外部代码如果匹配 `SkuVariant` 且没有通配符 arm，将会 **静默编译失败**——这正是重点。但内部代码呢？没有 `#[non_exhaustive]`，你在 *同一个 crate* 中的 `match` 在没有通配符的情况下编译，添加新变体会破坏你自己的构建。

标记枚举为 `#[non_exhaustive]` 强制 **外部 crate** 在匹配它时包含通配符 arm。在定义的 crate 内部，`#[non_exhaustive]` 没有效果——你仍然可以写穷尽匹配。

**为什么这有用：** 当你从库 crate 发布 `SkuVariant`（或工作空间中的共享子 crate）时，下游代码被迫处理未知的未来变体。当下一代添加 `S4001` 时，下游代码已经编译——它们有通配符 arm。

```rust,ignore
// 在 gpu_sel crate（定义 crate）中：
#[non_exhaustive]
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum SkuVariant {
    S1001,
    S2001,
    S2002,
    S2003,
    S3001,
    // 当下一代 SKU 发布时，在这里添加。
    // 外部使用者已经有通配符 arm — 对它们零破坏。
}

// 在 gpu_sel 内部 — 允许穷尽匹配（不需要通配符）：
fn diag_path_internal(sku: SkuVariant) -> &'static str {
    match sku {
        SkuVariant::S1001 => "legacy_gen1",
        SkuVariant::S2001 => "gen2_accel_diag",
        SkuVariant::S2002 => "gen2_alt_diag",
        SkuVariant::S2003 => "gen2_alt_hf_diag",
        SkuVariant::S3001 => "gen3_accel_diag",
        // 在定义 crate 内部不需要通配符。
        // 在这里添加 S4001 将导致此 match 的编译错误，
        // 这正是你想要的 — 它强制你更新它。
    }
}
```

```rust,ignore
// 在二进制 crate 中（依赖于 inventory 的下游 crate）：
fn diag_path_external(sku: inventory::SkuVariant) -> &'static str {
    match sku {
        inventory::SkuVariant::S1001 => "legacy_gen1",
        inventory::SkuVariant::S2001 => "gen2_accel_diag",
        inventory::SkuVariant::S2002 => "gen2_alt_diag",
        inventory::SkuVariant::S2003 => "gen2_alt_hf_diag",
        inventory::SkuVariant::S3001 => "gen3_accel_diag",
        _ => "generic_diag",  // 外部 crate 需要 #[non_exhaustive]
    }
}
```

> **工作空间提示：** 如果你所有代码都在单个 crate 中，`#[non_exhaustive]` 不会帮助——它只影响跨 crate 边界。对于项目的大型工作空间，将演进的枚举放在共享 crate（`core_lib` 或 `inventory`）中，这样属性就能保护其他工作空间 crate 中的使用者。

#### 候选者

| 枚举 | 模块 | 为什么 |
|------|--------|-----|
| `SkuVariant` | `inventory`, `net_inventory` | 每代新 SKU |
| `SensorType` | `protocol_lib` | IPMI 规范为 OEM 保留 0xC0–0xFF |
| `CompletionCode` | `protocol_lib` | 自定义 BMC 供应商添加代码 |
| `Component` | `event_handler` | 新硬件类别（最近添加了 NewSoC） |

***

### 技巧4 — 类型状态构建器

第5章展示了 *协议* 的类型状态（会话生命周期、链路训练）。同样的想法适用于 *构建器*——其 `build()` / `finish()` 只能在所有必填字段都设置后才能调用的结构体。

#### 流畅构建器的问题

`diag_framework/src/der.rs` 中的 `DerBuilder` 今天看起来像这样（简化）：

```rust,ignore
// 当前流畅构建器 — finish() 始终可用
pub struct DerBuilder {
    der: Der,
}

impl DerBuilder {
    pub fn new(marker: &str, fault_code: u32) -> Self { ... }
    pub fn mnemonic(mut self, m: &str) -> Self { ... }
    pub fn fault_class(mut self, fc: &str) -> Self { ... }
    pub fn finish(self) -> Der { self.der }  // ← 始终可调用！
}
```

这编译没有错误，但产生一个不完整的 DER 记录：

```rust,ignore
let bad = DerBuilder::new("CSI_ERR", 62691)
    .finish();  // 哎呀 — 没有 mnemonic，没有 fault_class
```

#### 类型状态构建器：`finish()` 需要两个字段

```rust,ignore
pub struct Missing;
pub struct Set<T>(T);

pub struct DerBuilder<Mnemonic, FaultClass> {
    marker: String,
    fault_code: u32,
    mnemonic: Mnemonic,
    fault_class: FaultClass,
    description: Option<String>,
}

// 构造函数：两个必填字段都从 Missing 开始
impl DerBuilder<Missing, Missing> {
    pub fn new(marker: &str, fault_code: u32) -> Self {
        DerBuilder {
            marker: marker.to_string(),
            fault_code,
            mnemonic: Missing,
            fault_class: Missing,
            description: None,
        }
    }
}

// 设置 mnemonic（不管 fault_class 的状态）
impl<FC> DerBuilder<Missing, FC> {
    pub fn mnemonic(self, m: &str) -> DerBuilder<Set<String>, FC> {
        DerBuilder {
            marker: self.marker, fault_code: self.fault_code,
            mnemonic: Set(m.to_string()),
            fault_class: self.fault_class,
            description: self.description,
        }
    }
}

// 设置 fault_class（不管 mnemonic 的状态）
impl<MN> DerBuilder<MN, Missing> {
    pub fn fault_class(self, fc: &str) -> DerBuilder<MN, Set<String>> {
        DerBuilder {
            marker: self.marker, fault_code: self.fault_code,
            mnemonic: self.mnemonic,
            fault_class: Set(fc.to_string()),
            description: self.description,
        }
    }
}

// 可选字段 — 在任何状态下都可用
impl<MN, FC> DerBuilder<MN, FC> {
    pub fn description(mut self, desc: &str) -> Self {
        self.description = Some(desc.to_string());
        self
    }
}

/// 完全构建的 DER 记录。
pub struct Der {
    pub marker: String,
    pub fault_code: u32,
    pub mnemonic: String,
    pub fault_class: String,
    pub description: Option<String>,
}

// finish() 仅在两个必填字段都是 Set 时可用
impl DerBuilder<Set<String>, Set<String>> {
    pub fn finish(self) -> Der {
        Der {
            marker: self.marker,
            fault_code: self.fault_code,
            mnemonic: self.mnemonic.0,
            fault_class: self.fault_class.0,
            description: self.description,
        }
    }
}
```

现在有问题的调用是编译错误：

```rust,ignore
// ✅ 编译 — 两个必填字段都设置了（任意顺序）
let der = DerBuilder::new("CSI_ERR", 62691)
    .fault_class("GPU Module")   // 顺序不重要
    .mnemonic("ACCEL_CARD_ER691")
    .description("Thermal throttle")
    .finish();

// ❌ 编译错误 — finish() 在 DerBuilder<Set<String>, Missing> 上不存在
let bad = DerBuilder::new("CSI_ERR", 62691)
    .mnemonic("ACCEL_CARD_ER691")
    .finish();  // 错误：找不到方法 `finish`
```

#### 何时使用类型状态构建器

| 使用当… | 不要麻烦当… |
|-----------|-------------------|
| 省略字段导致静默 bug（DER 缺失 mnemonic） | 所有字段都有合理的默认值 |
| 构建器是公共 API 的一部分 | 构建器是仅用于测试的脚手架 |
| 超过 2-3 个必填字段 | 单个必填字段（就在 `new()` 中获取） |

***

### 技巧5 — `FromStr` 作为验证边界

第7章展示了用于二进制数据（FRU 记录、SEL 条目）的 `TryFrom<&[u8]>`。对于 **字符串** 输入——配置文件、CLI 参数、JSON 字段——类似的边界是 `FromStr`。

#### 问题

```rust,ignore
// C++ / 未验证的 Rust：静默退回到默认值
fn route_diag(level: &str) -> DiagMode {
    if level == "quick" { ... }
    else if level == "standard" { ... }
    else { QuickMode }  // 配置中的拼写错误？  ¯\_(ツ)_/¯
}
```

带有 `"diag_level": "extendedd"`（拼写错误）的配置文件会静默获得 `QuickMode`。

#### 模式（来自 `config_loader/src/diag.rs`）

```rust,ignore
use std::str::FromStr;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum DiagLevel {
    Quick,
    Standard,
    Extended,
    Stress,
}

impl FromStr for DiagLevel {
    type Err = String;
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        match s.to_lowercase().as_str() {
            "quick"    | "1" => Ok(DiagLevel::Quick),
            "standard" | "2" => Ok(DiagLevel::Standard),
            "extended" | "3" => Ok(DiagLevel::Extended),
            "stress"   | "4" => Ok(DiagLevel::Stress),
            other => Err(format!("unknown diag level: '{other}'")),
        }
    }
}
```

现在拼写错误会立即被捕获：

```rust,ignore
let level: DiagLevel = "extendedd".parse()?;
// Err("unknown diag level: 'extendedd'")
```

#### 三个好处

1. **快速失败：** 坏输入在解析边界被捕获，而不是在诊断逻辑深处三层。
2. **别名是显式的：** `"MEM"`、`"DIMM"` 和 `"MEMORY"` 都映射到 `Component::Memory`——match 分支记录了映射。
3. **`.parse()` 人体工学：** 因为 `FromStr` 与 `str::parse()` 集成，你得到干净的单行代码：`let level: DiagLevel = config["level"].parse()?;`

#### 实际代码库使用

项目已有 8 个 `FromStr` 实现：

| 类型 | 模块 | 值得注意的别名 |
|------|--------|----------------|
| `DiagLevel` | `config_loader` | `"1"` = Quick, `"4"` = Stress |
| `Component` | `event_handler` | `"MEM"` / `"DIMM"` = Memory, `"SSD"` / `"NVME"` = Disk |
| `SkuVariant` | `net_inventory` | `"Accel-X1"` = S2001, `"Accel-M1"` = S2002, `"Accel-Z1"` = S3001 |
| `SkuVariant` | `inventory` | 相同别名（独立模块，相同模式） |
| `FaultStatus` | `config_loader` | 故障生命周期状态 |
| `DiagAction` | `config_loader` | 修复操作类型 |
| `ActionType` | `config_loader` | 操作类别 |
| `DiagMode` | `cluster_diag` | 多节点测试模式 |

与 `TryFrom` 的对比：

| | `TryFrom<&[u8]>` | `FromStr` |
|---|---|---|
| 输入 | 原始字节（二进制协议） | 字符串（配置、CLI、JSON） |
| 典型来源 | IPMI、PCIe 配置空间、FRU | JSON 字段、环境变量、用户输入 |
| 章节 | ch07 | ch11 |
| 都使用 | `Result` — 强制调用者处理无效输入 | |

***

### 技巧6 — Const 泛型用于编译时大小验证

当硬件缓冲区、寄存器组或协议帧具有固定大小时，const 泛型让编译器强制执行它们：

```rust,ignore
/// 固定大小的寄存器组。大小是类型的一部分。
/// `RegisterBank<256>` 和 `RegisterBank<4096>` 是不同类型。
pub struct RegisterBank<const N: usize> {
    data: [u8; N],
}

impl<const N: usize> RegisterBank<N> {
    /// 读取给定偏移处的寄存器。
    /// 编译时：N 是已知的，所以数组大小是固定的。
    /// 运行时：只检查偏移量。
    pub fn read(&self, offset: usize) -> Option<u8> {
        self.data.get(offset).copied()
    }
}

// PCIe 传统配置空间：256 字节
type PciConfigSpace = RegisterBank<256>;

// PCIe 扩展配置空间：4096 字节
type PcieExtConfigSpace = RegisterBank<4096>;

// 这些是不同类型 — 不能意外地将一个传递给另一个：
fn read_extended_cap(config: &PcieExtConfigSpace, offset: usize) -> Option<u8> {
    config.read(offset)
}
// read_extended_cap(&pci_config, 0x100);
//                   ^^^^^^^^^^^ expected RegisterBank<4096>, found RegisterBank<256> ❌
```

**使用 const 泛型的编译时断言：**

```rust,ignore
/// NVMe 管理命令使用 4096 字节缓冲区。在编译时强制执行。
pub struct NvmeBuffer<const N: usize> {
    data: Box<[u8; N]>,
}

impl<const N: usize> NvmeBuffer<N> {
    pub fn new() -> Self {
        // 运行时断言：只允许 512 或 4096
        assert!(N == 4096 || N == 512, "NVMe buffers must be 512 or 4096 bytes");
        NvmeBuffer { data: Box::new([0u8; N]) }
    }
}
// NvmeBuffer::<1024>::new();  // 使用此形式会在运行时 panic
// 对于真正的编译时强制执行，参见技巧 9（const 断言）。
```

> **何时使用：** 固定大小的协议缓冲区（NVMe、PCIe 配置空间）、DMA 描述符、硬件 FIFO 深度。任何大小是硬件常量、在运行时不应改变的地方。

***

### 技巧7 — `unsafe` 周围的安全包装器

项目目前零 `unsafe` 块。但当你添加 MMIO 寄存器访问、DMA 或 FFI 到 accel-mgmt/accel-query 时，你需要 `unsafe`。正确性构造方法：**将每个 `unsafe` 块包装在安全抽象中**，这样不安全就被包含且可审计。

```rust,ignore
/// MMIO 映射的寄存器。指针在映射的生命周期内有效。
/// 所有 unsafe 都包含在这个模块中 — 调用者使用安全方法。
pub struct MmioRegion {
    base: *mut u8,
    len: usize,
}

impl MmioRegion {
    /// # Safety
    /// - `base` 必须是 MMIO 映射区域的有效指针
    /// - 区域必须在该结构体的生命周期内保持映射
    /// - 其他代码不能别名此区域
    pub unsafe fn new(base: *mut u8, len: usize) -> Self {
        MmioRegion { base, len }
    }

    /// 安全读取 — 边界检查防止越界 MMIO 访问。
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        if offset + 4 > self.len { return None; }
        // SAFETY: 偏移量在上方边界检查，base 根据 new() 契约有效
        Some(unsafe {
            core::ptr::read_volatile(self.base.add(offset) as *const u32)
        })
    }

    /// 安全写入 — 边界检查防止越界 MMIO 访问。
    pub fn write_u32(&self, offset: usize, value: u32) -> bool {
        if offset + 4 > self.len { return false; }
        // SAFETY: 偏移量在上方边界检查，base 根据 new() 契约有效
        unsafe {
            core::ptr::write_volatile(self.base.add(offset) as *mut u32, value);
        }
        true
    }
}
```

**结合幽灵类型（ch09）用于类型化 MMIO：**

```rust,ignore
use std::marker::PhantomData;

pub struct ReadOnly;
pub struct ReadWrite;

pub struct TypedMmio<Perm> {
    region: MmioRegion,
    _perm: PhantomData<Perm>,
}

impl TypedMmio<ReadOnly> {
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        self.region.read_u32(offset)
    }
    // 没有写方法 — 如果尝试写入 ReadOnly 区域则是编译错误
}

impl TypedMmio<ReadWrite> {
    pub fn read_u32(&self, offset: usize) -> Option<u32> {
        self.region.read_u32(offset)
    }
    pub fn write_u32(&self, offset: usize, value: u32) -> bool {
        self.region.write_u32(offset, value)
    }
}
```

> **`unsafe` 包装器指南：**
>
> | 规则 | 为什么 |
> |------|-----|
> | 一个带文档化 `# Safety` 不变量的 `unsafe fn new()` | 调用者一次承担责任 |
> | 所有其他方法都是安全的 | 调用者不能触发 UB |
> | 每个 `unsafe` 块上的 `# SAFETY:` 注释 | 审计员可以本地验证 |
> | 用 `#[deny(unsafe_op_in_unsafe_fn)]` 包装在模块中 | 即使在 `unsafe fn` 内部，每个操作也需要 `unsafe` |
> | 在包装器上运行 `cargo +nightly miri test` | 验证内存模型合规性 |

---

### ✅ 检查点：技巧 1–7

你现在有七个日常技巧。快速记分卡：

| 技巧 | 消除的 bug 类别 | 采用工作量 |
|:-----:|----------------------|:---------------:|
| 1 | 哨兵值混淆（0xFF） | 低 — 边界处一个 `match` |
| 2 | 未授权的 trait 实现 | 低 — 添加 `Sealed` 超 trait |
| 3 | 枚举增长后破坏使用者 | 低 — 一行属性 |
| 4 | 缺失构建器字段 | 中 — 额外的类型参数 |
| 5 | 字符串类型配置中的拼写错误 | 低 — `impl FromStr` |
| 6 | 错误的缓冲区大小 | 低 — const 泛型参数 |
| 7 | unsafe 分散在代码库中 | 中 — 包装器模块 |

技巧 8–14 **更高级** — 它们涉及 async、const 求值、会话类型、`Pin` 和 `Drop`。如果需要可以在这里休息一下；上面的技术已经是高价值、低工作量的胜利，你可以明天采用。

***

### 技巧8 — 异步类型状态机

当硬件驱动使用 `async`（例如，async BMC 通信、async NVMe I/O）时，类型状态仍然有效——但在 `.await` 点之间的所有权需要小心：

```rust,ignore
use std::marker::PhantomData;

pub struct Idle;
pub struct Authenticating;
pub struct Active;

pub struct AsyncSession<S> {
    host: String,
    _state: PhantomData<S>,
}

impl AsyncSession<Idle> {
    pub fn new(host: &str) -> Self {
        AsyncSession { host: host.to_string(), _state: PhantomData }
    }

    /// 转换 Idle → Authenticating → Active。
    /// Session 被消耗（移动到 future）跨过 .await。
    pub async fn authenticate(self, user: &str, pass: &str)
        -> Result<AsyncSession<Active>, String>
    {
        // 阶段 1：发送凭证（消耗 Idle session）
        let pending: AsyncSession<Authenticating> = AsyncSession {
            host: self.host,
            _state: PhantomData,
        };

        // 模拟 async BMC 认证
        // tokio::time::sleep(Duration::from_secs(1)).await;

        // 阶段 2：返回 Active session
        Ok(AsyncSession {
            host: pending.host,
            _state: PhantomData,
        })
    }
}

impl AsyncSession<Active> {
    pub async fn send_command(&mut self, cmd: &[u8]) -> Vec<u8> {
        // 这里的 async I/O...
        vec![0x00]
    }
}

// 用法：
// let session = AsyncSession::new("192.168.1.100");
// let mut session = session.authenticate("admin", "pass").await?;
// let resp = session.send_command(&[0x04, 0x2D]).await;
```

**async 类型状态的关键规则：**

| 规则 | 为什么 |
|------|-----|
| 转换方法获取 `self`（按值），不是 `&mut self` | 所有权转移跨 `.await` 工作 |
| 对于可恢复错误返回 `Result<NextState, (Error, PrevState)>` | 调用者可以从上一个状态重试 |
| 不要跨多个 future 拆分状态 | 一个 future 拥有一个 session |
| 如果使用 tokio::spawn，使用 `Send + 'static` 边界 | session 必须可以跨线程移动 |

> **警告：** 如果你需要在错误时获取 *上一个* 状态（重试），返回 `Result<AsyncSession<Active>, (Error, AsyncSession<Idle>)>` 这样调用者获取所有权。没有这个，失败的 `.await` 会永久丢弃 session。

***

### 技巧9 — 通过 Const 断言的精化类型

当数值约束是编译时不变量（不是运行时数据）时，使用 `const` 求值来强制执行。这与技巧 6（提供类型级大小区分）不同——这里我们在编译时 *拒绝无效值*：

```rust,ignore
/// 必须在 IPMI SDR 范围内（0x01..=0xFE）的传感器 ID。
/// 当 N 是 const 时，约束在编译时检查。
pub struct SdrSensorId<const N: u8>;

impl<const N: u8> SdrSensorId<N> {
    /// 编译时验证：如果 N 超出范围，在编译期间 panic。
    pub const fn validate() {
        assert!(N >= 0x01, "Sensor ID must be >= 0x01");
        assert!(N <= 0xFE, "Sensor ID must be <= 0xFE (0xFF is reserved)");
    }

    pub const VALIDATED: () = Self::validate();

    pub const fn value() -> u8 { N }
}

// 用法：
fn read_sensor_const<const N: u8>() -> f64 {
    let _ = SdrSensorId::<N>::VALIDATED;  // 编译时检查
    // 读取传感器 N...
    42.0
}

// read_sensor_const::<0x20>();   // ✅ 编译 — 0x20 是有效的
// read_sensor_const::<0x00>();   // ❌ 编译错误 — "Sensor ID must be >= 0x01"
// read_sensor_const::<0xFF>();   // ❌ 编译错误 — 0xFF 是保留的
```

**更简单的形式 — 有界风扇 ID：**

```rust,ignore
pub struct BoundedFanId<const N: u8>;

impl<const N: u8> BoundedFanId<N> {
    pub const VALIDATED: () = assert!(N < 8, "Server has at most 8 fans (0..7)");

    pub const fn id() -> u8 {
        let _ = Self::VALIDATED;
        N
    }
}

// BoundedFanId::<3>::id();   // ✅
// BoundedFanId::<10>::id();  // ❌ 编译错误
```

> **何时使用：** 编译时已知的硬件定义固定 ID（传感器 ID、风扇槽位、PCIe 槽号）。当值来自运行时数据（配置文件、用户输入）时，使用 `TryFrom` / `FromStr`（ch07、技巧 5）代替。

***

### 技巧10 — 用于通道通信的会话类型

当两个组件通过通道通信时（例如，诊断 orchestrator ↔ 工作线程），**会话类型** 在类型系统中编码协议：

```rust,ignore
use std::marker::PhantomData;

// 协议：客户端发送 Request，服务器发送 Response，然后完成。
pub struct SendRequest;
pub struct RecvResponse;
pub struct Done;

/// 类型化通道端点。`S` 是当前协议状态。
pub struct Chan<S> {
    // 真实代码：包装 mpsc::Sender/Receiver 对
    _state: PhantomData<S>,
}

impl Chan<SendRequest> {
    /// 发送请求 — 转换到 RecvResponse 状态。
    pub fn send(self, request: DiagRequest) -> Chan<RecvResponse> {
        // ... 在通道上发送 ...
        Chan { _state: PhantomData }
    }
}

impl Chan<RecvResponse> {
    /// 接收响应 — 转换到 Done 状态。
    pub fn recv(self) -> (DiagResponse, Chan<Done>) {
        // ... 从通道接收 ...
        (DiagResponse { passed: true }, Chan { _state: PhantomData })
    }
}

impl Chan<Done> {
    /// 关闭通道 — 仅在协议完成时才可能。
    pub fn close(self) { /* drop */ }
}

pub struct DiagRequest { pub test_name: String }
pub struct DiagResponse { pub passed: bool }

// 协议必须按顺序遵循：
fn orchestrator(chan: Chan<SendRequest>) {
    let chan = chan.send(DiagRequest { test_name: "gpu_stress".into() });
    let (response, chan) = chan.recv();
    chan.close();
    println!("Result: {}", if response.passed { "PASS" } else { "FAIL" });
}

// 不能在发送前接收：
// fn wrong_order(chan: Chan<SendRequest>) {
//     chan.recv();  // ❌ Chan<SendRequest> 上没有方法 `recv`
// }
```

> **何时使用：** 线程间诊断协议、BMC 命令序列、任何顺序重要的请求-响应模式。对于复杂的多消息协议，考虑 [`session-types`](https://crates.io/crates/session-types) 或 [`rumpsteak`](https://crates.io/crates/rumpsteak) crate。

***

### 技巧11 — `Pin` 用于自引用状态机

某些类型状态机需要持有指向自身数据的引用（例如，在其拥有的缓冲区中跟踪位置的解析器）。Rust 通常禁止这样做，因为移动结构体会使内部指针无效。`Pin<T>` 通过保证值 **不会被移动** 来解决这个问题：

```rust,ignore
use std::pin::Pin;
use std::marker::PhantomPinned;

/// 流式解析器，持有指向自身缓冲区的引用。
/// 一旦固定，就不能移动 — 内部引用保持有效。
pub struct StreamParser {
    buffer: Vec<u8>,
    /// 指向 `buffer`。仅在固定时有效。
    cursor: *const u8,
    _pin: PhantomPinned,  // 选择退出 Unpin — 防止意外取消固定
}

impl StreamParser {
    pub fn new(data: Vec<u8>) -> Pin<Box<Self>> {
        let parser = StreamParser {
            buffer: data,
            cursor: std::ptr::null(),
            _pin: PhantomPinned,
        };
        let mut boxed = Box::pin(parser);

        // 设置 cursor 指向固定缓冲区
        let cursor = boxed.buffer.as_ptr();
        // SAFETY：我们有独占访问权且解析器已固定
        unsafe {
            let mut_ref = Pin::as_mut(&mut boxed);
            Pin::get_unchecked_mut(mut_ref).cursor = cursor;
        }

        boxed
    }

    /// 读取下一个字节 — 仅可通过 Pin<&mut Self> 调用。
    pub fn next_byte(self: Pin<&mut Self>) -> Option<u8> {
        // 解析器不能被移动，所以 cursor 保持有效
        if self.cursor.is_null() { return None; }
        // ... 通过缓冲区推进 cursor ...
        Some(42) // stub
    }
}

// 用法：
// let mut parser = StreamParser::new(vec![0x01, 0x02, 0x03]);
// let byte = parser.as_mut().next_byte();
```

**关键见解：** `Pin` 是自引用结构体问题的正确性构造解决方案。没有它，你需要 `unsafe` 和手动生命周期跟踪。有了它，编译器防止移动，内部指针不变量被维护。

| 使用 `Pin` 当… | 不要使用 `Pin` 当… |
|-----------------|----------------------|
| 状态机持有结构内引用 | 所有字段独立拥有 |
| 跨 `.await` 借用的 async future | 不需要自引用 |
| 不能在内存中重新定位的 DMA 描述符 | 数据可以自由移动 |
| 具有内部游标的硬件环形缓冲区 | 基于简单索引的迭代工作 |

***

### 技巧12 — RAII / `Drop` 作为正确性保证

Rust 的 `Drop` trait 是一个正确性构造机制：清理代码 **不能被遗忘**，因为编译器自动插入它。这对于必须精确释放一次的硬件资源特别有价值。

```rust,ignore
use std::io;

/// 一个 IPMI 会话，完成时必须关闭。
/// `Drop` 实现保证即使在 panic 或提前 `?` 返回时也会清理。
pub struct IpmiSession {
    handle: u32,
}

impl IpmiSession {
    pub fn open(host: &str) -> io::Result<Self> {
        // ... 协商 IPMI 会话 ...
        Ok(IpmiSession { handle: 42 })
    }

    pub fn send_raw(&self, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![0x00])
    }
}

impl Drop for IpmiSession {
    fn drop(&mut self) {
        // 关闭会话命令：始终运行，即使在 panic/提前返回时。
        // 在 C 中，忘记 CloseSession() 会泄漏 BMC 会话槽。
        let _ = self.send_raw(&[0x06, 0x3C]);
        eprintln!("[RAII] session {} closed", self.handle);
    }
}
// 用法：
fn diagnose(host: &str) -> io::Result<()> {
    let session = IpmiSession::open(host)?;
    session.send_raw(&[0x04, 0x2D, 0x20])?;
    // 不需要显式关闭 — Drop 在此自动运行
    Ok(())
    // 即使 send_raw 返回 Err(...)，会话仍然关闭。
}
```

**RAII 消除的 C/C++ 失败模式：**

```text
C:     session = ipmi_open(host);
       ipmi_send(session, data);
       if (error) return -1;        // 🐛 泄漏的会话 — 忘记 close()
       ipmi_close(session);

Rust:  let session = IpmiSession::open(host)?;
       session.send_raw(data)?;     // ✅ Drop 在 ? 返回时运行
       // Drop 始终运行 — 泄漏不可能
```

**将 RAII 与类型状态（ch05）结合用于有序清理：**

你不能专门化泛型参数上的 `Drop`（Rust 错误 E0366）。相反，使用 **每个状态单独的包装类型**：

```rust,ignore
use std::marker::PhantomData;

pub struct Open;
pub struct Locked;

pub struct GpuContext<S> {
    device_id: u32,
    _state: PhantomData<S>,
}

impl GpuContext<Open> {
    pub fn lock_clocks(self) -> LockedGpu {
        // ... 锁定 GPU 时钟以进行稳定基准测试 ...
        LockedGpu { device_id: self.device_id }
    }
}

/// 锁定状态的单独类型 — 有自己的 Drop。
/// 我们不能做 `impl Drop for GpuContext<Locked>`（E0366），
/// 所以我们使用拥有锁定资源的不同包装器。
pub struct LockedGpu {
    device_id: u32,
}

impl LockedGpu {
    pub fn run_benchmark(&self) -> f64 {
        // ... 使用锁定的时钟进行基准测试 ...
        42.0
    }
}

impl Drop for LockedGpu {
    fn drop(&mut self) {
        // 在 drop 时解锁时钟 — 仅对锁定包装器触发。
        eprintln!("[RAII] GPU {} clocks unlocked", self.device_id);
    }
}

// GpuContext<Open> 没有特殊的 Drop — 没有要解锁的时钟。
// LockedGpu 在 drop 时始终解锁，即使在 panic 或提前返回时。
```

> **为什么不是 `impl Drop for GpuContext<Locked>`？** Rust 要求 `Drop` 实现应用于泛型类型的 *所有* 实例化。要获取特定状态的清理，使用以下之一：
>
> | 方法 | 优点 | 缺点 |
> |----------|------|------|
> | 单独的包装类型（上面） | 干净，零成本 | 额外的类型名称 |
> | 泛型 `Drop` + 运行时 `TypeId` 检查 | 单一类型 | 需要 `'static`，运行时成本 |
> | 带穷尽 match 的 `enum` 状态在 `Drop` 中 | 单一泛型类型 | 运行时分发，较少类型安全 |

> **何时使用：** BMC 会话、GPU 时钟锁、DMA 缓冲区映射、文件句柄、互斥锁守卫、任何有强制释放步骤的资源。如果你发现自己写 `fn close(&mut self)` 或 `fn cleanup()`，它几乎肯定应该是 `Drop` 而不是。

***

### 技巧13 — 错误类型层次结构作为正确性

良好设计的错误类型防止静默错误吞没，并确保调用者适当处理每个失败模式。使用 `thiserror` 用于结构化错误是一个正确性构造模式：编译器强制穷尽匹配。

```toml
# Cargo.toml
[dependencies]
thiserror = "1"
# 用于应用级错误处理（可选）：
# anyhow = "1"
```

```rust,ignore
use thiserror::Error;

#[derive(Debug, Error)]
pub enum DiagError {
    #[error("IPMI communication failed: {0}")]
    Ipmi(#[from] IpmiError),

    #[error("sensor {sensor_id:#04x} reading out of range: {value}")]
    SensorRange { sensor_id: u8, value: f64 },

    #[error("GPU {gpu_id} not responding")]
    GpuTimeout { gpu_id: u32 },

    #[error("configuration invalid: {0}")]
    Config(String),
}

#[derive(Debug, Error)]
pub enum IpmiError {
    #[error("session authentication failed")]
    AuthFailed,

    #[error("command {net_fn:#04x}/{cmd:#04x} timed out")]
    Timeout { net_fn: u8, cmd: u8 },

    #[error("completion code {0:#04x}")]
    CompletionCode(u8),
}

// 调用者必须处理每个变体 — 没有静默吞没：
fn run_thermal_check() -> Result<(), DiagError> {
    // 如果这返回 IpmiError，它通过 #[from] 属性自动转换为 DiagError::Ipmi
    let temp = read_cpu_temp()?;
    if temp > 105.0 {
        return Err(DiagError::SensorRange {
            sensor_id: 0x20,
            value: temp,
        });
    }
    Ok(())
}

# fn read_cpu_temp() -> Result<f64, DiagError> { Ok(42.0) }
```

**为什么这是正确性构造：**

| 没有结构化错误 | 使用 `thiserror` 枚举 |
|--------------------------|----------------------|
| `fn op() -> Result<T, String>` | `fn op() -> Result<T, DiagError>` |
| 调用者得到不透明字符串 | 调用者匹配特定变体 |
| 无法区分认证失败和超时 | `DiagError::Ipmi(IpmiError::AuthFailed)` vs `Timeout` |
| 日志吞没错误 | `match` 强制处理每个情况 |
| 新错误变体 → 没人注意到 | 新变体 → 编译器警告未匹配 arm |

**`anyhow` vs `thiserror` 决策：**

| 使用 `thiserror` 当… | 使用 `anyhow` 当… |
|-----------------------|-------------------|
| 编写库/crate | 编写二进制/CLI |
| 调用者需要匹配错误变体 | 调用者只记录和退出 |
| 错误类型是公共 API 的一部分 | 内部错误管道 |
| `protocol_lib`, `accel_diag`, `thermal_diag` | `diag_tool` 主二进制 |

> **何时使用：** 工作空间中的每个 crate 都应该用 `thiserror` 定义自己的错误枚举。顶层二进制 crate 可以使用 `anyhow` 来聚合它们。这为库调用者提供编译时错误处理保证，同时保持二进制的人体工学。

***

### 技巧14 — `#[must_use]` 用于强制消费

`#[must_use]` 属性将忽略的返回值转换为编译器警告。这是一个轻量级正确性构造工具，与本指南中的每个模式配对：

```rust,ignore
/// 一个必须使用的校准令牌 — 静默丢弃它是 bug。
#[must_use = "calibration token must be passed to calibrate(), not dropped"]
pub struct CalibrationToken {
    _private: (),
}

/// 一个必须检查的诊断结果 — 忽略失败是 bug。
#[must_use = "diagnostic result must be inspected for failures"]
pub struct DiagResult {
    pub passed: bool,
    pub details: String,
}

/// 返回重要值的函数也应标记：
#[must_use = "the authenticated session must be used or explicitly closed"]
pub fn authenticate(user: &str, pass: &str) -> Result<Session, AuthError> {
    // ...
#   unimplemented!()
}
#
# pub struct Session;
# pub struct AuthError;
```

**编译器告诉你：**

```text
warning: unused `CalibrationToken` that must be used
  --> src/main.rs:5:5
   |
5  |     CalibrationToken { _private: () };
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: calibration token must be passed to calibrate(), not dropped
```

**将这些模式应用 `#[must_use]`：**

| 模式 | 要注解的内容 | 为什么 |
|---------|-----------------|-----|
| 单次使用令牌（ch03） | `CalibrationToken`, `FusePayload` | 丢弃而不使用 = 逻辑 bug |
| 能力令牌（ch04） | `AdminToken` | 认证但忽略令牌 |
| 类型状态转换 | `authenticate()`, `activate()` 的返回类型 | 创建但从未使用的会话 |
| 结果 | `DiagResult`, `SensorReading` | 静默失败吞没 |
| RAII 句柄（技巧 12） | `IpmiSession`, `LockedGpu` | 打开但不使用资源 |

> **经验法则：** 如果丢弃值而不使用它永远是 bug，添加 `#[must_use]`。如果有时是有意的（例如 `Vec`），不要。`_` 前缀（`let _ = foo()`）显式确认并静默警告——当丢弃是有意的时候这是可以的。

## 关键要点

1. **边界处的哨兵值 → Option** — 在解析时将魔法值转换为 `Option`；编译器强制调用者处理 `None`。
2. **密封 trait 关闭实现漏洞** — 私有超 trait 意味着只有你的 crate 可以实现该 trait。
3. **`#[non_exhaustive]` + `#[must_use]` 是一行高价值注解** — 将它们添加到演进枚举和消费令牌。
4. **类型状态构建器强制必填字段** — `finish()` 仅在所有必填类型参数都是 `Set` 时存在。
5. **每个技巧针对特定的 bug 类别** — 增量采用；没有技巧需要重写你的架构。

---
