---
proverb: "Don't panic."
number: 19
category: actionable
sources:
  - https://go.dev/wiki/CodeReviewComments#dont-panic
  - https://go.dev/blog/defer-panic-and-recover
  - https://go.dev/doc/effective_go (Panic; Recover)
---

# #19 — Don't panic

## What it means

`panic` stops normal execution, unwinds the stack running deferred functions, and — if not caught by `recover` — crashes the program. It is Go's mechanism for **truly exceptional, unrecoverable** situations, not for ordinary error handling. In Go, failures are **values** (`error`), not control flow. You return them, you handle them (see #15 and #16) — you don't throw them.

From *Go Code Review Comments*:

> Don't use panic for normal error handling. Use error and multiple return values.

> If a function returns an error, check it to make sure the function succeeded. Handle the error, return it, or, in truly exceptional situations, panic.

From *Defer, Panic, and Recover*:

> When the function F calls panic, execution of F stops, any deferred functions in F are executed normally, and then F returns to its caller. To the caller, F then behaves like a call to panic. The process continues up the stack until all functions in the current goroutine have returned, at which point the program crashes.

> The convention in the Go libraries is that even when a package uses panic internally, its external API still presents explicit error return values.

## When panic *is* acceptable (narrow list)

1. **Truly unreachable code / failed invariants** — `default:` in a switch over a closed set you're certain can't fire; programmer errors that should crash loudly during development.
2. **Package `init()` or startup** — missing required config, failed compile-time regex, unrecoverable environment. `log.Fatal` is often a cleaner alternative here.
3. **Internal stack unwinding** caught by a `recover` at the package boundary (the `encoding/json` pattern).
4. **`Must*` constructors with literal inputs** — `regexp.MustCompile`, `template.Must`, `sql.MustExec` called with compile-time-known arguments.
5. **Tests** — though `t.Fatal` / `require.NoError` are the idiomatic equivalents; avoid raw panics in test code unless a hard crash is what you actually want.

## What to look for (code smells)

- `panic(err)` as a reflexive response to `if err != nil` in library code.
- `panic("not implemented")` left in shipped code.
- `panic` used for control flow (early-exit from deep recursion without a matching `recover`).
- `panic` carrying a user-input-derived value — an attacker can trigger it repeatedly.
- Library functions that panic on bad caller input instead of returning an error.
- HTTP handlers that panic with no top-level `recover` middleware — crashes the server.
- `recover()` calls outside a deferred function — they are no-ops.
- `defer func() { recover() }()` with no logging or error handling — silently swallows crashes.
- `Must*` helpers called with runtime values (defeats the point — use the plain `Compile` form).
- `panic` in a goroutine launched without a recovery wrapper — crashes the whole process.
- Worker pools / background tasks / long-running goroutines with no panic handler at their boundary.
- `panic` where `log.Fatal` would read more clearly (startup failures, no meaningful caller to unwind to).

## The three-layer pattern (internal panic → external error)

The idiom from `encoding/json`: use `panic` internally for fast stack unwinding, but keep the public API error-returning.

```go
// Public API — returns error
func (d *Decoder) Decode(v interface{}) (err error) {
    defer func() {
        if r := recover(); r != nil {
            if e, ok := r.(decodeError); ok {
                err = e.err
            } else {
                panic(r) // re-panic anything we don't recognise
            }
        }
    }()
    d.decode(v) // may panic(decodeError{...}) to unwind quickly
    return nil
}
```

Key points: the recover **discriminates** between the package's own panic values and foreign ones (it re-panics the latter), and the public API never exposes `panic` to callers.

## The `MustXxx` convention

Valid — input is a compile-time literal, so failure is a programmer error:

```go
var addrRE = regexp.MustCompile(`^\d+\.\d+\.\d+\.\d+$`)
```

**Wrong** — input comes from a caller or network:

```go
// WRONG: panics on bad user input
pat := regexp.MustCompile(r.FormValue("pattern"))

// CORRECT
pat, err := regexp.Compile(r.FormValue("pattern"))
if err != nil {
    // handle
}
```

## Recovering at goroutine boundaries

A panic in a goroutine that no one recovers will crash the entire process. Long-running goroutines should wrap their work in a recovery:

```go
go func() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("worker panic: %v\n%s", r, debug.Stack())
            // restart, alert, report, etc.
        }
    }()
    work()
}()
```

The same pattern belongs in HTTP middleware, job runners, and any long-lived background worker.

## Good vs bad

### Bad

```go
func Parse(input string) *Result {
    if input == "" {
        panic("empty input")
    }
    // ...
}
```

### Good

```go
func Parse(input string) (*Result, error) {
    if input == "" {
        return nil, errors.New("parse: empty input")
    }
    // ...
}
```

## Review checklist

- [ ] Does this panic represent a truly unrecoverable state, or is it lazy error handling?
- [ ] Could the panic be replaced with `return ..., err`?
- [ ] Is this panic in a library function called from user code? If yes, almost certainly wrong.
- [ ] If there's a `recover()`, is it inside a deferred function?
- [ ] Does the `recover()` discriminate between expected and unexpected panics, re-panicking the latter?
- [ ] Is `log.Fatal` a better fit for a startup failure?
- [ ] Are `MustXxx` helpers only called with literals known at compile time?
- [ ] Does every long-running goroutine have a recovery wrapper?
- [ ] Is `panic("not implemented")` left in code meant to ship?
- [ ] Does the HTTP server have middleware that recovers handler panics?
- [ ] Does the package's public API return errors even if it uses panic internally?

## Suggested finding phrasing

- "`panic(err)` in library code — return the error instead; let the caller decide."
- "`regexp.MustCompile` with user input — use `regexp.Compile` and handle the error."
- "Goroutine launched with no recovery wrapper — a panic here crashes the whole process."
- "`recover()` outside a deferred function — this is a no-op."
- "`defer func() { recover() }()` silently swallows panics — at minimum, log them and re-panic unrecognised values."
- "`panic(\"not implemented\")` in shipped code — replace with a real implementation or return a clear error."
- "HTTP handler panics have no top-level middleware recovering them — add one."
- "Public API calls `panic` directly; switch to an error return even if you keep the panic internally (see `encoding/json` pattern)."

## When NOT to apply (panic *is* OK)

- `init()`-time setup failures (missing required env var, compile-time regex that fails means the binary is broken).
- Failed invariants / unreachable code — `panic("unreachable")`.
- `Must*` constructors with literal inputs.
- Internal fast-unwind where the public API still returns `error`.
- Tests, where `t.Fatal` / `require.NoError` play the same role.

## Sources

- <https://go.dev/wiki/CodeReviewComments#dont-panic> — Go Code Review Comments, *Don't panic* (primary source quoting the rule)
- <https://go.dev/blog/defer-panic-and-recover> — Andrew Gerrand, *Defer, Panic, and Recover* (mechanics, library convention, `encoding/json` pattern)
- <https://go.dev/doc/effective_go#panic> — Effective Go, *Panic* / *Recover*
