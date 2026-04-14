---
proverb: "interface{} says nothing."
number: 6
category: actionable
sources:
  - https://google.github.io/styleguide/go/decisions (prefer `any`)
  - https://go.dev/doc/tutorial/generics (type parameters)
  - https://go.dev/doc/effective_go (Interfaces)
---

# #6 — interface{} says nothing

## What it means

An empty interface (`interface{}`, or `any` since Go 1.18) is the weakest possible type. It documents no behavior, enforces no contract, and forces every caller into type assertions or reflection to do anything useful with the value. Pike's proverb says: when you reach for `interface{}`, you're telling readers you know *nothing* about what the value is or can do — and that almost always means you need a real interface, a concrete type, or (since 1.18) a type parameter instead.

From the *Google Go Style Guide*:

> Go 1.18 introduces an `any` type as an alias to `interface{}`. Because it is an alias, `any` is equivalent to `interface{}` in many situations and in others it is easily interchangeable via an explicit conversion. Prefer to use `any` in new code.

Since Go 1.18, generics replace many previously-legitimate uses of `interface{}` in container and utility code with full type safety.

## What to look for (code smells)

- `func Process(v interface{})` / `func Process(v any)` where the function always handles one or two known types.
- Containers such as `type Cache struct { items map[string]interface{} }` that could be parameterised: `map[string]V`.
- APIs that return `interface{}` / `any` and force every caller to type-assert the result.
- Type switches with 3+ cases in a hot path — usually a small interface with a single method would be cleaner.
- Slices/maps of `interface{}` used to simulate generics (pre-1.18 style) in new code.
- JSON decoding into `map[string]interface{}` when a concrete struct would work.
- Test helpers that take `interface{}` as "equal any two things" when a generic `Equal[T comparable]` is possible.
- `interface{}` used as a "flexible options bag" instead of a real options struct.
- Legacy `interface{}` in files otherwise written after Go 1.18 — should be `any`.

## Good alternatives

| Instead of... | Prefer... |
|---|---|
| `func Max(a, b interface{}) interface{}` | `func Max[T cmp.Ordered](a, b T) T` |
| `map[string]interface{}` container | `map[string]V` with a type parameter, or a concrete struct |
| `func Handle(v interface{}) { switch v.(type) {...} }` | A small interface with a `Handle()` method, implemented by each type |
| `interface{}` as "flexible config" | A concrete options struct (ideally zero-value-useful — see #5) |
| `json.Unmarshal(data, &map[string]interface{}{})` | `json.Unmarshal(data, &MyStruct{})` |
| `List []interface{}` | `List []T` with a type parameter |

## Legitimate uses of `any`

The proverb is a default, not a ban. `any` is genuinely the right choice for:

- `fmt.Print`, `fmt.Printf` family — must accept anything.
- Logging key/value attributes (`slog.Attr` values) — values are heterogeneous by design.
- `encoding/json` round-tripping genuinely unknown or dynamic shapes.
- `context.Value` keys and values — `context` is explicitly a bag for opaque data.
- Cache libraries where the cache really stores arbitrary types across call sites.
- Reflection-based code (marshalers, validators, ORMs) operating on arbitrary types.
- Variadic test helpers intended to work across any user type.

In each of these the lack of type information is a deliberate, documented part of the contract — not an escape hatch.

## Review checklist

- [ ] Does this `interface{}` / `any` represent a real "accepts anything" contract, or hide a missing abstraction?
- [ ] Could a type parameter (Go 1.18+) replace it with no loss of flexibility and full type safety?
- [ ] Could a small behavioural interface (even a one-method one) replace it?
- [ ] Does the function body do a type switch? How many cases? Would a real interface be clearer?
- [ ] New code: is it written as `any` rather than `interface{}`?
- [ ] Does the API force callers to type-assert the return value? If yes, the return type is probably wrong.
- [ ] Is the "flexibility" real, or is it just "I didn't know the type yet"?

## Suggested finding phrasing

- "`Process(v interface{})` only handles `*User` and `*Order`; replace with an interface or a type parameter."
- "`Cache[string]interface{}` — since Go 1.18 this should be `Cache[string, V any]` (or concrete if only one type is ever stored)."
- "`json.Unmarshal` into `map[string]interface{}` — define a struct `type Event struct { … }` so the shape is documented and type-checked."
- "Type switch on `any` with 5 cases — extract a `Handler` interface with a single method and let each type implement it."
- "`interface{}` in new code — use `any` (Go style guide recommends the alias for new code)."

## When NOT to apply

- Interop with existing APIs that require `any` (e.g., `context.Value`, `database/sql.Scan`, `json.RawMessage` round-trips).
- Framework-level code whose purpose is to be type-agnostic (logging, metrics, encoders, generic caches).
- Third-party library boundaries where you don't control the signature.
- Genuinely dynamic config (e.g., reading arbitrary YAML into a map before validation).

## Sources

- <https://google.github.io/styleguide/go/decisions#any> — Google Go Style Guide: prefer `any` over `interface{}` in new code
- <https://go.dev/doc/tutorial/generics> — Go generics tutorial: type parameters eliminate many previous uses of `interface{}`
- <https://go.dev/doc/effective_go#interfaces> — Effective Go, *Interfaces*
