---
id: concurrency
title: Sync vs async and the per-database operation queue
scope: react-native-nitro-sqlite
keywords: sync, async, queue, DatabaseQueue, busy, Database is busy, ordering, serial, setImmediate, JS thread, UI thread, blocking, in progress, concurrency
---

# Sync vs async and the per-database operation queue

This is the most important behavioral concept in the library and the source of the most confusing errors.

## Mental model

Each open database has its own JS-level **operation queue** (`{ queue: [], inProgress: boolean }`), keyed by the db `name`. It exists to serialize the operations that go through it so they never corrupt each other or interleave mid-transaction.

**Crucial detail — only three operations use this queue:** `executeBatch`, `executeBatchAsync`, and `transaction`. Everything else (`execute`, `executeAsync`, `loadFile`, `loadFileAsync`) calls the native module **directly** and does not touch the JS queue at all.

Two orthogonal axes:

- **Blocking vs off-thread.** `execute`, `executeBatch`, `loadFile`, and `tx.execute`/`commit`/`rollback` run synchronously on the **JS thread** and block it until SQLite returns. `executeAsync`, `executeBatchAsync`, `transaction`, and `loadFileAsync` run off the JS thread and return a `Promise`.
- **Queued vs not.** Independently, only `executeBatch` / `executeBatchAsync` / `transaction` are serialized through the JS queue (and only `executeBatch`, being synchronous, can throw "busy").

## Which calls touch the queue

| Call | Queue behavior |
|---|---|
| `db.execute` (sync) | **Bypasses the queue.** Blocks the JS thread; never throws "busy". |
| `db.executeAsync` (async) | **Bypasses the queue.** Off-thread; not serialized by the JS queue. |
| `db.loadFile` (sync) | **Bypasses the queue.** Blocks the JS thread; never throws "busy". |
| `db.loadFileAsync` (async) | **Bypasses the queue.** Off-thread; not serialized by the JS queue. |
| `db.executeBatch` (sync) | Uses `startOperationSync` → **throws if the db is busy** with a queued/in-progress async batch or transaction. |
| `db.executeBatchAsync` (async) | Enqueued (`queueOperationAsync`) — runs serially in submission order. |
| `db.transaction` (async) | Enqueued — runs serially; can't interleave with other transactions/batches. |

> Practical takeaway: **transactions and async batches are strictly ordered**, and a **synchronous `executeBatch` will throw** if you run it while a batch/transaction is pending. `execute`/`executeAsync`/`loadFile`/`loadFileAsync` are never blocked by the queue and never throw "busy".

## The "Database is busy" error

```
NitroSQLiteError: Cannot run synchronous operation on database.
Database <name> is busy with another operation.
```

This happens when a **synchronous `executeBatch`** is called while the queue has an in-progress or pending async batch/transaction (it is the only synchronous queue-aware op):

```ts
// ❌ Will throw if the async batch hasn't finished
db.executeBatchAsync(bigInsert)   // not awaited → queued, inProgress
db.executeBatch(moreInserts)      // throws: database is busy
```

Fixes:

```ts
// ✅ await the async op first
await db.executeBatchAsync(bigInsert)
db.executeBatch(moreInserts)

// ✅ or just use the async variant for both
await db.executeBatchAsync(bigInsert)
await db.executeBatchAsync(moreInserts)
```

## Ordering guarantees

Async queue operations run in the exact order they were submitted, even if you don't `await` each one:

```ts
const results = await Promise.all([
  db.transaction(async (tx) => tx.execute('UPDATE c SET v = v + 1')),
  db.transaction(async (tx) => tx.execute('UPDATE c SET v = v + 1')),
  db.executeBatchAsync([{ query: 'INSERT INTO log DEFAULT VALUES' }]),
])
// Each ran to completion before the next started, in this order.
```

## Closing while busy

`close()` does not block on the queue: if operations are still pending it logs a warning and closes anyway. Always `await` outstanding async work before `close()`:

```ts
await Promise.all(pendingOps)
db.close()
```

## Choosing sync vs async — guidance

- **Default to async** (`executeAsync`, `executeBatchAsync`, `transaction`) for anything that touches more than a few rows, runs at startup, or happens during animation/scroll. It keeps the JS/UI thread responsive.
- **Use sync** (`execute`) only for tiny, instant reads where a few-millisecond block is acceptable and you know no async op is in flight.
- **Never** mix a not-awaited async batch/transaction with a following synchronous `executeBatch` — that's the classic "busy" crash.

## Why not just always sync? (it's JSI, so it's "fast")

JSI removes the *bridge* overhead, but the SQL work itself still takes real time. A large query run synchronously blocks the JS thread → dropped frames. Async moves the SQLite work off-thread; the queue additionally keeps batches and transactions correctly ordered.

## Gotchas

- **"Busy" only comes from synchronous `executeBatch`** — never from `execute`, `executeAsync`, `loadFile`, or `loadFileAsync`. If you see it, find the un-awaited async batch/transaction before it.
- **`loadFile`/`loadFileAsync` bypass the queue** — they call native directly. `loadFile` blocks the JS thread (like `execute`); `loadFileAsync` runs off-thread but is not serialized by the JS queue.
- **Plain `execute` won't throw "busy"**, but it still blocks the JS thread and can run concurrently with native async work — prefer wrapping related writes in a transaction.
- **One queue per db `name`.** Two different databases have independent queues and can run truly in parallel.
- **`setImmediate` scheduling** means even a queue with one item yields a tick before running — async ops are never synchronously resolved.

## Pointers

- Source: `package/src/DatabaseQueue.ts` (`queueOperationAsync`, `startOperationSync`, `startOperationAsync`)
- Related: [queries.md](./queries.md), [transactions.md](./transactions.md), [batch-and-files.md](./batch-and-files.md)
