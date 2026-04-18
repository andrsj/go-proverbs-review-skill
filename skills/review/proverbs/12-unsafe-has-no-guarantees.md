---
proverb: "With the unsafe package there are no guarantees."
number: 12
category: actionable
sources:
  - https://pkg.go.dev/unsafe
---

# #12 — With the unsafe package there are no guarantees

## What it means

The `unsafe` package steps around Go's type system. From the package docs:

> Package unsafe contains operations that step around the type safety of Go programs.
>
> Packages that import unsafe may be non-portable and are not protected by the Go 1 compatibility guidelines.

That's the proverb in a sentence — nothing you do with `unsafe.Pointer` is guaranteed to keep working across Go versions, GOARCH, GOOS, or even compiler changes. The six "valid patterns" documented on `pkg.go.dev/unsafe` are the **only** shapes the compiler and `go vet` treat as legitimate. Anything else is likely already broken or will be broken by a future release.

The proverb has two components:

1. **No Go 1 compatibility guarantee** — your code can stop working on the next Go release.
2. **No portability guarantee** — sizes, alignments, and layouts vary by GOARCH/GOOS.

## What to look for (code smells)

- `unsafe.Pointer` used outside the six valid patterns.
- Storing a `uintptr` in a variable **between** `Pointer → uintptr → Pointer` conversions — the GC can reclaim or move the object in between.
- `uintptr` arithmetic that walks off the end of an allocated object (not allowed, even `&s + sizeof(s)`).
- `reflect.SliceHeader` / `reflect.StringHeader` declared as standalone values. Since Go 1.20 these are effectively deprecated; use `unsafe.Slice`, `unsafe.String`, `unsafe.StringData`, `unsafe.SliceData` instead.
- Struct field access via `unsafe.Offsetof` + pointer arithmetic where normal field access works.
- `*(*T)(unsafe.Pointer(&x))` type-punning between types of different sizes.
- Bypassing safety "for performance" without benchmarks to justify it.
- `unsafe` used to reach into unexported fields or cross package boundaries.
- Missing `go vet` in CI (it catches many unsafe misuses, though silence is not a guarantee).
- `unsafe` files with no comment explaining *why* unsafe was necessary.
- Unsafe code without tests that explicitly exercise the unsafe path.

## The six valid patterns (summary)

1. **`*T1` ↔ `Pointer` ↔ `*T2`** when `T2` is no larger than `T1` and memory layouts match (e.g., `math.Float64bits`).
2. **`Pointer → uintptr`** (but not back) — the `uintptr` is a plain integer, has no pointer semantics, and the GC does not track it. Usual use: printing the address.
3. **`Pointer → uintptr → Pointer` with arithmetic** — must be in a **single expression**, pointer must remain within the original allocated object. Cannot walk past the end, even by one byte.
4. **`Pointer → uintptr` when calling `syscall.Syscall`** — the conversion must appear in the call argument list itself; the compiler arranges to keep the referent alive during the call.
5. **Converting `reflect.Value.Pointer()` / `UnsafeAddr()` result** to `Pointer` — must happen in the same expression as the reflect call.
6. **`reflect.SliceHeader` / `reflect.StringHeader` `Data` field** — only when pointing at a real slice/string via `*reflect.SliceHeader`, never as a standalone struct. Now superseded by `unsafe.Slice` / `unsafe.String`.

> [!WARNING]
> Silence from `go vet` is not a guarantee that unsafe code is valid. These patterns are the contract with the compiler, not a lint rule.

## Modern replacements (Go 1.17+)

| Older pattern | Modern replacement |
|---|---|
| `reflect.SliceHeader{Data: …, Len: …, Cap: …}` | `unsafe.Slice(ptr, len)` |
| `reflect.StringHeader{Data: …, Len: …}` | `unsafe.String(ptr, len)` |
| Extracting the pointer from a string | `unsafe.StringData(s)` |
| Extracting the pointer from a slice | `unsafe.SliceData(s)` |
| Manual `uintptr` arithmetic | `unsafe.Add(ptr, len)` |

Prefer these over manual header or pointer arithmetic. They are type-safe at the language level and recognised by the compiler.

## Canonical valid example (from the docs)

```go
// math.Float64bits — pattern 1
func Float64bits(f float64) uint64 {
    return *(*uint64)(unsafe.Pointer(&f))
}
```

## Canonical invalid examples (from the docs)

```go
// INVALID: uintptr stored in a variable between conversions
u := uintptr(unsafe.Pointer(p))
p = unsafe.Pointer(u + offset)

// INVALID: end points outside allocated space
var s thing
end = unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Sizeof(s))

// INVALID: standalone reflect.StringHeader — Data is not a reference
var hdr reflect.StringHeader
hdr.Data = uintptr(unsafe.Pointer(p))
hdr.Len = n
s := *(*string)(unsafe.Pointer(&hdr))
```

## Review checklist

- [ ] Is this use of `unsafe` necessary? What concrete problem would be unsolvable without it?
- [ ] Does it match one of the six valid patterns documented in `pkg.go.dev/unsafe`?
- [ ] Does `go vet` pass on this file? Is `go vet` wired into CI?
- [ ] Is every `Pointer → uintptr → Pointer` conversion contained in a single expression, with no intermediate variable?
- [ ] Does the pointer always stay within its original allocated object?
- [ ] Could `unsafe.Slice`, `unsafe.String`, `unsafe.Add`, `unsafe.StringData`, `unsafe.SliceData` replace manual header / arithmetic code?
- [ ] Are `reflect.SliceHeader` / `reflect.StringHeader` used? They're effectively deprecated — migrate.
- [ ] Is there a comment explaining *why* `unsafe` was the only option?
- [ ] Is there a benchmark demonstrating the performance win (if that was the reason)?
- [ ] Are there tests covering the unsafe code path?
- [ ] Does the code assume a specific GOARCH (word size, alignment)? If yes, is it guarded?

## When NOT to apply (i.e., `unsafe` *is* acceptable)

- Reading low-level type representations (`math.Float64bits`, `math.Float32bits`).
- Zero-copy conversions between `[]byte` and `string` where the input is genuinely immutable — with extreme care.
- Interop with syscalls / cgo where pointers must be passed as `uintptr` in the call expression itself.
- Reflection-heavy libraries (encoding, ORMs, validators) where the ecosystem cost is accepted and the code is well-tested.
- Tight loops where a benchmark proves `unsafe` is justified and the scope is small and contained.

## Suggested finding phrasing

- "`unsafe.Pointer` arithmetic walks past `&s + Sizeof(s)` — this is explicitly listed as invalid in the unsafe docs."
- "`u := uintptr(unsafe.Pointer(p))` stored before conversion back — invalid, the GC can move or free `p` between the two lines."
- "`reflect.StringHeader` declared as a standalone struct; migrate to `unsafe.String(ptr, len)` (Go 1.17+)."
- "`unsafe.Pointer` used to read an unexported field — this is intentionally not a valid pattern and will likely break."
- "No `// WHY unsafe:` comment on this file; the reason is not obvious — either justify it or remove the import."
- "`go vet` is not run in CI and this file uses `unsafe` — add the vet step."

## Sources

- <https://pkg.go.dev/unsafe> — `unsafe` package documentation, including the Go 1 compatibility disclaimer and the six valid Pointer patterns (primary source for every quote and invalid example above)
