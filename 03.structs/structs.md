# Structuring Related Data with Structs

- [Structures](#structures)
- [Methods](#methods)
- [Associated Functions](#associated-functions)
- [Ownership & Structs](#ownership--structs)

## Structures

Structs are custom data types that let you group related data together under a single type.
They are similar to structs/classes in other languages, but Rust structs only store data by themselves. Behavior is added separately through impl blocks.

There are three types of structs in Rust:

1. Normal Structs:

```rust
// A normal struct is defined using the struct keyword
struct User {
    username: String,
    email: String,
    active: bool,
    sign_in_count: u64,
}

fn main() {
    let username = String::from("ali");

    // Instantiated like this
    let user1 = User {
    username, // field init shorthand syntax
    email: String::from("example@email.com"),
    active: true,
    sign_in_count: 1,
    };

    // Properties are accessed with the . syntax
    println!("{}", user1.email);

    // Struct .. syntax
    let user2 = User {
        email: String::from("example@email.com"),
        ..user1 // *moves* the rest from user1 to here
    }

}
```

2. Tuple-Structs:
   Tuple structs are structs whose fields are unnamed.
   They behave like tuples but create a distinct type.
   They're kind of like "Positional Records" in C# but the fields are unnamed.

```rust
// This is how we define tuple-structs
struct Rgb(u8, u8, u8);
struct Point(i32, i32, i32);

fn main() {
    // Instantiated like this:
    let black: Rgb = Rgb(0, 0, 0);
    let origin: Point = Point(0, 0, 0);

    // Access happens the same as it happens with tuples
    let r = black.0;
    let g = black.1;
    let b = black.2;
}
// Useful for when field names are unnecessary
// or you want lightweight wrappers around your values.
struct UserId(u64);
struct ProductId(u64);
```

3. Unit-Like Structs:
   Unit-like structs are structs with no fields; useful to use as marker types, zero sized types, or implementing traits without storing state, etc. They're called unit-like because they resemble the unit type "()".

```rust
// Defined like this
struct Logger;
impl Logger { // More on this in the next section
    fn log(&self, msg: &str) {
        println!({msg});
    }
}

fn main() {
    // Instantiated like this
    let logger = Logger;
    logger.log("hello, world!");
}
```

## Methods

Methods are functions associated with a type. They are defined inside impl blocks, take "self" as the first parameter, and are called with the dot '.' syntax.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}
impl Rectangle {
    fn area(&self) -> u32 {
        return self.width * self.height;
    }
}

fn main() {
    let rect = Rectangle {
        width: 30,
        height: 50,
    };

    println!("The rectangle's area is {}", rect.area());
}

```

### More on "self":

"self" is shorthand for "self: Self". The "Self" type itself is a placeholder for whatever type comes after the impl block.

```rust
struct Point(i32, i32, i32)
impl Point {
    fn print_point(self: &Self) /*Self = Point*/ {
        println!("{} {} {}", self.0, self.1, self.2);
    }
}
```

Rust supports three common forms of self:

```rust
struct Example {
    x: i32,
    y: i32,
}

impl example {
    // 1. Ownership consuming self
    fn method1(self) {};

    // 2. Immutable borrow
    fn method2(&self) {};

    // 3. Mutable borrow
    fn method3(&mut self) {};
}

fn main() {
    let e = Example {
        x: 12,
        y: 34,
    };

    // e becomes invalid after this. Because it was
    // moved and freed inside of method1().
    e.method1();
}
```

## Associated Functions

Associated functions are functions tied to a type but without a self parameter. Associated functions are commonly used as constructors, utility functions, conversion helpers, factory methods, etc. They are similar to static methods in C#.

```rust
struct Rectangle {
    width: u32,
    height: u32,
}
impl Rectangle {
    // Takes no self
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    // Called like this
    let sq = Rectangle::square(10);
    // Instead of something sq.square()
}
```

## Ownership & Structs

Structs fully participate in Rust ownership rules.
Each field inside a struct owns its data unless the field itself is a reference.

```rust
struct User {
// Here, the struct owns the String.
    username: String,
}

fn main() {
    // When the struct is dropped, all owned
    // fields are dropped automatically.
    let user1 = User {
    username: String::from("ali"),
    };

    // Moving a struct moves all non-Copy fields;
    // after this point user1 is invalid
    // because ownership moved it to user2.
    let user2 = user1;

} // username String is dropped here
```

An important detail is partial moves.

```rust
struct Person {
    name: String,
    age: u32,
}

fn main() {
    let person = Person {
        name: String::from("Ali"),
        age: 22,
    };

    // Here, person.name is moved out.
    let name = person.name;

    // But this works with no error because
    // u32 implements Copy
    println!("{}", person.age);

    // As expected this does not work since
    // the person struct was partially moved.
    // aka. person does not own the value inside of
    // name property anymore.
    println!("{:?}", person);
}
```

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
