# Channels and Select in Go

## What is a Channel?

A channel is a typed conduit that lets goroutines communicate by passing values to each other. The key principle in Go is:

> *Do not communicate by sharing memory; instead, share memory by communicating.*

Channels enforce synchronization at the point of data transfer — whoever sends and whoever receives must both be ready at the same time (for unbuffered channels).

```go
ch := make(chan int)   // declares a channel that carries int values
```

### Sending and Receiving

```go
ch <- 42      // send the value 42 into the channel
v := <-ch     // receive a value from the channel, assign it to v
<-ch          // receive and discard the value
```

---

## Unbuffered Channels

An unbuffered channel has no internal queue. It holds zero values at any point in time.

```go
ch := make(chan int)   // capacity = 0, unbuffered
```

### The Synchronization Contract

- A **send** (`ch <- v`) blocks until another goroutine is ready to receive.
- A **receive** (`<-ch`) blocks until another goroutine is ready to send.

Both sides must arrive at the handoff simultaneously. If either side is not ready, it waits. This is what makes unbuffered channels a synchronization primitive — they act like a rendezvous point between two goroutines.

```
Goroutine A: ch <- 42   ──► blocks until Goroutine B executes <-ch
Goroutine B: v := <-ch  ──► blocks until Goroutine A executes ch <- 42
```

### Buffered vs Unbuffered at a glance

| | Unbuffered | Buffered |
|---|---|---|
| Capacity | 0 | N (specified) |
| Send blocks when | no receiver is ready | buffer is full |
| Receive blocks when | no sender is ready | buffer is empty |
| Guarantees | sender knows receiver got the value | sender only knows value entered buffer |

---

## How `select` Works

`select` lets a goroutine wait on multiple channel operations simultaneously. It is similar to a `switch`, but each case is a channel send or receive.

```go
select {
case msg := <-ch1:
    // received from ch1
case ch2 <- val:
    // sent to ch2
case msg := <-ch3:
    // received from ch3
}
```

### Rules

1. **Blocking** — `select` blocks until at least one case can proceed.
2. **Random selection** — if multiple cases are ready at the same instant, Go picks one at random (no priority).
3. **Default case** — adding `default` makes select non-blocking; it runs immediately if no other case is ready.

```go
select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("no value ready")
}
```

---

## The Fibonacci Code

```go
package main

import "fmt"

func fibonacci(c, quit chan int) {
    x, y := 0, 1
    for {
        select {
        case c <- x:
            x, y = y, x+y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```

### Setup

```
main goroutine                   anonymous goroutine
──────────────────               ──────────────────────────────
c    := make(chan int)  ───────► shared unbuffered channel
quit := make(chan int)  ───────► shared unbuffered channel
go func() { ... }()    ──spawn─► goroutine starts, immediately blocks on <-c
fibonacci(c, quit)               (waiting for a value)
```

Two unbuffered channels are created. The anonymous goroutine is spawned and immediately hits `<-c` — it blocks, waiting for `fibonacci` to send.

`main` then calls `fibonacci` directly (no `go`), which enters its infinite loop and hits `select`.

---

## Step-by-Step Execution

### Iterations 1–10 (the counting loop)

At each iteration the two goroutines rendezvous on `c`:

```
fibonacci (main goroutine)           anonymous goroutine
──────────────────────────           ───────────────────
select evaluates its cases:
  case c <- x  ── can it proceed? ──► yes, <-c is waiting
                                        value received, printed
  (case <-quit is not ready)
x, y = y, x+y                        i++ → loop back → blocks on <-c again
```

The `select` unblocks only the `case c <- x` arm because the anonymous goroutine is always parked on `<-c`. The `quit` channel has no receiver, so that case is never selected during the first 10 iterations.

**Fibonacci values produced and printed:**

| Iteration (`i`) | `x` sent | `x,y` after update |
|---|---|---|
| 0 | 0 | 1, 1 |
| 1 | 1 | 1, 2 |
| 2 | 1 | 2, 3 |
| 3 | 2 | 3, 5 |
| 4 | 3 | 5, 8 |
| 5 | 5 | 8, 13 |
| 6 | 8 | 13, 21 |
| 7 | 13 | 21, 34 |
| 8 | 21 | 34, 55 |
| 9 | 34 | 55, 89 |

### Shutdown

After the 10th receive (`i = 9`), the loop ends and the anonymous goroutine executes:

```go
quit <- 0
```

This blocks until someone receives from `quit`. Meanwhile `fibonacci` is back in `select`:

```
fibonacci (main goroutine)           anonymous goroutine
──────────────────────────           ───────────────────
select evaluates its cases:
  case c <- x  ── can it proceed? ──► NO, nobody is doing <-c
  case <-quit  ── can it proceed? ──► YES, quit <- 0 is waiting
fmt.Println("quit")
return
```

`fibonacci` returns, `main` returns, the program exits.

---

## Why Unbuffered Channels Make This Work

The critical property here is that `fibonacci` never "runs ahead" of the consumer. Every send on `c` waits for the anonymous goroutine to receive. This means:

- No value is computed until it is needed.
- The producer (`fibonacci`) and consumer (anonymous goroutine) stay perfectly in step.
- The `quit` signal is guaranteed to arrive only after all 10 values have been consumed — because the loop uses the same `c` channel for synchronization throughout.

If `c` were buffered, `fibonacci` could compute and queue multiple values before the consumer reads them, and the shutdown sequencing would become harder to reason about.

---

## Reading from a Closed Channel and the `ok` Idiom

### What happens when you close a channel

Closing a channel signals that no more values will be sent. Any goroutine still waiting to receive unblocks immediately. After that, the channel can still be read from — it will keep returning the zero value for its type, forever, without blocking.

```go
ch := make(chan int)
close(ch)

v := <-ch         // returns 0 (zero value for int), does not block
v  = <-ch         // returns 0 again — this never stops
```

This means a goroutine that reads from a closed channel in a loop will spin indefinitely, consuming CPU while processing meaningless zero values. It will never block and never know the channel is exhausted.

### The two-value receive form

To distinguish a real value from a closed-channel zero value, use the two-value receive:

```go
v, ok := <-ch
```

- `ok == true` — a real value was sent by another goroutine; `v` holds it
- `ok == false` — the channel is closed and drained; `v` is the zero value

### Applying it in a pipeline stage

```go
case v, ok := <-in:
    if !ok {
        return   // upstream closed — nothing left to process
    }
    // v is a real value, process it
```

Without `!ok { return }`, a closed upstream would deliver an endless stream of zero values into the stage. With it, the goroutine exits cleanly, hits its `defer close(out)`, and propagates the closure downstream to the next stage.

### Two shutdown signals, one outcome

In a pipeline with a shared `done` channel, each stage has two independent reasons to stop:

| Signal | Meaning |
|---|---|
| `case <-done` | Orchestrator told everyone to stop |
| `ok == false` | My upstream already stopped |

Both should produce the same result: the goroutine returns and closes its own output channel. The `ok` check handles the **indirect** case — your upstream received `done`, exited, and closed its channel. By the time `ok=false` reaches you, `done` is already closed, so you could technically rely on `case <-done` on the next iteration. But the `ok` check is faster (fires immediately) and more importantly makes the stage **self-contained**.

### Why `ok` is the correct pattern regardless

A stage that checks `ok` does not need to know anything about `done` or the rest of the pipeline. It simply says: "if my input is gone, I stop." This means you can rewire stages arbitrarily — a stage could sit after a `cappedGenerator` that closes after N items with no `done` signal at all, and it would still shut down correctly.

```
close(done)
  → generator:       case <-done fires → returns → defer close(randNums)
  → primeFilter:     ok=false on randNums → returns → defer close(primes)
  → cappedGenerator: ok=false on primes → returns → defer close(out)
  → for range loop:  exits
```

Each stage cleans up because of its upstream, not because it knows about `done`. That cascade is what makes channel-based pipelines composable.

### `for range` as syntactic sugar for the `ok` idiom

`for range` over a channel is exactly equivalent to the two-value receive in a loop that breaks when `ok` is false:

```go
// these two are identical
for v := range ch { ... }

// expands to:
for {
    v, ok := <-ch
    if !ok {
        break
    }
    // ...
}
```

This means `for range` on a **closed and drained** channel exits immediately — not because of special closed-channel detection, but because the first receive returns `ok == false` and the loop breaks. There is no blocking, no spinning on zero values. A closed but non-empty channel is drained normally first; the loop exits only after the last real value has been consumed and the channel is empty.

This is why the final consumer in a pipeline is almost always a `for range` loop: it requires no explicit `ok` check yet still shuts down cleanly the moment its upstream closes.

---

## sync.Once

### What it does

`sync.Once` guarantees that a function is executed **exactly once**, regardless of how many goroutines call it or how many times it is invoked. After the first successful call, every subsequent call is a no-op.

```go
var once sync.Once
once.Do(func() { fmt.Println("runs only once") })
once.Do(func() { fmt.Println("never runs") })
once.Do(func() { fmt.Println("never runs") })
// output: runs only once
```

### Internal structure

```go
type Once struct {
    done atomic.Uint32  // 0 = not done, 1 = done
    m    Mutex
}
```

The `Do` method uses a two-layer check:

```
1. atomic load of done
   → if 1: return immediately (fast path, no lock)
   → if 0: acquire mutex

2. inside mutex, check done again (double-check locking)
   → if still 0: call f(), then atomically set done = 1
   → release mutex
```

The atomic check on the fast path means that after the first call, all subsequent calls cost only a single atomic read — no lock contention.

The mutex handles the race where two goroutines simultaneously see `done == 0`. Only one executes `f()`; the other blocks until `f()` completes, then sees `done == 1` and returns. This is the critical safety guarantee: **`Do` never returns until `f()` has fully finished**.

### The double-close problem

Closing a channel is a one-way, non-idempotent operation. Closing an already-closed channel causes an immediate runtime panic:

```
panic: close of closed channel
```

This becomes a problem when two independent code paths both need to close the same channel — for example, a signal handler and a `defer` in `main`:

```go
// BROKEN: panics if signal fires before main returns
done := make(chan struct{})
defer close(done)  // path A

go func() {
    <-sig
    close(done)    // path B — races with path A
}()
```

Whichever path runs second will panic.

### sync.Once as the fix

Wrap `close(done)` in a `sync.Once` and expose it as a reusable function:

```go
done := make(chan struct{})

var once sync.Once
closeDone := func() { once.Do(func() { close(done) }) }
defer closeDone()  // path A: normal exit

go func() {
    <-sig
    fmt.Println("shutting down")
    closeDone()    // path B: SIGINT / SIGTERM
}()
```

`closeDone` is a variable holding an anonymous function. Both call sites share the same `once` instance — whichever fires first executes `close(done)`; the second call is a silent no-op.

**Normal exit (pipeline finishes):**
1. Pipeline completes, `main` returns.
2. `defer closeDone()` fires → `once.Do` sees `done == 0` → closes channel → sets `done = 1`.
3. Signal goroutine is still blocked on `<-sig`, never runs.

**SIGINT fires mid-run:**
1. Signal goroutine calls `closeDone()` → `once.Do` sees `done == 0` → closes channel → sets `done = 1`.
2. Cancellation propagates through pipeline stages, `main` returns.
3. `defer closeDone()` fires → `once.Do` sees `done == 1` → returns immediately. No panic.

### Key properties

| Property | Behaviour |
|---|---|
| Execution count | `f()` runs exactly once per `Once` instance |
| Concurrent callers | All block until `f()` completes, then return |
| Subsequent calls | Instant no-op (single atomic read) |
| Different `f` on second call | Still not executed — `Once` tracks the instance, not the function |
| Zero value | Ready to use; no initialisation needed |

### Caveat: `defer` does not run when the process is killed

`sync.Once` solves the **double-close race** between two code paths. It does not help with a separate, orthogonal problem: **`defer` only executes on a clean exit**.

`defer` runs when:
- the surrounding function returns normally
- a `panic` unwinds the stack (and is not recovered)

`defer` does **not** run when:
- the process receives **SIGKILL** (`kill -9`) — SIGKILL cannot be caught or intercepted by any program; the OS removes the process immediately
- the process receives **SIGTERM** or **SIGINT** and those signals are **not handled** — Go's default signal behaviour calls `runtime.exit`, which bypasses the defer stack

This is why the `sync.Once` pattern pairs `defer closeDone()` with a **signal goroutine** that explicitly calls `closeDone()` on SIGTERM/SIGINT:

```go
go func() {
    <-sig            // catches SIGTERM / SIGINT
    closeDone()      // explicit call — does not rely on defer
}()
```

The explicit call is the actual shutdown path for signals. `defer closeDone()` only fires on the normal-return path (pipeline runs to completion). SIGKILL cannot be caught by either path — if the process is force-killed, neither `defer` nor the signal goroutine runs, and all cleanup is abandoned.

The practical implication: `defer` is reliable for **normal exits and caught signals**. If you need cleanup to survive a forced kill (e.g., flushing a write-ahead log, releasing an external lock), you need an out-of-process mechanism — a sidecar, an OS watchdog, or a persistent side-effect that a restart can detect and repair.

---

## Fan-Out / Fan-In Pipeline Pattern

### The problem

A single pipeline stage processes values one at a time. If that stage is CPU-bound — like checking primality — it becomes a bottleneck. The upstream generator can produce numbers far faster than a single goroutine can filter them.

Fan-out solves this by distributing the work across multiple goroutines. Fan-in solves the follow-on problem: collecting the results back into a single channel for the next stage.

```
                     ┌─► primeFilter goroutine 1 ─┐
randNums ──► fanOut ─┼─► primeFilter goroutine 2 ─┼─► fanIn ──► merged ──► cappedGenerator
                     └─► primeFilter goroutine N ─┘
```

### Fan-Out

```go
func fanOut(done <-chan struct{}, in <-chan int) []<-chan int {
    numWorkers := runtime.NumCPU()
    channels := make([]<-chan int, numWorkers)
    for i := range numWorkers {
        channels[i] = primeFilter(done, in)
    }
    return channels
}
```

`fanOut` calls `primeFilter` N times — once per CPU — passing the **same** `in` channel each time. This is the key insight: you do not need a dispatcher goroutine. Go channels are safe for concurrent reads. All N goroutines pull from `in` simultaneously, competing for the next value. Whichever goroutine is free takes the next number and checks it. The channel acts as the work queue automatically.

The number of workers is set to `runtime.NumCPU()` because primality checking is pure CPU work — there is no I/O to overlap, so one goroutine per core is the natural ceiling.

### Fan-In

```go
func fanIn(done <-chan struct{}, channels []<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for {
                select {
                case <-done:
                    return
                case v, ok := <-c:
                    if !ok {
                        return
                    }
                    select {
                    case out <- v:
                    case <-done:
                        return
                    }
                }
            }
        }(ch)
    }
    go func() {
        wg.Wait()
        close(out)
    }()
    return out
}
```

`fanIn` takes the slice of output channels from `fanOut` and merges them into a single `out` channel. It spins up one goroutine per input channel — each reads from its assigned channel and forwards values into the shared `out`. The goroutines run concurrently so no single input channel can block the others.

### Why sync.WaitGroup is needed

`fanIn` has a specific problem: `out` is written to by N independent goroutines. A channel must only be closed once — closing an already-closed channel panics. So the question becomes: **who closes `out`, and when?**

The answer is: `out` must be closed only after the **last** goroutine has finished writing. But goroutines finish at different times — there is no way to know in advance which one will be last.

`sync.WaitGroup` is the coordination mechanism:

```
wg.Add(1)        ← called once per goroutine, before it starts
defer wg.Done()  ← called when the goroutine returns
wg.Wait()        ← blocks until all Done() calls have fired
```

A separate goroutine calls `wg.Wait()` and then closes `out`. It blocks until every writer goroutine has exited, then closes exactly once:

```go
go func() {
    wg.Wait()
    close(out)
}()
```

Without a WaitGroup there is no safe way to close `out`. You cannot use a counter with a mutex (fragile), you cannot pick an arbitrary goroutine to close it (it may not be the last one), and you cannot leave it open (the downstream `range` loop would never terminate).

The reason `wg.Wait()` runs in its own goroutine rather than inline is that `out` must be returned to the caller before anyone starts reading from it. Calling `wg.Wait()` inline would block `fanIn` from returning — nobody would receive from `out`, the writer goroutines would be stuck trying to send, and the program would deadlock.

```
goroutine 1 ──► done ──► wg.Done()
goroutine 2 ──► done ──► wg.Done()  ──► wg.Wait() unblocks ──► close(out)
goroutine N ──► done ──► wg.Done()
```

### Why concurrent channel reads are safe

Go channels are implemented in the runtime (`runtime/chan.go`) with an internal mutex that protects all operations on the channel's state:

```
hchan struct {
    qcount   uint           // number of elements in buffer
    buf      unsafe.Pointer // circular buffer (buffered channels)
    sendq    waitq          // goroutines blocked on send
    recvq    waitq          // goroutines blocked on receive
    lock     mutex          // protects everything above
}
```

When any goroutine executes `<-ch`, the runtime acquires `lock`, checks if a value is available, and either takes it or parks the goroutine on `recvq` — then releases the lock. Because only one goroutine can hold `lock` at a time, two goroutines racing to read from the same channel can never receive the same value.

With N `primeFilter` goroutines all reading from `randNums`:

```
goroutine 1: acquires lock → takes value 42  → releases lock
goroutine 2: acquires lock → takes value 107 → releases lock
goroutine 3: acquires lock → no value yet    → parked on recvq
```

Each number is delivered to exactly one goroutine. No duplication, no data race. The channel is the synchronization point — which is the core design philosophy behind Go's concurrency model: rather than sharing memory and protecting it with your own locks, you communicate through channels that have locking built in. The channel *is* the mutex, just with a cleaner interface.

### Wiring it together in main

```go
randNums := generator(done, randomNumber)
primes   := fanOut(done, randNums)           // []<-chan int, N goroutines
merged   := fanIn(done, primes)              // <-chan int, single merged stream
capped   := cappedGenerator(done, merged, 600)

for n := range capped {
    fmt.Println(n)
}
```

`primeFilter` on line 147 is replaced by the `fanOut` → `fanIn` pair. The rest of the pipeline — `generator`, `cappedGenerator`, shutdown via `done` — is unchanged. The fan-out/fan-in is a drop-in replacement for any single pipeline stage.
