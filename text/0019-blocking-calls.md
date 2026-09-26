# RFC 0019: Blocking Calls

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Calls that block an OS thread, such as foreign calls, run on a separate, growable blocking pool rather than on the scheduler's worker threads. `spawn_blocking` runs a closure on that pool and returns a handle with the same shape and panic isolation as `spawn`. Calling a foreign function directly outside `spawn_blocking` is not an error; it fires the `blocking-call-on-worker` lint.

## Motivation

Tasks ([RFC 0017](0017-tasks-and-channels.md)) suspend transparently when they block on I/O, a channel or a sleep, because the runtime intercepts those operations. A foreign call ([RFC 0018](0018-ffi-and-unsafe.md)) is opaque native code: it has no suspension point the runtime can observe.

When a task calls `cblas_sgemm` and the call runs for 200 milliseconds, the worker thread that made it is unavailable for that time and no other task can use it. With as many concurrent long foreign calls as there are worker threads, the whole scheduler stalls.

Programs that call numerical libraries or device drivers while serving concurrent work make such calls routinely. They need a way to run them without taking worker threads away from other tasks, and without splitting functions into blocking and non-blocking kinds.

## Guide-level explanation

The runtime has two thread pools. The worker pool runs tasks; its size follows the number of cores and does not change, and code on it is expected not to block. The blocking pool runs foreign calls and other thread-blocking work; it grows on demand up to a configurable cap, and idle threads exit.

`spawn_blocking` moves a closure onto the blocking pool:

```paco
// The task suspends; its worker thread runs other tasks meanwhile.
let result = spawn_blocking(|| {
    unsafe { cblas_sgemm(/* ... */) }
}).join()?;
```

The handle works like the one `spawn` returns. `join()` suspends the calling task until the closure finishes and returns `Result<T, TaskPanic>`: a panic inside the closure is captured at the pool boundary and returned as `Err`, and does not take down the caller ([RFC 0005](0005-error-handling.md)).

A direct foreign call outside `spawn_blocking` compiles, with a warning:

```paco
let n = unsafe { strlen(s) };   // warning: blocking-call-on-worker
```

For a call known to return quickly, such as `strlen`, the warning can be ignored. For a long call, the fix is to wrap it in `spawn_blocking`.

## Reference-level explanation

`spawn_blocking` is an ordinary prelude function taking a closure ([RFC 0013](0013-prelude.md)). It schedules the closure on the blocking pool, suspends nothing by itself, and returns a handle. `handle.join()` suspends the caller as a normal blocking operation, so the worker thread is free while the closure runs.

There is no blocking function type, no keyword marking a declaration as blocking, and no separate standard library for blocking code. A function that contains a blocking call has the same type as one that does not. This keeps the property [RFC 0017](0017-tasks-and-channels.md) relies on: one kind of function.

The blocking pool is separate from the worker pool. It grows when all its threads are busy and a new closure arrives, up to its cap; beyond the cap, closures queue. Threads that stay idle are released.

A call to an `extern` function outside a `spawn_blocking` closure fires the `blocking-call-on-worker` lint, which has a stable code and suggests wrapping the call in `spawn_blocking`. It is a lint and not a compile error because the compiler cannot know whether a foreign function blocks: `strlen` returns immediately, `cblas_sgemm` may run for a long time, and their declarations give the type checker no way to tell them apart. An error would force `spawn_blocking`, and its handoff cost, around every trivial foreign call. The lint leaves the decision to the programmer, who knows what the function does, while pointing at the risk.

## Drawbacks

There are two pools to size. A poorly chosen cap on the blocking pool causes either excessive thread growth or queueing delay, and neither is visible at the call site that caused it.

The mechanism depends on the programmer judging which foreign calls block. The lint names the risk but cannot decide it, and a blocking call left on a worker still stalls that worker.

Moving work to the blocking pool has a handoff cost, so wrapping every foreign call out of caution is also wrong. The cost should be paid only where a call warrants it.

## Rationale and alternatives

**Do nothing.** Document that foreign calls block and advise against long ones in tasks. This needs no runtime work, but the scheduler stalls under exactly the workloads that call native libraries heavily.

**Detect stalled workers and add threads.** The scheduler notices a worker stuck in a foreign call and starts a replacement. Nothing changes at the call site, but detection is heuristic and late: the latency has already been paid when a stall is recognized. Under load it also produces unbounded thread growth with no signal in the source.

**A separate blocking pool with `spawn_blocking` (chosen).** It is explicit and bounded, and the cost appears where it is incurred. It requires the programmer to recognize blocking calls and adds a second pool to tune, but both costs are visible.

**Making direct foreign calls an error.** Rejected for the reason given above: the compiler cannot distinguish blocking from non-blocking foreign functions.

## Prior art

Tokio's `spawn_blocking` and its separate blocking thread pool solve the same problem for an M:N runtime that coexists with blocking native code. Go's runtime takes the stall-detection approach: a goroutine in a system call or cgo call releases its processor, and the runtime starts or reuses another thread.

## Unresolved questions

None.

## Future possibilities

The lint could learn to recognize foreign functions that are known not to block, reducing warnings on trivial calls. The blocking pool's cap is global; whether it should ever be set per call site is left to evidence from real workloads.
