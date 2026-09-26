# RFC 0017: Tasks and Channels

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Paco has no `async`/`await` and no function coloring. `spawn` starts a lightweight task, scheduled M:N over a pool of OS threads, with a stack that grows on demand. A blocking operation suspends the task transparently. Tasks communicate over channels, which move ownership of the values sent, so tasks cannot race on shared mutable state. A panic inside a task ends only that task: `join()` on its handle returns `Result<T, TaskPanic>`. For purely local sequences, `iter fn` with `yield` provides a synchronous generator that never touches the scheduler.

## Motivation

A program that handles many concurrent connections or requests needs many concurrent units of work. OS threads do not scale to that number: each commits a large fixed stack, and switching between them goes through the kernel.

Explicit `async`/`await` scales, but it splits functions into two kinds. An async function can only be awaited from another async function, so the marker spreads up every call stack that reaches an I/O operation, and libraries end up offering a sync and an async version of the same API.

Paco needs concurrency that scales like the async model while keeping one kind of function. It also needs concurrent tasks that cannot race on memory, and a failure in one task that does not take down the process.

## Guide-level explanation

Any function can run as a task. `spawn` is a prefix keyword that takes a call or a block and returns a handle:

```paco
spawn compute(data)            // fire and forget

let h = spawn risky();         // keep the handle
match h.join() {               // wait for the task and get its result
    Ok(value)  => use_it(value),
    Err(panic) => log("task died: " + panic.message()),
}
```

Nothing marks `compute` or `risky` as asynchronous. When a task makes a blocking call (I/O, a channel operation, a sleep), the runtime suspends it and runs another task on the same OS thread. The function's signature and its call sites look the same whether or not it ever blocks.

Tasks communicate over channels:

```paco
let (tx, rx) = channel<i64>(capacity: 8);

fn produce(tx: Sender<i64>) -> Result<i64, Error> {
    for i in 0..10 { tx.send(i)? }
    tx.close();
    Ok(10)
}

let producer = spawn produce(tx);

for value in rx {              // iterates until the channel closes
    print(value)
}
```

Sending a value moves it. After `tx.send(v)` the sender cannot use `v`, so two tasks never hold mutable access to the same data. Data that must be shared across tasks is wrapped in `Arc` with explicit synchronization.

`select` waits on several channel operations at once and runs the arm of the first one ready. A `timeout` arm bounds the wait, and a `default` arm makes the `select` non-blocking:

```paco
// `Duration` lives in stdlib::time; the file imports it with `use stdlib::time`.
select {
    v = rx1.recv() => handle(v),
    v = rx2.recv() => handle(v),
    timeout(time::Duration::seconds(1)) => print("took too long"),
}

select {
    v = rx1.recv() => handle(v),
    default        => print("no data available"),
}
```

A panic inside a task is caught at the task boundary. The spawner sees it as the `Err` arm of `join()`, a `TaskPanic`, and handles it like any other failed operation ([RFC 0005](0005-error-handling.md)).

For a sequence produced and consumed locally, a task and a channel are unnecessary overhead. An `iter fn` is a synchronous generator that the caller pulls:

```paco
iter fn fibonacci() -> i64 {
    let mut a = 0;
    let mut b = 1;
    loop {
        yield a;
        let next = a + b;
        a = b;
        b = next
    }
}

for n in fibonacci().take(10) {
    print(n)
}
```

An `iter fn` runs no concurrency, allocates nothing, and does not involve the scheduler. `iter` is a synchronous sequence you pull; `spawn` and a channel are concurrent work the runtime schedules.

## Reference-level explanation

### Tasks

`spawn` is an expression: `spawn <call>` or `spawn { ... }`. It schedules the task and evaluates to a handle. `handle.join()` suspends the caller until the task finishes and returns `Result<T, TaskPanic>`, where `T` is the task's result type.

Tasks are scheduled M:N over a pool of worker OS threads. A task's stack starts at a few kilobytes and grows on demand, so thousands of tasks are normal.

Blocking operations (I/O, channel send and receive, sleep) are suspension points. The runtime parks the task and wakes it through the platform's event poller (epoll, kqueue or IOCP) when the operation can proceed. No syntax marks a suspension point, and the type of a function does not depend on whether it suspends.

A foreign call is opaque native code and cannot be suspended this way. [RFC 0019](0019-blocking-calls.md) covers how such calls run without stalling the worker pool.

### Panic isolation

A panic that reaches a task's entry point stops there. The runtime stores it and returns it as the `Err` of that task's `join()`. `TaskPanic::message()` gives the panic message.

If the handle is dropped without being joined, the panic is logged and the task ends; the process keeps running. A panic in `main` has no spawner to receive it and ends the process with exit status 101.

### Channels

`channel<T>(capacity: n)` returns a `(Sender<T>, Receiver<T>)` pair. `send` moves its argument into the channel. A `Receiver<T>` is iterable and the loop ends when the channel is closed.

A non-`Arc` value sent over a channel is moved, so the borrow checker's ordinary rules ([RFC 0001](0001-ownership-and-borrowing.md)) rule out data races between tasks at compile time.

### `select`

`select { arms }` is an expression. An arm is one of:

- `name = expr => handler`: `expr` is a channel operation such as `rx.recv()`; its result is bound to `name` in `handler`.
- `expr => handler`: a plain arm, used for `timeout(duration)`.
- `default => handler`: runs immediately when no other arm is ready.

The channel operations in the arms are not executed one after another. `select` registers all of them with the scheduler, suspends the task until one is ready, and runs that arm's handler. With a `default` arm it does not suspend.

### Generators

`iter fn` declares a generator. `yield expr` is valid only inside an `iter fn` body. Calling an `iter fn` returns an iterator; each pull resumes the body until the next `yield`, whose value it returns, and the iterator ends when the body returns.

An `iter fn` compiles to a synchronous state machine driven by the caller, with no scheduler, channel or allocation. `iter` does not combine with `unsafe`, `extern` or `comptime`. Restricting `yield` to `iter fn` keeps a locally pulled sequence and concurrently scheduled work syntactically distinct.

## Drawbacks

The language owns a large runtime: an M:N scheduler, growable stacks and a platform-specific event poller. It is the most complex component of the runtime, and every concurrent program depends on its correctness.

Transparent suspension also hides where a task may yield. A reader cannot tell from a call site whether the call suspends.

## Rationale and alternatives

**OS threads only.** Concurrency maps directly onto what the operating system provides and needs almost no runtime. It does not scale to large numbers of concurrent tasks because of fixed stack sizes and kernel context switches.

**Explicit `async`/`await`.** The compiler turns async functions into state machines, which needs a much smaller runtime and no growable stacks. It splits functions into two kinds, forces the marker up every caller, and duplicates APIs across the boundary.

**Implicit M:N tasks (chosen).** Tasks scale like the async model with a single kind of function. Ownership provides race safety, the task boundary contains panics, and `iter fn` covers the local case without the scheduler. The cost sits in the runtime rather than in the language surface.

**`spawn` as a function taking a closure** (`spawn(|| ...)`) was considered. A prefix keyword needs no closure for the common case of spawning a call (`spawn f(x)`), and `spawn { ... }` covers the rest.

## Prior art

Go's goroutines and channels use M:N scheduling with transparent suspension and a `select` statement over channels. Erlang isolates failures at the process boundary. Rust, JavaScript and C# use explicit `async`/`await`, and the function-coloring problem is well documented in those communities. Python and C# generators are the model for `yield` in a synchronous iterator.

## Unresolved questions

None.

## Future possibilities

Structured concurrency: scoped task groups on top of `spawn` and `join` that guarantee every child task finishes or is cancelled before the scope exits. It fits this model without new keywords.

Scheduler tuning such as work stealing and worker-pool sizing is not observable from Paco source and can change without affecting the language.
