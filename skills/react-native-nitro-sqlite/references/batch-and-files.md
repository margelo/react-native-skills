---
id: batch-and-files
title: Batch execution and loading SQL files
scope: react-native-nitro-sqlite
keywords: executeBatch, executeBatchAsync, BatchQueryCommand, BatchQueryResult, params array of arrays, bulk insert, loadFile, loadFileAsync, FileLoadResult, dump, seed
---

# Batch execution and loading SQL files

## Batch: many statements in one transaction

`executeBatch` / `executeBatchAsync` run a list of statements inside a **single transaction** — ideal for bulk inserts and seeding. Each command is a `{ query, params? }`.

```ts
import type { BatchQueryCommand } from 'react-native-nitro-sqlite'

const commands: BatchQueryCommand[] = [
  { query: 'CREATE TABLE IF NOT EXISTS t (id INTEGER, age INTEGER)' },
  { query: 'INSERT INTO t (id, age) VALUES (?, ?)', params: [1, 10] },
  { query: 'INSERT INTO t (id, age) VALUES (?, ?)', params: [2, 20] },
]

const { rowsAffected } = db.executeBatch(commands)        // sync
// or
const r = await db.executeBatchAsync(commands)            // async (queued)
```

### Signatures and types

```ts
interface BatchQueryCommand {
  query: string
  params?: SQLiteQueryParams | SQLiteQueryParams[]   // one set, OR many sets
}

interface BatchQueryResult {
  rowsAffected?: number
}

db.executeBatch(commands: BatchQueryCommand[]): BatchQueryResult
db.executeBatchAsync(commands: BatchQueryCommand[]): Promise<BatchQueryResult>
```

### One query, many parameter sets (efficient bulk insert)

When you run the **same** statement many times, declare it **once** and pass `params` as an **array of arrays**. The library executes the prepared statement for each inner array — far cheaper than N separate commands.

```ts
const commands: BatchQueryCommand[] = [
  {
    query: 'INSERT INTO t (id, age) VALUES (?, ?)',
    params: [
      [3, 30],
      [4, 40],
      [5, 50],
    ],
  },
]
await db.executeBatchAsync(commands)
```

So `params` is either:
- a single param set: `[1, 10]`, or
- multiple param sets for the same query: `[[1, 10], [2, 20]]`.

### Atomicity

The whole batch runs in one transaction: if any statement fails, the batch errors (as a `NitroSQLiteError`) and nothing is committed.

## Loading SQL files

`loadFile` / `loadFileAsync` read a `.sql` file from disk inside a transaction. The loader executes **each non-empty line as a separate SQL command**; it does not parse complete SQL statements.

```ts
const r = db.loadFile('/absolute/path/to/dump.sql')          // sync
// or
const r2 = await db.loadFileAsync('/absolute/path/to/dump.sql')  // async

// FileLoadResult extends BatchQueryResult:
r.rowsAffected   // number | undefined — total rows affected
r.commands       // number | undefined — how many SQL commands were executed
```

```ts
interface FileLoadResult extends BatchQueryResult {
  rowsAffected?: number
  commands?: number
}
```

- Use an **absolute path** (subject to the iOS sandbox — copy the file into the app dir first if needed; see [connections.md](./connections.md)).
- Prefer `loadFileAsync` for large files so you don't block the JS thread.
- Keep every command on exactly one line:

```sql
CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT NOT NULL);
INSERT INTO users (name) VALUES ('Marc');
```

Normal multiline `CREATE TABLE` statements, triggers, and formatted SQL dumps will fail because their individual lines are not complete commands. Standalone comment lines are not safe input either.

For general migration files, use a trusted SQL parser or define the statements explicitly as `BatchQueryCommand[]`, then call `executeBatchAsync`. Do not split arbitrary SQL on semicolons; strings and triggers can contain semicolons.

## Sync vs async (important)

- `executeBatch` (sync) goes through the **busy check**: if the database is currently running or has a queued async batch/transaction, it **throws** `Database <name> is busy with another operation.` Use `executeBatchAsync`, or `await` the pending work first.
- `executeBatchAsync` is **queued** and runs serially in submission order (alongside `transaction`).
- `loadFile` / `loadFileAsync` **do not use the queue** — they call the native module directly (like `execute` / `executeAsync`). `loadFile` blocks the JS thread; `loadFileAsync` runs off-thread. Neither throws "busy" and neither is serialized by the JS queue.

See [concurrency.md](./concurrency.md) for the full queue model.

## Choosing batch vs transaction

- **Batch** — fixed list of statements, especially many inserts of the same shape. Simplest and fastest for bulk writes.
- **Transaction** — when you need to read intermediate results, branch logic, or compute later params from earlier rows. See [transactions.md](./transactions.md).

## Gotchas

- **`params` array-of-arrays only makes sense with one `query`** repeated. Don't mix it up with separate command objects unless the queries differ.
- **`BatchQueryResult` has only `rowsAffected`** — no per-statement results. If you need returned rows, use individual `execute`/`executeAsync` calls or a transaction.
- **`loadFile` is line-oriented.** Use it only for files where every non-empty line is one complete command.
- **Sync batch throws when busy** — the most common surprise; switch to async.

## Pointers

- Source: `package/src/operations/executeBatch.ts`, `package/src/operations/session.ts`, `package/src/types.ts`
- Related: [transactions.md](./transactions.md), [concurrency.md](./concurrency.md)
