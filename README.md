# go-proverbs-review

A Claude Code skill that reviews Go code against [Rob Pike's Go Proverbs](https://go-proverbs.github.io/).

Each proverb is its own rule file with motivation, code smells, good/bad examples, a review checklist, and primary sources (mostly [go.dev](https://go.dev/)). At review time, one subagent per actionable proverb runs in parallel, so each check is focused on a single principle and the main context stays small.

## Install

```sh
git clone https://github.com/andrsj/go-proverbs-review-skill.git
cp -R go-proverbs-review-skill/go-proverbs-review ~/.claude/skills/
```

Then in any Go project ask Claude Code to "review this with Go proverbs" (or review `.go` files normally — the skill activates automatically).

## What's inside

- `go-proverbs-review/SKILL.md` — the index + review workflow
- `go-proverbs-review/proverbs/NN-*.md` — one file per proverb (19 total)

## Categories

| Category | Count | Meaning |
|---|---|---|
| `actionable` | 15 | Generates findings during review |
| `tooling` | 1 | Verifies the tool is wired up (gofmt/golangci-lint), no code findings |
| `philosophical` | 3 | Informs judgment, no findings emitted |

## How a review works

1. Main agent identifies the `.go` files in scope.
2. For each actionable proverb, dispatch a subagent with the rule file and the file list.
3. Each subagent loads only its own rule file, reads the code, and returns findings as `<file>:<line> — <observation>` or the literal `no findings`.
4. Main agent groups results by proverb and produces the final report.

This keeps the main context small and each check focused. You can also ask for a partial review (e.g. "check only #4 and #15") or a single-proverb explanation.

## The 19 proverbs

See [`go-proverbs-review/SKILL.md`](go-proverbs-review/SKILL.md) for the full index with links to each rule file.

## Sources

Rule content is synthesized from official Go documentation where possible:

- [go.dev/doc/effective_go](https://go.dev/doc/effective_go)
- [go.dev/doc/comment](https://go.dev/doc/comment)
- [go.dev/blog](https://go.dev/blog) — especially *Errors are values*, *Laws of Reflection*, *Defer, Panic, and Recover*
- [go.dev/wiki/CodeReviewComments](https://go.dev/wiki/CodeReviewComments)
- [pkg.go.dev](https://pkg.go.dev/) — for `syscall`, `unsafe`, `reflect`, `errors`, `cmd/cgo`
- [Google Go Style Guide](https://google.github.io/styleguide/go/)
- [Dave Cheney's blog](https://dave.cheney.net/) — *Don't just check errors, handle them gracefully*, *cgo is not Go*

Every proverb file lists its sources at the bottom.

## License

MIT — see [LICENSE](LICENSE).
