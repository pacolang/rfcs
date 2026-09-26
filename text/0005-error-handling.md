# RFC 0005: Error Handling

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Recoverable failures in Paco are values: `Result<T, E>` for an operation that can fail, `Option<T>` for a value that can be absent. The `?` operator propagates them. When the error type produced under `?` differs from the error type the enclosing function returns, `?` calls `DstError::from(e)`, using a `From<T>` trait that lives in the prelude. The conversion is always written by the programmer; a missing `From` implementation is a type error. `panic` is reserved for unrecoverable invariant violations, and a panic inside a task is captured at the task boundary.

## Motivation

A program has two kinds of failure. Some are conditions a caller can reasonably handle: a missing file, a malformed input, a string boundary that falls inside a code point, two run-time shapes that disagree. Others are bugs: an invariant the program relies on has been violated. Treating both the same way either makes ordinary failures easy to ignore or makes bugs look recoverable. Paco separates them in the type system.

Propagating a `Result` raises a second problem. The error type returned by a callee is rarely the error type the caller declares. Without a conversion mechanism, every call site needs its own adapter:

```paco
let text = read_file("config.toml").map_err(|e| AppError::Io(e))?;
let cfg  = parse(text).map_err(|e| AppError::Parse(e))?;
```

This is mechanical and repeated in exactly the code that calls the most lower-level APIs. It adds noise without adding information a reader needs.

## Guide-level explanation

### Failures are values

There are no exceptions and no `null`. Two prelude types cover recoverable failure:

```paco
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`?` returns early from the enclosing function. On a `Result`, it returns the `Err`; on an `Option`, it returns `None`:

```paco
fn read_config() -> Result<Config, Error> {
    let text = read_file("config.toml")?;   // if Err, returns early
    let cfg  = parse(text)?;
    Ok(cfg)
}

fn first_admin(us: &[]User) -> Option<&User> {
    let u = us.iter().find(|u| u.admin)?;
    Some(u)
}
```

### Converting errors with `From`

`From<T>` is in the prelude ([RFC 0013](0013-prelude.md)):

```paco
trait From<Src> {
    fn from(e: Src) -> Self;
}
```

An error type absorbs another error type by defining a `from` function for it. Trait satisfaction is implicit ([RFC 0004](0004-traits.md)), so no separate declaration is needed:

```paco
enum AppError {
    Io(IoError),
    Parse(ParseError),

    fn from(e: IoError) -> Self    { AppError::Io(e) }
    fn from(e: ParseError) -> Self { AppError::Parse(e) }
}

fn load() -> Result<Config, AppError> {
    let text = read_file("config.toml")?;   // IoError → AppError::Io
    let cfg  = parse(text)?;                // ParseError → AppError::Parse
    Ok(cfg)
}
```

Every conversion `?` performs is visible in the destination type's own definition.

### Panics are for bugs

`panic` exists for invariant violations the program cannot recover from, never for ordinary control flow. Where a failure is a condition the caller can handle, the API returns a value instead:

- `s.get(range)` on a string returns `Option<&string>` rather than panicking on a boundary inside a code point ([RFC 0010](0010-strings.md)).
- A run-time check that two dynamic dimensions agree returns a `Result` ([RFC 0020](0020-shape-generics.md)).
- Narrowing a float past the target's range produces infinity, not a panic ([RFC 0007](0007-floating-point-types.md)).

### Panics in tasks

A panic inside a task ends only that task. It is captured at the task boundary, and the task's handle reports it through `join()`, which returns `Result<T, TaskPanic>`:

```paco
let h = spawn risky();

match h.join() {
    Ok(value)  => use_it(value),
    Err(panic) => log("task died: " + panic.message()),
}
```

A panicking task is handled the same way as any other fallible operation. See [RFC 0017](0017-tasks-and-channels.md) for the task model.

## Reference-level explanation

When `?` is applied to a `Result<T, SrcError>` inside a function returning `Result<U, DstError>`:

- If `SrcError` is `DstError`, the `Err` value is returned unchanged.
- If they differ, the compiler emits a call to `DstError::from(src_error)` and returns the converted value.
- If no `From<SrcError>` implementation exists for `DstError`, the `?` expression is a type error, reported at that site.

A `from` function with the matching signature is a `From` implementation under the ordinary rules of implicit trait satisfaction. The compiler never synthesizes a conversion that is not written in the program.

A panic is captured at the entry point of the task it occurs in and stored for retrieval through that task's `join()`. It does not propagate past the task boundary.

## Drawbacks

An error type at the boundary of many subsystems accumulates one `from` function per source type. Each is short and explicit, but the block can grow long.

`?` is coupled to one specific trait, `From`. This is a small piece of compiler behavior tied to a library item.

## Rationale and alternatives

**No automatic conversion.** Requiring `.map_err(...)` at every call site keeps every conversion visible where it happens, but it produces the repetitive boilerplate this design removes, and the cost grows with the number of modules a function calls into.

**Compiler-synthesized conversions.** The compiler could infer conversions between error types without programmer-written code. This maximizes ergonomics, but the conversion logic would be something the reader has to trust rather than read.

**Auto-calling a programmer-written `From`.** `?` does the mechanical work of invoking the conversion, and every conversion that exists is written inside the destination type. This removes the boilerplate without hiding any logic.

**Panicking on handleable conditions.** Panicking on a bad string boundary or a shape mismatch is familiar and keeps signatures short, but it makes a recoverable condition unrecoverable and hides the failure path from the signature. Returning `Option` or `Result` makes the failure visible where it can occur.

**Process-wide panics.** Letting a task panic end the process is simpler, but one faulty task would take down every other task in the program. Capturing the panic at the task boundary and returning it as a `Result` lets the spawner decide.

## Prior art

Rust's `Result`, `Option`, `?` operator, and `From`/`Into` traits follow the same shape: `?` calls `From::from` on the error. Go returns errors as ordinary values and recovers panics with `recover`. Erlang isolates failures to a single process and reports them to a supervisor.

## Unresolved questions

None.

## Future possibilities

For the common "wrap this error type in that variant" shape, `comptime` ([RFC 0016](0016-comptime.md)) could derive routine `from` functions, without changing how `?` and `From` interact.
