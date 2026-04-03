## Unsafe Rust

> **学习内容：** `unsafe` 允许的操作（原始指针、FFI、未检查的强制转换）、安全包装模式、
> C# P/Invoke 与 Rust FFI 的对比以调用原生代码，以及 `unsafe` 代码块的安全检查清单。
>
> **难度：** 🔴 高级

Unsafe Rust 允许你执行借用检查器无法验证的操作。应谨慎使用，并附有清晰的文档。

> **高级覆盖**：关于 unsafe 代码的安全抽象模式（arena 分配器、无锁结构、自定义 vtable），请参阅 [Rust Patterns](../../source-docs/RUST_PATTERNS.md)。

### 何时需要 Unsafe
```rust
// 1. 解引用原始指针
let mut value = 42;
let ptr = &mut value as *mut i32;
unsafe {
    *ptr = 100; // 必须在 unsafe 代码块中
}

// 2. 调用 unsafe 函数
unsafe fn dangerous() {
    // 需要调用者维护不变量的内部实现
}

unsafe {
    dangerous(); // 调用者承担责任
}

// 3. 访问可变的静态变量
static mut COUNTER: u32 = 0;
unsafe {
    COUNTER += 1; // 非线程安全 — 调用者必须确保同步
}

// 4. 实现 unsafe trait
unsafe trait UnsafeTrait {
    fn do_something(&self);
}
```

### C# 对比：unsafe 关键字
```csharp
// C# unsafe - 相似概念，不同作用域
unsafe void UnsafeExample()
{
    int value = 42;
    int* ptr = &value;
    *ptr = 100;
    
    // C# unsafe 是关于指针运算的
    // Rust unsafe 是关于所有权/借用规则放松的
}

// C# fixed - 固定托管对象
unsafe void PinnedExample()
{
    byte[] buffer = new byte[100];
    fixed (byte* ptr = buffer)
    {
        // ptr 仅在此代码块内有效
    }
}
```

### 安全包装器
```rust
/// 关键模式：用安全的 API 包装 unsafe 代码
pub struct SafeBuffer {
    data: Vec<u8>,
}

impl SafeBuffer {
    pub fn new(size: usize) -> Self {
        SafeBuffer { data: vec![0; size] }
    }
    
    /// 安全的 API — 带边界检查的访问
    pub fn get(&self, index: usize) -> Option<u8> {
        self.data.get(index).copied()
    }
    
    /// 快速无检查访问 — unsafe 但通过边界检查安全包装
    pub fn get_unchecked_safe(&self, index: usize) -> Option<u8> {
        if index < self.data.len() {
            // SAFETY: 我们刚检查了索引在范围内
            Some(unsafe { *self.data.get_unchecked(index) })
        } else {
            None
        }
    }
}
```

***

## 通过 FFI 与 C# 互操作

Rust 可以暴露 C 兼容的函数，C# 可以通过 P/Invoke 调用。

```mermaid
graph LR
    subgraph "C# Process"
        CS["C# Code"] -->|"P/Invoke"| MI["Marshal Layer\nUTF-16 → UTF-8\nstruct layout"]
    end
    MI -->|"C ABI call"| FFI["FFI Boundary"]
    subgraph "Rust cdylib (.so / .dll)"
        FFI --> RF["extern \"C\" fn\n#[no_mangle]"]
        RF --> Safe["Safe Rust\ninternals"]
    end

    style FFI fill:#fff9c4,color:#000
    style MI fill:#bbdefb,color:#000
    style Safe fill:#c8e6c9,color:#000
```

### Rust 库（编译为 cdylib）
```rust
// src/lib.rs
#[no_mangle]
pub extern "C" fn add_numbers(a: i32, b: i32) -> i32 {
    a + b
}

#[no_mangle]
pub extern "C" fn process_string(input: *const std::os::raw::c_char) -> i32 {
    let c_str = unsafe {
        if input.is_null() {
            return -1;
        }
        std::ffi::CStr::from_ptr(input)
    };
    
    match c_str.to_str() {
        Ok(s) => s.len() as i32,
        Err(_) => -1,
    }
}
```

```toml
# Cargo.toml
[lib]
crate-type = ["cdylib"]
```

### C# 消费者（P/Invoke）
```csharp
using System.Runtime.InteropServices;

public static class RustInterop
{
    [DllImport("my_rust_lib", CallingConvention = CallingConvention.Cdecl)]
    public static extern int add_numbers(int a, int b);
    
    [DllImport("my_rust_lib", CallingConvention = CallingConvention.Cdecl)]
    public static extern int process_string(
        [MarshalAs(UnmanagedType.LPUTF8Str)] string input);
}

// Usage
int sum = RustInterop.add_numbers(5, 3);  // 8
int len = RustInterop.process_string("Hello from C#!");  // 15
```

### FFI 安全检查清单

将 Rust 函数暴露给 C# 时，以下规则可以防止最常见的错误：

1. **始终使用 `extern "C"`** — 没有它，Rust 使用自己的（不稳定的）调用约定。C# P/Invoke 期望的是 C ABI。

2. **`#[no_mangle]`** — 防止 Rust 编译器修改函数名。没有它，C# 无法找到该符号。

3. **永远不要让 panic 跨越 FFI 边界** — Rust panic 解栈到 C# 中是**未定义行为**。在 FFI 入口点捕获 panic：
    ```rust
    #[no_mangle]
    pub extern "C" fn safe_ffi_function() -> i32 {
        match std::panic::catch_unwind(|| {
            // 实际逻辑
            42
        }) {
            Ok(result) => result,
            Err(_) => -1,  // 返回错误码而不是向 C# 抛出 panic
        }
    }
    ```

4. **透明 vs 不透明结构体** — 如果 C# 仅持有指针（不透明句柄），则不需要 `#[repr(C)]`。如果 C# 通过 `StructLayout` 读取结构体字段，你**必须**使用 `#[repr(C)]`：
    ```rust
    // 不透明 — C# 仅持有 IntPtr。不需要 #[repr(C)]。
    pub struct Connection { /* 仅 Rust 字段 */ }

    // 透明 — C# 直接封送字段。必须使用 #[repr(C)]。
    #[repr(C)]
    pub struct Point { pub x: f64, pub y: f64 }
    ```

5. **空指针检查** — 在解引用前始终验证指针。C# 可以传递 `IntPtr.Zero`。

6. **字符串编码** — C# 内部使用 UTF-16。`MarshalAs(UnmanagedType.LPUTF8Str)` 转换为 UTF-8 以供 Rust 的 `CStr` 使用。明确记录此约定。

### 端到端示例：不透明句柄与生命周期管理

这是生产环境中的常见模式：Rust 拥有对象，C# 持有不透明句柄，显式的创建/销毁函数管理生命周期。

**Rust 端**（`src/lib.rs`）：
```rust
use std::ffi::{c_char, CStr};

pub struct ImageProcessor {
    width: u32,
    height: u32,
    pixels: Vec<u8>,
}

/// 创建新的处理器。尺寸无效时返回空指针。
#[no_mangle]
pub extern "C" fn processor_new(width: u32, height: u32) -> *mut ImageProcessor {
    if width == 0 || height == 0 {
        return std::ptr::null_mut();
    }
    let proc = ImageProcessor {
        width,
        height,
        pixels: vec![0u8; (width * height * 4) as usize],
    };
    Box::into_raw(Box::new(proc)) // 在堆上分配，返回原始指针
}

/// 应用灰度滤镜。成功返回 0，空指针返回 -1。
#[no_mangle]
pub extern "C" fn processor_grayscale(ptr: *mut ImageProcessor) -> i32 {
    let proc = match unsafe { ptr.as_mut() } {
        Some(p) => p,
        None => return -1,
    };
    for chunk in proc.pixels.chunks_exact_mut(4) {
        let gray = (0.299 * chunk[0] as f64
                  + 0.587 * chunk[1] as f64
                  + 0.114 * chunk[2] as f64) as u8;
        chunk[0] = gray;
        chunk[1] = gray;
        chunk[2] = gray;
    }
    0
}

/// 销毁处理器。可以传入空指针。
#[no_mangle]
pub extern "C" fn processor_free(ptr: *mut ImageProcessor) {
    if !ptr.is_null() {
        // SAFETY: ptr 由 processor_new 通过 Box::into_raw 创建
        unsafe { drop(Box::from_raw(ptr)); }
    }
}
```

**C# 端**：
```csharp
using System.Runtime.InteropServices;

public sealed class ImageProcessor : IDisposable
{
    [DllImport("image_rust", CallingConvention = CallingConvention.Cdecl)]
    private static extern IntPtr processor_new(uint width, uint height);

    [DllImport("image_rust", CallingConvention = CallingConvention.Cdecl)]
    private static extern int processor_grayscale(IntPtr ptr);

    [DllImport("image_rust", CallingConvention = CallingConvention.Cdecl)]
    private static extern void processor_free(IntPtr ptr);

    private IntPtr _handle;

    public ImageProcessor(uint width, uint height)
    {
        _handle = processor_new(width, height);
        if (_handle == IntPtr.Zero)
            throw new ArgumentException("Invalid dimensions");
    }

    public void Grayscale()
    {
        if (processor_grayscale(_handle) != 0)
            throw new InvalidOperationException("Processor is null");
    }

    public void Dispose()
    {
        if (_handle != IntPtr.Zero)
        {
            processor_free(_handle);
            _handle = IntPtr.Zero;
        }
    }
}

// 使用 — IDisposable 确保 Rust 内存被释放
using var proc = new ImageProcessor(1920, 1080);
proc.Grayscale();
// proc.Dispose() 自动调用 → processor_free() → Rust 释放 Vec
```

> **关键洞察**：这是 Rust 等价于 C# 的 `SafeHandle` 模式。Rust 的 `Box::into_raw` / `Box::from_raw` 在 FFI 边界转移所有权，C# 的 `IDisposable` 包装器确保清理。

---

## 练习

<details>
<summary><strong>🏋️ 练习：原始指针的安全包装器</strong>（点击展开）</summary>

你从 C 库收到一个原始指针。编写一个安全的 Rust 包装器：

```rust
// 模拟 C API
extern "C" {
    fn lib_create_buffer(size: usize) -> *mut u8;
    fn lib_free_buffer(ptr: *mut u8);
}
```

要求：
1. 创建一个包装原始指针的 `SafeBuffer` 结构体
2. 实现 `Drop` 以调用 `lib_free_buffer`
3. 通过 `as_slice()` 提供安全的 `&[u8]` 视图
4. 确保如果指针为空，`SafeBuffer::new()` 返回 `None`

<details>
<summary>🔑 答案</summary>

```rust,ignore
struct SafeBuffer {
    ptr: *mut u8,
    len: usize,
}

impl SafeBuffer {
    fn new(size: usize) -> Option<Self> {
        let ptr = unsafe { lib_create_buffer(size) };
        if ptr.is_null() {
            None
        } else {
            Some(SafeBuffer { ptr, len: size })
        }
    }

    fn as_slice(&self) -> &[u8] {
        // SAFETY: ptr 为非空（在新函数中已检查），len 是
        // 分配的大小，且我们持有独占所有权。
        unsafe { std::slice::from_raw_parts(self.ptr, self.len) }
    }
}

impl Drop for SafeBuffer {
    fn drop(&mut self) {
        // SAFETY: ptr 由 lib_create_buffer 分配
        unsafe { lib_free_buffer(self.ptr); }
    }
}

// 使用：所有 unsafe 都封装在 SafeBuffer 中
fn process(buf: &SafeBuffer) {
    let data = buf.as_slice(); // 完全安全的 API
    println!("First byte: {}", data[0]);
}
```

**关键模式**：用 `// SAFETY:` 注释将 `unsafe` 封装在一个小模块中。暴露 100% 安全的公共 API。这就是 Rust 标准库的工作方式 — `Vec`、`String`、`HashMap` 内部都包含 unsafe 但提供安全的接口。

</details>
</details>

***


