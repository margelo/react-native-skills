---
id: transactions
title: Transactions
scope: react-native-nitro-sqlite
keywords: transaction, tx, commit, rollback, BEGIN, COMMIT, ROLLBACK, atomic, auto commit, auto rollback, finalized transaction, queueOperationAsync, isExclusive
---

# Transactions

## Mental model

`db.transaction(callback)` runs a set of statements atomically. It is **async-only** (returns a `Promise`). Internally it:

1. Goes through the per-database operation queue (so it can't interleave with other queued ops — see [concurrency.md](./concurrency.md)).
2. Runs `BEGIN TRANSACTION`.
3. Invokes your callback with a `tx` object.
4. If the callback **resolves** without having committed → it runs `COMMIT`.
5. If the callback **throws/rejects** → it runs `ROLLBACK` and re-throws as a `NitroSQLiteError`.

```ts
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (id, name) VALUES (?, ?)', [1, 'Marc'])
  await tx.executeAsync('UPDATE accounts SET balance = balance - ? WHERE id = ?', [100, 1])
})
// committed automatically here
```

## Signature

```ts
db.transaction: <Result = void>(
  cb: (tx: Transaction) => Promise<Result>
) => Promise<Result>

interface Transaction {
  execute: ExecuteQuery        // sync, scoped to this transaction
  executeAsync: ExecuteAsyncQuery
  commit(): NitroSQLiteQueryResult
  rollback(): NitroSQLiteQueryResult
}
```

The value your callback returns is forwarded as the resolved value of `db.transaction(...)`:

```ts
const newId = await db.transaction(async (tx) => {
  const r = tx.execute('INSERT INTO users (name) VALUES (?)', ['Marc'])
  return r.insertId
})
```

## Auto commit (default)

If you don't call `tx.commit()` / `tx.rollback()` yourself and the callback resolves, the transaction commits automatically:

```ts
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (id, name) VALUES (?, ?)', [1, 'Marc'])
  // no explicit commit → auto-committed
})
```

## Auto rollback on error

Throwing anywhere in the callback rolls everything back:

```ts
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (id, name) VALUES (?, ?)', [1, 'Marc'])
  throw new Error('abort')      // → ROLLBACK, nothing persisted
}).catch((e) => {
  // e is a NitroSQLiteError
})
```

## Manual commit / rollback

You can finalize explicitly. After committing or rolling back, the transaction is **finalized** — any further `tx.execute` / `tx.commit` / `tx.rollback` throws `Cannot execute query on finalized transaction`.

```ts
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (id) VALUES (?)', [1])

  if (someCondition) {
    tx.rollback()   // finalize: undo
    return
  }

  tx.commit()       // finalize: persist
  // tx.execute(...) here would throw — transaction is finalized
})
```

## Mixing sync and async inside a transaction

Both `tx.execute` (sync) and `tx.executeAsync` (async) are available and operate on the same transaction. The whole transaction already holds its queue slot, so internal calls run in the order you write them:

```ts
await db.transaction(async (tx) => {
  const a = tx.execute('SELECT count(*) AS n FROM users')
  await tx.executeAsync('INSERT INTO log (n) VALUES (?)', [a.results[0].n])
})
```

## Ordering of concurrent transactions

Because transactions go through the queue, multiple `db.transaction(...)` calls run **serially in submission order** — you can fire several without `await`ing each and they won't interleave:

```ts
const ps = []
for (let i = 0; i < 10; i++) {
  ps.push(db.transaction(async (tx) => {
    tx.execute('UPDATE counter SET v = v + 1')
  }))
}
await Promise.all(ps) // applied one after another, in order
```

## Gotchas

- **`transaction` is async only** — there is no synchronous `db.transaction`. Always `await` it (or handle the promise).
- **Don't `open`/`close` inside a transaction.**
- **Finalized = locked.** After an explicit `commit()`/`rollback()`, don't touch `tx` again.
- **Throwing rolls back.** Don't swallow errors inside the callback if you want the rollback to happen; let them propagate.
- **The error you catch is a `NitroSQLiteError`** (the original error is wrapped/preserved). See [migration-and-errors.md](./migration-and-errors.md).
- **For pure bulk inserts**, `executeBatch`/`executeBatchAsync` is simpler and also atomic — see [batch-and-files.md](./batch-and-files.md). Use `transaction` when you need to read intermediate results or branch logic.

## Pointers

- Source: `package/src/operations/transaction.ts`, `package/src/DatabaseQueue.ts`
- Related: [concurrency.md](./concurrency.md), [batch-and-files.md](./batch-and-files.md)
