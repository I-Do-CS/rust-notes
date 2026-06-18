# A (semi) Comprehensive Guide to Rust Pointers

- [Pointers](#pointers)
- [Raw Pointers](#raw-pointers)
- [Smart Pointers](#smart-pointers)
- [Thin vs Fat Pointers](#thin-vs-fat-pointers)
- [`Deref`](#deref)
  - [Deref Coercion](#deref-coercion)
  - [`DerefMut`](#derefmut)
- [`Drop`](#drop)
- [The Box](#the-box)
- [The Reference-Counted](#the-reference-counted)
- [Strong & Weak References](#strong--weak-references)
- [Interior Mutability](#interior-mutability)
- [The Cell Duo](#the-cell-duo)
- [Smart Pointers Quick Reference](#smart-pointers-quick-reference)
  - [`Box<T>`](#boxt)
  - [`Rc<T>`](#rct)
  - [`Arc<T>`](#arct)
  - [`Weak<T>`](#weakt)
  - [`Cell<T>`](#cellt)
  - [`RefCell<T>`](#refcellt)
  - [`Rc<RefCell<T>>`](#rcrefcellt)
  - [`Mutex<T>`](#mutext)
  - [`Arc<Mutex<T>>`](#arcmutext)
  - [`RwLock<T>`](#rwlockt)
  - [`Arc<RwLock<T>>`](#arcrwlockt)
  - [`Cow<'a, T>`](#cowa-t)
  - [`OnceCell<T>`](#oncecellt)
  - [`OnceLock<T>`](#oncelockt)
  - [`LazyLock<T>`](#lazylockt)
  - [`Pin<T>`](#pint)
  - [`UnsafeCell<T>`](#unsafecellt)

## Overview

This one got a bit unwieldy. Initially, it was supposed to be a recap of [The Book's](https://doc.rust-lang.org/book/) chapter on smart pointers; but since it had the potential, I turned it into a semi-comprehensive guide on Rust pointers.

## Pointers

A pointer is a value that contains the memory address of another value. Rather than storing the data itself, a pointer provides a way to access data stored elsewhere in memory (or simply put, pointers point to where some data is in memory).

The most common pointer type in Rust is a reference. References are indicated by the `&` symbol and allow code to borrow access to a value without taking ownership of it. References are lightweight, have no runtime overhead, and are checked by Rust's borrow checker to ensure memory safety.

## Raw Pointers

Raw pointers are low-level pointer types that bypass Rust's usual safety guarantees. They are represented by `*const T` (immutable) and `*mut T` (mutable) and are similar to `C`/`C++` pointers.

Unlike references, raw pointers are not checked by the borrow checker, may be null, may be dangling, and can violate Rust's aliasing and borrowing rules. Dereferencing a raw pointer is therefore an unsafe operation and must be performed within an `unsafe` block.

Raw pointers are primarily used when interacting with foreign code (FFI), implementing low-level abstractions, or working with memory that Rust's ownership system cannot reason about directly.

## Smart Pointers

Smart pointers are **_types_** that **_behave like pointers_** while also providing additional functionality, metadata, or ownership semantics. Unlike references, smart pointers often own the data they point to and manage that data according to specific rules.

Rust's standard library provides several smart pointers, each designed for a particular ownership model. Examples include `Box<T>` for heap allocation and single ownership, `Rc<T>` for shared ownership, and `RefCell<T>` for interior mutability.

Smart pointers are typically implemented as structs and usually implement the `Deref` trait, allowing them to behave like references, and the `Drop` trait, allowing them to customize cleanup behavior when they go out of scope. These traits enable smart pointers to extend Rust's ownership system while remaining ergonomic to use.

## Thin vs Fat Pointers

In Rust, not all pointers have the same size. Some pointers contain only a memory address, while others store additional metadata alongside the address.

A thin pointer consists solely of a memory address and therefore occupies one machine word. References and pointers to types whose size is known at compile time are typically thin pointers.

A fat pointer contains both a memory address and additional metadata required to work with dynamically sized types. As a result, fat pointers occupy more memory than thin pointers.

Common examples of fat pointers include:

- Slices/String slices (`&[T]`/`&str`), which store a pointer and a length.
- Trait objects (`&dyn Trait`), which store a pointer to the data and a pointer to a virtual method table (vtable).

The additional metadata allows Rust to work safely with values whose size or behavior cannot be fully determined at compile time.

## `Deref`

Implementing the `Deref` trait allows you to customize the behavior of the dereference operator (`*`). By implementing `Deref` in such a way that a smart pointer can be treated like a regular reference, you can write code that operates on references and use that code with smart pointers too.

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);
    assert_eq!(5, *y); // This will cause a compiler error
    // because an instance of MyBox<T> cannot be dereferenced
}
```

This is the corrected version of the above snippet:

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

// We implement the Deref trait for our type
impl<T> Deref for MyBox<T> {
    type Target = T;

    // We return a reference to the first data field of self
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn main() {
    let x = 5;
    let y = MyBox::new(x);

    assert_eq!(5, x);

    // Behind the scenes, Rust turns *y to
    // *(y.deref()) .aka *(&self.0)
    assert_eq!(5, *y);
}
```

### Deref Coercion

Deref Coercion is Rust's mechanism for automatically converting one reference type into another by following `Deref` implementations. When a context requires `&U`, Rust can repeatedly apply dereferencing to convert `&T` into a compatible reference type required by the current context.

Consider the following:

```rust
struct MyBox<T>(T);
impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}
impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

fn hello(name: &str) {
    println!("Hello, {name}!");
}
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&m);
}
```

In the above snippet Rust starts with `&MyBox<String>` (`&m`). Since `MyBox<T>` implements `Deref<Target = T>`, it can dereference `&MyBox<String>` to `&String`. Because `String` implements `Deref<Target = str>`, Rust can dereference `&String` to `&str`. The resulting `&str` matches the parameter type of `hello()`, so the call is accepted.

Essentially: `&MyBox<String>` ==`deref`==> `&String` ==`deref`==> `&str`

Without Deref Coercion, we would have to write it like this:

```rust
fn main() {
    let m = MyBox::new(String::from("Rust"));
    hello(&(*m)[..]);
    // 1. *m uses Deref to access the inner String
    // 2. [..] produces a str slice
    // 3. & creates an &str
}
```

### `DerefMut`

Similar to how you use the `Deref` trait to implement/override the `*` operator on immutable references, you can use the `DerefMut` trait to override the `*` operator on mutable references.

## `Drop`

The `Drop` trait allows you to customize the code that’s run when an instance of the value goes out of scope. You can provide an implementation for the `Drop` trait on any type, and that code can be used to release resources like files or network connections.

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Dropping CustomSmartPointer with data `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer {
        data: String::from("my stuff"),
    };
    let d = CustomSmartPointer {
        data: String::from("other stuff"),
    };
    println!("CustomSmartPointers created");
} // c and d are dropped here
```

When a scope ends, local variables are dropped in reverse order of their declaration.

We can prematurely drop a value by calling `std::mem::drop`. Calling the `drop` method directly is not allowed.

```rust
c.drop(); // Error
std::mem::drop(c); // OK
```

## The Box

The most straightforward smart pointer is a box `Box<T>`. Boxes allow you to store data on the heap rather than the stack. What remains on the stack is the pointer to the heap data.

Boxes are mostly useful for one of the following situations:

- Unknown size: Put the value in a Box so Rust only has to store a pointer of known size.
  - Recursive Types like Cons List

- Large data: Put the data in a Box so moving it only moves its pointer, not the entire value.
  - Avoiding expensive moves/copies of large values.

- Trait objects: Put the value in a Box so different types can be treated uniformly through a trait.
  - Dynamic dispatch with trait objects (`Box<dyn Trait>`).
  - More on trait objects later

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}

use List::{Cons, Nil};

fn main() {
    let list = Cons(
        1,
        Box::new(Cons(
            2,
            Box::new(Cons(
                3,
                Box::new(Nil),
            )),
        )),
    );
}
```

Boxes only provide indirection and heap allocation; they don’t have any other special capabilities thus, incur no performance overhead. They can be useful in cases like the cons list where the indirection is the only feature we need.
When a `Box<T>` value goes out of scope, the heap data that the box is pointing to is cleaned up as well because of the `Drop` trait implementation.

## The Reference-Counted

The Reference-Counted Smart Pointer (`Rc<T>`) is Rust's mechanism for shared ownership. Cloning an `Rc<T>` creates another owner of the same value rather than copying the value itself. Internally, `Rc<T>` keeps a reference count of how many owners point to the value and guarantees that the value remains alive as long as at least one strong reference exists. Because multiple owners may exist simultaneously, `Rc<T>` only provides immutable access to the value it contains. To mutate shared data, `Rc<T>` is commonly combined with interior mutability (more on this later) types such as `RefCell<T>`.

```rust
pub enum List {
    Cons(i32, Rc<List>),
    Nil,
}
fn main() {
    let a: Rc<List> = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    println!(
        "reference count after creating a = {}",
        Rc::strong_count(&a) // -> 1
    );

    // `_b` owns its own List node, but shares `a` as its tail
    let _b: List = Cons(3, Rc::clone(&a));
    println!(
        "reference count after creating b = {}",
        Rc::strong_count(&a) // -> 2
    );

    {
        // same as `_b`
        let _c: List = Cons(4, Rc::clone(&a));
        println!(
            "reference count after creating c = {}",
            Rc::strong_count(&a) // -> 3
        );
    } // _c is dropped here

    // Both `_b` and `_c` want to own a as their tail.
    // Box<T> would require transferring ownership,
    // but Rc<T> allows shared ownership.

    println!(
        "reference count after c goes out of scope = {}",
        Rc::strong_count(&a) // -> 2
    );
} // Reference count becomes 0 and the value is freed
```

`Rc<T>` is only intended for single-threaded scenarios. For shared ownership across threads, Rust provides `Arc<T>` (Atomic Reference Counted Pointer), which performs thread-safe reference counting but comes with some overhead.

## Strong & Weak References

When using `Rc<T>`, Rust distinguishes between strong references and weak references.

A strong reference (`Rc<T>`) represents ownership of a value. Every time an `Rc<T>` is cloned, the strong reference count increases. The value remains allocated as long as at least one strong reference exists.

A weak reference (`Weak<T>`) provides a non-owning reference to a value. Weak references do not contribute to the strong reference count and therefore do not keep the value alive. They are typically used when a value needs to be referenced without expressing ownership.

Because `Weak<T>` does not participate in ownership, it does not guarantee that the referenced value is still alive. To access the value, a weak reference must first be upgraded into an `Rc<T>`. This upgrade succeeds only while at least one strong reference exists and fails once the value has been deallocated.

Weak references are particularly important when building graphs, trees, and other data structures with bidirectional relationships. By replacing some ownership relationships with weak references, reference cycles can be avoided.

A reference cycle occurs when two or more values own each other through `Rc<T>`. Since each value keeps the other's reference count above zero, none of them can be deallocated, resulting in a memory leak. Using `Weak<T>` for non-owning relationships breaks these cycles and allows memory to be freed correctly (More on this later).

## Interior Mutability

Interior mutability is a Rust design pattern that allows a value to modify part of its internal state even when it is accessed through an immutable reference (`&T`). Under Rust's normal borrowing rules, an immutable reference guarantees read-only access to the referenced value. Interior mutability provides a controlled exception to this rule by wrapping specific fields in special types that permit mutation while still maintaining memory safety.

This pattern is useful when a type's public interface represents an operation as logically read-only, but the implementation needs to update internal details that are not part of the value's observable state. E.g. caching computed results, recording usage statistics, maintaining shared ownership metadata, or implementing data structures whose internal bookkeeping must change during otherwise read-only operations.

Essentially, Interior mutability is Rust's controlled exception to the usual rule that data behind an immutable reference cannot be modified.

Rust implements interior mutability using `UnsafeCell<T>`, a low-level primitive that allows mutation through shared references. Higher-level types such as `Cell<T>` and `RefCell<T>` build safe abstractions on top of `UnsafeCell<T>` by enforcing rules that prevent invalid access and preserve Rust's guarantees about memory safety.

## The Cell Duo

As previously mentioned, two of the most common smart pointers which provide interior mutability are `Cell<T>` and `RefCell<T>`.
They allow mutation through an immutable reference (`&T`), but they do so in very different ways and are designed for different use cases.

`Cell<T>` is the simplest interior mutability type. It stores a value and allows that value to be replaced, retrieved, or swapped even when the Cell itself is accessed through an immutable reference. `Cell<T>`'s main strength is that it never gives out references to its contents; so, no references to the inner value can exist, thus, Rust never has to worry about aliasing or borrow violations. Because of this, `Cell<T>` requires no runtime borrow checking and cannot panic due to borrowing errors.

Instead of borrowing the inner value:

- `get()` copies the value out (for `Copy` types)
- `set()` replaces the value
- `replace()` swaps the value and returns the old one
- `take()` removes the value and replaces it with `Default::default()`

```rust
use std::cell::Cell;

// Isn't declared with `mut`
let count = Cell::new(0);

count.set(1);
count.set(2);

println!("{}", count.get());
```

`RefCell<T>` provides interior mutability using a completely different strategy. Instead of forbidding references, it allows them; however, because references are allowed, borrow rules must still be enforced somehow; therefore, borrow checking takes place during runtime instead of compile time.

- `.borrow()` creates an immutable borrow
- `.borrow_mut()` creates a mutable borrow

The same borrowing rules (either one mutable reference or any number of immutable references) that Rust enforces at compile time are also enforced with these borrows but at runtime (if we break them, the program panics).

Sometimes Rust's compile-time analysis is too conservative. So, a `RefCell<T>` is a good solution.

Another common use case for `RefCell<T>` is combining it with `Rc<T>` to create `Rc<RefCell<T>>`. This pattern provides multiple ownership with interior mutability, allowing several owners to share and modify the same value.

```rust
use std::cell::RefCell;
use std::rc::Rc;

let data = Rc::new(RefCell::new(5));

let a = Rc::clone(&data);
let b = Rc::clone(&data);

// Even though a and b both point to the same value, RefCell<T>
// still enforces Rust's borrowing rules at runtime. Attempting
// to create multiple active mutable borrows will cause a panic.
*a.borrow_mut() += 1; // the mutable reference here is dropped at the end of the statement
*b.borrow_mut() += 1;

```

`Rc<RefCell<T>>` is a powerful tool, but it should be used carefully. Because ownership is shared and borrowing rules are enforced at runtime, complex structures can become harder to reason about than ordinary ownership and borrowing.

One common pitfall is the creation of reference cycles. Suppose two nodes, `A` and `B`, each store an `Rc` pointing to the other. Even after all external owners are dropped, `A` keeps `B` alive and `B` keeps `A` alive. As a result, neither reference count ever reaches zero, so neither value is deallocated. The memory remains allocated for the lifetime of the program, resulting in a memory leak.

```rust
use std::{cell::RefCell, rc::Rc};

struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
}

fn main() {
    let a = Rc::new(Node {
        value: 1,
        next: RefCell::new(None),
    });

    let b = Rc::new(Node {
        value: 2,
        next: RefCell::new(None),
    });

    // A -> B
    *a.next.borrow_mut() = Some(Rc::clone(&b));

    // B -> A
    *b.next.borrow_mut() = Some(Rc::clone(&a));

    println!("a strong count = {}", Rc::strong_count(&a)); // 2
    println!("b strong count = {}", Rc::strong_count(&b)); // 2
}
```

Rust provides `Weak<T>` to break such cycles. Unlike `Rc<T>`, a `Weak<T>` does not contribute to the reference count and therefore does not keep a value alive.

```rust
use std::{
    cell::RefCell,
    rc::{Rc, Weak},
};

#[derive(Debug)]
struct Node {
    value: i32,
    next: RefCell<Option<Rc<Node>>>,
    prev: RefCell<Option<Weak<Node>>>,
}

fn main() {
    let a = Rc::new(Node {
        value: 1,
        next: RefCell::new(None),
        prev: RefCell::new(None),
    });

    let b = Rc::new(Node {
        value: 2,
        next: RefCell::new(None),
        prev: RefCell::new(None),
    });

    // A strongly owns B
    *a.next.borrow_mut() = Some(Rc::clone(&b));

    // B weakly references A
    *b.prev.borrow_mut() = Some(Rc::downgrade(&a));

    println!("a strong count = {}", Rc::strong_count(&a)); // 1
    println!("b strong count = {}", Rc::strong_count(&b)); // 2

    println!("a weak count = {}", Rc::weak_count(&a)); // 1
}
```

## Smart Pointers Quick Reference

Up to this point, we've covered the smart pointers most commonly used in single-threaded Rust. The remaining types target more specialized use cases, particularly concurrency and advanced ownership patterns. Concurrent primitives will be discussed only at a high level here, since they will receive their own dedicated `.md` later on.

This section is intended as a rough quick reference for most of the useful smart pointers available in `std`.

### `Box<T>`

`Box<T>` provides single ownership of heap-allocated data.

Normally, Rust stores values on the stack when their size is known at compile time. `Box<T>` moves a value onto the heap while the box itself remains on the stack.

Characteristics:

- Large values you don't want copied around.
- Recursive types.
- Trait objects (`Box<dyn Trait>`).
- Explicit heap allocation.
- Mutable access follows normal borrowing rules.
- Value is dropped when the box is dropped.

Example:

```rust
    let x = Box::new(42);
    println!("{}", *x);
```

Common recursive type:

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

### `Rc<T>`

`Rc<T>` (Reference Counted) is Rust's mechanism for shared ownership in single-threaded programs. Cloning an `Rc<T>` creates another owner of the same value rather than copying the value itself.

Internally, `Rc<T>` maintains a strong reference count. The value remains alive as long as at least one strong reference exists.

Characteristics:

- Multiple owners
- Immutable access only
- Single-threaded
- Automatically deallocates when the strong count reaches zero

```rust
use std::rc::Rc;

let value = Rc::new(String::from("hello"));

let a = Rc::clone(&value);
let b = Rc::clone(&value);
```

### `Arc<T>`

`Arc<T>` (Atomic Reference Counted) is the thread-safe counterpart to `Rc<T>`.

Like `Rc<T>`, it provides shared ownership. Unlike `Rc<T>`, its reference count is maintained using atomic operations, making it safe to share across threads, though it comes with some overhead.

Characteristics:

- Multiple owners.
- Immutable access only.
- Thread-safe.
- Slightly more expensive than `Rc<T>`.

```rust
use std::sync::Arc;

let data = Arc::new(String::from("hello"));
```

`Arc<T>` only makes ownership thread-safe. It does not make mutation thread-safe.

### `Weak<T>`

`Weak<T>` is a non-owning reference into data managed by `Rc<T>` or `Arc<T>`.

Unlike strong references, weak references do not contribute to the strong reference count and therefore do not keep a value alive.

Characteristics:

- Non-owning.
- Prevent reference cycles.
- Must be upgraded before use.

```rust
use std::rc::Rc;

// Strong reference = owner.
// Weak reference = observer.

let rc = Rc::new(5);
let weak = Rc::downgrade(&rc);

assert!(weak.upgrade().is_some());

drop(rc);

assert!(weak.upgrade().is_none());
```

### `Cell<T>`

`Cell<T>` provides interior mutability for small `Copy` types.

Rather than handing out references, `Cell<T>` copies values in and out of the container. Because references are never exposed, no runtime borrow checking is required.

Characteristics:

- Interior mutability.
- No runtime borrow checking.
- Best suited for `Copy` types.
- Single-threaded.

```rust
use std::cell::Cell;

let count = Cell::new(0);

count.set(5);

println!("{}", count.get());
```

### `RefCell<T>`

`RefCell<T>` provides interior mutability through runtime borrow checking.

Instead of enforcing borrowing rules at compile time, `RefCell<T>` enforces them while the program runs.

Characteristics:

- Interior mutability.
- Runtime borrow checking.
- Single-threaded.
- Violations cause a panic.

```rust
use std::cell::RefCell;

let value = RefCell::new(5);

*value.borrow_mut() += 1;
```

### `Rc<RefCell<T>>`

`Rc<RefCell<T>>` combines shared ownership with interior mutability.

`Rc<T>` provides multiple owners, while `RefCell<T>` allows those owners to mutate shared data safely through runtime borrow checking.

Characteristics:

- Multiple owners.
- Shared mutable data.
- Single-threaded.

```rust
use std::cell::RefCell;
use std::rc::Rc;

let data = Rc::new(RefCell::new(0));

let a = Rc::clone(&data);
let b = Rc::clone(&data);

*a.borrow_mut() += 1;
*b.borrow_mut() += 1;
```

### `Mutex<T>`

`Mutex<T>` (Mutual Exclusion) protects data by allowing only one thread at a time to access it.

Access to the underlying value requires acquiring a lock.

Characteristics:

- Interior mutability.
- Thread-safe.
- One active accessor at a time.

```rust
use std::sync::Mutex;

let value = Mutex::new(0);

{
    let mut guard = value.lock().unwrap();
    *guard += 1;
}
```

### `Arc<Mutex<T>>`

`Arc<Mutex<T>>` is the standard pattern for shared mutable state across threads.

`Arc<T>` provides shared ownership while `Mutex<T>` provides synchronized mutation.

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));
```

### `RwLock<T>`

`RwLock<T>` (Reader-Writer Lock) allows multiple concurrent readers or one exclusive writer.

This can improve performance when reads are far more common than writes.

Characteristics:

- Many readers.
- One writer.
- Thread-safe.

```rust
use std::sync::RwLock;

let value = RwLock::new(5);

let read = value.read().unwrap();
```

### `Arc<RwLock<T>>`

`Arc<RwLock<T>>` combines shared ownership with a reader-writer lock.

This pattern is commonly used for caches, configuration, and shared lookup structures.

```rust
use std::sync::{Arc, RwLock};

let data = Arc::new(RwLock::new(vec![1, 2, 3]));
```

### `Cow<'a, T>`

`Cow` stands for Clone-On-Write.

It allows a value to be borrowed when possible and cloned only when mutation becomes necessary.

Internally, a `Cow<T>` is either:

```rust
Borrowed(&'a T)
Owned(T)
```

```rust
use std::borrow::Cow;

fn uppercase(s: &str) -> Cow<str> {
    if s.chars().all(char::is_uppercase) {
        Cow::Borrowed(s)
    } else {
        Cow::Owned(s.to_uppercase())
    }
}
```

The primary benefit of `Cow<T>` is avoiding unnecessary allocations.

### `OnceCell<T>`

`OnceCell<T>` stores a value that can be initialized exactly once.

After initialization, the value can be read many times but never replaced.

Characteristics:

- Write once.
- Read many.
- Single-threaded.

```rust
use std::cell::OnceCell;

let cell = OnceCell::new();

cell.set("hello").unwrap();
```

### `OnceLock<T>`

`OnceLock<T>` is the thread-safe version of `OnceCell<T>`.

It is commonly used for global configuration and lazily initialized shared state.

```rust
use std::sync::OnceLock;

static CONFIG: OnceLock<String> = OnceLock::new();

CONFIG.get_or_init(|| "production".to_string());
```

### `LazyLock<T>`

`LazyLock<T>` automatically initializes itself on first access.

It is built on top of `OnceLock<T>` and removes the need to call `get_or_init()` manually.

```rust
use std::sync::LazyLock;

static CONFIG: LazyLock<String> =
    LazyLock::new(|| "production".to_string());
```

### `Pin<T>`

`Pin<T>` guarantees that a value will not move in memory after being pinned.

This guarantee is required by self-referential types, async futures, generators, and other structures that depend on stable memory addresses.

Characteristics:

- Prevents movement.
- Does not prevent mutation.
- Common in async Rust.

```rust
use std::pin::Pin;

let value = Box::pin(String::from("hello"));
```

The key guarantee of `Pin<T>` is address stability.

### `UnsafeCell<T>`

`UnsafeCell<T>` is the fundamental primitive that powers Rust's interior mutability system.

Normally, Rust assumes that data behind an immutable reference cannot be mutated. `UnsafeCell<T>` is the one legal mechanism that allows this assumption to be bypassed.

Most programmers never use `UnsafeCell<T>` directly. Instead, higher-level abstractions are built on top of it:

- `Cell<T>`
- `RefCell<T>`
- `Mutex<T>`
- `RwLock<T>`
- `OnceCell<T>`

```rust
use std::cell::UnsafeCell;

let value = UnsafeCell::new(5);
```

Direct access to the inner value requires `unsafe` code.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
