---
proverb: "Syscall must always be guarded with build tags."
number: 9
category: actionable
sources:
  - https://pkg.go.dev/syscall
  - https://pkg.go.dev/cmd/go (build constraints)
  - https://pkg.go.dev/golang.org/x/sys
---

# #9 — Syscall must always be guarded with build tags

## What it means

The `syscall` package is OS- and architecture-specific. Values, type layouts, and even function signatures differ between Linux / Darwin / Windows / BSD and between 32-bit / 64-bit. A single `.go` file that calls `syscall` without a build tag will either fail to compile on other platforms, silently produce wrong behaviour, or break the cross-platform guarantees the rest of the codebase depends on.

Pike's proverb is an absolute rule: **every file touching `syscall` must be gated by a build constraint** — either an explicit `//go:build` line or a filename suffix like `_linux.go`, `_darwin_amd64.go`.

From the `syscall` package overview:

> Package syscall contains an interface to the low-level operating system primitives. **The details vary depending on the underlying system**, and by default, godoc will display the syscall documentation for the current system. [...]
>
> The primary use of syscall is inside other packages that provide a more portable interface to the system, such as `os`, `time` and `net`. Use those packages rather than this one if you can.
>
> NOTE: Most of the functions, types, and constants defined in this package are also available in the `golang.org/x/sys` package. That package has more system call support than this one, and most new code should prefer that package where possible.

So the proverb really has two layers:

1. **If you must use `syscall`, guard every file with build tags.**
2. **Prefer not using `syscall` at all** — reach for `os` / `net` / `time` or `golang.org/x/sys/{unix,windows}` instead.

## What to look for (code smells)

- A `.go` file that imports `syscall` with **no** `//go:build` line and **no** OS filename suffix.
- Mixed per-OS code in the same file gated by runtime checks (`if runtime.GOOS == "linux"`) instead of build tags.
- A single file declaring types/constants from `syscall` that only make sense on one OS.
- `syscall` used in business logic that should be calling higher-level packages (`os`, `net`, `time`).
- New code using `syscall.*` when `golang.org/x/sys/unix` or `golang.org/x/sys/windows` would work.
- Tests importing `syscall` but running on all platforms without guards.
- No stub file for unsupported platforms — breaks builds on CI targets.

## Build-tag patterns

### Explicit `//go:build`

```go
//go:build linux || darwin

package foo
```

The `//go:build` line must be near the top of the file, before the package clause, followed by a blank line.

### Filename convention (preferred when possible)

| Filename | Implicit constraint |
|---|---|
| `foo_linux.go` | `GOOS=linux` |
| `foo_darwin.go` | `GOOS=darwin` |
| `foo_windows.go` | `GOOS=windows` |
| `foo_linux_amd64.go` | `GOOS=linux && GOARCH=amd64` |
| `foo_unix.go` | needs an explicit `//go:build unix` (virtual term added in Go 1.19) |

### Typical per-OS layout

```
agent_linux.go    // //go:build linux
agent_darwin.go   // //go:build darwin
agent_windows.go  // //go:build windows
agent_stub.go     // //go:build !linux && !darwin && !windows
```

Each file implements the same exported API; only the guarded variant compiles on each platform.

## Review checklist

- [ ] Does this file import `syscall`? If yes → it must have a build constraint.
- [ ] Is the constraint expressed via `//go:build` **or** filename suffix?
- [ ] Could this be replaced by `os` / `net` / `time` or `golang.org/x/sys/*`?
- [ ] Does the package build cleanly on all platforms declared in `go.mod` / CI?
- [ ] Is there a stub/fallback file for unsupported platforms so the package still builds?
- [ ] Are constants like `O_CLOEXEC`, `AF_UNIX`, `SIGINT` wrapped in per-OS files (they differ across OSes)?
- [ ] Do tests that touch `syscall` have matching build guards?
- [ ] Is `runtime.GOOS` used where a build tag would be more correct?

## When NOT to apply

The rule is effectively absolute. The only "exception" is: **don't use `syscall` at all** when a portable higher-level package works. That removes the guard requirement because the problem goes away.

## Suggested finding phrasing

- "`agent.go` imports `syscall.Kill` but has no build constraint — add `//go:build linux || darwin` or rename to `agent_unix.go` + `agent_windows.go`."
- "`syscall.AF_UNIX` used in a file compiled on Windows — this won't build; split into per-OS files."
- "`if runtime.GOOS == \"linux\" { syscall.... }` — convert to a build-tagged file instead."
- "Uses `syscall.Mkfifo`; replace with `golang.org/x/sys/unix.Mkfifo` for better cross-platform support."
- "No stub file for non-Linux platforms — package fails to build on Darwin/Windows CI."

## Sources

- <https://pkg.go.dev/syscall> — `syscall` package overview, portability warning, and pointer to `golang.org/x/sys`
- <https://pkg.go.dev/cmd/go#hdr-Build_constraints> — `cmd/go` build constraints reference (`//go:build`, filename suffixes, `GOOS`/`GOARCH` patterns)
- <https://pkg.go.dev/golang.org/x/sys> — the preferred modern replacement for most `syscall` usage
