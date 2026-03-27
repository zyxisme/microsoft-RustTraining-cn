# C# → Rust Training: Merged Chapter Plan

## Source Documents

| Doc | File | Lines |
|-----|------|-------|
| **Bootstrap (B)** | `RustBootstrapForCSharp.md` | 5,363 |
| **Advanced (A)** | `RustTrainingForCSharp.md` | 3,021 |
| **Total raw** | | **8,384** |
| **Estimated merged** | (after dedup) | **~5,800** |

## Mermaid Diagrams Inventory (13 total — all in Advanced doc)

| # | Adv Line | Subject | Target Chapter |
|---|----------|---------|----------------|
| M1 | L84 | Development Model Comparison | ch01 |
| M2 | L173 | Memory Management: GC vs RAII | ch01 |
| M3 | L282 | C# Null Handling Evolution | ch06.1 |
| M4 | L410 | C# Discriminated Unions (Workarounds) | ch06 |
| M5 | L536 | C# Pattern Matching Limitations | ch06.1 |
| M6 | L667 | C# Records — Shallow Immutability | ch03.1 |
| M7 | L829 | Runtime Safety vs Compile-Time Safety | ch07.1 |
| M8 | L998 | C# Inheritance Hierarchy | ch10.2 |
| M9 | L1153 | C# Exception Model | ch09 |
| M10 | L1290 | C# LINQ Characteristics | ch12 |
| M11 | L1463 | C# Generic Constraints | ch10.1 |
| M12 | L2156 | C# Thread Safety Challenges | ch13 |
| M13 | L2850 | Migration Strategy Decision Tree | ch16 |

---

## Chapter Structure

### Chapter 0: Introduction
<!-- ch00: Introduction -->

**File:** `ch00-introduction.md`
**Estimated lines:** ~30
**Content:** Book overview, how to use this guide, prerequisites (C# experience assumed).
**Source:** New content (modeled on C/C++ book ch00 pattern).

---

### Chapter 1: Introduction and Motivation
<!-- ch01: Introduction and Motivation -->

**File:** `ch01-introduction-and-motivation.md`
**Estimated lines:** ~380
**Mermaid diagrams:** M1, M2

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch01.1: Quick Reference --> | B L93–110 | 18 | Quick Reference table — **unique to C# doc, keep verbatim** |
| <!-- ch01.2: Language Philosophy --> | A L70–125 | 56 | C# vs Rust philosophy; includes **M1** |
| <!-- ch01.3: GC vs RAII --> | A L126–214 | 89 | GC vs Ownership overview; includes **M2** |
| <!-- ch01.4: The Case for Rust --> | B L111–221 | 111 | Performance, memory safety arguments |
| <!-- ch01.5: C# Pain Points --> | B L222–348 | 80 | Trim to ~80 lines (null, exceptions, GC pain points — remove overlap with A philosophy already covered in ch01.2–01.3) |
| <!-- ch01.6: When to Choose --> | B L349–400 | 52 | When Rust vs C#, real-world impact |

**Overlap resolution:** Bootstrap "Pain Points" §1 (Null) and §3 (GC) partially overlap with Advanced's Philosophy and GC-vs-RAII. Keep Advanced versions (they have Mermaid diagrams), trim Bootstrap Pain Points to avoid duplication. Pain Point §2 (Hidden Exceptions) is unique — keep fully.

---

### Chapter 2: Getting Started
<!-- ch02: Getting Started -->

**File:** `ch02-getting-started.md`
**Estimated lines:** ~170

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch02.1: Installation --> | B L401–434 | 34 | rustup, tools comparison table |
| <!-- ch02.2: First Program --> | B L435–486 | 52 | Hello World comparison C# vs Rust |
| <!-- ch02.3: Cargo vs NuGet --> | B L487–564 | 78 | Project config, commands, workspace vs solution |

#### Sub-chapter: ch02.1 — Essential Rust Keywords for C# Developers
<!-- ch02.1: Keywords Reference -->

**File:** `ch02-1-keywords-reference.md`
**Estimated lines:** ~400
**Source:** B L842–1244 (403 lines)
**Notes:** This comprehensive keyword mapping table is **unique to the C# doc**. Covers visibility, memory, control flow, type definition, function, variable, pattern matching, and safety keywords — all mapped from C# equivalents. Keep verbatim. The ~400-line size justifies a dedicated sub-chapter.

---

### Chapter 3: Built-in Types
<!-- ch03: Built-in Types -->

**File:** `ch03-built-in-types.md`
**Estimated lines:** ~280

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch03.1: Variables and Mutability --> | B L565–641 | 77 | let vs var, mut, const, shadowing |
| <!-- ch03.2: Primitive Types --> | B L642–707 | 66 | Type comparison table, size types, inference |
| <!-- ch03.3: String Types --> | B L708–782 | 75 | String vs &str, practical examples |
| <!-- ch03.4: Comments and Docs --> | B L783–841 | 59 | Comments, doc comments, rustdoc |

#### Sub-chapter: ch03.1 — True Immutability Deep Dive
<!-- ch03.1: True Immutability -->

**File:** `ch03-1-true-immutability.md`
**Estimated lines:** ~136
**Source:** A L577–712 (136 lines)
**Mermaid diagrams:** M6
**Notes:** C# records "immutability theater" vs Rust true immutability. Includes **M6** (Records — Shallow Immutability diagram). This content is **unique to C# doc** — C# developers need to understand why `record` isn't truly immutable.

---

### Chapter 4: Control Flow
<!-- ch04: Control Flow -->

**File:** `ch04-control-flow.md`
**Estimated lines:** ~280

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch04.1: Functions vs Methods --> | B L1638–1745 | 108 | Declaration, expression vs statement, params/returns |
| <!-- ch04.2: Conditionals --> | B L1748–1792 | 45 | if/else, if-let, ternary equivalents |
| <!-- ch04.3: Loops --> | B L1793–1886 | 93 | loop, while, for, loop control (break/continue labels) |
| <!-- ch04.4: Pattern Matching Preview --> | B L1887–1978 | 35 | Brief intro only (~35 lines trimmed from 92); full treatment in ch06. Add forward reference: "See Chapter 6 for comprehensive coverage." |

**Notes:** The full Pattern Matching Introduction (B L1887–1978, 92 lines) overlaps heavily with ch06. Extract only the basic `match` syntax preview (~35 lines) and forward-reference ch06.

---

### Chapter 5: Data Structures
<!-- ch05: Data Structures -->

**File:** `ch05-data-structures.md`
**Estimated lines:** ~380

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch05.1: Arrays and Slices --> | B L2445–2548 | 104 | C# arrays vs Rust arrays, slices, string slices |
| <!-- ch05.2: Structs vs Classes --> | B L2673–2807 | 135 | Struct definition, creating instances, init patterns |
| <!-- ch05.3: Methods and Associated Functions --> | B L2808–2941 | 134 | impl blocks, &self/&mut self/self, method receiver types |

#### Sub-chapter: ch05.1 — Constructor Patterns
<!-- ch05.1: Constructor Patterns -->

**File:** `ch05-1-constructor-patterns.md`
**Estimated lines:** ~210
**Source:** B L3084–3291 (208 lines)
**Notes:** C# constructors vs Rust `new()` convention, `Default` trait, builder pattern implementation. This is a large self-contained section that warrants its own sub-chapter.

#### Sub-chapter: ch05.2 — Collections: Vec, HashMap, and Iteration
<!-- ch05.2: Collections -->

**File:** `ch05-2-collections.md`
**Estimated lines:** ~390

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch05.2.1: Vec vs List --> | B L2163–2307 | 145 | Creating, initializing, common operations, safe access |
| <!-- ch05.2.2: HashMap vs Dictionary --> | B L2308–2444 | 137 | Operations, entry API, ownership with keys/values |
| <!-- ch05.2.3: Working with Collections --> | B L2549–2672 | 110 | Iteration patterns, IntoIterator/Iter, collecting results (trimmed — LINQ-style iterator content moves to ch12) |

**Overlap note:** The "Working with Collections" section (B L2549–2672) contains some iterator chain content that overlaps with ch12 (Closures/LINQ). Keep basic iteration patterns here, move advanced iterator chains and LINQ comparisons to ch12.

---

### Chapter 6: Enums and Pattern Matching
<!-- ch06: Enums and Pattern Matching -->

**File:** `ch06-enums-and-pattern-matching.md`
**Estimated lines:** ~320
**Mermaid diagrams:** M4

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch06.1: C# Enum Limitations --> | B L3296–3342 | 47 | Why C# enums are limited |
| <!-- ch06.2: Rust Enum Power --> | B L3343–3378 | 36 | Enum variants with data |
| <!-- ch06.3: Algebraic Data Types --> | A L319–451 | 100 | ADTs vs C# unions; includes **M4**. Trim from 133 to ~100 (remove overlap with basic enum coverage above) |
| <!-- ch06.4: Pattern Matching --> | B L3379–3461 | 83 | Match expressions, destructuring |
| <!-- ch06.5: Guards and Advanced --> | B L3462–3502 | 41 | Match guards, nested patterns |

#### Sub-chapter: ch06.1 — Exhaustive Matching and Null Safety
<!-- ch06.1: Exhaustive Matching and Null Safety -->

**File:** `ch06-1-exhaustive-matching-and-null-safety.md`
**Estimated lines:** ~300
**Mermaid diagrams:** M3, M5

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch06.1.1: Exhaustive Matching --> | A L452–576 | 125 | Compiler guarantees vs runtime errors; includes **M5** |
| <!-- ch06.1.2: Null Safety: Option --> | A L215–318 | 80 | Nullable<T> vs Option<T>; includes **M3**. Trim from 104 to ~80 (remove overlap with B's Option section) |
| <!-- ch06.1.3: Option and Result --> | B L3503–3615 | 113 | Option<T> and Result<T,E> practical usage |

**Overlap resolution:** Both docs cover Option<T>. Advanced version (A L215–318) has the Mermaid diagram and deeper "evolution of null handling" narrative — use as the conceptual intro. Bootstrap version (B L3503–3615) has practical code examples — keep for the hands-on portion. Deduplicate overlapping examples.

---

### Chapter 7: Ownership and Borrowing
<!-- ch07: Ownership and Borrowing -->

**File:** `ch07-ownership-and-borrowing.md`
**Estimated lines:** ~330

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch07.1: C# Memory Model --> | B L1249–1267 | 19 | C# reference types, GC review |
| <!-- ch07.2: Ownership Rules --> | B L1268–1316 | 49 | Three rules, Move for C# developers, Copy vs Move |
| <!-- ch07.3: Practical Examples --> | B L1317–1348 | 32 | Swapping values example |
| <!-- ch07.4: Borrowing --> | B L1349–1472 | 124 | Shared/mutable refs, borrowing rules, ref safety comparison |
| <!-- ch07.5: Move Semantics --> | B L1540–1637 | 98 | Value/reference types vs move semantics, avoiding moves |

#### Sub-chapter: ch07.1 — References, Pointers, and Memory Safety
<!-- ch07.1: Memory Safety Deep Dive -->

**File:** `ch07-1-references-pointers-and-memory-safety.md`
**Estimated lines:** ~220
**Mermaid diagrams:** M7

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch07.1.1: References vs Pointers --> | B L1473–1539 | 67 | C# unsafe pointers vs Rust safe references, lifetime basics |
| <!-- ch07.1.2: Memory Safety --> | A L713–870 | 158 | Runtime checks vs compile-time proofs; includes **M7**. This is the deepest treatment of why Rust's ownership prevents entire bug categories — **unique depth for C# audience** |

---

### Chapter 8: Crates and Modules
<!-- ch08: Crates and Modules -->

**File:** `ch08-crates-and-modules.md`
**Estimated lines:** ~340

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch08.1: Modules vs Namespaces --> | B L3674–3882 | 209 | C# namespace → Rust module mapping, hierarchy, visibility, file organization |
| <!-- ch08.2: Crates vs Assemblies --> | B L3883–4009 | 127 | Assembly model vs crate model, crate types, workspace vs solution |

#### Sub-chapter: ch08.1 — Package Management Deep Dive
<!-- ch08.1: Package Management -->

**File:** `ch08-1-package-management.md`
**Estimated lines:** ~235

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch08.1.1: Dependencies --> | B L4010–4055 | 46 | Cargo.toml vs .csproj, dependency types |
| <!-- ch08.1.2: Version Management --> | B L4056–4089 | 34 | Semantic versioning, Cargo.lock |
| <!-- ch08.1.3: Package Sources --> | B L4090–4132 | 43 | crates.io vs NuGet, alternative registries |
| <!-- ch08.1.4: Features --> | B L4133–4182 | 50 | Feature flags vs #if DEBUG conditional compilation |
| <!-- ch08.1.5: External Crates --> | B L4183–4244 | 62 | Popular crate list, HTTP client migration example |

---

### Chapter 9: Error Handling
<!-- ch09: Error Handling -->

**File:** `ch09-error-handling.md`
**Estimated lines:** ~350
**Mermaid diagrams:** M9

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch09.1: C# Exception Model --> | A L1046–1089 | 44 | Exception-based handling, problems; part of **M9** context |
| <!-- ch09.2: Exceptions vs Result --> | A L1090–1194 | 105 | Result-based error handling (Advanced version — deeper, with Mermaid **M9**) |
| <!-- ch09.3: The ? Operator --> | B L2057–2084 | 28 | ? operator explained as "like C#'s await" |
| <!-- ch09.4: Custom Error Types --> | B L3616–3673 | 58 | thiserror-based custom errors (moved from Enums chapter) |
| <!-- ch09.5: Error Handling Deep Dive --> | B L4558–4715 | 120 | Comprehensive error handling patterns (trim from 158 — remove overlap with A's Result coverage above) |

**Overlap resolution:** Three sources cover error handling:
1. **B L1979–2162 "Error Handling Basics"** (184 lines) — introductory
2. **B L4558–4715 "Error Handling Deep Dive"** (158 lines) — advanced patterns
3. **A L1046–1194 "Exceptions vs Result"** (149 lines) — conceptual comparison with Mermaid

**Strategy:** Use A's version for the conceptual framing (it has M9 diagram and deeper C# comparison). Use B Deep Dive for practical patterns. Drop B Basics (it's redundant with the combination of A + B Deep Dive). Keep ? operator explanation from B Basics since it's uniquely well-explained there.

#### Sub-chapter: ch09.1 — Error Handling Best Practices
<!-- ch09.1: Error Handling Best Practices -->

**File:** `ch09-1-error-handling-best-practices.md`
**Estimated lines:** ~80
**Source:** Extracted from B L4612–4715 (practical patterns not covered in main ch09), plus A L2916–2938 (error handling strategy from Best Practices section).
**Notes:** Covers when to use `anyhow` vs `thiserror`, error conversion patterns, error context chaining. Following the C/C++ book pattern of ch09 + ch09.1.

---

### Chapter 10: Traits and Generics
<!-- ch10: Traits and Generics -->

**File:** `ch10-traits.md`
**Estimated lines:** ~380

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch10.1: Traits vs Interfaces --> | B L4245–4383 | 139 | Definition, implementation, C# interface comparison |
| <!-- ch10.2: Implementing Behavior --> | B L2942–3083 | 100 | Trait implementation on structs, multiple impls (trim from 142 — remove overlap with ch10.1) |
| <!-- ch10.3: Trait Objects --> | B L4385–4443 | 59 | Dynamic dispatch, dyn Trait, Box<dyn Trait> |
| <!-- ch10.4: Derived Traits --> | B L4444–4491 | 48 | #[derive], common derivable traits |
| <!-- ch10.5: Std Library Traits --> | B L4492–4557 | 40 | Display, Debug, Clone, Iterator (trim — From/Into moves to ch11) |

#### Sub-chapter: ch10.1 — Generics and Constraints
<!-- ch10.1: Generics -->

**File:** `ch10-1-generics.md`
**Estimated lines:** ~170
**Source:** A L1338–1505 (168 lines)
**Mermaid diagrams:** M11
**Notes:** C# `where T : class` vs Rust trait bounds, monomorphization, associated types. Includes **M11** (Generic Constraints diagram). The Advanced doc's treatment is significantly deeper than what Bootstrap covers.

#### Sub-chapter: ch10.2 — Inheritance vs Composition
<!-- ch10.2: Inheritance vs Composition -->

**File:** `ch10-2-inheritance-vs-composition.md`
**Estimated lines:** ~175
**Source:** A L871–1045 (175 lines)
**Mermaid diagrams:** M8
**Notes:** C# inheritance hierarchy vs Rust composition model. Includes **M8** (Inheritance Hierarchy diagram). **Unique and valuable for C# developers** who must unlearn class hierarchies. Covers: trait objects as polymorphism, newtype pattern, delegation.

---

### Chapter 11: From and Into Traits
<!-- ch11: From and Into Traits -->

**File:** `ch11-from-and-into-traits.md`
**Estimated lines:** ~120

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch11.1: From/Into Basics --> | B L4492–4530 | 40 | From<T> implementation, automatic Into<T> (extracted from Std Library Traits section) |
| <!-- ch11.2: Conversion Patterns --> | New | 40 | C# implicit/explicit operators vs From/Into, TryFrom/TryInto |
| <!-- ch11.3: Error Conversions --> | B L4617–4650 | 30 | From<E> for error type conversions (extracted from Error Handling Deep Dive) |
| <!-- ch11.4: Practical Examples --> | New | 10 | String conversions, numeric type conversions |

**Notes:** Neither source doc has an explicit From/Into chapter. Content is assembled from Bootstrap's Std Library Traits section (From/Into examples) and Error Handling (From for error conversion). Some new bridging content needed for C# implicit/explicit cast operator mapping. Smaller chapter (~120 lines) but follows C/C++ book structure for cross-book consistency.

---

### Chapter 12: Closures and Iterators
<!-- ch12: Closures and Iterators -->

**File:** `ch12-closures-and-iterators.md`
**Estimated lines:** ~300
**Mermaid diagrams:** M10

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch12.1: Closures --> | New | 60 | C# lambda expressions vs Rust closures, Fn/FnMut/FnOnce traits, capture semantics (C# developers know lambdas well — focus on ownership differences) |
| <!-- ch12.2: LINQ vs Iterators --> | A L1195–1337 | 143 | Comprehensive LINQ-to-Iterator mapping; includes **M10**. **Unique and high-value for C# developers** |
| <!-- ch12.3: Advanced Iteration --> | B L2595–2672 | 78 | Iterator/IntoIterator/Iter distinction, collecting results (moved from ch05 Working with Collections — the advanced iteration content) |

**Notes:** The C/C++ books have "Closures" as ch12. For C# developers, closures themselves are familiar (they use lambdas daily), so the focus shifts to: (1) how Rust closures differ (ownership capture), and (2) the LINQ-to-Iterator mapping which is the killer content. The Advanced doc's LINQ section is excellent and unique.

---

### Chapter 13: Concurrency
<!-- ch13: Concurrency -->

**File:** `ch13-concurrency.md`
**Estimated lines:** ~260
**Mermaid diagrams:** M12

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch13.1: Thread Safety --> | A L1947–2155 | 209 | Convention vs type system guarantees, Send/Sync, Arc/Mutex, channels; includes **M12** |
| <!-- ch13.2: Async Comparison --> | A L2156–2204 | 49 | Rust async/await vs C# async/await, tokio runtime |

**Notes:** Entirely from Advanced doc. The Advanced doc's Thread Safety section is comprehensive and includes the M12 Mermaid diagram showing C# thread safety challenges. The async comparison naturally follows. No Bootstrap content needed here (Bootstrap doesn't cover concurrency).

---

### Chapter 14: Unsafe Rust and FFI
<!-- ch14: Unsafe Rust and FFI -->

**File:** `ch14-unsafe-rust-and-ffi.md`
**Estimated lines:** ~120

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch14.1: Unsafe Blocks --> | New | 50 | C# `unsafe` keyword vs Rust `unsafe` blocks, what unsafe permits, safety invariants |
| <!-- ch14.2: FFI Basics --> | New | 40 | C# P/Invoke + COM Interop vs Rust FFI (`extern "C"`), bindgen |
| <!-- ch14.3: When to Use Unsafe --> | New | 30 | Guidelines, unsafe abstractions with safe APIs |

**Notes:** Neither source doc has explicit unsafe/FFI content (the Advanced ToC mentions it but the sections were never written). This chapter needs new content. For C# developers, the key mappings are: `unsafe {}` blocks, P/Invoke → `extern "C"`, COM Interop → FFI bindings. Keep concise since this is less commonly needed by C# developers transitioning to Rust.

---

### Chapter 15: Case Studies and Practical Migration
<!-- ch15: Case Studies -->

**File:** `ch15-case-studies.md`
**Estimated lines:** ~400

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch15.1: Config Management --> | B L4720–4854 | 135 | C# IConfiguration → Rust config crate migration |
| <!-- ch15.2: Data Processing --> | B L4855–5039 | 185 | LINQ pipeline → Rust iterator pipeline |
| <!-- ch15.3: HTTP Client --> | B L5040–5218 | 80 | HttpClient → reqwest migration (trim from 179 — remove overlap with Essential Crates UserService example in ch15.2) |

#### Sub-chapter: ch15.1 — Common Patterns and Essential Crates
<!-- ch15.1: Common Patterns and Essential Crates -->

**File:** `ch15-1-common-patterns-and-essential-crates.md`
**Estimated lines:** ~400

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch15.1.1: Repository Pattern --> | A L1506–1625 | 120 | C# repository → Rust trait-based repository |
| <!-- ch15.1.2: Builder Pattern --> | A L1626–1743 | 118 | C# builder → Rust builder with consuming self |
| <!-- ch15.1.3: Essential Crates --> | A L1744–1946 | 160 | **Unique to C# doc.** Cargo.toml template mapping every C# library to Rust equivalent (serde↔Json, reqwest↔HttpClient, tokio↔Task, thiserror↔Exception, sqlx↔EF, etc.) + full UserService example. Trim from 203 to ~160 (remove overlap with ch15 HTTP client) |

#### Sub-chapter: ch15.2 — Adoption Strategy and Concept Mapping
<!-- ch15.2: Adoption Strategy -->

**File:** `ch15-2-adoption-strategy.md`
**Estimated lines:** ~390

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch15.2.1: Concept Mapping --> | A L2428–2595 | 168 | **Unique and high-value.** DI → trait injection, LINQ → iterator chains, EF → SQLx, IConfiguration → config crate. Each with side-by-side C#/Rust code |
| <!-- ch15.2.2: Incremental Adoption --> | A L2205–2427 | 120 | Phase 1/2/3 adoption strategy (trim from 223 — remove overlap with Essential Crates and Concept Mapping) |
| <!-- ch15.2.3: Team Timeline --> | A L2596–2708 | 100 | Month 1/2/3+ timeline with concrete milestones (trim from 113 — remove overlap with adoption phases) |

---

### Chapter 16: Best Practices
<!-- ch16: Best Practices -->

**File:** `ch16-best-practices.md`
**Estimated lines:** ~340
**Mermaid diagrams:** M13

| Sub-section marker | Source | Lines | Notes |
|---|---|---|---|
| <!-- ch16.1: Mindset Shifts --> | A L2886–2891 | 6 | Key mental model changes |
| <!-- ch16.2: Code Organization --> | A L2892–2915 | 24 | Project structure recommendations |
| <!-- ch16.3: Testing Patterns --> | A L2939–2974 | 36 | #[test], #[cfg(test)], integration tests |
| <!-- ch16.4: Common Mistakes --> | A L2975–3021 | 47 | Inheritance attempts, unwrap abuse, excessive clone, RefCell overuse |
| <!-- ch16.5: Performance Comparison --> | A L2709–2883 | 130 | Managed vs native perf characteristics, benchmarks, CPU workloads, decision criteria; includes **M13** (Migration Strategy Decision Tree). Trim from 175 to ~130 (remove overlap with ch01 "When to Choose") |
| <!-- ch16.6: Common Pitfalls --> | B L5288–5363 | 76 | Ownership confusion, borrow checker fights, expecting null |

#### Sub-chapter: ch16.1 — Learning Path and Resources
<!-- ch16.1: Learning Path -->

**File:** `ch16-1-learning-path.md`
**Estimated lines:** ~100
**Source:** B L5219–5287 (69 lines) + curated subset of B L5269–5287 (resources)
**Notes:** Week-by-week and month-by-month learning plan. Books, online resources, practice projects. Trim from 145 to ~100 (the timeline content overlaps with ch15.2 Team Timeline).

---

## SUMMARY.md (mdbook format)

```markdown
# Summary

[Introduction](ch00-introduction.md)

---

- [1. Introduction and Motivation](ch01-introduction-and-motivation.md)
- [2. Getting Started](ch02-getting-started.md)
    - [Keywords Reference](ch02-1-keywords-reference.md)
- [3. Built-in Types](ch03-built-in-types.md)
    - [True Immutability Deep Dive](ch03-1-true-immutability.md)
- [4. Control Flow](ch04-control-flow.md)
- [5. Data Structures](ch05-data-structures.md)
    - [Constructor Patterns](ch05-1-constructor-patterns.md)
    - [Collections: Vec, HashMap, and Iteration](ch05-2-collections.md)
- [6. Enums and Pattern Matching](ch06-enums-and-pattern-matching.md)
    - [Exhaustive Matching and Null Safety](ch06-1-exhaustive-matching-and-null-safety.md)
- [7. Ownership and Borrowing](ch07-ownership-and-borrowing.md)
    - [References, Pointers, and Memory Safety](ch07-1-references-pointers-and-memory-safety.md)
- [8. Crates and Modules](ch08-crates-and-modules.md)
    - [Package Management Deep Dive](ch08-1-package-management.md)
- [9. Error Handling](ch09-error-handling.md)
    - [Error Handling Best Practices](ch09-1-error-handling-best-practices.md)
- [10. Traits and Generics](ch10-traits.md)
    - [Generics](ch10-1-generics.md)
    - [Inheritance vs Composition](ch10-2-inheritance-vs-composition.md)
- [11. From and Into Traits](ch11-from-and-into-traits.md)
- [12. Closures and Iterators](ch12-closures-and-iterators.md)
- [13. Concurrency](ch13-concurrency.md)
- [14. Unsafe Rust and FFI](ch14-unsafe-rust-and-ffi.md)
- [15. Case Studies](ch15-case-studies.md)
    - [Common Patterns and Essential Crates](ch15-1-common-patterns-and-essential-crates.md)
    - [Adoption Strategy and Concept Mapping](ch15-2-adoption-strategy.md)
- [16. Best Practices](ch16-best-practices.md)
    - [Learning Path and Resources](ch16-1-learning-path.md)
```

---

## Overlap Resolution Summary

| Overlapping Topic | Bootstrap Source | Advanced Source | Resolution |
|---|---|---|---|
| **Option/Null Safety** | B L2085–2133, B L3503–3615 | A L215–318 (M3) | Use A for conceptual intro (has Mermaid). Use B L3503–3615 for practical examples. Drop B L2085–2133 (redundant). → ch06.1 |
| **Error Handling** | B L1979–2162 (basics), B L4558–4715 (deep) | A L1046–1194 (M9) | Use A for conceptual framing (has Mermaid). Use B deep dive for patterns. Drop B basics (redundant). → ch09 |
| **Pattern Matching** | B L1887–1978 (intro), B L3379–3502 (full) | A L452–576 (M5) | Brief preview in ch04 (~35 lines from B intro). Full coverage in ch06 from B L3379+. Advanced exhaustive matching from A. → ch04, ch06 |
| **Traits/Interfaces** | B L4245–4557 (full), B L2942–3083 (impl) | A L871–1045 (inheritance, M8) | B for trait mechanics (ch10 main). A for inheritance-vs-composition philosophy (ch10.2). Merge B impl section into ch10 main. |
| **GC vs Ownership** | B L222–270 (pain point) | A L126–214 (M2) | Use A (has Mermaid). Trim B pain point to avoid duplication. → ch01 |
| **Philosophy/Motivation** | B L111–400 (case + pain points) | A L70–125 (M1) | Use A for deep philosophy (has Mermaid). Use B for practical motivation args. → ch01 |
| **Collections/Iteration** | B L2549–2672 (working with) | A L1195–1337 (LINQ, M10) | Basic iteration in ch05.2. LINQ comparison in ch12 from A. Advanced iteration from B moves to ch12. |

---

## Estimated Line Counts by Chapter

| Chapter | Main | Sub-chapters | Total |
|---------|------|-------------|-------|
| ch00 Introduction | 30 | — | 30 |
| ch01 Intro & Motivation | 380 | — | 380 |
| ch02 Getting Started | 170 | ch02.1 Keywords (400) | 570 |
| ch03 Built-in Types | 280 | ch03.1 Immutability (136) | 416 |
| ch04 Control Flow | 280 | — | 280 |
| ch05 Data Structures | 380 | ch05.1 Constructors (210) + ch05.2 Collections (390) | 980 |
| ch06 Enums & Matching | 320 | ch06.1 Exhaustive/Null (300) | 620 |
| ch07 Ownership | 330 | ch07.1 Memory Safety (220) | 550 |
| ch08 Crates & Modules | 340 | ch08.1 Pkg Mgmt (235) | 575 |
| ch09 Error Handling | 350 | ch09.1 Best Practices (80) | 430 |
| ch10 Traits & Generics | 380 | ch10.1 Generics (170) + ch10.2 Inheritance (175) | 725 |
| ch11 From/Into | 120 | — | 120 |
| ch12 Closures & Iterators | 300 | — | 300 |
| ch13 Concurrency | 260 | — | 260 |
| ch14 Unsafe & FFI | 120 | — | 120 |
| ch15 Case Studies | 400 | ch15.1 Patterns/Crates (400) + ch15.2 Adoption (390) | 1,190 |
| ch16 Best Practices | 340 | ch16.1 Learning Path (100) | 440 |
| **TOTAL** | | | **~7,986** |

**Reduction from raw total:** 8,384 → ~5,800 unique content (after dedup) + ~120 new content (ch11 bridging, ch14 new) ≈ **5,920 lines of merged output** spread across 16 chapters + 14 sub-chapters.

---

## Unique C#-Specific Content Preserved

| Content | Source | Chapter | Why It Matters |
|---------|--------|---------|----------------|
| Quick Reference Table | B L93–110 | ch01 | At-a-glance C#→Rust mapping |
| Keywords Reference (400 lines) | B L842–1244 | ch02.1 | Comprehensive C# keyword → Rust mapping |
| True Immutability vs Records | A L577–712 | ch03.1 | C# `record` isn't truly immutable |
| 13 × Mermaid Diagrams | A various | various | Visual concept comparisons |
| LINQ vs Iterators | A L1195–1337 | ch12 | Maps every LINQ method to Rust |
| DI → Trait Injection | A L2430–2478 | ch15.2 | IServiceCollection → generic constructors |
| EF → SQLx Mapping | A L2514–2555 | ch15.2 | DbContext → sqlx::query_as! |
| IConfiguration → config | A L2556–2595 | ch15.2 | appsettings.json → config crate |
| Essential Crates Mapping | A L1744–1946 | ch15.1 | Every C# lib → Rust crate equivalent |
| Repository Pattern | A L1506–1625 | ch15.1 | IRepository → trait + async_trait |
| Builder Pattern | A L1626–1743 | ch15.1 | C# builder → consuming-self builder |
| Thread Safety Guarantees | A L1947–2204 | ch13 | Convention → type system enforcement |
| Migration Decision Tree | A L2850–2883 | ch16 | Mermaid flowchart for adoption decisions |
| Performance Benchmarks | A L2709–2830 | ch16 | Managed vs native perf data |
| Team Adoption Timeline | A L2596–2708 | ch15.2 | Month-by-month rollout plan |
