# Validating References with Lifetimes

- [Lifetimes](#lifetimes)
- [The Borrow Checker](#the-borrow-checker)
- [Lifetime Annotation Syntax](#lifetime-annotation-syntax)
  - [Lifetime Annotation with Structs](#lifetime-annotation-with-structs)
- [Lifetime Elision](#lifetime-elision)
- [Important Things to Note](#important-things-to-note)
- [Traits Bounds, Generic Types, & Lifetime Annotations](#traits-bounds-generic-types--lifetime-annotations)

## Lifetimes

Lifetimes describe how long references are valid.
They're the system Rust uses to reason about these relationships;
their purpose is to prevent dangling references.

A dangling reference happens when a reference points to data but the data has already been dropped.
Rust prevents this entirely at compile time with the help of the borrow checker and lifetimes.
Rust often infers lifetimes automatically, but sometimes relationships become ambiguous and require explicit annotations which I'll discuss soon.

One important thing to note is that lifetimes do not change how long values live;
they only describe relationships between references so the compiler can verify correctness.
They are primarily a compile-time analysis tool.

## Lifetime Annotation Syntax

Lifetime annotation syntax is used to explicitly describe relationships between references.

```rust
// Lifetime annotations use apostrophe-prefixed names
'a
'b
'my_lifetime
```

Consider the following function:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

This won't compile because the function returns a reference but the compiler doesn't knowwhether it's `x`'s or `y`'s reference.
The borrow checker needs to know how long the returned reference is guaranteed too remain valid (aka. its lifetime). So, we use the annotation syntax to let the compiler know.
Like Types, we can pass generic lifetime parameters to our function/method signatures (as well as to our, structs, enums, etc).

```rust
// This will compile
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

In the above example, just like `T` is to types, `'a` is also a generic parameter but for lifetimes. It could be named anything else as well.
So essentially, what we're doing is that we have a generic lifetime known as `'a`, we don't know how long it lasts; we don't know which scope it comes from either; it's just a generic parameter that we use to annotate the relationship between our arguments and return types.

This:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str { ... }
```

basically means that our return type (`-> &'a str`) lasts at least as long as `x` and `y` (both have the same lifetime here). So, the compiler will know that this code is safe to compile since our returned value won't be freed before `x` and `y`.

Also keep in mind that since our return value could be either `x` or `y`, we _HAVE to_ annotate a lifetime for _BOTH_ of them; but if for example, our function only ever returned `x` we wouldn't have to annotate a lifetime for `y`.

### Lifetime Annotation with Structs

We can also use lifetime annotation with structs with the following syntax:

```rust
struct ExampleStruct<'a> {
    prop1: &'a str
}
```

`prop1` is a reference with lifetime `'a`. Rust guarantees that any ExampleStruct<'a> cannot outlive the data referenced by `prop1`

- Read the the rest of this section after you've read [Lifetime Elision](#lifetime-elision)

When we implement methods on a struct with lifetimes, we use the same syntax as that of generic type parameters. where we declare and use the lifetime parameters depends on whether they’re related to the struct fields or the method parameters and return values.

```rust
impl<'a> ExampleStruct<'a> {
    // Lifetime Elision makes it possible to write methods like this.
    // Where lifetimes are inferred.
    fn example_method(&self, x: &str) -> &str { ... }

    // Without the elision rules, we'd have to write
    // the above method like this
    fn example_method<'b, 'c>(&'b self, x: &'c str) -> &'b str { ... }

}
```

Here's a breakdown of what each lifetime generic represents:

- `'a`: How long the references stored inside ExampleStruct must remain valid.
- `'b`: How long the borrow of `self` lasts for this method call.
- `'c`: How long the reference x remains valid.

- I will explain multiple lifetime parameters and their use-cases later. For now, let's get familiar with the Elision rules.

## Lifetime Elision

The patterns programmed into Rust’s analysis of references are called the _lifetime elision rules_. These aren’t rules for programmers to follow; they’re a set of particular cases that the compiler will consider, and if your code fits these cases, you don’t need to write the lifetimes explicitly.

The compiler uses three rules to figure out the lifetimes of the references when there aren’t explicit annotations. The first rule applies to input lifetimes, and the second and third rules apply to output lifetimes. If the compiler gets to the end of the three rules and there are still references for which it can’t figure out lifetimes, the compiler will stop with an error. These rules apply to `fn` definitions as well as `impl` blocks.

1. The first rule is that the compiler assigns a lifetime parameter to each parameter that’s a reference.

```rust
// The compiler turns this:
fn example(a: &str, b: &str, c: &str) { ... }

// Into something like this:
fn example<'a, 'b, 'c>(a: &'a str, b: &'b str, c: &'c str) { ... }
```

2. The second rule is that, if there is exactly one input lifetime parameter, that lifetime is assigned to all output lifetime parameters.

```rust
// The compiler turns this:
fn example(a: &i32) -> &i32 { ... }

// Into something like this:
fn example<'a>(a: &'a i32) -> &'a i32 { ... }
```

3. The third rule is that, if there are multiple input lifetime parameters, but one of them is `&self` or `&mut self` because this is a method, the lifetime of `self` is assigned to all output lifetime parameters. This third rule makes methods much nicer to read and write because fewer symbols are necessary.

```rust
// The compiler turns this:
fn example(&self, a: &str) -> &str { ... }

// Into something like this:
fn example<'a, 'b>(&'a self, a: &'b str) -> &'a str { ... }
```

- One subtle point: when people say "the compiler turns this into...", that's a useful mental model, but technically the Rust compiler applies the elision rules during lifetime analysis and infers the omitted lifetimes as if those annotations had been written. It doesn't **_"turn"_** source code into something else.

After the compiler applies (or tries to) all three elision rules and there's still ambiguity about lifetimes, it will then ask for explicitness and we'll have to use the Annotation Syntax.

Consider the following:

```rust
// The two following signatures are functionally the same will compile
fn example1(s: &str) -> &str { ... }

fn example2<'a>(s: &'a str) -> &'a str { ... }
```

The lifetime for `example1`'s references is automatically inferred by the compiler using the lifetime elision rules. However, lifetime elision only works in specific cases. When the compiler cannot unambiguously determine the relationship between input and output lifetimes, explicit lifetime annotations are required.

This is why the following function does not compile:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

The function has two input lifetimes (one for `x` and one for `y`), but the return type contains an omitted lifetime. The lifetime elision rules cannot determine whether the returned reference should be tied to `x`'s lifetime or `y`'s lifetime, so the compiler reports an error and requires an explicit lifetime annotation.

## Important Things to Note

Here is a list of list of important things to know in regards to lifetimes that I couldn't fit anywhere else in this file.

- The `'static` lifetime:

  `'static` is the longest possible lifetime. Meaning that a reference remains valid for the entire duration of a program. A good example of this are string literals; string literals are stored in the programs binary; so, they have the `'static` lifetime.

- Multiple generic lifetime parameters:

  In practice, it's uncommon to need more than one explicit lifetime parameter. Most functions either rely on lifetime elision or can be expressed using a single lifetime such as 'a.
  Multiple lifetime parameters become useful when working with multiple references that have independent lifetime relationships. By introducing separate lifetime parameters, we can express that different references may live for different lengths of time without unnecessarily constraining them to share the same lifetime.

- If you're still confused about lifetimes, you probably don't understand Borrowing and the Borrow Checker.
  [This](https://www.reddit.com/r/rust/comments/1ck2716/comment/l2kdij3/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button) comment by [u/kohugaly](https://www.reddit.com/user/kohugaly/) does a great job clarifying things and how lifetimes fit into all this.

- Also, [this video](https://www.youtube.com/watch?v=rAl-9HwD858)
  by [Jon Gjengset](https://www.youtube.com/@jonhoo) is a great watch for anyone who wants to learn lifetimes.

## Traits Bounds, Generic Types, & Lifetime Annotations

Similar to what I did before, I'll provide examples that use a combination of trait bounds, generic types and lifetime annotations to show them in practice and clarify the syntax.

- Generic function with trait bounds and lifetimes:

```rust
use std::fmt::Display;

fn longest_with_announcement<'a, T>(x: &'a str, y: &'a str, ann: T) -> &'a str
where
    T: Display,
{
    println!("Announcement: {ann}");

    if x.len() > y.len() { x } else { y }
}
```

- Generic struct with lifetime:

```rust
struct Holder<'a, T>
where
    T: Display,
{
    value: &'a T,
}
```

- Generic impl block with trait bounds and lifetimes

```rust
use std::fmt::Display;

struct Container<'a, T> {
    item: &'a T,
}

impl<'a, T> Container<'a, T>
where
    T: Display,
{
    fn print(&self) {
        println!("{}", self.item);
    }
}
```

- Returning references with generics and trait bounds

```rust
fn largest<'a, T>(list: &'a [T]) -> &'a T
where
    T: PartialOrd,
{
    let mut largest = &list[0];

    for item in list {
        if item > largest {
            largest = item;
        }
    }

    largest
}
```

- Methods using all three together

```rust
use std::fmt::Display;

struct Pair<'a, T> {
    first: &'a T,
    second: &'a T,
}

impl<'a, T> Pair<'a, T>
where
    T: Display + PartialOrd,
{
    fn largest(&self) -> &'a T {
        if self.first >= self.second {
            self.first
        } else {
            self.second
        }
    }
}
```

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
