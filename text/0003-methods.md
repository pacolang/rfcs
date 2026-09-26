# RFC 0003: Methods

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A struct's or enum's methods are written inside its own definition block, so its data and its operations read as one unit. A separate `methods T { ... }` block exists only to extend a type defined in another module. Every method writes its receiver explicitly as `&self`, `&mut self` or `self`, and a function in the block with no receiver is an associated function, called as `Type::name(...)`.

## Motivation

A language with methods has to decide where they live relative to the data. If methods are separated from the type, a reader has to find other blocks, possibly in other files, to learn what the type can do. If methods are free functions with a receiver parameter, a type's behavior can be spread across a whole package with nothing keeping it together. Classes keep data and behavior together but bring inheritance and implicit `this`.

Paco wants a type's definition to show its data and its primary operations together, without inheritance, and still allow adding methods to a type the programmer does not own. It also wants exactly one way to write a method on a local type, so that every codebase places methods the same way.

## Guide-level explanation

Methods on a type you define go inside its block, next to the fields:

```paco
pub struct File {
    handle: RawHandle,

    pub fn open(path: &string) -> Result<File, IoError> { /* ... */ }
    pub fn read(&self, buf: &mut []byte) -> Result<i64, IoError> { /* ... */ }
    pub fn seek(&mut self, pos: i64) { /* ... */ }
    pub fn close(self) { /* ... */ }
}
```

Each method states how it takes the value:

- `&self` borrows it for reading. This is the common case.
- `&mut self` borrows it exclusively for writing.
- `self` takes ownership and consumes it, typically to turn the value into something else (`into_bytes()`).

`self` alone always moves. It is never a silent borrow. The compiler warns when a method takes `self` by move but only reads it, and suggests `&self`.

A function in the block without a receiver is an associated function. It is called through the type name, which is how constructors work:

```paco
let f = File::open(&path)?;
let mut v = Vec::new();
```

At a call site, `value.method(args)` borrows or moves `value` as the receiver declares; the caller does not write the `&`.

To add methods to a type from another module, write a `methods` block in your own code:

```paco
methods tensor::Tensor<f32> {
    fn normalize(&mut self) { /* ... */ }
}
```

Extending a generic type declares its parameters and any bounds ([RFC 0004](0004-traits.md)):

```paco
methods<T> Vec<T> where T: Display {
    fn show_all(&self) { /* ... */ }
}
```

## Reference-level explanation

A `struct` or `enum` block may contain fields (or variants), methods, associated functions, and associated type declarations used to satisfy traits ([RFC 0004](0004-traits.md)). Each member is private unless marked `pub` ([RFC 0012](0012-modules-and-visibility.md)).

The receiver grammar is:

```ebnf
Receiver = [ "&" | "&mut" ] "self" ;
```

A receiver, when present, is the first parameter. `&self` and `&mut self` are ordinary shared and exclusive borrows of the value and follow the aliasing rule of [RFC 0001](0001-ownership-and-borrowing.md). `self` is an ordinary move. A `&mut self` method may be called only on a mutable binding or through a `&mut` borrow ([RFC 0002](0002-mutability.md)).

A `methods` block is written `methods [GenericParams] Type [WhereClause] { ... }` and may contain methods, associated functions and associated type declarations. It is for types defined in another module. Methods on a type defined in the current module must be written in that type's own block; `methods` is not an alternative form for them.

Method resolution for a concrete type looks in two places: the type's own block, and the `methods` blocks for that type that are in scope.

The compiler warns when a method's receiver is `self` and the body only reads from `self`, suggesting `&self`.

## Drawbacks

A type with many methods produces one long definition block, and its methods cannot be spread across files the way separate blocks would allow.

Method lookup has two sources, the type's block and extension blocks, instead of one.

## Rationale and alternatives

**Receiver functions at top level.** Methods are ordinary functions with a receiver parameter and may be declared anywhere in the package. Nothing keeps a type's methods together, and it adds a second way to define a method.

**Separate implementation blocks for every type.** Data and behavior are fully decoupled and blocks may live in several files. For a type defined and used locally this buys nothing, and every block repeats the type's name.

**Methods only inside the type.** This has no boilerplate for the local case but offers no way to extend a type whose source you do not control, which is a common need.

**Nested methods plus `methods T` for external types (chosen).** The local case needs no extra block, and extension is still possible where nesting cannot reach.

**Receiver spelling.** A suffix form such as `self&` was considered. Paco writes `&` as a prefix everywhere a borrow appears: in borrow types (`&T`, `&mut []f32`) and in borrow expressions (`&x`, `&mut buf`). A suffix at the receiver would be the only exception, at the most frequently written position in the language. `&self` and `&mut self` keep one rule.

**Implicit receiver.** Making bare `self` mean a borrow would save a character on the common case, but it would hide whether a method consumes its value. Writing the receiver explicitly makes ownership visible in every signature.

## Prior art

Zig and Swift define methods inside the type body. Rust's `impl` blocks, and extensions in Swift and Kotlin, add methods to a type outside its definition, the role `methods T` plays here. Go's receiver functions show the top-level alternative. Rust writes receivers as `&self`, `&mut self` and `self`.

## Unresolved questions

None.

## Future possibilities

If long definition blocks become a recurring problem, a lint for oversized type bodies could be added without changing where methods live.
