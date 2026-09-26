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

The list of RFCs, grouped by area, is in [`INDEX.md`](INDEX.md).

## Status values

- **Draft** — open for discussion, not yet decided.
- **Accepted** — decided; this is how Paco behaves, or will.
- **Rejected** — considered and declined.
- **Superseded by RFC NNNN** — replaced; read the newer RFC instead.

## Proposing an RFC

1. Copy [`0000-template.md`](0000-template.md) to `text/NNNN-short-name.md`,
   `NNNN` being the next free number.
2. Fill it in, add it to [`INDEX.md`](INDEX.md) under its area, and open a
   pull request.
3. Discussion happens on the PR. Once resolved, a maintainer merges it with
   `Status: Accepted`, or closes it and records `Status: Rejected`.

An RFC that changes an accepted one either edits it in the same pull request
(for corrections and clarifications) or supersedes it (for a different
decision).

## Everyday work

Bugs, features already covered by an accepted RFC, and chores are tracked as
issues on the relevant repository, not here.
