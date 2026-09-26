# Paco RFCs

**Read this in:** **English** · [Português](README.pt-BR.md) · [Español](README.es.md)

Design decisions for the [Paco programming language](https://github.com/pacolang/paco),
recorded as RFCs — one file per decision, living in [`text/`](text/).

## What belongs in an RFC

An RFC describes behavior a Paco user can observe:

- **The language** — syntax, types, semantics, diagnostics.
- **The standard library** — what `std` and the prelude contain, and the
  rules for what may go there.
- **The toolchain** — behavior of `paco` commands and package management
  that users depend on.

An RFC is *not* the place for compiler internals (intermediate
representations, code generators, caches), roadmaps, priorities, or project
planning. Those are implementation choices and live with the code in the
repository they concern.

## Index

**Memory and behavior**

| RFC | Title |
|---|---|
| [0001](text/0001-ownership-and-borrowing.md) | Ownership and Borrowing |
| [0002](text/0002-mutability.md) | Mutability |
| [0003](text/0003-methods.md) | Methods |
| [0004](text/0004-traits.md) | Traits |
| [0005](text/0005-error-handling.md) | Error Handling |

**Types and values**

| RFC | Title |
|---|---|
| [0006](text/0006-integer-overflow.md) | Integer Overflow |
| [0007](text/0007-floating-point-types.md) | Floating-Point Types |
| [0008](text/0008-constants.md) | Constants |
| [0009](text/0009-arrays-and-slices.md) | Arrays and Slices |
| [0010](text/0010-strings.md) | Strings |
| [0011](text/0011-collections.md) | Collections |

**Program structure**

| RFC | Title |
|---|---|
| [0012](text/0012-modules-and-visibility.md) | Modules and Visibility |
| [0013](text/0013-prelude.md) | The Prelude |
| [0014](text/0014-packages.md) | Packages |
| [0015](text/0015-standard-library-scope.md) | Standard Library Scope |
| [0016](text/0016-comptime.md) | Compile-Time Execution |

**Concurrency and foreign code**

| RFC | Title |
|---|---|
| [0017](text/0017-tasks-and-channels.md) | Tasks and Channels |
| [0018](text/0018-ffi-and-unsafe.md) | Foreign Functions and `unsafe` |
| [0019](text/0019-blocking-calls.md) | Blocking Calls |

**Numerics**

| RFC | Title |
|---|---|
| [0020](text/0020-shape-generics.md) | Shape Generics |
| [0021](text/0021-automatic-differentiation.md) | Automatic Differentiation |

## Status values

- **Draft** — open for discussion, not yet decided.
- **Accepted** — decided; this is how Paco behaves, or will.
- **Rejected** — considered and declined.
- **Superseded by RFC NNNN** — replaced; read the newer RFC instead.

## Proposing an RFC

1. Copy [`0000-template.md`](0000-template.md) to `text/NNNN-short-name.md`,
   `NNNN` being the next free number.
2. Fill it in and open a pull request.
3. Discussion happens on the PR. Once resolved, a maintainer merges it with
   `Status: Accepted`, or closes it and records `Status: Rejected`.

An RFC that changes an accepted one either edits it in the same pull request
(for corrections and clarifications) or supersedes it (for a different
decision).

## Everyday work

Bugs, features already covered by an accepted RFC, and chores are tracked as
issues on the relevant repository, not here.
