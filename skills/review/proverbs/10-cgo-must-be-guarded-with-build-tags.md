---
proverb: "Cgo must always be guarded with build tags."
number: 10
category: actionable
sources:
  - https://pkg.go.dev/cmd/cgo
  - https://go.dev/blog/cgo
  - https://pkg.go.dev/cmd/go (build constraints, CGO_ENABLED)
---

# #10 — Cgo must always be guarded with build tags

## What it means

Cgo makes a Go package depend on:

- a working C toolchain (`gcc`/`clang`) at build time
- platform-specific headers and libraries
- the C ABI and linker behaviour of the target OS
- a dynamic or static link step

A file with `import "C"` compiles only when `CGO_ENABLED=1` **and** the C compiler can find the referenced headers/libs **and** you're building for a platform that has them. Without explicit build tags **plus a non-cgo fallback**, the package silently breaks cross-compilation, static binaries (`CGO_ENABLED=0`), scratch Docker images, and unsupported OS targets.

Pike's proverb: always guard cgo files with build tags, and always provide a non-cgo implementation for the `!cgo` case.

### Important nuance

`import "C"` already **implicitly** adds the `//go:build cgo` constraint — from `cmd/cgo`:

> When cgo is disabled, files importing `"C"` will not be built because the import implies the `"cgo"` build constraint: `//go:build cgo`.

That implicit tag is the *minimum*. The proverb demands more:

1. Add an **explicit** `//go:build cgo` (or `//go:build cgo && linux`) so readers see the intent.
2. Provide a **stub/fallback file** guarded by `//go:build !cgo` so the package still builds when cgo is off.
3. If the cgo code is platform-specific, add OS tags too.

## What to look for (code smells)

- `import "C"` in a file without an **explicit** `//go:build` line.
- No `_nocgo.go` / `_stub.go` companion file for the `!cgo` case.
- cgo code that only works on one OS with no OS guard (`//go:build cgo && linux`).
- Packages that fail to build with `CGO_ENABLED=0 go build ./...` in CI or Docker.
- Mixed cgo and pure-Go implementations in the same file behind runtime branches.
- `FROM scratch` Dockerfile that fails because `CGO_ENABLED=0` was forgotten.
- Pure-Go alternative exists (e.g. `modernc.org/sqlite` vs `github.com/mattn/go-sqlite3`) but wasn't considered.
- `C.CString` / `C.malloc` without a matching `defer C.free` — a memory leak, not a build issue, but cgo-review territory.

## Canonical layout

```
native_cgo.go         // //go:build cgo
native_nocgo.go       // //go:build !cgo
native_cgo_linux.go   // //go:build cgo && linux
native_cgo_darwin.go  // //go:build cgo && darwin
```

Each file implements the same exported API. The `_nocgo.go` stub can return an error, use a pure-Go fallback, or be a no-op depending on your needs.

## Minimal example

`hash_cgo.go`:

```go
//go:build cgo

package hash

// #include <openssl/sha.h>
import "C"

func Sum256(data []byte) [32]byte {
    // call OpenSSL via cgo
}
```

`hash_nocgo.go`:

```go
//go:build !cgo

package hash

import "crypto/sha256"

func Sum256(data []byte) [32]byte {
    return sha256.Sum256(data)
}
```

With this layout the package builds both with and without cgo, with no runtime branches and no platform surprises.

## Review checklist

- [ ] Does this file import `"C"`? If yes, does it have an **explicit** `//go:build cgo` line?
- [ ] Is there a non-cgo fallback file guarded by `//go:build !cgo`?
- [ ] Does the package build with `CGO_ENABLED=0 go build ./...`?
- [ ] Does CI test both `CGO_ENABLED=0` and `CGO_ENABLED=1`?
- [ ] If the cgo code is OS-specific, does the constraint include the OS?
- [ ] Are Docker images that want `FROM scratch` actually static-linkable?
- [ ] Is there a pure-Go alternative that would remove the cgo requirement entirely?
- [ ] Are C allocations (`C.CString`, `C.malloc`) paired with `defer C.free(unsafe.Pointer(...))`?
- [ ] Are exported Go symbols / C callbacks handled correctly for the current platform?

## Suggested finding phrasing

- "`native.go` imports `\"C\"` with no explicit `//go:build cgo` — add the tag even though it's implicit, and add a `native_nocgo.go` fallback."
- "Package fails to build with `CGO_ENABLED=0` — add a `_nocgo.go` stub so static builds and scratch Docker images still work."
- "cgo file has no OS guard; OpenSSL headers aren't available on Windows — split into `hash_cgo_linux.go` / `hash_cgo_darwin.go` and stub the rest."
- "`C.CString(s)` is leaked; add `defer C.free(unsafe.Pointer(cs))` immediately after allocation."

## When NOT to apply

The rule is absolute. "Not applying" really means "don't use cgo at all" — see proverb #11 *"Cgo is not Go."* If you have a pure-Go alternative, prefer it; if you must use cgo, guard every file.

## Sources

- <https://pkg.go.dev/cmd/cgo> — `cmd/cgo` reference, including the implicit `//go:build cgo` constraint and `CGO_ENABLED` behaviour
- <https://go.dev/blog/cgo> — "C? Go? Cgo!" — original introduction with the `C.CString` + `defer C.free` idiom
- <https://pkg.go.dev/cmd/go#hdr-Build_constraints> — `cmd/go` build constraints reference
