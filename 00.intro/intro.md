# Introduction

- [The Rust Programming Language](#the-rust-programming-language)
- [Cargo](#cargo)
- [Cargo Terminology](#cargo-terminology)

## The Rust Programming Language

Rust is a systems programming language focused on performance, memory safety, concurrency, and reliability. It was originally created at Mozilla and is now maintained by the Rust Foundation and the open-source community.

Rust is often compared to languages like C and C++ because it gives low-level control over memory and hardware while still producing highly optimized native binaries. However, unlike C/C++, Rust prevents many common bugs at compile time through its ownership and borrowing system. This includes things like null pointer dereferences, use-after-free bugs, double frees, data races, and dangling pointers.

One of Rust’s main goals is “fearless concurrency.” The compiler enforces rules that make multithreaded code much safer than in many traditional systems languages.

## Cargo

Cargo is Rust’s build system and package manager. It is one of the biggest reasons Rust has such a pleasant developer experience.

Cargo handles tooling concerns such as:

- Project creation
- Dependency management
- Building
- Running
- Testing
- Documentation generation
- Publishing crates

```bash
# To create new projects
cargo new my_project

# To create new library
cargo new my_lib --lib

# To build the projects
cargo build

# To build for specific profiles
cargo build --release

# To check wether a code builds successfully without building it
cargo check

# To generate documentation for the current project and its dependencies; the --open flag opens the generated docs in the browser
cargo doc --open
```

Cargo projects have a Cargo.toml manifest file in the root which contains metadata and dependencies about the project and a Cargo.lock file that cargo uses to manage dependency version resolution for reproducible builds.

## Cargo Terminology

In the Rust eco-system and Cargo there are a few overlapping terms that refer to the concept of a unit of code and how these units are organized. The following explains and clarifies them.

1. Package
   - A package is the broadest unit, it encompasses crates and modules.
2. Crate:
   - A Crate is Rust's fundamental compilation unit; when Rust compiles, it compiles crates.
   - There are two types of Crates:
     1. Binary Crate: Binary crates produce executables for our apps.

        ```rust
        // src/main.rs
        fn main() {
            let name = "Ali";
            println!("Hello, {name}");
        }
        ```

     2. Library Crate: A library crate produces reusable code that other crates can use.
        ```rust
        // src/lib.rs
        pub fn greet(name: String) {
            println!("Hello, {name}");
        }
        ```

3. Module:
   - Modules are ways to organize code inside crates; we can use them to create namespaces and split code _inside_ of crates. A crate can have many modules.

So, a Cargo **_package_** can contain many **_binary crates_** but no more than one **_library crate_**; though that single library crate can contain many **_modules_**.

```bash
my_package/     # Package
    Cargo.toml      # Package Manifest
    src/
        lib.rs      # Library Crate
        main.rs     # Binary Crate
        utils.rs        # Module
        auth.rs         # Module
    src/bin/
        tool1.rs    # Binary Crate
        tool2.rs    # Binary Crate
        tool3.rs    # Binary Crate
```

More on Cargo, Crates and managing complex projects later...

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
