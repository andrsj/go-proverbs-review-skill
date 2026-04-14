---
proverb: "Don't just check errors, handle them gracefully."
number: 16
category: actionable
sources:
  - https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully
  - https://go.dev/blog/go1.13-errors
  - https://pkg.go.dev/errors
---

# #16 — Don't just check errors, handle them gracefully

## What it means

Checking an error (`if err != nil`) is not the same as handling one. This proverb, traced to Dave Cheney's 2016 post of the same name, argues that every error must be handled **once, and only once** — and "handle" means one of:

1. **Recover from it** — retry, fallback, use a default.
2. **Log it as the final destination** — with enough context that the log is actionable.
3. **Wrap it with context and return it up the stack** — so someone else can handle it.
4. **Deliberately swallow it** — with a comment explaining why.

What you must *not* do is the typical anti-pattern: log the error **and** return it, so two or three layers all log the same failure with different amounts of context — or strip the error of context by returning it raw, so the top-level logger sees `"no such file or directory"` with no clue which file.

From Cheney:

> You should only handle errors once. Handling an error means inspecting the error value, and making a decision.

> Treat errors as part of your public API with the same care given to any exported interface.

From the Go 1.13 errors post:

> Wrap an error to expose it to callers. [...] Do not wrap an error when doing so would expose implementation details.

> Wrapping an error makes that error part of your public API. Only wrap errors you're willing to support long-term.

## Relationship to proverb #15

**#15** says errors are *values* you can program with. **#16** says: once you have that value, decide what to *do* with it — don't mechanically pass it up untouched, and don't handle it twice.

## What to look for (code smells)

- **Double-logging**: `log.Println("failed:", err); return err` — both this layer and an upstream layer will log.
- **Bare re-raise**: `if err != nil { return err }` without adding context via `fmt.Errorf("...: %w", err)`.
- **Silent swallow**: `_ = someCall()` or `if err := x(); err != nil {}` with no handling and no comment.
- **Handle twice**: returning the error *and* performing a fallback at the same layer.
- **Handle nowhere**: the error propagates to a goroutine boundary where it's dropped.
- **Context stripping**: `fmt.Errorf("%v", err)` when `%w` would have preserved the chain for callers that need `errors.Is` / `errors.As`.
- **Boolean-ing errors**: `_, err := strconv.Atoi(x); if err != nil { return nil }` — converting a typed error to an empty result with no log or signal.
- **Catching to panic**: `if err != nil { panic(err) }` in library code.
- **Leaking implementation errors**: wrapping `*os.PathError` with `%w` when the package API should not expose the filesystem as a concept.
- **Filler wrapping**: every function wraps with `"failed to X"`, producing error chains like *"failed to fetch: failed to read: failed to open: no such file"*.
- Errors caught at the goroutine boundary with no way to surface them to the parent (no channel, no `errgroup`, no shared state).

## Good patterns

### Handle at the right layer

A low-level helper returns the error up; the outermost business boundary (HTTP handler, CLI command, goroutine launcher) logs, alerts, or maps to a response. Only one layer takes responsibility.

### Wrap with useful context, not filler

```go
return fmt.Errorf("fetching user %q: %w", id, err)
```

Include the *what* and *why* (identifiers, operation, input), not just `"failed"`.

### Decide wrap vs. opaque per API boundary

From the Go 1.13 post:

Opaque (hide implementation):

```go
f, err := os.Open(filename)
if err != nil {
    // Use %v instead of %w to hide the *os.PathError
    return fmt.Errorf("%v", err)
}
```

Wrapping (expose for callers):

```go
if err != nil {
    return fmt.Errorf("decompress %v: %w", name, err)
}
```

Wrap when the wrapped error is **part of your contract**. Keep opaque when it exposes an implementation detail you don't want to commit to.

### Behaviour over type (Cheney's recommendation)

Check what the error *can do*, not what type it is:

```go
type temporary interface {
    Temporary() bool
}

if t, ok := err.(temporary); ok && t.Temporary() {
    // retry
}
```

This decouples callers from concrete error types. The modern equivalent uses `errors.As` with a small local interface.

### Deliberate swallow — always commented

```go
_ = tx.Rollback() // rollback errors are harmless if commit already succeeded
```

## Review checklist

- [ ] For each `if err != nil` — is the error handled here, or propagated? Is the choice deliberate?
- [ ] Is any error both logged **and** returned from the same function?
- [ ] Does handling happen exactly *once* along the full call path?
- [ ] When wrapping, is useful context added (filenames, IDs, operations) rather than filler like `"failed"`?
- [ ] Is `%w` used when the wrapped error is meant to be inspectable by callers?
- [ ] Is `%v` used when exposing the underlying error would leak implementation details?
- [ ] Are swallowed errors accompanied by a comment explaining why?
- [ ] Are behaviour checks (`errors.As(err, &timeoutErr)`) used where they help callers decide?
- [ ] At the outermost boundary (HTTP handler, `main()`, goroutine), is there a real handling step?
- [ ] Are errors in library code *returned*, not `panic`'d?
- [ ] For goroutines, is there a mechanism (channel, `errgroup`, context) to surface errors to the parent?

## Suggested finding phrasing

- "`log.Println(err); return err` — both this layer and the caller will log; pick one."
- "Returning raw `os.ErrNotExist` from a package method — either wrap with context or translate to a domain error so callers aren't coupled to `os`."
- "`return err` with no context — add `fmt.Errorf(\"loading config %q: %w\", path, err)`."
- "`_ = tx.Rollback()` with no comment — add a note explaining why rollback errors are safe to drop here."
- "`fmt.Errorf(\"%v\", err)` used where callers need `errors.Is` to work — switch to `%w`."
- "Goroutine launched with `go f()` and any error inside is unreachable — use a channel, `errgroup`, or similar to surface it."
- "Every function wraps with `\"failed to X\"` — drop the filler and wrap only at meaningful boundaries."

## When NOT to apply

The proverb is close to absolute. The main "exception" is: for very small programs or prototypes, a single top-level `if err != nil { log.Fatal(err) }` can be the entire error-handling strategy — that's still *handled once*.

## Sources

- <https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully> — Dave Cheney, *"Don't just check errors, handle them gracefully"* (primary source; "handle once", behaviour-over-type, antipatterns)
- <https://go.dev/blog/go1.13-errors> — Damien Neil & Jonathan Amsterdam, *"Working with Errors in Go 1.13"* (`%w`, `errors.Is`, `errors.As`, wrap-vs-opaque guidance)
- <https://pkg.go.dev/errors> — `errors` package: `Is`, `As`, `Unwrap`, `Join`
