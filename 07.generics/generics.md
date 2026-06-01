# Generic Types

- [Overview](#overview)
- [Generic Constraints](#generic-constraints)
- [Generic Structs](#generic-structs)
- [Generic Enums](#generic-enums)
- [Methods with Generic Types](#methods-with-generic-types)
- [Generic Combos](#generic-combos)

## Overview

Generic types allow code to work with multiple types while preserving type safety.
Instead of writing separate versions of the same logic for different types, you write one generalized version for parameterized types like so:

```rust
// For integers
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

// For floats
fn largest_f64(list: &[f64]) -> &f64 {
        let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}

// Generics solve this duplication with type parameters
fn largest<T>(list: &[T]) -> &T {
    // More on this implementation later
}
```

Here's a few ways to call generic functions:

```rust
let items = [1, 2, 3, 4, 5];

// Most to least explicit
let x: i32 = largest::<i32>(&items);
let x = largest::<i32>(&items);
let x: &i32 = largest(&items);
let x = largest(&items);
```

In the above example, T is a type parameter. We can pass any type to the `largest<T>` function as long as it implements the `PartialOrd` trait. Technically, our `largest<T>` example is wrong, because we didn't specify this constraint; but we'll get to that shortly.

Essentially, Generics allow us to easily implement abstractions and write reusable, type safe code.

## Generic Constraints

Often, Generics need constraints because not all operations work on all types; for example, the `largest<T>` function from before;
I intentionally didn't implement its body cause it wouldn't compile.
What if a list of custom structs was passed into it? How do we find the largest of something, if we don't know what that something even is?

This is where generic constraints come in. Usually, we specify them as Rust Traits.

I'll cover Traits later; but for now, know that traits are basically contracts (similar to C# interfaces) that a type has to satisfy (implement).

The following is the correct version of the `largest<T>` function.

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

`T: PartialOrd` basically tells the compiler to make sure that every time a scope is calling `largest`, the values it's passing into it are satisfying the `PartialOrd` contract. However, there are a few other ways to express this in Rust; I'll also cover those the end of this chapter.

## Generic Structs

Rust structs can also benefit from generic parameterization with the following syntax.

```rust
// Defined like this
struct Point<T> {
    x: T,
    y: T,
}

// Can also contain more than one parameter
struct Color<T1, T2> {
    r: T1,
    g: T1,
    b: T1,
    opacity: T2,
}

fn main() {
    // Instantiated like this (Most to least explicit)
    let p1: Point<i32> = Point::<i32> { x: 7, y: 8 };
    let p2 = Point::<i32> { x: 5, y: 6 };
    let p3: Point<i32> = Point { x: 3, y: 4 };
    let p4 = Point { x: 1, y: 2 };

    let color1: Color<u8, f32> = Color::<u8, f32> {
        r: 0,
        g: 0,
        b: 0,
        opacity: 1.0,
    };

    let color2 = Color {
    r: 0,
    g: 0,
    b: 0,
    opacity: 1.0,
    };
}
```

## Generic enums

Rust enums can also benefit from generic parameterization. The `Options<T>` and `Result<T, E>` are great examples of this:

```rust
enum Options<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

## Methods with Generic Types

Methods can use generics in a few different ways:

1. Generic impl blocks:

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

2. Method-specific generic parameters:

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    // Methods themselves can introduce additional generics.
    fn mixup<V, W>(self, other: Point<V, W>) -> Point<T, W> {
        Point {
            x: self.x,
            y: other.y,
        }
    }
}

fn main() {
    // Pay attention to the concrete types
    let p1: Point<i32, f64> = Point { x: 5, y: 10.4 };
    let p2: <&str, char> = Point { x: "hello", y: 'c' };
    let p3: Point<i32, char> = p1.mixup(p2);
}
```

3. Specialized implementations:
   - These allow us to implement specific functionality for generics only when they have a certain concrete type.

```rust
struct Point<T> {
    x: T,
    y: T,
}

// Every method inside here is available for every Point<T>
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// But methods inside here are only available for Point<i32>
impl Point<i32> {
    fn print_type(&self) {
        println!("My type is i32");
    }
}

fn main() {
    let p_int = Point {x: 12, y: 34};
    let p_float = Point {x: 1.2, y: 3.4};

    p_int.x();
    p_float.x();

    p_int.print_type(); // Only available for i32s
    p_float.print_type(); // Causes error
}
```

## Generic Combos

_You may want to read about Traits before proceeding._

Rust has two distinct syntaxes for parameterizing generics. The `<T: TraitName>` and `impl TraitName`. There's also the ability to write `where` clauses for the former syntax to clean it up. You can specify more than one trait with the `+` operator as well.

All of the above in combination may look confusing; so, as promised, here's some examples in increasing complexity to get a feel for them:

```rust
use std::fmt::Display;

// SETTING UP TYPES AND TRAITS

// Any type that implements this must have a summarize method available.
trait Summary {
    fn summarize(&self) -> String;
}

struct NewsArticle {
    headline: String,
    location: String,
    author: String,
    content: String,
}
// This is how we implement traits for structs
impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        return format!("{}, by {} ({})", self.headline, self.author, self.location);
    }
}
impl Display for NewsArticle {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return std::fmt::Result::Ok(());
    }
}

struct SocialPost {
    username: String,
    content: String,
    reply: bool,
    repost: bool,
}
impl Summary for SocialPost {
    fn summarize(&self) -> String {
        return format!("{}: {}", self.username, self.content);
    }
}
impl Display for SocialPost {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        return std::fmt::Result::Ok(());
    }
}
impl Clone for SocialPost {
    fn clone(&self) -> Self {
        return SocialPost {
            username: "".to_string(),
            content: "".to_string(),
            reply: true,
            repost: true,
        };
    }
}


// END OF SETUP. EXAMPLES START HERE

// Any type that implements the Summary trait
// (NewsArticle and SocialPost)
fn example1<T: Summary>(item: &T) -> String {
    return item.summarize();
}

// Same as above; `item: &impl Summary` reads as:
// "Any reference type that implements the Summary trait"
// It's syntax sugar for `fn example<T: Summary>(item: &T)`
fn example1b(item: &impl Summary) -> String {
    return item.summarize();
}

// item1 and item2 are reference types that implement the
// Summary trait. item3 is a value type that implements Display
fn example2<T1: Summary, T2: Display>(item1: &T1, item2: &T1, item3: T2) {}

// Same as above but T1 is a generic for multiple traits instead of one
fn example3<T1: Summary + Clone + Copy, T2: Display>(item1: &T1, item2: &T1, item3: T2) {}

// Same idea, different syntax
fn example4(item1: &impl Summary, item2: impl Display, item3: &impl Copy) {}

// Same as example3 but with the impl syntax
fn example5(
    item1: impl Summary + Clone + Copy,
    item2: impl Display,

    // Use parenthesis for reference types
    item3: &(impl Summary + Clone + Copy),
) {
}

// This is similar to example 3 but with a `where`
// clause used to clean up the its signature.
// Note that return type comes before the where clause
fn example6<T1, T2, T3>(item1: T1, item2: &T2, item3: &T3) -> i32
where
    T1: Summary + Clone + Copy,
    T2: Display,
    T3: Summary + Clone + Display,
{
    1234
}

// Similar to the last one but the return value is
// any type that implements the three specified traits.
fn example7<T1, T2, T3>(item1: T1, item2: &T2, item3: &T3) -> impl Summary + Clone + Display
where
    T1: Summary + Clone + Copy,
    T2: Display,
    T3: Summary + Clone + Copy,
{
    // Our SocialPost struct does in fact, satisfy
    // all three traits we specified
    return SocialPost {
        content: "".to_string(),
        reply: true,
        repost: true,
        username: "".to_string(),
    };
}
```
## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
