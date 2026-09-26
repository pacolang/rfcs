# RFC 0001: Ownership and Borrowing

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Every value in Paco has one owner. Assigning, passing or returning a value moves it unless its type is `Copy`. Code that only needs access borrows with `&` (shared) or `&mut` (exclusive), and the compiler infers how long each borrow lives in the common case, so ordinary signatures carry no lifetime annotations. When ownership is not tree-shaped, `Rc<T>` and `Arc<T>`, combined with `Cell`, `RefCell`, `Mutex` or `RwLock`, provide shared ownership at a visible cost. Resources are released deterministically when their owner goes out of scope.

## Motivation

Programs with latency or memory budgets, such as games, real-time audio, low-latency services and numeric code, cannot absorb the pauses and memory overhead of a garbage collector. Manual allocation and deallocation avoids that cost but produces use-after-free, double-free and data-race bugs.

Ownership with compile-time borrow checking removes both problems: memory is freed at a known point, and the invalid accesses are rejected before the program runs. The remaining problem is annotation burden. If every function that takes or returns a borrow must name its lifetimes, the safety model costs attention in every signature. Paco needs the guarantees without making lifetime annotations part of ordinary code.

## Guide-level explanation

A value is owned by the binding that holds it. Assigning it elsewhere moves it, and the old binding cannot be used afterwards:

```paco
fn main() {
    let s = load_text();   // `s` owns a string
    consume(s);            // `s` moves into `consume`
    print(s);              // ERROR: use after move
}

fn consume(s: string) { /* now the owner */ }
```

Values that are only bits are `Copy` and are duplicated instead of moved: the numeric primitives, `bool`, `char`, `byte`, shared borrows, and tuples and arrays of `Copy` elements. Types that own a heap buffer (`string`, `Vec<T>`, `[]T`) move. A user-defined `struct` or `enum` moves by default and opts into copying with `#[derive(Copy)]`, which the compiler accepts only when every field is `Copy`.

To use a value without taking it, borrow it:

```paco
fn length(s: &string) -> i64 { s.len() }             // shared borrow
fn append(b: &mut StringBuf, c: char) { b.push(c) }  // exclusive borrow
```

The aliasing rule is: any number of `&` borrows, or exactly one `&mut`, never both at the same time.

No lifetime appears in either signature. The compiler works out how long each borrow must live from how it is used. A function that returns a borrow is inferred to borrow from its reference arguments (from `self` for a method):

```paco
fn first_word(s: &string) -> &string { /* ... */ }  // result lives as long as `s`
```

When a returned borrow could follow more than one input, inference cannot decide, and the compiler asks for an explicit lifetime. Its error message suggests the exact annotation:

```paco
fn longest<'a>(x: &'a string, y: &'a string) -> &'a string {
    if x.len() > y.len() { x } else { y }
}
```

Structs, enum variants, tuples, collections and closures may hold borrows without annotations. The compiler rejects any way such a value could outlive what it borrows from:

```paco
struct View { r: &i64 }

fn wrap(p: &Point) -> View { View { r: &p.x } }   // fine: borrows the caller's data

fn broken() -> View {
    let t = 5;
    View { r: &t }                                // ERROR: `t` is dropped here
}
```

When a value genuinely needs more than one owner, such as a node in a graph with cycles or state shared between tasks, use reference counting:

```paco
let node = Rc::new(Node { value: 1 });
let other = node.clone();   // increments the count; the node is not copied
```

`Rc<T>` is for a single task and `Arc<T>` is safe to share across tasks. Shared contents are read-only; to mutate them, wrap the data in `Cell<T>` or `RefCell<T>` under `Rc`, or `Mutex<T>` or `RwLock<T>` under `Arc`. Each of these is an ordinary library type whose cost (a count update, a runtime borrow check, a lock) shows up as a call in the code.

A value is dropped when its owner goes out of scope. A type runs custom cleanup by providing `fn drop(&mut self)` (the `Drop` trait), so a file or socket closes at a point the reader can see in the source.

## Reference-level explanation

**Moves.** Assigning, passing or returning a value of a non-`Copy` type moves it. Using a binding after it has been moved on any path is a compile error. A value moved inside a loop is valid on the next iteration only if every path through the body assigns it a new value first. Whether a binding copies depends only on its type. Inside a generic item, a parameter bounded by `Copy` copies and an unbounded one moves.

**Copy.** Raw pointers are also `Copy`. `&mut T` is never `Copy`, because duplicating it would break the aliasing rule. `#[derive(Copy)]` on a user type is rejected when a field is not `Copy`, and the diagnostic names the first such field.

**Borrows.** `&expr` creates a shared borrow and `&mut expr` an exclusive one. While a `&mut` borrow is live, no other borrow of the same place may be used. While any `&` borrow is live, the place may not be mutated or moved. A borrow must not outlive the value it refers to.

**Lifetime inference.** Every borrow has a lifetime that the checker verifies, but the programmer does not write it in the common case. A value built from borrows is inferred to borrow from everything it was built from. A function result that may hold a borrow is inferred to borrow from the function's reference parameters. The compiler rejects:

- returning such a value from the function that owns the borrowed local, directly or wrapped in a struct, variant, tuple or collection;
- storing it into a place that is used after the borrowed value's scope ends, or that lives outside the loop whose body declares the borrowed value;
- storing it through a reference parameter, which always outlives the function's locals.

**Reference counting.** `Rc<T>`, `Arc<T>`, `Cell<T>`, `RefCell<T>`, `Mutex<T>` and `RwLock<T>` are library types in the prelude. The borrow checker needs no special knowledge of them: the counted handle is the owner of the shared value, and dropping the last handle drops the value.

**Drop.** A value that owns something is dropped exactly once: when its binding leaves scope (in reverse declaration order, including on `return`, `break`, `continue` and `?`), when an assignment overwrites the place holding it, or, for a temporary, at the end of the statement that created it. A moved-out binding is not dropped, including when the move happened on only some paths. A type's `drop` method runs before its fields are dropped, and fields drop in declaration order. A panic does not run drops.

## Drawbacks

Lifetime inference cannot cover every borrowing pattern. Where it fails, the programmer gets a diagnostic about a lifetime they never wrote, which can be harder to act on than an error against an explicit annotation.

Ownership still rejects some correct programs, most visibly cyclic and graph-shaped data. Those programs need `Rc`/`Arc` and interior mutability, which add runtime cost and a runtime failure mode (`RefCell` panics on a conflicting borrow).

## Rationale and alternatives

**Garbage collection.** It is the easiest model to program against and handles cycles automatically. Its pauses and memory overhead conflict with the latency and footprint requirements of the programs Paco targets.

**Manual memory management.** It gives full control and no compiler friction, but it leaves use-after-free, double-free and data races to the programmer, which is the class of bug this design exists to rule out.

**Ownership with explicit lifetimes everywhere.** It delivers the same safety and performance, but every signature that involves a borrow must carry lifetime parameters. That is a cost paid in ordinary code for cases the compiler can usually work out on its own.

**Ownership with inferred lifetimes (chosen).** It keeps compile-time safety and deterministic cleanup with no collector, and keeps lifetimes out of ordinary signatures. Reference counting covers the shapes ownership cannot express, and its cost appears in the code as a type and a call.

## Prior art

Rust's ownership and borrow checking is the direct precedent for the move, borrow and aliasing rules. Swift's automatic reference counting and reference counting in general inform the `Rc`/`Arc` escape hatch. RAII in C++ is the precedent for deterministic, scope-based cleanup. Go and Java are examples of the garbage-collected alternative.

## Unresolved questions

How far inference extends before an explicit lifetime is required. The set of programs that need one is expected to shrink as inference covers more borrowing patterns.

## Future possibilities

None.
