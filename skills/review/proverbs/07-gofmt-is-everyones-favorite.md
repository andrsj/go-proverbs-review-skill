---
proverb: "Gofmt's style is no one's favorite, yet gofmt is everyone's favorite."
number: 7
category: tooling
sources:
  - https://go.dev/blog/gofmt
  - https://pkg.go.dev/cmd/gofmt
  - https://golangci-lint.run/
---

# #7 — Gofmt's style is no one's favorite, yet gofmt is everyone's favorite

## What it means

Go doesn't have style debates because everyone runs the same formatter. No one picked tabs, 80-column wrapping, or `func foo() {` vs `func foo()\n{` — `gofmt` did, and the argument ended. The payoff is that every Go file anywhere looks the same, and no human time is ever spent on formatting reviews.

The proverb is **tooling**, not code: there is no per-file smell to detect. The reviewer's job is to confirm the tool is wired up so that all future code arrives already formatted.

## What to verify

Check the repo for at least one of these, in descending order of strength:

- **CI step** that runs `gofmt -l .` (or `goimports -l .`) and fails on non-empty output.
- **CI step** that runs `golangci-lint run` with `gofmt` / `goimports` / `gofumpt` enabled.
- **Pre-commit hook** (`.pre-commit-config.yaml`, `.githooks/`, `lefthook.yml`, Husky) that formats or checks on commit.
- **`Makefile` / task runner target** that runs the formatter as part of a standard workflow.
- **Editor config** (`.editorconfig`, `.vscode/settings.json`, `.idea/*.xml`) that enables format-on-save.
- **`gofumpt`** configured (stricter superset of `gofmt`) — bonus, not required.

If *none* of the above is present, emit one finding: *"No automated formatter enforcement found — add a CI check or pre-commit hook so `gofmt` / `goimports` runs on every change."*

If at least one is present, emit `no findings`.

## How to use this file

- This is the single subagent for the `tooling` category mentioned in `SKILL.md`.
- The subagent does **not** walk `.go` files looking for formatting violations — that's `gofmt` itself. It only verifies that the enforcement exists.
- Running `gofmt -l .` locally is a quick sanity check; if it outputs file paths, the formatter is not being enforced anywhere, which is itself the finding.

## Suggested finding phrasing

- "No CI or pre-commit formatter check found. Add `gofmt -l .` (or `golangci-lint run`) to CI and fail on non-empty output."
- "CI runs tests but not formatting — add a `gofmt` step or enable the `gofmt` / `goimports` linter in `golangci-lint`."
- "`.editorconfig` enforces tabs but there's no project-level formatter check — please add one so non-editor-config users can't bypass it."
- "`gofmt -l .` locally reports `N` unformatted files; the formatter is effectively not enforced."

## Sources

- <https://go.dev/blog/gofmt> — Andrew Gerrand, *"go fmt your code"*
- <https://pkg.go.dev/cmd/gofmt> — `gofmt` command reference
- <https://golangci-lint.run/> — `golangci-lint` (the usual way to enforce `gofmt`/`goimports` in CI)
