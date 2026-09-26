# RFC 0015: Standard Library Scope

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A type or module belongs in the standard library, `stdlib`, only for one of four reasons: the compiler recognizes it by name, it is a prelude entry, it is a thin wrapper over a runtime intrinsic or an `extern` block, or every official library's public interface is written in terms of it. The first three are checked mechanically. A compiler feature being usable with a type is not a reason to put that type in `stdlib`. Domain types such as `Tensor`, `Matrix` and `DataFrame` are ordinary libraries built from general language mechanisms, with no privileged API. The compiler never names a library type. The core (compiler, runtime and `stdlib`) is versioned as one unit and ships with the toolchain; official libraries are versioned independently and fetched like any other dependency.

## Motivation

Without a rule for what belongs in `stdlib`, it grows by accretion. Every type placed there is released on the compiler's schedule, so a bug fix to a domain type waits for a compiler release. Every link requirement placed there applies to every program that imports it. And every domain type there tempts the compiler to depend on it, which makes the type impossible to replace or evolve independently.

Domain types need a design principle too. Numeric and data-analysis work needs matrices, tensors and data frames, and there is a choice between building them into the language with dedicated syntax, or giving the language general mechanisms and implementing the types as libraries. The second option keeps the language uniform and puts third-party libraries on the same footing as official ones.

## Guide-level explanation

`stdlib` holds what the language itself needs: the prelude (`Option`, `Result`, the operator and conversion traits, `Vec`, `Map`, `Set`), string and I/O functions over the runtime, synchronization primitives, testing assertions, and the few items the compiler treats specially, such as `grad` ([RFC 0021](0021-automatic-differentiation.md)) and the named-dimension vocabulary `Shaped` and `DimError` ([RFC 0020](0020-shape-generics.md)).

Everything else is a library. An official library lives in its own repository, has its own version, and is fetched with `paco get` like any other dependency ([RFC 0014](0014-packages.md)):

```paco
module main;

use stdlib::io;
use example.com/team/tensor;

fn main() {
    // io:: and tensor:: are called the same way; neither is privileged
}
```

A type like `Tensor` or `Matrix` is written with the same tools any user has: operator traits ([RFC 0004](0004-traits.md)), generics, `comptime` code generation ([RFC 0016](0016-comptime.md)), and `#[repr]` layout control. The compiler knows nothing about `Matrix` beyond what it knows about any generic type that implements `Add`, `Mul` and `Index`. A user-defined matrix type built the same way is indistinguishable from an official one.

`comptime` lets a library generate specialized code without compiler support. A data frame whose schema is a struct can derive typed column accessors, so column access is checked at compile time and involves no runtime lookup:

```paco
#[derive(Schema)]
struct LogRow {
    timestamp: i64,
    level:     string,
    message:   string,
}

let mut df = DataFrame<LogRow>::new();
df.push(LogRow { timestamp: 1_000, level: "INFO", message: "started" });
let ts: &[]i64 = df.column(.timestamp);
```

## Reference-level explanation

**Admission.** A type or module is in `stdlib` only for one of these reasons:

1. **Compiler-recognized.** It carries a `#[builtin(...)]` attribute, or the compiler recognizes its exact name or path during type checking or lowering. Examples: `grad`; the assertions in `stdlib::test`; `Shaped`, `DimError` and the dimension intrinsics `dim`, `with_dims`, `as_dims`, `assume_dims` and `erase_dims`.
2. **Prelude entry.** It is listed in the prelude ([RFC 0013](0013-prelude.md)), or the module exists only to make prelude names importable by an explicit path (for example `stdlib::collections`, `stdlib::sync`).
3. **Runtime wrapper.** It is a thin wrapper over a runtime-intrinsic function the compiler resolves by name, or over an `extern "C"` block. Example: `stdlib::io` wrapping the runtime's file-reading and stderr functions.
4. **Common vocabulary.** No compiler feature needs it, but every official library's public interface is written in terms of it.

Reasons 1 to 3 are structural facts about the source and are checked mechanically in CI. Reason 4 is a judgment, made and recorded once per item.

**Usefulness is not admission.** Shape generics, `Differentiable`, `#[derivative]`, SIMD, the float types and the numeric traits work on any type that meets their requirements. None of them needs a particular tensor or matrix type, so none of them is a reason for such a type to be in `stdlib`.

**Dependency rules.**

- The compiler and runtime never depend on or name a library type. The only names the compiler knows are those admitted by reasons 1 to 3.
- `stdlib` imports only `stdlib`.
- An official library imports only `stdlib` and other official libraries. It never depends on community code, and official libraries never form a cycle.

**Versioning.** The core (compiler, runtime and `stdlib`) has one version number; a release of one is a release of all three, because the prelude and the compiler that implements it are one contract. Each official library follows its own semantic version and declares in its `paco.mod` the range of core versions it supports.

**Distribution.** The installed toolchain ships `stdlib` alongside the compiler, so an installed `paco` finds its standard library without any environment variable or source checkout. Official libraries are not bundled; a program fetches them with `paco get`.

**Linking.** Because a program links only what it imports ([RFC 0014](0014-packages.md)), a library that binds a foreign library affects only the programs that import it. A `stdlib` module never forces dynamic linking on a program that does not use foreign code.

## Drawbacks

Users have two places to look for functionality: `stdlib`, and the official libraries. Official libraries must be listed in one place and cross-linked from the core documentation.

A compiler change can break an official library without the compiler's own test suite noticing, since the library is built and released separately.

Domain ergonomics depend on the libraries being written. The language provides the mechanisms, but a user does not get a good `DataFrame` until someone builds one.

## Rationale and alternatives

Building `Matrix` and `DataFrame` into the language, with matrix literals or broadcast operators, would allow tighter syntax. It would also create types no user can replicate, put domain knowledge into the compiler, and freeze an API on the compiler's release cycle.

Keeping domain types in `stdlib` as ordinary code avoids compiler knowledge but still ties their releases to the core and applies their link requirements to every program that imports them.

Leaving domain types entirely to the community costs the core nothing, but fragments the ecosystem and leaves no common vocabulary for libraries to share. Official libraries outside `stdlib` keep a maintained, coordinated set without coupling it to the compiler.

Deciding `stdlib` membership by review alone is flexible but produces inconsistent answers over time. Three of the four reasons are checkable facts, so they are checked by tools, and only the fourth needs a person.

## Prior art

Swift's compiler differentiates any type conforming to `Differentiable` and ships no tensor type in its standard library. C++'s Eigen uses expression templates to generate fused numeric kernels at compile time with no compiler knowledge of matrix types. Python keeps NumPy, pandas and SciPy outside the language core; PEP 465 added only the `@` operator and its `__matmul__` hook. Gleam and Crystal keep one core repository and put official libraries in separate repositories.

## Unresolved questions

None.

## Future possibilities

A `MatMul` operator trait with an `@` infix operator, following PEP 465, could be added if usage shows the ergonomic gain is worth the grammar change. It would be an operator hook like `Mul`, not compiler knowledge of any type.
