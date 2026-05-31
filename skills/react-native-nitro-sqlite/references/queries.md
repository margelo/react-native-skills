---
id: queries
title: Executing queries and reading results
scope: react-native-nitro-sqlite
keywords: execute, executeAsync, params, parameter binding, placeholder, QueryResult, results, rows, _array, item, length, rowsAffected, insertId, metadata, ColumnType, typed rows, generic, SQLiteValue, blob, ArrayBuffer, null
---

# Executing queries and reading results

## Mental model

A single SQL statement runs via `execute` (sync, on the JS thread) or `executeAsync` (async, off the JS thread). Both call the native module directly — they do **not** go through the per-database operation queue, so they never throw the "database is busy" error. Both return — directly or via `Promise` — the **same** `QueryResult` shape. Pick async by default to avoid blocking the UI (see [concurrency.md](./concurrency.md)).

```ts
const r = db.execute('SELECT * FROM users WHERE age > ?', [21])
const r2 = await db.executeAsync('SELECT * FROM users WHERE age > ?', [21])
```

## Signatures

```ts
type SQLiteValue = boolean | number | string | ArrayBuffer | null
type SQLiteQueryParams = SQLiteValue[]

// Optional generic Row type for the returned rows
db.execute<Row>(query: string, params?: SQLiteQueryParams): QueryResult<Row>
db.executeAsync<Row>(query: string, params?: SQLiteQueryParams): Promise<QueryResult<Row>>
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
type QueryResult<Row = Record<string, SQLiteValue>> = {
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
const r = db.execute<{ id: number; name: string }>('SELECT id, name FROM users')

// Option A — plain array
for (const row of r.results) {
  console.log(row.id, row.name)
}

// Option B — TypeORM-style view
console.log(r.rows.length)        // number of rows
const first = r.rows.item(0)      // Row | undefined
const all = r.rows._array         // Row[]
```

`rowsAffected` — number of rows changed by INSERT/UPDATE/DELETE. `insertId` — the auto-generated rowid after an INSERT (present when applicable):

```ts
const ins = db.execute('INSERT INTO users (name) VALUES (?)', ['Marc'])
console.log(ins.rowsAffected) // 1
console.log(ins.insertId)     // e.g. 1
```

> SELECT queries return `rowsAffected: 0` and an empty `insertId`; their data is in `results` / `rows`.

## Typed rows

Pass a generic to get typed row objects (no runtime validation — it's a TypeScript convenience):

```ts
interface User { id: number; name: string; age: number }

const { results } = await db.executeAsync<User>('SELECT id, name, age FROM users')
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

`ColumnType` enum members: `BOOLEAN`, `NUMBER`, `INT64`, `TEXT`, `ARRAY_BUFFER`, `NULL_VALUE`. Declared type is `"UNKNOWN"`-equivalent for dynamic/computed columns (e.g. function results), where SQLite can't infer a static type.

## Storing blobs (ArrayBuffer)

```ts
const bytes = new Uint8Array([1, 2, 3, 255])
db.execute('INSERT INTO files (id, blob) VALUES (?, ?)', [1, bytes.buffer])

const { results } = db.execute<{ blob: ArrayBuffer }>('SELECT blob FROM files WHERE id = ?', [1])
const view = new Uint8Array(results[0].blob)
```

Bind the underlying `ArrayBuffer` (e.g. `typedArray.buffer`), and read it back as an `ArrayBuffer`.

## Sync vs async — when to use which

- `execute` (sync): small, fast, one-off reads/writes where briefly blocking the JS thread is fine. It calls native directly and never throws "busy", but it does block the JS thread for the duration of the query.
- `executeAsync` (async): anything that could be slow (large reads, many rows, complex joins) — keeps the UI thread free. It also bypasses the operation queue; only `executeBatch`/`executeBatchAsync`/`transaction` are serialized. See [concurrency.md](./concurrency.md).

## Gotchas

- **Both `results` and `rows` exist** on every result — don't assume only one. `rows` is mainly for TypeORM compatibility.
- **`insertId` only after INSERT.** It's `undefined` for SELECT/UPDATE/DELETE.
- **Generic types are not validated.** `execute<User>` only affects TypeScript; the runtime returns whatever SQLite produced.
- **No automatic JSON.** Store objects by serializing to TEXT yourself (`JSON.stringify` / `JSON.parse`).
- **Booleans round-trip as the JS `boolean` type** via `SQLiteValue`, but SQLite stores them as integers under the hood; filter with `WHERE flag = ?` passing `true`/`false`.

## Pointers

- Source: `package/src/operations/execute.ts`, `package/src/types.ts`, `package/src/specs/NitroSQLiteQueryResult.nitro.ts`
- Related: [transactions.md](./transactions.md), [batch-and-files.md](./batch-and-files.md), [concurrency.md](./concurrency.md)
