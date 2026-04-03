## 使用闭包压缩赋值金字塔

> **你将学到：** 如何将 Rust 基于表达式的语法和闭包将深层嵌套的 C++ `if/else` 验证链展平为简洁的线性代码。

- C++ 通常需要多块 `if/else` 链来赋值变量，尤其是在涉及验证或回退逻辑时。Rust 基于表达式的语法和闭包将这些压缩为扁平的线性代码。

### 模式 1：使用 `if` 表达式的元组赋值

```cpp
// C++ — 三个变量在多块 if/else 链中设置
uint32_t fault_code;
const char* der_marker;
const char* action;
if (is_c44ad) {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
} else if (error.is_hardware_error()) {
    fault_code = 67956; der_marker = "CSI_ERR"; action = "Replace GPU";
} else {
    fault_code = 32709; der_marker = "CSI_WARN"; action = "No action";
}
```

```rust
// Rust 等效：accel_fieldiag.rs
// 单个表达式同时赋值所有三个变量：
let (fault_code, der_marker, recommended_action) = if is_c44ad {
    (32709u32, "CSI_WARN", "No action")
} else if error.is_hardware_error() {
    (67956u32, "CSI_ERR", "Replace GPU")
} else {
    (32709u32, "CSI_WARN", "No action")
};
```

### 模式 2：IIFE（即刻调用函数表达式）用于可失败的链

```cpp
// C++ — JSON 导航的金字塔噩梦
std::string get_part_number(const nlohmann::json& root) {
    if (root.contains("SystemInfo")) {
        auto& sys = root["SystemInfo"];
        if (sys.contains("BaseboardFru")) {
            auto& bb = sys["BaseboardFru"];
            if (bb.contains("ProductPartNumber")) {
                return bb["ProductPartNumber"].get<std::string>();
            }
        }
    }
    return "UNKNOWN";
}
```

```rust
// Rust 等效：framework.rs
// 闭包 + ? 操作符将金字塔压缩为线性代码：
let part_number = (|| -> Option<String> {
    let path = self.args.sysinfo.as_ref()?;
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    let ppn = json
        .get("SystemInfo")?
        .get("BaseboardFru")?
        .get("ProductPartNumber")?
        .as_str()?;
    Some(ppn.to_string())
})()
.unwrap_or_else(|| "UNKNOWN".to_string());
```

闭包创建一个 `Option<String>` 作用域，其中 `?` 在任何步骤提前退出。`.unwrap_or_else()` 在末尾一次性提供回退值。

### 模式 3：使用迭代器链替代手动循环 + push_back

```cpp
// C++ — 带有中间变量的手动循环
std::vector<std::tuple<std::vector<std::string>, std::string, std::string>> gpu_info;
for (const auto& [key, info] : gpu_pcie_map) {
    std::vector<std::string> bdfs;
    // ... 将 bdf_path 解析为 bdfs
    std::string serial = info.serial_number.value_or("UNKNOWN");
    std::string model = info.model_number.value_or(model_name);
    gpu_info.push_back({bdfs, serial, model});
}
```

```rust
// Rust 等效：peripherals.rs
// 单链：values() → map → collect
let gpu_info: Vec<(Vec<String>, String, String, String)> = self
    .gpu_pcie_map
    .values()
    .map(|info| {
        let bdfs: Vec<String> = info.bdf_path
            .split(')')
            .filter(|s| !s.is_empty())
            .map(|s| s.trim_start_matches('(').to_string())
            .collect();
        let serial = info.serial_number.clone()
            .unwrap_or_else(|| "UNKNOWN".to_string());
        let model = info.model_number.clone()
            .unwrap_or_else(|| model_name.to_string());
        let gpu_bdf = format!("{}:{}:{}.{}",
            info.bdf.segment, info.bdf.bus, info.bdf.device, info.bdf.function);
        (bdfs, serial, model, gpu_bdf)
    })
    .collect();
```

### 模式 4：使用 `.filter().collect()` 替代循环 + `if (condition) continue`

```cpp
// C++
std::vector<TestResult*> failures;
for (auto& t : test_results) {
    if (!t.is_pass()) {
        failures.push_back(&t);
    }
}
```

```rust
// Rust — 来自 accel_diag/src/healthcheck.rs
pub fn failed_tests(&self) -> Vec<&TestResult> {
    self.test_results.iter().filter(|t| !t.is_pass()).collect()
}
```

### 总结：何时使用每种模式

| **C++ 模式** | **Rust 替代方案** | **主要优势** |
|----------------|---------------------|----------------|
| 多块变量赋值 | `let (a, b) = if ... { } else { };` | 所有变量原子性地绑定 |
| 嵌套 `if (contains)` 金字塔 | 带 `?` 操作符的 IIFE 闭包 | 线性、扁平、提前退出 |
| `for` 循环 + `push_back` | `.iter().map(\|\|).collect()` | 无需中间 mut Vec |
| `for` + `if (cond) continue` | `.iter().filter(\|\|).collect()` | 声明式意图 |
| `for` + `if + break`（查找首个） | `.iter().find_map(\|\|)` | 单次遍历搜索和转换 |

----

# 实践练习：诊断事件管道

🔴 **挑战** — 综合练习，结合枚举、特质、迭代器、错误处理和泛型

这个综合练习将枚举、特质、迭代器、错误处理和泛型结合在一起。你将构建一个简化的诊断事件处理管道，类似于生产级 Rust 代码中使用的模式。

**要求：**

1. 定义一个 `enum Severity { Info, Warning, Critical }`，实现 `Display`，并定义一个包含 `source: String`、`severity: Severity`、`message: String` 和 `fault_code: u32` 的 `struct DiagEvent`
2. 定义一个 `trait EventFilter`，其中包含方法 `fn should_include(&self, event: &DiagEvent) -> bool`
3. 实现两个过滤器：`SeverityFilter`（仅包含大于等于给定严重级别的事件）和 `SourceFilter`（仅包含来自特定源字符串的事件）
4. 编写函数 `fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String>`，返回通过**所有**过滤器的事件的格式化报告行
5. 编写 `fn parse_event(line: &str) -> Result<DiagEvent, String>`，解析形式为 `"source:severity:fault_code:message"` 的行（对错误输入返回 `Err`）

**起始代码：**

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        todo!()
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}
// TODO: impl EventFilter for SeverityFilter

struct SourceFilter {
    source: String,
}
// TODO: impl EventFilter for SourceFilter

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    // TODO: 过滤通过所有过滤器的事件，格式化为
    // "[SEVERITY] source (FC:fault_code): message"
    todo!()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    // 解析 "source:severity:fault_code:message"
    // 对无效输入返回 Err
    todo!()
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    // 解析所有行，收集成功结果并报告错误
    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    // 应用过滤器：仅限来自 accel_diag 的 Critical 和 Warning 事件
    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
```

<details><summary>解决方案（点击展开）</summary>

```rust
use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, PartialOrd, Ord)]
enum Severity {
    Info,
    Warning,
    Critical,
}

impl fmt::Display for Severity {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Severity::Info => write!(f, "INFO"),
            Severity::Warning => write!(f, "WARNING"),
            Severity::Critical => write!(f, "CRITICAL"),
        }
    }
}

impl Severity {
    fn from_str(s: &str) -> Result<Self, String> {
        match s {
            "Info" => Ok(Severity::Info),
            "Warning" => Ok(Severity::Warning),
            "Critical" => Ok(Severity::Critical),
            other => Err(format!("Unknown severity: {other}")),
        }
    }
}

#[derive(Debug, Clone)]
struct DiagEvent {
    source: String,
    severity: Severity,
    message: String,
    fault_code: u32,
}

trait EventFilter {
    fn should_include(&self, event: &DiagEvent) -> bool;
}

struct SeverityFilter {
    min_severity: Severity,
}

impl EventFilter for SeverityFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.severity >= self.min_severity
    }
}

struct SourceFilter {
    source: String,
}

impl EventFilter for SourceFilter {
    fn should_include(&self, event: &DiagEvent) -> bool {
        event.source == self.source
    }
}

fn process_events(events: &[DiagEvent], filters: &[&dyn EventFilter]) -> Vec<String> {
    events.iter()
        .filter(|e| filters.iter().all(|f| f.should_include(e)))
        .map(|e| format!("[{}] {} (FC:{}): {}", e.severity, e.source, e.fault_code, e.message))
        .collect()
}

fn parse_event(line: &str) -> Result<DiagEvent, String> {
    let parts: Vec<&str> = line.splitn(4, ':').collect();
    if parts.len() != 4 {
        return Err(format!("Expected 4 colon-separated fields, got {}", parts.len()));
    }
    let fault_code = parts[2].parse::<u32>()
        .map_err(|e| format!("Invalid fault code '{}': {e}", parts[2]))?;
    Ok(DiagEvent {
        source: parts[0].to_string(),
        severity: Severity::from_str(parts[1])?,
        fault_code,
        message: parts[3].to_string(),
    })
}

fn main() {
    let raw_lines = vec![
        "accel_diag:Critical:67956:ECC uncorrectable error detected",
        "nic_diag:Warning:32709:Link speed degraded",
        "accel_diag:Info:10001:Self-test passed",
        "cpu_diag:Critical:55012:Thermal throttling active",
        "accel_diag:Warning:32710:PCIe link width reduced",
    ];

    let events: Vec<DiagEvent> = raw_lines.iter()
        .filter_map(|line| match parse_event(line) {
            Ok(e) => Some(e),
            Err(e) => { eprintln!("Parse error: {e}"); None }
        })
        .collect();

    let sev_filter = SeverityFilter { min_severity: Severity::Warning };
    let src_filter = SourceFilter { source: "accel_diag".to_string() };
    let filters: Vec<&dyn EventFilter> = vec![&sev_filter, &src_filter];

    let report = process_events(&events, &filters);
    for line in &report {
        println!("{line}");
    }
    println!("--- {} event(s) matched ---", report.len());
}
// 输出：
// [CRITICAL] accel_diag (FC:67956): ECC uncorrectable error detected
// [WARNING] accel_diag (FC:32710): PCIe link width reduced
// --- 2 event(s) matched ---
```

</details>

----
