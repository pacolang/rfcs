# RFC 0011: Collections

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

`Vec<T>`, `Map<K, V>`, `Set<T>`, `StringBuf`, and every other heap-allocating collection are constructed through associated functions such as `Vec::new()`. The language has no collection literal syntax. Building from an existing sequence uses `collect()`. Arrays (`[1, 2, 3]`, `[0; 64]`) keep their literal forms because they allocate nothing.

## Motivation

A literal syntax for a built-in collection gives standard-library types a grammar rule that no user-defined type can have. A user's graph or priority queue would always be constructed differently from `Vec` and `Map`, however similar in purpose.

A literal can also hide an allocation behind punctuation. When every heap allocation is a named function call, the cost of the code is visible in the code, whichever type is allocating.

## Guide-level explanation

Every collection is constructed with an associated function, then filled with the type's own methods:

```paco
let mut v = Vec::new();
v.push(1);
v.push(2);

let mut m = Map::new();
m.insert("host", "localhost");
m.insert("port", "8080");
```

To build a collection from a sequence you already have, collect an iterator:

```paco
let squares = (1..=5).map(|n| n * n).collect<Vec<i64>>();
```

The same pattern applies to a user-defined type: `Graph::new()` followed by `add_edge` calls reads exactly like `Vec::new()` followed by `push` calls.

Arrays are the exception. `[1, 2, 3]` and `[0; 64]` are literals because an array's length is part of its type and it allocates nothing ([RFC 0009](0009-arrays-and-slices.md)). There is no allocation for the literal to hide.

## Reference-level explanation

`Vec::new()`, `Map::new()`, `Set::new()`, `StringBuf::new()`, and `Rc::new(value)` are ordinary associated functions: functions with no receiver, declared inside the type's own block ([RFC 0003](0003-methods.md)).

The grammar has no production for a collection literal. The only bracketed literal is the array literal. The standard library's collections follow the same rule as user-defined types, with no compiler exception for any of them.

## Drawbacks

For a small collection whose elements are all known when the code is written, `Vec::new()` followed by several `push` calls is longer than a literal would be.

## Rationale and alternatives

Literal syntax, such as `[1, 2, 3]` for `Vec` or `{k: v}` for `Map`, is the shortest option for small, known collections. It privileges the types the grammar names, leaves every user-defined collection without an equivalent, and adds rules the parser and formatter must carry.

Macro-based construction, such as `vec![...]` or `map!{...}`, would in principle be open to any type that defines its own macro. Paco has no syntax macros ([RFC 0016](0016-comptime.md)), so this option does not exist without adding a macro system.

Associated functions treat every type the same and keep every allocation a named call. Building from an iterator with `collect()` covers the common case of constructing a collection from data the program already has.

## Prior art

Rust constructs `Rc` with `Rc::new()` and vectors with the `vec!` macro. Python and JavaScript have list and dictionary literals.

## Unresolved questions

None.

## Future possibilities

Initializing a `Map` from a fixed set of key-value pairs is the most likely construction to feel verbose. If it does, a general `comptime` construction helper that any type can use ([RFC 0016](0016-comptime.md)) fits this design better than a literal rule for one standard-library type.
