---
id: migration-and-errors
title: Error handling and migrating from react-native-quick-sqlite
scope: react-native-nitro-sqlite
keywords: NitroSQLiteError, error handling, instanceof, fromError, react-native-quick-sqlite, migration, QuickSQLite, SQLBatchTuple, BatchQueryCommand, metadata
---

# Error handling and migration

## Error handling

`NitroSQLiteError` is the library's normalized error type, but not every public operation normalizes errors.

These paths explicitly wrap failures:

- `open` and `close`
- `execute` and `executeAsync`
- `executeBatch` and `executeBatchAsync`
- `transaction`, including errors thrown by its callback

`delete`, `attach`, `detach`, `loadFile`, and `loadFileAsync` call the native object directly. Their failures are not guaranteed to satisfy `error instanceof NitroSQLiteError`.

```ts
import { NitroSQLiteError } from 'react-native-nitro-sqlite'

try {
  await db.executeAsync('SELECT * FROM does_not_exist')
} catch (error) {
  if (error instanceof NitroSQLiteError) {
    console.warn('SQLite error:', error.message)
    return
  }

  throw error
}
```

When one error-handling path must cover every operation, normalize at the boundary:

```ts
try {
  await db.loadFileAsync(sqlFilePath)
} catch (error) {
  const sqliteError = NitroSQLiteError.fromError(error)
  console.warn(sqliteError.message)
}
```

`fromError` preserves an `Error`'s message and stack. Its `.cause` copies `error.cause`; it does **not** automatically set the wrapped `Error` itself as the cause. For a non-`Error` value, that value becomes the cause.

### Common errors

| Message | Meaning | Fix |
|---|---|---|
| `Database <name> is already open...` | `open()` was called twice for one name | Reuse one connection |
| `Database <name> is not open...` | A queue-aware method was used after `close()` | Open a new connection |
| `...is busy with another operation.` | Sync `executeBatch` ran while an async batch or transaction was pending | Await the work or use `executeBatchAsync` |
| `Cannot execute ... on finalized transaction...` | The transaction was used after `commit()` or `rollback()` | Return immediately after finalizing |

## Migrating from `react-native-quick-sqlite` v8

The package rename is only one part of the migration. Audit imports, raw module calls, batch commands, transaction callback types, and result metadata.

### 1. Replace dependencies

```bash
npm uninstall react-native-quick-sqlite
npm install react-native-nitro-sqlite react-native-nitro-modules
npx pod-install
```

React Native must satisfy the package's `>= 0.75` peer range. New Architecture is not required solely for NitroSQLite; see [setup.md](./setup.md).

### 2. Prefer the connection API

If v8 code uses the raw `QuickSQLite` export, migrate it to the connection returned by `open`:

```ts
// v8
import { QuickSQLite } from 'react-native-quick-sqlite'

QuickSQLite.open('app.sqlite')
const before = QuickSQLite.execute('app.sqlite', 'SELECT * FROM users')

// v9
import { open } from 'react-native-nitro-sqlite'

const db = open({ name: 'app.sqlite' })
const after = db.execute('SELECT * FROM users')
```

Code that already used v8's `open({ name, location })` wrapper keeps the same connection shape after changing the package import.

### 3. Convert batch tuples to command objects

v8 accepts `SQLBatchTuple[]`; v9 accepts `BatchQueryCommand[]`:

```ts
// v8
import type { SQLBatchTuple } from 'react-native-quick-sqlite'

const before = [
  ['CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)'],
  ['INSERT INTO users (name) VALUES (?)', ['Marc']],
  ['INSERT INTO users (name) VALUES (?)', [['Mo'], ['Freya']]],
] satisfies SQLBatchTuple[]

// v9
import type { BatchQueryCommand } from 'react-native-nitro-sqlite'

const after = [
  { query: 'CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)' },
  { query: 'INSERT INTO users (name) VALUES (?)', params: ['Marc'] },
  {
    query: 'INSERT INTO users (name) VALUES (?)',
    params: [['Mo'], ['Freya']],
  },
] satisfies BatchQueryCommand[]

await db.executeBatchAsync(after)
```

### 4. Make transaction callbacks async

v8 permits a synchronous callback; v9 requires a callback that returns a `Promise`:

```ts
// v8
await db.transaction((tx) => {
  tx.execute('INSERT INTO users (name) VALUES (?)', ['Marc'])
})

// v9
await db.transaction(async (tx) => {
  tx.execute('INSERT INTO users (name) VALUES (?)', ['Marc'])
})
```

### 5. Update result and metadata access

v8 exposes optional `rows` and array-shaped metadata. v9 adds the plain `results` array, always builds the TypeORM-compatible `rows` view, and exposes metadata as a record keyed by column name.

```ts
// v8
const firstBefore = result.rows?.item(0)
const idBefore = result.metadata?.find(
  (column) => column.columnName === 'id',
)

// v9
const firstAfter = result.results[0]
const sameRow = result.rows.item(0)
const idAfter = result.metadata?.id
// { name, type, index }
```

Also re-check value semantics in [queries.md](./queries.md): booleans read as `0`/`1`, large integers can lose precision, and write counters should be consumed only from their corresponding write result.

### 6. Reapply native and TypeORM configuration

- Replace quick-sqlite target, flag, and app-group names with the NitroSQLite equivalents in [setup.md](./setup.md).
- Keep TypeORM's driver alias pointed at `react-native-nitro-sqlite`; use production migrations and `synchronize: false` as described in [typeorm.md](./typeorm.md).

## Migration checklist

- [ ] Replace the dependency and install `react-native-nitro-modules`
- [ ] Replace `QuickSQLite` calls or imports
- [ ] Convert `SQLBatchTuple[]` to `BatchQueryCommand[]`
- [ ] Make transaction callbacks return a `Promise`
- [ ] Update metadata array access to keyed record access
- [ ] Verify boolean, integer, and write-result assumptions
- [ ] Reapply native compile flags, app groups, and TypeORM configuration

## Pointers

- Source: `package/src/index.ts`, `package/src/types.ts`, `package/src/NitroSQLiteError.ts`
- Related: [queries.md](./queries.md), [setup.md](./setup.md), [typeorm.md](./typeorm.md)
