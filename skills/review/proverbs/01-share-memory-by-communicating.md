---
proverb: "Don't communicate by sharing memory, share memory by communicating."
number: 1
category: actionable
sources:
  - https://go.dev/blog/codelab-share
  - https://go.dev/doc/effective_go (Concurrency → Share by communicating)
---

# #1 — Share memory by communicating

## What it means

Go discourages the traditional "mutex-protected shared state" concurrency model. Instead, goroutines pass values to each other through channels, and at any given moment exactly one goroutine owns a piece of data. Data races become impossible **by design**. The idea traces back to Hoare's Communicating Sequential Processes (CSP) and is the same mental model as Unix pipelines — communication *is* the synchronizer.

From *Effective Go*:

> Concurrent programming in many environments is made difficult by the subtleties required to implement correct access to shared variables. Go encourages a different approach in which shared values are passed around on channels and, in fact, never actively shared by separate threads of execution. Only one goroutine has access to the value at any given time. Data races cannot occur, by design. To encourage this way of thinking we have reduced it to a slogan: **Do not communicate by sharing memory; instead, share memory by communicating.**

## What to look for (code smells)

- Goroutines touching the same slice/map/struct field guarded by `sync.Mutex` where the data has a clear "producer → worker → consumer" flow.
- Structs that bundle data + `sync.Mutex` + boolean flags like `polling`, `inUse`, `busy`, `processing`. These flags are a tell that the code is reinventing queueing on top of locks.
- Passing pointers to mutable state into multiple goroutines when a channel of values would do.
- A single `sync.Mutex` locked in more than a handful of call sites across a file — ownership is unclear.
- `go func()` closures capturing shared variables by reference without any synchronization discipline.

## Bad example

From `go.dev/blog/codelab-share`:

```go
type Resource struct {
    url        string
    polling    bool
    lastPolled int64
}

type Resources struct {
    data []*Resource
    lock *sync.Mutex
}

func Poller(res *Resources) {
    for {
        // get the least recently-polled Resource
        // and mark it as being polled
        res.lock.Lock()
        var r *Resource
        for _, v := range res.data {
            if v.polling {
                continue
            }
            if r == nil || v.lastPolled < r.lastPolled {
                r = v
            }
        }
        if r != nil {
            r.polling = true
        }
        res.lock.Unlock()
        if r == nil {
            continue
        }

        // poll the URL

        // update the Resource's polling and lastPolled
        res.lock.Lock()
        r.polling = false
        r.lastPolled = time.Nanoseconds()
        res.lock.Unlock()
    }
}
```

Notice how much of the code is bookkeeping (`polling` flag, lock/unlock pairs, least-recently-used scan) instead of the actual work.

## Good example

```go
type Resource string

func Poller(in, out chan *Resource) {
    for r := range in {
        // poll the URL

        // send the processed Resource to out
        out <- r
    }
}
```

Ownership of each `*Resource` is explicit: it lives on the `in` channel, moves into `Poller` for processing, then leaves on `out`. No locks, no flags, no bookkeeping.

## When NOT to apply

Effective Go explicitly qualifies the proverb:

> This approach can be taken too far. Reference counts may be best done by putting a mutex around an integer variable, for instance. But as a high-level approach, using channels to control access makes it easier to write clear, correct programs.

So a mutex is the right tool when:

- Guarding a simple counter or reference count.
- Protecting a single-field atomic-like update (often better: `sync/atomic`).
- A hot path where channel send/receive overhead is measurable and unacceptable.
- Caches, pools, or data structures with no natural "flow" — data is queried, not passed along.

## Review checklist

- [ ] Is there shared mutable state accessed by more than one goroutine?
- [ ] Can the access pattern be expressed as "data flows through channels" (producer → worker → consumer) instead?
- [ ] If `sync.Mutex` is used, is it guarding *more* than a simple counter or flag?
- [ ] Do any structs carry state flags (`inUse`, `busy`, `processing`) that exist only because of the locking model?
- [ ] Could a `chan T` replace a `[]T + sync.Mutex` pair?
- [ ] Is goroutine ownership of data clear at every point in the code path?

## Suggested finding phrasing

- "`Resources` bundles data + mutex + `polling` flag; consider a channel-based pipeline so each resource has a single owner at any time."
- "Goroutines read/write `cache` under `mu`; if the access is producer→consumer, a `chan *Entry` would remove the lock entirely."
- "Mutex locked in 6 places across this file — ownership is unclear. Consider whether this state should flow through a channel instead."

## Sources

- <https://go.dev/blog/codelab-share> — "Share Memory By Communicating" (Andrew Gerrand) — origin of the Poller example, verbatim bad/good code
- <https://go.dev/doc/effective_go#sharing> — Effective Go, *Share by communicating*, with the CSP / Unix pipelines analogy and the reference-count caveat
- <https://go.dev/wiki/MutexOrChannel> — Go Wiki: "Use a sync.Mutex or a channel?" (complementary decision guide)
