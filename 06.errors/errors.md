# Error Handling

- [Error Handling in Software](#error-handling-in-software)
- [Error Handling with Rust](#error-handling-with-rust)
- [Unrecoverable Errors with `panic!()`](#unrecoverable-errors-with-panic)
- [Recoverable Errors with Result<T, E>](#recoverable-errors-with-resultt-e)
  - [`unwrap()` & `expect()`](#unwrap--expect)
- [Error Propagation](#error-propagation)
- [To `panic!()` or not to `panic!()`](#to-panic-or-not-to-panic)

## Error Handling in Software

Error handling is the process of detecting, representing, and responding to failures or unexpected situations in a program.

Real software constantly encounters problems such as, files not existing, network requests failing, invalid inputs, etc. Without error handling, programs would regularly crash, corrupt data, and so on. This is why good error handling is a must.

## Error Handling with Rust

Rust strongly emphasizes explicit error handling.
Instead of hiding failures through exceptions or nulls, Rust typically encodes failure directly in types such as `Result<T, E>`.

Rust Generally separates errors in two different categories.

1. Unrecoverable Errors
2. Recoverable Errors

## Unrecoverable Errors with `panic!()`

The `panic!()` macro is Rust’s mechanism for unrecoverable errors.
It immediately stops normal execution because the program has reached an invalid or impossible state.

A `panic!` is commonly caused by out-of-bounds indexing, failed assertions, unwrap() on invalid values, or explicitly `panic!`ing.

```rust
panic!("Something went terribly wrong!");
```

Unlike other languages like C, this:

```rust
let v = vec![1, 2, 3];
v[99];
```

panics because the index is invalid.
Rust chooses safety over undefined behavior.

When a panic occurs:

1. The current thread begins termination
2. Cleanup may occur based onwhether it was configured (in Cargo.toml) as `unwind` or `abort`
3. An error message is printed
4. Optionally a backtrace is shown

# Recoverable Errors with Result<T, E>

The Result<T, E> enum is Rust's primary recoverable error type. It has two variants:

1. `Ok(T)`: Represents success and contains a value.
2. `Err(E)`: Represents failure and contains the error.

```rust
fn divide(a: f64, b: f64) -> Result<f64, String> {
    if b == 0.0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

fn main() {
    match divide(10.0, 2.0) {
        Ok(value) => println!("{value}"),
        Err(error) => println!("{error}"),
    }
}
```

It's also possible to branch differently based on the kind of error:

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let result = File::open("hello.txt");
    match result {
        Ok(file) => file,

        Err(error) => match error.kind() {
            ErrorKind::NotFound => {
                panic!("File not found");
            }

            other_error => {
                panic!("Problem opening file: {other_error:?}");
            }
        },
    };
}
```

### `unwrap()` & `expect()`

`unwrap()` and `expect()` are two methods that are used to extract success values from the `Option<T>` and the `Result<T, E>` types. Here, we'll focus on Result<T, E>:

- `unwrap()`

```rust
fn main() {
    let x: Result<i32, &str> = Ok(5);
    println!("{}", x.unwrap()); // => 5

    let x: Result<i32, &str> = Err("error");
    println!("{}", x.unwrap()); // panics
}
```

- `expect()` does pretty much the same thing but with custom messages

```rust
fn main() {
    let x: Result<i32, &str> = Ok(5);
    println!("{}", x.expect("Oops! Something went wrong!")); // => 5

    let x: Result<i32, &str> = Err("error");
    println!("{}", x.expect("Oops! Something went wrong!")); // panics with custom message
}
```

- Please note that both `unwrap()` and `expect()` will panic if used on a `Err` or `None`. For recoverable errors, proper error handling/propagation is preferred.

## Error Propagation

Propagating an error is when instead of handling it locally (in the current scope), we return that error to the caller (the outer scope) to handle.
This allows the higher level code to decide how to handle the error.

Obviously we have to propagate errors from inside of functions/methods; in other words, we return `Result<T, E>` and let the calling scope deal with it. here's an example:

```rust
use std::fs::File;
use std::io::{self, Read};

// Note that we're returning the Result<T, E> type.
fn read_username() -> Result<String, io::Error> {
    let file_result = File::open("hello.txt");

    let mut file = match file_result {
        Ok(f) => f,
        // We return an error to the caller if the operation fails
        Err(e) => return Err(e),
    };

    let mut username = String::new();
    match file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        // We do the same here
        Err(e) => Err(e),
    }
}

fn main() {
    let result = read_username();
    let username = match result {
        Ok(value) => value,
        Err(error) => {
            println!("Failed to read username with error: {error}");
            println!("Exiting...");
            return; // Terminate program
        }
    };
}
```

Rust provides the `?` operator for concise error propagation. When used on a `Result<T, E>`, `?` returns the value inside `Ok(T)` if the operation succeeds. If the result is `Err(E)`, it immediately returns that error from the current function.

```rust
// Same function as above but with the ? operator
fn read_username() -> Result<String, io::Error> {
    // If this fails, the error is propagated to the caller
    // without us having to cover such case in a match construct.
    let mut file = File::open("hello.txt")?;
    let mut username = String::new();

    // Same thing applies here
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

## To `panic!()` or not to `panic!()`

Obviously `panic!`ing at every error is not the most optimal way to handle them. A `panic!` is supposed to represent an absolutely unrecoverable state, bug, failure, etc; not ordinary failures that are expected and are recoverable.

In cases where we encounter unrecoverable errors, `panic!`s help us avoid data/state corruption, security vulnerabilities, and so one; but most other cases should be handled gracefully.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
