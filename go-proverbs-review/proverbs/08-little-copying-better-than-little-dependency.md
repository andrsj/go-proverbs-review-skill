---
proverb: "A little copying is better than a little dependency."
number: 8
category: actionable
sources:
  - https://go-proverbs.github.io/
  - https://dave.cheney.net/ (package design posts)
  - https://go.dev/ref/mod (Go modules reference)
---

# #8 — A little copying is better than a little dependency

## What it means

Adding an external dependency is never free. Every import brings:

- transitive dependencies you didn't choose
- a version pin you now have to track and update
- potential breaking changes across versions
- supply-chain risk (typo-squatting, account takeover, malicious updates)
- build-time cost and binary-size cost
- a contract someone else controls

If a package gives you only a handful of lines you actually use, copying those lines (with attribution + license compliance) is often the better trade — especially when the upstream package is small, unmaintained, or pulls in a heavy dep tree for one utility.

This lines up with Sandy Metz's maxim: *"Duplication is far cheaper than the wrong abstraction."* Pike's proverb is the dependency-level version of the same idea.

## What to look for (code smells)

- `go.mod` dependencies whose only usage is a single function (often 5-10 lines of real code).
- Packages imported that transitively pull in 20+ deps to provide a one-liner.
- "Convenience" libraries re-exporting the stdlib with minor changes.
- `slices` / `maps` / `strings` / `strconv` functionality reimplemented via a third-party dep, when the standard library now covers it (Go 1.21+).
- Dependencies added for testing helpers that a 10-line local helper would cover.
- Unmaintained dependencies (last commit years ago) still in `go.mod`.
- `replace` directives pointing at forks — a maintenance-tax signal.
- `go.sum` growing faster than the codebase itself.
- A single third-party import used in one file for one trivial operation.

## Decision heuristic

**Copy when:**

- The upstream package is small (say, under a few hundred lines you'd actually use).
- The code is stable and unlikely to need upstream fixes.
- The dep's license allows copying and you can attribute cleanly.
- The dep pulls in heavy or risky transitive deps.
- The upstream is unmaintained or has open security concerns.
- You only need one or two functions out of a much larger API.

**Depend when:**

- The code is complex, evolving, or security-critical (crypto, compression, parsers, TLS).
- Upstream fixes bugs and handles edge cases you don't want to track yourself.
- The dependency is well-maintained and widely audited.
- The functionality is domain-specific and correctness matters more than control.
- The dep is already transitively present in your graph anyway.

## Examples of the proverb in practice

- The Go standard library itself follows it: `net/http` doesn't depend on external HTTP utility libraries.
- `slices` and `maps` (added to stdlib in 1.21) were effectively "copied in" from `golang.org/x/exp` rather than leaving everyone to depend on the experimental module forever.
- Go's stdlib minimises vendored third-party code as a deliberate policy.

## Review checklist

- [ ] Does this dependency provide substantial value, or is it under ~50 lines of actual use?
- [ ] What transitive deps come along for the ride? (`go mod graph` / `go mod why`.)
- [ ] Is the dep actively maintained? Last release? Open issues?
- [ ] Is the functionality now in stdlib (`slices`, `maps`, `cmp`, `errors`, `log/slog`, `context`)?
- [ ] Could the same result be achieved with a short, well-commented local copy?
- [ ] If copying: is the license compatible and attribution present?
- [ ] Does the dep have a history of breaking changes?
- [ ] If this project has a `go.mod`, has it been audited recently for unused or single-use deps?

## When NOT to apply

- **Never copy cryptography, compression, parsers, or security-critical code** — don't roll your own, even "a little".
- Widely-used, well-maintained deps the whole ecosystem already pulls in.
- Code that needs ongoing updates (timezone data, TLS roots, CLDR data, Unicode tables).
- When license terms prohibit copying.
- When a security-advisory feed makes upstream tracking essential.
- Frameworks where reinventing the wheel would cost far more than tracking upstream (HTTP routers in non-trivial services, database drivers, protobuf/gRPC).

## Suggested finding phrasing

- "`go.mod` imports `github.com/foo/stringutil` for a single `ToTitleCase` call — copy the function (10 lines) and drop the dep."
- "`github.com/bar/slices-helper` is now redundant; `slices` in the standard library (Go 1.21+) covers this."
- "`pkg/util` imports three libraries for functionality each used once; consider inlining small copies."
- "Dependency last updated 4 years ago with open security issues — copy the needed function locally and remove the import."

## Sources

- <https://go-proverbs.github.io/> — Rob Pike's Go Proverbs (source of the proverb itself; the talk is the primary explanation)
- <https://dave.cheney.net/> — Dave Cheney's posts on Go package design, including the Sandy Metz "duplication vs. wrong abstraction" framing
- <https://go.dev/ref/mod> — Go modules reference (for `go mod graph`, `go mod why`, and dependency hygiene commands)

> [!NOTE]
> Unlike most proverbs on this list, this one is primarily cultural rather than documented on go.dev. The clearest evidence is in Go's own stdlib design philosophy: minimal external dependencies, vendored only when essential.
