# RFC 0014: Packages

| | |
|---|---|
| **Status** | Accepted |
| **Authors** | Paco Core Team |
| **Created** | 2026-09-26 |

## Summary

A Paco package is a module in its own version-control repository. A program declares its dependencies in a `paco.mod` manifest, each one identified by its repository URL and pinned to a version tag. There is no central registry. `paco get` fetches dependencies and records the exact commit of each in `paco.lock`; `paco mod tidy` reports dependencies that are declared but unused, or used but undeclared. A module's `paco.mod` may declare the range of core compiler versions it supports. A program links only the modules it imports and builds to a single binary.

## Motivation

Every program beyond a single file needs code written by someone else. A package system has to answer three questions: how a dependency is named, how a specific version of it is chosen, and how it is fetched.

A central registry answers the naming question with a namespace it owns. That namespace needs hosting, a governance policy, and an answer to name squatting. Version-control hosts already provide globally unique names (a domain plus a path) and immutable-by-convention version markers (tags). Building on them gives every package a name without any new infrastructure, and lets anyone publish by pushing a tag.

A build also has to be reproducible. The same manifest must produce the same dependency code on every machine, even if a tag is later moved.

## Guide-level explanation

A dependency is imported by its URL-shaped module path. The last path segment is the name the module is bound to:

```paco
module main;

use example.com/team/json;             // referred to as json::
use example.com/team/json as parser;   // alias

fn main() {
    let value = json::parse("{\"a\": 1}");
}
```

The standard library is imported with `::` paths (`use stdlib::io;`); an external module always has a domain as its first segment and `/` between segments. As with any import, the whole module is bound and every call site stays qualified ([RFC 0012](0012-modules-and-visibility.md)).

The project's `paco.mod`, a TOML file at its root, names the module and lists its dependencies, each pinned to a tag:

```toml
module = "example.com/me/myprogram"

[dependencies]
"example.com/team/json" = "v1.2.0"
```

`paco get` reads `paco.mod`, fetches each dependency at its tag into a local package cache, and writes `paco.lock` recording the exact commit each tag resolved to. `paco build` and `paco run` then resolve every `use example.com/...` against those fetched sources. `paco mod tidy` compares the manifest against the `use` declarations in the project's source files and reports each dependency that is never used and each external `use` with no matching entry.

A dependency can also point at a local directory, for working on two modules side by side:

```toml
[dependencies]
"example.com/team/json" = { path = "../json" }
```

A library states which core compiler versions it supports:

```toml
module = "example.com/team/json"
paco = ">=0.4, <0.6"
```

If the running compiler is outside a module's range, the build stops before compiling with an error naming the module and the required range.

## Reference-level explanation

**Identity.** A module is identified by a path of the form `domain/segment/...`. One repository holds exactly one module; the repository's URL is the module path, fetched over `https://<module path>`. A tag on that repository names a version of that module. Tags follow semantic versioning.

**Manifest.** `paco.mod` is TOML with these fields:

| Field | Required | Meaning |
|---|---|---|
| `module` | yes | This module's own path. |
| `dependencies` | no | A table from module path to source: a tag string, or a table `{ path = "<dir>" }` for a local directory relative to this `paco.mod`. |
| `paco` | no | A comma-separated list of version constraints (`>=`, `<=`, `<`, and so on) that the running core compiler version must all satisfy. |

A malformed `paco.mod` is a compile error, never a silent fallback.

**Resolution.** A `use` of a domain path resolves to the declared dependency whose path is the longest prefix of it. A `use` with no matching dependency is an error that names the missing entry.

**Lock file.** `paco get` writes `paco.lock`, an array of `[[package]]` entries, each with the dependency's `path`, the `tag` it was declared with, and the `commit` it resolved to. On later runs, a dependency whose declared tag still matches its lock entry is checked out at the locked commit, not at whatever the tag points to now. Changing the tag in `paco.mod` triggers a fresh resolution for that dependency. Local `path` dependencies are not fetched, cached, or locked.

**Package cache.** Fetched dependencies live in a shared cache, `$PACO_PKG_CACHE`, else `~/.paco/pkg`, one directory per module path and tag. Concurrent `paco get` runs sharing the cache do not corrupt it.

**Compiler version range.** Before compiling, the compiler checks the root module's `paco` range and the `paco` range of every fetched dependency against its own version. Any mismatch is an error.

**Linking.** A program includes only the modules it imports, directly or transitively. The output is a single binary. A program whose imported modules contain no `extern` blocks links statically; importing a module that binds a foreign library affects only the programs that import it ([RFC 0018](0018-ffi-and-unsafe.md)).

## Drawbacks

A dependency depends on its host. If the upstream repository is deleted or made private, a fresh machine cannot fetch it, even with a lock file.

Names are tied to hosting. Moving a repository to a different host changes its module path and every `use` line that imports it.

Fetching requires `git` on the developer's machine.

## Rationale and alternatives

A central registry gives short names and a single authority to resolve them. It also requires hosting, a namespace policy, and ongoing moderation, and it makes the registry a single point of failure for every build. URL-based identity needs none of this and works with any git host.

Resolving a tag at every build is simpler than a lock file, but a force-moved tag would then silently change the code a program builds against. Locking the commit makes a build reproducible from `paco.mod` plus `paco.lock`.

Allowing several modules per repository, each versioned separately, would let related modules share one repository. Tags are per repository, so this would need per-directory version markers that git does not provide. One module per repository keeps a tag's meaning unambiguous.

Leaving compatibility with the compiler to documentation was considered. A declared `paco` range lets the toolchain report an incompatible library before compilation produces confusing type errors inside it.

## Prior art

Go modules identify dependencies by source-control URL and version tag, with `go get` and `go mod tidy` as the fetch and reconcile commands, and a module cache under the user's home directory. Cargo's `Cargo.lock` and npm's `package-lock.json` record exact resolved versions for reproducible builds. crates.io and npm are the central-registry model.

## Unresolved questions

None.

## Future possibilities

A vanity import domain, a dedicated hostname that redirects to the real repository location, would make module paths independent of the hosting provider. It needs HTTP-based path resolution that fetching does not specify today, and can be added later as an alias without breaking existing imports.

A mirror or caching proxy could protect builds against an upstream repository disappearing.
