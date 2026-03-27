# 整合一切 — 完整的诊断平台 🟡

> **你将学到：** 七个核心模式（ch02–ch09）如何组合成单个诊断工作流——认证、会话、类型化命令、审计令牌、量纲结果、验证数据和幽灵类型寄存器——总运行时开销为零。
>
> **交叉引用：** 每个核心模式章节（ch02–ch09），[ch14](ch14-testing-type-level-guarantees.md)（测试这些保证）

## 目标

本章将 **七个模式** 从第2-9章组合成单个现实的诊断工作流。我们将构建一个服务器健康检查，它：

1. **认证**（能力令牌 — ch04）
2. **打开 IPMI 会话**（类型状态 — ch05）
3. **发送类型化命令**（类型化命令 — ch02）
4. **使用单次使用令牌** 进行审计日志（单次使用类型 — ch03）
5. **返回量纲结果**（量纲分析 — ch06）
6. **验证 FRU 数据**（验证边界 — ch07）
7. **读取类型化寄存器**（幽灵类型 — ch09）

```rust,ignore
use std::marker::PhantomData;
use std::io;
// ──── 模式1: 量纲类型 (ch06) ────

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Celsius(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Rpm(pub f64);

#[derive(Debug, Clone, Copy, PartialEq, PartialOrd)]
pub struct Volts(pub f64);

// ──── 模式2: 类型化命令 (ch02) ────

/// 与 ch02 相同的 trait 形状，使用方法（不是关联常量）
/// 以保持一致性。当值真正固定于每个类型时，关联常量
///（`const NETFN: u8`）是同样有效的替代方案。
pub trait IpmiCmd {
    type Response;
    fn net_fn(&self) -> u8;
    fn cmd_byte(&self) -> u8;
    fn payload(&self) -> Vec<u8>;
    fn parse_response(&self, raw: &[u8]) -> io::Result<Self::Response>;
}

pub struct ReadTemp { pub sensor_id: u8 }
impl IpmiCmd for ReadTemp {
    type Response = Celsius;   // ← 量纲类型！
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.sensor_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Celsius> {
        if raw.is_empty() {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "empty"));
        }
        Ok(Celsius(raw[0] as f64))
    }
}

pub struct ReadFanSpeed { pub fan_id: u8 }
impl IpmiCmd for ReadFanSpeed {
    type Response = Rpm;
    fn net_fn(&self) -> u8 { 0x04 }
    fn cmd_byte(&self) -> u8 { 0x2D }
    fn payload(&self) -> Vec<u8> { vec![self.fan_id] }
    fn parse_response(&self, raw: &[u8]) -> io::Result<Rpm> {
        if raw.len() < 2 {
            return Err(io::Error::new(io::ErrorKind::InvalidData, "need 2 bytes"));
        }
        Ok(Rpm(u16::from_le_bytes([raw[0], raw[1]]) as f64))
    }
}

// ──── 模式3: 能力令牌 (ch04) ────

pub struct AdminToken { _private: () }

pub fn authenticate(user: &str, pass: &str) -> Result<AdminToken, &'static str> {
    if user == "admin" && pass == "secret" {
        Ok(AdminToken { _private: () })
    } else {
        Err("authentication failed")
    }
}

// ──── 模式4: 类型状态会话 (ch05) ────

pub struct Idle;
pub struct Active;

pub struct Session<State> {
    host: String,
    _state: PhantomData<State>,
}

impl Session<Idle> {
    pub fn connect(host: &str) -> Self {
        Session { host: host.to_string(), _state: PhantomData }
    }

    pub fn activate(
        self,
        _admin: &AdminToken,  // ← 需要能力令牌
    ) -> Result<Session<Active>, String> {
        println!("Session activated on {}", self.host);
        Ok(Session { host: self.host, _state: PhantomData })
    }
}

impl Session<Active> {
    /// 执行类型化命令 — 仅在 Active 会话上可用。
    /// 返回 io::Result 以传播传输错误（与 ch02 一致）。
    pub fn execute<C: IpmiCmd>(&mut self, cmd: &C) -> io::Result<C::Response> {
        let raw_response = self.raw_send(cmd.net_fn(), cmd.cmd_byte(), &cmd.payload())?;
        cmd.parse_response(&raw_response)
    }

    fn raw_send(&self, _nf: u8, _cmd: u8, _data: &[u8]) -> io::Result<Vec<u8>> {
        Ok(vec![42, 0x1E]) // stub: 原始 IPMI 响应
    }

    pub fn close(self) { println!("Session closed"); }
}

// ──── 模式5: 单次使用审计令牌 (ch03) ────

/// 每次诊断运行获得唯一的审计令牌。
/// 不是 Clone，不是 Copy — 确保每个审计条目是唯一的。
pub struct AuditToken {
    run_id: u64,
}

impl AuditToken {
    pub fn issue(run_id: u64) -> Self {
        AuditToken { run_id }
    }

    /// 消耗令牌以写入审计日志条目。
    pub fn log(self, message: &str) {
        println!("[AUDIT run_id={}] {}", self.run_id, message);
        // 令牌被消耗 — 不能两次记录相同的 run_id
    }
}

// ──── 模式6: 验证边界 (ch07) ────
// 简化自 ch07 的完整 ValidFru — 仅包含这个
// 组合示例所需的字段。参见 ch07 获取完整的 TryFrom<RawFruData> 版本。

pub struct ValidFru {
    pub board_serial: String,
    pub product_name: String,
}

impl ValidFru {
    pub fn parse(raw: &[u8]) -> Result<Self, &'static str> {
        if raw.len() < 8 { return Err("FRU too short"); }
        if raw[0] != 0x01 { return Err("bad FRU version"); }
        Ok(ValidFru {
            board_serial: "SN12345".to_string(),  // stub
            product_name: "ServerX".to_string(),
        })
    }
}

// ──── 模式7: 幽灵类型寄存器 (ch09) ────

pub struct Width16;
pub struct Reg<W> { offset: u16, _w: PhantomData<W> }

impl Reg<Width16> {
    pub fn read(&self) -> u16 { 0x8086 } // stub
}

pub struct PcieDev {
    pub vendor_id: Reg<Width16>,
    pub device_id: Reg<Width16>,
}

impl PcieDev {
    pub fn new() -> Self {
        PcieDev {
            vendor_id: Reg { offset: 0x00, _w: PhantomData },
            device_id: Reg { offset: 0x02, _w: PhantomData },
        }
    }
}

// ──── 组合工作流 ────

fn full_diagnostic() -> Result<(), String> {
    // 1. 认证 → 获取能力令牌
    let admin = authenticate("admin", "secret")
        .map_err(|e| e.to_string())?;

    // 2. 连接并激活会话（类型状态：Idle → Active）
    let session = Session::connect("192.168.1.100");
    let mut session = session.activate(&admin)?;  // 需要 AdminToken

    // 3. 发送类型化命令（响应类型匹配命令）
    let temp: Celsius = session.execute(&ReadTemp { sensor_id: 0 })
        .map_err(|e| e.to_string())?;
    let fan: Rpm = session.execute(&ReadFanSpeed { fan_id: 1 })
        .map_err(|e| e.to_string())?;

    // 类型不匹配会被捕获：
    // let wrong: Volts = session.execute(&ReadTemp { sensor_id: 0 })?;
    //  ❌ 错误：期望 Celsius，发现 Volts

    // 4. 读取幽灵类型 PCIe 寄存器
    let pcie = PcieDev::new();
    let vid: u16 = pcie.vendor_id.read();  // 保证是 u16

    // 5. 在边界验证 FRU 数据
    let raw_fru = vec![0x01, 0x00, 0x00, 0x01, 0x01, 0x00, 0x00, 0xFD];
    let fru = ValidFru::parse(&raw_fru)
        .map_err(|e| e.to_string())?;

    // 6. 颁发单次使用审计令牌
    let audit = AuditToken::issue(1001);

    // 7. 生成报告（所有数据都是类型化的和验证的）
    let report = format!(
        "Server: {} (SN: {}), VID: 0x{:04X}, CPU: {:?}, Fan: {:?}",
        fru.product_name, fru.board_serial, vid, temp, fan,
    );

    // 8. 消耗审计令牌 — 不能两次记录
    audit.log(&report);
    // audit.log("oops");  // ❌ 使用了已移动的值

    // 9. 关闭会话（类型状态：Active → dropped）
    session.close();

    Ok(())
}
```

### 编译器证明了什么

| Bug 类别 | 如何防止 | 模式 |
|-----------|-------------------|---------|
| 未认证访问 | `activate()` 需要 `&AdminToken` | 能力令牌 |
| 在错误会话状态下的命令 | `execute()` 仅存在于 `Session<Active>` | 类型状态 |
| 错误的响应类型 | `ReadTemp::Response = Celsius`，由 trait 固定 | 类型化命令 |
| 单位混淆（°C vs RPM） | `Celsius` ≠ `Rpm` ≠ `Volts` | 量纲类型 |
| 寄存器宽度不匹配 | `Reg<Width16>` 返回 `u16` | 幽灵类型 |
| 处理未验证数据 | 必须首先调用 `ValidFru::parse()` | 验证边界 |
| 重复审计条目 | `AuditToken` 在 log 时被消耗 | 单次使用类型 |
| 乱序电源排序 | 每步需要前一个令牌 | 能力令牌（ch04） |

**所有这些保证的总运行时开销：为零。**

每次检查都在编译时发生。生成的汇编与未检查的 C 代码完全相同——但 **C 可能会有 bug，这个不会**。

## 关键要点

1. **七个模式无缝组合** — 能力令牌、类型状态、类型化命令、单次使用类型、量纲类型、验证边界和幽灵类型都可以协同工作。
2. **编译器证明八类 bug 不可能** — 参见上方的"编译器证明了什么"表。
3. **零总运行时开销** — 生成的汇编与未检查的 C 代码完全相同。
4. **每个模式独立有用** — 你不需要全部七个；可以增量采用。
5. **整合章节是一个设计模板** — 用它作为你自己类型化诊断工作流的起点。
6. **从 IPMI 到大规模 Redfish** — ch17 和 ch18 将这七个模式（加上 ch08 的能力混合）应用于完整的 Redfish 客户端和服务器。这里的 IPMI 工作流是基础；Redfish 演练展示组合如何扩展到具有多个数据源和模式版本约束的生产系统。

---
