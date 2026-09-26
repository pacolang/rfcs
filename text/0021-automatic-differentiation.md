# RFC 0021: Automatic Differentiation

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Paco computes gradients in reverse mode by transforming the typed,
borrow-checked program at compile time. For each differentiated function the
compiler generates an augmented primal and a pullback. Gradients behave the
same under `paco run`, `paco build` and `paco build --release`. The boundary
between compiler and libraries is the `stdlib::autodiff::Differentiable` trait:
the compiler knows the derivatives of float primitives and intrinsics, and
every other type, including tensors, becomes differentiable by implementing
the trait and registering derivatives for its own operations.

## Motivation

Numeric programs such as model training, optimization and physics simulation
need gradients of ordinary code. Writing them by hand is slow and error-prone.

Gradients should work wherever the program runs. If they are available only in
some build profiles, a program cannot be tested in development the way it runs
in production.

When a function cannot be differentiated, the error should name the Paco
expression responsible and be reported before the program runs.

The compiler should not have to know about tensors or any other domain type
for any of this to work.

## Guide-level explanation

`stdlib::autodiff::grad(f, inputs)` returns the value of `f` at `inputs` together
with the gradient of each input:

```paco
use stdlib::autodiff;

fn loss(w: f64, x: f64) -> f64 {
    let y = w * x - 1.0;
    y * y
}

let (value, (dw, dx)) = autodiff::grad(loss, (2.0, 3.0));
```

The float types satisfy `Differentiable` natively, with `Tangent = Self`. Any
other type satisfies it by providing the trait's items:

```paco
pub trait Differentiable {
    type Tangent;
    fn zero_tangent(&self) -> Self::Tangent;
    fn move_by(&mut self, offset: &Self::Tangent);
}

struct Vec2 {
    x: f64,
    y: f64,
    type Tangent = Vec2;

    fn zero_tangent(&self) -> Vec2 { Vec2 { x: 0.0, y: 0.0 } }
    fn move_by(&mut self, offset: &Vec2) { self.x = self.x + offset.x; self.y = self.y + offset.y }
}
```

Each gradient `grad` returns has its input's `Tangent` type.

A function can supply its own derivative, which is used instead of
differentiating its body:

```paco
fn norm(v: &Vec2) -> f64 {
    (v.x * v.x + v.y * v.y).sqrt()
}

struct NormPullback {
    v: Vec2,
    n: f64,
    type Seed = f64;
    type Gradients = Vec2;

    fn pullback(self, seed: f64) -> Vec2 {
        Vec2 { x: seed * self.v.x / self.n, y: seed * self.v.y / self.n }
    }
}

#[derivative(of = norm)]
fn norm_derivative(v: &Vec2) -> (f64, NormPullback) {
    let n = norm(v);
    (n, NormPullback { v: Vec2 { x: v.x, y: v.y }, n: n })
}
```

The derivative takes the original function's parameters and returns its
result together with a value satisfying `stdlib::autodiff::Pullback`, which maps
the gradient of the result to the gradients of the parameters. It can be
declared in any module, including a fetched library. This is how a tensor
library registers the derivatives of its own operations, such as matrix
multiplication and convolution.

## Reference-level explanation

### Transformation

After type and borrow checking, the compiler transforms the differentiated
function and everything it calls into two functions:

- The **augmented primal** performs the original computation and records what
  the backward pass needs: which branches were taken, and the value of
  anything the computation later overwrites.
- The **pullback** runs the recorded computation in reverse and computes the
  vector-Jacobian product.

Both are ordinary compiled code, so they run the same way in every build
profile and under `paco run`. A pullback can itself be differentiated, which
gives second derivatives.

The transformation operates on the compiler's typed intermediate
representation (MIR), after borrow checking and before code generation.

### Mutation

Every write in the checked program is an explicit assignment to a named place,
and the borrow checker has already proved that a `&mut` place is not aliased
while it is written. The transformation uses these guarantees to decide which
overwritten values the pullback reads, and saves only those, with no alias
analysis of its own.

### Compiler and library boundary

`stdlib::autodiff` declares `Differentiable` (with an associated `Tangent`,
`zero_tangent` and `move_by`), `Pullback`, and `grad`.

The compiler supplies derivatives only for float primitives and intrinsics:
`+`, `*`, `sqrt`, `exp`, `tanh` and the rest of the float math operations. It
never names a tensor or any other domain type. A library type is
differentiable because it implements `Differentiable`, and a library operation
has a derivative because the library registers one with
`#[derivative(of = f)]`.

A construct the transformation cannot differentiate, such as a call to an
`extern` function with no registered derivative, is a compile-time error that
names the expression and the call chain that reached it.

### Shapes

Gradients preserve the shapes of the primal values they are computed from,
including the symbolic dimension names and witnesses of
[RFC 0020](0020-shape-generics.md). A gradient of a
`Grid<f32, batch, 784>` has the same `batch`. Dimensions are never
differentiable quantities.

## Drawbacks

Derivatives are generated before the program is optimized. For scalar-heavy
code with long loops, the recorded state can be larger and the backward pass
slower than with differentiation of already-optimized code. The code
generator's optimizations recover part of the difference. Tensor workloads are
dominated by kernels whose derivatives are written by hand, so the gap matters
mostly for scalar code.

Only reverse mode is provided. Second derivatives come from differentiating a
pullback, not from a forward mode.

The derivative rules for every float primitive and intrinsic are maintained by
the compiler.

## Rationale and alternatives

**A runtime tape.** Operations record themselves on a tape as the program
runs, and the tape is replayed backwards. It is simple to build and familiar.
But it puts allocation and dynamic dispatch on every operation, a cost the
program does not show, and it cannot reject non-differentiable code at compile
time.

**Differentiating optimized low-level IR.** Differentiating LLVM IR after
optimization, as Enzyme does, produces faster scalar derivatives because it
analyzes the optimized program to minimize saved state. But it ties gradients
to one code generator, and when differentiation fails it reports the error in
terms of low-level IR, after inlining has removed the Paco construct
responsible.

**A source-level transform with implicit mutation.** Transforming a
representation where mutation is implicit and aliasing unknown, as Zygote
does, makes mutation hard to differentiate correctly. Paco's transform runs on
a representation where every write is explicit and aliasing is already ruled
out by the borrow checker.

**Structural differentiability.** Treating any struct with float fields as
differentiable needs no trait implementation. But it would accept structs that
are not vectors, reject vectors that do not store floats directly, and tie the
compiler to how a library spells its tensor type.

A compile-time transformation of the checked program works in every build
profile, reports errors in Paco's own terms, and handles mutation with
guarantees the borrow checker already provides.

## Prior art

Swift's differentiable programming transforms SIL, a typed, ownership-checked
representation between type checking and code generation. It defines the
compiler and library boundary through a `Differentiable` protocol with an
associated tangent type, and the compiler ships no tensor type.

Enzyme differentiates LLVM IR after optimization.

Zygote.jl differentiates Julia's intermediate representation and documents
difficulty with mutation.

PyTorch uses a runtime tape. PyTorch and JAX both write derivatives of tensor
kernels by hand rather than differentiating the kernels' loops.

## Unresolved questions

None.

## Future possibilities

A forward mode (Jacobian-vector products) could be added alongside reverse
mode.

The scalar performance gap could be narrowed by optimizing a function before
generating its pullback.
