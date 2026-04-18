---
proverb: "The bigger the interface, the weaker the abstraction."
number: 4
category: actionable
sources:
  - https://go.dev/doc/effective_go (Interfaces)
  - https://go.dev/wiki/CodeReviewComments (Interfaces)
---

# #4 — The bigger the interface, the weaker the abstraction

## What it means

In Go, an interface defines *what behavior is required*, not what a type can do. The smaller the interface (fewer methods), the more types can satisfy it and the more places it can be used. A one-method `io.Reader` abstracts files, network sockets, buffers, gzip streams, crypto streams — anything that produces bytes. A ten-method interface limits implementers dramatically and usually encodes the shape of one specific implementation rather than a genuine abstraction.

From *Effective Go*:

> Interfaces with only one or two methods are common in Go code, and are usually given a name derived from the method, such as `io.Writer` for something that implements `Write`.

From *Go Code Review Comments*:

> Go interfaces generally belong in the package that uses values of the interface type, not the package that implements those values. The implementing package should return concrete (usually pointer or struct) types: that way, new methods can be added to implementations without requiring extensive refactoring.
>
> Do not define interfaces on the implementor side of an API "for mocking"; instead, design the API so that it can be tested using the public API of the real implementation.
>
> Do not define interfaces before they are used: without a realistic example of usage, it is too difficult to see whether an interface is even necessary, let alone what methods it ought to contain.

The proverb has **two linked dimensions**:

1. **Size** — fewer methods produce stronger, more reusable abstractions.
2. **Location** — interfaces belong in the consumer package, not the producer.

## What to look for (code smells)

- Interfaces with 5+ methods that mirror a concrete struct's public API one-to-one.
- Interfaces named `*Service`, `*Manager`, `*Repository` that contain every method of their only implementation.
- Interfaces defined in the **same package** as their only implementation, exposing the full surface of the concrete type.
- Interfaces created purely "for mocking" in tests, with no second real implementation.
- `NewFoo()` constructors that return an interface instead of a concrete type (violates "return concrete, accept interface").
- Large interfaces that could be split into role-based sub-interfaces and composed (e.g., `io.Reader` + `io.Writer` + `io.Closer` → `io.ReadWriteCloser`).
- A function parameter typed as a fat interface when it only calls one or two methods — narrow the parameter.

## Bad example (from Code Review Comments, verbatim)

Producer-side interface defined "for mocking":

```go
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

## Good example

Producer returns a concrete type; consumer defines its own narrow interface:

```go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }
func NewThinger() Thinger { return Thinger{ … } }
```

```go
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

```go
package consumer  // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }

// if Foo(fakeThinger{…}) == "x" { … }
```

## Composition over fat interfaces (from Effective Go)

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// ReadWriter is the interface that combines the Reader and Writer interfaces.
type ReadWriter interface {
    Reader
    Writer
}
```

Prefer several small, focused interfaces that compose over one big interface declared up-front.

## Review checklist

- [ ] Does this interface have more than 2-3 methods? If so, can it be split?
- [ ] Does every method genuinely describe *behavior the consumer needs*, or does it mirror the implementation?
- [ ] Is the interface defined in the **consumer** package or the producer package?
- [ ] Does it have more than one real implementation, or was it created "for mocking"?
- [ ] Does `NewFoo()` return a concrete type or an interface? (Prefer concrete.)
- [ ] Could the interface be composed of smaller interfaces (like `io.ReadWriter = Reader + Writer`)?
- [ ] Does each call site of a function using this interface actually need *all* of its methods?

## When NOT to apply

- Stable, well-known interfaces (`http.Handler`, `error`, `io.Reader`, `fmt.Stringer`) — already minimal.
- Genuinely polymorphic APIs where multiple real implementations exist (database drivers, codecs, transport backends).
- Interfaces defined in a shared package purely because multiple consumer packages need the same contract.
- Method sets that are inherently cohesive and always used together — artificially splitting them just to hit a method-count target is worse than one honest interface.

## Suggested finding phrasing

- "`UserService` interface has 11 methods that mirror `userServiceImpl` 1:1 — consider returning the concrete type and letting consumers define narrower interfaces they actually need."
- "`Storage` interface defined in the same package as its only implementation; move the interface to the consumer package (or drop it and return concrete)."
- "`NewClient()` returns `Client` (interface). Prefer returning `*client` (concrete) so methods can be added without breaking consumers."
- "`Foo(repo Repository)` only calls `repo.Get`; narrow the parameter to a one-method `Getter` interface."

## Sources

- <https://go.dev/doc/effective_go#interfaces> — Effective Go, *Interfaces* — small interfaces, `io.Reader`/`io.Writer`, interface composition
- <https://go.dev/wiki/CodeReviewComments#interfaces> — Go Code Review Comments, *Interfaces* — consumer-side definition, concrete returns, anti-mocking guidance (verbatim bad/good example)
