# Common Collections in Rust

- [Vectors](#vectors)
  - [Vectors vs Arrays vs Slices](#vectors-vs-arrays-vs-slices)
- [Text in Rust](#text-in-rust)
  - [Strings & Indexing](#strings--indexing)
- [Hash Maps](#hash-maps)

## Vectors

In Rust, Vectors are dynamically sized contiguous collections.
They store elements of the same type in heap and can grow or shrink at runtime.

Internally, Vectors are pointers to heap memory with length, and capacity metadata.

```rust
fn main() {
    // Initialized like this
    let mut v1: Vec<i32> = Vec::new();

    // or with the vec![] macro
    let v2: Vec<i32> = vec![1, 2, 3, 4, 5];

    // Reading
    let third: i32 = v2[2];
    let third: Option<&i32> = v2.get(2); // Safer

    // Writing
    v1.push(32);
    v1.push(12);
    v1.push(223);
    v1.push(3456);
    v1.push(44);
    v1.push(5678);

    // Deleting
    let last: Option<i32> = v1.pop(); // Deletes the last item and returns it
    let third: i32 = v1.remove(2); // Removes and returns item at
    // index 2 and shifts the remaining elements to left;

    // Updating
    v1[3] = 0; // By indexing or

    // Getting a mutable reference
    if let Some(mut_ref) = v1.get_mut(2) {
        // dereferencing and updating it
        *mut_ref = 0;
    }
}
```

### Vectors vs Arrays vs Slices

The similarities between Vectors, Arrays, and Slices might be confusing so here's a breakdown:

- Arrays: Arrays have a fixed size that's known at compile time, they are allocated on the stack by default, and cannot grow or shrink.

- Vectors: Vectors have dynamic size, they can grow and shrink, are allocated on the heap so they can be reallocated if they grow or shrink too much for their current chunk of memory.

- Slices: Unlike Arrays and Vectors, a Slice does not own it's content; they are simply borrowed views(references) of other contiguous data. They can reference both Arrays and Vectors.

## Text in Rust

In Rust, text content is stored as UTF-8. UTF-8 is a variable-width Unicode encoding; meaning it allows for different characters to occupy different numbers of bytes.

```rust
let english = "hello";
let russian = "Здравствуйте";
let emoji = "🔥";

println!("{}", emoji.len()); // => 4
```

Text in Rust is stored with the `&str` and `String` types.

- A `String` is an owned, growable UTF-8 string whose contents are stored on the heap.
- A &str is a borrowed view into UTF-8 string data owned by something else.

Internally, strings are basically byte arrays:

```rust
"hello" == [104, 101, 108, 108, 111]
```

But non-ASCII text like "🔥" becomes multiple bytes.
This variable-width encoding makes string indexing and iteration impossible.

### Strings & Indexing

As I mentioned before the variable-length of UTF-8 characters makes it impossible for us to index into rust strings. The same thing is true for iteration. Instead we can do byte access like so:

```rust
let bytes = "hello, world!".as_bytes();

println!("{}", bytes[2]);

for b in bytes {
    // ...
}
```

In the end, strings are pretty complex collection types. Rust does a great job at highlighting that complexity and limiting us in what we can do with them. So, for complex string-related functionality, it's recommended to use already-existing Crates.

## Hash Maps

Hash Maps are Rust's standard implementation of what other languages call a Dictionary. It stores key value pairs.
They provide fast, average-case lookups.
HashMap keys must implement the `Eq` and `Hash` traits.

```rust
use std::collections::HashMap;

fn main() {
    // Instantiated like this
    let mut scores: HashMap<String, i32> = HashMap::new();

    // Writing/Adding
    scores.insert("Math".to_string(), 100);
    scores.insert("History".to_string(), 87);
    scores.insert("Geography".to_string(), 78);

    // Conditional Insertion with the entry API
    // This will get a mutable reference for the key "CS" if it exists
    // if not, it will insert it with the value of 100 then returns a
    // mutable reference to the newly added key-value pair
    let cs_score: &mut i32 = scores.entry("CS".to_string()).or_insert(100);

    // Reading
    let math_score: Option<&i32> = scores.get("Math");
    let history_score: Option<&i32>: Option<&i32> = scores.get("History");
    let geo_score: Option<&i32> = scores.get("Geography");

    // Updating/Overwriting
    if let Some(score_ref) = math_score
    /* Update */
    {
        // *score_ref = 66;
        // This will cause compiler error; but it'd be possible
        // if we used get_mut("Math") above.
    }

    scores.insert("Math".to_string(), 23); // Overwrite

    // Delete
    let removed: Option<i32> = scores.remove("Math");
}
```

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
