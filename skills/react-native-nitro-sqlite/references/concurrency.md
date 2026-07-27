---
id: concurrency
title: Sync vs async and the per-database operation queue
scope: react-native-nitro-sqlite
keywords: sync, async, queue, busy, Database is busy, ordering, serial, JS thread, UI thread, blocking, in progress, concurrency
---

# Sync vs async and the per-database operation queue

This behavior matters when batches and transactions overlap.

## Mental model

For each open database name, the library serializes async batches and transactions so they do not interleave.

Only `executeBatch`, `executeBatchAsync`, and `transaction` participate in that scheduling. `execute`, `executeAsync`, `loadFile`, and `loadFileAsync` call the native module directly.

Two orthogonal axes:

- **Blocking vs off-thread.** `execute`, `executeBatch`, `loadFile`, and `tx.execute`/`commit`/`rollback` run synchronously on the **JS thread** and block it until SQLite returns. `executeAsync`, `executeBatchAsync`, and `loadFileAsync` run their native SQL work off-thread and return a `Promise`.
- **Transaction callbacks still run in JavaScript.** `transaction` returns a `Promise`, but `tx.execute` inside the callback remains synchronous; use `tx.executeAsync` for potentially slow statements.
- **Serialized vs direct.** Independently, only `executeBatch` / `executeBatchAsync` / `transaction` participate in batch/transaction serialization (and only sync `executeBatch` performs the library's "busy" check).

## Which calls touch the queue

| Call | Queue behavior |
|---|---|
| `db.execute` (sync) | Direct. Blocks the JS thread; does not perform the batch/transaction "busy" check. |
| `db.executeAsync` (async) | Direct. Off-thread; not serialized with batches or transactions. |
| `db.loadFile` (sync) | Direct. Blocks the JS thread; does not perform the batch/transaction "busy" check. |
| `db.loadFileAsync` (async) | Direct. Off-thread; not serialized with batches or transactions. |
| `db.executeBatch` (sync) | **Throws if an async batch or transaction is pending.** |
| `db.executeBatchAsync` (async) | Runs serially with other async batches and transactions. |
| `db.transaction` (async) | Runs serially with other transactions and async batches. |

> Practical takeaway: **transactions and async batches are strictly ordered**, and a **synchronous `executeBatch` will throw** if you run it while a batch or transaction is pending. Direct operations do not participate in that ordering.

## The "Database is busy" error

```
NitroSQLiteError: Cannot run synchronous operation on database.
Database <name> is busy with another operation.
```

This happens when a **synchronous `executeBatch`** is called while an async batch or transaction is in progress or pending:

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

- **The library's "busy with another operation" error comes from sync `executeBatch`.** If you see it, find an un-awaited async batch or transaction before it.
- **`loadFile`/`loadFileAsync` are direct operations.** `loadFile` blocks the JS thread; `loadFileAsync` runs off-thread but is not serialized with batches and transactions.
- **Plain `execute` is direct**, but it still blocks the JS thread and can run concurrently with native async work. Put related writes in one transaction.
- **Scheduling is per database name.** Different open database names are independent.

## Pointers

- Source: `package/src/DatabaseQueue.ts` (`queueOperationAsync`, `startOperationSync`, `startOperationAsync`)
- Related: [queries.md](./queries.md), [transactions.md](./transactions.md), [batch-and-files.md](./batch-and-files.md)
