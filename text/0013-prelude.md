# RFC 0013: The Prelude

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

The prelude is the set of names available in every file without a `use`. It contains what the language's own rules force a programmer to use, not what is merely convenient: the types the compiler desugars to, the traits it resolves operators and derives against, the collections, the ownership wrappers the language names as the answer to specific problems, and the types the concurrency keywords produce. This RFC lists every prelude name, the names deliberately left out, and the shadowing rule.

## Motivation

Name resolution needs a fixed list of names that are in scope everywhere. Without one, the compiler has nothing to consult and examples cannot be checked.

The list matters more in Paco than in a language with selective imports. [RFC 0012](0012-modules-and-visibility.md) allows only whole-module imports, so a name outside the prelude is always qualified at the call site: `sync::Mutex::new(...)`, not `Mutex::new(...)`. A frequently used name left out of the prelude is noisy at every occurrence.

## Guide-level explanation

The prelude holds five groups of names, plus two free functions.

**Desugaring targets.** `Option`, `Some`, `None`, `Result`, `Ok`, and `Err`. The `?` operator and every fallible signature depend on them directly ([RFC 0005](0005-error-handling.md)).

**Traits the compiler resolves.** `From`, `Into`, `Display`, `Clone`, `Copy`, `Drop`, `Eq`, `Ord`, `Hash`, `Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Index`, `Iter`, `Call`, and `Numeric`. Operators resolve against them ([RFC 0004](0004-traits.md)), `?` calls `From::from`, `for` desugars through `Iter`, and `#[derive(...)]` names them. A trait the compiler already knows by name should not need an import.

**Collections.** `Vec`, `Map`, `Set`, and `StringBuf`. Collections have no literal syntax and are always built with a call such as `Vec::new()` ([RFC 0011](0011-collections.md)). Requiring an import as well would make the most common construction in a program cost two lines.

**Ownership escape hatches.** `Rc`, `Arc`, `Cell`, `RefCell`, `Mutex`, and `RwLock`. Other RFCs name these as the answer to a problem the language creates. [RFC 0001](0001-ownership-and-borrowing.md) names `Rc` and `Arc` for ownership that is not tree-shaped. [RFC 0002](0002-mutability.md) names `Cell` and `RefCell` for mutating one field while the rest of a value stays immutable. [RFC 0008](0008-constants.md) forbids mutable globals and names `Arc<Mutex<T>>` as the replacement. A construct the language mandates should be easy to reach.

**Concurrency.** `channel`, `Sender`, `Receiver`, `spawn_blocking`, and `TaskPanic`. `spawn` and `select` are keywords, and the types they produce and consume must be reachable without ceremony ([RFC 0017](0017-tasks-and-channels.md), [RFC 0019](0019-blocking-calls.md)). `TaskPanic` arrives in the `Err` arm of `handle.join()` without the caller ever naming it.

**Free functions.** `print` and `panic`.

## Reference-level explanation

The complete prelude:

| Group | Names |
|---|---|
| Desugaring targets | `Option`, `Some`, `None`, `Result`, `Ok`, `Err` |
| Traits the compiler resolves | `From`, `Into`, `Display`, `Clone`, `Copy`, `Drop`, `Eq`, `Ord`, `Hash`, `Add`, `Sub`, `Mul`, `Div`, `Rem`, `Neg`, `Index`, `Iter`, `Call`, `Numeric` |
| Collections | `Vec`, `Map`, `Set`, `StringBuf` |
| Ownership escape hatches | `Rc`, `Arc`, `Cell`, `RefCell`, `Mutex`, `RwLock` |
| Concurrency | `channel`, `Sender`, `Receiver`, `spawn_blocking`, `TaskPanic` |
| Free functions | `print`, `panic` |

Names deliberately left out:

| Name | Where it lives | Why |
|---|---|---|
| `Duration` | `stdlib::time` | Needed only for timeouts and sleeps. One import in the files that use them. |
| `Bencher` | injected by `#[bench]` | Reachable only inside a `#[bench]` function; the attribute brings it into scope the way a parameter would. |
| `read_file`, `print_err`, ... | `stdlib::io` | I/O is a module a program opts into, not an ambient capability. |
| `sqrt`, `pow`, ... | `stdlib::math` | Same reasoning as I/O. |
| `grad`, ... | `stdlib::autodiff` | Same reasoning as I/O. |
| `Matrix`, `DataFrame`, `Tensor` | official libraries outside the standard library | These are library types with no privileged status ([RFC 0015](0015-standard-library-scope.md)). A prelude entry is a privilege. |

A module-level declaration may shadow a prelude name. The local definition wins, with no warning: a module that defines its own `Set` gets its own `Set`. This keeps the prelude from acting as a list of reserved words.

The operator traits are the exception. Shadowing `Add` at module scope does not change what `+` means in that module, because each operator binds to the prelude trait itself, not to whatever `Add` currently names. Otherwise a local type could redefine arithmetic for a whole module without anyone intending it.

## Drawbacks

Each prelude name is a name a module cannot declare without shadowing. The list has about forty names, each justified on its own, which together take a noticeable share of the namespace.

The prelude is larger than it would be in a language with selective imports. This is a direct consequence of [RFC 0012](0012-modules-and-visibility.md)'s whole-module imports.

## Rationale and alternatives

A minimal prelude, with only `Option`, `Result`, and the operator traits, has the smallest namespace cost. Under whole-module imports it would force `Vec::new()` to need an import, and the shared-state idiom [RFC 0008](0008-constants.md) mandates would become `Arc<sync::Mutex<T>>` with an import and two qualifications.

No prelude at all makes every name's origin visible at every use, the same argument that supports qualified imports. But `Option` and `Result` appear in nearly every signature, and `?` desugars to `From::from`. Requiring an import for names the compiler itself generates during desugaring would be incoherent.

The chosen prelude costs more namespace than the minimal one. In exchange, the idioms the language mandates are convenient to write.

## Prior art

Rust's prelude has the same general shape: a small set of ambient names covering what operators and desugaring depend on. It leaves out `Rc`, `Arc`, `Mutex`, and `RefCell`, which Rust programs bring in with selective `use` once per file.

## Unresolved questions

None.

## Future possibilities

As the standard library grows, new names may be proposed for the prelude. Each is judged by the same test: whether the language's own rules force a programmer to use it, not whether it would be convenient to have in scope.
