---
id: queries
title: Executing queries and reading results
scope: react-native-nitro-sqlite
keywords: execute, executeAsync, params, parameter binding, placeholder, QueryResult, results, rows, _array, item, length, rowsAffected, insertId, metadata, ColumnType, typed rows, generic, SQLiteValue, blob, ArrayBuffer, null
---

# Executing queries and reading results

## Mental model

A single SQL statement runs via `execute` (sync, on the JS thread) or `executeAsync` (async, off the JS thread). Both call the native module directly — they do **not** go through the per-database operation queue or its synchronous "database is busy" check. Both return — directly or via `Promise` — the **same** `QueryResult` shape. Pick async by default to avoid blocking the UI (see [concurrency.md](./concurrency.md)).

```ts
const r = db.execute('SELECT * FROM users WHERE age > ?', [21])
const r2 = await db.executeAsync('SELECT * FROM users WHERE age > ?', [21])
```

## Signatures

```ts
type SQLiteValue = boolean | number | string | ArrayBuffer | null
type SQLiteQueryParams = SQLiteValue[]
type QueryResultRow = Record<string, SQLiteValue>

// Optional generic Row type for the returned rows
db.execute<Row extends QueryResultRow>(
  query: string,
  params?: SQLiteQueryParams,
): QueryResult<Row>
db.executeAsync<Row extends QueryResultRow>(
  query: string,
  params?: SQLiteQueryParams,
): Promise<QueryResult<Row>>
```

## Parameter binding

Always bind values with `?` placeholders and a params array — never string-concatenate (SQL injection + type coercion bugs).

```ts
db.execute(
  'INSERT INTO users (id, name, age) VALUES (?, ?, ?)',
  [1, 'Marc', 24]
)

const { results } = db.execute(
  'SELECT * FROM users WHERE name = ? AND age >= ?',
  ['Marc', 18]
)
```

Bindable types: `boolean | number | string | ArrayBuffer | null`. `ArrayBuffer` is stored/read as a SQLite **BLOB**; `null` binds SQL `NULL`.

## The `QueryResult` shape

```ts
type QueryResult<Row extends QueryResultRow = QueryResultRow> = {
  rowsAffected: number
  insertId?: number
  results: Row[]                  // plain array of row objects (keyed by column name)
  metadata?: Record<string, { name: string; type: ColumnType; index: number }>
  rows: {                         // TypeORM-style view of the same rows
    _array: Row[]
    length: number
    item: (idx: number) => Row | undefined
  }
}
```

Two equivalent ways to read rows — use whichever you like, they hold the same data:

```ts
import type { QueryResultRow } from 'react-native-nitro-sqlite'

type UserRow = QueryResultRow & {
  id: number
  name: string
}

const r = db.execute<UserRow>('SELECT id, name FROM users')

// Option A — plain array
for (const row of r.results) {
  console.log(row.id, row.name)
}

// Option B — TypeORM-style view
console.log(r.rows.length)        // number of rows
const first = r.rows.item(0)      // Row | undefined
const all = r.rows._array         // Row[]
```

Consume `rowsAffected` and `insertId` from the result of the write that produced them:

```ts
const ins = db.execute('INSERT INTO users (name) VALUES (?)', ['Marc'])
console.log(ins.rowsAffected) // 1
console.log(ins.insertId)     // e.g. 1
```

> The native result reads SQLite's connection-level change count and last-insert-rowid after every statement. A later `SELECT` can therefore retain values from an earlier write. Do not interpret `rowsAffected` or `insertId` on a read result.

## Typed rows

Pass a generic to get typed row objects (no runtime validation — it's a TypeScript convenience):

```ts
import type { QueryResultRow } from 'react-native-nitro-sqlite'

type UserRow = QueryResultRow & {
  id: number
  name: string
  age: number
}

const { results } = await db.executeAsync<UserRow>(
  'SELECT id, name, age FROM users',
)
results[0].name // typed as string
```

## Column metadata

When `metadata` is present, it maps each column name to its declared type and index:

```ts
import { ColumnType } from 'react-native-nitro-sqlite'

const { metadata } = db.execute('SELECT id, name FROM users LIMIT 1')
if (metadata) {
  for (const [col, meta] of Object.entries(metadata)) {
    console.log(col, meta.type, meta.index)
  }
}
```

`ColumnType` enum members are `BOOLEAN`, `NUMBER`, `INT64`, `TEXT`, `ARRAY_BUFFER`, and `NULL_VALUE`. Treat metadata as informational only; do not use `metadata.type` for runtime validation or value decoding.

## Storing blobs (ArrayBuffer)

```ts
const bytes = new Uint8Array([1, 2, 3, 255])
db.execute('INSERT INTO files (id, blob) VALUES (?, ?)', [1, bytes.buffer])

const { results } = db.execute<QueryResultRow & { blob: ArrayBuffer }>(
  'SELECT blob FROM files WHERE id = ?',
  [1],
)
const view = new Uint8Array(results[0].blob)
```

Bind the underlying `ArrayBuffer` (e.g. `typedArray.buffer`), and read it back as an `ArrayBuffer`.

## Sync vs async — when to use which

- `execute` (sync): small, fast, one-off reads/writes where briefly blocking the JS thread is fine. It bypasses the JS operation queue, but it blocks the JS thread for the duration of the query.
- `executeAsync` (async): anything that could be slow (large reads, many rows, complex joins) — keeps the UI thread free. It also bypasses the operation queue; only `executeBatch`/`executeBatchAsync`/`transaction` are serialized. See [concurrency.md](./concurrency.md).

## Gotchas

- **Both `results` and `rows` exist** on every result — don't assume only one. `rows` is mainly for TypeORM compatibility.
- **Write counters are connection state.** Read `rowsAffected` and `insertId` only from the corresponding write result.
- **Generic types must extend `QueryResultRow` and are not validated.** The generic only affects TypeScript; the runtime returns whatever SQLite produced.
- **No automatic JSON.** Store objects by serializing to TEXT yourself (`JSON.stringify` / `JSON.parse`).
- **Booleans read back as numbers.** Boolean parameters bind as SQLite integers, and result integers are returned as JavaScript numbers (`0` or `1`). Convert explicitly, for example `const enabled = row.enabled === 1`.
- **SQLite integers are read as JavaScript numbers.** Values outside the safe integer range lose precision. Read exact 64-bit values as text, for example with `CAST(id AS TEXT)`.

## Pointers

- Source: `package/src/operations/execute.ts`, `package/src/types.ts`, `package/src/specs/NitroSQLiteQueryResult.nitro.ts`
- Related: [transactions.md](./transactions.md), [batch-and-files.md](./batch-and-files.md), [concurrency.md](./concurrency.md)
