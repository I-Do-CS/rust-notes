# Enums, Patterns & Matching

- [Enums](#enums)
- [Patterns](#patterns)
- [The `match` Construct](#the-match-construct)
- [Other Pattern Matching Constructs](#other-pattern-matching-constructs)
  1. [`if let`](#1-if-let)
  2. [`if let ... else`](#2-if-let--else)
  3. [`else if let`](#3-else-if-let)
  4. [`let ... else`](#4-let--else)
  5. [`while let`](#5-while-let)

## Enums

Enums (enumerations) are types that allow a value to be one of several possible variants. They allow you to model states and data safely and explicitly.
Unlike many languages, Rust each variant of an enum can also contain data.

```rust
// Defined like this
enum IpAddrKind {
    V4,
    V6,
}

// Variants can contain data
enum IpAddr {
    V4(u8, u8, u8, u8),
    V6(String),
}

fn main() {
    // Instantiated like this
    let four = IpAddrKind::V4;
    let six = IpAddrKind::V6;

    let home = IpAddr::V4(127, 0, 0, 1);
    let loopback = IpAddr::V6(String::from("::1"));
}
```

### More on variants with data:

There are different styles of enum variants with data. The following snippet highlights them:

```rust
// Enum variants
enum EnumVariants {
    A,                        // This is a Unit-like variant
    B(i32, u8),               // This is a Tuple-like variant
    C(f32),                   // This is also a Tuple-like variant
    D { id: i32, cont: u32 }, // This is a Struct-like variant
}

fn main() {
    let unit_like: EnumVariants = EnumVariants::A;
    let tuple_like1: EnumVariants = EnumVariants::B(1, 2);
    let tuple_like2: EnumVariants = EnumVariants::C(3.14);
    let struct_like: EnumVariants = EnumVariants::D {
        id: 1234,
        cont: 5678,
    };
}
```

## Patterns

Patterns and pattern matching are essential parts of rust and I'll cover them in more depth later; for now, think of patterns as Rust's special way of describing/understanding the shape of data. Once we describe the shape of some data correctly, we can extract or bind it from/to variables and types.

```rust
// `(x, y)` is the pattern here. This reads as:
// "I expect a tuple with two elements,
// the first goes inside of 'x' the second goes in 'y'".
let (x, y) = (10, 20)
```

I'll explain patterns more as I cover `match`, and the other pattern matching constructs.

## The `match` Construct

`match` is Rust’s primary pattern matching construct.
It compares a value against patterns and executes matching branches.
`match` is expression-based and it returns values.

```rust
fn main() {
    let number = 3;

    // Each branch is called an arm.
    match number {
        1 => println!("one"),
        2 => println!("two"),
        3 => println!("three"),
        _ => println!("other"), // Catch-all/Default
    };

    // Can be used with let as well
    let result = match number {
        1 => "one",
        2 => "two",
        3 => "three",
        _ => "other",
    };
}
```

Example with enums:

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

fn main() {
    let msg = Message::Write(String::from("hello"));

    match msg {
        // This is a pattern; it reads as:
        // "if `msg` is the `Quit` variant of the Message enum
        // then do whatever the arrow's pointing to"
        Message::Quit => println!("quit"),

        // This is a pattern; it reads as:
        // "if `msg` is the `Move` variant of
        // the `Message` enum containing `x` and `y`,
        // pass x and y into the incoming scope"
        Message::Move { x, y } => {
            println!("{x}, {y}");
        }

        // Also a pattern, you get the gist.
        Message::Write(text) => {
            println!("{text}");
        }
    }
}

```

- One very important rule to note is the fact that `match`es are exhaustive; meaning, all possible cases should be either covered individually or with a `_`.

## Other Pattern Matching Constructs

In Rust, there are other pattern matching constructs aside from `match`; mainly for reducing boilerplate.

### 1. `if let`

Useful for when we only care about one pattern and don't want to cover every case.

```rust
let cmd = Command::Delete(42);

// With match
match cmd {
    Command::Delete(id) => {
        println!("Deleting {id}");
    }
    _ => {}
};

// With if let
if let Command::Delete(id) = cmd {
     println!("Deleting {id}");
}
```

`let Command::Delete(id)` is a pattern; it's describing a value that's the `Delete()` variant of the `Command` enum.
`if let Command::Delete(id) = cmd` checkswhether the `cmd`
variable matches the pattern we described. If it does, the field inside its `Delete(...)` is bound to the `id` variable and the block executes.

### 2. `if let ... else`

Used to handle the `else` case when the `if let` block doesn't execute.

```rust
let name = None;

// With match
match name {
    Some(name) => println!("Hello {name}"),
    None => println!("No name found"),
}

// With if let ... else
if let Some(name) = name {
    println!("Hello {name}");
} else {
    println!("No name found");
}
```

### 3. `else if let`

Used to check multiple patterns.

```rust
let cmd = Command::Read(10, 1);

// With match
match cmd {
    Command::Write => println!("Write"),
    Command::Read(fd, flags) => println!("Read: {fd}, {flags}"),
    _ => println!("Something else"),
}

// with else if let
if let Command::Write = cmd {
    println!("Write");
} else if let Command::Read(fd, flags) = cmd {
    println!("Read: {fd}, {flags}");
} else {
    println!("Something else");
}
```

### 4. `let ... else`

Similar to `if let ... else` but often cleaner.

```rust
// With match
fn greet1(name: Option<&str>) {
    match name {
        Some(name) => println!("hello {name}"),
        None => println!("hello stranger"),
    };
}

// With if let ... else
fn greet2(name: Option<&str>) {
    if let Some(name) = name {
        println!("hello {name}");
    } else {
        println!("hello stranger");
    }
}

// With let ... else
fn greet3(name: Option<&str>) {
    let Some(name) = name else {
        println!("hello stranger");
        return;
    };

    println!("hello {name}");
}
```

### 5. `while let`

Used when we want a loop to run as long a pattern matches.

```rust
let mut stack = vec![1, 2, 3, 4, 5];

while let Some(value) = stack.pop() {
    println!("{value}");
} // Stops running when there are no more values to pop
```

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
