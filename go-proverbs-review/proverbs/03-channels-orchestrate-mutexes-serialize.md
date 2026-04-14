---
proverb: "Channels orchestrate; mutexes serialize."
number: 3
category: actionable
sources:
  - https://go.dev/wiki/MutexOrChannel
  - https://go.dev/doc/effective_go (Concurrency)
  - https://pkg.go.dev/sync
---

# #3 — Channels orchestrate; mutexes serialize

## What it means

Channels and `sync.Mutex` are not interchangeable tools with different syntax — they solve different problems.

- **Channels orchestrate.** They move ownership of data between goroutines, distribute work, and signal completion. Use them when data *flows*.
- **Mutexes serialize.** They protect shared in-memory state (caches, counters, config) where multiple goroutines need concurrent read/write access to the same structure. Use them when data *sits*.

Picking the wrong tool produces either over-complex channel topologies or sprawling lock hierarchies. The proverb is a reminder to match the tool to the shape of the problem.

From the Go Wiki (*Use a sync.Mutex or a channel?*):

> Use whichever is most expressive and/or most simple.
>
> A common Go newbie mistake is to over-use channels and goroutines just because it's possible, and/or because it's fun. Don't be afraid to use a `sync.Mutex` if that fits your problem best.

And the guiding table:

| Channel | Mutex |
|---|---|
| passing ownership of data | caches |
| distributing units of work | state |
| communicating async results | |

Plus the decision rule:

> If you ever find your sync.Mutex locking rules are getting too complex, ask yourself whether using channel(s) might be simpler.

## What to look for (code smells)

- Complex locking rules: nested locks, documented lock ordering, a single `sync.Mutex` locked in 5+ places across a file → probably should be channels.
- Channels used for a trivial shared counter or boolean flag → probably should be a mutex or `sync/atomic`.
- `chan struct{}` used purely as a semaphore for a single value that a `sync.Mutex` would express more plainly.
- Goroutines using a mutex to hand work to each other (`mu.Lock(); queue = append(queue, item); mu.Unlock()`) — that's channel territory.
- A cache or lookup table guarded by channel send/receive instead of `sync.RWMutex` — usually slower and less clear.
- Read-heavy shared state behind a regular `sync.Mutex` where `sync.RWMutex` would be a better fit.

## Decision matrix

| Use a channel when... | Use a mutex when... |
|---|---|
| Passing ownership of data between goroutines | Protecting a cache |
| Distributing units of work | Protecting shared state (config, counters) |
| Communicating async results | Reference counts |
| Signaling completion or cancellation | Short, localized critical sections |
| Implementing a pipeline | Read-mostly data (`sync.RWMutex`) |

## Review checklist

- [ ] Is this channel really orchestrating data flow, or is it masquerading as a mutex?
- [ ] Is this mutex protecting genuine shared state, or is it masquerading as a channel?
- [ ] Are the locking rules simple enough to hold in your head? If not → channels.
- [ ] Is the channel topology simple enough to draw? If not → is the underlying problem actually "protect this state"?
- [ ] Could `sync.RWMutex` or `sync/atomic` replace a clumsy channel dance?
- [ ] Are channels and mutexes *combined* on the same state (almost always a smell)?

## When NOT to apply

The wiki itself is explicit: *"Use whichever is most expressive and/or most simple."* If both work and one reads more naturally for the problem, that's the right choice — the proverb is a heuristic, not a rule. Also note that channel communication, mutexes, and wait-groups are complementary and can be combined in the same program.

## Suggested finding phrasing

- "`cache` is guarded by a `chan struct{}` semaphore; a `sync.RWMutex` would express read-mostly access more clearly."
- "Nested locks on `aMu`/`bMu` with documented ordering — consider expressing this as a channel pipeline so ownership flows instead."
- "`counter int64` guarded by `sync.Mutex`; `sync/atomic.Int64` is simpler and faster for this use."

## Sources

- <https://go.dev/wiki/MutexOrChannel> — Go Wiki: "Use a sync.Mutex or a channel?" (primary source for the decision table)
- <https://go.dev/doc/effective_go#concurrency> — Effective Go, Concurrency section
- <https://pkg.go.dev/sync> — `sync` package documentation (`Mutex`, `RWMutex`, `WaitGroup`)
