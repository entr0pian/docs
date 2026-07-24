# SQS Worker — Context, Cancellation and Shutdown

## Goroutines are not child processes

The SQS worker is launched as a goroutine:

```go
go worker.SQSWorker(ctx, sqs.NewFromConfig(cfg), queueURL, database, logQueue)
```

A goroutine is **not** a child process or OS thread. It is a lightweight unit of concurrency managed entirely by the Go runtime, running inside the same process as `main`. The runtime multiplexes thousands of goroutines onto a small pool of OS threads. When a goroutine is waiting (e.g. blocked on a network call), the scheduler moves the thread to another goroutine — no CPU is wasted.

---

## Where the worker halts and waits

The goroutine parks at `client.ReceiveMessage`:

```go
out, err := client.ReceiveMessage(ctx, &sqs.ReceiveMessageInput{
    QueueUrl:            aws.String(queueURL),
    MaxNumberOfMessages: 10,
    WaitTimeSeconds:     20,
})
```

Under the hood this is an HTTP request to the SQS API. The AWS SDK opens a TCP connection and holds it open. SQS keeps the connection alive for up to 20 seconds (`WaitTimeSeconds`), responding the moment messages arrive or the window expires. During this wait the goroutine is suspended — it consumes no CPU.

This is **long polling**. Without it (`WaitTimeSeconds: 0`) the call would return immediately on an empty queue, causing the loop to hammer the API with empty responses.

---

## Context

A `context.Context` carries two things: a **cancellation signal** and an optional **deadline**. You pass it into long-running operations so they know when to stop.

### `context.Background()`

The root context — never cancels, has no deadline. Everything is built on top of it.

### `context.WithCancel(parent)`

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
```

Returns a derived context and a `cancel` function. Calling `cancel()` closes `ctx.Done()`, a channel that all operations watching this context will detect. `defer cancel()` is a safety net: if `main` exits for any reason the cancel fires, preventing goroutine leaks.

### `context.WithTimeout(parent, duration)`

```go
shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 25*time.Second)
defer shutdownCancel()
```

Like `WithCancel` but automatically cancels after the given duration. Note this starts from `context.Background()`, **not** from the already-cancelled `ctx` — if it inherited from `ctx` it would be cancelled immediately and `srv.Shutdown` would have no time to drain.

---

## Shutdown sequence

```
SIGTERM received
  │
  ├─ <-quit unblocks main
  │
  ├─ cancel() called
  │     └─ ctx.Done() closes
  │           └─ SQS worker's ReceiveMessage aborts (HTTP connection torn down)
  │           └─ goroutine hits ctx.Err() != nil check and returns
  │
  ├─ shutdownCtx created with 25s timeout
  │
  └─ srv.Shutdown(shutdownCtx) blocks until:
        - all in-flight HTTP handlers finish, OR
        - 25 seconds elapse
```

The preStop hook sleeps 5 seconds before SIGTERM is sent, giving kube-proxy time to drain new traffic away from the pod. By the time `Shutdown` is called, the server is typically idle and drains immediately.

---

## How each operation handles cancellation

### `client.ReceiveMessage(ctx, ...)`

The SDK watches `ctx.Done()` internally. When `cancel()` fires it aborts the in-flight HTTP request, tears down the TCP connection, and returns a context cancellation error. The goroutine exits at the `ctx.Err()` guard.

### `database.QueryRowContext(ctx, ...)`

The Postgres driver also watches `ctx`. If cancelled mid-query:

- Query not yet committed → no row written, clean.
- Query committed but response not yet returned → row exists in DB but code never reaches `DeleteMessage`. The SQS message stays on the queue and reappears after the visibility timeout (30s). It will be processed again on next startup, producing a duplicate task.

### `client.DeleteMessage(ctx, ...)`

If cancelled here the DB insert has already committed. The message reappears on the queue after the visibility timeout and gets inserted again — same duplicate scenario as above.

---

## The duplicate task edge case

The DB insert and the SQS delete are two separate operations with no atomicity between them. If the process is killed between them, the message is processed twice. This is the standard SQS trade-off — the queue guarantees **at-least-once** delivery, not exactly-once.

For a task app a duplicate `todo` entry is a minor nuisance, not a correctness issue. If strict once-only semantics were required the solution would be to store the SQS `MessageId` in the DB and check for it before inserting:

```sql
INSERT INTO tasks (title, description, status, sqs_message_id)
VALUES ($1, $2, 'todo', $3)
ON CONFLICT (sqs_message_id) DO NOTHING
```

---

## SQS cost

Each `ReceiveMessage` call counts as one API request regardless of how many messages it returns.

| Scenario | Calls/month |
|---|---|
| 1 worker, empty queue | ~130,000 |
| 4 workers (2 pods × 2 envs), empty queue | ~520,000 |
| AWS free tier | 1,000,000 |

At this scale the cost is effectively $0. SQS pricing only becomes relevant at very high message throughput.
