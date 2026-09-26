# RFC 0009: Arrays and Slices

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Paco has one family of contiguous sequence types. An array `[T; N]` has its length in its type and allocates nothing. `[]T` is an owned, fixed-length heap buffer that frees its elements when it drops. `&[]T` and `&mut []T` are borrowed views of contiguous elements, produced by the ordinary borrow operator. `Vec<T>` is the growable member of the family.

## Motivation

A program that works with sequences of values needs to answer three separate questions about each one: who owns the elements, whether the sequence can change length, and whether the length is known at compile time. Each combination that matters in practice should have exactly one type, and the relationship between the types should follow from rules the language already has, not from special cases.

A buffer stored in a struct field is the case that forces the design. A struct that holds its own data must own it, and a struct should not need a lifetime parameter just to hold a sequence. At the same time, a function that only reads a sequence should be able to accept any contiguous storage without taking ownership.

## Guide-level explanation

An array has a fixed length that is part of its type. It is written as a list of elements or as a repeated value:

```paco
let rgb: [i64; 3] = [255, 128, 0];
let zeros: [f32; 64] = [0.0; 64];   // 64 elements, all 0.0
```

An array allocates nothing on its own. An array whose elements are all `Copy` is itself `Copy`.

`[]T` is an owned buffer whose length is fixed once it is created. It owns its elements, frees them when it drops, and moves like any other owning value. It has no spare capacity and cannot grow:

```paco
pub struct Linear {
    w: []f32,          // Linear owns its weights; no lifetime needed

    pub fn weights(&self) -> &[]f32 { &self.w }
    pub fn into_weights(self) -> []f32 { self.w }
}
```

A function that only needs to read or write elements takes a borrowed view, `&[]T` or `&mut []T`:

```paco
fn sum(xs: &[]f32) -> f32 {
    let mut total = 0.0;
    for i in 0..xs.len() { total += xs[i]; }
    total
}

fn scale(xs: &mut []f32, k: f32) {
    for i in 0..xs.len() { xs[i] *= k; }
}
```

Borrowing a range of a buffer produces a view of that range:

```paco
let head: &[]f32 = &buf[0..n];
```

When the sequence needs to grow, use `Vec<T>` ([RFC 0011](0011-collections.md)). The three owning and borrowing forms compare as follows:

| Type | Owns | Grows |
|---|---|---|
| `Vec<T>` | yes | yes |
| `[]T` | yes | no |
| `&[]T` / `&mut []T` | no | no |

## Reference-level explanation

The type forms are:

```ebnf
ArrayType = "[" Type ";" IntegerLiteral "]" ;   (* [i64; 64] *)
SliceType = "[" "]" Type ;                      (* []f32 *)
```

A view is a `SliceType` under a `BorrowType`: `&[]T`, `&mut []T`, or with an explicit lifetime, `&'a []T`. There is no separate view type. The borrow rules of [RFC 0001](0001-ownership-and-borrowing.md) apply to views unchanged: any number of `&[]T` or exactly one `&mut []T` at a time.

The literal forms are:

```ebnf
SliceExpr = "[" [ ExprList ] "]"      (* [1, 2, 3]   [] *)
          | "[" Expr ";" Expr "]" ;   (* [0; 64] *)
```

`[e; n]` fills `n` elements with the value `e`. For an array type, `n` must be known at compile time, because it is part of the type.

Slicing a range yields a view: `&buf[a..b]` has type `&[]T`, and `&mut buf[a..b]` has type `&mut []T`.

`&[]T` is represented as a fat pointer: a data pointer and a length. The compiler collapses what would otherwise be a pointer to a pointer, so indexing through a borrowed view costs one indirection, not two.

`[]T` is not `Copy`. It owns a heap buffer and moves on assignment, like `Vec<T>` and `string`. Arrays are `Copy` when their element type is.

## Drawbacks

A function that only reads a sequence takes `&[]T`, not `[]T`. Every read-only call site writes one explicit `&`.

There are four sequence types to learn: arrays, `[]T`, views, and `Vec<T>`. Each one answers a different combination of ownership, growth, and compile-time length.

## Rationale and alternatives

`[]T` as a borrowed view by default was considered. Parameters would read `x: []f32` instead of `x: &[]f32`, which is lighter at call sites. But this reading still needs a separate owned, fixed-length type for the cases that own their data. And a struct field of type `[]T` would be a borrow, so the struct would need a lifetime parameter. [RFC 0001](0001-ownership-and-borrowing.md) keeps lifetime annotations out of ordinary code, and struct fields are ordinary code.

Making `[]T` owned lets the borrow operator produce the view with no new concept. `&` already means "a view of" everywhere else in the language, so `&[]T` follows from the general rule. The owned reading has two concepts where the view reading needs three: the owned buffer and its borrow, versus a view type, an owned fixed-length type, and a borrow of that owned type.

## Prior art

Go's `[]T` is a view over a backing array with its own length and capacity, and is the reading this RFC decides against. Rust separates `[T; N]`, `Box<[T]>`, `&[T]`, and `Vec<T>`, the same four positions as Paco's family. C arrays carry no length at runtime; Paco's views always do.

## Unresolved questions

None.

## Future possibilities

None.
