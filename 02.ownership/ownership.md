# Ownership & Borrowing

- [Ownership](#ownership)
- [Stack vs Heap](#stack-vs-heap)
- [RAII](#raii-resource-acquisition-is-initialization)
- [Move](#move)
- [References](#references)
- [Borrowing](#borrowing)
- [Dangling References](#dangling-references)

## Ownership

Ownership is Rust’s core memory management model. It replaces the need for a garbage collector while still preventing many memory safety bugs at compile time.
In Rust, every value has an owner. The owner is the variable responsible for cleaning up that value when it is no longer needed. When s goes out of scope, Rust automatically frees the memory.

```rust
{
    let s = String::from("hello");
} // s goes out of scope here, memory is freed
```

Rust’s ownership system is built around three main rules
These rules allow Rust to know exactly when memory should be cleaned up, entirely at compile time.

1. Each value has exactly one owner.
2. A value can only have one owner at a time.
3. When the owner goes out of scope, the value is dropped (memory freed).

Ownership mainly matters for heap-allocated data like:
Simple stack values like integers often behave differently because they implement Copy (more on Copy later).

```rust
String
Vec<T>
Box<T>
// & custom heap structures
```

## Stack vs Heap

Simply put, the stack and heap are two different memory regions used by programs and runtimes.

- The stack is very fast, ordered, automatically managed.
  Stack memory works like a stack of plates (push values on top, pop values off the top).
  Data on the stack must have a known fixed size at compile time.
- The heap is more flexible, slower, manually managed in many languages, and used for dynamically sized data.
  Heap allocation is needed when size is unknown at compile time, data may grow/shrink, data must outlive certain scopes (very important), large allocations are needed.

Usually, Heap allocation requires, requesting memory from the allocator, tracking ownership, freeing memory later.
Rust’s ownership system automates this safely without garbage collection.

## RAII: Resource Acquisition Is Initialization

In C++, RAII is a major idiom where resources are tied to object lifetime. it's when an object acquires resources in its constructor and releases them in its destructor.

```cpp
// When the object leaves scope, cleanup happens automatically.
{
    File file("data.txt");
} // destructor automatically closes file
```

Rust uses a very similar idea with the ownership system.
When a value goes out of scope, Rust automatically calls its drop logic.

```rust
// Internally, String implements the Drop trait.
{
let s = String::from("hello");
} // drop called automatically
```

Unlike C++, Rust additionally enforces ownership rules statically, preventing:

- double free
- use after free
- dangling pointers
- Basically RAII + strict compile-time ownership enforcement

## Move

A move transfers ownership from one variable to another.

```rust
let s1 = String::from("hello");
let s2 = s1;

s1 // invalid
s2 // owns the String
```

In the above example, **_s1_** owns heap memory. If both **_s1_** and **_s2_** owned the same memory, Rust could accidentally free it twice; so Rust invalidates **_s1_** after a move.

Moves happen:

- During assignment (like above)
- When passing ownership into functions/methods
- When returning ownership in many pattern matching situations

But not all types move.
Simple stack-only types like integers implement the Copy trait; so, instead of moving, their values are copied.

A type cannot implement the Copy trait if it implements the Drop trait. The reason why has to do with a combination of references and the heap (more on this coming up).

## References

Simply put, A reference allows you to access a value without taking ownership of it. Allowing safe access while avoiding unnecessary moves or cloning.

```rust
let mut s: String = String::from("hello");

// We use the & operator to pass references
let s_ref: &String = &s;
println!("{s_ref}");

// &mut operator for mutable references
let s_mut_ref: &mut String = &mut s;
s_mut_ref.push_str(", world!");
```

The above example contains a mutable variable; mutable variables can have mutable references (their values can be changed using their references) but the opposite is not true.

## Borrowing

Borrowing means temporarily accessing a value through references without taking ownership.

```rust
fn calc_len(s: &String) -> i32 {
    // Borrows s and returns its length
    return s.len();
}

fn main() {
    let s = String::from("hello");
    let len = calc_len(&s);
    println!("{len}");
}
```

There are 4 very important rules for borrowing:

1. You can have **_any number_** of **_immutable_** references.
2. You can have **_only one mutable_** reference at a time.
3. Mutable and immutable references cannot coexist simultaneously.
4. References must always be valid.

The reasons for these rules boil down to preventing data races and invalid memory access.

The following two snippets highlight the difference in semantics between moving and borrowing.

```rust
fn x_moved(x: String) {
    // x is moved to this scope and is freed afterwards
    println!("{x}");
}

fn x_moved_and_returned(x: String) -> String {
    // x is moved to this scope and is
    // returned to the calling scope afterwards
    println!("{x}");
    return x;
}

fn x_borrowed(x: &String) {
    println!("{x}");
}

fn x_borrowed_and_changed(x: &mut String) {
    x.push_str("goodbye, world!");
    println!("{x}");
}

fn main() {
    let mut x: String = String::from("hello, world!");

    x_borrowed(&x);
    println!("{x}"); // x still available

    // we couldn't have done the following
    // if x wasn't a mutable variable in the first place
    x_borrowed_and_changed(&mut x);
    println!("{x}"); // x still available with its value changed

    x = x_moved_and_returned(x);
    println!("{x}"); // x still available

    x_moved(x);
    // Causes compiler error because x was dropped
    println!("{x}");
}
```

## Dangling References

A dangling reference is a reference pointing to memory that has already been freed. This is a major source of bugs in languages like C and C++. Rust prevents dangling references entirely with the help of the borrow-checker, ownership semantics, and lifetimes.

```rust
// This won't compile since 's' is destroyed at function end
// and the returned reference would point to freed memory.
fn dangle() -> &String {
    let s = String::from("hello");
    &s // return &s;
}

// Instead, ownership should be returned:
fn no_dangle() -> String {
    let s = String::from("hello");
    s // return s;
}
```

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
