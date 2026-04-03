## 避免无检查的索引访问

> **你将学到：** 为什么 `vec[i]` 在 Rust 中是危险的（越界时会 panic），以及安全替代方案如 `.get()`、迭代器和 `HashMap` 的 `entry()` API。用显式处理替代 C++ 的未定义行为。

- 在 C++ 中，`vec[i]` 和 `map[key]` 存在未定义行为/在键缺失时自动插入。Rust 的 `[]` 在越界时会 panic。
- **规则**：除非你能*证明*索引有效，否则使用 `.get()` 而不是 `[]`。

### C++ → Rust 对比
```cpp
// C++ — silent UB or insertion
std::vector<int> v = {1, 2, 3};
int x = v[10];        // UB! No bounds check with operator[]

std::map<std::string, int> m;
int y = m["missing"]; // Silently inserts key with value 0!
```

```rust
// Rust — safe alternatives
let v = vec![1, 2, 3];

// Bad: panics if index out of bounds
// let x = v[10];

// Good: returns Option<&i32>
let x = v.get(10);              // None — no panic
let x = v.get(1).copied().unwrap_or(0);  // 2, or 0 if missing
```

### 真实案例：生产级 Rust 代码中的安全字节解析
```rust
// Example: diagnostics.rs
// Parsing a binary SEL record — buffer might be shorter than expected
let sensor_num = bytes.get(7).copied().unwrap_or(0);
let ppin = cpu_ppin.get(i).map(|s| s.as_str()).unwrap_or("");
```

### 真实案例：使用 `.and_then()` 链式安全查找
```rust
// Example: profile.rs — double lookup: HashMap → Vec
pub fn get_processor(&self, location: &str) -> Option<&Processor> {
    self.processor_by_location
        .get(location)                              // HashMap → Option<&usize>
        .and_then(|&idx| self.processors.get(idx))   // Vec → Option<&Processor>
}
// Both lookups return Option — no panics, no UB
```

### 真实案例：安全的 JSON 导航
```rust
// Example: framework.rs — every JSON key returns Option
let manufacturer = product_fru
    .get("Manufacturer")            // Option<&Value>
    .and_then(|v| v.as_str())       // Option<&str>
    .unwrap_or(UNKNOWN_VALUE)       // &str (safe fallback)
    .to_string();
```
对比 C++ 模式：`json["SystemInfo"]["ProductFru"]["Manufacturer"]` —— 任何键缺失都会抛出 `nlohmann::json::out_of_range`。

### 何时可以使用 `[]`
- **在边界检查之后**：`if i < v.len() { v[i] }`
- **在测试中**：需要 panic 的行为时
- **使用常量**：在 `assert!(!v.is_empty());` 之后立即使用 `let first = v[0];`

----

## 使用 unwrap_or 安全提取值

- `unwrap()` 在 `None` / `Err` 时会 panic。在生产代码中，优先使用安全替代方案。

### unwrap 系列方法
| **方法** | **None/Err 时的行为** | **适用场景** |
|-----------|------------------------|-------------|
| `.unwrap()` | **Panic** | 仅在测试中，或可证明不会失败时 |
| `.expect("msg")` | 带消息的 panic | 当 panic 是合理的，说明原因 |
| `.unwrap_or(default)` | 返回 `default` | 有廉价的常量回退值 |
| `.unwrap_or_else(\|\| expr)` | 调用闭包 | 回退值计算代价较高 |
| `.unwrap_or_default()` | 返回 `Default::default()` | 类型实现了 `Default` |

### 真实案例：使用安全默认值解析
```rust
// Example: peripherals.rs
// Regex capture groups might not match — provide safe fallbacks
let bus_hex = caps.get(1).map(|m| m.as_str()).unwrap_or("00");
let fw_status = caps.get(5).map(|m| m.as_str()).unwrap_or("0x0");
let bus = u8::from_str_radix(bus_hex, 16).unwrap_or(0);
```

### 真实案例：使用 fallback struct 的 `unwrap_or_else`
```rust
// Example: framework.rs
// Full function wraps logic in an Option-returning closure;
// if anything fails, return a default struct:
(|| -> Option<BaseboardFru> {
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    // ... extract fields with .get()? chains
    Some(baseboard_fru)
})()
.unwrap_or_else(|| BaseboardFru {
    manufacturer: String::new(),
    model: String::new(),
    product_part_number: String::new(),
    serial_number: String::new(),
    asset_tag: String::new(),
})
```

### 真实案例：在配置反序列化中使用 `unwrap_or_default`
```rust
// Example: framework.rs
// If JSON config parsing fails, fall back to Default — no crash
Ok(json) => serde_json::from_str(&json).unwrap_or_default(),
```
对应的 C++ 写法是在 `nlohmann::json::parse()` 周围使用 `try/catch`，并在 catch 块中手动构造默认值。

----

## 函数式转换：map、map_err、find_map

- `Option` 和 `Result` 上的这些方法让你在不拆开包装的情况下转换内部值，用线性链式调用替代嵌套的 `if/else`。

### 快速参考
| **方法** | **适用类型** | **功能** | **C++ 等价写法** |
|-----------|-------|---------|-------------------|
| `.map(\|v\| ...)` | `Option` / `Result` | 转换 `Some`/`Ok` 值 | `if (opt) { *opt = transform(*opt); }` |
| `.map_err(\|e\| ...)` | `Result` | 转换 `Err` 值 | 在 catch 块中添加上下文 |
| `.and_then(\|v\| ...)` | `Option` / `Result` | 链式调用返回 `Option`/`Result` 的操作 | 嵌套的 if 检查 |
| `.find_map(\|v\| ...)` | 迭代器 | 一次遍历完成 `find` + `map` | 使用 `if + break` 的循环 |
| `.filter(\|v\| ...)` | `Option` / 迭代器 | 只保留满足谓词的值 | `if (!predicate) return nullopt;` |
| `.ok()?` | `Result` | 将 `Result → Option` 并传播 `None` | `if (result.has_error()) return nullopt;` |

### 真实案例：用于 JSON 字段提取的 `.and_then()` 链
```rust
// Example: framework.rs — finding serial number with fallbacks
let sys_info = json.get("SystemInfo")?;

// Try BaseboardFru.BoardSerialNumber first
if let Some(serial) = sys_info
    .get("BaseboardFru")
    .and_then(|b| b.get("BoardSerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)     // Only accept non-empty, valid serials
{
    return Some(serial.to_string());
}

// Fallback to BoardFru.SerialNumber
sys_info
    .get("BoardFru")
    .and_then(|b| b.get("SerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)
    .map(|s| s.to_string())   // Convert &str → String only if Some
```
在 C++ 中这会是一个 `if (json.contains("BaseboardFru")) { if (json["BaseboardFru"].contains("BoardSerialNumber")) { ... } }` 的金字塔结构。

### 真实案例：`find_map` — 一次遍历完成搜索和转换
```rust
// Example: context.rs — find SDR record matching sensor + owner
pub fn find_for_event(&self, sensor_number: u8, owner_id: u8) -> Option<&SdrRecord> {
    self.by_sensor.get(&sensor_number).and_then(|indices| {
        indices.iter().find_map(|&i| {
            let record = &self.records[i];
            if record.sensor_owner_id() == Some(owner_id) {
                Some(record)
            } else {
                None
            }
        })
    })
}
```
`find_map` 是 `find` + `map` 的融合：它在第一个匹配处停止并转换它。对应的 C++ 写法是使用 `for` 循环配合 `if` + `break`。

### 真实案例：使用 `map_err` 添加错误上下文
```rust
// Example: main.rs — add context to errors before propagating
let json_str = serde_json::to_string_pretty(&config)
    .map_err(|e| format!("Failed to serialize config: {}", e))?;
```
将 `serde_json::Error` 转换为一个描述性的 `String` 错误，包含关于*什么*失败了的上下文信息。

----

## JSON 处理：nlohmann::json → serde

- C++ 团队通常使用 `nlohmann::json` 进行 JSON 解析。Rust 使用 **serde** + **serde_json** —— 这更强大，因为 JSON schema 被编码*在类型系统中*。

### C++ (nlohmann) vs Rust (serde) 对比

```cpp
// C++ with nlohmann::json — runtime field access
#include <nlohmann/json.hpp>
using json = nlohmann::json;

struct Fan {
    std::string logical_id;
    std::vector<std::string> sensor_ids;
};

Fan parse_fan(const json& j) {
    Fan f;
    f.logical_id = j.at("LogicalID").get<std::string>();    // throws if missing
    if (j.contains("SDRSensorIdHexes")) {                   // manual default handling
        f.sensor_ids = j["SDRSensorIdHexes"].get<std::vector<std::string>>();
    }
    return f;
}
```

```rust
// Rust with serde — compile-time schema, automatic field mapping
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Fan {
    pub logical_id: String,
    #[serde(rename = "SDRSensorIdHexes", default)]  // JSON key → Rust field
    pub sensor_ids: Vec<String>,                     // Missing → empty Vec
    #[serde(default)]
    pub sensor_names: Vec<String>,                   // Missing → empty Vec
}

// One line replaces the entire parse function:
let fan: Fan = serde_json::from_str(json_str)?;
```

### 重要的 serde 属性（来自生产级 Rust 代码的真实案例）

| **属性** | **用途** | **C++ 等价写法** |
|--------------|------------|--------------------|
| `#[serde(default)]` | 对缺失字段使用 `Default::default()` | `if (j.contains(key)) { ... } else { default; }` |
| `#[serde(rename = "Key")]` | 将 JSON 键名映射到 Rust 字段名 | 手动 `j.at("Key")` 访问 |
| `#[serde(flatten)]` | 将未知键吸收到 `HashMap` 中 | `for (auto& [k,v] : j.items()) { ... }` |
| `#[serde(skip)]` | 不序列化/反序列化此字段 | 不存储在 JSON 中 |
| `#[serde(tag = "type")]` | 内部带标签的枚举（判别字段） | `if (j["type"] == "gpu") { ... }` |

### 真实案例：完整的配置结构体
```rust
// Example: diag.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DiagConfig {
    pub sku: SkuConfig,
    #[serde(default)]
    pub level: DiagLevel,            // Missing → DiagLevel::default()
    #[serde(default)]
    pub modules: ModuleConfig,       // Missing → ModuleConfig::default()
    #[serde(default)]
    pub output_dir: String,          // Missing → ""
    #[serde(default, flatten)]
    pub options: HashMap<String, serde_json::Value>,  // Absorbs unknown keys
}

// Loading is 3 lines (vs ~20+ in C++ with nlohmann):
let content = std::fs::read_to_string(path)?;
let config: DiagConfig = serde_json::from_str(&content)?;
Ok(config)
```

### 使用 `#[serde(tag = "type")]` 进行枚举反序列化
```rust
// Example: components.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]                   // JSON: {"type": "Gpu", "product": ...}
pub enum PcieDeviceKind {
    Gpu { product: GpuProduct, manufacturer: GpuManufacturer },
    Nic { product: NicProduct, manufacturer: NicManufacturer },
    NvmeDrive { drive_type: StorageDriveType, capacity_gb: u32 },
    // ... 9 more variants
}
// serde automatically dispatches on the "type" field — no manual if/else chain
```
对应的 C++ 写法是：`if (j["type"] == "Gpu") { parse_gpu(j); } else if (j["type"] == "Nic") { parse_nic(j); } ...`

# 练习：使用 serde 进行 JSON 反序列化

- 定义一个可以从以下 JSON 反序列化的 `ServerConfig` 结构体：
```json
{
    "hostname": "diag-node-01",
    "port": 8080,
    "debug": true,
    "modules": ["accel_diag", "nic_diag", "cpu_diag"]
}
```
- 使用 `#[derive(Deserialize)]` 和 `serde_json::from_str()` 来解析它
- 给 `debug` 添加 `#[serde(default)]`，使其在缺失时默认为 `false`
- **加分题**：添加一个 `enum DiagLevel { Quick, Full, Extended }` 字段，带有 `#[serde(default)]`，默认为 `Quick`

**起始代码**（需要 `cargo add serde --features derive` 和 `cargo add serde_json`）：
```rust
use serde::Deserialize;

// TODO: 定义 DiagLevel 枚举并实现 Default

// TODO: 定义 ServerConfig 结构体，添加 serde 属性

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    // TODO: 反序列化并打印配置
    // TODO: 尝试解析缺少 "debug" 字段的 JSON —— 验证它默认为 false
}
```

<details><summary>答案（点击展开）</summary>

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize, Default)]
enum DiagLevel {
    #[default]
    Quick,
    Full,
    Extended,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    hostname: String,
    port: u16,
    #[serde(default)]       // defaults to false if missing
    debug: bool,
    modules: Vec<String>,
    #[serde(default)]       // defaults to DiagLevel::Quick if missing
    level: DiagLevel,
}

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    let config: ServerConfig = serde_json::from_str(json_input)
        .expect("Failed to parse JSON");
    println!("{config:#?}");

    // Test with missing optional fields
    let minimal = r#"{
        "hostname": "node-02",
        "port": 9090,
        "modules": []
    }"#;
    let config2: ServerConfig = serde_json::from_str(minimal)
        .expect("Failed to parse minimal JSON");
    println!("debug (default): {}", config2.debug);    // false
    println!("level (default): {:?}", config2.level);  // Quick
}
// Output:
// ServerConfig {
//     hostname: "diag-node-01",
//     port: 8080,
//     debug: true,
//     modules: ["accel_diag", "nic_diag", "cpu_diag"],
//     level: Quick,
// }
// debug (default): false
// level (default): Quick
```

</details>

----


