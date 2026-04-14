---
proverb: "Documentation is for users."
number: 18
category: actionable
sources:
  - https://go.dev/doc/comment
  - https://go.dev/doc/effective_go (Commentary)
  - https://go.dev/wiki/CodeReviewComments (Doc Comments)
---

# #18 — Documentation is for users

## What it means

Doc comments exist for the **caller** — the person reading your package's godoc page or their IDE's hover tooltip. They should answer "what does this do, what can I expect, how do I use it?" — *not* "how is it implemented?". Internal comments explaining the algorithm belong inside the function body, not in the doc comment.

This proverb is the companion to #17: once you've decided to document every exported symbol, #18 says *write it from the user's side of the API, not the implementer's*.

From *Go Doc Comments*:

> Doc comments should not explain internal details such as the algorithm used in the current implementation. Those are best left to comments inside the function body.

Doc comments should cover: **special cases, asymptotic bounds, concurrency guarantees, zero-value behaviour, preconditions, error conditions, side effects.** Everything a caller needs to use the API correctly, nothing about how it's built.

## Canonical examples (from the stdlib / go.dev/doc/comment)

```go
// Quote returns a double-quoted Go string literal representing s.
func Quote(s string) string { ... }

// Exit causes the current program to exit with the given status code.
func Exit(code int) { ... }

// HasPrefix reports whether the string s begins with prefix.
func HasPrefix(s, prefix string) bool

// Sort sorts data in ascending order as determined by the Less method.
// It makes one call to data.Len to determine n and O(n*log(n)) calls to
// data.Less and data.Swap.
func Sort(data Interface) { ... }
```

Notice: `Sort`'s comment mentions **complexity and what it calls** — both caller-visible contract — but says nothing about *which* sorting algorithm. The user doesn't care about "introsort" vs "quicksort" — they care that it's O(n log n) and that it calls `Less`/`Swap`.

## What to look for (implementer's-view smells)

- "This function works by first doing X, then Y, then looping over Z..."
- Algorithm name or variant mentioned (`"uses Boyer-Moore"`, `"implements Raft"`) as if the caller needs to know.
- Data-structure internals described (`"maintains a red-black tree internally"`).
- References to private fields (`"sets c.state to busy"`).
- `TODO` / `FIXME` inside the doc comment.
- `"Changed in v2.1 because..."` — changelog belongs in `CHANGELOG.md`, not godoc.
- `"See also function foo()"` where `foo` is unexported.
- Commented-out code pretending to be documentation.
- Implementation warnings (`"this is slow, will be optimized later"`) with no user-visible contract.
- Stale comments describing behaviour that changed.
- Comments that say what the name already says: `// Get returns the value` on a method called `Get`.
- Over-qualified comments: `// Reader is a struct that represents...` — just say what it *does* for users.
- Mentions of internal goroutines, channels, or mutexes that the caller never touches.

## What doc comments *should* cover

All caller-visible:

- **What it does**, in one sentence (the first sentence appears on the package index page).
- **Return values** — including zero-value behaviour and when each is returned.
- **Error conditions** — what errors can be returned, how callers should branch.
- **Special cases** — `nil` input, empty slice, negative numbers, empty string.
- **Preconditions** — what the caller must arrange before calling.
- **Side effects** — mutation of arguments, I/O, global state.
- **Concurrency guarantees** — safe for concurrent use? Must be called from a single goroutine?
- **Asymptotic bounds** — O(n), O(log n), when non-obvious or contractual.
- **Links to related types/functions** via bare symbol names (godoc auto-links).

## Good vs bad

### Bad (implementer's view)

```go
// Sum uses Kahan summation internally to avoid floating point errors.
// It iterates through the slice and maintains a running compensation.
func Sum(nums []float64) float64
```

### Good (user's view)

```go
// Sum returns the sum of nums. It handles floating-point precision
// gracefully for long slices. For an empty slice, Sum returns 0.
func Sum(nums []float64) float64
```

### Bad

```go
// UserStore wraps a *sql.DB and caches user lookups in a sync.Map.
type UserStore struct { ... }
```

### Good

```go
// UserStore persists and retrieves User records. It is safe for
// concurrent use by multiple goroutines.
type UserStore struct { ... }
```

## Review checklist

- [ ] Does the doc comment explain what the caller gets, not how you built it?
- [ ] Is it phrased as a complete sentence starting with the symbol name?
- [ ] Does it mention special cases (nil, empty, zero)?
- [ ] If it can return errors, are they documented?
- [ ] Is concurrency-safety mentioned when non-obvious?
- [ ] If behaviour depends on inputs, is each case documented?
- [ ] Does the comment avoid private type names, internal field references, algorithm names?
- [ ] Is there a short `Example*` test for non-trivial APIs where it would help?
- [ ] Is the comment *still accurate* for the current code?
- [ ] For types: does it say what an instance *represents*, not how it's laid out in memory?

## Suggested finding phrasing

- "Doc comment on `Process` describes the algorithm ('uses a linked list internally'); move that to an internal comment and describe what callers get instead."
- "`Sum` doesn't document what it returns for an empty slice — add a sentence."
- "`Client` doc comment doesn't mention concurrency safety; add one line since callers need to know."
- "Comment references private field `c.state` — callers don't see that; describe observable behaviour instead."
- "Comment says `// Reader is a struct` — replace with what `Reader` does for a user."
- "`TODO` note inside doc comment — move to the function body or an issue tracker."
- "`// Get returns the value` — redundant with the name; either delete or add useful information (errors, zero value, concurrency)."

## When NOT to apply

- Internal/unexported implementation code can freely describe how it works.
- In-function comments (not doc comments) are the right place for algorithm notes.
- Package-level internal docs or `ARCHITECTURE.md` files can dive into implementation — they're not godoc.
- Non-library code (`main`, binaries) where "users" may really be maintainers — still favour clarity, but the proverb is softer.

## Relationship to other proverbs

- **#17** says *document*; **#18** says *document for the reader*. Together they produce good godoc.
- **#13** ("clear is better than clever"): a confusing implementation often produces a confusing doc comment. Simplify the code and the comment usually writes itself.

## Sources

- <https://go.dev/doc/comment> — *Go Doc Comments* (primary source; contains the "algorithm belongs in the function body" guidance and the stdlib examples above)
- <https://go.dev/doc/effective_go#commentary> — Effective Go, *Commentary*
- <https://go.dev/wiki/CodeReviewComments> — Go Code Review Comments, *Doc Comments* section (full-sentence / start-with-name rule)
