# Traits in Rust

- [Traits](#traits)
- [Default Implementations](#default-implementations)
- [Traits & Generics](#traits--generics)
- [Implementing Methods Conditionally with Trait Bounds](#implementing-methods-conditionally-with-trait-bounds)

## Traits

Traits are Rust’s primary abstraction mechanism for polymorphism, code reuse, and defining shared behavior.
A trait specifies, what functionality a type must provide, which methods exist, and optional default behaviors.
Rust heavily favors composition through traits instead of inheritance.

```rust
// Defined with the trait keyword
// Any type implementing Vehicle promises it can
// perform the move_it() behavior.
trait Vehicle {
    // Only the signature
    fn move_it(&self) -> i32;
}

struct Car {
    // ...
}

// This is how we implement traits on a type
impl Vehicle for Car {
    fn move_it(&self) {
        // Logic to move car
    }
}
```

- Examples from the standard library:

```rust
Clone
// Allows a value to be explicitly
// duplicated by creating a deep or custom copy.

Copy
// Allows a value to be duplicated
// implicitly through a simple bitwise copy.

Debug
// Enables a type to be printed in
// a programmer-friendly format using `{:?}`.

Display
// Enables a type to be printed in
// a user-friendly format using `{}`.

Iterator
// Defines how a type produces a
// sequence of values one at a time via `next()`.

IntoIterator
// Allows a type to be converted
// into an iterator so it can be used in a `for` loop.

PartialEq
// Enables equality comparisons (`==` and `!=`) between values.

Ord
// Enables total ordering comparisons (`<`, `>`, `<=`, `>=`)
// between values.

Read
// Provides methods for reading
// bytes from a data source such as a file or stream.

Write
// Provides methods for writing
// bytes to a destination such as a file or stream.
```

- A type can implement many traits

```rust
impl Clone for Car { /* ... */ }
impl Debug for Car { /* ... */ }
impl Display for Car { /* ... */ }
```

## Default Implementations

Traits can optionally provide default method implementations.

```rust
trait Vehicle {
    // Unlike this,
    fn honk(&self);

    // This has a default implementation
    fn move_it(&self) -> i32 {
        println!("I'm a vehicle and I'm moving!");
        return 0;
    }
}

struct Car {
    // ...
}
impl Vehicle for Car {
    fn honk(&self) {
        println!("HOOOONK!");
    }

    // Notice that we don't implement the `move_it` method
    // and relied on its default implementation.
}
```

## Traits & Generics

Traits are commonly used with generic type parameters in method/function signatures to enforce all sorts of constraints.
Here's an example:

```rust
fn notify(item: &impl Summary) -> i32 {
    println!("{}", item.summarize());
    return 0;
}
// Or
fn notify<T: Summary>(item: &T) -> i32 {
    println!("{}", item.summarize());
    return 0;
}
// Or
fn notify<T>(item: &T) -> i32
where
    T: Summary,
{
    println!("{}", item.summarize());
    return 0;
}
```

- Checkout the last section of my
  [Generic Types](https://github.com/I-Do-CS/rust-notes/tree/main/07.generics/generics.md)
  markdown for more complex examples.

## Implementing Methods Conditionally with Trait Bounds

Trait bounds are constraints that specify which traits a generic type must implement in order to be used.
I've been using them in my examples all this time but didn't call them out explicitly.

```rust
// This is a Trait Bound
<T: Display + Copy>

// This is also a trait bound
x: &(impl Display + Copy)
```

If you remember from my notes on generics, we implemented methods for generic types like this:

```rust
Point<T> {
    x: T,
    y: T,
}
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
// Or
impl<T, U> Point<T, U> {
    // Methods themselves can introduce additional generics.
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}
```

We could also implement methods for a generic type only when it had a specific concrete type like so:

```rust
impl Point<i32> {
    fn print_type(&self) {
        println!("My type is i32");
    }
}
```

We can combine generic `impl` blocks with trait bounds to conditionally implement methods that are only available when a generic type satisfies certain trait requirements.

```rust
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("Largest is x = {}", self.x);
        } else {
            println!("Largest is y = {}", self.y);
        }
    }
}
```

The trait bound `T: Display + PartialOrd` ensures that `cmp_display` is only available for `Pair<T>` when `T` implements both `Display` and `PartialOrd`.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
