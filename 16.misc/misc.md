# Miscellaneous Topics

- [Overview](#overview)
- [Trait Objects](#trait-objects)
  - [Static Dispatch](#static-dispatch)
  - [Dynamic Dispatch](#dynamic-dispatch)
  - [Object Safety](#object-safety)
  - [Trait Objects & Performance](#trait-object--performance)
- [Unsafe Rust](#)
  - [The Unsafe Superpowers](#the-unsafe-superpowers)
  - [Raw Pointers](#raw-pointers)
  - [Calling Unsafe Functions](#calling-unsafe-functions)
  - [Creating Safe Abstractions over Unsafe Code](#creating-safe-abstractions-over-unsafe-code)
  - [Interfacing with Foreign Code](#interfacing-with-foreign-code)
  - [Accessing or Modifying Mutable Static Variables](#accessing-or-modifying-mutable-static-variables)
  - [Implementing Unsafe Traits](#implementing-unsafe-traits)
  - [Accessing Fields of `union`s](#accessing-fields-of-unions)
- [Advanced Traits](#advanced-traits)
  - [Associated Types](#associated-types)
  - [Default Generic Type Parameters](#default-generic-type-parameters)
  - [Operator Overloading](#operator-overloading)
  - [Fully Qualified Syntax](#fully-qualified-syntax)
  - [Supertraits](#supertraits)
- [Advanced Types](#advanced-types)
  - [The Newtype Pattern](#the-newtype-pattern)
  - [Type Aliases](#type-aliases)
  - [The Never Type (`!`)](#the-never-type-)
  - [Dynamically Sized Types (DSTs)](#dynamically-sized-types-dsts)
  - [The `Sized` Trait](#the-sized-trait)
- [Advanced Functions and Closures](#advanced-functions-and-closures)
  - [Function Pointers](#function-pointers)
  - [Function Pointers vs Closures](#function-pointers-vs-closures)
  - [Returning Closures](#returning-closures)
  - [Returning Multiple Closure Types](#returning-multiple-closure-types)
- [Macros](#macros)
  - [Declarative vs Procedural Macros](#declarative-vs-procedural-macros)
  - [Why Macros Are So Powerful](#why-macros-are-so-powerful)
  - [Learning Resources](#learning-resources)

## Overview

In this file, I will explore a few topics that I couldn't fit anywhere else and aren't complex enough to warrant their own dedicated file.

## Trait Objects

Trait objects are Rust's mechanism for runtime polymorphism. They allow a value to be treated as "some type that implements a particular trait" without knowing the concrete type at compile time. Unlike generics, which use **_static dispatch_** and generate specialized code for each concrete type, trait objects use **_dynamic dispatch_**, meaning the method to call is determined at runtime.

Consider the following:

```rust
trait Draw {
    fn draw(&self);
}

// Multiple types implement this
struct Button;
struct TextBox;

impl Draw for Button {
    fn draw(&self) {
        // Implementation specific to this type
        println!("Drawing a button");
    }
}

impl Draw for TextBox {
    fn draw(&self) {
        // Implementation specific to this type
        println!("Drawing a text box");
    }
}
```

Instead of working with a specific concrete type, a trait object works with any value implementing the trait.

```rust
// This could be either Button or TextBox
let widget: Box<dyn Draw> = Box::new(Button);
widget.draw();

// Or even this is possible
let widgets: Vec<Box<dyn Draw>> = vec![
    Box::new(Button),
    Box::new(TextBox),
    Box::new(TextBox),
    Box::new(Button),
    Box::new(Button),
    Box::new(Button),
    Box::new(TextBox),
];

for widget in widgets {
    widget.draw();
}
```

Here, `dyn Draw` means "some type that implements `Draw`." The concrete type is hidden, leaving only the trait's interface.

If you paid attention, in the aforementioned example our trait objects were boxed. This is because of the fact that trait objects are Dynamically Sized Types (DSTs) and their size is unknown at compile time; So, they have to be behind a pointer such as Box, &, Rc, or Arc.

Essentially, trait objects are fat pointers that contain two machine-word-sized pointers; one is a data pointer that points to the actual object; and the other is a pointer to a vtable, which contains metadata for the concrete type, including pointers to the implementations of the trait's methods.

Trait objects are most useful when the concrete type is not known until runtime or when multiple different types need to be treated uniformly. Common use cases include collections containing varied types (such as `Vec<Box<dyn Draw>>`), extensibility systems, event handlers and callbacks, and APIs that operate on any implementation of a trait without requiring the caller to use generics.

### Static Dispatch

Static dispatch is a form of polymorphism where the compiler determines the exact function or method to call at compile time. Rust achieves this through generics and _monomorphization_, generating specialized code for each concrete type. Because the target is known ahead of time, the compiler can _inline_ function calls and perform aggressive optimizations.

### Dynamic Dispatch

Dynamic dispatch determines the function or method to call at runtime. Trait objects (`dyn Trait`) use dynamic dispatch by storing a pointer to a vtable, which contains the implementations of the trait's methods for the underlying concrete type (data pointer). When a method is called, Rust looks it up in the vtable and invokes the appropriate implementation.

### Object Safety

Not every trait can be used as a trait object. A trait must be _object-safe_, meaning its methods must be callable without knowing the concrete implementing type.

Object safety is the set of rules that determines whether a trait can be used as a trait object (`dyn Trait`). The underlying principle is that every method in the trait must be callable without knowing the concrete implementing type. Since a trait object _"hides"_ the concrete type and only retains a pointer to the value and its vtable, methods cannot rely on information that has been _"hidden"_.

For example, our `Draw` trait from earlier examples is object-safe because its method can be called through the vtable without needing to know the concrete type:

In contrast, the following trait is not object-safe:

```rust
trait Cloneable {
    fn clone(&self) -> Self;
}
```

The problem is that the return type is `Self`. Once a value has been hidden into a trait object (`dyn Cloneable`), Rust no longer knows what `Self` refers to, so it cannot determine what type `clone()` should return.

Similarly, traits with generic methods are not object-safe because a vtable contains a fixed set of function pointers and cannot accommodate every possible instantiation of a generic method. As a rule of thumb, a trait is generally object-safe if its methods do not return `Self` and are not themselves generic. These restrictions ensure that every method can be invoked through a trait object using only the information stored in its vtable.

In short, a trait is object-safe if it satisfies the following:

- The methods take `self`, `&self` or `&mut self`
- Don't return `Self`
- Are not generic

### Trait Object & Performance

Trait objects incur a small runtime cost because every method call requires an indirect lookup through the vtable instead of a direct function call. Additionally, since the compiler cannot know the concrete type, it generally cannot inline trait method calls or perform some optimizations that are possible with static dispatch. This overhead is usually negligible unless the method is called extremely frequently in performance-critical code.

## Unsafe Rust

Rust's ownership model and borrow checker eliminate many classes of bugs at compile time. However, some low-level programming tasks cannot be expressed within Rust's normal safety guarantees. For these situations, Rust provides **Unsafe Rust**, a restricted opt-out from some of the compiler's safety checks.

Unsafe Rust is primarily intended for implementing low-level abstractions, interacting with foreign code, and performing operations that the compiler cannot verify as safe. Most Rust code should remain entirely safe, while unsafe code is typically isolated to small, well-thought-out sections.

Importantly, **`unsafe` does not disable the borrow checker or turn off Rust's safety guarantees entirely**. It only grants permission to perform a handful of operations that are otherwise rejected by the compiler.

### The Unsafe Superpowers

An `unsafe` block grants exactly five additional capabilities:

- Dereference raw pointers.
- Call unsafe functions or methods.
- Access or modify mutable static variables.
- Implement unsafe traits.
- Access fields of `union`s.

These capabilities are commonly referred to as Unsafe Rust's **five superpowers**.

```rust
unsafe {
    // Unsafe operations are allowed here.
}
```

Marking a block as `unsafe` does **not** make everything inside it unsafe. Safe operations remain safe, and only the specific unsafe operations require the `unsafe` block.

### Raw Pointers

Raw pointers (`*const T` and `*mut T`) are similar to references but are not subject to Rust's borrowing rules. Unlike references, raw pointers may be null, dangling, alias mutable data, and do not guarantee valid memory.

Creating a raw pointer is safe, but **dereferencing** one is unsafe because the compiler cannot verify that it points to valid memory.

```rust
let x = 5;
let ptr = &x as *const i32;

unsafe {
    println!("{}", *ptr);
}
```

Raw pointers are commonly used when interfacing with C libraries, implementing data structures, or working directly with memory.

### Calling Unsafe Functions

Some functions are marked `unsafe` because the compiler cannot enforce their safety requirements. The caller must guarantee that the function's documented preconditions are satisfied.

Declaring a function as `unsafe` shifts the responsibility for maintaining its safety invariants to its callers.

```rust
unsafe fn dangerous() {}

unsafe {
    dangerous();
}
```

### Creating Safe Abstractions over Unsafe Code

One of Rust's core design principles is to keep unsafe code as small and localized as possible. Unsafe operations are often wrapped inside safe APIs that enforce the necessary invariants internally.

This allows the rest of a program to remain entirely safe while still benefiting from low-level implementations hidden behind well-designed interfaces.

### Interfacing with Foreign Code

Rust can call functions written in other languages, such as C, through the **Foreign Function Interface (FFI)**. Foreign functions are declared inside an `extern` block.

Because Rust cannot verify the behavior of foreign code, calling these functions is unsafe.

```rust
unsafe extern "C" {
    fn abs(input: i32) -> i32;
}

unsafe {
    println!("{}", abs(-5));
}
```

### Accessing or Modifying Mutable Static Variables

A `static` variable has a fixed memory location for the duration of the program. Immutable statics are always safe, but mutable statics (`static mut`) are unsafe because simultaneous access from multiple threads can cause data races.

Whenever possible, synchronization primitives such as `Mutex` or atomic types should be preferred over `static mut`.

```rust
static mut COUNTER: u32 = 0;

unsafe {
    COUNTER += 1;
}
```

### Implementing Unsafe Traits

Some traits are marked `unsafe` because incorrect implementations can violate Rust's safety guarantees.

Marking a trait or implementation as `unsafe` indicates that the implementer must uphold additional invariants beyond what the compiler can verify.

```rust
unsafe trait MyTrait {}

unsafe impl MyTrait for MyType {}
```

### Accessing Fields of `union`s

A `union` allows multiple fields to occupy the same memory location. Since Rust cannot know which field currently contains a valid value, reading a union field is unsafe.

Unions are primarily used for low-level systems programming and interoperability with C.

```rust
union Value {
    i: u32,
    f: f32,
}

let value = Value { i: 42 };

unsafe {
    println!("{}", value.i);
}
```

## Advanced Traits

Rust's trait system is powerful enough to express much more than shared behavior. In addition to defining methods, traits can specify associated types, default generic parameters, relationships with other traits, and even provide mechanisms for resolving naming conflicts.

These features are considered "advanced" not because they are difficult, but because they are needed less frequently. Nevertheless, they are fundamental building blocks used throughout the standard library and many third-party crates.

### Associated Types

An **associated type** is a placeholder type defined as part of a trait. Instead of making the trait itself generic, the implementer specifies what concrete type the placeholder represents.

```rust
trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;
}
```

When implementing the trait, the associated type is defined once.

```rust
struct Counter;

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<Self::Item> {
        // ...
    }
}
```

Associated types make traits easier to use because a type can implement the trait only once for a given associated type, eliminating the ambiguity that would arise with a generic trait. They also make the associated type part of the trait's contract.

### Default Generic Type Parameters

Generic parameters can have **default types**, allowing implementations to omit the generic argument when the default is appropriate.

```rust
trait Add<Rhs = Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}
```

The `std::ops::Add` trait uses this feature so that, by default, the right-hand operand has the same type as the left-hand operand.

Default generic parameters are commonly used to reduce boilerplate, preserve backward compatibility, support operator overloading.

```rust
impl Add for Point {
    type Output = Point;

    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}
```

### Operator Overloading

Rust allows certain operators (such as `+`, `-`, `*`, and `==`) to be overloaded by implementing traits from `std::ops`.

For example, implementing the `Add` trait defines the behavior of the `+` operator for a type.

```rust
use std::ops::Add;

impl Add for Point {
    type Output = Point;

    fn add(self, rhs: Point) -> Point {
        Point {
            x: self.x + rhs.x,
            y: self.y + rhs.y,
        }
    }
}
```

Rust intentionally limits operator overloading to predefined operators and traits; you cannot invent entirely new operators.

### Fully Qualified Syntax

Sometimes multiple traits (or a trait and the type itself) define methods with the same name. In these situations, Rust needs help determining which implementation you intend to call.

Suppose two traits both define `fly()`.

```rust
Pilot::fly(&person);
Wizard::fly(&person);

person.fly();
```

The first two explicitly call the trait implementations, while the last calls the method defined directly on the type of `&person`.

For associated functions (those without a `self` parameter), even more explicit syntax may be required.

```rust
<Dog as Animal>::baby_name()
```

This is known as **Fully Qualified Syntax**, and its general form is:

```rust
<Type as Trait>::function(arguments...)
```

Most of the time Rust can infer the correct implementation, so fully qualified syntax is only needed to resolve ambiguity.

### Supertraits

A **supertrait** is a trait that another trait depends on. By requiring one trait to inherit from another, the derived trait gains access to all of the supertrait's associated items.

```rust
use std::fmt::Display;

trait OutlinePrint: Display {
    fn outline_print(&self) {
        // Uses Display::fmt()
    }
}
```

Here, `OutlinePrint` requires every implementer to also implement `Display`, allowing `outline_print()` to use the formatting functionality provided by `Display`.

Supertraits are Rust's way of expressing trait dependencies.

## Advanced Types

Rust's type system includes several advanced features that improve code clarity, expressiveness, and flexibility. These features don't introduce new kinds of data, but rather new ways of working with types.

In this section, we'll cover the **newtype pattern**, **type aliases**, the **never type (`!`)**, and **dynamically sized types (DSTs)**. While these features are less common than structs, enums, and generics, they appear throughout the standard library and are useful to understand.

### The Newtype Pattern

The **newtype pattern** creates a new type by wrapping an existing type in a tuple struct.

```rust
struct UserId(u32);
```

Although `UserId` contains a `u32`, it is a completely distinct type. This allows the compiler to prevent accidentally mixing it with other `u32` values.

Beyond working around the _orphan rule_, the newtype pattern has two additional benefits:

- It provides **stronger type safety** by preventing logically different values from being mixed.
- It allows a type to expose a **custom public API**, hiding the implementation details of the wrapped type.

Since the wrapper exists only at the type level, the compiler optimizes it away, so the pattern introduces no runtime overhead.

### Type Aliases

A **type alias** gives an existing type another name using the `type` keyword.

```rust
type Kilometers = i32;
```

Unlike a newtype, a type alias does **not** create a new type, it simply introduces another name for the same type. Consequently, `Kilometers` and `i32` are interchangeable.

Type aliases are most commonly used to simplify long or repetitive type signatures.

```rust
type Thunk = Box<dyn Fn() + Send + 'static>;
```

They are also widely used in the standard library. For example, `std::io::Result<T>` is simply an alias for `Result<T, std::io::Error>`, saving users from repeatedly writing the full type.

### The Never Type (`!`)

The **never type**, written `!`, represents computations that **never produce a value**.

Examples include functions that panic, terminate the program, or loop forever.

```rust
fn never_returns() -> ! {
    panic!("This function never returns!");
}
```

Since a value of type `!` can never exist, Rust can automatically coerce it into any other type. This allows expressions like `panic!()`, `continue`, and `return` to appear where a value of another type is expected.

```rust
let x = match value {
    Some(v) => v,
    None => panic!("No value"),
};
```

Here, one match arm produces an `i32`, while the other produces `!`. The expression is still valid because `!` can coerce into any type.

### Dynamically Sized Types (DSTs)

Most types in Rust have a size that is known at compile time. These are called **Sized** types.

Some types, however, do not have a compile-time size. These are known as **Dynamically Sized Types (DSTs)**.

The most common DST is `str`.

```rust
let s: &str = "hello";
```

A bare `str` cannot exist as a standalone value because its size is unknown. Instead, DSTs must always be used **behind a pointer**, such as `&str`, `Box<str>`, `Rc<str>`.

The pointer stores not only the address of the data, but also the metadata needed to determine its size at runtime.

### The `Sized` Trait

Rust automatically assumes that every generic type parameter implements the `Sized` trait.

```rust
fn foo<T>(value: T) {
    // T: Sized is assumed
}
```

This is equivalent to writing:

```rust
fn foo<T: Sized>(value: T) {
    // ...
}
```

If a generic function should also accept dynamically sized types, the implicit bound can be relaxed using `?Sized`.

```rust
fn foo<T: ?Sized>(value: &T) {
    // ...
}
```

Notice that the parameter is a reference. Since a DST has no known size, it cannot be passed or stored by value and must instead be accessed through some form of pointer.

## Advanced Functions and Closures

Functions and closures are closely related in Rust, and in most situations they can be used interchangeably. This chapter explores two advanced topics: **function pointers** and **returning closures**. These features are especially useful when writing generic libraries, inter-operating with other languages, or building APIs that produce behavior dynamically.

### Function Pointers

A **function pointer** is a pointer to a regular function. Its type is written as `fn`, which should not be confused with the closure traits (`Fn`, `FnMut`, and `FnOnce`).

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn do_twice(f: fn(i32) -> i32, arg: i32) -> i32 {
    f(arg) + f(arg)
}
```

Because functions have a concrete type, they can be passed around just like any other value.

### Function Pointers vs Closures

Function pointers and closures have similar call syntax, but they are not the same.

A function pointer refers to an existing function and **cannot capture values from its surrounding environment**. A closure, on the other hand, is an anonymous function that **can capture variables** from the scope in which it is created.

One important relationship between them is that **function pointers implement all three closure traits** (`Fn`, `FnMut`, and `FnOnce`). As a result, a function can always be passed wherever a closure is expected.

```rust
fn add_one(x: i32) -> i32 {
    x + 1
}

fn apply<F>(f: F, x: i32) -> i32
where
    F: Fn(i32) -> i32,
{
    f(x)
}

apply(add_one, 5);
```

For this reason, it is generally preferable to accept a generic closure (`F: Fn...`) rather than a function pointer (`fn...`), since doing so allows callers to provide either a regular function or a closure. The primary exception is when interfacing with foreign languages such as C, which understand function pointers but not closures.

### Returning Closures

Each closure in Rust has its own unique, anonymous type. Consequently, you cannot write a function whose return type is simply "a closure."

Instead, closures are typically returned using `impl Trait`.

```rust
fn make_adder() -> impl Fn(i32) -> i32 {
    |x| x + 1
}
```

The compiler knows the concrete closure type, even though it remains hidden from users of the function.

### Returning Multiple Closure Types

The `impl Trait` approach works only when every execution path returns **the same concrete closure type**.

If different branches return different closures, the return type must be **erased** using a trait object.

```rust
fn make_adder() -> Box<dyn Fn(i32) -> i32> {
    Box::new(|x| x + 1)
}
```

Boxing the closure behind `dyn Fn` gives all returned closures a common type, making it possible to store different closures in collections or return them from the same function.

## Macros

**Macros** are Rust's meta-programming facility. Unlike functions, which operate on values at runtime, macros operate on Rust code itself during compilation. They can generate, transform, or eliminate code before the compiler performs type checking and code generation.

Macros are most commonly used to reduce boilerplate, implement domain-specific languages (DSLs), and provide functionality that would be impossible or impractical to express with ordinary functions. Familiar examples from the standard library include `println!`, `vec!`, `format!`, and `dbg!`.

### Declarative vs Procedural Macros

Rust supports two kinds of macros.

**Declarative macros** (also called _macro_rules!_ macros) use pattern matching to transform one piece of Rust syntax into another. They are ideal for eliminating repetitive code and creating concise, expressive syntax.

**Procedural macros** are regular Rust programs that receive Rust code as input and produce Rust code as output. They are significantly more powerful than declarative macros and come in three forms:

- **Derive macros** (`#[derive(...)]`)
- **Attribute macros** (`#[my_attribute]`)
- **Function-like macros** (`my_macro!(...)`)

Procedural macros power many popular libraries, including serialization, web frameworks, and ORMs.

### Why Macros Are So Powerful

Macros are one of Rust's most powerful language features because they can generate code at compile time. This allows them to:

- Eliminate repetitive boilerplate.
- Create expressive APIs and mini domain-specific languages.
- Generate implementations automatically.
- Perform compile-time validation.
- Extend Rust's syntax in ways that functions cannot.

Many of Rust's most ergonomic libraries, including crates like **Serde**, **Tokio**, **Diesel**, and **Axum**, rely heavily on procedural macros to provide concise, declarative APIs.

At the same time, macros are considerably more complex than ordinary Rust code. They operate on Rust's syntax rather than values, making them harder to write, debug, and understand. Fortunately, most Rust programmers spend far more time **using** macros than **writing** them.

### Learning Resources

Macros are a substantial topic in their own right and deserve a dedicated study once you're comfortable with the rest of the language. The following resources are excellent references:

- The Rust Book, Chapter 20.5: _Macros_.
- **The Little Book of Rust Macros**; a comprehensive guide focused entirely on Rust macros.
- **The Rust Reference**; the authoritative specification for macro syntax and behavior.
- The documentation for the `syn`, `quote`, and `proc-macro2` crates, which form the foundation of most procedural macro development.

Unless you plan to write libraries or frameworks, you don't need to master macros immediately. Being comfortable reading and using common macros such as `println!`, `vec!`, and `derive` macros is sufficient for most day-to-day Rust programming.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
