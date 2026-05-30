# Common Programming Concepts in Rust

- [Variables](#variables)
- [Shadowing](#shadowing)
- [Types in Rust](#types-in-rust)
  - [Scalar Types](#scalar-types)
  - [Compound Types](#compound-types)
- [Expressions vs Statements](#expressions-vs-statements)
- [Preludes](#preludes)

## Variables

By default, all variables in rust are immutable, meaning that their value cannot be changed afterwards; this encourages safer code. In order to declare variables that can change we have to specify them as such.
Rust also has constants which are pretty similar to consts in other languages; they are declared with the const keyword, must be initialized with constant expressions, are evaluated at compile time, cannot mutate whatsoever etc.

The following snippet show a few example of variable declarations:

```rust
// Standard immutable variable
let var1: i32 = 1234;

// Mutable variable
let mut var2: i32 = 1234;
var2 = 5678;

// Constant variable
const MAX_PLAYERS: u32 = 100;
```

## Shadowing

Shadowing means declaring a new variable with the same name as a previous variable; essentially, _shadowing_ the previous variable. Shadowing also allows us to change the type, mutability of a variable, type conversions, and ownership transitions.

```rust
let x = 5;
let x = x + 1;
let x = x * 2;
println!("{x}"); // Prints 12 to the terminal

let x: String = "Not a number anymore".to_string();
x.push_str("Mutating x"); // Causes compiler error

let mut x: String = x;
x.push_str("Mutating x");
```

## Types in Rust

Rust's type system has two main families of data types; Scalar Types and Compound Types.

### Scalar Types

Scalar types represent a single value. Rust has four main scalar types:

1. Integers:

```rust
let signed32bit: i32 = -1234;
let unsigned32bit: u32 = 1234;

// All available int types:
// i8, i16, i32, i64, i128, isize (Architecture dependant)
// u8, u16, u32, u64, u128, usize (Architecture dependant)
```

2. Floating-point numbers:

```rust
let a32bit_float: f32 = 3.14;
let a64bit_float: f64 = 3.14;
```

3. Booleans:

```rust
let b1: bool = true;
let b2: bool = false;
```

4. Characters:

```rust
// Characters use ''; strings use "".
let letter: char = 'A';
let emoji: char = '🔥';
```

### Compound Types

Compound types group multiple values. Rust has two built-in compound types:

1. Tuples:

```rust
// Tuples can contain values of different types
let person: (&str, i32, bool) = ("Ali", 22, true);

println!(
    "Name: {}, Age: {}, Married: {}",
     person.0, person.1, person.2);
```

2. Arrays:

```rust
// Arrays are contiguous blocks of memory.
// Have fixed size.
// Are stack allocated by default
let numbers: [i32; 5] = [1, 2, 3, 4, 5];
println!("{}", numbers[0]);
```

## Expressions vs Statements

Rust distinguishes between statements and expressions very clearly. A statement performs an action but does not result in (return) a value. An expression on the other hand always results in a value.

```rust
let x = 1;                    // Statement

fn funny_func() -> i32 { 5 }  // Statement

1 + 1                         // Expression

5 + funny_func()              // Expression

// In Rust, this:
5 // no semicolons
// is the same as this:
return 5;
```

## Preludes

A prelude in Rust is a collection of commonly used items that are automatically imported into scope.
The standard prelude allows you to use many common types and traits without manually importing them.

For example:

```rust
// These are available automatically because of the standard prelude.
Vec<T>
Option<T>
Result<T, E>
Drop
Copy

// That’s why this works without imports
let numbers = Vec::new();
```

Under the hood, Rust implicitly imports the standard library's prelude into every module.
The prelude mainly exists for ergonomics. Without it, many common programs would require constant imports.

Traits (more on traits later) in the prelude are especially important because trait methods only work when the trait is in scope.
For example, iterator methods like .map() and .filter() work because iterator traits are in the prelude.

So generally,
The standard library prelude import is automatic.
Library preludes are manual convenience imports; their purpose is reducing repetitive use statements and improving ergonomics.

### Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
