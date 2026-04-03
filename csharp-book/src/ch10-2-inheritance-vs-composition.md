## 继承与组合

> **你将学到：** 为什么 Rust 没有类继承，traits + structs 如何取代深度的类层次结构，以及通过组合实现多态的实用模式。
>
> **难度：** 🟡 中级

```csharp
// C# - 基于类的继承
public abstract class Animal
{
    public string Name { get; protected set; }
    public abstract void MakeSound();
    
    public virtual void Sleep()
    {
        Console.WriteLine($"{Name} is sleeping");
    }
}

public class Dog : Animal
{
    public Dog(string name) { Name = name; }
    
    public override void MakeSound()
    {
        Console.WriteLine("Woof!");
    }
    
    public void Fetch()
    {
        Console.WriteLine($"{Name} is fetching");
    }
}

// 基于接口的契约
public interface IFlyable
{
    void Fly();
}

public class Bird : Animal, IFlyable
{
    public Bird(string name) { Name = name; }
    
    public override void MakeSound()
    {
        Console.WriteLine("Tweet!");
    }
    
    public void Fly()
    {
        Console.WriteLine($"{Name} is flying");
    }
}
```

### Rust 组合模型
```rust
// Rust - 通过 traits 实现组合优于继承
pub trait Animal {
    fn name(&self) -> &str;
    fn make_sound(&self);
    
    // 默认实现（类似 C# 的虚方法）
    fn sleep(&self) {
        println!("{} is sleeping", self.name());
    }
}

pub trait Flyable {
    fn fly(&self);
}

// 将数据与行为分离
#[derive(Debug)]
pub struct Dog {
    name: String,
}

#[derive(Debug)]
pub struct Bird {
    name: String,
    wingspan: f64,
}

// 为类型实现行为
impl Animal for Dog {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn make_sound(&self) {
        println!("Woof!");
    }
}

impl Dog {
    pub fn new(name: String) -> Self {
        Dog { name }
    }
    
    pub fn fetch(&self) {
        println!("{} is fetching", self.name);
    }
}

impl Animal for Bird {
    fn name(&self) -> &str {
        &self.name
    }
    
    fn make_sound(&self) {
        println!("Tweet!");
    }
}

impl Flyable for Bird {
    fn fly(&self) {
        println!("{} is flying with {:.1}m wingspan", self.name, self.wingspan);
    }
}

// 多个 trait 约束（类似多个接口）
fn make_flying_animal_sound<T>(animal: &T) 
where 
    T: Animal + Flyable,
{
    animal.make_sound();
    animal.fly();
}
```

```mermaid
graph TD
    subgraph "C# 继承层次结构"
        CS_ANIMAL["Animal (abstract class)"]
        CS_DOG["Dog : Animal"]
        CS_BIRD["Bird : Animal, IFlyable"]
        CS_VTABLE["Virtual method dispatch<br/>Runtime cost"]
        CS_COUPLING["[ERROR] 紧耦合<br/>[ERROR] 菱形问题<br/>[ERROR] 深层次结构"]
        
        CS_ANIMAL --> CS_DOG
        CS_ANIMAL --> CS_BIRD
        CS_DOG --> CS_VTABLE
        CS_BIRD --> CS_VTABLE
        CS_ANIMAL --> CS_COUPLING
    end
    
    subgraph "Rust 组合模型"
        RUST_ANIMAL["trait Animal"]
        RUST_FLYABLE["trait Flyable"]
        RUST_DOG["struct Dog"]
        RUST_BIRD["struct Bird"]
        RUST_IMPL1["impl Animal for Dog"]
        RUST_IMPL2["impl Animal for Bird"]
        RUST_IMPL3["impl Flyable for Bird"]
        RUST_STATIC["静态分发<br/>零成本"]
        RUST_FLEXIBLE["[OK] 灵活的组合<br/>[OK] 无层次结构限制<br/>[OK] 混合匹配 traits"]
        
        RUST_DOG --> RUST_IMPL1
        RUST_BIRD --> RUST_IMPL2
        RUST_BIRD --> RUST_IMPL3
        RUST_IMPL1 --> RUST_ANIMAL
        RUST_IMPL2 --> RUST_ANIMAL
        RUST_IMPL3 --> RUST_FLYABLE
        RUST_IMPL1 --> RUST_STATIC
        RUST_IMPL2 --> RUST_STATIC
        RUST_IMPL3 --> RUST_STATIC
        RUST_ANIMAL --> RUST_FLEXIBLE
        RUST_FLYABLE --> RUST_FLEXIBLE
    end
    
    style CS_COUPLING fill:#ffcdd2,color:#000
    style RUST_FLEXIBLE fill:#c8e6c9,color:#000
    style CS_VTABLE fill:#fff3e0,color:#000
    style RUST_STATIC fill:#c8e6c9,color:#000
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：用 Traits 替换继承</strong>（点击展开）</summary>

这段 C# 代码使用了继承。请用 Rust 的 trait 组合重写它：

```csharp
public abstract class Shape { public abstract double Area(); }
public abstract class Shape3D : Shape { public abstract double Volume(); }
public class Cylinder : Shape3D
{
    public double Radius { get; }
    public double Height { get; }
    public Cylinder(double r, double h) { Radius = r; Height = h; }
    public override double Area() => 2.0 * Math.PI * Radius * (Radius + Height);
    public override double Volume() => Math.PI * Radius * Radius * Height;
}
```

要求：
1. `HasArea` trait，带有 `fn area(&self) -> f64`
2. `HasVolume` trait，带有 `fn volume(&self) -> f64`
3. `Cylinder` 结构体实现两者
4. 一个函数 `fn print_shape_info(shape: &(impl HasArea + HasVolume))` —— 注意 trait 约束的组合（不需要继承）

<details>
<summary>🔑 解答</summary>

```rust
use std::f64::consts::PI;

trait HasArea {
    fn area(&self) -> f64;
}

trait HasVolume {
    fn volume(&self) -> f64;
}

struct Cylinder {
    radius: f64,
    height: f64,
}

impl HasArea for Cylinder {
    fn area(&self) -> f64 {
        2.0 * PI * self.radius * (self.radius + self.height)
    }
}

impl HasVolume for Cylinder {
    fn volume(&self) -> f64 {
        PI * self.radius * self.radius * self.height
    }
}

fn print_shape_info(shape: &(impl HasArea + HasVolume)) {
    println!("Area:   {:.2}", shape.area());
    println!("Volume: {:.2}", shape.volume());
}

fn main() {
    let c = Cylinder { radius: 3.0, height: 5.0 };
    print_shape_info(&c);
}
```

**关键洞察**：C# 需要 3 层层次结构（Shape → Shape3D → Cylinder）。Rust 使用扁平的 trait 组合 —— `impl HasArea + HasVolume` 组合能力而不需要继承深度。

</details>
</details>

***

