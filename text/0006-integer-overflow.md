# RFC 0006: Integer Overflow

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

In the debug profile (`paco build`), integer arithmetic is checked and an overflow panics. In the release profile (`paco build --release`), integer arithmetic wraps in two's complement. Every integer type also has `wrapping_*`, `saturating_*`, `checked_*`, and `overflowing_*` methods, which behave the same in both profiles, so code with a specific intent about overflow can state it.

## Motivation

A language has to say what `i32::MAX + 1` evaluates to. The answer is observable in any numeric program, and a compiler cannot be written against "defined" without a definition.

Two behaviors are defensible for optimized code. Wrapping arithmetic is what the hardware does: `i32::MAX + 1` becomes `i32::MIN`. Saturating arithmetic clamps at the type's bounds: `i32::MAX + 1` stays `i32::MAX`. Both have real uses, and the language has to pick one as the default and make the other easy to ask for.

## Guide-level explanation

In the release profile, arithmetic wraps. `i32::MAX + 1` is `i32::MIN`, with no extra instructions spent checking.

In the debug profile, the compiler checks every arithmetic operation and panics on overflow. An overflow that would wrap silently in a release build surfaces during development, at the point it happens.

When the default is not what the code means, each integer type offers explicit forms:

| Method | Behavior |
|---|---|
| `wrapping_add(b)` | wraps |
| `saturating_add(b)` | clamps at the type's bounds |
| `checked_add(b)` | `Option<T>`, `None` on overflow |
| `overflowing_add(b)` | `(T, bool)`, the wrapped result and an overflow flag |

The same family exists for `sub`, `mul`, `div`, `rem`, `neg`, `shl`, and `shr`:

```paco
fn hash_step(h: u32, x: u32) -> u32 {
    h.wrapping_mul(2654435761).wrapping_add(x)   // wrapping is intended
}

fn accumulate(level: i8, delta: i8) -> i8 {
    level.saturating_add(delta)                  // quantized value: clamp
}

fn buffer_len(count: u64, size: u64) -> Option<u64> {
    let n = count.checked_mul(size)?;            // overflow is an error
    Some(n)
}
```

These methods do not depend on the build profile. An intentional wrap written as `wrapping_add` behaves identically in debug and release.

## Reference-level explanation

Integer arithmetic operators:

- In the debug profile, panic on overflow.
- In the release profile, produce the two's-complement wrapped result.

The explicit methods are defined on every integer type and have the same result in both profiles:

- `wrapping_op(b) -> T`: the two's-complement wrapped result.
- `saturating_op(b) -> T`: the result clamped to the type's minimum or maximum.
- `checked_op(b) -> Option<T>`: `Some(result)`, or `None` if the operation overflows.
- `overflowing_op(b) -> (T, bool)`: the wrapped result, and `true` if it overflowed.

### Why wrapping is the release default

Counter-based pseudorandom number generators need wrapping arithmetic to work. Philox and Threefry, which JAX and PyTorch use for reproducible randomness, are built on integer multiplication and addition that is expected to wrap. Under saturating arithmetic they do not just produce weaker output; they collapse toward a fixed point. The same holds for hash functions and for PCG, xorshift, and linear congruential generators.

Randomness underlies weight initialization, dropout, shuffling, augmentation, sampling, and every reproducible-seed guarantee. A default that breaks the language's own generators is the wrong default.

Wrapping is also free, because it is what the hardware does. Saturating costs a compare and a conditional move on scalar paths.

Saturating arithmetic is the correct operation for quantized inference, where clamping to the representable range is what int8 quantization means, and vector instruction sets provide it natively (`paddsb`, `paddusw`, `SQADD`). That case is served by writing `saturating_add` explicitly, which puts the decision in the code.

### Why the profiles differ

A check on every arithmetic operation is too expensive for release code and too valuable to skip during development. An overflow that is a bug surfaces as a panic in the debug profile. An overflow that is intended is written as a `wrapping_*` call.

## Drawbacks

The two profiles can disagree about a program's behavior. Code that panics under `paco build` can wrap silently under `paco build --release`, and a release build can carry an overflow bug that testing never triggered. The conformance suite runs in both profiles, and a test whose expected result depends on overflow states which profile it asserts against.

In the release profile, an index computation that overflows produces a wrong index rather than a panic. Bounds checking catches most of the resulting out-of-range accesses, but not all.

## Rationale and alternatives

**Saturating by default.** Correct for quantized inference, and an overflow never produces a wildly wrong magnitude. It breaks counter-based PRNGs and hash functions and costs instructions in the hottest loops.

**Checked in every profile.** Panicking on any overflow everywhere gives the strongest correctness story. It puts a branch on every arithmetic operation, which is not acceptable for numeric code, and makes the wrapping that PRNGs require reachable only through method calls.

**Checked in debug, wrapping in release.** Keeps PRNGs and hashes working, costs nothing in the release hot path, and catches accidental overflow during development. The cost is that the profiles can disagree, which the conformance suite mitigates by running in both.

## Prior art

Rust checks for overflow in debug builds and wraps in release builds, and offers the same four method families. Philox, Threefry, PCG, xorshift, and linear congruential generators all depend on wrapping integer arithmetic.

## Unresolved questions

None.

## Future possibilities

None.
