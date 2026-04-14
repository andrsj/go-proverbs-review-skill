---
proverb: "Reflection is never clear."
number: 14
category: actionable
sources:
  - https://go.dev/blog/laws-of-reflection
  - https://pkg.go.dev/reflect
---

# #14 — Reflection is never clear

## What it means

`reflect` is a runtime-only type system. Code using it is harder to read, harder to statically check, prone to runtime panics, an order of magnitude slower than direct calls, and invisible to most tooling (`go vet`, refactorers, type-aware IDE navigation). The proverb isn't "never use reflect" — it's "reflection is never *clear*" — which means: when you reach for it, you're trading clarity for flexibility, and you need a good reason every time.

From *The Laws of Reflection* (Rob Pike, 2011):

> Reflection in computing is the ability of a program to examine its own structure, particularly through types; it's a form of metaprogramming. It's also a great source of confusion. [...] It's a powerful tool that should be used with care and **avoided unless strictly necessary**.

From the `reflect` package docs:

> Calling a method inappropriate to the kind of type causes a run-time panic.

## The three laws (context, not smells)

1. Reflection goes from an interface value to a reflection object (`reflect.TypeOf`, `reflect.ValueOf`).
2. Reflection goes from a reflection object back to an interface value (`v.Interface()`).
3. To modify via reflection, the value must be **settable** — it must hold the original's address (pass a pointer, then `Elem()`).

These laws are the mental model, not the danger. The danger is everything you do *after* you have a `reflect.Value`.

## What to look for (code smells)

- `reflect` used in a hot path (often 10–100× slower than direct calls).
- `reflect` in code that could use a **type parameter** (Go 1.18+) or a small **interface** instead.
- No `Kind()` check before a kind-specific method (`Int()`, `Float()`, `Field()`, `Elem()`) — will panic at runtime.
- Missing `CanSet()` / `CanAddr()` / `IsValid()` guard before a mutation.
- Reflection used to read unexported fields — panics on `v.Interface()`.
- Generic-looking APIs taking `interface{}` and internally reflecting — usually a type parameter fits.
- No tests for the reflection path, or tests that don't cover bad `Kind` inputs.
- Reflection used to copy struct fields that could be copied directly.
- Reflection over structs with no fallback when `reflect.TypeOf(x).Kind() != reflect.Struct`.
- `FieldByName` inside a loop (O(n) per call) instead of caching `Field(i)` indexes.
- Repeatedly calling `reflect.TypeOf` on the same type in a hot loop instead of caching.
- Tag parsing (`StructTag.Get`) in a hot loop — allocates.

## When reflection *is* acceptable

- Serialization libraries (`encoding/json`, `encoding/xml`, custom codecs) — unavoidable.
- Generic test helpers (`reflect.DeepEqual`, `cmp.Equal`) — unavoidable.
- ORM / schema mapping where the field set is genuinely dynamic.
- Template engines, dependency injection frameworks, validators.
- One-time setup code (wiring at program start) where cost is amortised.
- Diagnostic / dumping / pretty-printing tools.

## Decision rule

If you can replace `reflect` with:

- a type parameter (Go 1.18+) → do that
- a small interface with one method → do that
- a type switch over a known set → do that
- a code generator (`go generate`) → consider that

…then you should. Reflection is the fallback when the type set is genuinely unknown at compile time.

## Modern alternatives (Go 1.18+)

| Instead of... | Prefer... |
|---|---|
| `func Sum(s interface{})` + reflect to iterate | `func Sum[T Numeric](s []T) T` |
| `reflect.DeepEqual` for struct comparison | `github.com/google/go-cmp/cmp.Equal` (still reflection, but explicit and configurable) |
| Reflection-based config validation | Struct tags + a validator library, or a handwritten validator |
| Reflection-based copying between known types | Explicit assignment or `go generate` code |
| `interface{}` + type switch as a sum type | A closed interface with a sealed method |

## Review checklist

- [ ] Could a type parameter (generic) replace this reflection?
- [ ] Could a small interface replace it?
- [ ] Is every kind-specific call preceded by a `Kind()` check?
- [ ] Is every `Set*` call preceded by a `CanSet()` / `IsValid()` check?
- [ ] Are potential panics caught or documented? Is this code behind a `recover` boundary?
- [ ] Is the reflection code on a hot path? If yes, is there a benchmark and a type/field cache?
- [ ] Are struct-tag lookups done per call or cached?
- [ ] Are `FieldByName` calls in a loop replaced with indexed `Field(i)` lookups?
- [ ] Is there a comment explaining *why* reflect was the only option?
- [ ] Is there a test that exercises the unhappy path (wrong kind, invalid value, unexported field)?

## Suggested finding phrasing

- "`reflect.ValueOf(x).Int()` with no `Kind()` check — panics at runtime for non-int kinds; add a guard."
- "`FieldByName` inside a hot loop — cache the field index with `Field(i)` instead."
- "This function reflects over `interface{}` to sum numbers — replace with `Sum[T Numeric]` (Go 1.18+)."
- "No `CanSet()` check before `v.SetString(...)` — will panic on unexported fields."
- "Reflection used to copy between two known struct types — just assign the fields directly."
- "`reflect.DeepEqual` on structs containing function fields — will panic; switch to `cmp.Equal` with explicit options."
- "`reflect.TypeOf(x)` recomputed on every call — cache it at package init."

## When NOT to apply (reflection is fine)

- `encoding/json` and friends operating on genuinely dynamic data.
- Test equality helpers (`reflect.DeepEqual`, `cmp.Equal`).
- Framework-level code whose whole purpose is to be type-agnostic.
- Interop with dynamic data (YAML/JSON with unknown schema, CLI argument decoding, ORM hydration).

## Sources

- <https://go.dev/blog/laws-of-reflection> — Rob Pike, *The Laws of Reflection* (primary source for the three laws and the "avoided unless strictly necessary" quote)
- <https://pkg.go.dev/reflect> — `reflect` package documentation (panic semantics, kind checks, settability rules)
