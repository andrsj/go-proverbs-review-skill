---
proverb: "Errors are values."
number: 15
category: actionable
sources:
  - https://go.dev/blog/errors-are-values
  - https://go.dev/blog/error-handling-and-go
  - https://pkg.go.dev/errors
---

# #15 — Errors are values

## What it means

In Go, `error` is just an interface with one method returning a string — it's a plain value that flows through your program like any other. That sounds obvious, but Pike's proverb is pushing against the habit of treating `if err != nil { return err }` as the *entire vocabulary* for errors. Since errors are values, you can store them, wrap them, accumulate them, define domain-specific error types, cache them in a struct field to batch operations, route them, and pattern-match on them. The point isn't to skip checking — *"Whatever you do, always check your errors!"* — it's that the checking itself can be structured like any other code rather than mechanical noise.

From Rob Pike's *Errors are values*:

> Errors are values. Values can be programmed, and since errors are values, errors can be programmed.

> The key lesson [...] is that errors are values and the full power of the Go programming language is available for processing them. Use the language to simplify your error handling. But remember: Whatever you do, always check your errors!

From *Error handling and Go*:

> since `error` is an interface, you can use arbitrary data structures as error values, to allow callers to inspect the details of the error.

The `error` interface itself:

```go
type error interface {
    Error() string
}
```

## Three patterns from Pike's post

### 1. Sticky error (bufio.Scanner style)

The loop doesn't check; the caller checks once after:

```go
scanner := bufio.NewScanner(input)
for scanner.Scan() {
    process(scanner.Text())
}
if err := scanner.Err(); err != nil {
    // handle
}
```

Pike: *"The client's code therefore feels more natural: loop until done, then worry about errors. Error handling does not obscure the flow of control."*

### 2. errWriter — a struct that swallows subsequent operations once one has failed

```go
type errWriter struct {
    w   io.Writer
    err error
}

func (ew *errWriter) write(buf []byte) {
    if ew.err != nil {
        return
    }
    _, ew.err = ew.w.Write(buf)
}
```

Usage:

```go
ew := &errWriter{w: fd}
ew.write(p0[a:b])
ew.write(p1[c:d])
ew.write(p2[e:f])
if ew.err != nil {
    return ew.err
}
```

Pike: *"This is cleaner [...] and also makes the actual sequence of writes being done easier to see on the page. There is no clutter anymore."*

### 3. Custom error types

Since `error` is an interface, attach structured fields that callers can inspect:

```go
type SyntaxError struct {
    Msg    string // description of error
    Offset int64  // error occurred after reading Offset bytes
}

func (e *SyntaxError) Error() string { return e.Msg }
```

## Modern companions (Go 1.13+)

- `fmt.Errorf("...: %w", err)` — **wrap** and preserve the chain
- `errors.Is(err, io.EOF)` — sentinel comparison through wrapping
- `errors.As(err, &myErr)` — type extraction through wrapping
- `errors.Join(errs...)` (Go 1.20+) — combine multiple errors into one

## What to look for (code smells)

- Long runs of `if err != nil { return err }` with **no wrapping context** — callers see `"not found"` with no idea where it came from.
- Repeated near-identical error checks inside a loop that could be a sticky-error struct (like `errWriter`).
- Comparing errors with `==` instead of `errors.Is` when the target can be wrapped.
- Type-asserting errors with `err.(*FooError)` instead of `errors.As`.
- Wrapping with `fmt.Errorf("%v", err)` instead of `%w` — loses the chain.
- Flat string errors for conditions callers need to branch on — no way to inspect.
- Sentinel errors declared but not exported, so callers can't `errors.Is` them.
- Errors used for non-error control flow ("not found" as a loop terminator where a bool would do).
- `panic` used where a returned `error` would be cleaner.
- `err.Error()` strings matched with `strings.Contains` instead of typed checks.
- "Log and return" at every layer — pick one layer to handle, others to propagate.
- Domain errors declared as `errors.New("string")` when inspectable fields would help callers.

## Good patterns

### Typed errors for programmable inspection

```go
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %q not found", e.Resource, e.ID)
}
```

### Sentinel plus wrap

```go
var ErrNotFound = errors.New("not found")

func GetUser(id string) (*User, error) {
    // ...
    return nil, fmt.Errorf("user %q: %w", id, ErrNotFound)
}

// caller
if errors.Is(err, ErrNotFound) {
    // handle
}
```

### Sticky-error accumulators for builders and fluent APIs

Same pattern as `errWriter` — any method on the builder returns early if `b.err != nil`, and the caller checks once at the end.

### Centralised handler mapping

```go
type appHandler func(http.ResponseWriter, *http.Request) error

func (fn appHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    if err := fn(w, r); err != nil {
        // one place that knows how to turn errors into HTTP responses
    }
}
```

### Batch partial failures

```go
var errs []error
for _, item := range items {
    if err := process(item); err != nil {
        errs = append(errs, err)
    }
}
return errors.Join(errs...) // Go 1.20+
```

## Review checklist

- [ ] Do errors returned from this package carry enough context for the caller?
- [ ] Are errors wrapped with `%w` (not `%v`) when context is added?
- [ ] Is `errors.Is` used instead of `==` for sentinel comparisons?
- [ ] Is `errors.As` used instead of type assertions for structured errors?
- [ ] For repetitive `if err != nil { return err }`, could an `errWriter`-style sticky error simplify it?
- [ ] Do domain errors have enough structure to be programmable, not just `errors.New("string")`?
- [ ] Are sentinel errors exported so callers can check them?
- [ ] Is the error used for genuine failure, not control flow?
- [ ] Is there a pattern of "log and return" at every layer? Pick one.
- [ ] For concurrent operations, is `errors.Join` or an error-collecting channel used appropriately?

## Suggested finding phrasing

- "`fmt.Errorf(\"%v\", err)` — use `%w` to preserve the chain so callers can `errors.Is` / `errors.As`."
- "`err == io.EOF` — use `errors.Is(err, io.EOF)` in case the error has been wrapped."
- "`errors.New(\"invalid input\")` — callers have no way to inspect *which* field was invalid; define a typed error."
- "Three near-identical write/check pairs in a row — extract an `errWriter`-style helper."
- "This function logs the error and then returns it — pick one layer to log, propagate from the rest."
- "`panic(err)` used where returning `error` would be idiomatic."

## When NOT to apply

The proverb is almost always in force. The main trap is over-engineering — don't build an `errWriter`-style abstraction for a single two-line sequence. Use the pattern when the repetition is real and the resulting code is genuinely clearer. And the single non-negotiable: **always check your errors.**

## Sources

- <https://go.dev/blog/errors-are-values> — Rob Pike, *"Errors are values"* (primary source for the proverb, `bufio.Scanner` and `errWriter` examples)
- <https://go.dev/blog/error-handling-and-go> — Andrew Gerrand, *"Error handling and Go"* (the `error` interface, custom types, handler mapping)
- <https://pkg.go.dev/errors> — `errors` package: `Is`, `As`, `Join`, `Unwrap`
