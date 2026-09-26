# RFC 0007: Floating-Point Types

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Alongside `f32` and `f64`, Paco has four reduced-precision floating-point primitives: `f16` and `bf16`, which are full arithmetic types, and `f8e4m3` and `f8e5m2`, which are storage and interchange formats with no arithmetic operators. No floating-point type converts to another implicitly, in either direction. Narrowing rounds to nearest, ties to even, and overflows to infinity. The accumulation width of a reduction belongs to the operation, not to the element type.

## Motivation

Mixed-precision numeric workloads do not run on `f32` and `f64` alone. Training keeps activations and gradients in `bf16` or `f16` and accumulates in `f32`. Inference stores weights and activations in 8-bit formats to fit more of a model in less memory. A program that does this work needs to name these types.

These formats cannot be library types layered over `u16` or `u8`. Their literals and conversions have to be understood by the type checker, and the compiler has to map them to the hardware's conversion and arithmetic instructions. They are primitives.

## Guide-level explanation

| Type | Layout | Arithmetic |
|---|---|---|
| `f16` | IEEE 754 binary16: 1 sign, 5 exponent, 10 mantissa bits | Yes |
| `bf16` | bfloat16: 1 sign, 8 exponent, 7 mantissa bits | Yes |
| `f8e4m3` | OCP FP8: 1 sign, 4 exponent, 3 mantissa bits | No |
| `f8e5m2` | OCP FP8: 1 sign, 5 exponent, 2 mantissa bits | No |

`bf16` keeps `f32`'s exponent range with a shorter mantissa. Converting between `bf16` and `f32` is cheap, and the dynamic range gradients need survives the round trip.

### FP8 is storage

`f8e4m3` and `f8e5m2` have no arithmetic operators. To compute with an FP8 value, widen it:

```paco
let w: f8e4m3 = load_weight();
let acc: f32  = (w as f32) * (x as f32);
```

This reflects the hardware. FP8 values are consumed by matrix-multiply units as an input format and accumulated in a wider type. Scalar FP8 arithmetic would promise behavior no target provides.

### No implicit conversion

Every precision change is a written `as`, including widening:

```paco
let a: bf16 = 1.5;          // the literal takes its type from the annotation
let b: f32  = a;            // ERROR: no implicit widening
let c: f32  = a as f32;     // OK
let d: bf16 = c as bf16;    // OK: narrowing, precision loss is visible
```

An accidental widening inside a hot loop is a hidden performance cost, just as an accidental narrowing is a hidden accuracy loss. Both are written out.

### Accumulation width

A reduction accumulates at the width its signature states, not at the width of its elements. A standard library `sum` over `[]bf16` that returns `f32` is the expected shape, not a special case.

## Reference-level explanation

- `f16` and `bf16` support the arithmetic and comparison operators of `f32` and `f64`.
- `f8e4m3` and `f8e5m2` support equality only. They have no arithmetic or ordering operators.
- A float literal takes the floating-point type required by its context.
- No implicit conversion exists between any two floating-point types. Conversions use `as`.
- Narrowing with `as` rounds to nearest, ties to even. A value outside the target's finite range becomes an infinity. This is defined behavior, not a panic: it matches the hardware, and a panic in an inner loop is not something a numeric kernel can recover from ([RFC 0005](0005-error-handling.md)).
- Where a target provides native instructions for a type, the compiler uses them. Otherwise it emulates the type in software and emits a warning, because the fallback can be slow enough to make efficient-looking code slow.

## Drawbacks

Mixed-precision code reads more heavily than in languages where conversions are implicit. The visible `as` is the point: every precision change can be found by searching the source.

Support for `f16`, `bf16`, and the FP8 formats varies by target. The conformance suite covers these types on every supported target and profile.

There are two classes of floating-point type, arithmetic and storage-only, which is one more distinction to learn.

## Rationale and alternatives

**Only `f32` and `f64`, with library emulation.** Reduced-precision types as library wrappers over `u16`/`u8` need no compiler support, but they lose literal syntax, real type checking, and native instructions.

**Every format a full arithmetic type.** Uniform, but it promises scalar FP8 arithmetic the hardware does not have. The compiler would widen behind the scenes, and code that looks efficient would not be.

**Implicit widening only.** Allowing the "safe" direction implicitly still hides a cost in hot loops, and makes the conversion rules asymmetric.

**Arithmetic for `f16`/`bf16`, storage-only for FP8, all conversions explicit.** The type system matches what the hardware does. A slow scalar FP8 loop cannot be written by accident, because the operators it would need do not exist.

## Prior art

IEEE 754 binary16 and Google Brain's bfloat16 define `f16` and `bf16`; both are native arithmetic types on current accelerators. The Open Compute Project's FP8 specification defines the E4M3 and E5M2 layouts, and current hardware uses them as matrix-unit input formats rather than general arithmetic types. PyTorch and JAX mixed-precision training accumulates in a wider type than the elements being reduced.

## Unresolved questions

The boundary between arithmetic and storage-only types may need to move as hardware changes. Native scalar arithmetic for an 8-bit or smaller format on a supported target would be the trigger to revisit it for that format.

## Future possibilities

A fused FP8 matrix-multiply intrinsic, or a `comptime`-generated kernel ([RFC 0016](0016-comptime.md)) that performs widen-multiply-accumulate as one operation, once a target makes that pattern common enough to name.
