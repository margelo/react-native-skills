---
id: migration-and-errors
title: Error handling and migrating from react-native-quick-sqlite
scope: react-native-nitro-sqlite
keywords: NitroSQLiteError, error handling, instanceof, fromError, try catch, react-native-quick-sqlite, migration, deprecated, 9.0.0, breaking changes, QuickSQLite, open
---

# Error handling and migrating from react-native-quick-sqlite

## Error handling — `NitroSQLiteError`

Every operation that can fail throws (or rejects with) a `NitroSQLiteError`. The library wraps native and JS errors through `NitroSQLiteError.fromError`, preserving the message, `cause`, and stack where possible.

```ts
import { open, NitroSQLiteError } from 'react-native-nitro-sqlite'

try {
  const db = open({ name: 'app.sqlite' })
  await db.executeAsync('SELECT * FROM does_not_exist')
} catch (e) {
  if (e instanceof NitroSQLiteError) {
    console.warn('SQLite error:', e.message)
    console.warn('caused by:', e.cause)   // original error, when available
  } else {
    throw e
  }
}
```

`NitroSQLiteError`:
- `name` is always `'NitroSQLiteError'`.
- `instanceof NitroSQLiteError` works (the prototype chain is maintained).
- `.cause` carries the original error when one was wrapped.
- Static `NitroSQLiteError.fromError(unknown)` converts any thrown value into a `NitroSQLiteError` (used internally; useful if you re-wrap).

### Errors you'll commonly see

| Message | Meaning | Fix |
|---|---|---|
| `Database <name> is already open...` | `open()` called twice for the same name | Reuse the existing connection (singleton) |
| `Database <name> is not open...` | `executeBatch`/`executeBatchAsync`/`transaction` after `close()`, or never opened | `open()` first; don't use a closed connection |
| `...is busy with another operation.` | Synchronous `executeBatch` while a queued async batch/transaction is pending | `await` the async op, or use `executeBatchAsync` — see [concurrency.md](./concurrency.md) |
| `Cannot execute query on finalized transaction...` | Using `tx` after `commit()`/`rollback()` | Don't touch `tx` once finalized — see [transactions.md](./transactions.md) |
| SQL syntax / constraint errors | The SQL itself failed | Inspect `e.message` / `e.cause` |

Async errors reject the promise (catch with `try/await` or `.catch`); sync errors throw synchronously.

## Migrating from `react-native-quick-sqlite`

`react-native-quick-sqlite` is **deprecated** in favor of this package. From major version **9.0.0**, the package name is `react-native-nitro-sqlite`. The JS API is intentionally close, so migration is mostly mechanical.

### 1. Swap the dependency

```bash
npm uninstall react-native-quick-sqlite
npm install react-native-nitro-sqlite react-native-nitro-modules
npx pod-install
```

`react-native-nitro-modules` is now a **required peer dependency** (it wasn't for quick-sqlite). Requires RN 0.75+ / New Architecture — see [setup.md](./setup.md).

### 2. Update imports

```ts
// before
import { open } from 'react-native-quick-sqlite'
// after
import { open } from 'react-native-nitro-sqlite'
```

The `open({ name, location })` → `NitroSQLiteConnection` pattern is the same. `execute`, `executeAsync`, `executeBatch`, `executeBatchAsync`, `transaction`, `attach`, `detach`, `loadFile`, `close`, `delete` all carry over.

### 3. Result shape

Results expose `results` (plain `Row[]`), `rows` (`_array` / `length` / `item`), `rowsAffected`, `insertId`, and optional `metadata` — see [queries.md](./queries.md). If your quick-sqlite code read `rows._array` / `rows.item(i)`, that still works.

### 4. TypeORM users

The driver export is `typeORMDriver` and the babel alias is the same `react-native-sqlite-storage` → (now) `react-native-nitro-sqlite`. See [typeorm.md](./typeorm.md).

### 5. Re-check the concurrency model

Behavior around the per-database queue and the **"database is busy"** error is the area most likely to surface a latent bug after migrating. Audit any not-awaited async batch/transaction immediately followed by a synchronous `executeBatch`. See [concurrency.md](./concurrency.md).

### 6. Native compile flags / system SQLite

Quick-sqlite users who set FTS5 flags, `*_USE_PHONE_VERSION`, or app groups should re-apply them with the nitro-sqlite names: `NITRO_SQLITE_USE_PHONE_VERSION`, `nitroSqliteFlags`, target `RNNitroSQLite`, `RNNitroSQLite_AppGroup`. See [setup.md](./setup.md).

## Gotchas

- **Don't catch and silently drop errors inside a `transaction` callback** — that prevents the auto-rollback.
- **Check `instanceof NitroSQLiteError`** rather than matching message strings where possible.
- **Bug fixes for `react-native-quick-sqlite@8.x` continue for a limited time only** — plan the migration.

## Pointers

- Source: `package/src/NitroSQLiteError.ts`, `package/src/index.ts`
- Related: [concurrency.md](./concurrency.md), [setup.md](./setup.md), [typeorm.md](./typeorm.md)
