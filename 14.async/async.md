# Asynchronous Programming In Rust

- [A Word](#a-word)
- [Overview](#overview)
- [Prerequisites](#prerequisites)
  - [CPU Bound Operations](#cpu-bound-operations)
  - [IO Bound Operations](#io-bound-operations)
  - [Blocking Calls](#blocking-calls)
  - [Non-Blocking Calls](#non-blocking-calls)
  - [OS Threads](#os-threads)
  - [User-Level Threads](#user-level-threads)
- [Modeling Concurrency Over OS Threads](#modeling-concurrency-over-os-threads)
  - [Why Abstract Over OS Threads](#why-abstract-over-os-threads)
- [Coroutines](#coroutines)
- [Stackful Coroutines](#stackful-coroutines)
  - [Fibers](#fibers)
  - [Green Threads](#green-threads)
  - [M:N Scheduling](#mn-scheduling)
- [Stackless Coroutines](#stackless-coroutines)
  - [State Machines](#state-machines)
    - [Compiler Generated State Machines](#compiler-generated-state-machines)
- [Event Queueing](#event-queueing)
- [OS Event Queues](#os-event-queues)
  - [Readiness-Based Event Queues](#readiness-based-event-queues)
    - [`kqueue` (macOS)](#kqueue-macos)
    - [`epoll` (Linux)](#epoll-linux)
  - [Completion-Based Event Queues](#completion-based-event-queues)
    - [IOCP (Windows)](#iocp-windows)
- [Checkpoint](#checkpoint)
- [Async In Rust](#async-in-rust)
  - [`async` Blocks](#async-blocks)
  - [`async move`](#async-move)
  - [What the Compiler Sees](#what-the-compiler-sees)
- [Futures](#futures)
  - [Leaf Futures & Non-Leaf Futures](#leaf-futures--non-leaf-futures)
  - [Self-Referential Futures](#self-referential-futures)
    - [Why Moving Them is Dangerous](#why-moving-them-is-dangerous)
    - [`Pin<T>`](#pint)
    - [`Unpin`](#unpin)
- [Tasks](#tasks)
  - [`.await` vs `spawn`](#await-vs-spawn)
  - [`JoinHandle`](#joinhandle)
  - [Detached Tasks](#detached-tasks)
  - [Task Cancellation](#task-cancellation)
  - [Task Scheduling](#task-scheduling)
- [Features the Standard Library Provides](#features-the-standard-library-provides)
- [Features the Standard Library Doesn't Provide](#features-the-standard-library-doesnt-provide)
- [Runtime](#runtime)
  - [Executor](#executor)
  - [Reactor](#reactor)
  - [Wakers](#wakers)

## A Word

This file focuses on the concepts behind asynchronous programming and how Rust implements them. It intentionally takes a high-level, runtime-agnostic approach, emphasizing the underlying ideas and design rather than the APIs of any particular async runtime.

As a result, you'll find relatively few concrete examples or runtime-specific code snippets here. I plan to write separate notes covering popular runtimes (such as Tokio), where I'll focus on their APIs, practical usage, and implementation details. I'll link those notes here once they're available.

## Overview

Asynchronous programming is a programming model that allows a task to suspend execution while waiting for an operation to complete, allowing other tasks to make progress in the meantime. It is a language or library abstraction for expressing concurrency without requiring one operating system thread per task.

Operating systems already provide another mechanism for concurrent execution: **threads**. Using multiple OS threads to execute tasks concurrently is known as multithreaded programming.

Both asynchronous programming and multithreading enable concurrent execution, but they do so in different ways. Threads rely on the operating system scheduler to switch between threads, whereas asynchronous programming relies on tasks that voluntarily yield control at suspension points (such as `await`). Async code is often easier to scale to large numbers of waiting tasks, while threads are generally a better fit for CPU-bound work.

## Prerequisites

Here, I'll give a brief explanation of a few important topics that are necessary for the rest of this file.

### CPU Bound Operations

A CPU-bound operation spends most of its time performing computations. Its execution speed is primarily limited by the processing power of the CPU rather than waiting for external resources. Because these tasks actively use the CPU, running multiple CPU-bound operations simultaneously often benefits from multiple CPU cores.

Examples include compressing a file, rendering an image, encrypting data, performing mathematical calculations, etc.

### IO Bound Operations

An I/O-bound operation spends most of its time waiting for input or output rather than performing computations. During this waiting period, the CPU has very little work to do. Since the CPU is mostly idle while waiting, I/O-bound workloads benefit greatly from concurrent execution.

examples include reading or writing files, sending or receiving network requests, querying a database, waiting for user input, etc.

### Blocking Calls

A blocking call prevents the current thread from doing anything else until the requested operation completes.

For example, if a thread calls a blocking function to read a file, the operating system suspends that thread until the data becomes available. While blocked, the thread cannot execute other work.

Blocking is simple to reason about, but can lead to poor resource utilization when many operations spend most of their time waiting.

### Non-Blocking Calls

A non-blocking call returns immediately instead of waiting for an operation to finish.
If the requested work cannot be completed immediately, the call returns without waiting. The caller can continue executing other work and obtain the result once the operation completes.

Asynchronous programming is largely built around non-blocking operations.

### OS Threads

An operating system thread (or kernel thread) is the unit of execution managed and scheduled by the operating system.
Each thread has its own call stack, registers, and execution state. The operating system's scheduler decides when to pause one thread and allow another to run, a process known as preemption.

Threads make it possible for multiple tasks to execute concurrently, and on multi-core processors, some threads may execute in parallel.

Because OS threads are relatively expensive to create and context-switch, many languages and runtimes provide their own lightweight execution units that are scheduled onto a smaller number of OS threads.

### User-Level Threads

A user-level thread is a lightweight unit of execution managed entirely by a language runtime or library instead of the operating system.

Unlike OS threads, the operating system is unaware of user-level threads. Instead, the runtime schedules many user-level threads onto a smaller number of OS threads. This scheduling strategy is commonly called the M:N threading model, because M user-level threads are multiplexed onto N operating system threads.

Because user-level threads are much cheaper to create and switch between than OS threads, applications can efficiently manage thousands or even millions of concurrent tasks.
Examples include Go goroutines and Java virtual threads.

## Modeling Concurrency Over OS Threads

Operating system threads are a powerful abstraction for concurrent execution, but they are not without cost. Each thread requires its own call stack, is created and managed by the operating system, and must be scheduled by the kernel. While this overhead is perfectly acceptable for dozens or even hundreds of threads, it becomes increasingly expensive when an application needs to manage tens or hundreds of thousands of concurrent operations.

This is particularly common in I/O-bound applications such as web servers, where most tasks spend the majority of their lifetime waiting for network or disk operations rather than actively using the CPU. Dedicating an entire operating system thread to every waiting task would waste both memory and scheduling resources.

To address this, many languages and runtimes introduce their own lightweight units of execution that are managed in user space instead of by the operating system.

### Why Abstract Over OS Threads

Operating system threads remain the foundation of concurrent execution. Regardless of the programming language or runtime being used, code ultimately executes on one or more OS threads.

The goal of higher-level concurrency abstractions is therefore not to replace OS threads, but to use them more efficiently. Instead of creating one OS thread for every concurrent task, a runtime can schedule many lightweight tasks onto a much smaller number of OS threads. This greatly reduces memory usage, lowers scheduling overhead, and allows applications to efficiently manage enormous numbers of concurrent operations.

Different languages achieve this in different ways, but nearly all modern concurrency abstractions fall into one of two categories: stackful coroutines or stackless coroutines.

## Coroutines

In its simplest form, a coroutine is just a task that can stop and resume by yielding control to either its caller, another coroutine, or a scheduler.

## Stackful Coroutines

A stackful coroutine is a lightweight unit of execution that owns its own call stack, much like an operating system thread. Because each coroutine maintains an independent stack, it can suspend execution at almost any point in the call chain and later resume exactly where it left off.

This makes stackful coroutines feel very similar to ordinary threads while remaining significantly cheaper to create and manage. Since they are scheduled by a language runtime rather than the operating system, thousands or even millions of them can exist simultaneously.

### Fibers

Fibers are one of the simplest forms of stackful coroutines. They are cooperatively scheduled, meaning a running fiber voluntarily yields execution rather than being preempted by the operating system. Because every fiber owns its own stack, switching between fibers preserves the complete execution state, including nested function calls and local variables.

### Green Threads

Green threads extend the same idea by providing a thread-like abstraction that is managed entirely by a language runtime instead of the operating system. Rather than manually switching between execution contexts, the runtime automatically schedules green threads, allowing programmers to write code that closely resembles traditional multithreaded programs while avoiding the cost of creating large numbers of OS threads.

### M:N Scheduling

Stackful coroutines are commonly scheduled using an M:N scheduling model, where M user-level threads are multiplexed onto N operating system threads.

The runtime determines which coroutine should execute next, while the operating system schedules only the smaller pool of OS threads. This allows applications to support vast numbers of concurrent tasks without requiring an equally large number of operating system threads.

## Stackless Coroutines

Unlike stackful coroutines, stackless coroutines do not own an independent call stack. Instead, they suspend execution only at explicitly defined suspension points, such as an await or yield expression.

Because they do not require a separate stack for every coroutine, they are substantially smaller and cheaper to create than stackful coroutines. As a result, applications can efficiently manage millions of stackless coroutines while using only a small number of operating system threads.

Modern asynchronous programming languages commonly adopt this approach. Although the terminology varies between ecosystems, the underlying concept is largely the same. For example, Rust exposes stackless coroutines as Futures, JavaScript uses Promises together with async/await, and C# represents asynchronous operations with Tasks and so on.

### State Machines

A state machine in its simplest form is a data structure that has a predetermined set of states it can be in. In the case of stackless coroutines, each state represents a possible pause/resume point. We don’t store the state needed to pause/resume the task in a separate stack (like stackful coroutines do); We save it in a data structure instead.

This has some advantages, the most prominent ones being they’re very efficient and flexible. The downside is that you’d never want to write these state machines by hand, so you need some kind of support from the compiler or some other mechanism.

#### Compiler Generated State Machines

Rather than preserving an entire call stack, a stackless coroutine is transformed by the compiler into a state machine. This state machine stores only the information necessary to resume execution later, such as local variables and the point at which execution previously suspended.

Each time the coroutine resumes, the state machine continues execution from the appropriate state until it either suspends again or completes. This transformation allows stackless coroutines to achieve their small memory footprint while still preserving the illusion that execution simply paused and later continued from the same location.

## Event Queueing

Since non-blocking operations complete at unpredictable times, the operating system needs a way to inform applications when they are ready to continue.

Rather than forcing applications to repeatedly check whether every operation has completed (a technique known as busy polling) modern operating systems record completed or ready events in a queue. Applications simply wait for new events to appear, process them, and then continue executing.

This event-driven model allows a single thread to efficiently manage thousands of simultaneous I/O operations without continuously wasting CPU time checking whether each one has finished.

This naturally raises another question: how does the operating system notify applications that an asynchronous operation has become ready? The answer is event queues.

## OS Event Queues

Every major operating system provides a mechanism for monitoring large numbers of I/O operations simultaneously. Although their implementations differ, they all serve the same purpose; efficiently notifying applications when an operation can make progress.

These mechanisms fall into two categories; readiness-based event queues and completion-based event queues.

### Readiness-Based Event Queues

A readiness-based event queue notifies an application that an operation can now proceed without blocking. The operating system does not perform the operation on the application's behalf; instead, it simply reports that the underlying resource has become ready.

For example, a network socket may become ready for reading because new data has arrived, or ready for writing because sufficient buffer space is available. Once notified, the application performs the actual read or write operation itself.

This model is widely used on Unix-like operating systems.

#### `kqueue` (macOS)

`kqueue` is the readiness notification interface provided by macOS and other BSD-based operating systems. Applications register the resources they wish to monitor, and the kernel reports whenever one of those resources becomes ready for an operation.

#### `epoll` (Linux)

`epoll` is Linux's readiness notification mechanism for efficiently monitoring large numbers of file descriptors. Rather than repeatedly checking every descriptor, applications wait for `epoll` to report which resources are ready, greatly improving scalability for network servers and other I/O-intensive applications.

### Completion-Based Event Queues

A completion-based event queue takes a different approach. Instead of notifying an application that an operation is ready to begin, it reports that the requested operation has already finished.

In this model, the application submits an I/O request to the operating system, which performs the operation asynchronously. Once the work is complete, the operating system places a completion event into the queue, allowing the application to retrieve the result without performing the operation itself.

This model eliminates the need for the application to retry or resume the I/O operation after receiving a notification, since the work has already been completed.

#### IOCP (Windows)

I/O Completion Ports (IOCP) are Windows' high-performance completion-based I/O mechanism. Applications submit asynchronous I/O requests to the operating system, which performs the operations in the background. As each operation completes, the kernel places a completion event into the completion port, allowing worker threads to efficiently process completed requests. This design enables Windows applications to scale to very large numbers of concurrent I/O operations while using only a relatively small pool of threads.

## Checkpoint

Hi there! From this point onward, the discussion becomes much more Rust-specific, so this is probably a good place to take a break.

I've tried to introduce each concept in a natural order, but a few ideas won't fully click until you've seen the whole picture—especially the role of a **runtime** and its **executor**.

For now, just remember one important fact:
Rust provides the language primitives for asynchronous programming, but it does **not** provide anything that actually runs asynchronous code.

To execute async programs, you need an asynchronous **runtime**. One of the runtime's primary components is the **executor**, whose job is to drive futures to completion by polling them and scheduling tasks.

We'll cover runtimes, executors, reactors, and wakers in detail near the end of this file. For now, it's enough to think of the executor as the component responsible for making futures actually progress.

## Async In Rust

Rust implements asynchronous programming using stackless coroutines. Every `async` code block and `async fn` is transformed by the compiler into a state machine that implements the `Future` trait (more on this later).

Unlike operating system threads, these state machines do not execute on their own. Instead, they represent asynchronous computations that can be suspended and later resumed by an **_executor_**. Each time a future is resumed, it makes progress until it either reaches another suspension point or completes.

### `async` Blocks

An `async` block creates a future from an arbitrary block of code.

```rust
let future = async {
    println!("Hello");
    42
};
```

Like every future in Rust, `async` blocks are **lazy**. Creating it does not execute any of its code.
Nothing inside the block runs until the future is polled, typically by awaiting it or by submitting it to an executor as part of a task.

```rust
let future = async {
    println!("Running...");
    42
};

println!("Created the future.");
let value = future.await;
println!("{value}");
```

Produces:

```
Created the future.
Running...
42
```

The body of an async block can contain `.await` expressions, local variables, loops, conditionals, and nearly any other Rust construct.

Similar to futures returned by `async fn`, futures produced by `async` blocks can also be stored, passed to functions, composed with other futures, or spawned onto an executor.

### `async move`

By default, an async block captures variables from its surrounding scope in much the same way that closures do. The compiler decides whether each variable should be borrowed or moved based on how it is used.

Sometimes, however, the future must own everything it uses. This is achieved with an `async move` block.

```rust
let s = String::from("Hello");

let future = async move {
    println!("{s}");
};
```

Much like how it behaves with closures, the `move` keyword transfers ownership of captured values into the future itself. After `move`, the original scope can no longer use those values.

Because the future now owns its captured data, it also becomes responsible for cleaning it up. If the future completes normally, or is cancelled by being dropped, all values moved into the future are dropped as part of the future's own destruction.

This ownership transfer is especially important when spawning tasks. Since a spawned task may continue executing long after the current scope has ended, it generally cannot borrow local variables. Instead, it must own any data it needs, making `async move` the common choice when spawning new tasks.

### What the Compiler Sees

Although `async` and `.await` look like language constructs, the compiler ultimately lowers them into ordinary Rust code.

Consider the following function:

```rust
async fn hello() -> String {
    String::from("Hello")
}
```

Conceptually, the compiler transforms it into something similar to:

```rust
fn hello() -> impl Future<Output = String> {
    async {
        String::from("Hello")
    }
}
```

The exact generated type is an anonymous compiler-generated state machine, but from the programmer's perspective it simply implements `Future<Output = String>`.

This explains an important property of `async` functions:
Calling an `async fn` does **not** execute its body.
Instead, calling the function constructs and returns a future.

```rust
let future = hello(); // Nothing has run yet.
```

Execution begins only when that future is first polled, usually by awaiting it.
The same principle applies to ordinary async blocks.

```rust
let future = async {
    println!("Hello");
};
```

Creating the block merely constructs another future. The code inside the block remains dormant until that future is polled.

Since futures are lazy, they must eventually be driven by something else. In practice, this usually happens in one of two ways:

- Awaiting the future, causing it to become part of the current task.
- Spawning the future onto an executor, creating a new task that the runtime polls independently.

Regardless of how the future is driven, the underlying mechanism is always the same: the executor repeatedly calls `poll()` until the future eventually returns `Poll::Ready`.

## Futures

A Future, as the name implies, represents an operation whose result will become available in the future.
Unlike a function call, creating a future does not immediately begin executing it. A future is lazy, meaning it performs no work until it is repeatedly polled by an **_executor_**.

Futures do not drive themselves. An executor polls them, and if a future returns Pending, it arranges to be woken when it can make further progress.

Every future eventually reaches one of two states:

- `Poll::Pending`: the future cannot make further progress right now.
- `Poll::Ready(output)`: the future has completed and produced its output.

This behavior is captured by the `Future` trait:

```rust
pub trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>,
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

The `poll()` method takes two arguments:

- `self: Pin<&mut Self>` gives the future mutable access to its own state. The reason futures are wrapped in `Pin` is an important topic that we'll cover later.
- `cx: &mut Context<'_>` provides the future with information about the current task. Most importantly, it gives the future access to a `Waker`, which it can use to notify the executor when it is ready to make more progress after returning `Poll::Pending`.

Each call to `poll()` asks the future to make as much progress as possible without blocking the current thread. If it cannot continue, it returns `Poll::Pending`. Once the asynchronous operation has completed, it returns `Poll::Ready(output)`.

### Leaf Futures & Non-Leaf Futures

Not all futures perform asynchronous work themselves. A leaf future is one that directly represents an asynchronous operation, such as waiting for a timer to expire or reading data from a socket. It interacts with the operating system or another asynchronous source and is ultimately responsible for determining when progress can be made.

A non-leaf future is built by composing one or more other futures. Rather than performing asynchronous work directly, it polls its child futures and combines their results. Most futures produced by async functions are non-leaf futures, as they simply coordinate and await other asynchronous operations. When polled, a non-leaf future propagates the poll down the chain until it eventually reaches one or more leaf futures.

A leaf future performs actual asynchronous work. It communicates directly with the operating system or an asynchronous runtime to initiate I/O operations.

So essentially, Executors poll the root future, which in turn polls its child futures, and so on, until the chain reaches one or more leaf futures. The leaf futures are responsible for interacting with the underlying asynchronous resources, while each non-leaf future simply coordinates and delegates progress to the futures it contains.

Because `async` functions return futures, awaiting a future requires the caller to become asynchronous as well. As a result, asynchrony propagates up the call chain. A leaf future performs the actual asynchronous work, while every caller that awaits it becomes a non-leaf future that composes and coordinates the operation.

### Self-Referential Futures

Rust compiles every `async fn` and `async` block into a state machine. Whenever execution reaches an `.await`, the state machine must preserve enough information to resume execution later. This includes local variables that remain in scope after the suspension point.

Consider the following example:

```rust
async fn example() {
    let s = String::from("Hello");
    let r = &s;

    some_async_operation().await;

    println!("{r}");
}
```

When the compiler transforms this function into a state machine, both `s` and `r` become fields of the generated future. Since `r` references `s`, _the future now contains a pointer to one of its own fields_.
A future that contains references to its own data is called self-referential.

Because async functions can become self-referential, the compiler and the `Future` API must support futures that cannot safely be moved after polling begins. As a result, _futures cannot generally be treated as values that may be freely moved in memory after execution begins_.

#### Why Moving Them is Dangerous

Moving an ordinary Rust value is perfectly safe because ownership is transferred along with its contents.
However, moving a self-referential future is different. Although the future's fields move to a new memory location, any references stored inside the future still point to the old location. Those references immediately become invalid.

To prevent this, once a future begins being polled, it must remain at a fixed memory address for the rest of its lifetime.
This requirement explains why the Future trait defines `poll()` like this:

```rust
pub trait Future {
    type Output;

    fn poll(
        self: Pin<&mut Self>, // Pin<T>:
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

Instead of receiving a mutable reference (`&mut Self`), `poll()` receives a pinned mutable reference (`Pin<&mut Self>`), guaranteeing that the future will not be moved while it is being polled.

#### `Pin<T>`

`Pin<T>` is a wrapper type that guarantees the value it contains will not be moved after it has been pinned. This allows self-referential futures to safely store references to their own fields across suspension points.

Pinning is a language-level guarantee rather than a runtime feature. The executor does not keep track of object addresses or prevent futures from moving; instead, Rust's type system ensures that once a future is pinned, safe Rust code cannot move it.

For asynchronous programming, the most important consequence is that executors always poll futures through `Pin<&mut Self>`, ensuring that any internal self-references remain valid throughout the future's execution.

#### `Unpin`

Fortunately, not every type is self-referential. Most Rust types can be safely moved even after they have been pinned. Such types implement the `Unpin` auto trait; similar to `Send` and `Sync`.

For `Unpin` types, `Pin<T>` imposes no additional restrictions—they can be treated much like ordinary mutable references. Pinning only has a meaningful effect for types that do **not** implement `Unpin`, such as many compiler-generated futures.

In other words, `Pin<T>` is designed to support the small number of types that require a stable memory address, while `Unpin` allows the vast majority of Rust types to remain freely movable.

Not every compiler-generated future is actually self-referential, but the `Future` trait has to accommodate those that are. That's why `poll` always takes `Pin<&mut Self>`, even though many futures end up being `Unpin`.

## Tasks

One of the most common sources of confusion in async Rust is the distinction between a future and a task.

A future is simply a value that represents an asynchronous computation. By itself, it performs no work and remains idle until something polls it.

_A task is a future that has been submitted to an executor for execution_. The executor owns the task, polls its future, stores its state while it is suspended, and resumes it whenever it is able to make further progress.

In other words, a future describes what work should be performed, while a task represents that work being actively managed by an asynchronous runtime.
This distinction is important because executors schedule tasks, not bare futures.

Every spawned task owns exactly one root future, although that future may itself contain many child futures created through await.

### `await` vs `spawn`

`await` is Rust's mechanism whereas `spawn` is a helper function that's usually exposed by async runtime crates like `tokio`.

Both `await` and `spawn` are mechanisms for running asynchronous computations, but they do so in fundamentally different ways.

Awaiting a future does not create a new task. Instead, the awaited future becomes part of the current task's state machine. When execution reaches an `.await` that cannot make immediate progress, the current task is suspended until the awaited future can continue.

Spawning is different. Rather than executing the future within the current task, the future is submitted to the executor as an entirely new task. The executor schedules it independently, allowing both the original task and the spawned task to make progress concurrently.

In short, `await` composes asynchronous operations into a single task, whereas spawning creates an additional task that is scheduled independently.

### `JoinHandle`

When a future is spawned as a new task, the runtime typically returns a `JoinHandle`. Similar to spawning a new thread.

A `JoinHandle` represents the ability to observe the result of a spawned task. Awaiting the handle waits for the task to complete and retrieves its output.

The handle does not execute the task or cause it to begin running. The task starts executing as soon as it is submitted to the executor; the handle only provides a way to synchronize with it later.

### Detached Tasks

A spawned task does not necessarily need to be awaited.

If the `JoinHandle` is dropped without being awaited, the spawned task may continue executing independently. Such a task is commonly called a detached task.

Detaching a task means giving up the ability to retrieve its result while allowing the runtime to continue scheduling it. The exact behavior depends on the runtime, but the concept remains the same: the task's lifetime is no longer tied to the code that spawned it.

```rust
use std::time::Duration;

fn main() {
    // trpl is a helper crate that re-exports
    // functionality from the `futures` and `tokio` crates
    trpl::block_on(async {
        // This is a detached task because we ignore the `JoinHandle`
        // returned by `spawn_task()`, giving up our ability to await
        // or retrieve its result.
        //
        // In this example, the task does not finish because the runtime
        // shuts down once the root task completes, dropping any remaining
        // detached tasks.
        trpl::spawn_task(async {
            for i in 1..10 {
                println!("hi number {i} from the first task!");
                trpl::sleep(Duration::from_millis(500)).await
            }
        });

        for i in 1..5 {
            println!("hi number {i} from the second task!");
            trpl::sleep(Duration::from_millis(500)).await;
        }
    });
}
```

### Task Cancellation

A task is cancelled by dropping the future it owns.

Recall that a future is just a value representing an asynchronous computation. As long as the future exists, the executor may continue polling it. Once the future is dropped, however, it can never be polled again, so the asynchronous operation permanently stops making progress.

Unlike many asynchronous systems, Rust does not require a dedicated cancellation mechanism. Cancellation is simply a consequence of ownership and RAII. Dropping a future runs the destructors of all values it owns, immediately releasing resources such as file handles, network sockets, mutex guards, and heap allocations, just as they would be in synchronous code.

Cancellation is also **cooperative**, not **preemptive**. An executor cannot interrupt a future while it is actively executing. Instead, a future can only be cancelled after it has yielded control back to the executor by returning `Poll::Pending`. Once polling begins, the future continues executing until it either reaches another suspension point or completes.

```rust
use tokio::time::{sleep, Duration};

async fn work() {
    println!("Started");

    sleep(Duration::from_secs(5)).await;

    println!("Finished");
}

#[tokio::main]
async fn main() {
    let task = tokio::spawn(work());

    sleep(Duration::from_secs(1)).await;

    task.abort();
}
```

In this example, the task is aborted while the future is suspended at `.await`. Tokio cancels the task by dropping the future. As a result, `"Finished"` is never printed, the future is never polled again, and all resources owned by the future are cleaned up automatically.

### Task Scheduling

Unlike operating system threads, which are preemptively scheduled by the operating system, asynchronous tasks are cooperatively scheduled. A task continues executing until it voluntarily yields control, usually by reaching an `.await` whose underlying future cannot make immediate progress. At that point, the future returns `Poll::Pending`, allowing the executor to schedule other runnable tasks.

This differs fundamentally from preemptive scheduling. An operating system can interrupt a running thread at almost any point and switch execution to another thread, even if the thread is performing a long-running computation. An executor, however, cannot interrupt a running task. Once a task begins executing, it continues running until it either completes or yields by returning from `poll()`. Ordinary synchronous code inside an async function therefore runs uninterrupted.

Consider the following task:

```rust
async fn work() {
    heavy_computation();      // Runs uninterrupted.
    read_socket().await;      // May suspend here.
    process_data();           // Runs uninterrupted.
}
```

The executor cannot schedule another task while `heavy_computation()` or `process_data()` are executing. Only when `read_socket().await` suspends does the executor regain control and become free to poll other tasks.

Because of this, asynchronous tasks should avoid performing long-running CPU-bound work without yielding. Such computations can _monopolize_ the executor, delaying other tasks and reducing the responsiveness of the application. Async programming is therefore best suited to workloads that spend much of their time waiting for external events, while CPU-intensive work is typically delegated to dedicated threads or thread pools.

_Many asynchronous runtimes also take advantage of parallelism by executing tasks on multiple operating system threads. However, asynchronous programming is primarily about improving concurrency rather than accelerating CPU-bound work. Its greatest benefits are seen in I/O-bound applications that spend much of their time waiting for external events._

## Features the Standard Library Provides

Rust's standard library, together with compiler support, defines the core abstractions required for asynchronous programming.

These include:

- The `Future` trait
- The `Poll` enum
- The `Context` type
- The `Waker` type
- The `Pin` API, which allows futures to be safely polled without being moved
- Compiler support for `async`/`await`

## Features the Standard Library Doesn't Provide

While the standard library defines _what_ a future is, it does not provide anything that actually executes asynchronous code.

In particular, the standard library does not include:

- an Executor
- an Event Loop
- Timers
- Asynchronous File I/O
- Asynchronous Networking
- Task Scheduling
- Spawning APIs

For example, this program compiles:

```rust
async fn hello() {
  println!("Hello!");
}

fn main() {
  let future = hello();
  println!("Future created!");
}
```

Its output is:
_`Future created!`_.

_`Hello!`_ is never printed. The future is merely constructed; it is never polled.
To actually execute it, an executor, like the one provided by the `tokio` crate, is required.

```rust
#[tokio::main]
async fn main() {
  // The executor polls the future until it completes.
  hello().await;
}
```

**_This separation is intentional_**. The Rust standard library provides the language primitives for asynchronous programming, while asynchronous runtimes such as **Tokio**, **`async-std`**, and **`smol`** provide the machinery that drives futures, schedules tasks, and interfaces with the operating system. This design gives the programmer freedom and allows multiple runtimes to coexist without requiring the language itself to choose a single asynchronous implementation.

## Runtime

A runtime is the infrastructure responsible for driving asynchronous programs. It provides the machinery required to execute futures, schedule and execute tasks, interact with the operating system's event queue, and wake suspended tasks when they are able to make progress.

The Rust standard library intentionally does not include a runtime. Instead, runtimes are provided by external libraries such as _Tokio_, _`async-std`_, and _`smol`_. This allows different runtimes to optimize for different workloads while sharing the same language-level async primitives.

Although implementations differ, most asynchronous runtimes contain two major components; an Executor, which schedules and polls futures, and a Reactor, which interfaces with the operating system's event notification mechanisms.
Together, these components allow asynchronous tasks to execute efficiently while using only a small number of operating system threads

### Executor

The executor is the part of the runtime responsible for executing asynchronous tasks.

Recall that creating a future does not execute it. A future only makes progress when its `poll()` method is called. The executor repeatedly polls runnable tasks until their futures eventually return `Poll::Ready`.

If polling a future returns `Poll::Pending`, the executor stops polling it and moves on to other runnable tasks. The suspended task will remain idle until something notifies the executor that it may be able to make progress again.

Notice what the executor **doesn't** do. It does not wait for network packets, timers, or file I/O to complete. Its only job is to execute Rust code by polling futures.

### Reactor

While the executor runs Rust code, the reactor waits for external events.

Suppose a future tries to read from a socket:

```rust
async fn download() -> String {
    socket.read().await
}
```

If no data is currently available, the future cannot continue executing. Instead of blocking the thread, it asks the reactor to monitor the socket.

The reactor registers interest in the event with the operating system using facilities such as `epoll`, `kqueue`, or IOCP. It then waits for the operating system to report that the socket has become readable.

The reactor never executes Rust code itself. Its job is simply to detect when external events occur.

### Wakers

Once the reactor learns that an event has occurred, it must somehow notify the executor that the corresponding task should be polled again.

This is the purpose of a `Waker`.

Whenever a future returns `Poll::Pending`, it receives a `Waker` through its `Context`.

```rust
fn poll(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>,
) -> Poll<Self::Output>
```

Before returning `Poll::Pending`, the future stores or registers this Waker with whatever asynchronous resource it is waiting on.

Later, when that resource becomes ready; for example, when the reactor learns that a socket has become readable, it calls `waker.wake()`.

Calling `wake()` does not execute the future immediately. It simply tells the executor that the associated task should be placed back into its runnable queue. The executor will eventually poll it again, allowing execution to resume where it previously suspended.

## Author

[Ali Ghelichkhani](https://github.com/I-Do-CS)
