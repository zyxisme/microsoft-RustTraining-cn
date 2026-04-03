# Rust 泛型

> **你将学到什么：** 泛型类型参数、单态化（零成本泛型）、trait bounds，以及 Rust 泛型如何与 C++ 模板比较——具有更好的错误消息和没有 SFINAE。

- 泛型允许相同的算法或数据结构跨数据类型重用
    - 泛型参数作为标识符出现在 `<>` 中，例如：`<T>`。参数可以是任何合法标识符名称，但通常保持简短以简洁</    - 编译器在编译时执行单态化，即为遇到的每个 `T` 变体生成一个新类型
```rust
// Returns a tuple of type <T> composed of left and right of type <T>
fn pick<T>(x: u32, left: T, right: T) -> (T, T) {
   if x == 42 {
    (left, right) 
   } else {
    (right, left)
   }
}
fn main() {
    let a = pick(42, true, false);
    let b = pick(42, "hello", "world");
    println!("{a:?}, {b:?}");
}
```

# Rust 泛型
- 泛型也可以应用于数据类型和关联方法。可以为特定的 `<T>` 专门化实现（例如：`f32` vs. `u32`）
```rust
#[derive(Debug)] // We will discuss this later
struct Point<T> {
    x : T,
    y : T,
}
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point {x, y}
    }
    fn set_x(&mut self, x: T) {
         self.x = x;       
    }
    fn set_y(&mut self, y: T) {
         self.y = y;       
    }
}
impl Point<f32> {
    fn is_secret(&self) -> bool {
        self.x == 42.0
    }    
}
fn main() {
    let mut p = Point::new(2, 4); // i32
    let q = Point::new(2.0, 4.0); // f32
    p.set_x(42);
    p.set_y(43);
    println!("{p:?} {q:?} {}", q.is_secret());
}
```

# 练习：泛型

🟢 **入门级**
- 修改 `Point` 类型，为 x 和 y 使用两种不同的类型（`T` 和 `U`）

<details><summary>Solution (click to expand)</summary>

```rust
#[derive(Debug)]
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn new(x: T, y: U) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p1 = Point::new(42, 3.14);        // Point<i32, f64>
    let p2 = Point::new("hello", true);   // Point<&str, bool>
    let p3 = Point::new(1u8, 1000u64);    // Point<u8, u64>
    println!("{p1:?}");
    println!("{p2:?}");
    println!("{p3:?}");
}
// Output:
// Point { x: 42, y: 3.14 }
// Point { x: "hello", y: true }
// Point { x: 1, y: 1000 }
```

</details>

### 结合 Rust traits 和泛型
- Traits 可用于对泛型类型施加限制（约束）
- 约束可以使用 `: ` 在泛型类型参数后指定，或使用 `where`。以下定义了一个泛型函数 `get_area`，它接受任何实现了 `ComputeArea` trait 的类型 `T`
```rust
    trait ComputeArea {
        fn area(&self) -> u64;
    }
    fn get_area<T: ComputeArea>(t: &T) -> u64 {
        t.area()
    }
```
- [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

### 结合 Rust traits 和泛型
- 可以使用多个 trait 约束
```rust
trait Fish {}
trait Mammal {}
struct Shark;
struct Whale;
impl Fish for Shark {}
impl Fish for Whale {}
impl Mammal for Whale {}
fn only_fish_and_mammals<T: Fish + Mammal>(_t: &T) {}
fn main() {
    let w = Whale {};
    only_fish_and_mammals(&w);
    let _s = Shark {};
    // Won't compile
    only_fish_and_mammals(&_s);
}
```

### Rust trait 约束在数据类型中的应用
- Trait 约束可以与数据类型中的泛型组合使用
- 在下面的示例中，我们定义 ```PrintDescription``` trait 和一个泛型 ```struct``` ```Shape```，其成员受该 trait 约束
```rust
trait PrintDescription {
    fn print_description(&self);
}
struct Shape<S: PrintDescription> {
    shape: S,
}
// Generic Shape implementation for any type that implements PrintDescription
impl<S: PrintDescription> Shape<S> {
    fn print(&self) {
        self.shape.print_description();
    }
}
```
- [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

# 练习：Trait 约束和泛型

🟡 **中级**
- 实现一个带有泛型成员 ```cipher``` 的 ```struct```，该成员实现了 ```CipherText```
```rust
trait CipherText {
    fn encrypt(&self);
}
// TO DO
//struct Cipher<>

```
- 接下来，在 ```struct``` ```impl``` 上实现名为 ```encrypt``` 的方法，该方法调用 ```cipher``` 上的 ```encrypt```
```rust
// TO DO
impl for Cipher<> {}
```
- 接下来，在两个名为 ```CipherOne``` 和 ```CipherTwo``` 的 struct 上实现 ```CipherText```（只需 ```println()``` 即可）。创建 ```CipherOne``` 和 ```CipherTwo```，并使用 ```Cipher``` 来调用它们

<details><summary>Solution (click to expand)</summary>

```rust
trait CipherText {
    fn encrypt(&self);
}

struct Cipher<T: CipherText> {
    cipher: T,
}

impl<T: CipherText> Cipher<T> {
    fn encrypt(&self) {
        self.cipher.encrypt();
    }
}

struct CipherOne;
struct CipherTwo;

impl CipherText for CipherOne {
    fn encrypt(&self) {
        println!("CipherOne encryption applied");
    }
}

impl CipherText for CipherTwo {
    fn encrypt(&self) {
        println!("CipherTwo encryption applied");
    }
}

fn main() {
    let c1 = Cipher { cipher: CipherOne };
    let c2 = Cipher { cipher: CipherTwo };
    c1.encrypt();
    c2.encrypt();
}
// Output:
// CipherOne encryption applied
// CipherTwo encryption applied
```

</details>

### Rust 类型状态模式与泛型
- Rust 类型可用于在*编译时*强制执行状态机转换
    - 考虑一个具有两种状态的 ```Drone```：```Idle``` 和 ```Flying```。在 ```Idle``` 状态下，唯一允许的方法是 ```takeoff()```。在 ```Flying``` 状态下，我们允许 ```land()```
    
- 一种方法是使用类似以下内容对状态机进行建模
```rust
enum DroneState {
    Idle,
    Flying
}
struct Drone {x: u64, y: u64, z: u64, state: DroneState}  // x, y, z are coordinates
```
- This requires a lot of runtime checks to enforce the state machine semantics — [▶ try it](https://play.rust-lang.org/) to see why

### Rust 类型状态模式泛型
- 泛型允许我们在*编译时*强制执行状态机。这需要使用一个特殊的泛型叫做 ```PhantomData<T>```
- ```PhantomData<T>``` 是一个```零大小```的标记数据类型。在这种情况下，我们使用它来表示 ```Idle``` 和 ```Flying``` 状态，但它具有 ```零``` 运行时大小
- 注意 ```takeoff``` 和 ```land``` 方法将 ```self``` 作为参数。这被称为```消费```（与使用借用的 ```&self``` 对比）。基本上，一旦我们在 ```Drone<Idle>``` 上调用 ```takeoff()```，我们只能返回 ```Drone<Flying>```，反之亦然
```rust
struct Drone<T> {x: u64, y: u64, z: u64, state: PhantomData<T> }
impl Drone<Idle> {
    fn takeoff(self) -> Drone<Flying> {...}
}
impl Drone<Flying> {
    fn land(self) -> Drone<Idle> { ...}
}
```
    - [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

### Rust 类型状态模式泛型
- 关键要点：
    - 状态可以用 struct（零大小）表示
    - 我们可以将状态 ```T``` 与 ```PhantomData<T>```（零大小）组合
    - 为状态机的特定阶段实现方法现在只是 ```impl State<T>``` 的问题
    - 使用消费 ```self``` 的方法从一种状态转换到另一种状态
    - 这为我们提供了```零成本```抽象。编译器可以在编译时强制执行状态机，除非状态正确，否则无法调用方法

### Rust 构建器模式
- 消费 ```self``` 可用于构建器模式
- 考虑一个具有几十个引脚的 GPIO 配置。引脚可以配置为高或低（默认为低）
```rust
#[derive(default)]
enum PinState {
    #[default]
    Low,
    High,
} 
#[derive(default)]
struct GPIOConfig {
    pin0: PinState,
    pin1: PinState
    ... 
}
```
- The builder pattern can be used to construct a GPIO configuration by chaining — [▶ Try it](https://play.rust-lang.org/)


