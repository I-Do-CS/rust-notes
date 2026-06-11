# Closures & Iterators

- [Overview](#overview)
- [Annotating Closures](#annotating-closures)
- [The Ownership Semantics of Closures](#the-ownership-semantics-of-closures)
  - [Using Closures with Threads](#using-closures-with-threads)
- [Moving Captured Values Out of Closures](#moving-captured-values-out-of-closures)
  - [Trait Hierarchy](#trait-hierarchy)
- [Closures in Signatures](#closures-in-signatures)
- [Iterators](#iterators)

## Overview

A Rust closure is similar to a JavaScript arrow function with lexical capture, but the captured variables participate in Rust's ownership and borrowing system. The compiler decides whether each captured value is borrowed immutably, borrowed mutably, or moved into the closure, and the resulting closure type implements one or more of `Fn`, `FnMut`, and `FnOnce`.

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 } // A function for reference

// Different ways of writing a closure
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```

## Annotating Closures

Unlike functions, closures' parameter and return types are usually inferred by the compiler; but rarely, when they can't be inferred, we have to annotate them explicitly:

```rust
    // We didn't specify any type here
    let example_closure = |x| x;

    // The compiler here infers that the closure takes and returns a String here
    let s = example_closure(String::from("hello"));

    // We break the compiler's inference here;
    // so, this will cause a compiler error
    let n = example_closure(5);

    // We can fix it like so
    let n = example_closure(5.to_string());
```

## The Ownership Semantics of Closures

Closures can capture values from their surrounding environment. Rust analyzes how the closure uses each captured value and automatically chooses the least restrictive of the following capture modes.

- Immutable Borrow

  In the following. `list` is borrowed immutably because the closure only reads it.

  ```rust
  let list = vec![1, 2, 3];

  let print_list = || println!("{list:?}");

  print_list();
  print_list();

  println!("{list:?}"); // list is still accessible
  ```

- Mutable Borrow

  If a closure modifies a captured value, it captures a mutable reference (`&mut T`):

  ```rust
  // Because the closure mutates count, it requires mutable access to the captured variable.
  let mut count = 0;

  let mut increment = || {
      count += 1;
  };

  increment();
  increment();

  println!("{count}");
  ```

- Taking Ownership

  If a closure needs ownership of a captured value, Rust moves the value into the closure:

  ```rust
  // The move keyword forces ownership of captured values to be transferred into the closure.
  let text = String::from("hello");

  // The `move` keyword explicitly moves a ownership of the value
  let consume = move || {
      println!("{text}");
  };

  consume();
  ```

### Using Closures with Threads

A common use of `move` closures is spawning threads.
When a new thread is created, it may outlive the current scope. Borrowing local variables would therefore be unsafe. By moving ownership into the closure, Rust guarantees the captured data remains valid for the lifetime of the thread.

```rust
use std::thread;

let numbers = vec![1, 2, 3];

let handle = thread::spawn(move || {
    println!("{numbers:?}");
});

handle.join().unwrap();
```

Without move, the closure would attempt to borrow numbers, which is not allowed because the spawned thread may continue executing after the current scope ends.

## Moving Captured Values Out of Closures

Every closure implements at least one of Rust's three closure traits.
The trait implemented depends on how the closure uses its captured values.

1. `Fn`

   A closure that only reads captured values implements `Fn`. `Fn` closures can be called repeatedly because they do not modify or consume their captured values.

   ```rust
   let x = 10;

   let add = |y| x + y;

   println!("{}", add(5));
   println!("{}", add(10));
   ```

2. `FnMut`
   A closure that mutates captured values implements `FnMut`. Because the closure modifies its environment, it requires mutable access and therefore implements `FnMut`.

   ```rust
   let mut count = 0;

   let mut increment = || {
       count += 1;
   };

   increment();
   increment();
   ```

3. `FnOnce`
   A closure that consumes a captured value implements `FnOnce`. In the following, after the first call, `text` has been moved and dropped, so the closure cannot be called again.

   ```rust
   let text = String::from("hello");

   let consume = move || {
       drop(text);
   };

   consume();
   ```

- A closure implementing `FnOnce` does not necessarily consume its captures. `FnOnce` simply means it _might_ consume them and therefore can only be guaranteed to be callable once. Rust conservatively uses the least restrictive trait that the closure satisfies.

### Trait Hierarchy

The closure traits form the following hierarchy:

```
Fn      => call(&self)
FnMut   => call_mut(&mut self)
FnOnce  => call_once(self)
```

This means:

| Trait    | May Read | May Mutate | May Consume |
| -------- | -------- | ---------- | ----------- |
| `Fn`     | `true`   | `false`    | `false`     |
| `FnMut`  | `true`   | `true`     | `false`     |
| `FnOnce` | `true`   | `true`     | `true`      |

## Closures in Signatures

We can define functions/methods that take or return closures with the following syntax:

```rust
impl<T> Option<T> {
    pub fn unwrap_or_else<F>(self, f: F) -> T
    where
        F: FnOnce() -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```

## Iterators

Iterators in Rust provide a uniform way to process a sequence of values one item at a time. Rather than exposing how data is stored internally, an iterator exposes a stream of items that can be consumed, transformed, filtered, combined, or collected into new data structures. Any type that implements the `Iterator` trait can become an iterator, regardless of whether the values come from a collection, a range, a file, or are generated on demand.

```rust
fn main() {
    // Iterator from a collection
    let numbers = vec![1, 2, 3];

    // Iterator from a range
    let range = 4..=6;

    // Iterator generated on demand
    let generated = std::iter::repeat("hello").take(3);

    for n in numbers.into_iter() {
        println!("{n}");
    }

    for n in range {
        println!("{n}");
    }

    for word in generated {
        println!("{word}");
    }
}
```

One of the key advantages of iterators is that they are lazy. Like LINQ in C#, most iterator operations do not perform any work immediately; instead, they build a pipeline that only executes when values are actually requested. This allows Rust to optimize iterator chains efficiently while keeping the code expressive and concise.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];

    let doubled = numbers
        .iter()
        .map(|n| {
            println!("Doubling {n}");
            n * 2
        });

    // No output has been printed yet!
    // Methods such as collect() are called consuming adapters.
    // Calling them uses up the iterator and performs our operations.
    let result: Vec<_> = doubled.collect();
    println!("{result:?}");
}
```

When iterating over collections, Rust commonly provides three ways to access elements: by taking ownership of each element, by borrowing elements immutably, or by borrowing them mutably. Which approach is used determines whether the iterator yields owned values, shared references, or mutable references.

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    // Takes ownership of each element
    for n in numbers.clone().into_iter() {
        println!("Owned: {n}");
    }

    // Immutable references (&i32)
    for n in numbers.iter() {
        println!("Borrowed: {n}");
    }

    let mut numbers = vec![1, 2, 3];

    // Mutable references (&mut i32)
    for n in numbers.iter_mut() {
        *n *= 10;
    }

    println!("{numbers:?}");
}
```

Conceptually, an iterator is any object capable of producing a sequence of values on demand until no more values remain. The `Iterator` trait defines this behavior with the `next()` method.

```rust
fn main() {
    let mut iter = vec![10, 20, 30].into_iter();

    println!("{:?}", iter.next()); // Some(10)
    println!("{:?}", iter.next()); // Some(20)
    println!("{:?}", iter.next()); // Some(30)
    println!("{:?}", iter.next()); // None
}
```

Iterators are one of Rust's **_"Zero-Cost Abstractions"_** and supposedly have no performance overhead.
They are preferred over normal loops because of how cleanly you can express the flow of an operation with the help of its method APIs and Rust's closures.

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    // A chain of iterator adapters can express complex
    // logic cleanly without sacrificing performance
    let sum: i32 = numbers
        .iter() // Borrows each number
        .filter(|n| *n % 2 == 0) // Only keeps even numbers
        .map(|n| n * n) // Squares them
        .sum(); // Sums the results

    println!("{sum}");
}
```

The equivalent loop would be:

```rust
    let mut sum = 0;

    for n in &numbers {
        if n % 2 == 0 {
            sum += n * n;
        }
    }
```

- Notice that closures play an important role in how cleanly we can work with iterators.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
