---
proverb: "Cgo is not Go."
number: 11
category: philosophical
sources:
  - https://pkg.go.dev/cmd/cgo
  - https://dave.cheney.net/2016/01/18/cgo-is-not-go
---

# #11 — Cgo is not Go

## What it means

Using cgo gives up most of what makes Go *Go*: fast builds, easy cross-compilation, a single static binary, goroutine-friendly I/O, the race detector, the Go runtime's memory safety at the boundary. In exchange you get C libraries. The proverb is a warning that the trade is larger than it looks — a package that imports `"C"` is no longer quite a Go package.

Typical costs:

- Cross-compilation requires a C toolchain for the target (`CC_FOR_${GOOS}_${GOARCH}`).
- Build times increase substantially (cgo compiles C and links it).
- `CGO_ENABLED=0` breaks the package; scratch Docker images and static binaries need a non-cgo fallback.
- Calls across the Go/C boundary have overhead (tens to hundreds of ns each) and hide goroutine scheduling.
- Memory management crosses two heaps: Go's GC doesn't see C allocations, and C doesn't see Go's.
- `go vet`, the race detector, and most analysis tools don't cross the boundary.
- Security: C buffer errors re-enter the program.
- The Go 1 compatibility promise doesn't cover cgo.

## Why it's not a review rule

The actionable, mechanical rules around cgo are already captured by proverb **#10** *"Cgo must always be guarded with build tags."* That's where the reviewer emits findings: missing `//go:build` tags, missing `!cgo` fallback files, broken `CGO_ENABLED=0` builds, unpaired `C.CString` / `C.free`, etc.

Proverb #11 is the *philosophical* companion: it asks *"should this be cgo at all?"* — a judgement call that a mechanical reviewer can't make responsibly. Whether to accept the cgo cost depends on what the package is for, what alternatives exist, who's maintaining it, and how the binary is deployed. Those are design-review questions, not lint rules.

## How reviewers should use it

Use this as background when:

- A PR **adds** a cgo dependency — ask whether a pure-Go library exists (`modernc.org/sqlite` vs `github.com/mattn/go-sqlite3`; `filippo.io/edwards25519` vs cgo OpenSSL), and whether the project's deployment model can still accommodate `CGO_ENABLED=0`.
- A PR introduces `import "C"` for a single helper function — almost always a sign the cost isn't worth it.
- A project complains about build times, cross-compilation, or scratch Docker images — cgo is usually the cause and worth questioning.

When a specific cgo *mechanical* problem is found (missing build tag, no fallback file, leaked `C.CString`), file it under **#10**, not here.

## Sources

- <https://pkg.go.dev/cmd/cgo> — `cmd/cgo` reference (the primary, authoritative doc)
- <https://dave.cheney.net/2016/01/18/cgo-is-not-go> — Dave Cheney, *"cgo is not Go"* — a long-form explanation of the costs this proverb warns about
