# Concurrency in Rust

- [Concurrent Programming](#concurrent-programming)
- [Parallel Programming](#parallel-programming)
- [Concurrency vs Parallelism](#concurrency-vs-parallelism)
- [Threads](#threads)
- [Process vs Thread](#process-vs-thread)
- [Rust's 1:1 Thread Implementation](#rusts-11-thread-implementation)
- [Other Thread Implementations](#other-thread-implementations)
  - [1:1 Threading](#11-threading)
  - [N:1 Threading](#n1-threading)
  - [M:N Threading](#mn-threading)
  - [Green Threads](#green-threads)
- [Why Rust Chose 1:1](#why-rust-chose-11)
- [Spawning Threads and `JoinHandle`s](#spawning-threads-and-joinhandles)
- [Ownership Across Threads](#ownership-across-threads)
- [Message Passing](#message-passing)
- [Channels in Rust](#channels-in-rust)
  - [`channel()`](#channel)
  - [`sync_channel()`](#sync_channel)
  - [Channels for Synchronization](#channels-for-synchronization)
- [Shared-State Concurrency](#shared-state-concurrency)
  - [`Mutex<T>`](#mutext)
    - [`MutexGuard`](#mutexguard)
    - [Mutex Poisoning](#mutex-poisoning)
  - [`Arc<T>`](#arct)
  - [`Arc<Mutex<T>>`](#arcmutext)
- [Message Passing vs Shared-State Concurrency](#message-passing-vs-shared-state-concurrency)
- [`Send` & `Sync`](#send--sync)
- [Deadlocks](#deadlocks)

## Concurrent Programming

Concurrent programming is the practice of structuring a program so that multiple tasks can make progress during the same period of time. The important idea is not that tasks are executing simultaneously, but that they are in progress together. Concurrency primarily improves responsiveness and throughput.

Imagine a web server. One client requests a file, another submits a form, and a third opens a websocket connection. If the server handled each request from start to finish before touching the next one, most of the CPU would spend its time waiting on disk reads and network operations. Concurrency allows the server to begin processing one request, pause when it must wait, work on another request, then return later to the original task.

Concurrency exists because real-world programs rarely perform one uninterrupted stream of CPU work. They spend time waiting for input/output operations, user interactions, network responses, database queries, timers, and many other external events. A concurrent design prevents the program from hanging whenever one task is blocked.

## Parallel Programming

Parallel programming is the practice of executing multiple computations at the same instant.
The key distinction is that parallelism requires actual simultaneous execution. This usually means multiple CPU cores.

Suppose you need to process a billion records; with a single CPU core, the work must happen one piece at a time. Even if the code is concurrent, only one instruction stream executes at any given moment.

With four CPU cores, you might split the records into four chunks and process all four chunks simultaneously. This is parallelism.

## Concurrency vs Parallelism

Essentially, concurrency is **_managing_** many things at once while parallelism is **_doing_** many things at once. A program can be concurrent without being parallel.

Imagine a single-core CPU running a web server. The server handles thousands of connections by switching between tasks whenever one is waiting for network activity. Only one task executes at a time, yet many tasks are in progress. This is concurrency without parallelism.

Most modern systems combine both. A browser, for example, is concurrent because it manages many independent tasks, and parallel because those tasks may run on different CPU cores.

## Threads

A thread is the smallest unit of execution that an operating system can schedule. When we say a program is "running", what is actually running is at least one thread.

Every Rust program begins with a single thread called the main thread. As the program grows, it may create additional threads to perform work concurrently. Each thread has its own execution flow, meaning it can execute different instructions independently of other threads.

## Process vs Thread

A process is an independent running program.
When you launch an application the OS creates a process for it; each process receives its own virtual memory space and OS resources.

A thread exists inside a process. Multiple threads belonging to the same process share most of the process's resources, including its memory. Because they share memory, threads can communicate much more easily than separate processes. However, this shared access also creates the possibility of race conditions, deadlocks, and data corruption.

## Rust's 1:1 Thread Implementation

Rust's standard library uses a 1:1 threading model. This means one Rust thread corresponds directly to one OS thread.

When you call `std::thread::spawn()`, Rust asks the operating system to create a real OS thread. There is no language-level scheduler between Rust threads and OS threads; scheduling is handled entirely by the operating system.

This is an important design decision because languages such as Go, Erlang, and Java's virtual-thread implementations introduce an additional runtime layer that schedules many user-level threads onto a smaller number of OS threads. Rust deliberately avoids this in the standard library, so no language-specific runtime is required for threading.

The downside is that OS threads are relatively expensive. Each thread requires its own stack and operating-system resources, and creating, destroying, and context-switching between large numbers of threads incurs overhead. As a result, creating thousands of OS threads is often impractical.

## Other Thread Implementations

This section goes through alternative thread implementations. Feel free to skip it if you're not interested.

### 1:1 Threading

In a 1:1 model, every user thread corresponds to a real OS thread. This is the model Rust uses, but there are crates available with other thread implementations.

The major advantage is simplicity. If the operating system knows about every thread, the OS scheduler can freely distribute them across CPU cores.
The major disadvantage is cost. Creating and managing large numbers of OS threads consumes memory and scheduler resources.

### N:1 Threading

In an N:1 model, many user threads are mapped onto a single OS thread. Here, the language runtime performs all the scheduling itself; the OS only sees one thread.

This allows user threads to be extremely lightweight because creating a new user thread does not require creating a new OS thread; however, this comes with major limitations:

If one user thread performs a blocking operation, the entire OS thread becomes blocked.
Since all user threads depend on that single OS thread, every thread stops making progress.
Another limitation is that true parallelism is impossible because only one OS thread exists.
Historically, many early green-thread (more on this later) implementations used this approach.

### M:N Threading

In an M:N model, many user threads are scheduled across multiple OS threads and the runtime scheduler decides which user threads run on which OS threads.
This combines many advantages of both previous models:

- User threads remain lightweight.
- Multiple CPU cores can be utilized.
- Blocking one user thread does not necessarily block all others.

The downside is complexity; the runtime must implement an advanced scheduler, handle synchronization with the operating system, manage thread migration, and coordinate work distribution.
Go's goroutine scheduler is a variant of this model.

### Green Threads

A green thread is a thread managed entirely by a language runtime rather than the operating system.
The term originally referred to threads implemented in user space.

Because the runtime controls scheduling, green threads are usually very lightweight, fast to create, and cheap to switch between. Go goroutines are an example of this.

Green threads are not tied to a specific mapping model. Most modern green-thread systems use some variation of M:N scheduling. The important characteristic is simply that the runtime, not the operating system, decides when execution switches between tasks.

## Why Rust Chose 1:1

Rust's philosophy strongly favors giving programmers direct control over system behavior.
A 1:1 threading model aligns with that goal; by relying on OS threads, Rust avoids requiring a large language runtime. This keeps binaries smaller and startup faster. It also allows Rust to integrate naturally with existing OS facilities such as schedulers, debuggers, profilers, and thread-management tools.

Another reason is that Rust's ownership and type systems already solve many concurrency problems at compile time. Rust does not need a specialized runtime scheduler to enforce safety.

The tradeoff is that spawning OS threads remains relatively expensive.
As a result, Rust developers often use, thread pools, async runtimes, work-stealing schedulers, etc when they need to manage extremely large numbers of concurrent tasks.

## Spawning Threads and `JoinHandle`s

In Rust, new threads are created with `std::thread::spawn()`. The function accepts a closure that contains the work that the new thread should perform and returns a `JoinHandle<T>`.

A spawned thread begins executing independently of the thread that created it. Because execution continues immediately after `spawn()` returns, both threads can run concurrently.

The `join()` method can be called on a `JoinHandle` to block the current thread until the spawned thread finishes executing. If the main thread exits before a spawned thread completes, the entire process terminates and any remaining threads are stopped.

```rust
use std::thread;
use std::time::Duration;

fn main() {
    let handle = thread::spawn(|| {
        for i in 1..10 {
            println!("hi number {i} from the spawned thread!");
            thread::sleep(Duration::from_millis(1));
        }
    });

    handle.join().unwrap(); // Block until the spawned thread finishes

    for i in 1..5 {
        println!("hi number {i} from the main thread!");
        thread::sleep(Duration::from_millis(1));
    }

    // We could also place `handle.join().unwrap()` here.
    // In that case, both loops would run concurrently, and
    // the main thread would wait for the spawned thread only
    // after finishing its own work.
}
```

## Ownership Across Threads

In a single-threaded program, ownership is primarily about preventing use-after-free, double frees, and invalid references. Once multiple threads enter the picture, ownership becomes a concurrency safety mechanism as well. Rust must guarantee that data remains valid even when multiple execution flows are running independently.

The `move` keyword forces a closure to take ownership of the values it captures from its environment. Instead of borrowing variables from the surrounding scope, ownership is transferred into the closure itself. Once ownership has moved, the closure becomes responsible for those values. You can use `move` with any closure, even in completely single-threaded code. Thread spawning simply happens to be one of the most common places where ownership transfer is required.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {v:?}");
    });

    handle.join().unwrap();

    println!("Here's a vector: {v:?}"); // compiler error; v was moved
}
```

If multiple threads genuinely need access to the same value, moving ownership into one thread is not sufficient. In that situation, ownership itself must become shareable; we solve this with shared ownership through reference counting (a pattern I'll explain after Message Passing).

## Message Passing

One of Rust's preferred approaches to concurrency is message passing. Instead of allowing multiple threads to directly manipulate the same piece of memory, Rust encourages threads to communicate by sending values to one another. This philosophy comes from a famous idea popularized by languages such as Go and Erlang:

> _"Do not communicate by sharing memory; instead, share memory by communicating."_

Traditionally, concurrent programming often begins with shared memory. Multiple threads are given access to the same data structure, and synchronization primitives such as mutexes are used to prevent corruption. This works, but it also introduces many opportunities for mistakes. Threads can acquire locks in the wrong order, forget to synchronize access, create race conditions, or hold locks for too long.

Message passing approaches the problem from the opposite direction. Instead of multiple threads owning the same data, ownership is transferred between threads as messages. At any given moment, a piece of data typically has one owner. If another thread needs that data, ownership is sent to it. This fits remarkably well with Rust's ownership system because ownership transfer is already a core language concept.

## Channels in Rust

Rust implements message passing through channels. A channel can be thought of as a pipe connecting threads; one end is used to send values, the other is used to receive values. Whenever a value is sent through a channel, ownership moves from the sender to the receiver. This is an extremely important property because it means Rust can maintain its ownership guarantees even when data travels between threads.

Rust's implementation of Channels is MPSC (Multi Producer, Single Consumer); meaning, you can make many clones of your senders and move them to different threads, but only one receiver may exist at a time. This means multiple sending threads may feed values into the same channel, while a single receiving thread consumes them.

The most common channel constructors in `std::sync::mpsc` are `channel()` and `sync_channel()`.

### `channel()`

```rust
use std::{sync::mpsc, thread, time::Duration};

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {received}");
    }
}
```

Creates an unbounded channel. Messages are stored in an internal buffer that can grow as needed, so send() does not block unless the receiver has been dropped.
This is convenient when producers should be able to continue sending messages regardless of how quickly the receiver processes them, however, memory usage can grow if messages are produced faster than they are consumed.

### `sync_channel()`

```rust
use std::{sync::mpsc, thread, time::Duration};

fn main() {
    let (tx, rx) = mpsc::sync_channel(1024);

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {received}");
    }
}

```

Creates a bounded channel with a fixed capacity. Once the buffer is full, further calls to send() block until the receiver removes a message. This introduces backpressure, meaning fast producers are forced to slow down when consumers cannot keep up. Backpressure helps prevent unbounded memory growth and keeps producers and consumers operating at a sustainable rate.

A special case is a capacity of 0:

```rust
let (tx, rx) = std::sync::mpsc::sync_channel(0);
```

This creates a rendezvous channel, where no buffering occurs. Each send operation waits until a receiver is ready to accept the message.

### Channels for Synchronization

As well as being great data-transport mechanisms, Channels are also a great mechanism for synchronization between threads; suppose one thread sends a value and another thread waits to receive it, the receiving thread cannot continue until a message arrives. Therefore, the act of sending and receiving creates a coordination point between threads. This means channels often eliminate the need for explicit locks on data by simply _communicating ownership_. Many concurrent designs become much simpler this way.

## Shared-State Concurrency

Up to this point, we've looked at concurrency through the lens of ownership transfer. Channels work because ownership moves from one thread to another, ensuring there is always a single owner of a piece of data. However, not every problem can be solved by transferring ownership.

Sometimes multiple threads genuinely need access to the same value at the same time (shared cache, global config object, connection pool, etc.) and it would be neither efficient nor practical to keep move ownership of resources between threads; this is where shared-state concurrency enters the picture.

Shared-state concurrency is the traditional concurrency model most programmers are familiar with. Multiple threads share access to the same memory. But this comes with the challenge of ensuring data-integrity. If multiple threads have access to the same value how can we be sure they won't corrupt it?

Rust's solution to this comes in the form of specialized smart pointers; mainly `Arc<T>` for thread-safe shared ownership. and `Mutex<T>` for thread-safe synchronized access. Similar to `Rc<RefCell<T>>` but thread-safe.

### `Mutex<T>`

A mutex (short for mutual exclusion) is a synchronization primitive that ensures only one thread may access protected data at a time.

The fundamental idea is simple; before a thread accesses the protected data, it must acquire the lock.
While the lock is held, every other thread attempting to acquire the lock must wait; when the thread finishes, it releases the lock and another waiting thread may proceed.

The mutex therefore serializes access to shared state. Even though multiple threads may exist, only one thread may access the protected value at any given moment. This prevents race conditions and ensures that reads and writes occur in a well-defined order.

#### `MutexGuard`

When a Rust mutex is locked, you do not directly receive access to the protected value; instead, you're given a `MutexGuard`.
A `MutexGuard` is a smart pointer that represents ownership of the lock. As long as a thread owns the guard, the mutex remains locked and access to the protected value is permitted. Because only one guard may exist at a time, access is mutually exclusive.
Once the guard is dropped (goes out of scope), the lock is automatically released and can be given to other threads.

Because `MutexGuard` implements `Drop`, the lock is automatically released when the guard leaves scope, even during panic unwinding.
This eliminates an enormous category of bugs that plague languages requiring explicit unlock operations.

```rust
use std::sync::Mutex;

fn main() {
    let counter = Mutex::new(0);

    {
        let mut guard = counter.lock().unwrap();
        *guard += 1;

        println!("counter = {}", *guard);
    } // guard is dropped here, releasing the lock

    println!("counter = {}", *counter.lock().unwrap());
}
```

#### Mutex Poisoning

Rust introduces an additional safety mechanism called poisoning.

Suppose a thread acquires a mutex and begins modifying shared data; halfway through the update, the thread panics.
At this point, the data may be in an inconsistent state. Perhaps only part of a structure was updated before execution stopped.
Instead of silently allowing other threads to continue, the mutex becomes poisoned and future lock attempts return a `PoisonError`, signaling that a previous owner panicked while holding the lock. this error result however, does not make the mutex unusable. You may still recover the data if you intentionally choose to do so.

Conceptually, Rust is saying:

> "The previous owner of this lock panicked while modifying the data. I cannot guarantee the protected value is still consistent."

The important point is that Rust forces you to acknowledge the possibility of corruption rather than ignoring it.

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() { // More on Arc incoming
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data_clone = Arc::clone(&data);

    let handle = thread::spawn(move || {
        let mut numbers = data_clone.lock().unwrap();

        numbers.push(4);

        panic!("Something went wrong!"); // This poisons the lock
    });

    let _ = handle.join();

    match data.lock() {
        Ok(guard) => {
            println!("Data: {:?}", *guard);
        }
        Err(poisoned) => {
            println!("Mutex was poisoned!");

            // Explicitly recover the data
            let guard = poisoned.into_inner();
            println!("Recovered data: {:?}", *guard);
        }
    }
}
```

### `Arc<T>`

At this point we have a mechanism for synchronized mutable access; however, another problem remains. Who owns the mutex itself?

Suppose several threads need access to the same mutex; ownership cannot simply be moved into one thread because the other threads would lose access.

We need shared ownership. But we can't use `Rc<T>` here because it's not thread-safe.
An `Rc<T>` stores a reference count alongside the value.
Whenever a clone occurs the count is incremented, whenever an `Rc<T>` is dropped it's decremented, and
when the count finally reaches zero the value is dropped.The problem is that these operations are not synchronized.

Imagine two threads incrementing the counter simultaneously; both threads might read the same count value; both compute the same new value and both write it back; one update is effectively lost and the reference count becomes incorrect.

Incorrect reference counts eventually lead to memory-safety problems, which Rust absolutely cannot allow.

For this reason, `Rc<T>` does not implement `Send` or `Sync` (More on these later); the compiler simply refuses to let it cross thread boundaries.

Ok I yapped too much; all of this is to say we need to use `Arc<T>` instead. Arc stands for Atomic Reference Counted and it's the thread-safe alternative to `Rc<T>`.

`Arc<T>` achieves this thread-safety by reference counting atomically.
Atomic operations guarantee that updates remain correct even when multiple CPU cores modify the counter simultaneously. Since the counter is synchronized, the reference count cannot become corrupted.

It's a drop in replacement for `Rc<T>` for thread-safe use cases but comes with some overhead.

### `Arc<Mutex<T>>`

By combining Arc and Mutex, we can have thread-safe, multi-owner (reference-counted), mutable shared-state.
`Arc<T>` gives us thread-safe shared ownership and `Mutex<T>` gives us thread-safe mutable access.

- Sharing a Mutex Between Threads

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);

        let handle = thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
        });

        handles.push(handle);
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("Final count: {}", *counter.lock().unwrap());
}
```

## Message Passing vs Shared-State Concurrency

Message Passing and Shared-State Concurrency solve similar problems but use very different philosophies; In shared-state concurrency multiple threads can access the same data but access must be synchronized; however, message passing avoids sharing ownership whenever possible and transfers it instead. Neither approach is universally superior.

Message passing often leads to simpler designs because fewer objects are shared.

Shared-state concurrency can be more efficient when many threads genuinely need access to the same data.

In practice, real Rust applications often use both. A program might use channels to distribute work between threads while using mutexes to protect a shared cache or configuration object.
Rust provides tools for both approaches because each has situations where it is the better fit.

All that said, Rust's ownership system naturally favors channels since they're essentially ownership-transfer pipelines.
with channels ownership remains unambiguous, thus, entire categories of concurrency bugs become impossible and the compiler can verify that no thread continues using a value after it has been sent elsewhere.

## `Send` & `Sync`

You might wonder how Rust knows whether a type can be used safely across threads. The answer is **Auto Traits**. Auto traits are traits that the compiler automatically implements for a type when all of its fields satisfy the same trait.

The most important auto traits are `Send` and `Sync`:

- `Send`

  A type is `Send` if ownership of its value can be safely transferred from one thread to another. Most Rust types are `Send` including: Integers, Strings, Vectors, HashMaps any struct that is entirely composed of `Send` types, etc.

  `Rc<T>` is the classic example of a type that is not `Send`. Its reference count is not synchronized. If ownership could freely move between threads, the count could become corrupted. Rust therefore refuses to allow it.

- `Sync`

  A type is Sync if references to the type can be safely shared between threads. For example, immutable data is often `Sync` because multiple threads can safely read the same value simultaneously. A `Mutex<T>` is also Sync because the mutex guarantees that mutable access remains synchronized.

  `RefCell<T>` is not `Sync`.
  Because its borrow tracking is not thread-safe. Two threads attempting to manipulate the borrow state simultaneously would create problems.

A struct containing only `Send` and `Sync` fields will automatically implement `Send` and `Sync` itself.

Neither `Send` nor `Sync` provides any methods or behavior. Instead, they exist solely to describe thread-safety properties to the compiler. Traits that exist only to convey information about a type and do not define behavior are called **Marker Traits**.
Therefore, `Send` and `Sync` are both **Auto Traits** and **Marker Traits**.

## Deadlocks

One of Safe Rust's greatest strengths is its ability to prevent data races at compile time. However, not all concurrency bugs can be eliminated through the type system. Some concurrency failures arise from flawed program logic rather than unsafe memory access. One of the most common examples is a deadlock, where two or more threads become permanently blocked while waiting for resources held by one another; waiting for each other indefinitely, and none of them can make progress. The program is still running. Nothing has crashed. No panic occurred. Yet the affected threads are permanently stuck.

The most common deadlocks involve mutexes. Imagine two shared resources protected by two mutexes: Lock A and Lock B.
Thread 1 acquires Lock A and then tries to acquire Lock B; at the same time, Thread 2 acquires Lock B and then tries to acquire Lock A; now both threads are waiting and neither thread can continue because each needs a lock currently owned by the other.

The program has reached a **_Deadlock_**.

`MutexGuard` helps reduce some locking mistakes because lock lifetime is tied directly to scope. By keeping scopes small, locks are released sooner, which can reduce opportunities for deadlocks.

There are other practical ways of avoiding deadlocks but they're beyond the scope of this file.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
