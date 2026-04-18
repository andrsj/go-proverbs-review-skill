# go-proverbs-review

A [Claude Code plugin](https://code.claude.com/docs/en/plugins) shipping a single skill that reviews Go code against [Rob Pike's Go Proverbs](https://go-proverbs.github.io/).

Each proverb is its own rule file with motivation, code smells, good/bad examples, a review checklist, and primary sources (mostly [go.dev](https://go.dev/)). At review time, one subagent per actionable proverb runs in parallel, so each check is focused on a single principle and the main context stays small.

## Install

This repo doubles as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) (`.claude-plugin/marketplace.json`) — so it's installable via the official `/plugin install` flow. Three install paths, in order of recommendation:

### Option 1 — Via the marketplace (recommended)

Inside any Claude Code session, run:

```text
/plugin marketplace add andrsj/go-proverbs-review-skill
/plugin install go-proverbs-review@andrsj-skills
/reload-plugins
```

The plugin is registered globally (user scope by default), available in every project. The skill appears namespaced as `/go-proverbs-review:review`. Updates flow automatically when you run `/plugin marketplace update andrsj-skills`.

### Option 2 — `--plugin-dir` (for development / live-edit)

If you've cloned the repo locally and want to iterate on skill files with instant feedback, start Claude Code with `--plugin-dir` pointing at the clone:

```bash
git clone https://github.com/andrsj/go-proverbs-review-skill.git
claude --plugin-dir ./go-proverbs-review-skill
```

Edits to any skill file pick up live with `/reload-plugins` — no re-install needed.

### Option 3 — Standalone copy (no namespace)

If you want the skill in your personal skills directory without the plugin namespace, copy *and rename* it so the slash command doesn't collide with other generic `/review` skills:

```bash
git clone https://github.com/andrsj/go-proverbs-review-skill.git
cp -R go-proverbs-review-skill/skills/review ~/.claude/skills/go-proverbs-review
# then edit ~/.claude/skills/review/SKILL.md — change `name: review` to `name: go-proverbs-review`
```

The skill then activates as `/go-proverbs-review` (un-namespaced), or automatically whenever you ask Claude Code to review `.go` files.

## What's inside

- `skills/review/SKILL.md` — the index + review workflow
- `skills/review/proverbs/NN-*.md` — one file per proverb (19 total)

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

See [`skills/review/SKILL.md`](skills/review/SKILL.md) for the full index with links to each rule file.

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

## Related

- Sister plugin/skill: [`google-go-styleguide`](https://github.com/andrsj/google-go-styleguide) — reviews against [Google's Go Style Guide framework](https://google.github.io/styleguide/go/). Different lens, complementary findings.

## License

MIT — see [LICENSE](LICENSE).
