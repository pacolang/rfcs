# RFC 0010: Strings

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Paco has two string types: `string`, an owned, immutable, UTF-8 primitive, and `StringBuf`, a standard-library type for building a string incrementally. Strings have no range indexing (`s[n..m]`) and no byte indexing (`s[n]`). Substrings come from `s.get(range)`, which returns `Option<&string>`; raw bytes come from `s.as_bytes()`; characters come from `s.chars()`.

## Motivation

A `string` is always valid UTF-8, and one character occupies one to four bytes. A byte range can end in the middle of a character, and the language has to decide what happens then. There are three options: offer range syntax and panic on a bad boundary, offer range syntax and return an `Option`, or offer no range syntax on strings.

A bad boundary is a condition the program can handle, not a broken invariant. Under [RFC 0005](0005-error-handling.md), a handleable condition is a value, not a panic. That rules out the first option. The second has its own problem, described under *Rationale and alternatives*.

## Guide-level explanation

`string` holds text. It is immutable. To build text piece by piece, use `StringBuf`:

```paco
let mut b = StringBuf::new();
b.push_str("caf");
b.push('é');
let s: string = b.to_string();
```

A substring is a method call that returns an `Option`:

```paco
let s = "café";

match s.get(0..3) {
    Some(sub) => print(sub),    // "caf": byte 3 is a character boundary
    None      => handle_error(),
}

let bad = s.get(0..4);          // None: byte 4 falls inside 'é'
```

For raw bytes, convert to `[]byte` first. Byte slices carry no encoding promise, so ordinary slicing ([RFC 0009](0009-arrays-and-slices.md)) applies:

```paco
let raw: []byte = s.as_bytes();
let head: &[]byte = &raw[0..3];
```

To walk a string, iterate by character or by byte:

```paco
for c in s.chars() { ... }
for (i, c) in s.chars().enumerate() { ... }
let third: Option<byte> = s.bytes().nth(2);
```

`s.len()` counts bytes. `s.chars().count()` counts characters.

## Reference-level explanation

`string` is a primitive type. It owns its buffer, is not `Copy`, and is guaranteed to hold valid UTF-8. `==` on strings compares contents. There is no type named `String`.

`StringBuf` is a standard-library type in the prelude ([RFC 0013](0013-prelude.md)). `push_str` and `push(c)` append in amortized constant time, and `to_string()` copies the result into a `string`.

String methods:

| Method | Returns | Behavior |
|---|---|---|
| `s.get(range)` | `Option<&string>` | `None` when either end of the range falls inside a multi-byte character. Never panics. |
| `s.as_bytes()` | `[]byte` | Copies the bytes into a new buffer. |
| `s.chars()` | iterator of `char` | Yields Unicode code points. |
| `s.bytes()` | iterator of `byte` | Yields raw bytes. `s.bytes().nth(n)` gives `Option<byte>`. |

`string_from_bytes(&bytes, start, end)` converts a byte range back to `Some(string)` when the range is valid UTF-8, and `None` otherwise.

The indexing expressions `s[n]` and `s[a..b]` are type errors when `s` is a `string`.

The `string-byte-boundary` lint reports boundary violations that can be proven at compile time. The `Option` returned by `get` covers the cases the lint cannot decide at compile time.

## Drawbacks

Programmers used to `s[0:3]` or `&s[0..3]` will write it and get an error. The lint and a targeted error message reduce this friction but do not remove it.

## Rationale and alternatives

Range syntax that panics on a bad boundary is the most familiar option. It turns a handleable condition into a crash, which contradicts the rule that panics are for broken invariants.

Range syntax that returns `Option<&string>` avoids the panic. But `arr[0..3]` on a slice gives a view directly, so the same syntax would return an `Option` on one type and a plain value on another. The meaning of an indexing expression would depend on the type to its left.

No range syntax on strings avoids both problems. `s.get(0..n)` and `s.as_bytes()` are different operations, and they look different at the call site. Byte, code point, and grapheme access stay separate, so the cost of each operation is visible; grapheme segmentation is left to libraries.

## Prior art

Rust's `str` panics on direct indexing at an invalid boundary and also offers `str::get`, returning `Option<&str>`. Paco keeps only the `get` form. Go indexes strings by byte without checking UTF-8 boundaries. Python slices by code point and hides the byte representation.

## Unresolved questions

None.

## Future possibilities

A dedicated diagnostic that recognizes `s[n..m]` on a string and suggests the matching `s.get(n..m)` call, rather than a generic type error.
