# RFC 0008: Constants

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

`const` names a value fixed at compile time. Its type annotation is mandatory, and its initializer must be evaluable at compile time: a literal, an operation over other constants, or a `comptime` call. A constant has no address and no storage; it is substituted at each use site. Paco has no `static`, no `static mut`, and no module-level `var`. Mutable state that outlives a scope is owned by something.

## Motivation

Numeric code is full of values that are fixed by design: tolerances, tile sizes, block dimensions, mathematical constants, hyperparameters. They need names, so a program can write `PI` instead of `3.141592653589793` and change a tile size in one place.

Naming values at module scope raises a second question: whether that scope can also hold mutable values. A mutable global is reachable from every task at once and owned by none of them. That is exactly the kind of value the ownership model cannot reason about, and Paco promises freedom from data races at compile time ([RFC 0017](0017-tasks-and-channels.md)). The design of constants has to settle both questions together.

## Guide-level explanation

```paco
pub const EPS: f32 = 1e-6;
const TILE: i64 = 64;
const TILE_AREA: i64 = TILE * TILE;

pub struct Tensor<T> {
    data: []T,

    pub const RANK: i64 = 2;       // associated constant
}
```

Constants live at module scope, or as associated constants inside a `struct`, `enum`, `trait`, or `methods` block.

A constant is not a memory location. It is a name for a value the compiler already has, and each use gets that value substituted, exactly as if the literal had been written there. Using a constant costs what writing the literal costs.

An initializer can call into the compile-time evaluator ([RFC 0016](0016-comptime.md)):

```paco
comptime fn block_size(lanes: i64, unroll: i64) -> i64 {
    lanes * unroll
}

const BLOCK: i64 = block_size(8, 4);
```

### No global mutable state

There is no `static`, no `static mut`, and no module-level `var`. State that must outlive a single scope is owned by something concrete ([RFC 0001](0001-ownership-and-borrowing.md)):

- passed down explicitly as a parameter, or
- shared through `Arc<Mutex<T>>` or `Arc<RwLock<T>>`, with the cost of sharing written where it happens.

## Reference-level explanation

```
ConstDecl = "const" Identifier ":" Type "=" Expr ";" ;
```

A constant declaration follows three rules:

1. **The type annotation is mandatory.** A constant is part of a module's API, and its type is part of that contract regardless of how the initializer computes it.
2. **The initializer is evaluable at compile time.** It is a literal, an operation over other constants, or a call evaluated by `comptime`. Anything else is a compile error.
3. **A constant has no address and no storage.** It is substituted at each use site. Taking a reference to a constant borrows a temporary, as borrowing a literal does.

A constant may be declared at module level, or as a member of a `struct`, `enum`, `trait`, or `methods` block. Visibility follows the ordinary rules ([RFC 0012](0012-modules-and-visibility.md)).

Const generic parameters, such as the dimensions in `Tensor<f32, M, K>`, are specified in [RFC 0020](0020-shape-generics.md).

## Drawbacks

A large constant table is substituted at every use site, which can duplicate data a program would rather keep in one place.

Code ported from languages that rely on package-level globals has to thread state through parameters or wrap it in `Arc`.

## Rationale and alternatives

**No constants.** Literals only, or associated functions that return a value. Cheap to specify, but a language that cannot name a number is unworkable for numeric code, and a function standing in for a constant depends on inlining to cost nothing.

**`const` plus `static` and `static mut`.** Covers every familiar systems case, including large tables and global caches. `static mut` breaks compile-time race freedom, and globals with run-time initialization bring initialization-order questions that substituted constants avoid by construction.

**`const` only.** Covers the need to name values without weakening the concurrency guarantee, and without adding `static`, initialization order, or an `unsafe` carve-out before there is evidence they are needed.

**Optional type annotation.** Inferring a constant's type from its initializer is shorter, but it lets a change to the initializer silently change a public type.

## Prior art

Rust's `const` is a compile-time-evaluated, address-free constant with a mandatory type. Rust's `static mut` and Go's package-level `var` are mutable globals; Go ships a run-time race detector to find the races they allow.

## Unresolved questions

Whether Paco needs an immutable `static`: a value with a stable address, for large tables that should not be duplicated at every use site. It is a narrower need than a mutable global, and it stays open until there is evidence that `const` substitution is a problem in practice.

## Future possibilities

None.
