---
name: review
description: Use when reviewing Go code against Rob Pike's Go Proverbs. Activates on explicit request ("review with Go proverbs", "/go-proverbs-review:review") or whenever the user asks to review `.go` files. Flags violations and groups findings by proverb, each tied to a detailed rule file in `proverbs/`.
---

# Go Proverbs Review

Code review skill based on Rob Pike's 19 Go Proverbs (<https://go-proverbs.github.io/>), delivered as a short mapping from each proverb to a detailed rule file under `proverbs/`.

## When to use

- User explicitly asks for a "Go proverbs review" or types `/go-proverbs-review:review`
- User asks to review Go (`.go`) code — apply all `actionable` proverbs
- User asks about a specific proverb — load the matching file directly

## How to use

Reviews run as a **parallel fan-out of subagents**, one per actionable proverb. This keeps the main context small and each check focused on a single principle.

### Full review

1. Identify the Go files in scope and hold only their paths in the main thread.
2. For each `actionable` proverb in the index, dispatch a subagent with:
   - The proverb number, short title, and path to `proverbs/NN-*.md`
   - The list of `.go` file paths to review
   - Instruction: load only that one rule file, read the code, return findings in the format `<file>:<line> — <one-sentence observation>` or the literal string `no findings`
3. Dispatch all subagents **in parallel** (single message, multiple tool calls).
4. For `tooling` proverbs, dispatch a single subagent that verifies the tool is configured (e.g. `golangci-lint` config present) and returns pass/fail — no code findings.
5. Skip `philosophical` and `postponed` proverbs entirely — they inform judgment, not findings.
6. Aggregate all subagent results, group by proverb, and produce the final report (see *Output format*).

### Partial review

If the user asks to check against specific proverbs ("review for #4 and #15 only"), dispatch only those subagents. Same fan-out pattern, smaller set.

### Single-proverb question

If the user asks about one proverb specifically ("explain #8" / "does this code violate #13?"), skip the subagent fan-out: load the single rule file directly in the main thread and answer.

### Why this pattern

- Main context never holds more than one or two rule files at a time
- Each subagent reasons about exactly one principle — fewer false positives
- Parallel execution keeps wall-clock time low
- The user can always ask for a partial review to narrow scope further

## Categories

| Category | Meaning |
|---|---|
| `actionable` | Generates findings during review |
| `tooling` | Verify the tool is wired up, don't flag code |
| `philosophical` | Informs judgment, no findings emitted |

## Proverb index

| # | Proverb | Category | Rule file |
|---|---------|----------|-----------|
| 1 | Don't communicate by sharing memory, share memory by communicating | actionable | [proverbs/01-share-memory-by-communicating.md](proverbs/01-share-memory-by-communicating.md) |
| 2 | Concurrency is not parallelism | philosophical | [proverbs/02-concurrency-is-not-parallelism.md](proverbs/02-concurrency-is-not-parallelism.md) |
| 3 | Channels orchestrate; mutexes serialize | actionable | [proverbs/03-channels-orchestrate-mutexes-serialize.md](proverbs/03-channels-orchestrate-mutexes-serialize.md) |
| 4 | The bigger the interface, the weaker the abstraction | actionable | [proverbs/04-bigger-interface-weaker-abstraction.md](proverbs/04-bigger-interface-weaker-abstraction.md) |
| 5 | Make the zero value useful | actionable | [proverbs/05-make-the-zero-value-useful.md](proverbs/05-make-the-zero-value-useful.md) |
| 6 | interface{} says nothing | actionable | [proverbs/06-interface-any-says-nothing.md](proverbs/06-interface-any-says-nothing.md) |
| 7 | Gofmt's style is no one's favorite, yet gofmt is everyone's favorite | tooling | [proverbs/07-gofmt-is-everyones-favorite.md](proverbs/07-gofmt-is-everyones-favorite.md) |
| 8 | A little copying is better than a little dependency | actionable | [proverbs/08-little-copying-better-than-little-dependency.md](proverbs/08-little-copying-better-than-little-dependency.md) |
| 9 | Syscall must always be guarded with build tags | actionable | [proverbs/09-syscall-must-be-guarded-with-build-tags.md](proverbs/09-syscall-must-be-guarded-with-build-tags.md) |
| 10 | Cgo must always be guarded with build tags | actionable | [proverbs/10-cgo-must-be-guarded-with-build-tags.md](proverbs/10-cgo-must-be-guarded-with-build-tags.md) |
| 11 | Cgo is not Go | philosophical | [proverbs/11-cgo-is-not-go.md](proverbs/11-cgo-is-not-go.md) |
| 12 | With the unsafe package there are no guarantees | actionable | [proverbs/12-unsafe-has-no-guarantees.md](proverbs/12-unsafe-has-no-guarantees.md) |
| 13 | Clear is better than clever | actionable | [proverbs/13-clear-is-better-than-clever.md](proverbs/13-clear-is-better-than-clever.md) |
| 14 | Reflection is never clear | actionable | [proverbs/14-reflection-is-never-clear.md](proverbs/14-reflection-is-never-clear.md) |
| 15 | Errors are values | actionable | [proverbs/15-errors-are-values.md](proverbs/15-errors-are-values.md) |
| 16 | Don't just check errors, handle them gracefully | actionable | [proverbs/16-handle-errors-gracefully.md](proverbs/16-handle-errors-gracefully.md) |
| 17 | Design the architecture, name the components, document the details | actionable | [proverbs/17-design-architecture-name-components-document-details.md](proverbs/17-design-architecture-name-components-document-details.md) |
| 18 | Documentation is for users | actionable | [proverbs/18-documentation-is-for-users.md](proverbs/18-documentation-is-for-users.md) |
| 19 | Don't panic | actionable | [proverbs/19-dont-panic.md](proverbs/19-dont-panic.md) |

## Output format

Group findings under each violated proverb:

```
### Proverb #N — <short title>
- path/to/file.go:42 — <one-sentence finding>
- path/to/file.go:87 — <one-sentence finding>
  See: proverbs/NN-<slug>.md
```

End the review with a summary line: total findings, number of distinct proverbs violated, files touched.
