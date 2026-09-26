# RFC 0012: Modules and Visibility

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A directory is a module. Every `.paco` file in it shares one namespace and sees every declaration in the module. Subdirectories are separate modules. Each file begins with `module name;`, then its imports, then its items. Declarations are private by default and exported with `pub`. Imports bind a whole module under a qualified name; there are no selective or wildcard imports.

## Motivation

A library needs a stable public surface. If every declaration were visible outside its module, every internal helper would become part of the contract, and no type could keep its representation private. A tensor type, for example, needs its internal layout out of its public contract so the library can change that layout without breaking callers.

Readers also need to know where a name comes from. When call sites are qualified by module, a fragment of code read out of context says where each callee is defined.

## Guide-level explanation

Every file opens with a module declaration matching its directory name:

```paco
module nn;

use stdlib::math;
use example.com/team/tensor as tensor;

pub const VERSION: string = "0.1";
const EPS: f32 = 1e-6;              // private to module nn

pub struct Linear {
    w: tensor::Tensor<f32>,         // private field
    b: tensor::Tensor<f32>,

    pub fn forward(&self, x: &tensor::Tensor<f32>) -> tensor::Tensor<f32> {
        // ...
    }

    fn init_weights(&mut self) { /* private to module nn */ }
}
```

The `module` line repeats the directory name on purpose. It makes each file self-describing: a reader who opens one file knows which module it belongs to without looking at the filesystem.

A declaration without `pub` is private to its module. All files in the same directory can see it. Files in a subdirectory cannot, because a subdirectory is a different module.

`pub` on a `struct` or `enum` exports the type. Its fields, methods, and associated constants each need their own `pub`. Enum variants are the exception: they are exported with the enum and take no `pub`.

Imports always bind a module, never a name inside it:

```paco
use stdlib::io;                           // referred to as io::
use example.com/team/json;                // referred to as json::
use example.com/team/json as parser;      // referred to as parser::
```

Call sites stay qualified: `io::read_file(path)`, never a bare `read_file`. The prelude is the only exception ([RFC 0013](0013-prelude.md)).

## Reference-level explanation

A file has a fixed shape:

```ebnf
Program   = ModuleDecl { UseDecl } { AttributedItem } ;
ModuleDecl = "module" Identifier ";" ;
UseDecl   = "use" ModulePath [ "as" Identifier ] ";" ;
```

Nothing may precede the module declaration, it appears exactly once, and imports may not be interleaved with items. The module name must match the directory name. Because the shape is fixed, the formatter checks it mechanically.

Visibility rules:

- No marker: private to the module.
- `pub`: exported from the module.
- `pub struct` / `pub enum` export the type only. Each field, method, and associated constant carries its own visibility.
- Enum variants share the enum's visibility. A variant that cannot be seen cannot be matched, which would make exhaustiveness checking impossible outside the module.
- There is one level of visibility. There is no `pub(crate)`, no `internal`, and no friend mechanism.

A `use` binds the last segment of the module path, or the `as` name when one is given. Two modules may export the same name without conflict, since every use is qualified by its module.

A module path that begins with a domain, such as `example.com/team/json`, names an external package. How that path resolves to a directory is described in [RFC 0014](0014-packages.md).

## Drawbacks

Library code writes `pub` often, and qualified call sites are longer than bare names. `module` and `pub` are reserved words.

Renaming a directory renames its module, so every file's `module` line must change with it. Until the formatter corrects the mismatch automatically, a rename touches every file in the directory.

## Rationale and alternatives

Capitalization-based visibility, where an initial capital letter exports a name, needs no keyword. It conflicts with Paco's naming conventions: functions are `snake_case`, so exporting `read_config` would require `Read_config`. The formatter would then either accept a break in uniform style or rewrite the name and change what the program exports. Visibility would also be implied by spelling rather than stated.

Exporting everything, with the module as the only boundary, needs the least specification. It prevents any library from hiding its representation, and adding privacy afterwards would break every program that reached across a module boundary.

Explicit `pub`, private by default, works with any naming convention and is a searchable token.

Selective imports (`use stdlib::io::{Read, Write}`) and wildcard imports shorten call sites. They also let a bare name in a file refer to any of several modules, so a reader cannot tell where a callee comes from without resolving the imports.

## Prior art

Go treats a directory as a package, with no implicit relationship between nested directories, and exports by capitalization. Rust, Swift, and Kotlin mark exported items with a keyword (`pub`, `public`).

## Unresolved questions

None.

## Future possibilities

The formatter detecting a `module` line that mismatches its directory after a rename, and correcting it automatically rather than reporting an error.
