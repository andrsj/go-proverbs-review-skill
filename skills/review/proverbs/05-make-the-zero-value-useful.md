---
proverb: "Make the zero value useful."
number: 5
category: actionable
sources:
  - https://go.dev/doc/effective_go (Allocation with new; Constructors and composite literals)
  - https://go.dev/ref/spec (The zero value)
---

# #5 — Make the zero value useful

## What it means

Every Go value has a zero value — and well-designed types make that zero value immediately usable without an `Init()` call or a `NewFoo()` constructor. `var b bytes.Buffer` is a ready-to-use empty buffer. `var mu sync.Mutex` is a ready-to-use unlocked mutex. A nil `[]T` is a valid slice you can `append` to. Designing for useful zero values makes types easier to compose (as struct fields, as map values) and removes boilerplate for callers.

From *Effective Go*:

> Since the memory returned by `new` is zeroed, it's helpful to arrange when designing your data structures that the zero value of each type can be used without further initialization. This means a user of the data structure can create one with `new` and get right to work.
>
> For example, the documentation for `bytes.Buffer` states that "the zero value for `Buffer` is an empty buffer ready to use." Similarly, `sync.Mutex` does not have an explicit constructor or `Init` method. Instead, the zero value for a `sync.Mutex` is defined to be an unlocked mutex.

The property is **transitive**: if every field of a struct is zero-value-useful, the struct itself is too.

## What to look for (code smells)

- `NewFoo()` constructor that only does `return &Foo{}` — delete it; callers can write `var f Foo` or `&Foo{}`.
- `NewFoo()` that sets defaults that are already the zero value (e.g., `Items: []string{}`, `Count: 0`).
- `func (f *Foo) Init()` that must be called before use — move the initialization into lazy methods.
- Methods that nil-panic on a fresh struct when they could treat nil as "empty" or lazily allocate.
- Callers forced to use a factory because the zero struct panics or silently misbehaves.
- Struct contains an internal `map[K]V` that methods write to without allocating it — either make `New()` honest, or have methods lazily `make` it on first write.
- Types that require setting 3+ fields before any method works (strong signal the type is under-designed).
- Embedded types whose zero value is *not* usable, breaking transitivity for the outer struct.

## Good patterns

- `bytes.Buffer{}` / `var b bytes.Buffer` — works immediately.
- `sync.Mutex`, `sync.RWMutex`, `sync.Once`, `sync.WaitGroup`, `sync.Map` — all zero-value-useful.
- `time.Time{}` — the zero value satisfies `IsZero()` and represents "no time".
- Nil slices: `var s []int; s = append(s, 1)` is valid.
- Nil maps are safe to **read** (return zero value); writes still require `make`, so lazy-init inside methods is a common pattern.
- A config struct whose zero value represents "all defaults".

## Good example (from Effective Go)

```go
type SyncedBuffer struct {
    lock    sync.Mutex
    buffer  bytes.Buffer
}

p := new(SyncedBuffer)  // type *SyncedBuffer
var v SyncedBuffer      // type  SyncedBuffer
```

Both `p` and `v` are ready to use — no constructor needed. `SyncedBuffer` inherits zero-value-usefulness from its two embedded fields.

## Review checklist

- [ ] Can `var x T` be used without any setup? If not, why not?
- [ ] Does `NewT()` do anything the zero value couldn't?
- [ ] Do methods panic on a zero-value receiver when they could reasonably handle it as "empty"?
- [ ] Are internal maps/channels lazily allocated inside methods, or do they require `New()`?
- [ ] If this type embeds other types, are *they* zero-value-useful?
- [ ] Would removing the constructor break anything meaningful, or just make callers nicer?
- [ ] Is a nil receiver handled gracefully where it makes sense (e.g., nil linked list = empty)?

## When NOT to apply

- Types wrapping external resources (file handles, DB connections, network sockets) — the zero value genuinely isn't "ready", and a constructor is the honest signal.
- Types whose correct behavior requires validated caller input (`regexp.Regexp` needs a pattern, `time.Ticker` needs a duration).
- Types with required dependencies (logger, clock, RNG) that have no sensible default in your domain.
- When a constructor genuinely adds value (validation, wiring, measurement hooks, dependency injection).
- When the zero value would be *dangerously wrong* rather than "empty" (e.g., a `Rate` of 0 that silently disables throttling).

## Suggested finding phrasing

- "`NewOptions()` only returns `&Options{}`; delete it and let callers use `&Options{}` or `var opts Options`."
- "`Cache.Get` panics on a fresh `Cache{}` because `c.entries` is nil — lazily allocate inside `Set` so the zero value is usable."
- "`NewBuffer()` sets `buf: make([]byte, 0)` — a nil slice is already valid here, drop the constructor."
- "`Queue{}` silently misbehaves until `Init()` is called; move the initialization into the first method that needs it."

## Sources

- <https://go.dev/doc/effective_go#allocation_new> — Effective Go, *Allocation with new* — `bytes.Buffer`, `sync.Mutex`, `SyncedBuffer` example
- <https://go.dev/doc/effective_go#composite_literals> — Effective Go, *Constructors and composite literals*
- <https://go.dev/ref/spec#The_zero_value> — Go specification, *The zero value* — zero-value rules per type
