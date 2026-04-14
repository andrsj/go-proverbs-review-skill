---
proverb: "Clear is better than clever."
number: 13
category: actionable
sources:
  - https://google.github.io/styleguide/go/guide
  - https://google.github.io/styleguide/go/decisions
  - https://go.dev/doc/effective_go
---

# #13 — Clear is better than clever

## What it means

Go culture prioritises *readability* — what the next developer (including you in six months) can understand at a glance — over compactness, cleverness, or minimal character count. When a clear three-line version and a clever one-liner both work, pick the three-line version.

The Google Go Style Guide puts clarity first in the core-principles hierarchy:

1. **Clarity** — the code's purpose and rationale is clear to the reader
2. **Simplicity** — accomplishes goals in the simplest way
3. **Concision** — high signal-to-noise ratio
4. **Maintainability** — easily maintained
5. **Consistency** — aligns with the broader codebase

Direct quotes:

> Clarity is primarily achieved with effective naming, helpful commentary, and efficient code organization.

> It is more important that code be easy to read than easy to write.

> We strive for a codebase that avoids unnecessary complexity so that when complexity does appear, it indicates that the code in question requires care.

And explicitly:

> [Simplicity] may often be mutually exclusive with 'clever' code.

## What to look for (code smells)

- Dense one-liners combining three or more operations (type assertion, arithmetic, and function call all in one expression).
- Clever bit manipulation where a standard-library call would be obvious.
- Functions named `do`, `run`, `handle`, `process` without clarifying context.
- Single-letter variable names outside small scopes (`i`, `j`, `err`, `ctx` are fine; `x` across a 40-line function is not).
- Abbreviated identifiers (`usrMgr`, `cfgSvc`, `dbConn`) that save typing but cost reading. The style guide: *"Do not simply drop letters to save typing. For example `Sandbox` is preferred over `Sbx`, particularly for exported names."*
- Map-indexed "ternary" hacks: `map[bool]int{true: 1, false: 0}[cond]`.
- Unnecessary reflection or `interface{}` where a concrete type would serve (see #6, #14).
- Comments that explain *how* a clever trick works instead of just writing the code clearly.
- Chained method calls across 5+ lines where intermediate variables would name the steps.
- `if-else` trees instead of early returns (pyramid of doom).
- Recursive closures defined inline where a named helper would be clearer.
- Reusing one variable for multiple unrelated purposes across a function.
- "Stringly-typed" APIs where a named type would document intent.
- Relying on operator precedence instead of parentheses (`a && b || c && d`).

## Good patterns

### Early returns

```go
if err != nil {
    return err
}
// happy path continues unindented
```

The style guide: *"Code that runs if the terminal condition is not met should appear after the `if` block, and should not be indented in an `else` clause."*

### Named intermediate steps

Instead of:

```go
return strings.TrimSpace(strings.ToLower(strings.Split(s, ",")[0]))
```

prefer:

```go
parts := strings.Split(s, ",")
first := strings.ToLower(parts[0])
return strings.TrimSpace(first)
```

### Named types for intent

```go
type UserID int64
type OrderID int64

func Fetch(id UserID) (*User, error) { … }
```

beats passing `int64` everywhere and risking argument mix-ups.

### Scope-proportional names

The style guide: *"The length of a name should be proportional to the size of its scope and inversely proportional to the number of times that it is used within that scope."*

`i` inside a three-line loop: fine. `i` as a field on a long-lived struct: not fine.

## Review checklist

- [ ] Can a reader understand this in one pass without running it mentally?
- [ ] Would a three-line version be clearer than this one-liner?
- [ ] Do variable names describe *what they hold*, not just *what type they are*?
- [ ] Are abbreviations justified by repeated use in a small scope, or just saving keystrokes?
- [ ] Are we using early returns to flatten nesting?
- [ ] Are operator-precedence assumptions made explicit with parentheses?
- [ ] Does the clever trick buy measurable performance, or just look smart?
- [ ] If the code needs a comment to explain how it works, could the code itself be made clearer?
- [ ] Is the function named for what it does, not how it's implemented?
- [ ] Are there named types where the primitive alone hides intent?

## When NOT to apply (i.e., "clever" is justified)

- Hot paths where a benchmark proves the clever version is substantially faster **and** the code is well-commented.
- Stdlib-style primitives (`math/bits`, `math.Float64bits`) where cleverness is the whole point of the package.
- Idiomatic patterns every Go reader recognises: `_, ok := m[key]`, `x, y = y, x`, `defer f.Close()`.
- Generated code where readability targets the generator, not the reader.

## Suggested finding phrasing

- "`return strings.TrimSpace(strings.ToLower(strings.Split(s, \",\")[0]))` — extract intermediate variables to name each step."
- "Nested `if` pyramid 4 levels deep — invert conditions and return early."
- "`usrMgr` / `cfgSvc` abbreviations — prefer `userManager` / `configService` per the style guide."
- "Function named `do` at package scope — rename to describe what it does."
- "`x := map[bool]int{true: 1, false: 0}[cond]` — use an `if` statement."
- "`i` used across a 60-line function — rename to something scope-appropriate."

## Sources

- <https://google.github.io/styleguide/go/guide> — Google Go Style Guide, core principles (Clarity #1, Simplicity #2, *"easy to read than easy to write"*)
- <https://google.github.io/styleguide/go/decisions> — Google Go Style Guide decisions (naming rules, early returns, operator precedence)
- <https://go.dev/doc/effective_go> — Effective Go (naming, control flow, and idiomatic patterns)
