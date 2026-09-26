# RFC 0004: Traits

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A trait names a set of functions, constants and associated types. A type satisfies a trait implicitly, by having members with matching signatures; there is no `implements` declaration. Generic code states what it needs with bounds (`T: Ord + Copy`, or a `where` clause), is type-checked once against those bounds, and is checked again at each instantiation, with an error at the call site when a concrete type does not satisfy them. Operators and indexing resolve to a fixed set of operator traits, so a user type gets `+` or `m[i, j]` by providing the matching method. Calls through bounds are dispatched statically.

## Motivation

Reusable code needs to say what it requires of a type: a `max` needs ordering, a hash map needs hashing and equality, a numeric kernel needs arithmetic. Without a way to state and check those requirements, generic code either cannot call anything on its type parameters or fails only after substitution, with errors deep inside library code.

User types also need to take part in the language's own syntax. A vector type should support `+`, a matrix should support `m[i, j]`, and an iterator should work in a `for` loop, all checked before the program runs and without runtime dispatch.

Finally, requiring every type to declare each trait it satisfies couples the type's author to every trait that might describe it. A type defined in one library cannot satisfy a trait defined later in another without one of them changing.

## Guide-level explanation

### Declaring a trait

A trait lists required methods, which end in `;`, and may give default bodies:

```paco
trait Shape {
    fn area(&self) -> i64;
    fn is_empty(&self) -> bool { self.area() == 0 }
}
```

### Satisfying a trait

A type satisfies `Shape` by having an `area` method with the same signature. Nothing else is written:

```paco
struct Square {
    side: i64,

    fn area(&self) -> i64 { self.side * self.side }
}
```

`Square` does not define `is_empty`, so it gets the trait's default.

### Bounds

A generic parameter lists the traits it requires. Inside the body, only what those traits provide may be used:

```paco
fn total<T: Shape>(x: &T) -> i64 { x.area() }

fn largest<T>(a: T, b: T) -> T where T: Ord + Copy {
    if a > b { a } else { b }
}
```

`T: A + B` and `where T: A + B` mean the same thing. Bounds may appear on functions, structs, enums, traits and `methods` blocks.

Calling a method no bound provides is an error at the definition:

```paco
fn f<T: Shape>(x: &T) -> i64 { x.perimeter() }
// ERROR: `perimeter` is not provided by the bounds of `T` (`Shape`)
```

Using a generic item with a type that does not satisfy its bounds is an error at the use site:

```paco
struct Circle { r: i64 }

total(&Circle { r: 1 });
// ERROR: `Circle` does not satisfy `Shape`: missing method `area`
```

If `Circle` had `fn area(&self) -> f64`, the error would report a signature mismatch and show both signatures.

### Associated types

A trait can leave a type for the implementing type to choose:

```paco
trait Index<Idx> {
    type Output;
    fn index(&self, i: Idx) -> &Self::Output;
}
```

The implementing type supplies it with a `type` member in its own block (or in a `methods` block):

```paco
pub struct Matrix {
    data: []f32,
    cols: i64,

    type Output = f32;

    pub fn index(&self, i: (i64, i64)) -> &f32 {
        &self.data[i.0 * self.cols + i.1]
    }
}
```

When exactly one trait method mentions the associated type, the compiler can read it from that method's signature, and the `type` member may be left out. An iterator whose `next` returns `Option<i64>` has `Item = i64`:

```paco
struct Countdown {
    n: i64,

    fn next(&mut self) -> Option<i64> {
        if self.n == 0 { None } else { self.n = self.n - 1; Some(self.n) }
    }
}
```

Generic code refers to the chosen type through the parameter, as `I::Item`.

### Operators

Operators are calls to methods of operator traits in the prelude ([RFC 0013](0013-prelude.md)):

| Syntax | Trait | Method |
|---|---|---|
| `a + b`, `a - b`, `a * b`, `a / b`, `a % b` | `Add`, `Sub`, `Mul`, `Div`, `Rem` | `fn add(&self, other: Self) -> Self`, ... |
| `-a` | `Neg` | `fn neg(&self) -> Self` |
| `a == b`, `a != b` | `Eq` | `fn eq(&self, other: Self) -> bool` |
| `a < b`, `a <= b`, `a > b`, `a >= b` | `Ord` | `fn cmp(&self, other: Self) -> i64` |
| `a[i]`, `a[i, j]`, ... | `Index<Idx>` | `fn index(&self, i: Idx) -> &Self::Output` |
| `for x in it` | `Iter` | `fn next(&mut self) -> Option<Self::Item>` |
| `f(...)` on a value | `Call` | `call` |

A type with an `add` method of the right shape supports `+`:

```paco
struct Vec2 {
    x: f64,
    y: f64,

    fn add(&self, other: Self) -> Self { Vec2 { x: self.x + other.x, y: self.y + other.y } }
}

let v = v1 + v2;   // Vec2::add(&v1, v2)
```

A bound on an operator trait enables the operator in generic code: `fn sum<T: Add + Copy>(a: T, b: T) -> T { a + b }` works for `i64` and for `Vec2`.

Indexing takes one subscript or several. `a[i]` resolves through `Index<i64>`, `m[i, j]` through `Index<(i64, i64)>`, and `t[i, j, k]` through `Index<(i64, i64, i64)>`.

The primitive types satisfy the relevant traits natively: `1 + 2` and `a == b` work because the compiler knows the primitives' behavior, not because a library defines methods on them.

## Reference-level explanation

### Trait declarations

```ebnf
TraitDecl        = "trait" Identifier [ GenericParams ] [ WhereClause ] "{" { TraitMember } "}" ;
TraitMember      = { OuterAttribute } ( TraitFunctionDecl | ConstDecl | AssocTypeDecl ) ;
AssocTypeDecl    = "type" Identifier [ ":" TraitBoundList ] [ "=" Type ] ";" ;
```

A trait function ending in `;` is required. One ending in a block is a default method. Member signatures are resolved with `Self`, the trait's generic parameters and projections such as `Self::Output` in scope; an unknown type in a member signature is reported at that type.

A struct, enum or `methods` block binds an associated type with `type Name = Type;`.

### Satisfaction

A concrete type `C` satisfies trait `Tr<Args>` when, after substituting `Self` with `C`, the trait's parameters with `Args`, and each associated type with its binding:

- `C` has a method or associated function for every required member of `Tr`, with the same name, the same receiver kind (`&self`, `&mut self` or `self`, exactly), the same parameter types and the same return type;
- every associated type of `Tr` is bound, either by a `type` member or by inference.

Methods come from `C`'s own block and from `methods` blocks for `C` in scope ([RFC 0003](0003-methods.md)). An associated type is inferred when it has no `type` member and exactly one trait method mentions it; if inference is ambiguous, the compiler asks for an explicit `type` member. Each distinct set of trait arguments is a distinct trait: satisfying `Index<i64>` says nothing about `Index<(i64, i64)>`.

No declaration links a type to a trait, so there are no overlapping implementations and no rules about which module may declare one.

Traits with no members, such as `Copy` and `Numeric`, would be satisfied by every type under the structural rule, so the compiler decides them: `Copy` by the rules of [RFC 0001](0001-ownership-and-borrowing.md), and `Numeric` natively for the integer and floating-point primitives only.

### Native satisfaction by primitives

- Integer and float primitives satisfy `Add`, `Sub`, `Mul`, `Div`, `Rem`, `Eq`, `Ord`, `Clone`, `Copy`, `Display` and `Numeric`. Signed integers and floats also satisfy `Neg`.
- `bool` and `char` satisfy `Eq`, `Ord`, `Clone`, `Copy` and `Display`.
- `string` satisfies `Eq`, `Ord`, `Clone` and `Display`.
- Every integer type, `bool`, `char` and `string` satisfy `Hash`. Floats do not.

### Bounds

```ebnf
GenericParam   = Identifier [ ":" TraitBoundList ] | ... ;
WhereClause    = "where" WhereBound { "," WhereBound } [ "," ] ;
WhereBound     = ( Lifetime | Type ) ":" TraitBoundList ;
TraitBoundList = TraitBound { "+" TraitBound } ;
```

A bound naming something that is not a trait is an error. A `?Trait` bound is rejected as not supported.

Inside a generic item, a method call, associated function call, operator or associated type projection (`T::Item`) on a bounded parameter resolves against the signatures of its bounding traits, including their default methods. Anything not provided by a bound is an error at that expression, naming the parameter and its bounds. The body is checked once, at its definition.

At each use of a generic item with concrete type arguments, each argument is checked against its bounds by the satisfaction rule. A failure is reported at the use site and names the type, the trait, and the missing member or the mismatched signatures.

### Default methods

A default method is available on every satisfying type that does not define a method of the same name. It is instantiated for each concrete type that uses it, with `Self` bound to that type. A type that defines the method uses its own.

### Dispatch

Generic items are instantiated per set of concrete type arguments. A call through a bound compiles to a direct call to the concrete type's method in each instance, with no table or runtime lookup.

### Operator resolution

A binary operator `a op b` is the call `a.method(b)` of the corresponding trait. The operands follow the method's parameters: the left operand is borrowed for a `&self` receiver; the right operand is moved when the parameter is `Self` by value and the type is not `Copy`, and borrowed when it is `&Self`. `a < b` and the other ordering operators evaluate `a.cmp(b)` against `0`. `!=` is the negation of `eq`. A unary `-a` is `a.neg()`.

An index expression with one subscript `a[e]` resolves through `Index<E>`, where `E` is the type of `e`. With several subscripts `a[e1, ..., en]` it resolves through `Index<(E1, ..., En)>`, with the subscripts passed as a tuple. The type of the index expression is the `Output` of that trait for the indexed type.

A `for` loop over a value calls its `Iter::next` until it returns `None`.

Operators bind to the prelude's operator traits, not to whatever those names resolve to in the current module. Shadowing `Add` in a module does not change what `+` means there ([RFC 0013](0013-prelude.md)).

The bitwise operators `&`, `|`, `^`, `<<`, `>>` and prefix `~` apply to integer types only and are not overloadable.

## Drawbacks

A type can satisfy a trait by coincidence, when it happens to have a method with the same name and signature. Matching whole signatures makes this rarer than matching names, but it cannot rule it out.

Because no declaration ties a type to a trait, renaming or changing a method can silently stop a type from satisfying a trait. The error appears where the type is used with a bounded generic, which may be far from the change.

Checking satisfaction at every instantiation, and generating code per instantiation, adds compile time and binary size in heavily generic programs.

The type of an index expression depends on trait resolution, so the checker must resolve `Index` before it can report a precise error for a malformed index.

## Rationale and alternatives

**Explicit declarations (`impl Trait for Type`, `implements`).** They document intent and prevent accidental satisfaction. They also require the type's author or the trait's author to write the link, and bring rules about where such declarations may live and how overlapping ones are resolved. Structural satisfaction lets independent libraries meet without either changing, and has no overlap to resolve.

**Matching by method name only.** Simpler, but a same-named method with different types would satisfy the trait and fail later. Matching the full signature, including the receiver kind, keeps satisfaction precise.

**Checking generic bodies only after substitution.** This needs no bounds at all, but errors appear inside library code for each bad instantiation. Checking the body against its bounds once gives errors at the definition, and checking instantiations gives errors at the call site.

**Runtime dispatch for operators and reflection.** Resolving `+` or indexing through runtime tables is flexible, but it moves type errors to runtime and adds a lookup to every operation. Operator traits resolve at compile time to a direct call.

**Tuple or method syntax for multidimensional indexing.** `a[(i, j)]` needs no new grammar but doubles the brackets in the code that uses indexing most. `a.at(i, j)` also needs no grammar change but is not how indexing is written in numeric code. `a[i, j]` needs one small grammar change and reads like the mathematics.

**Traits without associated types.** Without them, `Index`, `Iter` and similar traits cannot state their output type. Associated types let the implementing type choose it once, and let generic code name it as `T::Item`.

## Prior art

Go interfaces are satisfied structurally, without a declaration, and checked statically. Rust traits provide bounds, `where` clauses, default methods and associated types, with explicit `impl` declarations. Swift protocols have associated types. C++ templates check after substitution; C++ concepts add checked requirements. Haskell type classes are the origin of bounded polymorphism with explicit instances. NumPy, Julia and Fortran write multidimensional indexing as `a[i, j]`.

## Unresolved questions

- How constant members of a trait take part in satisfaction.

## Future possibilities

- Trait objects (`dyn Trait`) for dynamic dispatch, with the cost of a table lookup visible in the type.
