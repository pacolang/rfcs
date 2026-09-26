# RFC 0020: Shape Generics

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A generic parameter can be a value instead of a type. A dimension in a type is
one of three kinds: static (`const N: int`, known at compile time), symbolic
(`dim B`, a local witness, or an existential `?n`, known at run time but named
in the type), or anonymous (`Dyn`, known only at run time). The type checker
compares static and symbolic dimensions and rejects operations whose
dimensions it cannot prove equal. A run-time extent is checked once, where
data enters the program, with `with_dims()?`; every later operation on that
value needs no `Result` and no run-time check.

## Motivation

A shape mismatch is one of the most common errors in numeric code, and in most
numeric tooling it shows up only when the program runs, often long after the
mistake was made. Catching it at compile time needs dimensions in the type.

Real programs have dimensions nobody knows at compile time. Batch size changes
between training steps and deployments; sequence length changes with every
input. A design that handles only static shapes covers a minority of real code.

A design that treats every run-time dimension as unknown has the opposite
problem. If every `a + b` on two run-time extents returns a `Result`, the
program repeats a check it already made when the batch was loaded, and needs a
`?` on every operation. What the programmer needs is a way to say "this extent
is not known at compile time, but these two values share it", once, and have
the type system rely on it afterwards.

## Guide-level explanation

### Const generic parameters

A generic parameter declared `const N: int` is a compile-time integer. It
extends the `const` items of [RFC 0008](0008-constants.md) to generic
positions. Const arguments are inferred at the call site from the argument
types, like type parameters:

```paco
fn widen<const N: int>(v: &Grid<f32, N>) -> Grid<f32, N + 1>;

let a: Grid<f32, 3> = Grid::zeros();
let b = widen(&a);             // N = 3, b: Grid<f32, 4>
let c = widen<3>(&a);          // explicit argument
```

`Grid` and `Tensor` in this RFC are library types. The mechanism is not tied
to any type: any struct with `const` or `dim` parameters is shape-checked the
same way, whether it is a buffer, an image, an audio frame or a tensor.

### Three kinds of dimension

| Kind | Written | Known | In the instance key | Compared |
|------|---------|-------|---------------------|----------|
| Static | `const N: int`, a literal | compile time | yes | by the type checker |
| Symbolic | `dim B`, a local witness, `?B` | run time; identity in the type | no | by the type checker, on names |
| Anonymous | `Dyn` | run time; no identity | no | only by explicit code |

Static dimensions catch the mistakes that happen most often: a hidden size of
512 where 768 was expected is a typo or a stale configuration value, and it is
rejected before the program runs.

```paco
fn forward<dim B>(x: &Tensor<f32, B, 768>, w: &Tensor<f32, 768, 3072>) -> Tensor<f32, B, 3072>;

let bad: Tensor<f32, Dyn, 512> = load();
forward(&bad, &w);             // error: 512 and 768 differ
```

A symbolic dimension `dim B` is a generic parameter whose value is known only
at run time. Two values that share the name `B` are known to have the same
extent. In `forward`, the batch size is dynamic and the feature sizes are
static, which is the shape of most real models.

```paco
fn matmul<dim M, dim K, dim N>(a: &Grid<f32, M, K>, b: &Grid<f32, K, N>) -> Grid<f32, M, N>;
```

### Local witnesses and the boundary check

Inside a function body, a symbolic dimension is introduced by reading it off a
value:

```paco
fn step(x: Grid<f32, Dyn, 784>, y: Grid<i64, Dyn>) -> Result<f32, DimError> {
    let batch = x.dim(0);                               // an i64 and a rigid name
    let x: Grid<f32, batch, 784> = x.with_dims()?;      // no comparison needed
    let y: Grid<i64, batch> = y.with_dims()?;           // compared once, here
    let h = x + &x;                                     // no Result, no check
    Ok(loss(&h, &y))
}
```

`x.dim(0)` on an immutable binding returns the extent as an `i64` and binds a
rigid name to it. `with_dims()` moves the value into the annotated type
without copying, comparing each run-time extent with its target once and
returning an error on a mismatch. The extent the witness was read from needs
no comparison.

After this boundary, an operation whose dimensions are provably equal returns
its result directly. An operation that cannot prove the equality it needs does
not compile. The diagnostic offers two fixes: name the extent with
`with_dims`, or call the `checked_*` form of the operation, which returns a
`Result` for code that wants to handle a mismatch as a value.

A use of `Dyn` that needs no equality compiles. A `matmul` whose batch is
`Dyn` and whose inner dimension is static needs nothing proved about the
batch.

### Existentials

A type that mentions a witness outside the scope that introduced it says so
with `?`:

```paco
fn nonzero(v: Grid<f32, Dyn>) -> Grid<f32, ?n>;

struct Batch {
    x: Grid<f32, ?b, 784>,
    y: Grid<i64, ?b>,
}
```

Both fields of `Batch` share `?b`, so constructing a `Batch` requires their
extents to be proved equal. A caller opens an existential with `let` and
receives a fresh name, scoped to that binding.

## Reference-level explanation

### Syntax

A generic parameter list accepts `const Identifier ":" Type` and
`dim Identifier` alongside type and lifetime parameters. Inside the item, a
dimension parameter reads as an `i64` value. A dimension position in a type
accepts an integer literal, a `const` or `dim` parameter, a dimension name, an
existential `?name`, `Dyn`, or an arithmetic expression over these.

### Equality of dimensions

Dimension expressions are compared by normalizing them into a canonical
polynomial over the integers. Under this normal form `B + B` equals `2 * B`,
`(N + 1) * 2` equals `2 * N + 2`, and `28 * 28` equals `784`.

Division and remainder are opaque atoms: `N / 2` equals only `N / 2`.
Divisibility, bounds and inequalities are never decided by the type checker. A
program that needs one of those properties checks it in code.

A coefficient that would overflow during normalization makes its term opaque
rather than risking wrong arithmetic in the type checker.

Two dimensions that are provably different are a type error naming both. Two
dimensions that must be equal but cannot be proved equal are a separate
compile-time error, never a hidden run-time check.

`Dyn` never unifies with anything: not with a constant, not with a name, and
not with another `Dyn`. Whether two anonymous extents match is a run-time
question, and only explicit code can answer it.

### Inference

A const or `dim` parameter is inferred from the argument types only from a
position where it appears alone, or from an expression `N + c` with a literal
`c`. Any other position requires an explicit argument.

### Compilation

Each distinct set of const arguments produces a separate instantiation of a
generic item. The compiler enforces a configurable cap on the number of
instantiations per item. An item that exceeds it is rejected with a diagnostic
listing the constants that caused it.

Symbolic dimensions are erased after type checking. A `dim` parameter is
passed as a hidden `i64` argument. It never creates a separate instantiation
and never counts toward the cap. A symbolic position in a type's dimensions is
carried the same way, so the type's own methods can read its value.

### Soundness rules

- The extent of a value whose type names a symbolic dimension cannot change
  while the name applies. An operation that changes an extent consumes the
  value and returns a fresh existential, and a borrowed view prevents the
  change while the borrow lives.
- Extents are never negative. `with_dims` rejects a negative extent.
- Arithmetic on dimensions is checked for overflow in every build profile,
  including `paco build --release`.
- Broadcasting is explicit. Each source axis must equal its target or be the
  literal `1`. A run-time extent is never treated as `1` implicitly.
- A named dimension is never weakened to `Dyn` implicitly, nor carried out of
  the scope that bound it except through an existential. `erase_dims()` is the
  only way to forget a name.

## Drawbacks

There are three kinds of dimension to learn, each with its own rules.

`Dyn` not unifying with itself surprises users who expect run-time shapes to
be compared structurally.

Every value entering a program needs a `with_dims` call before it can take part
in checked operations.

Monomorphization over constants multiplies instantiations. The cap bounds
this, but does not remove the cost.

The normal form cannot prove divisibility or bounds. Code that relies on them
needs explicit checks.

## Rationale and alternatives

**Static dimensions only, with a separate unchecked type for dynamic data.**
This is the simplest to specify and gives every checked value a fully known
shape. But batch size is dynamic in almost every model, so most real code
would live in the unchecked type, and the library would need two parallel
APIs.

**Every dimension dynamic, refined by flow-sensitive analysis.** This needs no
annotations, but a signature does not say whether anything about its shapes
was checked. Whether an operation is checked would depend on control flow the
signature does not show. Flow-sensitive shape inference across function
boundaries is also an open research problem.

**Only static and `Dyn`, with a `Result` per operation.** Here any operation
that needs two `Dyn` extents to agree checks them at run time and returns a
`Result`. It adds nothing to learn beyond `Dyn`, but every operation on
dynamic data needs its own `?`, and the check repeats a fact the program
already established at the boundary.

**Structural equality only.** Comparing dimension expressions after constant
folding and reordering of `+` and `*` needs no algebra in the type checker.
But it rejects ordinary code: `B + B` differs from `2 * B`, and `(N + 1) * 2`
differs from `2 * N + 2`. Polynomial normalization is linear in the size of
the expressions and needs no solver, so the stronger rule costs little.

**Full dependent types or a general constraint solver.** These could prove
divisibility, bounds and inequalities. General constraint solving is
undecidable or slow in the worst case, and its error messages become
unpredictable exactly when a program most needs a clear one.

Named symbolic dimensions prove equality from names carried in the type, so
the check happens once and is visible in the signature, while keeping
dimension equality decidable and fast.

## Prior art

PyTorch and JAX report shape mismatches at run time. Rust and C++ have const
generics over integer values, without run-time named dimensions. Dependent
type systems such as Idris and Agda can express shapes in full generality at
the cost of proof obligations on the programmer.

## Unresolved questions

None.

## Future possibilities

The normal form could grow beyond polynomials over the integers, for example
to decide divisibility or bounds, if programs show a need for it.
