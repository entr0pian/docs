# Pipeline Pattern in Go

## What is a Pipeline?

A pipeline is a series of stages connected by channels. Each stage is a function that:

1. Receives values from an **inbound** channel
2. Processes them
3. Sends results to an **outbound** channel

This lets you compose independent processing steps that run concurrently, each in its own goroutine.

```
source ──► stage 1 ──► stage 2 ──► stage 3 ──► sink
        chan      chan          chan          chan
```

---

## A Simple Pipeline

```go
package main

import "fmt"

// gen sends values into a channel and closes it when done.
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// sq receives integers, squares them, and sends the result.
func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

func main() {
    // Wire stages together.
    c := gen(2, 3, 4)
    out := sq(c)

    for v := range out {
        fmt.Println(v) // 4, 9, 16
    }
}
```

### What happens

```
gen goroutine         sq goroutine          main
─────────────         ────────────          ────
sends 2 ──────────► receives 2
                      squares → sends 4 ──► receives 4, prints
sends 3 ──────────► receives 3
                      squares → sends 9 ──► receives 9, prints
sends 4 ──────────► receives 4
                      squares → 16 ────── ► receives 16, prints
close(out)            range exits
                      close(out)            range exits
```

Each stage blocks until the next is ready — the pipeline naturally flows at the pace of the slowest stage.

---

## Closing Channels and `range`

The idiomatic way to signal "no more values" is to `close` the channel. A `range` loop over a channel exits cleanly when the channel is closed and drained.

```go
for v := range ch {   // exits when ch is closed and empty
    process(v)
}
```

> A stage must always close its outbound channel, otherwise the next stage blocks forever.

---

## Fan-Out and Fan-In

### Fan-Out

Distribute work across multiple goroutines reading from the same channel.

```go
c1 := sq(in)
c2 := sq(in)   // both goroutines read from the same 'in' channel
```

Each value is consumed by exactly one of the workers — Go's channel receive is race-safe.

### Fan-In (Merge)

Collect results from multiple channels into one.

```go
func merge(cs ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup

    output := func(c <-chan int) {
        for v := range c {
            out <- v
        }
        wg.Done()
    }

    wg.Add(len(cs))
    for _, c := range cs {
        go output(c)
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

The `WaitGroup` ensures `out` is closed only after every source channel has been fully drained.

```
worker 1 ──► merge goroutine ──► out
worker 2 ──►        │
worker 3 ──►        │
                wg.Wait() → close(out)
```

---

## Cancellation with `context`

A pipeline stage must stop when the consumer is no longer interested. Passing a `context.Context` is the standard way to propagate cancellation upstream.

```go
func sq(ctx context.Context, in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            select {
            case out <- n * n:
            case <-ctx.Done():
                return   // consumer cancelled — stop processing
            }
        }
    }()
    return out
}
```

The `select` races between "send result" and "context cancelled". Whichever case is ready first wins. If the consumer calls `cancel()`, all stages unblock and their goroutines exit cleanly.

---

## Key Rules

| Rule | Why |
|---|---|
| Always `close` the outbound channel | Downstream `range` loops must exit |
| Use `context` for cancellation | Avoids goroutine leaks when consumers stop early |
| Each stage owns its goroutine | Keeps stages independent and composable |
| Prefer directional channel types (`<-chan`, `chan<-`) | Makes data flow explicit at compile time |

---

## Relation to This Codebase

`apps/backend/internal/worker/` uses the same pattern: a goroutine reads from a channel
(`logQueue`), processes items, and exits when the channel is closed or the context is
cancelled. The future SQS worker (`TaskWorker`) will follow the same structure — polling
a queue maps cleanly onto the generator stage of a pipeline.
