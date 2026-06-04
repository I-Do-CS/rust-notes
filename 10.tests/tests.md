# Writing Automated Tests

- [Overview](#overview)
- [How to Write Tests](#how-to-write-tests)
  - [Checking for Panics with `should_panic`](#checking-for-panics-with-should_panic)
  - [Using Result<T, E> in Tests](#using-resultt-e-in-tests)
- [Controlling How Tests Are Run](#controlling-how-tests-are-run)
- [Test Organization](#test-organization)
  - [Unit Tests](#unit-tests)
  - [Integration Tests](#integration-tests)

## Overview

Like most other languages, Rust provides us with features to write & run tests and testable code. In this file I'll cover these features.
Tests are Rust functions that verify that the non-test code is functioning in the expected manner. The bodies of test functions typically perform the following actions:

- Setting up any needed data or state
- Run the code we're testing
- Assert the results are what we expect

## How to Write Tests

At its simplest, a test in Rust is a function that’s annotated with the `test` attribute.
To change a function into a test function, add the `#[test]` attribute on the line before `fn`.
When you run your tests with the `cargo test` command, Rust builds a test runner binary that runs the annotated functions and reports on whether each test function passes or fails.

```rust
#[cfg(test)] // More on this later
mod tests {
    #[test]
    fn test1 {
        // ...
    }

    #[test]
    fn test2 {
        // ...
    }
}
```

Rust provides us with useful macros to write tests. The most common being:

1. `assert!()`

   ```rust
   #[derive(Debug)]
   pub struct Rectangle {
       width: u32,
       height: u32,
   }

   impl Rectangle {
       pub fn can_hold(&self, other: &Rectangle) -> bool {
           self.width > other.width && self.height > other.height
       }
   }

   #[cfg(test)]
   mod tests {
       #[test]
       fn larger_can_hold_smaller() {
           let larger = Rectangle {
               width: 8,
               height: 7,
           };
           let smaller = Rectangle {
               width: 5,
               height: 1,
           };

           assert!(larger.can_hold(&smaller));
       }
   }
   ```

2. `assert_eq!()`

   ```rust
   pub fn biggest(a: i32, b: i32) -> i32 {
       a.max(b)
   }

   #[cfg(test)]
   mod tests {
       #[test]
       fn biggest_returns_correct() {
           let result = biggest(-100, 100);
           assert_eq!(
               result, 100,
               "biggest must return 100 here but returned {result}"
           );
           // You can pass a custom message as the third parameter to
           // assert!, assert_eq!, and assert_ne!
       }
   }
   ```

3. `assert_ne!()`

   ```rust
   pub fn biggest(a: i32, b: i32) -> i32 {
       a.max(b)
   }

   #[cfg(test)]
   mod tests {
       #[test]
       fn biggest_returns_correct() {
           let result = biggest(-100, 100);
           assert_ne!(
               result, -100,
               "biggest must not return -100 here but did!"
           );
           // You can pass a custom message as the third parameter to
           // assert!, assert_eq!, and assert_ne!
       }
   }
   ```

### Checking for Panics with `should_panic`

We can use the `#[should_panic]` attribute to verify that a test causes a panic.

```rust
pub struct Guess {
    value: i32,
}

impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 || value > 100 {
            panic!("Guess value must be between 1 and 100, got {value}.");
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

The test succeeds if the code inside the test function panics. However, `#[should_panic]` alone only checks that a panic occurred, not why it occurred. As a result, the test may still pass if an unrelated panic happens. To make the test more precise, we can use `#[should_panic(expected = "...")]` to verify that the panic message contains a specific substring:

```rust
pub struct Guess {
    value: i32,
}
impl Guess {
    pub fn new(value: i32) -> Guess {
        if value < 1 {
            panic!("Guess value must be greater than or equal to 1, got {value}.");
        } else if value > 100 {
            panic!("Guess value must be less than or equal to 100, got {value}.");
        }

        Guess { value }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    #[should_panic(expected = "less than or equal to 100")]
    fn greater_than_100() {
        Guess::new(200);
    }
}
```

### Using `Result<T, E>` in Tests

Tests can return a `Result<T, E>` instead of using assertions directly:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_works() -> Result<(), String> {
        let result = add(2, 2);

        if result == 4 {
            Ok(())
        } else {
            Err(String::from("two plus two does not equal four"))
        }
    }
}
```

A test passes when it returns `Ok(())` and fails when it returns an `Err`. This approach also allows the use of the question mark `?` operator, making it convenient to propagate errors in tests.

Note that tests returning a `Result<T, E>` cannot use the `#[should_panic]` attribute. If you need to verify that an operation returns an error, use `assert!(value.is_err())` instead.

## Controlling How Tests Are Run

Similar to `cargo run`, which compiles and runs your program, `cargo test` compiles your code in test mode and runs the generated test binary. Use `cargo test --help` to see Cargo's test options, and `cargo test -- --help` to see options passed directly to the test binary after the `--` separator.
Here are some of the main options:

- Running Tests in Parallel or Consecutively

  By default tests are run in parallel, you can use the following to customize the number of threads. Using 1 makes the tests run consecutively:

  ```bash
  $ cargo test -- --test-threads=1
  ```

- Showing Function Output

  By default, if a test prints something to `stdout` and it passes, its output will be captured by the test library. Only the output of tests that fail will be printed. Use the following to see the output of tests that pass as well.

  ```bash
  $ cargo test -- --show-output
  ```

- Running a Subset of Tests

  The following shows how to run/ignore different tests in different scenarios

  ```bash
  # Run a test by its name
  $ cargo test auth_login_successful

  # Runs any test that starts with auth
  $ cargo test auth_

  # Runs any test that has the '#[ignore]' attribute
    cargo test -- --ignored

  # Runs all tests including ones that are "ignored"
    cargo test -- --include-ignored
  ```

## Test Organization

The Rust community thinks about tests in terms of two main categories: unit tests and integration tests. Unit tests are small and more focused, testing one module in isolation at a time, and can test private interfaces. Integration tests are entirely external to your library and use your code in the same way any other external code would, using only the public interface and potentially exercising multiple modules per test.
Writing both kinds of tests is important to ensure that the pieces of your library are doing what you expect them to, separately and together.

### Unit Tests

The purpose of unit tests is to test each unit of code in isolation from the rest of the code to quickly pinpoint where code is and isn’t working as expected. You’ll put unit tests in the `src` directory in each file with the code that they’re testing. The convention is to create a module named `tests` in each file to contain the test functions and to annotate the module with `cfg(test)` like so:

```
my_project/
├── Cargo.toml
└── src/
    ├── main.rs
    │
    ├── math.rs
    │   ├── add()
    │   ├── subtract()
    │   └── #[cfg(test)]
    │       mod tests
    │
    ├── string_utils.rs
    │   ├── capitalize()
    │   ├── truncate()
    │   └── #[cfg(test)]
    │       mod tests
    │
    └── models.rs
        ├── struct User
        ├── impl User
        └── #[cfg(test)]
            mod tests
```

The `#[cfg(test)]` annotation on the tests module tells Rust to compile and run the test code only when we run `cargo test`, not when we run `cargo build`. This saves compile time when we only want to build the library and saves space in the resultant compiled artifact because the tests are not included. Since unit tests go in the same files as the code, we’ll have to use `#[cfg(test)]` to specify that they shouldn’t be included in the compiled result.

### Integration Tests

In Rust, integration tests are entirely external to your library. They use your library in the same way any other code would, which means they can only call functions that are part of your library’s public API. Their purpose is to test whether many parts of your library work together correctly. Units of code that work correctly on their own could have problems when integrated, so test coverage of the integrated code is important as well. To create integration tests, you first need a tests directory.

We create a tests directory at the top level of our project directory, next to src. Cargo knows to look for integration test files in this directory. We can then make as many test files as we want, and Cargo will compile each of the files as an individual crate.

```
my_project/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── math.rs
│   └── string_utils.rs
│
└── tests/
    ├── math_tests.rs
    ├── string_utils_tests.rs
    └── api_tests.rs
```

```rust
// src/math.rs

pub fn add(a: i32, b: i32) -> i32 {
    a + b
}
```

The line `use my_project::math::add;` imports the _public API_ of our source file and we can test how they behave from the **_"outside"_**.

```rust
// tests/math_tests.rs

use my_project::math::add;

#[test]
fn adds_numbers() {
    assert_eq!(add(2, 3), 5);
}
```

Note that we don't need to annotate our tests in the _`tests/`_ directory with `#[cfg(test)]`; Cargo treats it specially and compiles files in this directory only when we run cargo test.

Also, note that integration tests can't be used for binary crates; only library crates.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
