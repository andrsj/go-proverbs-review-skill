---
proverb: "Design the architecture, name the components, document the details."
number: 17
category: actionable
sources:
  - https://go.dev/doc/effective_go (Names; Commentary)
  - https://go.dev/wiki/CodeReviewComments (Doc Comments; Package Names; Receiver Names; Variable Names; Interface Names; Package Comments)
  - https://go.dev/doc/comment (Go Doc Comments)
---

# #17 — Design the architecture, name the components, document the details

## What it means

The proverb is a three-step discipline for building Go packages:

1. **Design the architecture** — think about boundaries, data flow, and responsibilities *before* writing code. What are the components? How do they communicate? Where are the seams?
2. **Name the components** — pick names that are short, evocative, and honest. The names *are* the API. `bufio.Reader` is clearer than `BufferedInputReaderWrapper`.
3. **Document the details** — every exported symbol gets a doc comment in the canonical form (complete sentence, starts with the identifier name). Package-level docs describe the purpose.

This is where code becomes a library that others — including future-you — can actually use. Shape and clarity precede implementation detail, and all three steps are part of the same craft.

## Key guidance from the official sources

On package names (*Effective Go*):

> Packages are given lower case, single-word names; there should be no need for underscores or mixedCaps. Err on the side of brevity, since everyone using your package will be typing that name.
>
> The buffered reader type in the `bufio` package is called `Reader`, not `BufReader`, because users see it as `bufio.Reader`.
>
> The function to make new instances of `ring.Ring` would normally be called `NewRing`, but since `Ring` is the only type exported by the package [...] it's called just `New`, which clients of the package see as `ring.New`.

On interface names:

> By convention, one-method interfaces are named by the method name plus an -er suffix or similar modification to construct an agent noun: `Reader`, `Writer`, `Formatter`, `CloseNotifier`.

On getters:

> It's neither idiomatic nor necessary to put `Get` into the getter's name. If you have a field called `owner` (lower case, unexported), the getter method should be called `Owner`, not `GetOwner`. A setter function, if needed, will likely be called `SetOwner`.

On doc comments (*Go Code Review Comments*):

> All top-level, exported names should have doc comments, as should non-trivial unexported declarations. [...] Comments documenting declarations must be full sentences. Begin with the name of the thing being described. End with a period.

On package comments:

> Package comments must appear adjacent to the package clause with no blank line:
>
> ```go
> // Package math provides basic constants and mathematical functions.
> package math
> ```

On receiver names:

> Use one or two letter abbreviations reflecting the type [...]. Avoid generic names like `me`, `this`, `self`. Be consistent: if you use `c` in one method, use `c` in all methods of that type.

On avoiding meaningless package names:

> Avoid meaningless package names: `util`, `common`, `misc`, `api`, `types`, `interfaces`.

## What to look for (naming smells)

- Package named `util`, `common`, `helpers`, `misc`, `api`, `types`, `interfaces`, `pkg`.
- Package names with underscores or mixed case (`user_store`, `userStore`).
- Types repeating the package name: `user.UserService`, `user.UserModel`, `user.UserRepo`.
- `GetFoo()` where `Foo()` would read better.
- Generic types named `Manager`, `Handler`, `Service`, `Controller`, `Helper` without a clarifying noun.
- One-method interfaces **not** named with `-er` suffix.
- Inconsistent receiver names across methods of the same type (`c *Client` in one, `client *Client` in another).
- `me`, `this`, `self` as receiver names.
- `ServeHttp`, `XmlParse`, `urlPath`, `appId` — initialisms with wrong case. Should be `ServeHTTP`, `XMLParse`, `urlPath` → `URLPath`, `appID`.
- Long descriptive locals where short is better (`lineCounter` inside a 5-line loop).
- Very short names for distant-scope variables (`i` used 30 lines from its declaration).
- Abbreviations that save typing but lose meaning (`usrMgr`, `cfgSvc`, `dbConn`).

## What to look for (documentation smells)

- Exported symbols (types, functions, constants, vars) without doc comments.
- Doc comments that don't start with the symbol name: `// This function returns...`.
- Doc comments that are sentence fragments with no period.
- Package with no `// Package X ...` comment at the top of one file.
- `// Package main ...` missing or in the wrong form (should be `// Command X ...` or `// Binary X ...`).
- Doc comments describing *how* the code works internally instead of *what callers get* (see proverb #18).
- Out-of-date doc comments that contradict the code.
- Non-trivial unexported functions with no comment at all.
- Commented-out code pretending to be documentation.

## Good example

```go
// Package bufio implements buffered I/O. It wraps an io.Reader or io.Writer
// object, creating another object (Reader or Writer) that also implements
// the interface but provides buffering and some help for textual I/O.
package bufio

// Reader implements buffering for an io.Reader object.
type Reader struct { /* ... */ }

// NewReader returns a new Reader whose buffer has the default size.
func NewReader(rd io.Reader) *Reader { /* ... */ }
```

What it gets right:

- Package comment explains the purpose.
- Package name is a single lowercase word.
- `Reader` is short because `bufio.Reader` is how callers see it.
- Each exported symbol has a doc comment that starts with its name and is a full sentence.
- `NewReader` follows the `New*` constructor convention.

## Review checklist — architecture

- [ ] Does the package have a clear, stated purpose (not "utility functions")?
- [ ] Does the package name match its responsibility?
- [ ] Are the components (types, functions) cohesive — would a newcomer understand what lives here vs. elsewhere?
- [ ] Do seams exist where you'd want them (testing, replacement, extension)?

## Review checklist — naming

- [ ] Package name: single lowercase word, short, evocative, not `util` / `common` / `misc`?
- [ ] Types/functions don't repeat the package name?
- [ ] One-method interfaces use the `-er` convention?
- [ ] Getters named without `Get`?
- [ ] Receiver names are short, consistent, not `this`/`self`/`me`?
- [ ] Initialisms (`URL`, `ID`, `HTTP`, `XML`, `API`) have consistent case?
- [ ] Local names are proportional to scope (short for small scopes, longer for wide use)?
- [ ] Exported symbols have names the caller would write naturally?
- [ ] `Manager`, `Handler`, `Service`, `Controller` are clarified or renamed?

## Review checklist — documentation

- [ ] Every exported symbol has a doc comment?
- [ ] Each comment is a complete sentence starting with the symbol name?
- [ ] Non-trivial unexported declarations have a comment explaining *why*?
- [ ] Package has a `// Package X ...` comment (or `// Command X ...` for `main`)?
- [ ] Doc comments describe usage (caller perspective), not implementation (see #18)?
- [ ] No commented-out code pretending to be docs?

## Suggested finding phrasing

- "Package named `util` — rename to reflect what it actually does."
- "`user.UserService` repeats the package name; rename to `user.Service` so callers write `user.Service`."
- "`GetOwner()` — drop the `Get` prefix per Go convention: `Owner()`."
- "`this *Client` receiver — use `c *Client` and be consistent across methods."
- "Exported `func Process(...)` has no doc comment — add one starting with `Process ...`."
- "Doc comment `// This function does X` — rewrite as `// FuncName does X.`"
- "One-method interface `Closeable` — rename to `Closer` per the `-er` convention."
- "`appId` — initialism should be `appID`."
- "Package `common` with 12 unrelated helpers — move each helper to its closest real package."

## When NOT to apply

The rule is essentially always on. The one caveat: in a **test file** or **small internal tool**, the "doc comment for every exported symbol" rule can be relaxed — but even then, package naming and interface `-er` conventions still matter.

## Sources

- <https://go.dev/doc/effective_go#names> — Effective Go, *Names* (package, interface, getter, MixedCaps, receiver conventions)
- <https://go.dev/doc/effective_go#commentary> — Effective Go, *Commentary*
- <https://go.dev/doc/comment> — *Go Doc Comments* — canonical form and godoc extraction rules
- <https://go.dev/wiki/CodeReviewComments> — Go Code Review Comments (Doc Comments, Package Names, Receiver Names, Variable Names, Interface Names, Package Comments)
