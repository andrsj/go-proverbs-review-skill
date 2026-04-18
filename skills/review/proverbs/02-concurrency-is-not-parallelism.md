---
proverb: "Concurrency is not parallelism."
number: 2
category: philosophical
sources:
  - https://go.dev/blog/waza-talk
---

# #2 — Concurrency is not parallelism

## What it means

Concurrency is a way to *structure* a program — breaking work into independently-progressing parts that can be reasoned about separately. Parallelism is a way to *execute* — running multiple computations at the same instant on multiple cores. Go makes the first cheap and ergonomic (goroutines, channels, `sync`); the runtime may or may not turn it into the second depending on `GOMAXPROCS`, the work mix, and the scheduler.

Pike's point from the *Waza* talk (2012):

> Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once. Not the same, but related.

You can have:

- **Concurrency without parallelism** — a single CPU time-slicing between goroutines (most I/O-bound servers on a 1-core VM).
- **Parallelism without concurrency** — SIMD, GPU lanes, a single tight loop auto-vectorised by the compiler.
- **Both** — the typical Go server on a multi-core machine.

A well-structured concurrent program will often *also* run in parallel when the hardware allows — but parallelism is not the goal, it's a consequence.

## Why it's not a review rule

This proverb is a **mental model**, not a code smell detector. It shapes how you *design* with goroutines — whether you reach for them for structure (independent work, cancellation, deadlines) rather than as a reflex for speedup. Real "misuse of concurrency" bugs tend to surface as more specific issues that other tools and proverbs already catch:

- Goroutine leaks (missing cancellation / `context`) — static analysis, `go vet`, race detector
- Channel deadlocks — runtime panic, `go test -race`
- Over-spawning goroutines for CPU-bound work that doesn't scale — profiling
- Assuming sequential ordering across goroutines — `-race`
- Contention on shared state — pprof mutex profile
- Using channels where a mutex fits (and vice-versa) — covered by proverb #3

Because the smells are better served by those specific tools and proverbs, a code-review subagent for #2 would mostly produce false positives. This file exists so the mindset is documented for the reviewer to apply as context, not as a findings generator.

## How reviewers should use it

Use this as background when reasoning about any of the following:

- A PR that adds goroutines "for performance" — ask *which* of concurrency-as-structure or parallelism-as-execution is actually the goal, and whether the code needs to achieve it.
- A design discussion about whether to parallelise a sequential loop — a clean concurrent structure (channels, fan-out) is often a better starting point than parallel primitives.
- A benchmark that shows no speedup from goroutines — the structure may be fine; the hardware or work mix just doesn't parallelise.

If any of these turns into a concrete smell, the finding belongs under a different proverb (#1, #3, #5) rather than #2.

## Sources

- <https://go.dev/blog/waza-talk> — Rob Pike, *"Concurrency is not parallelism"* (the Waza talk slides and video, the canonical explanation)
