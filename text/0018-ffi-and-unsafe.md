# RFC 0018: Foreign Functions and `unsafe`

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

Paco calls and exports functions across the C ABI. `extern "C"` blocks declare foreign functions, which are implicitly `unsafe fn`. `pub extern "C" fn` exports a Paco function with the C ABI. An `unsafe { }` expression block is the only place where three unchecked operations are legal: dereferencing a raw pointer, calling an `unsafe fn` and calling a foreign function. `*const T` and `*mut T` are raw pointers with no lifetime or aliasing guarantee. `#[repr(C)]` gives a struct the C layout. The C ABI is Paco's only interop boundary.

## Motivation

Much of the software a program needs to reach is exposed through the C ABI: operating system interfaces, system libraries, every BLAS implementation, and accelerator drivers and their libraries (CUDA, cuBLAS, cuDNN, NCCL, ROCm, Metal, oneDNN). Without a way to call C, a Paco program can reach none of them.

Some data has a layout defined outside Paco: a file format read through `mmap`, a struct a C library fills in, a buffer shared with a device. Working with it requires pointers and a layout Paco can match exactly.

Existing C and C++ programs also need to call into Paco. A team should be able to replace one kernel in such a program with a Paco implementation without rewriting the program around it.

All of this involves operations the compiler cannot check. The design has to allow them while keeping them confined to marked, small, searchable regions, so that ordinary Paco code keeps its guarantees.

## Guide-level explanation

A foreign function is declared in an `extern "C"` block with a Paco signature, using raw pointers where C uses pointers:

```paco
module blas;

#[link(name = "blas")]
extern "C" {
    fn cblas_sgemm(
        order: i32, transa: i32, transb: i32,
        m: i32, n: i32, k: i32,
        alpha: f32,
        a: *const f32, lda: i32,
        b: *const f32, ldb: i32,
        beta: f32,
        c: *mut f32, ldc: i32,
    );
}
```

The compiler cannot verify that the symbol exists, that the signature matches, or that the call respects Paco's aliasing rules, so calling it requires `unsafe`. The usual pattern is a safe wrapper around a narrow unsafe core:

```paco
pub fn sgemm(a: &[]f32, b: &[]f32, c: &mut []f32, m: i64, n: i64, k: i64) {
    unsafe {
        cblas_sgemm(
            ROW_MAJOR, NO_TRANS, NO_TRANS,
            m as i32, n as i32, k as i32,
            1.0, a.as_ptr(), k as i32,
            b.as_ptr(), n as i32,
            0.0, c.as_mut_ptr(), n as i32,
        )
    }
}
```

`sgemm` is an ordinary safe function. Its author is responsible for the invariants the compiler cannot check inside the block; its callers are not.

`unsafe { ... }` is an expression and yields its final value, so the unsafe region can be exactly as large as the operation that needs it:

```paco
let x = unsafe { *p };
```

Exporting works in the other direction:

```paco
pub extern "C" fn paco_kernel(data: *mut f32, len: i32) { /* ... */ }
```

A C or C++ program can link against the result and call `paco_kernel` like any C function.

A struct that crosses the boundary is marked `#[repr(C)]`:

```paco
#[repr(C)]
struct Rect {
    x: i32,
    y: i32,
    w: i32,
    h: i32,
}
```

## Reference-level explanation

### Foreign declarations and exports

`extern "C" { fn name(params) -> R; ... }` declares foreign functions. Each is implicitly `unsafe fn`.

`pub extern "C" fn` defines a Paco function with the C calling convention and an unmangled symbol, callable from C.

`#[link(name = "...")]` on an `extern` block names a library to link. It may repeat, and an optional `kind = "static"` or `kind = "dylib"` selects the form; by default the linker chooses. An `extern` block without `#[link]` links no extra library, which is correct for libc and libm symbols. `paco build -L <dir>` adds a library search directory.

### `unsafe`

`unsafe { ... }` is an expression block. Inside it, exactly three additional operations are legal:

1. dereferencing a raw pointer;
2. calling an `unsafe fn`;
3. calling a foreign function.

Nothing else changes. `unsafe` does not disable the borrow checker, does not allow use after move, and does not relax exhaustiveness checking.

### Raw pointers

`*const T` and `*mut T` exist for the FFI boundary. A raw pointer carries no lifetime and no aliasing guarantee, may be null, and is exempt from the borrow checker. It is not an escape hatch for ordinary code; `Rc` and `Arc` remain the answer when ownership is awkward ([RFC 0001](0001-ownership-and-borrowing.md)).

Creating and manipulating a pointer is safe; only reading or writing through it is unsafe. Converting a borrow to a pointer loses information rather than fabricating it, and a pointer is only dangerous at the point where it is used.

Safe operations:

- `&x as *const T`, `&mut x as *mut T`, and `s.as_ptr()` / `s.as_mut_ptr()` on a slice.
- `p.offset(n)` and `p.add(n)`, scaled by the element size and wrapping.
- `ptr_null<T>()`, `ptr_null_mut<T>()` and `p.is_null()`.
- Casts between pointer types (`p as *const u8`, `*mut T` to `*const T`) and between pointers and `u64`. `*const T` to `*mut T` is allowed only as an explicit `as` cast.

Unsafe operations: `*p`, `p.read()` and `p.write(v)`, for scalar and aggregate `T`.

### Function pointers

`extern "C" fn(A1, ..., An) -> R` (optionally `unsafe extern "C" fn(...)`) is the type of a C function pointer: thin and non-capturing. An `extern "C" fn` item coerces to it when the signatures match; a function declared in an `extern` block coerces to the `unsafe extern "C" fn` form. Such values can be stored, passed to and returned from foreign functions, and called from Paco inside `unsafe`.

Closures do not coerce, because they carry an environment. State passes to C as a `*mut u8` user-data argument.

A panic inside a Paco function called from C aborts the process with the panic message and exit status 101. It never unwinds into C frames.

### Layout

Without an attribute, Paco makes no promise about a struct's field order or padding and may reorder fields for packing. `#[repr(C)]` gives a struct exactly the target's C layout: size, alignment, field offsets and padding.

A `#[repr(C)]` struct may contain numeric primitives, `bool`, raw pointers, C function pointers, fixed-size arrays `[T; N]` and other `#[repr(C)]` structs. It may appear in `extern "C"` signatures by value and behind a pointer, following the platform C calling convention. A struct without `#[repr(C)]`, or a type with no C equivalent (`string`, `Vec`, `Rc`, enums with data), in an `extern` signature is a compile error.

`size_of<T>()` and `align_of<T>()` return the size and alignment of any sized type, for allocation and copy sizes passed to C.

### Interop boundary

The C ABI is Paco's only interop boundary. Paco does not interoperate with Python or any other language runtime. Libraries above that boundary, such as tensors, automatic differentiation, layers, optimizers and data loading, are written in Paco.

### Safety claim

With FFI, a Paco program can crash with a memory error. The language's claim is "memory safe outside `unsafe`": code that contains no `unsafe` block, and calls only safe functions, cannot violate memory safety. Searching for `unsafe` bounds the code that can.

### Blocking

A foreign call blocks the OS thread that makes it, and the task scheduler ([RFC 0017](0017-tasks-and-channels.md)) cannot suspend it. [RFC 0019](0019-blocking-calls.md) defines how such calls run.

## Drawbacks

Paco programs can segfault. "Memory safe outside `unsafe`" is a weaker claim than "memory safe" and is documented as such.

The feature adds surface area: the `extern` and `unsafe` keywords, two pointer types, a function pointer type, and layout and link attributes.

Having no interop boundary other than C means libraries that exist only in another runtime's ecosystem must be reimplemented in Paco or reached through a C API.

## Rationale and alternatives

**No FFI.** Keeps an unconditional safety claim and a smaller compiler, but rules out system libraries, BLAS, accelerators and externally defined data layouts.

**`extern` without `unsafe`.** Declaring a foreign signature would itself count as the assertion of correctness, and calls would look like any other call. This removes ceremony at the call site but hides the one place memory safety can break behind ordinary-looking syntax. Nothing in the source would mark the unsafe surface, so it could not be found or bounded by inspection.

**`extern`, `unsafe` and raw pointers (chosen).** The unsafe surface is explicit, local and searchable. The ceremony at the boundary is what keeps unsafe regions small and pushes code toward safe wrappers.

**Interop with other language runtimes** (for example Python) would give direct access to their libraries, at the cost of a second runtime, memory model and threading model inside Paco programs. Paco keeps the C ABI as the single boundary and builds the libraries above it in Paco.

## Prior art

Rust's `unsafe` blocks, `extern` blocks, raw pointers, `#[repr(C)]` and `#[link]` have the same shape. Julia reaches hardware through the C ABI (BLAS, CUDA) and writes its numerical and framework layers in Julia itself, the same boundary Paco draws. Go's cgo and Zig's C interop are other examples of a language treating C as its interop layer.

## Unresolved questions

- Variadic foreign functions, in the style of `printf`.
- Callbacks from C into Paco beyond the panic rule: which task context a callback runs in, especially when C invokes it on a thread the Paco runtime does not own.

## Future possibilities

Compiling Paco itself to GPU device code (PTX, SPIR-V), rather than only calling existing kernels through the C ABI, would let kernels be written in Paco. It needs a second code generation target, an address-space-aware memory model and a device launch mechanism, and would be the subject of its own RFC.
