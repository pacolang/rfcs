# RFC 0002: Mutability

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Mutability is a property of the binding that holds a value, never of a type or a field. A value bound with `let` is entirely read-only; a value bound with `let mut` is entirely mutable. A `&mut` borrow of a struct grants write access to all of its fields. When one part of a value must change while the rest stays fixed, the tools are encapsulation and the interior mutability containers from [RFC 0001](0001-ownership-and-borrowing.md).

## Motivation

Something has to decide whether a struct's fields can be written. The two candidates are the binding that holds the value and a modifier on each field in the type's definition. Allowing both would give the programmer two overlapping rules to track, and would force the borrow checker to decide what a `&mut` borrow of the whole struct means for fields that did not opt into mutation. Paco picks one rule.

## Guide-level explanation

Whether a field can be written depends only on how the value was bound:

```paco
let cfg = Config { host: "localhost", port: 8080 };
cfg.port = 9090;     // ERROR: `cfg` is an immutable binding

let mut cfg2 = Config { host: "localhost", port: 8080 };
cfg2.port = 9090;    // OK
cfg2.host = "prod";  // OK: the whole struct is mutable
```

Method receivers follow the same rule ([RFC 0003](0003-methods.md)). A `&self` method reads, whatever the caller's binding. A `&mut self` method requires the caller to hold a `let mut` binding or a `&mut` borrow.

To let callers change one field but not another, make the fields private and expose a narrow method such as `fn set_port(&mut self, port: i64)`. To mutate data that is shared, wrap it in a container:

- `Cell<T>` for `Copy` values, with no runtime check;
- `RefCell<T>` for other values under `Rc`, with the borrow rule checked at runtime;
- `Mutex<T>` or `RwLock<T>` for values shared across tasks under `Arc`.

## Reference-level explanation

`let` introduces an immutable binding and `let mut` a mutable one. Assigning to a binding, to any field reached through it, or to any element reached through it requires the binding to be mutable or to be reached through a `&mut` borrow. Taking `&mut` of a place has the same requirement.

There is no field-level mutability modifier. A `&mut T` borrow of a struct or enum grants write access to every field, and a `&T` borrow grants write access to none. The borrow checker tracks only whether a borrow is shared or exclusive, never per-field write permissions across a call.

`Cell`, `RefCell`, `Mutex` and `RwLock` are ordinary library types. Their methods mutate the contained value through a shared borrow of the container, and the checker needs no special rule for them.

## Drawbacks

A type cannot state in its own definition that one field may change and another may not. That invariant moves into the type's methods (private fields plus setters), which is more code than a field modifier would be.

## Rationale and alternatives

**Per-field `mut` modifiers.** They express "mostly read-only, one counter" directly in the struct. The cost appears with borrows: the language would have to define whether `&mut T` grants access to unmarked fields, and enforce that definition across every call. Programmers would also track two rules at once, the binding's and the field's.

**Binding-level mutability only (chosen).** The rule fits in one sentence, `&mut T` has a single meaning, and bindings and receivers express mutability the same way. Control over which fields can change is handled by visibility and methods, which Paco already has.

## Prior art

Rust uses the same `let` / `let mut` binding-level rule with no field modifiers. Several ML-family languages instead mark individual record fields as mutable.

## Unresolved questions

None.

## Future possibilities

None.
