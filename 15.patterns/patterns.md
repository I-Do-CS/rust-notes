# Patterns & Matching Revisited

- [Overview](#overview)
- [What are Patterns](#what-are-patterns)
- [What is Pattern Matching](#what-is-pattern-matching)
- [All the Places We Can Use Patterns](#all-the-places-we-can-use-patterns)
  - [`match` Arms](#match-arms)
  - [`let` Statements](#let-statements)
  - [Conditional `if let` Statements](#conditional-if-let-statements)
  - [`let...else` Expressions](#letelse-expressions)
  - [`while let` Conditional Loops](#while-let-conditional-loops)
  - [`for` Loops](#for-loops)
  - [Function Parameters](#function-parameters)
  - [Closure Parameter Lists](#closure-parameter-lists)
- [Irrefutability vs Refutability](#irrefutability-vs-refutability)
- [Pattern Matching 101](#pattern-matching-101)
  - [Matching Literals](#matching-literals)
  - [Matching Named Variables](#matching-named-variables)
  - [Matching Multiple Patterns](#matching-multiple-patterns)
  - [Matching Ranges](#matching-ranges)
  - [Destructuring with Patterns](#destructuring-with-patterns)
    - [Destructuring Structs](#destructuring-structs)
    - [Destructuring Enums](#destructuring-enums)
    - [Destructuring Nested Structs and Enums](#destructuring-nested-structs-and-enums)
    - [Destructuring Structs and Tuples](#destructuring-structs-and-tuples)
  - [Ignoring Values in a Pattern](#ignoring-values-in-a-pattern)
    - [Ignoring an Entire Value with `_`](#ignoring-an-entire-value-with-_)
    - [Ignoring Parts of a Value with `_`](#ignoring-parts-of-a-value-with-_)
    - [Ignoring Unused Variables with the `_` Prefix](#ignoring-unused-variables-with-the-_-prefix)
    - [Ignoring Remaining Parts of a Value with `..`](#ignoring-remaining-parts-of-a-value-with-)
  - [Match Guards](#match-guards)
  - [Bindings with `@`](#bindings-with-)

## Overview

Patterns and pattern matching are fundamental features of _idiomatic Rust_. While I briefly introduced them in my [earlier notes](https://github.com/I-Do-CS/rust-notes/blob/main/04.enums/enums.md#patterns), this chapter explores them in greater depth. We'll examine what patterns are, where they can be used throughout the language, how Rust matches values against them, and the powerful techniques they enable for writing expressive, concise code.

## What Are Patterns?

A **pattern** is Rust's way of describing the _shape_ of a value. Rather than referring to a specific value directly, a pattern specifies what a value should look like in order to match. A pattern can match something as simple as a single literal or variable, or something much more complex, such as a struct, tuple, enum variant, or a deeply nested combination of these.

Patterns are also capable of binding parts of a matched value to new variables, ignoring values that are not needed, and expressing multiple possible matches. This makes them a concise and expressive way to inspect and decompose data.

For example, in the pattern `(x, y)`, Rust expects a two-element tuple. If the value is `(10, 20)`, the pattern matches successfully and binds `10` to `x` and `20` to `y`.

## What is Pattern Matching?

**Pattern matching** is the process of comparing a value against one or more patterns to determine whether its structure satisfies a particular pattern. If a match succeeds, Rust can extract values from the matched data, bind them to variables, and execute code associated with that pattern.

Pattern matching is deeply integrated into Rust. Although it is most commonly associated with the `match` expression, it also appears in many other parts of the language, including `let` statements, `if let`, `while let`, function parameters, `for` loops, and closure parameter lists. Together, patterns and pattern matching provide a uniform mechanism for inspecting, destructuring, and working with data throughout Rust.

For example, when matching an `Option<i32>`, the pattern `Some(value)` matches only if the value is the `Some` variant, binding the contained integer to `value`. If the value is `None`, the pattern does not match.

```rust
let x = Some(42);

match x {
    Some(value) => println!("{value}"),
    None => println!("No value"),
}
```

## All the Places We Can Use Patterns

This section explores all the places where patterns are valid and can be used throughout Rust.

Don't worry if the _matching_ aspect of some examples doesn't make sense yet; we'll cover that shortly when we discuss _refutability_ and _pattern matching_.

For each construct below, I've also noted whether it requires an **irrefutable pattern**. This will become useful after reading the section on refutability.

### `match` Arms

Requires Irrefutable Patterns: `False`

The most common place to use patterns is in the arms of a `match` expression. Each arm contains a pattern that is compared against the matched value. The first pattern that matches is selected, and its corresponding code is executed.

```rust
let number = 2;

match number {
    1 => println!("One"),
    2 => println!("Two"),
    _ => println!("Something else"),
}
```

### `let` Statements

Requires Irrefutable Patterns: `True`

Patterns can also appear in `let` statements to destructure values as they are bound to variables. This allows complex values to be split into their individual components in a single statement.

```rust
let (x, y) = (10, 20);

println!("{x}, {y}");
```

### Conditional `if let` Statements

Requires Irrefutable Patterns: `False`

The `if let` construct is a convenient way to match a single pattern without writing a full `match` expression. The body executes only if the value matches the specified pattern.

```rust
let value = Some("Rust");

if let Some(language) = value {
    println!("{language}");
}
```

### `let...else` Expressions

Requires Irrefutable Patterns: `False`

A `let...else` expression requires a value to match a pattern. If the match succeeds, execution continues normally; otherwise, the `else` block must diverge (for example, by returning or panicking).

```rust
let Some(name) = Some("Alice") else {
    return;
};

println!("{name}");
```

### `while let` Conditional Loops

Requires Irrefutable Patterns: `False`

A `while let` loop repeatedly matches a pattern and continues looping as long as the match succeeds. It is commonly used when processing values until a particular condition is reached (aka. pattern doesn't match anymore).

```rust
let mut stack = vec![1, 2, 3];

while let Some(value) = stack.pop() {
    println!("{value}");
} // Stops once .pop() returns None
```

### `for` Loops

Requires Irrefutable Patterns: `True`

Patterns can be used in `for` loops to destructure each item produced by an iterator as it is iterated over.

```rust
let points = [(1, 2), (3, 4), (5, 6)];

for (x, y) in points {
    println!("({x}, {y})");
}
```

### Function Parameters

Requires Irrefutable Patterns: `True`

Function parameters are themselves patterns. This allows values passed to a function to be destructured immediately instead of inside the function body.

```rust
fn print_point((x, y): (i32, i32)) {
    println!("({x}, {y})");
}
```

### Closure Parameter Lists

Requires Irrefutable Patterns: `True`

Like function parameters, closure parameters are patterns. This makes it possible to destructure arguments directly as they are received by the closure.

```rust
let points = [(1, 2), (3, 4), (5, 6)];

points.iter().for_each(|&(x, y)| {
    println!("({x}, {y})");
});
```

## Irrefutability vs Refutability

Every pattern in Rust falls into one of two categories: **irrefutable** or **refutable**. The distinction is based on a single question:

> **Can this pattern ever fail to match?**

An **irrefutable pattern** is one that is guaranteed to match every possible value of the expected type. Since it can never fail, Rust knows execution can safely continue after the pattern is applied.

For example, the pattern `x` is irrefutable because it matches any value by simply binding that value to the variable `x`. Likewise, the tuple pattern `(x, y)` is irrefutable when matching a two-element tuple, because every value of type `(T1, T2)` necessarily has exactly two elements.

```rust
let x = 42; // Never fails
let (x, y) = (10, 20); // Never fails
```

A **refutable pattern**, on the other hand, only matches values with a particular shape. If the value has a different shape, the match fails.

For example, the pattern `Some(x)` only matches values of the form `Some(...)`. If the value is `None`, the pattern does not match.

```rust
let value = Some(42);

// This succeeds
if let Some(x) = value {
    println!("{x}");
}

let value = None;

// This fails
if let Some(x) = value {
    println!("{x}");
}
```

The distinction between refutable and irrefutable patterns determines **where a pattern is allowed to appear**. Constructs such as `let` statements, function parameters, and `for` loops require **irrefutable patterns**, because there is no meaningful way for execution to continue if the pattern were allowed to fail. Since these constructs don't provide a mechanism for handling failure, Rust requires patterns that are guaranteed to succeed.

By contrast, constructs such as `match`, `if let`, `while let`, and `let...else` are specifically designed to handle the possibility of a failed match. They therefore accept **refutable patterns**, allowing your program to take different actions depending on whether the pattern matches.

Attempting to use a refutable pattern where an irrefutable one is required results in a compile-time error.

```rust
let value: Option<i32> = Some(42);

let Some(x) = value; // Error!
```

The compiler rejects this because `value` could have been `None`, meaning the pattern `Some(x)` is not guaranteed to match. In situations like this, you should instead use a construct that is capable of handling failure, such as `if let` or `let...else`.

## Pattern Matching 101

Now that we've covered where patterns can be used and the distinction between refutable and irrefutable patterns, it's time to learn the pattern syntax itself. In this section, we'll gradually build up from the simplest patterns to more powerful ones, eventually covering destructuring, match guards, and bindings.

### Matching Literals

The simplest kind of pattern is a **literal pattern**, which matches a single, exact value. A literal pattern succeeds only if the matched value is equal to that literal.

```rust
let x = 2;

match x {
    1 => println!("One"),
    2 => println!("Two"),
    3 => println!("Three"),
    _ => println!("Something else"),
}
```

Here, the pattern `2` matches because `x` is exactly `2`. Had `x` been any other value, Rust would have continued checking the remaining patterns until one matched.

### Matching Named Variables

A variable in a pattern doesn't compare against an existing variable; it **creates a new binding**. Consequently, a variable pattern matches any value and binds that value to the variable.

```rust
let x = Some(5);

match x {
    Some(value) => println!("The value is {value}"),
    None => println!("No value"),
}
```

In this example, the pattern `Some(value)` matches any `Some(...)` variant and binds its contents to the newly created variable `value`.

Be careful when using variable patterns inside a `match`: if a variable with the same name already exists, the pattern creates a **new variable that shadows the existing one** rather than comparing against it.

```rust
let x = Some(5);
let y = 10;

match x {
    Some(y) => println!("Matched, y = {y}"),
    None => {}
}

println!("Outer y = {y}");
```

This prints:

```text
Matched, y = 5
Outer y = 10
```

The `y` inside the `match` is a new binding that temporarily shadows the outer `y`.

### Matching Multiple Patterns

Sometimes several patterns should produce the same result. Instead of duplicating code, Rust allows multiple patterns to be combined using the `|` operator (pronounced **or**). The match succeeds if any of the listed patterns match.

```rust
let x = 2;

match x {
    1 | 2 => println!("One or two"),
    3 => println!("Three"),
    _ => println!("Something else"),
}
```

This is equivalent to writing separate arms for `1` and `2`, but is shorter and avoids repetition.

### Matching Ranges

Rust also allows patterns to match an inclusive range of values using the `..=` operator. Range patterns are supported for numeric types and `char`s because those values have a well-defined ordering known at compile time.

```rust
let x = 4;

match x {
    1..=5 => println!("Between one and five"),
    _ => println!("Something else"),
}
```

The pattern `1..=5` matches any integer from `1` through `5`, inclusive.

Range patterns work equally well with characters.

```rust
let c = 'm';

match c {
    'a'..='z' => println!("Lowercase letter"),
    'A'..='Z' => println!("Uppercase letter"),
    _ => println!("Something else"),
}
```

Since `'m'` falls within the range `'a'..='z'`, the first arm is selected.

Multiple patterns and ranges can also be combined naturally.

```rust
let x = 10;

match x {
    1 | 3 | 5 => println!("An odd number"),
    10..=20 => println!("Between ten and twenty"),
    _ => println!("Something else"),
}
```

The pattern `10..=20` matches because `10` lies within the specified range.

### Destructuring with Patterns

One of the greatest strengths of patterns is their ability to **destructure** values; that is, to break complex data types into their individual components. Rather than accessing fields or elements one at a time, a pattern can extract everything you need in a single operation.

Destructuring works with many of Rust's compound types, including structs, enums, tuples, and even combinations of these. As you'll see, patterns can be nested arbitrarily, allowing you to work with deeply structured data in a concise and expressive way.

#### Destructuring Structs

Structs are destructured by writing a pattern that mirrors the struct's fields. Each matched field can either be bound to a new variable or ignored if it isn't needed.

```rust
struct Point {
    x: i32,
    y: i32,
}

let point = Point { x: 10, y: 20 };

match point {
    Point { x, y } => {
        println!("x = {x}, y = {y}");
    }
}
```

When the variable name is the same as the field name, Rust allows the shorthand `x` instead of `x: x`.

If you'd like to bind a field to a variable with a different name, you can do so explicitly.

```rust
struct Point {
    x: i32,
    y: i32,
}

let point = Point { x: 10, y: 20 };

match point {
    Point { x: x_coord, y: y_coord } => {
        println!("({x_coord}, {y_coord})");
    }
}
```

You can also match specific field values while binding others.

```rust
struct Point {
    x: i32,
    y: i32,
}

let point = Point { x: 0, y: 7 };

match point {
    Point { x: 0, y } => println!("On the y-axis at {y}"),
    Point { x, y: 0 } => println!("On the x-axis at {x}"),
    Point { x, y } => println!("({x}, {y})"),
}
```

#### Destructuring Enums

Enum variants are destructured by writing a pattern for the specific variant you wish to match. If the variant contains associated data, that data can be extracted directly into variables.

```rust
enum Message {
    Quit,
    Write(String),
    Move { x: i32, y: i32 },
}

let message = Message::Move { x: 5, y: 10 };

match message {
    Message::Quit => println!("Quit"),
    Message::Write(text) => println!("{text}"),
    Message::Move { x, y } => println!("Move to ({x}, {y})"),
}
```

Notice that each variant has its own shape. The pattern must mirror that shape in order to match successfully.

#### Destructuring Nested Structs and Enums

Patterns can be nested, allowing you to destructure values inside other values. Each level of the pattern mirrors the structure of the data being matched.

```rust
enum Color {
    Rgb(u8, u8, u8),
}

enum Message {
    ChangeColor(Color),
}

let message = Message::ChangeColor(Color::Rgb(255, 0, 0));

match message {
    Message::ChangeColor(Color::Rgb(r, g, b)) => {
        println!("Red = {r}, Green = {g}, Blue = {b}");
    }
}
```

Even though the value consists of multiple nested types, the entire structure is matched and destructured in a single pattern.

#### Destructuring Structs and Tuples

Patterns are not limited to a single data type. Since patterns describe the shape of a value, they can freely combine structs, tuples, enums, and other compound types whenever those types are nested inside one another.

```rust
struct Point {
    x: i32,
    y: i32,
}

let ((a, b), Point { x, y }) = (
    (1, 2),
    Point { x: 3, y: 4 },
);

println!("{a}, {b}, {x}, {y}");
```

The pattern mirrors the structure of the value exactly: first a tuple containing another tuple and a struct, then patterns that destructure each of those values into their individual components.

### Ignoring Values in a Pattern

Sometimes you're only interested in part of a value. In these situations, Rust allows you to ignore the fields, elements, or bindings that you don't need. This keeps patterns concise while making it clear which parts of a value are actually relevant.

Rust provides several ways to ignore values, each suited to a slightly different situation.

#### Ignoring an Entire Value with `_`

The underscore (`_`) is a special pattern that matches **any value** without binding it to a variable. It is commonly used when you don't care about the matched value at all.

```rust id="pkj2wd"
let x = Some(5);

match x {
    Some(_) => println!("Got a value!"),
    None => println!("No value."),
}
```

Even though the `Some` variant contains a value, the pattern ignores it completely.

You'll often see `_` used as the final arm of a `match` expression to handle every remaining case.

```rust id="x9b47m"
let number = 42;

match number {
    1 => println!("One"),
    2 => println!("Two"),
    _ => println!("Something else"),
}
```

Here, `_` acts as a catch-all pattern that matches anything not handled by the previous arms.

#### Ignoring Parts of a Value with `_`

The `_` pattern can also be used inside larger patterns to ignore only specific fields or elements while extracting the ones you care about.

```rust id="wfj3ba"
let point = (3, 7);

match point {
    (x, _) => println!("x = {x}"),
}
```

Only the first element is bound to `x`; the second is ignored.

The same idea applies when destructuring structs and enums.

```rust id="g4e8tz"
struct Point {
    x: i32,
    y: i32,
}

let point = Point { x: 5, y: 10 };

match point {
    Point { x, y: _ } => println!("x = {x}"),
}
```

#### Ignoring Unused Variables with the `_` Prefix

A variable whose name begins with an underscore is still **bound** to the matched value, but Rust suppresses the warning that the variable is unused.

```rust
let s = Some(String::from("hello"));

if let Some(_text) = s {
    println!("Got a string!");
}
```

This differs from the plain `_` pattern. The pattern `_text` creates a variable, whereas `_` does not.

This distinction matters because binding a value can affect ownership.

```rust id="h2sk0x"
let s = Some(String::from("hello"));

if let Some(_) = s {
    println!("Matched!");
}

println!("{s:?}");
```

Since `_` doesn't bind the `String`, `s` is still available after the match.

By contrast:

```rust id="v4p7gs"
let s = Some(String::from("hello"));

if let Some(_text) = s {
    println!("Matched!");
}

// println!("{s:?}"); // Error: `s` was partially moved.
```

Although `_text` is never used, it still takes ownership of the `String`, causing `s` to be partially moved.

#### Ignoring Remaining Parts of a Value with `..`

When a value contains many fields or elements, writing `_` for each one can become tedious. The `..` pattern allows you to ignore **all remaining parts** of a value that aren't explicitly matched.

For example, suppose we only care about the first and last elements of a tuple.

```rust
let numbers = (1, 2, 3, 4, 5);

match numbers {
    (first, .., last) => {
        println!("first = {first}, last = {last}");
    }
}
```

Likewise, when destructuring a struct, `..` can ignore every field that isn't explicitly listed.

```rust id="tm6f8n"
struct Point3D {
    x: i32,
    y: i32,
    z: i32,
}

let point = Point3D {
    x: 1,
    y: 2,
    z: 3,
};

match point {
    Point3D { x, .. } => println!("x = {x}"),
}
```

The `..` pattern is especially useful when working with large structs or tuples, allowing you to focus only on the fields that matter while ignoring everything else.

### Match Guards

A **match guard** is an additional condition that can be attached to a `match` arm using the `if` keyword. The guard is evaluated **only after the pattern itself has successfully matched**. The arm is selected only if both the pattern and the guard evaluate to `true`.

Match guards are useful when a pattern alone cannot express the desired matching logic.

```rust
let num = Some(4);

match num {
    Some(x) if x % 2 == 0 => println!("{x} is even"),
    Some(x) => println!("{x} is odd"),
    None => println!("No value"),
}
```

Here, both arms use the pattern `Some(x)`. The first arm is selected only when the additional condition `x % 2 == 0` is satisfied. Otherwise, Rust continues checking the remaining arms.

Because guards are arbitrary boolean expressions, they can reference variables introduced by the pattern as well as variables from the surrounding scope.

```rust
let x = Some(5);
let y = 5;

match x {
    Some(n) if n == y => println!("Matched!"),
    _ => println!("No match."),
}
```

Notice that the pattern introduces the variable `n`, while `y` comes from the surrounding scope. The guard compares the two values after the pattern has matched.

One important consequence of using a match guard is that the compiler can no longer exhaustively analyze the guarded arm. As a result, you'll often need an additional arm to handle the remaining cases.

### Bindings with `@`

The `@` operator allows you to **simultaneously test a value against a pattern and bind that value to a variable**. Without `@`, you often have to choose between checking a pattern and capturing the matched value.

Consider the following enum:

```rust
enum Message {
    Hello { id: i32 },
}

let msg = Message::Hello { id: 5 };
```

Suppose we want to match IDs between `3` and `7`, while also keeping the matched ID available. The `@` operator makes this straightforward.

```rust
match msg {
    Message::Hello {
        id: id_variable @ 3..=7,
    } => {
        println!("Found an ID in range: {id_variable}");
    }
    Message::Hello { id: 10..=12 } => {
        println!("Found an ID between 10 and 12");
    }
    Message::Hello { id } => {
        println!("Some other ID: {id}");
    }
}
```

The pattern `id_variable @ 3..=7` can be read as:

> Match the pattern `3..=7`, and if it succeeds, bind the matched value to `id_variable`.

Without `@`, you could still determine whether the ID falls within the desired range, but you would have to perform the range check separately after binding the value or duplicate the matching logic. The `@` operator combines both operations into a single, expressive pattern.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
